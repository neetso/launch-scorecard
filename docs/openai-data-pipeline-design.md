# OpenAI Data Pipeline — Design Doc

## Approach: Hybrid (Option C — simplified)

CLI / rake tasks for triggering extraction and evidence management. Minimal web UI for the ingestion review queue where the operator makes accept/skip/edit decisions. Everything else (event CRUD, commitment editing, evidence adds, status changes) is rake tasks or direct API calls.

**Why this approach:**
- The trigger step ("ingest this URL") is a one-shot action — a rake task or API call handles it fine.
- The review step is where the operator looks at 30–150 items, edits fields, and makes judgment calls. Doing this in JSON files or CLI output is painful at scale.
- The ingestion review queue is the only UI surface that gates data quality for extraction. Evidence is added manually via CLI.

**What was descoped (see bottom of doc for rationale):**
- Automated evidence matching (LLM Prompt 2 + search + evidence review queue)
- Continuous release notes monitoring (scraping + auto-classification)
- Announcement entity (intermediate grouping) — commitments link directly to events
- Structured scope JSON → free-text `scope_notes`
- Confidence field → `evidence_strength` (computed from evidence types)
- Tags, merge operations, bulk review queue actions

---

## Pipeline Overview

```
                        ┌─────────────────────────────────────────────┐
                        │           SOURCE CAPTURE                     │
                        │                                              │
                        │  Major launch post ──► Operator pastes URL   │
                        │  Release notes ──► Operator checks weekly    │
                        └──────────────────────┬──────────────────────┘
                                               │
                          ┌────────────────────▼─────────────────────┐
                          │  EXTRACTION (rake task + background job)  │
                          │                                           │
                          │  Fetch URL → LLM Prompt 1 → structured   │
                          │  JSON of commitments → staging table      │
                          └────────────────────┬─────────────────────┘
                                               │
                                               ▼
                        ┌─────────────────────────────────────────────┐
                        │         INGESTION REVIEW QUEUE               │
                        │         (web UI — operator reviews)          │
                        │                                              │
                        │  Per item: Accept / Edit+Accept / Skip       │
                        │  On accept ──► Commitment + StatusHistory    │
                        └──────────────────────┬──────────────────────┘
                                               │
                                               ▼
                        ┌─────────────────────────────────────────────┐
                        │         EVIDENCE COLLECTION (manual)         │
                        │                                              │
                        │  Operator finds shipping evidence            │
                        │  rake evidence:add ──► Evidence + Status     │
                        └──────────────────────┬──────────────────────┘
                                               │
                                               ▼
                        ┌─────────────────────────────────────────────┐
                        │         WEEKLY REVIEW (operator checklist)   │
                        │                                              │
                        │  Check shipping sources for non-terminal     │
                        │  commitments. Add evidence. Flag overdue.    │
                        └─────────────────────────────────────────────┘
```

---

## Stage 1: Source Capture

Operator identifies new OpenAI content and triggers ingestion manually.

**What counts as an Event for OpenAI:**

| Type | Example | `event_type` | When to create |
|------|---------|-------------|----------------|
| Named major launch | "DevDay 2026", "Spring Update 2026-03" | `DEV_CONF` or `PRODUCT_LAUNCH` | Official post bundles multiple launches or frames a major moment |
| Release wave bucket | `openai-release-wave-2026-q1` | `OTHER` | Fallback when shipping is continuous and not bundled. Split to monthly if > 25 eligible items in a quarter |

**Implementation — rake task:**

```
rake events:create company=OPENAI name="DevDay 2026" type=DEV_CONF \
  start_date=2026-10-01 hub_urls="https://openai.com/..."
```

Creates the `Event` record. Fields map directly to the Event entity schema. Low-volume operation (a few per company per year) — no UI needed.

**For release notes (weekly):** Operator checks OpenAI's product releases hub and ChatGPT release notes. Significant items are ingested via `rake ingestion:extract` with the URL or pasted text. The eligibility rubric from the product spec serves as operator guidance for what to track vs skip.

### OpenAI source map

| Purpose | Source | Evidence type |
|---------|--------|---------------|
| **Announcement anchors** | Product releases hub, major launch posts | `BLOG_OFFICIAL` |
| **Shipping evidence** | ChatGPT release notes, model/API notes | `RELEASE_NOTES` |
| **Shipping evidence** | API reference, product docs | `DOCS` |
| **Shipping evidence** | "Available now" blog posts | `BLOG_OFFICIAL` |
| **Disallowed** | Social posts (X, Reddit), community chatter | — |
| **Weak evidence only** | Secondary press | `THIRD_PARTY_VERIFICATION` (evidence_strength: Weak) |

