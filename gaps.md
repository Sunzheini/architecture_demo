# Architecture & Tasks — Gap Analysis

**Reviewed:** `architecture_specifics.md` (503 lines, 18 sections) and `Tasks.md` (562 lines, 7 phases, 576 person-days)
**Date:** April 29, 2026

The plan is unusually mature — IaC-first, contracts-driven, transactional outbox, PII redaction, AI red-teaming. Below are the gaps and inconsistencies worth addressing, grouped by severity.

---

## 🔴 High-impact gaps (likely to bite you in production)

### 1. No environment-promotion / release strategy beyond "push to main"
- CI/CD section (§12) says `main → dev`, `tag v* → staging`, manual → prod. But Tasks.md has **zero tasks** for:
  - **Database migration strategy across environments** (Django migrations on a live multi-replica ACA — who runs `migrate`? Job container? Init container? What about rollback of a bad migration?).
  - **Blue/green or canary** for ACA revisions. ACA supports traffic splitting natively; not used.
  - **Schema-breaking change protocol** (expand → migrate → contract). DB-25 catches drift but not the *process* for evolving a column without downtime.

### 2. Secret rotation & credential lifecycle
- Key Vault is mentioned ~10 times, but there's **no task** for:
  - Secret rotation policy (Postgres password, Service Bus SAS, ACR pull tokens, Pact Broker creds, GitHub Packages PAT in INFRA-25 — the PAT for local dev never expires?).
  - Managed Identity assignment per Container App (DB-24 mentions it but only for Django; what about Service Bus, Key Vault, ACR pulls?).
  - Entra ID app-registration secret expiry monitoring.
- Add: `INFRA-27 Secret rotation runbook + Key Vault rotation policies`.

### 3. Network isolation is asserted but not designed
- §15 says "services communicate within Azure VNet" and §8.3 says Azure OpenAI is on a private endpoint, but:
  - **No `modules/network/` Terraform task** in Phase 0. There's no VNet, subnet, NSG, or private DNS zone task. INFRA-03 (ACA Environment) needs a VNet input — currently undefined.
  - APIM VNet integration (INFRA-09) requires Premium tier, not Standard (which is the costed tier in Scenario 2/3). **Cost estimate is wrong** if you actually want VNet integration.
  - Private endpoints for PostgreSQL, Redis, Key Vault, Storage, Service Bus, ACR are all implied but never authored.
- Add: `INFRA-0X Author core networking module (VNet, subnets, NSGs, private DNS zones, private endpoints)` — probably a 5-day task that's currently invisible.

### 4. Cost estimates are optimistic / inconsistent
- **APIM Standard tier** is ~$700/month (~$23/day), not $7/day as listed in Scenario 2 & 3. Premium (required for VNet injection) is ~$2,800/month (~$93/day). The whole prod/scale total is understated by $20–90/day.
- **Entra ID P1** is **$6/user/month**, so 50 users = $300/month = $10/day ✅, 200 users = $1,200/month = $40/day ✅ — these are right.
- **Log Analytics**: Pay-as-you-go is **$2.76/GB**, so 10 GB/day = $27.60/day, not $23. Minor.
- **ACA AI compute** at scale (Scenario 3, 7 containers × avg 3 replicas × 1 vCPU × 10 hrs) computes to ~$30/day, not $6. The 5× understatement matters.
- **Pact Broker** (INFRA-24), **Defender for Storage** (INFRA-26), **Front Door WAF rules**, **Application Insights ingestion** are all missing or rolled into other lines.
- **Recommendation:** add a "cost assumptions & SKU pinning" subsection so tier choices (APIM Standard vs Premium, Redis Basic vs Standard vs Enterprise) are explicit per environment.

### 5. AI Orchestrator routing is described but not designed
- AI-02 ("routing rules per `EventType`") is 4 days. But the architecture never specifies:
  - **What is a "task"?** Is the orchestrator a long-running stateful workflow (LangGraph checkpointer)? Where is checkpoint state persisted? Postgres? Redis? It scales to zero — that means in-flight workflows die on idle.
  - **Retry / compensation semantics** when one agent in a multi-step workflow fails.
  - **Why is the orchestrator needed at all** for single-agent flows like Generation? Module APIs already publish to `ai.generation.requests` directly per the queue table — the Orchestrator only sits on cross-agent flows. This isn't documented.

### 6. SSE + scale-to-zero contradiction for Generation pipeline
- BE-L-13 / BE-M-10 / BE-A-14 hold an SSE connection open while the Generation Agent runs.
- Generation Agent has **min replicas = 0**. Cold start on Azure OpenAI-backed containers is typically 5–15 s.
- The user-facing `correlation_id` SSE channel will time out before the first chunk on a cold start.
- **Suggest:** either set Generation Agent min=1 in prod for the streaming path, or document a "warming up" UX state and a heartbeat strategy in CORE-20.

