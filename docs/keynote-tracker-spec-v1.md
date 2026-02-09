# Keynote Promise Tracker — Product & Data Spec (v1)

## Goal
Create a trustworthy tracker that answers, for major tech companies:
- What was announced (as atomic “commitments”)?
- Did it ship? When? In what scope (geo/tier/platform)?
- What evidence supports that status?
- (Later) Was it “successful” via transparent proxy signals?

This is an **evidence-led ledger**. The database and status history are the moat.

---

## Non-goals (v1)
- Perfect success scoring across all products (do later)
- Fully automated extraction without human review
- Exhaustive coverage of all minor features

---

## Coverage Phases
### Phase 1 (clean evidence sources)
- OpenAI, AWS, Microsoft, Google Cloud (optionally Google Workspace)

### Phase 2 (messier / mixed: hardware + platform)
- NVIDIA (GTC announcements, product availability, SDK/toolkit shipping)

### Phase 3 (messiest: consumer OS + social products)
- Apple (WWDC promises, staggered rollouts, region gating)
- Meta (Connect promises, fragmented product channels)

---

## Core Concepts & Glossary
- **Event**: a keynote / developer conference / launch moment (e.g., WWDC 2024 keynote).
- **Announcement**: a coarse “announced thing” grouped within an event (e.g., “Apple Intelligence”).
- **Commitment** (core unit): an atomic promise statement that can be judged shipped/not shipped.
- **Evidence**: a dated source proving shipping status (release notes, docs, roadmap, earnings, etc.).
- **Status History**: immutable ledger of status changes, each tied to evidence.

---

## Status Taxonomy (global)
Use a small cross-company state machine:

- `ANNOUNCED`
- `PREVIEW` (private/limited preview)
- `PUBLIC_BETA`
- `GA` (generally available / shipped)
- `PARTIAL` (shipped but limited by region/tier/platform)
- `DELAYED` (explicitly pushed out)
- `REPLACED` (superseded by another feature/product)
- `CANCELLED`

Notes:
- Prefer `PARTIAL` over overconfident `GA` when scope is limited.
- `REPLACED` is distinct from `CANCELLED` (something else took its place).

---

## Data Model (DB-agnostic schema)

### Entity: `Event`
Represents a launch moment (keynote, conference day, major livestream).

Fields:
- `event_id` (string, stable slug; e.g., `aws-reinvent-2025`)
- `company` (enum: APPLE, OPENAI, GOOGLE, META, MICROSOFT, NVIDIA, AWS)
- `event_name` (string)
- `event_type` (enum: KEYNOTE, DEV_CONF, PRODUCT_LAUNCH, EARNINGS, OTHER)
- `start_date` (date)
- `end_date` (date, nullable)
- `official_hub_urls` (array[string])
- `video_urls` (array[string], optional)
- `notes` (text, optional)
- `created_at`, `updated_at` (timestamps)

### Entity: `Announcement`
A coarse grouping within an event (usually a recap section).

Fields:
- `announcement_id` (string/uuid)
- `event_id` (fk)
- `title` (string)
- `category` (enum: MODEL, API, CLOUD_SERVICE, OS_FEATURE, HARDWARE, PLATFORM, PRICING, PARTNERSHIP, OTHER)
- `source_url` (string; official)
- `source_excerpt` (string <= 240 chars, optional)
- `created_at`, `updated_at`

### Entity: `Commitment`  ✅ core unit
Atomic promise statement.

Fields:
- `commitment_id` (string, **stable public slug**; never reused)
- `announcement_id` (fk)
- `promise_text` (string; one sentence)
- `product_area` (string; e.g. “AWS Bedrock”, “iOS Siri”, “OpenAI API”)
- `scope` (object/json):
  - `platform` (array[string])  // e.g. iOS, macOS, Azure, Web
  - `region` (array[string])    // e.g. US, EU, global
  - `tier` (array[string])      // e.g. Free, Pro, Enterprise
  - `audience` (array[string])  // e.g. devs, consumers, enterprises
  - `notes` (string)
- `availability_claim_raw` (string; exact language like “later this year”, “rolling out”)
- `target_window_start` (date, nullable)
- `target_window_end` (date, nullable)
- `status` (enum: see taxonomy)
- `first_ship_date` (date, nullable) // earliest evidence of availability
- `confidence` (float 0–1) // especially for Apple/Meta/NVIDIA messy evidence
- `tags` (array[string], optional) // e.g. “AI”, “privacy”, “on-device”
- `created_at`, `updated_at`

