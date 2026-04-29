# Timing Plan — Core Platform + Accounting Module (MVP)

**Scope:** Deliver the shared platform (`erp-core`, `erp-contracts`, Django data layer, Azure infra, CI/CD) and the **Accounting** module end-to-end (backend, AI Generation/Analysis agents, frontend, QA, production readiness).

**Out of scope (deferred — see Section 9):** Legal & Marketing modules, AI Search / Ingestion / Classification agents.

**Team:** 11 named people from `people.md`, ramping **7 → 9 → 10 FTE** over the project.

**Headline estimate:** **15 calendar weeks (~3.5 months)** to production go-live, with Dimitar (joins week 14) effectively acting as schedule insurance.

> **Why 15 not 14 weeks?** The earlier "~14 weeks" figure used *average* FTE. The actual `people.md` ramp delivers only 413 productive days by end of week 14 — 25 days short of the 440 needed. Week 15 lifts capacity to 444 d (just enough). Week 16 gives a comfortable 9% buffer.

---

## 1. In-Scope Effort

| # | Group | Person-days |
|---|---|---|
| 1 | Phase 0 — Infrastructure & DevOps | 74 |
| 2 | Phase 1a — `erp-core` package | 54 |
| 3 | Phase 1a — `erp-contracts` package | 23 |
| 4 | Phase 1b — Data Layer (Django) | 63 |
| 5 | Phase 2 — Accounting FastAPI | 43 |
| 6 | Phase 2 — Celery Worker | 7 |
| 7 | Phase 3 — AI Orchestrator | 16 |
| 8 | Phase 3 — AI Generation Agent (financial narratives) | 25 |
| 9 | Phase 3 — AI Analysis Agent (financial anomaly detection) | 15 |
| 10 | Phase 4 — Design (FE-DSGN-01/02 absorbed by FEs, +2 d overhead) | **8** |
| 11 | Phase 4 — Shell Frontend | 35 |
| 12 | Phase 4 — Accounting Frontend | 25 |
| 13 | Phase 5 — QA (Accounting-scoped subset) | 30 |
| 14 | Phase 6 — Production Readiness | 22 |
| | **Total** | **440 person-days** |

**QA subset (30 d):** QA-01 (3) + QA-04 (5) + QA-05 (2) + QA-06 (2) + QA-07 (3) + QA-08 (4) + QA-09 partial (3) + QA-11 (3) + QA-12 (2) + QA-13 (2) + QA-14 scoped (1).

---

## 2. Team & FTE Ramp (from `people.md`)

| Person | Skills | FTE | Available | Primary role in this plan |
|---|---|---|---|---|
| **Dimo** | DevOps, Backend | 1.0 | Day 1 | DevOps lead — Phase 0 + CI/CD + APIM/FrontDoor |
| **Ivan** | Backend, Database | 1.0 | Day 1 | Django data layer owner — DB-01..26 + internal REST API |
| **Veronika** | Frontend, Backend | 1.0 | Day 1 | Floats: Backend in Phase A/B (`erp-core` v1.0 + Orchestrator), Frontend from Phase C onwards |
| **Evgeni** | Backend | 1.0 | Day 1 | Backend lead — `erp-core` v1.1 (DLQ, outbox, PII, prompt-guard, SSE) + AI Orchestrator + AI Generation |
| **Evsin** | Backend | 0.5 | Day 1 | `erp-contracts` package, then AI Analysis Agent (anomaly detection) |
| **Denis** | Frontend | 1.0 | Day 1 | Shell Frontend (FE-SH-01..13) + Accounting FE |
| **Yoana** | Frontend | 0.5 | Day 1 | Wireframes (FE-DSGN-01) + design tokens + Accounting FE pages |
| **QA Person** | QA | 1.0 | Day 1 | Phase 5 prep from week 1, full Phase 5 from week 11 |
| **Martin** | Backend, DevOps | 1.0 | **Day +30 (wk 5)** | Second DevOps in Phase B (network/private endpoints, INFRA-37/38/41), then Backend (Accounting BE extended) |
| **Nikolay** | Backend | 1.0 | **Day +30 (wk 5)** | Accounting BE engineer A (BE-A-01..06: dossier, journal entries, validation, periods, statements) |
| **Dimitar** | Backend | 1.0 | **Day +90 (wk 14)** | **Insurance.** Joins for Phase E bug-fix swarm + production hardening; if all green, freed for the *next* module |

