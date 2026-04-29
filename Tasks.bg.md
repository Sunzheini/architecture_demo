# ERP Платформа — Списък със задачи
**Последна актуализация:** 28 април 2026 г.  
**Архитектурна референция:** `architecture_specifics.md`

---

## Как да четете този документ
- Задачите са групирани по **услуга/област** и подредени по **зависимост**.
- Всяка задача отговаря на една разгръщаема единица от Container App Map в `architecture_specifics.md`.
- `Depends On` (Зависи от) препраща към ID-та на задачи в този файл.
- **Est. (days)** = оценено усилие в **човеко-дни** (един инженер, фокусирана работа). Използвайте това за изчисляване на необходимия персонал:
  `Необходими FTE ≈ (Общо дни на групата) / (Налични календарни работни дни × коефициент на фокус 0.7)`.
- **Общи стойности на групите** са посочени в долната част на всяка таблица; **общата сума** е в края на документа.
- **Definition of Done** (по-долу) се прилага за **всяка задача** в този документ. Конкретни **критерии за приемане** за задачите с най-висок риск са изброени в *Приложение с критерии за приемане* в края.

---

## Definition of Done (прилага се за всяка задача)

Задача може да бъде маркирана като завършена само когато **всичко** от следното е истина:

1. Всички критерии за приемане на задачата са изпълнени (вижте AC по задача в приложението; ако няма посочени AC, колоната *Deliverable* на задачата е имплицитният AC).
2. Кодът е слят в `main` чрез PR с поне 1 одобрение от ревюер; CI е зелен.
3. Добавени/актуализирани са unit тестове; покритието на ниво услуга е ≥ **80%** (≥ **90%** за `erp-core` и `erp-contracts`).
4. Добавени/актуализирани са интеграционни тестове, когато задачата засяга външна граница (БД, Service Bus, HTTP, AI API).
5. API контрактите в `erp-contracts` са актуализирани, ако са променени формите на заявките/отговорите; Pact contract тестовете преминават успешно.
6. Type checks преминават (`mypy --strict` за backend, `tsc --noEmit` за frontend); линтерите преминават (`ruff`, `eslint`).
7. Добавено е структурирано логване на INFO/ERROR границите с `correlation_id`; без `print()` или голи `logging.info`.
8. Документацията е актуализирана: съответен раздел в `architecture_specifics.md`, `README.md` на услугата и OpenAPI / Swagger схема.
9. Observability: новите метрики или аларми са свързани с Azure Monitor / Application Insights, ако задачата въвежда видимо за потребителя поведение или код, релевантен към мащабиране.
10. Разгърнато в **dev** среда чрез стандартния CI/CD pipeline; smoke-checked ръчно или чрез автоматизиран probe.

### Production Definition of Done (допълнителна врата за Phase 6 / prod release)

В допълнение към проектния DoD по-горе:

1. Разгърнато в **staging**; пълен Playwright E2E suite е зелен.
2. Load test (k6 / Locust) е преминат на конфигурирания праг за autoscaling.
3. Security gates са зелени: OWASP ZAP сканът е чист **и** AI red-teaming suite (QA-14) е чист.
4. Runbook е актуализиран: процедурата за rollback е верифицирана за променената услуга.
5. Sign-off от product owner е записан в одобрение на GitHub Environment.
6. Image SHA е фиксиран в `infra/image-versions/<env>/` (PROD-04).

---

## Phase 0 — Инфраструктура и DevOps настройка
> Област: **DevOps** | Repo: `erp-infrastructure`
> **IaC-first:** всеки Azure ресурс по-долу е описан като **Terraform** модул в `erp-infrastructure` и се прилага чрез `terraform apply`. **Без click-ops в Azure Portal.** Всяка задача "provision X" = "напишете Terraform модула за X + приложете го в **dev** средата". Staging и prod след това се провизират чрез повторно прилагане на същите модули с environment-specific `tfvars` (вижте INFRA-21).

| ID | Задача | Deliverable | Зависи от | Est. (дни) |
|----|------|-------------|------------|-------------|
| INFRA-00 | Bootstrap на `erp-infrastructure` Terraform repo: remote state в Azure Storage (със state locking чрез blob lease), оформление на модули (`modules/`, `envs/{dev,staging,prod}/`), pinning на provider, конвенция за именуване, политика за тагове | `erp-infrastructure/` repo skeleton; конфигуриран remote backend; `README.md` с конвенциите | — | 3 |
| INFRA-01 | Напишете Terraform модул за Azure Resource Groups + приложете към всички среди (dev / staging / prod) | `modules/resource-group/`; RG-та създадени чрез Terraform | INFRA-00 | 1 |
| INFRA-02 | Напишете Terraform модул за Azure Container Registry (ACR) + приложете в dev | `modules/acr/`; ACR инстанция с repositories | INFRA-01 | 1 |
| INFRA-03 | Напишете Terraform модул за Azure Container Apps Environment + приложете в dev | `modules/aca-environment/`; ACA Environment — dev | INFRA-02 | 2 |
| INFRA-04 | Разширете ACA-environment модула със staging / prod tfvars (само конфигурация, същият модул) | Дефиниции за staging + prod ACA env в `envs/{staging,prod}/` (приложение се случва в INFRA-21) | INFRA-03 | 2 |
| INFRA-05 | Напишете Terraform модул за Azure Database for PostgreSQL (Flexible Server) + приложете в dev | `modules/postgresql/`; Multi-AZ активирано чрез tfvars в prod | INFRA-01 | 3 |
| INFRA-06 | Напишете Terraform модул за Azure Cache for Redis (clustered) + приложете в dev | `modules/redis/`; Redis инстанция | INFRA-01 | 1 |
| INFRA-07 | Напишете Terraform модул за Azure Service Bus (опашки + topics от архитектурната спецификация — забележка: `ai.search.requests` **не** е включен; search е sync HTTP) + приложете в dev | `modules/service-bus/`; namespace с всички опашки освен search | INFRA-01 | 2 |
| INFRA-08 | Напишете Terraform модул за Azure Key Vault (един на среда) + приложете в dev | `modules/key-vault/`; Key Vault инстанция | INFRA-01 | 1 |
| INFRA-09 | Напишете Terraform модул за Azure API Management + приложете в dev (политика за JWT валидация, правила за маршрутизация, VNet интеграция, **streaming policy: response timeout вдигнат на 120 s + `X-Accel-Buffering: no` запазен end-to-end за `/api/<module>/ai/generation/stream/*` SSE маршрути**) | `modules/apim/`; APIM инстанция с политики като код | INFRA-01 | 4 |
| INFRA-10 | Напишете Terraform модул за Azure Front Door (CDN + WAF + TLS) + приложете в dev | `modules/front-door/`; routing `/api/*` към APIM и статични активи към frontend приложенията | INFRA-09 | 2 |
| INFRA-11 | Напишете Terraform модул за Azure Entra ID (app registration, scopes, роли, MFA / Conditional Access) + приложете в dev | `modules/entra-id/`; роли: `legal.reader`, `legal.editor`, `marketing.admin`, `accounting.editor` и т.н. | INFRA-01 | 3 |
| INFRA-12 | Напишете Terraform модул за Azure Blob Storage (file uploads) + приложете в dev | `modules/blob-storage/`; storage account + контейнери на модул | INFRA-01 | 1 |
| INFRA-13 | Напишете Terraform модул за Azure OpenAI Service (GPT-4o, GPT-4o-mini, text-embedding-3-large) + приложете в dev | `modules/azure-openai/`; endpoints чрез private endpoint в VNet | INFRA-01 | 2 |
| INFRA-14 | Настройте GitHub Actions: базов reusable workflow (build → push ACR → deploy ACA) | `.github/workflows/deploy-template.yml` във всеки repo | INFRA-02, INFRA-03 | 3 |
| INFRA-15 | Конфигурирайте GitHub Environments (dev / staging / prod) с правила за защита | GitHub Environments с reviewer sign-off за prod | INFRA-14 | 1 |
| INFRA-16 | Напишете Terraform модул за Azure Monitor + Log Analytics workspace + приложете в dev | `modules/monitoring/`; routing на container логовете | INFRA-03 | 2 |
| INFRA-17 | Настройте Grafana dashboards (CPU, памет, брой реплики, latency) — provisioned чрез Terraform Grafana provider | `modules/grafana-dashboards/`; dashboards чрез Azure Monitor data source | INFRA-16 | 3 |
| INFRA-18 | Конфигурирайте OpenTelemetry → Azure Application Insights (всички FastAPI услуги); App Insights ресурс чрез Terraform | Application Insights ресурс; OTEL exporter конфигурация в `erp-core` | INFRA-16 | 2 |
| INFRA-19 | Напишете Terraform модул за Azure Budget Alerts (80% и 100% разход) + приложете към всички среди | `modules/budget-alerts/`; alert правила на среда | INFRA-01 | 1 |
| INFRA-20 | Настройте Docker Compose за локална разработка (PostgreSQL, RabbitMQ, Redis) | `docker-compose.dev.yml` споделен между repo-та | — | 2 |
| INFRA-21 | Приложете съществуващия Terraform stack към **staging + prod** среди (само конфигурация, повторно прилагане с env-specific tfvars; drift check на dev) | Staging + prod напълно provisioned от IaC; `terraform plan` показва нулев drift на dev | INFRA-01–INFRA-13, INFRA-16, INFRA-19 | 3 |
| INFRA-22 | Добавете CI gates в `erp-infrastructure` repo: `terraform fmt`, `tflint`, `checkov` (security/policy), `terraform plan` на PR-и, manual-approval `terraform apply` на среда | `.github/workflows/iac.yml`; задължителни status checks на `main` | INFRA-00 | 2 |
| INFRA-23 | Документирайте IaC contribution workflow + конвенции за модули + runbook за промоция между среди | `erp-infrastructure/README.md`; `docs/runbooks/iac-promotion.md` | INFRA-21, INFRA-22 | 1 |
| INFRA-24 | Provision на **Pact Broker** като self-hosted Container App (с Postgres) — изисква се от CONT-07 contract тестовете | `modules/pact-broker/`; Pact Broker URL в Key Vault | INFRA-03, INFRA-05 | 2 |
| INFRA-25 | Конфигурирайте **GitHub Packages** auth: org-level federated identity за CI; PAT за локална разработка; документирана `pip` и `npm` install настройка; secret разпределен към всички repo-та на услугите | `docs/runbooks/github-packages.md`; `GH_PACKAGES_TOKEN` secret във всеки repo | INFRA-15 | 2 |
| INFRA-26 | Provision на **Azure Defender for Storage** (malware scanning на Blob качвания) — backend за BE-L-11 virus scan | `modules/defender-storage/`; Defender активиран на storage account за качвания | INFRA-12 | 1 |
| INFRA-29 | Напишете Django **migration ACA Job** (отделно Container App Job, не startup-а на web контейнера) свързан в deploy pipeline-a на `erp-django-core` като **pre-traffic** стъпка; `python manage.py migrate --check` се изпълнява като dry-run на всеки PR; job-ът блокира новата ревизия от получаване на трафик при провал | `modules/aca-job-migrate/`; `.github/workflows/migrate-job.yml` reusable стъпка | INFRA-03, DB-22 | 2 |
| INFRA-30 | Превключете всички FastAPI + Django ACA приложения на **multi-revision режим с traffic splitting** (canary 10% → 100%); добавете автоматизиран **post-deploy smoke probe** (`/health`, `/ready`, 1 представително API повикване) и **auto-rollback** (връщане на трафика към предишната ревизия) в `.github/workflows/deploy-template.yml`; rollback SLO ≤ 2 min | Актуализиран `deploy-template.yml`; ACA app config в Terraform задава `revisionMode = "multiple"` | INFRA-03, INFRA-14 | 3 |
| INFRA-32 | Добавете **release-please** (или conventional-commits + changelog automation) към всичките 18 repo-та за автоматични version bump-ове + changelogs на release; конфигурирайте GitHub Environment **deployment freeze windows** (напр. без петък следобед prod deploys) | `.github/workflows/release-please.yml` доставен във всеки repo; freeze политика в правилата за защита на GitHub Environment | INFRA-15 | 2 |
| INFRA-37 | Напишете Terraform модул **`modules/network/`**: VNet на среда (без припокриване на CIDR-и — dev `10.10.0.0/16`, staging `10.20.0.0/16`, prod `10.30.0.0/16`), subnets (`aca-infra` delegated `/23`, `private-endpoints` `/24`, `apim` `/27`, `pg-flex` delegated), NSG-та (deny-by-default + експлицитни allow) и **private DNS zones** за postgres / redis / keyvault / blob / servicebus / acr / openai. Приложете в dev. | `modules/network/`; VNet + subnets + NSG-та + private DNS zones в dev | INFRA-01 | 5 |
| INFRA-38 | Актуализирайте INFRA-05 / 06 / 07 / 08 / 12 / 13 модулите да задават `publicNetworkAccess = Disabled` и **да създават private endpoints + DNS A-записи** в зоните от INFRA-37. Re-apply в dev. | Всички PaaS ресурси са достъпни само през private endpoint в dev VNet | INFRA-37, INFRA-05, INFRA-06, INFRA-07, INFRA-08, INFRA-12, INFRA-13 | 3 |
| INFRA-39 | Актуализирайте INFRA-03 (ACA Environment) да инжектира `aca-infra` subnet, да задава `internalLoadBalancerEnabled = true` за staging/prod (External за dev) и да добавя Front Door → ACA Private Link конфигурация. | Актуализиран `modules/aca-environment/`; ACA env работи вътре във VNet | INFRA-37, INFRA-03 | 2 |
| INFRA-40 | Актуализирайте INFRA-09 (APIM) да работи в **internal VNet режим**; pin SKU на среда (Developer в dev, **Premium** в staging + prod). Актуализирайте сценариите за разходи в `architecture_specifics.md` съответно. | Актуализиран `modules/apim/`; APIM прикачен към `apim` subnet; per-env SKU `tfvars` | INFRA-37, INFRA-09 | 2 |
| INFRA-41 | Networking validation suite: `Test-NetConnection` скрипт + `terraform plan` policy check (Checkov/conftest), който проверява (a) нито един Azure ресурс няма `publicNetworkAccess = Enabled` извън allow-списъка (Front Door, APIM external за dev, ACA Environment public ingress), (б) всеки консумиран PaaS ресурс има private endpoint в същата VNet. | `.github/workflows/network-policy.yml`; задължителен status check на PR-и в `erp-infrastructure` | INFRA-37, INFRA-38, INFRA-22 | 2 |
| **Общо за групата** | | | | **74** |

