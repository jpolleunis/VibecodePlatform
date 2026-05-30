# REQUIREMENTS — VibeCoding Platform

**Scope:** Phase 1 pilot (v1)
**Source:** PROJECT.md Active + research/FEATURES.md + research/SUMMARY.md
**Last updated:** 2026-05-30

> **Requirement quality rule.** Every Active item is *specific, testable, user-centric, atomic, independent*. Vague items get rewritten or split.

---

## v1 Requirements (Phase 1)

### Authentication & Access

- [ ] **AUTH-01**: Gebruiker logt in op portal en op pilot-apps via Azure AD SSO (TAG-tenant, Entra ID v2 token, tenant GUID pinned). Anonieme toegang is overal geweigerd.
- [ ] **AUTH-02**: Twee rollen actief in Phase 1: `Admin` (alles) en `Developer` (uploaden + test, geen prod). Rol-controle blokkeert UI-acties + server-actions in portal en endpoints in proxy.
- [ ] **AUTH-03**: Azure AD app-registration `vibecoding-platform` is gedocumenteerd (manifest in repo, redirect-URIs, app-roles) en herhaalbaar opzetbaar via Bicep + handmatige consent-stap.

### Infrastructure (IaC)

- [ ] **INFRA-01**: Alle Azure resources voor Phase 1 zijn gedefinieerd in Bicep met version-pinned Azure Verified Modules (`br/public:avm/...`); `az deployment group what-if` toont 0 onverwachte deletes bij herdeploy.
- [ ] **INFRA-02**: Eén Container Apps Environment, één ACR (Basic SKU, 7d retention untagged), één Key Vault (RBAC enabled), één Log Analytics workspace + workspace-based Application Insights, één Postgres Flex Server (B1ms, pgbouncer ON).
- [ ] **INFRA-03**: Budget alert op €100/maand voor RG `rg-vibecoding-pilot` + tweede alert op €50/maand (early warning).
- [ ] **INFRA-04**: Naming- en tagging-conventie (`{rg-,kv-,acr-,cae-,ca-}vibecoding-{env}` + tags `project`, `env`, `owner`) is gedocumenteerd en consequent gevolgd in alle Bicep-modules.
- [ ] **INFRA-05**: Test en Prod Container Apps bestaan per app met resource-limits zoals in PROJECT.md (test: 0.5 vCPU / 1 GB / `minReplicas: 0` / `maxReplicas: 1`; prod: 2 vCPU / 4 GB / `minReplicas: 1` / `maxReplicas: 5` / autoscale CPU>70%). LLM-proxy heeft `minReplicas: 1` in beide env's (cap-state correctness).
- [ ] **INFRA-06**: Log Analytics workspace heeft daily-cap 1 GB en 30-dagen retentie; Application Insights `samplingPercentage` = 20% in dev, 5% in proxy-prod (CRIT-6 mitigatie).

### Pilot-app — Meeting Agent

- [ ] **APP-01**: Meeting Agent statische artifact draait in Nginx-container, met Container Apps Easy Auth (Entra ID) ervoor; ongeauthenticeerde bezoekers worden redirected naar Microsoft login.
- [ ] **APP-02**: Node Express LLM-proxy met exact één endpoint `POST /api/llm`; key voor Anthropic Claude API komt uit Key Vault via Managed Identity, niet uit env-var of pipeline-variable.
- [ ] **APP-03**: Per-user rate-limit (10 req/min default) **én** hard daily-cap $10/dag op Anthropic-uitgave; cap-state in Postgres met `SELECT … FOR UPDATE`-transactie, 90% safety margin, cost-berekening via Anthropic `usage`-block (input + output + cache_creation + cache_read).
- [ ] **APP-04**: Manual security check vóór go-live bewijst dat in Meeting Agent géén Claude-key, géén DB-credential, géén Auth.js-secret en géén sessie-token zichtbaar is in browser/DevTools.

### Management portal

