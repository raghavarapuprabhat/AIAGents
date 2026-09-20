# WalkMe vs Compass — Feature Comparison Matrix

**Legend:** ✅ full support · ⚠️ partial / with caveats · ❌ not available
Sources for WalkMe: WalkMe Help Center, product pages, and third-party analyst/reseller data (2026). Compass: our HLD/LLD as built and specified.

---

# PART A — CORE (NON-AI) FEATURES

## A1. Guidance content types

| Feature | WalkMe | Compass |
|---|---|---|
| Guided walk-throughs | ✅ Smart Walk-Thrus (flow-chart authoring, branching) | ✅ Flows (branching, auto-advance, cross-page resume) |
| Contextual tooltips | ✅ SmartTips (guidance + validation, in Sets) | ✅ Tips (hover/focus/always-on + form validation) |
| On-demand help buttons | ✅ Launchers (visible or invisible) | ✅ Launchers (overlay **or inline injected into host DOM**) |
| Announcements / pop-ups | ✅ ShoutOuts (scheduled, segmented, goal-tracked) | ✅ Announcements (modal/banner/toast, scheduled) |
| Surveys / NPS | ✅ Surveys (customizable types, frequency) | ✅ Surveys (NPS, scale, single/multi, text) |
| Navigation shortcuts | ✅ Shuttles + Permalinks | ✅ Deep Links (`?compass_flow=` start) |
| Help menu / resource panel | ✅ Player Menu + Resources (articles, videos) | ✅ Assist panel (search over published guides) |
| Task lists / onboarding checklists | ✅ Onboarding Tasks with goal completion | ❌ Not in scope (composable from Flows + Launchers) |
| Conversational task automation | ✅ ActionBot (form-filling, API-free automation) | ❌ Out of scope (deliberate — see C4 in AI doc) |
| Spotlight / dim backdrop | ✅ Balloon overlay | ✅ Per-step: none / dim+spotlight / block-interaction |
| Element usage discovery | ✅ Discovery (employee config only) | ❌ Not in scope |

## A2. Authoring experience

| Feature | WalkMe | Compass |
|---|---|---|
| Authoring tool | ⚠️ **Desktop app** (Editor) paired to browser via extension | ✅ **Browser extension** (MV3) — no pairing bridge, no desktop install |
| No-code authoring | ⚠️ Marketed as no-code; reviewers widely report jQuery/CSS/JS skill needed | ✅ No code path exists — declarative JSON only |
| On-page WYSIWYG editing | ✅ Edit on Page | ✅ Edit in the **production renderer** (preview mode = pixel parity) |
| Element picker | ✅ Picker + manual jQuery fallback | ✅ Picker with **live anchor-strength score (0–1)** and remediation warnings |
| Flow map / visual sequencing | ✅ Flow-chart view | ✅ Flow map in console (cycle/orphan detection) |
| Step add / insert / reorder | ✅ | ✅ (explicit `goto` targets never silently re-linked) |
| Preview before publish | ✅ Preview mode | ✅ Preview + forced-RTL preview |
| Recording-based authoring | ❌ | ✅ Record journey → draft steps with anchors captured |

## A3. Element detection & maintenance

| Feature | WalkMe | Compass |
|---|---|---|
| Detection technology | ✅ DeepUI (patented recognition) + **jQuery 1.8.3** selector fallback | ✅ Multi-signal fingerprints (data-guide, id, ARIA role+name, text, stable classes, structural path, geometry) |
| Confidence scoring | ❌ Not exposed | ✅ Weighted score; accept ≥0.85 w/ 0.10 margin, degrade 0.60–0.84, skip <0.60 |
| Drift detection after UI change | ⚠️ DeepUI "self-adapts"; reviewers report content still breaks and needs ongoing maintenance | ✅ Drift telemetry per anchor vs 7-day baseline → `health=degraded` |
| Self-healing workflow | ❌ No heal queue / approval flow | ✅ Heal proposal queue with **one-click checker approval** (versioned like content) |
| Author-time fragility warning | ❌ | ✅ Picker warns before capture ("no data-guide — step would be skipped at runtime") |
| SPA support (React/Angular) | ✅ Continuous evaluation; dynamic elements a known pain point | ✅ History wrap + debounced MutationObserver; React re-render storm is a gallery release gate |
| iframes | ⚠️ Supported; **cross-domain requires snippet inside the iframe** | ⚠️ Same-origin frame paths supported; cross-origin needs agent in frame |
| Shadow DOM | ⚠️ Special internal API required | ✅ Renders in own closed shadow root; shadow hosts in test gallery |
| Ongoing maintenance effort | ⚠️ Commonly reported: dedicated staffer / CoE required | ✅ Designed to reduce to heal-queue approvals; `data-guide` convention in app-team DoD |

