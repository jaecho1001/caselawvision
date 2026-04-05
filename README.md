# CaseLawVision — Multi-Agent AI Platform for Canadian Law Firms

A comprehensive AI platform for Canadian law firms, powered by a multi-agent orchestration architecture. Each practice area has its own specialized AI agent, all sharing a common integration layer.

## Platform Architecture

```
┌─────────────────────────────────────────────────┐
│              CaseLawVision Orchestrator          │
│         (routes requests to domain agents)       │
└──────────┬──────────┬──────────┬────────────────┘
           │          │          │
    ┌──────▼──┐ ┌─────▼────┐ ┌──▼──────────┐
    │  Legal   │ │  Matter  │ │   EZWill    │
    │ Timeline │ │   Mail   │ │  (Estate)   │  ... more agents
    │  Agent   │ │  Agent   │ │   Agent     │
    └──────┬──┘ └─────┬────┘ └──┬──────────┘
           │          │          │
    ┌──────▼──────────▼──────────▼────────────┐
    │      ix_* Integration Layer (PostgreSQL) │
    │   ix_cross_client_map, ix_sync_events,   │
    │   ix_communications, ix_tasks, etc.      │
    └─────────────────────────────────────────┘
```

## Applications

| App | Port | Stack | Status | Repo |
|-----|------|-------|--------|------|
| **Legal Timeline** | :8080/:8000/:4200 | Quarkus + Python + Angular | Active | [legal-timeline](https://github.com/jaecho1001/legal-timeline) |
| **AI Reception** | :8002 | FastAPI (Python) | Active | [ai-reception](https://github.com/jaecho1001/ai-reception) |
| **MatterMail** | :8001 | FastAPI (Python) | Active | [mattermail](https://github.com/jaecho1001/mattermail) |
| **EZWill** | :8003/:3000 | FastAPI + Next.js 16 | Active | [ezwill](https://github.com/jaecho1001/ezwill) |

## Shared Infrastructure

- **PostgreSQL 16** with `firm_{id}` schema isolation
- **Integration Layer**: 8 `ix_*` tables for cross-app sync
- **Event Bus**: Cursor-based polling on `ix_sync_events`
- **Client Identity**: `ix_cross_client_map` links client IDs across all apps

## Key Conventions

- UUID primary keys with `gen_random_uuid()`
- `TIMESTAMPTZ NOT NULL DEFAULT now()` timestamps
- snake_case column names
- JSONB for flexible metadata
- Parameterized queries only (`%s`, never string interpolation)
- Tenant validation: `^firm_[a-z0-9_]+$`

## Practice Areas (Current + Planned)

| Area | Agent | App |
|------|-------|-----|
| Estate Planning | `will_agent` | EZWill |
| Client Intake | `reception_agent` | AI Reception |
| Document Management | `mail_agent` | MatterMail |
| Litigation Timeline | `timeline_agent` | Legal Timeline |
| Immigration (planned) | `immigration_agent` | — |
| Family Law (planned) | `family_agent` | DivorceMateAI |
| Real Estate (planned) | `realestate_agent` | — |
| Corporate (planned) | `corporate_agent` | — |

## Bilingual Support

All client-facing apps support EN/KO (English/Korean) bilingual operation, targeting Korean-Canadian community legal services.

## License

Proprietary — Vaturi & Cho LLP
