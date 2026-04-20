# Architecture Specification
**Last Updated:** April 20, 2026

## Overview
A web-based ERP application consisting of 3 independently usable modules: **Legal**, **Marketing**, and **Accounting**. Each module can be used standalone or as part of the integrated platform.

---

## Architecture Decisions

### 1. Frontend — Core Shell (Micro-Frontend)
- **Technology:** React.js (TypeScript)
- **Pattern:** Webpack Module Federation
- The **Shell App** hosts the core frontend: shared styles, design system, navigation, and global auth state.
- Shared dependencies (React, design system) are loaded once from the shell using `singleton: true` in Module Federation config to prevent version conflicts.

### 2. Frontend — Module Frontends (Micro-Frontend Remotes)
- **3 Remote Apps:** Legal, Marketing, Accounting — each an independent React.js application.
- Exposed and consumed via Webpack Module Federation.
- Each module team can build, test, and deploy their frontend **independently** without redeploying the shell.
- Responsive design (desktop, tablet, mobile) with WCAG accessibility compliance.

### 3. Core Backend — Django
- **Technology:** Django (Python)
- **Responsibility:** Data layer only — ORM, database migrations, admin panel, and shared data models.
- Django owns the PostgreSQL connection and all schema migrations.
- FastAPI services must **not** write directly to the DB — they interact via Django's internal service layer or a shared `core` Python package.

### 4. Shared Backend Core — Python Package (`erp-core`)

A shared internal Python package (`erp-core`) installed as a dependency in all FastAPI services and the Django backend. It contains all common cross-cutting logic so no service reimplements it independently.

- **Package location (monorepo):** `/packages/erp-core`
- **Installed via:** `pip install -e ../../packages/erp-core` in each service's `pyproject.toml`
- **Contents:**

| Module | Path | Responsibility |
|--------|------|----------------|
| Logging | `erp_core/logging.py` | Structured JSON logging with `correlation_id`, `service_name`, `user_id` fields; OpenTelemetry trace injection |
| Config | `erp_core/config.py` | Pydantic `BaseSettings` class; loads from Azure Key Vault + env vars with validation |
| Interfaces | `erp_core/interfaces/` | Abstract base classes (ABCs) for service-layer patterns: `BaseService`, `BaseRepository`, `BaseEventPublisher` |
| Exceptions | `erp_core/exceptions.py` | Shared exception hierarchy (`AppError`, `NotFoundError`, `ConflictError`, `ValidationError`) mapped to HTTP status codes |
| Middleware | `erp_core/middleware/` | FastAPI middleware for: request ID injection, structured access logging, error response formatting |
| Messaging | `erp_core/messaging/` | Azure Service Bus publisher/subscriber base classes; RabbitMQ adapter for local dev |
| HTTP Client | `erp_core/http_client.py` | Shared `httpx` async client with retry policy, timeout config, and `X-Correlation-ID` header propagation |
| Health Check | `erp_core/health.py` | Standard `/health` and `/ready` endpoint factory, included in all FastAPI apps |
| Security | `erp_core/security.py` | JWT claim extraction helpers (user ID, roles, module access) from gateway-forwarded headers |
| Constants | `erp_core/constants.py` | Shared enums: `ModuleName`, `AuditAction`, `EventType` |

- **Rule:** No business logic lives in `erp-core` — it is infrastructure and plumbing only.
- **Versioning:** `erp-core` uses semantic versioning; breaking changes require a major version bump and a migration note in `CHANGELOG.md`.

---

### 5. Module Backend Services — FastAPI
- **Technology:** FastAPI (Python), one service per module (Legal, Marketing, Accounting).
- **Responsibility:** Business logic, module-specific API endpoints, async task handling.
- All services import `erp-core` for logging, config, middleware, messaging, and base interfaces — no duplication of cross-cutting concerns.
- Services are **auth-free** — authentication is enforced at the API Gateway level before requests reach FastAPI.
- Communication between module services uses **Azure Service Bus** (async/event-driven).
- **All inter-service communication must be governed by explicit API contracts**, defined as shared Pydantic v2 models in a `contracts` package (monorepo: `/packages/contracts`). See [`documentation/api_contracts.md`](documentation/api_contracts.md) for the full contract specifications.
- Contract compliance is enforced in CI via **Pact** consumer-driven contract testing — no service may be deployed if it violates a contract.