OpenAI is private — no earnings calls, so `EARNINGS` event type and `EARNINGS_IR` evidence are skipped entirely.

---

## Stage 2: Extraction

Triggered by operator after creating an event (or when reviewing weekly release notes).

**Implementation — rake task:**

```
rake ingestion:extract event_id=openai-devday-2026 \
  url="https://openai.com/blog/devday-2026"
```

**What it does:**
1. Fetch the URL, convert HTML to clean text (or accept pasted text via `text=` param)
2. Run `IngestionExtractJob` (Solid Queue) — sends text through LLM Prompt 1 ("split recap into atomic commitments")
3. LLM returns structured JSON:
   ```json
   {
     "commitments": [
       {
         "promise_text": "GPT-5 Turbo available via API for all paid tiers",
         "product_area": "OpenAI API",
         "category": "MODEL",
         "availability_claim_raw": "available today for Plus, Team, and Enterprise",
         "target_window_start": "2026-10-01",
         "target_window_end": "2026-10-01",
         "scope_notes": "API, Plus/Team/Enterprise tiers, global",
         "source_excerpt": "We're launching GPT-5 Turbo today..."
       }
     ]
   }
   ```
4. Write extracted items to `raw_release_items` with `status: pending_review` and `event_id` linked

### LLM reliability measures

To make extraction repeatable and consistent:

- **Pin model version** explicitly (e.g., `claude-sonnet-4-20250514`). Update intentionally, not silently.
- **Temperature 0** for all extraction calls.
- **Version prompts as files** (`lib/llm/prompts/extraction_v1.txt`). Store prompt version on each `raw_release_item`.
- **Use structured output / tool_use mode** rather than raw JSON from text completion.
- **Validate LLM output** against a JSON schema before writing to the database. Reject and retry on malformed output.
- **Store raw LLM response** alongside extracted items (model ID + prompt version + input hash) for reproducibility auditing and regression testing.
- **Golden test set**: maintain 5–10 known source texts with hand-verified expected outputs. Run as regression tests when prompts or models change.

### Granularity rules — when to split

- Tier-based: Free vs Pro vs Enterprise ship independently
- Platform: web vs iOS vs API ship independently
- Region/language: different availability dates
- Component: model launch vs API support vs UI surface

**Keep together when:** one endpoint/SKU ships as a single unit, or announcement indicates a single GA moment.

---

## Stage 3: Ingestion Review Queue (Web UI)

The only review queue screen. It's where extracted candidates become real commitments.

**URL:** `/operator/ingestion/review`

### What the operator sees

A list of cards, one per extracted candidate. Each card shows:

| Field | Type | Notes |
|-------|------|-------|
| `promise_text` | Editable text | One sentence, atomic promise |
| `product_area` | Editable text | e.g., "OpenAI API", "ChatGPT" |
| `category` | Dropdown | MODEL, API, PLATFORM, PRICING, etc. |
| `availability_claim_raw` | Editable text | Exact language from the source |
| `target_window_start` / `end` | Date pickers | Nullable |
| `scope_notes` | Editable text | Free-text: platform/region/tier constraints |
| Source excerpt | Read-only | Original text from the source, for reference |

### Operator actions per item

| Action | What happens |
|--------|-------------|
| **Accept** | Creates: Commitment (with stable public slug, linked to event_id) + StatusHistory (initial: `ANNOUNCED`) + Evidence record (`BLOG_OFFICIAL` or `RELEASE_NOTES`, linking to source URL) |
| **Edit + Accept** | Same as Accept, with the operator's edits applied to fields |
| **Skip** | Item marked `skipped` in `raw_release_items`. Does not become a commitment |

### Implementation notes

- Server-rendered cards using Hotwire (Turbo Frames).
- Accept/Skip buttons submit via Turbo Stream — the card is removed from the list without a full page reload.
- Edit mode: clicking "Edit" expands the card inline with form fields. "Save + Accept" submits.
- No polish needed. Function over form.

---

## Stage 4: Evidence Collection (Manual)

After commitments exist, the operator finds and attaches evidence of delivery from OpenAI's shipping sources.

**Implementation — rake task:**

```
rake evidence:add commitment_id=openai-gpt5-turbo-api \
  url="https://platform.openai.com/docs/models/gpt-5-turbo" \
  type=DOCS published_at=2026-10-01 \
  excerpt="GPT-5 Turbo is available for all paid API tiers." \
  supports_status=GA
```

Creates Evidence record + StatusHistory entry (old_status → new_status, linked to evidence).

### Allowed evidence sources for OpenAI

