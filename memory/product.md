# Product — CaseLawVision platform bible

> Last verified: 2026-07-14 against `README.md` and `docs/`.

## Problem

Canadian law firms lose time to fragmented tools: intake, email, documents, case timelines,
and estate/will work live in silos that don't share context. A single client can have a case
file, an email thread, an intake call, and a will draft — as four disconnected records.

## What this platform does

CaseLawVision (Vaturi & Cho LLP) is a multi-agent AI platform for Canadian law firms. Each
practice area is an independent app with its own AI agent, unified by a shared PostgreSQL
database + `ix_*` integration layer, moving toward one orchestrator that routes requests to
domain agents via a standard `/agents/{type}/invoke` protocol, with human oversight for legal
judgment. Client-facing apps target EN/KO bilingual operation for the Korean-Canadian community.

## This repo's job

`caselawvision` is a **docs-only Platform Bible**: the canonical architecture, `ix_*`
integration schema (full v1.1 DDL), agent protocol, and app blueprints every app reads before
development. It ships docs and standards, not application code.

⚠️ It overlaps heavily with a second bible, `caselawvision-platform`. Consolidation vs.
which-is-canonical is an open question — see `decisions.md` and AGENTS.md open questions.

## Users

- **Firm owners / lawyers (Vaturi & Cho LLP first)** — the platform's operators and primary users.
- **Their clients** — served through the domain apps (intake, email, case timelines, estate/wills);
  Korean-Canadian clients served bilingually (EN/KO).
- **AI agents / developers** — read this repo to build platform-consistent apps and agents.

## Scope

- **In scope:** platform architecture, tenancy model, the 8 `ix_*` shared tables, the agent
  invoke/discovery protocol, and per-app blueprints (EZWill documented in depth here).
- **Out of scope:** application source code (lives in each app's own repo), running services,
  and infrastructure provisioning.

## Pillar fit

Primarily **Connected tools** (Pillar 4): the shared tenancy model, the `ix_*` integration
layer, the table-prefix registry, and the standard agent protocol are the connective backbone.
It indirectly advances **Legal delivery** (Pillar 1) by standardizing the domain apps (Legal
Timeline, MatterMail, AI Reception, EZWill) that actually deliver legal work.

## Success

Every app in the ecosystem builds against the same tenancy, integration, and agent-protocol
standards; a new agent can be added by following `docs/AGENT_PROTOCOL.md`; and the `ix_*` DDL
in `docs/INTEGRATION_SCHEMA.md` is the single reference for cross-app data sharing.

## Constraints

Legal work carries liability — human-in-the-loop for legal judgment; tenant isolation and
privacy by default; verifiable outputs. EZWill work is grounded in Ontario statute
(SLRA, SDA) and is draft-first / lawyer-reviewed.
