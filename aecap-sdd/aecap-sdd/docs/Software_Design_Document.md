# Software Design Document
## AI Email & Chat Automation Platform (AECAP)

**Document Version:** 1.0
**Status:** Draft — Pending Architecture Approval
**Classification:** Internal / Engineering

---

## Executive Summary

The AI Email & Chat Automation Platform (AECAP) is a multi-tenant SaaS system that ingests unstructured communication (email, chat, documents, manual input), applies AI-driven understanding (entity extraction, intent classification, urgency detection, summarization), and orchestrates deterministic business actions (CRM, tickets, sheets, notifications, calendar, webhooks).

The core design principle is **"Unstructured In, Structured Out, Auditable Everywhere."** Every message flowing through the system produces a signed, versioned, validated JSON envelope that acts as the single source of truth for all downstream automations.

---

## 1. Complete System Architecture

### 1.1 Architectural Style

We adopt **Clean / Hexagonal Architecture** with a **Modular Monolith** on the backend that is **service-ready** (each bounded context can be extracted into a microservice without refactoring). This gives us:

- Fast iteration speed of a monolith
- Strict domain boundaries (via ports & adapters)
- Independent testability
- Zero coupling between AI logic, ingestion adapters, and action executors

### 1.2 Logical Layers

| Layer | Responsibility | Depends On |
|---|---|---|
| **Presentation** | Next.js dashboard, admin console, public API | API Gateway |
| **API Gateway** | FastAPI edge — authN, rate limits, request shaping | Application |
| **Application (Use Cases)** | Orchestrates domain logic (e.g. `IngestMessageUseCase`) | Domain |
| **Domain** | Pure business logic: `Message`, `ExtractionResult`, `Action`, `Tenant` | — (no external deps) |
| **Infrastructure** | Adapters: Gemini, OpenAI, Gmail, Slack, Supabase, n8n | Domain (via ports) |
| **Automation Runtime** | n8n workflows for long-running / branching automations | Infrastructure |

### 1.3 High-Level Component Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                       CLIENT TIER (Vercel)                          │
│  Next.js 14 (App Router) · TypeScript · Tailwind · Shadcn UI        │
│  Dashboard · Inbox View · Config UI · Admin Console                 │
└─────────────────────────────────────────────────────────────────────┘
                                │  HTTPS / JWT (Supabase)
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    API GATEWAY (FastAPI @ Railway)                  │
│  Auth middleware · Rate limiter · RequestID · CORS · Idempotency    │
└─────────────────────────────────────────────────────────────────────┘
                                │
     ┌──────────────────────────┼───────────────────────────┐
     ▼                          ▼                           ▼
┌──────────┐            ┌────────────────┐          ┌───────────────┐
│ Ingestion│            │  Application   │          │   Automation  │
│ Adapters │──events──► │  Use Cases     │──jobs──► │   Runtime     │
│          │            │  (Orchestrator)│          │   (n8n)       │
│ Gmail    │            │                │          │               │
│ Outlook  │            │  Message Svc   │          │ CRM Push      │
│ Slack    │            │  AI Svc        │          │ Sheets Push   │
│ Webhook  │            │  Action Svc    │          │ Slack Notify  │
│ Upload   │            │  Tenant Svc    │          │ Calendar      │
└──────────┘            └────────────────┘          │ Webhooks      │
                                │                   └───────────────┘
                                ▼
                        ┌───────────────┐
                        │  AI PROVIDER  │
                        │  ABSTRACTION  │
                        │  ┌─────────┐  │
                        │  │ Gemini  │  │
                        │  │ OpenAI  │  │
                        │  │ Fallback│  │
                        │  └─────────┘  │
                        └───────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│               PERSISTENCE (Supabase Postgres + Storage)             │
│  RLS-secured tables · Row-level tenancy · pgvector · Object storage │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│   OBSERVABILITY: OpenTelemetry → Grafana Cloud · Sentry · Logtail   │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.4 Key Architectural Decisions (ADRs summary)

