# Roadmap — CaseLawVision platform bible

> Last updated: 2026-07-14. This bible repo's roadmap is about the platform contract (the
> `ix_*` layer, tenancy model, agent protocol, app blueprints), not app features — those live
> in each app's own roadmap.

## Connected tools (primary — the integration backbone)

- **Now:** keep the bible authoritative and current; the `ix_*` v1.1 DDL, tenancy model, and
  agent protocol are the source of truth every app depends on.
- **Next:** resolve the overlap with `caselawvision-platform` — decide which bible is canonical
  (or consolidate), and reconcile the `ix_*` table-name drift so one table set is authoritative.
- **Later:** a live cross-client identity + activity feed proven across at least two apps
  (e.g. AI Reception → EZWill handoff writing to `ix_sync_events` + `ix_cross_client_map`).
- **Done:** documented `ix_*` schema (full v1.1 DDL), sync event-bus protocol, table-prefix map,
  tenancy model, and DB conventions.

## Agent layer

- **Now:** the standard `/agents/{type}/invoke` protocol documented, with the Will agent (EZWill)
  Active.
- **Next:** implement the Planned agents — `reception`, `mail`, `timeline` — against the same
  protocol; wire cross-agent handoffs (e.g. reception → will `draft_will`).
- **Later:** the orchestrator moving from README design to a running router; add future
  practice-area agents (immigration, family/DivorceMate, real estate, corporate).
- **Done:** invoke/discovery contract, agent registry, and per-agent capability specs documented.

## Legal delivery / Marketing / Branding

- Legal delivery is delivered by the apps (Legal Timeline, MatterMail, AI Reception, EZWill),
  not this repo; this bible standardizes how they connect. Marketing/Branding is out of scope
  here (the `contentmachine` repo is that engine in the wider platform).

## Critical path

`Bible authoritative → resolve overlap with caselawvision-platform → implement Planned agents →
prove cross-app sync across two apps → orchestrator live.` Legal output stays human-approved at
every phase.