---

## Phase 1a — Споделени пакети

### `erp-core` Python пакет
> Област: **Backend** | Repo: `erp-core`

| ID | Задача | Deliverable | Зависи от | Est. (дни) |
|----|------|-------------|------------|-------------|
| CORE-01 | Инициализирайте `erp-core` Python пакет (pyproject.toml, src layout) | `erp_core/` package skeleton, GitHub Packages publish pipeline | — | 1 |
| CORE-02 | Имплементирайте `erp_core/config.py` — Pydantic `BaseSettings` зареждащ от Azure Key Vault + env променливи | `erp_core/config.py` | CORE-01 | 2 |
| CORE-03 | Имплементирайте `erp_core/logging.py` — структурирано JSON логване с `correlation_id`, `service_name`, `user_id`; OpenTelemetry trace инжекция | `erp_core/logging.py` | CORE-01 | 3 |
| CORE-04 | Имплементирайте `erp_core/exceptions.py` — споделена йерархия от изключения (`AppError`, `NotFoundError`, `ConflictError`, `ValidationError`) мапирани към HTTP status codes | `erp_core/exceptions.py` | CORE-01 | 1 |
| CORE-05 | Имплементирайте `erp_core/middleware/` — request ID инжекция, структурирано access логване, форматиране на error отговори | `erp_core/middleware/` | CORE-03, CORE-04 | 3 |
| CORE-06 | Имплементирайте `erp_core/interfaces/` — ABC-та: `BaseService`, `BaseRepository`, `BaseEventPublisher`, `BaseAIAgent` | `erp_core/interfaces/` | CORE-01 | 2 |
| CORE-07 | Имплементирайте `erp_core/messaging/` — Azure Service Bus publisher/subscriber базови класове; RabbitMQ адаптер за локална разработка; превключваемо чрез `MESSAGING_BACKEND` env променлива | `erp_core/messaging/` | CORE-06 | 5 |
| CORE-08 | Имплементирайте `erp_core/http_client.py` — споделен `httpx` async клиент с retry политика, timeout конфигурация, `X-Correlation-ID` header пропагация | `erp_core/http_client.py` | CORE-03 | 2 |
| CORE-09 | Имплементирайте `erp_core/health.py` — стандартен `/health` и `/ready` endpoint factory | `erp_core/health.py` | CORE-01 | 1 |
| CORE-10 | Имплементирайте `erp_core/security.py` — JWT claim extraction helper-и (user ID, роли, module access) от APIM-forwarded headers | `erp_core/security.py` | CORE-01 | 2 |
| CORE-11 | Имплементирайте `erp_core/constants.py` — споделени enums: `ModuleName`, `AuditAction`, `EventType` | `erp_core/constants.py` | CORE-01 | 1 |
| CORE-12 | Напишете unit тестове за всички `erp-core` модули (≥90% покритие) | `tests/` в `erp-core` repo | CORE-02–CORE-11 | 5 |
| CORE-13 | Публикувайте `erp-core` **v1.0.0** в GitHub Packages (без Django клиент — отблокира Django + FastAPI scaffolding) | GitHub Packages private PyPI запис | CORE-12, INFRA-25 | 1 |
| CORE-14 | Имплементирайте `erp_core/clients/django_client.py` — typed async клиент за Django internal REST API (автоматично генериран от Django OpenAPI схемата с `openapi-python-client`); обвива `erp_core/http_client.py` с managed-identity auth, retries и съответствие с `BaseRepository` интерфейса | `erp_core/clients/django_client.py`; CI стъпка, която регенерира клиента когато Django OpenAPI схемата се промени | CORE-08, DB-17 | 3 |
| CORE-15 | Публикувайте `erp-core` **v1.1.0** в GitHub Packages (включва Django клиент, DLQ/idempotency, transactional outbox, PII redactor, prompt guard, SSE fan-out, Search клиент) — изисква се от всички FastAPI услуги и AI агенти | GitHub Packages release | CORE-14, CORE-16, CORE-17, CORE-18, CORE-19, CORE-20, CORE-21 | 1 |
| CORE-16 | Имплементирайте **DLQ + idempotency** в `erp_core/messaging`: Service Bus subscriber-ите автоматично преместват poison съобщения в DLQ след N retry-та; consumer-ите проследяват обработени message ID-та в Redis за справяне с at-least-once redelivery | `erp_core/messaging/dlq.py`, `erp_core/messaging/idempotency.py`; интеграционни тестове с Azure Service Bus emulator | CORE-07 | 4 |
| CORE-17 | Имплементирайте **transactional outbox pattern** в `erp_core/messaging`: producer-ите записват събития в `outbox` таблица в същата DB транзакция като техния domain write; relay процес източва outbox към Service Bus с at-least-once доставка. Изисква се от всяка BE-*-publishes-event задача. | `erp_core/messaging/outbox.py`; outbox table миграция доставена в Django (DB-XX); relay worker режим в `erp_core.messaging` | CORE-07, DB-16 | 5 |
| CORE-18 | Имплементирайте **PII redaction layer** в `erp_core/ai/pii_redactor.py` чрез **Microsoft Presidio**: маха/маскира PII (имена, имейли, телефони, IBAN-и, ЕГН/Булстат) от всеки текст преди да бъде изпратен към Azure OpenAI. Всички AI агенти и Generation/Analysis prompt-builder-и трябва да го извикват. | `erp_core/ai/pii_redactor.py`; Presidio recognizer config за EN + BG; unit тестове с примерни legal/financial PII | CORE-01 | 4 |
| CORE-19 | Имплементирайте **prompt-injection guardrails** в `erp_core/ai/prompt_guard.py`: input sanitization (премахва system-prompt overrides), валидация на изхода спрямо очакваната Pydantic схема, детекция на jailbreak паттерни, max-length и token cap-ове | `erp_core/ai/prompt_guard.py`; jailbreak pattern библиотека; unit тестове | CORE-01 | 3 |
| CORE-20 | Имплементирайте **SSE fan-out helper** в `erp_core/streaming/sse.py`: канали по `correlation_id` подкрепени от **Redis Pub/Sub**, така че всяка реплика на module API да може да доставя chunks към отворената SSE връзка независимо коя реплика се е абонирала за `ai.results` първа. Включва **15 s heartbeat** (`: heartbeat\n\n` SSE коментар за поддържане на proxy-та и `EventSource` живи), експлицитни **`event: status`** frames носещи `{"phase": "warming_up" | "generating" | "completed" | "failed", "reason"?: str}` при всеки преход на workflow състоянието, детекция на client-disconnect и back-pressure. Използва се от всички module API-та за streamed AI output. | `erp_core/streaming/sse.py`; конвенция за наименуване на Redis Pub/Sub канали; интеграционен тест с 2 реплики + 1 producer покриващ: heartbeat оцелява 60 s тишина; `phase` преходите пристигат подредени | CORE-01, INFRA-06 | 3 |
| CORE-21 | Имплементирайте typed **`SearchClient`** в `erp_core/clients/search_client.py` — sync HTTP wrapper около `POST /search` endpoint-а на AI Search Agent. Обвива `erp_core.http_client` с retry, timeout, **circuit breaker**, **per-module token-bucket rate limit** (по подразбиране 20 req/s, конфигурируемо на среда) и `correlation_id` пропагация; излага `search(query, filters) -> SearchResult` typed спрямо договора в `erp-contracts`. `SearchResult` носи `degraded: bool` флаг зададен на `true`, когато breaker-ът е отворен или rate limit-ът е надхвърлен, така че извикващите могат да преминат grаciously към fallback (вижте BE-L-14). | `erp_core/clients/search_client.py`; интеграционен тест срещу stubbed Search Agent покриващ: breaker-open → `degraded=true`; rate-limited → `degraded=true`; per-module изолация (Marketing burst не засяга Legal) | CORE-08 | 2 |
| **Общо за групата** | | | | **54** |

