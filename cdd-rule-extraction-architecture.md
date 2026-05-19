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

## 5. LLM Pipeline — Stage-1 (Extraction)

Stage-1 converts parsed DOCX cells into typed JSON **verbatim** — no inference, no normalisation, no date parsing. It is the auditable "what the document says" snapshot.

### 5.1 Stage-1 JSON Schema (OpenAI Structured Outputs)

```json
{
  "name": "DispensationRuleExtraction",
  "strict": true,
  "schema": {
    "type": "object",
    "additionalProperties": false,
    "required": ["request_id", "title", "row_a", "row_b", "row_c", "row_d", "extraction_meta"],
    "properties": {
      "request_id":     { "type": "string", "description": "e.g. REQUEST-0000001023" },
      "title":          { "type": "string", "description": "Header row text" },
      "reference_code": { "type": "string", "description": "e.g. A-1" },

      "row_a": {
        "type": "object", "additionalProperties": false,
        "required": ["heading", "value"],
        "properties": {
          "heading": { "type": "string" },
          "value":   { "type": "string", "description": "Verbatim text of cell A" }
        }
      },
      "row_b": {
        "type": "object", "additionalProperties": false,
        "required": ["heading", "value"],
        "properties": {
          "heading": { "type": "string" },
          "value":   { "type": "string", "description": "Verbatim baseline requirement" }
        }
      },
      "row_c": {
        "type": "object", "additionalProperties": false,
        "required": ["heading", "paragraphs"],
        "properties": {
          "heading":    { "type": "string" },
          "paragraphs": { "type": "array", "items": { "type": "string" } }
        }
      },
      "row_d": {
        "type": "object", "additionalProperties": false,
        "required": ["applicable_client_type", "applicable_business_segment", "expiry_date_text"],
        "properties": {
          "applicable_client_type":      { "type": "array", "items": { "type": "string" } },
          "applicable_business_segment": { "type": "array", "items": { "type": "string" } },
          "expiry_date_text":            { "type": "string", "description": "Raw date as printed, e.g. '12 Oct 2026'" }
        }
      },
      "extraction_meta": {
        "type": "object", "additionalProperties": false,
        "required": ["model", "confidence", "warnings"],
        "properties": {
          "model":      { "type": "string" },
          "confidence": { "type": "number", "minimum": 0, "maximum": 1 },
          "warnings":   { "type": "array", "items": { "type": "string" } }
        }
      }
    }
  }
}
```

### 5.2 Stage-1 Prompt

**System prompt**

```text
You are a document-extraction service for a global bank's compliance team.
You receive the parsed contents of a single Country Dispensation table from a
Microsoft Word document. Your job is to capture what the table SAYS, verbatim,
into the provided JSON Schema. You are NOT interpreting, reasoning, normalising,
or reformatting policy content. A separate downstream step will do that.

OPERATING RULES (must follow):

1. FAITHFULNESS
   - Copy cell text exactly as it appears. Preserve punctuation, quotation
     marks, capitalisation, and inline abbreviations (e.g. "DOB", "ID&V",
     "HKID card").
   - Do NOT translate, summarise, paraphrase, expand acronyms, or correct
     spelling/grammar.
   - Do NOT convert dates, codes, or names to canonical forms. "12 Oct 2026"
     stays "12 Oct 2026".

2. ROW MAPPING
   - The source table has four labelled rows: A, B, C, D.
   - Row A -> `row_a` (Group CDD Standards document + section reference).
   - Row B -> `row_b` (the Group baseline requirement).
   - Row C -> `row_c` (the requested dispensation, often multi-paragraph).
   - Row D -> `row_d` (applicability metadata: client type, business segment,
     expiry date).
   - The header row above A holds the title (e.g. "Country Dispensation
     (Less Stringent Requirement)") and reference code (e.g. "A-1") -> map
     to `title` and `reference_code`.
   - The `REQUEST-xxxxxxxxxx` identifier is the `request_id`.

3. ROW C PARAGRAPHS
   - Split cell C into one array entry per visual paragraph in the source.
   - Keep each paragraph intact; do not merge or further split.

4. ROW D LISTS
   - `applicable_client_type` and `applicable_business_segment` are arrays.
   - Split on newlines, semicolons, or bullet markers as they appear in the
     cell. Trim whitespace. Do not deduplicate, reorder, or rename items.
   - `expiry_date_text` is the raw printed date string.

5. MISSING / UNREADABLE CONTENT
   - If a cell is empty, emit an empty string (or empty array where the
     schema requires an array).
   - If a cell contains content you cannot extract (e.g. an image, a
     redaction, a corrupted run), emit what you can and add a concise note
     to `extraction_meta.warnings`, e.g. "row_b contains an embedded image
     that was not transcribed".

6. CONFIDENCE
   - Set `extraction_meta.confidence` in [0, 1] based on text clarity and
     row-mapping certainty. Reduce confidence if the table layout deviates
     from the standard A/B/C/D structure.

7. OUTPUT FORMAT
   - Emit only the JSON object defined by the response schema. No prose,
     no markdown, no code fences.
```

