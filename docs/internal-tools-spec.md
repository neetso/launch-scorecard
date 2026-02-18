# Internal Tools Spec

Operator-facing tools for managing the Launch Scorecard data pipeline. These are not customer-facing — they support the ingestion, review, evidence collection, and weekly review workflows described in the product spec and pipeline design doc.

The internal tools do not need to be polished. Function over form. The priority is: fast, correct, auditable.

---

## Principles

- **Human review is mandatory.** LLM-assisted extraction produces drafts; a human always reviews before publishing.
- **Every write is traceable.** Status changes require evidence. Evidence has sources. Operator actions are logged.
- **Minimal clicks for common paths.** The post-keynote ingestion review queue is the hot path — optimize for that.

---

## Tool 1: Event Manager

Create and manage Events.

### Capabilities
- **Create event**: company, event name, type, dates, official hub URLs, video URLs, notes
- **Edit event**: update fields on an existing event
- **List events**: filter by company, date range, type
- **Link to UpcomingEvent** (v1.1): when an UpcomingEvent materializes, link it to the newly created Event

### v1 scope
Manual form or rake task. Fields map directly to the `Event` entity schema. No automation needed here — events are low-volume (a few per company per year).

---

## Tool 2: Ingestion Pipeline

Extract commitments from source material.

### Workflow
1. **Input**: operator pastes or provides URL to an official recap page, transcript, or blog post
2. **Fetch + clean**: pull page text (HTML → clean text, or accept pasted text)
3. **LLM extraction**: run the "split recap into atomic commitments" prompt (see product spec, Prompt 1) → returns structured JSON of commitments
4. **Review queue**: display extracted items for human review

### Review queue UI
For each extracted commitment, show:
- `promise_text` (editable)
- `product_area` (editable)
- `category` (dropdown)
- `availability_claim_raw` (editable)
- `target_window_start` / `target_window_end` (date pickers, nullable)
- `scope_notes` (editable text — free-text description of platform/region/tier constraints)
- Source excerpt from the recap (read-only, for reference)

Operator actions per item:
- **Accept** → creates Commitment (linked to event_id) + initial StatusHistory entry (status: ANNOUNCED) + Evidence record (type: BLOG_OFFICIAL or VIDEO_TRANSCRIPT, linking back to the source)
- **Edit + Accept** → same as above, with operator's edits applied
- **Skip** → item does not become a commitment (did not pass eligibility)

### Implementation notes
- Server-rendered cards using Hotwire (Turbo Frames).
- Accept/Skip buttons submit via Turbo Stream — card removed without page reload.
- Edit mode: clicking "Edit" expands the card inline. "Save + Accept" submits.

### SLA targets (from calendar doc)
| Event type          | Draft available | Published (reviewed) |
|---------------------|-----------------|----------------------|
| Major keynote       | Within 2h       | Within 24–48h        |
| Earnings call       | Within 4h       | Within 48h           |
| Product launch post | Within 1h       | Within 24h           |
| Weekly release notes| Operator-reviewed | Same-day            |

---

## Tool 3: Evidence & Status Manager

Manually add evidence and update statuses for existing commitments. This is the primary tool for the weekly review cycle.

### Capabilities
- **Add evidence**: operator provides URL, evidence type, published date, excerpt, and the status it supports. Creates Evidence record + StatusHistory entry.
- **Change status**: must provide or link an evidence record (enforced — no status change without evidence, except ANNOUNCED which is the initial state).
- **Edit commitment**: all fields on the Commitment entity. All changes logged to `operator_audit_logs`.

### v1 scope
Rake tasks (CLI). No UI needed — these are low-volume operations (a few per week during the weekly review cycle).

```
# Add evidence
rake evidence:add commitment_id=openai-gpt5-turbo-api \
  url="https://..." type=DOCS published_at=2026-10-01 \
  excerpt="GPT-5 Turbo is available..." supports_status=GA

# Change status
rake commitments:change_status commitment_id=openai-gpt5-turbo-api \
  new_status=DELAYED evidence_id=<uuid> \
  note="Blog indicates Q1 2027 target"

# Edit commitment
rake commitments:edit commitment_id=openai-gpt5-turbo-api \
  promise_text="GPT-5 Turbo available via API for Plus, Team, and Enterprise"
```

### Audit log
All operator actions (create, edit, status change) are logged with:
- Operator ID
- Timestamp
- Action type
- Before/after values for changed fields

---

## Tool 4: Upcoming Events Calendar (v1.1 stub)

**Not in v1 scope.** Stub for when we build the forward-looking calendar.

### Planned capabilities
- **CRUD for UpcomingEvent** records (see product spec for entity schema)
- **Seed from sources**: quarterly import from IR calendars (earnings dates), official event pages (conference dates)
- **Monitor for surprises**: RSS/feed watch on company newsrooms for unscheduled launch events
- **Promote to Event**: when an upcoming event happens, create the actual Event entity and link it (`linked_event_id`)
- **User-facing calendar view**: cross-company timeline of upcoming events (part of the product UI, spec TBD)

### Why deferred
The core v1 value is the retrospective ledger (what was promised, did it ship). The forward-looking calendar is a distinct feature surface that adds sourcing complexity, a new UI, and date-confidence tracking. Build it once the ledger is proven and there's demand for "what's next."

---

## Descoped from v1

The following tools from the original design were descoped to reduce pipeline complexity. See the pipeline design doc and product spec for full rationale.

| Original tool | What it did | Replaced by |
|---------------|-------------|-------------|
| **Evidence Matcher** (Tool 3 in original) | Automated search of shipping sources + LLM validation (Prompt 2) + evidence review queue | Manual evidence add via rake tasks (Tool 3 above) |
| **Weekly Refresh** (Tool 4 in original) | Cron job re-running evidence matching for all non-terminal commitments | Manual weekly review by operator |
| **Merge operation** | Duplicate resolution: redirect old commitment_id, move evidence/history | Manual handling — operator skips duplicates during review |
| **Bulk review actions** | "Accept all", "Skip all", "Set shared fields" in review queue | Per-item review. Add if batch sizes regularly exceed 100. |

---

## Implementation notes

- Tools 1–3 start as rake tasks. The review queue (Tool 2) is the most important piece to get right — it's the quality gate.
- The LLM prompt (Prompt 1 for extraction) is defined in the product spec. The internal tools wire it up with the review queue.
- All tools write to the same database as the public-facing product. There is no separate "staging" — the review step IS the staging gate.