**FTE ramp summary:**

| Window | FTE | Cumulative productive days |
|---|---|---|
| Weeks 1–4 (day 1) | **7.0** | 98 |
| Weeks 5–13 (Martin + Nikolay land) | **9.0** | 381.5 |
| Weeks 14–15 (still 9; Dimitar onboarding) | **9.0** | **444** |
| Week 16 (safety variant) | **10.0** | 476 |

> **All 8-FTE roles from `Tasks.md` §"Effort Summary" are covered:** DevOps × 2 (Dimo + Martin), Backend × 4 (Evgeni + Veronika BE-mode + Nikolay + Martin BE-mode + Evsin 0.5 + Dimitar buffer), Database × 1 (Ivan, overlapping Backend), Frontend × 2 (Denis + Yoana 0.5 + Veronika FE-mode), QA × 1 (QA Person). Designer role absorbed by FEs using a stock component library (Material UI / Chakra / Tailwind UI).

---

## 3. Critical Path

```
Wk 1–2  INFRA-00 → INFRA-01 → INFRA-05/08 (Postgres + Key Vault)         [Dimo]
Wk 1–3  CORE-01 → CORE-02..11 → CORE-12 → CORE-13 (erp-core v1.0.0)      [Evgeni + Veronika]
Wk 3–5  DB-10 → DB-02..05 → DB-16 (Django internal API, 8 d)              [Ivan]      ← bottleneck
Wk 5–7  CORE-14..21 → CORE-15 publish (erp-core v1.1.0)                   [Evgeni]
Wk 7–11 BE-A-01..14 (Accounting FastAPI, 2-way split)                     [Nikolay + Martin/Veronika]
Wk 11–13 FE-AC-01..08 (Accounting FE, against real BE + Pact stubs)       [Denis + Yoana + Veronika]
Wk 13–15 QA → PROD-05 → PROD-06                                           [QA Person + all hands]
```

**Critical-path floor: ~14 weeks.** 15 weeks is the realistic schedule (1 week overhead from FTE ramp).

---

## 4. Phased Timeline (15 weeks, with safety overflow into week 16)

### Phase A — Foundations (Weeks 1–3) — "Stand up the platform"

| Workstream | Tasks | Owner |
|---|---|---|
| Infra bootstrap | INFRA-00 → INFRA-13, INFRA-20, INFRA-25 | **Dimo** |
| `erp-core` v1.0.0 split | CORE-01..06 (config / logging / exceptions / middleware / interfaces) | **Evgeni** |
| `erp-core` v1.0.0 split | CORE-07..12 (messaging, http_client, health, security, constants, tests) → CORE-13 publish | **Veronika** (BE-mode) |
| `erp-contracts` skeleton + Accounting DTOs | CONT-01, CONT-04 (Accounting), CONT-05 (AI), CONT-06 (events) | **Evsin (0.5)** |
| Django bootstrap | DB-01 (PostgreSQL + pgvector), DB-02 (`core` schema), DB-05 (`accounting` schema), DB-06..09, DB-10 (Django project init) | **Ivan** |
| Wireframes + design tokens | FE-DSGN-01 (wireframes against `Tasks.md` page descriptions, using stock component library), FE-DSGN-02 (tokens picked from library) | **Yoana (0.5)** |
| Shell scaffolding | FE-SH-01 (Module Federation host), FE-SH-03 (MSAL.js login), FE-SH-04 (JWT sharing) — no design dep | **Denis** |
| QA prep | Pact Broker familiarisation, Playwright skeleton, k6 + ZAP setup | **QA Person** |

**Exit gate (end of week 3):** `erp-core` v1.0.0 published; dev Azure environment provisioned via Terraform; `accounting` Django schema in place; Shell can authenticate against Entra ID dev tenant; wireframes signed off.

---

