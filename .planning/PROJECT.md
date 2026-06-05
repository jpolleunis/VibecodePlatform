# VibeCoding Platform

## What This Is

Intern Azure-gehost platform van Tactical Advisory Group (TAG) waarmee medewerkers "vibecoded" apps (Claude artifacts, mini-tools, kleine internal apps) op een gestructureerde manier live kunnen zetten voor collega's. Levert SSO via een self-hosted Keycloak, applicatie-isolatie via Azure Container Apps, gescheiden test- en productie-omgevingen, en een management-portal voor overzicht en rolbeheer.

In Fase 1 dient het als pilot voor één bewezen pilot-app (Meeting Agent) — het doel is om het hosting- en deploy-patroon end-to-end te valideren voor toekomstige apps van TAG-collega's, mét minimale Azure-vendor-lock-in en lage maandelijkse infra-kost.

## Core Value

Een TAG-collega kan een "vibecoded" interne app gebruiken via een deelbare URL na Microsoft-login, zonder dat de developer geheimen moet lekken of zelf cloud-infrastructuur moet opzetten.

## Requirements

### Validated

<!-- Shipped and confirmed valuable. -->

(None yet — ship to validate)

### Active

<!-- Current scope (Fase 1). Building toward these. -->

**Authentication & Access**
- [ ] **AUTH-01**: Gebruiker logt in op portal en op pilot-apps via SSO door de self-hosted Keycloak (OIDC). Keycloak federate-t naar TAG's Microsoft-tenant indien beschikbaar; anders lokale user-database in Keycloak.
- [ ] **AUTH-02**: Rol-based access met twee rollen: `Admin` (alles) en `Developer` (uploaden + test).
- [ ] **KEYCLOAK-01**: Keycloak draait als Container App met `minReplicas: 1`, Postgres-backed config (zelfde self-host Postgres), realm-config en clients-config geversioneerd in git.

**Infrastructure (Container-Apps-only)**
- [ ] **INFRA-01**: Alle infra gedefinieerd in Bicep (één-bestand, geen AVM-afhankelijkheid vereist), idempotent deploybaar.
- [ ] **INFRA-02**: Eén Container Apps Environment (Consumption), één Storage Account met Azure Files shares voor persistent volumes (Postgres, Keycloak, Docker registry), één Log Analytics workspace (verplicht door ACA, gratis tier).
- [ ] **INFRA-03**: Budget alert op €30/maand voor de Resource Group (target ~€10-15 effectief na free-tier).
- [ ] **INFRA-04**: Naming + tagging-conventie gedocumenteerd en consequent gevolgd.
- [ ] **INFRA-05**: Test en Prod Container Apps per pilot-app met resource-limits (test: 0.25 vCPU / 0.5 GiB / scale-to-zero / max 1; prod: 0.5 vCPU / 1 GiB / `minReplicas: 1` / max 3 / autoscale CPU>70%). LLM-proxy heeft `minReplicas: 1` in beide env's (cap-state correctness).
- [ ] **INFRA-06**: Log Analytics daily-cap 500 MB en 30-dagen retentie; applicatie-logs structureel (JSON via Python `structlog`).
- [ ] **PGSELF-01**: Self-hosted Postgres 16 als Container App, `/var/lib/postgresql/data` op Azure Files-mount. Nightly `pg_dump` naar tweede Azure Files share. Data-loss recovery via TAG's externe Python platform indien volume corrupt raakt.
- [ ] **REGISTRY-01**: Self-hosted Docker registry (`distribution/distribution:3`) als Container App, `/var/lib/registry` op Azure Files-mount. Achter Keycloak OIDC voor `docker login`.
- [ ] **ACASECRETS-01**: Alle geheimen via Container Apps built-in `secrets:` (geen Key Vault). Rotation = revision-bump met nieuwe secret-waarde via pipeline-variable; procedure in runbook.

**Pilot-app — Meeting Agent**
- [ ] **APP-01**: Meeting Agent statische artifact draait in Nginx-container, met een Keycloak-gatekeeper sidecar (`oauth2-proxy` of Python-FastAPI-middleware) ervoor; ongeauthenticeerde bezoekers worden redirected naar Keycloak.
- [ ] **APP-02**: Python FastAPI LLM-proxy met exact één endpoint `POST /api/llm`; key voor Anthropic Claude API komt uit ACA `secrets:` (env-var), niet uit applicatie-code.
- [ ] **APP-03**: Per-user rate-limit + hard daily-cap $10/dag op Anthropic-uitgave; cap-state in self-host Postgres met `SELECT … FOR UPDATE`-transactie, 90% safety margin, cost-berekening via Anthropic `usage`-block.
- [ ] **APP-04**: Manual security check vóór go-live bewijst dat in Meeting Agent géén Claude-key, géén DB-credential en géén sessie-token zichtbaar is in browser/DevTools.

