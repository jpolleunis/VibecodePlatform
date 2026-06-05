# Roadmap: VibeCoding Platform (Container-Apps-only)

## Overview

VibeCoding Platform is built as 6 vertical-MVP phases in a single Phase-1 pilot milestone. After Sessie 2 (scope reduction), Azure-managed PaaS is dropped in favour of self-hosted open-source containers (Postgres, Keycloak, Docker registry) and Python (FastAPI) services. Each phase ends with a demonstrable user-visible slice; by Phase 6 all v1 requirements are met, the 10 Definition-of-Done checks pass, and Phase 2 (Nimbus, self-service upload, custom domain) can start.

Target: ~5.5 working weeks with 2 developers + Claude Code. Effective Azure cost target: ~€10-15/month after free tier (jaarbudget < €200).

## Phases

- [ ] **Phase 1: Foundations** — ACA Env + Storage + self-host Postgres + Keycloak + Docker registry + portal login-stub. Demonstrable: TAG-collega logt in via Keycloak en ziet lege portal-dashboard.
- [ ] **Phase 2: Meeting Agent Live behind SSO** — Nginx static + oauth2-proxy sidecar + Python FastAPI LLM-proxy + ACA-secrets. Demonstrable: praat met Claude vanuit Meeting Agent, geen key in browser.
- [ ] **Phase 3: Daily-Cap + Rate-Limit** — Postgres-backed cap (`SELECT … FOR UPDATE`, 90% margin), per-user rate-limit via slowapi, usage-accounting. Demonstrable: 11× $1 call → HTTP 429.
- [ ] **Phase 4: Portal Dashboard (real) + Role Admin** — `app_users` table, RBAC enforcement, dashboard reads DB + ACA REST, admin-pagina voor rol-toewijzing. Demonstrable: admin wijst Developer-rol toe; UI verbergt admin-acties.
- [ ] **Phase 5: Path-Triggered Pipelines + Promote Flow + Audit** — ADO pipelines per artefact, Trivy gate, `promote.yml` retag-only in self-host registry, portal-knop, append-only audit-log. Demonstrable: push naar main → test-deploy auto; admin klikt Promote → prod-revision update + audit-row.
- [ ] **Phase 6: Documentation + DoD Validation** — `docs/architecture.md`, `docs/runbook.md`, `docs/onboarding.md`, `docs/management.md` (al gestart in Phase 1), alle 10 DoD-checks groen. Demonstrable: tweede dev volgt onboarding van clean machine.

## Phase Details

### Phase 1: Foundations
**Goal**: Bicep-deploy vanaf nul levert lege RG met alle gedeelde Container Apps (Postgres, Keycloak, registry) + werkende SSO via Keycloak op lege portal-pagina.
**Mode:** mvp
**Depends on**: TAG IT request (RG + Storage + ACA Env quota + ADO project)
**Requirements**: AUTH-01, KEYCLOAK-01, INFRA-01..06, PGSELF-01, REGISTRY-01, ACASECRETS-01, CICD-04, SEC-01, SEC-05, DOC-04, DOC-05, DOC-06, DOC-07 (draft)
**Success Criteria**:
  1. `az deployment group create -f infra/main.bicep` slaagt vanaf lege RG in ≤ 30 min met 0 onverwachte changes bij re-run (`what-if`).
  2. ADO pipeline `infra.yml` draait via SP-secret service connection en deployt naar RG.
  3. Self-host Postgres draait, `/var/lib/postgresql/data` op Azure Files Premium mount; pg_dump-job draait nightly en schrijft naar pg-backups share.
  4. Keycloak draait, persistent op Postgres, realm `vibecoding` met clients `portal` + `meeting-agent-proxy` + roles `Admin`/`Developer` aanwezig via realm-export.
  5. Self-host Docker registry draait achter `oauth2-proxy`; `docker login` werkt via Keycloak; anonymous `curl /v2/_catalog` → 401.
  6. TAG-gebruiker bezoekt portal-URL → 302 → Keycloak login → na inloggen ziet lege dashboard met "Hello {sub}".
  7. `/healthz/secrets` retourneert 200.
  8. `docs/it-request.md` + `README.md` + initiële `docs/management.md` bestaan.
**Plans**: 7 plans

Plans:
- [ ] 01-01: Bicep single-file skeleton (RG-scope) — Storage Account + Files shares, LAW, ACA Env, ACA Job placeholder for pg-dump.
- [ ] 01-02: Self-host Postgres Container App + Azure Files Premium mount + pgbouncer sidecar + initial schemas (`portal`, `proxy_meeting_agent_test`, `proxy_meeting_agent_prod`, `keycloak`).
- [ ] 01-03: pg_dump ACA Job (cron) + restore-drill script + DoD verification.
- [ ] 01-04: Self-host Docker registry Container App + `oauth2-proxy` sidecar config + Azure Files mount + registry GC job.
- [ ] 01-05: Keycloak Container App + realm-export.json + clients + bootstrap-admin + `keycloak-realm-sync.yml` pipeline.
- [ ] 01-06: Python FastAPI portal stub (login via Authlib + Keycloak) deployed to single Container App with ACA-secrets.
- [ ] 01-07: Write `docs/it-request.md` + `README.md` + initial `docs/management.md` (timeline + cost-table) + initial `docs/app-developer-guide.md` (platform-contract + workflow voor vibecoders).