### Phase B — Data layer + contracts + early Accounting BE (Weeks 4–7) — "Database and APIs are real"

**Team grows to 9 FTE: Martin + Nikolay arrive at start of week 5.**

| Workstream | Tasks | Owner |
|---|---|---|
| Django internal REST API | DB-11..22, DB-24, DB-26 — focus DB-16 (**8 d, biggest single task**) | **Ivan** |
| `erp-core` v1.1.0 | CORE-14 (Django client) → CORE-16 (DLQ/idempotency) → CORE-17 (outbox) → CORE-18 (PII redactor) → CORE-19 (prompt guard) → CORE-20 (SSE fan-out) → CORE-15 publish | **Evgeni** |
| `erp-contracts` finalise | CONT-07 (Pact), CONT-08 (publish v1.0.0), CONT-09 (`@erp/contracts-ts` npm package) | **Evsin (0.5)** + **Veronika** (BE-mode helps wrap up) |
| AI Orchestrator (early) | AI-01, AI-02 (routing) — runs ahead of agents using mocked workers | **Veronika** (transitions wk 6) |
| Accounting BE (early start, wk 6 once `erp-core` v1.1 ready) | BE-A-01 (init), BE-A-02 (Chart of Accounts), BE-A-03 (Journal Entries) | **Nikolay** |
| Infra hardening | INFRA-14, INFRA-15, INFRA-16, INFRA-18, INFRA-22, INFRA-24 (Pact Broker), INFRA-26 (Defender), INFRA-29 (migration job) | **Dimo** |
| Networking (private endpoints) | INFRA-37, INFRA-38, INFRA-39 (ACA in VNet), INFRA-40 (APIM internal), INFRA-41 (validation) | **Martin** (DevOps mode wk 5–7) |
| Shell FE — design system + nav | FE-SH-02 (design system from tokens — wk 4), FE-SH-05, FE-SH-06, FE-SH-07, FE-SH-08, FE-SH-09 | **Denis** |
| Accounting FE — Pact-stub scaffolding | FE-AC-01 (init, against contracts), FE-AC-08 (Nginx + CI/CD) | **Yoana (0.5)** |
| QA — early E2E | QA-01 (login E2E against Shell + dev Entra), QA-06 (APIM JWT) | **QA Person** |

**Exit gate (end of week 7):** Django internal API live in dev with all `accounting`/`core` endpoints; `erp-core` v1.1.0 published; `erp-contracts` v1.0.0 + `@erp/contracts-ts` published; Pact Broker live; staging environment provisioned (INFRA-21); Shell + auth + design system complete; first 3 Accounting BE endpoints live.

---

### Phase C — Accounting BE + AI agents + Accounting FE (Weeks 8–11) — "Build the module"

| Workstream | Tasks | Owner |
|---|---|---|
| Accounting BE — core | BE-A-04 (double-entry validation), BE-A-05 (periods + closure), BE-A-06 (financial statements: Balance Sheet, P&L, Cash Flow) | **Nikolay** |
| Accounting BE — extended | BE-A-07 (VAT + corporate tax — *engage Bulgarian accounting SME by wk 7*), BE-A-08 (audit), BE-A-09 (NAP dossier), BE-A-10 (risk alerts), BE-A-11 (AI narrative trigger via outbox), BE-A-12 (tests), BE-A-13 (CI/CD), BE-A-14 (SSE endpoint) | **Veronika** (BE-mode wk 8–10, then FE-mode wk 11) + **Martin** (BE-mode wk 8 onwards) |
| AI Orchestrator finalise | AI-03 (aggregator), AI-04 (LangSmith), AI-05 (CI/CD), AI-06 (Postgres checkpointer), AI-07 (failure semantics), AI-08 (scope ADR) | **Evgeni** |
| AI Generation Agent | AI-GEN-01..08 + AI-GEN-09 (cold-start SLO + prod min replicas = 1) — financial-narrative prompts (AI-GEN-04/06) | **Evgeni** (after Orchestrator) |
| AI Analysis Agent | AI-ANA-01, AI-ANA-03 (financial anomaly detection), AI-ANA-05, AI-ANA-06 | **Evsin (0.5)** |
| Celery worker | CEL-01..04 (period-close jobs, scheduled report runs) | **Ivan** (after DB-16 lands) |
| Accounting Frontend pages (start wk 11) | FE-AC-02 (Journal Entries — **5 d, biggest FE task**), FE-AC-03 (Chart of Accounts), FE-AC-04 (Periods + closure), FE-AC-05 (Financial dashboard) | **Denis + Yoana (0.5)** + **Veronika** (FE-mode from wk 11) |
| Shell FE finalise | FE-SH-10 (logout), FE-SH-11 (tests), FE-SH-12 (Nginx), FE-SH-13 (CI/CD) | **Denis** (parallel) |
| Infra finalise | INFRA-09 (APIM with SSE policy), INFRA-10 (Front Door), INFRA-17 (Grafana), INFRA-19 (budget), INFRA-21 (apply staging+prod), INFRA-23, INFRA-30 (multi-revision + canary), INFRA-32 (release-please) | **Dimo** |
| DB hardening | DB-20 (backup/restore validation), DB-25 (schema-drift contract test) | **Ivan** (after DB-16) |
| QA — integration | QA-05 (Pact verification across in-scope services), QA-09 partial (generation + analysis pipeline integration) | **QA Person** |

