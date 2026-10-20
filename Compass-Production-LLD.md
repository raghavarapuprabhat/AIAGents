# Compass — Production LLD (Phase 3: Productionization)

**Audience:** Implementation engineers / AI-assisted development, platform & security reviewers
**Baseline:** the successful **PoC build** (Compass LLD, PoC edition) plus the **AI layer** (Compass AI LLD, Phase 2). This document specifies only what changes to make that build production-grade.
**Companion documents:** Architecture v1.0 (HLD), Architecture v2.0 (AI edition), Compass LLD (PoC), Compass AI LLD, WalkMe→Compass Migration Plan.

---

## 0. Decisions taken (recorded)

| # | Decision | Consequence in this document |
|---|---|---|
| PD-01 | **Runtime: OpenShift/Kubernetes with Argo CD GitOps** | Part 2: containerization, Helm charts, Argo apps, environment promotion by Git |
| PD-02 | **Analytics store stays PostgreSQL** (ClickHouse revisited at scale) | Part 4: partitioned events table, rollup jobs, retention automation, documented scale triggers |
| PD-03 | **Topology: single DC + DR (active/passive)** | Part 3: DR design, RPO/RTO targets, failover runbook, ME edge as delivery-only cache |
| PD-04 | **Authentication deferred — tracked as a named production gap** | Part 6: interim perimeter controls (mandatory), gap register entry PG-01 with owner and date |
| PD-05 | **Signing keys: HashiCorp Vault Transit** (JKS keystore fallback) | Part 5: sign-as-a-service, key versions, rotation, break-glass |
| PD-06 | **Go-live scope: Phase 1 + all AI tiers incl. C1 assistant** | Part 8: AI production hardening (rate limits, model serving HA, red-team gate, capability register) |

## 0.1 Production gap register (the honest list)

Every deferral is named, owned, and dated. This register is a go-live artifact and is reviewed at each gate.

| ID | Gap | Risk if unaddressed | Interim control (mandatory before go-live) | Target close |
|---|---|---|---|---|
| **PG-01** | **No application authentication** (console + `/api/*` use `X-Dev-User`) | Anyone with network reach can author, approve, and publish signed content into live banking screens; audit actor is self-asserted, which also voids maker-checker as a real control | (a) All console/API routes reachable only from the internal network via the reverse proxy — ideally behind the bank's existing SSO proxy for internal tools; (b) edge→service mTLS or shared-secret header so the origin cannot be bypassed; (c) `X-Dev-User` values restricted to an allow-list of named author accounts held in `tenant_user` config; (d) publish + kill-switch actions additionally restricted by source IP allow-list | **Sprint +2 after go-live** — OIDC per HLD §8; owner: Platform lead. Security-review conditional approval expected, not guaranteed |
| PG-02 | No SIEM integration | Audit exists but isn't monitored | Nightly audit export to the bank's log store; alert on publish/kill events | Sprint +1 |
| PG-03 | ClickHouse deferred (PD-02) | Analytics slow-down beyond ~10M events/day | Partitioning + rollups (Part 4); documented scale triggers | On trigger |
| PG-04 | Second DC (PD-03) | Full-DC loss = guidance outage (fail-open, so no host-app impact) | DR runbook + tested restore; ME edge caches absorb short outages | Next capacity cycle |
| PG-05 | Studio extension distributed manually | Author onboarding friction | Signed CRX + enterprise policy request submitted | Before tenant 2 |

**Explicit statement for the security review:** with PG-01 open, Compass must be treated as an internal-network-only system whose publishing capability is protected by network perimeter and named-account allow-listing, not by user authentication. The maker-checker separation remains enforced structurally (DB constraint) but is only as trustworthy as the allow-listed identities. This is acceptable only as a time-boxed condition with PG-01 closed on the stated date.

---

# PART 1 — What changes from the PoC (delta map)

| Area | PoC | Production |
|---|---|---|
| Runtime | Local processes (`run.sh`) | Containers on OpenShift; Helm + Argo CD; HPA; PodDisruptionBudgets (Part 2) |
| Config/secrets | Local files, `var/keys` | ConfigMaps + Vault-injected secrets; no secret in Git (Part 5) |
| Signing | Local Ed25519 keypair | Vault Transit sign-as-a-service, versioned keys (Part 5) |
| Object store | Local directory | MinIO (or bank S3-compatible), versioning + lifecycle (Part 4) |
| Analytics | Single `events` table | Partitioned + rolled-up PostgreSQL, retention jobs (Part 4) |
| Delivery | Service serves directly | Nginx/OpenShift Route edge cache tier + ME edge nodes (Part 3) |
| Identity | `X-Dev-User` header | Same code path + perimeter controls (PG-01), OIDC-ready seam |
| Media AV | Skipped | ICAP scan reinstated in the media pipeline (Part 6) |
| CORS | Permissive | Per-tenant `allowed_origins` reflection enforced (Part 6) |
| Observability | Actuator only | Prometheus + Grafana dashboards, alert rules, SLOs (Part 7) |
| Backup/DR | None | PG PITR, object-store replication, tested restores (Part 3) |
| AI serving | Local Ollama single instance | vLLM (or Ollama) on GPU nodes, 2 replicas, queue + rate limits (Part 8) |

Nothing in this list changes wire formats, schemas, the agent's behavior, or the Studio UX. Production is an **operational** upgrade: same application, industrial packaging.

---

# PART 2 — Containerization & GitOps (PD-01)

## 2.1 Images

One image per service (`delivery`, `content`, `targeting`, `identity`, `collector`, `admin`, `ai-gateway`) plus `studio-console` (nginx-served static build) and `rulec` (Node CLI, used as a Job/sidecar by Content).

- Base: bank-approved UBI9 minimal (Java: `ubi9/openjdk-21-runtime`); non-root user; read-only root filesystem; no shell in final layer where feasible.
- Multi-stage builds; dependencies resolved through the bank artifact proxy only (no public registry at build time).
- Every image: SBOM generated (Syft), signed (Cosign), scanned (Trivy/bank scanner) — build fails on High/Critical with no accepted exception.
- Tag = git SHA; `latest` is forbidden. Image digests pinned in Helm values.

## 2.2 Helm chart structure

```
infra/helm/compass/
├── Chart.yaml
├── values.yaml                 # defaults (non-secret)
├── values-{sit,uat,prod}.yaml  # env overrides (image digests, replicas, hostnames)
└── templates/
    ├── {service}-deploy.yaml   # Deployment, Service, ServiceAccount
    ├── {service}-hpa.yaml
    ├── ingress-route.yaml      # OpenShift Routes / Ingress w/ TLS
    ├── configmap-*.yaml
    ├── externalsecret-*.yaml   # Vault → K8s secret (Part 5)
    ├── networkpolicy.yaml      # default-deny + explicit allows (Part 6)
    ├── pdb.yaml
    ├── cronjobs.yaml           # analytics rollup, retention, AI jobs, rag-index
    └── job-flyway.yaml         # migration Job, Helm pre-upgrade hook
```

## 2.3 Deployment shape (prod)

| Workload | Replicas | Requests / Limits | Notes |
|---|---|---|---|
| delivery | 3 | 250m/512Mi → 1/1Gi | Stateless; HPA on CPU 60% |
| content | 2 | 500m/1Gi → 2/2Gi | Publish is CPU-bursty; PDB minAvailable 1 |
| targeting | 2 | 250m/512Mi → 1/1Gi | Invokes `rulec` Job |
| identity | 2 | 250m/512Mi → 1/1Gi | |
| collector | 3 | 500m/512Mi → 2/1Gi | **StatefulSet** — each pod owns its spool PVC (20Gi RWO) |
| admin | 2 | 250m/512Mi → 1/1Gi | |
| ai-gateway | 2 | 500m/1Gi → 2/2Gi | Guardrails CPU-bound; queues to model serving |
| model-serving | 2 | GPU 1 each | Part 8; node selector on GPU pool |
| studio-console | 2 | 100m/128Mi → 500m/256Mi | Static |

**Collector is the one stateful service.** Spool PVCs must never be lost silently: `StatefulSet` with stable pod identity, `persistentVolumeClaimRetentionPolicy: Retain`, and a startup check that adopts any orphaned segments on the mounted volume before serving traffic (behavior already specified in the PoC LLD §3.5 — it now matters operationally).

## 2.4 GitOps (Argo CD)