### Entity: `Evidence`
Dated proof that supports a status (or scope limitation).

Fields:
- `evidence_id` (string/uuid)
- `commitment_id` (fk)
- `evidence_type` (enum):
  - RELEASE_NOTES
  - CHANGELOG
  - DOCS
  - ROADMAP
  - BLOG_OFFICIAL
  - EARNINGS_IR
  - PARTNER_ANNOUNCEMENT
  - THIRD_PARTY_VERIFICATION
  - VIDEO_TRANSCRIPT
- `url` (string)
- `publisher` (string; domain/org)
- `published_at` (date)
- `excerpt` (string <= 240 chars)
- `supports_status` (enum: PREVIEW, PUBLIC_BETA, GA, PARTIAL, DELAYED, REPLACED, CANCELLED)
- `scope_observed` (object/json, optional) // what this evidence actually indicates
- `added_at` (timestamp)

### Entity: `StatusHistory`  ✅ defensibility ledger
Immutable record of status changes.

Fields:
- `history_id` (string/uuid)
- `commitment_id` (fk)
- `old_status` (enum, nullable for initial)
- `new_status` (enum)
- `changed_at` (timestamp)
- `evidence_id` (fk, required for non-initial transitions)
- `note` (string, optional)

### (Optional v1.1) Entity: `OutcomeSignals`
Keep optional until you have demand.

Fields:
- `signal_id`
- `commitment_id`
- `signal_type` (enum: PRICING, EARNINGS_MENTION, PARTNER_LAUNCH, USAGE_PROXY, COMPETITIVE_RESPONSE, CASE_STUDY)
- `url`
- `published_at`
- `excerpt`
- `weight` (int)
- `added_at`

---

## Public URLs (important for citations + SEO)
- Event: `/event/{event_id}`
- Company dashboard: `/company/{company}`
- Commitment detail: `/c/{commitment_id}`  (stable, never changes)
- Optional: `/tag/{tag}`

---

## Core User Workflows
### Analyst workflow (v1)
1) Open event scorecard → scan commitments → click evidence-backed detail page → cite.
2) Filter by status (“unshipped”, “delayed”) and export CSV.

### Operator workflow (internal/admin)
1) Create event → ingest official recap → review extracted announcements/commitments.
2) Run evidence matching → add evidence → write status changes into history.
3) Weekly refresh for all “not GA” items.

---

## UI Requirements (minimum defensible v1)

### Page: Event Scorecard
- Table columns:
  - Commitment
  - Availability claim (raw)
  - Target window
  - Status
  - First shipped date
  - Evidence count (with quick links)
- Filters: category, status, product_area, tag
- KPI chips: ship rate, delayed count (rough is OK)

### Page: Commitment Detail (citation target)
- Promise text + scope
- Timeline (StatusHistory)
- Evidence list (dated, with excerpt)
- Notes + confidence

### Page: Company Dashboard
- Recent changes (StatusHistory feed)
- Unshipped commitments
- Delayed commitments
- Simple reliability stats (ship rate, median time-to-GA)

---

## Source Map (by Phase)
Goal: prefer primary, structured sources; avoid scraping social unless for outcome signals.

### Phase 1 — OpenAI, AWS, Microsoft, Google Cloud (+ Workspace optional)

**OpenAI**
- Announcements:
  - Product releases hub
  - Keynote/event posts (DevDay-style)
- Shipping evidence:
  - ChatGPT release notes
  - API/model release notes / docs changelogs where available
- Notes:
  - Keep scope fields for plan/tier/regional availability.

**AWS**
- Announcements:
  - re:Invent “Top announcements” posts
  - What’s New at AWS feed (also acts as announcement stream)
- Shipping evidence:
  - What’s New at AWS + service-specific release notes
  - Official AWS blogs / docs release notes

**Microsoft**
- Announcements:
  - Build hub + “Book of News”
  - Ignite “Book of News” (for enterprise / M365)
- Shipping evidence:
  - Microsoft 365 Roadmap (status + timing)
  - Product release notes (Teams, Windows, Azure as needed)

