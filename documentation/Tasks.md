# ERP Platform — Task List
**Last Updated:** April 27, 2026  
**Architecture Reference:** `architecture_specifics.md`

---

## Team
| Area | Members |
|------|---------|
| Frontend | Veronika, Yoana |
| Database | Ivan, Dimo |
| Backend | Veronika, Ivan, Martin, Evsin |
| QA | Gosho |
| DevOps | Dimo, Martin |

---

## How to Read This Document
- Tasks are grouped by **service/area**, ordered by **dependency**.
- Each task maps to one deployable unit from the Container App Map.
- `Depends On` references task IDs within this file.
- **Assignee** reflects team ownership per `architecture_specifics.md`.

---

## Phase 0 — Infrastructure & DevOps Setup
> Owner: **Dimo, Martin** | Repo: `erp-infrastructure`

| ID | Task | Deliverable | Depends On | Assignee |
|----|------|-------------|------------|----------|
| INFRA-01 | Create Azure Resource Groups (dev / staging / prod) | Azure RG per environment | — | Dimo |
| INFRA-02 | Provision Azure Container Registry (ACR) | ACR instance with repositories | INFRA-01 | Dimo |
| INFRA-03 | Provision Azure Container Apps Environment (dev) | ACA Environment — dev | INFRA-02 | Martin |
| INFRA-04 | Provision Azure Container Apps Environment (staging / prod) | ACA Environment — staging, prod | INFRA-03 | Martin |
| INFRA-05 | Provision Azure Database for PostgreSQL (Flexible Server) | PostgreSQL instance, Multi-AZ (prod) | INFRA-01 | Ivan |
| INFRA-06 | Provision Azure Cache for Redis (clustered) | Redis instance | INFRA-01 | Dimo |
| INFRA-07 | Provision Azure Service Bus (queues + topics) | Service Bus namespace with all queues from architecture spec | INFRA-01 | Martin |
| INFRA-08 | Provision Azure Key Vault (one per environment) | Key Vault instances | INFRA-01 | Dimo |
| INFRA-09 | Provision Azure API Management (APIM) | APIM instance, JWT validation policy, routing rules to FastAPI services | INFRA-01 | Martin |
| INFRA-10 | Configure Azure Front Door (CDN + WAF + TLS) | Front Door profile routing `/api/*` to APIM and static assets to frontend apps | INFRA-09 | Martin |
| INFRA-11 | Configure Azure Entra ID (app registration, scopes, roles) | App registration; roles: `legal.reader`, `legal.editor`, `marketing.admin`, `accounting.editor`, etc. | INFRA-01 | Dimo |
| INFRA-12 | Provision Azure Blob Storage (file uploads) | Storage account + containers per module | INFRA-01 | Dimo |
| INFRA-13 | Provision Azure OpenAI Service (GPT-4o, GPT-4o-mini, text-embedding-3-large) | Azure OpenAI endpoints via private endpoint in VNet | INFRA-01 | Martin |
| INFRA-14 | Setup GitHub Actions: base reusable workflow (build → push ACR → deploy ACA) | `.github/workflows/deploy-template.yml` in each repo | INFRA-02, INFRA-03 | Martin |
| INFRA-15 | Configure GitHub Environments (dev / staging / prod) with protection rules | GitHub Environments with reviewer sign-off for prod | INFRA-14 | Dimo |
| INFRA-16 | Setup Azure Monitor + Log Analytics workspace | Log Analytics workspace, container log routing | INFRA-03 | Martin |
| INFRA-17 | Setup Grafana dashboards (CPU, memory, replica count, latency) | Grafana dashboards via Azure Monitor data source | INFRA-16 | Dimo |
| INFRA-18 | Configure OpenTelemetry → Azure Application Insights (all FastAPI services) | Application Insights resource; OTEL exporter config in `erp-core` | INFRA-16 | Martin |
| INFRA-19 | Setup Azure Budget Alerts (80% and 100% spend) | Budget alert rules | INFRA-01 | Dimo |
| INFRA-20 | Setup Docker Compose for local development (PostgreSQL, RabbitMQ, Redis) | `docker-compose.dev.yml` shared across repos | — | Martin |
| INFRA-21 | Write Bicep / Terraform for all Azure resources (IaC) | `erp-infrastructure` repo with complete IaC | INFRA-01–INFRA-13 | Dimo, Martin |

---

## Phase 1a — Shared Packages

### `erp-core` Python Package
> Owner: **Backend (Veronika, Ivan, Martin, Evsin)** | Repo: `erp-core`