- Repos: `compass` (code) → CI builds/pushes images and opens a PR to `compass-deploy` (env values with digests). Argo watches `compass-deploy`.
- App-of-apps: one Argo Application per environment; sync policy **automated for SIT**, **manual (gated) for UAT and PROD** — production sync is a human action tied to a change ticket.
- Sync waves: (1) ConfigMaps/ExternalSecrets → (2) Flyway Job → (3) services → (4) CronJobs/Routes. Flyway Job failure blocks the wave, so a bad migration never half-deploys.
- Drift detection on; manual `kubectl edit` in prod is a policy violation and Argo reverts it.
- Rollback: `argocd app rollback` to the prior revision. **Note:** application rollback and *content* rollback are different operations — content rollback stays the manifest re-point (seconds, no deploy).

## 2.5 Migration & release safety

- Migrations must be **backward-compatible for one release** (expand/contract): add columns nullable, backfill in a Job, switch reads, drop in a later release. No destructive DDL in the same release that changes code paths.
- Blue/green not required for stateless services (rolling update, `maxUnavailable: 0`, readiness gates). Collector rolls one pod at a time with spool adoption verified before proceeding.
- Agent versioning: the agent bundle is served from Delivery under `/a/compass-agent-<semver>.js`; host apps pin a version + SRI hash. Publishing a new agent version does **not** change any host app until that app updates its snippet — so agent rollout is per-tenant and reversible by the tenant, and `agent_min` in the manifest is the only global lever (used to force-retire a broken version).

---

# PART 3 — Topology, availability & DR (PD-03)

## 3.1 Target topology

```
                         ┌──────────── DC1 (ACTIVE) ─────────────┐
 users ──HTTPS──► GSLB ──►  OpenShift: edge Route → services      │
                         │  PostgreSQL primary (+ sync standby)   │
                         │  MinIO (versioned)  ·  Vault (HA)      │
                         │  GPU pool: model-serving ×2            │
                         └───────────────┬────────────────────────┘
                                         │ async replication (PG WAL, MinIO)
                         ┌───────────────▼──── DC2 (PASSIVE/DR) ──┐
                         │  OpenShift namespace pre-provisioned    │
                         │  PG standby (async) · MinIO mirror      │
                         │  Services scaled to 0 until failover    │
                         └─────────────────────────────────────────┘

 ME markets:  in-country EDGE NODE = Nginx cache + signed-bundle serving + beacon spool-forwarder
              (delivery-only; no DB, no control plane, no trust required beyond serving bytes)
```

## 3.2 Availability targets & failure semantics

| Scenario | Behavior | User impact |
|---|---|---|
| One pod/node lost | HPA/scheduler reschedules; PDBs hold minimum | None |
| Control plane down (content/targeting/admin) | Delivery + Collector unaffected; authoring paused | **None for end users** — guidance keeps serving from cached bundles |
| Delivery down (all replicas) | Agents fail-open; edge cache serves until TTL, then no guidance | Guidance absent; host apps normal |
| PostgreSQL primary lost | Sync standby promoted (see 3.3) | Authoring pause ~minutes; guidance unaffected |
| Full DC1 loss | DR invocation (3.4) | Guidance absent until DR up (edge caches extend coverage in ME); host apps normal |
| Model serving down (AI) | ai-gateway returns unavailable; console AI affordances disable; assistant Search tab works | Manual paths fully functional (coexistence rule) |

**SLOs (production):** delivery availability **99.9%** monthly; manifest p99 ≤50 ms at edge, bundle p99 ≤150 ms; collector ingest availability 99.9%; publish→visible ≤90 s; C1 assistant p95 ≤6 s (SLO applies only when AI enabled).