**Google Cloud**
- Announcements:
  - Google I/O developer keynote recap
  - Google Cloud blog “Next” / major launch recaps (if you include them)
- Shipping evidence:
  - Google Cloud release notes (product-specific)
  - Workspace admin updates table (optional coverage)

### Phase 2 — NVIDIA (mixed hardware + software)
**Announcements**
- GTC keynote recap pages / live updates
- Official press releases / launch posts

**Shipping evidence**
- Product availability pages (availability + SKU/region)
- CUDA / NGC / toolkit release notes
- Partner announcements for major systems (as PARTNER_ANNOUNCEMENT evidence)

### Phase 3 — Apple + Meta (messiest)
**Apple announcements**
- WWDC keynote pages + press releases
- Session pages for deeper feature details

**Apple shipping evidence**
- OS release notes (iOS/macOS/iPadOS)
- Apple Developer “Releases” (Xcode/OS releases)
- Support docs where features are described as available

**Meta announcements**
- Meta Connect recap hubs
- Meta newsroom posts

**Meta shipping evidence**
- Product-specific blogs/changelogs:
  - Quest / Horizon / developer platform updates
  - WhatsApp / Instagram / Facebook feature posts
- Note: expect lots of “rolling out” ambiguity → `PARTIAL` + lower confidence until verified.

---

## Messy Data Strategy (Phase 2 & 3)

### Confidence + Partial Shipping as first-class concepts
- Always store `scope_observed` in Evidence.
- If evidence indicates limited rollout, set status `PARTIAL` and keep `confidence` < 1.

### Canonical “Promise Extraction” inputs
For Apple/Meta especially, do NOT rely on a single recap page.
Use multiple official inputs:
1) Keynote transcript (or captions) → promise candidates
2) Press release / “All the new things” page → structured list
3) Feature pages / session pages → scope details (“US only”, “requires iPhone 15 Pro”, etc.)

### Apple WWDC video transcriptions approach (recommended)
Yes—use WWDC keynote transcript/captions as a promise source:
- Extract timestamped candidate promises as `Evidence` with type `VIDEO_TRANSCRIPT`.
- Then **require** a second evidence source (release notes/docs) to mark `GA`.
- Use transcript evidence to:
  - anchor “what exactly was promised”
  - link to precise timestamp for analysts

Implementation idea:
- Store `video_timestamp_start` / `video_timestamp_end` inside Evidence (optional fields)
- Keep transcript chunks small and referenceable.

### NVIDIA hardware availability
- Treat “announced” vs “available” carefully:
  - “Sampling now” vs “shipping” vs “general availability”
- Use `scope` for:
  - market segment (data center vs consumer)
  - partner availability vs direct

### Meta rollout ambiguity
- Many features ship progressively with limited official precision.
- Use:
  - official help center / changelog posts when they explicitly say “available now”
  - otherwise keep `PARTIAL` + lower confidence
- Add `tier` scope for “test group / limited users” if mentioned.

---

## Evidence Priority Rules
When deciding status:
1) Release notes / changelogs / roadmaps (best)
2) Official docs that describe feature as available
3) Official blog posts with explicit availability
4) Partner announcements (if official)
5) Credible third-party verification (only when primary sources absent)

Never upgrade to `GA` without at least one primary source (1–3).

---

## Backfill Plan — First Events to Seed (high ROI)
Start with the cleanest sources + biggest analyst interest.

### Phase 1 backfill (suggested first 10)
**AWS**
1. `aws-reinvent-2025`
2. `aws-reinvent-2024`

**Microsoft**
3. `ms-build-2025`
4. `ms-build-2024`
5. `ms-ignite-2024` (enterprise-heavy, good roadmap linkage)

**Google**
6. `google-io-2025`
7. `google-io-2024`
8. `google-cloud-next-2025` (optional if you decide “Cloud Next” is in-scope)

**OpenAI** (candidate for first end-2-end release, see appendix)

9. `openai-product-updates-2025` (as an “event” bucket if there isn’t a single keynote)
10. `openai-dev-event-2024` (DevDay-style event recap post)

### Phase 2 backfill (starter)
11. `nvidia-gtc-2025`
12. `nvidia-gtc-2024`

### Phase 3 backfill (starter)
13. `apple-wwdc-2025`
14. `apple-wwdc-2024`
15. `meta-connect-2025`
16. `meta-connect-2024`

