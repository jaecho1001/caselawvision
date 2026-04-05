# CaseLawVision Agent Protocol

## Overview

Every domain agent in the CaseLawVision platform exposes a standard interface for the orchestrator (or other agents) to invoke capabilities. This document defines the request/response format, capability discovery, error handling, and examples for each current agent.

## Standard Invoke Endpoint

```
POST /agents/{agent_type}/invoke
Content-Type: application/json
Authorization: Bearer <jwt>
X-Tenant: firm_{id}
```

### Request Schema

```json
{
  "capability": "string (required)",
  "payload": { },
  "correlation_id": "uuid (optional)",
  "source_agent": "string (optional)"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `capability` | string | Yes | The specific action to perform |
| `payload` | object | Yes | Capability-specific input data |
| `correlation_id` | UUID | No | For tracing requests across agents |
| `source_agent` | string | No | Which agent initiated this call |

### Response Schema

```json
{
  "capability": "string",
  "result": { },
  "correlation_id": "uuid or null"
}
```

### Error Response

```json
{
  "detail": "Unknown capability: foo_bar"
}
```

HTTP status codes:
- `200` — Success
- `400` — Unknown capability or invalid payload
- `401` — Missing or invalid JWT
- `403` — Tenant mismatch
- `404` — Referenced entity not found
- `500` — Internal agent error

## Agent Registry

| Agent Type | Endpoint Prefix | App | Status |
|-----------|----------------|-----|--------|
| `will` | `/agents/will/invoke` | EZWill | Active |
| `reception` | `/agents/reception/invoke` | AI Reception | Planned |
| `mail` | `/agents/mail/invoke` | MatterMail | Planned |
| `timeline` | `/agents/timeline/invoke` | Legal Timeline | Planned |

## Capability Discovery

Each agent should expose a capabilities listing:

```
GET /agents/{agent_type}/capabilities
```

Response:

```json
{
  "agent_type": "will",
  "capabilities": [
    {
      "name": "draft_will",
      "description": "Create a new will draft and generate a magic link for the client",
      "payload_schema": { ... }
    },
    {
      "name": "get_draft_status",
      "description": "Get the current status of a will draft",
      "payload_schema": { ... }
    }
  ]
}
```

## Agent Specifications

### Will Agent (EZWill)

**Endpoint**: `POST /agents/will/invoke`

#### Capability: `draft_will`

Creates a new will draft and generates a magic link for the client questionnaire.

Request:
```json
{
  "capability": "draft_will",
  "payload": {
    "prefill": {
      "firstName": "John",
      "lastName": "Doe",
      "email": "john@example.com",
      "phone": "+14165550000",
      "language": "en"
    }
  },
  "correlation_id": "550e8400-e29b-41d4-a716-446655440000",
  "source_agent": "reception"
}
```

Response:
```json
{
  "capability": "draft_will",
  "result": {
    "draft_id": "a1b2c3d4-...",
    "magic_link": "https://app.ezwill.ca/will?t=token123",
    "token": "token123"
  },
  "correlation_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

#### Capability: `get_draft_status`

Returns the current status and progress of a will draft.

Request:
```json
{
  "capability": "get_draft_status",
  "payload": {
    "draft_id": "a1b2c3d4-..."
  }
}
```

Response:
```json
{
  "capability": "get_draft_status",
  "result": {
    "draft_id": "a1b2c3d4-...",
    "status": "in_progress",
    "current_step": 3,
    "completed_steps": [0, 1, 2],
    "client_name": "John Doe",
    "tier": 1,
    "will_type": "single"
  }
}
```

#### Capability: `run_ai_flags`

Triggers the AI flagging engine to analyze the draft and return compliance warnings.

Request:
```json
{
  "capability": "run_ai_flags",
  "payload": {
    "draft_id": "a1b2c3d4-..."
  }
}
```

Response:
```json
{
  "capability": "run_ai_flags",
  "result": {
    "flags": [
      {
        "flag_id": "minor_beneficiary_no_trust",
        "severity": "critical",
        "title": "Minor beneficiary without trust provision",
        "statute": "SLRA s.42"
      }
    ],
    "flag_count": 1
  }
}
```

### Reception Agent (AI Reception) — Planned

**Endpoint**: `POST /agents/reception/invoke`

Planned capabilities:
- `lookup_client` — Search for existing client by phone/email/name
- `check_conflicts` — Run conflict check against all apps
- `schedule_callback` — Create a callback task
- `record_intake` — Record structured intake information
- `transfer_to_staff` — Initiate handoff to human staff

### Mail Agent (MatterMail) — Planned

**Endpoint**: `POST /agents/mail/invoke`

Planned capabilities:
- `classify_email` — AI classification of an email
- `file_to_case` — Associate email with a case
- `check_limitation` — Scan for limitation period deadlines
- `send_ack` — Send automated acknowledgment
- `create_task` — Create follow-up task from email

### Timeline Agent (Legal Timeline) — Planned

**Endpoint**: `POST /agents/timeline/invoke`

Planned capabilities:
- `create_intake` — Create or update client intake
- `process_document` — Upload and OCR a document
- `query_case` — AI Q&A over case documents
- `extract_timeline` — Extract timeline entries from document
- `check_conflicts` — Check new client against existing cases

## Cross-Agent Communication

Agents can invoke each other through the orchestrator or directly. When AI Reception's voice agent identifies that a caller wants to create a will, it calls the Will Agent:

```
AI Reception Voice Agent
  --> POST /agents/will/invoke { capability: "draft_will", source_agent: "reception" }
  <-- { result: { magic_link: "..." } }
  --> Voice response: "I've created a will questionnaire for you. You'll receive a link shortly."
```

The `correlation_id` ties the entire flow together across agents, and `ix_sync_events` captures each step for audit and cross-app visibility.

## Adding a New Agent

1. Choose a table prefix (e.g., `im_` for immigration)
2. Register the prefix in the platform table prefix registry
3. Create database migrations for the agent's owned tables
4. Implement the `/agents/{type}/invoke` endpoint
5. Add `ew_client_id`-equivalent column to `ix_cross_client_map` if tracking clients
6. Publish events to `ix_sync_events` for cross-app visibility
7. Document capabilities in this file