Error budget policy: burn >50% of monthly budget ⇒ feature freeze on the agent until the budget recovers (guidance changes still allowed — they're content, not code).

## 3.3 Database HA
- **Intra-DC:** PostgreSQL 16 with a synchronous standby (`synchronous_commit=remote_write`) and Patroni (or the bank's standard PG operator) for automated failover; VIP/service abstraction so services reconnect without config change. Connection pooling via PgBouncer sidecar/service (transaction pooling; note jOOQ uses no session state that breaks this).
- **Cross-DC:** asynchronous streaming replica in DC2.
- Targets: intra-DC failover **RTO ≤2 min, RPO 0**; cross-DC **RTO ≤4 h, RPO ≤15 min**.

## 3.4 Backup, restore & DR
| Asset | Backup | Restore test cadence |
|---|---|---|
| PostgreSQL | pgBackRest full weekly + WAL continuous (PITR) to object storage, 35-day retention | **Quarterly restore drill to a scratch namespace** (mandatory evidence for audit) |
| Object store (bundles, media) | MinIO versioning + async mirror to DC2; lifecycle: bundles retained 180 days after supersession | Quarterly fetch-and-verify of a signed bundle from the mirror |
| Vault | Bank's Vault HA + its own backup regime (out of scope, dependency documented) | Per Vault team |
| Config | Everything in Git (`compass-deploy`) | Argo re-sync = config restore |

**DR runbook (summary, full version `docs/runbooks/dr.md`):** declare → promote DC2 PG standby → scale DC2 services to target replicas → repoint GSLB → verify signed-bundle fetch + publish smoke test → notify tenants. **Key property:** because bundles are immutable, signed, and content-addressed, DR correctness does not depend on replication timing for *reads* — a stale mirror still serves valid, verifiable guidance. Only authoring loses the RPO window.

**Content re-derivability:** bundles are compilations of PostgreSQL state. If the object store were lost entirely, publishing again from the DB regenerates byte-identical bundles (JCS canonical + deterministic compile). This is a documented recovery path and is exercised in the quarterly drill.

## 3.5 ME edge nodes
Delivery-only pods (Nginx + small forwarder) in-region: cache manifests (60 s) and bundles (immutable, long TTL); buffer beacons to local disk and forward to DC1 Collector with retry. Provisioned per market via the same Helm chart with `profile=edge`. Because content is signed, an edge node is untrusted infrastructure by design — compromise yields cached public content, not authoring capability.

---

# PART 4 — Data layer productionization

## 4.1 PostgreSQL as the analytics store (PD-02)

Additive migration set `V200+` (no changes to existing tables' semantics):

```sql
-- V200: convert events to a partitioned table (expand/contract, one release)
CREATE TABLE events_p (LIKE events INCLUDING DEFAULTS INCLUDING CONSTRAINTS)
  PARTITION BY RANGE (ts);
-- monthly partitions created ahead by a CronJob; 13-month retention by DETACH+DROP
CREATE INDEX ON events_p (tenant, e, ts);
CREATE INDEX ON events_p (tenant, g, gv, st, ts);   -- funnel queries
-- V201: rollups (materialized, refreshed by CronJob every 15 min)
CREATE TABLE agg_funnel_daily (tenant text, env text, g text, gv int, st text, d date,
  views bigint, completes bigint, skips bigint, PRIMARY KEY (tenant,env,g,gv,st,d));
CREATE TABLE agg_guide_daily  (tenant text, env text, g text, gv int, d date,
  starts bigint, completions bigint, abandons bigint, p50_ms int, PRIMARY KEY (tenant,env,g,gv,d));
CREATE TABLE agg_anchor_daily (tenant text, anchor_id text, d date,
  resolves bigint, low_conf bigint, fails bigint, PRIMARY KEY (tenant,anchor_id,d));
-- V202: read-only analytics role for Insights queries; no table-level writes
```

Rules: **Insights and the AI Insights copilot query only `agg_*` tables** (bounded, indexed, fast); raw `events_p` is reachable only by the rollup jobs and ad-hoc investigation with the analytics role. This keeps dashboard latency independent of raw volume and is what makes PD-02 viable.

**Documented scale triggers → revisit ClickHouse (PG-03):** any one of — sustained >10M events/day; `agg_*` refresh exceeding 5 min; raw retention >2 TB; or a dashboard p95 >3 s. The `EventSink` interface from the PoC remains in place, so the swap is a sink implementation plus a backfill job, not an application rewrite.

## 4.2 Retention & privacy automation
- CronJobs: monthly partition create/detach; 13-month raw purge; `ai_interaction` purge at 13 months; `rag_chunk` purge for bundles superseded >30 days; survey verbatim purge per tenant policy.
- Per-tenant retention overrides in `tenant.config`; a tenant may set analytics off entirely (guidance unaffected — Architecture v2 §5).
- Pseudonymization keys (per-tenant HMAC) move to **Vault** (Part 5); the re-identification procedure (dual control, documented in `docs/runbooks/reidentification.md`) requires two named approvers and writes an audit event — it exists only for lawful investigation and has never-by-default posture.

## 4.3 Object store
MinIO (or bank S3): versioning on; bucket-per-environment; lifecycle rules (bundle supersession 180 d, media orphan GC 90 d after unreference); server-side encryption; access via short-lived credentials from Vault; **immutability**: bundles written with `if-none-match` semantics and never overwritten (already the PoC behavior — now enforced by bucket policy too).

---

# PART 5 — Secrets & signing (PD-05, Vault)

## 5.1 Vault Transit for bundle signing (primary design)

Vault's **Transit** engine keeps the private key inside Vault — the Content Service never holds it:

- Key: `transit/keys/compass-bundle`, type `ed25519`, `min_decryption_version`/`min_encryption_version` managed by rotation policy; `derived=false`; `exportable=false`, `allow_plaintext_backup=false`.
- Publish pipeline step 4 becomes: compute JCS canonical form → SHA-512 digest → `POST transit/sign/compass-bundle` with `prehashed=true` → receive `vault:v3:<sig>`; store the **key version** in the manifest as `sig_key_id = "compass-bundle:v3"`.
- Agent: pinned key map now holds one public key per Vault key version (`v1..vN`), fetched at build time from `transit/keys/compass-bundle` (public material only) and compiled into the agent bundle. Verification selects by `sig_key_id`.
- **Rotation (quarterly + on demand):** `POST transit/keys/compass-bundle/rotate` → new version `vN+1`; agent release N+1 ships the new public key **before** publishing switches to it (two-release overlap, exactly as the PoC LLD's rotation runbook described); old versions retained for verification of unrolled-back bundles, then `min_encryption_version` raised.
- Break-glass: if Vault is unavailable, **publishing is blocked** (correct behavior — never publish unsigned); guidance delivery is unaffected because existing bundles are already signed. Documented in `docs/runbooks/vault-outage.md`.

**Fallback (only if Transit is unavailable to this team):** software keystore — Ed25519 private key in Vault KV v2, mounted as a file via the Vault Agent injector into the Content pod, loaded into memory at startup, never written to disk by the app. This is weaker (key material transits into the pod) and is a documented exception with the same rotation cadence.

## 5.2 Other secrets in Vault
Database credentials (ideally **dynamic** via Vault's PG secrets engine with short TTL), object-store credentials, per-tenant HMAC pseudonymization keys (KV v2 or Transit HMAC), AI provider API key (only if the external adapter is ever enabled), Vault-issued TLS material where the service mesh doesn't cover it. Delivery mechanism: **External Secrets Operator** or Vault Agent injector — the choice follows bank platform standard; either way **no secret is ever committed to Git** and CI enforces this with a secret scanner on every PR.

---

# PART 6 — Security hardening (incl. PG-01 interim controls)

## 6.1 Reinstated controls (deferred in PoC)
| Control | Production configuration |
|---|---|
| **CORS** | Delivery reflects `Origin` only if it is in `tenant.allowed_origins`; no wildcard; preflight cached 10 min |
| **Media AV** | ICAP call to the bank AV service inserted in the media pipeline before store; fail-closed (upload rejected if scanner unavailable); SVG sanitizer retained and unchanged |
| **Rate limiting** | Edge: per-IP limits on `/api/*` and `/c/e`; service: per-tenant token buckets on publish, media upload, and all `ai-gateway` endpoints |
| **CSP** | Documented per-tenant delta published in the onboarding runbook; Compass's own console served with strict CSP + nonces |
| **Headers** | HSTS, `X-Content-Type-Options`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy` minimal on all Compass-served responses |
| **SIEM (PG-02)** | Nightly `audit_event` + `ai_interaction` export; real-time alert (webhook) on publish, rollback, kill-switch, flag change, AI block spike |
| **Network policy** | Default-deny per namespace; explicit allows: edge→services, services→PG/MinIO/Vault, ai-gateway→model-serving. Model-serving has **no egress** (verified by policy test) |
| **Pod security** | Restricted SCC/PSA: non-root, no privilege escalation, read-only root FS, seccomp `RuntimeDefault`, dropped capabilities |

## 6.2 PG-01: interim controls for the authentication gap (mandatory)
1. **Perimeter:** console and `/api/*` exposed only on the internal network; publicly routable hostname serves **only** `/d/*` (signed bundles, media) and `/c/e` (beacon ingest). Enforced at the Route/ingress level and verified by an automated external-exposure test in CI.
2. **Preferably behind the bank's existing internal SSO proxy** (e.g., the standard reverse-proxy SSO used for internal tools): if available, this closes PG-01 in effect at near-zero application cost, with the app still reading `X-Dev-User` — set by the proxy, not the caller. **Strip inbound `X-Dev-User`/`X-Dev-Role` at the edge unconditionally**, so a client can never assert its own identity.
3. **Named-account allow-list:** new `tenant_user` table (`tenant_id, username, role`); requests with a username not in the allow-list are rejected. Removes anonymous authorship.
4. **Elevated-action IP allow-list:** publish, rollback, kill switch, and AI flag changes additionally require the source IP to be in an operations allow-list.
5. **Audit clarity:** while PG-01 is open, `audit_event.actor` is annotated `(unauthenticated-perimeter)` in exports so no reader mistakes it for a verified identity.
6. **Review checkpoint:** PG-01 status is a standing agenda item at every release gate until closed; the OIDC implementation is pre-specified in HLD §8 so closing it is configuration + wiring, not design.

## 6.3 Assurance activities before go-live
Threat-model refresh (STRIDE) including the AI plane; internal pen test; **external pen test** covering the public surface (`/d/*`, `/c/e`) and the agent; dependency/SCA clean; container image scan clean; DAST on the console; AI red-team suite (Part 8) green; DR restore drill evidence; audit walkthrough with Internal Audit.

---

# PART 7 — Observability & operations

## 7.1 Metrics, dashboards, alerts
- **Metrics:** Micrometer → Prometheus (ServiceMonitor per service). Golden signals per endpoint plus domain metrics: `bundle_publish_seconds`, `manifest_cache_hit_ratio`, `spool_lag_seconds`, `spool_bytes`, `event_insert_failures_total`, `anchor_resolve_outcome_total{outcome}`, `agent_error_total{code}`, `agent_boot_ms` (1% sampled), `ai_calls_total{capability,outcome}`, `ai_latency_ms`, `rag_index_lag_seconds`.
- **Dashboards (Grafana):** *Service health*, *Delivery & cache*, *Ingest & spool*, *Guidance health* (anchor outcomes, drift by app release), *Authoring throughput* (drafts, approvals, publishes), *AI usage & guardrails*, *Tenant view* (per-tenant slice for content owners).
- **Alert rules (paged):** delivery 5xx >1% for 5 min; manifest p99 >200 ms for 10 min; spool lag >15 min or spool >70%; event insert failures >0 for 10 min; publish failures; Vault sign errors; `agent_error` rate >0.5% of sessions; anchor-fail rate >3× 7-day baseline (per tenant); AI block-rate spike or model-serving unavailable (ticket, not page, since manual paths persist).
- **Synthetic monitoring:** every 5 min from inside and outside the perimeter — fetch manifest, fetch bundle, **verify its signature**, POST a synthetic beacon, and run one headless walk-thru against a canary page. Signature verification in the synthetic check is what catches a mis-signed publish before users do.

## 7.2 Logging
Structured JSON (service, tenant, trace_id, actor, route, latency, outcome); no PII, no content bodies; correlation via W3C `traceparent` propagated from edge; retention per bank standard; shipped to the bank log platform (PG-02 covers the audit-specific export).

## 7.3 Runbooks (`docs/runbooks/`, each with a tested drill)
`kill-switch.md` · `content-rollback.md` · `app-rollback.md` · `spool-saturation.md` · `pg-failover.md` · `dr.md` · `vault-outage.md` · `key-rotation.md` · `anchor-drift-storm.md` (a host release broke many anchors — triage, bulk heal, or tenant kill) · `ai-incident.md` (bad AI output reported: disable capability flag, purge affected drafts, review `ai_interaction`) · `reidentification.md` · `tenant-onboarding.md` · `agent-version-retirement.md`.

Each runbook states: trigger, blast radius, decision authority, steps, verification, comms. Drills: kill switch and content rollback **monthly**; PG failover and DR **quarterly**; the rest annually or on change.

## 7.4 Support model
- **L1** (service desk): known-issue KB, "guidance not appearing" triage tree, escalation criteria.
- **L2** (platform team): dashboards, runbooks, anchor heal queue, tenant comms.
- **L3** (engineering): code defects, agent releases.
- Guidance issues are tagged distinctly in the ticket system (a metric the migration plan already relies on). Tenant content owners handle content questions — not the platform team; this boundary is stated in the onboarding pack to prevent the platform team becoming a content help desk.

---

# PART 8 — AI production hardening (PD-06: all tiers at go-live)

Everything from the AI LLD applies; production adds:

## 8.1 Model serving
- **vLLM** (preferred for throughput) or Ollama on the GPU node pool; 2 replicas behind a Service; models **pinned by digest** and pulled from the bank artifact repo (weights vendored — no runtime download). One instruct model (7–8B class) + one embedding model (768-d).
- Queue + admission control at `ai-gateway`: max in-flight per capability, per-tenant QPS, request timeout 20 s (C1: 10 s) — on saturation return unavailable so the UI degrades to manual rather than hanging.
- **No egress** from model-serving pods (network policy verified in CI). External provider adapter remains **disabled**; enabling it requires a config change plus a recorded approval (`ai.external.enabled` is itself audited).
- Capacity: sized for C1 at pilot scale first (assume ≤2% of sessions open the assistant, ≤3 questions each); scale on `ai_latency_ms` p95 and queue depth. Tier-1/2 batch jobs are scheduled off-peak and share the pool with lower priority.

## 8.2 Guardrails & governance in production
- Red-team suite (AI LLD Part 7 §3) is a **release gate**, re-run on every prompt-template change *and* every model version change — a model swap is a code change for review purposes.
- Prompt templates are versioned in Git; changes require review by the capability owner; the template hash logged per call means any produced output is reproducible.
- **Capability register** (Architecture v2 §5) generated weekly from `ai_capability_flag` + `ai_interaction` volumes: what is on, per tenant, with usage and block rates — the artifact for the AI governance forum.
- Kill controls: per-capability flag off (immediate), `ai-gateway` scaled to 0 (hard stop), and for C1 the manifest `features.assist_ai=false` (agents drop the Ask tab within 60 s, Search tab unaffected).
- Human-oversight invariants unchanged and re-verified at go-live: nothing AI-produced publishes without maker-checker; C1 is grounded-only with citations; coexistence rule (manual always present) holds in every screen.

## 8.3 AI-specific SLOs & monitoring
C1 p95 ≤6 s, grounding-failure rate <2% of answers, block-rate baseline established in week 1 with alerting on 3× deviation, thumbs-down rate tracked per tenant with weekly review of the bottom decile of answers (sampled, hashes only — no verbatim storage beyond `ai_interaction`).

---

# PART 9 — Environments & release process

| Env | Purpose | Data | Sync | Notes |
|---|---|---|---|---|
| SIT | Integration, automated tests | Synthetic | Argo auto | Gallery + e2e run here |
| UAT | Author training, tenant acceptance, migration staging | Sanitized/synthetic; real tenant content drafts | Argo manual | Studio authors work against UAT host apps |
| PROD | Live | Real | Argo manual + change ticket | Publishing is a business action, deploys are IT changes — deliberately separate |

**Release cadence:** agent + services fortnightly (or on demand for fixes); content publishing continuous (business-controlled, unrelated to release trains). **Change classes:** (1) content publish — business approval, no CAB; (2) service/agent release — standard change, CAB per bank policy; (3) config/flag change — recorded, dual-approval for AI flags and kill switch.

**Definition of ready for production go-live (checklist):**
- [ ] Full gallery green on the release candidate; flags-off regression green
- [ ] Perf budgets met (agent ≤80 KB gz, boot ≤50 ms p95) on the RC
- [ ] AI red-team suite green; capability register signed off
- [ ] External pen-test findings closed or risk-accepted with dates
- [ ] DR restore drill evidence filed; PG failover drill passed
- [ ] Runbooks complete; L1/L2 trained; on-call rota staffed
- [ ] Dashboards + alerts live and tested (deliberate alert firing verified)
- [ ] PG-01 interim controls implemented and verified by external-exposure test; OIDC delivery date agreed in writing
- [ ] Vault Transit signing in use; rotation runbook drilled once
- [ ] Tenant onboarding pack issued; pilot tenant content approved and published in UAT
- [ ] Gap register reviewed and accepted by Security, Risk, and the CIO delegate

---

# PART 10 — Delivery plan (production hardening)

| Sprint | Focus | Exit |
|---|---|---|
| P1 (2 wks) | Containerize all services; Helm chart; SIT on OpenShift; CI image build/scan/sign | SIT green end-to-end on cluster |
| P2 (2 wks) | Vault Transit signing + secrets; ExternalSecrets; agent key-map build step | Publish→verify works with `sig_key_id` versions; rotation drilled |
| P3 (2 wks) | Data layer: partitioning, rollups, retention CronJobs, MinIO, PG HA/PgBouncer | Insights read only `agg_*`; failover drill passed |
| P4 (2 wks) | Security hardening: CORS, AV/ICAP, rate limits, NetworkPolicies, PG-01 interim controls, exposure test | Internal pen test clean; exposure test in CI |
| P5 (2 wks) | Observability: dashboards, alerts, synthetics (incl. signature check), runbooks + drills | Alerts verified by induced failure; monthly drills scheduled |
| P6 (2 wks) | AI hardening: vLLM ×2, admission control, red-team gate, capability register, kill controls | AI SLOs met; register produced |
| P7 (2 wks) | DR: DC2 standby, replication, DR drill; backups + restore evidence | RTO/RPO demonstrated |
| P8 (2 wks) | UAT hardening, external pen test, go-live checklist, pilot tenant cutover | Go-live gate passed |

~16 weeks with the existing team; P1–P4 are sequential, P5–P7 partially parallel. Migration of WalkMe content (separate plan) starts during P5 in UAT and cuts over after go-live per its own cohort schedule.

# PART 11 — Post-go-live backlog (ordered)

> The WalkMe feature-parity programme (help hub, tasks, rich content, designer, flow-engine and authoring upgrades) is specified separately in **Part 12** and runs alongside items 1–5 below; items in Part 12 that depend on authentication are tagged and sequenced after PG-01 closes.

1. **Close PG-01** — OIDC + RBAC per HLD §8 (committed date).
2. PG-02 SIEM real-time integration.
3. Second DC active/active (PG-04) at the next capacity cycle.
4. ClickHouse swap when a PD-02 scale trigger fires (PG-03).
5. Extension enterprise distribution (PG-05).
6. Tier-3 AI expansion: C2 depth variants at scale, C3 conversational surveys (needs its own consent/UX review).
7. Mobile surface (out of current scope; would extend the agent concept to in-app SDKs).

---

# PART 12 — Feature-parity backlog: WalkMe gaps (non-AI)

**Source:** WalkMe non-AI capability research (Oct 2026) and the comparison matrix section A8. **Scope:** the seven gaps observed in the bank's WalkMe deployments plus the additional gaps rated worth building. **Out of scope by decision:** session replay/DXA, mobile SDK, desktop menu, ActionBot, TeachMe/learning, licence discovery, custom CSS, push notifications.

## 12.0 Rules every item must follow

These are inherited from the architecture and are acceptance criteria for every item below, not aspirations.

1. **Content stays data.** Every new capability is a schema extension (guide schema **v1.2**, rich-text AST v2), interpreted by the agent. No authored script, no raw HTML, no CSS ever reaches a host page.
2. **Signed and governed.** New item types (tasks, resources, links, templates, goals, tracked events, panel config) flow through the existing draft → maker-checker → signed bundle pipeline. Nothing bypasses it.
3. **Fail-open.** Every new runtime feature degrades to "nothing shown", never to a broken host screen. Features that write to host DOM or change host behaviour (placeholder tips, element blocking, auto-steps) are **tenant-gated** like inline placement (§5.7) and need host-app-team sign-off.
4. **No new render-path dependency.** Guidance still renders from the cached bundle. The only new runtime call is the per-user state read (12.1.2), which is asynchronous, cached and non-blocking.
5. **Agent budget.** Core path stays ≤45 KB gz. New capabilities ship as **lazy modules** loaded only when a tenant's bundle uses them. Total agent ceiling rises from 80 KB to **120 KB gz across all lazy modules** — this needs an ADR (proposed ADR-012).
6. **Privacy unchanged.** No PII in bundles or events. Dynamic text renders claims client-side only and is never echoed into analytics.
7. **Backward compatible.** v1.0/v1.1 guides stay valid; a v1.2 bundle served to an older agent degrades (unknown types ignored, `agent_min` raised only when a tenant actually uses a v1.2 feature).
8. **Studio-managed, end to end.** Every feature is authored, configured, reviewed and published through Studio (console or extension) — no JSON editing, no API-only features, no back-office scripts. Each item's acceptance includes its Studio surface; §12.10 is the normative map.

## 12.1 Phase P0 — Foundations (2–3 weeks)

### 12.1.1 Schema v1.2 and AST v2
- New guide `type` values: `resource`, `link`, `task_list`, `goal`, `tracked_event`, `template`.
- New tenant-level document: `panel_config` (published inside the bundle).
- AST v2 adds node types used in P2: `video`, `iframe`, `flow_link`, `token`, and style marks `color`, `size`, `align` (constrained palettes, not free values).
- `rulec` gains the P4 condition sources; golden vectors extended for every new opcode.
- Migration: schema validators accept v1.0–v1.2; bundle compiler emits the lowest version a bundle actually needs.

### 12.1.2 User State Service (new, Java)
Several features need per-user state that must survive devices and cache clears: task progress, goal completions, inbox read flags, per-user data store keys.

- **Table** (additive migration `V300`):
  `user_state(tenant_id, u, kind, key, value jsonb, updated_at, PRIMARY KEY(tenant_id,u,kind,key))`, where `kind ∈ {task, goal, inbox, data}`; `u` is the existing pseudonymous HMAC id.
- **API:** `GET /api/state/me?tenant=` returns all of the caller's state (≤16 KB); `POST /api/state/me` applies a batch of idempotent upserts (`{kind,key,value,client_ts}`; last-write-wins by `client_ts`).
- **Agent behaviour:** read once after boot, **after** first render; cache in `localStorage` (`cmp:<t>:<e>:state`); writes are queued and flushed with the beacon cadence. If the service is down, features use the cached copy and retry — guidance still renders.
- **Hard prerequisite — signed context.** The PoC replaced the JWS context token with plain JSON (delta D3), and production hasn't restored it. Per-user state keyed on a self-asserted `u` is spoofable. Before 12.1.2 goes live, **reinstate the JWS context token** (Identity Adapter signs, User State Service verifies). This is independent of console authentication (PG-01) and can ship first.
- Retention: 13 months after last update; purged with the existing retention jobs.

### 12.1.3 Lazy module loader
Agent kernel gets `loadModule(name)`: fetches `/a/compass-<module>-<ver>.js` with SRI hashes taken from a signed module map in the manifest. Modules: `panel`, `tasks`, `designer-render`, `media`, `blocker`, `autostep`, `debug`. A module failing to load disables only that feature.

### 12.1.4 Goals and milestones (needed by Tasks)
- `goal` document: `{id, condition: <bytecode>, scope: page|session|ever, milestones: [{id, condition}]}`.
- Conditions use existing sources: URL match, element exists/clicked (anchor reference), flow completed, tracked event fired.
- Agent evaluates goals on route change, element mutation (debounced), flow completion and tracked events. On first true: emit `goal_reached` event and write `user_state(kind=goal)`.
- Flows and tasks reference goals by id. Goals are authored in the console with the existing segment builder plus an anchor picker.
- **AC:** a goal "IBAN form submitted" completes for a user who submits the form *without* any guidance, and appears in Insights.

**P0 exit:** v1.2 bundles publish and verify; state service passes a 1,000 rps read soak; JWS context verified end to end; flags-off regression green.

## 12.2 Phase P1 — Help hub and tasks (5–6 weeks) — *covers observed gaps 1 and 2*

### 12.2.1 Tabbed help panel (replaces the single assist panel)
- `panel_config`: `{tabs:[{id, type: help|tasks|resources|inbox|links, label (i18n key), icon, items:[{ref, folder?}], default?:bool}], footer:{links:[…]}, mode: docked|floating, position: right|left}`. Max 6 tabs.
- **Search** runs across all tabs (fixes a WalkMe limitation, where search is per tab), over titles, keywords and resource descriptions. A tab only renders if the user has at least one eligible item after targeting.
- The Phase-1 search panel becomes the "Help" tab. If the AI assistant is enabled, its "Ask AI" tab sits alongside (coexistence rule unchanged).
- **Console — Panel organizer:** two-pane screen (eligible items left, live panel preview right), drag-and-drop ordering, folders, rename tabs, choose default tab, preview as a segment. Panel config is versioned and goes through maker-checker like content.
- **AC:** an author builds a 3-tab panel with folders entirely in the console; a user's tab list changes with their segment; RTL mirrored.

### 12.2.2 Resources and Links
- `resource`: `{kind: article|video, url | asset_id, open: new_tab|panel_viewer|lightbox, description, keywords}`. URLs must match the tenant link allow-list; videos come from the media store (MP4) or an allow-listed host (see 12.3.1). The **panel viewer** renders an article URL in a sandboxed iframe only if the host is allow-listed for framing; otherwise it opens in a new tab.
- `link` (WalkMe "Shuttle"): `{url, target: same|new, mailto?}` with goal tracking on click.
- Both can be placed in panel tabs, launcher actions, announcement buttons and task lists.

### 12.2.3 Task lists (Onboarding Tasks)
- `task_list`: `{id, title, items:[{ref (flow|resource|link), completion: {by: goal|flow_complete|opened, goal_id?}, activation?: <bytecode>, optional?: bool}], show_progress, strike_completed, allow_replay}`.
- **Runtime:** a Tasks tab shows the list with a progress bar. Locked items (activation rule false) are greyed with an explanation tooltip. Completion is read from and written to the User State Service — so progress follows the user across devices.
- Completion is **never** reversed automatically; an admin "reset progress for segment" action exists in the console (audited, maker-checker).
- **Analytics:** `task_started`, `task_completed`, list completion funnel.
- **AC:** a user completes task 2 by finishing the underlying process without the flow (goal-based), sees it struck through on another device, and task 3 unlocks via its activation rule.

### 12.2.4 Footer help-desk links and ticket deep-link (v1)
- Footer: one or many links (`label`, `url` or `mailto`), per locale.
- "Raise a ticket" v1 = a link to the ServiceNow/Zendesk catalogue form with **non-PII** context appended as query parameters (application, page route, guide id). No backend integration and no user identity required.

### 12.2.5 Notification inbox (web)
- Announcements gain `persist_to_inbox: bool` and `expires_at`. Persisted announcements appear in an **Inbox** tab, newest first; unread count shows as a badge on the panel launcher.
- Read/unread and dismissal live in User State Service (`kind=inbox`).
- No push and no OS notifications (out of scope).

### 12.2.6 External knowledge search (v1)
- New **Search Connector** module (Java, inside Content Service or a separate service): scheduled crawl of **approved, organisation-wide** SharePoint sites and Confluence spaces via their APIs with a service account. Indexes title, URL, excerpt into PostgreSQL full-text (`kb_doc` table); respects a tenant allow-list of spaces.
- Panel search shows "From the knowledge base" results under guide results; clicking opens the source page.
- **v1 limitation stated in the UI:** only content readable by everyone is indexed. Per-user permission trimming is v2 (P6), because it requires real user identity.
- If the AI layer is on, the same index can feed RAG (`rag_chunk`) — one crawler, two consumers.

**P1 exit:** help hub live for the pilot tenant; task progress consistent across two devices; state service p95 < 80 ms; agent core budget unchanged.

## 12.3 Phase P2 — Content richness, links, launchers, form guidance (4 weeks) — *covers observed gaps 3, 4, 5, 6*

### 12.3.1 Media: video, iframe, GIF
- AST `video` node: `{asset_id | url, poster_asset, width, height, captions_asset?}`. Media-store MP4 is preferred (served from the edge, same-origin as bundles). External URLs only for **tenant allow-listed hosts** (e.g., the bank's internal video platform). Rendered as a native `<video controls>` inside the shadow root; no autoplay with sound.
- AST `iframe` node: `{url, height, title}` — allow-listed hosts only, rendered with `sandbox="allow-scripts allow-same-origin"` scoped per host config, `referrerpolicy="no-referrer"`, lazy-loaded.
- **CSP delta** for host apps: `media-src` (edge origin) and `frame-src` (only the allow-listed hosts). Added to the onboarding runbook.
- GIF: allowed in the image pipeline; under `prefers-reduced-motion` the first frame is shown with a play control.
- Media pipeline: MP4 size cap (default 50 MB), AV scan, poster frame required, captions (WebVTT) strongly recommended and flagged in QA when missing.
- **AC:** a step shows a captioned training video; a host app with strict CSP plus the documented delta plays it; removing the host from the allow-list blocks it at publish time, not at runtime.

### 12.3.2 Flow links and permalinks
- AST `flow_link` node: `{flow_id, step_id?, label}`. Clicking ends the current flow (recorded as `flow_handoff`) and starts the target — same path as deep links.
- Permalink grammar (all validated against the published bundle):
  - `?compass_flow=<flow>[:<step>]` — start at a step; step entry still requires the step's anchor to resolve
  - `?compass_ann=<announcement_id>`
  - `?compass_panel=<tab_id>[:<item_id>]` — opens the panel at a tab/item
- Console: "Copy link" on flows, steps, announcements and panel items.

### 12.3.3 Dynamic text
- AST `token` node: `{source: claim.<name> | data.<key> | url.param.<name> | date.today, format?: upper|lower|title|date_short, fallback: "<text>"}`.
- **Claims are tenant allow-listed** (e.g., `first_name`, `branch`, `role_label`). Adding a claim to the allow-list is an admin change with privacy review recorded.
- Rendered client-side only; never written into events, beacons or the bundle. QA preview shows fallback and a sample persona value.
- **AC:** "Welcome, Aisha" renders from the context token; the analytics event for the same step contains no name.

### 12.3.4 Launcher templates and actions (gap 6)
- Launcher `design`: `{template_id?, icon: gallery_id | asset_id, label, label_position: none|right|left|below|above, size, states: {default, hover, focus}}`. Label text is per launcher even when a template is used (WalkMe's "Different text").
- **Icon gallery:** bank-curated SVG set (sanitised), plus tenant uploads through the media pipeline.
- **Template gallery:** tenant- and org-scoped launcher templates (also used in 12.4).
- `actions`: ordered list of `start_flow | open_resource | open_link | show_announcement | open_panel(tab) | validate_set(set_id)`.
- Behaviour: `show_on: always|hover_on_anchor`, `delay_ms`.
- **AC:** a "Help" icon with caption below renders identically in LTR and RTL and opens the panel on the Tasks tab.

### 12.3.5 Form guidance upgrades (gap 4)
- **Icon placement grid:** tip `icon: {position: tl|tc|tr|ml|mc|mr|bl|bc|br, offset_x, offset_y}` relative to the anchor; tooltip placement `auto` or a fixed one of the existing 12 placements.
- **Triggers:** add `click` to hover/focus/always.
- **Placeholder mode** (tenant-gated, host-DOM write): sets the input's `placeholder` attribute, restores the original on teardown or kill switch. Skipped if the field already has a non-empty placeholder unless the author forces override.
- **Validation success state:** optional green indicator when the rule passes on blur.
- **Check My Form:** tips grouped into a `validation_set`; launcher action `validate_set` evaluates all and scrolls to the first failure; emits `form_checked{passed, failed_count}`.
- **Preset rules** (compiled to bytecode by `rulec`): required, email, numeric range, length, date range, **IBAN (mod-97)**, account-number pattern (safe regex, see 12.5.4).
- **AC:** a 6-field transfer form validates on demand with one launcher; success icons show on valid fields; placeholder tips restore cleanly after kill switch.

### 12.3.6 Constrained text styling
AST style marks: `color` (5 tenant palette slots), `size` (S/M/L), `align` (start/center/end — logical, so RTL works). No free hex values, no fonts.

**P2 exit:** CSP-strict gallery case passes with media; all P2 features covered by new hostile-gallery cases (video in nested scroller, iframe in RTL, placeholder tip on React-controlled input).

## 12.4 Phase P3 — Announcement designer and surveys (4 weeks) — *covers observed gap 7*

### 12.4.1 Announcement layout model
- `layout`: `modal | banner | toast | slide_in | full_screen | mini_menu`.
- `position`: 9-point anchor + `offset_x/y` (px or %); `size`: width/height (px or %, clamped to viewport).
- `regions`: ordered blocks — `media` (image/video), `title`, `body` (AST), `buttons`, `footer_note`.
- `style`: background (palette slot or one of 6 approved two-stop gradients), radius, border, shadow level (0–3), overlay dim (0–3).
- `buttons[]`: label + one action from the shared action list (12.3.4) + `dismiss | snooze(days)`.
- **Mini-menu:** a compact list of up to 6 actions (resources, links, flows) — the WalkMe "Mini Menu" format.

### 12.4.2 Template gallery and designer
- `template` documents: `{scope: tenant|org, layout, regions, style, preview_asset}`; org-scope templates require an org-admin approver.
- **Console designer:** form-based editor with **live preview using the production renderer** (same principle as the extension preview); drag to reorder regions; position by dragging in a scaled viewport frame. Deliberately not a free canvas — no layers, no arbitrary shapes — so output stays declarative, accessible and RTL-safe.
- Updating a template **does not** change announcements already built from it (WalkMe behaviour) unless the author chooses "Apply template update" (creates new draft versions → maker-checker).
- **Accessibility gate:** contrast check on text vs background, focus order, alt text, button labels — publish blocked on failure.

### 12.4.3 Survey upgrades
- `questions[].logic: [{when: <answer condition>, goto: question_id | end}]`.
- `display: popup | full_page`.
- Survey templates (NPS, CSAT, task feedback) in the template gallery.
- Flow `end_experience: {survey_id}` triggers a survey on flow completion (frequency-capped).
- **AC:** an NPS ≤6 answer branches to a "what went wrong" question; ≥9 ends; full-page mode is keyboard-navigable.

## 12.5 Phase P4 — Flow engine, conditions, enforcement, tracked events (4–5 weeks)

### 12.5.1 Flow step additions
- **Popup step:** `anchor: null`, `position` as in 12.4.1. Advance rules unchanged.
- **Peer steps:** `extra_anchors: [{anchor, content, placement}]` — up to 3 additional tooltips shown with the main step; all torn down together.
- **Start points:** `flow.start_points: [{step_id, condition: <bytecode>}]` evaluated bottom-up when the flow starts; first true wins. Lets a user who is already half-way through a process start at the right step.
- **Connect / restart:** goto targets `@flow:<id>[:<step>]` and `@restart`. Cross-flow hops are counted in the existing FSM loop guard.
- **Cross-frame:** same-origin frames already supported via `frame_path`. Cross-origin frames: agent must be installed in the frame; parent and child agents exchange flow state over `postMessage` with origin allow-list and a per-session nonce. Host-app team must install the snippet in the framed app.

### 12.5.2 Per-user data store and "set data" step
- Step type `set_data: {key, value | increment, scope: session|user}`. User scope persists via User State Service; session scope stays in `sessionStorage`.
- Condition source `data.<key>`. Keys are declared per tenant (name, type, max length) — undeclared keys are rejected at publish.
- Typical use: "shown the beneficiary warning 3 times → stop showing".

### 12.5.3 Richer conditions (rulec + agent)
New sources, each tenant allow-listed:
- `cookie.<name>` — named cookies only, never all cookies.
- `js.<path>` — read-only lookup of an allow-listed dotted path on `window` (e.g., `appConfig.module`); values truncated to 256 chars; no function calls.
- `referrer`, `viewport.width/height`, `browser.family`, `time.local` (HH:MM), `day_of_week`.
- `data.<key>` (12.5.2).

Bytecode limits from LLD §1.7 unchanged (512 ops, 1 ms, max 4 element probes).

### 12.5.4 Safe regex
- `MATCH_RE` opcode for URL and value conditions.
- `rulec` accepts only a **linear-time subset**: no backreferences, no lookaround, no nested quantifiers; pattern ≤200 chars; tested input truncated to 2 KB. Patterns failing the check are rejected at publish with a reason.
- **AC:** a catastrophic-backtracking pattern is rejected by the compiler; the gallery's adversarial vector set passes.

### 12.5.5 Element blocking (tenant-gated)
- `blocker` item: `{anchor, message (AST), action?: start_flow|open_resource, condition}`.
- Agent places a transparent overlay over the anchor inside its shadow root (repositioned with the positioning engine) and adds a capture-phase listener on the anchor that cancels click and Enter/Space while the condition holds; shows the message on attempt. `aria-disabled` is set on the anchor and restored on teardown.
- **Stated limitation (in the UI and docs):** blocking is a **UX guardrail, not an access control**. If the agent fails (fail-open) or is killed, the element works. Anything that must be enforced belongs in the host application.
- **AC:** "Submit" is blocked until checklist goal X is reached; kill switch restores it within 60 s; screen readers announce it as disabled.

### 12.5.6 Auto-steps (tenant-gated, risk sign-off required)
- Step types `auto_fill {anchor, value: literal | token}`, `auto_select {anchor, option_value}`, `auto_click {anchor}`.
- **Guardrails:**
  - Allowed only on anchors the tenant has tagged `automation_safe` (a reviewed allow-list, typically navigation, tabs, filters, non-financial fields).
  - Never on submit/confirm controls; the console blocks anchors whose role/name matches submit/confirm/pay/transfer patterns unless the tenant admin overrides with a recorded justification.
  - Values only from literals or allow-listed tokens; never from free user data.
  - Framework-safe input: uses the native value setter and dispatches `input` and `change` events so React/Angular register the change.
  - Every auto-step emits an `auto_action` event (anchor, action, step) for audit.
- **AC:** auto-fill on a React-controlled input is recognised by the app's state; an auto-click on a "Transfer" button cannot be published without override.

### 12.5.7 Tracked events
- `tracked_event` documents: `{id, name, trigger: click(anchor) | page(url_pattern) | element_appears(anchor), properties: [static key/values]}`; authored with the extension picker.
- Agent emits `tracked{id}` — no element text, no field values. These events feed goals, tasks and dashboards.
- Not DXA: only author-defined events are captured.

## 12.6 Phase P5 — Authoring and operations (4 weeks)

### 12.6.1 Content manager
- Console screen listing all items with **folders**, **custom fields** (tenant-defined: owner, business process, review date, …), **saved and shared views** (filters + columns), and **bulk actions**: move, tag, assign segment, archive, submit for review, publish (publish still requires approved versions — bulk never bypasses maker-checker).
- Tables (`V310`): `folder`, `custom_field_def`, `custom_field_value`, `saved_view`. All bulk actions are single transactions with one audit entry per item.

### 12.6.2 Comments
`comment(id, tenant_id, entity_type, entity_id, version?, author, body, mentions[], created_at, resolved_at)`. @mentions notify through the console notification bell (no email in v1). Comments are kept with version history and exportable.

### 12.6.3 Version history, compare and restore; archive
- Every item gets a history view: version list with status and approver; **side-by-side diff** of any two versions (JSON diff rendered as readable changes plus visual preview of both).
- **Restore** creates a new draft from the chosen version — it never republishes directly; maker-checker applies.
- New status `archived`: hidden from authoring lists and bundles; **delete is only allowed for archived items not referenced** by any panel, task list, flow link or launcher action (reference check enforced in the service).

### 12.6.4 Flow debugger (author-only)
- Activated by `?compass_debug=1` **plus** a short-lived debug token issued by the Content Service to an allow-listed author (until PG-01 closes, issuance is restricted by the same perimeter + allow-list controls).
- Lazy `debug` module shows a side panel: eligible/ineligible guides with the targeting result of each condition, anchor scores per step, FSM state and last transition, goal states, user-state snapshot, and a "reset my progress" button for test users.
- Never available to end users; ignored when the token is absent or expired.

### 12.6.5 Reusable solutions across tenants
- **Org template library:** announcement, launcher, survey and task-list templates with `scope=org`, published by an org admin.
- **Solution package:** export a set of items (flows, tasks, resources, templates) as a signed JSON package; import into another tenant creates **drafts** with anchors marked "needs capture" (anchors are app-specific). Fits the HR/Ops onboarding model.

### 12.6.6 Translation extras
- XLSX import/export alongside XLIFF (Apache POI), same review states.
- Translation REST API: export/import by guide set and locale (service credentials; listed under PG-01 dependencies).
- Vendor connector SPI: `TranslationConnector{submit(job), poll(jobId), fetch(jobId)}` with a first implementation for the bank's chosen vendor; jobs tracked in `translation_job`.

### 12.6.7 Admin: roles and activity API
- **Custom role matrix:** permissions per tenant and environment (author, review, publish, manage templates, manage panel, manage gated features, analytics, admin). Built-in roles become presets. Maker-checker DB constraint unchanged.
- **Activity log API:** read-only, paginated, filtered (`actor`, `action`, `entity`, `from/to`) over `audit_event`; CSV export.

## 12.7 Phase P6 — Analytics and identity-dependent integrations (3–4 weeks; starts after PG-01 closes)

### 12.7.1 Custom dashboards
- Widget types: KPI, line, bar, funnel, table. Data source: **fixed metric catalog** over the `agg_*` rollups (same principle as the AI insights copilot — no free-form SQL).
- Dashboards are per tenant, shareable by role, exportable to CSV/PDF, with scheduled email in a later increment.
- New rollups: `agg_task_daily`, `agg_goal_daily`, `agg_tracked_daily`.

### 12.7.2 Ticket integration v2
ServiceNow/Zendesk API connector: "my tickets" search and create-ticket from the panel with page context attached. Needs the user's real identity (not the pseudonymous id), so it requires OIDC (PG-01) and a privacy review. Credentials in Vault; calls proxied through Compass backend.

### 12.7.3 Knowledge search v2
Per-user permission trimming for SharePoint/Confluence results using the user's delegated token (OIDC). Same dependency as 12.7.2.

### 12.7.4 SCIM 2.0
`/scim/v2/Users` and `/Groups` for author/reviewer provisioning from the bank IdP; maps groups to the custom role matrix.

## 12.8 Cross-cutting work

- **Hostile gallery additions:** tabbed panel in RTL; tasks across two simulated devices; video in nested scroller; iframe under CSP-strict; placeholder tip on React-controlled input; blocker on dynamically re-rendered button; auto-fill on React/Angular inputs; peer steps with collisions; cross-origin frame relay; catastrophic regex rejection.
- **Migration toolkit update:** the WalkMe converter (`compass-migrate convert`) gains mappings for Onboarding Tasks → `task_list` + goals, Resources → `resource`, Shuttles → `link`, Menu tabs → `panel_config`, ShoutOut templates → `template`, Auto-Steps → `auto_*` (imported **disabled** pending risk sign-off), Invisible launchers → `blocker` (disabled pending tenant gate). Unmappable items still become draft warnings.
- **Docs and training:** author guide sections per feature; tenant onboarding runbook updated with the CSP delta, gated-feature sign-off form, and claim allow-list request form.
- **ADRs:** ADR-012 agent lazy-module budget (120 KB total); ADR-013 User State Service as the only per-user runtime call; ADR-014 gated host-DOM features (placeholder, blocker, auto-steps) and their sign-off process; ADR-015 safe-regex subset.

## 12.9 Delivery plan and sequencing

| Phase | Weeks | Depends on | Observed gaps covered | Exit gate |
|---|---|---|---|---|
| P0 Foundations | 2–3 | JWS context restored | — | v1.2 bundles signed and verified; state service soak; flags-off regression |
| P1 Help hub & tasks | 5–6 | P0 | 1 (tasks), 2 (tabs) | Pilot tenant hub live; cross-device task progress |
| P2 Content, links, launchers, forms | 4 | P0 | 3 (media), 4 (field tips), 5 (flow links), 6 (icon + caption) | CSP-strict media case; new gallery cases green |
| P3 Designer & surveys | 4 | P2 (action list) | 7 (announcement UI) | Accessibility gate enforced; template update semantics verified |
| P4 Engine, conditions, gated features | 4–5 | P0, P1 (goals) | — | Gated features need tenant sign-off; regex vectors pass |
| P5 Authoring & ops | 4 | P0 | — | Bulk actions audited; restore always via draft |
| P6 Analytics & identity integrations | 3–4 | **PG-01 closed** | — | Privacy review signed; SCIM provisioning works |

**Effort (AI-assisted):** roughly 26–30 weeks with 3–4 engineers; P2/P3 and P5 can run in parallel with P4 once P0 lands, bringing elapsed time to about 18–20 weeks. The seven observed gaps are all closed by the end of P3 (≈13–15 weeks from start).

**Priority if capacity is tight:** P0 → P1 (tasks, tabs, goals) → P2 (media, flow links, dynamic text, form helpers, launchers) → P4.5 element blocking → P4 conditions → P5 content manager/version history → P3 designer → rest.


## 12.10 Studio surfaces — how every feature is managed

This section is normative: a feature is not "done" until its Studio surfaces below exist, are permission-checked (custom role matrix, §12.6.7) and go through maker-checker where they change published content.

### 12.10.1 Console navigation after Part 12

| Area | Screens (new in **bold**) |
|---|---|
| Guides | Guides list (now the **Content manager**: folders, saved views, bulk actions, custom fields) · Guide editor · **Version history & compare** |
| Help hub | **Panel organizer** · **Resources** · **Links** · **Task lists** · **Inbox** (persisted announcements) |
| Experiences | Announcements (**layout designer**) · Surveys (**logic builder**, full-page mode) · Launchers (**template & action editor**) · **Validation sets** · **Blockers** · **Auto-step safe list** |
| Logic | Segments (**new condition sources**, safe-regex tester) · **Goals & milestones** · **Tracked events** · **Data keys** |
| Library | Media (now incl. **video, captions, GIF**) · **Template gallery** (tenant/org) · **Icon gallery** · **Solution packages** (export/import) |
| Translations | XLIFF + **XLSX** import/export · **Vendor jobs** |
| Review | Review queue (now with **comments**, diff of any two versions, AI/manual origin badge) |
| Publish | Pre-flight checklist (adds: CSP delta needed, gated features used, accessibility gate, unresolved anchor hints) |
| Insights | Fixed reports · **Custom dashboards** (P6) · **Task & goal reports** |
| Admin | Tenants · **Role matrix** · **Feature gates** · **Claim allow-list** · **Host & media allow-lists** · **Search connectors** · **Help-desk & ticket links** · **Activity log** · Kill switch |

### 12.10.2 Per-feature management map

| Feature | Authored in | Configured / enabled by | Review & publish | Notes |
|---|---|---|---|---|
| Tabbed help panel | Console → Panel organizer (drag-drop tabs, folders, default tab, preview-as-segment) | Tenant admin sets max tabs, docked/floating, footer links | Panel config is versioned → maker-checker → bundle | Preview uses production renderer |
| Resources (article/video) | Console → Resources (URL or media asset, open mode, keywords) | URL/host allow-list in Admin | Maker-checker | Placed via Panel organizer, launcher/announcement action pickers |
| Links (Shuttles) | Console → Links | URL allow-list | Maker-checker | |
| Task lists | Console → Task lists (pick flows/resources/links, completion = goal / flow / opened, activation rule builder, progress & strike-out toggles) | — | Maker-checker | **Reset progress for segment** = admin action, audited, dual approval |
| Goals & milestones | Console → Goals (condition builder + anchor picker launched into the extension) | — | Maker-checker | Referenced by tasks, flows, dashboards |
| Inbox | Announcement editor toggle "Keep in inbox" + expiry | — | Maker-checker | Console → Inbox shows what's live per segment |
| Knowledge search v1 | — | Admin → Search connectors (SharePoint site / Confluence space allow-list, schedule, last crawl status) | Admin change audited | Results preview in console |
| Help-desk & ticket deep-link | — | Admin → Help-desk & ticket links (per locale) | Admin change audited | |
| Video / iframe / GIF in content | Extension bubble & console step editor: **Insert → Video / Embedded page / Image** | Admin → Host & media allow-lists | Maker-checker; publish blocked if host not allow-listed | Pre-flight shows required CSP delta |
| Flow links in text | Editor toolbar **Insert → Link to flow** (flow + optional step picker) | — | Maker-checker | Broken references flagged at review |
| Permalinks | "Copy link" on flows, steps, announcements, panel items | — | n/a (read-only) | |
| Dynamic text | Editor toolbar **Insert → Personalised text** (token + fallback, preview as persona) | Admin → Claim allow-list (privacy review field mandatory) | Maker-checker | |
| Text styling | Editor toolbar (palette colour, size, alignment) | Tenant theme defines palette slots | Maker-checker | |
| Launcher design & actions | Console → Launchers (template, icon from gallery, label position, per-state styles, actions list, hover/delay) + extension placement | Icon gallery & templates managed in Library | Maker-checker | |
| Form tips (grid, offsets, triggers, success) | Extension: pick field → tip bubble shows **icon position grid** and offset handles; console for bulk edit | — | Maker-checker | |
| Placeholder tips | Extension tip editor "Show as placeholder" | **Admin → Feature gates** (tenant) + host-app sign-off record | Maker-checker | Option hidden until gate on |
| Validation sets & Check My Form | Console → Validation sets (group tips, preset rules incl. IBAN, success indicator); launcher action `validate_set` | — | Maker-checker | |
| Announcement layouts & designer | Console → Announcements (layout, position drag in scaled viewport, regions, style, buttons) | Templates in Library; org templates need org admin | Maker-checker; **accessibility gate** blocks publish | |
| Template gallery | Console → Library → Templates (create from any announcement/launcher/survey/task list) | Org scope: org admin | Template changes don't alter existing items unless "Apply update" (creates drafts) | |
| Survey logic & full-page | Console → Surveys (question flow with branch arrows, display mode, templates, end-of-flow link from flow settings) | — | Maker-checker | |
| Popup & peer steps | Extension: "+ Add step → Popup" / "Add companion tooltip" on a selected step | — | Maker-checker | Flow map shows peer steps grouped |
| Start points | Console flow map: mark step "Can start here" + condition | — | Maker-checker | Debugger shows which start point matched |
| Connect / restart flow | Flow map: goto target picker includes other flows / restart | — | Maker-checker | Loop guard warning in editor |
| Cross-frame flows | Extension works inside frames where the snippet exists | Admin: frame origin allow-list | Maker-checker | |
| Data keys & "set data" step | Console → Data keys (name, type, max length, scope); step type in flow editor | — | Maker-checker | Undeclared keys rejected |
| New condition sources & regex | Segment / condition builder: new source types; **regex tester** shows accept/reject + reason | Admin: cookie names, JS paths allow-list | Maker-checker | |
| Element blocking | Console → Blockers + extension picker for the element | **Feature gates** + host-app sign-off | Maker-checker; pre-flight warns "guardrail, not access control" | |
| Auto-steps | Flow editor step types (fill / select / click) | **Admin → Auto-step safe list** (anchors tagged automation-safe) + **Feature gates** + risk sign-off; override for submit-like controls requires justification | Maker-checker; checker sees auto-steps highlighted | |
| Tracked events | Extension picker "Track this click / page / element" → console Tracked events list | — | Maker-checker | Feed goals and dashboards |
| Content manager | Console → Guides | Custom fields defined by tenant admin | Bulk actions never bypass maker-checker | |
| Comments | Panel on every item and review screen | — | n/a | Kept with versions |
| Version compare / restore / archive | Item → History tab | Delete restricted to admin role | Restore = new draft → maker-checker | |
| Flow debugger | Console "Debug on page" button opens host app with debug token | Debug token issuance limited to allow-listed authors | n/a | Never visible to end users |
| Solution packages | Library → Solution packages (export selection / import into tenant) | Import requires tenant admin | Imported items land as drafts, anchors "needs capture" | |
| Translation extras | Translations → XLSX import/export, Vendor jobs (submit, status, fetch) | Admin: vendor connector credentials (Vault) | Translation review states unchanged | |
| Role matrix | — | Admin → Role matrix (per tenant & environment) | Changes audited, dual approval for admin grants | |
| Activity log | Admin → Activity log (filters, CSV export) | — | n/a | Same data as API |
| Custom dashboards (P6) | Insights → Dashboards (widget picker over metric catalog, share by role) | — | n/a | |
| Ticket v2, KB search v2, SCIM (P6) | — | Admin → Integrations (credentials in Vault) | Admin change audited | Requires PG-01 closed |

### 12.10.3 Studio extension changes
- Step editor: **Insert** menu (video, embedded page, image, link to flow, personalised text); popup-step and companion-tooltip creation; auto-step types (shown only when gated on).
- Tip editor: icon position grid with offset handles, trigger picker (hover/focus/click/always), placeholder option (gated), success indicator toggle.
- Pickers launched from console: goal conditions, tracked events, blockers, auto-step safe-list tagging — the console opens the host app with the extension in "pick for …" mode and receives the fingerprint back.
- Debug mode overlay (12.6.4).

### 12.10.4 Delivery note
Each phase in §12.9 includes its Studio surfaces in scope and estimate; a phase exits only when its features are fully manageable in Studio by an author with no developer help (AC verified with the authoring UAT script, extended per phase). UI reference mockups for the new console screens (Panel organizer, Task lists, Announcement designer, Validation sets, Feature gates) should be produced at the start of each phase, following Part 9 conventions of the PoC LLD.