**User prompt (template)**

```text
Extract the following parsed Word table into the Stage-1 schema.

source_filename: {{ original_docx_filename }}
parsed_table:
{{ table_as_structured_text }}
```

Where `parsed_table` is the output of `python-docx` rendered as a labelled
block per cell — for example:

```text
HEADER | title: "Country Dispensation (Less Stringent Requirement)" | ref: "A-1"
REQUEST_ID | "REQUEST-0000001023"
ROW A | heading: "Group CDD Standards – Document Name and Section Number"
ROW A | value:   "Group CDD Standards; Appendix G: ..."
ROW B | heading: "Requirement from the Group CDD Standards"
ROW B | value:   "Under Appendix G, where the Date of Birth ..."
ROW C | heading: "Requested Country Dispensation"
ROW C | paragraph[0]: "Where the DOB of individuals ..."
ROW C | paragraph[1]: "For such individuals, Business would ..."
ROW D | applicable_client_type:      ["All individual clients", "All individuals related to entity clients"]
ROW D | applicable_business_segment: ["CCIB", "BB", "PPB"]
ROW D | expiry_date_text:            "12 Oct 2026"
```

The Python call mirrors Stage-2 but uses `DispensationRuleExtraction` as the
`response_format` schema, `temperature=0`, `seed=42`.

---

### 5.3 Stage-1 Output for REQUEST-0000001023

```json
{
  "request_id": "REQUEST-0000001023",
  "title": "Country Dispensation (Less Stringent Requirement)",
  "reference_code": "A-1",
  "row_a": {
    "heading": "Group CDD Standards – Document Name and Section Number",
    "value": "Group CDD Standards; Appendix G: Identification and Verification (\"ID&V\") requirements for individuals related to Entity Clients"
  },
  "row_b": {
    "heading": "Requirement from the Group CDD Standards",
    "value": "Under Appendix G, where the Date of Birth (\"DOB\") of individuals is required to be verified, the day, month and year of such date must be provided."
  },
  "row_c": {
    "heading": "Requested Country Dispensation",
    "paragraphs": [
      "Where the DOB of individuals is required to be verified under Appendix G and such individual provides his/her Hong Kong Identification Card (\"HKID card\") as proof of identity, SCB Hong Kong may verify only the year of birth where such HKID card does not contain the day and month of birth. For avoidance of doubt, this applies to all risk ratings.",
      "For such individuals, Business would define a consistent approach in documenting their date of birth in the client's CDD file."
    ]
  },
  "row_d": {
    "applicable_client_type": [
      "All individual clients",
      "All individuals related to entity clients"
    ],
    "applicable_business_segment": ["CCIB", "BB", "PPB"],
    "expiry_date_text": "12 Oct 2026"
  },
  "extraction_meta": {
    "model": "gpt-4o-2026-xx",
    "confidence": 0.97,
    "warnings": []
  }
}
```

---

## 6. LLM Pipeline — Stage-2 (Reasoning & Canonicalisation)

Stage-2 consumes the Stage-1 JSON and produces the **canonical rule** used by downstream storage, dashboards, and reports. It classifies the rule, normalises enums, parses dates to ISO, builds a machine-readable condition tree, and writes a plain-English summary.

### 6.1 Stage-2 JSON Schema (OpenAI Structured Outputs)