| # | Decision | Rationale |
|---|---|---|
| ADR-001 | Modular Monolith over microservices | Team scale, deployment simplicity, lower ops cost until 100+ tenants |
| ADR-002 | FastAPI over Node backend | Python-native AI SDKs, Pydantic validation aligns with structured-output contract |
| ADR-003 | n8n for orchestration | Visual auditability, non-eng authoring, retry/back-off built in |
| ADR-004 | Supabase (Postgres + Auth + Storage) | Single provider, RLS-based tenancy, native JWT with the Next.js frontend |
| ADR-005 | Provider abstraction over Gemini/OpenAI | Vendor lock-in avoidance, cost routing, graceful degradation |
| ADR-006 | Event-driven internal bus (Postgres LISTEN/NOTIFY + outbox) | Reliable, cheap, no Kafka overhead at current scale |
| ADR-007 | JSON Schema + Pydantic for all extractions | Guarantees downstream contract validity |

---

## 2. User Journey

**Persona:** Ops Manager at an SMB.

1. **Sign up** → email/password or Google OAuth via Supabase Auth.
2. **Onboarding wizard** → creates a Workspace (tenant), picks industry template (Sales / Support / Logistics).
3. **Connect a source** → OAuth to Gmail; grants read scope; a Supabase-stored refresh token begins polling.
4. **Configure routing rules** → "If `intent = support_request` and `urgency >= high`, create ticket + notify #ops in Slack."
5. **Live inbox view** → sees a new email arrive; within seconds the message shows `status = extracted`, with structured JSON, summary, and confidence scores.
6. **Review & approve** → low-confidence extractions land in a *review queue*; user edits and approves; the correction feeds a fine-tuning dataset.
7. **Business action executed** → CRM lead visible in HubSpot; Slack notification received.
8. **Analytics** → weekly digest: volume by intent, urgency mix, action success rate.

### 2.1 User Journey Diagram

```
[Sign Up] → [Onboard] → [Connect Source] → [Define Rules]
    │                                          │
    ▼                                          ▼
[Live Inbox] ◄──── [Message Ingested] ──── [AI Processed]
    │
    ├─► High confidence  → [Auto Action]  → [Audit Log]
    └─► Low confidence   → [Review Queue] → [Human Approve] → [Auto Action]
```

---

## 3. Admin Journey

**Persona:** Platform Admin (internal SRE / Customer Success).

1. **Global dashboard** — tenants, MAU, message volume, error budget.
2. **Tenant management** — impersonate (with audit trail), suspend, quota adjustments.
3. **AI configuration** — provider routing (Gemini primary, OpenAI fallback), prompt version deployment (canary → rollout).
4. **Prompt registry** — semantic versioned prompts, A/B tests, evaluator scores.
5. **Feature flags** — per-tenant toggles (e.g. WhatsApp beta).
6. **Billing / usage** — token consumption per tenant.
7. **Compliance** — data retention overrides, DSAR (delete-my-data) execution, export logs.
8. **Incident console** — DLQ inspector, redrive failed jobs, replay from raw payload.

---

## 4. Workflow Diagram

```
                 ┌─────────────────┐
                 │  INGESTION      │
                 │  (Adapter → API)│
                 └────────┬────────┘
                          │  POST /v1/messages (normalized envelope)
                          ▼
                 ┌─────────────────┐
                 │  Message Store  │  (raw + normalized, S3-URI in Supabase)
                 └────────┬────────┘
                          │  emits: message.received
                          ▼
                 ┌─────────────────┐
                 │  AI Orchestrator│
                 │  · Preprocess   │
                 │  · Classify     │
                 │  · Extract      │
                 │  · Validate     │
                 └────────┬────────┘
                          │  emits: message.extracted
                          ▼
                 ┌─────────────────┐
                 │  Rule Engine    │  (per-tenant JSON rules)
                 └────────┬────────┘
                          │  emits: action.dispatch
                          ▼
                 ┌─────────────────┐
                 │  n8n Runtime    │  ← retries, DLQ, human-in-the-loop
                 └────────┬────────┘
                          │
      ┌─────────┬─────────┼─────────┬─────────┬──────────┐
      ▼         ▼         ▼         ▼         ▼          ▼
   CRM Lead  Ticket    DB Save   Sheets    Slack     Email/Cal/Webhook
      │         │         │         │         │          │
      └─────────┴────► [Audit Log] ◄──────────┴──────────┘
```

