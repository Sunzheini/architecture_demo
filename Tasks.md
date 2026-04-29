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
- **Definition of Done** (below) applies to **every task** in this document. Per-task **Acceptance Criteria** for the highest-risk tasks are listed in the *Acceptance Criteria Appendix* at the bottom.

---

## Definition of Done (applies to every task)

A task may only be marked complete when **all** of the following are true:

1. All acceptance criteria for the task are met (see per-task AC in the appendix; if no AC is listed, the task's *Deliverable* column is the implicit AC).
2. Code merged to `main` via PR with at least 1 reviewer approval; CI is green.
3. Unit tests added/updated; service-wide coverage ≥ **80%** (≥ **90%** for `erp-core` and `erp-contracts`).
4. Integration tests added/updated where the task touches an external boundary (DB, Service Bus, HTTP, AI API).
5. API contracts in `erp-contracts` updated if request/response shapes changed; Pact contract tests pass.
6. Type checks pass (`mypy --strict` for backend, `tsc --noEmit` for frontend); linters pass (`ruff`, `eslint`).
7. Structured logging added at INFO/ERROR boundaries with `correlation_id`; no `print()` or naked `logging.info`.
8. Docs updated: relevant section of `architecture_specifics.md`, service `README.md`, and OpenAPI / Swagger schema.
9. Observability: new metrics or alerts wired into Azure Monitor / Application Insights if the task introduces user-visible behaviour or scaling-relevant code paths.
10. Deployed to **dev** environment via the standard CI/CD pipeline; smoke-checked manually or by automated probe.

### Production Definition of Done (additional gate for Phase 6 / prod release)

In addition to the project DoD above:

1. Deployed to **staging**; full Playwright E2E suite green.
2. Load test (k6 / Locust) passed at the configured autoscaling threshold.
3. Security gates green: OWASP ZAP scan clean **and** AI red-teaming suite (QA-14) clean.
4. Runbook updated: rollback procedure verified for the changed service.
5. Product owner sign-off recorded in the GitHub Environment approval.
6. Image SHA pinned in `infra/image-versions/<env>/` (PROD-04).

---

## Phase 0 — Infrastructure & DevOps Setup
> Area: **DevOps** | Repo: `erp-infrastructure`
> **IaC-first:** every Azure resource below is authored as a **Terraform** module in `erp-infrastructure` and applied via `terraform apply`. **No click-ops in the Azure Portal.** Each "provision X" task = "author the Terraform module for X + apply it to the **dev** environment". Staging and prod are then provisioned by re-applying the same modules with environment-specific `tfvars` (see INFRA-21).

| ID | Task | Deliverable | Depends On | Est. (days) |
|----|------|-------------|------------|-------------|
| INFRA-00 | Bootstrap `erp-infrastructure` Terraform repo: remote state in Azure Storage (with state locking via blob lease), module layout (`modules/`, `envs/{dev,staging,prod}/`), provider pinning, naming convention, tagging policy | `erp-infrastructure/` repo skeleton; remote backend configured; `README.md` with conventions | — | 3 |
| INFRA-01 | Author Terraform module for Azure Resource Groups + apply to all envs (dev / staging / prod) | `modules/resource-group/`; RGs created via Terraform | INFRA-00 | 1 |
| INFRA-02 | Author Terraform module for Azure Container Registry (ACR) + apply to dev | `modules/acr/`; ACR instance with repositories | INFRA-01 | 1 |
| INFRA-03 | Author Terraform module for Azure Container Apps Environment + apply to dev | `modules/aca-environment/`; ACA Environment — dev | INFRA-02 | 2 |
| INFRA-04 | Extend ACA-environment module with staging / prod tfvars (config-only, same module) | Staging + prod ACA env definitions in `envs/{staging,prod}/` (apply happens in INFRA-21) | INFRA-03 | 2 |
| INFRA-05 | Author Terraform module for Azure Database for PostgreSQL (Flexible Server) + apply to dev | `modules/postgresql/`; Multi-AZ enabled via tfvars in prod | INFRA-01 | 3 |
| INFRA-06 | Author Terraform module for Azure Cache for Redis (clustered) + apply to dev | `modules/redis/`; Redis instance | INFRA-01 | 1 |
| INFRA-07 | Author Terraform module for Azure Service Bus (queues + topics from architecture spec — note: `ai.search.requests` is **not** included; search is sync HTTP) + apply to dev | `modules/service-bus/`; namespace with all queues except search | INFRA-01 | 2 |
| INFRA-08 | Author Terraform module for Azure Key Vault (one per environment) + apply to dev | `modules/key-vault/`; Key Vault instance | INFRA-01 | 1 |
| INFRA-09 | Author Terraform module for Azure API Management + apply to dev (JWT validation policy, routing rules, VNet integration, **streaming policy: response timeout raised to 120 s + `X-Accel-Buffering: no` preserved end-to-end for `/api/<module>/ai/generation/stream/*` SSE routes**) | `modules/apim/`; APIM instance with policies as code | INFRA-01 | 4 |
| INFRA-10 | Author Terraform module for Azure Front Door (CDN + WAF + TLS) + apply to dev | `modules/front-door/`; routing `/api/*` to APIM and static assets to frontend apps | INFRA-09 | 2 |
| INFRA-11 | Author Terraform module for Azure Entra ID (app registration, scopes, roles, MFA / Conditional Access) + apply to dev | `modules/entra-id/`; roles: `legal.reader`, `legal.editor`, `marketing.admin`, `accounting.editor`, etc. | INFRA-01 | 3 |
| INFRA-12 | Author Terraform module for Azure Blob Storage (file uploads) + apply to dev | `modules/blob-storage/`; storage account + containers per module | INFRA-01 | 1 |
| INFRA-13 | Author Terraform module for Azure OpenAI Service (GPT-4o, GPT-4o-mini, text-embedding-3-large) + apply to dev | `modules/azure-openai/`; endpoints via private endpoint in VNet | INFRA-01 | 2 |
| INFRA-14 | Setup GitHub Actions: base reusable workflow (build → push ACR → deploy ACA) | `.github/workflows/deploy-template.yml` in each repo | INFRA-02, INFRA-03 | 3 |
| INFRA-15 | Configure GitHub Environments (dev / staging / prod) with protection rules | GitHub Environments with reviewer sign-off for prod | INFRA-14 | 1 |
| INFRA-16 | Author Terraform module for Azure Monitor + Log Analytics workspace + apply to dev | `modules/monitoring/`; container log routing | INFRA-03 | 2 |
| INFRA-17 | Setup Grafana dashboards (CPU, memory, replica count, latency) — provisioned via Terraform Grafana provider | `modules/grafana-dashboards/`; dashboards via Azure Monitor data source | INFRA-16 | 3 |
| INFRA-18 | Configure OpenTelemetry → Azure Application Insights (all FastAPI services); App Insights resource via Terraform | Application Insights resource; OTEL exporter config in `erp-core` | INFRA-16 | 2 |
| INFRA-19 | Author Terraform module for Azure Budget Alerts (80% and 100% spend) + apply to all envs | `modules/budget-alerts/`; alert rules per env | INFRA-01 | 1 |
| INFRA-20 | Setup Docker Compose for local development (PostgreSQL, RabbitMQ, Redis) | `docker-compose.dev.yml` shared across repos | — | 2 |
| INFRA-21 | Apply the existing Terraform stack to **staging + prod** environments (config-only re-apply with env-specific tfvars; drift check on dev) | Staging + prod fully provisioned from IaC; `terraform plan` shows zero drift on dev | INFRA-01–INFRA-13, INFRA-16, INFRA-19 | 3 |
| INFRA-22 | Add CI gates to `erp-infrastructure` repo: `terraform fmt`, `tflint`, `checkov` (security/policy), `terraform plan` on PRs, manual-approval `terraform apply` per env | `.github/workflows/iac.yml`; required status checks on `main` | INFRA-00 | 2 |
| INFRA-23 | Document IaC contribution workflow + module conventions + environment promotion runbook | `erp-infrastructure/README.md`; `docs/runbooks/iac-promotion.md` | INFRA-21, INFRA-22 | 1 |
| INFRA-24 | Provision **Pact Broker** as a self-hosted Container App (Postgres-backed) — required by CONT-07 contract tests | `modules/pact-broker/`; Pact Broker URL in Key Vault | INFRA-03, INFRA-05 | 2 |
| INFRA-25 | Configure **GitHub Packages** auth: org-level federated identity for CI; PAT for local dev; documented `pip` and `npm` install setup; secret distributed to all service repos | `docs/runbooks/github-packages.md`; `GH_PACKAGES_TOKEN` secret in every repo | INFRA-15 | 2 |
| INFRA-26 | Provision **Azure Defender for Storage** (malware scanning on Blob uploads) — backend for BE-L-11 virus scan | `modules/defender-storage/`; Defender enabled on file-upload storage account | INFRA-12 | 1 |
| INFRA-29 | Author Django **migration ACA Job** (separate Container App Job, not the web container's startup) wired into the `erp-django-core` deploy pipeline as a **pre-traffic** step; `python manage.py migrate --check` runs as a dry-run on every PR; job blocks the new revision from receiving traffic on failure | `modules/aca-job-migrate/`; `.github/workflows/migrate-job.yml` reusable step | INFRA-03, DB-22 | 2 |
| INFRA-30 | Switch all FastAPI + Django ACA apps to **multi-revision mode with traffic splitting** (canary 10% → 100%); add automated **post-deploy smoke probe** (`/health`, `/ready`, 1 representative API call) and **auto-rollback** (shift traffic back to previous revision) into `.github/workflows/deploy-template.yml`; rollback SLO ≤ 2 min | Updated `deploy-template.yml`; ACA app config in Terraform sets `revisionMode = "multiple"` | INFRA-03, INFRA-14 | 3 |
| INFRA-32 | Add **release-please** (or conventional-commits + changelog automation) to all 18 repos for automatic version bumps + per-release changelogs; configure GitHub Environment **deployment freeze windows** (e.g. no Friday-afternoon prod deploys) | `.github/workflows/release-please.yml` shipped to every repo; freeze policy in GitHub Environment protection rules | INFRA-15 | 2 |
| INFRA-37 | Author Terraform module **`modules/network/`**: VNet per env (non-overlapping CIDRs — dev `10.10.0.0/16`, staging `10.20.0.0/16`, prod `10.30.0.0/16`), subnets (`aca-infra` delegated `/23`, `private-endpoints` `/24`, `apim` `/27`, `pg-flex` delegated), NSGs (deny-by-default + explicit allows), and **private DNS zones** for postgres / redis / keyvault / blob / servicebus / acr / openai. Apply to dev. | `modules/network/`; VNet + subnets + NSGs + private DNS zones in dev | INFRA-01 | 5 |
| INFRA-38 | Update INFRA-05 / 06 / 07 / 08 / 12 / 13 modules to set `publicNetworkAccess = Disabled` and **create private endpoints + DNS A-records** in the zones from INFRA-37. Re-apply to dev. | All PaaS resources reachable only via private endpoint in the dev VNet | INFRA-37, INFRA-05, INFRA-06, INFRA-07, INFRA-08, INFRA-12, INFRA-13 | 3 |
| INFRA-39 | Update INFRA-03 (ACA Environment) to inject the `aca-infra` subnet, set `internalLoadBalancerEnabled = true` for staging/prod (External for dev), and add Front Door → ACA Private Link config. | Updated `modules/aca-environment/`; ACA env runs inside the VNet | INFRA-37, INFRA-03 | 2 |
| INFRA-40 | Update INFRA-09 (APIM) to run in **internal VNet mode**; pin SKU per env (Developer in dev, **Premium** in staging + prod). Update cost scenarios in `architecture_specifics.md` accordingly. | Updated `modules/apim/`; APIM joined to the `apim` subnet; per-env SKU `tfvars` | INFRA-37, INFRA-09 | 2 |
| INFRA-41 | Networking validation suite: `Test-NetConnection` script + `terraform plan` policy check (Checkov/conftest) that asserts (a) no Azure resource has `publicNetworkAccess = Enabled` outside the allow-list (Front Door, APIM external for dev, ACA Environment public ingress), (b) every consumed PaaS resource has a private endpoint in the same VNet. | `.github/workflows/network-policy.yml`; required status check on `erp-infrastructure` PRs | INFRA-37, INFRA-38, INFRA-22 | 2 |
| **Group total** | | | | **74** |

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
| CORE-13 | Publish `erp-core` **v1.0.0** to GitHub Packages (without Django client — unblocks Django + FastAPI scaffolding) | GitHub Packages private PyPI entry | CORE-12, INFRA-25 | 1 |
| CORE-14 | Implement `erp_core/clients/django_client.py` — typed async client for the Django internal REST API (auto-generated from Django's OpenAPI schema using `openapi-python-client`); wraps `erp_core/http_client.py` with managed-identity auth, retries, and `BaseRepository` interface compliance | `erp_core/clients/django_client.py`; CI step that regenerates the client when the Django OpenAPI schema changes | CORE-08, DB-17 | 3 |
| CORE-15 | Publish `erp-core` **v1.1.0** to GitHub Packages (includes Django client, DLQ/idempotency, transactional outbox, PII redactor, prompt guard, SSE fan-out, Search client) — required by all FastAPI services and AI agents | GitHub Packages release | CORE-14, CORE-16, CORE-17, CORE-18, CORE-19, CORE-20, CORE-21 | 1 |
| CORE-16 | Implement **DLQ + idempotency** in `erp_core/messaging`: Service Bus subscribers automatically move poison messages to DLQ after N retries; consumers track processed message IDs in Redis to handle at-least-once redelivery | `erp_core/messaging/dlq.py`, `erp_core/messaging/idempotency.py`; integration tests with Azure Service Bus emulator | CORE-07 | 4 |
| CORE-17 | Implement **transactional outbox pattern** in `erp_core/messaging`: producers write events to an `outbox` table inside the same DB transaction as their domain write; a relay process drains the outbox to Service Bus with at-least-once delivery. Required by every BE-*-publishes-event task. | `erp_core/messaging/outbox.py`; outbox table migration shipped in Django (DB-XX); relay worker mode in `erp_core.messaging` | CORE-07, DB-16 | 5 |
| CORE-18 | Implement **PII redaction layer** in `erp_core/ai/pii_redactor.py` using **Microsoft Presidio**: strips/masks PII (names, emails, phones, IBANs, EGN/Bulstat) from any text before it is sent to Azure OpenAI. All AI agents and the Generation/Analysis prompt-builders must call it. | `erp_core/ai/pii_redactor.py`; Presidio recognizer config for EN + BG; unit tests with sample legal/financial PII | CORE-01 | 4 |
| CORE-19 | Implement **prompt-injection guardrails** in `erp_core/ai/prompt_guard.py`: input sanitization (strip system-prompt overrides), output validation against expected Pydantic schema, jailbreak-pattern detection, max-length and token caps | `erp_core/ai/prompt_guard.py`; jailbreak pattern library; unit tests | CORE-01 | 3 |
| CORE-20 | Implement **SSE fan-out helper** in `erp_core/streaming/sse.py`: per-`correlation_id` channels backed by **Redis Pub/Sub** so any replica of a module API can deliver chunks to the open SSE connection regardless of which replica subscribed to `ai.results` first. Includes **15 s heartbeat** (`: heartbeat\n\n` SSE comment to keep proxies and `EventSource` alive), explicit **`event: status`** frames carrying `{"phase": "warming_up" | "generating" | "completed" | "failed", "reason"?: str}` on every workflow-state transition, client-disconnect detection, and back-pressure. Used by all module APIs for streamed AI output. | `erp_core/streaming/sse.py`; Redis Pub/Sub channel naming convention; integration test with 2 replicas + 1 producer covering: heartbeat survives 60 s of silence; `phase` transitions arrive in order | CORE-01, INFRA-06 | 3 |
| CORE-21 | Implement typed **`SearchClient`** in `erp_core/clients/search_client.py` — sync HTTP wrapper around the AI Search Agent's `POST /search` endpoint. Wraps `erp_core.http_client` with retry, timeout, **circuit breaker**, and `correlation_id` propagation; exposes `search(query, filters) -> SearchResult` typed against the contract in `erp-contracts`. | `erp_core/clients/search_client.py`; integration test against a stubbed Search Agent | CORE-08 | 2 |
| **Group total** | | | | **54** |

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
| CONT-07 | Setup Pact consumer-driven contract tests in CI | `.github/workflows/pact.yml`; Pact Broker config | CONT-02–CONT-06, INFRA-24 | 3 |
| CONT-08 | Publish `erp-contracts` v1.0.0 to GitHub Packages | GitHub Packages private PyPI entry | CONT-07, INFRA-25 | 1 |
| CONT-09 | Generate **TypeScript types** from `erp-contracts` Pydantic models using `pydantic2ts` / `datamodel-codegen`; publish as npm package `@erp/contracts-ts` to GitHub Packages on every contracts release. CI fails if types are out of date. | `scripts/generate_ts.py`; `@erp/contracts-ts` npm package; CI job in `erp-contracts` | CONT-08 | 3 |
| **Group total** | | | | **23** |

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
| DB-26 | Add `core.ai_workflow_state` table for the **LangGraph Postgres checkpointer** (used by AI-06): columns for `correlation_id`, `thread_id`, `checkpoint_data` (JSONB), `created_at`, `updated_at`, `status`; index on `(correlation_id, updated_at)`; TTL job prunes terminal states older than 30 days, keeps in-progress indefinitely | Django model + migration in `core` schema; Alembic-equivalent rollback path tested | DB-02 | 1 |
| **Group total** | | | | **63** |

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
| BE-L-06 | Implement document upload → trigger `ai.ingestion.requests` Service Bus message (only after virus scan + validation pass) | `routes/documents.py`; Service Bus publisher | BE-L-01, CORE-07, INFRA-07, BE-L-11, BE-L-12 | 3 |
| BE-L-07 | Implement AI Search integration via **sync HTTP** call to the Search Agent (`erp_core.clients.SearchClient`) — no Service Bus involved | `routes/search.py` | BE-L-01, CORE-15 | 3 |
| BE-L-08 | Implement audit logging for all write operations (via `erp-core` middleware) | Audit events in `core_audit_log` | BE-L-01, DB-06 | 2 |
| BE-L-09 | Write unit + integration tests (pytest + httpx) | `tests/` (≥80% coverage) | BE-L-01–BE-L-08 | 5 |
| BE-L-10 | Write Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | BE-L-01, INFRA-14 | 2 |
| BE-L-11 | Implement **virus / malware scan** on every uploaded document via Azure Defender for Storage (blocking — file is quarantined and rejected if Defender flags it) before publishing to `ai.ingestion.requests` | `services/upload_scanner.py`; Defender event-grid subscription | BE-L-01, INFRA-26 | 2 |
| BE-L-12 | Implement **upload validation**: max file-size limit (configurable per env), magic-byte content-type validation (reject executables disguised as PDFs/DOCX), filename sanitization | `services/upload_validator.py` | BE-L-01 | 1 |
| BE-L-13 | Implement **AI streaming SSE endpoint** `GET /api/legal/ai/generation/stream/{correlation_id}` (Content-Type `text/event-stream`): publishes the generation request to `ai.generation.requests` via the outbox, **immediately** emits `event: status {"phase":"warming_up"}` so the client never sees a blank screen, then holds the SSE connection open and pushes chunks delivered by `CORE-20`'s Redis Pub/Sub fan-out as the Generation Agent emits them (transitioning to `phase=generating` on the first chunk and `phase=completed` on terminal). **Hard ceiling 180 s**: if no terminal frame, emit `event: status {"phase":"failed","reason":"timeout"}` and close. Replaces the previously planned in-agent SSE endpoint. | `routes/ai_stream.py`; `consumers/ai_results_subscriber.py` (Service Bus → Redis Pub/Sub bridge) | BE-L-01, CORE-15, CORE-20, INFRA-07 | 2 |
| **Group total** | | | | **31** |

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
| BE-M-10 | Implement **AI streaming SSE endpoint** `GET /api/marketing/ai/generation/stream/{correlation_id}` — same pattern as BE-L-13 (immediate `phase=warming_up`, bridge `ai.results` to SSE via CORE-20, hard 180 s ceiling with `phase=failed` on timeout) | `routes/ai_stream.py`; `consumers/ai_results_subscriber.py` | BE-M-01, CORE-15, CORE-20, INFRA-07 | 2 |
| **Group total** | | | | **29** |

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
| BE-A-14 | Implement **AI streaming SSE endpoint** `GET /api/accounting/ai/generation/stream/{correlation_id}` — same pattern as BE-L-13 (immediate `phase=warming_up`, bridge `ai.results` to SSE via CORE-20, hard 180 s ceiling with `phase=failed` on timeout) | `routes/ai_stream.py`; `consumers/ai_results_subscriber.py` | BE-A-01, CORE-15, CORE-20, INFRA-07 | 2 |
| **Group total** | | | | **43** |

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
| AI-02 | Implement task routing logic — routes incoming `ai.*` requests to correct agent via Service Bus (routing-rules table + dispatch only; checkpointing → AI-06, failure semantics → AI-07) | `orchestrator/supervisor.py`; routing rules per `EventType` | AI-01, INFRA-07 | 2 |
| AI-03 | Implement result aggregation — consumes `ai.results` topic, returns to requesting FastAPI service | `orchestrator/aggregator.py` | AI-02 | 3 |
| AI-04 | Setup LangSmith tracing integration for orchestrator | LangSmith config in `erp_core/logging.py` (AI trace extensions) | AI-01 | 1 |
| AI-05 | Write Dockerfile + GitHub Actions CI/CD pipeline (scale-to-zero in ACA) | `Dockerfile`, `.github/workflows/ci-cd.yml` | AI-01, INFRA-14 | 2 |
| AI-06 | Wire **LangGraph Postgres checkpointer** into the Orchestrator (using `DB-26`'s `core.ai_workflow_state` table) so multi-step workflows survive scale-to-zero. Each step's checkpoint commits in the same DB transaction as its outbox publish (reuses CORE-17). Integration test: pause a workflow mid-flight, scale Orchestrator to 0, requeue → workflow resumes from the last committed step. | `orchestrator/checkpointer.py`; integration test under `tests/test_resume.py` | AI-02, DB-26, CORE-17 | 3 |
| AI-07 | Define **workflow failure semantics**: terminal failure publishes `AIWorkflowFailed` event to `ai.results` with `correlation_id` / `failed_step` / `error_class`; module APIs (BE-L-13 / BE-M-10 / BE-A-14) surface it via SSE; add `AIWorkflowFailed` Pydantic model to `erp-contracts` (CONT-05/06). **No automatic compensation in MVP** — documented as a manual operator runbook in PROD-07. | `orchestrator/failure.py`; `erp_contracts/ai/AIWorkflowFailed`; `docs/runbooks/ai-workflow-failure.md` | AI-02, CONT-05, CONT-06, BE-L-13, BE-M-10, BE-A-14 | 2 |
| AI-08 | Document **Orchestrator scope** in `architecture_specifics.md` §8.1: decision matrix (direct module→agent flows: Ingestion, Search vs orchestrated multi-agent flows: Analysis pipelines, end-to-end document workflows) + canonical sequence diagram (upload → ingest → classify → summary); ADR `0007-ai-orchestrator-scope.md` | Updated §8.1; `/docs/adr/0007-ai-orchestrator-scope.md` | AI-02 | 1 |
| **Group total** | | | | **16** |

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
| AI-GEN-07 | Subscribe to `ai.generation.requests`; publish **streamed chunks** to `ai.results` topic with `correlation_id` and monotonically increasing `sequence_number` (consumed by module APIs' SSE fan-out via CORE-20). First chunk sets `phase=generating`; terminal chunk sets `phase=completed` (or `failed` with `reason`). | `consumers/generation_consumer.py`; chunked `AIResultChunk` Pydantic schema in `erp-contracts` | AI-GEN-01, CORE-07 | 3 |
| AI-GEN-08 | Write Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | AI-GEN-01, INFRA-14 | 1 |
| AI-GEN-09 | Configure ACA scale rules: **prod `min replicas = 1`** for the Generation Agent (cost ≈ $0.50/day, eliminates cold-start on the streaming path); dev/staging keep `min = 0`. Add a synthetic monitor that asserts the **first-content-chunk SLO**: p95 ≤ **20 s** on cold start (dev/staging), p95 ≤ **5 s** when warm (prod). Wired into Application Insights. | Updated `terraform/aca/ai-generation/`; `tests/synthetic/first_chunk_slo.py`; alert rule | AI-GEN-07, INFRA-16 | 2 |
| AI-GEN-10 | Frontend SSE warm-up UX (reference implementation in `erp-legal-frontend`, mirrored in marketing/accounting via FE-M-* / FE-AC-* SSE pages): consume the `event: status` frames from CORE-20 and render "Preparing your response…" while `phase=warming_up`; stream content while `phase=generating`; surface `phase=failed` (with `reason`) into the global Notification Center | `src/components/AIStreamPanel.tsx` in `erp-legal-frontend`; copied/imported by FE-M-06 + FE-AC-05 streaming pages | CORE-20, FE-L-05, FE-SH-09 | 2 |
| **Group total** | | | | **25** |

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
| AI-SRC-05 | Expose `POST /search` HTTP endpoint (FastAPI + Uvicorn) with `erp-core` middleware, structured logging, and standard `/health` + `/ready` endpoints. **Sync request/response** — search is interactive and latency-sensitive; Service Bus is intentionally not used for this agent. | `routes/search.py`; `main.py` (FastAPI app factory) | AI-SRC-01, CORE-15 | 2 |
| AI-SRC-06 | Implement token usage tracking + budget enforcement (Redis-backed) | `services/token_service.py` | AI-SRC-01, INFRA-06 | 3 |
| AI-SRC-07 | Write Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | AI-SRC-01, INFRA-14 | 1 |
| **Group total** | | | | **16** |

---

## Phase 4 — Frontend

### Design (Wireframes + Figma) — must complete before any FE implementation
> Area: **Frontend / Design**

| ID | Task | Deliverable | Depends On | Est. (days) |
|----|------|-------------|------------|-------------|
| FE-DSGN-01 | Wireframes + hi-fi Figma designs for all module screens (Legal, Marketing, Accounting) — covers every page listed in FE-L-*, FE-M-*, FE-AC-* | Figma file(s) reviewed and signed off by product + module owners | — | 8 |
| FE-DSGN-02 | Define **design tokens** in Figma (colors, spacing, typography, elevation) and export to JSON for `FE-SH-02` consumption | `design-tokens.json`; Figma Tokens plugin export | FE-DSGN-01 | 2 |
| **Group total** | | | | **10** |

### Shell Frontend (Micro-Frontend Host)
> Area: **Frontend** | Repo: `erp-shell-frontend`

| ID | Task | Deliverable | Depends On | Est. (days) |
|----|------|-------------|------------|-------------|
| FE-SH-01 | Initialize Shell App (React.js + TypeScript + Webpack Module Federation) | `frontends/erp-shell/` with Module Federation host config (`singleton: true`) | — | 2 |
| FE-SH-02 | Setup shared design system + component library (shared via Module Federation) — built from `FE-DSGN-02` design tokens | `src/design-system/` (buttons, forms, typography, colors) | FE-SH-01, FE-DSGN-02 | 5 |
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
| FE-L-01 | Initialize Legal remote app (React.js + TypeScript + Webpack Module Federation remote config) | `frontends/erp-legal/` | FE-SH-01, FE-DSGN-01 | 1 |
| FE-L-02 | Implement Legal Dossier list + detail pages | `src/pages/Dossiers/` | FE-L-01, BE-L-02 | 4 |
| FE-L-03 | Implement Legal Case management UI | `src/pages/Cases/` | FE-L-01, BE-L-03 | 4 |
| FE-L-04 | Implement Legal Deadline calendar / tracker UI | `src/pages/Deadlines/` | FE-L-01, BE-L-04 | 3 |
| FE-L-05 | Implement Law Acts / Knowledge Base search UI with AI chat (SSE streaming) | `src/pages/Search/`; SSE integration with the Legal FastAPI streaming endpoint (`GET /api/legal/ai/generation/stream/{correlation_id}`) | FE-L-01, BE-L-07, BE-L-13 | 5 |
| FE-L-06 | Implement document upload UI (triggers ingestion pipeline) | `src/components/DocumentUpload.tsx` | FE-L-01, BE-L-06 | 3 |
| FE-L-07 | Write Nginx config + Dockerfile + GitHub Actions CI/CD | `nginx.conf`, `Dockerfile`, `.github/workflows/ci-cd.yml` | FE-L-01, INFRA-14 | 2 |
| **Group total** | | | | **22** |

### Marketing Frontend (Micro-Frontend Remote)
> Area: **Frontend** | Repo: `erp-marketing-frontend`

| ID | Task | Deliverable | Depends On | Est. (days) |
|----|------|-------------|------------|-------------|
| FE-M-01 | Initialize Marketing remote app | `frontends/erp-marketing/` | FE-SH-01, FE-DSGN-01 | 1 |
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
| FE-AC-01 | Initialize Accounting remote app | `frontends/erp-accounting/` | FE-SH-01, FE-DSGN-01 | 1 |
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
| QA-09 | AI agent integration test — full ingestion → analysis → generation pipeline | `tests/ai_pipeline_integration.py` | AI-ING-01–AI-GEN-08 | 5 |
| QA-10 | LangSmith evaluation — AI output quality for all agent types | LangSmith evaluation dataset + scores | AI-01–AI-SRC-07 | 3 |
| QA-11 | Security scan: OWASP ZAP against all public endpoints | ZAP scan report; fix identified issues | All BE + FE | 3 |
| QA-12 | Accessibility audit (WCAG 2.1 AA) on all frontend modules | Lighthouse / axe-core report; fix critical issues | All FE | 3 |
| QA-13 | Responsive design testing (desktop, tablet, mobile) | Cross-device test report | All FE | 2 |
| QA-14 | **AI red-teaming**: prompt-injection + PII-leak test suite run against all 5 agents weekly in CI; validates `CORE-18` (PII redactor) and `CORE-19` (prompt guard) stay effective as prompts evolve | `tests/ai_redteam/`; LangSmith evaluation suite; weekly scheduled GitHub Action | CORE-18, CORE-19, AI-01–AI-SRC-07 | 3 |
| **Group total** | | | | **48** |

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
| PROD-09 | Author + maintain Acceptance Criteria appendix for remaining ~130 tasks (highest-risk 20 are pre-seeded — see appendix); enforce DoD checklist via PR template | Updated *Acceptance Criteria Appendix* in `Tasks.md`; `.github/PULL_REQUEST_TEMPLATE.md` shipped to every repo with the DoD checklist | All phases | 5 |
| INFRA-31 | Write **expand → migrate → contract** schema-evolution runbook + **migration rollback** runbook (forward-only by default; emergency PITR from DB-20 + redeploy previous image SHA from PROD-04); verify both via a staging "game day" | `docs/runbooks/db-migrations.md`, `docs/runbooks/release-rollback.md` | INFRA-29, INFRA-30, DB-20, PROD-04 | 2 |
| **Group total** | | | | **22** |

---

## Acceptance Criteria Appendix (highest-risk tasks)

> **How to read this appendix:** Each entry below adds **specific** acceptance criteria on top of the project-wide Definition of Done. If a task is not listed here, its *Deliverable* column in the main tables is the implicit acceptance criterion. Remaining tasks will be filled in under **PROD-09** by the responsible task owner before work starts.

### Phase 0 — Infrastructure (security-critical)

| Task | Acceptance Criteria |
|---|---|
| **INFRA-09** APIM | • APIM JWT validation rejects expired and invalid-signature tokens with HTTP 401 (verified by automated pytest against the live dev APIM instance).<br>• Per-route rate limit is configurable via a Terraform variable; default 100 req/min/user.<br>• Routing rules cover Legal, Marketing, Accounting APIs and the Django internal API; **deny-by-default** for any other path.<br>• Forwarded claim headers (`x-user-id`, `x-user-roles`) are present on every upstream request. |
| **INFRA-11** Entra ID | • App registration created with the role list defined in DB-14 (`legal.reader`, `legal.editor`, `marketing.admin`, `accounting.editor`, etc.).<br>• MFA enforced via Conditional Access for any role ending in `.admin` or `.editor`.<br>• Token TTL ≤ 60 min, refresh TTL ≤ 8 h.<br>• Test user accounts seeded per environment. |
| **INFRA-26** Defender for Storage | • Defender enabled on the upload storage account in dev, staging, prod.<br>• Defender event-grid topic emits `Microsoft.Security.MalwareScanningResult` events.<br>• Cost alert configured at $5/day per env. |

### Phase 1a — `erp-core` (foundational + AI safety)

| Task | Acceptance Criteria |
|---|---|
| **CORE-14** Django client | • Client is regenerated automatically when the Django OpenAPI schema changes (CI fails if the committed client is stale).<br>• All requests include managed-identity bearer token.<br>• Retries: exponential backoff with max 3 attempts on 5xx and network errors; never retries on 4xx.<br>• Implements `BaseRepository` for at least one resource and passes the shared repository contract test. |
| **CORE-16** DLQ + idempotency | • Poison message moved to DLQ after **5** delivery attempts (configurable).<br>• Idempotency cache TTL = 24 h, keyed by `MessageId` in Redis.<br>• Re-delivery of the same `MessageId` is a no-op — verified by integration test that publishes the same message twice and asserts a single side-effect.<br>• DLQ depth exposed as a metric to Azure Monitor. |
| **CORE-17** Transactional outbox | • Outbox row insert and domain write are in the same DB transaction (verified by failure-injection test that aborts after the domain write — no Service Bus message is published).<br>• Relay process drains outbox in FIFO order with ≤ 5 s p99 lag under steady load.<br>• Outbox table TTL: rows older than 7 days and marked sent are pruned daily. |
| **CORE-18** PII redactor | • Detects and masks: BG **EGN**, **IBAN**, email, phone, person names (EN+BG), BG addresses.<br>• Achieves ≥ **95% recall** on a labelled 50-text PII corpus committed to the repo.<br>• Adds < **30 ms** latency for inputs up to 4 KB (benchmark in CI).<br>• Public API: `redact(text: str) -> RedactedText` and `is_safe(text: str) -> bool`.<br>• **Hard rule:** any call to Azure OpenAI not preceded by `redact()` fails CI's static-analysis check. |
| **CORE-19** Prompt-injection guardrails | • Strips known system-prompt-override patterns (`"ignore previous instructions"`, jailbreak corpus from the OWASP LLM Top 10).<br>• Validates LLM output against the expected Pydantic schema; non-conforming output triggers a single retry then raises `PromptGuardError`.<br>• Token cap enforced per request (configurable; default 8 K input / 2 K output).<br>• Test corpus of 30 known jailbreak attempts: ≥ **90% blocked**. |

### Phase 1b — Data Layer

| Task | Acceptance Criteria |
|---|---|
| **DB-16** Django internal REST API | • Endpoints follow `/internal/v1/<schema>/<resource>` pattern; OpenAPI schema is the source of truth.<br>• All endpoints require service-to-service auth (DB-24).<br>• Response shapes match the Pydantic models in `erp-contracts` (validated by Pact).<br>• p95 latency < **100 ms** for single-record reads against a 1M-row table on the dev DB. |
| **DB-20** Backup / restore validation | • Point-in-time restore from a 24-h-old backup completes in < 30 min on a staging-sized DB.<br>• Restored DB passes a row-count + checksum diff against the source.<br>• Geo-redundant backup verified in a secondary Azure region.<br>• Runbook (`docs/runbooks/restore.md`) walks through the exact `az postgres flexible-server restore` command and includes screenshots. |
| **DB-24** Service-to-service auth | • Only Container Apps with role `internal-api-caller` can call the Django internal API; all others return HTTP 403.<br>• Tokens validated against Entra ID's public keys; no shared secrets.<br>• Token caching in Redis with TTL = token expiry minus 60 s.<br>• Audit log entry written for every internal API call. |
| **DB-25** Schema drift contract test | • CI job spins up a fresh Django container, runs migrations, then runs every FastAPI service's integration test suite against it.<br>• Job fails (and PR is blocked) if any FastAPI integration test fails due to schema mismatch.<br>• Job runs in < 10 min total. |

### Phase 2 — Module APIs (business-correctness-critical)

| Task | Acceptance Criteria |
|---|---|
| **BE-A-04** Double-entry validation | • Debit total must equal credit total within **0.01 BGN** tolerance.<br>• Unbalanced entries return HTTP **422** with body `{"errors": [{"line": <n>, "delta": <amount>}, ...]}`.<br>• Validation runs in < **50 ms** for entries with ≤ 100 lines.<br>• Pure function — no DB writes; called from `routes/journal_entries.py` before persistence.<br>• ≥ 20 test cases covering: balanced, off-by-0.01, off-by-large, single line, empty, negative amounts. |
| **BE-A-06** Financial statements | • Balance Sheet, P&L, and Cash Flow generated for any closed period; numbers tie to the sum of underlying journal lines (verified by reconciliation test).<br>• Output formats: JSON (API) and PDF (download).<br>• Previous-period comparison column included.<br>• Generation completes in < 5 s for a year of data on the dev DB. |
| **BE-A-07** VAT + corporate tax | • VAT declaration matches the official NAP XML schema (validated against the published XSD in CI).<br>• Corporate tax declaration covers Bulgarian standard rates current as of January 2026.<br>• Generated declaration is byte-identical when re-run on the same input (deterministic).<br>• Sample declarations from a known input set are committed and diffed in CI. |
| **BE-L-11** Virus / malware scan | • Files flagged by Defender within **5 min** of upload are quarantined and the user is notified via the Notification Center.<br>• `ai.ingestion.requests` is **never** published until the Defender scan returns clean.<br>• Defender errors (timeout / unreachable) **fail closed**: file is rejected, not allowed through.<br>• Quarantined files retained for 30 days then auto-deleted. |
| **BE-L-12** Upload validation | • Max file size: **50 MB** (configurable per env via Key Vault secret).<br>• Magic-byte check rejects executables (PE, ELF, Mach-O), scripts, and archives disguised as PDF/DOCX/XLSX.<br>• Filename sanitisation strips path separators and non-printable chars; preserves Unicode (Bulgarian filenames).<br>• Rejection returns HTTP **415** with a machine-readable error code. |

### Phase 3 — AI Search Agent (sync HTTP)

| Task | Acceptance Criteria |
|---|---|
| **AI-SRC-05** HTTP `/search` endpoint | • Sync request/response — no Service Bus involvement.<br>• p95 latency ≤ **800 ms** for vector search over a 100 K-embedding corpus on the dev DB.<br>• Returns HTTP **503** within **5 s** if Azure OpenAI or pgvector is unreachable (fail-fast — circuit breaker on the caller side handles backoff).<br>• `correlation_id` header is propagated from request to response and into all downstream logs/traces.<br>• Min replicas = **1** (no cold starts); HTTP-driven autoscaling up to max = 5. |

### Phase 5 — QA

| Task | Acceptance Criteria |
|---|---|
| **QA-14** AI red-teaming | • Suite covers all 5 AI agents.<br>• Includes the OWASP LLM Top 10 prompt-injection patterns + a custom corpus of 30 ERP-domain jailbreak attempts.<br>• Includes 50 PII-leak test cases (must produce zero PII in agent output).<br>• Runs weekly in CI as a scheduled GitHub Action; failure pages on-call.<br>• Results published to the LangSmith evaluation dashboard. |

### Phase 6 — Production Readiness (the prod gate)

| Task | Acceptance Criteria |
|---|---|
| **PROD-05** Staging smoke + sign-off | • Full Playwright E2E suite green on staging.<br>• Load test (k6) at the configured autoscale threshold passes with p95 latency < target SLO and zero 5xx.<br>• Security gates green: OWASP ZAP scan + QA-14 AI red-teaming suite both clean.<br>• Sign-off recorded in `docs/signoffs/<release-tag>.md` by product owner + tech lead. |
| **PROD-06** Production go-live | • All Production DoD items (above) green.<br>• GitHub Environment manual approval recorded with reviewer name + timestamp.<br>• Smoke probe (`/health` + 1 representative API call per service) green within 5 min of deploy.<br>• Image SHAs committed to `infra/image-versions/prod/` (PROD-04). |
| **PROD-07** Runbooks | • One runbook per scenario: restore-from-backup, service-rollback, AI-agent-failover, APIM-key-rotation, incident-response.<br>• Each runbook tested in a staging "game day" within 30 days of writing.<br>• Each runbook includes: pre-checks, exact commands, validation steps, escalation contacts. |

---

## Effort Summary & Headcount Calculation

| Phase / Group | Est. (person-days) |
|---|---|
| Phase 0 — Infrastructure & DevOps | 74 |
| Phase 1a — `erp-core` package | 54 |
| Phase 1a — `erp-contracts` package | 23 |
| Phase 1b — Data Layer (Django) | 63 |
| Phase 2 — Legal FastAPI | 31 |
| Phase 2 — Marketing FastAPI | 29 |
| Phase 2 — Accounting FastAPI | 43 |
| Phase 2 — Celery Worker | 7 |
| Phase 3 — AI Orchestrator | 16 |
| Phase 3 — AI Ingestion Agent | 18 |
| Phase 3 — AI Analysis Agent | 15 |
| Phase 3 — AI Generation Agent | 25 |
| Phase 3 — AI Classification Agent | 10 |
| Phase 3 — AI Search Agent | 16 |
| Phase 4 — Design (Figma) | 10 |
| Phase 4 — Shell Frontend | 35 |
| Phase 4 — Legal Frontend | 22 |
| Phase 4 — Marketing Frontend | 22 |
| Phase 4 — Accounting Frontend | 25 |
| Phase 5 — Integration, QA & Testing | 48 |
| Phase 6 — Production Readiness | 22 |
| **Grand Total** | **608 person-days** |

### Effort by Discipline

| Discipline | Est. (person-days) | Groups |
|---|---|---|
| **DevOps** | 74 + 22 = **96** | Phase 0, Phase 6 |
| **Backend (shared + services + AI)** | 54 + 23 + 31 + 29 + 43 + 7 + 16 + 18 + 15 + 25 + 10 + 16 = **287** | Phase 1a, Phase 2, Phase 3 |
| **Database (Django data layer)** | **63** | Phase 1b |
| **Frontend / Design** | 10 + 35 + 22 + 22 + 25 = **114** | Phase 4 |
| **QA** | **48** | Phase 5 |
| **Total** | **608** | |

### Headcount Formula

```
Required FTEs (per discipline) = Discipline person-days / (Project calendar days × Focus factor)
```

- **Project calendar days** = working days in the planned schedule (e.g. 6 months ≈ 120 working days).
- **Focus factor** = 0.7 (accounts for meetings, code review, ramp-up, blockers).

**Worked example — 6-month delivery (≈ 120 working days × 0.7 focus → 84 productive days per FTE):**

| Discipline | Days | FTEs needed | Round up |
|---|---|---|---|
| DevOps | 96 | 96 / 84 = 1.14 | **2** |
| Backend | 287 | 287 / 84 = 3.42 | **4** |
| Database | 63 | 63 / 84 = 0.75 | **1** (can overlap with Backend) |
| Frontend / Design | 114 | 114 / 84 = 1.36 | **2** (1 designer-leaning + 1 dev-leaning, or 2 devs + contracted design) |
| QA | 48 | 48 / 84 = 0.57 | **1** |
| **Team total** | **608** | **7.24** | **~8 people** for a 6-month MVP |

> Adjust the calendar days and focus factor to your actual schedule to recompute the FTE requirement.

