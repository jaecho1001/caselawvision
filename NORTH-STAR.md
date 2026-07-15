# North Star

> The shared mission every project in the platform ladders up to.
> Last verified: 2026-07-14. Keep the shared-mission section identical across repos.

## Shared mission

Build agentic AI that scales law firms and makes legal service dramatically more
efficient — for lawyers and the clients they serve.

## What we're building

One connected legal platform where AI agents handle the work that slows firms down,
across four pillars:

1. **Legal delivery** — intake, document processing, timelines, drafting, review, filings.
2. **Marketing** — client acquisition, content, campaigns, SEO, reputation.
3. **Branding** — consistent firm voice, identity, and positioning across every touchpoint.
4. **Connected tools** — integrations that unify the firm's stack (email, calendar,
   docs, DMS, CRM, e-sign, billing) so agents act across systems, not silos.

## Who it serves

- **Firm owners / partners** — scale capacity and revenue without linear headcount.
- **Lawyers** — less admin, faster turnaround, more time on legal judgment.
- **Clients** — faster, clearer, more affordable service.

## How

Agentic AI is the connective layer: agents plan, use tools, and complete workflows
end-to-end, with humans in the loop wherever judgment and liability demand it.

## Principles

- **Verifiable outputs** — citations and provenance on anything that informs legal work.
- **Human-in-the-loop** for legal judgment, filings, and anything carrying liability.
- **Legible changes** — every change is explained in plain language a non-engineer
  stakeholder (a lawyer, a firm owner) can follow, git operations included. The
  humans in our loop are lawyers, not engineers; you cannot approve what you cannot
  understand, so plain-English legibility is what makes human-in-the-loop a real
  checkpoint instead of a rubber stamp. This is why every git action carries a
  plain-English explanation (see each repo's conventions).
- **Privacy and compliance by default** — never expose secrets, PII, or client data.
- **Every project ladders up** — if a repo does not advance a pillar, say why it exists.

## This repo's role

`caselawvision` is a **docs-only Platform Bible**: the canonical architecture, integration
schema, agent protocol, and app blueprints for the CaseLawVision multi-agent legal platform
(Vaturi & Cho LLP). It holds no application code — only `README.md` and four docs
(`docs/ARCHITECTURE.md`, `docs/AGENT_PROTOCOL.md`, `docs/INTEGRATION_SCHEMA.md`,
`docs/EZWILL.md`). It advances the **Connected tools** pillar (Pillar 4) most directly — the
`ix_*` integration layer, `firm_{id}` tenancy model, table-prefix registry, and the standard
`/agents/{type}/invoke` protocol are the connective backbone — and indirectly advances **Legal
delivery** (Pillar 1) through the domain apps it standardizes (Legal Timeline, MatterMail, AI
Reception, EZWill).

⚠️ **Overlap with `caselawvision-platform` — flagged as an Open Question (possible
consolidation).** A separate repo, `caselawvision-platform`, is *also* a "CaseLawVision
platform bible" covering the same mission, the same `ix_*` integration model, and the same app
roster. The two are not identical: this repo is leaner (one flat `docs/` with four files, a
full v1.1 `ix_*` DDL, a deep EZWill spec, and repo links marking all four apps **Active**),
while `caselawvision-platform` is richer (nested `docs/architecture|integration|standards|apps/`,
plus a reusable `brain-kit/` + `IMPLEMENTATION.md`) and marks EZWill "Planned" and adds
DivorceMate AI as a fifth app. They also disagree on app **status** and on some `ix_*` table
names. Which one is canonical is unresolved — see `memory/decisions.md` and the AGENTS.md Open
questions. Do not assume this repo is a strict duplicate; verify against both before asserting
platform facts downstream.
