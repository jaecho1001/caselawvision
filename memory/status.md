# Status — where we left off

> Last updated: 2026-07-14.

- **Branch:** `add-agent-brain` (PR to `main`). This repo previously held only the bible
  (`README.md` + `docs/`, a single "Initial commit: CaseLawVision platform bible"); this change
  adds the agent brain on top.
- **Done this change:**
  - Added the agent brain: `NORTH-STAR.md` (shared mission + this repo's bible role),
    `AGENTS.md`, pointer `CLAUDE.md`, and `memory/` populated from `README.md` + `docs/`.
  - Preserved the existing `docs/` bible and `README.md` untouched.
- **Verified from the docs:** the 8 `ix_*` tables + full v1.1 DDL (`docs/INTEGRATION_SCHEMA.md`),
  the four apps and their ports/prefixes (`README.md`, `docs/ARCHITECTURE.md`), the
  `firm_{id}` tenancy model + `clv_token` auth, the `/agents/{type}/invoke` protocol
  (`docs/AGENT_PROTOCOL.md`), and the EZWill blueprint (`docs/EZWILL.md`).
- **Next step:** merge this PR, then get a human decision on the overlap with
  `caselawvision-platform` (canonical vs. consolidate) and reconcile the `ix_*` table-name drift.
- **Open questions:** which bible is canonical (`caselawvision` vs. `caselawvision-platform`);
  true app status (README marks four "Active"); is the orchestrator built or design-stage;
  which `ix_*` table set is actually deployed. (Full list in AGENTS.md.)
- **Guardrail:** legal output is draft-only and human-approved; never overwrite `docs/` blindly;
  this repo is docs-only (no app code).
