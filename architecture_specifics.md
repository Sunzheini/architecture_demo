# Architecture Specification
**Last Updated:** April 20, 2026

## Team
Frontend: Veronika, Yoana
Database: Ivan, Dimo
Backend: Veronika, Ivan, Martin, Evsin
QA: Gosho
Devops: Dimo, Martin


## Overview
A web-based ERP application consisting of 3 independently usable modules: **Legal**, **Marketing**, and **Accounting**. 
Each module can be used standalone or as part of the integrated platform.

---

## Architecture Decisions

### 1. Frontend — Core Shell (Micro-Frontend)
- **Technology:** React.js (TypeScript)
- **Pattern:** Webpack Module Federation
- The **Shell App** hosts the core frontend: shared styles, design system, navigation, and global auth state.
- Shared dependencies (React, design system) are loaded once from the shell using `singleton: true` in Module Federation 
config to prevent version conflicts.

### 2. Frontend — Module Frontends (Micro-Frontend Remotes)
- **3 Remote Apps:** Legal, Marketing, Accounting — each an independent React.js application.
- Exposed and consumed via Webpack Module Federation.
- Each module team can build, test, and deploy their frontend **independently** without redeploying the shell.
- Responsive design (desktop, tablet, mobile) with WCAG accessibility compliance.

### 3. Data Layer — Django
- **Technology:** Django (Python)
- **Responsibility:** Database management only — ORM, schema migrations, admin panel, and shared data models. It is **not** a general backend; it owns 
the database and nothing else.
- Django owns the PostgreSQL connection and all schema migrations.
- FastAPI services must **not** write directly to the DB — they interact via Django's internal service layer or the shared `erp-core` Python package.

### 4. Shared Backend Core — Python Package (`erp-core`)
A shared internal Python package (`erp-core`) installed as a dependency in all FastAPI services and the Django backend. It contains all common 
cross-cutting logic so no service reimplements it independently.

- **Package location (monorepo):** `/packages/erp-core`
- **Installed via:** `pip install -e ../../packages/erp-core` in each service's `pyproject.toml`
- **Contents:**

| Module | Path | Responsibility |
|--------|------|----------------|
| Logging | `erp_core/logging.py` | Structured JSON logging with `correlation_id`, `service_name`, `user_id` fields; OpenTelemetry trace injection |
| Config | `erp_core/config.py` | Pydantic `BaseSettings` class; loads from Azure Key Vault + env vars with validation |
| Interfaces | `erp_core/interfaces/` | Abstract base classes (ABCs) for service-layer patterns: `BaseService`, `BaseRepository`, `BaseEventPublisher`, `BaseAIAgent` |
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
- All **AI agent services** also import `erp-core` — they use `BaseAIAgent`, structured logging, Azure Key Vault config, and the Service Bus messaging base classes.
- Services are **auth-free** — authentication is enforced at the API Gateway level before requests reach FastAPI.
- Communication between module services uses **Azure Service Bus** (async/event-driven).
- **All inter-service communication must be governed by explicit API contracts**, defined as shared Pydantic v2 models in a `contracts` package (monorepo: `/packages/contracts`). See [`documentation/api_contracts.md`](documentation/api_contracts.md) for the full contract specifications.
- Contract compliance is enforced in CI via **Pact** consumer-driven contract testing — no service may be deployed if it violates a contract.
- AI agent input/output schemas are also defined in the `contracts` package as Pydantic v2 models (`IngestionRequest`, `AnalysisResult`, `GenerationRequest`, etc.) to ensure the Orchestrator and agents stay in sync.

### 6. Database — PostgreSQL with Separate Schemas
- **Technology:** Azure Database for PostgreSQL (Flexible Server), Multi-AZ for high availability.
- Single shared PostgreSQL instance with **separate schemas** per module (`legal`, `marketing`, `accounting`, `core`) for data isolation.
- **pgvector extension** enabled for AI embedding storage (MVP phase).
- Migrations managed exclusively by Django.

