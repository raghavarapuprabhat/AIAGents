# Compass AI Layer — Low-Level Design & Implementation Specification (Phase 2)

**Audience:** Implementation engineers / AI-assisted development (Claude/Sonnet)
**Baseline:** the **working Phase-1 Compass application** built from the Compass LLD (PoC edition): TypeScript agent + Studio, Java 21/Spring services, PostgreSQL (incl. `events` table), local object store, local Ed25519 keys, `X-Dev-User` identity, no containers.
**Parent documents:** Architecture v2.0 (AI edition) for rationale and governance; this LLD is the buildable spec.

---

## 0. Ground rules (normative)

1. **Strictly additive.** No existing table is altered (new migrations `V91+` only), no existing endpoint changes shape, no agent behavior changes with flags off. **All AI flags off ⇒ the system is byte-identical to Phase 1.** CI enforces this with a "flags-off regression" run of the full Phase-1 gallery + e2e suite against the Phase-2 build.
2. **AI proposes; the Phase-1 machinery disposes.** Every AI output lands in an existing Phase-1 pathway: drafts into maker-checker, heal proposals into the existing heal queue, translations into the existing `machine` status, assistant flow-launches through the existing deep-link mechanism. No new publish path, no auto-approval, ever.
3. **One choke point.** Every model call — generation, embedding, classification — goes through the new `ai-gateway` service. Nothing else may hold a model endpoint or key. The agent never calls models directly; the assistant talks to `ai-gateway` via the edge like any other API.
4. **Grounded or silent.** User-facing generation (C1) answers only from retrieved, tenant-published content with citations; empty retrieval ⇒ honest "I don't have guidance on that" + human-support link. Internal generation (Tier 1) is always human-reviewed, so it may generalize.
5. **PoC pragmatics.** Model serving defaults to a **local OpenAI-compatible endpoint** (Ollama or vLLM on the dev box/GPU host); the external provider adapter (e.g., Anthropic API) exists behind config and is OFF by default. Same PII shield either way.

## 0.1 Repo additions

```
compass/
├── services/
│   └── ai-gateway/            # Java 21/Spring Boot — providers, guardrails, prompts, jobs
│       ├── src/main/java/...  # GatewayController, ProviderRouter, Guardrails, PromptRegistry
│       ├── prompts/           # versioned YAML prompt templates (hash-logged per call)
│       └── openapi.yaml
├── packages/agent/src/
│   ├── assist/                # C1 chat widget (flag-gated, lazy-loaded)
│   └── struggle/              # B2 heuristics (flag-gated)
├── packages/studio-console/src/ai/   # Tier-1 UI: draft, copy assist, QA, translations, insights copilot
├── infra/pg/V91..V96__*.sql   # additive migrations only
└── mockups/mockup-ai-console.html · mockup-ai-assistant.html   # Part 8, normative UI reference
```

---

# PART 1 — AI Plane foundations

## 1.1 Additive migrations

```sql
-- V91: capability flags (per tenant per capability; all default OFF)
CREATE TABLE ai_capability_flag (
  tenant_id text REFERENCES tenant, capability text,          -- 'A1','A2','A3','A4','A5','B1','B2','B3','B4','B5','C1','C2'
  enabled bool NOT NULL DEFAULT false, config jsonb NOT NULL DEFAULT '{}',
  updated_by text, updated_at timestamptz DEFAULT now(),
  PRIMARY KEY (tenant_id, capability));

-- V92: RAG store (requires CREATE EXTENSION IF NOT EXISTS vector)
CREATE TABLE rag_chunk (
  id bigserial PRIMARY KEY, tenant_id text, locale text,
  source_type text CHECK (source_type IN ('guide','kb')), source_id text, source_version int,
  bundle_hash text,                       -- provenance: which published bundle this chunk came from
  chunk text NOT NULL, embedding vector(768) NOT NULL,
  UNIQUE (tenant_id, source_type, source_id, source_version, locale, md5(chunk)));
CREATE INDEX rag_chunk_ann ON rag_chunk USING hnsw (embedding vector_cosine_ops);

-- V93: full AI audit (append-only, like audit_event)
CREATE TABLE ai_interaction (
  id bigserial PRIMARY KEY, ts timestamptz DEFAULT now(),
  tenant_id text, capability text, actor text,                -- dev identity or 'agent:'+anon hash for C1
  provider text, model text, prompt_template text, prompt_version text,
  prompt_hash text, output_hash text, input_tokens int, output_tokens int, latency_ms int,
  outcome text CHECK (outcome IN ('ok','blocked_input','blocked_output','ungrounded','error')),
  disposition text);                                          -- e.g. 'draft gd_x1 v1 created', 'answer served w/ 2 citations'

-- V94: translation glossary (A3)
CREATE TABLE glossary (tenant_id text, locale text, term text, translation text, PRIMARY KEY (tenant_id, locale, term));

-- V95: proficiency (C2) — deterministic nightly job output
CREATE TABLE user_proficiency (tenant_id text, u text, score numeric, depth text CHECK (depth IN ('full','tips','minimal')),
  computed_at timestamptz, PRIMARY KEY (tenant_id, u));

-- V96: guide depth variants (C2) — additive column, nullable = Phase-1 behavior
ALTER TABLE guide_version ADD COLUMN depth_variants jsonb;    -- {"tips": {...overrides}, "minimal": {...}} — validated by schema v1.1
```