```json
{
  "name": "DispensationRuleCanonical",
  "strict": true,
  "schema": {
    "type": "object",
    "additionalProperties": false,
    "required": [
      "request_id", "rule_class", "source", "baseline_requirement",
      "dispensation", "scope", "documentation_action", "expiry_date",
      "summary_plain_english", "machine_conditions", "reasoning_meta"
    ],
    "properties": {
      "request_id": { "type": "string" },

      "rule_class": {
        "type": "string",
        "enum": [
          "country_dispensation_less_stringent",
          "country_dispensation_more_stringent",
          "clarification",
          "scope_restriction"
        ]
      },

      "source": {
        "type": "object", "additionalProperties": false,
        "required": ["standard", "section", "topic"],
        "properties": {
          "standard": { "type": "string" },
          "section":  { "type": "string" },
          "topic":    { "type": "string" }
        }
      },

      "baseline_requirement": {
        "type": "object", "additionalProperties": false,
        "required": ["attribute", "verification_granularity", "narrative"],
        "properties": {
          "attribute":                { "type": "string", "description": "Canonical attribute key, e.g. date_of_birth" },
          "verification_granularity": {
            "type": "array",
            "items": { "type": "string", "enum": ["day", "month", "year", "full_value", "presence_only"] }
          },
          "narrative":                { "type": "string" }
        }
      },

      "dispensation": {
        "type": "object", "additionalProperties": false,
        "required": ["trigger", "exception", "applies_to_risk_ratings"],
        "properties": {
          "trigger": {
            "type": "object", "additionalProperties": false,
            "required": ["document_presented", "document_condition"],
            "properties": {
              "document_presented": { "type": "string", "description": "Canonical doc code, e.g. HKID" },
              "document_condition": { "type": "string" }
            }
          },
          "exception": {
            "type": "object", "additionalProperties": false,
            "required": ["attribute", "verification_granularity"],
            "properties": {
              "attribute":                { "type": "string" },
              "verification_granularity": {
                "type": "array",
                "items": { "type": "string", "enum": ["day", "month", "year", "full_value", "presence_only"] }
              }
            }
          },
          "applies_to_risk_ratings": {
            "type": "array",
            "items": { "type": "string", "enum": ["low", "medium", "high"] }
          }
        }
      },

      "scope": {
        "type": "object", "additionalProperties": false,
        "required": ["country", "legal_entity", "client_types", "business_segments"],
        "properties": {
          "country":           { "type": "string", "description": "ISO-3166 alpha-2, e.g. HK" },
          "legal_entity":      { "type": "string" },
          "client_types": {
            "type": "array",
            "items": {
              "type": "string",
              "enum": ["all_individuals", "individuals_related_to_entity_clients", "entity_clients", "high_net_worth"]
            }
          },
          "business_segments": {
            "type": "array",
            "items": { "type": "string", "enum": ["CCIB", "BB", "PPB", "WM"] }
          }
        }
      },

      "documentation_action": { "type": "string", "description": "Operational instruction the business must follow" },

      "expiry_date": {
        "type": "string",
        "description": "ISO-8601 date, YYYY-MM-DD",
        "pattern": "^\\d{4}-\\d{2}-\\d{2}$"
      },

      "summary_plain_english": { "type": "string", "description": "One-paragraph reviewer-friendly summary" },

      "machine_conditions": {
        "type": "object",
        "additionalProperties": false,
        "required": ["if", "then"],
        "properties": {
          "if": {
            "type": "object",
            "additionalProperties": false,
            "required": ["all"],
            "properties": {
              "all": {
                "type": "array",
                "items": {
                  "type": "object",
                  "additionalProperties": false,
                  "required": ["fact", "op", "value"],
                  "properties": {
                    "fact":  { "type": "string", "description": "e.g. id_document.type" },
                    "op":    { "type": "string", "enum": ["equals", "in", "not_equals", "contains", "missing"] },
                    "value": { "type": "string" }
                  }
                }
              }
            }
          },
          "then": {
            "type": "object",
            "additionalProperties": false,
            "required": ["action", "attribute", "value"],
            "properties": {
              "action":    { "type": "string", "enum": ["relax_verification", "tighten_verification", "require_additional_doc", "no_change"] },
              "attribute": { "type": "string" },
              "value":     { "type": "string" }
            }
          }
        }
      },

      "reasoning_meta": {
        "type": "object", "additionalProperties": false,
        "required": ["model", "confidence", "warnings", "stage1_input_hash"],
        "properties": {
          "model":             { "type": "string" },
          "confidence":        { "type": "number", "minimum": 0, "maximum": 1 },
          "warnings":          { "type": "array", "items": { "type": "string" } },
          "stage1_input_hash": { "type": "string", "description": "SHA-256 of Stage-1 JSON, ties reasoning to extraction" }
        }
      }
    }
  }
}
```

### 6.2 Stage-2 Prompt

**System prompt**

