# Backend Developer Interview — Questions & Answers
**Project:** ERP Platform (Legal / Marketing / Accounting)
**Scope:** Tasks in `documentation/Tasks.md` — Phases 1a, 1b, 2, 3 (`erp-core`, `erp-contracts`, Django data layer, FastAPI services, Celery, AI agents)
**Stack reference:** `architecture_specifics.md`

> Use this as a structured interview guide. Each section is graded by difficulty (Junior / Mid / Senior). Pick ~12 questions per candidate; don't run all 52.

---

## Part 1 — Python Core (Junior → Mid)

### Q1. What is the difference between a `@staticmethod`, a `@classmethod`, and a normal instance method?
**Answer.**
- **Instance method** — first argument is `self` (the instance). Can read/write instance state and class state. Called on an instance: `obj.method()`.
- **`@classmethod`** — first argument is `cls` (the class itself). Cannot access instance state, but can access/modify class-level attributes. Useful for **alternative constructors** (`User.from_dict(...)`). Called on the class or an instance.
- **`@staticmethod`** — receives no implicit first argument. No access to `cls` or `self`. Just a plain function namespaced inside the class. Use it when the logic is conceptually related to the class but doesn't need its state — e.g. a helper validator.

```python
class Invoice:
    tax_rate = 0.20

    def __init__(self, amount):
        self.amount = amount

    def total(self):                 # instance method
        return self.amount * (1 + Invoice.tax_rate)

    @classmethod
    def from_cents(cls, cents):      # alternative constructor
        return cls(cents / 100)

    @staticmethod
    def is_valid_amount(x):          # pure helper
        return x >= 0
```
**Follow-up.** When prefer `@classmethod` over `@staticmethod`?
*Whenever the helper needs to be polymorphic across subclasses — `cls` resolves to the actual subclass, so `Subclass.from_dict(...)` builds a `Subclass` instance.*

---

### Q2. Mutable vs immutable types — why does it matter?
**Answer.** Immutable: `int`, `float`, `str`, `tuple`, `frozenset`, `bytes`. Mutable: `list`, `dict`, `set`, most user-defined classes.
- Immutable objects are **hashable** → can be dict keys / set members.
- Passing a mutable object to a function shares state — mutation is visible to the caller.
- The classic gotcha: **mutable default arguments**.
- Can a class be a key in a dict? Only if it implements `__hash__` and `__eq__` correctly, and is immutable. By default, user-defined classes are mutable and unhashable.

```python
def append_item(item, bucket=[]):     # BUG: bucket is shared across calls
    bucket.append(item)
    return bucket
```
Fix: `def append_item(item, bucket=None): bucket = bucket or []`.

---

### Q3. Explain `*args` and `**kwargs`.
**Answer.** `*args` packs extra positional arguments into a tuple; `**kwargs` packs extra keyword arguments into a dict. Used to write wrappers that forward arguments without 
knowing the signature in advance — e.g. decorators.

---

### Q4. How does Python's GIL affect concurrency? When threads vs `asyncio` vs multiprocessing?
**Answer.**
- **GIL** = Global Interpreter Lock; only one thread executes Python bytecode at a time in CPython.
- **Threads** — good for I/O-bound work where the GIL is released during syscalls (file/network).
- **`asyncio`** — preferred model for I/O-bound work. Single-threaded cooperative concurrency, no GIL contention, very low per-task overhead. FastAPI is built on it.
- **`multiprocessing`** — for CPU-bound work; bypasses the GIL by spawning separate processes.
- For our FastAPI services almost everything is I/O — DB calls, Service Bus, HTTP to Azure OpenAI — so `async/await` everywhere.

---

### Q5. What is a context manager? How do you write one?
**Answer.** Object implementing `__enter__` / `__exit__` (or `__aenter__` / `__aexit__` for async). Used with `with` to guarantee cleanup. Two common ways:

```python
from contextlib import contextmanager

@contextmanager
def db_transaction(session):
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()
```
Used heavily in `erp-core` for DB sessions, HTTP client lifetimes, Service Bus message receivers.

---

### Q6. Difference between `is` and `==`.
**Answer.** `is` checks identity (same object in memory). `==` checks equality (`__eq__`). Only use `is` for `None`, `True`, `False`, and singletons.

---