**Management portal**
- [ ] **PORTAL-01**: Python FastAPI portal met server-rendered HTML (Jinja2 + HTMX, geen JS-framework), achter Keycloak OIDC.
- [ ] **PORTAL-02**: Dashboard toont alle geregistreerde apps met test-URL, prod-URL en laatst-bekende status per Container App.
- [ ] **PORTAL-03**: Admin-pagina laat admin een TAG-gebruiker `Admin` of `Developer` toewijzen; wijziging schrijft naar Postgres + audit-log.
- [ ] **PORTAL-04**: Audit-log tabel is append-only op DB-niveau (REVOKE UPDATE/DELETE voor app-role) en bevat `promote_test_to_prod` en `role_change` events met actor, target, timestamp (UTC `timestamptz`), metadata.

**CI/CD**
- [ ] **CICD-01**: Azure DevOps Pipelines bouwen elk artefact (portal, meeting-agent-static, meeting-agent-proxy, infra) op path-triggers, pushen naar de self-host registry, deployen naar Container App met `revisionSuffix = git-sha`.
- [ ] **CICD-02**: Trivy `--severity HIGH,CRITICAL --ignore-unfixed` stap blokkeert deploy bij HIGH+ vulnerabilities; `.trivyignore` met expiry-dates ondersteund.
- [ ] **CICD-03**: "Promote test→prod" als aparte pipeline doet **retag van bestaand digest in self-host registry** (niet rebuild) en is alleen triggerbaar door admin; promotion schrijft audit-log row.
- [ ] **CICD-04**: ADO service connection gebruikt simple Service Principal secret (geen WIF — past niet bij self-host strategie); secret 90-daags rotatable via pipeline-variable.

**Security**
- [ ] **SEC-01**: Alle geheimen in Container Apps `secrets:`. Geen secrets in env-vars op image-niveau, geen secrets in repo, geen secrets in pipeline-logs.
- [ ] **SEC-02**: CSP-headers actief op alle web-endpoints. Portal gebruikt strict CSP via FastAPI-middleware. Geen `'unsafe-eval'`.
- [ ] **SEC-03**: HTTPS-only (ACA default), secure + httpOnly + sameSite=lax cookies, sessie-management via Authlib (OIDC) of FastAPI session middleware.
- [ ] **SEC-04**: Secret-rotation procedure gedocumenteerd in `docs/runbook.md` (Claude API key, Keycloak admin password, Postgres password, ACA-revision-bump-flow). Niet noodzakelijk uitgevoerd in Fase 1.
- [ ] **SEC-05**: `/healthz/secrets` readiness-probe op portal en proxy retourneert 503 wanneer een vereist secret leeg of onbereikbaar is.

**Documentatie**
- [ ] **DOC-01**: `docs/architecture.md` met Mermaid component-diagram, trust-boundaries, dataflows.
- [ ] **DOC-02**: `docs/runbook.md` met copy-pasteable commando's voor: deploy vanaf nul, rollback, secret rotation, incident response.
- [ ] **DOC-03**: `docs/onboarding.md` zodat een nieuwe dev binnen 1 dag lokaal kan draaien, een test-deploy kan triggeren en weet wie aanspreekpunt is voor Azure-toegang.
- [ ] **DOC-04**: `docs/it-request.md` met exacte resources, rollen, quotas die TAG IT moet provisioneren (vereenvoudigd — alleen RG + ACA env + Storage Account + ADO).
- [ ] **DOC-05**: Repo-`README.md` met 10-regel pitch, link naar PROJECT.md, link naar elk van DOC-01..04.
- [ ] **DOC-06**: `docs/management.md` — management-facing architecture + user stories + timeline + Container-Apps-only kost.

### Out of Scope

<!-- Fase 1-grenzen. -->

- **Azure managed PaaS-services** (Azure DB for Postgres, Key Vault, Container Registry, App Insights, AAD app-registration, Easy Auth, WIF) — bewust gedropt om vendor lock-in en kost te beperken; alles self-built in Python of self-hosted via open-source images.
- **Developer self-service upload van nieuwe apps via UI** — Fase 2; risico op RCE-vector
- **Volledige review/approval-workflow voor app-launches** — Fase 2
- **Nimbus-integratie + database-sync tussen apps** — Fase 2 (let op: Nimbus is wel de externe Python-recovery-laag voor data-loss-scenarios, maar de integratie zelf is Fase 2)
- **Viewer-rol** — Fase 1 heeft enkel Admin + Developer
- **Custom domain** — pilot gebruikt default `*.azurecontainerapps.io`
- **VNET-isolatie, private endpoints, multi-region failover**
- **Andere LLM-providers** — pilot gebruikt enkel Anthropic Claude API direct
- **Native mobile apps**
- **Managed Postgres backup/PITR** — accepteren beperkter data-recovery, mitigated via nightly pg_dump + extern recovery-platform
- **Multi-tenancy / RLS / billing** — interne single-tenant pilot

## Context

