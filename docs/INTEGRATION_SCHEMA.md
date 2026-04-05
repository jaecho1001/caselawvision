# CaseLawVision Integration Layer Schema

> **Version**: 1.1 — 2026-04-03
> **Apps**: Legal Timeline, MatterMail, AI Reception, EZWill
> **Strategy**: Each app owns its tables. A shared Integration Layer syncs entities via cross-reference maps and an event bus.

## Overview

The integration layer consists of 8 `ix_*` tables created in every `firm_{id}` tenant schema alongside each app's owned tables. All apps can read and write to these tables.

## Full DDL

```sql
-- ============================================================
-- INTEGRATION LAYER SCHEMA
-- Created in each firm_{id} tenant schema
-- All apps read and write to these tables
-- ============================================================

CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS pg_trgm;  -- for fuzzy name matching


-- ============================================================
-- IX-1. CROSS-CLIENT IDENTITY MAP
-- Maps client IDs across apps. Each app maintains its own
-- clients table; this table links them by phone/email/name.
-- ============================================================
CREATE TABLE IF NOT EXISTS ix_cross_client_map (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- App-specific client references
  lt_client_id      UUID NULL,          -- FK to clients (Legal Timeline)
  mm_contact_id     UUID NULL,          -- FK to contacts (MatterMail)
  ar_client_id      UUID NULL,          -- FK to ar_clients (AI Reception)
  ew_client_id      UUID NULL,          -- FK to ew_will_drafts.ew_client_id (EZWill)

  -- Canonical identity fields (best-known values)
  canonical_first_name   VARCHAR(120),
  canonical_last_name    VARCHAR(120),
  canonical_email        VARCHAR(255),
  canonical_phone        VARCHAR(50),
  canonical_display_name VARCHAR(255),

  -- Match quality
  match_method      VARCHAR(30) NOT NULL DEFAULT 'manual',
    -- Values: phone_exact, email_exact, name_fuzzy, manual, system_merge
  match_confidence  NUMERIC(5,2) DEFAULT 1.00,
  verified_by_user  BOOLEAN DEFAULT FALSE,
  verified_at       TIMESTAMPTZ,
  verified_by_user_id UUID NULL,

  -- Lifecycle
  merge_status      VARCHAR(20) DEFAULT 'active',
    -- Values: active, split, superseded
  superseded_by_id  UUID NULL REFERENCES ix_cross_client_map(id),

  metadata          JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- At least one app reference must be non-null
ALTER TABLE ix_cross_client_map
  ADD CONSTRAINT chk_at_least_one_ref
  CHECK (lt_client_id IS NOT NULL OR mm_contact_id IS NOT NULL
         OR ar_client_id IS NOT NULL OR ew_client_id IS NOT NULL);

CREATE INDEX IF NOT EXISTS idx_ix_ccm_lt_client ON ix_cross_client_map(lt_client_id);
CREATE INDEX IF NOT EXISTS idx_ix_ccm_mm_contact ON ix_cross_client_map(mm_contact_id);
CREATE INDEX IF NOT EXISTS idx_ix_ccm_ar_client ON ix_cross_client_map(ar_client_id);
CREATE INDEX IF NOT EXISTS idx_ix_ccm_ew_client ON ix_cross_client_map(ew_client_id);
CREATE INDEX IF NOT EXISTS idx_ix_ccm_email ON ix_cross_client_map(canonical_email);
CREATE INDEX IF NOT EXISTS idx_ix_ccm_phone ON ix_cross_client_map(canonical_phone);
CREATE INDEX IF NOT EXISTS idx_ix_ccm_name_trgm ON ix_cross_client_map
  USING gin ((canonical_first_name || ' ' || canonical_last_name) gin_trgm_ops);


-- ============================================================
-- IX-2. SYNC EVENT BUS
-- Append-only event log. Each app publishes events here
-- when it creates/updates entities. Other apps consume
-- events by polling with a cursor.
-- ============================================================
CREATE TABLE IF NOT EXISTS ix_sync_events (
  id                BIGSERIAL PRIMARY KEY,  -- monotonic for cursor-based polling
  event_id          UUID NOT NULL DEFAULT gen_random_uuid(),

  -- Source
  source_app        VARCHAR(30) NOT NULL,
    -- Values: legal_timeline, mattermail, ai_reception, ezwill
  source_service    VARCHAR(50),

  -- Event type
  event_type        VARCHAR(80) NOT NULL,
    -- Examples:
    --   client.created, client.updated, client.merged
    --   case.created, case.status_changed
    --   conversation.completed, conversation.extracted
    --   task.created, task.completed
    --   document.uploaded, document.ocr_completed
    --   handoff.requested, handoff.completed
    --   ew.will.created, ew.will.submitted, ew.poa.created
    --   conflict.detected, limitation.detected

  -- Entity reference
  entity_type       VARCHAR(30) NOT NULL,
  entity_id         UUID NOT NULL,

  -- Cross-reference
  cross_client_map_id UUID NULL REFERENCES ix_cross_client_map(id),

  -- Payload (key fields that changed, NOT full record)
  payload           JSONB NOT NULL DEFAULT '{}'::jsonb,

  -- Processing
  published_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  idempotency_key   VARCHAR(255),

  -- Metadata
  triggered_by_user_id UUID NULL,
  correlation_id    UUID NULL,
  metadata          JSONB NOT NULL DEFAULT '{}'::jsonb
);

CREATE INDEX IF NOT EXISTS idx_ix_sync_events_source ON ix_sync_events(source_app);
CREATE INDEX IF NOT EXISTS idx_ix_sync_events_type ON ix_sync_events(event_type);
CREATE INDEX IF NOT EXISTS idx_ix_sync_events_entity ON ix_sync_events(entity_type, entity_id);
CREATE INDEX IF NOT EXISTS idx_ix_sync_events_published ON ix_sync_events(published_at);
CREATE INDEX IF NOT EXISTS idx_ix_sync_events_idempotency ON ix_sync_events(idempotency_key);
CREATE INDEX IF NOT EXISTS idx_ix_sync_events_correlation ON ix_sync_events(correlation_id);

CREATE UNIQUE INDEX IF NOT EXISTS uq_ix_sync_events_idempotency
  ON ix_sync_events(idempotency_key) WHERE idempotency_key IS NOT NULL;


-- ============================================================
-- IX-3. SYNC CURSORS
-- Each app tracks which events it has consumed.
-- Enables reliable at-least-once delivery.
-- ============================================================
CREATE TABLE IF NOT EXISTS ix_sync_cursors (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  consumer_app      VARCHAR(30) NOT NULL,
  consumer_service  VARCHAR(50) NOT NULL,
  event_type_filter VARCHAR(80),
  last_event_id     BIGINT NOT NULL DEFAULT 0,
  last_consumed_at  TIMESTAMPTZ,
  metadata          JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX IF NOT EXISTS uq_ix_sync_cursors_consumer
  ON ix_sync_cursors(consumer_app, consumer_service, COALESCE(event_type_filter, '__all__'));


-- ============================================================
-- IX-4. UNIFIED COMMUNICATIONS LOG
-- Denormalized summary of ALL communications across channels.
-- Each app writes a row when it processes a communication.
-- ============================================================
CREATE TABLE IF NOT EXISTS ix_unified_communications (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  cross_client_map_id UUID NULL REFERENCES ix_cross_client_map(id),

  source_app        VARCHAR(30) NOT NULL,
  source_entity_type VARCHAR(30) NOT NULL,
  source_entity_id  UUID NOT NULL,

  channel           VARCHAR(30) NOT NULL,
    -- Values: email, phone, sms, fax, chat, webform, video
  direction         VARCHAR(20) NOT NULL,
    -- Values: inbound, outbound, internal
  status            VARCHAR(30) NOT NULL DEFAULT 'completed',

  subject           VARCHAR(500),
  summary_text      TEXT,
  from_identifier   VARCHAR(255),
  to_identifier     VARCHAR(255),
  participant_count INTEGER DEFAULT 1,

  case_id           UUID NULL,
  case_number       VARCHAR(100),

  occurred_at       TIMESTAMPTZ NOT NULL,
  duration_seconds  INTEGER,

  has_attachments   BOOLEAN DEFAULT FALSE,
  requires_followup BOOLEAN DEFAULT FALSE,
  is_ai_handled     BOOLEAN DEFAULT FALSE,
  language_detected VARCHAR(20),

  metadata          JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_ix_uc_cross_client ON ix_unified_communications(cross_client_map_id);
CREATE INDEX IF NOT EXISTS idx_ix_uc_channel ON ix_unified_communications(channel);
CREATE INDEX IF NOT EXISTS idx_ix_uc_case_id ON ix_unified_communications(case_id);
CREATE INDEX IF NOT EXISTS idx_ix_uc_occurred ON ix_unified_communications(occurred_at);
CREATE INDEX IF NOT EXISTS idx_ix_uc_source ON ix_unified_communications(source_app, source_entity_id);


-- ============================================================
-- IX-5. SHARED TASKS
-- Unified task queue visible to all apps.
-- ============================================================
CREATE TABLE IF NOT EXISTS ix_shared_tasks (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  cross_client_map_id UUID NULL REFERENCES ix_cross_client_map(id),

  source_app        VARCHAR(30) NOT NULL,
  source_entity_type VARCHAR(30) NOT NULL,
  source_entity_id  UUID NOT NULL,

  task_type         VARCHAR(50) NOT NULL,
    -- Values: callback, deadline, followup, review, conflict_check,
    --         limitation_alert, schedule_consult, request_documents,
    --         send_retainer, book_interpreter, waiting, general
  title             VARCHAR(255) NOT NULL,
  description       TEXT,
  priority          VARCHAR(20) DEFAULT 'normal',
    -- Values: urgent, high, normal, low

  assigned_to_user_id UUID NULL,
  assigned_team     VARCHAR(100),

  case_id           UUID NULL,
  case_number       VARCHAR(100),

  due_at            TIMESTAMPTZ,
  status            VARCHAR(20) NOT NULL DEFAULT 'open',
    -- Values: open, in_progress, completed, cancelled, deferred
  completed_at      TIMESTAMPTZ,
  completed_by_user_id UUID NULL,

  origin_channel    VARCHAR(30),
  origin_summary    TEXT,

  metadata          JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_ix_st_cross_client ON ix_shared_tasks(cross_client_map_id);
CREATE INDEX IF NOT EXISTS idx_ix_st_status ON ix_shared_tasks(status);
CREATE INDEX IF NOT EXISTS idx_ix_st_assigned ON ix_shared_tasks(assigned_to_user_id);
CREATE INDEX IF NOT EXISTS idx_ix_st_due ON ix_shared_tasks(due_at);
CREATE INDEX IF NOT EXISTS idx_ix_st_case_id ON ix_shared_tasks(case_id);
CREATE INDEX IF NOT EXISTS idx_ix_st_source ON ix_shared_tasks(source_app, source_entity_id);


-- ============================================================
-- IX-6. EXTERNAL SYSTEM LINKS
-- Maps internal entities to external systems (Twilio, GHL, etc.)
-- ============================================================
CREATE TABLE IF NOT EXISTS ix_external_system_links (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  entity_type       VARCHAR(30) NOT NULL,
  entity_id         UUID NOT NULL,
  owner_app         VARCHAR(30) NOT NULL,

  system_name       VARCHAR(30) NOT NULL,
    -- Values: twilio, ghl, whatconverts, documo, dropbox, onedrive,
    --         microsoft_graph, google_workspace, clio, pclaw, cosmolex
  external_id       VARCHAR(255) NOT NULL,
  external_url      TEXT,
  external_status   VARCHAR(50),

  last_synced_at    TIMESTAMPTZ,
  sync_direction    VARCHAR(20),

  metadata          JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_ix_esl_entity ON ix_external_system_links(entity_type, entity_id);
CREATE INDEX IF NOT EXISTS idx_ix_esl_system ON ix_external_system_links(system_name, external_id);
CREATE INDEX IF NOT EXISTS idx_ix_esl_owner ON ix_external_system_links(owner_app);

CREATE UNIQUE INDEX IF NOT EXISTS uq_ix_esl_entity_system
  ON ix_external_system_links(entity_type, entity_id, system_name, external_id);


-- ============================================================
-- IX-7. ACTIVITY FEED
-- Unified chronological feed for Client 360 view.
-- ============================================================
CREATE TABLE IF NOT EXISTS ix_activity_feed (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  cross_client_map_id UUID NULL REFERENCES ix_cross_client_map(id),
  case_id           UUID NULL,

  source_app        VARCHAR(30) NOT NULL,
  source_entity_type VARCHAR(30) NOT NULL,
  source_entity_id  UUID NOT NULL,

  activity_type     VARCHAR(50) NOT NULL,
    -- Values: call_received, call_completed, sms_received, sms_sent,
    --         email_received, email_sent, email_classified, email_filed,
    --         fax_received, fax_sent, intake_created, intake_updated,
    --         document_uploaded, document_processed, document_ocr_completed,
    --         case_created, case_status_changed, task_created, task_completed,
    --         handoff_requested, handoff_completed, conflict_detected,
    --         limitation_detected, consent_captured, client_stage_changed,
    --         client_merged, will_created, will_submitted, poa_created
  title             VARCHAR(255) NOT NULL,
  description       TEXT,
  icon              VARCHAR(30),

  performed_by_type VARCHAR(20) NOT NULL,
    -- Values: ai, staff, system
  performed_by_user_id UUID NULL,
  performed_by_label VARCHAR(120),

  occurred_at       TIMESTAMPTZ NOT NULL,
  is_internal       BOOLEAN DEFAULT FALSE,
  is_pinned         BOOLEAN DEFAULT FALSE,

  metadata          JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_ix_af_cross_client ON ix_activity_feed(cross_client_map_id);
CREATE INDEX IF NOT EXISTS idx_ix_af_case_id ON ix_activity_feed(case_id);
CREATE INDEX IF NOT EXISTS idx_ix_af_occurred ON ix_activity_feed(occurred_at);
CREATE INDEX IF NOT EXISTS idx_ix_af_type ON ix_activity_feed(activity_type);
CREATE INDEX IF NOT EXISTS idx_ix_af_source ON ix_activity_feed(source_app, source_entity_id);


-- ============================================================
-- IX-8. DATA LINEAGE / PROVENANCE
-- Tracks data flow between apps.
-- ============================================================
CREATE TABLE IF NOT EXISTS ix_data_lineage (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  from_app          VARCHAR(30) NOT NULL,
  from_entity_type  VARCHAR(30) NOT NULL,
  from_entity_id    UUID NOT NULL,

  to_app            VARCHAR(30) NOT NULL,
  to_entity_type    VARCHAR(30) NOT NULL,
  to_entity_id      UUID NOT NULL,

  lineage_type      VARCHAR(30) NOT NULL,
    -- Values: sync, import, merge, reference, derive
  sync_event_id     BIGINT NULL REFERENCES ix_sync_events(id),

  field_mapping     JSONB NOT NULL DEFAULT '{}'::jsonb,
  metadata          JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_ix_dl_from ON ix_data_lineage(from_app, from_entity_type, from_entity_id);
CREATE INDEX IF NOT EXISTS idx_ix_dl_to ON ix_data_lineage(to_app, to_entity_type, to_entity_id);


-- ============================================================
-- TRIGGERS: auto-update updated_at
-- ============================================================
CREATE OR REPLACE FUNCTION ix_set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

DROP TRIGGER IF EXISTS trg_ix_ccm_updated ON ix_cross_client_map;
CREATE TRIGGER trg_ix_ccm_updated
BEFORE UPDATE ON ix_cross_client_map
FOR EACH ROW EXECUTE FUNCTION ix_set_updated_at();

DROP TRIGGER IF EXISTS trg_ix_cursors_updated ON ix_sync_cursors;
CREATE TRIGGER trg_ix_cursors_updated
BEFORE UPDATE ON ix_sync_cursors
FOR EACH ROW EXECUTE FUNCTION ix_set_updated_at();

DROP TRIGGER IF EXISTS trg_ix_tasks_updated ON ix_shared_tasks;
CREATE TRIGGER trg_ix_tasks_updated
BEFORE UPDATE ON ix_shared_tasks
FOR EACH ROW EXECUTE FUNCTION ix_set_updated_at();
```

