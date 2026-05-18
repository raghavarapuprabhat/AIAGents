# Solution Architecture: Compliance Rule Extraction & Reporting System

## 1. Solution Overview

A batch-oriented document intelligence system that ingests Country Dispensation Word documents, parses the structured table content, uses GPT-4o to extract and reason about the rules, presents results to a human reviewer for validation, and produces a structured rule repository with reports.

**Example interpretation** for the attached request (REQUEST-0000001023):

```
Baseline:    Individual DOB must be verified (day + month + year) per Appendix G
Trigger:     Individual presents HKID card AND HKID lacks day/month of birth
Exception:   Verify year-of-birth only
Scope:       All risk ratings; All individual clients + individuals related to entities
Segments:    CCIB, BB, PPB
Expires:     2026-10-12
Action:      Business must define a consistent DOB documentation approach in CDD file
```

---

## 2. Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                       ON-PREM LINUX SERVER (Ubuntu 22.04)                    │
│                                                                              │
│  ┌─────────────────┐                                                         │
│  │   Reviewer /    │   HTTPS                                                 │
│  │   Compliance    │◄────────┐                                               │
│  │     User        │         │                                               │
│  └─────────────────┘         │                                               │
│                              ▼                                               │
│                    ┌──────────────────┐                                      │
│                    │  Nginx (TLS)     │  reverse proxy + static assets       │
│                    └────────┬─────────┘                                      │
│                             │                                                │
│            ┌────────────────┼──────────────────┐                             │
│            ▼                ▼                  ▼                             │
│   ┌────────────────┐ ┌──────────────┐ ┌──────────────────┐                   │
│   │  React SPA     │ │  FastAPI     │ │  Reports         │                   │
│   │ (Vite + TS)    │ │  REST API    │ │  Endpoint        │                   │
│   │  - Upload      │ │  /upload     │ │  Excel / PDF     │                   │
│   │  - Review UI   │ │  /rules      │ │  (openpyxl,      │                   │
│   │  - Dashboard   │ │  /review     │ │   WeasyPrint)    │                   │
│   └────────────────┘ └──────┬───────┘ └──────────────────┘                   │
│                             │                                                │
│                             ▼                                                │
│              ┌──────────────────────────────┐                                │
│              │   Celery Worker(s)           │   async batch processing       │
│              │   (Redis broker)             │                                │
│              └──────────────┬───────────────┘                                │
│                             │                                                │
│        ┌────────────────────┼─────────────────────┐                          │
│        ▼                    ▼                     ▼                          │
│ ┌─────────────┐    ┌────────────────┐   ┌─────────────────┐                  │
│ │ DOCX Parser │    │ Rule Extractor │   │ Rule Reasoner   │                  │
│ │ python-docx │───►│ (Stage-1 LLM)  │──►│ (Stage-2 LLM)   │                  │
│ │ Tables A–D  │    │ Field-level    │   │ Semantic logic  │                  │
│ │ extraction  │    │ JSON via       │   │ + scope + risk  │                  │
│ │             │    │ Structured     │   │ + expiry parse  │                  │
│ │             │    │ Output         │   │                 │                  │
│ └─────────────┘    └────────┬───────┘   └────────┬────────┘                  │
│                             │                    │                           │
│                             └────────┬───────────┘                           │
│                                      │                                       │
│                                      ▼                                       │
│                          ┌───────────────────────┐         ╔═══════════════╗ │
│                          │   GPT-4o API Client   │────────►║  OpenAI       ║ │
│                          │   (httpx + retry)     │  HTTPS  ║  GPT-4o API   ║ │
│                          └───────────┬───────────┘         ║ (egress only) ║ │
│                                      │                     ╚═══════════════╝ │
│                                      ▼                                       │
│                          ┌───────────────────────┐                           │
│                          │ Pydantic v2 Validator │   schema enforcement      │
│                          └───────────┬───────────┘                           │
│                                      │                                       │
│                                      ▼                                       │
│        ┌────────────────────────────────────────────────────┐                │
│        │              PostgreSQL 16                         │                │
│        │   rules │ rule_versions │ audit_log │ users        │                │
│        │   reviews │ documents │ attachments_index          │                │
│        └────────────────────────────────────────────────────┘                │
│                                                                              │
│        ┌────────────────────────────────────────────────────┐                │
│        │  Filesystem: /var/lib/cdd-rules/{inbox,archive}    │                │
│        │  Logs: /var/log/cdd-rules (loguru → journald)      │                │
│        │  Secrets: HashiCorp Vault (or systemd-creds)       │                │
│        └────────────────────────────────────────────────────┘                │
│                                                                              │
│  Observability:  Prometheus  ◄──  FastAPI/Celery metrics  ──►  Grafana       │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. End-to-End Data Flow