### `erp-contracts` Pydantic пакет
> Област: **Backend** | Repo: `erp-contracts`

| ID | Задача | Deliverable | Зависи от | Est. (дни) |
|----|------|-------------|------------|-------------|
| CONT-01 | Инициализирайте структура на `erp-contracts` Python пакет | `erp_contracts/` package skeleton | CORE-01 | 1 |
| CONT-02 | Дефинирайте REST контракти — Legal модул DTO-та (Pydantic v2 модели за заявка/отговор) | `erp_contracts/legal/` | CONT-01 | 3 |
| CONT-03 | Дефинирайте REST контракти — Marketing модул DTO-та | `erp_contracts/marketing/` | CONT-01 | 3 |
| CONT-04 | Дефинирайте REST контракти — Accounting модул DTO-та | `erp_contracts/accounting/` | CONT-01 | 4 |
| CONT-05 | Дефинирайте AI agent I/O схеми (`IngestionRequest`, `AnalysisResult`, `GenerationRequest`, `ClassificationRequest`, `SearchRequest`, `SearchResult`) | `erp_contracts/ai/` | CONT-01 | 3 |
| CONT-06 | Дефинирайте async event схеми за Service Bus съобщения | `erp_contracts/events/` | CONT-01 | 2 |
| CONT-07 | Настройте Pact consumer-driven contract тестове в CI | `.github/workflows/pact.yml`; Pact Broker конфигурация | CONT-02–CONT-06, INFRA-24 | 3 |
| CONT-08 | Публикувайте `erp-contracts` v1.0.0 в GitHub Packages | GitHub Packages private PyPI запис | CONT-07, INFRA-25 | 1 |
| CONT-09 | Генерирайте **TypeScript типове** от `erp-contracts` Pydantic моделите чрез `pydantic2ts` / `datamodel-codegen`; публикувайте като npm пакет `@erp/contracts-ts` в GitHub Packages при всяко contracts release. CI се проваля, ако типовете са остарели. | `scripts/generate_ts.py`; `@erp/contracts-ts` npm пакет; CI job в `erp-contracts` | CONT-08 | 3 |
| **Общо за групата** | | | | **23** |

---

## Phase 1b — Слой за данни (Django)
> Област: **Database + Backend** | Repo: `erp-django-core`

| ID | Задача | Deliverable | Зависи от | Est. (дни) |
|----|------|-------------|------------|-------------|
| DB-01 | Настройте PostgreSQL 16 + pgvector extension (Docker Compose локална разработка) | `docker-compose.dev.yml` DB сервиз; `CREATE EXTENSION vector` в init скрипт | INFRA-20 | 1 |
| DB-02 | Проектирайте и създайте `core` схема (Organization, Role, User, Permission таблици) | Django модели + миграция: `core_organization`, `core_role`, `core_user_role`, `core_permission` | DB-01 | 3 |
| DB-03 | Проектирайте и създайте `legal` схема (LegalDossier, LegalCase, LegalDeadline, LawAct, Attachment, DocumentVersion) | Django модели + миграция за `legal` схемата | DB-02 | 4 |
| DB-04 | Проектирайте и създайте `marketing` схема (Campaign, Lead, Ticket, Operator, ContactPerson) | Django модели + миграция за `marketing` схемата | DB-02 | 3 |
| DB-05 | Проектирайте и създайте `accounting` схема (Account, JournalEntry, JournalEntryLine, Period, TaxDeclaration) | Django модели + миграция за `accounting` схемата | DB-02 | 4 |
| DB-06 | Добавете `AuditLog` таблица с JSON diff (споделена, под `core` схема) | `core_audit_log` модел + миграция | DB-02 | 2 |
| DB-07 | Добавете `Attachment` + `DocumentVersion` таблици (споделени, под `core` схема) | `core_attachment`, `core_document_version` модели + миграция | DB-02 | 2 |
| DB-08 | Активирайте pgvector в PostgreSQL; добавете embedding колони и vector индекси на релевантните таблици | `CREATE INDEX ivfflat` на embedding колони | DB-01 | 2 |
| DB-09 | Създайте индекси за всички foreign keys във всички схеми | Индекси на всички `*_id` колони | DB-02–DB-05 | 1 |
| DB-10 | Инициализирайте `erp-django-core` Django проект | `erp-django-core/` project skeleton; Gunicorn config; `pyproject.toml` с `erp-core` зависимост | CORE-13 | 2 |
| DB-11 | Конфигурирайте Django per-schema database routing (отделна схема на модул в една PostgreSQL инстанция) | `db_router.py`; Django настройки за `legal`, `marketing`, `accounting`, `core` схеми | DB-10, DB-02 | 3 |
| DB-12 | Имплементирайте `Organization`, `Department`, `Team` модели в `core` приложение | `core/models.py` | DB-10 | 2 |
| DB-13 | Имплементирайте Django admin панел (вътрешен management UI) | `admin.py` регистрации за всички модели | DB-10 | 3 |
| DB-14 | Имплементирайте RBAC модели съгласувани с Azure Entra ID роли (четене на роли от JWT claims, без локално парола управление) | `rbac/models.py`, `rbac/service.py` — само мапиране роля-към-разрешение | DB-10, INFRA-11 | 4 |
| DB-15 | Имплементирайте audit log middleware за Django admin действия | `middleware/audit.py` | DB-06 | 2 |
| DB-16 | Имплементирайте Django **internal REST API** консумиран от всички FastAPI услуги (единственият път до DB — FastAPI никога не се свързва директно с PostgreSQL). Endpoint-ите следват шаблона `/internal/v1/<schema>/<resource>` и връщат Pydantic-съвместим JSON. Всеки модул (legal, marketing, accounting, core) получава свой DRF viewset модул. | `internal_api/` Django app; viewsets за всички module модели; URL routing под `/internal/v1/`; OpenAPI схема публикувана на `/internal/v1/openapi.json` | DB-10, DB-02, DB-03, DB-04, DB-05 | 8 |
| DB-17 | Конфигурирайте DRF + `drf-spectacular` за **internal API** (генериране на OpenAPI 3 схема; схемата е източникът на истината за `erp-core` Django клиента) | DRF конфигурация; `drf-spectacular` настройки; OpenAPI схема валидирана в CI; артефакт схема публикуван в GitHub Packages при Django release | DB-16 | 3 |
| DB-18 | Команда за seed на permissions (всички модули) | `management/commands/seed_permissions.py` | DB-14 | 1 |
| DB-19 | Конфигурирайте Azure Database for PostgreSQL връзка (SSL, connection pooling чрез pgBouncer) | Django `DATABASES` настройки с Azure credentials от Key Vault | DB-10, INFRA-05, INFRA-08 | 2 |
| DB-20 | Конфигурирайте автоматизирана валидация на backup + point-in-time restore тест + geo-redundant backup | Restore test runbook; `docs/runbooks/restore.md` | DB-19 | 2 |
| DB-21 | Напишете Dockerfile за `erp-django-core` (само вътрешен, Gunicorn) | `Dockerfile` | DB-10 | 1 |
| DB-22 | Настройте GitHub Actions CI/CD pipeline за `erp-django-core` (публикува OpenAPI схема артефакт при release) | `.github/workflows/ci-cd.yml` (lint → test → build → push ACR → deploy ACA internal → publish OpenAPI schema) | DB-21, INFRA-14, CORE-13 | 2 |
| DB-23 | Имплементирайте GDPR съответствие: PII tagging в моделите, soft-delete + anonymization helpers, region/data-residency конфигурация | `core/gdpr.py`; PII metadata на model полета; anonymize management команда | DB-02–DB-05 | 3 |
| DB-24 | Имплементирайте **service-to-service автентикация** за Django internal API (Azure Managed Identity tokens валидирани от Django middleware; само ACA Container Apps в същата VNet с роля `internal-api-caller` могат да го извикат) | `internal_api/auth.py` middleware; Entra ID app registration за internal API ресурс; списък с разрешени извикващи на среда | DB-16, INFRA-08, INFRA-11 | 3 |
| DB-25 | Имплементирайте **schema drift / contract тест** в CI: всеки PR изпълнява FastAPI integration suite срещу прясно мигриран Django контейнер за откриване на breaking schema промени преди merge | `.github/workflows/schema-contract.yml`; споделен docker-compose за тестовия job | DB-16, DB-22 | 2 |
| DB-26 | Добавете `core.ai_workflow_state` таблица за **LangGraph Postgres checkpointer** (използвана от AI-06): колони за `correlation_id`, `thread_id`, `checkpoint_data` (JSONB), `created_at`, `updated_at`, `status`; индекс на `(correlation_id, updated_at)`; TTL job изчиства terminal състояния по-стари от 30 дни, пази in-progress безкрайно | Django модел + миграция в `core` схема; Alembic-еквивалентен rollback path тестван | DB-02 | 1 |
| **Общо за групата** | | | | **63** |

---

## Phase 2 — Module Backend услуги (FastAPI)

### Legal FastAPI услуга
> Област: **Backend** | Repo: `erp-legal-api`

