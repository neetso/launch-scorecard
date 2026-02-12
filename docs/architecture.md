# Launch Scorecard — Architecture

## Overview

Rails monolith, API-first, PostgreSQL. The public site, operator tools, and future data feed API are all consumers of the same Rails API layer.

---

## Tech Stack

| Layer | Choice | Why |
|---|---|---|
| Framework | Rails 8 (API + views) | Relational CRUD, background jobs, admin tools, SSR by default |
| Database | PostgreSQL | Relational schema with FK chains, JSON columns for scope objects, append-only ledger |
| Background jobs | Solid Queue (Rails 8 built-in) | LLM extraction, evidence matching, weekly refresh — no external dependency |
| Frontend | Hotwire (Turbo + Stimulus) + Tailwind | Server-rendered HTML, partial page updates for filters/sort, minimal JS |
| Auth | Devise + Pundit | Three-tier access (anonymous, authenticated, operator) with policy objects |
| ORM | ActiveRecord | Native Rails; migrations, associations, validations, scopes |
| LLM | Anthropic SDK (Ruby) | Extraction prompts, evidence validation — called from background jobs |

---

## Application Structure

```
app/
├── controllers/
│   ├── api/v1/          # JSON API — all business logic flows through here
│   │   ├── events_controller.rb
│   │   ├── commitments_controller.rb
│   │   ├── evidence_controller.rb
│   │   └── ...
│   ├── pages/           # Public HTML views (consume API layer internally)
│   │   ├── scorecards_controller.rb
│   │   ├── dashboards_controller.rb
│   │   └── commitments_controller.rb
│   └── operator/        # Operator tools (auth-gated, consume same API layer)
│       ├── events_controller.rb
│       ├── ingestion_controller.rb
│       ├── review_controller.rb
│       └── refresh_controller.rb
├── models/              # Domain logic, validations, associations, computed fields
├── policies/            # Pundit policies (anonymous / authenticated / operator)
├── services/            # Business operations (status transitions, evidence attachment)
├── jobs/                # Solid Queue jobs (LLM extraction, evidence matching, refresh)
├── views/
│   ├── pages/           # Public Hotwire views
│   ├── operator/        # Operator Hotwire views
│   └── api/             # Jbuilder or serializers (if needed beyond `render json:`)
└── lib/
    └── llm/             # LLM prompt runners (extraction, validation)
```

### Key principle

Controllers are thin. Business logic lives in models and services. The `pages/` and `operator/` controllers call the same service layer that the `api/v1/` controllers do — they just render HTML instead of JSON.

---

## API Design

### Public endpoints (anonymous or authenticated)

```
GET /api/v1/events
GET /api/v1/events/:event_id
GET /api/v1/events/:event_id/commitments    ?status=&category=&q=&sort=&page=
GET /api/v1/companies/:company
GET /api/v1/companies/:company/commitments   ?status=&sort=&page=
GET /api/v1/companies/:company/changes       ?since=7d|30d|all&type=upgrades|slips|new
GET /api/v1/commitments/:commitment_id
GET /api/v1/commitments/:commitment_id/evidence
GET /api/v1/commitments/:commitment_id/history
```

### Operator endpoints (role-gated)

```
POST   /api/v1/operator/events
PATCH  /api/v1/operator/events/:event_id
POST   /api/v1/operator/ingestion/extract     # submit URL/text → kick off LLM job
GET    /api/v1/operator/ingestion/review       # pending extracted items
POST   /api/v1/operator/ingestion/accept       # accept/edit/skip/merge items
POST   /api/v1/operator/evidence/match         # run evidence matching for scope
GET    /api/v1/operator/evidence/review        # pending evidence matches
POST   /api/v1/operator/evidence/accept        # accept/edit/reject matches
POST   /api/v1/operator/refresh                # trigger weekly refresh
PATCH  /api/v1/operator/commitments/:id
POST   /api/v1/operator/commitments/:id/evidence   # manual evidence add
POST   /api/v1/operator/commitments/:id/status      # manual status change (requires evidence)
```

### Response conventions

- All list endpoints support `?page=` and `?per_page=` (default 50)
- Filter/sort params mirror URL query params on public pages
- Computed fields (ship_rate, overdue_days, days_to_ga, evidence_strength) are included in responses — never computed client-side
- Anonymous requests return gated data (e.g., first 5 commitments for an event, counts only for the rest)

---

## Database Schema

Follows the product spec entity definitions directly. Key implementation notes:

### Tables