**Exit gate (end of week 11):** Accounting backend feature-complete in dev; AI generation streams financial narratives end-to-end via SSE; first 4 Accounting FE pages render against real backend.

---

### Phase D — Accounting FE finish + Integration QA (Weeks 12–13) — "Round it out and prove it works"

| Workstream | Tasks | Owner |
|---|---|---|
| Accounting Frontend remaining | FE-AC-06 (VAT/tax download), FE-AC-07 (Audit), AI-GEN-10 (frontend SSE warm-up panel reused in FE-AC-05) | **Denis + Yoana (0.5)** + **Veronika** (FE-mode) |
| E2E tests | QA-04 (Accounting full Playwright flow — 5 d) | **QA Person** |
| Security & RBAC | QA-07 (RBAC matrix), QA-11 (OWASP ZAP) | **QA Person** + **Martin** (DevOps-mode for ZAP) |
| Performance | QA-08 (k6 load test, autoscaling validation) | **QA Person** + **Dimo** |
| AI quality | QA-14 scoped to Generation + Analysis agents (red-teaming, PII leak) | **QA Person** + **Evgeni** |
| Accessibility & responsive | QA-12, QA-13 | **QA Person** + **Denis** |
| Backend bug-fix from FE integration | parallel | **Nikolay + Veronika + Martin** |

**Exit gate (end of week 13):** All in-scope QA tasks green on dev; full accounting flow demo to product owner.

---

### Phase E — Production hardening, game-day, go-live (Weeks 14–15) — "Ship it"

**Dimitar joins at start of week 14 — assigned to bug-fix swarm + runbook authoring + serves as schedule insurance.**

| Workstream | Tasks | Owner |
|---|---|---|
| Autoscaling & monitoring | PROD-01, PROD-02, PROD-03 | **Dimo** |
| Image pinning | PROD-04 | **Dimo** |
| Runbooks | PROD-07 (restore, rollback, AI failover, APIM key rotation, incident response), INFRA-31 (expand→migrate→contract + rollback game day on staging) | **Martin** + **Dimitar** + **Nikolay** |
| Docs | PROD-08 (per-service README, Swagger), PROD-09 (AC appendix + PR templates) | **All** |
| Staging smoke + sign-off | PROD-05 — full Playwright + k6 + ZAP + AI red-teaming all green on staging | **Whole team** |
| **Production go-live** | **PROD-06** (manual approval gate in GitHub Environments) | **Dimo** + product owner |

**Exit gate (end of week 15):** Accounting module live in production behind APIM + Front Door, with Entra ID auth, RBAC, audit logging, AI narrative streaming, financial reports, ДДС/corporate tax declaration generation.

> **Safety variant (week 16):** If any Phase D QA finding requires substantial rework, slip go-live to end of week 16. Dimitar's productive week 15 + 16 capacity (~10 d) is the buffer that absorbs this without breaking the schedule.