### 6. Database — PostgreSQL with Separate Schemas
- **Technology:** Azure Database for PostgreSQL (Flexible Server), Multi-AZ for high availability.
- Single shared PostgreSQL instance with **separate schemas** per module (`legal`, `marketing`, `accounting`, `core`) for data isolation.
- **pgvector extension** enabled for AI embedding storage (MVP phase).
- Migrations managed exclusively by Django.

### 7. API Gateway & Authentication
- **API Gateway:** Azure API Management (APIM)
  - Single entry point for all frontend-to-backend communication.
  - Handles JWT token validation, rate limiting, routing, and CORS.
  - Routes requests to the appropriate FastAPI module service.
- **Identity Provider:** Azure Entra ID (formerly Azure AD)
  - OAuth2 / PKCE flow for frontend SPAs.
  - Role-Based Access Control (RBAC) per module configured at the gateway level.
  - FastAPI services trust the gateway — no auth logic in individual services.

### 8. AI Services
- **LLM Provider:** Azure OpenAI Service
  - GPT-4o for analysis, generation, and reasoning tasks.
  - `text-embedding-3-large` for generating vector embeddings.
  - Hosted within Azure VNet for private, secure access.
- **AI Orchestrator:** LangGraph (Supervisor pattern)
  - Coordinates per-task AI agents: Ingestion Agent, Analysis Agent, Generation Agent, etc.
  - Ensures agents work together without conflict.
- **Vector Database Strategy:**
  - **MVP:** `pgvector` extension on existing PostgreSQL — zero additional cost or infrastructure.
  - **Scale:** Migrate to **Azure AI Search** for hybrid search (vector + keyword), native Azure integration, and multi-module tenancy support.

### 9. Async Messaging
- **Production:** Azure Service Bus (fully managed, queues + topics/subscriptions)
- **Local Development:** RabbitMQ via Docker Compose
- Used for: inter-module events, background AI tasks, audit log streaming, notifications.

### 10. Caching & Load Balancing
- **Cache:** Azure Cache for Redis (clustered mode)
  - Session storage, rate limiting, AI response caching to reduce Azure OpenAI costs.
- **Load Balancing:** Azure API Management + Azure Container Apps built-in ingress (no Nginx Ingress Controller needed)

### 11. Deployment — Azure Container Apps (ACA)

Every service is a **separate Docker container** deployed as an independent **Azure Container App** inside a shared **Container Apps Environment** (one per environment: dev / staging / prod). There are **no VMs to manage** — Azure handles all underlying infrastructure. You pay only for actual CPU and memory consumed per second, with native scale-to-zero.

> **Why ACA over AKS:** AI services call Azure OpenAI API only — no local model inference, no GPU nodes required. ACA eliminates VM baseline costs, provides native scale-to-zero, and reduces operational overhead significantly while fully supporting our microservices architecture.

#### Container App Map

| Service | Container Image | Min Replicas | Max Replicas | Ingress | Notes |
|---------|----------------|-------------|-------------|---------|-------|
| **Shell Frontend** | `acr/shell-frontend` | 1 | 5 | External (public) | Nginx serving static React build |
| **Legal Frontend** | `acr/legal-frontend` | 1 | 5 | External (public) | Nginx serving static React build |
| **Marketing Frontend** | `acr/marketing-frontend` | 1 | 5 | External (public) | Nginx serving static React build |
| **Accounting Frontend** | `acr/accounting-frontend` | 1 | 5 | External (public) | Nginx serving static React build |
| **Django Core** | `acr/django-core` | 1 | 6 | Internal only | Gunicorn + Django; unreachable from internet |
| **Legal FastAPI** | `acr/legal-api` | 1 | 10 | Internal (via APIM) | Uvicorn |
| **Marketing FastAPI** | `acr/marketing-api` | 1 | 10 | Internal (via APIM) | Uvicorn |
| **Accounting FastAPI** | `acr/accounting-api` | 1 | 10 | Internal (via APIM) | Uvicorn |
| **AI Orchestrator** | `acr/ai-orchestrator` | 0 | 5 | Internal only | LangGraph; calls Azure OpenAI API; scales to zero when idle |
| **Celery Worker** | `acr/celery-worker` | 0 | 8 | None (no ingress) | Background tasks; scales to zero when queue empty |

