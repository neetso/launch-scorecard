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

### (v1.1) Entity: `UpcomingEvent`
Forward-looking calendar of expected events. Serves two purposes: (1) operational trigger for ingestion readiness, (2) user-facing "what's coming" view across tracked companies. Deferred from v1 — not needed for the MVP scorecard, and pulls in calendar sourcing, date confidence, and a new UI surface. See `docs/internal-tools-spec.md` for the operational stub.

Fields (provisional):
- `upcoming_event_id` (string/uuid)
- `company` (enum)
- `expected_name` (string; e.g., "re:Invent 2026")
- `event_type` (enum; same as Event)
- `expected_date` (date, nullable)
- `date_confidence` (enum: CONFIRMED, EXPECTED, RUMORED)
- `source_urls` (array[string]; IR page, event page, etc.)
- `notes` (text, optional)
- `linked_event_id` (fk to Event, nullable; set once the event actually happens and an Event is created)
- `created_at`, `updated_at`

---

## Computed Fields

These fields are derived server-side from stored data. They are not stored directly on entities but are calculated on read. They drive KPI chips, table columns, sort orders, and the methodology popover. Definitions here are canonical — use these in customer-facing methodology documentation.

### `overdue_days` (per commitment)

How late a commitment is relative to its claimed target window.

**Inputs:** `target_window_end`, `first_ship_date`, `status`, current date

**Calculation:**
- If `target_window_end` is null → `null` (no target to measure against)
- If `status` is `GA` or `PARTIAL` and `first_ship_date` exists → `first_ship_date − target_window_end` in days
  - Positive = shipped late
  - Zero or negative = shipped on time or early
- If `status` is not terminal (`GA`, `PARTIAL`, `CANCELLED`, `REPLACED`) and `target_window_end < today` → `today − target_window_end` in days (accumulating overdue)
- If `target_window_end >= today` and not yet shipped → `null` (not yet overdue)

**Display:**
- Positive → `+Xd` red overdue chip
- Zero or negative (shipped) → `on time` green label
- Null → dash

**Note:** `DELAYED` commitments show overdue_days against the *original* target window. Resetting the clock on a slip would hide accountability.

---

### `days_to_ga` (per commitment)

Calendar days from announcement to first evidence of general availability.

**Inputs:** `event.start_date` (from parent event), `first_ship_date`, `status`

**Calculation:**
- If `status = GA` and `first_ship_date` exists → `first_ship_date − event.start_date` in days
- Otherwise → `null`

**Aggregation:**
- **Median time to GA** (per company or event): median of `days_to_ga` across all GA commitments in scope. Displayed on the company dashboard.
- Ignore null values (non-GA commitments) when computing the median.

**Why event date, not target_window_start:** This answers "how long between promise and delivery," which is the analyst-useful metric. Measuring from the claimed target window would answer "how accurate was the estimate" — a valid but secondary question that `overdue_days` already covers.

---

### `evidence_strength` (per commitment)

A label reflecting the quality of the best available evidence, based on the evidence priority rules.

**Inputs:** all `Evidence` records for this commitment, specifically their `evidence_type` values

**Calculation:** Take the highest-priority evidence type attached to the commitment and map it:

| Best evidence type present | Label |
|---|---|
| `RELEASE_NOTES`, `CHANGELOG`, or `DOCS` | **Strong** |
| `BLOG_OFFICIAL` or `ROADMAP` | **Medium** |
| `PARTNER_ANNOUNCEMENT` or `THIRD_PARTY_VERIFICATION` | **Weak** |
| No evidence records | **None** |

**Priority order** (matches Evidence Priority Rules section):
1. `RELEASE_NOTES` / `CHANGELOG` / `DOCS` (best)
2. `BLOG_OFFICIAL` / `ROADMAP`
3. `PARTNER_ANNOUNCEMENT`
4. `THIRD_PARTY_VERIFICATION`

If a commitment has both a `BLOG_OFFICIAL` and a `RELEASE_NOTES` evidence record, the strength is **Strong** (highest wins).

**Display:** Small label badge (Strong = green, Medium = blue, Weak = gray, None = faint/hidden).

---

### `ship_rate` (per event or company)

Percentage of commitments that have reached a shipped state.

**Inputs:** all commitments in scope (for an event or a company), their `status` values

**Calculation:**
```
ship_rate = count(status IN [GA, PARTIAL]) / count(status NOT IN [REPLACED, CANCELLED])
```

- **Numerator:** commitments with `status = GA` or `status = PARTIAL`
- **Denominator:** all commitments *excluding* those with `status = REPLACED` or `status = CANCELLED`

**Why exclude REPLACED and CANCELLED from the denominator:** A replaced commitment means something else took its place — not a broken promise. A cancelled commitment, while potentially negative, reflects transparency rather than silent abandonment. Penalizing companies for being explicit about cancellations would create perverse incentives. Both are excluded so the rate measures "of things still in play, what percentage shipped."