---

## 5. Visual Gantt (15 weeks)

```
Week:                              1   2   3   4   5   6   7   8   9  10  11  12  13  14  15
                                   |───|───|───|───|───|───|───|───|───|───|───|───|───|───|───|
Dimo     (Infra Phase 0)           ████████████░░░░░░░░░░░░░░░░░░░░
Dimo     (APIM/FD/Mon)                         ████████████████████░░░░░░░░░░░░
Dimo     (Prod readiness)                                                         ████████
Martin   (DevOps — network)                    ████████████░░░░░░░░░░░░░░░░░░░░  ← arrives wk 5
Martin   (Backend — BE-A ext)                              ████████████████░░░░░░
Martin   (Runbooks)                                                               ████████
Evgeni   (erp-core v1.0)           ████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
Evgeni   (erp-core v1.1)                       ████████████░░░░░░░░░░░░░░░░░░░░
Evgeni   (Orchestrator + Gen)                              ████████████████░░░░░░
Evgeni   (Phase E support)                                                ████████
Veronika (BE — CORE-07..12)        ████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
Veronika (BE — Orch + BE-A ext)                ████████████████░░░░░░░░░░░░░░░░
Veronika (FE — Acct FE)                                              ████████████░░
Ivan     (DB bootstrap)            ████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
Ivan     (DB-11..16 internal API)              ████████████████░░░░░░░░░░░░░░░░
Ivan     (Celery + DB hardening)                           ████████████░░░░░░░░░░
Evsin (BE 0.5 — contracts)         ████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░
Evsin (BE 0.5 — AI Analysis)                               ████████████████░░░░░░
Nikolay  (Acct BE A)                           ████████████████████████████░░░░  ← arrives wk 5
Dimitar  (Bug-fix + runbooks)                                                     ████  ← arrives wk 14
Denis    (Shell scaffold)          ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
Denis    (Shell DS + nav + tests)          ████████████████████████░░░░░░░░░░░░
Denis    (Acct FE)                                                   ████████████░░
Yoana (FE 0.5 — Wireframes)        ████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
Yoana (FE 0.5 — Acct FE pages)                     ████████████████████████░░░░░░
QA Person (Setup + early E2E)      ████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░
QA Person (Integration QA)                                           ████████████
QA Person (Sign-off + go-live)                                                    ████████
```

(`█` = active, `░` = idle for that workstream)

---

## 6. Capacity vs. Demand

| Window | FTE-days available | Work demand | Running buffer |
|---|---|---|---|
| Weeks 1–4 (7 FTE) | 98 | ~95 (Phase A) | +3 d (1.6%) |
| Weeks 5–7 (9 FTE) | 94.5 | ~95 (Phase B) | ±0 d cumulative |
| Weeks 8–11 (9 FTE) | 126 | ~135 (Phase C peak) | **−7 d** ⚠️ — covered by Veronika BE→FE float |
| Weeks 12–13 (9 FTE) | 63 | ~60 (Phase D) | +3 d |
| Weeks 14–15 (9 FTE + Dimitar onboarding) | 63 | ~55 (Phase E) | **+8 d safety buffer** |
| **Total** | **~444 d** | **~440 d** | **+4 d (1%)** |

**Phase C is the tightest window** — weeks 8–11 need careful daily-standup tracking. The Veronika BE→FE transition at week 11 is the lever that saves the schedule. If Veronika has to stay BE longer (e.g., BE-A-07 NAP XML edge cases blow up), pull Yoana fully to FE and slip Phase D by 2–3 days into Phase E's buffer.

---

## 7. Best Time Estimate

| Scenario | Duration | Conditions |
|---|---|---|
| **Recommended (this plan)** | **15 weeks (~3.5 months)** | Team ramps as scheduled; Veronika splits BE/FE per phase; Dimitar (wk 14) serves as insurance |
| Aggressive (no slack) | 14 weeks | Requires Bulgarian accounting SME pre-engaged for BE-A-07 by week 1; QA spreads work into Phase C; **breaks if anything slips** |
| Conservative / safe | **16 weeks (~3.7 months)** | Same plan + 1 extra week of QA buffer, Dimitar contributes 1 full productive week |

