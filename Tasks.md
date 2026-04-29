# ERP Platform — Task List
**Last Updated:** April 28, 2026  
**Architecture Reference:** `architecture_specifics.md`

---

## How to Read This Document
- Tasks are grouped by **service/area**, ordered by **dependency**.
- Each task maps to one deployable unit from the Container App Map in `architecture_specifics.md`.
- `Depends On` references task IDs within this file.
- **Est. (days)** = estimated effort in **person-days** (one engineer, focused work). Use it to derive headcount:
  `Required FTEs ≈ (Group total days) / (Calendar working days available × focus factor 0.7)`.
- **Group totals** are listed at the bottom of each table; the **grand total** is at the end of the document.

---

## Phase 0 — Infrastructure & DevOps Setup
> Area: **DevOps** | Repo: `erp-infrastructure`

| ID | Task | Deliverable | Depends On | Est. (days) |
|----|------|-------------|------------|-------------|
| INFRA-01 | Create Azure Resource Groups (dev / staging / prod) | Azure RG per environment | — | 1 |
| INFRA-02 | Provision Azure Container Registry (ACR) | ACR instance with repositories | INFRA-01 | 1 |
| INFRA-03 | Provision Azure Container Apps Environment (dev) | ACA Environment — dev | INFRA-02 | 2 |
| INFRA-04 | Provision Azure Container Apps Environment (staging / prod) | ACA Environment — staging, prod | INFRA-03 | 2 |
| INFRA-05 | Provision Azure Database for PostgreSQL (Flexible Server) | PostgreSQL instance, Multi-AZ (prod) | INFRA-01 | 3 |
| INFRA-06 | Provision Azure Cache for Redis (clustered) | Redis instance | INFRA-01 | 1 |
| INFRA-07 | Provision Azure Service Bus (queues + topics) | Service Bus namespace with all queues from architecture spec | INFRA-01 | 2 |
| INFRA-08 | Provision Azure Key Vault (one per environment) | Key Vault instances | INFRA-01 | 1 |
| INFRA-09 | Provision Azure API Management (APIM) | APIM instance, JWT validation policy, routing rules to FastAPI services | INFRA-01 | 4 |
| INFRA-10 | Configure Azure Front Door (CDN + WAF + TLS) | Front Door profile routing `/api/*` to APIM and static assets to frontend apps | INFRA-09 | 2 |
| INFRA-11 | Configure Azure Entra ID (app registration, scopes, roles, MFA / Conditional Access) | App registration; roles: `legal.reader`, `legal.editor`, `marketing.admin`, `accounting.editor`, etc. | INFRA-01 | 3 |
| INFRA-12 | Provision Azure Blob Storage (file uploads) | Storage account + containers per module | INFRA-01 | 1 |
| INFRA-13 | Provision Azure OpenAI Service (GPT-4o, GPT-4o-mini, text-embedding-3-large) | Azure OpenAI endpoints via private endpoint in VNet | INFRA-01 | 2 |
| INFRA-14 | Setup GitHub Actions: base reusable workflow (build → push ACR → deploy ACA) | `.github/workflows/deploy-template.yml` in each repo | INFRA-02, INFRA-03 | 3 |
| INFRA-15 | Configure GitHub Environments (dev / staging / prod) with protection rules | GitHub Environments with reviewer sign-off for prod | INFRA-14 | 1 |
| INFRA-16 | Setup Azure Monitor + Log Analytics workspace | Log Analytics workspace, container log routing | INFRA-03 | 2 |
| INFRA-17 | Setup Grafana dashboards (CPU, memory, replica count, latency) | Grafana dashboards via Azure Monitor data source | INFRA-16 | 3 |
| INFRA-18 | Configure OpenTelemetry → Azure Application Insights (all FastAPI services) | Application Insights resource; OTEL exporter config in `erp-core` | INFRA-16 | 2 |
| INFRA-19 | Setup Azure Budget Alerts (80% and 100% spend) | Budget alert rules | INFRA-01 | 1 |
| INFRA-20 | Setup Docker Compose for local development (PostgreSQL, RabbitMQ, Redis) | `docker-compose.dev.yml` shared across repos | — | 2 |
| INFRA-21 | Write Bicep / Terraform for all Azure resources (IaC) | `erp-infrastructure` repo with complete IaC | INFRA-01–INFRA-13 | 8 |
| **Group total** | | | | **47** |

---

## Phase 1a — Shared Packages

### `erp-core` Python Package
> Area: **Backend** | Repo: `erp-core`