| ID | Задача | Deliverable | Зависи от | Est. (дни) |
|----|------|-------------|------------|-------------|
| BE-L-01 | Инициализирайте `erp-legal-api` FastAPI проект (достъпът до данни е изключително чрез `erp_core.clients.django_client` — **без директна DB връзка**) | `services/legal-api/` skeleton; `pyproject.toml` с `erp-core>=1.1.0` + `erp-contracts`; lint правило блокиращо `psycopg`/`sqlalchemy` import-и | CORE-15, CONT-08 | 1 |
| BE-L-02 | Имплементирайте Legal Dossier CRUD endpoints (делегира persistence към Django internal API) | `routes/dossiers.py` | BE-L-01, DB-16 | 3 |
| BE-L-03 | Имплементирайте Legal Case management endpoints (чрез Django internal API) | `routes/cases.py` | BE-L-01, DB-16 | 3 |
| BE-L-04 | Имплементирайте Legal Deadline tracking endpoints (чрез Django internal API) | `routes/deadlines.py` | BE-L-01, DB-16 | 2 |
| BE-L-05 | Имплементирайте Law Acts / Knowledge Base endpoints (чрез Django internal API) | `routes/law_acts.py` | BE-L-01, DB-16 | 2 |
| BE-L-06 | Имплементирайте document upload → trigger `ai.ingestion.requests` Service Bus съобщение (само след преминаване на virus scan + валидация) | `routes/documents.py`; Service Bus publisher | BE-L-01, CORE-07, INFRA-07, BE-L-11, BE-L-12 | 3 |
| BE-L-07 | Имплементирайте AI Search интеграция чрез **sync HTTP** повикване към Search Agent (`erp_core.clients.SearchClient`) — без Service Bus | `routes/search.py` | BE-L-01, CORE-15 | 3 |
| BE-L-08 | Имплементирайте audit логване за всички write операции (чрез `erp-core` middleware) | Audit събития в `core_audit_log` | BE-L-01, DB-06 | 2 |
| BE-L-09 | Напишете unit + integration тестове (pytest + httpx) | `tests/` (≥80% покритие) | BE-L-01–BE-L-08 | 5 |
| BE-L-10 | Напишете Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | BE-L-01, INFRA-14 | 2 |
| BE-L-11 | Имплементирайте **virus / malware scan** на всеки качен документ чрез Azure Defender for Storage (блокиращо — файлът е поставен в карантина и отхвърлен ако Defender го флагне) преди публикуване в `ai.ingestion.requests` | `services/upload_scanner.py`; Defender event-grid subscription | BE-L-01, INFRA-26 | 2 |
| BE-L-12 | Имплементирайте **upload валидация**: лимит за максимален размер на файл (конфигурируем на среда), magic-byte content-type валидация (отхвърля изпълними файлове прикрити като PDF/DOCX), sanitization на име на файл | `services/upload_validator.py` | BE-L-01 | 1 |
| BE-L-13 | Имплементирайте **AI streaming SSE endpoint** `GET /api/legal/ai/generation/stream/{correlation_id}` (Content-Type `text/event-stream`): публикува generation заявката в `ai.generation.requests` чрез outbox-а, **незабавно** излъчва `event: status {"phase":"warming_up"}` така че клиентът никога не вижда празен екран, после държи SSE връзката отворена и изпраща chunks доставени от Redis Pub/Sub fan-out на CORE-20 докато Generation Agent ги излъчва (преминавайки в `phase=generating` на първия chunk и `phase=completed` на терминалния). **Hard ceiling 180 s**: ако няма terminal frame, излъчете `event: status {"phase":"failed","reason":"timeout"}` и затворете. Заменя предишно планирания in-agent SSE endpoint. | `routes/ai_stream.py`; `consumers/ai_results_subscriber.py` (Service Bus → Redis Pub/Sub bridge) | BE-L-01, CORE-15, CORE-20, INFRA-07 | 2 |
| BE-L-14 | Имплементирайте **search graceful-degradation fallback** (референтна имплементация; Marketing/Accounting наследяват шаблона в техните съществуващи search routes): когато `SearchClient` връща `degraded=true` (breaker отворен или rate-limited), премини към Django-internal-API keyword search чрез `ILIKE` + Postgres `tsvector` върху релевантните Legal таблици; отговорът носи `degraded: true` така че frontend banner-а (FE-L-05) може да покаже "Показват се keyword резултати — semantic search временно недостъпен." | `services/search_fallback.py`; нов `/internal/v1/legal/search` endpoint в Django (DB-16 amend); contract тест, който твърди, че `SearchClient` outage никога не връща 5xx на потребителя | BE-L-07, CORE-21, DB-16 | 3 |
| **Общо за групата** | | | | **34** |

### Marketing FastAPI услуга
> Област: **Backend** | Repo: `erp-marketing-api`

| ID | Задача | Deliverable | Зависи от | Est. (дни) |
|----|------|-------------|------------|-------------|
| BE-M-01 | Инициализирайте `erp-marketing-api` FastAPI проект (достъпът до данни е изключително чрез `erp_core.clients.django_client` — **без директна DB връзка**) | `services/marketing-api/` skeleton; `pyproject.toml` с `erp-core>=1.1.0` | CORE-15, CONT-08 | 1 |
| BE-M-02 | Имплементирайте Campaign management endpoints (CRUD, чрез Django internal API) | `routes/campaigns.py` | BE-M-01, DB-16 | 3 |
| BE-M-03 | Имплементирайте Lead management + lead-to-client conversion endpoints (чрез Django internal API) | `routes/leads.py` | BE-M-01, DB-16 | 3 |
| BE-M-04 | Имплементирайте Ticket / Support request endpoints (чрез Django internal API) | `routes/tickets.py` | BE-M-01, DB-16 | 3 |
| BE-M-05 | Имплементирайте Operator assignment + routing логика (чрез Django internal API) | `services/operator_service.py` | BE-M-01, DB-16 | 3 |
| BE-M-06 | Имплементирайте campaign performance анализ → trigger `ai.analysis.requests` | `routes/analytics.py`; Service Bus publisher | BE-M-01, CORE-07 | 3 |
| BE-M-07 | Имплементирайте ETL data import (bulk lead/campaign import) | `services/etl_service.py` | BE-M-01 | 4 |
| BE-M-08 | Напишете unit + integration тестове | `tests/` (≥80% покритие) | BE-M-01–BE-M-07 | 5 |
| BE-M-09 | Напишете Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | BE-M-01, INFRA-14 | 2 |
| BE-M-10 | Имплементирайте **AI streaming SSE endpoint** `GET /api/marketing/ai/generation/stream/{correlation_id}` — същият шаблон като BE-L-13 (незабавен `phase=warming_up`, мост на `ai.results` към SSE чрез CORE-20, hard 180 s ceiling с `phase=failed` при timeout) | `routes/ai_stream.py`; `consumers/ai_results_subscriber.py` | BE-M-01, CORE-15, CORE-20, INFRA-07 | 2 |
| **Общо за групата** | | | | **29** |

### Accounting FastAPI услуга
> Област: **Backend** | Repo: `erp-accounting-api`

| ID | Задача | Deliverable | Зависи от | Est. (дни) |
|----|------|-------------|------------|-------------|
| BE-A-01 | Инициализирайте `erp-accounting-api` FastAPI проект (достъпът до данни е изключително чрез `erp_core.clients.django_client` — **без директна DB връзка**) | `services/accounting-api/` skeleton; `pyproject.toml` с `erp-core>=1.1.0` | CORE-15, CONT-08 | 1 |
| BE-A-02 | Създайте Сметкоплан (НСС стандарт) — persisted чрез Django internal API | `routes/accounts.py`; `data/chart_of_accounts.json`; seed команда | BE-A-01, DB-16 | 3 |
| BE-A-03 | Имплементирайте Journal Entry + Journal Entry Lines endpoints (чрез Django internal API) | `routes/journal_entries.py`; `models/journal_entry.py` | BE-A-01, DB-16 | 4 |
| BE-A-04 | Имплементирайте double-entry validation услуга | `services/validation_service.py` (debit == credit проверка) | BE-A-03 | 2 |
| BE-A-05 | Имплементирайте Accounting Period + Period Closure (чрез Django internal API) | `routes/periods.py`; `models/period.py` | BE-A-01, DB-16 | 3 |
| BE-A-06 | Имплементирайте генериране на финансови отчети (Balance Sheet, P&L, Cash Flow) | `services/financial_statement_service.py` | BE-A-03, BE-A-05 | 5 |
| BE-A-07 | Имплементирайте генериране на ДДС + декларация за корпоративен данък | `services/tax_service.py` | BE-A-03 | 4 |
| BE-A-08 | Имплементирайте audit план + findings endpoints (чрез Django internal API) | `routes/audit.py` | BE-A-01, DB-16 | 3 |
| BE-A-09 | Имплементирайте контролно досие за инспекция от НАП | `services/control_dossier_service.py` | BE-A-08 | 3 |
| BE-A-10 | Имплементирайте risk alert engine | `services/risk_alert_service.py` | BE-A-06 | 3 |
| BE-A-11 | Тригер на AI генериране за финансови наративи → `ai.generation.requests` | Service Bus publisher в `services/narrative_service.py` | BE-A-01, CORE-07 | 3 |
| BE-A-12 | Напишете unit + integration тестове | `tests/` (≥80% покритие) | BE-A-01–BE-A-11 | 5 |
| BE-A-13 | Напишете Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | BE-A-01, INFRA-14 | 2 |
| BE-A-14 | Имплементирайте **AI streaming SSE endpoint** `GET /api/accounting/ai/generation/stream/{correlation_id}` — същият шаблон като BE-L-13 (незабавен `phase=warming_up`, мост на `ai.results` към SSE чрез CORE-20, hard 180 s ceiling с `phase=failed` при timeout) | `routes/ai_stream.py`; `consumers/ai_results_subscriber.py` | BE-A-01, CORE-15, CORE-20, INFRA-07 | 2 |
| **Общо за групата** | | | | **43** |

### Celery Background Worker
> Област: **Backend** | Repo: `erp-celery-worker`