#### Traffic Flow

```
User Browser
    │
    ▼
Azure Front Door (CDN + WAF + TLS termination)
    │
    ├──► Static assets ──► Frontend Container Apps (Nginx)
    │
    └──► /api/* ──► Azure API Management (APIM)
                        │  (JWT validation, rate limiting, routing)
                        ├──► legal-api.internal.env
                        ├──► marketing-api.internal.env
                        └──► accounting-api.internal.env
                                    │
                                    ▼
                        django-core.internal.env  (internal only)
                                    │
                                    ▼
                        Azure DB for PostgreSQL (outside ACA env)

                        AI Orchestrator ──► Azure OpenAI API (HTTPS)
```

#### Key Deployment Rules

- **No VMs to manage.** Azure Container Apps Environment runs on serverless infrastructure — you define CPU/memory per container, Azure handles placement.
- **Frontends** are built to static files (`npm run build`) and served by a minimal Nginx container.
- **Django Core** has internal-only ingress. Only FastAPI Container Apps and the AI Orchestrator can reach it by internal DNS (`django-core` within the environment).
- **Each Container App** has its own `Dockerfile` and its own CI/CD pipeline — fully independent deployments.
- **`erp-core` package** is baked into each backend container image at build time (`pip install`). It is not a running service.
- **Environments:** `dev`, `staging`, and `prod` are separate Container Apps Environments, each in its own Azure Resource Group.
- **Secrets** are never baked into images — injected at runtime via **Azure Key Vault** references in Container App configuration.

#### Autoscaling Strategy

Azure Container Apps has **built-in autoscaling** with native scale-to-zero. No HPA, KEDA, or Cluster Autoscaler configuration needed — it is all declarative per container app.

| Service | Scale Trigger | Min | Max | Scale-to-zero |
|---------|--------------|-----|-----|---------------|
| Legal FastAPI | HTTP concurrent requests | 1 | 10 | ❌ (prod) / ✅ (dev) |
| Marketing FastAPI | HTTP concurrent requests | 1 | 10 | ❌ (prod) / ✅ (dev) |
| Accounting FastAPI | HTTP concurrent requests | 1 | 10 | ❌ (prod) / ✅ (dev) |
| Django Core | HTTP concurrent requests | 1 | 6 | ❌ (prod) / ✅ (dev) |
| AI Orchestrator | Azure Service Bus queue depth | 0 | 5 | ✅ Always |
| Celery Worker | Azure Service Bus queue depth | 0 | 8 | ✅ Always |
| All Frontends | HTTP concurrent requests | 1 | 5 | ❌ (prod) / ✅ (dev) |

> Default scale trigger: **10 concurrent HTTP requests per replica** before adding a new instance. Configurable per app.

**Cost guardrails:**
- Azure Budget Alerts fire at 80% and 100% of monthly spend targets.
- AI Orchestrator and Celery Workers scale to zero when idle — zero cost outside business hours.
- Dev and staging environments scale all apps to zero overnight via minimum replica schedules.

### 12. CI/CD Pipeline
- **Technology:** GitHub Actions or Azure DevOps Pipelines
- Per-service pipelines: lint → unit tests → integration tests → build Docker image → push to **Azure Container Registry (ACR)** → deploy to **Azure Container Apps** via `az containerapp update`.
- Environment promotion: dev → staging → prod with manual approval gate before prod.
- Each Container App is updated independently — deploying Legal FastAPI does not touch Marketing FastAPI.

