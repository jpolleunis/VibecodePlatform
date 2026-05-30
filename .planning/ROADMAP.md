# Roadmap: VibeCoding Platform

## Overview

VibeCoding Platform is built as 6 vertical-MVP phases inside a single Phase-1-pilot milestone. Each phase ends with a demonstrable end-to-end user capability — Phase 1 already produces a working SSO login on an empty dashboard, and every subsequent phase adds one observable user-visible slice on top. By Phase 6, all 28 v1 requirements are met, the 8 Definition-of-Done checks pass, and Phase 2 (developer self-service, Nimbus, DB-sync) can start cleanly.

Target: ~2 working weeks with 2 developers + Claude Code, contingent on the TAG IT request (RG, WIF service connection, Entra app-registration) being filed on day 1 of Phase 1.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Foundations + Empty Dashboard** — Azure infra, Entra app-reg, ADO+WIF, portal stub with login-only. Demonstrable: TAG-collega logt in via Microsoft, ziet leeg dashboard.
- [ ] **Phase 2: Meeting Agent Live behind SSO** — Containerize static artifact + Node Express proxy, Easy Auth, KV-secrets via MI. Demonstrable: praat met Claude vanuit Meeting Agent, geen key in browser.
- [ ] **Phase 3: Daily-Cap + Rate-Limit** — Postgres-backed cap (`SELECT … FOR UPDATE`, 90% margin), per-user rate-limit, usage-accounting via Anthropic `usage` block. Demonstrable: 11× $1 call triggert HTTP 429.
- [ ] **Phase 4: Portal Dashboard (real) + Role Admin** — `app_users` table, RBAC enforcement, dashboard reads apps+status, admin-pagina voor rol-toewijzing. Demonstrable: admin wijst Developer-rol toe; UI verbergt admin-acties voor Developer.
- [ ] **Phase 5: Path-Triggered Pipelines + Promote Flow + Audit** — ADO pipelines per artefact, Trivy gate, `promote.yml` retag-only, portal-knop, append-only audit-log. Demonstrable: push naar main → test-deploy auto; admin klikt Promote → prod-revision update + audit-row.
- [ ] **Phase 6: Documentation + DoD Validation** — `docs/architecture.md`, `docs/runbook.md`, `docs/onboarding.md`, alle 8 DoD-checks groen. Demonstrable: tweede dev volgt onboarding-doc en doet test-deploy binnen 4 uur.

## Phase Details

### Phase 1: Foundations + Empty Dashboard
**Goal**: Bicep-deploy vanaf nul levert lege RG met alle gedeelde services + werkende SSO op een lege portal-pagina.
**Mode:** mvp
**Depends on**: TAG IT request (out-of-band — file day 1)
**Requirements**: AUTH-01, AUTH-03, INFRA-01, INFRA-02, INFRA-03, INFRA-04, INFRA-06, CICD-04, SEC-01, SEC-05, DOC-04, DOC-05
**Success Criteria** (what must be TRUE):
  1. `az deployment group create -f infra/main.bicep` slaagt vanaf lege RG in ≤ 30 min met 0 onverwachte changes bij re-run (`what-if`).
  2. ADO pipeline `infra.yml` draait via WIF service connection en deployt naar RG zonder long-lived SP secret.
  3. TAG-gebruiker bezoekt portal-URL → 302 → Microsoft login → na inloggen ziet lege dashboard-pagina met "Hello {upn}".
  4. `/healthz/secrets` retourneert 200 (alle vereiste secrets aanwezig in KV).
  5. `docs/it-request.md` + `README.md` bestaan en zijn verifieerbaar accuraat.
**Plans**: 5 plans

Plans:
- [ ] 01-01: Bicep skeleton (RG-scope, AVM-pinned modules) — KV, LAW, App Insights, ACR, ACA Env, Postgres Flex (empty schema).
- [ ] 01-02: Entra app-registration `vibecoding-platform` + app-roles `Admin`+`Developer` + redirect-URIs (portal + Easy Auth callbacks).
- [ ] 01-03: ADO project setup + WIF service connection + initial `infra.yml` pipeline.
- [ ] 01-04: Next.js portal stub (Auth.js v5 `microsoft-entra-id`, tenant-pinned, JWT session) deployed to single Container App.
- [ ] 01-05: Write `docs/it-request.md` + `README.md` + commit.

### Phase 2: Meeting Agent Live behind SSO
**Goal**: Pilot-app (Meeting Agent static + Node Express proxy) draait achter Easy Auth; gebruiker praat met Claude via server-side key.
**Mode:** mvp
**Depends on**: Phase 1
**Requirements**: APP-01, APP-02, APP-04, SEC-02, SEC-03
**Success Criteria** (what must be TRUE):
  1. Browser opent `meeting-agent-test.<...>.azurecontainerapps.io` → Microsoft login → static UI laadt.
  2. UI roept `/api/llm` aan → proxy roept Anthropic → antwoord verschijnt in UI.
  3. Manual DevTools-check: geen Claude-key, geen Auth.js-secret, geen DB-credential zichtbaar in HTML, JS, network calls, of console.
  4. CSP-headers actief (geen `'unsafe-eval'`, `script-src` strict op portal; artifact-iframe geïsoleerd op subdomein indien nodig).
  5. Cookies zijn `Secure`, `HttpOnly`, `SameSite=Lax`.
