# REQUIREMENTS — VibeCoding Platform

**Scope:** Phase 1 pilot (v1)
**Source:** PROJECT.md Active + research/* (na Sessie 2 — Container-Apps-only)
**Last updated:** 2026-05-30

---

## v1 Requirements (Phase 1)

### Authentication & Access

- [ ] **AUTH-01**: Gebruiker logt in op portal en op pilot-apps via OIDC tegen self-hosted Keycloak. Anonieme toegang overal geweigerd.
- [ ] **AUTH-02**: Twee rollen actief: `Admin` (alles) en `Developer` (uploaden + test, geen prod). Rolcontrole blokkeert UI-acties + FastAPI-endpoints.
- [ ] **KEYCLOAK-01**: Keycloak draait als Container App (`minReplicas: 1`), backed door self-host Postgres. Realm-config + client-config geversioneerd in `infra/keycloak/realm-export.json`. Bootstrap-admin via ACA-secret, herzien op eerste login.

### Infrastructure (Container-Apps-only)

- [ ] **INFRA-01**: Phase-1-Bicep single-file (`infra/main.bicep`, `infra/main.bicepparam`) deployt RG-scope, idempotent (`what-if` toont 0 onverwachte changes bij re-run).
- [ ] **INFRA-02**: Resources binnen RG: 1× Storage Account + Azure Files shares (pg-data, keycloak-data, registry-data, pg-backups); 1× Log Analytics workspace; 1× Container Apps Environment (Consumption); N Container Apps (zie INFRA-05).
- [ ] **INFRA-03**: Budget alert op €30/maand voor RG + tweede alert op €15/maand (early warning).
- [ ] **INFRA-04**: Naming + tagging-conventie (`{rg-,st,law-,cae-,ca-}vibecoding-{env}` + tags `project`, `env`, `owner`) gedocumenteerd en gevolgd.
- [ ] **INFRA-05**: Test + Prod Container Apps per pilot-app met resource-limits: test 0.25 vCPU / 0.5 GiB / scale-to-zero / max 1; prod 0.5 vCPU / 1 GiB / `minReplicas: 1` / max 3 / autoscale CPU>70%. LLM-proxy: `minReplicas: 1` in beide env's (cap-state correctness).
- [ ] **INFRA-06**: Log Analytics daily-cap 500 MB en 30-dagen retentie. Apps loggen structureel JSON via Python `structlog`; geen losse APM-laag.
- [ ] **PGSELF-01**: Self-host Postgres 16 als Container App (`postgres:16-alpine`), `/var/lib/postgresql/data` op Azure Files mount. Nightly `pg_dump` Container App-job dumpt naar tweede Azure Files share met 14d-retentie. Connection-string in ACA-secret per app.
- [ ] **REGISTRY-01**: Self-host Docker registry (`distribution/distribution:3`) als Container App achter Keycloak OIDC-proxy (`oauth2-proxy` of FastAPI gatekeeper). `/var/lib/registry` op Azure Files. Webhook of pull-pipeline-helper niet vereist in Phase 1.
- [ ] **ACASECRETS-01**: Alle app-secrets via Container Apps `secrets:` (bicep `secrets: [...]` lijst + `env: [{ name, secretRef }]`). Geen Key Vault. Rotation: pipeline-variable update + revision-bump.

### Pilot-app — Meeting Agent

- [ ] **APP-01**: Meeting Agent statische artifact in Nginx-container draait achter Keycloak-OIDC-gatekeeper sidecar (`oauth2-proxy:7.6+`). Ongeauth → 302 naar Keycloak.
- [ ] **APP-02**: Python FastAPI LLM-proxy met exact één endpoint `POST /api/llm`. Anthropic key uit ACA-secret env-var, niet uit code/repo. `@anthropic-ai/anthropic` Python SDK ≥ 0.40.
- [ ] **APP-03**: Per-user rate-limit (10 req/min default) + hard daily-cap $10/dag. Cap-state in self-host Postgres met `SELECT … FOR UPDATE`-transactie, 90% safety margin. Cost-berekening via Anthropic `usage`-block (alle 4 token-typen).
- [ ] **APP-04**: Manual security check vóór go-live bewijst dat in Meeting Agent géén Claude-key, géén DB-credential, géén sessie-token of OIDC-secret zichtbaar is in browser/DevTools.

### Management portal

- [ ] **PORTAL-01**: Python FastAPI portal achter Keycloak OIDC (Authlib). Server-rendered HTML (Jinja2 templates + HTMX voor interactiviteit, geen JS-framework). `trusted_hosts`-middleware actief.
- [ ] **PORTAL-02**: Dashboard toont alle geregistreerde apps met test-URL, prod-URL en laatst-bekende status (revision running ja/nee) — via Azure CLI of REST API.
- [ ] **PORTAL-03**: Admin-pagina laat admin een TAG-gebruiker `Admin` of `Developer` toewijzen; wijziging schrijft naar Postgres + audit-log.
- [ ] **PORTAL-04**: Audit-log tabel append-only op DB-niveau (REVOKE UPDATE/DELETE voor app-role); bevat `promote_test_to_prod` en `role_change` events met actor, target, timestamp (UTC `timestamptz`), metadata.

### CI/CD

- [ ] **CICD-01**: Azure DevOps Pipelines bouwen elk artefact (portal, meeting-agent-static, meeting-agent-proxy, infra, keycloak-realm-sync) op path-triggers; pushen naar **self-host registry** met tag = git-sha; deployen via `az containerapp update --image`.
- [ ] **CICD-02**: Trivy stap `--severity HIGH,CRITICAL --ignore-unfixed --exit-code 1` blokkeert deploy bij HIGH+ vulnerabilities; `.trivyignore` met expiry-dates ondersteund.
- [ ] **CICD-03**: "Promote test→prod" als aparte pipeline (`promote.yml`) doet **retag van bestaand digest in self-host registry** (niet rebuild) en is alleen triggerbaar door admin; promotion schrijft audit-log row.
- [ ] **CICD-04**: ADO service connection gebruikt Service Principal client-secret (geen WIF). SP-secret 90-daags rotatable; rotation-procedure in runbook.

### Security

- [ ] **SEC-01**: Alle geheimen in Container Apps `secrets:` (Bicep + pipeline variables). Geen secrets in repo, env-vars op image-niveau, pipeline-logs.
- [ ] **SEC-02**: CSP-headers actief op alle web-endpoints (portal + Meeting Agent static). Portal: strict CSP via FastAPI-middleware met nonce per request. Geen `'unsafe-eval'`.
- [ ] **SEC-03**: HTTPS-only (ACA default), secure + httpOnly + sameSite=lax cookies, sessie-management via Authlib OIDC.
- [ ] **SEC-04**: Secret-rotation procedure gedocumenteerd in `docs/runbook.md` (Anthropic key, Keycloak admin password, Postgres password, SP-secret).
- [ ] **SEC-05**: `/healthz/secrets` readiness-probe op portal en proxy retourneert 503 wanneer een vereist secret leeg of onbereikbaar is.

### Documentatie

- [ ] **DOC-01**: `docs/architecture.md` met Mermaid component-diagram, trust-boundaries, dataflows.
- [ ] **DOC-02**: `docs/runbook.md` met deploy/rollback/secret-rotation/incident-response.
- [ ] **DOC-03**: `docs/onboarding.md` zodat een nieuwe dev binnen 1 dag lokaal draait + test-deploy doet.
- [ ] **DOC-04**: `docs/it-request.md` (vereenvoudigd: alleen RG, Storage Account, ACA Env quota, ADO org/project).
- [ ] **DOC-05**: Repo-`README.md` met 10-regel pitch + links naar docs.
- [ ] **DOC-06**: `docs/management.md` — management-facing architectuur, user stories, timeline, Container-Apps-only kost.

---

## v2 Requirements (deferred to Phase 2)

- **CUSTOM-DOMAIN-01**, **LOGS-IN-PORTAL-01**, **VIEWER-ROLE-01**, **COST-DASH-01**, **APP-REG-SELF-SVC-01**, **HEALTH-CHECK-01**, **ROLLBACK-BTN-01**, **MULTI-LLM-01**, **AUDIT-HARDEN-01**, **NIMBUS-DB-SYNC-01**, **TEAMS-NOTIFY-01**, **KEYCLOAK-MS-FED-01** (Microsoft-tenant federation in Keycloak)

---

## Out of Scope (anti-features)

Reden = "prevent re-adding".

- **Azure managed PaaS-services** (Postgres Flex, Key Vault, ACR, App Insights, AAD app-reg, Easy Auth, WIF) — gedropt vanwege kost + vendor lock-in
- **UI-upload van Dockerfile/zip** — RCE-vector
- **Multi-reviewer approval-workflow** — single-admin promote in Phase 1
- **Native mobile apps**
- **Multi-LLM-provider abstractie**
- **Per-PR preview deployments**
- **Backstage Software Catalog**
- **Stripe-style metered billing / multi-tenancy / RLS**
- **Public status page**
- **LaunchDarkly / sophisticated feature-flags**
- **OpenTelemetry export naar 3rd-party APM**
- **CAPTCHA / bot-protection**
- **Managed Postgres backup/PITR** — vervangen door pg_dump nightly + externe Python-recovery

---

## Definition of Done (Phase 1)

Phase 1 is "done" wanneer **alle** v1 requirements `[x]` zijn én:

1. Bicep-deploy vanaf lege RG slaagt in één run (≤ 30 min, `what-if` toont 0 onverwachte changes).
2. TAG-collega logt in via Microsoft (federated via Keycloak) of via Keycloak-lokaal → ziet portal-dashboard → klikt Meeting Agent → praat met Claude → DevTools toont géén Claude-key.
3. Admin promoot test→prod via portal-knop → audit-row staat in DB binnen 30s.
4. Trivy faalt deploy bij geïnjecteerde HIGH-CVE in test-image.
5. Daily-cap-test: 11× $1 simulated call triggert HTTP 429 + cap-row in DB toont ~$10.
6. Postgres-restore-drill: corrupt Azure Files mount, restore vanuit `pg_dump`-share → portal werkt binnen 1 uur (ofwel verifieer extern Python-recovery-pad).
7. Self-host registry test: `docker login <registry-url>` met Keycloak credentials werkt; anonymous pull faalt.
8. Cost-alert e-mail komt aan na simulated overspend test.
9. Nieuwe dev volgt `docs/onboarding.md` van clean machine → lokaal draaien + test-deploy binnen 4 uur.
10. Azure cost-rapport eerste maand toont < €30 effectief.

---

## Traceability

| REQ-ID | Phase | Notes |
|--------|-------|-------|
| AUTH-01 | 1 | Keycloak OIDC-login werkt op portal + 1 dummy ACA app |
| AUTH-02 | 4 | Postgres-backed roles + RBAC middleware |
| KEYCLOAK-01 | 1 | Realm + clients deployed + persisted |
| INFRA-01 | 1 | Bicep single-file |
| INFRA-02 | 1 | Shared resources |
| INFRA-03 | 1 | Budget-alerts |
| INFRA-04 | 1 | Naming + tags |
| INFRA-05 | 2 | Pilot-app test+prod ACA |
| INFRA-06 | 1 | LAW + structlog |
| PGSELF-01 | 1 | Postgres-container + Azure Files mount + pg_dump-job |
| REGISTRY-01 | 1 | Distribution-container + oauth2-proxy + Azure Files |
| ACASECRETS-01 | 1 | Bicep `secrets:` + rotation pattern |
| APP-01 | 2 | Meeting Agent static + oauth2-proxy sidecar |
| APP-02 | 2 | FastAPI LLM-proxy + ACA-secret |
| APP-03 | 3 | Postgres-backed daily-cap + rate-limit |
| APP-04 | 2 | Manual DevTools-check |
| PORTAL-01 | 4 | FastAPI + Jinja2 + HTMX + Authlib |
| PORTAL-02 | 4 | Dashboard reads DB + ACA API |
| PORTAL-03 | 4 | Admin-pagina + RBAC |
| PORTAL-04 | 5 | Append-only audit log |
| CICD-01 | 5 | Path-triggered ADO pipelines + self-host registry push |
| CICD-02 | 5 | Trivy gate HIGH+ |
| CICD-03 | 5 | promote.yml retag-only |
| CICD-04 | 1 | SP-secret service connection |
| SEC-01 | 1 | ACA secrets |
| SEC-02 | 2 | CSP headers |
| SEC-03 | 2 | HTTPS + cookies + Authlib |
| SEC-04 | 5 | Rotation procedure in runbook |
| SEC-05 | 1 | `/healthz/secrets` readiness |
| DOC-01 | 6 | Architecture polish |
| DOC-02 | 5+6 | Runbook |
| DOC-03 | 6 | Onboarding shake-out |
| DOC-04 | 1 | IT-request day 1 |
| DOC-05 | 1 | README |
| DOC-06 | 1 | Management-facing MD |