- `events` — slug-based `event_id` as primary key
- `announcements` — UUID primary key, FK to events
- `commitments` — slug-based `commitment_id` as primary key (stable public URLs)
- `evidence` — UUID primary key, FK to commitments
- `status_histories` — UUID primary key, FK to commitments + evidence. Append-only (no updates or deletes).
- `raw_release_items` — staging table for ingestion pipeline. Not exposed publicly.
- `operator_audit_logs` — all operator writes logged with before/after values.
- `users` — Devise-managed. `role` enum: `user`, `operator`.

### JSON columns

- `commitments.scope` — `{ platform: [], region: [], tier: [], audience: [], notes: "" }`
- `evidence.scope_observed` — same structure

### Computed fields

Implemented as model methods (or database views if query performance demands it):

```ruby
# Commitment model
def overdue_days ...    # see spec: Computed Fields section
def days_to_ga ...      # see spec
def evidence_strength . # see spec: Strong / Medium / Weak / None

# Event / Company scope
def ship_rate ...       # see spec
def median_days_to_ga . # see spec
```

These are serialized into API responses. If list queries become slow, materialize as database columns refreshed on write.

### Indexes

- `commitments`: index on `(event_id, status)`, `(announcement_id)`, `(product_area)`, full-text on `promise_text`
- `evidence`: index on `(commitment_id, evidence_type)`
- `status_histories`: index on `(commitment_id, changed_at)`
- `events`: index on `(company, start_date)`

---

## Auth Model

Three tiers, single `users` table:

| Tier | Access | How |
|---|---|---|
| Anonymous | Public API, gated data (partial rows, no export) | No session |
| Authenticated | Full read access, CSV export, watchlists, alerts | Devise session or API token |
| Operator | All writes: ingestion, evidence, status changes, CRUD | `role: operator` + Pundit policy |

API authentication via session cookie (for browser) or Bearer token (for API consumers / future enterprise tier).

---

## Background Jobs

All async work runs through Solid Queue.

| Job | Trigger | What it does |
|---|---|---|
| `IngestionExtractJob` | Operator submits URL/text | Fetch + clean text → LLM extraction (Prompt 1) → write to review queue |
| `EvidenceMatchJob` | Operator selects scope | Search shipping sources → LLM validation (Prompt 2) → write to review queue |
| `WeeklyRefreshJob` | Scheduled (cron) or manual | Run EvidenceMatchJob for all non-terminal commitments, grouped by company |
| `ExportJob` | User requests CSV | Generate CSV in background → serve download link |

Jobs write draft results to the database (review queue tables). Nothing is published without operator review.

---

## Frontend Approach

### Public pages (Hotwire + Tailwind)

- **Scorecard table**: Turbo Frame wrapping the table body. Filter/sort changes submit as GET params → server re-renders the frame. URL updates via Turbo Drive. No client-side state management needed.
- **KPI chips**: rendered server-side from computed fields.
- **Status pills, evidence badges, overdue chips**: Tailwind-styled HTML components (partials).
- **Logged-out gate**: server decides how many rows to render. Blur is CSS on the remaining placeholder rows.
- **Methodology popover**: Stimulus controller for show/hide.
- **Copy citation / copy evidence**: Stimulus controller that writes to clipboard.

### Operator tools

- Standard Rails forms with Turbo for inline updates.
- Review queue: list of cards, each with accept/edit/skip/merge buttons. Turbo Streams for removing reviewed items without full page reload.
- No need for high polish — function over form (per internal tools spec).

### Shared UI components (partials)

- `_status_pill.html.erb`
- `_evidence_badge.html.erb`
- `_overdue_chip.html.erb`
- `_scope_chips.html.erb`
- `_confidence_bar.html.erb`
- `_kpi_chip.html.erb`
- `_methodology_popover.html.erb`

---

## Deployment

| Component | Target |
|---|---|
| Rails app | Render, Fly.io, or Railway (single web process + worker process) |
| PostgreSQL | Managed instance from same provider (or Neon for serverless) |
| Solid Queue | Runs as a separate process in the same deployment (`bin/jobs`) |
| Assets | Propshaft (Rails 8 default) + CDN via provider |
| DNS/SSL | Provider-managed or Cloudflare |

Single-region to start. No Redis needed (Solid Queue uses PostgreSQL as its backend).

---

## What We're Skipping in v1

- Redis / caching layer (data volume is small)
- Full-text search engine (PostgreSQL `tsvector` is sufficient)
- Separate frontend deployment
- GraphQL
- WebSockets / real-time updates
- Multi-region / read replicas
- Rate limiting (add when API tier launches)