| ID | Задача | Deliverable | Зависи от | Est. (дни) |
|----|------|-------------|------------|-------------|
| CEL-01 | Инициализирайте Celery worker услуга | `services/celery-worker/` с `pyproject.toml` | CORE-13 | 1 |
| CEL-02 | Имплементирайте scheduled background задачи (генериране на отчети, почистване на данни, нотификации) | `tasks/` директория | CEL-01 | 3 |
| CEL-03 | Конфигурирайте Azure Service Bus като Celery broker (prod) / RabbitMQ (локална разработка) | `celeryconfig.py` с `MESSAGING_BACKEND` switch | CEL-01, INFRA-07 | 2 |
| CEL-04 | Напишете Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | CEL-01, INFRA-14 | 1 |
| **Общо за групата** | | | | **7** |

---

## Phase 3 — AI услуги

### AI Orchestrator (LangGraph Supervisor)
> Област: **Backend** | Repo: `erp-ai-orchestrator`

| ID | Задача | Deliverable | Зависи от | Est. (дни) |
|----|------|-------------|------------|-------------|
| AI-01 | Инициализирайте AI Orchestrator услуга (LangGraph Supervisor) | `services/ai-orchestrator/` skeleton; LangGraph зависимост | CORE-13, CONT-08 | 2 |
| AI-02 | Имплементирайте логика за task routing — маршрутизира входящи `ai.*` заявки към правилния агент чрез Service Bus (таблица с правила за маршрутизация + само dispatch; checkpointing → AI-06, failure semantics → AI-07) | `orchestrator/supervisor.py`; routing правила на `EventType` | AI-01, INFRA-07 | 2 |
| AI-03 | Имплементирайте result агрегация — консумира `ai.results` topic, връща към заявяващата FastAPI услуга | `orchestrator/aggregator.py` | AI-02 | 3 |
| AI-04 | Настройте LangSmith tracing интеграция за orchestrator-а | LangSmith конфигурация в `erp_core/logging.py` (AI trace разширения) | AI-01 | 1 |
| AI-05 | Напишете Dockerfile + GitHub Actions CI/CD pipeline (scale-to-zero в ACA) | `Dockerfile`, `.github/workflows/ci-cd.yml` | AI-01, INFRA-14 | 2 |
| AI-06 | Свържете **LangGraph Postgres checkpointer** в Orchestrator-а (използвайки таблицата `core.ai_workflow_state` от DB-26), така че multi-step workflows да оцеляват scale-to-zero. Checkpoint-ът на всяка стъпка commit-ва в същата DB транзакция като нейния outbox publish (преизползва CORE-17). Интеграционен тест: пауза на workflow по средата, scale Orchestrator до 0, requeue → workflow-ът продължава от последно commit-натата стъпка. | `orchestrator/checkpointer.py`; интеграционен тест в `tests/test_resume.py` | AI-02, DB-26, CORE-17 | 3 |
| AI-07 | Дефинирайте **workflow failure семантика**: terminal failure публикува `AIWorkflowFailed` събитие в `ai.results` с `correlation_id` / `failed_step` / `error_class`; module API-тата (BE-L-13 / BE-M-10 / BE-A-14) го показват чрез SSE; добавете `AIWorkflowFailed` Pydantic модел в `erp-contracts` (CONT-05/06). **Без автоматична компенсация в MVP** — документирано като ръчен operator runbook в PROD-07. | `orchestrator/failure.py`; `erp_contracts/ai/AIWorkflowFailed`; `docs/runbooks/ai-workflow-failure.md` | AI-02, CONT-05, CONT-06, BE-L-13, BE-M-10, BE-A-14 | 2 |
| AI-08 | Документирайте **Orchestrator scope** в `architecture_specifics.md` §8.1: матрица за решения (директни module→agent flows: Ingestion, Search vs orchestrated multi-agent flows: Analysis pipelines, end-to-end document workflows) + canonical sequence diagram (upload → ingest → classify → summary); ADR `0007-ai-orchestrator-scope.md` | Актуализирана §8.1; `/docs/adr/0007-ai-orchestrator-scope.md` | AI-02 | 1 |
| **Общо за групата** | | | | **16** |

### AI Ingestion Agent
> Област: **Backend** | Repo: `erp-ai-ingestion`

| ID | Задача | Deliverable | Зависи от | Est. (дни) |
|----|------|-------------|------------|-------------|
| AI-ING-01 | Инициализирайте Ingestion Agent услуга | `services/ai-ingestion/` skeleton | CORE-13, CONT-08 | 1 |
| AI-ING-02 | Имплементирайте file parsing: PDF, DOCX, XLSX извличане на текст | `parsers/` (PyMuPDF / python-docx / openpyxl) | AI-ING-01 | 5 |
| AI-ING-03 | Имплементирайте OCR за изображения чрез GPT-4o Vision | `ocr/gpt4o_vision.py` | AI-ING-01, INFRA-13 | 3 |
| AI-ING-04 | Имплементирайте text chunking + генериране на embeddings (`text-embedding-3-large` чрез Azure OpenAI) | `embeddings/chunker.py`, `embeddings/embedder.py` | AI-ING-01, INFRA-13 | 4 |
| AI-ING-05 | Съхранете embeddings в pgvector (MVP) | `storage/pgvector_store.py` | AI-ING-04, DB-08 | 2 |
| AI-ING-06 | Абонирайте се за `ai.ingestion.requests` Service Bus опашка | `consumers/ingestion_consumer.py` | AI-ING-01, CORE-07 | 2 |
| AI-ING-07 | Напишете Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | AI-ING-01, INFRA-14 | 1 |
| **Общо за групата** | | | | **18** |

### AI Analysis Agent
> Област: **Backend** | Repo: `erp-ai-analysis`

| ID | Задача | Deliverable | Зависи от | Est. (дни) |
|----|------|-------------|------------|-------------|
| AI-ANA-01 | Инициализирайте Analysis Agent услуга | `services/ai-analysis/` skeleton | CORE-13, CONT-08 | 1 |
| AI-ANA-02 | Имплементирайте clause risk scoring (правни документи) чрез GPT-4o | `analyzers/clause_risk.py` | AI-ANA-01, INFRA-13 | 4 |
| AI-ANA-03 | Имплементирайте откриване на финансови аномалии чрез GPT-4o | `analyzers/financial_anomaly.py` | AI-ANA-01, INFRA-13 | 4 |
| AI-ANA-04 | Имплементирайте campaign performance insights чрез GPT-4o | `analyzers/campaign_insights.py` | AI-ANA-01, INFRA-13 | 3 |
| AI-ANA-05 | Абонирайте се за `ai.analysis.requests` Service Bus опашка; публикувайте в `ai.results` | `consumers/analysis_consumer.py` | AI-ANA-01, CORE-07 | 2 |
| AI-ANA-06 | Напишете Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | AI-ANA-01, INFRA-14 | 1 |
| **Общо за групата** | | | | **15** |

### AI Generation Agent
> Област: **Backend** | Repo: `erp-ai-generation`

| ID | Задача | Deliverable | Зависи от | Est. (дни) |
|----|------|-------------|------------|-------------|
| AI-GEN-01 | Инициализирайте Generation Agent услуга | `services/ai-generation/` skeleton | CORE-13, CONT-08 | 1 |
| AI-GEN-02 | Имплементирайте генериране на резюме на правни документи (GPT-4o) | `generators/legal_summary.py` | AI-GEN-01, INFRA-13 | 3 |
| AI-GEN-03 | Имплементирайте генериране на маркетингово съдържание (GPT-4o) | `generators/marketing_copy.py` | AI-GEN-01, INFRA-13 | 3 |
| AI-GEN-04 | Имплементирайте генериране на финансови наративи (GPT-4o) | `generators/financial_narrative.py` | AI-GEN-01, INFRA-13 | 3 |
| AI-GEN-05 | Имплементирайте генериране на email чернови (GPT-4o) | `generators/email_draft.py` | AI-GEN-01, INFRA-13 | 3 |
| AI-GEN-06 | Създайте и управлявайте prompt темплейти (на персона / use case) | `data/prompts/` (≥10 prompt темплейта покриващи всички модули) | AI-GEN-01 | 4 |
| AI-GEN-07 | Абонирайте се за `ai.generation.requests`; публикувайте **streamed chunks** в `ai.results` topic с `correlation_id` и монотонно нарастващ `sequence_number` (консумиран от SSE fan-out на module API-тата чрез CORE-20). Първият chunk задава `phase=generating`; терминалният chunk задава `phase=completed` (или `failed` с `reason`). | `consumers/generation_consumer.py`; chunked `AIResultChunk` Pydantic схема в `erp-contracts` | AI-GEN-01, CORE-07 | 3 |
| AI-GEN-08 | Напишете Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | AI-GEN-01, INFRA-14 | 1 |
| AI-GEN-09 | Конфигурирайте ACA scale правила: **prod `min replicas = 1`** за Generation Agent (цена ≈ $0.50/ден, елиминира cold-start на streaming пътя); dev/staging запазват `min = 0`. Добавете synthetic monitor, който проверява **first-content-chunk SLO**: p95 ≤ **20 s** на cold start (dev/staging), p95 ≤ **5 s** при warm (prod). Свързано с Application Insights. | Актуализиран `terraform/aca/ai-generation/`; `tests/synthetic/first_chunk_slo.py`; alert правило | AI-GEN-07, INFRA-16 | 2 |
| AI-GEN-10 | Frontend SSE warm-up UX (референтна имплементация в `erp-legal-frontend`, отразена в marketing/accounting чрез FE-M-* / FE-AC-* streaming страници): консумирайте `event: status` frames от CORE-20 и рендерирайте "Подготвяме отговора ви…" докато `phase=warming_up`; стриймирайте съдържание докато `phase=generating`; покажете `phase=failed` (с `reason`) в глобалния Notification Center | `src/components/AIStreamPanel.tsx` в `erp-legal-frontend`; копиран/импортиран от FE-M-06 + FE-AC-05 streaming страници | CORE-20, FE-L-05, FE-SH-09 | 2 |
| **Общо за групата** | | | | **25** |

### AI Classification Agent
> Област: **Backend** | Repo: `erp-ai-classification`

| ID | Задача | Deliverable | Зависи от | Est. (дни) |
|----|------|-------------|------------|-------------|
| AI-CLS-01 | Инициализирайте Classification Agent услуга | `services/ai-classification/` skeleton | CORE-13, CONT-08 | 1 |
| AI-CLS-02 | Имплементирайте document категоризация чрез GPT-4o-mini | `classifiers/document_classifier.py` | AI-CLS-01, INFRA-13 | 3 |
| AI-CLS-03 | Имплементирайте task routing / intent detection (GPT-4o-mini) | `classifiers/intent_detector.py` | AI-CLS-01, INFRA-13 | 3 |
| AI-CLS-04 | Абонирайте се за `ai.classification.requests`; публикувайте в `ai.results` | `consumers/classification_consumer.py` | AI-CLS-01, CORE-07 | 2 |
| AI-CLS-05 | Напишете Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | AI-CLS-01, INFRA-14 | 1 |
| **Общо за групата** | | | | **10** |