## A4. Targeting, localization & delivery

| Feature | WalkMe | Compass |
|---|---|---|
| Segmentation | ✅ Rule Engine (attributes, jQuery conditions, behavior segments) | ✅ Rule tree → compiled bytecode (512-op / 1 ms limits) |
| Where targeting is evaluated | ⚠️ Client-side, with cloud-dependent data | ✅ Client-side, **zero per-user server calls** |
| Multi-language | ✅ String-based; XLIFF/XLSX export-import; translation partners | ✅ Per-locale bundles; XLIFF round-trip; fallback chains |
| RTL / Arabic layout mirroring | ⚠️ **Undocumented** — strings can be entered, layout mirroring not evidenced | ✅ Full RTL: logical placements, mirrored arrows/progress, forced-RTL QA preview |
| Per-locale images | ❌ Not documented | ✅ Locale image variants with author warning |
| Content delivery | ✅ AWS S3 → **Akamai CDN** (vendor-controlled) | ✅ Bank-controlled edge + **in-country nodes for restricted markets** |
| Install method | ✅ Snippet **or** browser extension (not both) | ✅ Single snippet + SRI (extension only for authors) |
| Desktop app guidance | ✅ Workstation (Windows/macOS) | ❌ Web only (in scope) |
| Mobile | ⚠️ Separate WalkMe Mobile product / add-on | ❌ Out of scope (roadmap) |

## A5. Governance, security & operations

| Feature | WalkMe | Compass |
|---|---|---|
| Hosting model | ❌ **Cloud-only SaaS**; "self-hosted" option offloads content files only — control plane stays in WalkMe cloud | ✅ **100% on-premises**, control plane included |
| Data residency | ⚠️ US / EU data centers only (+SAP US/EU for AI) | ✅ Bank DC + in-country edge; no data leaves premises |
| Content integrity / signing | ❌ Not published | ✅ **Ed25519-signed immutable bundles** (Vault Transit), verified in-browser via WebCrypto before render |
| Code executing in host app | ⚠️ Injects JS/jQuery into host pages | ✅ **No authored code ever executes** — declarative JSON + closed bytecode; CSP-strict compatible |
| Maker-checker approval | ⚠️ Role separation (Content Creator vs Publisher); no enforced approval workflow | ✅ Enforced at **API and DB-constraint** level (maker ≠ checker) |
| Audit trail | ⚠️ Activity/version log per environment | ✅ **Append-only** audit (who/what/when/diff) on every content & admin action |
| Rollback | ⚠️ Republish / version selection | ✅ **Seconds** — manifest repoint to prior bundle hash, no rebuild |
| Emergency kill switch | ❌ Not published | ✅ Fleet-wide unload **within 60 seconds** |
| Failure behavior | ⚠️ Performance "may drop when many assets used"; vendor outage affects host UX | ✅ **Fail-open invariant** — any failure = no guidance, never a broken banking screen |
| Performance budget | ❌ No published budget | ✅ ≤80 KB gz agent, ≤50 ms boot p95, CI-enforced |
| Test rigor | ❌ Not published | ✅ 24-case hostile-page gallery as release gate (nested scrollers, iframes, RTL, forced-colors, chaos) |
| Certifications | ✅ SOC 2 Type II, ISO 27001/27017/27018/27701, CSA STAR, FedRAMP | ⚠️ Inherits bank's own controls; no separate vendor certification needed |
| Vendor dependency for fixes | ⚠️ External queue; enhancements on vendor roadmap | ✅ In-house backlog, bank-paced |

## A6. Analytics