### Phase 2: Meeting Agent Live behind SSO
**Goal**: Pilot-app (Nginx static + Python FastAPI proxy) draait achter Keycloak via oauth2-proxy sidecar; gebruiker praat met Claude via server-side key uit ACA-secret.
**Mode:** mvp
**Depends on**: Phase 1
**Requirements**: APP-01, APP-02, APP-04, INFRA-05, SEC-02, SEC-03
**Success Criteria**:
  1. Browser opent `meeting-agent-test...azurecontainerapps.io` → Keycloak login → static UI laadt.
  2. UI roept `/api/llm` aan → FastAPI proxy roept Anthropic → antwoord in UI.
  3. Manual DevTools-check: geen Claude-key, geen OIDC-secret, geen DB-credential zichtbaar.
  4. CSP-headers actief (geen `'unsafe-eval'`, strict op portal; static-app strict baseline).
  5. Cookies zijn `Secure`, `HttpOnly`, `SameSite=Lax`.
**Plans**: 4 plans

Plans:
- [ ] 02-01: Containerize Meeting Agent static (Nginx + oauth2-proxy sidecar in same Container App with two containers).
- [ ] 02-02: Python FastAPI LLM-proxy met enkel `POST /api/llm`, ACA-secret env-var + `/healthz/secrets` endpoint.
- [ ] 02-03: Bicep deploy van twee Container Apps (`meeting-agent-static-{test,prod}` + `meeting-agent-proxy-{test,prod}`) met juiste ingress + scaling-rules.
- [ ] 02-04: Manual security audit + CSP-middleware in FastAPI + cookie-flags + secure-by-default check.

### Phase 3: Daily-Cap + Rate-Limit
**Goal**: Anthropic-spend hard begrensd op $10/dag via Postgres-backed cap; per-user rate-limit via slowapi met Keycloak `sub`-keying.
**Mode:** mvp
**Depends on**: Phase 2
**Requirements**: APP-03
**Success Criteria**:
  1. Postgres `daily_cap`, `llm_calls`, `rate_limit` tabellen bestaan; Alembic-migrations idempotent.
  2. Cap-transactie gebruikt `SELECT … FOR UPDATE` + 90% safety-margin; 11× $1 simulated call levert na ~$9 een HTTP 429.
  3. Rate-limit per Keycloak `sub` blokkeert na 10 req/min met HTTP 429 + `Retry-After`.
  4. Eind-van-dag reconciliatie-script: `llm_calls.cost_cents` totaal vs Anthropic Usage API; delta < 5%.
  5. Multi-replica test (`minReplicas: 2`) houdt cap correct.
**Plans**: 3 plans

Plans:
- [ ] 03-01: Alembic-migrations voor proxy-schemas + Pydantic models.
- [ ] 03-02: Cap-transactie + cost-calc uit Anthropic `usage`-block + `tenacity` retry-with-jitter + circuit breaker op 529-storms.
- [ ] 03-03: `slowapi` key-func op Keycloak `sub` + multi-replica race-test.

### Phase 4: Portal Dashboard (real) + Role Admin
**Goal**: Portal toont alle apps + status uit DB en RBAC werkt: Admin/Developer-acties gescheiden.
**Mode:** mvp
**Depends on**: Phase 2 (Meeting Agent bestaat om in dashboard te tonen)
**Requirements**: AUTH-02, PORTAL-01, PORTAL-02, PORTAL-03
**Success Criteria**:
  1. Postgres `portal.app_users` + `portal.apps` tabellen + initial Jasper-as-Admin seed.
  2. Dashboard toont Meeting Agent test + prod URL's + revision-status via Azure REST.
  3. Admin-pagina werkt: kies TAG-gebruiker (via Keycloak admin API of dropdown), ken `Admin`/`Developer` toe; wijziging in DB.
  4. Developer-rol ziet admin-acties **niet** in UI én krijgt 403 op direct API-call.
  5. Tenant-mismatch / role-mismatch test: token met andere realm → 403.
**Plans**: 4 plans

