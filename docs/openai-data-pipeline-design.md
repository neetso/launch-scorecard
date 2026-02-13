# OpenAI Data Pipeline — Design Doc

## Approach: Hybrid (Option C)

CLI / rake tasks for triggering extraction and evidence matching. Minimal web UI for the two review queues where the operator makes accept/skip/edit decisions. Everything else (event CRUD, commitment editing, manual evidence adds) starts as rake tasks or direct API calls.

**Why this approach:**
- The trigger step ("ingest this URL", "run evidence matching") is a one-shot action — a rake task or API call handles it fine.
- The review step is where the operator looks at 30–150 items, edits fields, and makes judgment calls. Doing this in JSON files or CLI output is painful at scale.
- The two review queues (ingestion review + evidence review) are the only UI surfaces that gate data quality. Everything else can be headless.

---

## Pipeline Overview

```
                        ┌─────────────────────────────────────────────┐
                        │           SOURCE CAPTURE                     │
                        │                                              │
                        │  Major launch post ──► Operator pastes URL   │
                        │  Release notes feed ──► raw_release_items    │
                        └──────────────┬──────────────┬────────────────┘
                                       │              │
                          ┌────────────▼──┐    ┌──────▼───────────┐
                          │  EXTRACTION   │    │  WEEKLY TRIAGE   │
                          │  (rake task)  │    │  (rake task)     │
                          │               │    │                  │
                          │  Prompt 1 ──► │    │  Rubric scoring  │
                          │  LLM output   │    │  score >= 2 ──►  │
                          └──────┬────────┘    └──────┬───────────┘
                                 │                    │
                                 ▼                    ▼
                        ┌─────────────────────────────────────────────┐
                        │         INGESTION REVIEW QUEUE               │
                        │         (web UI — operator reviews)          │
                        │                                              │
                        │  Per item: Accept / Edit+Accept / Skip / Merge│
                        │  On accept ──► Commitment + StatusHistory    │
                        └──────────────────────┬──────────────────────┘
                                               │
                                               ▼
                        ┌─────────────────────────────────────────────┐
                        │         EVIDENCE MATCHING                    │
                        │         (rake task / cron job)               │
                        │                                              │
                        │  Search shipping sources for each commitment │
                        │  Prompt 2 ──► LLM validates each candidate  │
                        └──────────────────────┬──────────────────────┘
                                               │
                                               ▼
                        ┌─────────────────────────────────────────────┐
                        │         EVIDENCE REVIEW QUEUE                │
                        │         (web UI — operator reviews)          │
                        │                                              │
                        │  Per match: Accept / Edit+Accept / Reject / Flag│
                        │  On accept ──► Evidence + StatusHistory      │
                        └──────────────────────┬──────────────────────┘
                                               │
                                               ▼
                        ┌─────────────────────────────────────────────┐
                        │         WEEKLY REFRESH                       │
                        │         (cron — re-runs evidence matching)   │
                        │                                              │
                        │  All commitments NOT IN [GA, CANCELLED,      │
                        │  REPLACED] ──► evidence matching ──► review  │
                        └─────────────────────────────────────────────┘
```

---

## Stage 1: Source Capture

### 1a. Major launch posts (event-driven)

Operator identifies a new OpenAI launch post and triggers ingestion manually.

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

### 1b. Release notes (continuous monitoring)

New release note entries from OpenAI's product releases hub and ChatGPT release notes land in the `raw_release_items` staging table.

**Implementation — rake task or scheduled job:**

```
rake openai:collect_release_notes
```

Fetches new entries since last collection, stores each as a `raw_release_items` row (date + text + URL + `status: pending`). This table is internal-only, never exposed publicly.

### OpenAI source map

| Purpose | Source | Evidence type |
|---------|--------|---------------|
| **Announcement anchors** | Product releases hub, major launch posts | `BLOG_OFFICIAL` |
| **Shipping evidence** | ChatGPT release notes, model/API notes | `RELEASE_NOTES` |
| **Shipping evidence** | API reference, product docs | `DOCS` |
| **Shipping evidence** | "Available now" blog posts | `BLOG_OFFICIAL` |
| **Disallowed** | Social posts (X, Reddit), community chatter | — |
| **Reduced confidence only** | Secondary press | `THIRD_PARTY_VERIFICATION` (confidence capped at 0.7) |

