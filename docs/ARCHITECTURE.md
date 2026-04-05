# CaseLawVision Platform Architecture

## 1. Overview

CaseLawVision is a multi-agent AI platform for Canadian law firms. Each practice area (litigation timelines, email management, client intake, estate planning) is handled by an independent application with its own AI agent. All apps share a PostgreSQL database with schema isolation and communicate through a shared integration layer.

### Design Principles

- **Independent apps, shared data**: Each app owns its tables and can be deployed/scaled independently.
- **Single writer per table**: Only the owning app writes to its tables. Other apps read via the integration layer.
- **Schema isolation**: Every law firm tenant gets a dedicated PostgreSQL schema (`firm_{id}`).
- **Event-driven sync**: Apps publish events to `ix_sync_events`; other apps poll with cursors for at-least-once delivery.
- **Cross-app identity**: `ix_cross_client_map` links client records across all apps by phone, email, or fuzzy name match.

## 2. System Topology

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CaseLawVision Platform                              │
│                                                                             │
│   ┌───────────────┐  ┌───────────────┐  ┌───────────────┐  ┌───────────┐  │
│   │ LEGAL TIMELINE│  │  MATTERMAIL   │  │ AI RECEPTION  │  │  EZWILL   │  │
│   │  Quarkus:8080 │  │ FastAPI:8001  │  │ FastAPI:8002  │  │ FastAPI   │  │
│   │  Angular:4200 │  │               │  │               │  │ :8003     │  │
│   │  Python:8000  │  │               │  │               │  │ Next.js   │  │
│   │               │  │               │  │               │  │ :3000     │  │
│   │  Cases        │  │  Email sync   │  │  Voice AI     │  │ Will      │  │
│   │  Documents    │  │  Classify     │  │  SMS/Chat     │  │ builder   │  │
│   │  OCR/AI       │  │  Forward      │  │  Fax intake   │  │ POA       │  │
│   │  Timeline     │  │  Conflicts    │  │  Korean NLP   │  │ Clause    │  │
│   │  KG graph     │  │  Accounting   │  │  Extraction   │  │ library   │  │
│   │  Intakes      │  │  Tasks        │  │  Handoff      │  │ Doc gen   │  │
│   │  Clients      │  │  Digest       │  │  Consent      │  │ AI flags  │  │
│   └───────┬───────┘  └───────┬───────┘  └───────┬───────┘  └─────┬─────┘  │
│           │                  │                  │                 │         │
│           └──────────────────┼──────────────────┼─────────────────┘         │
│                              │                                              │
│                    ┌─────────▼─────────┐                                   │
│                    │ INTEGRATION LAYER │                                    │
│                    │                   │                                    │
│                    │ ix_cross_client_  │                                    │
│                    │   map             │                                    │
│                    │ ix_sync_events    │                                    │
│                    │ ix_sync_cursors   │                                    │
│                    │ ix_unified_       │                                    │
│                    │   communications  │                                    │
│                    │ ix_shared_tasks   │                                    │
│                    │ ix_external_      │                                    │
│                    │   system_links    │                                    │
│                    │ ix_activity_feed  │                                    │
│                    │ ix_data_lineage   │                                    │
│                    └─────────┬─────────┘                                   │
│                              │                                              │
│              ┌───────────────┼───────────────┐                             │
│              │               │               │                              │
│         ┌────▼────┐    ┌────▼────┐    ┌─────▼─────┐                       │
│         │PostgreSQL│    │  Redis  │    │ Pub/Sub   │                       │
│         │  :5432   │    │  :6379  │    │  :8085    │                       │
│         │firm_{id} │    │Sessions │    │ sync.*    │                       │
│         │ schemas  │    │JWT revoc│    │ doc-ocr   │                       │
│         └──────────┘    └─────────┘    └───────────┘                       │
│                                                                             │
│   External Services:                                                        │
│   Twilio (Voice/SMS) | Documo (Fax) | GHL (CRM) | WhatConverts            │
│   Google (OAuth/GCS/Vision/Pub/Sub) | Dropbox/OneDrive                     │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 3. Tenant Isolation Model

All apps use the same tenant isolation pattern:

```
Request --> JWT cookie (clv_token) --> Extract law_firm_id claim
                                              |
                                              v
                                     schema = "firm_" + law_firm_id
                                              |
                      +-------------------+---+-------------------+
                      |                   |                       |
               Quarkus/Hibernate    Python psycopg2          Python psycopg2
               CustomTenantResolver SET search_path          SET search_path
               (automatic)          TO firm_{id}             TO firm_{id}
```

**Rules:**

- Tenant ID regex: `^firm_[a-z0-9_]+$`
- Default dev tenant: `firm_demo`
- Each tenant schema contains ALL tables (all apps + integration layer)
- `public` schema: shared `users` table, `invites`
- `platform` schema: superuser admin, tenant registry, usage tracking

## 4. App Ownership Map

Each app is the single writer for its owned tables. Other apps read via the integration layer.

| App | Table Prefix | Table Count |
|-----|-------------|-------------|
| Legal Timeline | `lt_` (no prefix in DB) | ~22 tables |
| MatterMail | `mm_` (no prefix in DB) | ~18 tables |
| AI Reception | `ar_` | 12 tables |
| EZWill | `ew_` | 10 tables |
| Integration Layer | `ix_` | 8 tables |

## 5. Integration Layer Design

The integration layer consists of 8 `ix_*` tables that enable cross-app communication. All apps can read and write to these tables. See `docs/INTEGRATION_SCHEMA.md` for full DDL.