| Feature | WalkMe | Compass |
|---|---|---|
| Guidance engagement metrics | ✅ Insights (engagement, completion) | ✅ Funnels, step drop-off, tip engagement, survey results |
| Full behavioral analytics | ✅ DXA — all clicks/inputs/page views | ⚠️ Guidance events + struggle signals only (by design) |
| Session replay | ✅ Session Playback — **captures DOM + page assets** to vendor cloud (1,000/mo default quota) | ❌ Deliberately not built (privacy posture) |
| Funnels | ✅ Funnels + Flow Analytics | ✅ Funnels with per-step drop-off |
| Guidance-health analytics | ❌ | ✅ Anchor drift/fail rate per app release |
| Data retention | ⚠️ **1 year** | ✅ 13 months raw, aggregates beyond; per-tenant configurable |
| Where analytics data lives | ❌ Vendor cloud (AWS) | ✅ Bank-hosted PostgreSQL (ClickHouse-ready) |
| PII capture | ⚠️ DXA/Playback capture on-screen content | ✅ **No field values, no DOM content**; per-tenant HMAC pseudonymous IDs |
| Export | ✅ S3/Azure/GCS/SFTP/webhooks; daily refresh 09:00 UTC | ✅ Direct SQL/API access — it's your database |

## A7. Commercial model

| Feature | WalkMe | Compass |
|---|---|---|
| Pricing basis | ⚠️ Per-user/seat, quote-only, tiered by users **and application count** | ✅ No licence — build + run cost only |
| Typical annual cost | ⚠️ Median ~$39K; average ~$79K; ranges to $400–800K for large rollouts | ✅ $0 licence (StraightBank currently ~$200K/yr) |
| Cost of adding an application | ⚠️ Tier upgrade / added cost per app (Standard ≈1–3 apps, Enterprise ≈4+) | ✅ **Zero marginal cost** per additional tenant |
| Cost of adding users | ⚠️ Scales per seat | ✅ Zero marginal cost |
| Premium modules | ⚠️ Insights +$10–50K, ActionBot +$15–40K, Mobile separate | ✅ All capabilities included |
| Annual uplift | ⚠️ Typically 5–10%/yr in contracts | ✅ None |
| Implementation cost | ⚠️ ~$30–150K one-time; ~4 months to implement, ~17 months to ROI | ✅ Internal build (already delivered through PoC) |
| Lock-in / exit | ⚠️ Proprietary content model; SAP contract terms | ✅ Bank owns code, content, and data outright |

---

# PART B — AI FEATURES

## B1. AI architecture & governance

| Feature | WalkMe | Compass |
|---|---|---|
| AI hosting | ❌ **Cloud-only** (WalkMe/SAP US/EU data centers) | ✅ **On-premises** open-weight models (vLLM/Ollama) |
| Bring-your-own model | ⚠️ **Azure OpenAI only** | ✅ Any local model; external provider optional, **off by default** |
| Model transparency | ⚠️ Default engines not publicly named | ✅ Models pinned by digest, vendored in bank artifact repo |
| Data leaving the bank | ⚠️ "Public AI" routes to external LLMs | ✅ **None** — model pods have zero egress (CI-verified) |
| AI on/off control | ⚠️ Package-level purchase | ✅ **Per tenant, per capability flag**, default OFF; flags off = byte-identical to non-AI build |
| Manual path when AI enabled | ⚠️ Not a stated design rule | ✅ **Mutual coexistence rule** — manual always primary, user chooses per action |
| Prompt/response audit | ⚠️ AI consumption dashboards | ✅ Every call logged: model, prompt hash, output hash, outcome, disposition |
| Guardrails | ⚠️ Not publicly detailed | ✅ PII shield → injection screen → topic lock → output filter → grounding check |
| Human oversight | ⚠️ Varies by feature | ✅ **Nothing AI-produced publishes without maker-checker** |
| AI pricing | ⚠️ Moves to consumption-based **SAP AI Units from Jan 2027** (rates unpublished) | ✅ No per-call cost — own GPU infrastructure |

## B2. AI for authoring (internal users)

| Feature | WalkMe | Compass |
|---|---|---|
| Generate guidance from a document | ❌ Not documented | ✅ A1 — SOP → draft flow with anchor hints, then capture walk |
| AI copy rewriting | ⚠️ Generative writing in WalkMeX (end-user focused) | ✅ A2 — rewrite concise/friendly/plain-language, output validated to rich-text AST |
| AI translation | ✅ Via AWS / SAP translation services | ✅ A3 — on-prem LLM with **enforced glossary**; misses left empty, never guessed |
| Pre-review QA checks | ❌ | ✅ A4 — tone-guide and banned-claim findings, advisory only (never blocks) |
| AI-created tooltips | ✅ AI SmartTips via UI Intelligence | ✅ A5 — draft from recorded journey with anchors attached |
| Output governance | ⚠️ Not stated | ✅ All output = drafts with `AI-assisted` badge into existing review queue |

## B3. AI for operations & insight

