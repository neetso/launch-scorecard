# Spec v1 — Remaining Gaps

Tracking changes needed to bring the spec in line with the wireframe v3.1, wireframe feedback doc, and event calendar doc. Work through these across sessions; check off and commit as we go.

**Resolved in prior sessions:**
- [x] Importance tiers → ingestion gate only, no stored field
- [x] PRODUCT_RELEASE_WAVE → not a global event type, OpenAI uses OTHER
- [x] Computed fields → overdue_days, days_to_ga, evidence_strength, ship_rate formally defined
- [x] UpcomingEvent → deferred to v1.1 with provisional schema in spec + stub in `docs/internal-tools-spec.md`
- [x] Earnings as input type → added to spec with workflow, evidence rules, and confidence caps

**Source:** `docs/event_calendar-event_capture-earnings_input_type_automate_with_review.md`

The `EARNINGS` event_type exists in the enum but the spec never describes the workflow. The calendar doc recommends:
- Treat earnings transcripts as **announcement sources** (extract product/feature claims from prepared remarks and Q&A)
- Also use earnings transcripts as **evidence** for existing commitments (e.g., CEO confirms "we shipped X to all enterprise customers last quarter" → `EARNINGS_IR` evidence record)

**What to add:** A short section (similar to the OpenAI company rules) describing how earnings events are ingested, what gets extracted, and how claims map to commitments vs. evidence.

---

## 3. Ingestion automation and capture methods

**Source:** `docs/event_calendar-event_capture-earnings_input_type_automate_with_review.md`