| ID | Task | Deliverable | Depends On | Assignee |
|----|------|-------------|------------|----------|
| CORE-01 | Initialize `erp-core` Python package (pyproject.toml, src layout) | `erp_core/` package skeleton, GitHub Packages publish pipeline | — | Martin |
| CORE-02 | Implement `erp_core/config.py` — Pydantic `BaseSettings` loading from Azure Key Vault + env vars | `erp_core/config.py` | CORE-01 | Ivan |
| CORE-03 | Implement `erp_core/logging.py` — structured JSON logging with `correlation_id`, `service_name`, `user_id`; OpenTelemetry trace injection | `erp_core/logging.py` | CORE-01 | Evsin |
| CORE-04 | Implement `erp_core/exceptions.py` — shared exception hierarchy (`AppError`, `NotFoundError`, `ConflictError`, `ValidationError`) mapped to HTTP status codes | `erp_core/exceptions.py` | CORE-01 | Martin |
| CORE-05 | Implement `erp_core/middleware/` — request ID injection, structured access logging, error response formatting | `erp_core/middleware/` | CORE-03, CORE-04 | Evsin |
| CORE-06 | Implement `erp_core/interfaces/` — ABCs: `BaseService`, `BaseRepository`, `BaseEventPublisher`, `BaseAIAgent` | `erp_core/interfaces/` | CORE-01 | Ivan |
| CORE-07 | Implement `erp_core/messaging/` — Azure Service Bus publisher/subscriber base classes; RabbitMQ adapter for local dev; switchable via `MESSAGING_BACKEND` env var | `erp_core/messaging/` | CORE-06 | Martin |
| CORE-08 | Implement `erp_core/http_client.py` — shared `httpx` async client with retry policy, timeout config, `X-Correlation-ID` header propagation | `erp_core/http_client.py` | CORE-03 | Evsin |
| CORE-09 | Implement `erp_core/health.py` — standard `/health` and `/ready` endpoint factory | `erp_core/health.py` | CORE-01 | Martin |
| CORE-10 | Implement `erp_core/security.py` — JWT claim extraction helpers (user ID, roles, module access) from APIM-forwarded headers | `erp_core/security.py` | CORE-01 | Ivan |
| CORE-11 | Implement `erp_core/constants.py` — shared enums: `ModuleName`, `AuditAction`, `EventType` | `erp_core/constants.py` | CORE-01 | Martin |
| CORE-12 | Write unit tests for all `erp-core` modules (≥90% coverage) | `tests/` in `erp-core` repo | CORE-02–CORE-11 | Evsin |
| CORE-13 | Publish `erp-core` v1.0.0 to GitHub Packages | GitHub Packages private PyPI entry | CORE-12 | Martin |

### `erp-contracts` Pydantic Package
> Owner: **Backend** | Repo: `erp-contracts`

| ID | Task | Deliverable | Depends On | Assignee |
|----|------|-------------|------------|----------|
| CONT-01 | Initialize `erp-contracts` Python package structure | `erp_contracts/` package skeleton | CORE-01 | Ivan |
| CONT-02 | Define REST contracts — Legal module DTOs (request/response Pydantic v2 models) | `erp_contracts/legal/` | CONT-01 | Ivan |
| CONT-03 | Define REST contracts — Marketing module DTOs | `erp_contracts/marketing/` | CONT-01 | Veronika |
| CONT-04 | Define REST contracts — Accounting module DTOs | `erp_contracts/accounting/` | CONT-01 | Martin |
| CONT-05 | Define AI agent I/O schemas (`IngestionRequest`, `AnalysisResult`, `GenerationRequest`, `ClassificationRequest`, `SearchRequest`, `SearchResult`) | `erp_contracts/ai/` | CONT-01 | Martin |
| CONT-06 | Define async event schemas for Service Bus messages | `erp_contracts/events/` | CONT-01 | Evsin |
| CONT-07 | Setup Pact consumer-driven contract tests in CI | `.github/workflows/pact.yml`; Pact Broker config | CONT-02–CONT-06 | Evsin |
| CONT-08 | Publish `erp-contracts` v1.0.0 to GitHub Packages | GitHub Packages private PyPI entry | CONT-07 | Martin |

---

## Phase 1b — Data Layer (Django)
> Owner: **Ivan, Dimo** (DB schema) + **Backend** (Django service) | Repo: `erp-django-core`

