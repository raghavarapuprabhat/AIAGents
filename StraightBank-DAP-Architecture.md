# StraightBank Digital Adoption Platform (DAP) — Solution Architecture

**Codename:** Compass (working name) · **Version:** 1.0 Draft for Architecture Review Board
**Author:** Enterprise Architecture · **Date:** July 2026
**Classification:** Internal
**Decision basis:** Option A — fully custom runtime with zero third-party code delivered to end-user browsers; standard open-source permitted for server-side infrastructure and build/test tooling only.

---

## 1. Executive summary

StraightBank currently licenses WalkMe (~USD 200,000/year for 60,000 users) for in-app guidance. Concerns around cloud-hosted delivery, vendor jurisdiction and Middle East market access restrictions, slow vendor turnaround, runtime coupling of a third-party script to the banking user experience, and per-user cost escalation across new internal platforms (HR, Operations) motivate an in-house replacement.

Compass is an on-premises, enterprise-shared Digital Adoption Platform that delivers functional parity with WalkMe — Smart Walk-Thrus, SmartTips, Launchers, ShoutOuts, Shuttles, Surveys, and multi-language content including RTL — across web applications, modern SPAs, and legacy portals. The runtime JavaScript agent, rendering engine, anchoring engine, and authoring studio are built entirely in-house with no third-party code shipped to the browser, giving the bank full control of its supply chain, patch cycle, and data residency. The platform is multi-tenant by design so HR, Operations, and future portals onboard with a single script tag and no per-user licensing.

## 2. Goals, scope and non-goals

### 2.1 Goals

The platform must reach capability parity with the WalkMe features in production use, run entirely inside the bank's data centers with no external network calls at runtime, serve all target countries including restricted Middle East markets, allow non-technical business authors to create and publish guidance without code, support at least 60,000 concurrent-eligible users at launch scaling to 150,000+, keep the incremental page-load cost on host applications under a strict performance budget, and provide first-party analytics on guidance engagement and funnel completion.

### 2.2 In scope

Web applications (server-rendered and SPA — React, Angular, and classic JSP/JSF legacy portals), iframe-embedded legacy screens, desktop-browser and mobile-browser rendering, English plus configured languages including Arabic RTL, and enterprise tenants (StraightBank retail/corporate web, HR portal, Operations portal, future onboarding).

### 2.3 Non-goals (phase 1)

Native mobile SDKs (iOS/Android), thick-client desktop applications, session-replay/heat-mapping, and AI-generated content authoring are explicitly deferred. The architecture leaves seams for all four (see §14 roadmap).

## 3. Architecture principles and key decisions

| # | Decision | Rationale |
|---|----------|-----------|
| AD-01 | Zero third-party code in the browser runtime | Eliminates supply-chain CVE exposure and license drift; the core reason for leaving WalkMe applies equally to OSS runtime libraries |
| AD-02 | Standard OSS permitted server-side and in tooling | PostgreSQL, ClickHouse, Nginx run inside the DC and never ship to users; rewriting them adds no security value |
| AD-03 | On-prem, active-active across two data centers | Sovereignty, restricted-market access, DR posture consistent with bank standards |
| AD-04 | Guidance content is data, not code | Guides compile to signed, versioned, immutable JSON bundles; the agent is a generic interpreter. Publishing content never redeploys software |
| AD-05 | Single lightweight agent, multi-tenant | One snippet serves every application; tenant resolved by API key + origin; enterprise reuse is native |
| AD-06 | Fail-open, never fail-closed | Any DAP outage degrades to "no guidance shown"; the host application is never blocked, delayed, or broken |
| AD-07 | No PII in the DAP | The agent receives opaque hashed user identifiers and coarse attributes (role, segment, language); analytics events carry no customer data and mask all input values |
| AD-08 | Maker-checker on all published content | Guidance renders inside banking screens; content changes get change-management rigor: four-eyes approval, audit trail, instant rollback |
| AD-09 | Automated hostile-page test harness is a release gate | The custom runtime's substitute for open-source community debugging; no agent release without a green cross-matrix run |