Schema note: `guide.schema.json` bumps to **v1.1** adding only optional fields (`depth_variants`, `assist_keywords`); v1.0 documents remain valid (agent ignores unknown optional fields already).

## 1.2 `ai-gateway` service

**Endpoints** (all require `X-Dev-User`; per-capability flag checked server-side before any provider call):

| Method/Path | Purpose |
|---|---|
| `POST /api/ai/generate` | `{tenant, capability, template, vars, locale}` → guardrails → prompt assembly → provider → guardrails → structured JSON output (template-declared schema) |
| `POST /api/ai/embed` | batch text → vectors (embedding model) |
| `POST /api/ai/assist` | C1 only: `{tenant, locale, question, page, history[]}` → RAG retrieve → grounded generate → `{answer_ast, citations[], actions[]}` |
| `POST /api/ai/jobs/{type}/run` | trigger async jobs (index, heal-scan, verbatim, gap-mine, proficiency) — also cron-scheduled |
| `GET /api/ai/flags?tenant=` | effective flags for console + (agent-relevant subset) for manifest injection |

**Provider router.** Adapters behind one interface: `LOCAL_OPENAI` (Ollama/vLLM, `http://localhost:11434/v1`, models pinned in config: one 7–8B instruct for generation/classification, `nomic-embed-text`-class 768-d for embeddings) and `ANTHROPIC` (optional, off by default; requires `ai.external.enabled=true` AND per-capability `allow_external`). Temperature 0 for all structured tasks. Every call writes one `ai_interaction` row — including blocked ones.

**Prompt registry.** Templates are YAML files in-repo: `id`, `version`, `system`, `user` (Mustache vars), `output_schema` (JSON Schema the response must validate against; one retry with repair instruction on failure, then `outcome=error`). Template hash logged per call so any output is reproducible to an exact prompt.

**Guardrails module (applies in order):**
1. *PII shield (input):* `contracts/pii-patterns.json` scrub over all user-supplied vars before assembly — even for local models.
2. *Injection screen:* strip/neutralize instruction-like sequences in retrieved chunks and user text ("ignore previous…", role tags); retrieved content is wrapped in a `<data>` envelope and the system prompt states data-not-instructions.
3. *Topic lock (C1):* lightweight classifier prompt — question must be about using the application; financial-advice / off-topic ⇒ polite scope message, `outcome=blocked_input`.
4. *Output filter:* PII echo scan; C1 grounding check (below); schema validation.
5. *Limits:* per-capability token ceilings; C1 per-session 20 turns/hr, per-tenant QPS from `config`.

**C1 grounding check (hard rule):** the output schema requires `citations: [chunk_id...]` non-empty and `answer_only_from_citations: true` self-attestation; the gateway then verifies every cited `chunk_id` exists in the retrieval set **and** cosine(answer embedding, best-cited-chunk) ≥ 0.55; failure ⇒ `outcome=ungrounded`, serve the fallback message. Red-team fixture suite (Part 7) gates this.

---

# PART 2 — Tier 1: authoring assist (console-only; output = drafts)

All Tier-1 UI lives in the console (and one extension hook for A5); every action shows a "✨ AI" affordance only when the tenant flag is on, and every product of these features enters review with an `origin: ai_assisted` badge visible to checkers.

