# **UX Review 3 screens**

## **My understanding (with confidence + time-to-clarity)**

### **5) Time-to-clarity**

**~10s** to form a confident mental model (Company → Event → Commitment with status/evidence), **~30s** to feel confident about the *intended user workflow* (monitor → investigate → cite/export).

### **6) What I believe the product is**

A **promise/launch tracking dashboard** that aggregates **announcements/commitments** made by a company (grouped by events/releases), tracks **whether/when they ship**, and provides **evidence + citations** to support the status (GA/preview/delayed/etc.).

*(I’m not using the product name as proof; this is inferred from “commitments,” “shipped,” “evidence,” “copy citation,” “status history,” and “delayed/overdue.”)*

### **7) Primary user + JTBD**

**Primary user (inferred):** analysts, product marketers, competitive intelligence folks, journalists/researchers, investor-relations watchers—anyone who needs a credible, auditable view of “what was promised vs what shipped.”

**JTBD:** “Help me quickly understand what a company said it would deliver, what actually shipped (and when), what slipped, and give me evidence I can export/cite.”

### **8) Single most important outcome these 3 screens drive**

**Enable a user to confidently answer (and share):**

> “For this company and this event, what’s shipped vs unshipped, what’s overdue, and what evidence supports that status?”

------

## **Expected hierarchy / flow**

### **9) Screen roles + top questions answered**

**Screen 1 — Company dashboard (overview + action hub)**

Answers:

- “How is this company doing overall (ship rate, delays, unshipped)?”
- “What changed recently (activity feed) and what’s most overdue?”

**Screen 2 — Event scorecard (compare/list + filtering)**

Answers:

- “For this specific event/release, which commitments are GA/preview/partial/delayed/announced?”
- “Which commitments are risky (late, low confidence, weak evidence) and where should I click next?”

**Screen 3 — Commitment detail (detail + verification + citation)**

Answers:

- “What exactly was claimed, what’s the current status, and why?”
- “What evidence supports this, and how do I cite/share it?”

### **10) Primary navigation model (what happens on click)**

**Company → Event → Commitment**:

- Clicking an **event row** on Screen 1 takes you to the **event scorecard** (Screen 2).
- Clicking a **commitment row** on Screen 2 takes you to **commitment detail** (Screen 3).
- Breadcrumbs allow stepping back up (Commitment → Event → Company).
- “Export CSV” likely exports the currently scoped table (company-level commitments or event-level commitments).

### **11) Key assumptions I’m making (because not explicit)**

1. **“Commitment” = a discrete announced deliverable/feature** (High).
2. **“Shipped” means publicly available to the scoped audience/region** (Medium) — scope chips imply this, but definition isn’t shown.
3. **Status labels (GA/Preview/Beta/Partial/Announced/Delayed) have standardised rules** (Medium) — implied by consistency, but rules aren’t visible.
4. **Confidence score is a model/human estimate of likelihood/quality** (Medium) — shown as 0.xx with bar, but provenance is unclear.
5. **Evidence strength (“Strong/Medium/Weak”) follows a rubric** (Low–Medium) — not explained; could be subjective.

------

## **Evidence from the wireframe (what caused these beliefs)**

### **12) What in the UI led to the beliefs above?**

- **Entities & hierarchy:**
  - Breadcrumbs: “Companies > OpenAI” (Screen 1), “OpenAI > Spring Launch 2026” (Screen 2), “OpenAI > Spring Launch 2026 > Commitment” (Screen 3).
  - Page titles: company name, event name, commitment name.
- **Promise vs reality framing:**
  - KPI tiles: “Ship rate,” “Median to GA,” “Unshipped,” “Delayed,” “Changes (30d)” (Screen 1).
  - Event summary chips: “GA / Preview/Beta / Partial / Delayed / Announced” and “% Shipped” (Screen 2).
  - Overdue column with “+158d / +38d” (Screen 1 unshipped list).