## 4. Logical architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│  HOST APPLICATIONS (StraightBank web · HR portal · Ops portal · ...)     │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  Compass Agent (custom JS, ~28 KB loader+core gzipped)             │  │
│  │  Loader → Core → [ Anchoring | Rendering | Guidance FSM |          │  │
│  │  Targeting evaluator | Surveys | i18n/RTL | Event buffer ]         │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└───────────────┬───────────────────────────────────────┬──────────────────┘
    fetch signed bundles (HTTPS, in-region)             │ batched analytics
                ▼                                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  EDGE TIER — on-prem Nginx reverse proxy + cache (both DCs)              │
└───────────────┬───────────────────────────────────────┬──────────────────┘
                ▼                                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  PLATFORM CORE (Kubernetes / OpenShift, active-active)                   │
│  Delivery Service · Content Service · Targeting Service ·                │
│  Analytics Collector · Identity Adapter · Admin Service                  │
└──────┬───────────────┬──────────────────┬────────────────────────────────┘
       ▼               ▼                  ▼
  PostgreSQL      Object store       Spool → ClickHouse → Superset/Grafana
  (content,       (published         (durable buffer,     (dashboards)
   config, audit)  bundles, media)     batched inserts)
                ▲
┌───────────────┴──────────────────────────────────────────────────────────┐
│  AUTHORING & GOVERNANCE                                                   │
│  Authoring Studio (browser extension + web console) · Translation        │
│  workbench · Maker-checker workflow · Publishing pipeline · RBAC/audit   │
└──────────────────────────────────────────────────────────────────────────┘
```

### 4.1 Component inventory

| Component | Type | Responsibility |
|---|---|---|
| Compass Agent | Custom JS (browser) | Loads guidance bundles, anchors to DOM elements, renders all widget types, runs walk-thru state machine, evaluates targeting rules client-side, buffers analytics |
| Delivery Service | Stateless service | Serves versioned bundle manifests and immutable content bundles; edge-cached |
| Content Service | Stateful service | CRUD for guides/steps/segments/translations; versioning; maker-checker workflow; publish pipeline |
| Targeting Service | Stateless service | Compiles segment definitions into rule bytecode embedded in bundles; server-side evaluation API for sensitive rules |
| Identity Adapter | Stateless service | Validates host-app-issued context tokens (OIDC/SAML claims → hashed DAP context) |
| Analytics Collector | Stateless service | Ingests batched agent events, validates schema, strips/rejects anything resembling PII, appends to a local durable disk spool; a background writer batch-inserts into ClickHouse |
| Analytics Store & BI | ClickHouse + Superset/Grafana | Funnels, completion rates, drop-off, survey results, per-tenant dashboards |
| Authoring Studio | Custom browser extension + web console | Visual element picker, WYSIWYG step editor, live preview on the real application, flow designer |
| Admin Service | Service + console | Tenants, environments, API keys, RBAC, audit log query, kill switches |

## 5. The Compass Agent (browser runtime)

The agent is the highest-risk, highest-value component. It is written in TypeScript, compiled with zero runtime dependencies, and delivered as two parts.

### 5.1 Loader and boot sequence

Host applications embed a single snippet: a <2 KB inline loader referencing the versioned agent bundle from the bank's edge tier with `async defer` and a Subresource Integrity (SRI) hash. Boot sequence: loader reads its `data-tenant` and `data-env` attributes → fetches the bundle **manifest** (a tiny JSON naming the current published content-bundle version for this tenant/environment/language) → fetches the immutable content bundle (cache-forever, content-addressed filename) → initializes the core. Budgets: loader + core ≤ 30 KB gzipped; content bundle ≤ 50 KB per application per language; zero synchronous work on the host's critical rendering path; initialization deferred to `requestIdleCallback` or first user interaction. If any fetch fails or exceeds its 800 ms budget, the agent silently no-ops (AD-06).

### 5.2 UI isolation

All Compass UI renders inside a single custom element (`<compass-root>`) using **Shadow DOM in closed mode** with constructable stylesheets. The host application's CSS can never break guidance rendering and Compass styles can never leak into banking screens — critical when one agent must work across a 2005-era JSP portal and a 2025 React app. Overlays use the native `popover` top-layer API where available, with a computed max z-index fallback for legacy browsers.

### 5.3 Rendering engine (custom positioning)

A from-scratch positioning module. Contract: given an anchor element and a preferred placement, compute a collision-free position. The algorithm measures the anchor via `getBoundingClientRect`, walks the ancestor chain to identify every scroll container and clipping boundary (`overflow`, `contain`, CSS transforms that change containing blocks, iframe boundaries), attempts the preferred placement, and on collision applies flip → shift → resize in that order. It subscribes to `ResizeObserver`, `IntersectionObserver`, passive frame-throttled scroll, and orientation change to keep positions live. RTL is a first-class input: placements are expressed logically (`inline-start`/`inline-end`) and resolved against document direction, so Arabic layouts mirror automatically. This module is the most heavily tested unit in the codebase (§12).

### 5.4 Anchoring engine (self-healing selectors)

Brittle CSS selectors are the classic failure mode of guidance tools. Compass anchors elements with a **multi-signal fingerprint** captured at authoring time: id/name/data-* attributes, ARIA role and accessible name, tag plus a stable class subset (auto-filtering framework-generated hashed classes like `css-1x2y3z`), trimmed text content, label association, structural path with positional hints, and normalized geometry (relative position and size band). At runtime the engine scores candidates against the fingerprint with weighted matching. Above the confidence threshold (default 0.85) the widget anchors; a near-miss (0.6–0.85) anchors but emits a `low_confidence_anchor` analytics event so authors are alerted before the guide visibly breaks; below threshold the step is skipped or the flow branches to its authored fallback. Fingerprints are re-baselined from the Studio in one click when a host UI changes. A scoped, debounced `MutationObserver` re-resolves anchors when SPAs re-render, and the engine descends into same-origin iframes to support legacy portal frames. Host teams are additionally encouraged to adopt a `data-guide="..."` attribute convention on key elements — the cheapest single action that hardens anchoring program-wide.

### 5.5 Guidance engine (walk-thru state machine)

Each Smart Walk-Thru compiles to a deterministic finite state machine: states are steps; transitions fire on user actions (click on anchor, input change, custom DOM events), on element appearance/disappearance, on page/route change, or on explicit next/back. The engine supports branching (conditional transitions on element state, user attributes, or survey answers), auto-advance steps, and "wait for" steps with timeout fallbacks. Cross-page flows persist in-flight state (flow id, current step, variables) in `sessionStorage` under a namespaced key and resume after full page navigations — essential for legacy multi-page portals. SPA route changes are detected by wrapping `history.pushState`/`replaceState` and listening to `popstate`/`hashchange`, re-evaluating page-level targeting on every virtual navigation.

### 5.6 Widget catalogue (WalkMe parity map)

| WalkMe capability | Compass widget | Notes |
|---|---|---|
| Smart Walk-Thrus | Flows | FSM with branching, auto-advance, cross-page persistence |
| SmartTips | Tips | Hover/focus tooltips and inline validation hints bound to anchors |
| Launchers | Launchers | Authored buttons/icons injected beside anchors; trigger flows or links |
| ShoutOuts | Announcements | Modal/slide-in broadcasts with scheduling, frequency capping, dismissal memory |
| Shuttles | Deep links | URL entry points (`?compass_flow=...`) that start a flow from anywhere (intranet, email) |
| Surveys | Surveys | NPS, CSAT, single/multi-choice, free text (client-side PII pattern blocking); step- or event-triggered |
| Menu / Help widget | Assist panel | Persistent help launcher listing available flows and resources, searchable |
| Multi-language | i18n bundles | Per-language content bundles; language resolved from host locale → user profile → browser |

### 5.7 Targeting and segmentation

Segments are authored as rule trees over: user attributes supplied by the host (role, department, tenure band, language, country — passed as a signed context token, §8.2), environment (URL patterns, route, element presence/state), behavior (first visit, flow completed/dismissed before, N-th session), and schedule windows. Rules compile server-side into a compact JSON rule bytecode shipped inside the bundle; the agent evaluates it locally in microseconds with no network round-trip. Rules classed as sensitive (entitlement-dependent visibility) can be flagged for server-side evaluation via the Targeting Service instead, trading a small latency cost for non-disclosure of the rule to the client.

## 6. Authoring Studio

The Studio is what makes this a WalkMe replacement rather than a widget library, and it is fully custom.

**Composition.** A bank-signed browser extension (Chromium; distributed via enterprise policy) plus a web console. The extension injects the authoring overlay into any allow-listed application in a dedicated design mode: authors navigate the real application, click "capture element," and the extension records the multi-signal fingerprint of §5.4 with a live confidence preview showing how uniquely the element resolves. The web console hosts the flow designer (drag-drop step canvas with branch visualization), the WYSIWYG step editor (rich text against a strict allow-list schema — no raw HTML from authors, §8.2), theming (per-tenant design tokens so guidance matches each portal's brand), the translation workbench (side-by-side source/target editing, XLIFF export/import for the bank's translation-vendor workflow, per-language completeness tracking, RTL preview), and a simulation mode that runs any draft flow against the live application without publishing.

**Content lifecycle.** Draft → peer review (maker-checker: an author cannot approve own content) → publish per environment (dev → UAT → production, mirroring host-app environments) → monitor (anchor-health and engagement dashboards) → rollback (one click restores the previous immutable bundle version, effective within the ≤60-second edge-cache TTL). Every transition lands in an append-only audit log with actor, timestamp, diff, and approval evidence.

## 7. Data architecture

### 7.1 Core content model (PostgreSQL)

| Entity | Key fields | Notes |
|---|---|---|
| tenant | id, name, allowed_origins, api_key_hash, theme_tokens | One per application/portal |
| environment | tenant_id, name (dev/uat/prod), manifest_pointer | Manifest pointer flips atomically on publish |
| guide | tenant_id, type (flow/tip/launcher/announcement/survey/shuttle), status, owner, tags | |
| guide_version | guide_id, semver, definition (JSONB), created_by, approved_by, approved_at | Immutable once approved |
| step | guide_version_id, seq, widget_config (JSONB), anchor_fingerprint (JSONB), transitions (JSONB) | |
| segment | tenant_id, rule_tree (JSONB), compiled_bytecode | Reusable across guides |
| translation | guide_version_id, locale, strings (JSONB), status, translator, reviewed_by | |
| publish_record | environment_id, bundle_hash, manifest_version, published_by, published_at, rollback_of | Append-only |
| audit_event | actor, action, entity, before/after diff, ts | Append-only, WORM-exported to bank SIEM |

### 7.2 Published bundle format

Publishing compiles the relational model into one immutable JSON bundle per tenant × environment × locale: `{schema_version, tenant, generated_at, guides[], segments_bytecode, theme_tokens, signature}`. The bundle is signed (detached signature with a platform key held in the bank HSM; the agent pins the public key and refuses unsigned or invalid bundles), content-addressed (`bundle-<sha256>.json`), stored in the on-prem object store (bank-standard S3-compatible, e.g. MinIO), and cached at the edge indefinitely. Only the tiny manifest (≤1 KB, 60-second TTL) changes on publish — which is what makes rollback instantaneous.

### 7.3 Analytics model (spool → ClickHouse)

**Ingestion durability without a message broker.** The bank does not operate Kafka, and at this platform's volume (~3M events/day, ≈35 events/s average) introducing a new distributed messaging system would add operational burden without benefit. Instead, each Collector pod appends validated event batches to a **local durable disk spool** (append-only, size-capped for ~48 hours of peak traffic on persistent volumes); a background writer drains the spool into ClickHouse using native batched async inserts every few seconds — the write pattern ClickHouse is optimized for. If ClickHouse is unavailable, events accumulate in the spool and drain on recovery; if a spool approaches capacity, oldest analytics events are dropped first (guidance delivery is never affected — analytics remains strictly non-critical-path). Alerting jobs (anchor-fail spikes, agent errors) run as scheduled ClickHouse queries every 1–5 minutes feeding Grafana alerts, replacing stream processing. The Collector's internal interface is broker-shaped, so if the event stream later becomes an enterprise-wide data source with many downstream consumers, the spool can be swapped for Kafka or the bank's standard MQ without changes to the agent, schema, or ClickHouse model.

The agent batches events (≤20 events or 10 seconds; `navigator.sendBeacon` on unload) to the Collector. Event schema: `{tenant, env, anon_user_hash, session_id, event_type, guide_id, step_id, locale, ts, meta}` with `event_type ∈ {impression, flow_start, step_view, step_complete, flow_complete, flow_abandon, tip_hover, launcher_click, announcement_view, announcement_dismiss, survey_response, low_confidence_anchor, anchor_fail, agent_error}`. The Collector enforces the schema, rejects free-text fields except survey answers (which pass a PII-pattern scrubber: account-number, national-ID, email, phone classes), and never records input values, keystrokes, or page content. ClickHouse materialized views precompute funnel and drop-off aggregates; Superset serves per-tenant dashboards; a scheduled job alerts when a guide's `low_confidence_anchor`/`anchor_fail` rate spikes — the early-warning system for host-app UI changes.

## 8. Security architecture

### 8.1 Threat model summary

Primary risks: **T1** — the agent as an XSS vector into banking screens via authored content; **T2** — tampering with guidance content in transit or at rest (a modified tooltip instructing users to enter credentials somewhere is a phishing primitive); **T3** — leakage of customer PII through analytics; **T4** — the Studio extension as a privileged foothold; **T5** — availability coupling to host applications.

### 8.2 Controls

**Content integrity (T2).** Bundles signed with an HSM-backed key and verified by the agent before interpretation; the agent script itself pinned by SRI hash in the host snippet; all delivery over internal TLS with the edge tier inside the security perimeter; publish gated by maker-checker plus RBAC.

**XSS prevention (T1).** Authors never write HTML. Step content is a constrained rich-text AST (bold, italics, links to allow-listed domains, images from the platform media store only) rendered by the agent through a strict serializer — there is no `innerHTML` of author input anywhere in the runtime, enforced by lint rules and the code-review checklist. The agent is compatible with strict host-app Content Security Policies: it needs only `script-src` for its own origin (with SRI), injects no inline scripts, uses constructable stylesheets instead of inline `<style>`, and requires `connect-src` to the delivery/collector endpoints — a documented 3-line CSP delta per host application.

**Identity and context (T1/T3).** The agent never authenticates users. The host application, already holding an SSO session (bank-standard SAML/OIDC via the enterprise IdP), obtains a short-lived **context token** from the Identity Adapter: a signed claim set containing `anon_user_hash` (HMAC of the enterprise user ID with a per-tenant key — irreversible by the DAP), role, department, country, and locale. The agent receives this token from the host page, uses it for targeting evaluation and analytics attribution, and forwards it on Collector calls. No passwords, session cookies, or PII cross the DAP boundary.

**PII minimization (T3).** No input-value capture, no DOM content in events, survey free-text scrubbing (§7.3), analytics retention of 13 months with shorter per-tenant windows configurable, and the anonymization HMAC key held by each tenant's application team — the DAP cannot re-identify users even if compromised.

**Studio hardening (T4).** Extension distributed only via enterprise browser policy to the `dap-authors` AD group; design mode functions only on allow-listed origins; all Studio APIs require SSO with step-up MFA for publish actions; extension code passes the same SDLC gates as the agent.

**Availability (T5).** AD-06 fail-open behavior throughout; per-tenant and global **kill switches** in the Admin console that flip the manifest to an empty bundle within the 60-second TTL; an agent circuit breaker suspends event submission with exponential backoff if the Collector degrades.

**SDLC.** Threat-modeled design docs, mandatory security review for agent releases, SAST and secrets scanning in CI, annual penetration test scoped to agent + Studio + APIs. Because the runtime carries zero third-party code, SCA applies only to server-side images and build tooling — all vendored through the bank's internal artifact registry with no public-registry pulls at build time.

## 9. Deployment architecture

### 9.1 Topology

Two on-prem data centers, active-active. Each DC runs: the edge tier (Nginx reverse proxy with proxy-cache for bundles, terminating internal TLS), the platform core on the bank's Kubernetes/OpenShift landing zone (Delivery, Content, Targeting, Identity Adapter, Collector, Admin — all stateless, HPA-scaled), PostgreSQL (primary in DC1 with synchronous standby in DC2 under the bank's standard DB service; content workload is low-GB scale), the object store replicated across DCs, and a ClickHouse replica pair spanning both DCs — each DC's Collectors spool locally on persistent volumes and batch-insert into their in-DC replica, so analytics ingestion continues through a full DC or database outage. The bank's standard GSLB fronts the edge tier. Users in restricted markets are served entirely from bank infrastructure with no third-country dependency — directly resolving the Middle East access constraint.

### 9.2 Sizing and performance targets

Sixty thousand users generate trivially small read traffic thanks to immutability: steady-state load is manifest checks (≤1 KB, 60 s cache) plus rare bundle fetches after publishes. Targets: manifest p99 < 50 ms from edge cache; bundle p99 < 150 ms; Collector sustained 2,000 events/s with 10,000/s burst (a Monday-morning announcement to all staff), backed by per-pod spools sized for ≥48 hours of peak volume; agent main-thread cost on host pages < 50 ms total during boot; memory < 10 MB. Scaling to 150,000 users requires no architectural change — only Collector/ClickHouse horizontal scale.

### 9.3 Environments and release management

Platform environments (dev/UAT/prod) are distinct clusters; content environments (§6) are logical within each. The agent is versioned independently of content: host apps pin a major agent version via the SRI-hashed snippet; minor/patch releases roll out by publishing a new agent artifact and communicating updated SRI hashes through standard change management (or via an optional bank-controlled signed-manifest auto-update channel where a host team delegates SRI management to the platform). Canary strategy: new agent versions ship first to the Ops portal tenant (internal users) for one week before bank-wide rollout.

## 10. Multi-language and RTL

Content strings are externalized per guide version into locale documents (§7.1); the publish pipeline emits one bundle per locale and refuses to publish a locale below a configurable completeness threshold (default 100% for production). Locale resolution order: explicit host signal (`data-locale` or API call) → context-token claim → `navigator.language` → tenant default. RTL support is structural, not cosmetic: logical placements in the rendering engine (§5.3), mirrored flow-progress indicators, direction-aware animations, and a mandatory Arabic page set in the hostile-page harness. Fonts inherit from the host page — no font shipping — keeping the agent small and visually native to each portal.

## 11. Enterprise multi-tenancy and onboarding

A new portal onboards in four steps. First, Admin creates the tenant with allowed origins, theme tokens, and an API key. Second, the host team adds the snippet and the 3-line CSP delta, and wires the context-token call into their existing SSO flow (reference implementations provided for the bank's standard Java/Spring and .NET stacks). Third, the portal's business team is granted `dap-authors` membership scoped to that tenant. Fourth, authors build in dev, promote through UAT, and publish to production. Isolation is enforced throughout: content, segments, analytics, and RBAC are partitioned per tenant; theme tokens preserve each portal's look; each tenant has its own kill switch. The marginal cost of an additional application is infrastructure-only — the direct structural answer to WalkMe's user-based pricing model.

## 12. Testing strategy — the hostile page gallery

This is the non-negotiable condition of Option A. A permanent Playwright-driven harness runs on every commit against a gallery of adversarial test applications: a React SPA with aggressive re-rendering and hashed CSS classes; an Angular app with router transitions; a 2005-style JSP multi-page portal with nested framesets and iframes; pages with nested scroll containers and CSS-transformed ancestors; shadow-DOM component libraries; an Arabic RTL variant of each; high-zoom, small-viewport, and forced-colors accessibility modes; and pathological cases (10,000-node DOMs, elements appearing after 5-second delays, anchors that detach and reattach). Each run executes the full widget catalogue and a battery of flow scenarios, asserting position correctness (screenshot plus geometry assertions), anchor resolution scores, FSM behavior across navigations, and performance budgets (a Lighthouse trace diff of host pages with and without the agent — any regression fails the build). Accessibility is asserted in the same harness: WCAG 2.2 AA, full keyboard operation, focus-trap correctness in modal widgets, `aria-live` announcements, and automated screen-reader smoke tests. Unit-coverage gates: ≥90% on the positioning, anchoring, and FSM modules.

## 13. Migration from WalkMe

**M1 — Inventory.** Export the WalkMe account (guides, segments, analytics baselines) while the license is active; auto-classify by usage from WalkMe analytics. Typically ~30% of authored content is dead and is dropped.

**M2 — Translation.** A one-time converter maps WalkMe's exported flow definitions onto the Compass content model where structures align: steps, tips, launchers, and announcements translate mechanically; complex Smart Walk-Thru branching is rebuilt by authors in the Studio, with the converter emitting a work-list. Anchors are deliberately not migrated — they are recaptured in the Studio, because recapture with the multi-signal fingerprint is a quality upgrade and takes seconds per element.

**M3 — Parallel run.** Compass live on one StraightBank journey with WalkMe still active elsewhere; compare completion-rate analytics against the WalkMe baseline for one month.

**M4 — Cutover.** Per-application cutover, WalkMe snippet removal, license non-renewal aligned to the contract anniversary. Hold a 3-month WalkMe overlap budget in the business case as risk contingency.

## 14. Delivery roadmap

| Phase | Duration | Outcome |
|---|---|---|
| 0 — Foundations | 6 weeks | Landing zone, CI/CD, hostile-page harness v1, agent loader + shadow-DOM shell, delivery service, signed-bundle pipeline |
| 1 — Core runtime | 10 weeks | Positioning and anchoring engines; Tips, Launchers, Announcements; Studio element capture and step editor; dev/UAT publishing |
| 2 — Flows & parity | 10 weeks | Walk-thru FSM with branching and cross-page persistence; Surveys, Shuttles, Assist panel; targeting engine; analytics dashboards; i18n/RTL |
| 3 — Hardening & pilot | 6 weeks | Penetration test, accessibility certification, performance sign-off, pilot on the Ops portal (internal users), WalkMe converter |
| 4 — Migration & scale | 8–12 weeks | StraightBank parallel run, per-app cutover, HR portal onboarding, WalkMe decommission |

Post-launch: a permanent product team of 2–3 engineers plus a product owner, carrying the roadmap seams — native mobile SDK, desktop thick-client injection, and assisted authoring.

## 15. Cost model (indicative, to be refined with Finance)

Run cost is dominated by the permanent team (2–3 engineers, ≈ USD 350–500K/year depending on location mix) plus marginal infrastructure on existing landing zones (≈ USD 30–50K/year). Against WalkMe's USD 200K/year for a single application scope — plus the avoided incremental licensing for the HR and Operations portals and per-user escalation as headcount grows — the enterprise-wide comparison favors Compass increasingly as tenants are added, while the strategic returns (sovereignty, restricted-market coverage, patch-cycle control, roadmap ownership) accrue regardless. Build cost is treated as accepted per the Option A decision.

## 16. Risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Custom positioning/anchoring misses edge cases OSS solved years ago | Broken guidance in production | Hostile-page harness as release gate (§12); low-confidence anchor telemetry (§5.4); canary rollout (§9.3) |
| Host-app UI changes silently break anchors | Guides fail without alarms | Anchor-health analytics with alerting; one-click re-baseline in Studio; `data-guide` attribute convention with app teams |
| Authored content abused for internal phishing | Security incident | Signed bundles, maker-checker, no-HTML AST content, audit trail, kill switch (§8) |
| Business authors under-adopt the Studio | Platform built, content stale | Author enablement program during the Phase 3 pilot; Studio usability testing as a first-class deliverable; per-tenant content KPIs |
| Key-person dependency on a small custom codebase | Maintenance risk | Permanently funded team, full ADR/design documentation, ≥90% coverage on core modules, engineering-guild review |
| Scope creep toward session replay / heavy analytics | Timeline and privacy risk | Non-goals fixed at ARB approval; any addition re-enters threat modeling |

## 17. Open items for the Architecture Review Board

Confirmation of the Kubernetes landing zone and GSLB pattern for active-active; HSM key-ceremony ownership for the bundle-signing key; whether the bank's standard IdP can mint the context-token audience directly (which would remove the Identity Adapter's minting path); ClickHouse versus the bank's incumbent analytics warehouse; and the WalkMe overlap window to negotiate with procurement.

---

*Companion artifacts to follow: ADR set, OpenAPI specifications for the platform services, JSON Schema for the bundle format, and the agent integration guide for host-application teams.*