**A1 — Draft-from-document.** Console: *Guides → New guide → Draft with AI → paste text / upload .md/.txt* (PoC: text only; PDF/docx extraction deferred). Gateway template `a1_draft_flow@1`: input SOP text → output a v1.1 guide JSON **without anchors**: each step carries `anchor_hint` (human phrase, e.g. "the Submit transfer button") instead of `anchor`. Draft is created via the existing Content Service create API. The extension then offers **"Capture walk"**: iterates hint-carrying steps exactly like migration mode — author clicks the element per hint, fingerprint captured, hint replaced. AC: a 2-page SOP becomes a submittable 5-step draft in ≤10 minutes including capture.

**A2 — Copy assistant.** In the console step editor (and the extension bubble via the same API): selection → *Rewrite: concise / friendly / plain-language / fix grammar*. Template `a2_rewrite@1` returns rich-text AST (not free text) validated against the AST schema — the constrained node set is enforced by construction. Result replaces selection **in the editor only**; author still saves/submits normally.

**A3 — AI translation.** Translations screen: *Pre-fill locale with AI* per guide. Batch job: untranslated keys → `a3_translate@1` (vars: source strings, target locale, glossary rows) → writes translation table rows with status `machine` (existing state). Post-check: every glossary term found in source must map to its mandated translation, else the string is flagged `glossary_miss` and left empty. RTL preview flow unchanged. AC: glossary compliance 100% on the fixture set; human review still required to reach `approved`.

**A4 — QA copilot.** On *Submit for review*, a synchronous check pass annotates the review item: deterministic checks first (unreachable steps, missing locales, missing alt text — no model), then `a4_review@1` over the step copy for tone-guide violations and unapproved-claim phrasing (tenant tone guide + banned-phrase list from `config`). Findings render as advisory chips on the review screen; they never block — the checker decides. AC: zero false "blocking" behavior; findings list reproducible at temperature 0.

**A5 — Draft-from-recording.** The Phase-1 recordings endpoint already stores click-streams **with fingerprints captured at record time**. Job `a5_recording@1` converts a recording into a draft flow: step per meaningful click (filtering scrolls/focus noise via deterministic rules), fingerprints attached directly (no capture walk needed), step titles/bodies generated from element context, marked for author edit. AC: a recorded 6-click journey yields a draft whose anchors all resolve ≥0.85 on the same page.

---

# PART 3 — Tier 2: advisory intelligence (jobs + console surfaces)

**B1 — Semantic anchor healing.** Additive agent change (flag `B2`? no — `B1`, agent side flag `ai.heal_context`): when resolution lands <0.85, the low-confidence/fail beacon gains `m.cands`: up to 3 candidate summaries `{tag, id?, data_guide?, aria_name?, text≤80}` — PII-scrubbed client-side, size-capped 1 KB. Nightly job: for each `anchor.health='degraded'`, `b1_heal@1` receives stored fingerprint + aggregated candidate summaries → proposes a repaired fingerprint + confidence + one-line rationale → inserted into the **existing** heal queue with `origin: ai`, rendered with the same one-click approve. AC: on the gallery drift fixtures (renamed ids, reworded labels), ≥70% of proposals are the correct element; zero proposals auto-apply.

**B2 — Struggle detection.** Pure deterministic agent module (no model): signals — ≥3 clicks on one element <2 s (rage), ≥3 validation errors same field, back-forth navigation ×2 <30 s, idle >60 s on a form with focus. Emits `struggle` event (existing pipeline) and sets a session env var `struggle=true` that the **existing targeting bytecode** can reference (`LOAD_ENV` gains one name — additive opcode operand, vector file extended). Authors attach help via normal targeting rules ("show launcher when struggle"). AC: gallery struggle-simulation page triggers each signal exactly once; flags-off build contains zero struggle code (dead-code eliminated).

**B3 — Insights copilot.** Console Insights gains a question box. **No free-form SQL**: `b3_route@1` maps the question to one of a fixed catalog of parameterized queries (funnel, step drop compare, locale compare, trend, top-struggle pages) + extracted params; the Collector read API executes; `b3_narrate@1` writes a 2–3 sentence narration over the returned numbers only (numbers are injected back verbatim — the model never invents figures; a post-check verifies every number in the narration exists in the result set). Unmappable question ⇒ "I can answer questions about funnels, drop-off, locales, and trends" + catalog chips. AC: number-invention rate 0 on the fixture suite.