OpenAI is private — no earnings calls, so `EARNINGS` event type and `EARNINGS_IR` evidence are skipped entirely.

---

## Stage 2: Extraction

### 2a. Major launch extraction

Triggered by operator after creating an event.

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
     "announcements": [
       {
         "title": "GPT-5 Turbo",
         "category": "MODEL",
         "source_quote": "We're launching GPT-5 Turbo today...",
         "commitments": [
           {
             "promise_text": "GPT-5 Turbo available via API for all paid tiers",
             "product_area": "OpenAI API",
             "availability_claim_raw": "available today for Plus, Team, and Enterprise",
             "target_window_start": "2026-10-01",
             "target_window_end": "2026-10-01",
             "scope": {
               "platform": ["API"],
               "region": ["global"],
               "tier": ["Plus", "Team", "Enterprise"],
               "notes": ""
             }
           }
         ]
       }
     ]
   }
   ```
4. Write extracted items to `raw_release_items` with `status: pending_review` and `event_id` linked

**Granularity rules — when to split:**
- Tier-based: Free vs Pro vs Enterprise ship independently
- Platform: web vs iOS vs API ship independently
- Region/language: different availability dates
- Component: model launch vs API support vs UI surface

**Keep together when:** one endpoint/SKU ships as a single unit, or announcement indicates a single GA moment.

### 2b. Weekly release notes triage

For continuous release notes that aren't part of a named event.

**Implementation — rake task:**

```
rake openai:classify_release_notes
```

**What it does:**
1. Pull all `raw_release_items` with `status: pending` for OpenAI
2. Run LLM-assisted classification using the eligibility rubric:

   | Signal | Score |
   |--------|-------|
   | New model family / major model release | +3 |
   | Pricing / limits change | +3 |
   | New foundational API primitive | +3 |
   | Deprecation / retirement deadline | +3 |
   | Enterprise/admin/compliance feature | +2 |
   | Multimodal I/O addition | +2 |
   | Marketplace/distribution primitive | +2 |
   | Major staged rollout ("now for Pro, later for Free") | +2 |
   | Widely impactful ChatGPT workflow/UI change | +1 |
   | Meaningful integration/connectors expansion | +1 |
   | Minor UI/bugfix, copy change, small tweak | 0 |

3. **Score >= 2** → mark as `pending_review` (will appear in ingestion review queue)
4. **Score <= 1** → mark as `skipped` (stays as raw note, never becomes a commitment)

**Commitment eligibility test** — a candidate must change at least one of:
- **Capability:** new model or material capability unlock
- **Availability:** new tier/region/platform or major rollout stage
- **Economics:** price/limits/bundling
- **Distribution:** new channel, enterprise controls, marketplace primitive
- **Governance:** policy/safety changes that affect what's possible to build/use

Items that don't pass stay as raw notes only. The eligibility tier is an **ingestion gate**, not a stored field — there is no `importance` column on `Commitment`.

---

## Stage 3: Ingestion Review Queue (Web UI)

This is the first of two review queue screens. It's where extracted candidates become real commitments.

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
| `scope.platform` | Editable chips | e.g., API, Web, iOS |
| `scope.region` | Editable chips | e.g., US, EU, global |
| `scope.tier` | Editable chips | e.g., Free, Plus, Enterprise |
| `tags` | Multi-select | From canonical OpenAI tag list (see below) |
| Source excerpt | Read-only | Original text from the source, for reference |
| Eligibility score | Read-only | The rubric score that got this item here |

### Operator actions per item

| Action | What happens |
|--------|-------------|
| **Accept** | Creates: Announcement (if new) + Commitment (with stable public slug) + StatusHistory (initial: `ANNOUNCED`) + Evidence record (`BLOG_OFFICIAL` or `RELEASE_NOTES`, linking to source URL) |
| **Edit + Accept** | Same as Accept, with the operator's edits applied to fields |
| **Skip** | Item marked `skipped` in `raw_release_items`. Does not become a commitment |
| **Merge** | Link to an existing commitment (for duplicates from different sources). Adds as additional evidence on the existing commitment |

### Bulk actions

- **Accept all** (with confirmation modal)
- **Skip all remaining**
- **Set shared fields** — all items from the same source share event_id, source_url

### Canonical OpenAI tags

The operator selects 1–3 tags per commitment from:

`model_launch` · `model_deprecation` · `pricing_limits` · `api_primitive` · `tools_agents` · `multimodal` · `enterprise_admin` · `marketplace_distribution` · `safety_policy` · `memory_personalization` · `connectors_integrations` · `apps_desktop_mobile`

### Implementation notes

- Server-rendered cards using Hotwire (Turbo Frames).
- Accept/Skip buttons submit via Turbo Stream — the card is removed from the list without a full page reload.
- Edit mode: clicking "Edit" expands the card inline with form fields. "Save + Accept" submits.
- No polish needed. Function over form.

---

## Stage 4: Evidence Matching

After commitments exist, search OpenAI's shipping sources for proof of delivery.

**Implementation — rake task:**

```
# Match evidence for a single commitment
rake evidence:match commitment_id=openai-gpt5-turbo-api