---

## 5. AI Processing Flow

```
Raw Input
   │
   ▼
[1] Content Normalizer
     · MIME parsing, HTML→text, attachment extraction
     · PDF/CSV via unstructured.io + pypdf; OCR fallback (Tesseract)
   │
   ▼
[2] PII Detector & Redactor  (Presidio)
     · Optional per-tenant policy; produces redacted twin for LLM
   │
   ▼
[3] Language Detect + Chunker
     · fastText langid; token-aware chunking (max 8k)
   │
   ▼
[4] Retrieval (optional RAG)
     · pgvector lookup for tenant knowledge base / past threads
   │
   ▼
[5] Prompt Assembly
     · SystemPrompt + FewShotBank(tenant) + Schema + Content
   │
   ▼
[6] LLM Call (Provider Abstraction)
     · Structured Output mode (Gemini responseSchema / OpenAI json_schema)
     · Timeout + retry + fallback provider
   │
   ▼
[7] Schema Validator (Pydantic)
     · If invalid → self-repair loop (max 2) → else quarantine
   │
   ▼
[8] Post-Processors
     · Entity canonicalization (email/phone normalize, currency parse)
     · Urgency scorer (rules + LLM signal blend)
     · Confidence calibration
   │
   ▼
[9] Missing-Field Detector
     · Compares required schema fields vs extracted; flags gaps
   │
   ▼
[10] Emit → ExtractionResult (persisted + event on bus)
```

**Structured Output Contract (canonical envelope):**

```jsonc
{
  "extraction_id": "uuid",
  "tenant_id": "uuid",
  "source": { "channel": "gmail", "external_id": "..." },
  "intent": "support_request",
  "urgency": "high",
  "confidence": 0.92,
  "summary": "Customer reports login failure after MFA reset.",
  "entities": {
    "person": { "name": "Jane Doe", "email": "jane@acme.io" },
    "org":    { "name": "Acme" },
    "product":{ "sku": "PLAN-PRO" },
    "dates":  ["2026-07-24"]
  },
  "missing_fields": ["account_id"],
  "suggested_actions": ["create_ticket", "notify_slack"],
  "prompt_version": "extract@2.4.1",
  "model": "gemini-2.5-pro"
}
```

---

## 6. Folder Structure