- [ ] **PORTAL-01**: Next.js portal achter Auth.js v5 met `microsoft-entra-id` provider, JWT-session, `trustHost: true`; tenant-mismatch check in middleware blokkeert externe tenants.
- [ ] **PORTAL-02**: Dashboard toont alle geregistreerde apps met test-URL, prod-URL en laatst-bekende status (revision running ja/nee) per Container App.
- [ ] **PORTAL-03**: Admin-pagina laat admin een TAG-gebruiker `Admin` of `Developer` toewijzen; wijziging schrijft naar Postgres + audit-log.
- [ ] **PORTAL-04**: Audit-log tabel is append-only op DB-niveau (REVOKE UPDATE/DELETE voor app-role) en bevat in elk geval `promote_test_to_prod` en `role_change` events met actor, target, timestamp (UTC, `timestamptz`), metadata.

### CI/CD

- [ ] **CICD-01**: Azure DevOps Pipelines bouwen elk artefact (portal, meeting-agent-static, meeting-agent-proxy, infra) op path-triggers, pushen naar ACR met tag = git-sha, deployen naar Container App met `revisionSuffix = git-sha`.
- [ ] **CICD-02**: Trivy `--severity HIGH,CRITICAL --ignore-unfixed` stap blokkeert deploy bij HIGH+ vulnerabilities; `.trivyignore` met expiry-dates ondersteund.
- [ ] **CICD-03**: "Promote test→prod" als aparte pipeline (`promote.yml`) doet **retag van bestaand digest** (niet rebuild) en is alleen triggerbaar door admin (ADO permissions + portal-side rolcheck); promotion schrijft audit-log row.
- [ ] **CICD-04**: ADO service connection gebruikt Workload Identity Federation (geen long-lived SP secret); RBAC scoped op RG-niveau, niet subscription.

### Security

- [ ] **SEC-01**: Alle geheimen in Azure Key Vault; Container Apps gebruiken `secretref:` met Managed Identity; geen secrets in env-vars, pipeline-variables of repo. RBAC (geen access-policies).
- [ ] **SEC-02**: CSP-headers actief op alle web-endpoints (portal + Meeting Agent static). Portal gebruikt nonce-based CSP via middleware. Geen `'unsafe-eval'`.
- [ ] **SEC-03**: HTTPS-only, secure + httpOnly + sameSite=lax cookies, sessie-management via Auth.js v5 defaults; `AUTH_URL` per environment in KV.
- [ ] **SEC-04**: Secret-rotation procedure gedocumenteerd in `docs/runbook.md` (Claude API key, Auth.js secret, DB password). Procedure niet noodzakelijk uitgevoerd in Phase 1.
- [ ] **SEC-05**: `/healthz/secrets` readiness-probe op portal en proxy retourneert 503 wanneer een vereist secret leeg of onbereikbaar is (CRIT-5 mitigatie).

### Documentatie

- [ ] **DOC-01**: `docs/architecture.md` met Mermaid component-diagram, trust-boundaries, dataflows (login, LLM-call, promotion). Kopie van research/ARCHITECTURE.md, aangepast naar implementatie-realiteit.
- [ ] **DOC-02**: `docs/runbook.md` met copy-pasteable commando's voor: deploy vanaf nul, rollback (retag prev sha), secret rotation, incident response (cap-overshoot, KV-access-denied, Postgres-credit-depletion).
- [ ] **DOC-03**: `docs/onboarding.md` zodat een nieuwe dev binnen 1 dag lokaal kan draaien, een test-deploy kan triggeren en weet wie aanspreekpunt is voor Azure-toegang.
- [ ] **DOC-04**: `docs/it-request.md` met exacte resources, rollen, quotas, app-reg-eigenschappen die TAG IT moet provisioneren. Geschreven dag 1, niet aan eind.
- [ ] **DOC-05**: Repo-`README.md` met 10-regel pitch, link naar PROJECT.md, link naar elk van DOC-01..04, beknopte "Hoe doe ik X?"-recepten.

---

## v2 Requirements (deferred to Phase 2)

These will be expected by users once Phase 1 ships. Pilot validates that they are not needed **yet**.

