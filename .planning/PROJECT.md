# VibeCoding Platform

## What This Is

Intern Azure-gehost platform van Tactical Advisory Group (TAG) waarmee medewerkers "vibecoded" apps (Claude artifacts, mini-tools, kleine internal apps) op een gestructureerde manier live kunnen zetten voor collega's. Levert SSO via Azure AD, applicatie-isolatie via Azure Container Apps, gescheiden test- en productie-omgevingen, en een management-portal voor overzicht en rolbeheer.

In Fase 1 dient het als pilot voor één bewezen pilot-app (Meeting Agent) — het doel is om het hosting- en deploy-patroon end-to-end te valideren voor toekomstige apps van TAG-collega's.

## Core Value

Een TAG-collega kan een "vibecoded" interne app gebruiken via een deelbare URL na Microsoft-login, zonder dat de developer geheimen moet lekken of zelf cloud-infrastructuur moet opzetten.

## Requirements

### Validated

<!-- Shipped and confirmed valuable. -->

(None yet — ship to validate)

### Active

<!-- Current scope (Fase 1). Building toward these. -->

**Authentication & Access**
- [ ] **AUTH-01**: Gebruiker logt in op portal en op pilot-apps via Azure AD SSO (TAG-tenant)
- [ ] **AUTH-02**: Rol-based access met twee rollen: Admin (alles), Developer (uploaden + test)
- [ ] **AUTH-03**: Azure AD app-registration gedocumenteerd en herhaalbaar opzetbaar

**Infrastructure (IaC)**
- [ ] **INFRA-01**: Alle Azure resources gedefinieerd in Bicep, idempotent deploybaar
- [ ] **INFRA-02**: Eén Container Apps Environment, ACR, Key Vault, Log Analytics, Postgres Flex
- [ ] **INFRA-03**: Cost-budget alert op €100/maand voor Resource Group
- [ ] **INFRA-04**: Naming + tagging-conventie gedocumenteerd en gevolgd
- [ ] **INFRA-05**: Test + Prod Container Apps per app met de resource-limits uit ontwerpnotities (test: 0.5 vCPU / 1GB / scale-to-zero / max 1 replica; prod: 2 vCPU / 4GB / min 1 replica / max 5 / auto-scale CPU>70%)

**Pilot-app: Meeting Agent**
- [ ] **APP-01**: Meeting Agent (statische Claude-artifact) draait in Nginx-container achter Container Apps Easy Auth
- [ ] **APP-02**: Node Express LLM-proxy (één endpoint `/api/llm`) roept Anthropic Claude API met server-side key (key in Key Vault, ingelezen via managed identity)
- [ ] **APP-03**: LLM-proxy heeft per-gebruiker rate-limit + hard daily-cap van $10/dag op Claude-API-uitgave
- [ ] **APP-04**: Geen Claude-key, geen DB-credential, geen sessie-secret zichtbaar in browser/devtools

**Management portal**
- [ ] **PORTAL-01**: Next.js portal met login via Auth.js + Azure AD provider
- [ ] **PORTAL-02**: Dashboard toont alle geregistreerde apps + status (test/prod) + URL's
- [ ] **PORTAL-03**: Admin-pagina voor rol-toewijzing aan TAG-gebruikers
- [ ] **PORTAL-04**: Audit-log tabel registreert promotions test→prod en role-changes (wie, wat, wanneer)

**CI/CD**
- [ ] **CICD-01**: Azure DevOps Pipelines bouwen elk artefact (portal, meeting-agent-static, meeting-agent-proxy, infra), pushen naar ACR, deployen naar Container App
- [ ] **CICD-02**: Trivy image-scan in pipeline; fail bij HIGH+ vulnerabilities
- [ ] **CICD-03**: "Promote test→prod" als aparte pipeline-trigger, alleen aan te roepen door admin (via portal of pipeline-permissions)