```
aecap/
├── apps/
│   ├── web/                          # Next.js frontend
│   │   ├── app/                      # App Router
│   │   ├── components/               # Shadcn UI + custom
│   │   ├── features/                 # Feature-sliced (inbox, rules, admin)
│   │   ├── lib/                      # api client, supabase client, hooks
│   │   ├── styles/
│   │   └── tests/
│   │
│   └── api/                          # FastAPI backend (modular monolith)
│       ├── src/
│       │   ├── main.py               # ASGI app factory
│       │   ├── core/                 # config, logging, security, di
│       │   ├── domain/               # pure entities, value objects
│       │   │   ├── message/
│       │   │   ├── extraction/
│       │   │   ├── action/
│       │   │   └── tenant/
│       │   ├── application/          # use cases, orchestrators
│       │   │   ├── ingest_message/
│       │   │   ├── run_extraction/
│       │   │   ├── dispatch_action/
│       │   │   └── ports/            # abstract interfaces
│       │   ├── infrastructure/       # adapters implementing ports
│       │   │   ├── ai/
│       │   │   │   ├── gemini_adapter.py
│       │   │   │   ├── openai_adapter.py
│       │   │   │   └── provider_router.py
│       │   │   ├── sources/
│       │   │   │   ├── gmail/
│       │   │   │   ├── outlook/
│       │   │   │   ├── slack/
│       │   │   │   ├── upload/
│       │   │   │   └── manual/
│       │   │   ├── actions/
│       │   │   │   ├── crm/
│       │   │   │   ├── ticketing/
│       │   │   │   ├── sheets/
│       │   │   │   ├── slack/
│       │   │   │   ├── email/
│       │   │   │   ├── calendar/
│       │   │   │   └── webhook/
│       │   │   ├── persistence/
│       │   │   │   ├── supabase/
│       │   │   │   └── repositories/
│       │   │   └── automation/
│       │   │       └── n8n_client.py
│       │   ├── interfaces/           # transport layer
│       │   │   ├── http/
│       │   │   │   ├── v1/
│       │   │   │   └── middleware/
│       │   │   └── events/
│       │   └── shared/               # errors, types, utils
│       ├── tests/
│       │   ├── unit/
│       │   ├── integration/
│       │   └── contract/
│       └── pyproject.toml
│
├── automation/
│   └── n8n/                          # Exported workflows (JSON, versioned)
│
├── packages/
│   ├── schemas/                      # JSON Schemas (source of truth)
│   ├── prompts/                      # Prompt registry (yaml, versioned)
│   └── sdk-ts/                       # Optional TS SDK for external clients
│
├── infra/
│   ├── supabase/                     # migrations, RLS policies, seed
│   ├── railway/
│   ├── vercel/
│   └── terraform/                    # optional IaC
│
├── .github/workflows/                # CI/CD
├── docs/                             # ADRs, runbooks, this SDD
└── docker-compose.yml                # local dev
```

---

## 7. Backend Architecture

### 7.1 Ports & Adapters (Hexagonal)

Every external concern is expressed as an abstract **Port** in `application/ports/` and implemented as an **Adapter** in `infrastructure/`. Use cases depend only on ports, never on concrete SDKs.

Example ports:
- `AIProviderPort` — `extract(prompt, schema) -> ExtractionResult`
- `MessageRepositoryPort`
- `EventBusPort`
- `ActionDispatcherPort`
- `SecretsPort`

### 7.2 Request Lifecycle

1. **ASGI** receives request → Uvicorn workers.
2. **Middleware chain**: RequestID → CORS → Auth (Supabase JWT verify) → Rate limit (Redis token bucket) → Idempotency-Key check → Tenant resolver.
3. **Router** → thin controller → **Use Case** → **Domain** → **Ports**.
4. **Response** serialized via Pydantic; error interceptor maps domain exceptions to HTTP.

### 7.3 Concurrency & Background Work

- **Sync path**: FastAPI async endpoints for ingestion and CRUD.
- **Async path**: **Arq** (Redis-backed) for AI extraction & action dispatch (long tasks).
- **Scheduled**: Arq cron for Gmail/Outlook polling, DLQ redrive, analytics rollups.
- **n8n** handles multi-step branching workflows once the extraction envelope is finalized.

### 7.4 AI Provider Abstraction

```
AIProviderPort
   ├── GeminiAdapter        (primary)
   ├── OpenAIAdapter        (fallback)
   └── ProviderRouter       (policy: cost, latency, capability, tenant override)
        ├── Circuit breaker (pybreaker)
        ├── Retry (tenacity, exponential + jitter)
        └── Metrics (tokens, latency, cost)
```

Routing policy is data-driven per tenant: `{primary: gemini-2.5-pro, fallback: gpt-4o, budget_usd_month: 50}`.

---

## 8. Frontend Architecture

### 8.1 Stack Choices