### Q7. List comprehension vs generator expression?
**Answer.** List comp `[x*2 for x in xs]` materialises the whole list in memory. Generator expression `(x*2 for x in xs)` is lazy — yields one item at a time. 
Prefer generators for large/streaming data (e.g. parsing huge CSVs in `BE-M-07 ETL import`).

---

### Q8. Explain duck typing and Python's approach to interfaces.
**Answer.** "If it walks like a duck…" — Python doesn't check types, it checks behaviour. `abc.ABC` + `abstractmethod` lets you declare 
interfaces explicitly. `typing.Protocol` does it structurally (no inheritance required). In `erp-core` we use `ABC` for `BaseService`, `BaseRepository`, 
`BaseEventPublisher`, `BaseAIAgent` so all services share a contract.

---

### Q9. How does Python's `import` system work? What is `__init__.py`?
**Answer.** When you `import x`, Python searches `sys.path`, finds a module/package, executes its top-level code once, and caches 
it in `sys.modules`. `__init__.py` (optional in modern Python with namespace packages, but still common) marks a directory as a regular package 
and may expose a curated public API. In `erp-core` we use `__init__.py` to re-export the stable public surface (`from erp_core.config import Settings`).

---

### Q10. What does `@dataclass` give you over a regular class?
**Answer.** Auto-generated `__init__`, `__repr__`, `__eq__` from declared fields. `frozen=True` makes instances immutable & hashable. 
`field(default_factory=list)` avoids the mutable-default trap. For request/response DTOs we prefer **Pydantic v2** instead — it adds runtime 
validation and JSON schema; dataclasses are best for internal value objects.

---

### Q11. What do `__slots__` do?
**Answer.** Replaces the per-instance `__dict__` with a fixed set of attributes. Saves memory and slightly speeds attribute access. 
Trade-off: no dynamic attributes, awkward with multiple inheritance. Useful for hot-path value objects, irrelevant for typical request handlers.

---

### Q12. Shallow copy vs deep copy?
**Answer.** `copy.copy(x)` copies the top-level container but shares nested objects. `copy.deepcopy(x)` recursively copies 
everything. Mutating a nested list in a shallow copy mutates the original — common bug source.

---

## Part 2 — Async, Concurrency, Performance (Mid)

### Q13. What does `async def` actually return? What's an awaitable?
**Answer.** Calling an `async def` function returns a **coroutine object** — it does **not** start executing. To run it you must `await` it (inside another coroutine) or hand it to an event loop (`asyncio.run`, `asyncio.create_task`). An awaitable is anything with `__await__`: coroutines, `Task`s, `Future`s.

---

### Q14. Difference between `await coro` and `asyncio.create_task(coro)`?
**Answer.** `await coro` runs sequentially — blocks the current coroutine until it finishes. `create_task` schedules the coroutine to run **concurrently** on the event loop and returns a handle; you can `await task` later or `asyncio.gather(*tasks)` to wait for many in parallel. Critical for fan-out, e.g. parallel calls to Azure OpenAI from the AI Generation Agent.

---

### Q15. What goes wrong if you call a blocking function (e.g. `time.sleep`, sync `requests`) from an async handler?
**Answer.** It blocks the entire event loop — every other request stalls. Fixes:
- Use the async equivalent (`asyncio.sleep`, `httpx.AsyncClient`).
- Or wrap with `await asyncio.to_thread(blocking_fn, ...)` to offload to a thread pool.

---

### Q16. How would you implement a retry with exponential backoff for Azure Service Bus publishing?
**Answer.** Use `tenacity`:
```python
from tenacity import retry, wait_exponential, stop_after_attempt, retry_if_exception_type
@retry(
    wait=wait_exponential(multiplier=0.5, max=10),
    stop=stop_after_attempt(5),
    retry=retry_if_exception_type(ServiceBusError),
)
async def publish(msg): ...
```
Make sure the operation is **idempotent** (deduplication ID on the message) so retries are safe.

---

## Part 3 — FastAPI & Pydantic (Mid → Senior)

### Q17. How does FastAPI use Pydantic? What changes between v1 and v2?
**Answer.** FastAPI parses request bodies/query/path parameters into Pydantic models, validates them, and uses the same models to generate the OpenAPI schema. Pydantic **v2** is required by our `erp-contracts`:
- Rewritten in Rust (much faster).
- New API: `model_validate` / `model_dump` instead of `parse_obj` / `dict`.
- Strict typing by default; `ConfigDict` replaces inner `class Config`.
- `Field(..., validation_alias=...)` for input/output alias separation.
- `model_validator(mode="before"/"after")` replaces `@root_validator`.

