\# Promise Tracker — PR Checklist (P0/P1 Definition of Done)



\> Paste this into your repo as `.github/pull_request_template.md` or a “Release Checklist”.



\---



\## P0 — Must-pass (v1 ship readiness)



\### Data integrity & trust

\- [ ] **Stable IDs/URLs:** Every commitment has a stable `commitment_id` and public URL `/c/{commitment_id}` that does not change.

\- [ ] **Evidence-gated status:** Every commitment with `status != ANNOUNCED` has:

 \- [ ] ≥1 `Evidence` record

 \- [ ] ≥1 `StatusHistory` entry referencing the evidence (`evidence_id`)

\- [ ] **No orphan evidence:** Every `Evidence` row references an existing `commitment_id`.

\- [ ] **StatusHistory immutability:** Status changes append new rows (no edits-in-place).

\- [ ] **Methodology visible:** A **Methodology** popover exists and is accessible from:

 \- [ ] Company dashboard

 \- [ ] Event scorecard

 \- [ ] Commitment detail

 Includes:

 \- [ ] Status definitions

 \- [ ] Evidence priority rules

 \- [ ] Evidence strength definitions (Strong/Medium/Weak)

 \- [ ] "Last refreshed" timestamp



\### Analyst workflow (5-minute test)

\- [ ] On **Event scorecard**, user can:

 \- [ ] Filter by status and category

 \- [ ] Search commitments

 \- [ ] **Sort** by: `Most overdue` and `Newest shipped`

 \- [ ] See **Scope notes** on commitment detail page

 \- [ ] Click into a commitment detail page

 \- [ ] Export scorecard to **CSV** (logged-in)

\- [ ] On **Commitment detail**, user can:

 \- [ ] **Copy citation** (includes status + as-of date + URL)

 \- [ ] **Copy evidence list** (dates + evidence type + links + excerpts)



\### Core surfaces complete

\- [ ] **Company dashboard** includes:

 \- [ ] Ship rate (GA)

 \- [ ] Median time to GA

 \- [ ] Unshipped count

 \- [ ] Delayed count

 \- [ ] Status changes (30d)

 \- [ ] Events list

 \- [ ] Recent activity feed

 \- [ ] Unshipped commitments table (includes status + claimed/target window + age/overdue)

\- [ ] **Event scorecard** includes:

 \- [ ] KPI chips (shipped/GA, preview/beta, partial, delayed, announced-only)

 \- [ ] Commitments table columns:

  \- [ ] Commitment title

  \- [ ] Category

  \- [ ] Status pill

  \- [ ] Availability claim (raw)

  \- [ ] Target window

  \- [ ] First shipped date

  \- [ ] Evidence strength label (Strong/Medium/Weak)

  \- [ ] Evidence count link

\- [ ] **Commitment detail** includes:

 \- [ ] Promise text (canonical)

 \- [ ] Product area + category

 \- [ ] Availability claim

 \- [ ] Target window

 \- [ ] Scope notes (free text)

 \- [ ] Status + status-as-of date

 \- [ ] Full status history timeline

 \- [ ] Evidence list (type + date + excerpt + link)



\### Logged-out marketing surface (minimum)

\- [ ] Public event scorecard exists (`/event/{id}` logged-out mode) with:

 \- [ ] KPI chips visible

 \- [ ] First ~5 rows visible

 \- [ ] Remaining rows gated (blur/lock)

 \- [ ] Clear CTA to sign up

\- [ ] Pages render with correct metadata for sharing (title/description).



\---



\## P1 — Strongly recommended (stickiness + analyst delight)



\### “What changed?” first-class

\- [ ] Company dashboard has **Changes since** selector (7d / 30d / since event / custom).

\- [ ] Recent activity feed supports filters:

 \- [ ] Upgrades only (Preview→GA etc.)

 \- [ ] Slips only (missed target / delayed)

 \- [ ] New commitments only

\- [ ] **Status-change CSV export** exists (separate from commitments export).



\### Signal-to-noise controls

\- [ ] Coverage statement visible (e.g., "Excludes minor UI tweaks and bugfixes") on company dashboard and event scorecard.

\- [ ] Table state is URL-driven (filters/sort/search reflected in query params; shareable links).



\### Evidence quality is explicit

\- [ ] Evidence links show **type badges** (Release notes / Docs / Official blog / etc.).

\- [ ] Event scorecard shows **Evidence strength** label (Strong/Medium/Weak) derived from evidence types.



\### Logged-out scorecard “aha” improvement

\- [ ] Logged-out scorecard supports **basic filters** (status + category + search).

\- [ ] Export, alerts, watchlists remain gated.



\### Partial status is usable

\- [ ] `PARTIAL` rows show *why* (via scope notes on commitment detail page).

\- [ ] `scope_observed` on evidence records explains what was actually observed.



\---



\## Release sanity check (optional but useful)

\- [ ] For a seeded OpenAI event (~30 commitments):

 \- [ ] ≥80% of non-ANNOUNCED have primary evidence (Docs/Release notes/Official blog)

 \- [ ] No status changes without evidence linkage

 \- [ ] Analyst can find “most overdue” in <10s

 \- [ ] Analyst can copy citation + evidence list in <30s