**Security**
- [ ] **SEC-01**: Alle secrets in Azure Key Vault; Container Apps gebruiken managed identity, geen secrets in env-vars of pipeline-logs
- [ ] **SEC-02**: CSP-headers actief op alle web-endpoints (portal + pilot-apps)
- [ ] **SEC-03**: HTTPS-only, secure + httpOnly cookies, sessie-management via Auth.js defaults
- [ ] **SEC-04**: Secrets-rotation procedure gedocumenteerd (uitvoer niet vereist in Fase 1)

**Documentatie (eis: alles vindbaar voor de toekomst)**
- [ ] **DOC-01**: Architecture-doc met component-diagram + dataflow per fase (`docs/architecture.md`)
- [ ] **DOC-02**: Runbook voor deploy, rollback, secret-rotation, incident-response (`docs/runbook.md`)
- [ ] **DOC-03**: Onboarding-guide voor nieuwe dev/contributor — lokale dev-setup, repo-structuur, Azure-toegang vragen (`docs/onboarding.md`)
- [ ] **DOC-04**: IT-aanvraag-checklist met exacte resources, rollen, quotas die TAG IT moet provisioneren (`docs/it-request.md`)
- [ ] **DOC-05**: README in repo-root met 10-regel pitch + verwijzingen naar bovenstaande docs

### Out of Scope

<!-- Fase 1-grenzen. Bewust niet in scope om scope-creep te voorkomen. -->

- **Developer self-service upload van nieuwe apps via UI** — geplanned voor Fase 2; risico op RCE-vector + grote bouw; in Fase 1 = handmatig door admin
- **Volledige review/approval-workflow voor app-launches** — Fase 2; Fase 1 = enkel test→prod promotion-knop
- **Nimbus-integratie + database-sync tussen apps** — Fase 2; niet nodig voor pilot
- **Viewer-rol** — Fase 1 heeft enkel Admin + Developer; pattern voor Viewer blijft mogelijk, niet nodig nu
- **Custom domain (bv. `apps.tag-team.be`)** — pilot gebruikt default `*.azurecontainerapps.io`; custom domain + cert = Fase 1.5/2
- **VNET-isolatie, private endpoints, multi-region failover** — pilot is internal-only, blast-radius beperkt
- **Andere LLM-providers (Azure OpenAI, OpenAI, lokaal)** — pilot gebruikt enkel Anthropic Claude API direct
- **Native mobile apps** — web-only
- **Eigen Container Registry-vervanger** — ACR is goedkoop, secure, managed; bouwen = verspilling

## Context

- **Organisatie**: Tactical Advisory Group (TAG), B2B-advies. Microsoft-stack standaard; TAG-tenant = `tag-team.be`.
- **Aanleiding**: TAG-medewerkers maken regelmatig "vibecoded" tools in Claude (artifacts, kleine apps). Probleem: deze blijven in de Claude UI, geen deelbare URL, geen auth, geen versie-control, geen herhaalbare deploy. Resultaat = waarde-creatie blijft hangen.
- **Bestaande artifacts**: Meeting Agent (pilot-app, statische HTML/JS + Claude API-call vanuit browser — security-issue dat in Fase 1 opgelost wordt via backend-proxy).
- **TAG IT context**: Azure-tenant bestaat. Subscription bestaat. Developer heeft beperkte rechten — IT-aanvraag nodig voor Resource Group, Contributor-rol, ACR, Postgres-quota, Azure DevOps-org, Service Principal/Workload Identity Federation.
- **Team**: 2 developers (Jasper Polleunis + 1) met Claude Code als pair-programmer. Maakt agressievere timeline haalbaar.
- **Pilot-doelgroep**: Jasper + 1 dev-collega als testers. Brede TAG-uitrol = pas na pilot-validatie.
- **Prior repo**: TAGAI skills-repo (`C:\Users\JasperPolleunis\TAGAI`) bevat Claude-skills, niet de platform-codebase. Platform-code leeft in `C:\Users\JasperPolleunis\vibecoding-platform`.

