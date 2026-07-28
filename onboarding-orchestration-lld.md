# Low-Level Design — Onboarding v2 → Data Lake → v1 Conversion Orchestration Service

**Document version:** 1.3 · **Status:** Draft for review · **Audience:** Engineering, Ops, Architecture Review Board

**Changes in v1.2:**
- **Kafka removed entirely.** The Onboarding System hands cases to the Orchestrator over a **synchronous REST API** with idempotency and durable acceptance. Rationale: the available Kafka community setup is multi-partition without a per-key FIFO guarantee the team is confident in; a REST + Postgres design gives stronger, orchestrator-enforced ordering with one less piece of infrastructure.
- **Ordering is now enforced by the orchestrator itself** via a Postgres-based sequencer (§5.4) — stronger than partition ordering, and immune to out-of-order arrival caused by caller retries.

**Changes in v1.3:**
- **Sequencing is now CLIENT-level, not case-level.** Business rule: for one client, no event may start processing until the previous event for that client has fully completed — even across *different cases*. The sequencer key becomes `clientId`; every case/version for a client is processed strictly serially, while different clients run in parallel. Enforcement stays in the **orchestrator** (not the Onboarding System) because only the orchestrator knows when an event is *truly* terminal — including retries, dead-letters, and ops replays hours later. Onboarding-side gating is welcome as defense-in-depth but is not the control (§5.4.6).
- Onboarding may optionally supply a monotonic per-client `clientSequence` so ordering is semantically exact; without it the orchestrator serializes by arrival order (§5.4.2).

**Carried from v1.1:** all internal orchestration is Postgres-driven (work queue, retries, dead-lettering); Data Lake ingestion confirmation is the **journalId** issued by the lake, which gates progress and anchors reconciliation.

---

## 1. Context & Scope

The bank is migrating its client onboarding platform from **v1 to v2**. During coexistence, downstream consumers still depend on the **v1 message format**, and Core Banking System A (CBS-A) must receive onboarding data in parallel. When a case is **approved** in the v2 Onboarding System, the approved client data must be:

