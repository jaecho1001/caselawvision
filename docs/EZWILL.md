# EZWill — Ontario Estate Planning Platform

## Overview

| Field | Value |
|-------|-------|
| **App** | EZWill |
| **Table Prefix** | `ew_` |
| **Backend** | FastAPI (Python) on port 8003 |
| **Frontend** | Next.js 16 on port 3000 |
| **Purpose** | AI-powered estate planning: wills, powers of attorney, affidavits |
| **Jurisdiction** | Ontario, Canada (SLRA, SDA) |
| **Status** | Active |

## Three-Portal System

EZWill operates through three distinct interfaces:

### 1. Client Questionnaire Portal

Accessed via magic link (no account required). The client fills out a multi-step questionnaire covering:

1. **About You** — Personal information, marital status, citizenship
2. **Your Family** — Spouse, children, dependents (with minor detection)
3. **Your Estate** — Assets (real estate, bank, investment, RRSP/TFSA, insurance, etc.)
4. **Your Arrangements** — Executor, guardian, beneficiary, distribution preferences
5. **POA Property** — Continuing power of attorney for property
6. **POA Personal Care** — Power of attorney for personal care

The questionnaire supports EN/KO bilingual operation. AI flags are evaluated in real-time as the client progresses.

### 2. Lawyer Dashboard

For the reviewing lawyer. Provides:

- **Draft list** with status tracking (link_sent, opened, in_progress, submitted, in_review, approved, signed, archived)
- **Tier 1 / Tier 2 workflow toggle**
- **Clause editor** for Tier 2 (60+ clauses across 15 sections)
- **Design sheets** for meeting notes and design decisions
- **AI flag review** with dismiss/acknowledge workflow
- **Document generation** (8 document types)
- **Signing event recording** (SLRA s.4 in-person, s.21.1 remote video)

### 3. Review Portal

Read-only view for clients to review generated documents before signing.

## Tier System

### Tier 1 — Simple Will

For straightforward estates. The system generates documents directly from questionnaire answers using template variable substitution. No lawyer clause editing required.

Suitable when:
- Single will (no probate/non-probate split)
- No complex trusts
- No US-person beneficiaries
- No ODSP/disability considerations
- Standard executor/guardian arrangements

### Tier 2 — Complex Will

For estates requiring lawyer intervention. The lawyer reviews the client's answers and customizes the will using the clause library.

Triggers for Tier 2:
- Dual will structure (probate + non-probate)
- Minor beneficiaries requiring trust provisions
- Henson trusts for ODSP recipients
- US-person beneficiaries (QDT considerations)
- Complex asset distributions
- Business succession planning
- Blended family situations

## Clause Library

60+ clauses organized into 15 sections:

| Section | Description |
|---------|-------------|
| Preamble | Will identification, revocation of prior wills |
| Definitions | Legal terms, family definitions |
| Executor Appointment | Primary and backup executors, powers |
| Debts & Expenses | Payment of debts, funeral expenses |
| Specific Gifts | Particular items to named beneficiaries |
| Real Estate | Primary residence, investment properties |
| Residue | Distribution of remaining estate |
| Trust Provisions | Children's trusts, spousal trusts, Henson trusts |
| POA Property | Attorney appointment, powers, restrictions |
| POA Personal Care | Attorney appointment, wishes, end-of-life |
| Guardian | Guardian for minor children |
| Digital Assets | Online accounts, digital property |
| Business Interests | Business succession, share transfers |
| Tax Planning | GRE election, QDT provisions |
| General Provisions | Severability, governing law, interpretation |

Each clause has:
- Unique `clause_id` (e.g., `WILL_PREAMBLE_01`)
- Default HTML template with `{{variable}}` placeholders
- Section assignment and sort order
- Applicability rules (which will types, which scenarios)

Lawyers can customize clause text, reorder clauses, and add AI-generated clauses.

## Document Generation Pipeline

EZWill generates 8 document types as DOCX (with optional PDF export):