| ID | Task | Deliverable | Depends On | Assignee |
|----|------|-------------|------------|----------|
| DB-01 | Setup PostgreSQL 16 + pgvector extension (Docker Compose local dev) | `docker-compose.dev.yml` DB service; `CREATE EXTENSION vector` in init script | INFRA-20 | Ivan |
| DB-02 | Design and create `core` schema (Organization, Role, User, Permission tables) | Django models + migration: `core_organization`, `core_role`, `core_user_role`, `core_permission` | DB-01 | Ivan |
| DB-03 | Design and create `legal` schema (LegalDossier, LegalCase, LegalDeadline, LawAct, Attachment, DocumentVersion) | Django models + migration for `legal` schema | DB-02 | Ivan |
| DB-04 | Design and create `marketing` schema (Campaign, Lead, Ticket, Operator, ContactPerson) | Django models + migration for `marketing` schema | DB-02 | Dimo |
| DB-05 | Design and create `accounting` schema (Account, JournalEntry, JournalEntryLine, Period, TaxDeclaration) | Django models + migration for `accounting` schema | DB-02 | Ivan |
| DB-06 | Add `AuditLog` table with JSON diff (shared, under `core` schema) | `core_audit_log` model + migration | DB-02 | Ivan |
| DB-07 | Add `Attachment` + `DocumentVersion` tables (shared, under `core` schema) | `core_attachment`, `core_document_version` models + migration | DB-02 | Dimo |
| DB-08 | Enable pgvector on PostgreSQL; add embedding columns and vector indexes to relevant tables | `CREATE INDEX ivfflat` on embedding columns | DB-01 | Ivan |
| DB-09 | Create indexes for all foreign keys across all schemas | Indexes on all `*_id` columns | DB-02–DB-05 | Dimo |
| DB-10 | Initialize `erp-django-core` Django project | `erp-django-core/` project skeleton; Gunicorn config; `pyproject.toml` with `erp-core` dependency | CORE-13 | Ivan |
| DB-11 | Configure Django per-schema database routing (separate schema per module in single PostgreSQL instance) | `db_router.py`; Django settings for `legal`, `marketing`, `accounting`, `core` schemas | DB-10, DB-02 | Ivan |
| DB-12 | Implement `Organization`, `Department`, `Team` models in `core` app | `core/models.py` | DB-10 | Ivan |
| DB-13 | Implement Django admin panel (internal management UI) | `admin.py` registrations for all models | DB-10 | Dimo |
| DB-14 | Implement RBAC models aligned with Azure Entra ID roles (read roles from JWT claims, no local password management) | `rbac/models.py`, `rbac/service.py` — role-to-permission mapping only | DB-10, INFRA-11 | Ivan |
| DB-15 | Implement audit log middleware for Django admin actions | `middleware/audit.py` | DB-06 | Dimo |
| DB-16 | Implement internal service layer (Django → accessible by FastAPI via `erp-core` shared interfaces, not via HTTP) | `services/` in each Django app; used via `erp-core` `BaseRepository` | DB-10, CORE-06 | Ivan |
| DB-17 | Setup Django REST Framework + OpenAPI (internal API for Django admin) | DRF config, Swagger/OpenAPI schema | DB-10 | Dimo |
| DB-18 | Seed permissions command (all modules) | `management/commands/seed_permissions.py` | DB-14 | Dimo |
| DB-19 | Configure Azure Database for PostgreSQL connection (SSL, connection pooling via pgBouncer) | Django `DATABASES` settings with Azure credentials from Key Vault | DB-10, INFRA-05, INFRA-08 | Ivan |
| DB-20 | Configure automated backup validation + point-in-time restore test | Restore test runbook; `docs/runbooks/restore.md` | DB-19 | Dimo |
| DB-21 | Write Dockerfile for `erp-django-core` (internal only, Gunicorn) | `Dockerfile` | DB-10 | Martin |
| DB-22 | Setup GitHub Actions CI/CD pipeline for `erp-django-core` | `.github/workflows/ci-cd.yml` (lint → test → build → push ACR → deploy ACA internal) | DB-21, INFRA-14 | Martin |

---

## Phase 2 — Module Backend Services (FastAPI)

### Legal FastAPI Service
> Owner: **Backend (Veronika, Ivan)** | Repo: `erp-legal-api`

| ID | Task | Deliverable | Depends On | Assignee |
|----|------|-------------|------------|----------|
| BE-L-01 | Initialize `erp-legal-api` FastAPI project | `services/legal-api/` skeleton; `pyproject.toml` with `erp-core` + `erp-contracts` | CORE-13, CONT-08 | Veronika |
| BE-L-02 | Implement Legal Dossier CRUD endpoints | `routes/dossiers.py` | BE-L-01, DB-03 | Ivan |
| BE-L-03 | Implement Legal Case management endpoints | `routes/cases.py` | BE-L-01, DB-03 | Ivan |
| BE-L-04 | Implement Legal Deadline tracking endpoints | `routes/deadlines.py` | BE-L-01, DB-03 | Veronika |
| BE-L-05 | Implement Law Acts / Knowledge Base endpoints | `routes/law_acts.py` | BE-L-01, DB-03 | Veronika |
| BE-L-06 | Implement document upload → trigger `ai.ingestion.requests` Service Bus message | `routes/documents.py`; Service Bus publisher | BE-L-01, CORE-07, INFRA-07 | Ivan |
| BE-L-07 | Implement AI Search integration (publish to `ai.search.requests`, consume results) | `routes/search.py` | BE-L-01, CORE-07 | Veronika |
| BE-L-08 | Implement audit logging for all write operations (via `erp-core` middleware) | Audit events in `core_audit_log` | BE-L-01, DB-06 | Ivan |
| BE-L-09 | Write unit + integration tests (pytest + httpx) | `tests/` (≥80% coverage) | BE-L-01–BE-L-08 | Evsin |
| BE-L-10 | Write Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | BE-L-01, INFRA-14 | Martin |

### Marketing FastAPI Service
> Owner: **Backend (Veronika, Evsin)** | Repo: `erp-marketing-api`

| ID | Task | Deliverable | Depends On | Assignee |
|----|------|-------------|------------|----------|
| BE-M-01 | Initialize `erp-marketing-api` FastAPI project | `services/marketing-api/` skeleton | CORE-13, CONT-08 | Evsin |
| BE-M-02 | Implement Campaign management endpoints (CRUD) | `routes/campaigns.py` | BE-M-01, DB-04 | Evsin |
| BE-M-03 | Implement Lead management + lead-to-client conversion endpoints | `routes/leads.py` | BE-M-01, DB-04 | Veronika |
| BE-M-04 | Implement Ticket / Support request endpoints | `routes/tickets.py` | BE-M-01, DB-04 | Evsin |
| BE-M-05 | Implement Operator assignment + routing logic | `services/operator_service.py` | BE-M-01, DB-04 | Veronika |
| BE-M-06 | Implement campaign performance analysis → trigger `ai.analysis.requests` | `routes/analytics.py`; Service Bus publisher | BE-M-01, CORE-07 | Evsin |
| BE-M-07 | Implement ETL data import (bulk lead/campaign import) | `services/etl_service.py` | BE-M-01 | Veronika |
| BE-M-08 | Write unit + integration tests | `tests/` (≥80% coverage) | BE-M-01–BE-M-07 | Evsin |
| BE-M-09 | Write Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | BE-M-01, INFRA-14 | Martin |