### 7. Search Agent — single point of failure for "interactive" workload
- AI-SRC min=1 in prod, but it owns **all** semantic search across all 3 modules. One bad deploy or one OOM = no search anywhere.
- No regional failover, no read replica strategy, no multi-AZ guarantee called out for ACA.
- **Suggest:** either keep it single-instance and document the SLO / blast radius, or shard search per module (matches the polyrepo philosophy).

---

## 🟠 Medium-impact gaps

### 8. Missing tasks (referenced but not in the table)
- **DB-XX** — referenced in CORE-17 ("outbox table migration shipped in Django (DB-XX)") but not in DB-01–DB-25. Add it explicitly.
- `documentation/api_contracts.md` — referenced from §5 and §17 but doesn't exist in the workspace and has no task to author it.
- `infrastructure.md` exists in `documentation/` — never referenced from `architecture_specifics.md` or `Tasks.md`. Either link it or align it.

### 9. Frontend gaps
- **No state management library decision**. Module Federation + JWT in memory is described, but no Redux/Zustand/TanStack Query call-out. Three teams will pick three different things.
- **No i18n task** despite Bulgarian language clearly being in scope (CORE-18 mentions BG PII, BE-A-07 mentions NAP, EGN, IBAN). The whole UI is presumably bilingual EN/BG.
- **No frontend error boundary / global error handling** task — only the Notification Center (FE-SH-09).
- **No bundle-size budget / Module Federation version-conflict CI gate** — `singleton: true` only enforces at runtime; build-time check missing.
- **FE-SH-02 design system: 5 days is unrealistic** for a usable cross-module library (typography scale, dark mode, RTL?, BG/EN locale, accessibility, theming tokens, Storybook). Realistic: 10–15 days.

### 10. QA / Testing gaps
- **No chaos / failure-mode testing** (kill a Service Bus subscriber mid-message; corrupt an outbox row; slowloris an SSE endpoint).
- **No DB performance/index regression test** despite Django being the sole DB gateway — a missing index on a single internal endpoint becomes a system-wide P95 problem.
- **No data-migration test** between `pgvector` (MVP) and Azure AI Search (Scale) — §8.4 mentions the migration; nobody owns it.
- **No accessibility automation in CI** (QA-12 is one-shot). Lighthouse / axe should run on every FE PR.
- **QA-11 OWASP ZAP** run once is not enough; should be in CI nightly.
- Coverage gate (80%) is not enforced anywhere — no CI step task. Add to INFRA-14 template.

### 11. Observability gaps
- **No SLO / SLI definition** anywhere. Alerts (PROD-03) fire on "thresholds" but thresholds aren't defined.
- **No cost observability** beyond budget alerts — missing per-service cost attribution (tag-based) so you can tell which agent burns OpenAI tokens.
- **No log retention / PII-in-logs policy.** Log Analytics retention default is 30 days; GDPR + the PII redaction effort suggests you need an explicit retention + redaction policy on logs too.
- **No on-call rotation, paging integration, or incident-response tooling task** (just "write runbooks" in PROD-07).

### 12. Data-layer architecture risk
- Django as the sole DB gateway via `/internal/v1/<schema>/<resource>` is a **choke point**:
  - Every FastAPI request → HTTP hop → Django → ORM → DB. That's at least +20–50 ms p50 latency vs direct DB.
  - DB-16 acceptance criterion is "p95 < 100 ms for single-record reads" — that leaves no budget for a real query (joins, list endpoints, financial statement aggregations).
  - **Bulk operations** (BE-M-07 ETL, BE-A-06 Balance Sheet across a year of journal lines): doing this through HTTP-RPC to Django is going to be painful and chatty. There's no batching/streaming contract.
  - Also: Django becomes the bottleneck for *all* writes, but its scale ceiling is min=1, max=6 vs FastAPI services at max=10 each (3 services × 10 = 30 callers → 6 Django replicas).
- **Suggest:** either elevate this trade-off in §3 (acknowledging the latency tax + scale ceiling), or carve out an exception list (read-only reporting endpoints can hit a Postgres read replica directly via SQLAlchemy core).