| Document | Description |
|----------|-------------|
| `probate_will` | Last Will and Testament (for probate assets) |
| `non_probate_will` | Last Will and Testament — Non-Probate (for private assets) |
| `poa_property` | Continuing Power of Attorney for Property |
| `poa_personal_care` | Power of Attorney for Personal Care |
| `affidavit_execution` | Affidavit of Execution (probate will) |
| `affidavit_execution_np` | Affidavit of Execution (non-probate will) |
| `affidavit_execution_poa_prop` | Affidavit of Execution (POA property) |
| `affidavit_execution_poa_pc` | Affidavit of Execution (POA personal care) |

Generation uses `python-docx` with:
- Professional formatting (Vaturi & Cho LLP branding)
- Table-based signing pages
- Variable substitution from questionnaire data
- Clause rendering from selections (Tier 2)
- HTML-to-DOCX conversion for rich text clauses

## AI Flagging Engine

9 rules that analyze the draft and surface compliance warnings:

| Rule | Severity | Trigger |
|------|----------|---------|
| Minor beneficiary without trust | Critical | Child beneficiary under 18, no trust provision |
| No backup executor | Warning | Single executor with no alternate |
| ODSP beneficiary without Henson trust | Critical | Beneficiary receives ODSP, no Henson trust |
| US-person beneficiary without QDT | Warning | US citizen/resident beneficiary, no QDT election |
| Unequal distribution without explanation | Info | Non-equal shares among children |
| Missing digital assets clause | Info | No digital assets section configured |
| POA without restrictions clause | Warning | POA property with broad powers, no restrictions |
| Real estate in multiple jurisdictions | Warning | Properties outside Ontario |
| No guardian designated | Critical | Minor children, no guardian named |

Flags include bilingual titles (EN/KO) and statute references (SLRA, SDA).

## Ontario Legal Foundations

- **Succession Law Reform Act (SLRA)** — Will validity requirements (s.3-4), witnesses (s.4), revocation (s.15), dependant relief (Part V)
- **Substitute Decisions Act (SDA)** — POA for property (s.7), POA for personal care (s.46)
- **SLRA s.21.1** — Remote video witnessing provisions
- **SLRA s.42** — Trust for minor beneficiaries
- **Income Tax Act** — Graduated Rate Estate (GRE) election, Qualifying Domestic Trust (QDT)

## Database Tables (ew_ prefix)

| Table | Purpose |
|-------|---------|
| `ew_will_drafts` | Core draft record with JSONB sections |
| `ew_people` | Beneficiaries, executors, guardians, attorneys |
| `ew_assets` | Asset inventory with type categorization |
| `ew_ai_flags` | AI-generated compliance warnings |
| `ew_client_links` | Magic links with expiry and revocation |
| `ew_design_sheets` | Lawyer meeting notes per section |
| `ew_trusts` | Trust configurations (children's, spousal, Henson, GRE, QDT) |
| `ew_signing_events` | SLRA-compliant signing records |
| `ew_document_generations` | Document generation audit log |
| `ew_clause_selections` | Tier 2 clause choices per draft |
| `ew_document_configs` | Which document types are enabled per draft |

## Integration with CaseLawVision

- **Cross-client identity**: `ew_client_id` column added to `ix_cross_client_map`
- **Events published**: `ew.will.created`, `ew.will.submitted`, `ew.poa.created`
- **Agent endpoint**: `POST /agents/will/invoke` with capabilities: `draft_will`, `get_draft_status`, `run_ai_flags`
- **AI Reception handoff**: When a caller mentions estate planning, the reception agent can invoke the will agent to create a draft and send the magic link

## API Routes

| Route Group | Prefix | Endpoints |
|------------|--------|-----------|
| Drafts | `/api/drafts` | Create, get, update, submit, list |
| Links | `/api/links` | Create magic link, validate token, mark opened |
| Clauses | `/api/drafts/{id}/clauses` | Get clause library, save selections |
| Documents | `/api/documents` | Generate, download, list configs |
| Review | `/api/review` | Get review data, approve |
| Agents | `/agents/will/invoke` | Standard agent protocol |
| Health | `/`, `/ready` | Health checks |