**B4 — Survey verbatim intelligence.** Nightly job over new survey text answers (already PII-scrubbed at collection): embed → cluster (k-means over pgvector, k by silhouette in 3..8) → `b4_theme@1` names each cluster and picks 2 representative (already-scrubbed) quotes → themes table rendered on the Insights survey screen with counts and trend vs. prior period. AC: stable theme naming at temperature 0; no verbatim leaves the tenant boundary.

**B5 — Guidance gap mining.** Weekly job: join struggle/abandon hotspots (pages × counts from events) against the content inventory (pages covered by published guides) → ranked "uncovered struggle" list with evidence counts → rendered as an authoring-backlog card on the Guides screen (deterministic; `b5_describe@1` only phrases the one-line suggestion). AC: every suggestion links to the exact event query that produced it.

---

# PART 4 — Tier 3: in-app assistance (agent-side, most gated)

**C1 — Guidance assistant.** Upgrades the Phase-1 assist panel (substring search) into a chat, only when manifest `features.assist_ai=true` for the tenant:

- *Agent:* `assist/` module lazy-loads on first open (keeps boot budget); renders in the same closed Shadow DOM; input box + history (session-only, never persisted server-side beyond `ai_interaction` hashes); visible label **"AI answers from your organisation's guides"** + thumbs feedback + "Contact support" escape hatch (tenant-configured link).
- *Flow:* question → `POST /api/ai/assist` (via edge) with `{question, page:route-only, locale}` — **no DOM content is sent**. Gateway: topic lock → embed → pgvector top-6 (tenant+locale, fallback locale chain) → grounded generate → response `{answer_ast, citations:[{guide_id, title}], actions:[{type:'launch_flow', guide_id}]}`.
- *Render:* answer as constrained AST; citation chips open the guide's launcher context; **"Show me"** button per action triggers the existing deep-link start mechanism — the walk-thru that then runs is 100% Phase-1 code.
- *Indexing:* `rag-index` job runs on every publish (hooked on `publish_record` insert): chunks the published bundle's step/tip/announcement text (per locale) + optional KB markdown files uploaded per tenant → embeddings → `rag_chunk` rows keyed by `bundle_hash`; rollback re-points queries to the prior hash automatically (query filters on the tenant's *current* manifest hash — index rows for old bundles are retained 30 days then purged).
- AC: red-team suite (Part 7) green; ungrounded-question fixture set always yields the fallback; p95 answer latency ≤6 s on the local model; assist OFF ⇒ Phase-1 substring panel byte-identical.

**C2 — Adaptive guidance depth.** No model at runtime. Nightly deterministic job scores proficiency per (tenant, user-hash) from events (flows completed, error rate, tenure claim) → `user_proficiency.depth`. The Identity Adapter stub adds claim `depth` (additive; absent = `full` = Phase-1 behavior). Authors optionally define `depth_variants` per guide (v1.1 schema): `tips` (flow collapses to always-on tips) or `minimal` (guide suppressed except launcher). Agent applies the variant at guide-selection time — a pure lookup. AC: user with no proficiency row behaves exactly as Phase 1; variant application covered by gallery cases per depth.

**C3 — Conversational surveys: explicitly deferred** to a later phase (needs its own consent/UX review); schema leaves room (`survey.followup` reserved field).

---

# PART 5 — Feature-flag plumbing & manifest

- Console Admin gains an **AI capabilities** panel (per tenant, per capability toggle + config editor; audit-logged). Everything defaults OFF.
- Manifest (schema 1 → 1.1, additive field): `"features": {"assist_ai": bool, "struggle": bool}` — the only two agent-relevant flags. Absent field ⇒ both false (Phase-1 manifests remain valid).
- Console/extension read effective flags from `GET /api/ai/flags`; UI affordances render only when enabled — no greyed-out teasers in PoC.

# PART 6 — Observability

