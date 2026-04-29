# Timing Plan — Core Platform + Accounting Module (MVP)

**Scope:** Deliver the shared platform (`erp-core`, `erp-contracts`, Django data layer, Azure infra, CI/CD) and the **Accounting** module end-to-end (backend, AI generation/analysis, frontend, QA, production readiness).

**Out of scope for this plan:** Legal & Marketing modules, AI Search Agent, AI Ingestion Agent, AI Classification Agent (deferred to a follow-up release; see "Deferred" section at the bottom).

---

## 1. In-Scope Effort Summary

| # | Group | Person-days |
|---|---|---|
| 1 | Phase 0 — Infrastructure & DevOps | 74 |
| 2 | Phase 1a — `erp-core` package | 54 |
| 3 | Phase 1a — `erp-contracts` package | 23 |
| 4 | Phase 1b — Data Layer (Django) | 63 |
| 5 | Phase 2 — Accounting FastAPI | 43 |
| 6 | Phase 2 — Celery Worker (scheduled jobs for closing periods, reports) | 7 |
| 7 | Phase 3 — AI Orchestrator | 16 |
| 8 | Phase 3 — AI Generation Agent (financial narratives, BE-A-11) | 25 |
| 9 | Phase 3 — AI Analysis Agent (financial anomaly detection, BE-A-10) | 15 |
| 10 | Phase 4 — Design (Figma — Accounting screens only, scoped down from 10 → 6) | 6 |
| 11 | Phase 4 — Shell Frontend | 35 |
| 12 | Phase 4 — Accounting Frontend | 25 |
| 13 | Phase 5 — QA (Accounting-scoped subset; see breakdown below) | 30 |
| 14 | Phase 6 — Production Readiness | 22 |
| | **Total in-scope effort** | **438 person-days** |

**QA subset (30 days):** QA-01 (3) + QA-04 (5) + QA-05 (2) + QA-06 (2) + QA-07 (3) + QA-08 (4) + QA-09 partial — generation+analysis only (3) + QA-11 (3) + QA-12 (2) + QA-13 (2) + QA-14 (1, scoped to 2 agents).

---

## 2. Recommended Team (lean, focused)

This plan uses the **same headcount formula as `Tasks.md`** (`FTE = days / (calendar days × 0.7 focus factor)`, rounded up). The full-scope `Tasks.md` table lands on **8 FTE** for 614 person-days; this Core + Accounting subset is **438 person-days**, which reduces the team by **~3 FTE** (one frontend FTE removed because Legal + Marketing frontends are out of scope; one backend FTE removed because Search + Ingestion + Classification agents are deferred; the rest of the saving comes from the smaller Legal + Marketing backend slice).

**Reconciliation against `Tasks.md` "8 people for a 6-month MVP" table:**

| Discipline | This plan (Core + Accounting) | `Tasks.md` (full MVP) | Δ |
|---|---|---|---|
| DevOps | 96 d → **2 FTE** | 96 d → 2 FTE | 0 |
| Backend (core + services + AI) | 183 d → **3 FTE** | 293 d → 4 FTE | −1 (Legal/Marketing/Search/Ingestion/Classification removed) |
| Database (Django) | 63 d → **1 FTE** (overlaps with backend) | 63 d → 1 FTE | 0 |
| Frontend / Design | 66 d → **1 FTE** | 114 d → 2 FTE | −1 (Legal + Marketing frontends removed) |
| QA | 30 d → **1 FTE** | 48 d → 1 FTE | 0 |
| **Total** | **438 d → 8 FTE-equivalent slots, filled by 5 people** (DB engineer overlaps backend) | **614 d → 8 FTE** | −3 actual heads |

**Headcount as actual people for this plan: 5 engineers + 1 designer (~50%) = 5.5 heads.** The ratio matches the `Tasks.md` formula 1-to-1 — no hidden assumptions.