### 7. API Gateway & Authentication

#### Where does the login form live?
The **Shell Frontend** (`erp-shell-frontend`) owns the login UI — the login page, logout button, and session state displayed in the navigation bar. Module frontends (Legal, Marketing, Accounting) **never** implement their own login screens; they inherit the authenticated session from the Shell.

#### Who manages passwords and tokens?
**You write zero password or token management code.** That responsibility belongs entirely to **Azure Entra ID** (Microsoft's cloud identity service). It stores user accounts, hashes passwords, issues tokens, handles MFA, and manages token expiry and refresh. Your code only consumes the resulting JWT token.

#### How does the login flow work? (step by step)

```
1. User visits the Shell Frontend in their browser.

2. Shell detects no valid session → redirects to Azure Entra ID login page
   (hosted by Microsoft, not by your app).

3. User enters credentials on the Entra ID page.
   Entra ID validates them and issues a JWT access token + refresh token.

4. Browser is redirected back to the Shell Frontend with the JWT token.
   The Shell stores the token in memory (not localStorage — security best practice).

5. Shell Frontend loads the appropriate module frontend (Legal / Marketing / Accounting)
   and passes the JWT token to it via Module Federation shared state.

6. Module frontend makes an API call:
     GET /api/legal/contracts
     Authorization: Bearer <jwt-token>

7. Request hits Azure Front Door → Azure API Management (APIM).
   APIM validates the JWT signature against Entra ID's public keys.
   If invalid → 401 Unauthorized, request never reaches your code.
   If valid → APIM extracts user ID + roles from the token and forwards the request
   to the correct FastAPI service with the claims in headers.

8. FastAPI service receives the request. It does NOT check auth — it trusts APIM.
   erp-core's security.py reads the forwarded headers to get user ID and roles
   for use in business logic (e.g. "can this user edit this contract?").
```

#### Component responsibilities summary

| Component | Responsibility |
|-----------|---------------|
| **Azure Entra ID** | Stores users, validates passwords, issues JWT tokens, manages MFA and token refresh. **You do not write any of this.** |
| **Shell Frontend** | Shows the login button/page (redirects to Entra ID). Holds the JWT token in memory. Passes auth state to module frontends. |
| **Module Frontends** | Attach the JWT token to every API request. Show/hide UI based on roles from the token. Never handle passwords. |
| **Azure API Management** | Validates every incoming JWT token before the request reaches any backend service. Rejects invalid/expired tokens with 401. |
| **FastAPI services** | Auth-free — they trust APIM. Use `erp_core.security` to read user ID and roles from forwarded request headers. |
| **Django Data Layer** | No auth logic whatsoever — internal only, never reachable from the internet. |

#### Role-Based Access Control (RBAC)
Roles (e.g. `legal.reader`, `marketing.admin`, `accounting.editor`) are configured in Azure Entra ID and embedded in the JWT token. APIM enforces module-level access (e.g. a Marketing user cannot call `/api/legal/*`). FastAPI services perform finer-grained checks within their module using the roles from the token claims.

### 8. AI Services
All AI workloads run as **separate, independently deployable Container Apps**, each responsible for one group of tasks. They are coordinated by the 
**AI Orchestrator** (LangGraph Supervisor) which routes tasks to the appropriate agent service via Azure Service Bus.

#### 8.1 AI Orchestrator — LangGraph Supervisor
- Receives AI task requests from FastAPI module services (via Service Bus or direct HTTP).
- Determines which AI agent service(s) to invoke and in what order.
- Aggregates results and returns them to the requesting service.
- Scales to zero when no tasks are queued.

> **Streaming results back to the UI:** AI agents (in particular the Generation Agent) publish **chunked** results to the `ai.results` topic, each chunk tagged with the originating `correlation_id` and a monotonically increasing `sequence_number`. The **FastAPI module services** own the public-facing SSE endpoints (`GET /api/<module>/ai/generation/stream/{correlation_id}`); they bridge `ai.results` → SSE via the Redis Pub/Sub fan-out helper in `erp_core/streaming/sse.py`. AI agents themselves remain **internal-only and HTTP-free** — no agent serves browser traffic.

#### 8.2 AI Agent Services
| Service | Container | Transport | Responsibility |
|---------|-----------|-----------|----------------|
| **Ingestion Agent** | `acr/ai-ingestion` | Async (Service Bus) | Extracts raw text and metadata from uploaded files (PDF, DOCX, XLSX, images via OCR). Chunks content and stores embeddings in pgvector / Azure AI Search. |
| **Analysis Agent** | `acr/ai-analysis` | Async (Service Bus) | Analyses structured or unstructured content: clause risk scoring, financial anomaly detection, campaign performance insights. Calls GPT-4o. |
| **Generation Agent** | `acr/ai-generation` | Async (Service Bus, **chunked** results to `ai.results`) | Generates content: legal summaries, marketing copy, financial narratives, email drafts. Calls GPT-4o. |
| **Classification Agent** | `acr/ai-classification` | Async (Service Bus) | Fast, cheap triage: categorises documents, routes tasks, detects intent. Uses **GPT-4o-mini** to minimise cost. |
| **Search Agent** | `acr/ai-search` | **Sync HTTP** (`POST /search`) | Semantic search across all module data using vector similarity (pgvector or Azure AI Search). Returns ranked, cited results. **Min replicas = 1** (interactive workload — no cold starts); called via `erp_core.clients.SearchClient`. |

#### 8.3 LLM Provider
- **Azure OpenAI Service** — all agent services call Azure OpenAI APIs, no local model inference.
  - GPT-4o: Analysis, Generation agents.
  - GPT-4o-mini: Classification, light summarization (15× cheaper).
  - `text-embedding-3-large`: Ingestion Agent for embedding generation.
- Hosted within Azure VNet via private endpoint — no public internet exposure.

#### 8.4 Vector Database Strategy
- **MVP:** `pgvector` extension on existing PostgreSQL — zero additional infrastructure.
- **Scale:** Migrate to **Azure AI Search** for hybrid search (vector + keyword), native Azure integration, and multi-module index separation.

### 9. Async Messaging
- **Production:** Azure Service Bus (fully managed, queues + topics/subscriptions)
- **Local Development:** RabbitMQ via Docker Compose
- Used for: inter-module events, background AI tasks, audit log streaming, notifications.
- **Reliability primitives (provided by `erp-core`):**
  - **Dead-letter queues + idempotency:** every Service Bus subscriber automatically moves poison messages to DLQ after N retries; consumers track processed message IDs in Redis to handle at-least-once redelivery without double-processing.
  - **Transactional outbox:** producers write events to an `outbox` table inside the same DB transaction as their domain write; a relay process drains the outbox to Service Bus. Prevents the "DB write succeeded but Service Bus message lost" inconsistency. **No FastAPI service may publish directly to Service Bus** — all publishes go through `erp_core.messaging.outbox`.

**Dedicated AI queues:**

| Queue / Topic | Producer | Consumer |
|---------------|----------|----------|
| `ai.ingestion.requests` | FastAPI module services | Ingestion Agent |
| `ai.analysis.requests` | Orchestrator | Analysis Agent |
| `ai.generation.requests` | Orchestrator | Generation Agent |
| `ai.classification.requests` | Orchestrator | Classification Agent |
| `ai.results` | All AI agents | Orchestrator (aggregates results) **and** FastAPI module services (SSE bridge — subscribers filter by `correlation_id`; messages carry chunked `AIResultChunk` payloads with `sequence_number` for streamed UI delivery) |
| `module.events` | All FastAPI services | Cross-module subscribers |

> **Note — AI Search is sync HTTP, not Service Bus.** Search is interactive and latency-sensitive; the Search Agent exposes `POST /search` directly and is invoked via `erp_core.clients.SearchClient` (with retry / timeout / circuit breaker). It runs at min-replicas = 1 to avoid cold-start latency. There is no `ai.search.requests` queue.

### 10. Caching & Load Balancing
- **Cache:** Azure Cache for Redis (clustered mode)
  - Session storage, rate limiting, AI response caching to reduce Azure OpenAI costs.
  - **Search query cache:** the AI Search Agent caches recent query → result-set responses with a short TTL (default 60 s) to absorb repeated identical searches (autocomplete, paging) without re-running vector similarity.
- **Load Balancing:** Azure API Management + Azure Container Apps built-in ingress (no Nginx Ingress Controller needed)

### 11. Deployment — Azure Container Apps (ACA)
Every service is a **separate Docker container** deployed as an independent **Azure Container App** inside a shared **Container Apps Environment** 
(one per environment: dev / staging / prod). There are **no VMs to manage** — Azure handles all underlying infrastructure. You pay only for actual 
CPU and memory consumed per second, with native scale-to-zero.

> **Why ACA over AKS:** AI services call Azure OpenAI API only — no local model inference, no GPU nodes required. ACA eliminates VM 
> baseline costs, provides native scale-to-zero, and reduces operational overhead significantly while fully supporting our microservices architecture.

#### Container App Map
| Service | Container Image | Min Replicas | Max Replicas | Ingress | Notes |
|---------|----------------|-------------|-------------|---------|-------|
| **Shell Frontend** | `acr/shell-frontend` | 1 | 5 | External (public) | Nginx serving static React build |
| **Legal Frontend** | `acr/legal-frontend` | 1 | 5 | External (public) | Nginx serving static React build |
| **Marketing Frontend** | `acr/marketing-frontend` | 1 | 5 | External (public) | Nginx serving static React build |
| **Accounting Frontend** | `acr/accounting-frontend` | 1 | 5 | External (public) | Nginx serving static React build |
| **Django Data Layer** | `acr/django-core` | 1 | 6 | Internal only | Gunicorn + Django; DB management only; unreachable from internet |
| **Legal FastAPI** | `acr/legal-api` | 1 | 10 | Internal (via APIM) | Uvicorn |
| **Marketing FastAPI** | `acr/marketing-api` | 1 | 10 | Internal (via APIM) | Uvicorn |
| **Accounting FastAPI** | `acr/accounting-api` | 1 | 10 | Internal (via APIM) | Uvicorn |
| **AI Orchestrator** | `acr/ai-orchestrator` | 0 | 3 | Internal only | LangGraph Supervisor; routes tasks to AI agents; scales to zero when idle |
| **AI — Ingestion Agent** | `acr/ai-ingestion` | 0 | 5 | Internal only | File parsing, chunking, embedding; scales to zero when queue empty |
| **AI — Analysis Agent** | `acr/ai-analysis` | 0 | 5 | Internal only | GPT-4o powered analysis; scales to zero when queue empty |
| **AI — Generation Agent** | `acr/ai-generation` | 0 | 5 | Internal only | GPT-4o powered content generation; **publishes streamed chunks to `ai.results`** (consumed by module APIs for SSE delivery — agent itself has no HTTP ingress); scales to zero when queue empty |
| **AI — Classification Agent** | `acr/ai-classification` | 0 | 8 | Internal only | GPT-4o-mini triage/routing; cheapest agent, scales aggressively |
| **AI — Search Agent** | `acr/ai-search` | 1 | 5 | Internal (HTTP) | **Sync HTTP** `POST /search`; vector + keyword search across all modules; **min replicas = 1** to avoid cold-start on interactive search |
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
                        django-data-layer.internal.env  (DB management only, internal)
                                    │
                                    ▼
                        Azure DB for PostgreSQL (outside ACA env)

                        AI Orchestrator ──► Azure OpenAI API (HTTPS)
                            │
                            ├──► ai-ingestion.internal.env
                            ├──► ai-analysis.internal.env
                            ├──► ai-generation.internal.env
                            ├──► ai-classification.internal.env
                            └──► ai-search.internal.env
```

#### Key Deployment Rules
- **No VMs to manage.** Azure Container Apps Environment runs on serverless infrastructure — you define CPU/memory per container, Azure handles placement.
- **Frontends** are built to static files (`npm run build`) and served by a minimal Nginx container.
- **Django Data Layer** has internal-only ingress. Only FastAPI Container Apps and the AI Orchestrator can reach it by internal DNS (`django-data-layer` within the environment).
- **Each Container App** has its own `Dockerfile` and its own CI/CD pipeline — fully independent deployments.
- **`erp-core` package** is baked into each backend container image at build time (`pip install`). It is not a running service.
- **Environments:** `dev`, `staging`, and `prod` are separate Container Apps Environments, each in its own Azure Resource Group.
- **Secrets** are never baked into images — injected at runtime via **Azure Key Vault** references in Container App configuration.
- **IaC-first:** every Azure resource (ACA environments, ACR, PostgreSQL, Redis, Service Bus, Key Vault, APIM, Front Door, Entra ID app registrations, Blob Storage, Azure OpenAI, Monitor, Budget Alerts) is authored as a **Terraform** module in the `erp-infrastructure` repo and applied via `terraform apply`. **No click-ops in the Azure Portal.** Remote state lives in Azure Storage with state locking; staging and prod are provisioned by re-applying the same modules with environment-specific `tfvars`.

#### Autoscaling Strategy
Azure Container Apps has **built-in autoscaling** with native scale-to-zero. No HPA, KEDA, or Cluster Autoscaler configuration needed — it is all declarative per container app.

> **RabbitMQ vs autoscaling:** RabbitMQ is used **only in local development** (Docker Compose). It has no autoscaling — you run a fixed set of containers locally. In **production and staging**, Azure Service Bus is used, and its queue depth natively drives ACA autoscaling for all AI agents and Celery workers. The `erp-core` messaging abstraction (`BaseEventPublisher` / subscriber) means no code change is needed between environments — only a config switch (`MESSAGING_BACKEND=azure_service_bus` vs `rabbitmq`).

| Service | Scale Trigger (prod) | Min | Max | Scale-to-zero |
|---------|----------------------|-----|-----|---------------|
| Legal FastAPI | HTTP concurrent requests | 1 | 10 | ❌ (prod) / ✅ (dev) |
| Marketing FastAPI | HTTP concurrent requests | 1 | 10 | ❌ (prod) / ✅ (dev) |
| Accounting FastAPI | HTTP concurrent requests | 1 | 10 | ❌ (prod) / ✅ (dev) |
| Django Data Layer | HTTP concurrent requests | 1 | 6 | ❌ (prod) / ✅ (dev) |
| All Frontends | HTTP concurrent requests | 1 | 5 | ❌ (prod) / ✅ (dev) |
| AI Orchestrator | Azure Service Bus queue depth (`ai.*` topics) | 0 | 3 | ✅ Always |
| AI — Ingestion Agent | Azure Service Bus queue depth (`ai.ingestion.requests`) | 0 | 5 | ✅ Always |
| AI — Analysis Agent | Azure Service Bus queue depth (`ai.analysis.requests`) | 0 | 5 | ✅ Always |
| AI — Generation Agent | Azure Service Bus queue depth (`ai.generation.requests`) | 0 | 5 | ✅ Always |
| AI — Classification Agent | Azure Service Bus queue depth (`ai.classification.requests`) | 0 | 8 | ✅ Always |
| AI — Search Agent | HTTP concurrent requests | 1 | 5 | ❌ (always min 1 — interactive workload) |
| Celery Worker | Azure Service Bus queue depth (background task queue) | 0 | 8 | ✅ Always |

> Default HTTP scale trigger: **10 concurrent requests per replica** before adding a new instance. Configurable per app.
> Default queue scale trigger: **1 active message per replica** — i.e. if there are 5 messages unprocessed, ACA will spin up 5 replicas (up to the configured Max). Configurable per app.

**Cost guardrails:**
- Azure Budget Alerts fire at 80% and 100% of monthly spend targets.
- AI Orchestrator and Celery Workers scale to zero when idle — zero cost outside business hours.
- Dev and staging environments scale all apps to zero overnight via minimum replica schedules.

### 12. CI/CD Pipeline
- **Platform:** **GitHub Actions** (recommended — Microsoft-owned, native Azure integrations, free for public repos, tight ACR/ACA support via official actions).
- **Pattern:** Polyrepo — each repository has its own independent pipeline. A push to `main` on any repo triggers build → test → deploy for that service only. No other services are affected.

#### Per-Service Pipeline (runs on every push to `main`)
```
push to main
    │
    ├── lint + type check
    ├── unit tests (pytest / Jest)
    ├── integration tests
    │
    ├── docker build
    ├── push image to Azure Container Registry (ACR)
    │     tag: <sha>  +  latest
    │
    └── az containerapp update
          --name <service-name>
          --resource-group <env-rg>
          --image <acr-name>.azurecr.io/<service>:<sha>
```

#### Environment Promotion
| Branch | Auto-deploys to | Gate |
|--------|----------------|------|
| `main` | `dev` environment | Automatic on every merge |
| Release tag `v*` | `staging` environment | Automatic on tag push |
| Manual approval | `prod` environment | GitHub Environment protection rule — requires reviewer sign-off |

#### Shared Package Pipelines (`erp-core`, `contracts`)
`erp-core` and `contracts` are not deployed as services — they are Python packages. Their pipeline:
1. Run tests.
2. On version tag (e.g. `v1.2.0`) → build wheel → publish to **GitHub Packages** (private PyPI).
3. Each service repo pins a specific version in its `pyproject.toml` — a Dependabot PR is auto-raised in all dependent repos when a new version is published.

#### Release & Rollback Strategy (ACA revisions)

Every Container App runs in **multi-revision mode** with traffic splitting. A deploy never replaces the running revision in place; it ships a new revision next to the old one and shifts traffic gradually.

**Per-deploy flow (codified in the reusable workflow — INFRA-30):**
1. Build + push image to ACR with tag `<sha>`.
2. `az containerapp revision copy` creates a new revision receiving **0%** traffic.
3. Automated **smoke probe** runs against the new revision's revision-specific FQDN: `/health`, `/ready`, and one representative API call. Fail = abort, no traffic shifted.
4. Shift **10%** of traffic to the new revision (canary). Monitor error rate + latency for a configurable window (default 5 min).
5. Shift to **100%**; mark old revision inactive (kept for fast rollback).
6. **Auto-rollback** on smoke-probe failure or alert breach during the canary window: traffic is shifted back to the previous revision in ≤ **2 min** (rollback SLO).

**Database migrations** run as a separate **ACA Job** (INFRA-29), not inside the web container's startup. The job runs **before** the new revision receives traffic. Migrations are forward-only by default; breaking schema changes follow the **expand → migrate → contract** pattern documented in `docs/runbooks/db-migrations.md` (INFRA-31). Emergency rollback path: point-in-time restore from DB-20 + redeploy the previous image SHA from `infra/image-versions/<env>/` (PROD-04).

**Release governance:**
- Every repo uses **release-please** (or conventional-commits) for automatic version bumps and per-release changelogs (INFRA-32).
- Prod deploys honour a **deployment freeze window** policy enforced via GitHub Environment protection rules (e.g. no Friday-afternoon prod deploys).

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
- **Definition of Done:** every task in [`Tasks.md`](Tasks.md) inherits a project-wide Definition of Done (10 items: AC met, PR merged, ≥80% coverage, contract tests pass, type-checks/linters pass, structured logging, docs updated, observability wired, deployed to dev). A stricter **Production Definition of Done** (E2E green on staging, load test passed, OWASP ZAP + AI red-teaming clean, runbook verified, product-owner sign-off, image SHA pinned) gates every prod release. See `Tasks.md` for the full checklist and per-task acceptance criteria.

### 15. Security
- GDPR compliance: PII tagging in data models, soft-delete + anonymization, audit log tables per schema, Azure region data residency.
- Regular security audits, OWASP secure coding practices.
- All secrets managed via **Azure Key Vault**.
- Network isolation: services communicate within Azure VNet; no direct public DB access.
- **Upload hygiene:** every uploaded file is scanned by **Azure Defender for Storage** (malware) and validated for size + magic-byte content type **before** any downstream processing (e.g. AI ingestion). Infected or invalid files are quarantined and rejected.
- **AI safety (provided by `erp-core`):**
  - **PII redaction (Microsoft Presidio):** all text is passed through `erp_core.ai.pii_redactor` before being sent to Azure OpenAI. Strips names, emails, phones, IBANs, EGN/Bulstat, etc. for both EN and BG. Compliance-mandatory.
  - **Prompt-injection guardrails:** `erp_core.ai.prompt_guard` sanitises user-supplied input (strips system-prompt overrides), validates LLM output against the expected Pydantic schema, detects known jailbreak patterns, and enforces token caps.
  - **AI red-teaming:** prompt-injection and PII-leak suites run weekly in CI against all AI agents.

### 16. Data Backup & Disaster Recovery
- Azure Database for PostgreSQL automated backups (point-in-time restore, up to 35 days).
- Geo-redundant backup storage for critical data.
- Container Apps recovery via redeployment from ACR image tags — each environment's image SHAs are stored in Git, enabling full environment rebuild in minutes.
- **Full environment rebuild:** because all Azure resources are defined in Terraform (`erp-infrastructure`), an entire environment (network, ACA, DB, APIM, etc.) can be rebuilt from scratch by running `terraform apply` against a fresh subscription. Combined with image-SHA pinning above, this gives a deterministic DR path.

### 17. Documentation
- **API Docs:** Auto-generated via FastAPI's OpenAPI/Swagger UI.
- **Inter-Service Contracts:** See [`documentation/api_contracts.md`](documentation/api_contracts.md) for full REST and async event contracts between all FastAPI services.
- **Frontend ↔ Backend type bridge:** the `erp-contracts` Pydantic models are auto-converted to TypeScript and published as the npm package `@erp/contracts-ts` on every contracts release. All frontends import types from this package — hand-copying types is forbidden and CI-enforced.
- **Architecture Docs:** C4 model diagrams (Context, Container, Component).
- **ADRs:** Architecture Decision Records for all major decisions (stored in `/docs/adr`).
- **Codebase:** Docstrings enforced via linting; README per service.

### 18. Version Control — GitHub (Polyrepo)
**Platform:** GitHub. Chosen over GitLab for native Azure integration (Microsoft ecosystem), GitHub Actions built-in CI/CD, GitHub Packages for private Python packages, and GitHub Environments for deployment gates.
**Structure:** **Polyrepo** — one repository per deployable service or shared package. Each team owns their repo independently; no coordination needed for unrelated deployments.

#### Repository Map
| Repository | Type | Description |
|------------|------|-------------|
| `erp-core` | Python package | Shared backend infrastructure package. Published to GitHub Packages on version tag. |
| `erp-contracts` | Python package | Shared Pydantic v2 contracts for all inter-service communication. Published to GitHub Packages. |
| `erp-shell-frontend` | Container App | React.js Shell (Micro-Frontend host) |
| `erp-legal-frontend` | Container App | React.js Legal module frontend |
| `erp-marketing-frontend` | Container App | React.js Marketing module frontend |
| `erp-accounting-frontend` | Container App | React.js Accounting module frontend |
| `erp-django-core` | Container App | Django ORM / data layer |
| `erp-legal-api` | Container App | Legal FastAPI service |
| `erp-marketing-api` | Container App | Marketing FastAPI service |
| `erp-accounting-api` | Container App | Accounting FastAPI service |
| `erp-ai-orchestrator` | Container App | LangGraph AI Orchestrator |
| `erp-ai-ingestion` | Container App | AI Ingestion Agent |
| `erp-ai-analysis` | Container App | AI Analysis Agent |
| `erp-ai-generation` | Container App | AI Generation Agent |
| `erp-ai-classification` | Container App | AI Classification Agent |
| `erp-ai-search` | Container App | AI Search Agent |
| `erp-celery-worker` | Container App | Background task worker |
| `erp-infrastructure` | IaC only | **Terraform** modules for all Azure resources (ACA environments, ACR, PostgreSQL, Redis, Service Bus, Key Vault, APIM, Front Door, Entra ID, Blob Storage, Azure OpenAI, Monitor, Budget Alerts). Remote state in Azure Storage; CI gates: `terraform fmt`, `tflint`, `checkov`, `terraform plan` on PRs, manual-approval `terraform apply` per env. |

#### Branch Strategy (per repo)
- `main` — production-ready code; protected, requires PR + passing CI.
- `develop` — integration branch for feature work.
- Feature branches — short-lived, merged to `develop` via PR.
- Release tags (`v1.0.0`) — trigger staging and prod deployments.

---
## Cost Estimation (Azure, per day)

> **Assumptions:** Single Azure region, Azure West Europe pricing (April 2026). ACA Consumption plan: 
$0.000004/vCPU-second, $0.0000005/GiB-second. Dev/staging scale to zero overnight. No VMs — you pay only for actual container usage.

### Scenario 1 — MVP / Development Phase
*Small internal team, dev environment, scale-to-zero for all services when idle.*

| Component | Config | Daily Cost |
|-----------|--------|-----------|
| ACA Compute | 8 app containers (APIs + Django + Frontends), ~0.25 vCPU / 0.5 GB avg, ~6 hrs active | $2 |
| ACA Compute — AI services (7 containers) | Orchestrator + 5 Agents + Celery, scale-to-zero, ~1 hr/day combined | $0.20 |
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
| ACA Compute — AI services (7 containers) | Orchestrator + 5 Agents + Celery, scale-to-zero, ~4 hrs/day combined active | $1 |
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
| ACA Compute — AI services (7 containers) | Orchestrator + 5 Agents + Celery, scale-to-zero, avg 3 replicas each, 1 vCPU / 2 GB, ~10 hrs/day | $6 |
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
| **Total** | | **~$187 / day (~$5,610 / month)** |

> **vs AKS equivalent: ~$340/day — 45% cheaper**

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
| Data Layer (DB management) | Django + PostgreSQL (Azure DB for PostgreSQL) |
| Shared Backend Core | `erp-core` Python package (`/packages/erp-core`) |
| Module APIs | FastAPI (one service per module) |
| Frontend Shell | React.js + Webpack Module Federation (Shell App) |
| Module Frontends | React.js (3 Remote Apps via Module Federation) |
| Auth / API Gateway | Azure API Management + Azure Entra ID (OAuth2/JWT) |
| AI Models | Azure OpenAI (GPT-4o + GPT-4o-mini + text-embedding-3-large) |
| AI Orchestration | LangGraph Supervisor — coordinates 5 specialist AI agents |
| AI Agents | Ingestion, Analysis, Generation, Classification, Search (each a separate Container App) |
| Vector DB | pgvector (MVP) → Azure AI Search (scale) |
| Async Messaging | Azure Service Bus (prod) / RabbitMQ (local dev) |
| Cache | Azure Cache for Redis |
| Deployment | Azure Container Apps (ACA) — Consumption plan, no VMs |
| CI/CD | GitHub Actions (per-repo pipelines, auto-deploy to ACA on push to main) |
| Version Control | GitHub — Polyrepo (1 repo per service + 2 shared packages + 1 IaC repo) |
| Monitoring | Azure Monitor + Grafana + Application Insights |
| Secrets | Azure Key Vault |
| Storage | Azure Blob Storage |
| CDN | Azure Front Door |