### Accounting FastAPI Service
> Owner: **Backend (Martin, Ivan)** | Repo: `erp-accounting-api`

| ID | Task | Deliverable | Depends On | Assignee |
|----|------|-------------|------------|----------|
| BE-A-01 | Initialize `erp-accounting-api` FastAPI project | `services/accounting-api/` skeleton | CORE-13, CONT-08 | Martin |
| BE-A-02 | Create Chart of Accounts (NSS standard) | `routes/accounts.py`; `data/chart_of_accounts.json`; seed command | BE-A-01, DB-05 | Martin |
| BE-A-03 | Implement Journal Entry + Journal Entry Lines endpoints | `routes/journal_entries.py`; `models/journal_entry.py` | BE-A-01, DB-05 | Ivan |
| BE-A-04 | Implement double-entry validation service | `services/validation_service.py` (debit == credit check) | BE-A-03 | Martin |
| BE-A-05 | Implement Accounting Period + Period Closure | `routes/periods.py`; `models/period.py` | BE-A-01, DB-05 | Ivan |
| BE-A-06 | Implement financial statement generation (Balance Sheet, P&L, Cash Flow) | `services/financial_statement_service.py` | BE-A-03, BE-A-05 | Martin |
| BE-A-07 | Implement VAT + corporate tax declaration generation | `services/tax_service.py` | BE-A-03 | Martin |
| BE-A-08 | Implement audit plan + findings endpoints | `routes/audit.py` | BE-A-01, DB-05 | Ivan |
| BE-A-09 | Implement NAP inspection control dossier | `services/control_dossier_service.py` | BE-A-08 | Martin |
| BE-A-10 | Implement risk alert engine | `services/risk_alert_service.py` | BE-A-06 | Ivan |
| BE-A-11 | Trigger AI generation for financial narratives → `ai.generation.requests` | Service Bus publisher in `services/narrative_service.py` | BE-A-01, CORE-07 | Martin |
| BE-A-12 | Write unit + integration tests | `tests/` (≥80% coverage) | BE-A-01–BE-A-11 | Evsin |
| BE-A-13 | Write Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | BE-A-01, INFRA-14 | Martin |

### Celery Background Worker
> Owner: **Backend (Martin)** | Repo: `erp-celery-worker`

| ID | Task | Deliverable | Depends On | Assignee |
|----|------|-------------|------------|----------|
| CEL-01 | Initialize Celery worker service | `services/celery-worker/` with `pyproject.toml` | CORE-13 | Martin |
| CEL-02 | Implement scheduled background tasks (report generation, data cleanup, notifications) | `tasks/` directory | CEL-01 | Martin |
| CEL-03 | Configure Azure Service Bus as Celery broker (prod) / RabbitMQ (local dev) | `celeryconfig.py` with `MESSAGING_BACKEND` switch | CEL-01, INFRA-07 | Martin |
| CEL-04 | Write Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | CEL-01, INFRA-14 | Martin |

---

## Phase 3 — AI Services

### AI Orchestrator (LangGraph Supervisor)
> Owner: **Backend (Martin, Evsin)** | Repo: `erp-ai-orchestrator`

| ID | Task | Deliverable | Depends On | Assignee |
|----|------|-------------|------------|----------|
| AI-01 | Initialize AI Orchestrator service (LangGraph Supervisor) | `services/ai-orchestrator/` skeleton; LangGraph dependency | CORE-13, CONT-08 | Martin |
| AI-02 | Implement task routing logic — routes incoming `ai.*` requests to correct agent via Service Bus | `orchestrator/supervisor.py`; routing rules per `EventType` | AI-01, INFRA-07 | Martin |
| AI-03 | Implement result aggregation — consumes `ai.results` topic, returns to requesting FastAPI service | `orchestrator/aggregator.py` | AI-02 | Evsin |
| AI-04 | Setup LangSmith tracing integration for orchestrator | LangSmith config in `erp_core/logging.py` (AI trace extensions) | AI-01 | Martin |
| AI-05 | Write Dockerfile + GitHub Actions CI/CD pipeline (scale-to-zero in ACA) | `Dockerfile`, `.github/workflows/ci-cd.yml` | AI-01, INFRA-14 | Martin |

### AI Ingestion Agent
> Owner: **Backend (Evsin)** | Repo: `erp-ai-ingestion`