## Table Summary

| # | Table | Purpose | Key Columns |
|---|-------|---------|-------------|
| 1 | `ix_cross_client_map` | Links client IDs across all apps | lt_client_id, mm_contact_id, ar_client_id, ew_client_id, canonical_* |
| 2 | `ix_sync_events` | Append-only event bus | source_app, event_type, entity_type, entity_id, payload |
| 3 | `ix_sync_cursors` | Consumer cursor tracking | consumer_app, last_event_id |
| 4 | `ix_unified_communications` | Cross-channel communication log | channel, direction, subject, occurred_at |
| 5 | `ix_shared_tasks` | Unified task queue | task_type, title, priority, status, due_at |
| 6 | `ix_external_system_links` | External system references | system_name, external_id, owner_app |
| 7 | `ix_activity_feed` | Client 360 activity timeline | activity_type, title, occurred_at |
| 8 | `ix_data_lineage` | Data provenance tracking | from_app, to_app, lineage_type |

## Sync Event Bus Protocol

### Publishing

```python
INSERT INTO ix_sync_events (source_app, event_type, entity_type, entity_id, payload, idempotency_key)
VALUES ('ai_reception', 'conversation.completed', 'conversation', %(id)s, %(payload)s, %(key)s)
```

### Consuming

```python
SELECT * FROM ix_sync_events
WHERE id > (SELECT last_event_id FROM ix_sync_cursors WHERE consumer_app = 'legal_timeline')
ORDER BY id ASC
LIMIT 100
```

After processing, update the cursor:

```python
UPDATE ix_sync_cursors
SET last_event_id = %(max_id)s, last_consumed_at = now()
WHERE consumer_app = 'legal_timeline'
```

### Idempotency

Events include an `idempotency_key` with a unique constraint. Duplicate publishes are safely rejected. Consumers must also be idempotent since at-least-once delivery means events may be processed more than once.