### AI Search Agent
> Област: **Backend** | Repo: `erp-ai-search`

| ID | Задача | Deliverable | Зависи от | Est. (дни) |
|----|------|-------------|------------|-------------|
| AI-SRC-01 | Инициализирайте Search Agent услуга | `services/ai-search/` skeleton | CORE-13, CONT-08 | 1 |
| AI-SRC-02 | Имплементирайте vector similarity search (pgvector MVP) | `search/pgvector_search.py` | AI-SRC-01, DB-08 | 3 |
| AI-SRC-03 | Имплементирайте hybrid search (vector + keyword) | `search/hybrid_search.py` | AI-SRC-02 | 4 |
| AI-SRC-04 | Имплементирайте подреден, цитиран result response формат | `search/result_formatter.py` | AI-SRC-02 | 2 |
| AI-SRC-05 | Изложете `POST /search` HTTP endpoint (FastAPI + Uvicorn) с `erp-core` middleware, структурирано логване и стандартни `/health` + `/ready` endpoints. **Sync request/response** — search е интерактивен и латентно-чувствителен; Service Bus умишлено не се използва за този агент. | `routes/search.py`; `main.py` (FastAPI app factory) | AI-SRC-01, CORE-15 | 2 |
| AI-SRC-06 | Имплементирайте token usage tracking + budget enforcement (Redis-backed) | `services/token_service.py` | AI-SRC-01, INFRA-06 | 3 |
| AI-SRC-07 | Напишете Dockerfile + GitHub Actions CI/CD pipeline | `Dockerfile`, `.github/workflows/ci-cd.yml` | AI-SRC-01, INFRA-14 | 1 |
| AI-SRC-08 | Конфигурирайте prod **`min replicas = 2`** за Search Agent (dev/staging остават на `min=1`); проверете че ACA разпределя репликите между availability zones (четене на `replica.zone` от container metadata; assert ≥ 2 различни зони в prod). Добавете **chaos test**, който убива 1 реплика по време на стабилен search load и проверява нула видими за потребителя 5xx. | Актуализиран `terraform/aca/ai-search/`; `tests/chaos/test_replica_kill.py`; AZ-spread assertion в CI | AI-SRC-05, INFRA-03 | 2 |
| AI-SRC-09 | Напишете ADR **`/docs/adr/0008-search-agent-sharding-deferral.md`** документиращ умишленото решение да се запази един Search Agent за всичките 3 модула в MVP (vs per-module shards); тригери за преразглеждане: prod поддържа > 10 search/s/модул **или** индекс на който и да е модул надхвърля 5 GB | `/docs/adr/0008-search-agent-sharding-deferral.md` | AI-SRC-08 | 1 |
| **Общо за групата** | | | | **19** |

---

## Phase 4 — Frontend

### Дизайн (Wireframes + Figma) — трябва да се завърши преди всякаква FE имплементация
> Област: **Frontend / Design**

| ID | Задача | Deliverable | Зависи от | Est. (дни) |
|----|------|-------------|------------|-------------|
| FE-DSGN-01 | Wireframes + hi-fi Figma дизайни за всички екрани на модулите (Legal, Marketing, Accounting) — покрива всяка страница изброена в FE-L-*, FE-M-*, FE-AC-* | Figma файл(ове) ревюирани и одобрени от product + module owners | — | 8 |
| FE-DSGN-02 | Дефинирайте **design tokens** в Figma (цветове, отстояния, типография, elevation) и експортирайте в JSON за консумиране от `FE-SH-02` | `design-tokens.json`; Figma Tokens plugin export | FE-DSGN-01 | 2 |
| **Общо за групата** | | | | **10** |

### Shell Frontend (Micro-Frontend Host)
> Област: **Frontend** | Repo: `erp-shell-frontend`

| ID | Задача | Deliverable | Зависи от | Est. (дни) |
|----|------|-------------|------------|-------------|
| FE-SH-01 | Инициализирайте Shell App (React.js + TypeScript + Webpack Module Federation) | `frontends/erp-shell/` с Module Federation host config (`singleton: true`) | — | 2 |
| FE-SH-02 | Настройте споделена design система + component library (споделена чрез Module Federation) — изградена от `FE-DSGN-02` design tokens | `src/design-system/` (бутони, форми, типография, цветове) | FE-SH-01, FE-DSGN-02 | 5 |
| FE-SH-03 | Имплементирайте Azure Entra ID login flow (redirect към Entra ID; получаване на JWT; съхранение в памет — не localStorage) | `src/auth/authService.ts`; MSAL.js интеграция | FE-SH-01, INFRA-11 | 3 |
| FE-SH-04 | Имплементирайте JWT token споделяне към module remotes чрез Module Federation shared state | `src/auth/store.ts`; shared auth context | FE-SH-03 | 2 |
| FE-SH-05 | Имплементирайте глобална навигация + sidebar (module-aware, role-based видимост) | `src/components/Navigation.tsx` | FE-SH-02 | 3 |
| FE-SH-06 | Настройте routing за remote module lazy loading + module discovery | `src/router/index.tsx`; `src/modules/registry.ts` | FE-SH-01 | 3 |
| FE-SH-07 | Създайте споделен DataTable компонент (TanStack Table) | `src/components/DataTable.tsx` | FE-SH-02 | 3 |
| FE-SH-08 | Създайте споделен Form builder (React Hook Form + Zod) | `src/components/FormBuilder.tsx` | FE-SH-02 | 3 |
| FE-SH-09 | Създайте глобален Notification Center | `src/components/NotificationCenter.tsx` | FE-SH-02 | 2 |
| FE-SH-10 | Имплементирайте logout + session expiry handling | `src/auth/authService.ts` (logout, token refresh чрез Entra ID) | FE-SH-03 | 2 |
| FE-SH-11 | Напишете Jest + React Testing Library unit тестове | `src/__tests__/` | FE-SH-01–FE-SH-10 | 5 |
| FE-SH-12 | Напишете Nginx config + Dockerfile за статичен React build | `nginx.conf`, `Dockerfile` | FE-SH-01 | 1 |
| FE-SH-13 | Настройте GitHub Actions CI/CD pipeline | `.github/workflows/ci-cd.yml` | FE-SH-01, INFRA-14 | 1 |
| **Общо за групата** | | | | **35** |

### Legal Frontend (Micro-Frontend Remote)
> Област: **Frontend** | Repo: `erp-legal-frontend`

| ID | Задача | Deliverable | Зависи от | Est. (дни) |
|----|------|-------------|------------|-------------|
| FE-L-01 | Инициализирайте Legal remote app (React.js + TypeScript + Webpack Module Federation remote config) | `frontends/erp-legal/` | FE-SH-01, FE-DSGN-01 | 1 |
| FE-L-02 | Имплементирайте Legal Dossier list + detail страници | `src/pages/Dossiers/` | FE-L-01, BE-L-02 | 4 |
| FE-L-03 | Имплементирайте Legal Case management UI | `src/pages/Cases/` | FE-L-01, BE-L-03 | 4 |
| FE-L-04 | Имплементирайте Legal Deadline calendar / tracker UI | `src/pages/Deadlines/` | FE-L-01, BE-L-04 | 3 |
| FE-L-05 | Имплементирайте Law Acts / Knowledge Base search UI с AI chat (SSE streaming) | `src/pages/Search/`; SSE интеграция със streaming endpoint на Legal FastAPI (`GET /api/legal/ai/generation/stream/{correlation_id}`) | FE-L-01, BE-L-07, BE-L-13 | 5 |
| FE-L-06 | Имплементирайте document upload UI (тригерира ingestion pipeline) | `src/components/DocumentUpload.tsx` | FE-L-01, BE-L-06 | 3 |
| FE-L-07 | Напишете Nginx config + Dockerfile + GitHub Actions CI/CD | `nginx.conf`, `Dockerfile`, `.github/workflows/ci-cd.yml` | FE-L-01, INFRA-14 | 2 |
| **Общо за групата** | | | | **22** |

### Marketing Frontend (Micro-Frontend Remote)
> Област: **Frontend** | Repo: `erp-marketing-frontend`

| ID | Задача | Deliverable | Зависи от | Est. (дни) |
|----|------|-------------|------------|-------------|
| FE-M-01 | Инициализирайте Marketing remote app | `frontends/erp-marketing/` | FE-SH-01, FE-DSGN-01 | 1 |
| FE-M-02 | Имплементирайте Campaigns list + detail + creation UI | `src/pages/Campaigns/` | FE-M-01, BE-M-02 | 4 |
| FE-M-03 | Имплементирайте Lead management UI + lead-to-client conversion workflow | `src/pages/Leads/`; `src/components/LeadConversion.tsx` | FE-M-01, BE-M-03 | 4 |
| FE-M-04 | Имплементирайте Ticket / Support queue UI | `src/pages/Tickets/` | FE-M-01, BE-M-04 | 3 |
| FE-M-05 | Имплементирайте ETL Wizard (step-by-step bulk import UI) | `src/pages/ETLWizard/` | FE-M-01, BE-M-07 | 4 |
| FE-M-06 | Имплементирайте campaign analytics dashboard (графики чрез Recharts / Nivo) | `src/pages/Analytics/` | FE-M-01, BE-M-06 | 4 |
| FE-M-07 | Напишете Nginx config + Dockerfile + GitHub Actions CI/CD | `nginx.conf`, `Dockerfile`, `.github/workflows/ci-cd.yml` | FE-M-01, INFRA-14 | 2 |
| **Общо за групата** | | | | **22** |

### Accounting Frontend (Micro-Frontend Remote)
> Област: **Frontend** | Repo: `erp-accounting-frontend`