### 13. Monitoring & Logging
- **Metrics & Dashboards:** Azure Monitor + Container Apps built-in metrics (CPU, memory, replica count, request latency) fed into **Grafana** via Azure Monitor data source.
- **Logging:** All container stdout/stderr streams automatically collected by Azure Container Apps into **Azure Monitor Log Analytics** — no log agent setup needed.
- **Alerting:** Azure Monitor Alerts on replica count, HTTP error rate, and latency thresholds.
- **Distributed Tracing:** OpenTelemetry across all FastAPI services, exporting to **Azure Application Insights**.

### 14. Testing Strategy
- **Unit Tests:** pytest (backend), Jest + React Testing Library (frontend).
- **Integration Tests:** Per-service API tests using pytest + httpx.
- **End-to-End Tests:** Playwright (covers full micro-frontend flows).
- **AI Agent Tests:** LangSmith for tracing and evaluating AI agent outputs.

### 15. Security
- GDPR compliance: PII tagging in data models, soft-delete + anonymization, audit log tables per schema, Azure region data residency.
- Regular security audits, OWASP secure coding practices.
- All secrets managed via **Azure Key Vault**.
- Network isolation: services communicate within Azure VNet; no direct public DB access.

### 16. Data Backup & Disaster Recovery
- Azure Database for PostgreSQL automated backups (point-in-time restore, up to 35 days).
- Geo-redundant backup storage for critical data.
- Container Apps recovery via redeployment from ACR image tags — each environment's image SHAs are stored in Git, enabling full environment rebuild in minutes.

### 17. Documentation
- **API Docs:** Auto-generated via FastAPI's OpenAPI/Swagger UI.
- **Inter-Service Contracts:** See [`documentation/api_contracts.md`](documentation/api_contracts.md) for full REST and async event contracts between all FastAPI services.
- **Architecture Docs:** C4 model diagrams (Context, Container, Component).
- **ADRs:** Architecture Decision Records for all major decisions (stored in `/docs/adr`).
- **Codebase:** Docstrings enforced via linting; README per service.

### 18. Version Control
- **Git** with GitHub or Azure DevOps Repos.
- Monorepo structure (recommended) using Nx or Turborepo for managing frontend remotes + backend services in a single repo with independent deployment.

---

## Cost Estimation (Azure, per day)

> **Assumptions:** Single Azure region, Azure West Europe pricing (April 2026). ACA Consumption plan: $0.000004/vCPU-second, $0.0000005/GiB-second. Dev/staging scale to zero overnight. No VMs — you pay only for actual container usage.

### Scenario 1 — MVP / Development Phase
*Small internal team, dev environment, scale-to-zero for all services when idle.*

| Component | Config | Daily Cost |
|-----------|--------|-----------|
| ACA Compute | 8 containers, ~0.25 vCPU / 0.5 GB avg, ~6 hrs active | $2 |
| ACA Compute — AI Orchestrator + Celery | Scale-to-zero, ~1 hr/day active | $0.10 |
| Azure DB for PostgreSQL | 2 vCore, 8 GB RAM, no HA | $5 |
| Azure API Management | Developer tier | $2 |
| Azure Cache for Redis | C0 Basic (250 MB) | $1 |
| Azure OpenAI | Light usage (~200K tokens/day) | $2 |
| Azure Service Bus | Standard, low volume | $0.05 |
| Azure Container Registry | Basic | $0.17 |
| Azure Key Vault | Low operations | $0.10 |
| Azure Blob Storage | 50 GB | $0.05 |
| Azure Monitor / Log Analytics | 1 GB/day ingested | $2.30 |
| **Total** | | **~$15 / day (~$450 / month)** |

> **vs AKS equivalent: ~$52/day — 71% cheaper**

---

### Scenario 2 — Production (Moderate Traffic)
*Live application, 20–50 concurrent users, HA on DB, all 3 environments.*

