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
- **Load Balancing:** Azure API Management + AKS Ingress (Nginx Ingress Controller)

### 11. Deployment — Azure Kubernetes Service (AKS)

Every deployable unit is a **separate Docker container**, pushed to **Azure Container Registry (ACR)** and deployed as an independent **Kubernetes Deployment** on AKS. Nothing is co-located — each service scales, restarts, and is updated independently.

#### Container & Deployment Map

| Service | Container Image | K8s Deployment | Replicas (prod) | Notes |
|---------|----------------|----------------|-----------------|-------|
| **Shell Frontend** | `acr/shell-frontend` | `shell-frontend-deploy` | 2–3 | Nginx serving static React build |
| **Legal Frontend** | `acr/legal-frontend` | `legal-frontend-deploy` | 2–3 | Nginx serving static React build |
| **Marketing Frontend** | `acr/marketing-frontend` | `marketing-frontend-deploy` | 2–3 | Nginx serving static React build |
| **Accounting Frontend** | `acr/accounting-frontend` | `accounting-frontend-deploy` | 2–3 | Nginx serving static React build |
| **Django Core** | `acr/django-core` | `django-core-deploy` | 2–4 | Gunicorn + Django; internal-only (no public route) |
| **Legal FastAPI** | `acr/legal-api` | `legal-api-deploy` | 2–4 | Uvicorn; routed via APIM |
| **Marketing FastAPI** | `acr/marketing-api` | `marketing-api-deploy` | 2–4 | Uvicorn; routed via APIM |
| **Accounting FastAPI** | `acr/accounting-api` | `accounting-api-deploy` | 2–4 | Uvicorn; routed via APIM |
| **AI Orchestrator** | `acr/ai-orchestrator` | `ai-orchestrator-deploy` | 1–2 | LangGraph supervisor; GPU node pool |
| **Celery Worker** *(optional)* | `acr/celery-worker` | `celery-worker-deploy` | 2–4 | Background task workers; same image as Django Core with different entrypoint |

#### Traffic Flow

```
User Browser
    │
    ▼
Azure Front Door (CDN + WAF + TLS termination)
    │
    ├──► Static assets ──► Nginx containers (Frontend pods)
    │
    └──► /api/* ──► Azure API Management (APIM)
                        │  (JWT validation, rate limiting, routing)
                        ├──► legal-api-svc:8001
                        ├──► marketing-api-svc:8002
                        └──► accounting-api-svc:8003
                                    │
                                    ▼
                            django-core-svc:8000  (internal only)
                                    │
                                    ▼
                        Azure DB for PostgreSQL (outside AKS)
```

#### Key Deployment Rules

- **Frontends** are built to static files (`npm run build`) and served by a minimal Nginx container — they are not Node.js servers in production.
- **Django Core** has no public route. Only FastAPI services and the AI Orchestrator can reach it via internal Kubernetes DNS (`django-core-svc`).
- **Each FastAPI service** has its own `Dockerfile`, its own Helm chart, and its own CI/CD pipeline — fully independent deployments.
- **`erp-core` package** is baked into each backend container image at build time (`pip install`). It is not a running service.
- **Horizontal Pod Autoscaler (HPA)** is configured for all backend pods; frontend pods scale based on request rate via KEDA or standard HPA.
- **Environments:** `dev`, `staging`, and `prod` are separate AKS namespaces with separate ACR image tags (`dev-latest`, `staging-<sha>`, `prod-<sha>`).
- **Secrets** are never baked into images — injected at runtime via **Azure Key Vault + CSI driver** as Kubernetes secrets.

#### Autoscaling Strategy

Three complementary layers of autoscaling are used, each operating at a different level:

**Layer 1 — Horizontal Pod Autoscaler (HPA): Pod count scaling**
Scales the number of running pod replicas within a node based on real-time metrics.

| Service | Scale Metric | Min Pods | Max Pods | Target |
|---------|-------------|----------|----------|--------|
| Legal FastAPI | CPU utilization | 2 | 10 | 70% CPU |
| Marketing FastAPI | CPU utilization | 2 | 10 | 70% CPU |
| Accounting FastAPI | CPU utilization | 2 | 10 | 70% CPU |
| Django Core | CPU utilization | 2 | 8 | 70% CPU |
| AI Orchestrator | GPU utilization / queue depth | 1 | 4 | 60% GPU |
| Frontend pods (all 4) | HTTP request rate | 2 | 6 | 1000 req/s per pod |

> HPA polls metrics every **15 seconds**. Scale-down has a **5-minute stabilization window** to prevent thrashing.

**Layer 2 — KEDA (Kubernetes Event-Driven Autoscaling): Queue-driven scaling**
For services that process async tasks from **Azure Service Bus**, KEDA scales pods directly based on queue depth — more accurate than CPU for event-driven workloads.