| Evidence type | Can advance status? | Notes |
|---------------|-------------------|-------|
| `RELEASE_NOTES` | Yes | ChatGPT release notes, model/API notes |
| `DOCS` | Yes | API reference, product docs |
| `BLOG_OFFICIAL` | Yes | Must have explicit "available now" language |
| `CHANGELOG` | Yes | When available |
| `THIRD_PARTY_VERIFICATION` | Only when primary absent | evidence_strength: Weak |
| Social / Reddit / community | **No** | Never used for status upgrades |

### Status change rules for OpenAI

| Target status | Evidence requirement |
|---------------|---------------------|
| Beyond `ANNOUNCED` | At least one `BLOG_OFFICIAL`, `DOCS`, or `RELEASE_NOTES` |
| `GA` specifically | At least one primary source with **explicit** availability statement (not implied) |
| `PARTIAL` | Evidence indicates limited rollout → store `scope_observed` on evidence record |

### Other manual operations

**Manual status change:**

```
rake commitments:change_status commitment_id=openai-gpt5-turbo-api \
  new_status=DELAYED evidence_id=<uuid> \
  note="OpenAI blog post indicates Q1 2027 target"
```

Must reference an evidence record — no status change without evidence (except `ANNOUNCED`, which is the initial state).

**Commitment editing:**

```
rake commitments:edit commitment_id=openai-gpt5-turbo-api \
  promise_text="GPT-5 Turbo available via API for Plus, Team, and Enterprise"
```

Direct field updates. All changes logged to `operator_audit_logs`.

---

## Stage 5: Weekly Review

Operator reviews all non-terminal OpenAI commitments on a weekly cadence.

**Workflow:**
1. Review commitments where `status NOT IN [GA, CANCELLED, REPLACED]`
2. Check OpenAI's shipping sources (release notes, docs, changelogs, blogs) for new evidence
3. Add evidence via `rake evidence:add` for any status changes found
4. Flag overdue commitments for investigation:

| Condition | Action |
|-----------|--------|
| `overdue_days > 0` and status is `ANNOUNCED` or `PREVIEW` | Investigate — no progress |
| `overdue_days > 90` for any non-terminal status | May be quietly dropped |

5. Check release notes for new significant items → ingest via `rake ingestion:extract` if eligible

**SLA:** Same-day review. ~30–60 minutes for an operator at MVP scale.

---

## Data Flow Summary

### On a major OpenAI launch day

1. Operator runs `rake events:create` to create the event
2. Operator runs `rake ingestion:extract` with the launch post URL
3. `IngestionExtractJob` runs LLM Prompt 1, writes candidates to `raw_release_items`
4. Operator opens `/operator/ingestion/review`, reviews 30–100 candidates
5. Accepted items become Commitments with status `ANNOUNCED`
6. For items that announced "available today", operator adds evidence immediately via `rake evidence:add` → status moves to `GA` or `PARTIAL`

**SLA:** Draft within 1–2h. Published (reviewed) within 24–48h.

### On a regular week

1. Operator checks OpenAI release notes (product releases hub, ChatGPT release notes, API/model notes)
2. For significant new items, operator runs `rake ingestion:extract` with URL
3. New candidates appear in `/operator/ingestion/review` — operator accepts/skips
4. Operator reviews non-terminal commitments and checks shipping sources for evidence
5. Evidence added via `rake evidence:add` → statuses updated
6. Overdue commitments flagged for attention

**SLA:** Same-day review.

---

## Database Tables Involved

| Table | Role in pipeline |
|-------|-----------------|
| `events` | Created by operator (rake task). Links directly to commitments |
| `raw_release_items` | Staging table. LLM extraction output lands here. Never public |
| `commitments` | Created on ingestion accept. Stable public slug. FK to event. The core tracked unit |
| `evidence` | Created on evidence add. Dated proof of delivery |
| `status_histories` | Created on every status change. Append-only. FK to evidence (required for non-initial) |
| `operator_audit_logs` | Every operator write logged with before/after values |

---

## Configuration

Company-specific rules stored at `config/company_rules/openai.json`:

```json
{
  "company": "OPENAI",
  "event_strategy": {
    "preferred": ["named_major_launch_post", "dev_event"],
    "fallback": { "bucket": "quarterly", "split_if_eligible_over": 25 }
  },
  "sources": {
    "announcement_primary": [
      { "type": "BLOG_OFFICIAL", "name": "Product releases hub" },
      { "type": "BLOG_OFFICIAL", "name": "Major launch posts" }
    ],
    "shipping_primary": [
      { "type": "RELEASE_NOTES", "name": "ChatGPT release notes" },
      { "type": "DOCS", "name": "API/model documentation" },
      { "type": "BLOG_OFFICIAL", "name": "Launch availability posts" }
    ],
    "shipping_disallowed": ["SOCIAL", "REDDIT"]
  },
  "status_rules": {
    "ga_requires_primary": true,
    "partial_if_limited_rollout": true
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
  }
}
```

