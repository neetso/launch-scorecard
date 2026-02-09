this will work for the target audience *if* you lean hard into what analysts value most: **speed, trust, and citability**. Your wireframes are already pointing in the right direction (scorecards + stable commitment pages + evidence + exports). A few tweaks will make it feel “analyst-native” rather than “nice dashboard.”

## **What’s already strong (analyst-fit)**

- **Event Scorecard as the core surface**: exactly how analysts think (“WWDC 2024 — what actually happened?”).
- **Commitment Detail as a citation target** (stable URL + evidence + timeline): this is the product.
- **Status + confidence + evidence count** in the table: great—this is the credibility engine.
- **Logged-out scorecard with KPI chips + partial table + blur**: very solid SEO/marketing wedge.

## **Biggest gaps to address (to make analysts trust & pay)**

### **1) Make “trust” explicit, not implicit**

Right now, confidence bars and evidence counts are good, but analysts will still ask “*according to what rules?*”

Add a small **Methodology** popover (top right) everywhere:

- status definitions (GA vs Partial vs Preview)
- evidence priority order
- what “confidence” means (and when you cap it)
- last refresh timestamp

This reduces perceived subjectivity (especially for Apple/Meta later).

### **2) Analysts need “what changed?” more than “what exists”**

Your Company Dashboard has “Recent activity” — great. Make it more queryable:

- Add “**Changes since**: 7d / 30d / since event date” filter
- Add “**Show only upgrades** (e.g., Preview→GA) / only slips”
- Add a “**Status change feed export**” (CSV) separate from commitments

### **3) Missing the key analyst column: “Scope”**

On the Event Scorecard table, analysts will constantly ask:

- “GA for *whom*? which tier/region/API?”

  Add either:

- a **Scope** column (compact chips like “API · Pro/Team · US/EU”), or

- an expandable row “Scope + notes” that’s one click.

Right now scope is only visible deep in the commitment detail.

### **4) “Target window miss” should be a first-class signal**

You have “Delayed” and “Claimed”/“Target window” — good. Add:

- “**Days late**” (or “+X days”) when target window exists
- A sort option: **Most overdue**, **Most evidence-light**, **Highest confidence but not shipped**

Analysts love ranked lists.

## **Screen-by-screen notes**

### **Screen 1 — Company Dashboard**

What works:

- KPI strip + events list + unshipped list + activity feed = good “home base.”

Suggestions:

- Add a **time range selector** that affects all KPI chips (“last 12 months”, “since 2024”, etc.)
- Add a small **"coverage statement"** near "last updated" (e.g., "Excludes minor UI tweaks and bugfixes")
- Unshipped table: add “**Event**” as a clickable chip + “**Target window**” (if present)

### **Screen 2 — Event Scorecard**

What works:

- KPI chips + status filters + category filters + search + evidence column.

Suggestions:

- Add a **Sort** dropdown (it’s critical): newest shipped, most overdue, highest impact, lowest confidence, most cited.
- Add **Evidence strength** indicator (not just count): e.g., “Release notes + Docs” vs “Blog only.”
- “Availability claim” should preserve phrasing (you’re doing that) but also show derived window (you are) + “missed” label.

### **Screen 3 — Commitment Detail**

This is the money page.

What works:

- Quote block + scope chips + status history with evidence excerpts + “copy citation.”

Suggestions:

- Add **Promise source** more precisely:
  - “Announced at: [event] + timestamp (if video)” (later for Apple; but OpenAI can still have post anchors)
- Add **Evidence bundle download** (even just “copy evidence list”): analysts paste this into notes.
- In the citation box: include **“Accessed on {date}”** and **status as-of** (you already do status as-of).

### **Screen 4 — Public scorecard (logged out)**

What works:

- KPI chips are shareable; table preview; CTA is clear.

Suggestions:

- Let anon users use **basic filters** (status + category) even if you keep export gated. That increases “aha.”
- Add “**Why some rows are blurred**” microcopy: “Sign up for full evidence + filters + alerts.”

## **One structural recommendation: coverage statement**

Analysts will ask "which of these matters?" early on. The answer is handled at ingestion: we only track commitments that pass the eligibility rubric (no minor UI tweaks, bugfixes, or small behavior changes). A visible **coverage statement** (e.g., "Excludes minor UI tweaks and bugfixes") signals this to users without needing a per-commitment importance field.

## **Verdict for the target audience**

- **Yes, the core surfaces are right.** Analysts will understand it immediately.
- To make them *trust it* and *pay*, prioritize:
  1. Methodology + definitions everywhere
  2. Sorting/ranking + “days late”
  3. Scope visibility at table level
  4. Evidence strength (type) not just evidence count
  5. “What changed?” views/exports

----

Prioritized

## **P0 — must-have for analyst trust + paid conversion**

### **Global (all screens)**

- **Methodology popover** (top right):
  - status definitions (GA/Partial/Preview/Delayed/Replaced/Cancelled)
  - evidence priority order
  - what confidence means + when capped
  - "last refreshed" timestamp + coverage scope (e.g., "excludes minor UI tweaks")
