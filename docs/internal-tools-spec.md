# Internal Tools Spec

Operator-facing tools for managing the Launch Scorecard data pipeline. These are not customer-facing — they support the ingestion, review, evidence matching, and refresh workflows described in the product spec.

The internal tools do not need to be polished. Function over form. The priority is: fast, correct, auditable.

---

## Principles

- **Human review is mandatory.** LLM-assisted extraction produces drafts; a human always reviews before publishing.
- **Every write is traceable.** Status changes require evidence. Evidence has sources. Operator actions are logged.
- **Minimal clicks for common paths.** The weekly refresh and post-keynote ingestion are the hot paths — optimize for those.

---

## Tool 1: Event Manager

Create and manage Events.

### Capabilities
- **Create event**: company, event name, type, dates, official hub URLs, video URLs, notes
- **Edit event**: update fields on an existing event
- **List events**: filter by company, date range, type
- **Link to UpcomingEvent** (v1.1): when an UpcomingEvent materializes, link it to the newly created Event

### v1 scope
Manual form. Fields map directly to the `Event` entity schema. No automation needed here — events are low-volume (a few per company per year).

---

## Tool 2: Ingestion Pipeline

Extract announcements and commitments from source material.

### Workflow
1. **Input**: operator pastes or provides URL to an official recap page, transcript, or blog post
2. **Fetch + clean**: pull page text (HTML → clean text, or accept pasted text)
3. **LLM extraction**: run the "split recap into atomic commitments" prompt (see product spec, Prompt 1) → returns structured JSON of announcements + commitments
4. **Review queue**: display extracted items for human review

### Review queue UI
For each extracted commitment, show:
- `promise_text` (editable)
- `product_area` (editable)
- `category` (dropdown)
- `availability_claim_raw` (editable)
- `target_window_start` / `target_window_end` (date pickers, nullable)
- `scope` fields: platform, region, tier, audience (editable chips/tags)
- `tags` (multi-select from canonical tag list)
- Source excerpt from the recap (read-only, for reference)

Operator actions per item:
- **Accept** → creates Announcement (if new) + Commitment + initial StatusHistory entry (status: ANNOUNCED) + Evidence record (type: BLOG_OFFICIAL or VIDEO_TRANSCRIPT, linking back to the source)
- **Edit + Accept** → same as above, with operator's edits applied
- **Skip** → item does not become a commitment (did not pass eligibility)
- **Merge** → combine with an existing commitment (e.g., duplicate from a different source)

### Bulk actions
- Accept all (with confirmation)
- Skip all remaining
- Set shared fields (e.g., all items from this source share the same event_id, source_url)

### SLA targets (from calendar doc)
| Event type          | Draft available | Published (reviewed) |
|---------------------|-----------------|----------------------|
| Major keynote       | Within 2h       | Within 24–48h        |
| Earnings call       | Within 4h       | Within 48h           |
| Product launch post | Within 1h       | Within 24h           |
| Weekly release notes| Auto-classified | Same-day review      |

---

## Tool 3: Evidence Matcher

Find and attach evidence to existing commitments.

### Workflow
1. **Select scope**: a single commitment, all commitments for an event, or all non-terminal commitments for a company
2. **Candidate sources**: search the company's shipping sources (release notes feeds, docs, changelogs, official blogs) for matches
3. **LLM validation**: for each candidate, run the "does this evidence prove shipping?" prompt (see product spec, Prompt 2) → returns decision, date, excerpt, reasoning
4. **Review queue**: display matches for human review

### Review queue UI
For each candidate match, show:
- Commitment: promise_text, current status, target window
- Evidence candidate: URL, published date, excerpt, evidence type
- LLM decision: suggested status change, reasoning bullets
- `scope_observed` (editable, for partial shipping)

Operator actions per match:
- **Accept** → creates Evidence record + StatusHistory entry (old_status → new_status, linked to evidence)
- **Edit + Accept** → same, with operator corrections (e.g., change suggested status from GA to PARTIAL, adjust scope_observed)
- **Reject** → no evidence record created; optionally log as "reviewed, not relevant"
- **Flag for later** → mark for re-check in next refresh cycle

### Confidence auto-adjustment
When accepting evidence:
- If evidence type is `THIRD_PARTY_VERIFICATION` only → cap confidence at 0.7
- If status set to `PARTIAL` → cap confidence at 0.8
- If evidence type is `RELEASE_NOTES` or `DOCS` → allow confidence up to 1.0
- Operator can override confidence manually

---

## Tool 4: Weekly Refresh

Batch process for updating non-terminal commitments.

### Workflow
1. **Generate refresh list**: all commitments where `status NOT IN [GA, CANCELLED, REPLACED]`
2. **Group by company**: run evidence matching (Tool 3) per company, using that company's shipping sources
3. **Dashboard**: show summary of refresh results
   - How many commitments checked
   - How many new evidence matches found
   - How many status changes pending review
   - How many now overdue (target_window_end < today, still not shipped)
4. **Review + publish**: operator reviews pending changes (same UI as Tool 3 review queue)

### Overdue alerts
After each refresh, flag commitments where:
- `overdue_days > 0` and status is still `ANNOUNCED` or `PREVIEW` (no progress)
- `overdue_days > 90` for any non-terminal status (long-stale)
- `confidence < 0.5` and status is `PARTIAL` (weak evidence for partial claim)

These are internal flags for operator attention, not customer-facing alerts (those are a separate feature).

---

## Tool 5: Commitment Editor

Direct CRUD for individual commitments, announcements, and evidence.

### Capabilities
- **Edit commitment**: all fields on the Commitment entity
- **Edit announcement**: title, category, source
- **Add evidence manually**: for cases where automated matching didn't find it but operator has the source
- **Change status manually**: must provide or link an evidence record (enforced — no status change without evidence, except ANNOUNCED which is the initial state)
- **Merge commitments**: when duplicates are found (redirects old commitment_id to the canonical one)
- **Archive commitment**: soft-delete for items that were created in error (distinct from CANCELLED, which is a real status)

### Audit log
All operator actions (create, edit, status change, merge, archive) are logged with:
- Operator ID
- Timestamp
- Action type
- Before/after values for changed fields

---

## Tool 6: Upcoming Events Calendar (v1.1 stub)

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

## Implementation notes

- Tools 1–5 can start as a simple admin UI or even CLI scripts. The review queue (Tools 2 and 3) is the most important piece to get right — it's the quality gate.
- The LLM prompts (Prompt 1 for extraction, Prompt 2 for evidence validation) are already defined in the product spec. The internal tools just need to wire them up with the review queue.
- Consider building Tools 2 and 3 as a single "Ingestion + Evidence" workflow, since post-keynote ingestion often immediately flows into evidence matching for "available today" commitments.
- All tools write to the same database as the public-facing product. There is no separate "staging" — the review step IS the staging gate.