| Step | Stage | What happens |
|------|-------|--------------|
| 1 | **Upload** | Reviewer uploads `.docx` via React UI → FastAPI `/upload` → stored at `/var/lib/cdd-rules/inbox/<uuid>.docx`; `documents` row created with status `RECEIVED`. |
| 2 | **Queue** | FastAPI enqueues a Celery task `process_document(doc_id)`. |
| 3 | **Parse** | Worker uses `python-docx` to walk every table; for each row labelled A/B/C/D extracts the cell text and preserves the request ID, section reference, requirement, dispensation, client type, segment, expiry. |
| 4 | **Stage-1 LLM** | Sends the parsed cells to GPT-4o with a strict Pydantic schema (OpenAI **Structured Outputs**, `response_format=json_schema`) to produce the canonical `Rule` object. |
| 5 | **Stage-2 LLM** | A second GPT-4o call performs semantic reasoning: classifies the rule type (less-stringent / more-stringent / clarification), normalises the trigger document (HKID), expands the risk-rating scope, and produces a plain-English summary + machine-readable conditions tree. |
| 6 | **Validate** | Pydantic v2 enforces schema; date parser converts "12 Oct 2026" → ISO `2026-10-12`; row written to `rules` with status `PENDING_REVIEW`. |
| 7 | **Review** | Reviewer opens the case in the UI: source DOCX renders on the left (mammoth.js → HTML), extracted JSON on the right with inline edit. Reviewer Approves / Edits / Rejects; each change written to `rule_versions` + `audit_log` (who/when/what). |
| 8 | **Publish** | Approved rules move to status `ACTIVE`; expiry-watch job flags rules within 60 days of expiry. |
| 9 | **Report** | Dashboards (status, expiring soon, by segment) and on-demand Excel / PDF exports of the rule register. |

---

## 4. Detailed Tech Stack

| Layer | Technology | Purpose / Rationale |
|-------|-----------|---------------------|
| **OS** | Ubuntu 22.04 LTS (or RHEL 9) | Long-term support, broad package availability |
| **Runtime** | Python 3.11 | Mature ecosystem for doc parsing + LLM SDKs |
| **API framework** | FastAPI + Uvicorn (behind Gunicorn) | Async, typed, auto OpenAPI docs |
| **DOCX parsing** | `python-docx`, `lxml`, `mammoth` (for UI preview) | Native .docx XML traversal; mammoth for fidelity preview |
| **LLM provider** | OpenAI GPT-4o via official `openai` SDK | User-confirmed access; supports Structured Outputs |
| **LLM orchestration** | Plain OpenAI SDK + Pydantic (no LangChain) | Lower abstraction, easier audit for compliance |
| **Schema validation** | Pydantic v2 | Strict typed output, validation errors raise retry |
| **Prompt management** | Versioned prompts in `/prompts/*.md` + Git | Prompts are auditable artifacts |
| **Async / batch** | Celery 5 + Redis 7 | Decouples HTTP from long LLM calls; retry with backoff |
| **Database** | PostgreSQL 16 | Transactional integrity + JSONB columns for rule payload |
| **Migrations** | Alembic | Schema versioning |
| **ORM** | SQLAlchemy 2 | Typed sessions; works well with FastAPI |
| **Frontend** | React 18 + Vite + TypeScript + TanStack Query + shadcn/ui | Polished review UI, side-by-side doc viewer |
| **Doc viewer** | mammoth.js (DOCX → HTML in browser) | Renders source for reviewer alongside extracted JSON |
| **Auth** | Keycloak (OIDC) integrated with internal IdP | SSO + role-based access (Uploader / Reviewer / Admin) |
| **Reverse proxy** | Nginx with internal-CA TLS | TLS termination, static asset serving |
| **Process mgmt** | systemd units for `api`, `worker`, `beat`, `nginx` | Native Linux supervision |
| **Containerization** | Docker + Docker Compose (single-host) | Reproducible deploys; not k8s (overkill for low volume) |
| **Secrets** | HashiCorp Vault *or* `systemd-creds` | OpenAI API key, DB creds never in code/env files |
| **Reports** | `openpyxl` (Excel), `WeasyPrint` (PDF), Jinja2 templates | Generated server-side for download |
| **Observability** | Loguru → journald; Prometheus client + Grafana; OpenTelemetry traces | LLM latency, token cost, failure rates, queue depth |
| **CI/CD** | GitLab CI (on-prem runner) → Docker registry → Ansible deploy | Air-gap friendly |
| **Backup** | `pg_dump` nightly + filesystem snapshots of `/var/lib/cdd-rules` | RPO ~24h is fine for low-volume compliance data |