## Constraints

- **Tech stack (vastgelegd)**:
  - Hosting: Azure Container Apps (één Environment, suffix-naming voor test/prod)
  - Auth: Azure AD app-registration + Container Apps Easy Auth + Auth.js (NextAuth) voor portal
  - Portal: Next.js + TypeScript + Tailwind + shadcn/ui + Prisma + PostgreSQL (Azure DB for PostgreSQL Flex)
  - Pilot-app proxy: Node Express
  - Container Registry: Azure Container Registry (Basic SKU)
  - Secrets: Azure Key Vault + Managed Identity
  - Logging: Azure Log Analytics workspace (auto-gekoppeld aan Container Apps Env)
  - IaC: Bicep
  - CI/CD: Azure DevOps Pipelines (Repos + Pipelines)
  - LLM-provider: Anthropic Claude API direct (geen Azure OpenAI in Fase 1)
- **Timeline**: Fase 1 streefdoel ~2 werkweken. Realistisch met 2 devs + Claude Code, mits IT-aanvraag op dag 1.
- **Budget**:
  - Azure: cost-alert op €100/maand voor pilot-RG. Verwacht idle ~€60-80, normaal ~€100-150.
  - Claude API: hard daily-cap $10/dag in proxy. Per-user rate-limit op `/api/llm`.
- **Security (niet-onderhandelbaar)**:
  - Geen geheimen in browsercode of repo
  - Managed identity overal waar mogelijk; geen service-principal-keys in pipeline-vars
  - CSP-headers op alle web-endpoints
  - Trivy-image-scan fail bij HIGH+ vulns
  - HTTPS-only
- **Dependencies**:
  - TAG IT moet RG + rechten + Azure AD app-reg-toestemming + ADO-org/project provisioneren voor werk begint
  - Anthropic API-key (TAG-account) nodig voor proxy
- **Documentatie**: Elk fase-deliverable bevat bijbehorende docs (architecture, runbook, onboarding). Code zonder docs = niet "done".

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Azure Container Apps i.p.v. AKS/App Service | Container per app = goede isolatie, scale-to-zero, simpel; AKS = overkill pilot | — Pending |
| Azure DevOps i.p.v. Bitbucket/GitHub | Native Azure-integratie (service connections, ACR-task, Container Apps deploy-task) | — Pending |
| Eén Container Apps Environment, suffix-naming voor test/prod | Goedkoopst; resource-limits per app geven isolatie; volstaat pilot | — Pending |
| Azure AD app-registration + Easy Auth (geen eigen login-code voor pilot-apps) | Eigen auth bouwen = security-risico; Easy Auth = zero-code SSO voor de app zelf | — Pending |
| Next.js + Auth.js voor portal (i.p.v. .NET, Alpine, pure backend) | Eén deploy-unit, sterke Azure AD-integratie, TypeScript = minder runtime-bugs, schaalt naar Fase 2 | — Pending |
| Bicep IaC vanaf dag 1 (geen portal-clicks) | Reproduceerbaar, snel itereren, redeploy in minuten | — Pending |
| Anthropic Claude API direct (geen Azure OpenAI) | TAG gebruikt Claude al voor vibecoding; geen extra service nodig in pilot | — Pending |
| ACR gebruiken (geen eigen registry) | Bouwen = verspilling; ACR = goedkoop, secure, managed-identity-integratie | — Pending |
| LLM-proxy in Fase 1 (geen API-key in browser) | Key-in-browser = lek-risico zelfs intern; proxy-patroon valideert backend-stack voor toekomstige apps | — Pending |
| Geen developer self-service upload in Fase 1 | UI-upload van Dockerfile/zip = RCE-vector + grote bouw; Fase 2 zinvol | — Pending |
| Geen Viewer-rol in Fase 1 | Pilot heeft alleen Admin + Developer; pattern blijft mogelijk voor later | — Pending |

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
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-05-19 after initialization*