| ID | Task | Deliverable | Depends On | Assignee |
|----|------|-------------|------------|----------|
| AI-ING-01 | Initialize Ingestion Agent service | `services/ai-ingestion/` skeleton | CORE-13, CONT-08 | Evsin |
| AI-ING-02 | Implement file parsing: PDF, DOCX, XLSX text extraction | `parsers/` (PyMuPDF / python-docx / openpyxl) | AI-ING-01 | Evsin |
| AI-ING-03 | Implement OCR for images using GPT-4o Vision | `ocr/gpt4o_vision.py` | AI-ING-01, INFRA-13 | Evsin |
| AI-ING-04 | Implement text chunking + embedding generation (`text-embedding-3-large` via Azure OpenAI) | `embeddings/chunker.py`, `embeddings/embedder.py` | AI-ING-01, INFRA-13 | Evsin |
| AI-ING-05 | Store embeddings in pgvector (MVP) | `storage/pgvector_store.py` | AI-ING-04, DB-08 | Evsin |
| AI-ING-06 | Subscribe to `ai.ingestion.requests` Service Bus queue | `consumers/ingestion_consumer.py` | AI-ING-01, CORE-07 | Evsin |
| AI-ING-07 | Write Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | AI-ING-01, INFRA-14 | Martin |

### AI Analysis Agent
> Owner: **Backend (Ivan)** | Repo: `erp-ai-analysis`

| ID | Task | Deliverable | Depends On | Assignee |
|----|------|-------------|------------|----------|
| AI-ANA-01 | Initialize Analysis Agent service | `services/ai-analysis/` skeleton | CORE-13, CONT-08 | Ivan |
| AI-ANA-02 | Implement clause risk scoring (legal documents) using GPT-4o | `analyzers/clause_risk.py` | AI-ANA-01, INFRA-13 | Ivan |
| AI-ANA-03 | Implement financial anomaly detection using GPT-4o | `analyzers/financial_anomaly.py` | AI-ANA-01, INFRA-13 | Ivan |
| AI-ANA-04 | Implement campaign performance insights using GPT-4o | `analyzers/campaign_insights.py` | AI-ANA-01, INFRA-13 | Ivan |
| AI-ANA-05 | Subscribe to `ai.analysis.requests` Service Bus queue; publish to `ai.results` | `consumers/analysis_consumer.py` | AI-ANA-01, CORE-07 | Ivan |
| AI-ANA-06 | Write Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | AI-ANA-01, INFRA-14 | Martin |

### AI Generation Agent
> Owner: **Backend (Veronika)** | Repo: `erp-ai-generation`

| ID | Task | Deliverable | Depends On | Assignee |
|----|------|-------------|------------|----------|
| AI-GEN-01 | Initialize Generation Agent service | `services/ai-generation/` skeleton | CORE-13, CONT-08 | Veronika |
| AI-GEN-02 | Implement legal document summary generation (GPT-4o) | `generators/legal_summary.py` | AI-GEN-01, INFRA-13 | Veronika |
| AI-GEN-03 | Implement marketing copy generation (GPT-4o) | `generators/marketing_copy.py` | AI-GEN-01, INFRA-13 | Veronika |
| AI-GEN-04 | Implement financial narrative generation (GPT-4o) | `generators/financial_narrative.py` | AI-GEN-01, INFRA-13 | Veronika |
| AI-GEN-05 | Implement email draft generation (GPT-4o) | `generators/email_draft.py` | AI-GEN-01, INFRA-13 | Veronika |
| AI-GEN-06 | Create and manage prompt templates (per persona / use case) | `data/prompts/` (≥10 prompt templates covering all modules) | AI-GEN-01 | Veronika |
| AI-GEN-07 | Subscribe to `ai.generation.requests`; publish to `ai.results` | `consumers/generation_consumer.py` | AI-GEN-01, CORE-07 | Veronika |
| AI-GEN-08 | Implement AG-UI (SSE streaming for real-time agent output in UI) | `agui/emitter.py`; SSE endpoint | AI-GEN-01 | Evsin |
| AI-GEN-09 | Write Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | AI-GEN-01, INFRA-14 | Martin |

### AI Classification Agent
> Owner: **Backend (Evsin)** | Repo: `erp-ai-classification`

| ID | Task | Deliverable | Depends On | Assignee |
|----|------|-------------|------------|----------|
| AI-CLS-01 | Initialize Classification Agent service | `services/ai-classification/` skeleton | CORE-13, CONT-08 | Evsin |
| AI-CLS-02 | Implement document categorization using GPT-4o-mini | `classifiers/document_classifier.py` | AI-CLS-01, INFRA-13 | Evsin |
| AI-CLS-03 | Implement task routing / intent detection (GPT-4o-mini) | `classifiers/intent_detector.py` | AI-CLS-01, INFRA-13 | Evsin |
| AI-CLS-04 | Subscribe to `ai.classification.requests`; publish to `ai.results` | `consumers/classification_consumer.py` | AI-CLS-01, CORE-07 | Evsin |
| AI-CLS-05 | Write Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | AI-CLS-01, INFRA-14 | Martin |

### AI Search Agent
> Owner: **Backend (Martin)** | Repo: `erp-ai-search`

| ID | Task | Deliverable | Depends On | Assignee |
|----|------|-------------|------------|----------|
| AI-SRC-01 | Initialize Search Agent service | `services/ai-search/` skeleton | CORE-13, CONT-08 | Martin |
| AI-SRC-02 | Implement vector similarity search (pgvector MVP) | `search/pgvector_search.py` | AI-SRC-01, DB-08 | Martin |
| AI-SRC-03 | Implement hybrid search (vector + keyword) | `search/hybrid_search.py` | AI-SRC-02 | Martin |
| AI-SRC-04 | Implement ranked, cited result response formatting | `search/result_formatter.py` | AI-SRC-02 | Martin |
| AI-SRC-05 | Subscribe to `ai.search.requests`; publish to `ai.results` | `consumers/search_consumer.py` | AI-SRC-01, CORE-07 | Martin |
| AI-SRC-06 | Implement token usage tracking + budget enforcement (Redis-backed) | `services/token_service.py` | AI-SRC-01, INFRA-06 | Evsin |
| AI-SRC-07 | Write Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | AI-SRC-01, INFRA-14 | Martin |