Notes:
- You can add more years later; the key is building a repeatable pipeline.
- For OpenAI, you may represent major livestreams + “product release waves” as Events.

---

## Operational Cadence
- Weekly: refresh evidence matching for all commitments not in `GA/CANCELLED/REPLACED`.
- During keynote weeks: run event ingestion same-day, publish scorecard quickly with `ANNOUNCED` status.

---

## Export & Integrations (v1)
- CSV export for event tables and filtered queries
- “Copy citation” button on commitment pages (stable URL + evidence list)

---

## Acceptance Criteria (v1)
- You can publish an event scorecard with 30–150 commitments.
- Each commitment has a stable URL and at least one official source linking back to the promise.
- Any non-ANNOUNCED status has at least one Evidence record + StatusHistory entry.
- Analysts can cite the commitment page as a source with dated proof.

-------

## **Ingestion workflow (minimal but scalable)**

### **Step 1: Seed events manually (10 mins/event)**

For each keynote: add event row + the official recap URL(s).

### **Step 2: Extract candidate commitments from recap page (LLM-assisted)**

- Pull the recap page text (HTML → clean text)
- LLM outputs **announcement objects** + split into **atomic commitments**

### **Step 3: Evidence matching (semi-automated)**

For each commitment, you search only “shipping sources” for that company (release notes / roadmap / docs).

- When you find evidence, you add evidence rows and write a status_history entry.

### **Step 4: Weekly refresh**

Re-run evidence matching for “not GA” items.

This is where compounding happens.

-----

## **Prompt examples for extraction and validation**

### **Prompt 1 — “Split recap into atomic commitments”**

Use on an official recap page (AWS “Top announcements”, Microsoft “Book of News”, Google I/O recap, OpenAI post).

```text
You are extracting atomic product commitments from a keynote recap.

Input: text of an official recap page.
Task:

1) Identify announcements (coarse groupings).
2) For each announcement, split into ATOMIC commitments (each must be a single promise that could be judged shipped/not shipped).
3) For each commitment, capture scope and timing language exactly.

Rules:

- If the recap uses vague language (“coming later”), keep it as availability_claim_raw and leave target_window dates null.
- If the promise is conditional or partial, reflect that in scope (platform/region/tier).
- Do not invent details. If unknown, set null/empty.

Output JSON:
{
  "announcements":[
    {
      "title":"",
      "category":"",
      "source_quote":"",
      "commitments":[
        {
          "promise_text":"",
          "product_area":"",
          "availability_claim_raw":"",
          "target_window_start": null,
          "target_window_end": null,
          "scope": {"platform":[], "region":[], "tier":[], "notes":""}
        }
      ]
    }
  ]
}
```

### **Prompt 2 — “Does this evidence prove shipping, and what status?”**

Use on a candidate evidence page (release notes, docs, roadmap entry).

```text
You are validating whether a piece of evidence supports a delivery status for a specific commitment.

Commitment:

- promise_text: "<...>"
- scope: <json>
- current_status: "<...>"

Evidence text:
<page text>

Task:

- Decide if the evidence supports one of:
  Preview, Beta, GA shipped, Partial shipped, Delayed, Replaced, Cancelled, or No change.
- Extract the best-possible shipped/changed date (published date or explicit rollout date).
- Provide a short excerpt that justifies the decision (<= 240 chars).
- If evidence conflicts with scope (e.g., US only), mark Partial shipped and explain.

Output JSON:
{
  "decision": "no_change|preview|beta|ga|partial|delayed|replaced|cancelled",
  "date": null,
  "excerpt": "",
  "reasoning_bullets": ["", ""]
}
```

---

## OpenAI MVP

### **Why OpenAI is the best “single-company MVP”**

- **Analyst attention is intense** (models, distribution, pricing, regulation, partnerships).
- **Shipping cadence is high**, so your tracker shows value quickly (status changes every week/month, not twice a year).
- **Evidence is relatively centralised** (product releases + release notes), so your “evidence chain” is easier than Apple.
- The “promise vs shipped” gap is real (announcements, previews, staged rollouts, tier gating), so the ledger is useful.

## CompanyRules: OpenAI (Ingestion + Eligibility Filter)