# Match for all commitments in an event
rake evidence:match event_id=openai-devday-2026

# Match for all non-terminal OpenAI commitments
rake evidence:match company=OPENAI
```

**What it does:**
1. For each commitment in scope, search OpenAI's shipping sources (release notes, docs, changelogs, official blogs) for references to the commitment's product area, feature name, or keywords
2. Run `EvidenceMatchJob` (Solid Queue) — for each candidate, send through LLM Prompt 2 ("does this evidence prove shipping?")
3. LLM returns:
   ```json
   {
     "decision": "ga",
     "date": "2026-10-01",
     "excerpt": "GPT-5 Turbo is now available via the API for all paid tiers.",
     "reasoning_bullets": [
       "Blog post explicitly states 'available today'",
       "Scope matches: API, all paid tiers"
     ]
   }
   ```
4. Write candidate matches to a review table with `status: pending_review`

### Allowed evidence sources for OpenAI

| Evidence type | Can advance status? | Notes |
|---------------|-------------------|-------|
| `RELEASE_NOTES` | Yes | ChatGPT release notes, model/API notes |
| `DOCS` | Yes | API reference, product docs |
| `BLOG_OFFICIAL` | Yes | Must have explicit "available now" language |
| `CHANGELOG` | Yes | When available |
| `THIRD_PARTY_VERIFICATION` | Only when primary absent | Confidence capped at 0.7 |
| Social / Reddit / community | **No** | Never used for status upgrades |

### Status change rules for OpenAI

| Target status | Evidence requirement |
|---------------|---------------------|
| Beyond `ANNOUNCED` | At least one `BLOG_OFFICIAL`, `DOCS`, or `RELEASE_NOTES` |
| `GA` specifically | At least one primary source with **explicit** availability statement (not implied) |
| `PARTIAL` | Evidence indicates limited rollout → store `scope_observed`, cap confidence at 0.8 |

---

## Stage 5: Evidence Review Queue (Web UI)

The second review queue screen. This is where evidence candidates become real status changes.

**URL:** `/operator/evidence/review`

### What the operator sees

A list of cards, one per candidate evidence match. Each card shows two panels side by side:

**Left panel — the commitment:**

| Field | Notes |
|-------|-------|
| `promise_text` | Read-only |
| Current `status` | StatusPill component |
| `target_window` | Start → end dates |
| `overdue_days` | If applicable |

**Right panel — the evidence candidate:**

| Field | Type | Notes |
|-------|------|-------|
| URL | Link | Opens source in new tab |
| `published_at` | Date | When the evidence was published |
| `excerpt` | Read-only | <= 240 chars, from LLM extraction |
| `evidence_type` | Badge | RELEASE_NOTES, DOCS, BLOG_OFFICIAL, etc. |
| LLM decision | Badge | Suggested status: ga / partial / preview / etc. |
| LLM reasoning | Bullets | Why the LLM thinks this matches |
| `scope_observed` | Editable | For partial shipping — what the evidence actually indicates |

### Operator actions per match

| Action | What happens |
|--------|-------------|
| **Accept** | Creates: Evidence record + StatusHistory entry (old_status → new_status, linked to evidence). Confidence auto-adjusted (see below) |
| **Edit + Accept** | Same, with operator corrections (e.g., change GA → PARTIAL, adjust scope_observed, override confidence) |
| **Reject** | No evidence record created. Optionally logged as "reviewed, not relevant" |
| **Flag for later** | Marked for re-check in next refresh cycle |

### Confidence auto-adjustment on accept

| Condition | Confidence cap |
|-----------|---------------|
| Evidence type is `THIRD_PARTY_VERIFICATION` only | 0.7 |
| Status set to `PARTIAL` | 0.8 |
| Evidence type is `RELEASE_NOTES` or `DOCS` | Up to 1.0 |
| Operator override | Any value (manual) |

Default confidence for OpenAI: 0.9 (from company config).

### Implementation notes

- Same Hotwire/Turbo approach as the ingestion review queue.
- Side-by-side layout: commitment context on the left, evidence candidate on the right.
- Accept/Reject buttons remove the card via Turbo Stream.
- "Edit" expands the right panel with editable fields for status override, scope_observed, and confidence.

---

## Stage 6: Weekly Refresh

Re-run evidence matching for all non-terminal OpenAI commitments on a weekly schedule.

**Implementation — cron job + rake task:**

```
# Triggered by Solid Queue cron, or manually:
rake refresh:weekly company=OPENAI
```

**What it does:**
1. Generate refresh list: all OpenAI commitments where `status NOT IN [GA, CANCELLED, REPLACED]`
2. Run evidence matching (Stage 4) for each commitment against OpenAI's shipping sources
3. New candidate matches land in the evidence review queue (Stage 5)
4. Log a refresh summary:
   - How many commitments checked
   - How many new evidence matches found
   - How many status changes pending review
   - How many now overdue (`target_window_end < today`, still not shipped)

### Overdue alerts

After each refresh, flag commitments for operator attention (internal only, not customer-facing):

| Condition | Alert |
|-----------|-------|
| `overdue_days > 0` and status is `ANNOUNCED` or `PREVIEW` | No progress — needs investigation |
| `overdue_days > 90` for any non-terminal status | Long-stale — may be quietly dropped |
| `confidence < 0.5` and status is `PARTIAL` | Weak evidence for partial claim |

These are logged to stdout / operator dashboard. The operator addresses them during the weekly review cycle.

---

## Supporting Operations (CLI-only, no UI)

These are low-volume operations that don't need a review queue.

### Manual evidence add

```
rake evidence:add commitment_id=openai-gpt5-turbo-api \
  url="https://platform.openai.com/docs/models/gpt-5-turbo" \
  type=DOCS published_at=2026-10-01 \
  excerpt="GPT-5 Turbo is available for all paid API tiers." \
  supports_status=GA