---

### Q18. How do you implement dependency injection in FastAPI?
**Answer.** Use `Depends`. A dependency is any callable returning a value; FastAPI resolves it per request, with sub-dependencies and caching:

```python
def get_db() -> Iterator[Session]:
    s = SessionLocal()
    try: yield s
    finally: s.close()

@router.get("/dossiers/{id}")
def read(id: int, db: Session = Depends(get_db), user = Depends(get_current_user)):
    ...
```
Per architecture, `get_current_user` reads APIM-forwarded headers via `erp_core.security` — FastAPI itself does **not** validate JWTs.

---

### Q19. Where does authentication happen in our architecture? Why is FastAPI auth-free?
**Answer.** Authentication is centralised at **Azure API Management (APIM)**, which validates the Entra ID JWT signature against Microsoft's JWKS. Only authenticated requests are forwarded to FastAPI, with user ID and roles in headers. FastAPI **trusts the gateway** and only does fine-grained authorisation (e.g. "does this user own this dossier?") using `erp_core.security` helpers. Benefits: no duplicated JWT logic across 6 services, consistent enforcement, easy key rotation, services testable without auth in CI.

---

### Q20. How do you stream Server-Sent Events (SSE) from FastAPI?
**Answer.** Return a `StreamingResponse` with `media_type="text/event-stream"` and an async generator yielding `data: ...\n\n` chunks. Used in `AI-GEN-08 / FE-L-05` for real-time streaming of AI generation output:

```python
async def event_stream():
    async for chunk in agent.stream(prompt):
        yield f"data: {chunk}\n\n"
return StreamingResponse(event_stream(), media_type="text/event-stream")
```

---

### Q21. How would you handle errors consistently across all FastAPI services?
**Answer.** Use the shared `erp_core.exceptions` hierarchy (`AppError`, `NotFoundError`, `ConflictError`, `ValidationError`) plus a global `exception_handler` registered in the FastAPI app factory. Each maps to a fixed HTTP status (404 / 409 / 422) and a standard JSON envelope `{ "error": { "code", "message", "correlation_id" } }`. The middleware in `erp_core/middleware` injects the correlation ID for traceability across services.

---

### Q22. How do you test a FastAPI endpoint?
**Answer.**
- **Unit:** call the underlying service function directly with mocks.
- **Integration:** `httpx.AsyncClient(app=app, base_url="http://test")` with `pytest-asyncio`. Use `app.dependency_overrides` to swap real DB / Service Bus for fakes.
- **Contract:** Pact tests in CI verify provider/consumer agreement (CONT-07).

---

## Part 4 — Django Data Layer (Mid)

### Q23. Django is described as "data layer only" — what does that mean architecturally?
**Answer.** Django owns the PostgreSQL connection, ORM models, migrations, and the admin panel. It does **not** serve the public API — only an internal admin UI and an internal contract used by FastAPI through `erp-core`'s `BaseRepository`. FastAPI services **never** write to the DB directly. This keeps schema ownership single-sourced and avoids the multi-writer problem.

---

### Q24. How do per-module schemas work in a single PostgreSQL instance?
**Answer.** One physical database, multiple schemas (`legal`, `marketing`, `accounting`, `core`). In Django, configure a **database router** (`db_router.py`) that picks the schema based on the model's app label. Set `search_path` per connection or use schema-qualified table names (`legal.dossier`). Migrations stay independent per app, but live in one Postgres instance — saves cost, isolates data logically, simplifies backups.

---

### Q25. What is N+1 in the ORM and how do you fix it?
**Answer.** Loading parent rows then issuing one query per child = 1 + N queries. Fix with `select_related` (FK, single JOIN) and `prefetch_related` (M2M / reverse FK, two queries with IN clause). Always inspect with `django-debug-toolbar` or `EXPLAIN`.

---

### Q26. Why pgvector and not a separate vector DB?
**Answer.** For MVP, pgvector adds vector similarity search to our existing Postgres — zero extra infra, single backup target, transactional consistency between business data and embeddings. At scale we'd migrate to **Azure AI Search** (hybrid vector + keyword, native Azure integration). The `BaseRepository` abstraction means application code doesn't change.

---