| Deployment | KEDA Trigger | Scale-to-zero | Notes |
|------------|-------------|---------------|-------|
| AI Orchestrator | Service Bus queue depth | ✅ Yes (dev/staging) | Scales to 0 when no tasks queued |
| Celery Worker | Service Bus queue depth | ✅ Yes (dev/staging) | Background task processing |
| Legal FastAPI | Service Bus topic lag | ❌ No (prod) | Min 2 replicas always up in prod |

**Layer 3 — Cluster Autoscaler (CA): Node count scaling**
When pods cannot be scheduled because all nodes are full, the AKS Cluster Autoscaler adds new VM nodes to the pool automatically.

```
AKS Node Pools:
┌─────────────────────────────────────────────────────┐
│  system-pool    (Standard_D4s_v3)   min:2  max:5    │  ← K8s system pods
│  app-pool       (Standard_D8s_v3)   min:3  max:15   │  ← All app services
│  gpu-pool       (Standard_NC6s_v3)  min:0  max:3    │  ← AI Orchestrator only
└─────────────────────────────────────────────────────┘
```
- `gpu-pool` scales to **0 nodes at night** (outside business hours) to eliminate GPU cost when idle.
- `app-pool` scale-out triggers when pod pending time exceeds **2 minutes**.
- Scale-in (removing nodes) has a **10-minute cooldown** and only removes nodes where all pods can be safely rescheduled.

**Vertical Pod Autoscaler (VPA) — Recommendation mode only**
VPA runs in `Off` mode (recommendations only, no automatic resizing) to prevent disruptive pod restarts in production. Engineering reviews VPA recommendations weekly to adjust resource requests/limits in Helm values.

**Cost guardrails:**
- Azure Budget Alerts fire at 80% and 100% of monthly spend targets.
- `gpu-pool` has a hard `maxReplicas: 3` cap to prevent runaway GPU cost.
- All non-prod environments (`dev`, `staging`) allow full scale-to-zero overnight via scheduled KEDA scalers.

### 12. CI/CD Pipeline
- **Technology:** GitHub Actions or Azure DevOps Pipelines
- Per-service pipelines: lint → unit tests → integration tests → build Docker image → push to Azure Container Registry (ACR) → deploy to AKS.
- Environment promotion: dev → staging → prod with manual approval gate before prod.

### 13. Monitoring & Logging
- **Metrics & Dashboards:** Prometheus + Grafana (deployed in AKS)
- **Logging:** ELK Stack (Elasticsearch, Logstash, Kibana) or Azure Monitor + Log Analytics as managed alternative.
- **Alerting:** Grafana Alerts or Azure Monitor Alerts.
- **Distributed Tracing:** OpenTelemetry across all FastAPI services.

### 14. Testing Strategy
- **Integration Tests:** Per-service API tests using pytest + httpx.
- **End-to-End Tests:** Playwright (covers full micro-frontend flows).
- **AI Agent Tests:** LangSmith for tracing and evaluating AI agent outputs.

### 14. Security
- GDPR compliance: PII tagging in data models, soft-delete + anonymization, audit log tables per schema, Azure region data residency.
- Regular security audits, OWASP secure coding practices.
- All secrets managed via **Azure Key Vault**.
- Network isolation: services communicate within Azure VNet; no direct public DB access.

### 15. Data Backup & Disaster Recovery
- Azure Database for PostgreSQL automated backups (point-in-time restore, up to 35 days).
- Geo-redundant backup storage for critical data.
- AKS workload recovery via Helm charts and GitOps (ArgoCD or Flux).

### 16. Documentation
- **API Docs:** Auto-generated via FastAPI's OpenAPI/Swagger UI.
- **Inter-Service Contracts:** See [`documentation/api_contracts.md`](documentation/api_contracts.md) for full REST and async event contracts between all FastAPI services.
- **Architecture Docs:** C4 model diagrams (Context, Container, Component).
- **ADRs:** Architecture Decision Records for all major decisions (stored in `/docs/adr`).
- **Codebase:** Docstrings enforced via linting; README per service.

### 17. Version Control
- **Git** with GitHub or Azure DevOps Repos.
- Monorepo structure (recommended) using Nx or Turborepo for managing frontend remotes + backend services in a single repo with independent deployment.

---

## Cost Estimation (Azure, per day)

> **Assumptions:** Single Azure region, moderate EU/US business hours traffic, 3 environments (dev/staging/prod). All prices in USD based on Azure West Europe pricing (April 2026). Dev/staging use scale-to-zero overnight.

### Scenario 1 — MVP / Development Phase
*Small internal team, low traffic, minimal HA, dev environment only.*