---

## Phase 4 — Frontend

### Shell Frontend (Micro-Frontend Host)
> Owner: **Frontend (Veronika, Yoana)** | Repo: `erp-shell-frontend`

| ID | Task | Deliverable | Depends On | Assignee |
|----|------|-------------|------------|----------|
| FE-SH-01 | Initialize Shell App (React.js + TypeScript + Webpack Module Federation) | `frontends/erp-shell/` with Module Federation host config (`singleton: true`) | — | Yoana |
| FE-SH-02 | Setup shared design system + component library (shared via Module Federation) | `src/design-system/` (buttons, forms, typography, colors) | FE-SH-01 | Yoana |
| FE-SH-03 | Implement Azure Entra ID login flow (redirect to Entra ID; receive JWT; store in memory — not localStorage) | `src/auth/authService.ts`; MSAL.js integration | FE-SH-01, INFRA-11 | Veronika |
| FE-SH-04 | Implement JWT token sharing to module remotes via Module Federation shared state | `src/auth/store.ts`; shared auth context | FE-SH-03 | Veronika |
| FE-SH-05 | Implement global navigation + sidebar (module-aware, role-based visibility) | `src/components/Navigation.tsx` | FE-SH-02 | Yoana |
| FE-SH-06 | Setup routing for remote module lazy loading + module discovery | `src/router/index.tsx`; `src/modules/registry.ts` | FE-SH-01 | Yoana |
| FE-SH-07 | Create shared DataTable component (TanStack Table) | `src/components/DataTable.tsx` | FE-SH-02 | Yoana |
| FE-SH-08 | Create shared Form builder (React Hook Form + Zod) | `src/components/FormBuilder.tsx` | FE-SH-02 | Yoana |
| FE-SH-09 | Create global Notification Center | `src/components/NotificationCenter.tsx` | FE-SH-02 | Veronika |
| FE-SH-10 | Implement logout + session expiry handling | `src/auth/authService.ts` (logout, token refresh via Entra ID) | FE-SH-03 | Veronika |
| FE-SH-11 | Write Jest + React Testing Library unit tests | `src/__tests__/` | FE-SH-01–FE-SH-10 | Yoana |
| FE-SH-12 | Write Nginx config + Dockerfile for static React build | `nginx.conf`, `Dockerfile` | FE-SH-01 | Martin |
| FE-SH-13 | Setup GitHub Actions CI/CD pipeline | `.github/workflows/ci-cd.yml` | FE-SH-01, INFRA-14 | Martin |

### Legal Frontend (Micro-Frontend Remote)
> Owner: **Frontend (Veronika, Yoana)** | Repo: `erp-legal-frontend`

| ID | Task | Deliverable | Depends On | Assignee |
|----|------|-------------|------------|----------|
| FE-L-01 | Initialize Legal remote app (React.js + TypeScript + Webpack Module Federation remote config) | `frontends/erp-legal/` | FE-SH-01 | Yoana |
| FE-L-02 | Implement Legal Dossier list + detail pages | `src/pages/Dossiers/` | FE-L-01, BE-L-02 | Yoana |
| FE-L-03 | Implement Legal Case management UI | `src/pages/Cases/` | FE-L-01, BE-L-03 | Veronika |
| FE-L-04 | Implement Legal Deadline calendar / tracker UI | `src/pages/Deadlines/` | FE-L-01, BE-L-04 | Yoana |
| FE-L-05 | Implement Law Acts / Knowledge Base search UI with AI chat (SSE streaming) | `src/pages/Search/`; SSE integration with AI generation agent | FE-L-01, BE-L-07, AI-GEN-08 | Veronika |
| FE-L-06 | Implement document upload UI (triggers ingestion pipeline) | `src/components/DocumentUpload.tsx` | FE-L-01, BE-L-06 | Yoana |
| FE-L-07 | Write Nginx config + Dockerfile + GitHub Actions CI/CD | `nginx.conf`, `Dockerfile`, `.github/workflows/ci-cd.yml` | FE-L-01, INFRA-14 | Martin |

### Marketing Frontend (Micro-Frontend Remote)
> Owner: **Frontend (Veronika, Yoana)** | Repo: `erp-marketing-frontend`

| ID | Task | Deliverable | Depends On | Assignee |
|----|------|-------------|------------|----------|
| FE-M-01 | Initialize Marketing remote app | `frontends/erp-marketing/` | FE-SH-01 | Yoana |
| FE-M-02 | Implement Campaigns list + detail + creation UI | `src/pages/Campaigns/` | FE-M-01, BE-M-02 | Yoana |
| FE-M-03 | Implement Lead management UI + lead-to-client conversion workflow | `src/pages/Leads/`; `src/components/LeadConversion.tsx` | FE-M-01, BE-M-03 | Veronika |
| FE-M-04 | Implement Ticket / Support queue UI | `src/pages/Tickets/` | FE-M-01, BE-M-04 | Yoana |
| FE-M-05 | Implement ETL Wizard (step-by-step bulk import UI) | `src/pages/ETLWizard/` | FE-M-01, BE-M-07 | Veronika |
| FE-M-06 | Implement campaign analytics dashboard (charts via Recharts / Nivo) | `src/pages/Analytics/` | FE-M-01, BE-M-06 | Yoana |
| FE-M-07 | Write Nginx config + Dockerfile + GitHub Actions CI/CD | `nginx.conf`, `Dockerfile`, `.github/workflows/ci-cd.yml` | FE-M-01, INFRA-14 | Martin |