| Component | Config | Daily Cost |
|-----------|--------|-----------|
| ACA Compute — APIs + Django + Frontends | 8 containers, min 1 replica 24/7, avg 0.5 vCPU / 1 GB | $5 |
| ACA Compute — AI Orchestrator + Celery | Scale-to-zero, ~4 hrs/day active, 0.5 vCPU / 1 GB | $0.50 |
| Azure DB for PostgreSQL | 4 vCore, 16 GB RAM, Multi-AZ HA | $16 |
| Azure API Management | Standard tier | $7 |
| Azure Cache for Redis | C1 Standard (1 GB) | $1.30 |
| Azure OpenAI | Moderate (~1M tokens/day) | $10 |
| Azure Entra ID P1 | 50 users | $10 |
| Azure Service Bus | Standard, moderate volume | $0.10 |
| Azure Front Door | Standard tier (CDN + WAF) | $1.50 |
| Azure Container Registry | Standard | $0.67 |
| Azure Key Vault | Standard operations | $0.10 |
| Azure Blob Storage | 200 GB | $0.12 |
| Azure Monitor / Log Analytics | 3 GB/day ingested | $6.90 |
| **Total** | | **~$59 / day (~$1,770 / month)** |

> **vs AKS equivalent: ~$120/day — 51% cheaper**

---

### Scenario 3 — Scale (High Traffic / Peak Load)
*High concurrency, heavy AI usage, 200+ users, autoscaling firing.*

| Component | Config | Daily Cost |
|-----------|--------|-----------|
| ACA Compute — APIs + Django + Frontends | 8 containers, avg 5 replicas, 1 vCPU / 2 GB, 24/7 | $20 |
| ACA Compute — AI Orchestrator + Celery | Scale-to-zero, avg 3 replicas, 1 vCPU / 2 GB, 10 hrs/day | $3 |
| Azure DB for PostgreSQL | 8 vCore, 32 GB RAM, Multi-AZ HA | $30 |
| Azure API Management | Standard tier | $7 |
| Azure Cache for Redis | C2 Standard (6 GB) | $3 |
| Azure OpenAI | Heavy (~5M tokens/day) | $50 |
| Azure Entra ID P1 | 200 users | $40 |
| Azure Service Bus | Standard, high volume | $1 |
| Azure Front Door | Standard + high data transfer | $5 |
| Azure Container Registry | Standard | $0.67 |
| Azure Key Vault | High operations | $0.50 |
| Azure Blob Storage | 1 TB | $0.60 |
| Azure Monitor / Log Analytics | 10 GB/day ingested | $23 |
| **Total** | | **~$184 / day (~$5,520 / month)** |

> **vs AKS equivalent: ~$340/day — 46% cheaper**

---

### Cost Optimization Tips
- **Azure OpenAI tokens** remain the biggest cost driver at scale — cache responses in Redis; use GPT-4o-mini for classification/triage tasks (15x cheaper than GPT-4o).
- **Log Analytics ingestion** — filter verbose logs at source before ingesting; target under 5 GB/day to keep costs under $12/day.
- **ACA is already scale-to-zero** — AI Orchestrator and Celery cost nothing when idle; no extra configuration needed.
- **Azure Reserved Capacity** on PostgreSQL (1-year commit) saves ~35%.
- **Entra ID Free tier** covers basic auth for early-stage — upgrade to P1 only when Conditional Access / MFA enforcement is required.

---

## Architecture Decision Summary

| Layer | Technology |
|-------|------------|
| Core Data Backend | Django + PostgreSQL (Azure DB for PostgreSQL) |
| Shared Backend Core | `erp-core` Python package (`/packages/erp-core`) |
| Module APIs | FastAPI (one service per module) |
| Frontend Shell | React.js + Webpack Module Federation (Shell App) |
| Module Frontends | React.js (3 Remote Apps via Module Federation) |
| Auth / API Gateway | Azure API Management + Azure Entra ID (OAuth2/JWT) |
| AI Models | Azure OpenAI (GPT-4o + text-embedding-3-large) |
| AI Orchestration | LangGraph (Supervisor pattern) |
| Vector DB | pgvector (MVP) → Azure AI Search (scale) |
| Async Messaging | Azure Service Bus (prod) / RabbitMQ (local dev) |
| Cache | Azure Cache for Redis |
| Deployment | Azure Container Apps (ACA) — Consumption plan, no VMs |
| CI/CD | GitHub Actions / Azure DevOps |
| Monitoring | Azure Monitor + Grafana + Application Insights |
| Secrets | Azure Key Vault |
| Storage | Azure Blob Storage |
| CDN | Azure Front Door |
