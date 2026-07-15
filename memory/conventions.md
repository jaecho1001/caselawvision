# Conventions, standing rules, and gotchas

> Last verified: 2026-07-14 against `docs/ARCHITECTURE.md` and `docs/INTEGRATION_SCHEMA.md`.
> Add a rule when a verified mistake teaches one.

## Platform standards (authoritative in `docs/`)

- **Tenant isolation via `firm_{id}` schemas.** Validate `^firm_[a-z0-9_]+$`,
  `SET search_path TO firm_{id}` before queries, `psycopg2.sql.Identifier()` for quoting,
  parameterized (`%s`) queries only — never string-interpolate SQL.
- **Single writer per table.** Only the owning app writes its prefixed tables; other apps read
  via the `ix_*` layer.
- **App-owned tables under a registered prefix** (`lt_`, `mm_`, `ar_`, `ew_`, `ix_`). Never
  write another app's prefixed tables.
- **Integrate through `ix_*`, not direct cross-app reads.** Publish to `ix_sync_events` with an
  `idempotency_key`; consume by polling `WHERE id > last_event_id` and advancing
  `ix_sync_cursors`; resolve identity via `ix_cross_client_map`; surface activity through
  `ix_activity_feed` / `ix_unified_communications`. Consumers must be idempotent (at-least-once).
- **UUID PKs (`gen_random_uuid()`), `TIMESTAMPTZ NOT NULL DEFAULT now()`, snake_case columns,
  JSONB for flexible data** — for all tables.
- **New agent? Follow `docs/AGENT_PROTOCOL.md` → "Adding a New Agent"** (choose/register a
  prefix, migrate owned tables, implement `/agents/{type}/invoke`, add a client column to
  `ix_cross_client_map` if tracking clients, publish to `ix_sync_events`).

## Brain / workflow rules

- **This repo is docs + standards only** — no application code here; app changes go in each
  app's own repo.
- **Never overwrite `docs/` or existing brain files blindly** — read and merge.
- **Verify before you assert.** Cite the doc/file a fact came from; mark guesses `⚠️ ASSUMPTION`.
- **Keep brain commits separate:** `chore(brain): <what changed>`.
- **Explain every git action in plain English** for a non-engineer (a lawyer) — in the reply and
  mirrored in the commit body ("Legible changes" in `NORTH-STAR.md`).
- **Never commit secrets, tokens, or client/PII.** Names and boolean status only.
- **Dates `YYYY-MM-DD`; no emoji in new docs.**
- **Legal output is draft-only and human-approved.**

## Gotchas

- **Two overlapping bibles.** `caselawvision` and `caselawvision-platform` both claim to be the
  CaseLawVision platform bible and disagree on details (app status, `ix_*` table names, scope).
  When a platform fact matters, check both and prefer the one confirmed against deployed code;
  do not assume this repo is the single source of truth.
- **App status in `README.md` may be aspirational.** It marks four apps "Active" with public
  repo links, but `caselawvision-platform` marks some "Built/private" and EZWill "Planned".
  Confirm from each app's repo before asserting live status in that app's brain.
- **`ix_*` table-name drift.** This repo's DDL is the concrete v1.1 set
  (`ix_unified_communications`, `ix_shared_tasks`, `ix_external_system_links`, `ix_data_lineage`,
  `ix_sync_cursors`); other docs use shorter names. Use the DDL in `docs/INTEGRATION_SCHEMA.md`
  as the reference for this repo.
