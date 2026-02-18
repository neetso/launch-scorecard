# Designer Wireframe Update Brief

Several features have been descoped from v1. This note summarizes what needs to change in the wireframes.

---

## Components removed

| Component | Was | Action |
|-----------|-----|--------|
| **Confidence bar / value** | Float 0–1 displayed on scorecard table and commitment detail | **Remove entirely.** Replace with `evidence_strength` label where confidence appeared. |
| **Scope chips** | Structured chips for platform / region / tier / audience on scorecard and detail | **Replace with free-text `scope_notes`** on commitment detail page only. Not shown on scorecard table (too variable for a column). |
| **Tags** | Multi-select tag chips on commitment cards + tag filter on scorecard | **Remove entirely.** No tag column, no tag filter, no `/tag/{tag}` page. |
| **Evidence review queue** | Second operator review screen (`/operator/evidence/review`) with side-by-side commitment + evidence layout | **Remove entirely.** Evidence is added via CLI in v1. Only the ingestion review queue remains. |

---

## Components changed

### Event Scorecard table

| Column | Change |
|--------|--------|
| Confidence | **Remove.** Was a numeric value in the table. |
| Evidence | Keep "N sources" link (logged-in) and count + **evidence_strength label** (logged-out). Evidence strength is new: badge showing Strong (green) / Medium (blue) / Weak (gray). |

### Event Scorecard filters

| Filter | Change |
|--------|--------|
| "More filters" | Remove `tag` from the collapsed filter set. Keep `category` and `product_area` only. |

### Commitment Detail page

| Section | Change |
|---------|--------|
| Scope | Was structured chips (platform, region, tier, audience). Now **free-text `scope_notes`** — a simple text block. |
| Confidence | **Remove.** |
| Tags | **Remove.** |
| Related commitments sidebar | Simplify priority: 1. same event, 2. same product_area. Remove "shared tags" grouping. Show title + status only (no scope chips). |

### Methodology Popover

| Section | Change |
|---------|--------|
| Confidence | **Remove** the "Confidence: 0–1 scale; capped at 0.8 for PARTIAL..." paragraph. |
| Evidence strength | **Add** (if not already present): "Strong (release notes / changelogs / docs), Medium (official blog), Weak (third-party / indirect), None (no evidence)." |

### Ingestion Review Queue (operator)

| Element | Change |
|---------|--------|
| Scope fields | Was three rows of editable chips (platform, region, tier). Now **single editable text field** `scope_notes`. |
| Tags | **Remove** the multi-select tag picker. |
| Merge button | **Remove.** Only Accept, Edit+Accept, and Skip remain. |
| Bulk actions bar | **Remove** ("Accept all", "Skip all remaining", "Set shared fields"). |

### Evidence Review Queue (operator)

**Remove this entire screen.** It no longer exists in v1. Evidence is added via CLI.

---

## New component

### Evidence Strength Badge

Small label badge used wherever evidence quality is shown:
- **Strong** = green badge (release notes, changelogs, or docs)
- **Medium** = blue badge (official blog or roadmap)
- **Weak** = gray badge (third-party / indirect)
- **None** = faint/hidden (no evidence records)

Used on: Event Scorecard table (evidence column), Commitment Detail (evidence list), Methodology Popover.

---

## Summary of UI surfaces affected

| Surface | Changes needed |
|---------|---------------|
| Event Scorecard (logged-in) | Remove confidence column. Add evidence_strength badge in evidence column. Remove tag filter. |
| Event Scorecard (logged-out) | Same as above. Evidence column shows count + evidence_strength label. |
| Commitment Detail | Replace scope chips with scope_notes text. Remove confidence. Remove tags. Simplify related commitments sidebar. |
| Company Dashboard | Remove confidence references from any KPI chips or stats. |
| Methodology Popover | Replace confidence section with evidence strength definitions. |
| Ingestion Review Queue | Replace scope chips with text field. Remove tags picker. Remove merge button. Remove bulk actions. |
| Evidence Review Queue | **Delete entirely.** |
