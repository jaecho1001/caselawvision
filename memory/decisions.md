# Decision Log

> Last updated: 2026-07-14. Newest first. Platform-architecture decisions live in `docs/`;
> this log captures brain/bible-level choices and points to the docs for the rest.

## 2026-07-14 — Adopt the shared, self-improving agent brain

- **Decision:** add `NORTH-STAR.md`, `AGENTS.md`, a pointer `CLAUDE.md`, and `memory/`, on top
  of the existing bible (`README.md` + `docs/`), without altering the docs.
- **Why:** every Codex/Claude session should start already knowing the platform; the brain is
  additive documentation, the docs stay authoritative for the technical contract.
- **Consequence:** substantive sessions run the Self-Improvement Protocol and commit brain-only
  changes as `chore(brain): …`.

## 2026-07-14 — Flag the overlap with `caselawvision-platform` (do not assume duplicate)

- **Decision:** record the overlap between `caselawvision` and `caselawvision-platform` as an
  explicit Open Question (possible consolidation) rather than treating either as canonical.
- **Why:** both repos are "CaseLawVision platform bibles" for the same mission, `ix_*` model, and
  app roster, but they diverge on app status (this repo: EZWill/Legal Timeline/MatterMail/AI
  Reception all **Active**; `caselawvision-platform`: EZWill "Planned", the others "Built"), on
  `ix_*` table names, and on scope (`caselawvision-platform` adds `brain-kit/` +
  `IMPLEMENTATION.md` and a fifth app, DivorceMate). Asserting one as truth would risk
  propagating stale facts downstream.
- **Consequence:** downstream brains must verify against both; a human decision on
  which-is-canonical (or a merge) is pending.

## (from the docs) Platform architecture decisions

- **Per-tenant `firm_{id}` schema isolation** on one shared PostgreSQL 16, with all app tables
  and the `ix_*` layer inside each tenant schema — see `docs/ARCHITECTURE.md` §3.
- **Single writer per table + `ix_*` integration layer** (cross-client identity map, event bus
  with cursor-polled at-least-once sync) instead of direct cross-app reads — see
  `docs/ARCHITECTURE.md` §5 and `docs/INTEGRATION_SCHEMA.md`.
- **Standard `/agents/{type}/invoke` protocol** with `correlation_id` tracing and event capture
  in `ix_sync_events` — see `docs/AGENT_PROTOCOL.md`.
- **Google Cloud Run + Cloud SQL + Memorystore + Pub/Sub + GCS deployment topology** — see
  `docs/ARCHITECTURE.md` §8. ⚠️ ASSUMPTION: production topology is described as designed;
  confirm live-deployment status.
- **EZWill grounded in Ontario statute (SLRA, SDA)**, Tier 1/Tier 2 workflow, draft-first and
  lawyer-reviewed — see `docs/EZWILL.md`.
