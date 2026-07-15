# Glossary — CaseLawVision platform

> Last verified: 2026-07-14. Terms and entities with where they're defined.

- **Platform Bible** — this repo (`caselawvision`); the canonical architecture, integration
  schema, agent protocol, and app blueprints (`README.md`, `docs/`). ⚠️ A second bible,
  `caselawvision-platform`, overlaps it; consolidation is open (see `decisions.md`).
- **Orchestrator** — the design-target router that dispatches requests to domain agents
  (`README.md`; `docs/ARCHITECTURE.md` §6). ⚠️ ASSUMPTION: design-stage; `AGENT_PROTOCOL.md`
  lists only the Will agent as Active.
- **`firm_{id}` schema** — per-tenant PostgreSQL schema (one law firm = one schema), validated
  `^firm_[a-z0-9_]+$`; default dev tenant `firm_demo` (`docs/ARCHITECTURE.md` §3).
- **`public` schema** — shared `users` and `invites` tables (`docs/ARCHITECTURE.md` §3).
- **`platform` schema** — superuser admin, tenant registry, usage tracking (`docs/ARCHITECTURE.md` §3).
- **`clv_token`** — shared JWT cookie carrying a `law_firm_id` claim used to derive the tenant
  schema (`docs/ARCHITECTURE.md` §3, §7).
- **`ix_*` tables** — the 8-table integration layer shared across apps (see
  `memory/architecture.md`; full DDL in `docs/INTEGRATION_SCHEMA.md`).
- **Table prefix** — each app owns tables under a registered prefix: `lt_` Legal Timeline,
  `mm_` MatterMail, `ar_` AI Reception, `ew_` EZWill, `ix_` integration
  (`docs/ARCHITECTURE.md` §4).
- **Apps** — Legal Timeline (case mgmt), MatterMail (email), AI Reception (voice/SMS intake),
  EZWill (Ontario estate planning) (`README.md`, `docs/`).
- **Sync consumer** — per-app poller of `ix_sync_events` using an `ix_sync_cursors` cursor
  (cursor-based, at-least-once) (`docs/INTEGRATION_SCHEMA.md`).
- **Agent invoke contract** — `POST /agents/{type}/invoke` with `{capability, payload,
  correlation_id?, source_agent?}` (`docs/AGENT_PROTOCOL.md`).
- **Will Agent** — the EZWill agent (`/agents/will/invoke`); capabilities `draft_will`,
  `get_draft_status`, `run_ai_flags` (`docs/AGENT_PROTOCOL.md`, `docs/EZWILL.md`).
- **Tier 1 / Tier 2 (EZWill)** — simple template-driven will vs. lawyer-customized complex will
  using the 60+ clause library (`docs/EZWILL.md`).
- **SLRA / SDA** — Ontario Succession Law Reform Act / Substitute Decisions Act; EZWill's legal
  foundations (`docs/EZWILL.md`).
- **Agent brain** — the `NORTH-STAR/AGENTS/CLAUDE/memory` documentation layer that onboards AI
  agents to this repo; distinct from the `ix_*` integration layer.