- **Organisatie**: Tactical Advisory Group (TAG). Microsoft-stack standaard maar bewust beperken tot Container Apps + Storage voor deze pilot.
- **Aanleiding**: TAG-medewerkers maken "vibecoded" tools in Claude. Probleem: deze blijven in de Claude UI. Doel = "interne app met deelbare URL".
- **Bestaande artifacts**: Meeting Agent (statische HTML/JS + Claude API-call vanuit browser — security-issue, in Fase 1 opgelost via backend-proxy).
- **TAG IT context**: Azure-tenant + subscription bestaan. Developer heeft beperkte rechten; aanvraag voor RG + Storage Account + ACA Env + ADO-project.
- **Team**: 2 developers (Jasper Polleunis + 1) met Claude Code als pair-programmer.
- **Externe data-recovery**: TAG heeft een afzonderlijk Python-platform waarmee data-loss-scenarios worden afgehandeld; dit ontslaat ons van managed-DB-backup-eisen in Fase 1.
- **Prior repo**: TAGAI skills-repo. Platform-code leeft in `C:\Users\JasperPolleunis\vibecoding-platform`.

## Constraints

- **Tech stack (vastgelegd na rework)**:
  - Hosting: **Azure Container Apps** (Consumption profile, één Environment, suffix-naming voor test/prod). Géén andere Azure-managed-PaaS.
  - Auth: **Self-hosted Keycloak 26** als Container App, OIDC. Postgres-backed config. Optioneel: federation met Microsoft-tenant.
  - Portal: **Python 3.12 + FastAPI + Jinja2 + HTMX**. SQLAlchemy 2 (async) + asyncpg. Authlib voor OIDC. Pino-equivalent: `structlog`.
  - Pilot-app proxy: **Python 3.12 + FastAPI + `@anthropic-ai/sdk` Python**.
  - Database: **Self-hosted Postgres 16** in Container App, `/var/lib/postgresql/data` op Azure Files. Nightly pg_dump → tweede Azure Files share.
  - Container Registry: **Self-hosted Docker registry** (`distribution/distribution:3`) achter Keycloak OIDC. Azure Files voor opslag.
  - Secrets: **Container Apps built-in `secrets:`** + ACA-revision-bump voor rotation.
  - Logging: Azure Log Analytics workspace (verplicht door ACA, 5 GB/maand free tier).
  - IaC: **Bicep (single-file, geen AVM)**.
  - CI/CD: **Azure DevOps Pipelines** met SP-secret service connection (rotation in pilot Phase 5).
  - LLM-provider: Anthropic Claude API direct.
- **Timeline**: Fase 1 streefdoel **~5.5 weken** (eerlijke schatting na self-host scope) met 2 devs + Claude Code.
- **Budget**:
  - Azure: target ~€10-15/maand effectief; cost-alert op €30/maand (worst case). Jaar-target: <€200.
  - Claude API: hard daily-cap $10/dag in proxy.
- **Security (niet-onderhandelbaar)**:
  - Geen geheimen in browser of repo.
  - Self-host registry achter OIDC (geen publieke images).
  - CSP, HTTPS-only, secure cookies.
  - Trivy scan fail bij HIGH+.
- **Dependencies**:
  - TAG IT: RG + Storage Account + ACA Env quota + ADO org/project.
  - Anthropic API-key (TAG-account).
- **Documentatie**: Elk fase-deliverable bevat bijbehorende docs.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Azure Container Apps als enige Azure-managed-PaaS | Vendor lock-in beperken, kost ~€10-15/maand i.p.v. €100, jaarbudget <€200 | — Pending |
| Python (FastAPI) over Next.js voor portal + proxy | Sneller MVP, kleinere images, beste Anthropic SDK-fit voor proxy, team-leerbaar | — Pending |
| Self-host Keycloak voor SSO (i.p.v. Azure AD app-reg) | Geen Azure AD-licentie/consent-flow nodig; portable; OIDC standaard | — Pending |
| Self-host Postgres in Container App + Azure Files | Geen managed-DB-kost; externe Python-platform handelt data-loss af | — Pending |
| Self-host Docker registry achter Keycloak | Geen ACR-kost; images blijven niet publiek | — Pending |
| Container Apps `secrets:` i.p.v. Key Vault | Built-in, geen extra resource; rotation = revision-bump | — Pending |
| Bicep single-file (geen AVM-modules) | Minder vendor-binding aan AVM-versies; één leesbaar bestand | — Pending |
| ADO Service Principal secret i.p.v. WIF | Past bij self-host-strategie; rotation in runbook | — Pending |
| Anthropic Claude API direct | TAG gebruikt Claude al | — Pending |
| LLM-proxy in Fase 1 (geen API-key in browser) | Security-non-negotiable | — Pending |
| Geen developer self-service upload in Fase 1 | RCE-risico, Fase 2 zinvol | — Pending |
| Geen Viewer-rol in Fase 1 | Pattern blijft mogelijk voor later | — Pending |
| Timeline herzien naar ~5.5 weken | Eerlijk over self-host scope; was 2 weken | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check
3. Audit Out of Scope
4. Update Context

---
*Last updated: 2026-05-30 after Sessie 2 — scope reduction (Container-Apps-only + Python rebuild)*