---

## What to Build (ordered)

### Phase 1: Database + models
1. Migrations for all tables (`events`, `commitments`, `evidence`, `status_histories`, `raw_release_items`, `operator_audit_logs`)
2. ActiveRecord models with associations, validations, computed field methods
3. Status transition enforcement in `StatusHistory` (evidence required for non-initial)

### Phase 2: LLM integration
4. `lib/llm/extraction.rb` — Prompt 1 runner (Anthropic SDK), with structured output + JSON schema validation
5. `lib/llm/prompts/extraction_v1.txt` — versioned prompt file

### Phase 3: Rake tasks (CLI triggers)
6. `rake events:create` — event CRUD
7. `rake ingestion:extract` — fetch URL + run LLM extraction + write to raw_release_items
8. `rake evidence:add` — manual evidence creation + status history entry
9. `rake commitments:change_status` — manual status change (requires evidence)
10. `rake commitments:edit` — field updates

### Phase 4: Background jobs
11. `IngestionExtractJob` — async LLM extraction (Solid Queue)

### Phase 5: Ingestion review queue web UI
12. `/operator/ingestion/review` — review queue (list of cards, accept/skip/edit)
13. Operator auth gate (Devise + Pundit, role: operator)

### Phase 6: Verify done criteria
14. At least 2 Events ingested (1 named launch + 1 release-wave bucket)
15. >= 30 commitments with stable URLs
16. Every non-ANNOUNCED status has evidence + StatusHistory row
17. Company dashboard shows: unshipped list, recently changed statuses, time-to-GA stats

---

## Descoped Features — Rationale and Reintroduction Triggers

These features were evaluated and explicitly deferred to reduce pipeline complexity. Each is viable for a future version.

| Feature | What it was | Why deferred | Reintroduce when |
|---------|-------------|-------------|------------------|
| **Automated evidence matching** | LLM Prompt 2 searches shipping sources + validates matches → evidence review queue | Hardest technical problem (retrieval strategy underspecified). Second LLM touchpoint adds non-determinism. Manual evidence add is ~30–60 min/week at MVP scale. | Commitment count exceeds ~300, or structured APIs become available for shipping sources |
| **Evidence review queue** | Web UI for reviewing LLM-suggested evidence matches (Stage 5 in original design) | Dependent on automated evidence matching. No value without it. | When automated evidence matching is implemented |
| **Weekly automated refresh** | Cron job re-running evidence matching for all non-terminal commitments | Dependent on automated evidence matching. Manual weekly review replaces it. | When automated evidence matching is implemented |
| **Continuous release notes monitoring** | Scraping OpenAI's release notes pages → `raw_release_items` staging table + LLM rubric scoring | Scraping fragility, dual-purpose staging table, third LLM touchpoint. Operator can check release notes weekly. | Shipping cadence too high for manual monitoring, or structured RSS/API feeds available |
| **Announcement entity** | Intermediate grouping between Event and Commitment (e.g., "GPT-5 Turbo" as an announcement with sub-commitments) | Extra join table, extraction step, and UI layer. Not in any public URL or computed field. `category` on Commitment provides lightweight grouping. | Cross-event analysis needs coarse groupings |
| **Structured scope JSON** | `scope: { platform: [], region: [], tier: [], audience: [], notes: "" }` on Commitment + `scope_observed` JSON on Evidence | Hardest field for LLM to extract. Drives complex matching logic. Free-text `scope_notes` covers OpenAI use cases. | Multi-company comparisons need machine-queryable scope filtering |
| **Confidence field** | `confidence: float 0–1` with auto-adjustment rules (0.7 cap for third-party, 0.8 for partial) | `evidence_strength` (Strong/Medium/Weak) provides the same signal more interpretably. Confidence adds branching logic to every evidence acceptance. | Phase 3 (Apple/Meta) where ambiguity is higher |
| **Tags** | 12 canonical tags per commitment, multi-select in review UI | Extra LLM extraction field, filter dimension on every view. `category` + `product_area` + search covers filtering at MVP scale. | 500+ commitments or cross-company tag views needed |
| **Merge operation** | Duplicate resolution: redirect old commitment_id, move evidence/history between records | Requires redirect logic and complex data migration. Rare at MVP scale with one operator. | Multi-source ingestion creates frequent duplicates |
| **Bulk review actions** | "Accept all", "Skip all remaining", "Set shared fields" in review queue | UI complexity for an optimization. Per-item review is fine at 30–100 items. | Ingestion batches regularly exceed 100 items |