**Plans**: 4 plans

Plans:
- [ ] 02-01: Containerize Meeting Agent static (Nginx-alpine, multi-stage, `output: 'standalone'` patroon, COPY `public/` + `.next/static`).
- [ ] 02-02: Node Express LLM-proxy met enkel `POST /api/llm`, KV-secret via MI op cold start + retry-resilient.
- [ ] 02-03: Bicep deploy van twee Container Apps (`meeting-agent-test`, `meeting-agent-prod`) + Easy Auth wiring + ingress.
- [ ] 02-04: Manual security audit + CSP-headers + cookie-flags + secure-by-default check.

### Phase 3: Daily-Cap + Rate-Limit
**Goal**: Anthropic-spend is hard begrensd op $10/dag via Postgres-backed cap; per-user rate-limit beschermt tegen runaway-clients.
**Mode:** mvp
**Depends on**: Phase 2
**Requirements**: APP-03
**Success Criteria** (what must be TRUE):
  1. Postgres `daily_cap` tabel + `llm_calls` audit-tabel bestaan; schema-migrations idempotent.
  2. Cap-transactie gebruikt `SELECT … FOR UPDATE` + 90% safety-margin; 11× $1 simulated call levert na ~$9 de eerstvolgende een HTTP 429 + duidelijke error-body.
  3. Rate-limit per `x-ms-client-principal-id` blokkeert na 10 req/min met HTTP 429 + `Retry-After` header.
  4. Eind-van-dag reconciliatie-script vergelijkt `llm_calls.cost_cents` totaal met Anthropic Usage API; delta < 5%.
  5. Geen single-replica afhankelijkheid: scale-test met `minReplicas: 2` houdt cap correct (geen overrun).
**Plans**: 3 plans

Plans:
- [ ] 03-01: Postgres-schema (`daily_cap`, `llm_calls`, `rate_limit`) + Prisma model (proxy side, of raw SQL met `pg`).
- [ ] 03-02: Cap-transactie + cost-calc uit Anthropic `usage` block (alle 4 token-typen) + 503-fallback bij 529-storms.
- [ ] 03-03: `rate-limiter-flexible` PG-store per-user keying op `x-ms-client-principal-id`; multi-replica test.

### Phase 4: Portal Dashboard (real) + Role Admin
**Goal**: Portal toont alle apps + status uit DB en RBAC werkt: Admin/Developer-acties zijn gescheiden, niet-admins kunnen geen rollen wijzigen.
**Mode:** mvp
**Depends on**: Phase 2 (Meeting Agent bestaat om in dashboard te tonen)
**Requirements**: AUTH-02, PORTAL-01, PORTAL-02, PORTAL-03
**Success Criteria** (what must be TRUE):
  1. Postgres `app_users` + `apps` tabellen aangemaakt; Jasper geseed als Admin op eerste login.
  2. Dashboard toont Meeting Agent test + prod URL's + revision-status (running ja/nee) via Container Apps API.
  3. Admin-pagina werkt: admin selecteert TAG-gebruiker, kent `Admin`/`Developer` toe; wijziging schrijft DB + audit-stub (volledige audit-log volgt in Phase 5).
  4. Developer-rol ziet admin-acties **niet** in UI én krijgt 403 op direct API-call.
  5. Tenant-mismatch middleware test: token met andere `tid` krijgt 403.
**Plans**: 4 plans

Plans:
- [ ] 04-01: Prisma schema (`app_users`, `apps`) + migrations + initial seed.
- [ ] 04-02: Dashboard server-component: Postgres + Container Apps API fetch + shadcn table.
- [ ] 04-03: Admin-pagina met `next-safe-action` voor RBAC-gates + zod-validatie.
- [ ] 04-04: Middleware tenant-pin + role-check helpers + e2e role-enforcement test.

### Phase 5: Path-Triggered Pipelines + Promote Flow + Audit
**Goal**: Volledig CI/CD met Trivy-gate; admin promote-knop in portal triggert retag-only pipeline; alle role + promote events worden append-only audit-logged.
**Mode:** mvp
**Depends on**: Phase 4
**Requirements**: PORTAL-04, CICD-01, CICD-02, CICD-03, SEC-04
**Success Criteria** (what must be TRUE):
  1. Push naar main op pad `apps/portal/**` triggert enkel `portal.yml`, niet andere pipelines.
  2. Trivy stap met HIGH-CVE in image faalt deploy; `.trivyignore` met expiry-date werkt voor MEDIUM+.
  3. Admin klikt "Promote" in portal → `promote.yml` retag prod-image = test-digest → `meeting-agent-prod` revision update binnen 2 min.
  4. Postgres `audit_log` tabel weigert `UPDATE`/`DELETE` voor `portal_app_role`; row staat erin binnen 30s na promote.
  5. `docs/runbook.md` bevat secret-rotation procedure (Claude-key, Auth.js-secret, DB-password).