- **Evidence & auditability:**
  - “Evidence” column with count + strength tag (Screen 2).
  - “Status history” timeline and “Evidence” list (Screen 3).
  - “Copy citation” box with accessed date + URL (Screen 3).
- **User intent to share/export:**
  - “Export CSV” on Screens 1 & 2; “Copy citation” on Screen 3; “Share/Copy link” on Screen 2/3.

### **13) Where I felt uncertain (and why)**

- **Meaning of “Commitments” vs “Events” vs “Release waves”:** I infer events are launch moments grouping commitments, but naming isn’t defined.
- **Definition of “Shipped” and relationship to “GA”:** You show “Shipped %” and “GA count” simultaneously; unclear whether “shipped” includes beta/preview/partial.
- **Confidence & evidence strength:** You show numbers/bars (“0.60”, “0.95”) and “Strong/Medium/Weak,” but don’t reveal the rubric/method in-flow (there is a “Methodology” link in nav, but not contextually attached to the metrics).
- **Scope interpretation:** chips like “All paid,” “Plus/Pro,” “US,” “Global,” “Enterprise” exist, but unclear how they affect the shipped calculation.

------

## **Risks + fixes (P0/P1)**

### **14) Top 5 UX risks (ranked by severity)**

1. **Ambiguity: “Shipped” vs “GA” vs “Preview/Beta/Partial” relationship**
2. **Confidence score & evidence strength lack immediate interpretability**
3. **Company dashboard mixes two mental modes (tracking + investigating) without strong scoping cues**
4. **Over-filtering complexity on Event scorecard (status/category/tier/search/sort) risks analysis paralysis**
5. **CSV export ambiguity (what exactly is exported?) and share/citation expectation management**

### **15) For each risk: behaviour → smallest fix → priority**

**Risk 1: “Shipped” vs “GA” ambiguity**

- *Likely behaviour:* wrong conclusions (“56% shipped” but “18 GA” — is GA included?); miscommunication when sharing.
- *Smallest fix:* add microcopy next to “% Shipped” on Screens 1 & 2:
  - Example: “Shipped = GA + Preview/Beta + Partial (audience-scoped)” **or** whatever your rule is.
  - Add a tooltip on “Shipped” and “GA” chips with the rule.
- **P0**

**Risk 2: Confidence & evidence strength unclear**

- *Likely behaviour:* users ignore the metrics or over-trust them; reduced credibility.
- *Smallest fix:* inline tooltip on “Conf.” header and on “Strong/Medium/Weak” badge:
  - “Confidence reflects [source quality + recency + corroboration].”
  - “Strength = # independent sources + primary-source weight.”
- **P0**

**Risk 3: Company dashboard scoping ambiguity (Activity vs Unshipped vs Events)**

- *Likely behaviour:* users assume Activity is for the selected event when it’s actually company-wide (or vice versa); missed signals.
- *Smallest fix:* add explicit scoping labels above panels:
  - “Company activity (OpenAI) — last 7d”
  - “Unshipped commitments (company-wide)”
  - And/or allow clicking an event to filter Activity/Unshipped with a visible “Filtered by: Spring Launch 2026” pill.
- **P0** (if current behaviour is ambiguous)

**Risk 4: Filter density on Event scorecard**

- *Likely behaviour:* users don’t know what to do first; they fiddle with filters instead of scanning “most important” items.
- *Smallest fix:* provide a single default sort that matches the intended workflow (e.g., “Most overdue / lowest confidence / weakest evidence”), and label it as recommended: “Sort: Recommended (risk-first).”
- **P1** (unless testing shows consistent confusion)

**Risk 5: Export/share expectation mismatch**

- *Likely behaviour:* users click Export expecting filtered/scoped export but get everything; distrust or friction.
- *Smallest fix:* on Export button hover/click, show “Exports current view (filters + search applied)” and include a checkbox if not. Similar for “Share” (event-level link) and “Copy citation” (commitment-level).
- **P1** (could be P0 if export is a core promise)



------

Below are the **highest-leverage things to de-prioritise or hide**, framed around: *keep the analyst workflow fast, keep casual users from getting lost*.