1. Published to the **Central Data Lake** in **v2.1 format** — up to 9 schemas per case, delivered as a map of `schemaName → schemaJson`. The lake issues a **journalId per successful ingestion**, recorded as proof.
2. Retrieved back from the Central Data Lake — **always all 9 schemas**, regardless of how many were in the triggering request (absent ones hydrated from the lake's current state or defaulted).
3. Converted from the 9 v2.1 schemas into the **13 v1 schemas** (mapping rules out of scope; pluggable component).
4. The v1 output published back to the Data Lake (again receiving journalIds), and **in parallel** delivered to **CBS-A**.

All of steps 1–4 are owned by an **independent Orchestration Service**. The Onboarding System's only responsibility is to call this service's REST API once a case is approved and retry until it receives a durable `202`. No message broker is used anywhere in the solution.

**Out of scope:** field-level v2.1→v1 mapping rules, the approval workflow inside Onboarding, Data Lake internals.

---

## 2. Tech Stack

| Layer | Choice | Notes |
|---|---|---|
| Language / Runtime | Java 17 (LTS) | Spring ecosystem baseline |
| Framework | Spring Boot 3.x | REST ingestion API, REST clients, scheduling, actuator |
| Onboarding → Orchestrator | **REST over HTTPS** (`POST /api/v1/orchestrations`) with `Idempotency-Key` | Durable 202 acceptance; caller retries until 202 |
| Internal orchestration | **PostgreSQL 15 work queue** (state machine + jobs, `FOR UPDATE SKIP LOCKED`) | Retries and dead-lettering are DB tables |
| Persistence | Spring Data JPA + Flyway | State, payloads, journal tracking, sequencer, audit, recon |
| Lake / CBS-A integration | Spring `RestClient` to Data Lake ingest/read APIs and CBS-A API | Lake ingest returns/exposes **journalId** |
| Resilience | Resilience4j | Retry, circuit breaker, bulkhead per integration |
| Validation | JSON Schema validation (9 v2.1 + 13 v1 schemas versioned in-repo) | Fail fast on permanent data errors |
| Observability | Micrometer → Prometheus + Grafana; structured JSON logs with correlationId | |
| Ops UI | React 18 + TypeScript (Vite), served by Nginx | Tracking, dead-letter replay, reconciliation |
| Deployment | Linux VMs, systemd fat JARs, ≥2 instances active-active behind LB/VIP | DB locks coordinate all workers |
| Singleton scheduling | ShedLock (Postgres-backed) | Recon, stuck-state detector, sequencer promoter |

**Why this holds together without a broker:** durability is achieved at the *acceptance boundary* — the API commits the case to Postgres before returning 202, and the caller retries until it gets that 202 (ideally from its own outbox). From that instant the orchestrator owns delivery, and the `SKIP LOCKED` job table provides everything internal messaging needs: durable hand-offs, per-stage backoff retries, dead-lettering, multi-instance safety, and full queryability. At onboarding volumes (low throughput, high value) this is comfortably within Postgres capacity, and FIFO becomes a *database guarantee* rather than a broker property.

---

## 3. High-Level Component View

```
┌──────────────────┐  HTTPS POST /api/v1/orchestrations    ┌──────────────────────────────────────────────┐
│ Onboarding  v2   │ ────────────────────────────────────► │      ORCHESTRATION SERVICE (Spring Boot)     │
│ (case approved)  │  Idempotency-Key, retry until 202     │                                              │
└──────────────────┘  ◄──────────────────────────────────  │  [A] Ingestion API + Validator + Sequencer   │
                        202 {orchestrationId} / 4xx        │  [B] Stage Workers (Postgres work queue):    │
                                                           │      B1 Lake Publisher v2.1  (REST)          │
                                                           │      B2 Journal Confirmer                    │
                                                           │      B3 Hydrator (fetch ALL 9)   (REST)      │
                                                           │      B4 Conversion Engine (9 → 13)           │
                                                           │      B5 Lake Publisher v1     (REST)  ─┐     │
                                                           │      B6 CBS-A Publisher       (REST)  ─┤ ∥   │
                                                           │  [C] State Machine + Audit (Postgres)        │
                                                           │  [D] Retry/Dead-letter Manager (Postgres)    │
                                                           │  [E] Recon, Stuck-State, Sequencer schedulers│
                                                           │  [F] Ops API  ◄──── React Dashboard          │
                                                           └──────────────┬──────────────┬────────────────┘
                                                                REST      │              │  REST
                                                     ┌────────────────────▼───┐   ┌──────▼─────────────┐
                                                     │  Central Data Lake     │   │ Core Bank System A │
                                                     │  ingest API → journalId│   └────────────────────┘
                                                     │  read/query API        │
                                                     └────────────────────────┘
```

### 3.1 Ingestion API Contract (the only Onboarding-facing interface)

**`POST /api/v1/orchestrations`**

| Item | Value |
|---|---|
| Headers | `Idempotency-Key: {caseId}-{caseVersion}`, `X-Correlation-Id` (generated if absent), OAuth2 bearer / mTLS per bank standard |
| Body | `{ caseId, clientId, caseVersion, operation: ONBOARD\|MODIFY, clientSequence?, schemas: { schemaName: {...} } }` — 1..9 entries. `clientSequence` (optional, recommended): monotonic per-client event number assigned by Onboarding at approval time; when present the orchestrator processes the client's events in `clientSequence` order and can detect gaps; when absent it serializes by arrival order |
| `202` | `{ orchestrationId, status }` — case durably committed; orchestrator owns it from here. A duplicate `Idempotency-Key` returns the **original** 202 body (safe caller retries) |
| `400 / 422` | Permanent validation failure (unknown schema name, JSON Schema violation, missing identifiers) with machine-readable error detail — the caller owns the fix; the rejection is also recorded for recon (§4 `inbound_rejection`) |
| `409` | Same `caseId+caseVersion` previously accepted with a *different* payload hash — flagged, never silently overwritten |
| `5xx` | Nothing committed — caller must retry (their retry policy: exponential backoff until 202/4xx) |
| `GET /api/v1/orchestrations/{id}` | Status polling for the Onboarding side if it wants to track completion |

**Caller-side requirement (agreed contract):** Onboarding must treat the handoff as mandatory-delivery — retry on timeout/5xx until a 202 or 4xx, ideally driven from an outbox on their side. Recon check 1 (§8.2) is the safety net for anything that still slips.

### 3.2 Data Lake API Contract (assumed — confirm in §11)

`POST /ingest/{schemaName}` → `202 { "journalId": "J-..." }` on accepted ingestion. If the lake confirms asynchronously, it exposes `GET /ingest/status/{requestId}` → `{ status, journalId }` and the Journal Confirmer polls it (both supported; sync is primary). `GET /client/{clientId}/schema/{schemaName}` returns the latest stored version with its serving `journalId`. The same ingest API is used for the 13 v1 schemas (versioned path, e.g. `/ingest/v1/{schemaName}`).

---

## 4. Data Model (PostgreSQL)

```sql
CREATE TABLE orchestration_case (
    orchestration_id   UUID PRIMARY KEY,
    case_id            VARCHAR(64)  NOT NULL,
    client_id          VARCHAR(64),
    case_version       INT          NOT NULL,
    client_sequence    BIGINT,                           -- optional monotonic per-client order from Onboarding
    seq_no             BIGSERIAL,                        -- arrival order fallback for the sequencer
    operation          VARCHAR(16)  NOT NULL,            -- ONBOARD | MODIFY
    status             VARCHAR(32)  NOT NULL,            -- state machine, §5.2 (incl. QUEUED_BEHIND_PRIOR)
    schemas_received   JSONB        NOT NULL,            -- names of the 1..9 schemas present in the request
    request_hash       CHAR(64)     NOT NULL,            -- detects conflicting resubmission (409)
    correlation_id     VARCHAR(64)  NOT NULL,
    idempotency_key    VARCHAR(128) NOT NULL UNIQUE,
    received_at        TIMESTAMPTZ  NOT NULL DEFAULT now(),
    completed_at       TIMESTAMPTZ,
    last_error         TEXT,
    UNIQUE (case_id, case_version)
);
CREATE INDEX ix_case_status ON orchestration_case (status, received_at);
CREATE INDEX ix_case_seq    ON orchestration_case (client_id, client_sequence, seq_no);  -- client-level sequencer scans

-- Rejected inbound requests (4xx) — kept for recon visibility, not for processing
CREATE TABLE inbound_rejection (
    rejection_id   UUID PRIMARY KEY,
    case_id        VARCHAR(64),
    case_version   INT,
    correlation_id VARCHAR(64),
    http_status    INT NOT NULL,
    error_detail   JSONB,
    payload        JSONB,
    received_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE case_payload (
    orchestration_id UUID REFERENCES orchestration_case,
    stage            VARCHAR(16) NOT NULL,               -- INBOUND_V21 | HYDRATED_V21 | CONVERTED_V1
    payload          JSONB NOT NULL,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (orchestration_id, stage)
);

-- Internal work queue: the transport of the whole pipeline.
CREATE TABLE orchestration_job (
    job_id           UUID PRIMARY KEY,
    orchestration_id UUID NOT NULL,
    job_type         VARCHAR(32) NOT NULL,   -- PUBLISH_LAKE_V21 | CONFIRM_JOURNAL | HYDRATE
                                             -- | CONVERT | PUBLISH_LAKE_V1 | PUBLISH_CBS_A
    schema_name      VARCHAR(64),            -- set for per-schema jobs
    status           VARCHAR(16) NOT NULL DEFAULT 'PENDING',
                                             -- PENDING | IN_PROGRESS | DONE | RETRY_WAIT | DEAD
    attempts         INT NOT NULL DEFAULT 0,
    max_attempts     INT NOT NULL,
    next_attempt_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    locked_by        VARCHAR(64),
    locked_at        TIMESTAMPTZ,
    last_error       TEXT,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    finished_at      TIMESTAMPTZ
);
CREATE INDEX ix_job_poll ON orchestration_job (job_type, status, next_attempt_at);

-- Journal tracking: proof of every lake ingestion, both directions.
CREATE TABLE lake_journal (
    orchestration_id  UUID NOT NULL,
    message_version   VARCHAR(8)  NOT NULL,  -- V21 | V1
    schema_name       VARCHAR(64) NOT NULL,  -- one of 9 (V21) or 13 (V1)
    journal_id        VARCHAR(128),          -- NULL until lake confirms
    ingest_request_id VARCHAR(128),          -- if lake confirms asynchronously
    confirmed_at      TIMESTAMPTZ,
    hydration_source  VARCHAR(16),           -- V21 rows: MESSAGE | LAKE_QUERY | DEFAULTED
    PRIMARY KEY (orchestration_id, message_version, schema_name)
);
CREATE UNIQUE INDEX ux_journal_id ON lake_journal (journal_id) WHERE journal_id IS NOT NULL;

-- Dead-letter store for all stages
CREATE TABLE dead_letter (
    dead_letter_id   UUID PRIMARY KEY,
    orchestration_id UUID,
    job_id           UUID,
    job_type         VARCHAR(32) NOT NULL,
    error_class      VARCHAR(255),
    error_detail     TEXT,
    payload_ref      VARCHAR(32),            -- which case_payload stage to replay from
    status           VARCHAR(16) NOT NULL DEFAULT 'OPEN',   -- OPEN | REPLAYED | IGNORED
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_by      VARCHAR(64),
    resolved_at      TIMESTAMPTZ
);

CREATE TABLE orchestration_audit (
    audit_id         BIGSERIAL PRIMARY KEY,
    orchestration_id UUID NOT NULL,
    event_type       VARCHAR(64) NOT NULL,   -- STATE_CHANGE, JOB_RETRY, DEAD_LETTER, REPLAY,
                                             -- SEQUENCE_HELD, SEQUENCE_RELEASED, RECON_MISMATCH ...
    from_status      VARCHAR(32),
    to_status        VARCHAR(32),
    detail           JSONB,                  -- includes journalIds when relevant
    actor            VARCHAR(64) NOT NULL DEFAULT 'SYSTEM',
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE recon_run (
    run_id      UUID PRIMARY KEY,
    window_from TIMESTAMPTZ NOT NULL,
    window_to   TIMESTAMPTZ NOT NULL,
    status      VARCHAR(16) NOT NULL,        -- RUNNING | CLEAN | MISMATCHES | FAILED
    started_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    finished_at TIMESTAMPTZ
);

CREATE TABLE recon_finding (
    finding_id       UUID PRIMARY KEY,
    run_id           UUID REFERENCES recon_run,
    orchestration_id UUID,
    case_id          VARCHAR(64),
    finding_type     VARCHAR(48) NOT NULL,   -- MISSING_ORCHESTRATION, MISSING_JOURNAL_V21,
                                             -- MISSING_JOURNAL_V1, CBS_NOT_DELIVERED,
                                             -- JOURNAL_UNKNOWN_TO_LAKE, STUCK_STATE,
                                             -- ORPHAN_IN_LAKE, SEQUENCE_GAP
    detail           JSONB,
    resolution       VARCHAR(24) NOT NULL DEFAULT 'OPEN',
    resolved_by      VARCHAR(64),
    resolved_at      TIMESTAMPTZ
);
```

---

## 5. Happy Path — End-to-End Flow

### 5.1 Step-by-Step

**Step 1 — Onboarding calls the ingestion API.**
On case approval, Onboarding issues `POST /api/v1/orchestrations` with `Idempotency-Key: {caseId}-{caseVersion}` and retries on timeout/5xx until it receives a 202 (or a 4xx it must fix). Its responsibility ends at the 202.

**Step 2 — Synchronous validation, durable acceptance, sequencing.**
In one local DB transaction the API: checks the idempotency key (duplicate ⇒ return the original 202 — no reprocessing); validates schema names against the allowed set of 9 and each payload against its JSON Schema (failure ⇒ 422 + `inbound_rejection` row, nothing enters the pipeline); inserts `orchestration_case`, `case_payload(INBOUND_V21)`, and one `lake_journal(V21)` row per schema **in the full set of 9**. Then the **client sequencer gate** (§5.4): under the client's advisory lock, if no earlier event for this `clientId` (any case, any version) is still non-terminal, status = `ACCEPTED` and the `PUBLISH_LAKE_V21` jobs (one per received schema) are enqueued; otherwise status = `QUEUED_BEHIND_PRIOR` and no jobs are created yet. Commit, return `202 {orchestrationId}`. Either way the case is durable — a crash one millisecond after commit loses nothing.

**Step 3 — Publish v2.1 to the lake, capture journalIds.**
Stage workers poll their job type with `SELECT ... FOR UPDATE SKIP LOCKED` (multi-instance safe). Each `PUBLISH_LAKE_V21` job calls the lake ingest API for its schema; on `202 {journalId}` the journalId is stored in `lake_journal`, the job marked `DONE`, the journalId audited. If the lake confirms asynchronously, the worker stores `ingest_request_id` and enqueues a `CONFIRM_JOURNAL` polling job. **The journalId is the proof of ingestion — no journalId, no progress.** When every *sent* schema has its journalId, status → `LAKE_CONFIRMED` (guarded conditional update by whichever worker finishes last) and a `HYDRATE` job is enqueued.

**Step 4 — Hydration: assemble ALL 9 schemas.**
The Hydrator enforces the core rule — conversion always sees exactly 9 schemas:

1. **Schema was in the inbound request:** confirm the round-trip by reading `clientId + schemaName` from the lake and checking the served `journalId` **equals the one recorded at ingestion**. Match ⇒ use the lake-returned payload (source `MESSAGE`). Mismatch (read side lagging ingest) ⇒ `RETRY_WAIT` with backoff; persistent mismatch dead-letters as `JOURNAL_UNKNOWN_TO_LAKE`.
2. **Schema absent from the request:** read the latest stored version from the lake (source `LAKE_QUERY`), recording its serving journalId for lineage — the typical MODIFY-flow path where only changed schemas were re-sent.
3. **Lake has never held this schema for this client:** apply the registered default/empty structure (source `DEFAULTED`) so the conversion input always has exactly 9 keys. Defaults are business-approved per schema (§11).

Persist `case_payload(HYDRATED_V21)` with per-schema `hydration_source`; status → `HYDRATED`; enqueue `CONVERT`.

**Step 5 — Conversion v2.1 (9) → v1 (13).**
The stateless `ConversionEngine` produces the 13 v1 payloads; each validates against its v1 JSON Schema before acceptance. Persist `case_payload(CONVERTED_V1)`, stamp `conversionRuleVersion` into the audit, status → `CONVERTED`, and in the same transaction enqueue the **two parallel legs**: 13 `PUBLISH_LAKE_V1` jobs and 1 `PUBLISH_CBS_A` job, with 13 `lake_journal(V1)` rows awaiting journalIds.

**Step 6 — Parallel dual publish, then release the client's next event.**
The v1 lake workers publish each of the 13 schemas and record returned journalIds exactly as in step 3. The CBS-A worker delivers the CBS-A message (assumed REST) and records the ack reference. The legs retry independently — a CBS-A outage never delays the v1 lake leg and vice versa. All 13 v1 journalIds captured ⇒ `v1Complete`; CBS-A acked ⇒ `cbsComplete`; both ⇒ status `COMPLETED`. On reaching a terminal state, the completing transaction **releases the sequencer**: under the client's advisory lock, if any `QUEUED_BEHIND_PRIOR` event exists for the same `clientId`, the next one in order (lowest `clientSequence`, else earliest arrival) flips to `ACCEPTED` and its publish jobs are enqueued (audited as `SEQUENCE_RELEASED`).

### 5.2 State Machine

```
      (earlier event for this CLIENT in flight?)
ACCEPTED ◄── QUEUED_BEHIND_PRIOR
    │
    ▼
PUBLISHING_V21 → LAKE_CONFIRMED → HYDRATED → CONVERTED → PUBLISHING_FINAL → COMPLETED ──┐
    │                 │              │           │              │                       │ terminal ⇒ release
    └─────────────────┴──────────────┴───────────┴──────────────┴─► FAILED_<STAGE>* ────┤ next queued event
                                                            │                           ▼ for the SAME client
                                            V1_DONE_CBS_PENDING / CBS_DONE_V1_PENDING
                                        (* FAILED releases only after ops replay/supersede)
```

Every transition is `UPDATE ... WHERE orchestration_id=:id AND status=:from` plus an audit insert in the same transaction — concurrent workers and caller retries collapse to no-ops.

### 5.3 Idempotency Guarantees

| Point | Mechanism |
|---|---|
| Caller retries (timeout/5xx) | `Idempotency-Key` unique constraint ⇒ original 202 replayed, nothing reprocessed |
| Conflicting resubmission | `request_hash` compare ⇒ 409, flagged for investigation, never silently overwritten |
| Lake ingest duplicates | Worker crash after lake call, before commit ⇒ retry re-ingests; newer journalId supersedes. If the lake accepts a client dedupe key we send `orchestrationId+schemaName` for strict idempotency; else confirmed last-write-wins per client+schema (§11) |
| Job execution | `FOR UPDATE SKIP LOCKED` + lease ⇒ exactly one instance runs a job at a time |
| CBS-A | Message carries `orchestrationId` + `caseVersion` for downstream dedupe |

### 5.4 FIFO Without a Broker — the Client Sequencer

The business rule is **client-level**: for one client, event N must fully reach the lake, v1, and CBS-A before event N+1 starts — across cases, not just within one case. Two completed cases for client 1 are processed strictly one after the other; at no point does the second start while the first is in flight. The sequencer makes this a database invariant:

1. **Gate at acceptance, keyed by `clientId`:** a new event enters `QUEUED_BEHIND_PRIOR` if *any* event for the same client (any caseId, any version) is non-terminal. The check runs under a per-client advisory lock (`pg_advisory_xact_lock(hash(clientId))`), so two concurrent POSTs for the same client cannot both pass the gate; different clients never contend.
2. **Order definition:** if Onboarding supplies `clientSequence` (recommended), events run in that order and gaps are detectable. Without it, order = arrival order at the orchestrator (`seq_no`), which matches approval order as long as Onboarding submits sequentially per client — one reason enforcement belongs here and a per-client sequence from the source is worth having (§11.1).
3. **Release at termination:** the transaction that moves an event to a terminal state (`COMPLETED`, or `FAILED_*` explicitly resolved by ops) promotes the next queued event for that client, under the same advisory lock. Promotion is atomic with termination — there is no instant where two events for one client are both live.
4. **Out-of-order arrival is fine:** if the event with `clientSequence=7` arrives before 6 (caller retry timing), both are accepted; 7 waits until 6 arrives and completes. A ShedLock scheduler re-checks queued events every minute (covers a crash between "terminal" and "promote") and raises `SEQUENCE_GAP` if a queued event waits behind a sequence number that never arrives within a grace period — ops decides: wait, or force-release with audit.
5. **Failure policy:** a `FAILED_*` event **blocks** all later events for that client by default — processing event N+1 on top of a half-applied event N is exactly what the business rule forbids. Ops resolve by replaying or explicitly superseding the failed event; both are audited operator actions, and release happens automatically on resolution.
6. **Why the orchestrator, not the Onboarding System:** Onboarding could throttle submissions (hold case 2 until `GET /orchestrations/{id}` shows case 1 `COMPLETED`), and that is welcome defense-in-depth. But it cannot be the control: only the orchestrator knows true terminality across retries, dead-letters, and replays performed by ops hours later; an Onboarding-side gate would also have to survive its own restarts and would race its own retry loop. Enforcement at the point of processing is authoritative, auditable (`SEQUENCE_HELD`/`SEQUENCE_RELEASED` events per case), and visible on the dashboard as the client's event chain.

Strict-serial per client; unrelated clients proceed fully in parallel — throughput is unaffected because parallelism in this workload is across clients, not within one client.

---

## 6. Error Handling & Retries

### 6.1 Failure Taxonomy

| Class | Examples | Strategy |
|---|---|---|
| **Transient** | Lake/CBS-A 5xx or timeout, DB blip, LB hiccup on inbound | Automatic retry, exponential backoff + jitter (job `RETRY_WAIT`; inbound: caller retries to 202) |
| **Recoverable-with-delay** | JournalId not yet issued (async lake), lake read serving an older journal | `CONFIRM_JOURNAL` polling / hydration backoff |
| **Permanent (data)** | JSON Schema validation failure, unknown schema name, conversion rule error, invalid v1 output | Inbound ⇒ 422 to caller + `inbound_rejection`; pipeline ⇒ dead-letter immediately, alert, fix, replay |
| **Permanent (config)** | Bad credentials, unauthorized lake endpoint, unregistered schema | Circuit opens, ops alert; replay after fix |
| **Sequencing anomalies** | Sequence gap for a client, conflicting resubmission (409), failed event blocking a client's queue | Flag + audit; ops decision (force-release / supersede / replay) — never silent |

### 6.2 Job-Level Retry Policy (Postgres-driven)

A failed attempt sets `status='RETRY_WAIT'`, increments `attempts`, computes `next_attempt_at = now() + min(base·2^attempts, cap) + jitter`; the normal poller resumes it when due. After `max_attempts` the job goes `DEAD`, a `dead_letter` row is written, the case moves to `FAILED_<STAGE>`, and an alert fires.

| Job type | Backoff (default) | Max attempts | Notes |
|---|---|---|---|
| PUBLISH_LAKE_V21 / PUBLISH_LAKE_V1 | 1s → 2s → 4s … cap 5 min | 10 | Per schema — one bad schema never blocks the other 8/12 |
| CONFIRM_JOURNAL | 5s → cap 2 min | 30 (~1h) | Async-lake path only |
| HYDRATE | 30s → cap 15 min | 12 (~2h) | Long cap absorbs lake read-side lag behind ingestion |
| CONVERT | transient infra only: 3 × 10s | 3 | Deterministic rule/data errors dead-letter on first attempt |
| PUBLISH_CBS_A | 2s → cap 10 min | 15 | Fully independent of the v1 lake leg |

A **lock-reaper** (per instance, every minute) resets jobs stuck `IN_PROGRESS` past their lease (10 min) back to `PENDING`, covering worker/VM death mid-job — safe because every stage is idempotent.

### 6.3 Circuit Breakers & Bulkheads (Resilience4j)

Independent circuit breakers for **lake ingest**, **lake read**, and **CBS-A** (50% failure over a 20-call window ⇒ open 60s; half-open probes 3). While open, workers push `next_attempt_at` forward instead of consuming attempts — a 2-hour lake outage burns zero retry budget. Thread-pool bulkheads isolate the three integrations from each other and from the ingestion API's request threads, so a slow lake can never make the Onboarding-facing API unresponsive.

### 6.4 Journal Confirmation Discipline

A sent schema with no journalId within its retry budget dead-letters as `MISSING_JOURNAL_V21`; the case **never** advances to hydration on partial confirmation — the journalId is the contract of ingestion, and hydrating around a missing one would convert data the lake never stored.

### 6.5 Stuck-State Detector

ShedLock-guarded scan (every 5 min) for cases in non-terminal states past SLA with **no live job** — the orphan-case symptom of a missed enqueue or a crash in a gap. It re-enqueues the correct job, audits `STUCK_STATE_RECOVERED`, and alerts on repetition. The sequencer safety valve (§5.4.4) covers the queued-event equivalent.

### 6.6 Dead-Letter Handling & Replay

The `dead_letter` table plus `inbound_rejection` is the complete error inventory — one place, fully queryable. Ops replay via `POST /api/v1/orchestrations/{id}/replay?fromStage=HYDRATE`: status reset to the chosen stage, fresh jobs enqueued from persisted stage payloads, dead letters flipped to `REPLAYED`, audit records user + mandatory reason. Replays are always safe: stages are idempotent and re-runnable from persisted inputs; re-ingestion simply yields fresh journalIds that supersede the old ones.

### 6.7 Failure-Point Walkthrough

| Failure point | What happens | Data loss? |
|---|---|---|
| Orchestrator down when Onboarding calls | Caller gets timeout/5xx and retries per contract until 202; nothing half-committed | No |
| Crash right after 202 committed | Case + jobs are in Postgres; any instance's workers pick them up | No |
| Caller retries after receiving 202 (network blip on response) | Idempotency key ⇒ same 202 body replayed | No dupes |
| Lake ingest API down | Publish jobs cycle `RETRY_WAIT`; circuit opens; inbound API keeps accepting (Postgres buffers) | No |
| Lake accepts 5 of 7 sent schemas | 5 journalIds recorded; 2 jobs retry then dead-letter `MISSING_JOURNAL_V21`; no conversion on partial data | No |
| Lake read lags ingest during hydration | Served journalId ≠ recorded ⇒ back off and re-read until aligned | No |
| Conversion fails on bad data | Immediate dead-letter with schema/field context; fix + replay from CONVERT | No |
| CBS-A down for hours | v1 lake leg completes; `V1_DONE_CBS_PENDING`; CBS job retries on its own budget | No |
| Case 2 for client 1 arrives while case 1 in flight | Case 2 waits in `QUEUED_BEHIND_PRIOR`; auto-released the instant case 1 is terminal — never concurrent | Ordered |
| MODIFY arrives while the ONBOARD it modifies is in flight | Same client ⇒ same gate: MODIFY queues behind the ONBOARD | Ordered |
| Events arrive out of order (clientSequence 7 before 6) | Both accepted; 7 held until 6 arrives and completes; `SEQUENCE_GAP` raised if 6 never comes | Ordered |
| Case 1 for client 1 dead-letters | Client 1's queue holds (by design); ops replay/supersede releases it; other clients unaffected | Ordered |
| VM instance dies mid-job | Lease expires ⇒ lock-reaper requeues on the survivor; LB routes API traffic to the survivor | No |

---

## 7. Tracking & Observability

### 7.1 Correlation & Lineage

`correlationId` (from the inbound request, generated if absent) travels through every DB row, log line, lake call header, and the CBS-A message. `lake_journal` provides full lineage per case: up to 9 v2.1 journalIds in (with hydration source per schema) and 13 v1 journalIds out — the evidence chain a migration audit asks for.

### 7.2 Metrics (Micrometer → Prometheus + Grafana)

Key series: `orch_cases_received_total`, `orch_inbound_rejections_total`, `orch_cases_by_status` (gauge — including `QUEUED_BEHIND_PRIOR`, whose growth means client queues are backing up behind slow/failed predecessors — plus a per-client queue-depth gauge), `orch_stage_duration_seconds{stage}`, `orch_job_retries_total{job_type}`, `orch_dead_letter_total{job_type}`, `orch_job_queue_depth{job_type}` and oldest-pending-job age (the queue-lag equivalents), ingestion API latency/error rate (the Onboarding-facing SLO), circuit-breaker states, `orch_journal_confirm_latency_seconds`, `orch_hydration_source_total{source}` — a `DEFAULTED` spike is the early warning of upstream data gaps — and end-to-end p95 (received → COMPLETED).

Alerts: any dead-letter (page), oldest pending job > 5 min, ingestion API 5xx rate > threshold, circuit open > 5 min, cases past state SLA, `QUEUED_BEHIND_PRIOR` older than SLA, `SEQUENCE_GAP`, any single client's queue depth > threshold, recon mismatches, journal-confirm latency > threshold.

### 7.3 React Tracking Dashboard

Served by Nginx on the same VMs against the Ops API (bank SSO/OIDC; viewer vs operator roles). Screens: **Case Search & Timeline** — search by caseId / clientId / correlationId / **journalId**; per-stage timeline; **client event chain** view (all cases/versions for a client in processing order, with sequencer status per event — the direct visualization of the ordering guarantee); a 9-row schema grid (sent?, journalId, confirmed-at, hydration source) and a 13-row v1 grid; payload viewer with role-based PII masking. **Live Operations** — status distribution, queue depths, stage latencies, retry/dead-letter counters, circuit states. **Dead Letters & Replay** — filterable inventory (incl. inbound rejections), error detail, single/bulk replay with mandatory reason, force-release / supersede actions for sequencing anomalies. **Reconciliation** — run history, findings by type, resolve/ignore/replay. **Audit** — immutable per-case log including journalIds, sequencer events, and every ops action.

---

## 8. Reconciliation

Reconciliation answers: *did every approved case get 9 journalIds in, 13 journalIds out, and a CBS-A delivery — exactly once and in version order?* The journalId is the reconciliation currency throughout.

### 8.1 Continuous (built into the flow)

The journalId-gated pipeline is real-time recon of both lake legs: no `LAKE_CONFIRMED` without all sent-schema journalIds, hydration proves the lake *serves* those journals, completion requires all 13 v1 journalIds plus the CBS ack, and the sequencer proves ordering. Gaps surface within minutes as dead letters, not at end of day.

### 8.2 Scheduled Batch Recon (ShedLock singleton — hourly + end-of-day)

Over a sliding window (with overlap for late arrivals):

1. **Onboarding vs Orchestrator:** diff the Onboarding approved-case manifest (`GET /approved-cases?from&to`) against `orchestration_case` **and** `inbound_rejection` → `MISSING_ORCHESTRATION` (approved but never successfully handed over — the safety net for the REST handoff, catching cases where the caller's retry loop failed or was never triggered).
2. **Orchestrator vs Lake, v2.1 leg:** verify every recorded v2.1 journalId against the lake's journal-lookup API → unknown ⇒ `JOURNAL_UNKNOWN_TO_LAKE`; sent schema with no journalId ⇒ `MISSING_JOURNAL_V21`.
3. **v1 leg:** same verification for the 13 v1 journalIds of every `COMPLETED` case → `MISSING_JOURNAL_V1`.
4. **CBS-A:** match CBS-A's ack/receipt feed (API or daily file, per their capability) on `orchestrationId` → `CBS_NOT_DELIVERED`.
5. **Orphans:** lake journal entries for our client population with no matching `lake_journal` row → `ORPHAN_IN_LAKE`.
6. **Sequencing:** per-client event chains with sequence gaps or long-queued successors → `SEQUENCE_GAP`; verify no two events for one client were ever concurrently live (audit-derived check — the ordering attestation for the migration programme); terminal-state check for cases past SLA → `STUCK_STATE`.

Findings persist to `recon_finding`. **Auto-remediation** for allow-listed types (e.g., `MISSING_JOURNAL_V1` with `CONVERTED_V1` payload present ⇒ re-enqueue just that schema's publish, marked `AUTO_REPLAYED`); the rest stays `OPEN` for ops. The EOD run emits the signed-off daily report — cases in, journalIds per leg, CBS deliveries, ordering exceptions, open findings.

### 8.3 Recon Design Notes

The overlap window (re-scan 26h every 24h) absorbs late lake confirmation. Recon lake calls are rate-limited/batched so recon never degrades the hot path. If the lake offers no journal-lookup API, fallback is the read-API compare: fetch `client+schema` and verify the served journalId is ≥ the recorded one (confirm §11).

---

## 9. Deployment on VMs

Minimum two Linux VMs, each running the orchestration-service fat JAR under **systemd** (`Restart=always`, GC logs, heap sized to VM), Nginx serving the React build and proxying `/api`, and node_exporter/actuator scraping. A load balancer or keepalived VIP fronts the ingestion + Ops API — this is now the availability-critical entry point, so health checks (`/actuator/health` with a DB probe) gate LB membership. No broker to operate: work distribution is the inbound LB for API traffic plus `FOR UPDATE SKIP LOCKED` + job leases for workers; ShedLock keeps recon, the stuck-state detector, and the sequencer promoter singleton. Postgres is the bank's HA instance with PITR backups — **the DB is the backbone; its RPO is the service's RPO** — with HikariCP pools sized against worker thread pools. Flyway migrates on startup. Rolling deploys: drain instance A at the LB, let leased jobs finish or lapse, deploy, rejoin, repeat — safe because all work resumes from Postgres. Job-table hygiene: purge/partition `orchestration_job` rows older than N days (jobs are operational; `orchestration_audit` and `lake_journal` are the permanent record). Sizing: onboarding is low-throughput/high-value; a 4 vCPU / 8 GB pair is ample.

---

## 10. Security & Compliance Notes

OAuth2 client-credentials or mTLS (bank standard) on the ingestion API — it is now the only inbound door, so LB-level IP allow-listing to the Onboarding network segment is cheap defense-in-depth; same auth pattern outbound to the lake and CBS-A. PII payloads ⇒ Postgres encryption at rest, role-based masking in the dashboard, retention policy (purge `case_payload` after e.g. 90 days; keep `orchestration_audit` and `lake_journal` per bank record policy — journalIds are the durable evidence). Audit table append-only for the app role; replay / force-release / supersede endpoints operator-role-gated with mandatory reason.

---

## 11. Open Questions / Assumptions to Confirm

1. **Onboarding contract:** confirm (a) retry-until-202 (ideally outbox-driven), (b) the approved-cases manifest for recon check 1, and (c) whether Onboarding can supply a monotonic per-client `clientSequence` at approval — strongly recommended, since it upgrades client ordering from arrival-order to semantically exact and makes gaps detectable. Onboarding-side submission throttling per client is optional defense-in-depth, not the control (§5.4.6).
2. **Lake ingest contract:** journalId synchronous on `202`, or asynchronous (status-poll endpoint)? Both supported; sync is primary.
3. **Journal lookup for recon:** does the lake expose `GET /journal/{id}` or batch verification? Fallback = read-API compare (§8.3).
4. **Lake read semantics:** confirm the read API returns the serving journalId and the typical ingest→readable lag (drives hydration backoff caps).
5. **Ingest idempotency:** client-supplied dedupe key supported? If yes we send `orchestrationId+schemaName`; if no, confirm last-write-wins per client+schema so crash-retry re-ingest is harmless.
6. **CBS-A transport & ack:** assumed REST with sync ack — confirm (MQ/file only changes the adapter) and confirm an ack/receipt feed exists for recon check 4.
7. **Defaulting rules** per schema for never-seen schemas (`DEFAULTED`) need business sign-off.
8. **Failed-event policy:** default is a `FAILED_*` event blocks that client's entire queue until ops replay or supersede (§5.4.5) — confirm with business, since skip-ahead would process a later event over a half-applied predecessor, which is precisely what the client-level ordering rule forbids. Also confirm the target SLA for how long a client's queue may hold before escalation.
9. **SLA numbers** (journal-confirm budget, stage SLAs, queued-version SLA, end-to-end p95) are placeholders pending ops agreement.