| ID | Task | Deliverable | Depends On | Est. (days) |
|----|------|-------------|------------|-------------|
| CORE-01 | Initialize `erp-core` Python package (pyproject.toml, src layout) | `erp_core/` package skeleton, GitHub Packages publish pipeline | — | 1 |
| CORE-02 | Implement `erp_core/config.py` — Pydantic `BaseSettings` loading from Azure Key Vault + env vars | `erp_core/config.py` | CORE-01 | 2 |
| CORE-03 | Implement `erp_core/logging.py` — structured JSON logging with `correlation_id`, `service_name`, `user_id`; OpenTelemetry trace injection | `erp_core/logging.py` | CORE-01 | 3 |
| CORE-04 | Implement `erp_core/exceptions.py` — shared exception hierarchy (`AppError`, `NotFoundError`, `ConflictError`, `ValidationError`) mapped to HTTP status codes | `erp_core/exceptions.py` | CORE-01 | 1 |
| CORE-05 | Implement `erp_core/middleware/` — request ID injection, structured access logging, error response formatting | `erp_core/middleware/` | CORE-03, CORE-04 | 3 |
| CORE-06 | Implement `erp_core/interfaces/` — ABCs: `BaseService`, `BaseRepository`, `BaseEventPublisher`, `BaseAIAgent` | `erp_core/interfaces/` | CORE-01 | 2 |
| CORE-07 | Implement `erp_core/messaging/` — Azure Service Bus publisher/subscriber base classes; RabbitMQ adapter for local dev; switchable via `MESSAGING_BACKEND` env var | `erp_core/messaging/` | CORE-06 | 5 |
| CORE-08 | Implement `erp_core/http_client.py` — shared `httpx` async client with retry policy, timeout config, `X-Correlation-ID` header propagation | `erp_core/http_client.py` | CORE-03 | 2 |
| CORE-09 | Implement `erp_core/health.py` — standard `/health` and `/ready` endpoint factory | `erp_core/health.py` | CORE-01 | 1 |
| CORE-10 | Implement `erp_core/security.py` — JWT claim extraction helpers (user ID, roles, module access) from APIM-forwarded headers | `erp_core/security.py` | CORE-01 | 2 |
| CORE-11 | Implement `erp_core/constants.py` — shared enums: `ModuleName`, `AuditAction`, `EventType` | `erp_core/constants.py` | CORE-01 | 1 |
| CORE-12 | Write unit tests for all `erp-core` modules (≥90% coverage) | `tests/` in `erp-core` repo | CORE-02–CORE-11 | 5 |
| CORE-13 | Publish `erp-core` **v1.0.0** to GitHub Packages (without Django client — unblocks Django + FastAPI scaffolding) | GitHub Packages private PyPI entry | CORE-12 | 1 |
| CORE-14 | Implement `erp_core/clients/django_client.py` — typed async client for the Django internal REST API (auto-generated from Django's OpenAPI schema using `openapi-python-client`); wraps `erp_core/http_client.py` with managed-identity auth, retries, and `BaseRepository` interface compliance | `erp_core/clients/django_client.py`; CI step that regenerates the client when the Django OpenAPI schema changes | CORE-08, DB-17 | 3 |
| CORE-15 | Publish `erp-core` **v1.1.0** to GitHub Packages (includes Django client) — required by all FastAPI services | GitHub Packages release | CORE-14 | 1 |
| **Group total** | | | | **33** |

### `erp-contracts` Pydantic Package
> Area: **Backend** | Repo: `erp-contracts`

| ID | Task | Deliverable | Depends On | Est. (days) |
|----|------|-------------|------------|-------------|
| CONT-01 | Initialize `erp-contracts` Python package structure | `erp_contracts/` package skeleton | CORE-01 | 1 |
| CONT-02 | Define REST contracts — Legal module DTOs (request/response Pydantic v2 models) | `erp_contracts/legal/` | CONT-01 | 3 |
| CONT-03 | Define REST contracts — Marketing module DTOs | `erp_contracts/marketing/` | CONT-01 | 3 |
| CONT-04 | Define REST contracts — Accounting module DTOs | `erp_contracts/accounting/` | CONT-01 | 4 |
| CONT-05 | Define AI agent I/O schemas (`IngestionRequest`, `AnalysisResult`, `GenerationRequest`, `ClassificationRequest`, `SearchRequest`, `SearchResult`) | `erp_contracts/ai/` | CONT-01 | 3 |
| CONT-06 | Define async event schemas for Service Bus messages | `erp_contracts/events/` | CONT-01 | 2 |
| CONT-07 | Setup Pact consumer-driven contract tests in CI | `.github/workflows/pact.yml`; Pact Broker config | CONT-02–CONT-06 | 3 |
| CONT-08 | Publish `erp-contracts` v1.0.0 to GitHub Packages | GitHub Packages private PyPI entry | CONT-07 | 1 |
| **Group total** | | | | **20** |

---

## Phase 1b — Data Layer (Django)
> Area: **Database + Backend** | Repo: `erp-django-core`

| ID | Task | Deliverable | Depends On | Est. (days) |
|----|------|-------------|------------|-------------|
| DB-01 | Setup PostgreSQL 16 + pgvector extension (Docker Compose local dev) | `docker-compose.dev.yml` DB service; `CREATE EXTENSION vector` in init script | INFRA-20 | 1 |
| DB-02 | Design and create `core` schema (Organization, Role, User, Permission tables) | Django models + migration: `core_organization`, `core_role`, `core_user_role`, `core_permission` | DB-01 | 3 |
| DB-03 | Design and create `legal` schema (LegalDossier, LegalCase, LegalDeadline, LawAct, Attachment, DocumentVersion) | Django models + migration for `legal` schema | DB-02 | 4 |
| DB-04 | Design and create `marketing` schema (Campaign, Lead, Ticket, Operator, ContactPerson) | Django models + migration for `marketing` schema | DB-02 | 3 |
| DB-05 | Design and create `accounting` schema (Account, JournalEntry, JournalEntryLine, Period, TaxDeclaration) | Django models + migration for `accounting` schema | DB-02 | 4 |
| DB-06 | Add `AuditLog` table with JSON diff (shared, under `core` schema) | `core_audit_log` model + migration | DB-02 | 2 |
| DB-07 | Add `Attachment` + `DocumentVersion` tables (shared, under `core` schema) | `core_attachment`, `core_document_version` models + migration | DB-02 | 2 |
| DB-08 | Enable pgvector on PostgreSQL; add embedding columns and vector indexes to relevant tables | `CREATE INDEX ivfflat` on embedding columns | DB-01 | 2 |
| DB-09 | Create indexes for all foreign keys across all schemas | Indexes on all `*_id` columns | DB-02–DB-05 | 1 |
| DB-10 | Initialize `erp-django-core` Django project | `erp-django-core/` project skeleton; Gunicorn config; `pyproject.toml` with `erp-core` dependency | CORE-13 | 2 |
| DB-11 | Configure Django per-schema database routing (separate schema per module in single PostgreSQL instance) | `db_router.py`; Django settings for `legal`, `marketing`, `accounting`, `core` schemas | DB-10, DB-02 | 3 |
| DB-12 | Implement `Organization`, `Department`, `Team` models in `core` app | `core/models.py` | DB-10 | 2 |
| DB-13 | Implement Django admin panel (internal management UI) | `admin.py` registrations for all models | DB-10 | 3 |
| DB-14 | Implement RBAC models aligned with Azure Entra ID roles (read roles from JWT claims, no local password management) | `rbac/models.py`, `rbac/service.py` — role-to-permission mapping only | DB-10, INFRA-11 | 4 |
| DB-15 | Implement audit log middleware for Django admin actions | `middleware/audit.py` | DB-06 | 2 |
| DB-16 | Implement Django **internal REST API** consumed by all FastAPI services (sole DB access path — FastAPI never connects to PostgreSQL directly). Endpoints follow the pattern `/internal/v1/<schema>/<resource>` and return Pydantic-compatible JSON. Each module (legal, marketing, accounting, core) gets its own DRF viewset module. | `internal_api/` Django app; viewsets for all module models; URL routing under `/internal/v1/`; OpenAPI schema published at `/internal/v1/openapi.json` | DB-10, DB-02, DB-03, DB-04, DB-05 | 8 |
| DB-17 | Configure DRF + `drf-spectacular` for the **internal API** (OpenAPI 3 schema generation; schema is the source of truth for the `erp-core` Django client) | DRF config; `drf-spectacular` settings; OpenAPI schema validated in CI; schema artifact published to GitHub Packages on Django release | DB-16 | 3 |
| DB-18 | Seed permissions command (all modules) | `management/commands/seed_permissions.py` | DB-14 | 1 |
| DB-19 | Configure Azure Database for PostgreSQL connection (SSL, connection pooling via pgBouncer) | Django `DATABASES` settings with Azure credentials from Key Vault | DB-10, INFRA-05, INFRA-08 | 2 |
| DB-20 | Configure automated backup validation + point-in-time restore test + geo-redundant backup | Restore test runbook; `docs/runbooks/restore.md` | DB-19 | 2 |
| DB-21 | Write Dockerfile for `erp-django-core` (internal only, Gunicorn) | `Dockerfile` | DB-10 | 1 |
| DB-22 | Setup GitHub Actions CI/CD pipeline for `erp-django-core` (publishes OpenAPI schema artifact on release) | `.github/workflows/ci-cd.yml` (lint → test → build → push ACR → deploy ACA internal → publish OpenAPI schema) | DB-21, INFRA-14, CORE-13 | 2 |
| DB-23 | Implement GDPR compliance: PII tagging in models, soft-delete + anonymization helpers, region/data-residency configuration | `core/gdpr.py`; PII metadata on model fields; anonymize management command | DB-02–DB-05 | 3 |
| DB-24 | Implement **service-to-service authentication** for the Django internal API (Azure Managed Identity tokens validated by Django middleware; only ACA Container Apps in the same VNet with the `internal-api-caller` role can call it) | `internal_api/auth.py` middleware; Entra ID app registration for the internal API resource; allowed-callers list per environment | DB-16, INFRA-08, INFRA-11 | 3 |
| DB-25 | Implement **schema drift / contract test** in CI: every PR runs the FastAPI integration suite against a freshly-migrated Django container to detect breaking schema changes before merge | `.github/workflows/schema-contract.yml`; shared docker-compose for the test job | DB-16, DB-22 | 2 |
| **Group total** | | | | **62** |

---

## Phase 2 — Module Backend Services (FastAPI)

### Legal FastAPI Service
> Area: **Backend** | Repo: `erp-legal-api`

| ID | Task | Deliverable | Depends On | Est. (days) |
|----|------|-------------|------------|-------------|
| BE-L-01 | Initialize `erp-legal-api` FastAPI project (data access exclusively via `erp_core.clients.django_client` — **no direct DB connection**) | `services/legal-api/` skeleton; `pyproject.toml` with `erp-core>=1.1.0` + `erp-contracts`; lint rule blocking `psycopg`/`sqlalchemy` imports | CORE-15, CONT-08 | 1 |
| BE-L-02 | Implement Legal Dossier CRUD endpoints (delegates persistence to Django internal API) | `routes/dossiers.py` | BE-L-01, DB-16 | 3 |
| BE-L-03 | Implement Legal Case management endpoints (via Django internal API) | `routes/cases.py` | BE-L-01, DB-16 | 3 |
| BE-L-04 | Implement Legal Deadline tracking endpoints (via Django internal API) | `routes/deadlines.py` | BE-L-01, DB-16 | 2 |
| BE-L-05 | Implement Law Acts / Knowledge Base endpoints (via Django internal API) | `routes/law_acts.py` | BE-L-01, DB-16 | 2 |
| BE-L-06 | Implement document upload → trigger `ai.ingestion.requests` Service Bus message | `routes/documents.py`; Service Bus publisher | BE-L-01, CORE-07, INFRA-07 | 3 |
| BE-L-07 | Implement AI Search integration (publish to `ai.search.requests`, consume results) | `routes/search.py` | BE-L-01, CORE-07 | 3 |
| BE-L-08 | Implement audit logging for all write operations (via `erp-core` middleware) | Audit events in `core_audit_log` | BE-L-01, DB-06 | 2 |
| BE-L-09 | Write unit + integration tests (pytest + httpx) | `tests/` (≥80% coverage) | BE-L-01–BE-L-08 | 5 |
| BE-L-10 | Write Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | BE-L-01, INFRA-14 | 2 |
| **Group total** | | | | **26** |

### Marketing FastAPI Service
> Area: **Backend** | Repo: `erp-marketing-api`

| ID | Task | Deliverable | Depends On | Est. (days) |
|----|------|-------------|------------|-------------|
| BE-M-01 | Initialize `erp-marketing-api` FastAPI project (data access exclusively via `erp_core.clients.django_client` — **no direct DB connection**) | `services/marketing-api/` skeleton; `pyproject.toml` with `erp-core>=1.1.0` | CORE-15, CONT-08 | 1 |
| BE-M-02 | Implement Campaign management endpoints (CRUD, via Django internal API) | `routes/campaigns.py` | BE-M-01, DB-16 | 3 |
| BE-M-03 | Implement Lead management + lead-to-client conversion endpoints (via Django internal API) | `routes/leads.py` | BE-M-01, DB-16 | 3 |
| BE-M-04 | Implement Ticket / Support request endpoints (via Django internal API) | `routes/tickets.py` | BE-M-01, DB-16 | 3 |
| BE-M-05 | Implement Operator assignment + routing logic (via Django internal API) | `services/operator_service.py` | BE-M-01, DB-16 | 3 |
| BE-M-06 | Implement campaign performance analysis → trigger `ai.analysis.requests` | `routes/analytics.py`; Service Bus publisher | BE-M-01, CORE-07 | 3 |
| BE-M-07 | Implement ETL data import (bulk lead/campaign import) | `services/etl_service.py` | BE-M-01 | 4 |
| BE-M-08 | Write unit + integration tests | `tests/` (≥80% coverage) | BE-M-01–BE-M-07 | 5 |
| BE-M-09 | Write Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | BE-M-01, INFRA-14 | 2 |
| **Group total** | | | | **27** |

### Accounting FastAPI Service
> Area: **Backend** | Repo: `erp-accounting-api`

| ID | Task | Deliverable | Depends On | Est. (days) |
|----|------|-------------|------------|-------------|
| BE-A-01 | Initialize `erp-accounting-api` FastAPI project (data access exclusively via `erp_core.clients.django_client` — **no direct DB connection**) | `services/accounting-api/` skeleton; `pyproject.toml` with `erp-core>=1.1.0` | CORE-15, CONT-08 | 1 |
| BE-A-02 | Create Chart of Accounts (NSS standard) — persisted via Django internal API | `routes/accounts.py`; `data/chart_of_accounts.json`; seed command | BE-A-01, DB-16 | 3 |
| BE-A-03 | Implement Journal Entry + Journal Entry Lines endpoints (via Django internal API) | `routes/journal_entries.py`; `models/journal_entry.py` | BE-A-01, DB-16 | 4 |
| BE-A-04 | Implement double-entry validation service | `services/validation_service.py` (debit == credit check) | BE-A-03 | 2 |
| BE-A-05 | Implement Accounting Period + Period Closure (via Django internal API) | `routes/periods.py`; `models/period.py` | BE-A-01, DB-16 | 3 |
| BE-A-06 | Implement financial statement generation (Balance Sheet, P&L, Cash Flow) | `services/financial_statement_service.py` | BE-A-03, BE-A-05 | 5 |
| BE-A-07 | Implement VAT + corporate tax declaration generation | `services/tax_service.py` | BE-A-03 | 4 |
| BE-A-08 | Implement audit plan + findings endpoints (via Django internal API) | `routes/audit.py` | BE-A-01, DB-16 | 3 |
| BE-A-09 | Implement NAP inspection control dossier | `services/control_dossier_service.py` | BE-A-08 | 3 |
| BE-A-10 | Implement risk alert engine | `services/risk_alert_service.py` | BE-A-06 | 3 |
| BE-A-11 | Trigger AI generation for financial narratives → `ai.generation.requests` | Service Bus publisher in `services/narrative_service.py` | BE-A-01, CORE-07 | 3 |
| BE-A-12 | Write unit + integration tests | `tests/` (≥80% coverage) | BE-A-01–BE-A-11 | 5 |
| BE-A-13 | Write Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | BE-A-01, INFRA-14 | 2 |
| **Group total** | | | | **41** |

### Celery Background Worker
> Area: **Backend** | Repo: `erp-celery-worker`

| ID | Task | Deliverable | Depends On | Est. (days) |
|----|------|-------------|------------|-------------|
| CEL-01 | Initialize Celery worker service | `services/celery-worker/` with `pyproject.toml` | CORE-13 | 1 |
| CEL-02 | Implement scheduled background tasks (report generation, data cleanup, notifications) | `tasks/` directory | CEL-01 | 3 |
| CEL-03 | Configure Azure Service Bus as Celery broker (prod) / RabbitMQ (local dev) | `celeryconfig.py` with `MESSAGING_BACKEND` switch | CEL-01, INFRA-07 | 2 |
| CEL-04 | Write Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | CEL-01, INFRA-14 | 1 |
| **Group total** | | | | **7** |

---

## Phase 3 — AI Services

### AI Orchestrator (LangGraph Supervisor)
> Area: **Backend** | Repo: `erp-ai-orchestrator`

| ID | Task | Deliverable | Depends On | Est. (days) |
|----|------|-------------|------------|-------------|
| AI-01 | Initialize AI Orchestrator service (LangGraph Supervisor) | `services/ai-orchestrator/` skeleton; LangGraph dependency | CORE-13, CONT-08 | 2 |
| AI-02 | Implement task routing logic — routes incoming `ai.*` requests to correct agent via Service Bus | `orchestrator/supervisor.py`; routing rules per `EventType` | AI-01, INFRA-07 | 4 |
| AI-03 | Implement result aggregation — consumes `ai.results` topic, returns to requesting FastAPI service | `orchestrator/aggregator.py` | AI-02 | 3 |
| AI-04 | Setup LangSmith tracing integration for orchestrator | LangSmith config in `erp_core/logging.py` (AI trace extensions) | AI-01 | 1 |
| AI-05 | Write Dockerfile + GitHub Actions CI/CD pipeline (scale-to-zero in ACA) | `Dockerfile`, `.github/workflows/ci-cd.yml` | AI-01, INFRA-14 | 2 |
| **Group total** | | | | **12** |

### AI Ingestion Agent
> Area: **Backend** | Repo: `erp-ai-ingestion`

| ID | Task | Deliverable | Depends On | Est. (days) |
|----|------|-------------|------------|-------------|
| AI-ING-01 | Initialize Ingestion Agent service | `services/ai-ingestion/` skeleton | CORE-13, CONT-08 | 1 |
| AI-ING-02 | Implement file parsing: PDF, DOCX, XLSX text extraction | `parsers/` (PyMuPDF / python-docx / openpyxl) | AI-ING-01 | 5 |
| AI-ING-03 | Implement OCR for images using GPT-4o Vision | `ocr/gpt4o_vision.py` | AI-ING-01, INFRA-13 | 3 |
| AI-ING-04 | Implement text chunking + embedding generation (`text-embedding-3-large` via Azure OpenAI) | `embeddings/chunker.py`, `embeddings/embedder.py` | AI-ING-01, INFRA-13 | 4 |
| AI-ING-05 | Store embeddings in pgvector (MVP) | `storage/pgvector_store.py` | AI-ING-04, DB-08 | 2 |
| AI-ING-06 | Subscribe to `ai.ingestion.requests` Service Bus queue | `consumers/ingestion_consumer.py` | AI-ING-01, CORE-07 | 2 |
| AI-ING-07 | Write Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | AI-ING-01, INFRA-14 | 1 |
| **Group total** | | | | **18** |

### AI Analysis Agent
> Area: **Backend** | Repo: `erp-ai-analysis`

| ID | Task | Deliverable | Depends On | Est. (days) |
|----|------|-------------|------------|-------------|
| AI-ANA-01 | Initialize Analysis Agent service | `services/ai-analysis/` skeleton | CORE-13, CONT-08 | 1 |
| AI-ANA-02 | Implement clause risk scoring (legal documents) using GPT-4o | `analyzers/clause_risk.py` | AI-ANA-01, INFRA-13 | 4 |
| AI-ANA-03 | Implement financial anomaly detection using GPT-4o | `analyzers/financial_anomaly.py` | AI-ANA-01, INFRA-13 | 4 |
| AI-ANA-04 | Implement campaign performance insights using GPT-4o | `analyzers/campaign_insights.py` | AI-ANA-01, INFRA-13 | 3 |
| AI-ANA-05 | Subscribe to `ai.analysis.requests` Service Bus queue; publish to `ai.results` | `consumers/analysis_consumer.py` | AI-ANA-01, CORE-07 | 2 |
| AI-ANA-06 | Write Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | AI-ANA-01, INFRA-14 | 1 |
| **Group total** | | | | **15** |

### AI Generation Agent
> Area: **Backend** | Repo: `erp-ai-generation`

| ID | Task | Deliverable | Depends On | Est. (days) |
|----|------|-------------|------------|-------------|
| AI-GEN-01 | Initialize Generation Agent service | `services/ai-generation/` skeleton | CORE-13, CONT-08 | 1 |
| AI-GEN-02 | Implement legal document summary generation (GPT-4o) | `generators/legal_summary.py` | AI-GEN-01, INFRA-13 | 3 |
| AI-GEN-03 | Implement marketing copy generation (GPT-4o) | `generators/marketing_copy.py` | AI-GEN-01, INFRA-13 | 3 |
| AI-GEN-04 | Implement financial narrative generation (GPT-4o) | `generators/financial_narrative.py` | AI-GEN-01, INFRA-13 | 3 |
| AI-GEN-05 | Implement email draft generation (GPT-4o) | `generators/email_draft.py` | AI-GEN-01, INFRA-13 | 3 |
| AI-GEN-06 | Create and manage prompt templates (per persona / use case) | `data/prompts/` (≥10 prompt templates covering all modules) | AI-GEN-01 | 4 |
| AI-GEN-07 | Subscribe to `ai.generation.requests`; publish to `ai.results` | `consumers/generation_consumer.py` | AI-GEN-01, CORE-07 | 2 |
| AI-GEN-08 | Implement AG-UI (SSE streaming for real-time agent output in UI) | `agui/emitter.py`; SSE endpoint | AI-GEN-01 | 3 |
| AI-GEN-09 | Write Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | AI-GEN-01, INFRA-14 | 1 |
| **Group total** | | | | **23** |

### AI Classification Agent
> Area: **Backend** | Repo: `erp-ai-classification`

| ID | Task | Deliverable | Depends On | Est. (days) |
|----|------|-------------|------------|-------------|
| AI-CLS-01 | Initialize Classification Agent service | `services/ai-classification/` skeleton | CORE-13, CONT-08 | 1 |
| AI-CLS-02 | Implement document categorization using GPT-4o-mini | `classifiers/document_classifier.py` | AI-CLS-01, INFRA-13 | 3 |
| AI-CLS-03 | Implement task routing / intent detection (GPT-4o-mini) | `classifiers/intent_detector.py` | AI-CLS-01, INFRA-13 | 3 |
| AI-CLS-04 | Subscribe to `ai.classification.requests`; publish to `ai.results` | `consumers/classification_consumer.py` | AI-CLS-01, CORE-07 | 2 |
| AI-CLS-05 | Write Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | AI-CLS-01, INFRA-14 | 1 |
| **Group total** | | | | **10** |

### AI Search Agent
> Area: **Backend** | Repo: `erp-ai-search`

| ID | Task | Deliverable | Depends On | Est. (days) |
|----|------|-------------|------------|-------------|
| AI-SRC-01 | Initialize Search Agent service | `services/ai-search/` skeleton | CORE-13, CONT-08 | 1 |
| AI-SRC-02 | Implement vector similarity search (pgvector MVP) | `search/pgvector_search.py` | AI-SRC-01, DB-08 | 3 |
| AI-SRC-03 | Implement hybrid search (vector + keyword) | `search/hybrid_search.py` | AI-SRC-02 | 4 |
| AI-SRC-04 | Implement ranked, cited result response formatting | `search/result_formatter.py` | AI-SRC-02 | 2 |
| AI-SRC-05 | Subscribe to `ai.search.requests`; publish to `ai.results` | `consumers/search_consumer.py` | AI-SRC-01, CORE-07 | 2 |
| AI-SRC-06 | Implement token usage tracking + budget enforcement (Redis-backed) | `services/token_service.py` | AI-SRC-01, INFRA-06 | 3 |
| AI-SRC-07 | Write Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | AI-SRC-01, INFRA-14 | 1 |
| **Group total** | | | | **16** |

---

## Phase 4 — Frontend

### Shell Frontend (Micro-Frontend Host)
> Area: **Frontend** | Repo: `erp-shell-frontend`

| ID | Task | Deliverable | Depends On | Est. (days) |
|----|------|-------------|------------|-------------|
| FE-SH-01 | Initialize Shell App (React.js + TypeScript + Webpack Module Federation) | `frontends/erp-shell/` with Module Federation host config (`singleton: true`) | — | 2 |
| FE-SH-02 | Setup shared design system + component library (shared via Module Federation) | `src/design-system/` (buttons, forms, typography, colors) | FE-SH-01 | 5 |
| FE-SH-03 | Implement Azure Entra ID login flow (redirect to Entra ID; receive JWT; store in memory — not localStorage) | `src/auth/authService.ts`; MSAL.js integration | FE-SH-01, INFRA-11 | 3 |
| FE-SH-04 | Implement JWT token sharing to module remotes via Module Federation shared state | `src/auth/store.ts`; shared auth context | FE-SH-03 | 2 |
| FE-SH-05 | Implement global navigation + sidebar (module-aware, role-based visibility) | `src/components/Navigation.tsx` | FE-SH-02 | 3 |
| FE-SH-06 | Setup routing for remote module lazy loading + module discovery | `src/router/index.tsx`; `src/modules/registry.ts` | FE-SH-01 | 3 |
| FE-SH-07 | Create shared DataTable component (TanStack Table) | `src/components/DataTable.tsx` | FE-SH-02 | 3 |
| FE-SH-08 | Create shared Form builder (React Hook Form + Zod) | `src/components/FormBuilder.tsx` | FE-SH-02 | 3 |
| FE-SH-09 | Create global Notification Center | `src/components/NotificationCenter.tsx` | FE-SH-02 | 2 |
| FE-SH-10 | Implement logout + session expiry handling | `src/auth/authService.ts` (logout, token refresh via Entra ID) | FE-SH-03 | 2 |
| FE-SH-11 | Write Jest + React Testing Library unit tests | `src/__tests__/` | FE-SH-01–FE-SH-10 | 5 |
| FE-SH-12 | Write Nginx config + Dockerfile for static React build | `nginx.conf`, `Dockerfile` | FE-SH-01 | 1 |
| FE-SH-13 | Setup GitHub Actions CI/CD pipeline | `.github/workflows/ci-cd.yml` | FE-SH-01, INFRA-14 | 1 |
| **Group total** | | | | **35** |

### Legal Frontend (Micro-Frontend Remote)
> Area: **Frontend** | Repo: `erp-legal-frontend`

| ID | Task | Deliverable | Depends On | Est. (days) |
|----|------|-------------|------------|-------------|
| FE-L-01 | Initialize Legal remote app (React.js + TypeScript + Webpack Module Federation remote config) | `frontends/erp-legal/` | FE-SH-01 | 1 |
| FE-L-02 | Implement Legal Dossier list + detail pages | `src/pages/Dossiers/` | FE-L-01, BE-L-02 | 4 |
| FE-L-03 | Implement Legal Case management UI | `src/pages/Cases/` | FE-L-01, BE-L-03 | 4 |
| FE-L-04 | Implement Legal Deadline calendar / tracker UI | `src/pages/Deadlines/` | FE-L-01, BE-L-04 | 3 |
| FE-L-05 | Implement Law Acts / Knowledge Base search UI with AI chat (SSE streaming) | `src/pages/Search/`; SSE integration with AI generation agent | FE-L-01, BE-L-07, AI-GEN-08 | 5 |
| FE-L-06 | Implement document upload UI (triggers ingestion pipeline) | `src/components/DocumentUpload.tsx` | FE-L-01, BE-L-06 | 3 |
| FE-L-07 | Write Nginx config + Dockerfile + GitHub Actions CI/CD | `nginx.conf`, `Dockerfile`, `.github/workflows/ci-cd.yml` | FE-L-01, INFRA-14 | 2 |
| **Group total** | | | | **22** |

### Marketing Frontend (Micro-Frontend Remote)
> Area: **Frontend** | Repo: `erp-marketing-frontend`

| ID | Task | Deliverable | Depends On | Est. (days) |
|----|------|-------------|------------|-------------|
| FE-M-01 | Initialize Marketing remote app | `frontends/erp-marketing/` | FE-SH-01 | 1 |
| FE-M-02 | Implement Campaigns list + detail + creation UI | `src/pages/Campaigns/` | FE-M-01, BE-M-02 | 4 |
| FE-M-03 | Implement Lead management UI + lead-to-client conversion workflow | `src/pages/Leads/`; `src/components/LeadConversion.tsx` | FE-M-01, BE-M-03 | 4 |
| FE-M-04 | Implement Ticket / Support queue UI | `src/pages/Tickets/` | FE-M-01, BE-M-04 | 3 |
| FE-M-05 | Implement ETL Wizard (step-by-step bulk import UI) | `src/pages/ETLWizard/` | FE-M-01, BE-M-07 | 4 |
| FE-M-06 | Implement campaign analytics dashboard (charts via Recharts / Nivo) | `src/pages/Analytics/` | FE-M-01, BE-M-06 | 4 |
| FE-M-07 | Write Nginx config + Dockerfile + GitHub Actions CI/CD | `nginx.conf`, `Dockerfile`, `.github/workflows/ci-cd.yml` | FE-M-01, INFRA-14 | 2 |
| **Group total** | | | | **22** |

### Accounting Frontend (Micro-Frontend Remote)
> Area: **Frontend** | Repo: `erp-accounting-frontend`

| ID | Task | Deliverable | Depends On | Est. (days) |
|----|------|-------------|------------|-------------|
| FE-AC-01 | Initialize Accounting remote app | `frontends/erp-accounting/` | FE-SH-01 | 1 |
| FE-AC-02 | Implement Journal Entry list + creation UI (double-entry form) | `src/pages/JournalEntries/` | FE-AC-01, BE-A-03 | 5 |
| FE-AC-03 | Implement Chart of Accounts management UI | `src/pages/Accounts/` | FE-AC-01, BE-A-02 | 3 |
| FE-AC-04 | Implement Accounting Periods UI + period closure workflow | `src/pages/Periods/` | FE-AC-01, BE-A-05 | 3 |
| FE-AC-05 | Implement financial dashboard (Balance Sheet, P&L charts) | `src/components/FinancialDashboard.tsx` | FE-AC-01, BE-A-06 | 4 |
| FE-AC-06 | Implement VAT / Tax declaration download UI | `src/pages/TaxDeclarations/` | FE-AC-01, BE-A-07 | 3 |
| FE-AC-07 | Implement Audit plan + findings UI | `src/pages/Audit/` | FE-AC-01, BE-A-08 | 4 |
| FE-AC-08 | Write Nginx config + Dockerfile + GitHub Actions CI/CD | `nginx.conf`, `Dockerfile`, `.github/workflows/ci-cd.yml` | FE-AC-01, INFRA-14 | 2 |
| **Group total** | | | | **25** |

---

## Phase 5 — Integration, QA & Testing
> Area: **QA** with cross-team support

| ID | Task | Deliverable | Depends On | Est. (days) |
|----|------|-------------|------------|-------------|
| QA-01 | Write Playwright E2E test suite — login flow (Entra ID mock) | `e2e/auth.spec.ts` | FE-SH-03 | 3 |
| QA-02 | Write Playwright E2E tests — Legal module full flow | `e2e/legal.spec.ts` | FE-L-01–FE-L-06 | 5 |
| QA-03 | Write Playwright E2E tests — Marketing module full flow | `e2e/marketing.spec.ts` | FE-M-01–FE-M-06 | 5 |
| QA-04 | Write Playwright E2E tests — Accounting module full flow | `e2e/accounting.spec.ts` | FE-AC-01–FE-AC-07 | 5 |
| QA-05 | Validate Pact consumer-driven contract tests pass for all service pairs | Pact verification reports | CONT-07 | 2 |
| QA-06 | Validate APIM JWT rejection of invalid/expired tokens | Postman/pytest test collection against APIM | INFRA-09, BE-L-01 | 2 |
| QA-07 | Validate RBAC: roles block correct endpoints across all modules | Role-based test matrix | INFRA-11, QA-06 | 3 |
| QA-08 | Load test FastAPI services (10+ concurrent requests, autoscaling validation) | k6 / Locust load test report | All BE services | 4 |
| QA-09 | AI agent integration test — full ingestion → analysis → generation pipeline | `tests/ai_pipeline_integration.py` | AI-ING-01–AI-GEN-09 | 5 |
| QA-10 | LangSmith evaluation — AI output quality for all agent types | LangSmith evaluation dataset + scores | AI-01–AI-SRC-07 | 3 |
| QA-11 | Security scan: OWASP ZAP against all public endpoints | ZAP scan report; fix identified issues | All BE + FE | 3 |
| QA-12 | Accessibility audit (WCAG 2.1 AA) on all frontend modules | Lighthouse / axe-core report; fix critical issues | All FE | 3 |
| QA-13 | Responsive design testing (desktop, tablet, mobile) | Cross-device test report | All FE | 2 |
| **Group total** | | | | **45** |

---

## Phase 6 — Production Readiness
> Area: **DevOps** | Cross-team

| ID | Task | Deliverable | Depends On | Est. (days) |
|----|------|-------------|------------|-------------|
| PROD-01 | Configure ACA autoscaling rules per service (HTTP + Service Bus queue triggers) | ACA scale rules as per architecture spec table | All services deployed to ACA | 2 |
| PROD-02 | Configure overnight scale-to-zero schedules for dev + staging environments | ACA replica schedules | INFRA-03, INFRA-04 | 1 |
| PROD-03 | Configure Azure Monitor Alerts (replica count, HTTP error rate, latency thresholds) | Alert rules + action groups (email/Teams) | INFRA-16 | 2 |
| PROD-04 | Store all environment image SHA tags in Git (enable full environment rebuild) | `infra/image-versions/` per environment | All CI/CD pipelines | 1 |
| PROD-05 | Run full staging smoke test + sign-off | Staging sign-off checklist | All phases complete | 2 |
| PROD-06 | Production go-live deployment (manual approval gate in GitHub Environments) | All services live in prod ACA environment | PROD-05 | 1 |
| PROD-07 | Write runbooks: restore from backup, service rollback, incident response | `docs/runbooks/` | PROD-06 | 3 |
| PROD-08 | Write per-service README and API docs (FastAPI OpenAPI auto-generated) | `README.md` per repo; Swagger UI accessible at `/docs` | All services | 3 |
| **Group total** | | | | **15** |

---

## Effort Summary & Headcount Calculation

| Phase / Group | Est. (person-days) |
|---|---|
| Phase 0 — Infrastructure & DevOps | 47 |
| Phase 1a — `erp-core` package | 33 |
| Phase 1a — `erp-contracts` package | 20 |
| Phase 1b — Data Layer (Django) | 62 |
| Phase 2 — Legal FastAPI | 26 |
| Phase 2 — Marketing FastAPI | 27 |
| Phase 2 — Accounting FastAPI | 41 |
| Phase 2 — Celery Worker | 7 |
| Phase 3 — AI Orchestrator | 12 |
| Phase 3 — AI Ingestion Agent | 18 |
| Phase 3 — AI Analysis Agent | 15 |
| Phase 3 — AI Generation Agent | 23 |
| Phase 3 — AI Classification Agent | 10 |
| Phase 3 — AI Search Agent | 16 |
| Phase 4 — Shell Frontend | 35 |
| Phase 4 — Legal Frontend | 22 |
| Phase 4 — Marketing Frontend | 22 |
| Phase 4 — Accounting Frontend | 25 |
| Phase 5 — Integration, QA & Testing | 45 |
| Phase 6 — Production Readiness | 15 |
| **Grand Total** | **521 person-days** |

### Effort by Discipline

| Discipline | Est. (person-days) | Groups |
|---|---|---|
| **DevOps** | 47 + 15 = **62** | Phase 0, Phase 6 |
| **Backend (shared + services + AI)** | 33 + 20 + 26 + 27 + 41 + 7 + 12 + 18 + 15 + 23 + 10 + 16 = **248** | Phase 1a, Phase 2, Phase 3 |
| **Database (Django data layer)** | **62** | Phase 1b |
| **Frontend** | 35 + 22 + 22 + 25 = **104** | Phase 4 |
| **QA** | **45** | Phase 5 |
| **Total** | **521** | |

### Headcount Formula

```
Required FTEs (per discipline) = Discipline person-days / (Project calendar days × Focus factor)
```

- **Project calendar days** = working days in the planned schedule (e.g. 6 months ≈ 120 working days).
- **Focus factor** = 0.7 (accounts for meetings, code review, ramp-up, blockers).

**Worked example — 6-month delivery (≈ 120 working days × 0.7 focus → 84 productive days per FTE):**

| Discipline | Days | FTEs needed | Round up |
|---|---|---|---|
| DevOps | 62 | 62 / 84 = 0.74 | **1** |
| Backend | 248 | 248 / 84 = 2.95 | **3** |
| Database | 62 | 62 / 84 = 0.74 | **1** (can overlap with Backend) |
| Frontend | 104 | 104 / 84 = 1.24 | **2** |
| QA | 45 | 45 / 84 = 0.54 | **1** |
| **Team total** | **521** | **6.21** | **~7 people** for a 6-month MVP |

> Adjust the calendar days and focus factor to your actual schedule to recompute the FTE requirement.