**Plans**: 5 plans

Plans:
- [ ] 05-01: `portal.yml`, `meeting-agent-static.yml`, `meeting-agent-proxy.yml` pipelines met path-triggers + git-sha tag.
- [ ] 05-02: Trivy task `--severity HIGH,CRITICAL --ignore-unfixed --exit-code 1` + ADO test-publisher integratie.
- [ ] 05-03: `promote.yml` (manual trigger, retag-only) + ADO REST trigger vanuit portal-server-action.
- [ ] 05-04: `audit_log` tabel + grants (`REVOKE UPDATE, DELETE`), portal-server-action wrappers schrijven audit rows.
- [ ] 05-05: `docs/runbook.md` deploy + rollback + secret-rotation + incident-response sectie.

### Phase 6: Documentation + DoD Validation
**Goal**: Alle docs (architecture, runbook, onboarding) op productiekwaliteit; alle 8 DoD-checks aantoonbaar groen; tweede dev kan onboarden van nul.
**Mode:** mvp
**Depends on**: Phase 5
**Requirements**: DOC-01, DOC-02, DOC-03 (polish), alle DoD-verificaties uit REQUIREMENTS.md
**Success Criteria** (what must be TRUE):
  1. `docs/architecture.md` reflecteert geïmplementeerde realiteit (component-diagram, dataflows, secret-flow, RBAC).
  2. `docs/onboarding.md` is door tweede dev gevolgd van clean machine → lokaal draaien + test-deploy in ≤ 4 uur.
  3. Alle 8 DoD-checks uit REQUIREMENTS.md uitgevoerd en gedocumenteerd (Bicep-redeploy, E2E-SSO-Claude, audit row, Trivy-gate, daily-cap, KV-rotation, cost-alert, onboarding-tijd).
  4. PROJECT.md "Key Decisions" tabel-outcomes bijgewerkt (Pending → ✓/⚠️).
  5. Phase 1 milestone-summary geschreven; klaar voor Phase 2 (Nimbus, self-service upload, custom domain).
**Plans**: 3 plans

Plans:
- [ ] 06-01: `docs/architecture.md` polish (verschil tussen research/ARCHITECTURE.md en daadwerkelijke implementatie verwerken).
- [ ] 06-02: `docs/onboarding.md` shake-out met tweede dev (real onboarding-run, doc fixen wat verwart).
- [ ] 06-03: DoD-validatie scripts/runs + bevindingen in `.planning/VERIFICATION.md`.

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 → 5 → 6
Phase 3 and Phase 4 are independent (different code paths) and can run in parallel given 2 developers — see `Dev assignments` below.

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundations + Empty Dashboard | 0/5 | Not started | - |
| 2. Meeting Agent Live behind SSO | 0/4 | Not started | - |
| 3. Daily-Cap + Rate-Limit | 0/3 | Not started | - |
| 4. Portal Dashboard + Role Admin | 0/4 | Not started | - |
| 5. CI/CD + Promote + Audit | 0/5 | Not started | - |
| 6. Documentation + DoD Validation | 0/3 | Not started | - |

**Dev assignments (suggested):**

| Phase | Dev A | Dev B |
|-------|-------|-------|
| 1 | Bicep + Entra app-reg + WIF | Portal stub + Auth.js + IT-request doc |
| 2 | Meeting Agent static + CSP | LLM-proxy + Easy Auth wiring |
| 3 (parallel with 4) | Postgres daily-cap | — |
| 4 (parallel with 3) | — | Portal dashboard + admin + RBAC |
| 5 | Pipelines + Trivy + promote.yml | Audit-log + portal trigger |
| 6 | Architecture + runbook | Onboarding shake-out + DoD checks |

**Critical path:** Phase 1 IT-deps. Without RG + WIF + app-reg on day 1, Phase 1 stalls and everything downstream follows.

---

## Requirements Coverage (100%)

| REQ-ID | Phase |
|--------|-------|
| AUTH-01 | 1 |
| AUTH-02 | 4 |
| AUTH-03 | 1 |
| INFRA-01 | 1 |
| INFRA-02 | 1 |
| INFRA-03 | 1 |
| INFRA-04 | 1 |
| INFRA-05 | 2 (test+prod ACA voor pilot-app) |
| INFRA-06 | 1 |
| APP-01 | 2 |
| APP-02 | 2 |
| APP-03 | 3 |
| APP-04 | 2 |
| PORTAL-01 | 4 |
| PORTAL-02 | 4 |
| PORTAL-03 | 4 |
| PORTAL-04 | 5 |
| CICD-01 | 5 |
| CICD-02 | 5 |
| CICD-03 | 5 |
| CICD-04 | 1 |
| SEC-01 | 1 |
| SEC-02 | 2 |
| SEC-03 | 2 |
| SEC-04 | 5 |
| SEC-05 | 1 |
| DOC-01 | 6 |
| DOC-02 | 5 (skelet) + 6 (polish) |
| DOC-03 | 6 |
| DOC-04 | 1 |
| DOC-05 | 1 |

All 28 v1 requirements mapped to exactly one primary phase.