### Q27. Migrations: zero-downtime patterns?
**Answer.**
- Add columns as **nullable** first; backfill in a separate migration; then enforce `NOT NULL`.
- Never rename a column in one shot — add new, dual-write, backfill, switch reads, drop old.
- Create indexes `CONCURRENTLY` (custom migration with `RunSQL`) to avoid table locks.
- Test on a prod-shaped staging DB.

---

## Part 5 — Messaging, Service Bus & Distributed Systems (Mid → Senior)

### Q28. Why Azure Service Bus in prod and RabbitMQ locally? How is the abstraction done?
**Answer.** Service Bus is fully managed, has dead-letter queues, sessions, and natively drives ACA autoscaling via queue depth. Locally we don't want a cloud dependency, so RabbitMQ runs in Docker Compose. The `erp_core.messaging` package exposes `BaseEventPublisher` / `BaseEventConsumer` interfaces with two implementations selected by `MESSAGING_BACKEND` env var. Application code only sees the interface.

---

### Q29. Difference between a queue and a topic?
**Answer.** Queue = point-to-point, one consumer reads each message. Topic = pub/sub, multiple subscriptions each get their own copy. Our `module.events` is a topic (cross-module subscribers); `ai.ingestion.requests` is a queue (one ingestion agent processes each upload).

---

### Q30. What is at-least-once delivery? How do you write an idempotent consumer?
**Answer.** Service Bus guarantees at-least-once — a message can be delivered more than once after a crash or visibility timeout. Idempotent consumers:
- Use a **deduplication key** (e.g. `message_id` or business key) and check "have I processed this already?" against a store (DB unique constraint, Redis set).
- Make state changes commutative or use upserts.
- Only `complete` the message after the side effect is durably committed.

---

### Q31. How does ACA autoscaling on queue depth work, and what does that mean for consumer code?
**Answer.** ACA's KEDA-based scaler watches the active message count and adds replicas (default trigger: 1 message per replica). Implications:
- Consumers must be **stateless**.
- Long-running tasks should checkpoint or split into smaller messages so a new replica can take over.
- Cold start (a few seconds) is acceptable for AI agents.

---

### Q32. How would you debug "messages pile up but consumers don't scale"?
**Answer.** Check (a) ACA scale rule configuration — correct queue name, correct trigger, max replicas not capped too low; (b) consumer is acknowledging (`complete`) messages, not abandoning; (c) lock duration vs processing time — if processing > lock, messages get redelivered; (d) dead-letter queue depth (poison messages); (e) Azure Service Bus throttling on tier (Basic/Standard/Premium).

---

## Part 6 — `erp-core` & Architecture-Specific (Senior)

### Q33. What lives in `erp-core` and what doesn't?
**Answer.** `erp-core` is **infrastructure & plumbing only** — logging, config, exceptions, middleware, messaging base classes, HTTP client, health endpoints, JWT claim helpers, constants/enums, ABC interfaces. **No business logic.** Business logic stays in each module's FastAPI service. This rule keeps the package generic and avoids coupling unrelated modules through a shared library.

---

### Q34. How does `erp_core/config.py` load secrets at runtime?
**Answer.** A Pydantic `BaseSettings` subclass reads from env vars; for secret values it resolves Key Vault references — Azure Container Apps natively injects `secretref:` values from Key Vault as env vars at container startup. Locally, `.env` files are used. Validation runs on startup so a missing key fails fast.

---

### Q35. How is a correlation ID propagated through the stack?
**Answer.**
1. Front Door / APIM injects `X-Correlation-ID` if missing.
2. `erp_core/middleware` reads it, attaches it to the request scope, and binds it to the structured logger and OpenTelemetry trace.
3. `erp_core/http_client` automatically forwards the header on outbound calls.
4. `erp_core/messaging` puts it in message headers; consumers re-bind it on receive.

Result: every log line and every span across services for one user action shares the same ID.

---

### Q36. Why publish `erp-core` and `erp-contracts` to GitHub Packages instead of vendoring?
**Answer.** Polyrepo means each service consumes the package via `pip install` with a pinned version in `pyproject.toml`. Benefits: explicit semantic versioning, Dependabot PRs notify dependents on updates, immutable wheel artefacts, no submodule pain. Breaking changes require a major version bump and a `CHANGELOG.md` entry — services upgrade on their own schedule.

---

### Q37. Walk me through how a Pact contract test works between Legal API (provider) and Legal Frontend (consumer).
**Answer.**
1. Consumer test (frontend) records its expected request/response → Pact JSON file.
2. JSON published to Pact Broker.
3. Provider CI (Legal API) replays the recorded interactions against a running instance and verifies the response matches.
4. If the provider breaks the contract, deploy is blocked. CONT-07 wires this into CI.