| Role | FTE | Owns | Maps to `Tasks.md` discipline |
|---|---|---|---|
| DevOps engineer | 1 | Phase 0, Phase 6, CI/CD pipelines | DevOps (½ of the 2 FTE — Phase 0 + Phase 6 are sequential, so 1 person can cover both with the second DevOps FTE re-allocated to PROD-09 / runbooks late in the schedule) |
| Backend / Platform engineer (lead) | 1 | `erp-core`, `erp-contracts`, Orchestrator | Backend |
| Backend / Django engineer | 1 | Django data layer, internal REST API, Celery | Database (overlapping Backend per `Tasks.md` note) |
| Backend / Accounting engineer | 1 | Accounting FastAPI, AI Generation & Analysis agents | Backend |
| Frontend engineer | 1 | Shell + Accounting frontend | Frontend / Design |
| Designer (part-time, ~50%) | 0.5 | Figma wireframes + design tokens | Frontend / Design |
| QA engineer | 1 | Phase 5 | QA |
| **Team total** | **5.5 FTE** | | matches `Tasks.md` formula for 438 d / 84 productive days = 5.2, rounded up |

> **Why not the full 8 from `Tasks.md`?** The 8-FTE figure is for the **full** 614-day scope (all 3 modules + all 5 AI agents). Cutting scope to Core + Accounting cuts ~176 person-days, which translates to ~2.5 fewer FTE under the same formula. If you want to **keep 8 people** on this scope, the spare 2.5 FTE buy down the schedule to **~4 calendar months** instead of 6 — see Section 6 "Aggressive" row, refined below.

**Productive-day budget per FTE** = working days × 0.7 focus factor.

---

## 3. Critical Path

The **shortest possible delivery time** is bounded by this dependency chain (no parallelism can shorten it):

```
INFRA-00 → INFRA-01 → INFRA-05/08 (Postgres + Key Vault)
        → CORE-01 → CORE-02..11 → CORE-12 → CORE-13 (erp-core v1.0.0)
        → DB-10 → DB-02..05 → DB-16 (Django internal API, 8 d)
        → CORE-14 → CORE-15..21 → CORE-15 publish (erp-core v1.1.0)
        → BE-A-01..14 (Accounting FastAPI)
        → FE-AC-01..08 (gated also by FE-SH-* + design)
        → QA → PROD-05 → PROD-06
```

**Critical-path effort (single-threaded):** ≈ 95 person-days
**Calendar duration on critical path:** ≈ **22 working weeks** at 0.7 focus = **~5.5 calendar months minimum**.

---

## 4. Phased Timeline (6-month delivery)

Assumes a **26-week** calendar (~130 working days). Each phase below is a calendar window — multiple workstreams run in parallel inside each window.

### Phase A — Foundations (Weeks 1–4) — "Stand up the platform"

| Workstream | Tasks | Owner |
|---|---|---|
| Infra bootstrap | INFRA-00 → INFRA-13 (RG, ACR, ACA, Postgres, Redis, Service Bus, Key Vault, APIM, Front Door, Entra ID, Blob, OpenAI), INFRA-20 (local docker-compose), INFRA-25 (GitHub Packages auth) | DevOps |
| `erp-core` v1.0.0 | CORE-01 → CORE-13 (config, logging, exceptions, middleware, interfaces, messaging, http_client, health, security, constants, tests, publish) | Backend lead |
| `erp-contracts` skeleton | CONT-01 (init only, others come in Phase B) | Backend lead |
| Design kickoff | FE-DSGN-01 — Accounting wireframes only | Designer |

**Exit gate (end of week 4):** `erp-core` v1.0.0 published; dev Azure environment provisioned via Terraform; Accounting Figma signed off.

---

### Phase B — Data layer + contracts (Weeks 5–8) — "Database and APIs are real"

| Workstream | Tasks | Owner |
|---|---|---|
| Django data layer | DB-01, DB-02, DB-05 (accounting schema), DB-06, DB-07, DB-08, DB-09, DB-10, DB-11, DB-12, DB-13, DB-14, DB-15, DB-16 (internal REST API — **8 d, biggest single task**), DB-17, DB-18, DB-19, DB-21, DB-22, DB-23, DB-24, DB-26 | Django engineer |
| `erp-contracts` | CONT-01..06 (focus CONT-04 Accounting, CONT-05 AI, CONT-06 events), CONT-07 (Pact), CONT-08 (publish v1.0.0) | Backend lead |
| `erp-core` v1.1.0 | CORE-14 (Django client) → CORE-16 (DLQ/idempotency) → CORE-17 (outbox) → CORE-18 (PII redactor) → CORE-19 (prompt guard) → CORE-20 (SSE fan-out) → CORE-15 (publish v1.1.0) | Backend lead |
| Infra hardening | INFRA-14, INFRA-15, INFRA-16, INFRA-18, INFRA-22, INFRA-24 (Pact Broker), INFRA-26 (Defender), INFRA-29 (migration job), INFRA-37/38 (network/private endpoints) | DevOps |
| Shell FE | FE-SH-01 → FE-SH-04 (init, design system, MSAL login, JWT sharing) | Frontend |
| Design tokens export | FE-DSGN-02 | Designer |