**Recommendation: target 15 weeks, communicate 16 weeks externally** (1 week of stakeholder buffer = standard PM hygiene).

---

## 8. Top 5 Schedule Risks & Mitigations

| # | Risk | Impact | Mitigation |
|---|---|---|---|
| 1 | **DB-16** Django internal REST API slips (8-day task on critical path, single owner Ivan) | Cascades 1–2 weeks into prod date | Ivan dedicated to Phase B; Evgeni reviews draft API at end of week 5 |
| 2 | **BE-A-07** VAT/corporate tax declaration NAP XML edge cases | 1–2 week slip in Phase C | **Engage Bulgarian accounting SME by week 5** (well before Veronika+Martin start BE-A-07 in week 8); commit XSD test corpus by week 7 |
| 3 | **Veronika BE→FE transition at week 11** is fragile | Phase D slips if BE-A backlog isn't done | Daily standup; Yoana ready to absorb extra FE work; Veronika can defer to FE-AC-05 (financial dashboard) being last |
| 4 | **CORE-17** transactional outbox bugs surface late | Risk of lost AI events on prod | Failure-injection integration test by end of week 5; gate all `BE-A-11` publishes behind the outbox |
| 5 | **Phase C overcommit (7-day deficit)** | Overall schedule slips | Use the Veronika float; if still tight, pull Dimitar's start forward or accept 16-week variant |

---

## 9. What to do with the surplus capacity

This plan delivers the requested MVP in 15 weeks. **`Tasks.md`'s 8-FTE × 6-month sizing implied ~836 productive days available; we use ~440 here.** That leaves a substantial team-capacity surplus for the next increment.

**After go-live (week 15)** the team has roughly **6 weeks of remaining capacity through week 22** = ~6 × ~9.7 FTE × 5 d × 0.7 ≈ **200 productive days** for the next module.

| Next-module option | Cost | Fits in weeks 16–22? | Notes |
|---|---|---|---|
| **Marketing module** (BE-M-* + FE-M-*) | 51 d | ✅ Easily | Recommended next — simpler than Legal, no AI Search dep |
| **Legal module** (BE-L-* + FE-L-*) | 78 d | ✅ Easily | More complex; benefits from AI Search Agent (+19 d) |
| **AI Search Agent** | 19 d | ✅ Trivially | Standalone — useful for Marketing analytics too |
| **AI Ingestion Agent** | 18 d | ✅ Trivially | Required for Legal document workflows |
| **Marketing + AI Search** | 70 d | ✅ ~7 weeks | Good combination |
| **Legal + AI Ingestion + AI Search** | 115 d | ✅ ~12 weeks → end of week 27 | Full Legal stack |
| **All of Legal + Marketing + 3 missing AI agents** | 176 d | Tight — needs ~18 weeks total → month 8 | Full `Tasks.md` scope |

**Suggested roadmap:**
- **Months 0–3.5 (this plan):** Core + Accounting → production
- **Months 3.5–5:** Marketing module + AI Search Agent
- **Months 5–7:** Legal module + AI Ingestion + AI Classification

**Total to full `Tasks.md` scope: ~7 months** with the same 11-person team — vs. `Tasks.md`'s estimated 6 months for an 8-FTE team. The slight overrun is because Dimitar's late start (day +90) eats into early productivity.

---

## 10. Bottom Line

✅ **Core + Accounting MVP delivered in 15 calendar weeks (~3.5 months) with the team in `people.md`.**
✅ All 8-FTE roles from `Tasks.md` are covered (DevOps × 2, Backend × 4, Database × 1, Frontend × 2, QA × 1) — with **+1 backend (Dimitar) as schedule insurance**.
✅ Designer role absorbed by FEs using a stock component library (+2 d overhead).
⚠️ **Phase C (weeks 8–11) is the tightest window** — depends on Veronika successfully transitioning from Backend to Frontend at week 11.
💡 **~200 d of follow-on capacity** remains for the next module — Marketing or Legal can ship in months 3.5–5 of the same engagement without adding people.