This shifts integration failures left, before deployment.

---

## Part 7 — AI Services, LangGraph, Azure OpenAI (Senior)

### Q38. Why one Container App per AI agent?
**Answer.** Independent scale-to-zero per workload (the cheap classification agent at min=0/max=8 is very different from the heavy generation agent at min=0/max=5), independent deploy cadence, independent failure domains, simpler cost attribution, and library independence (e.g. only Ingestion needs PyMuPDF + python-docx + openpyxl).

---

### Q39. What does the LangGraph Supervisor do that simple `if/elif` routing wouldn't?
**Answer.** LangGraph models the orchestration as a state graph with conditional edges. Benefits over hand-rolled routing: declarative DAG, built-in retries and checkpointing, supports multi-step plans (analysis → search → generation), pluggable observability via LangSmith. For simple one-hop fan-out, a switch is enough; LangGraph pays off when agents call each other or loop.

---

### Q40. How do you keep Azure OpenAI costs under control?
**Answer.**
- Use **GPT-4o-mini** for classification/triage (≈15× cheaper).
- Cache deterministic responses in **Redis** keyed by prompt hash + model version.
- Truncate / chunk inputs to the smallest necessary context.
- Track tokens per request in Redis (`AI-SRC-06`) and short-circuit when a per-tenant/per-day budget is hit.
- Stream responses so users perceive latency early — and you can cancel mid-generation.

---

### Q41. What's the right chunking strategy for embedding documents?
**Answer.** Recursive character splitter with overlap (~200 chars overlap, 800–1200 char chunks) preserves context across boundaries. Respect structural separators (paragraphs, then sentences). Store both the chunk text and metadata (doc id, page, position). Re-embed when the embedding model version changes.

---

### Q42. How would you evaluate AI agent quality?
**Answer.** Two layers:
1. **Offline:** curated golden dataset; LangSmith eval runs comparing output against reference, scored by exact match / similarity / LLM-as-judge.
2. **Online:** sample N% of prod traces in LangSmith, human review, capture user feedback (👍/👎). Track metrics: task success rate, hallucination rate, latency, tokens.

---

## Part 8 — Testing, CI/CD, Observability (Mid)

### Q43. Test pyramid for a FastAPI service?
**Answer.** Many fast unit tests on services/validators (pytest) → fewer API integration tests with `httpx.AsyncClient` and a real DB in CI → a few E2E Playwright tests through the full Front Door → APIM → API path. AI tests use LangSmith evaluations as a separate track.

---

### Q44. What tools do you use for typing and quality?
**Answer.** `mypy` or `pyright` for static typing, `ruff` for lint+format (replaces black/isort/flake8), `pytest` + `pytest-cov` (≥80% coverage gate per task), `pre-commit` to run them locally before push, GitHub Actions to enforce in CI.

---

### Q45. How do you trace one request across Frontend → APIM → FastAPI → Service Bus → AI agent?
**Answer.** OpenTelemetry SDK in every Python service exports spans to Azure Application Insights. Use W3C `traceparent` propagation in HTTP headers and message properties. APIM is configured to forward the `traceparent` header. In Application Insights, the end-to-end transaction view shows the full waterfall.

---

## Part 9 — Security & Data (Mid → Senior)

### Q46. How do you handle secrets in the codebase?
**Answer.** Never in Git. All secrets live in Azure Key Vault; ACA injects them as env vars via `secretref`. `pyproject.toml` and `.env.example` document the names only. 
Pre-commit hook (`detect-secrets` or `gitleaks`) blocks accidental commits. Key rotation is a Key Vault operation, no code change.

---

### Q47. How would you implement GDPR right-to-be-forgotten?
**Answer.**
- Tag PII fields explicitly on models (`pii=True` metadata).
- Soft-delete first (`deleted_at`), then anonymise via a scheduled Celery job that overwrites PII columns with hashed/null values while preserving aggregate data.
- Audit every deletion request (who, when, scope) in `core_audit_log`.
- Cascade across modules via a `user.anonymized` event on the `module.events` topic so each service can scrub its own data.
- Backups: documented retention so the user is fully removed within the legal window.

---

