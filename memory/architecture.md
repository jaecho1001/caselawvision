# Architecture — CaseLawVision platform

> Last verified: 2026-07-14 against `docs/ARCHITECTURE.md`, `docs/INTEGRATION_SCHEMA.md`,
> `docs/AGENT_PROTOCOL.md`, `docs/EZWILL.md`, and `README.md`. This repo documents the
> platform; the `docs/` bible is authoritative.

## Shape

A suite of independent domain apps that share data and context through a common `ix_*`
integration layer on one PostgreSQL database. Each app owns its tables (single writer per
table); other apps read via the integration layer. Design target: an orchestrator routing
requests to domain agents through a standard `/agents/{type}/invoke` protocol.

## Applications

Ports and repos as stated in this repo's `README.md` and `docs/`.

| App | Repo (per README) | Stack | Port(s) | Prefix | Role |
|-----|-------------------|-------|---------|--------|------|
| Legal Timeline | `jaecho1001/legal-timeline` | Quarkus (Java) + Python + Angular | 8080 / 8000 / 4200 | `lt_` | Cases, documents, OCR/AI, timeline, KG graph, intakes |
| MatterMail | `jaecho1001/mattermail` | FastAPI (Python) | 8001 | `mm_` | Email sync, classify, forward, conflicts, tasks, digest |
| AI Reception | `jaecho1001/ai-reception` | FastAPI (Python) | 8002 | `ar_` | Voice/SMS/chat intake, fax, Korean NLP, extraction, handoff |
| EZWill | `jaecho1001/ezwill` | FastAPI + Next.js 16 | 8003 / 3000 | `ew_` | Ontario estate planning: wills, POA, affidavits |

⚠️ README marks all four **Active**; `caselawvision-platform` marks EZWill "Planned" and
Legal Timeline/MatterMail/AI Reception "Built" (private repos) and adds DivorceMate AI (`dm_`).
Confirm live status/repo names before asserting downstream (see Open questions in AGENTS.md).
Planned agents in the README: immigration (`im_`), family (`family_agent`/DivorceMateAI),
real estate (`re_`), corporate (`co_`).

## Integration layer (`ix_*` tables)

8 tables created in every `firm_{id}` tenant schema; all apps read and write. Full DDL is in
`docs/INTEGRATION_SCHEMA.md` (version 1.1, 2026-04-03).

| Table | Purpose |
|-------|---------|
| `ix_cross_client_map` | Links client IDs across apps (`lt_/mm_/ar_/ew_` refs + canonical name/email/phone; `match_method`, `match_confidence`) |
| `ix_sync_events` | Append-only event bus (`BIGSERIAL` id for cursor polling); e.g. `client.created`, `ew.will.submitted` |
| `ix_sync_cursors` | Per-consumer cursor tracking for at-least-once delivery |
| `ix_unified_communications` | Denormalized cross-channel comms log (email/phone/sms/fax/chat/video) |
| `ix_shared_tasks` | Unified task queue visible to all apps |
| `ix_external_system_links` | Maps entities to external systems (Twilio, GHL, Documo, etc.) |
| `ix_activity_feed` | Chronological "Client 360" activity timeline |
| `ix_data_lineage` | Provenance of data flowing between apps (sync/import/merge/reference/derive) |

Sync protocol: publishers `INSERT` into `ix_sync_events` with an `idempotency_key`
(unique-constrained); consumers poll `WHERE id > last_event_id` ordered ascending, then advance
`ix_sync_cursors`. Consumers must be idempotent (at-least-once). `pg_trgm` powers fuzzy name
matching; `pgcrypto` for `gen_random_uuid()`.

## Agent protocol

- **Invoke:** `POST /agents/{agent_type}/invoke` with `Authorization: Bearer <jwt>`,
  `X-Tenant: firm_{id}`, body `{capability, payload, correlation_id?, source_agent?}`.
- **Discovery:** `GET /agents/{agent_type}/capabilities`.
- **Registry:** `will` (EZWill) **Active**; `reception`, `mail`, `timeline` **Planned**.
- Agents can call each other; `correlation_id` ties a flow together and each step is captured
  in `ix_sync_events` for audit/cross-app visibility. (`docs/AGENT_PROTOCOL.md`.)

## Data model & tenancy

- One PostgreSQL 16 database; **per-tenant schema** `firm_{id}` (validated `^firm_[a-z0-9_]+$`;
  default dev tenant `firm_demo`). Each tenant schema holds ALL tables (every app + the `ix_*`
  layer).
- `public` schema: shared `users`, `invites`. `platform` schema: superuser admin, tenant
  registry, usage tracking.
- Auth: shared `clv_token` JWT cookie carrying a `law_firm_id` claim → `schema = "firm_" +
  law_firm_id`. Quarkus resolves the tenant via `CustomTenantResolver`; Python apps
  `SET search_path TO firm_{id}` via `psycopg2`.
- Conventions: UUID PKs via `gen_random_uuid()`; `TIMESTAMPTZ NOT NULL DEFAULT now()`;
  snake_case columns; JSONB for flexible data; parameterized (`%s`) queries only;
  `psycopg2.sql.Identifier()` for schema quoting.

## Environments

- **Development — Docker Compose:** `postgres:5432`, `redis:6379`, `pubsub:8085`;
  `quarkus:8080` + `python:8000` + `angular:4200` (Legal Timeline), `mattermail:8001`,
  `reception:8002`, `ezwill-api:8003` + `ezwill-web:3000`.
- **Production — Google Cloud Run:** one service per app (legal-timeline-api/-ai/-web,
  mattermail, ai-reception, ezwill-api, ezwill-web) on a shared VPC connector → Cloud SQL
  (PostgreSQL 16, `firm_{id}` schemas), Memorystore Redis 7, Pub/Sub topics/subscriptions, and
  a shared GCS bucket for documents.
- **Redis:** sessions + JWT revocation. **Pub/Sub topics:** `sync.*`, doc-ocr.

## External services

Twilio (Voice/SMS), Documo (inbound fax), GoHighLevel (CRM), WhatConverts (lead attribution),
Google (OAuth 2.0, GCS, Cloud Vision OCR, Pub/Sub), Dropbox + OneDrive (document sync).

## This repo's own structure

- `README.md` — platform overview, app table, conventions, practice-area/agent map.
- `docs/ARCHITECTURE.md` — topology, tenancy model, app ownership, sync flows, DB conventions.
- `docs/INTEGRATION_SCHEMA.md` — full v1.1 `ix_*` DDL + sync-bus protocol.
- `docs/AGENT_PROTOCOL.md` — invoke/discovery contract, agent registry, capability specs.
- `docs/EZWILL.md` — EZWill blueprint (three-portal system, tier system, clause library, AI
  flagging, Ontario SLRA/SDA foundations, `ew_*` tables, API routes).