```

Creates Evidence record + StatusHistory entry. For cases where automated matching didn't find the evidence but the operator has the source.

### Manual status change

```
rake commitments:change_status commitment_id=openai-gpt5-turbo-api \
  new_status=DELAYED evidence_id=<uuid> \
  note="OpenAI blog post indicates Q1 2027 target"
```

Must reference an evidence record — no status change without evidence (except `ANNOUNCED`, which is the initial state).

### Commitment editing

```
rake commitments:edit commitment_id=openai-gpt5-turbo-api \
  promise_text="GPT-5 Turbo available via API for Plus, Team, and Enterprise"
```

Direct field updates. All changes logged to `operator_audit_logs`.

### Merge commitments

```
rake commitments:merge source=openai-gpt5-api-duplicate target=openai-gpt5-turbo-api
```

Redirects old `commitment_id` to the canonical one. Evidence and status history from the source move to the target.

---

## Data Flow Summary

### On a major OpenAI launch day

1. Operator runs `rake events:create` to create the event
2. Operator runs `rake ingestion:extract` with the launch post URL
3. `IngestionExtractJob` runs LLM Prompt 1, writes candidates to `raw_release_items`
4. Operator opens `/operator/ingestion/review`, reviews 30–100 candidates
5. Accepted items become Commitments with status `ANNOUNCED`
6. For items that announced "available today", operator runs `rake evidence:match event_id=...`
7. `EvidenceMatchJob` runs LLM Prompt 2, writes candidate matches to review table
8. Operator opens `/operator/evidence/review`, accepts matches → status moves to `GA` or `PARTIAL`

**SLA:** Draft within 1–2h. Published (reviewed) within 24–48h.

### On a regular week

1. `rake openai:collect_release_notes` runs (scheduled or manual) — new entries land in `raw_release_items`
2. `rake openai:classify_release_notes` scores each item against the eligibility rubric
3. Items scoring >= 2 appear in `/operator/ingestion/review` — operator accepts/skips
4. `WeeklyRefreshJob` runs (cron) — evidence matching for all non-terminal commitments
5. New matches appear in `/operator/evidence/review` — operator accepts/rejects
6. Overdue alerts flagged for operator attention

**SLA:** Release notes auto-classified. Same-day review.

---

## Database Tables Involved

| Table | Role in pipeline |
|-------|-----------------|
| `events` | Created by operator (rake task). Links to announcements |
| `raw_release_items` | Staging table. LLM extraction output and release notes land here. Never public |
| `announcements` | Created on ingestion accept. Groups commitments within an event |
| `commitments` | Created on ingestion accept. Stable public slug. The core tracked unit |
| `evidence` | Created on evidence accept. Dated proof of delivery |
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
    "model_launch", "model_deprecation", "pricing_limits", "api_primitive",
    "tools_agents", "multimodal", "enterprise_admin", "marketplace_distribution",
    "safety_policy", "memory_personalization", "connectors_integrations",
    "apps_desktop_mobile"
  ]
}
```