- **Next.js 14 App Router** — RSC for data-heavy dashboards, Server Actions for mutations that don't need external clients.
- **TypeScript strict mode**, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`.
- **Tailwind CSS + Shadcn UI** — accessible primitives, theming via CSS vars.
- **State**: TanStack Query for server state; Zustand for ephemeral UI state; no Redux.
- **Forms**: React Hook Form + Zod (shared schemas with backend via `packages/schemas`).
- **Realtime**: Supabase Realtime for inbox updates.

### 8.2 Feature-Sliced Layout

```
apps/web/features/
├── inbox/          (list, detail, review-queue)
├── rules/          (rule builder DSL editor)
├── sources/        (OAuth connect flows)
├── admin/          (tenant admin, prompt registry, DLQ)
├── analytics/
└── auth/
```

Each feature exports: `api/`, `components/`, `hooks/`, `types/`, `index.ts` (public surface only).

### 8.3 Frontend ↔ Backend Contract

- OpenAPI schema auto-generated from FastAPI → `openapi-typescript` produces typed client.
- Any schema drift breaks CI.

---

## 9. Database Design

Supabase Postgres, multi-tenant via `tenant_id` + **Row-Level Security**.

### 9.1 Core Tables

| Table | Purpose | Notable Columns |
|---|---|---|
| `tenants` | Workspaces | `id`, `name`, `plan`, `settings jsonb` |
| `users` | Supabase auth mirror | `id`, `email`, `default_tenant_id` |
| `memberships` | User↔Tenant with role | `user_id`, `tenant_id`, `role` (owner/admin/member/viewer) |
| `sources` | Connected channels | `id`, `tenant_id`, `type`, `credentials_ref`, `status` |
| `messages` | Raw + normalized inbound | `id`, `tenant_id`, `source_id`, `raw_uri`, `normalized jsonb`, `hash` |
| `extractions` | AI output envelopes | `id`, `message_id`, `intent`, `urgency`, `confidence`, `payload jsonb`, `prompt_version`, `model`, `status` |
| `rules` | Routing rules | `id`, `tenant_id`, `dsl jsonb`, `enabled`, `version` |
| `actions` | Executed business actions | `id`, `extraction_id`, `type`, `status`, `result jsonb`, `attempts` |
| `audit_logs` | Immutable log | `id`, `actor_id`, `tenant_id`, `event`, `payload jsonb`, `created_at` |
| `secrets_refs` | Pointers to Supabase Vault | `id`, `tenant_id`, `key`, `vault_ref` |
| `webhooks` | Outbound webhook endpoints | `id`, `tenant_id`, `url`, `secret_ref`, `events[]` |
| `prompt_versions` | Prompt registry | `id`, `name`, `version`, `body`, `schema_ref`, `active` |
| `dead_letter_queue` | Failed jobs | `id`, `job_type`, `payload`, `error`, `attempts` |
| `usage_events` | Metering | `id`, `tenant_id`, `kind`, `qty`, `cost_usd`, `ts` |
| `documents_kb` | Optional RAG store | `id`, `tenant_id`, `embedding vector(1536)`, `chunk`, `meta` |

### 9.2 Indexes & Performance

- Partial indexes on `messages(tenant_id, created_at DESC) WHERE status='pending'`.
- `pgvector` HNSW index on `documents_kb.embedding`.
- Monthly partitioning on `messages`, `extractions`, `audit_logs` once >10M rows.

### 9.3 RLS Policy (illustrative)

```
CREATE POLICY tenant_isolation ON messages
USING (tenant_id = current_setting('request.jwt.claims', true)::json->>'tenant_id'::uuid);
```

All tables carry the same predicate; service-role key bypasses only for the backend.

---

## 10. ER Diagram

```
tenants ──1..*── memberships ──*..1── users
   │                                     │
   │1..*                                 │
   ▼                                     │
sources ──1..*── messages ──1..1── extractions ──1..*── actions
                    │                    │                 │
                    │                    │1..*             │
                    │                    ▼                 │
                    │                 rules(applied)       │
                    │                                      │
                    └────────────► audit_logs ◄────────────┘