### Accounting Frontend (Micro-Frontend Remote)
> Owner: **Frontend (Veronika, Yoana)** | Repo: `erp-accounting-frontend`

| ID | Task | Deliverable | Depends On | Assignee |
|----|------|-------------|------------|----------|
| FE-AC-01 | Initialize Accounting remote app | `frontends/erp-accounting/` | FE-SH-01 | Yoana |
| FE-AC-02 | Implement Journal Entry list + creation UI (double-entry form) | `src/pages/JournalEntries/` | FE-AC-01, BE-A-03 | Yoana |
| FE-AC-03 | Implement Chart of Accounts management UI | `src/pages/Accounts/` | FE-AC-01, BE-A-02 | Veronika |
| FE-AC-04 | Implement Accounting Periods UI + period closure workflow | `src/pages/Periods/` | FE-AC-01, BE-A-05 | Yoana |
| FE-AC-05 | Implement financial dashboard (Balance Sheet, P&L charts) | `src/components/FinancialDashboard.tsx` | FE-AC-01, BE-A-06 | Veronika |
| FE-AC-06 | Implement VAT / Tax declaration download UI | `src/pages/TaxDeclarations/` | FE-AC-01, BE-A-07 | Yoana |
| FE-AC-07 | Implement Audit plan + findings UI | `src/pages/Audit/` | FE-AC-01, BE-A-08 | Veronika |
| FE-AC-08 | Write Nginx config + Dockerfile + GitHub Actions CI/CD | `nginx.conf`, `Dockerfile`, `.github/workflows/ci-cd.yml` | FE-AC-01, INFRA-14 | Martin |

---

## Phase 5 — Integration, QA & Testing
> Owner: **Gosho (QA)** with support from all teams

| ID | Task | Deliverable | Depends On | Assignee |
|----|------|-------------|------------|----------|
| QA-01 | Write Playwright E2E test suite — login flow (Entra ID mock) | `e2e/auth.spec.ts` | FE-SH-03 | Gosho |
| QA-02 | Write Playwright E2E tests — Legal module full flow | `e2e/legal.spec.ts` | FE-L-01–FE-L-06 | Gosho |
| QA-03 | Write Playwright E2E tests — Marketing module full flow | `e2e/marketing.spec.ts` | FE-M-01–FE-M-06 | Gosho |
| QA-04 | Write Playwright E2E tests — Accounting module full flow | `e2e/accounting.spec.ts` | FE-AC-01–FE-AC-07 | Gosho |
| QA-05 | Validate Pact consumer-driven contract tests pass for all service pairs | Pact verification reports | CONT-07 | Gosho |
| QA-06 | Validate APIM JWT rejection of invalid/expired tokens | Postman/pytest test collection against APIM | INFRA-09, BE-L-01 | Gosho |
| QA-07 | Validate RBAC: roles block correct endpoints across all modules | Role-based test matrix | INFRA-11, QA-06 | Gosho |
| QA-08 | Load test FastAPI services (10+ concurrent requests, autoscaling validation) | k6 / Locust load test report | All BE services | Gosho |
| QA-09 | AI agent integration test — full ingestion → analysis → generation pipeline | `tests/ai_pipeline_integration.py` | AI-ING-01–AI-GEN-09 | Gosho |
| QA-10 | LangSmith evaluation — AI output quality for all agent types | LangSmith evaluation dataset + scores | AI-01–AI-SRC-07 | Gosho, Martin |
| QA-11 | Security scan: OWASP ZAP against all public endpoints | ZAP scan report; fix identified issues | All BE + FE | Gosho |
| QA-12 | Accessibility audit (WCAG 2.1 AA) on all frontend modules | Lighthouse / axe-core report; fix critical issues | All FE | Gosho, Yoana |
| QA-13 | Responsive design testing (desktop, tablet, mobile) | Cross-device test report | All FE | Gosho, Yoana |

---

## Phase 6 — Production Readiness
> Owner: **Dimo, Martin** | Cross-team

| ID | Task | Deliverable | Depends On | Assignee |
|----|------|-------------|------------|----------|
| PROD-01 | Configure ACA autoscaling rules per service (HTTP + Service Bus queue triggers) | ACA scale rules as per architecture spec table | All services deployed to ACA | Dimo |
| PROD-02 | Configure overnight scale-to-zero schedules for dev + staging environments | ACA replica schedules | INFRA-03, INFRA-04 | Dimo |
| PROD-03 | Configure Azure Monitor Alerts (replica count, HTTP error rate, latency thresholds) | Alert rules + action groups (email/Teams) | INFRA-16 | Martin |
| PROD-04 | Store all environment image SHA tags in Git (enable full environment rebuild) | `infra/image-versions/` per environment | All CI/CD pipelines | Martin |
| PROD-05 | Run full staging smoke test + sign-off | Staging sign-off checklist | All phases complete | Gosho, Dimo |
| PROD-06 | Production go-live deployment (manual approval gate in GitHub Environments) | All services live in prod ACA environment | PROD-05 | Dimo, Martin |
| PROD-07 | Write runbooks: restore from backup, service rollback, incident response | `docs/runbooks/` | PROD-06 | Dimo |
| PROD-08 | Write per-service README and API docs (FastAPI OpenAPI auto-generated) | `README.md` per repo; Swagger UI accessible at `/docs` | All services | All leads |