### 5.1 Cross-Client Identity Map (`ix_cross_client_map`)

The core of the integration pattern. Each app maintains its own clients/contacts table; this table links them by phone, email, or fuzzy name match.

| Column | Purpose |
|--------|---------|
| `lt_client_id` | Legal Timeline client reference |
| `mm_contact_id` | MatterMail contact reference |
| `ar_client_id` | AI Reception client reference |
| `ew_client_id` | EZWill client reference |
| `canonical_*` | Best-known name, email, phone |
| `match_method` | phone_exact, email_exact, name_fuzzy, manual |
| `match_confidence` | 0.00 to 1.00 |

### 5.2 Sync Event Bus (`ix_sync_events`)

Append-only event log. Each app publishes events when it creates or updates entities. Other apps consume events by polling with a cursor (`ix_sync_cursors`).

**Event types include:**
- `client.created`, `client.updated`, `client.merged`
- `conversation.completed`, `conversation.extracted`
- `task.created`, `task.completed`
- `document.uploaded`, `document.ocr_completed`
- `handoff.requested`, `handoff.completed`
- `ew.will.created`, `ew.will.submitted`

### 5.3 Unified Communications (`ix_unified_communications`)

A denormalized summary of all communications across all channels (email, phone, SMS, fax). Any app can show a unified timeline.

### 5.4 Shared Tasks (`ix_shared_tasks`)

A unified task queue visible to all apps. Each app can create tasks, and any app can display, assign, or complete them.

### 5.5 External System Links (`ix_external_system_links`)

Maps internal entities to external systems (Twilio, GHL, WhatConverts, Documo, Dropbox, etc.).

### 5.6 Activity Feed (`ix_activity_feed`)

Chronological feed of all significant events across all apps for a given client or case. Powers the "Client 360" view.

### 5.7 Data Lineage (`ix_data_lineage`)

Tracks when data flows between apps (sync, import, merge, reference, derive).

## 6. Agent Interface Contract

All domain agents expose a standard invoke endpoint:

```
POST /agents/{agent_type}/invoke
Content-Type: application/json
Authorization: Bearer <jwt>

{
  "capability": "<capability_name>",
  "payload": { ... },
  "correlation_id": "<uuid>",
  "source_agent": "<calling_agent>"
}
```

Response:

```json
{
  "capability": "<capability_name>",
  "result": { ... },
  "correlation_id": "<uuid>"
}
```

See `docs/AGENT_PROTOCOL.md` for full specification.

## 7. Authentication Strategy

**Current (dev):**
- Each app validates JWT independently
- Shared `clv_token` cookie with `law_firm_id` claim
- Auth middleware skips public webhook paths

**Planned (prod):**
- Shared JWT validation library
- OAuth 2.0 for external integrations
- API key auth for inter-service calls
- Webhook signature verification (Twilio, Documo, GHL)

## 8. Deployment Topology

**Development (docker-compose):**

```
postgres:5432    redis:6379    pubsub:8085
quarkus:8080     python:8000   angular:4200
mattermail:8001  reception:8002
ezwill-api:8003  ezwill-web:3000
```

**Production (Google Cloud Run):**

```
Service 1: legal-timeline-api     (Quarkus  :8080)
Service 2: legal-timeline-ai      (Python   :8000)
Service 3: mattermail             (FastAPI  :8001)
Service 4: ai-reception           (FastAPI  :8002)
Service 5: ezwill-api             (FastAPI  :8003)
Service 6: legal-timeline-web     (Angular  :4200)
Service 7: ezwill-web             (Next.js  :3000)

Shared VPC Connector --> Cloud SQL (PostgreSQL 16)
Shared Memorystore  --> Redis 7
Shared Pub/Sub      --> Topics & Subscriptions
Shared GCS Bucket   --> Document Storage
```

## 9. Cross-App Sync Flow

### Publishing (AI Reception --> Other Apps)

AI Reception writes events to `ix_sync_events`. Legal Timeline and MatterMail poll and react:

| Event | Legal Timeline | MatterMail |
|-------|---------------|------------|
| `client.created` | Show in dashboard | Create contact row |
| `conversation.completed` | Show in case activity | -- |
| `conversation.extracted` | Link to case if matched | Pre-fill sender_matter_map |
| `task.created` | Show in notifications | Show in task board |
| `handoff.requested` | Notify assigned staff | -- |

### Publishing (EZWill --> Other Apps)

| Event | Legal Timeline | AI Reception |
|-------|---------------|-------------|
| `ew.will.created` | Link to client file | -- |
| `ew.will.submitted` | Notify assigned lawyer | -- |
| `ew.poa.created` | Link to client file | -- |

### Consuming (Other Apps --> AI Reception)

| Event | AI Reception Action |
|-------|-------------------|
| `client.updated` (from LT) | Update canonical info in `ix_cross_client_map` |
| `case.created` (from LT) | Enrich caller context for future calls |
| `communication.received` (from MM) | -- (informational only) |

## 10. Database Conventions

Shared across all apps:

| Convention | Rule |
|-----------|------|
| Primary keys | UUID with `gen_random_uuid()` |
| Timestamps | `TIMESTAMPTZ NOT NULL DEFAULT now()` |
| Column names | snake_case |
| Flexible data | JSONB |
| Query params | `%s` placeholders only (never string interpolation) |
| Schema quoting | `psycopg2.sql.Identifier()` |
| Tenant validation | `^firm_[a-z0-9_]+$` regex |