```text
You are a compliance rules canonicaliser for a global bank's Customer Due
Diligence (CDD) programme. You receive a Stage-1 JSON object that contains the
verbatim contents of a Country Dispensation request table (rows A, B, C, D).
Your job is to produce a Stage-2 canonical rule that conforms exactly to the
provided JSON Schema.

OPERATING RULES (must follow):

1. FAITHFULNESS
   - Do not invent facts that are not present or directly implied by the
     Stage-1 input. If something required by the schema is genuinely not
     determinable, leave the field empty where the schema allows, and add a
     concise note to `reasoning_meta.warnings`.
   - Never alter the substantive meaning of the requirement or the
     dispensation. You are normalising, not rewriting policy.

2. CLASSIFICATION (`rule_class`)
   - `country_dispensation_less_stringent`: the dispensation relaxes the
     Group standard (e.g. fewer fields verified, fewer documents required).
   - `country_dispensation_more_stringent`: the dispensation tightens the
     Group standard (e.g. additional documents, narrower validity).
   - `clarification`: same effective requirement, just clarified wording.
   - `scope_restriction`: the dispensation only narrows applicability.
   Choose exactly one. Use the wording of the title (`Less Stringent` /
   `More Stringent`) and the substance of rows B vs C to decide.

3. CANONICAL ENUMS
   - `scope.country`: ISO-3166 alpha-2 (e.g. "Hong Kong" -> "HK").
   - `scope.client_types`: map free text to the schema enum. Examples:
       "All individual clients"                       -> all_individuals
       "All individuals related to entity clients"    -> individuals_related_to_entity_clients
       "Entity clients" / "Corporate clients"         -> entity_clients
   - `scope.business_segments`: only allow CCIB, BB, PPB, WM. Pass through
     codes that already match; reject unknown codes with a warning.
   - `dispensation.trigger.document_presented`: map common identity documents
     to canonical codes:
       "Hong Kong Identification Card" / "HKID card" -> HKID
       "Passport"                                     -> PASSPORT
       "National Identity Card"                       -> NATIONAL_ID
       "Driving Licence"                              -> DRIVING_LICENCE
     If the document is not in this list, use an uppercase SNAKE_CASE form
     and add a warning.

4. RISK RATINGS
   - If the text says "all risk ratings", emit `["low","medium","high"]`.
   - If specific ratings are named, emit only those.

5. DATE NORMALISATION
   - Convert `row_d.expiry_date_text` to ISO-8601 `YYYY-MM-DD`.
   - If the year is ambiguous (2-digit), refuse and add a warning instead of
     guessing.

6. VERIFICATION GRANULARITY
   - Use the controlled vocabulary: day, month, year, full_value, presence_only.
   - "day, month and year" -> ["day","month","year"]
   - "year of birth"       -> ["year"]

7. MACHINE CONDITIONS (`machine_conditions`)
   - Build a single `if.all` conjunction of facts that must hold for the
     dispensation to apply, plus a `then` clause describing the action.
   - Facts use dotted paths from this whitelist:
       client.country, client.type, client.risk_rating,
       id_document.type, id_document.dob_day, id_document.dob_month,
       id_document.dob_year
   - Ops: equals, in, not_equals, contains, missing.
   - For `in`, the `value` is a comma-separated list of enum values.
   - Keep the condition tree minimal but complete enough that a downstream
     rules engine could evaluate it deterministically.

8. SUMMARY
   - `summary_plain_english` is one paragraph (2-4 sentences) addressed to a
     compliance reviewer. State who it applies to, what triggers it, what
     changes, and when it expires. No marketing language.

9. TRACEABILITY
   - Copy `request_id` from the Stage-1 input unchanged.
   - Set `reasoning_meta.model` to your model identifier.
   - Set `reasoning_meta.stage1_input_hash` to the SHA-256 hash that is
     provided in the user message (do not compute it yourself).
   - Set `reasoning_meta.confidence` to your honest self-assessment in [0,1]
     based on input clarity and enum-mapping certainty.
   - Add a warning whenever you (a) chose an enum value by inference,
     (b) could not parse a field, or (c) noticed an internal contradiction
     in the source.

10. OUTPUT FORMAT
    - Emit only the JSON object defined by the response schema. No prose,
      no markdown, no code fences.
```

**User prompt (template)**

```text
Canonicalise the following Stage-1 extraction into a Stage-2 canonical rule.

stage1_input_hash: {{ sha256_of_stage1_json }}

stage1_json:
{{ stage1_json_pretty_printed }}
```

**Python call (OpenAI SDK)**