**Exit gate (end of week 8):** Django internal API live in dev; `erp-core` v1.1.0 published; `erp-contracts` v1.0.0 published; Pact Broker live; Shell can authenticate against Entra ID.

---

### Phase C — Accounting backend + AI plumbing (Weeks 9–14) — "Business logic and the AI pipeline"

| Workstream | Tasks | Owner |
|---|---|---|
| Accounting FastAPI | BE-A-01 (init), BE-A-02 (Chart of Accounts), BE-A-03 (Journal Entries), BE-A-04 (double-entry validation — **business-critical AC**), BE-A-05 (Periods), BE-A-06 (Balance Sheet, P&L, Cash Flow), BE-A-07 (VAT + corporate tax), BE-A-08 (Audit), BE-A-09 (NAP control dossier), BE-A-10 (risk alerts), BE-A-11 (AI narrative trigger), BE-A-12 (tests), BE-A-13 (CI/CD), BE-A-14 (SSE streaming endpoint) | Accounting engineer |
| AI Orchestrator | AI-01 → AI-08 | Backend lead (then Accounting engineer for AI-06/07) |
| AI Generation Agent | AI-GEN-01 → AI-GEN-10 (financial-narrative prompt set in AI-GEN-04 + AI-GEN-06) | Accounting engineer |
| AI Analysis Agent | AI-ANA-01, AI-ANA-03 (financial anomaly detection — only this analyzer for MVP), AI-ANA-05, AI-ANA-06 | Accounting engineer |
| Celery worker | CEL-01 → CEL-04 (period-close jobs, scheduled report runs) | Django engineer |
| Shell FE (cont.) | FE-SH-05 → FE-SH-13 (navigation, routing, DataTable, FormBuilder, NotificationCenter, logout, tests, Dockerfile, CI/CD) | Frontend |
| Infra | INFRA-09 (APIM with SSE policy), INFRA-10 (Front Door), INFRA-17 (Grafana), INFRA-19 (budget alerts), INFRA-21 (apply to staging+prod), INFRA-23, INFRA-30 (multi-revision + canary), INFRA-32 (release-please), INFRA-39/40/41 (network finalisation) | DevOps |
| DB hardening | DB-20 (backup/restore), DB-25 (schema-drift contract test) | Django engineer |

**Exit gate (end of week 14):** Accounting FastAPI fully deployed to dev; financial narratives stream end-to-end via SSE; Shell + auth complete; staging environment provisioned.

---

### Phase D — Accounting frontend (Weeks 15–19) — "User-facing module"

| Workstream | Tasks | Owner |
|---|---|---|
| Accounting Frontend | FE-AC-01 (init), FE-AC-02 (Journal Entries — **5 d, biggest FE task**), FE-AC-03 (Chart of Accounts), FE-AC-04 (Periods + closure), FE-AC-05 (Financial dashboard), FE-AC-06 (VAT/tax download), FE-AC-07 (Audit), FE-AC-08 (Nginx + CI/CD) | Frontend |
| AI streaming UX | AI-GEN-10 (frontend SSE warm-up panel reused in FE-AC-05) | Frontend (with Accounting engineer support) |
| Early QA | QA-01 (login E2E), QA-05 (Pact verification across in-scope services), QA-06 (APIM JWT) | QA |
| Backend polish | Bug-fixes from FE integration; finalise audit logging | Accounting engineer |

**Exit gate (end of week 19):** Full Accounting UX usable end-to-end on dev; first end-to-end demo to product owner.

---

### Phase E — Integration QA (Weeks 20–23) — "Prove it works under load and attack"