| Feature | WalkMe | Compass |
|---|---|---|
| Semantic anchor repair | ⚠️ DeepUI auto-adapts silently (no approval surface) | ✅ B1 — AI proposes repaired fingerprint + rationale into the heal queue; **≥70% top-1 benchmark**; never auto-applies |
| Struggle detection | ⚠️ Proactive AI recommendations (cloud) | ✅ B2 — **deterministic** on-device signals (rage clicks, repeat validation errors, idle); no model involved |
| Natural-language analytics | ⚠️ AI adoption/consumption dashboards | ✅ B3 — question → fixed query catalog → narration using **only result numbers** (zero-invention verified) |
| Survey verbatim analysis | ❌ Not documented | ✅ B4 — embedding clustering + theme naming + scrubbed representative quotes |
| Content gap discovery | ✅ Discovery (usage-based, employee config) | ✅ B5 — struggle/abandon hotspots vs content coverage, each suggestion traceable to its query |

## B4. AI for end users

| Feature | WalkMe | Compass |
|---|---|---|
| In-app AI assistant | ✅ WalkMe X / Contextual AI Assistance (Pinned, Proactive, On-Demand AI, AI Chat) | ✅ C1 — grounded assistant (flag-gated per tenant) |
| Answer grounding | ⚠️ Company AI (RAG) **and** Public AI (external LLMs, open-ended) | ✅ **Grounded-only** — RAG over published guidance; citations mandatory; ungrounded ⇒ honest fallback + support link |
| Launch guidance from an answer | ✅ Text-to-action | ✅ "Show me" starts the matching walk-thru via the standard deep-link engine |
| Scope control | ⚠️ Public AI answers general queries | ✅ Topic lock — usage help only; no financial advice by construction |
| Coexistence with manual search | ⚠️ AI-forward experience | ✅ **Two tabs**: Search guides (always present) + Ask AI |
| Knowledge sources | ✅ AI Center — upload files/webpages → vector embeddings | ✅ Published guidance + approved KB, tenant-isolated, re-indexed per publish |
| Adaptive experience | ✅ Proactive next-best-action | ✅ C2 — proficiency-based depth variants (deterministic, author-defined) |
| Conversational task automation | ✅ ActionBot fills forms / executes tasks | ❌ **Deliberately excluded** — no AI acting inside banking transactions |
| Shadow-AI governance | ✅ WalkMe for Shadow AI (block/redirect unsanctioned AI use) | ❌ Out of scope (different problem domain) |

---

# PART C — SUMMARY SCORECARD

| Dimension | WalkMe | Compass | Winner |
|---|---|---|---|
| Breadth of content types | Broader (ActionBot, Tasks, Discovery, Desktop, Mobile) | Core 7 types, deep | **WalkMe** |
| Authoring ergonomics | Desktop editor + pairing; jQuery often needed | Browser-native, anchor scoring, recording | **Compass** |
| Anchor resilience & maintenance | DeepUI, but maintenance widely reported as heavy | Scored fingerprints + drift telemetry + heal queue | **Compass** |
| Multi-language / RTL | Strings yes; RTL undocumented | Full RTL + per-locale images | **Compass** |
| Analytics breadth | DXA + session replay (much deeper) | Guidance-focused, privacy-first | **WalkMe** |
| Data sovereignty & residency | US/EU cloud only | On-prem + in-country edge | **Compass** |
| Security architecture | Certified SaaS; injects JS; no content signing | Signed bundles, no code execution, fail-open, kill switch | **Compass** |
| Governance / auditability | Role-based | Enforced maker-checker, append-only audit, seconds rollback | **Compass** |
| AI breadth | Broader (ActionBot, Shadow AI, cross-app copilot) | Focused, grounded, governed | **WalkMe** |
| AI control & sovereignty | Cloud-only, Azure-OpenAI-only BYO | On-prem models, zero egress, per-capability flags | **Compass** |
| Cost at scale | Per-seat + per-app + modules + 5–10% uplift | Zero marginal licence | **Compass** |
| Time to value | ~4 months implementation, ~17 months ROI | PoC delivered; production hardening underway | **Mixed** |

**Honest summary:** WalkMe is broader — ActionBot, Discovery, desktop and mobile surfaces, and session-replay analytics are real capabilities Compass does not have and should not pretend to. Compass wins on every axis that made the bank leave: sovereignty, security architecture, governance, anchor maintainability, RTL/market coverage, and cost structure. The strategic choice is trading **feature breadth we don't use** for **control we can't buy**.