```python
from openai import OpenAI
from hashlib import sha256
import json

client = OpenAI()

stage1_bytes = json.dumps(stage1_obj, sort_keys=True, separators=(",", ":")).encode()
stage1_hash  = "sha256:" + sha256(stage1_bytes).hexdigest()

response = client.chat.completions.create(
    model="gpt-4o",
    temperature=0,
    seed=42,
    messages=[
        {"role": "system", "content": STAGE2_SYSTEM_PROMPT},
        {"role": "user", "content": (
            "Canonicalise the following Stage-1 extraction into a Stage-2 "
            "canonical rule.\n\n"
            f"stage1_input_hash: {stage1_hash}\n\n"
            "stage1_json:\n"
            f"{json.dumps(stage1_obj, indent=2, ensure_ascii=False)}"
        )},
    ],
    response_format={
        "type": "json_schema",
        "json_schema": DISPENSATION_RULE_CANONICAL_SCHEMA,  # from §6.1
    },
)

stage2_obj = json.loads(response.choices[0].message.content)
```

**Prompt-engineering rationale**

| Choice | Why |
|---|---|
| `temperature=0` + fixed `seed` | Deterministic, reproducible reasoning — required for audit. |
| System prompt holds rules, user prompt holds data | Prompt versioning in Git is decoupled from data flow; system prompt is the auditable artefact. |
| Hash passed in, not computed by the LLM | LLMs are unreliable at hashing; computing it host-side guarantees a correct trace link. |
| Schema enforced via `response_format=json_schema` | Even if the prompt drifts, output is structurally valid; Pydantic only verifies semantics. |
| Whitelisted fact paths (rule 7) | Prevents the model inventing fact names the rules engine cannot evaluate. |
| Warnings are first-class | Reviewer UI surfaces them so low-confidence rules are scrutinised, not rubber-stamped. |

---

### 6.3 Stage-2 Output for REQUEST-0000001023

```json
{
  "request_id": "REQUEST-0000001023",
  "rule_class": "country_dispensation_less_stringent",

  "source": {
    "standard": "Group CDD Standards",
    "section": "Appendix G",
    "topic": "Identification and Verification (ID&V) for individuals related to Entity Clients"
  },

  "baseline_requirement": {
    "attribute": "date_of_birth",
    "verification_granularity": ["day", "month", "year"],
    "narrative": "Under Appendix G, the day, month and year of the individual's DOB must be provided when verifying DOB."
  },

  "dispensation": {
    "trigger": {
      "document_presented": "HKID",
      "document_condition": "HKID card does not contain day and month of birth"
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

  "documentation_action": "Business must define a consistent approach to documenting the individual's date of birth in the client's CDD file.",

  "expiry_date": "2026-10-12",

  "summary_plain_english": "For SCB Hong Kong clients (individuals and individuals related to entity clients across CCIB, BB and PPB), when an HKID card is used as proof of identity and the HKID does not show the day and month of birth, only the year of birth needs to be verified. This relaxation applies to all risk ratings and expires on 12 Oct 2026. Business must still document a consistent DOB approach in the CDD file.",

  "machine_conditions": {
    "if": {
      "all": [
        { "fact": "client.country",        "op": "equals", "value": "HK" },
        { "fact": "client.type",           "op": "in",     "value": "all_individuals,individuals_related_to_entity_clients" },
        { "fact": "id_document.type",      "op": "equals", "value": "HKID" },
        { "fact": "id_document.dob_day",   "op": "missing","value": "" },
        { "fact": "id_document.dob_month", "op": "missing","value": "" }
      ]
    },
    "then": {
      "action": "relax_verification",
      "attribute": "date_of_birth",
      "value": "year_only"
    }
  },

  "reasoning_meta": {
    "model": "gpt-4o-2026-xx",
    "confidence": 0.94,
    "warnings": [],
    "stage1_input_hash": "sha256:b7c1…e92a"
  }
}
```

### 6.4 What Stage-2 Adds On Top of Stage-1

| Concern | Stage-1 | Stage-2 |
|---|---|---|
| Date | `"12 Oct 2026"` (string) | `"2026-10-12"` (ISO) |
| Client type | Free-text list from cell | Canonical enum |
| Business segment | Free-text list from cell | Validated against enum (`CCIB`/`BB`/`PPB`/`WM`) |
| Risk ratings | Implicit ("applies to all risk ratings" in narrative) | Explicit array `["low","medium","high"]` |
| Rule type | Not classified | `rule_class` enum |
| Trigger document | Mentioned in prose | Canonical code (`HKID`) + condition string |
| Logic | Prose paragraphs | `machine_conditions` if/then tree (rules-engine-ready) |
| Country | Implicit ("SCB Hong Kong") | ISO-3166 `HK` |
| Traceability | Standalone | `stage1_input_hash` ties back to verbatim extraction |

This canonical Stage-2 object is what gets stored in the `rules` table, displayed to the reviewer, and exported to reports.

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