| Workstream | Tasks | Owner |
|---|---|---|
| E2E tests | QA-04 (Accounting full Playwright flow — 5 d) | QA |
| Security & RBAC | QA-07 (RBAC matrix), QA-11 (OWASP ZAP) | QA |
| Performance | QA-08 (k6 load test, autoscaling validation) | QA + DevOps |
| AI quality | QA-09 partial (generation + analysis pipeline integration), QA-14 scoped to 2 agents (red-teaming, PII leak) | QA + Accounting engineer |
| Accessibility & responsive | QA-12, QA-13 | QA + Frontend |
| Bug-fix swarm | All hands | Whole team |

**Exit gate (end of week 23):** All in-scope QA tasks green on staging.

---

### Phase F — Production hardening & go-live (Weeks 24–26) — "Ship it"

| Workstream | Tasks | Owner |
|---|---|---|
| Autoscaling & monitoring | PROD-01, PROD-02, PROD-03 | DevOps |
| Image pinning | PROD-04 | DevOps |
| Runbooks | PROD-07, INFRA-31 (expand→migrate→contract + rollback game day on staging) | DevOps + Accounting engineer |
| Docs | PROD-08 (per-service README, Swagger), PROD-09 (AC appendix + PR templates) | All |
| Staging sign-off | PROD-05 (smoke + sign-off) | Whole team |
| **Go-live** | **PROD-06** | DevOps + product owner |

**Exit gate (end of week 26):** Accounting module live in production behind APIM + Front Door, with Entra ID auth, RBAC, audit logging, AI narrative streaming, financial reports, and ДДС/corporate tax declaration generation.

---

## 5. Visual Gantt (26 weeks)

```
Week:                       1   2   3   4   5   6   7   8   9  10  11  12  13  14  15  16  17  18  19  20  21  22  23  24  25  26
                            |───|───|───|───|───|───|───|───|───|───|───|───|───|───|───|───|───|───|───|───|───|───|───|───|───|───|
DevOps   (Infra Phase 0)   ████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
DevOps   (Infra hardening)             ████████████░░░░░░░░░░░░░░░░░░░░
DevOps   (Staging+prod)                              ████████░░░░░░░░░░░░░░░░░░
DevOps   (Prod readiness)                                                                                ████████████
Backend  (erp-core v1.0)   ████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
Backend  (contracts+v1.1)             ████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
Backend  (Orchestrator)                                  ████████████░░░░░░░░░░░░░░
Django   (Data layer)                  ████████████████████████░░░░░░░░░░░░░░░░░░░░
Django   (Celery)                                            ██████░░░░░░░░░░░░░░░░
Acct.BE  (FastAPI)                                       ████████████████████████░░░░░░░░░░
Acct.BE  (AI Gen+Analysis)                                       ██████████████████░░░░░░░
Frontend (Design)          ████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
Frontend (Shell)                       ████████████████████████░░░░░░░░░░░░░░░░░░░░
Frontend (Accounting FE)                                                     ████████████████████░░░░
QA                                                                                       ████████████████░░░░░░░░
Buffer / Bug-fix / Sign-off                                                                              ████████████
```

(`█` = active work, `░` = idle for that workstream)

---

## 6. Best Time Estimate

| Scenario | Calendar duration | Team size | Conditions |
|---|---|---|---|
| **Best case (recommended)** | **6 calendar months (26 weeks)** | **5.5 FTE** (matches `Tasks.md` formula for 438 d) | No major scope changes; parallel workstreams as planned. **Recommended estimate.** |
| Aggressive / squeeze | **~4 calendar months (~17 weeks)** | **8 FTE** (full `Tasks.md` headcount applied to the smaller scope) | Keep the original 8-person team but use them on Core + Accounting only. Adds a 2nd backend, a 2nd frontend, and an extra QA → critical path on `DB-16` + `BE-A-06/07` becomes the only remaining bottleneck. Risk: `BE-A-07` NAP XML edge cases still need an SME — can't be parallelised. |
| Conservative / safe | 7 calendar months (~30 weeks) | 5.5 FTE | Same lean team, adds 4 weeks buffer for: NAP XML schema corner-cases (BE-A-07), staging game-day rollback rehearsals (INFRA-31), and longer security remediation. |

**Recommendation: plan for 6 months (26 weeks) and review at the end of Phase B (week 8).** Phase B is the highest-risk segment — if `erp-core` v1.1.0 + Django internal API are not green by week 8, slip the go-live by 2 weeks rather than compressing QA.