Plans:
- [ ] 04-01: SQLAlchemy models (`AppUser`, `App`) + Alembic-migration + initial-admin seed.
- [ ] 04-02: Dashboard endpoint: Postgres + Azure REST (`az containerapp revision list`) + Jinja2 + HTMX-polling.
- [ ] 04-03: Admin-pagina + FastAPI Depends(require_role("Admin")) decorator + zod-equivalent pydantic-validatie.
- [ ] 04-04: Middleware Keycloak-JWT verify + role-check helpers + pytest e2e role-enforcement test.

### Phase 5: Path-Triggered Pipelines + Promote Flow + Audit
**Goal**: Volledig CI/CD met Trivy-gate; admin promote-knop in portal triggert retag-only pipeline in self-host registry; alle role + promote events worden append-only audit-logged.
**Mode:** mvp
**Depends on**: Phase 4
**Requirements**: PORTAL-04, CICD-01, CICD-02, CICD-03, SEC-04, DOC-02
**Success Criteria**:
  1. Push naar main op pad `apps/portal/**` triggert enkel `portal.yml`.
  2. Trivy met HIGH-CVE in image faalt deploy.
  3. Admin klikt "Promote" in portal → `promote.yml` retag prod-tag = test-digest in self-host registry → `meeting-agent-proxy-prod` revision update binnen 2 min.
  4. Postgres `audit_log` tabel weigert `UPDATE`/`DELETE` voor app-role; row staat erin binnen 30s na promote.
  5. `docs/runbook.md` bevat secret-rotation procedure (Anthropic key, Keycloak admin, Postgres password, SP-secret).
**Plans**: 5 plans

Plans:
- [ ] 05-01: `portal.yml`, `meeting-agent-static.yml`, `meeting-agent-proxy.yml` pipelines met path-triggers + git-sha tag → self-host registry push.
- [ ] 05-02: Trivy task + `.trivyignore` met expiry-dates + ADO test-publisher.
- [ ] 05-03: `promote.yml` (manual trigger, retag-only via Docker CLI tegen self-host registry) + ADO REST trigger vanuit portal.
- [ ] 05-04: `audit_log` tabel + grants + portal-server-handler schrijft audit rows.
- [ ] 05-05: `docs/runbook.md` deploy + rollback + secret-rotation + incident-response.

### Phase 6: Documentation + DoD Validation
**Goal**: Alle docs op productiekwaliteit; alle 10 DoD-checks aantoonbaar groen; tweede dev kan onboarden van clean machine.
**Mode:** mvp
**Depends on**: Phase 5
**Requirements**: DOC-01, DOC-02 (polish), DOC-03, DOC-07 (polish), alle DoD-verificaties
**Success Criteria**:
  1. `docs/architecture.md` reflecteert geïmplementeerde realiteit.
  2. `docs/onboarding.md` is door tweede dev gevolgd van clean machine → lokaal draaien + test-deploy in ≤ 4 uur.
  3. Alle 10 DoD-checks uitgevoerd en gedocumenteerd (incl. Postgres-restore-drill + registry-anon-401).
  4. PROJECT.md "Key Decisions" outcomes bijgewerkt.
  5. Phase 1 milestone-summary geschreven; klaar voor Phase 2.
**Plans**: 3 plans

Plans:
- [ ] 06-01: `docs/architecture.md` polish.
- [ ] 06-02: `docs/onboarding.md` shake-out met tweede dev.
- [ ] 06-03: DoD-validatie scripts + bevindingen in `.planning/VERIFICATION.md`.

## Progress

**Execution Order:** Phase 1 → 2 → 3 || 4 → 5 → 6 (3 and 4 can run in parallel given 2 developers).

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundations | 0/7 | Not started | - |
| 2. Meeting Agent Live behind SSO | 0/4 | Not started | - |
| 3. Daily-Cap + Rate-Limit | 0/3 | Not started | - |
| 4. Portal Dashboard + Role Admin | 0/4 | Not started | - |
| 5. CI/CD + Promote + Audit | 0/5 | Not started | - |
| 6. Documentation + DoD Validation | 0/3 | Not started | - |

**Dev assignments (suggested):**

| Phase | Dev A | Dev B |
|-------|-------|-------|
| 1 | Postgres + pg_dump + Keycloak realm | Bicep skeleton + Storage + registry + portal stub |
| 2 | Meeting Agent static + oauth2-proxy | LLM-proxy FastAPI + ACA-secret |
| 3 (parallel with 4) | Postgres daily-cap + slowapi | — |
| 4 (parallel with 3) | — | Portal dashboard + admin + RBAC |
| 5 | Pipelines + Trivy + promote | Audit-log + portal trigger |
| 6 | Architecture + runbook | Onboarding shake-out + DoD checks |

**Critical path:** TAG IT-aanvraag indienen op dag 1. Zonder RG + Storage Account + ACA Env quota + ADO-project stalt Phase 1.

---

## Requirements Coverage (100%)

Zie REQUIREMENTS.md § Traceability — elke v1 REQ-ID is mapped naar een primary phase.