### Purpose
OpenAI ships via frequent “release waves” rather than a small number of annual keynotes. This rule-set ensures:
- we capture analyst-relevant commitments,
- we avoid drowning in release-note noise,
- we maintain strict evidence standards for status changes.

---

## 1) What counts as an Event for OpenAI

OpenAI “Events” can be any of:

### A) Named major launches (preferred)
Create an `Event` when there is an official post that bundles multiple launches or frames a major moment, e.g.:
- “DevDay YYYY”
- “Spring Update YYYY-MM”
- “New models and developer products …” style posts

**EventType:** `DEV_CONF` or `PRODUCT_LAUNCH`

### B) Release wave buckets (fallback)
If OpenAI shipping is continuous and not neatly bundled:
- Create quarterly buckets, e.g. `openai-release-wave-2026-q1`
- Or monthly buckets only if needed (avoid too granular)

**EventType:** `OTHER` (the "release wave" nature is captured in the event name, not a separate type)

**Rule:** If > 25 eligible items in a quarter, split monthly; otherwise keep quarterly.

---

## 2) Source map (OpenAI) — announcement vs shipping evidence

### Announcement anchors (primary)
- Official OpenAI “Product releases” hub and major launch posts
- Dedicated launch posts for major models, pricing, platform changes

### Shipping evidence (primary; allowed to advance status)
Allowed evidence types to move beyond `ANNOUNCED`:
1) `BLOG_OFFICIAL` (explicit “available now / rolling out”)
2) `DOCS` (API reference / product docs describing feature availability)
3) `CHANGELOG` / `RELEASE_NOTES` (ChatGPT release notes, model/API notes)

Disallowed for status upgrades:
- Social posts (X, etc.), Reddit, community chatter
- Secondary press *unless* used only as `THIRD_PARTY_VERIFICATION` when primary sources are missing, and confidence is reduced

---

## 3) Ingestion eligibility (OpenAI)
The tier system is an **ingestion gate**, not a stored field. Items are classified during ingestion to decide whether they become commitments. Everything that passes the gate is tracked equally — there is no `importance` field on Commitment.

### Track (promote to commitment)
Always track:
- New flagship model families / major variants
- Model/API deprecations with deadlines
- New foundational API primitives (e.g. Responses/Agents/tool execution primitives)
- Fine-tuning availability expansions or major changes
- Pricing changes, rate limit/usage limit changes, new plan tiers
- Enterprise/Teams/Edu availability changes or major admin/compliance features
- Marketplace/distribution primitives (store/monetization)
- Major safety/policy changes that gate usage materially

Track when broadly relevant or rollout-complex:
- Big ChatGPT workflow features (Projects/workspaces, Memory changes, connectors)
- Major desktop/mobile app milestones
- Large "quality of life" features that are headline items
- Limited preview → public beta → GA journeys

### Skip (do not promote to commitments)
- Minor UI tweaks, bugfixes, wording changes
- Small behavior changes not tied to named model releases
- Routine safety copy updates without gating changes

---

## 4) Commitment eligibility test (OpenAI)
A candidate item becomes a `Commitment` if it changes at least one:

- **Capability:** new model or material capability unlock (multimodal, reasoning, tools)
- **Availability:** new tier/region/platform or major rollout stage
- **Economics:** price/limits/bundling
- **Distribution:** new channel, enterprise controls, marketplace primitive
- **Governance:** policy/safety changes that affect what’s possible to build/use

If it does not pass, keep as raw note only.

---

## 5) Granularity rules (avoid over-splitting)
Split a single announcement into multiple commitments only if parts can ship independently:
- tier-based splits (Free vs Pro vs Enterprise)
- platform splits (web vs iOS vs API)
- region/language splits
- component splits (model launch vs API support vs UI surface)

Keep together when:
- one endpoint/SKU ships as one unit
- announcement clearly indicates a single GA moment

---

## 6) Evidence rules for status changes (OpenAI)
To set a status beyond `ANNOUNCED`, require at least one Evidence record from:
- `BLOG_OFFICIAL`, `DOCS`, or `RELEASE_NOTES`

To set `GA` specifically:
- require at least one primary evidence item that explicitly indicates availability (or docs that specify the feature as available), **not** implied availability.

If evidence indicates only limited rollout:
- set status `PARTIAL`
- store `scope_observed`
- set `confidence <= 0.8` until confirmed broader.