- **Sort dropdown** wherever there’s a commitments table:
  - Most overdue (target window miss)
  - Newest shipped
  - Evidence-light (0–1 sources)
  - Lowest confidence
  - Alphabetical

### **Event Scorecard (core surface)**

- Add **Scope visibility** at table level:
  - either a compact **Scope column** (chips: tier/region/platform), or
  - expand-on-row showing scope
- Add **Days late / Overdue**:
  - computed from target_window_end vs today (or first_ship_date)
  - show as +Xd with red accent; sortable
- Evidence column: show **type badges** not just counts:
  - e.g., Release notes, Docs, Official blog (counts remain as link)

### **Commitment Detail (citation page)**

- **Copy evidence list** button (in addition to copy citation)
  - output: bullet list of sources with dates + URLs + excerpts
- Citation box includes:
  - **Status as-of date**
  - **Accessed on** date (auto)
- Evidence timeline entries show **evidence type** + **published date** prominently

### **Company Dashboard**

- Add **“Changes since” selector** (7d / 30d / since event date / custom)
- Recent Activity: filters:
  - only upgrades (Preview→GA) / only slips / only new commitments
- Unshipped list: add **Target window** + **Overdue** chip if applicable

------

## **P1 — increases usefulness + shareability (strongly recommended)**

### **Logged-out public scorecard (SEO/marketing)**

- Allow anon **basic filters** (status + category + search)
  - Keep export + watchlists gated
- Add microcopy: “Showing top X rows. Sign up for full evidence, filters, alerts.”

### **Everywhere tables exist**

- Add **Evidence strength score** (simple):
  - Release notes/docs > official blog > third party
  - Display as small label (e.g., “Strong / Medium / Weak”)

### **Commitment Detail**

- “Related commitments” should prefer:
  - same announcement
  - same product_area
  - shared tags

### **Company dashboard**

- Add “Reliability” mini-breakdown:

  - ship rate by event
  - median time-to-GA (already shown) + distribution (small sparkline later)

  

-------

Look and feel

it’s already in the right territory: **calm, analyst-y, credible**. It reads like “modern internal tool” (Stripe-ish / Linear-ish), which is exactly what you want. A few look-and-feel tweaks would make it feel *more* like an analyst workstation and *less* like a generic SaaS dashboard.

## **What’s working visually**

- **Whitespace + restrained palette** → trustworthy, not hypey.
- **Clear hierarchy** (big title, then KPI chips, then tables).
- **Status pills** are readable and scan-friendly.
- **Table density** is reasonable (not cramped like Bloomberg, not airy like consumer apps).

## **What I’d refine (high leverage)**

### **1) Increase “table density” slightly (analysts prefer more rows on screen)**

Right now it’s a touch roomy. Consider:

- slightly smaller row height
- slightly smaller body font in tables
- keep headers + titles as-is

Analysts live in lists; more visible rows = more value.

### **2) Make “status colors” more semantic + consistent**

Your colors are good, but make sure they map to expectations:

- **GA = green**
- **Delayed = red**
- **Partial = amber**
- **Preview/Beta = blue/purple**
- **Announced = neutral gray**
- **Cancelled/Replaced = dark gray / muted**

Also ensure the *same status* uses the *same hue* everywhere (chips, bars, counts, legends).

### **3) Add one “seriousness” layer: evidence strength + methodology**

This is visual tone, not just content:

- Evidence badges should look “official”: tiny, monochrome icons + subtle tags (not colorful confetti)
- Methodology link should be always visible but quiet (like “i” help)

This signals “we’re rigorous.”

### **4) Improve scanning with subtle grid lines + alignment**

Analysts’ eyes track columns. You can boost this by:

- slightly stronger header background
- subtle vertical column separators (or row hover highlighting)
- consistent alignment (dates right-aligned, numbers right-aligned)

### **5) Make KPI chips feel less “marketing dashboard”**

They’re good, but you can make them more “instrument panel”:

- reduce roundedness a bit (still modern, just less bubbly)
- use lighter borders
- make the number heavier than the label (you already do this well)

### **6) Typography: one change that helps a lot**

If you haven’t chosen type yet:

- a neutral UI font (Inter / SF Pro / IBM Plex Sans) works
- use **tabular numerals** for dates and metrics (makes tables feel pro)

### **7) Logged-out scorecard: lean into “shareable artifact”**

Visually, that page should feel like a *report excerpt*:

- keep KPI strip, then a thin status distribution bar (you have this)
- keep CTA subtle and “analyst-friendly” (avoid big gradient buttons)
- blur is fine, but blur should look like “redacted report,” not “paywall trick”

## **One concrete suggestion: add a “citation-first” visual motif**

To reinforce what makes you different, you can make evidence feel like a first-class citizen:

- evidence count links styled like citations (e.g., “3 sources” as a small blue link)
- on hover, show mini-popover with source types + dates
- in commitment detail, show evidence cards like “footnotes”