### Q48. SQL injection in an ORM context — can it still happen?
**Answer.** Yes, if you use raw SQL (`Model.objects.raw(...)`, `connection.cursor().execute(f"... {value}")`) with string interpolation. Always 
pass parameters: `cursor.execute("... WHERE id = %s", [value])`. Same for `pgvector` similarity queries.

---

## Part 10 — Soft / Architectural Judgement (Senior)

### Q49. A new module needs to ship in 2 weeks. The architecture mandates a new FastAPI service + Pydantic contracts + Pact tests. How do you scope it down without violating the architecture?
**Good signals.** Candidate proposes:
- Reuse `erp-core` boilerplate so day-1 cost is low.
- Define only the contracts actually used by the first frontend screen.
- Skip Pact for not-yet-existing consumers; add when a second consumer appears.
- Defer AI integrations behind a feature flag if they're not on the critical path.
- Schema in Django from day 1 (migrations are expensive to retrofit).

**Bad signals.** Bypass APIM, write the DB directly from FastAPI, skip tests entirely, fork `erp-core`.

---

### Q50. APIM is down. What's your incident playbook?
**Answer.** (1) Confirm via Front Door / APIM health metrics. (2) Failover route in Front Door if a secondary APIM exists. (3) Communicate via status page. (4) Internal traffic between services still works (they're on the internal ACA env, no APIM in path). (5) Post-mortem; check if APIM SKU needs upgrade or scaling.

---

### Q51. A FastAPI service can't reach the Django data layer. What do you check?
**Answer.** Internal ingress configured (`internal-only` in ACA), correct internal DNS name (`erp-django-core.<env>.internal.azurecontainerapps.io`), 
VNet integration, NSG rules, Django container is healthy (`/health` endpoint), connection pool not exhausted, Key Vault DB credential 
rotated and not picked up (restart container).

---

### Q52. Where would you push back on the architecture?
**Good signals.** Discusses real trade-offs:
- Per-schema isolation in one DB simplifies ops but couples blast radius — one bad query can lock the whole instance. Worth a read-replica for analytics.
- LangGraph adds runtime complexity that may be premature for 5 simple agents.
- Polyrepo with 17 repos has a discoverability cost; consider a workspace tool.
- pgvector is fine for MVP but recall quality on >10M vectors can degrade — plan the AI Search migration trigger explicitly.

---

## Live Coding Exercises (45–60 min each)

Pick one based on the role:

1. **Idempotent consumer.** Implement `IngestionConsumer` that subscribes to `ai.ingestion.requests`, processes each message, and is safe to redeliver. Use `erp_core.messaging` + a dedup table (or Redis set). Provide a unit test with a duplicate message.
2. **FastAPI endpoint + Pydantic v2 contract.** Build `POST /journal-entries` with double-entry validation (debit total == credit total), correct error envelope, dependency-injected DB session. Provide one happy-path and one validation-failure test.
3. **Streaming AI endpoint.** Implement an SSE endpoint that streams output from a fake `AsyncIterator[str]` representing the Generation Agent. Frontend simulator just `print`s tokens as they arrive.
4. **Repository abstraction.** Given `BaseRepository`, implement `LegalDossierRepository` against Django ORM and a fake in-memory implementation. Show how a service is unit-testable against the fake.

---

## Scoring Rubric (suggested)

| Section | Junior pass | Mid pass | Senior pass |
|---|---|---|---|
| Python core (Q1–Q12) | 8 / 12 | 10 / 12 | 12 / 12 + idiomatic |
| Async (Q13–Q16) | 1 / 4 | 3 / 4 | 4 / 4 |
| FastAPI / Pydantic (Q17–Q22) | 2 / 6 | 4 / 6 | 6 / 6 |
| Django (Q23–Q27) | 2 / 5 | 4 / 5 | 5 / 5 |
| Messaging (Q28–Q32) | 1 / 5 | 3 / 5 | 5 / 5 |
| `erp-core` & arch (Q33–Q37) | — | 3 / 5 | 5 / 5 |
| AI services (Q38–Q42) | — | 2 / 5 | 4 / 5 |
| Testing / Observability (Q43–Q45) | 1 / 3 | 2 / 3 | 3 / 3 |
| Security (Q46–Q48) | 1 / 3 | 2 / 3 | 3 / 3 |
| Judgement (Q49–Q52) | — | discussion | strong opinions, defended |

> **Tip.** Pick ~12 questions across the relevant sections per candidate. Spend the remaining time on a live-coding task.