---

## 7) Weekly selection workflow (release notes without noise)
We ingest release notes into `raw_items` (internal table) but only promote eligible items to commitments.

Workflow:
1) Scrape/collect release notes into `raw_release_items` (date + text + url)
2) Classify each raw item as **track** or **skip** using the eligibility rubric (see below)
3) For items that pass:
   - create or match an existing Announcement + Commitment
   - add Evidence
   - update StatusHistory if status changed

---

## 8) OpenAI rubric: auto-classify eligibility (heuristic)
Assign points per raw item:

+3:
- new model family / major model release
- pricing / limits change
- new foundational API primitive
- deprecation / retirement deadline

+2:
- enterprise/admin/compliance feature
- multimodal I/O addition
- marketplace/distribution primitive
- major staged rollout ("now for Pro, later for Free")

+1:
- widely impactful ChatGPT workflow/UI change
- meaningful integration/connectors expansion

0:
- minor UI/bugfix, copy change, small tweak

Map:
- score ≥ 2 → **track** (promote to commitment)
- score ≤ 1 → **skip** (keep as raw note only)

---

## 9) Canonical OpenAI tags (recommended)
Add 1–3 tags to each commitment from:
- `model_launch`
- `model_deprecation`
- `pricing_limits`
- `api_primitive`
- `tools_agents`
- `multimodal`
- `enterprise_admin`
- `marketplace_distribution`
- `safety_policy`
- `memory_personalization`
- `connectors_integrations`
- `apps_desktop_mobile`

---

## 10) Suggested configuration JSON (optional for ingestion engine)

> Use this config to drive scrapers/classifiers. Keep it in-repo as `config/company_rules/openai.json`.

```json
{
  "company": "OPENAI",
  "event_strategy": {
    "preferred": ["named_major_launch_post", "dev_event"],
    "fallback": {"bucket": "quarterly", "split_if_eligible_over": 25}
  },
  "sources": {
    "announcement_primary": [
      {"type": "BLOG_OFFICIAL", "name": "Product releases hub"},
      {"type": "BLOG_OFFICIAL", "name": "Major launch posts"}
    ],
    "shipping_primary": [
      {"type": "RELEASE_NOTES", "name": "ChatGPT release notes"},
      {"type": "DOCS", "name": "API/model documentation"},
      {"type": "BLOG_OFFICIAL", "name": "Launch availability posts"}
    ],
    "shipping_disallowed": ["SOCIAL", "REDDIT"]
  },
  "status_rules": {
    "ga_requires_primary": true,
    "partial_if_limited_rollout": true,
    "default_confidence": 0.9,
    "partial_confidence_cap": 0.8
  },
  "ingestion_eligibility": {
    "track_min_score": 2,
    "signals": {
      "new_model_family": 3,
      "pricing_or_limits": 3,
      "new_foundational_api": 3,
      "deprecation_deadline": 3,
      "enterprise_admin": 2,
      "multimodal_io": 2,
      "marketplace_distribution": 2,
      "staged_rollout": 2,
      "major_workflow_ui": 1,
      "connectors_integrations": 1
    }
  },
  "tags": [
    "model_launch",
    "model_deprecation",
    "pricing_limits",
    "api_primitive",
    "tools_agents",
    "multimodal",
    "enterprise_admin",
    "marketplace_distribution",
    "safety_policy",
    "memory_personalization",
    "connectors_integrations",
    "apps_desktop_mobile"
  ]
}
```

------

## **11) Done criteria (OpenAI MVP)**

You have an end-to-end OpenAI tracker when:

- At least 2 Events (e.g., 1 named launch + 1 release-wave bucket) are ingested.

- ≥ 30 commitments exist with stable URLs.

- Every non-ANNOUNCED status has evidence + a StatusHistory row.

- The company dashboard shows:

  - unshipped list
  - recently changed statuses
  - (simple) time-to-GA stats


---

Implementation notes for UI:

- Build components around your schema:

  - StatusPill, ConfidenceBar, EvidenceBadges, ScopeChips, OverdueChip

- Compute fields server-side once:

  - overdue_days, days_to_ga, evidence_strength, ship_rate

- Make tables URL-driven:

  - /event/:id?status=...&category=...&q=...&sort=overdue
  - share links “just work”

  