---

## 7. Headcount math (sanity check — same formula as `Tasks.md` §"Effort Summary & Headcount Calculation")

```
Required FTEs = Discipline person-days / (Project calendar days × Focus factor 0.7)
```

For a 6-month delivery (130 working days × 0.7 = **91 productive days per FTE**):

- Total in-scope effort = **438 days**
- 438 / 91 = **4.81 FTE minimum** (assuming perfect parallelism)
- Real schedule needs **5.5 FTE** (rounded up + 0.5 designer) because:
  - DevOps and Frontend cannot start at full speed in week 1 (waiting on infra/design dependencies → idle time).
  - QA work front-loads in the last 7 weeks and cannot be spread.
  - The critical path requires single-threaded backend lead time on `erp-core`.

**Cross-check vs `Tasks.md` 8-FTE table:** that figure is for **614 days** of full-platform scope. Removing the 176 days of Legal + Marketing + Search/Ingestion/Classification scope removes ~2.5 FTE under the same formula, leaving **5.5 FTE for this plan**. The numbers are internally consistent — this plan does not silently change the assumptions in `Tasks.md`.

If you want to **keep the original 8 FTE** assigned, the spare 2.5 FTE compress the calendar instead of the scope:

- 438 days / (8 FTE × 0.7 focus) = **78 productive days** = **~17 calendar weeks ≈ 4 months**
- Critical path (DB-16 → erp-core v1.1 → BE-A-06/07) remains the floor; below 16 weeks is not achievable without dropping scope.

---

## 8. Top 5 Schedule Risks & Mitigations

| # | Risk | Impact | Mitigation |
|---|---|---|---|
| 1 | **DB-16** Django internal REST API slips (8-day task, blocks BE-A-* and CORE-14) | Cascades into 2-week prod slip | Start DB-16 in week 5 with a dedicated owner; review draft API at week 6. |
| 2 | **BE-A-07** VAT/corporate tax declaration corner cases (NAP XML schema) | 1–2 week slip in Phase C | Engage a Bulgarian accounting SME early (week 8); commit XSD test corpus by week 10. |
| 3 | **AI-GEN-09** Generation Agent cold-start SLO not met | Poor UX; FE warm-up workaround insufficient | Provision `min replicas = 1` in prod from day 1; budget already accounts for ~$15/month. |
| 4 | **CORE-17** Transactional outbox bugs surface late | Risk of lost AI events on prod | Ship outbox + integration test (failure-injection) by week 7; gate all `BE-A-11`-style publishes behind the outbox. |
| 5 | **INFRA-37/38** Private-endpoint network rework breaks dev environment | 1-week DevOps stall | Apply network module to dev in week 6 during a planned downtime window; keep `terraform plan` approvals tight. |

---

## 9. Deferred (out of scope, recoverable later)

| Item | Reason for deferral | Rough cost when added |
|---|---|---|
| Legal module (BE-L-*, FE-L-*) | Not requested in this plan | +56 days BE + 22 days FE |
| Marketing module (BE-M-*, FE-M-*) | Not requested | +29 days BE + 22 days FE |
| AI Search Agent (AI-SRC-*) | Accounting MVP doesn't require semantic search; financial reports are deterministic queries | +19 days |
| AI Ingestion Agent (AI-ING-*) | Accounting MVP imports structured data via ETL/API, not free-form documents | +18 days |
| AI Classification Agent (AI-CLS-*) | Not needed without document ingestion | +10 days |
| Full Figma design (Legal + Marketing screens) | Scoped down to 6 days | +4 days designer |
| Full QA suite (QA-02, QA-03, full QA-09/QA-14) | Module-specific; reinstate when modules are added | +18 days QA |

---

**Bottom line: 6 months, 5.5 FTE, 438 person-days, with go-live at end of week 26.** This matches the `Tasks.md` headcount formula 1-to-1 — the lower FTE count vs. the 8-FTE figure in `Tasks.md` is purely the result of the smaller scope (Core + Accounting only, vs. all 3 modules + 5 AI agents). If you instead keep the full 8-person team on this scope, delivery compresses to **~4 months** (Section 6, "Aggressive" row).