**Display:** percentage in KPI chip (e.g., "56% Shipped"). Shown on event scorecard and company dashboard. Also shown per-event in the company dashboard events list.

**Edge case:** If all commitments for an event are REPLACED or CANCELLED, the denominator is zero → display as "—" not "0%".

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
  - Late (overdue_days; displayed as "+Xd" chip when > 0, blank otherwise)
  - Evidence: "N sources" link (logged-in); count + evidence_strength label (logged-out, provides value before sign-up gate)
- Filters: category, status, product_area, tag
- Sort (dropdown, default "Most urgent"):
  - Most urgent: DELAYED first by overdue_days descending, then ANNOUNCED by proximity to target_window_end ascending, then shipped statuses last
  - Newest updates: updated_at descending
  - Most evidence: evidence count descending, then alphabetical by commitment title
- KPI chips: ship rate, delayed count (rough is OK)

### Page: Commitment Detail (citation target)
- Promise text + scope
- Timeline (StatusHistory)
- Evidence list (per item: type badge, URL, excerpt, published date, publisher, supported status)
- Copy evidence list button (copies bullet list: each item includes source type, URL, published date, excerpt)
- Copy citation button; canonical format:
  `{Company}, "{title}," Launch Scorecard, commitment ID {commitment_id}. Status: {status} (as of {status_date}). Accessed: {today}. URL: launchscorecard.dev/c/{commitment_id}`
- Notes + confidence

### Page: Company Dashboard
- Recent changes (StatusHistory feed)
  - "Since" period selector: 7d / 30d / All
  - Activity filter: upgrades only (e.g., Preview → GA), slips only, new commitments
  - Status-change CSV export (separate from commitments export)
- Unshipped commitments
  - Sort on unshipped table (same options as Event Scorecard)
- Delayed commitments
- Simple reliability stats (ship rate, median time-to-GA)

### Component: Methodology Popover

Accessible from all three main surfaces via an "(i) Methodology" trigger button. Heading: "How we track promises." Content is tailored per surface.

**Company Dashboard — full version:**
- Status definitions: GA (generally available — shipped to claimed scope), Partial (shipped but limited by region, tier, or platform), Beta (public beta / open preview), Preview (private / limited preview access), Delayed (explicitly pushed past target window), Announced (promised but no shipping evidence yet), Replaced (superseded by a different commitment), Cancelled (explicitly withdrawn; no longer planned)
- Evidence priority: 1) Release notes / changelogs, 2) Official docs, 3) Official blog with explicit availability, 4) Partner announcements, 5) Third-party verification (only when primary absent)
- Confidence: 0–1 scale; capped at 0.8 for PARTIAL; reduced when evidence is third-party only or rollout scope is ambiguous
- Footer: coverage scope statement + last-refreshed timestamp

**Event Scorecard — swaps evidence priority for evidence strength:**
- Status definitions (same 8 as above)
- Evidence strength: Strong (release notes + docs), Medium (official blog only), Weak (third-party / indirect)
- Confidence (same as above)
- Footer (same as above)

**Commitment Detail — compact; omits status definitions:**
- Evidence priority (same numbered list as Company Dashboard)
- Confidence (same as above, plus: "GA requires primary source evidence")
- Footer (same as above)

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

**Core principle:** auto-capture → LLM draft → human review → publish. Never publish without human review. See `docs/internal-tools-spec.md` for tool-level detail.

### **Step 1: Capture source material**

Source material arrives through different channels depending on the event type:

**Structured feeds (continuous monitoring)**
- RSS/webhook monitors on official release note feeds: AWS "What's New", Microsoft 365 Roadmap, Google Cloud release notes, OpenAI product releases blog
- These fire automatically when new content is published and feed into the `raw_release_items` pipeline

**Keynote/conference capture (event-driven)**
- **Livestream transcripts:** pull YouTube auto-generated captions or use speech-to-text on the livestream. Gives raw promise candidates within minutes of the event ending. Store with timestamps for later citation.
- **Official recap pages:** Microsoft "Book of News", AWS "Top announcements", Google I/O recap posts. These are structured and publish same-day — monitor the known URL patterns and trigger ingestion when live.
- **Press releases:** company newsrooms and PR wires (Business Wire, PR Newswire). Often go live during or right after the keynote.

**Earnings transcripts (quarterly)**
- Available within hours from IR pages. See "Earnings calls as an input type" section for the specific workflow.

**Manual input (fallback)**
- Operator pastes text or provides URL directly. Always available for sources that aren't covered by automated capture.

### **Step 2: Seed event + extract commitments (LLM-assisted)**