- **CUSTOM-DOMAIN-01**: `apps.tag-team.be` met managed cert per app
- **LOGS-IN-PORTAL-01**: laatste 100 log-regels per app, click-to-tail via Log Analytics API
- **PER-APP-SECRETS-01**: portal-pagina voor app-specifieke secrets, audit-logged
- **VIEWER-ROLE-01**: derde rol `Viewer` (alleen prod gebruiken)
- **COST-DASH-01**: per-app cost + Anthropic-spend dashboard
- **APP-REG-SELF-SVC-01**: developer registreert app-metadata zelf (deploy blijft admin-gated)
- **HEALTH-CHECK-01**: groen/rood status per app via periodieke HTTP-probe
- **ROLLBACK-BTN-01**: één-klik rollback naar laatste-bekende-goed image-tag
- **MULTI-LLM-01**: proxy-abstractie voor Azure OpenAI naast Anthropic
- **AUDIT-HARDEN-01**: hash-chained audit rows + dagelijkse export naar immutable Storage
- **DB-SYNC-01 / NIMBUS-01**: integraties met Nimbus + cross-app DB-sync (origineel Phase-2-onderwerp uit notes)
- **TEAMS-NOTIFY-01**: Teams-channel-notificaties voor promotions + incidenten

---

## Out of Scope (explicit anti-features)

Reden = "prevent re-adding". Niet bouwen in Phase 1, ook niet als shortcut.

- **UI-upload van Dockerfile/zip** — RCE-vector + grote bouw; manual onboarding in Phase 1, self-service in Phase 2 via template-repo's
- **Volledige review/approval-workflow met meerdere reviewers** — Phase 1 = single admin-trigger promote met audit; multi-reviewer pas Phase 2+
- **Native mobile apps** — web-only; mobile = browser-on-phone werkt prima
- **Eigen Container Registry-vervanger** — ACR is goedkoop + secure + MI-integratie; bouwen = verspilling
- **VNET-isolatie, private endpoints, multi-region** — pilot is internal-only, blast-radius beperkt; uitstellen tot er compliance-eis is
- **Per-PR preview deployments (Vercel-style)** — twee long-lived envs per app dekken het pilot-doel
- **Backstage Software Catalog** — past bij ~25 apps, niet bij 1
- **Multi-LLM-provider abstractie** — 3× proxy-code voor nul Phase-1-waarde; Anthropic-only
- **Row-Level Security voor tenant-isolatie** — single-tenant, RLS = onnodige complexiteit
- **Stripe-style metered billing** — interne gebruikers, Excel-chargeback volstaat
- **Public status page / statuspage.io** — Teams-channel dekt interne nood
- **LaunchDarkly / sophisticated feature-flags** — env-var-booleans volstaan voor 2 devs / 1 app
- **OpenTelemetry export naar 3rd-party APM** — App Insights dekt het
- **CAPTCHA / bot-protection** — alles achter Easy Auth, geen anonymous traffic
- **`organizations` of `common` issuer in Entra** — security-regressie; tenant-GUID pinnen

---

## Definition of Done (Phase 1)

Phase 1 is alleen "done" wanneer **alle** v1 requirements `[x]` zijn én:

1. Bicep-deploy vanaf lege RG slaagt in één run (≤ 30 min, `what-if` toont 0 onverwachte deletes).
2. TAG-collega logt in via Microsoft account → ziet portal-dashboard → klikt Meeting Agent → praat met Claude → krijgt antwoord → DevTools toont géén Claude-key.
3. Admin promoot test→prod via portal-knop → audit-row staat in DB binnen 30s.
4. Trivy faalt deploy bij geïnjecteerde HIGH-CVE in test-image.
5. Daily-cap-test: 11× $1 simulated call triggert HTTP 429 + cap-row in DB toont ~$10.
6. KV-rotatie van Claude-key + revision-bump = pilot blijft werken, oude revision wijst naar nieuwe secret-versie binnen 2 minuten.
7. Cost-alert e-mail komt aan na simulated overspend test.
8. Nieuwe dev volgt `docs/onboarding.md` van clean machine → lokaal draaien + test-deploy binnen 4 uur.

---

## Traceability (filled by ROADMAP.md)

| REQ-ID | Phase | Notes |
|--------|-------|-------|
| (filled by gsd-roadmapper) | | |