| ID | Задача | Deliverable | Зависи от | Est. (дни) |
|----|------|-------------|------------|-------------|
| FE-AC-01 | Инициализирайте Accounting remote app | `frontends/erp-accounting/` | FE-SH-01, FE-DSGN-01 | 1 |
| FE-AC-02 | Имплементирайте Journal Entry list + creation UI (double-entry форма) | `src/pages/JournalEntries/` | FE-AC-01, BE-A-03 | 5 |
| FE-AC-03 | Имплементирайте Chart of Accounts management UI | `src/pages/Accounts/` | FE-AC-01, BE-A-02 | 3 |
| FE-AC-04 | Имплементирайте Accounting Periods UI + period closure workflow | `src/pages/Periods/` | FE-AC-01, BE-A-05 | 3 |
| FE-AC-05 | Имплементирайте финансов dashboard (Balance Sheet, P&L графики) | `src/components/FinancialDashboard.tsx` | FE-AC-01, BE-A-06 | 4 |
| FE-AC-06 | Имплементирайте ДДС / Tax declaration download UI | `src/pages/TaxDeclarations/` | FE-AC-01, BE-A-07 | 3 |
| FE-AC-07 | Имплементирайте Audit план + findings UI | `src/pages/Audit/` | FE-AC-01, BE-A-08 | 4 |
| FE-AC-08 | Напишете Nginx config + Dockerfile + GitHub Actions CI/CD | `nginx.conf`, `Dockerfile`, `.github/workflows/ci-cd.yml` | FE-AC-01, INFRA-14 | 2 |
| **Общо за групата** | | | | **25** |

---

## Phase 5 — Интеграция, QA и тестване
> Област: **QA** с cross-team поддръжка

| ID | Задача | Deliverable | Зависи от | Est. (дни) |
|----|------|-------------|------------|-------------|
| QA-01 | Напишете Playwright E2E test suite — login flow (Entra ID mock) | `e2e/auth.spec.ts` | FE-SH-03 | 3 |
| QA-02 | Напишете Playwright E2E тестове — Legal модул пълен flow | `e2e/legal.spec.ts` | FE-L-01–FE-L-06 | 5 |
| QA-03 | Напишете Playwright E2E тестове — Marketing модул пълен flow | `e2e/marketing.spec.ts` | FE-M-01–FE-M-06 | 5 |
| QA-04 | Напишете Playwright E2E тестове — Accounting модул пълен flow | `e2e/accounting.spec.ts` | FE-AC-01–FE-AC-07 | 5 |
| QA-05 | Валидирайте, че Pact consumer-driven contract тестове преминават за всички двойки услуги | Pact verification отчети | CONT-07 | 2 |
| QA-06 | Валидирайте APIM JWT отхвърляне на невалидни/изтекли tokens | Postman/pytest test колекция срещу APIM | INFRA-09, BE-L-01 | 2 |
| QA-07 | Валидирайте RBAC: ролите блокират правилни endpoints във всички модули | Role-based test матрица | INFRA-11, QA-06 | 3 |
| QA-08 | Load test FastAPI услуги (10+ конкурентни заявки, валидация на autoscaling) | k6 / Locust load test отчет | Всички BE услуги | 4 |
| QA-09 | AI agent integration test — пълен ingestion → analysis → generation pipeline | `tests/ai_pipeline_integration.py` | AI-ING-01–AI-GEN-08 | 5 |
| QA-10 | LangSmith evaluation — качество на AI output за всички типове агенти | LangSmith evaluation dataset + резултати | AI-01–AI-SRC-07 | 3 |
| QA-11 | Security сканиране: OWASP ZAP срещу всички публични endpoints | ZAP scan отчет; поправяне на идентифицирани проблеми | Всички BE + FE | 3 |
| QA-12 | Accessibility audit (WCAG 2.1 AA) на всички frontend модули | Lighthouse / axe-core отчет; поправяне на критични проблеми | Всички FE | 3 |
| QA-13 | Тестване на Responsive design (desktop, tablet, mobile) | Cross-device test отчет | Всички FE | 2 |
| QA-14 | **AI red-teaming**: prompt-injection + PII-leak test suite се изпълнява срещу всички 5 агента седмично в CI; валидира че `CORE-18` (PII redactor) и `CORE-19` (prompt guard) остават ефективни докато prompts еволюират | `tests/ai_redteam/`; LangSmith evaluation suite; седмичен scheduled GitHub Action | CORE-18, CORE-19, AI-01–AI-SRC-07 | 3 |
| **Общо за групата** | | | | **48** |

---

## Phase 6 — Production Readiness
> Област: **DevOps** | Cross-team

| ID | Задача | Deliverable | Зависи от | Est. (дни) |
|----|------|-------------|------------|-------------|
| PROD-01 | Конфигурирайте ACA autoscaling правила на услуга (HTTP + Service Bus queue тригери) | ACA scale правила според таблицата в архитектурната спецификация | Всички услуги разгърнати в ACA | 2 |
| PROD-02 | Конфигурирайте overnight scale-to-zero schedules за dev + staging среди | ACA replica schedules | INFRA-03, INFRA-04 | 1 |
| PROD-03 | Конфигурирайте Azure Monitor Alerts (брой реплики, HTTP error rate, latency прагове) | Alert правила + action groups (email/Teams) | INFRA-16 | 2 |
| PROD-04 | Съхранете всички environment image SHA tags в Git (позволява пълен environment rebuild) | `infra/image-versions/` на среда | Всички CI/CD pipelines | 1 |
| PROD-05 | Изпълнете пълен staging smoke test + sign-off | Staging sign-off checklist | Всички фази завършени | 2 |
| PROD-06 | Production go-live разгръщане (manual approval gate в GitHub Environments) | Всички услуги работят в prod ACA среда | PROD-05 | 1 |
| PROD-07 | Напишете runbooks: restore от backup, service rollback, incident response | `docs/runbooks/` | PROD-06 | 3 |
| PROD-08 | Напишете README на услуга и API документация (FastAPI OpenAPI auto-generated) | `README.md` на repo; Swagger UI достъпен на `/docs` | Всички услуги | 3 |
| PROD-09 | Авторство + поддръжка на Acceptance Criteria приложение за останалите ~130 задачи (top 20 най-високи рискови са pre-seeded — вижте приложението); налагайте DoD checklist чрез PR template | Актуализирано *Приложение с критерии за приемане* в `Tasks.md`; `.github/PULL_REQUEST_TEMPLATE.md` доставен във всеки repo с DoD checklist | Всички фази | 5 |
| INFRA-31 | Напишете runbook за **expand → migrate → contract** schema-evolution + **migration rollback** runbook (forward-only по подразбиране; emergency PITR от DB-20 + redeploy на предишен image SHA от PROD-04); верифицирайте и двата чрез staging "game day" | `docs/runbooks/db-migrations.md`, `docs/runbooks/release-rollback.md` | INFRA-29, INFRA-30, DB-20, PROD-04 | 2 |
| **Общо за групата** | | | | **22** |

---

## Приложение с критерии за приемане (задачи с най-висок риск)

> **Как да четете това приложение:** Всеки запис по-долу добавя **специфични** критерии за приемане върху проектния Definition of Done. Ако дадена задача не е изброена тук, нейната колона *Deliverable* в основните таблици е имплицитният критерий за приемане. Останалите задачи ще бъдат попълнени под **PROD-09** от отговорния task owner преди започване на работа.

### Phase 0 — Инфраструктура (security-critical)

| Задача | Критерии за приемане |
|---|---|
| **INFRA-09** APIM | • APIM JWT валидацията отхвърля изтекли и invalid-signature tokens с HTTP 401 (верифицирано чрез автоматизиран pytest срещу live dev APIM инстанция).<br>• Per-route rate limit е конфигурируем чрез Terraform променлива; default 100 req/min/user.<br>• Routing правилата покриват Legal, Marketing, Accounting API-та и Django internal API; **deny-by-default** за всеки друг път.<br>• Forwarded claim headers (`x-user-id`, `x-user-roles`) са налични на всяка upstream заявка. |
| **INFRA-11** Entra ID | • App registration създаден със списъка с роли дефиниран в DB-14 (`legal.reader`, `legal.editor`, `marketing.admin`, `accounting.editor` и т.н.).<br>• MFA наложено чрез Conditional Access за всяка роля завършваща на `.admin` или `.editor`.<br>• Token TTL ≤ 60 min, refresh TTL ≤ 8 h.<br>• Тестови потребителски акаунти seeded на среда. |
| **INFRA-26** Defender for Storage | • Defender активиран на upload storage account в dev, staging, prod.<br>• Defender event-grid topic излъчва `Microsoft.Security.MalwareScanningResult` събития.<br>• Cost alert конфигуриран на $5/ден на среда. |

### Phase 1a — `erp-core` (фундаментален + AI safety)

| Задача | Критерии за приемане |
|---|---|
| **CORE-14** Django клиент | • Клиентът се регенерира автоматично, когато Django OpenAPI схемата се променя (CI се проваля, ако commit-натият клиент е остарял).<br>• Всички заявки включват managed-identity bearer token.<br>• Retries: exponential backoff с максимум 3 опита на 5xx и мрежови грешки; никога не повтаря на 4xx.<br>• Имплементира `BaseRepository` за поне един ресурс и преминава споделения repository contract тест. |
| **CORE-16** DLQ + idempotency | • Poison съобщение премествано в DLQ след **5** опита за доставка (конфигурируемо).<br>• Idempotency cache TTL = 24 h, ключ по `MessageId` в Redis.<br>• Re-delivery на същия `MessageId` е no-op — верифицирано чрез интеграционен тест, който публикува едно и също съобщение два пъти и проверява един единствен side-effect.<br>• DLQ depth изложен като метрика в Azure Monitor. |
| **CORE-17** Transactional outbox | • Outbox row insert и domain write са в същата DB транзакция (верифицирано чрез failure-injection тест, който abort-ва след domain write — не се публикува Service Bus съобщение).<br>• Relay процес източва outbox в FIFO ред с ≤ 5 s p99 lag при стабилен load.<br>• Outbox table TTL: записи по-стари от 7 дни и маркирани като изпратени се изчистват ежедневно. |
| **CORE-18** PII redactor | • Открива и маскира: BG **ЕГН**, **IBAN**, имейл, телефон, имена на лица (EN+BG), BG адреси.<br>• Постига ≥ **95% recall** върху labelled 50-text PII корпус commit-нат в repo-то.<br>• Добавя < **30 ms** латентност за входове до 4 KB (benchmark в CI).<br>• Публичен API: `redact(text: str) -> RedactedText` и `is_safe(text: str) -> bool`.<br>• **Hard rule:** всяко повикване към Azure OpenAI без предхождащ `redact()` проваля static-analysis check на CI. |
| **CORE-19** Prompt-injection guardrails | • Премахва известни system-prompt-override паттерни (`"ignore previous instructions"`, jailbreak корпус от OWASP LLM Top 10).<br>• Валидира LLM output срещу очакваната Pydantic схема; non-conforming output тригерира единичен retry, после raise-ва `PromptGuardError`.<br>• Token cap наложен на заявка (конфигурируем; default 8 K input / 2 K output).<br>• Test корпус от 30 известни jailbreak опита: ≥ **90% блокирани**. |