Every gateway call → `ai_interaction` (Part 1.1) + Actuator metrics: `ai_calls_total{capability,outcome}`, `ai_latency_ms`, `ai_tokens_total`, `rag_index_lag_seconds`. Console Admin shows a per-tenant AI usage card (calls, block rate, token spend proxy). Weekly job emails/posts the capability register snapshot (what's on, where) — the governance artifact from Architecture v2 §5.

# PART 7 — Testing (gates per milestone)

1. **Flags-off regression:** full Phase-1 gallery + e2e green against the Phase-2 build with every flag off; agent bundle size delta ≤ +1 KB (lazy modules excluded from core path).
2. **Prompt-template unit tests:** each template has ≥5 fixtures (input vars → schema-valid output at temperature 0 against the pinned local model; assertions on structure + key invariants, not exact wording).
3. **Grounding red-team suite (C1):** ≥60 cases — prompt-injection in questions, injection embedded in indexed guide text, off-topic finance questions, unanswerable questions, cross-tenant probe attempts; pass = 100% correct blocking/fallback, zero cross-tenant retrieval.
4. **Number-integrity suite (B3):** narration contains only result-set numbers.
5. **Heal-proposal benchmark (B1):** gallery drift fixtures, ≥70% top-1 correct.
6. **PII shield vectors:** shared `pii-vectors.json` extended with prompt-context cases; runs in gateway CI.

# PART 8 — Milestones

| Milestone | Contents | Exit |
|---|---|---|
| AI-0 (wk 1–2) | ai-gateway skeleton, provider router (local), prompt registry, V91–V93, flags panel, `ai_interaction` audit | Flags-off regression green; one template round-trips |
| AI-1 (wk 2–5) | Tier 1: A2 → A3 → A4 → A1 → A5 (this order: smallest blast radius first) | Part-2 ACs; authoring-time benchmark recorded |
| AI-2 (wk 5–8) | Tier 2: B2 (deterministic) → B1 → B4 → B3 → B5; V94 | Part-3 ACs; heal benchmark ≥70% |
| AI-3 (wk 8–11) | C1 assistant + RAG indexer; red-team suite; V92 in anger | Part-7 §3 green; pilot tenant demo |
| AI-4 (wk 11–12) | C2 depth (V95/V96), usage dashboards, capability register | Flags-off regression re-run; phase review |

# PART 9 — UI Reference Mockups (normative for Parts 2–4 UI)

Two interactive HTML mockups accompany this LLD (open directly in a browser; shared `_tokens.css`):

| File | Covers |
|---|---|
| `mockup-ai-console.html` | Tier-1/2 console surfaces: Draft with AI (A1), QA copilot findings on review (A4), AI translation pre-fill (A3), Insights copilot + verbatim themes (B3/B4), AI heal proposal styling (B1) |
| `mockup-ai-assistant.html` | C1 in-app assistant on a banking page: grounded answer, citation chips, "Show me" flow-launch, AI labelling, escape hatch, struggle-detection offer (B2) |

## 9.1 Console mockup — normative behaviors

- Every AI affordance uses the ✨ prefix and appears **only when the tenant capability flag is on**; AI-assisted items carry an `AI-assisted` badge through review (checker always sees origin).
- **Draft with AI (A1):** paste area → "Generate draft" → step list preview where each step shows its `anchor_hint` in an amber "needs capture" chip; primary action hands off to the extension Capture walk. The draft cannot be submitted while any hint chip remains.
- **QA copilot (A4):** findings render as advisory chips grouped Deterministic / AI-suggested; each AI chip shows the rule it cites (tone guide §, banned phrase); no blocking control exists.
- **Translations (A3):** pre-filled rows appear in existing `machine` state with a per-row glossary indicator; `glossary_miss` rows render empty with a warning, never a wrong forced term.
- **Insights copilot (B3):** answer = numbers panel (verbatim from query result) + ≤3-sentence narration + "how I answered" disclosure naming the catalog query used; unmappable questions show the capability chips.
- **Verbatim themes (B4):** theme cards with count, trend arrow, two scrubbed representative quotes; a "view underlying responses" link goes to the raw (scrubbed) list — no hidden data.

## 9.2 Assistant mockup — normative behaviors

- Panel opens from the existing launcher position; header label **"AI · answers from your organisation's guides"** always visible; every answer footer: citation chips + 👍/👎 + "Contact support".
- Grounded answer renders as constrained AST (no markdown surprises); **"Show me"** button appears only when an `action.launch_flow` is returned and triggers the standard deep-link start — the mockup demonstrates the handoff moment (panel minimizes, walk-thru step 1 appears with spotlight).
- Ungrounded/off-topic path shown verbatim: "I don't have guidance on that in this application. [Contact support]" — no partial answer.
- Struggle offer (B2): a small, dismissible toast ("Having trouble with this form? See the walk-through") — rendered via the normal announcement widget, targeted by the `struggle` env var; dismiss = standard frequency capping.
- Latency affordance: typing indicator with 6-s expectation text; timeout ⇒ apology + retry chip (never a spinner without exit).