------

## **What to de-prioritise / hide (v1)**

### **1) Hide “advanced dimensions” behind** 

### **More filters**

Right now, analysis paralysis mostly comes from *too many axes at once*.

**Keep visible (v1):**

- **Status** (incl. “Unshipped”, “Delayed” quick toggles)
- **Search** (commitment text / product area keywords)
- **Sort** (a single “Recommended” default, plus 1–2 alternates)

**Hide behind “More filters” (v1):**

- Category
- Tier / Region / Platform (scope facets)
- Tags
- Product area (as a filter; keep it as a column or secondary text)

**Why this works:** Users can still find things via search + status + default sort, and power users can expand when needed.



------

### **2) Reduce the number of** at-a-glance metrics on the Company dashboard**

Your company dashboard currently tries to do **health + investigation + recency** at once.

**De-prioritise / hide (v1):**

- “Median to GA” (it’s analytically interesting but cognitively heavy and ambiguous until users trust definitions)
- “Changes (30d)” as a KPI (keep “Activity” itself)

**Keep (v1):**

- Ship rate
- Unshipped count
- Delayed count
- “Updated X ago” (trust cue)

**Why:** Ship rate/unshipped/delayed map directly to the “promise ledger” value; “median to GA” invites interpretation debates.

---

### **3) Evidence “strength” + confidence: move from** primary UI to detail/tooltips**

Your spec includes confidence and evidence types. Those are great, but in v1 they can *read like algorithmic judgement* users will question.

**De-prioritise (v1 list views):**

- Evidence **strength labels** (Strong/Medium/Weak)
- Confidence **bars/numbers** as a visible column

**Keep (v1 list views):**

- **Evidence count** (with quick links) — this is concrete and easy

**Where to keep confidence/strength (v1):**

- On **Commitment Detail** only, as a small “How certain are we?” block with a tooltip (“confidence reflects evidence recency/quality + scope ambiguity”)

**Why:** List views should be for scanning; detail views can justify nuance.

----

### **4) Constrain sorting: one default + two “power” options**

Sort dropdowns are a classic “analysis paralysis accelerant.”

**Recommend (v1):**

- Default: **“Most urgent”** (e.g., overdue first, then delayed, then lowest evidence count)
- Optional: “Newest updates”
- Optional: “Most evidence”

**Hide (v1):**

- Custom multi-sort
- Too many sort modes (“lowest confidence”, “highest importance”) until you’ve validated users understand those concepts

----

### **5) Delay “Announcements” as a first-class concept in the UI**

Your data model includes Announcement as a grouping within an event — useful internally — but it can add one more mental object for users to parse.

**De-prioritise (v1 UI):**

- Showing “Announcement” as a separate level/navigation element

**Use it (v1) only as:**

- A lightweight secondary label/group header within the Event scorecard (optional), not a clickable entity

**Why:** The user’s mental model is already Company → Event → Commitment. Adding Announcement risks “wait, what’s the difference?”



## **Per-screen: what I’d simplify first**

### **Company Dashboard (v1)**

- Keep: Events list, Unshipped list, simple Activity feed.
- Hide: “Median to GA”, “Changes(30d)” KPI tile.
- Consider: Make **Unshipped the primary module**; Activity becomes secondary (or collapsible).

### **Event Scorecard (v1)**

- Keep columns aligned to your spec minimum: Commitment, claim, target window, status, first shipped, evidence count.
- Visible filters: Status + Search only.
- Everything else behind “More filters”.

### **Commitment Detail (v1)**

- Keep: promise text, scope, status history timeline, evidence list, copy citation.
- De-emphasize: confidence/strength as subtle metadata, not a headline.

------



## **One clarity upgrade**

**Add a one-line “purpose statement” per level** (tiny, under the title):

- Company: “Track what this company announced vs shipped, with evidence.”
- Event: “Scorecard of commitments announced for this event and their ship status.”
- Commitment: “Claim + status + evidence trail suitable for citation.”