---

## What to Build (ordered)

### Phase 1: Database + models
1. Migrations for all tables (`events`, `announcements`, `commitments`, `evidence`, `status_histories`, `raw_release_items`, `operator_audit_logs`)
2. ActiveRecord models with associations, validations, computed field methods
3. Status transition enforcement in `StatusHistory` (evidence required for non-initial)

### Phase 2: LLM integration
4. `lib/llm/extraction.rb` — Prompt 1 runner (Anthropic SDK)
5. `lib/llm/evidence_validation.rb` — Prompt 2 runner (Anthropic SDK)
6. `lib/llm/eligibility_classifier.rb` — rubric scoring

### Phase 3: Rake tasks (CLI triggers)
7. `rake events:create` — event CRUD
8. `rake ingestion:extract` — fetch URL + run LLM extraction + write to raw_release_items
9. `rake openai:collect_release_notes` — scrape release notes into raw_release_items
10. `rake openai:classify_release_notes` — score against eligibility rubric
11. `rake evidence:match` — run evidence matching for a scope
12. `rake evidence:add` — manual evidence creation
13. `rake commitments:change_status` — manual status change
14. `rake commitments:edit` — field updates
15. `rake commitments:merge` — duplicate resolution

### Phase 4: Background jobs
16. `IngestionExtractJob` — async LLM extraction
17. `EvidenceMatchJob` — async evidence validation
18. `WeeklyRefreshJob` — cron-triggered batch refresh

### Phase 5: Review queue web UI (the only UI that matters)
19. `/operator/ingestion/review` — ingestion review queue (list of cards, accept/skip/edit/merge)
20. `/operator/evidence/review` — evidence review queue (side-by-side commitment + evidence cards)
21. Operator auth gate (Devise + Pundit, role: operator)

### Phase 6: Verify done criteria
22. At least 2 Events ingested (1 named launch + 1 release-wave bucket)
23. >= 30 commitments with stable URLs
24. Every non-ANNOUNCED status has evidence + StatusHistory row
25. Company dashboard shows: unshipped list, recently changed statuses, time-to-GA stats