| Component | Config | Daily Cost |
|-----------|--------|-----------|
| AKS Compute — system-pool | 2× Standard_D4s_v3 (always on) | $9 |
| AKS Compute — app-pool | 3× Standard_D8s_v3 (min nodes) | $28 |
| AKS Compute — gpu-pool | 0 nodes (scale-to-zero) | $0 |
| Azure DB for PostgreSQL | 2 vCore, 8GB RAM, no HA | $5 |
| Azure API Management | Developer tier | $2 |
| Azure Cache for Redis | C0 Basic (250MB) | $1 |
| Azure OpenAI | Light usage (~200K tokens/day) | $2 |
| Azure Service Bus | Standard, low volume | $0.05 |
| Azure Container Registry | Basic | $0.17 |
| Azure Key Vault | Low operations | $0.10 |
| Azure Blob Storage | 50GB | $0.05 |
| Azure Monitor / Log Analytics | 2GB/day ingested | $5 |
| **Total** | | **~$52 / day (~$1,560 / month)** |

---

### Scenario 2 — Production (Moderate Traffic)
*Live application, 20–50 concurrent users, HA enabled, all 3 environments.*

| Component | Config | Daily Cost |
|-----------|--------|-----------|
| AKS Compute — system-pool | 2× Standard_D4s_v3 | $9 |
| AKS Compute — app-pool | 5× Standard_D8s_v3 (HPA avg) | $46 |
| AKS Compute — gpu-pool | 1× Standard_NC6s_v3 (8hrs/day) | $7 |
| Azure DB for PostgreSQL | 4 vCore, 16GB RAM, Multi-AZ HA | $16 |
| Azure API Management | Standard tier | $7 |
| Azure Cache for Redis | C1 Standard (1GB) | $1.30 |
| Azure OpenAI | Moderate (~1M tokens/day) | $10 |
| Azure Entra ID P1 | 50 users × $6/user/month | $10 |
| Azure Service Bus | Standard, moderate volume | $0.10 |
| Azure Front Door | Standard tier (CDN + WAF) | $1.50 |
| Azure Container Registry | Standard | $0.67 |
| Azure Key Vault | Standard operations | $0.10 |
| Azure Blob Storage | 200GB | $0.12 |
| Azure Monitor / Log Analytics | 5GB/day ingested | $12 |
| **Total** | | **~$120 / day (~$3,600 / month)** |

---

### Scenario 3 — Scale (High Traffic / Peak Load)
*High concurrency, autoscaler firing, heavy AI usage, 200+ users.*

| Component | Config | Daily Cost |
|-----------|--------|-----------|
| AKS Compute — system-pool | 3× Standard_D4s_v3 | $14 |
| AKS Compute — app-pool | 12× Standard_D8s_v3 (HPA max) | $110 |
| AKS Compute — gpu-pool | 3× Standard_NC6s_v3 (12hrs/day) | $32 |
| Azure DB for PostgreSQL | 8 vCore, 32GB RAM, Multi-AZ HA | $30 |
| Azure API Management | Standard tier | $7 |
| Azure Cache for Redis | C2 Standard (6GB) | $3 |
| Azure OpenAI | Heavy (~5M tokens/day) | $50 |
| Azure Entra ID P1 | 200 users | $40 |
| Azure Service Bus | Standard, high volume | $1 |
| Azure Front Door | Standard + high data transfer | $5 |
| Azure Container Registry | Standard | $0.67 |
| Azure Key Vault | High operations | $0.50 |
| Azure Blob Storage | 1TB | $0.60 |
| Azure Monitor / Log Analytics | 20GB/day ingested | $46 |
| **Total** | | **~$340 / day (~$10,200 / month)** |

---

### Cost Optimization Tips
- **Biggest cost driver at scale: Log Analytics ingestion** — filter noisy logs at source (Logstash/Fluent Bit) before ingesting to cut 40–60% of log cost.
- **Second biggest: Azure OpenAI tokens** — cache frequent AI responses in Redis; use GPT-4o-mini for classification/triage tasks instead of GPT-4o.
- **GPU pool** scales to 0 outside business hours — saves ~$20/day vs always-on.
- **Azure Reserved Instances** on app-pool and DB (1-year commit) saves ~35% on compute — worthwhile once traffic patterns stabilise.
- **Dev/staging environments** use scale-to-zero KEDA schedulers overnight, cutting non-prod cost by ~60%.

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
| Deployment | AKS (Azure Kubernetes Service) + Docker |
| CI/CD | GitHub Actions / Azure DevOps |
| Monitoring | Prometheus + Grafana + ELK / Azure Monitor |
| Secrets | Azure Key Vault |
| Storage | Azure Blob Storage |
| CDN | Azure Front Door / Cloudflare |
