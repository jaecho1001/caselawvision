# AGENTS.md — CaseLawVision platform bible

> Canonical, tool-neutral instructions for this repository.
> Last updated: 2026-07-14.

This repo is a **CaseLawVision Platform Bible** — canonical architecture, integration schema,
agent protocol, and app blueprints for the multi-agent legal platform. It holds docs and
standards, **not application code**. Preserve true existing content and verify claims against
the docs/code before changing the brain. ⚠️ It overlaps with a second bible,
`caselawvision-platform`; see NORTH-STAR.md "This repo's role" and the Open questions below.

## Before every session

1. Read `NORTH-STAR.md`.
2. Read `memory/00-index.md`, then `memory/status.md`, then task-relevant memory files.
3. For platform architecture/standards, read `docs/` — start with `docs/ARCHITECTURE.md`,
   then `docs/INTEGRATION_SCHEMA.md`, `docs/AGENT_PROTOCOL.md`, and `docs/EZWILL.md`.
4. Run `git status --short --branch` and `git log --oneline -10` before substantive work.

## Mission summary

CaseLawVision (by Vaturi & Cho LLP) is a multi-agent AI platform for Canadian law firms.
Each practice area is a separate app with its own AI agent — Legal Timeline (case management),
MatterMail (email), AI Reception (voice/SMS intake), EZWill (Ontario estate planning) — all
sharing one PostgreSQL database with `firm_{id}` schema isolation and communicating through a
shared `ix_*` integration layer (event bus + cross-client identity map). The design target is
one orchestrator routing requests to domain agents via a standard `/agents/{type}/invoke`
protocol, with humans in the loop for legal judgment. All client-facing apps target EN/KO
bilingual operation for the Korean-Canadian community. This repo is the contract those apps
build against.

## Standing rules

- **This repo is docs + standards only.** Don't add app code here; app changes belong in each
  app's own repo (`legal-timeline`, `ai-reception`, `mattermail`, `ezwill`).
- **Never overwrite Bible docs blindly.** Read `docs/` and merge; the docs are the
  authoritative platform contract that other repos depend on.
- **Tenant isolation is sacred.** Every app uses `firm_{id}` schema isolation
  (`^firm_[a-z0-9_]+$`), `SET search_path`, `psycopg2.sql.Identifier()` quoting, and
  parameterized (`%s`) queries only — never string-interpolate SQL.
- **App-owned tables under a registered prefix** (`lt_`, `mm_`, `ar_`, `ew_`, `ix_`);
  single writer per table, cross-app reads go through the `ix_*` layer.
- **Legal output is draft-only and human-approved.** The humans in our loop are lawyers;
  every change carries a plain-English explanation ("Legible changes" in `NORTH-STAR.md`).
- **Never track secrets, tokens, or client/PII data.** Names and boolean status only.
- **Brain commits are separate** from any docs change: `chore(brain): <what>`.
- **Dates `YYYY-MM-DD`; no emoji in new docs.**

## Architecture at a glance

Each app is a separate repo/service deployed to **Google Cloud Run**, sharing one **Cloud SQL
PostgreSQL 16** instance with `firm_{id}` per-tenant schemas, **Redis** (Memorystore) for
sessions/JWT revocation, **Pub/Sub** for async events (`sync.*`, doc-ocr), and **GCS** for
documents. Auth is a shared `clv_token` JWT cookie carrying a `law_firm_id` claim. Cross-app
data sharing goes through 8 `ix_*` tables (cross-client identity map, sync event bus + cursors,
unified communications, shared tasks, external system links, activity feed, data lineage) via
cursor-polled, at-least-once sync. Table prefixes: `lt_` (Legal Timeline), `mm_` (MatterMail),
`ar_` (AI Reception), `ew_` (EZWill), `ix_` (integration). Full detail in
`memory/architecture.md` and `docs/`.

## Key "commands" (a docs-only repo)

| Task | How |
|------|-----|
| Understand the platform | Read `docs/ARCHITECTURE.md` |
| See the shared-table DDL | Read `docs/INTEGRATION_SCHEMA.md` (full v1.1 `ix_*` DDL) |
| Learn the agent invoke contract | Read `docs/AGENT_PROTOCOL.md` |
| Understand the EZWill app | Read `docs/EZWILL.md` |
| Add a new agent | Follow `docs/AGENT_PROTOCOL.md` → "Adding a New Agent" |

## Memory map

| File | Holds |
|------|-------|
| `NORTH-STAR.md` | shared platform mission and this repo's role |
| `memory/product.md` | problem, users, scope, pillar fit |
| `memory/architecture.md` | apps, `ix_*` integration, data model, environments, external services |
| `memory/decisions.md` | dated brain/bible decisions, newest first |
| `memory/glossary.md` | `ix_*` tables, prefixes, `firm_{id}`, app names |
| `memory/conventions.md` | database/tenancy standards, brain rules, gotchas |
| `memory/roadmap.md` | Now / Next / Later / Done by pillar |
| `memory/status.md` | concise current state; next step |
| `memory/learnings.md` | append-only improvement journal |

## Self-Improvement Protocol

Run once at the end of every substantive session:

1. **Reflect** — identify facts, decisions, or lessons not yet captured.
2. **Update status** — always update `memory/status.md`.
3. **Update only what changed** — decisions, conventions, architecture, glossary,
   roadmap, or product; append one dated learning.
4. **Improve the system when warranted** — tighten `brain-kit/` prompts or conventions.
5. **Keep the index honest** — bump dates in `memory/00-index.md` for touched files.
6. **Verify** — review diffs, run proportional checks, confirm no secret/PII entered.
7. **Save back** — commit brain-only changes separately as `chore(brain): <what changed>`.

Guardrails: prefer additive, reversible edits; mark genuine guesses `⚠️ ASSUMPTION`; ask
before changing mission, deleting a standing rule, or weakening legal/compliance safeguards.

## Open questions

- **Overlap / consolidation:** `caselawvision` and `caselawvision-platform` are both
  "CaseLawVision platform bibles." Which is canonical? Should they be consolidated, or is one a
  superset (`caselawvision-platform` adds `brain-kit/` + `IMPLEMENTATION.md` and a fifth app,
  DivorceMate)? Confirm before treating either as the single source of truth.
- **App status conflict:** this repo's `README.md` marks EZWill (and Legal Timeline, MatterMail,
  AI Reception) **Active** with public repo links; `caselawvision-platform` marks EZWill
  "Planned" and Legal Timeline/MatterMail/AI Reception "Built" with private repos. Which is
  current?
- **`ix_*` naming drift:** this repo's DDL uses `ix_unified_communications`, `ix_shared_tasks`,
  `ix_external_system_links`, `ix_data_lineage`, `ix_sync_cursors`; `caselawvision-platform`'s
  memory uses `ix_communications`, `ix_tasks`, `ix_external_links`, `ix_document_refs`,
  `ix_matter_links`. Which table set is deployed?
- **Orchestrator status:** the README shows a "CaseLawVision Orchestrator" routing to domain
  agents, but `AGENT_PROTOCOL.md` lists only the Will agent as Active (others Planned). Is the
  orchestrator built or design-stage?