### 13. Inter-service communication — unspecified pieces
- `module.events` topic is listed but no task defines its schema, ownership, or any subscribers. It's currently "build it because §9 says so."
- No **event versioning** strategy (events live forever; consumers lag producers).
- No **schema registry** for Service Bus messages (just Pydantic models in `erp-contracts`, but no runtime check that what's on the wire matches).

### 14. AI cost & quality controls
- **No per-tenant / per-user AI quota** task. Token tracking exists in AI-SRC-06 but only for Search.
- **No prompt versioning** (AI-GEN-06 says "manage prompt templates" — version control? A/B testing? Rollback if a new prompt regresses LangSmith eval scores?).
- **No model fallback / Azure OpenAI quota-exhaustion handling** (when GPT-4o returns 429, do agents fall back to GPT-4o-mini? Queue? Fail?).

### 15. Definition of Done coverage gap
- DoD demands 80% coverage and contract tests on **every task**, but ~130 tasks have no per-task acceptance criteria. PROD-09 (5 days) plans to "author + maintain" them — this is wildly underestimated and pushed to the very end. Move PROD-09 acceptance-criteria authoring **into each phase**, owned by the phase lead, before work starts.

---

## 🟡 Lower-impact / housekeeping

### 16. Internal inconsistencies
- §3 "FastAPI services must not write directly to the DB — they interact via Django's internal service layer **or the shared `erp-core` Python package**" — `erp-core` explicitly forbids business logic (§4 rule). Drop the second clause to remove ambiguity.
- §5 vs §11: §5 says services use Service Bus; §11 has Search via sync HTTP. Both correct — but §5 doesn't note Search is the exception.
- Team in §1: "Evsin" is likely "Evgeni" (typo).
- Cost Scenario 2 lists Entra ID P1 only in scenarios 2 & 3, but auth via Entra ID is required from day one. P1 features (Conditional Access for MFA, Group-based RBAC) are needed before prod — the MVP scenario should at least mention "Entra ID Free for dev only."
- §6 says "Multi-AZ for HA" but cost scenario 1 says "no HA" for the same DB. Clarify it's a per-env setting.
- Section numbering in `architecture_specifics.md` jumps from §11 to §12 without a `---` after §11 (cosmetic).

### 17. Estimates that look optimistic

| Task | Estimate | Concern |
|---|---|---|
| FE-SH-02 design system | 5 d | Realistic 10–15 d (see §9 above) |
| INFRA-09 APIM | 4 d | JWT validation + routing + VNet + rate limit + 4 backends + IaC = 7–8 d |
| AI-ING-02 file parsing | 5 d | OCR fallbacks, malformed PDFs, encrypted DOCX, password-protected files — easily 8–10 d |
| BE-A-06 financial statements | 5 d | BG accounting standards + PDF generation + reconciliation testing = 8–12 d |
| BE-A-07 VAT/tax | 4 d | NAP XML schema validation + Bulgarian tax rules — likely 8–10 d |
| QA-08 load test | 4 d | k6 scripts for 11 services, scenario design, baseline + threshold validation = 6–8 d |
| PROD-09 author AC for 130 tasks | 5 d | Should be distributed across phases, not 5 days at end |

Add ~25–35 person-days buffer or re-estimate.

### 18. Polyrepo overhead not budgeted
- 18 repos × per-repo: branch protection rules, CODEOWNERS, PR template, Dependabot config, label scheme, release-please config, SECURITY.md, GH Packages auth.
- INFRA-25 covers GH Packages auth; everything else is implicit. Add `INFRA-28 Repo bootstrap automation` — generate new-repo template via a `scaffold` action so the 18-repo overhead doesn't multiply manual work.

### 19. Missing artifacts referenced in plan

| Reference | Where mentioned | Status |
|---|---|---|
| `documentation/api_contracts.md` | §5, §17 | Doesn't exist |
| `CHANGELOG.md` (erp-core) | §4 | No task to maintain it |
| C4 diagrams | §17 | No task |
| ADR directory `/docs/adr` | §17 | No task |
| `infra/image-versions/<env>/` | PROD-04 + Production DoD | Format / schema undefined |
| `tests/ai_redteam/` | QA-14 | Owner unspecified — QA or Backend? |

---

## ✅ Recommended next steps (in priority order)

1. **Add a networking module + revisit cost** (§3 + §4 above). This is the biggest hidden cost and timeline risk.
2. **Re-validate Django-as-DB-gateway** for hot paths (§12). Either accept the latency/scale cost explicitly or carve exceptions.
3. **Define DB migration & ACA release strategy** (revisions, traffic splitting, init jobs) — add 2–3 tasks to Phase 0.
4. **Distribute PROD-09 acceptance-criteria authoring** into each phase (per-task AC before work starts).
5. **Add missing tasks**: networking, secret rotation, repo scaffolding template, `module.events` ownership, prompt versioning, AI quota/per-tenant limits, frontend i18n, frontend state management decision.
6. **Re-estimate the 7 underestimated tasks** in §17 and add a 15% buffer to the 576-day total → realistic ≈ 660–700 person-days.
7. **Fix doc inconsistencies** (§16) and create the missing artifacts (§19) — quick wins.