For each event:
1. Create the Event record (company, name, type, dates, source URLs)
2. Pull the source material text (HTML → clean text, or transcript)
3. Run the "split recap into atomic commitments" prompt (see Prompt 1 below) → structured JSON of announcements + commitments
4. Operator reviews extracted items: accept, edit, skip, or merge (see internal tools spec, Tool 2)
5. Accepted items create Announcement + Commitment + initial StatusHistory (ANNOUNCED) + source Evidence record

### **Step 3: Evidence matching (semi-automated)**

For each commitment, search that company's shipping sources (release notes, docs, changelogs, official blogs) for evidence of delivery.

1. Candidate matching: search shipping sources for references to the commitment's product area, feature name, or related keywords
2. LLM validation: for each candidate, run the evidence validation prompt (see Prompt 2 below) → decision, date, excerpt
3. Operator reviews matches: accept, edit, reject, or flag for later
4. Accepted matches create Evidence record + StatusHistory entry

### **Step 4: Weekly refresh**

Re-run evidence matching (Step 3) for all commitments where `status NOT IN [GA, CANCELLED, REPLACED]`.

This is where compounding happens — each week, more commitments move from ANNOUNCED to shipped states, and the evidence chain grows.

### **SLA targets**

| Event type          | Draft (LLM-extracted) | Published (human-reviewed) |
|---------------------|-----------------------|----------------------------|
| Major keynote       | Within 2h             | Within 24–48h              |
| Earnings call       | Within 4h             | Within 48h                 |
| Product launch post | Within 1h             | Within 24h                 |
| Weekly release notes| Auto-classified       | Same-day review            |

-----

## **Earnings calls as an input type**

The `EARNINGS` event type and `EARNINGS_IR` evidence type exist in the schema but need explicit workflow rules. Earnings calls matter for two reasons: companies make forward-looking product claims on calls, and they sometimes quietly confirm or walk back prior commitments.

### Which companies have earnings (Phase 1)
- **Microsoft** (quarterly; covers Azure, M365, Copilot)
- **Google / Alphabet** (quarterly; covers Cloud, Workspace, AI)
- **Amazon / AWS** (quarterly; covers AWS services)
- **OpenAI** — private, no earnings calls. Skip.

### When to create an EARNINGS event
Create an Event with `event_type: EARNINGS` for each quarterly earnings call of a tracked public company. Use a slug like `ms-earnings-2026-q1`.

These are lightweight events — they rarely produce more than a handful of commitments. The primary value is as an evidence source for existing commitments.

### Earnings as an announcement source
Extract forward-looking product claims from prepared remarks and Q&A. These become commitments only if they meet the same eligibility criteria as keynote announcements:
- New capability, availability, economics, distribution, or governance claim
- Not a restatement of something already tracked

Typical earnings commitment: "We expect [feature] to be generally available to all enterprise customers by end of Q3." This becomes a commitment with `availability_claim_raw` from the transcript and a `target_window_end`.

Most earnings calls produce 0–5 new commitments. Do not over-extract — earnings language is often vague and aspirational.

### Earnings as evidence for existing commitments
This is the higher-value use case. Scan the transcript for references to existing tracked commitments:
- CEO/CFO confirms shipping: "We launched X to all enterprise customers last quarter" → `EARNINGS_IR` evidence record supporting `GA` or `PARTIAL`
- Management acknowledges delay: "X is taking longer than expected, we now expect H2" → `EARNINGS_IR` evidence supporting `DELAYED`, update `target_window_end`
- Feature quietly dropped: no mention of a previously hyped commitment across multiple quarters → not evidence by itself, but flag for operator attention during weekly refresh

### Evidence rules for EARNINGS_IR
- `EARNINGS_IR` evidence **can** advance a commitment to `PARTIAL` or support a `DELAYED` status change
- `EARNINGS_IR` evidence **cannot** alone advance a commitment to `GA` — require at least one primary source (`RELEASE_NOTES`, `DOCS`, or `BLOG_OFFICIAL`) to confirm GA. Earnings claims are management statements, not shipping proof.
- When `EARNINGS_IR` is the only evidence for a status, cap `confidence` at 0.7

### Ingestion workflow for earnings
1. Obtain transcript (available within hours from IR pages, Seeking Alpha, etc.)
2. Run LLM extraction: identify (a) new forward-looking claims → candidate commitments, (b) references to existing tracked commitments → candidate evidence
3. Operator reviews both sets in the standard review queue (see internal tools spec, Tool 2)
4. Publish

**SLA:** Draft within 4h of transcript availability. Published (reviewed) within 48h.

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

- Compute fields server-side once (see **Computed Fields** section for canonical definitions):

  - overdue_days, days_to_ga, evidence_strength, ship_rate

- Make tables URL-driven:

  - /event/:id?status=...&category=...&q=...&sort=overdue
  - share links “just work”

  