---

## Removed / Not Applicable Tasks (from original xlsx)

The following tasks from the original Excel file were **removed** because they conflict with the architecture defined in `architecture_specifics.md`:

| Original ID | Reason Removed |
|-------------|----------------|
| DB-02 / DB-03 | Multi-tenant master/template schema — architecture uses single DB with per-module schemas |
| DB-04 | `create_tenant()` function — no multi-tenancy |
| DB-20 | `migrate_tenants.py` runner — no multi-tenancy |
| DJ-02 | Multi-DB routing per tenant — not applicable |
| DJ-03 | `TenantUser` model — no tenant concept |
| DJ-04 / DJ-09 | Local JWT provider + local refresh — replaced by Azure Entra ID (zero password/token code written by team) |
| DJ-08 / DJ-13 | Tenant provisioning API + tenant module activation — no multi-tenancy |
| DJ-15 | Master admin React dashboard — not in scope |
| BE-INT-02 | Nginx reverse proxy as API Gateway — replaced by Azure API Management |
| BE-INT-03 | JWT validation in FastAPI services — handled by APIM; FastAPI is auth-free |
| DB-11 | `consulting` schema — renamed to `legal` to match architecture modules |
| DB-12 / DB-13 | `contracts` / `projects` schemas under consulting — merged into `legal` module |
| FE-02 / FE-03 | Vite + monorepo — architecture is Polyrepo with Webpack Module Federation |
| FE-11 | `consulting-ui` remote — modules are Legal, Marketing, Accounting only |
| FE-12 | `law-ui` — merged into `erp-legal-frontend` |
| BE-03 | `ConfigLoader (YAML + env)` — replaced by `erp_core/config.py` with Pydantic BaseSettings + Azure Key Vault |
| DB-17 / DB-18 | `hr` / `platform` schema tables — out of scope for current architecture |

---

## Task Summary by Assignee

| Assignee | Task IDs |
|----------|---------|
| **Dimo** | INFRA-01, INFRA-02, INFRA-06, INFRA-08, INFRA-11, INFRA-12, INFRA-17, INFRA-19, INFRA-21, DB-04, DB-07, DB-09, DB-13, DB-15, DB-17, DB-18, DB-20, PROD-01, PROD-02, PROD-06, PROD-07 |
| **Martin** | INFRA-03, INFRA-04, INFRA-07, INFRA-09, INFRA-10, INFRA-13, INFRA-14, INFRA-15, INFRA-16, INFRA-18, INFRA-20, INFRA-21, CORE-01, CORE-04, CORE-07, CORE-09, CORE-11, CORE-13, CONT-04, CONT-05, CONT-08, DB-21, DB-22, BE-A-01, BE-A-02, BE-A-04, BE-A-06, BE-A-07, BE-A-09, BE-A-11, BE-A-13, CEL-01–CEL-04, AI-01–AI-05, AI-SRC-01–AI-SRC-07, FE-SH-12, FE-SH-13, FE-L-07, FE-M-07, FE-AC-08, PROD-03, PROD-04, PROD-06, PROD-08 |
| **Ivan** | INFRA-05, DB-01–DB-03, DB-05, DB-06, DB-08, DB-10, DB-11, DB-12, DB-14, DB-16, DB-19, CORE-02, CORE-06, CORE-10, CONT-01, CONT-02, BE-L-02, BE-L-03, BE-L-06, BE-L-08, BE-A-03, BE-A-05, BE-A-08, BE-A-10, AI-ANA-01–AI-ANA-06 |
| **Veronika** | BE-L-01, BE-L-04, BE-L-05, BE-L-07, BE-M-03, BE-M-05, BE-M-07, CONT-03, FE-SH-03, FE-SH-04, FE-SH-09, FE-SH-10, FE-L-03, FE-L-05, FE-M-03, FE-M-05, FE-AC-03, FE-AC-05, FE-AC-07, AI-GEN-01–AI-GEN-07 |
| **Evsin** | CORE-03, CORE-05, CORE-08, CORE-12, CONT-06, CONT-07, BE-L-09, BE-M-01, BE-M-06, BE-M-08, BE-A-12, AI-GEN-08, AI-ING-01–AI-ING-07, AI-CLS-01–AI-CLS-05, AI-SRC-06 |
| **Yoana** | FE-SH-01, FE-SH-02, FE-SH-05, FE-SH-06, FE-SH-07, FE-SH-08, FE-SH-11, FE-L-01, FE-L-02, FE-L-04, FE-L-06, FE-M-01, FE-M-02, FE-M-04, FE-M-06, FE-AC-01, FE-AC-02, FE-AC-04, FE-AC-06, QA-12, QA-13 |
| **Gosho** | QA-01–QA-13, PROD-05 |