The spec has a basic 4-step ingestion workflow. The calendar doc describes a richer pipeline:
- RSS/webhook monitors on official sources (AWS What's New, M365 Roadmap, etc.)
- Livestream transcript capture (YouTube auto-captions / speech-to-text)
- Official recap/Book of News page scraping (same-day)
- LLM-assisted extraction with different SLAs per event type:

| Event type          | Draft scorecard | Published (reviewed) |
|---------------------|-----------------|----------------------|
| Major keynote       | Within 2h       | Within 24–48h        |
| Earnings call       | Within 4h       | Within 48h           |
| Product launch post | Within 1h       | Within 24h           |
| Weekly release notes| Auto-classified | Same-day review      |

**What to add:** Expand the ingestion workflow section with capture methods and SLA targets. Keep the principle: "never publish without human review."

---

## 4. Methodology popover

**Source:** `docs/wireframe/wireframe-feedback-implemented.md` (P0), implemented in wireframe v3.1

The wireframe has a methodology popover on all three main surfaces (company dashboard, event scorecard, commitment detail). The spec's UI Requirements section doesn't mention it.

**What to add to spec UI section:**
- Methodology popover accessible from every main surface
- Contents: status definitions, evidence priority rules, confidence meaning + caps, coverage scope statement, last-refreshed timestamp
- The wireframe already implements this; spec just needs to codify it

---

## 5. Sort options

**Source:** `docs/wireframe/wireframe-feedback-implemented.md` (P0), wireframe v3.1

The spec mentions filters but not sort. The wireframe v3.1 has:
- "Most urgent" (default — likely overdue_days descending)
- "Newest updates"
- "Most evidence"

The feedback doc also mentioned "Most overdue", "Evidence-light", "Lowest confidence".

**What to add:** Define the available sort options for the event scorecard and company dashboard unshipped table. Clarify what "Most urgent" means (probably: delayed first, then overdue descending, then announced).

---

## 6. Late/Overdue column on event scorecard

**Source:** `docs/wireframe/wireframe-feedback-implemented.md` (P0), wireframe v3.1

The wireframe has a "Late" column using the `overdue_days` computed field. The spec's UI Requirements → Event Scorecard table columns don't include it.

**What to add:** Add "Late/Overdue" to the event scorecard column list. Reference the `overdue_days` computed field definition.

---

## 7. Evidence display: count vs. strength vs. type badges

**Source:** wireframe v3.1, `docs/wireframe/wireframe-feedback-implemented.md`

The wireframe evolved across versions. Current state (v3.1):
- **Event scorecard (logged in):** evidence shown as "N sources" link only (no type badges, no strength label)
- **Event scorecard (logged out):** evidence count + evidence strength label (Strong/Medium/Weak)
- **Commitment detail:** full evidence list with type badges + published date

The spec's UI section just says "Evidence count (with quick links)."

**What to add:** Clarify the evidence display per surface. Document the logged-in vs. logged-out difference (logged-out shows strength to provide more "aha" in the free tier).

---

## 8. Copy evidence list button

**Source:** `docs/wireframe/wireframe-feedback-implemented.md` (P0), wireframe v3.1

The commitment detail page has "Copy citation" (already in spec) AND "Copy evidence list" (not in spec). The evidence list output is: bullet list of sources with dates + URLs + excerpts.

**What to add:** Add to Commitment Detail UI requirements.

---

## 9. Citation format

**Source:** `docs/wireframe/wireframe-feedback-implemented.md` (P0), wireframe v3.1

The wireframe's citation block includes:
- Company, commitment title, "Launch Scorecard", commitment ID
- Status + "as of" date
- "Accessed" date (auto-filled)
- Stable URL

The spec mentions "Copy citation button" but doesn't define the format.

**What to add:** Define the canonical citation format in the spec.

---

## 10. Company dashboard: changes-since selector + activity feed filters

**Source:** `docs/wireframe/wireframe-feedback-implemented.md` (P1), wireframe v3.1

The wireframe has:
- "Since" selector on activity feed: 7d / 30d / All
- Filter button on activity feed (for upgrades only, slips only, new commitments)
- Status-change CSV export from activity feed

The spec's Company Dashboard section doesn't cover any of this.

**What to add:** Add to Company Dashboard UI requirements.

---

## 11. Company dashboard: unshipped table enhancements

**Source:** `docs/wireframe/wireframe-feedback-implemented.md` (P0), wireframe v3.1

The wireframe's unshipped commitments table on the company dashboard includes:
- Target window column
- Overdue chip
- Event name in commitment meta row
- Sort dropdown (Most urgent / Newest updates / Most evidence)
- Filter pills (All / Unshipped / Delayed)

The spec's Company Dashboard section lists "Unshipped commitments" but not these details.

**What to add:** Flesh out the unshipped table requirements.

---

## 12. Company dashboard: ship rate per event + coverage statement

**Source:** `docs/wireframe/wireframe-feedback-implemented.md` (P0), wireframe v3.1

The wireframe shows:
- Ship rate percentage per event in the events list (e.g., "56% shipped" next to each event)
- Coverage statement (now "Excludes minor UI tweaks and bugfixes")

Neither is in the spec's Company Dashboard section.

**What to add:** Add both to Company Dashboard UI requirements.

---

## 13. Commitment detail: related commitments

**Source:** `docs/wireframe/wireframe-feedback-implemented.md` (P1), wireframe v3.1

The wireframe shows a "Related commitments" sidebar on the detail page. The feedback doc says prefer:
1. Same announcement
2. Same product_area
3. Shared tags

Not in the spec at all.

**What to add:** Define the related commitments logic and add to Commitment Detail UI requirements.

---

## 14. Logged-out scorecard enhancements

**Source:** `docs/wireframe/wireframe-feedback-implemented.md` (P1), wireframe v3.1

The wireframe's logged-out scorecard includes several things not in the spec:
- Basic filters (status + search) available for anon users
- Blur microcopy ("Showing 5 of 32 commitments. Sign up for full evidence, filters, export & alerts.")
- Status distribution bar (thin colored proportional bar)
- SEO methodology block (static, crawlable version of the popover content)
- Share bar (Twitter, LinkedIn, copy link)

The spec's logged-out requirements just mention "first ~5 rows visible, remaining gated, clear CTA."

**What to add:** Expand the logged-out scorecard requirements.

---

## 15. Scope: scorecard vs. detail-only

**Source:** wireframe v3.1 annotation, `docs/wireframe/wireframe-feedback-implemented.md`

The feedback doc originally said add scope to the scorecard table (compact chips or expandable row). The wireframe v3.1 explicitly removed it: "Removed Scope column (scope shown on detail only)."

The spec currently doesn't specify where scope appears in the UI.

**What to add:** Codify the decision — scope is shown on the commitment detail page only, not on the event scorecard table. This keeps the scorecard scannable.

---

## 16. "More filters" pattern

**Source:** wireframe v3.1

The wireframe uses a collapsed "More filters" button with an active-count badge + Reset link, rather than showing all filter dimensions (category, tags, etc.) inline. The primary filter bar shows status pills + search + sort only.

**What to add:** Document the two-level filter pattern in the UI requirements.