prompt_versions ── used_by ── extractions
webhooks ── receives ── action.dispatched
documents_kb ── retrieved_by ── extractions (optional RAG)
dead_letter_queue ── owns ── failed(actions/extractions)
```

---

## 11. Authentication Flow

### 11.1 End-User (Dashboard)

```
Browser ──► Supabase Auth (email/OAuth) ──► JWT (RS256)
Browser ──► FastAPI (Bearer JWT)
FastAPI ──► verify sig w/ JWKS (cached) ──► extract sub, tenant claim
FastAPI ──► set `SET LOCAL request.jwt.claims` in each txn ──► RLS enforced
```

- JWT contains `tenant_id` custom claim (set via Supabase hook on membership switch).
- Refresh token rotation handled by Supabase JS client on the frontend.

### 11.2 Machine-to-Machine (external API clients)

- **API keys** (scoped, hashed at rest with Argon2id, prefix visible for identification).
- Signed with HMAC on requests to `/v1/ingest`.

### 11.3 Source Providers (Gmail/Outlook/Slack)

- OAuth 2.0 authorization code + PKCE.
- Refresh tokens stored in **Supabase Vault**; only vault references live in `sources`.

### 11.4 n8n ↔ API

- Mutual authentication via short-lived service JWT + IP allowlist.

---

## 12. API Design

**Style:** REST + JSON; versioned under `/v1`. Cursor pagination. Idempotency-Key header on all POSTs.

### 12.1 Resource Endpoints (selected)

| Method | Path | Purpose |
|---|---|---|
| POST | `/v1/messages` | Ingest a normalized message (used by adapters + external clients) |
| GET | `/v1/messages` | List (filter by status/intent/urgency) |
| GET | `/v1/messages/{id}` | Detail incl. extraction |
| POST | `/v1/messages/{id}/reprocess` | Re-run AI |
| GET | `/v1/extractions/{id}` | Envelope + confidence |
| PATCH | `/v1/extractions/{id}` | Human correction (feeds training set) |
| POST | `/v1/uploads` | Signed URL for PDF/CSV |
| GET/POST/PATCH/DELETE | `/v1/rules` | Rule CRUD |
| GET/POST | `/v1/sources` | Manage integrations |
| POST | `/v1/sources/{id}/oauth/callback` | OAuth exchange |
| POST | `/v1/actions/{id}/redrive` | Retry a failed action |
| GET | `/v1/audit-logs` | Paginated audit |
| GET | `/v1/health` / `/v1/ready` | Probes |
| POST | `/v1/webhooks/inbound/{provider}` | Provider push (Gmail, Slack Events, etc.) |

### 12.2 Envelope Standard

All responses:
```json
{ "data": {...}, "meta": {"request_id":"...","trace_id":"..."} }
```
Errors follow **RFC 7807** (Problem Details).

### 12.3 Contract Testing

- Every endpoint has an OpenAPI schema.
- Consumer-driven tests via **Schemathesis** run in CI.

---

## 13. Validation Strategy

- **Layer 1 — Edge (FastAPI/Pydantic v2):** shape, types, ranges, format (email, UUID).
- **Layer 2 — Domain invariants:** e.g. an `Extraction` cannot exist without a `Message`; `urgency` ∈ {low, medium, high, critical}.
- **Layer 3 — AI Output (json_schema):** provider-side structured output *plus* server-side re-validation.
- **Layer 4 — Business rules:** rule engine evaluates a DSL against extraction; rejects unsafe combinations.
- **Layer 5 — DB constraints:** NOT NULL, FKs, CHECKs; RLS as authorization guard.

**Self-Repair Loop:** If AI JSON fails schema, we send a repair prompt with the validator errors (max 2 attempts) before quarantining to DLQ.

---

## 14. Error Handling Strategy

- **Typed domain errors** in `shared/errors.py`: `ValidationError`, `NotFoundError`, `ConflictError`, `AuthError`, `RateLimitError`, `UpstreamError`.
- **Global exception middleware** maps to RFC 7807.
- **Retries** only for idempotent + transient (`UpstreamError`, `503`, `429`) with exponential backoff + jitter.
- **Circuit breakers** on external adapters (AI providers, Slack, Gmail).
- **Dead Letter Queue** table with redrive UI.
- **Never swallow errors** — every catch either re-raises, converts to a domain error, or writes a structured log line with trace ID.

---

## 15. Security Strategy

### 15.1 Identity & Access
- Supabase Auth + RLS; least-privilege service roles.
- SoD: separate keys for read/write/admin operations.
- MFA required for admin console; SAML/SSO for enterprise plans.

### 15.2 Data Protection
- TLS 1.3 in transit; AES-256 at rest (Supabase managed).
- Field-level encryption for OAuth refresh tokens via **Supabase Vault**.
- PII redaction before LLM calls (opt-in per tenant); redaction map stored separately.

### 15.3 Application
- OWASP ASVS L2 baseline.
- Strict CSP, HSTS, secure cookies, SameSite=strict on dashboard.
- Input validation everywhere; output encoding in React by default.
- **Prompt-injection defenses:** system prompt hardening, tool-use allowlists, content-origin tags (`<untrusted>...</untrusted>`), no raw user content in tool arguments.

### 15.4 Supply Chain
- Dependabot + `pip-audit` + `npm audit --production` in CI.
- SBOM (CycloneDX) generated per build.
- Container images signed with **cosign**; verified at deploy.

### 15.5 Secrets
- No secrets in code. All via Supabase Vault / Railway secrets / Vercel env.
- Rotation policy: 90 days for API keys, immediate on employee offboarding.

### 15.6 Compliance-ready
- Data residency labeling per tenant.
- DSAR endpoints (`/v1/dsar/export`, `/v1/dsar/delete`).
- SOC 2 controls tracked in `docs/compliance/`.

---

## 16. Logging Strategy

- **Structured JSON logs** (one event per line) via `structlog`.
- Mandatory fields: `ts`, `level`, `service`, `env`, `trace_id`, `span_id`, `tenant_id`, `user_id`, `request_id`, `event`.
- **PII scrubbing** at logger level (regex + Presidio).
- **Log levels**: DEBUG (dev only), INFO (state changes), WARN (recoverable), ERROR (needs attention), CRITICAL (paging).
- **Sinks**: stdout → Railway → Logtail/BetterStack; 30-day hot, 1-year cold.
- **Correlation**: `trace_id` propagates from Next.js → FastAPI → n8n → LLM adapter → downstream.

---

## 17. Deployment Strategy

| Component | Platform | Notes |
|---|---|---|
| Frontend | **Vercel** | Preview per PR, prod on main |
| Backend API | **Railway** | Docker; horizontal replicas; health-checked |
| Workers (Arq) | **Railway** | Separate service, autoscale by queue depth |
| n8n | **Railway** (self-hosted) | Postgres-backed; versioned workflow exports in repo |
| Database + Auth + Storage | **Supabase** | Point-in-time recovery on; daily encrypted backups |
| Cache/queue | **Redis** (Railway addon) | Rate limits + Arq |

### 17.1 Environments
`local → dev → staging → prod`, each with isolated Supabase project.

### 17.2 Release Strategy
- **Trunk-based** development.
- **Blue-green** for API; migrations gated behind expand-contract pattern.
- **Feature flags** (Unleash-compatible) for risky changes.
- **Canary** LLM prompt versions: 5% → 25% → 100% with automatic rollback on eval drop.

---

## 18. Monitoring Strategy

### 18.1 Signals (Golden + Domain)
- **RED**: request rate, error rate, duration — per endpoint.
- **USE**: CPU/mem/queue depth — per service.
- **Domain KPIs**: messages ingested, extraction success %, avg confidence, action success %, LLM cost per tenant.

### 18.2 Stack
- **OpenTelemetry** SDK in FastAPI & Next.js → OTLP → **Grafana Cloud** (Tempo + Mimir + Loki).
- **Sentry** for exceptions (frontend + backend).
- **Supabase built-in** DB metrics + `pg_stat_statements`.
- **Synthetic checks** (Checkly) for critical user journeys.

### 18.3 Alerting (SLO-based)
- API availability SLO: 99.9% monthly.
- Extraction latency p95 < 8s.
- Alerts routed via **PagerDuty** with burn-rate policies.

---

## 19. CI/CD Strategy

### 19.1 CI (GitHub Actions)
Pipelines per app:
1. **Lint** (ruff, black, eslint, prettier, tsc --noEmit).
2. **Unit tests** (pytest, vitest) with coverage ≥ 85%.
3. **Type check** (mypy --strict, tsc).
4. **Security** (bandit, semgrep, pip-audit, npm audit, gitleaks).
5. **Contract tests** (Schemathesis vs OpenAPI).
6. **Build container** → push to GHCR with cosign.
7. **SBOM** upload.
8. **Preview deploy** (Vercel preview, ephemeral Railway env for backend PRs).
9. **DB migration dry-run** against staging clone.

### 19.2 CD
- Merge to `main` → auto-deploy staging → E2E (Playwright) → manual approval → prod.
- **Migrations** via Supabase CLI, forward-only, expand→migrate→contract phases.
- **n8n workflows** deployed by import script from `automation/n8n/*.json`.
- **Rollback**: image tag pinning + DB migration back-out only via reverse migration or restore-from-PITR.

---

## 20. Testing Strategy

### 20.1 Test Pyramid

| Level | Scope | Tooling | Target |
|---|---|---|---|
| Unit | Domain + Use Cases | pytest, vitest | 85%+ coverage on core |
| Integration | Adapters against fakes | pytest + testcontainers | All adapters |
| Contract | HTTP API vs OpenAPI | Schemathesis, Pact | 100% endpoints |
| E2E | Full user journey | Playwright | 12 critical flows |
| AI Eval | Prompt regression | promptfoo + golden set | Per prompt version |
| Load | Ingestion + AI pipeline | k6 | 200 RPS burst, 50 RPS sustained |
| Security | SAST/DAST | Semgrep, ZAP baseline | Per release |
| Chaos | DLQ redrive, provider outage | Toxiproxy | Weekly game day |

### 20.2 AI-Specific Testing
- **Golden dataset** of ~500 labeled messages per tenant vertical.
- **Deterministic seeds** where supported; snapshot tests on structured output.
- **Guardrail tests**: prompt-injection corpus must not trigger tool calls.
- **Cost regression**: token usage per fixture tracked; PR fails if >15% drift.

### 20.3 Test Data
- Synthetic PII-safe fixtures generated by Faker.
- No production data in lower environments.

---

## Non-Functional Requirements Summary

| NFR | Target |
|---|---|
| Availability | 99.9% API monthly |
| Ingestion latency (accept) | p95 < 300ms |
| Extraction latency (end-to-end) | p95 < 8s |
| Throughput | 200 msgs/sec burst per region |
| RPO / RTO | 15 min / 60 min |
| Data residency | US + EU on request |
| Accessibility | WCAG 2.1 AA on dashboard |

---

## Open Questions Awaiting Approval

1. **Confirm modular monolith** vs. immediate microservices split — recommendation: monolith now.
2. **Confirm n8n self-hosted on Railway** vs. n8n Cloud — recommendation: self-hosted (data locality + cost).
3. **Confirm Supabase Vault** for OAuth refresh token storage — recommendation: yes.
4. **Approve PII redaction default** (opt-out) vs. explicit opt-in — recommendation: opt-in for phase 1.
5. **Approve budget guardrails** per-tenant token cap with soft/hard limits.

---

## Sign-off Checklist

- [ ] Architecture reviewed by Principal Engineer
- [ ] Security review (threat model, data flow)
- [ ] SRE review (SLOs, runbooks placeholders)
- [ ] Product review (user & admin journeys)
- [ ] Legal/Compliance review (data handling, DSAR)

---

**Awaiting approval.** No implementation code will be produced until this document is signed off.