---

## 5. Canonical Rule Schema (Pydantic / JSON)

```json
{
  "request_id": "REQUEST-0000001023",
  "rule_class": "country_dispensation_less_stringent",
  "source": {
    "standard": "Group CDD Standards",
    "section": "Appendix G",
    "topic": "Identification and Verification (ID&V) for individuals related to entity clients"
  },
  "baseline_requirement": {
    "attribute": "date_of_birth",
    "verification_granularity": ["day", "month", "year"]
  },
  "dispensation": {
    "trigger": {
      "document_presented": "HKID",
      "document_condition": "HKID does not contain day and month of birth"
    },
    "exception": {
      "attribute": "date_of_birth",
      "verification_granularity": ["year"]
    },
    "applies_to_risk_ratings": ["low", "medium", "high"]
  },
  "scope": {
    "country": "HK",
    "legal_entity": "SCB Hong Kong",
    "client_types": ["all_individuals", "individuals_related_to_entity_clients"],
    "business_segments": ["CCIB", "BB", "PPB"]
  },
  "documentation_action": "Business defines consistent DOB documentation approach in CDD file",
  "expiry_date": "2026-10-12",
  "lifecycle": {
    "status": "PENDING_REVIEW",
    "extracted_at": "2026-05-18T10:23:00Z",
    "extracted_by": "gpt-4o-2026-xx",
    "confidence": 0.94
  }
}
```

---

## 6. Key Design Decisions & Tradeoffs

| Decision | Why |
|---------|-----|
| **Two-stage LLM (extract → reason)** instead of one big prompt | Smaller prompts = higher accuracy + cheaper retries; Stage-1 output is auditable JSON before any reasoning happens. |
| **Structured Outputs (JSON Schema)** over free-form parsing | Eliminates JSON-parse failures; Pydantic validation becomes a contract. |
| **Celery + Redis** instead of in-process async | Survives API restarts; retries on GPT-4o transient failures; concurrency control for API rate limits. |
| **No vector DB / RAG** in v1 | Volume is low and rules are self-contained tables — RAG adds complexity without value. Can add `pgvector` later for cross-rule semantic search. |
| **Mammoth.js for the reviewer view** | Reviewers must see the *exact* source layout to trust extraction. PDF rendering of DOCX loses fidelity. |
| **Versioned prompts in Git** | Compliance audits will ask "what prompt produced this rule?" — keep prompts as code. |
| **Append-only `rule_versions` + `audit_log`** | Regulator-grade traceability; every reviewer edit is preserved. |
| **Single-server Docker Compose, not Kubernetes** | <100 docs/day; k8s operational tax outweighs benefit. |
| **GPT-4o (cloud API) on an otherwise on-prem server** | Confirmed acceptable; only egress is the OpenAI endpoint. Recommend egress allow-list + outbound proxy logging for data-governance review. |

---

## 7. Open Items to Confirm

1. **Data residency for GPT-4o** — Does compliance allow rule text (which references regulatory standards but not PII) to leave the on-prem boundary? If not, swap to Azure OpenAI in a regional tenant or an on-prem model.
2. **Reviewer roles** — Single-reviewer approval, or maker/checker (2-person sign-off)? The latter is common for CDD content; easy to add.
3. **Rule deltas** — When a new version of the same `REQUEST-xxxx` is uploaded, do we want automatic diffing against the prior approved version? Recommended; cheap to add.
4. **Integration outbound** — Even though the goal is reporting only, do downstream systems (CLM, screening engine) need a read-only API to consume `ACTIVE` rules? Worth exposing `/v1/rules` from day one if so.