### Phase 1b — Слой за данни

| Задача | Критерии за приемане |
|---|---|
| **DB-16** Django internal REST API | • Endpoints следват `/internal/v1/<schema>/<resource>` шаблон; OpenAPI схемата е източник на истина.<br>• Всички endpoints изискват service-to-service auth (DB-24).<br>• Response shapes съответстват на Pydantic моделите в `erp-contracts` (валидирано чрез Pact).<br>• p95 латентност < **100 ms** за single-record reads срещу 1M-row таблица в dev DB. |
| **DB-20** Backup / restore валидация | • Point-in-time restore от 24-h-стар backup завършва в < 30 min на staging-sized DB.<br>• Restored DB преминава row-count + checksum diff срещу източника.<br>• Geo-redundant backup верифициран в secondary Azure регион.<br>• Runbook (`docs/runbooks/restore.md`) преминава точно през командата `az postgres flexible-server restore` и включва screenshots. |
| **DB-24** Service-to-service auth | • Само Container Apps с роля `internal-api-caller` могат да извикват Django internal API; всички други връщат HTTP 403.<br>• Tokens валидирани срещу публичните ключове на Entra ID; без shared secrets.<br>• Token caching в Redis с TTL = token expiry минус 60 s.<br>• Audit log запис написан за всяко повикване на internal API. |
| **DB-25** Schema drift contract тест | • CI job вдига свеж Django контейнер, изпълнява миграции, после изпълнява всеки FastAPI service integration test suite срещу него.<br>• Job-ът се проваля (и PR-ът е блокиран), ако някой FastAPI integration test се провали поради schema mismatch.<br>• Job-ът се изпълнява в < 10 min общо. |

### Phase 2 — Module APIs (business-correctness-critical)

| Задача | Критерии за приемане |
|---|---|
| **BE-A-04** Double-entry валидация | • Debit total трябва да е равен на credit total с толеранс **0.01 BGN**.<br>• Небалансирани entries връщат HTTP **422** с body `{"errors": [{"line": <n>, "delta": <amount>}, ...]}`.<br>• Валидацията се изпълнява в < **50 ms** за entries с ≤ 100 lines.<br>• Чиста функция — без DB writes; извиква се от `routes/journal_entries.py` преди persistence.<br>• ≥ 20 test случая покриващи: balanced, off-by-0.01, off-by-large, single line, empty, отрицателни суми. |
| **BE-A-06** Финансови отчети | • Balance Sheet, P&L и Cash Flow генерирани за всеки затворен период; цифрите се връзват към сумата на underlying journal lines (верифицирано чрез reconciliation тест).<br>• Output формати: JSON (API) и PDF (download).<br>• Включена колона за сравнение с предходен период.<br>• Генерирането завършва в < 5 s за година данни на dev DB. |
| **BE-A-07** ДДС + корпоративен данък | • ДДС декларацията съответства на официалната NAP XML схема (валидирана срещу публикуваната XSD в CI).<br>• Декларацията за корпоративен данък покрива българските стандартни ставки актуални към януари 2026.<br>• Генерираната декларация е byte-identical когато се изпълнява повторно върху същия input (детерминистична).<br>• Примерни декларации от известен input set са commit-нати и diff-нати в CI. |
| **BE-L-11** Virus / malware scan | • Файлове флагнати от Defender в рамките на **5 min** от качването са поставени в карантина и потребителят е нотифициран чрез Notification Center.<br>• `ai.ingestion.requests` **никога** не е публикуван докато Defender сканът не върне clean.<br>• Defender грешки (timeout / unreachable) **fail closed**: файлът е отхвърлен, не е пропуснат.<br>• Файлове в карантина се пазят 30 дни, после автоматично се изтриват. |
| **BE-L-12** Upload валидация | • Максимален размер на файл: **50 MB** (конфигурируем на среда чрез Key Vault secret).<br>• Magic-byte check отхвърля изпълними файлове (PE, ELF, Mach-O), скриптове и архиви прикрити като PDF/DOCX/XLSX.<br>• Filename sanitisation премахва path separators и non-printable символи; запазва Unicode (български имена на файлове).<br>• Отхвърлянето връща HTTP **415** с machine-readable error code. |

### Phase 3 — AI Search Agent (sync HTTP)

| Задача | Критерии за приемане |
|---|---|
| **AI-SRC-05** HTTP `/search` endpoint | • Sync request/response — без Service Bus.<br>• p95 латентност ≤ **800 ms** за vector search върху 100 K-embedding корпус на dev DB; **p99 ≤ 1.5 s**.<br>• **Availability SLO: 99.5% месечно** (≈ 3.6 h/месец error budget). Azure Monitor burn-rate alerts: **fast burn** (1 h burn > 14×) и **slow burn** (6 h burn > 6×) page on-call.<br>• Връща HTTP **503** в рамките на **5 s**, ако Azure OpenAI или pgvector е недостъпен (fail-fast — circuit breaker от страна на caller-а се справя с backoff; module API-тата преминават към keyword search fallback според BE-L-14).<br>• `correlation_id` header се пропагира от заявка към отговор и във всички downstream logs/traces.<br>• Min replicas = **2** в prod (разпределени между AZs — верифицирано от AI-SRC-08); HTTP-driven autoscaling до max = 5. |

### Phase 5 — QA

| Задача | Критерии за приемане |
|---|---|
| **QA-14** AI red-teaming | • Suite покрива всички 5 AI агента.<br>• Включва OWASP LLM Top 10 prompt-injection паттерни + custom корпус от 30 ERP-domain jailbreak опита.<br>• Включва 50 PII-leak test случая (трябва да дават нула PII в agent output).<br>• Изпълнява се седмично в CI като scheduled GitHub Action; failure page-ва on-call.<br>• Резултатите се публикуват в LangSmith evaluation dashboard. |

### Phase 6 — Production Readiness (prod вратата)

| Задача | Критерии за приемане |
|---|---|
| **PROD-05** Staging smoke + sign-off | • Пълен Playwright E2E suite зелен на staging.<br>• Load test (k6) на конфигурирания autoscale праг преминава с p95 латентност < target SLO и нула 5xx.<br>• Security gates зелени: OWASP ZAP scan + QA-14 AI red-teaming suite и двата чисти.<br>• Sign-off записан в `docs/signoffs/<release-tag>.md` от product owner + tech lead. |
| **PROD-06** Production go-live | • Всички Production DoD елементи (по-горе) зелени.<br>• GitHub Environment manual approval записан с име на reviewer + timestamp.<br>• Smoke probe (`/health` + 1 представително API повикване на услуга) зелен в рамките на 5 min от deploy.<br>• Image SHAs commit-нати в `infra/image-versions/prod/` (PROD-04). |
| **PROD-07** Runbooks | • Един runbook на сценарий: restore-from-backup, service-rollback, AI-agent-failover, APIM-key-rotation, incident-response.<br>• Всеки runbook тестван в staging "game day" в рамките на 30 дни от написването.<br>• Всеки runbook включва: pre-checks, точни команди, стъпки за валидация, escalation контакти. |

---

## Обобщение на усилието и изчисление на персонал

| Фаза / Група | Est. (човеко-дни) |
|---|---|
| Phase 0 — Инфраструктура и DevOps | 74 |
| Phase 1a — `erp-core` пакет | 54 |
| Phase 1a — `erp-contracts` пакет | 23 |
| Phase 1b — Слой за данни (Django) | 63 |
| Phase 2 — Legal FastAPI | 34 |
| Phase 2 — Marketing FastAPI | 29 |
| Phase 2 — Accounting FastAPI | 43 |
| Phase 2 — Celery Worker | 7 |
| Phase 3 — AI Orchestrator | 16 |
| Phase 3 — AI Ingestion Agent | 18 |
| Phase 3 — AI Analysis Agent | 15 |
| Phase 3 — AI Generation Agent | 25 |
| Phase 3 — AI Classification Agent | 10 |
| Phase 3 — AI Search Agent | 19 |
| Phase 4 — Дизайн (Figma) | 10 |
| Phase 4 — Shell Frontend | 35 |
| Phase 4 — Legal Frontend | 22 |
| Phase 4 — Marketing Frontend | 22 |
| Phase 4 — Accounting Frontend | 25 |
| Phase 5 — Интеграция, QA и тестване | 48 |
| Phase 6 — Production Readiness | 22 |
| **Обща сума** | **614 човеко-дни** |

### Усилие по дисциплина

| Дисциплина | Est. (човеко-дни) | Групи |
|---|---|---|
| **DevOps** | 74 + 22 = **96** | Phase 0, Phase 6 |
| **Backend (споделени + услуги + AI)** | 54 + 23 + 34 + 29 + 43 + 7 + 16 + 18 + 15 + 25 + 10 + 19 = **293** | Phase 1a, Phase 2, Phase 3 |
| **Database (Django data layer)** | **63** | Phase 1b |
| **Frontend / Дизайн** | 10 + 35 + 22 + 22 + 25 = **114** | Phase 4 |
| **QA** | **48** | Phase 5 |
| **Общо** | **614** | |

### Формула за персонал

```
Необходими FTE (на дисциплина) = Дисциплина човеко-дни / (Календарни дни на проекта × Коефициент на фокус)
```

- **Календарни дни на проекта** = работни дни в планирания график (напр. 6 месеца ≈ 120 работни дни).
- **Коефициент на фокус** = 0.7 (отчита срещи, code review, ramp-up, блокери).

**Пример — 6-месечно доставяне (≈ 120 работни дни × 0.7 фокус → 84 продуктивни дни на FTE):**

| Дисциплина | Дни | Необходими FTE | Закръглено нагоре |
|---|---|---|---|
| DevOps | 96 | 96 / 84 = 1.14 | **2** |
| Backend | 293 | 293 / 84 = 3.49 | **4** |
| Database | 63 | 63 / 84 = 0.75 | **1** (може да се припокрива с Backend) |
| Frontend / Дизайн | 114 | 114 / 84 = 1.36 | **2** (1 designer-leaning + 1 dev-leaning, или 2 dev-и + контрактиран дизайн) |
| QA | 48 | 48 / 84 = 0.57 | **1** |
| **Общ екип** | **614** | **7.31** | **~8 души** за 6-месечен MVP |

> Регулирайте календарните дни и коефициента на фокус към действителния си график за преизчисляване на FTE изискването.

