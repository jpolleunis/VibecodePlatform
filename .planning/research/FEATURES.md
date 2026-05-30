# Feature Research

**Domain:** Internal "vibecoding" platform / mini-PaaS / internal app-launcher (TAG-internal Azure-hosted)
**Researched:** 2026-05-19
**Confidence:** HIGH (synthesis of strong prior knowledge on Vercel/Render/Railway/fly.io/Heroku/Backstage/Coolify/Dokploy + verification searches; matches PROJECT.md Active scope and constraints)

## Context Framing

This is an **internal** platform. There are no external customers and no competitors to outflank. "Users" means:

- **TAG developers** (2 in pilot, ~5-10 long-term): the people who build and ship the vibecoded apps. They want zero infra friction, no secret-handling, fast feedback.
- **TAG end-users** (the rest of TAG): colleagues who consume the apps via a Microsoft-login URL. They want apps that just work, look trustworthy, and don't crash.
- **Admin** (1 in pilot, likely Jasper): the platform operator. Wants visibility, control over who can do what, and ability to stop bleeding (cost, broken app, leaked secret) fast.

"Differentiators" in an internal context = features that make TAG developers and end-users actively *love* this versus tolerating it. They are not competitive moats; they are adoption multipliers.

## Feature Landscape

### Table Stakes — Pilot (Phase 1, must ship in 2 weeks)

If any of these are missing, the pilot fails to validate the hosting/deploy pattern.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| **SSO via Azure AD on every endpoint** (portal + apps) | This is the entire reason the platform exists vs. Claude artifacts. End-users just click a link, see Microsoft login, land on the app. | S | Container Apps Easy Auth = zero code for the apps. Auth.js + Entra ID provider for portal. Maps to AUTH-01. |
| **Two-role RBAC: Admin + Developer** | Without it, anyone can do anything = no audit story, no IT sign-off. Two roles is the minimum to demonstrate the pattern. | S | Role stored in portal DB (Postgres via Prisma), checked in portal server actions. Maps to AUTH-02. Viewer-role explicitly out of scope (PROJECT.md). |
| **Test + Prod environment separation per app** | Devs need to try changes without breaking prod. Promote step is the "release process" that justifies the platform. | M | Two Container Apps per app, suffix-naming, separate resource limits per PROJECT.md INFRA-05. |
| **Server-side secrets (Key Vault + Managed Identity)** | The single most important security promise. Without this, devs paste keys into HTML and we've built nothing. | M | Bicep-provisioned Key Vault, Container Apps reference secrets via managed identity. Maps to SEC-01, APP-04. |
| **LLM-proxy backend pattern** | Pilot-app (Meeting Agent) is a static UI + LLM call. Browser calls `/api/llm`, Node Express proxies to Claude with server-side key. This *is* the pattern that all future apps follow. | M | Express + Anthropic SDK, ~150 LOC. One endpoint. Maps to APP-02. |
| **Per-user rate-limit + daily LLM spend cap** | Without this, one buggy app or one curious user can burn the Anthropic budget in an afternoon. Non-negotiable for "cost-bounded". | M | In-memory or Postgres counter per user/day, hard fail at threshold. Maps to APP-03. |
| **Management portal: app dashboard** | Admin needs *one* place to see "what is deployed, where, is it up". Without it, the platform is invisible. | M | Next.js page reading from Postgres app-registry + Container Apps URL status (or cached). Maps to PORTAL-02. |
| **Management portal: admin role assignment** | Admin must be able to grant Developer-role without going to Azure portal. This is the only "self-service" action in Phase 1. | S | Page lists TAG users (cached from Entra), checkbox grants role, writes to portal DB. Maps to PORTAL-03. |
| **Audit log: promotions + role changes** | IT compliance story. "Who promoted to prod yesterday at 14:32?" must be answerable. Two event types is enough. | S | Append-only Postgres table, written from portal server actions and promote-pipeline. Maps to PORTAL-04. |
| **Infrastructure-as-Code (Bicep) for every resource** | Manual Azure portal clicks = irreproducible. Lose the RG and you lose the platform. Pilot must prove redeploy from zero. | M | All resources in Bicep, idempotent, one-command deploy. Maps to INFRA-01/02. |
| **CI/CD with image scanning** | Without pipelines the "ship to prod" story is "Jasper SSHes in". Trivy gate is one-line config and catches HIGH+ CVEs. | M | Azure DevOps Pipelines + ACR + Container Apps deploy task. Maps to CICD-01/02. |
| **Promote test→prod as a distinct trigger** | The release process. Admin-gated. Tied to audit log. | S | Separate pipeline or pipeline stage with approval. Maps to CICD-03. |
| **Cost budget alert** | One slip and the €100/month becomes €1000. Native Azure feature, ~5 minutes. | S | Bicep `Microsoft.Consumption/budgets`. Maps to INFRA-03. |
| **HTTPS-only + CSP headers + secure cookies** | Bare-minimum web security; if missing, IT will refuse sign-off. | S | Container Apps does HTTPS for free; CSP via Next.js config + nginx config for static app. Maps to SEC-02/03. |
| **No secrets in browser code / DevTools** | The pilot-app exists because Meeting Agent today *does* leak keys. Fixing this *is* the success criterion. | S (consequence of other features) | Validated by manual check in pilot acceptance. Maps to APP-04. |
| **Architecture doc** | "What is this thing?" — without this, no one new can ever contribute. | S | One markdown file, one diagram, dataflow per phase. Maps to DOC-01. |
| **Runbook** | When prod breaks at 17:30 on a Friday, the runbook is what saves the platform's reputation. Deploy, rollback, secret-rotation, incident-response. | S | One markdown file, copy-paste-able commands. Maps to DOC-02. |
| **Onboarding guide** | A second dev (and any future dev) must be able to clone + run locally + ship a test deploy by reading one file. | S | Maps to DOC-03. |
| **IT-request checklist** | TAG IT will not provision Azure resources from a verbal request. A concrete list of resources/roles/quotas un-blocks dependencies on day 1. | S | Maps to DOC-04. Possibly the highest-leverage doc in Phase 1. |
| **README in repo root** | First thing every visitor sees. Pitch + pointers to the four docs above. | S | Maps to DOC-05. |

### Table Stakes — v2 (post-pilot, users will expect these once they exist)

Defer to Phase 2. Pilot validates that they're not needed *yet*; v2 is when usage exceeds 2 devs / 1 app.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| **Custom domain** (e.g. `apps.tag-team.be`) | Default `*.azurecontainerapps.io` URLs feel like prototypes. Once users share links with clients-adjacent colleagues, branded domain matters. | S | Container Apps supports managed certs; Bicep + DNS. Explicitly Phase 1.5/2 per PROJECT.md. |
| **App-level logs visible in portal** | Devs currently must open Azure portal → Log Analytics → KQL. v2: portal shows last 100 lines per app, click-to-tail. | M | Query Log Analytics API, render in shadcn table. |
| **Per-app secret management UI** | Pilot has one app, one secret (Claude key). v2 with N apps means devs need to register their own per-app secrets without touching Key Vault directly. | M | Portal page → calls Key Vault REST via managed identity, audit-logged. |
| **Viewer role** | Once non-dev colleagues want to see app status without being able to change anything. Pattern preserved in PROJECT.md. | S | Add to role enum, gate writes. |
| **Per-app cost & usage dashboard** | "How much is Meeting Agent costing this month?" — currently answerable only via Azure cost mgmt. | M | Aggregate Anthropic spend (proxy logs) + Azure cost API per resource tag. |
| **Self-service app registration** (limited) | Devs register a new app's metadata (name, owner, repo URL) without admin involvement. Deploy still admin-gated. | M | Portal form → writes to registry, triggers infra job. |
| **Health-check status display** | "Is Meeting Agent up?" answered with a green/red dot, not by clicking the URL. | S | HTTP probe every N minutes from a Container Apps job, store in DB. |
| **Rollback button in portal** | Today rollback = re-run pipeline with previous image tag. v2: one-click rollback to last-known-good. | M | Track previous prod image tag per app, expose button, audit log. |
| **Multi-LLM-provider support** (Azure OpenAI + Anthropic) | Once cost or compliance forces a switch, the proxy must abstract this. | M | Proxy interface, provider-specific adapter. |
| **Email/Teams notification on promotion + budget breach** | Pilot relies on Jasper noticing. v2 needs proactive alerts. | S | Webhook to Teams channel from pipeline + budget alert. |

### Differentiators (TAG-loving features — adoption multipliers, not competitive moats)

These make TAG developers prefer this over "I'll just paste the Claude artifact in a OneNote". Pick selectively — most defer to Phase 2.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| **"Drop your Claude artifact, get a URL" deploy template** | A repo template (e.g. `vibe-app-template`) where a dev pastes a Claude artifact HTML, adds a one-line `app.yml`, pushes, and gets a live URL in <5 minutes. This is what would make TAG developers actually use it. | M | Repo template + Azure DevOps starter pipeline + standard nginx Dockerfile. Phase 2 candidate, very high adoption value. |
| **First-class LLM-proxy as platform primitive** | Every vibecoded TAG app needs LLM calls. Making the proxy a reusable shared service (one URL, scoped tokens per app) means new apps need *only* a UI — no backend code. | L | Multi-tenant proxy, per-app scoped tokens, per-app spend caps. Phase 2/3. Differentiator because no public PaaS gives you this. |
| **Cost transparency per dev / per app / per LLM call** | Vercel/Heroku give you bills. TAG devs would love a portal page that says "Your Meeting Agent has cost €4.20 in Claude API this week, here's the top 5 most expensive prompts." | M | Anthropic API responses include `usage` tokens; proxy logs them. Aggregated view. Phase 2. |
| **Built-in "share with internal user" link** | Portal shows each app with a copy-to-clipboard "share with TAG" button + optional "require login" or "open to whole tenant" toggle. Removes ambiguity for non-technical owners. | S | Already implicit in PORTAL-02; explicitly making it a one-click action elevates UX. Phase 1 stretch. |
| **App owner field + "who do I bug" surface** | Each app shows its owner (TAG dev) in the portal so a colleague using Meeting Agent knows who to message. Trivial, hugely valuable for trust. | S | One column in app registry table. Phase 1 stretch (cheap). |
| **"Suggested app" review queue** | TAG colleagues submit "I wish there was a tool for X" requests via portal; devs see queue, claim, build. Closes the loop on vibecoding-as-internal-product. | M | Phase 3. Cultural feature more than technical. |
| **Templates for known app shapes** (static artifact, static + LLM proxy, full-stack + DB) | Three Bicep + repo templates covering 90% of TAG vibe-apps. Pick a template, fill in name, ship. | M | Phase 2. Builds on the pilot's Meeting Agent pattern. |
| **Audit-log export to TAG security / compliance Teams channel** | Once IT trusts this, exporting promotion events to a SOC-style channel removes friction from future security reviews. | S | Webhook from audit-log table. Phase 2. |

### Anti-Features (Commonly tempting in a vibecoding platform — deliberately NOT in Phase 1)

These all look productive but compound complexity, security risk, or scope. Documented per PROJECT.md Out of Scope plus the patterns we see across competitor analysis.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| **UI upload of Dockerfiles / zips by developers** | "Devs should be able to ship without pipelines!" Feels self-service-y. | RCE vector (uploaded Dockerfile runs as root in build agent). Large UI + sandboxing scope. Removes the audit story (no git commit = no traceability). | Admin pushes to git repo on behalf of dev in Phase 1; Phase 2 = git-based templates with admin approval. Already in PROJECT.md Out of Scope. |
| **Custom domain in Phase 1** | `*.azurecontainerapps.io` looks unprofessional. | Cert provisioning, DNS dependency on TAG IT, adds first-deploy delay by days. Pilot users (2 devs + 1 admin) don't care. | Defer to Phase 1.5/2 once pilot validates. Already in PROJECT.md Out of Scope. |
| **VNET isolation, private endpoints, multi-region** | "It's enterprise, so it must be locked down." | All-internal apps already behind SSO + Easy Auth. VNET adds Bicep complexity, debugging friction, often blocks Container Apps features. €€€. | Public ingress + SSO gate is the right blast-radius for pilot. Already PROJECT.md Out of Scope. |
| **Multiple LLM providers in Phase 1** | "What if Anthropic goes down / gets too expensive?" | Each provider has different request/response shape, different auth, different rate-limit semantics. Multi-provider = 3x the proxy code for zero pilot value. | Anthropic only in Phase 1. Abstract provider in Phase 2 when there is a real reason. PROJECT.md Out of Scope. |
| **Real-time everything (live logs, live metrics, websockets in portal)** | Looks cool. Feels modern. | WebSockets through Container Apps + Auth.js + reverse-proxy = surprising amount of work. Pilot needs "refresh the page", not Grafana. | Static page that re-fetches on load. Phase 2 = polling. Phase 3 = streaming if anyone actually asks. |
| **App marketplace / discovery feed** | "Other TAG devs should see what I built!" | Premature. With 1 app there's nothing to discover. With 5 apps, a static dashboard list works fine. | Dashboard table is the marketplace for now. Re-evaluate at >10 apps. |
| **Generic "build any container image" support (Buildpacks/Nixpacks)** | Coolify/Dokploy have it; users will assume parity. | Buildpack abstraction is a leaky moat once anyone needs apt-get. TAG vibecoded apps are 90% static HTML + small Node backend — Dockerfile per app is fine and explicit. | Provide a canonical Dockerfile in the repo template. Phase 2 consider Nixpacks if pain emerges. |
| **AKS / Kubernetes-native scheduling** | "Real platforms use K8s." | Container Apps is Kubernetes-under-the-hood without the operator tax. Going to AKS adds a sysadmin role no one in TAG has. | Stay on Container Apps. Already PROJECT.md Key Decision. |
| **App-level usage analytics (page views, click tracking)** | "We want to know who's using what." | Privacy footgun (TAG colleagues being tracked by their internal IT). Requires consent banner. Adds a JS bundle to every static app and undermines the "paste your artifact" simplicity. | Server-side request log per app (just count requests per user) is enough. Defer JS analytics indefinitely. |
| **Self-hosted Coolify/Dokploy on a VPS instead of Azure Container Apps** | "Open-source PaaS! Cheaper!" | TAG IT is Azure-standard. Running a VPS means TAG IT now owns OS patching, backups, monitoring of *another* surface. Adds an org-political surface. | Azure Container Apps is the right call for this org. Already PROJECT.md Key Decision. |
| **Eigen identity provider / username-password fallback** | "What if Entra ID is down?" | Entra ID being down means TAG is not working anyway. Adds a high-risk codepath (password hashing, reset flows) that almost never executes. | Entra ID only. If outage = read the runbook. |
| **Database-as-a-service per app (1-click Postgres)** | Coolify/Dokploy / Railway have this. | Pilot app doesn't need it. Adding per-app Postgres = Bicep template explosion + backup story + cost. | Phase 2 — one shared Postgres flex, schema-per-app. Or per-app when a real need appears. Pilot Meeting Agent is stateless. |
| **Approval workflow / multi-stage gates** | "Big companies have CAB!" | Promote-to-prod button + audit log = sufficient governance for 2 devs. CAB on a 5-person platform is theatre. | Single approval gate in promote pipeline. Already PROJECT.md Out of Scope (full workflow). |
| **GitOps / declarative app registry (Backstage-style catalog YAMLs)** | "Backstage does this!" | Backstage solves the problem of 200 microservices. TAG has 1. Catalog files are friction-without-payoff at this scale. | Postgres-backed app registry edited via portal UI. Re-evaluate if app count crosses ~25. |

## Feature Dependencies

```
Azure AD app-registration (AUTH-03)
    +-- Easy Auth on pilot-app (APP-01)
    +-- Auth.js login on portal (PORTAL-01)
            +-- Role-based access (AUTH-02)
                    +-- Admin role-assignment page (PORTAL-03)
                    +-- Audit log (PORTAL-04)
                    +-- Admin-gated promote pipeline (CICD-03)

Bicep IaC (INFRA-01)
    +-- Container Apps Env + ACR + Key Vault + Postgres + Log Analytics (INFRA-02)
            +-- Test+Prod Container Apps per app (INFRA-05)
                    +-- LLM-proxy deploy (APP-02)
                            +-- Per-user rate-limit + daily cap (APP-03)
            +-- Managed identity + secret references (SEC-01)
                    +-- No secrets in browser (APP-04)

Postgres (INFRA-02)
    +-- Portal app registry (PORTAL-02)
    +-- Audit log table (PORTAL-04)
    +-- Role storage (AUTH-02)

ACR (INFRA-02)
    +-- CI/CD push images (CICD-01)
            +-- Trivy scan gate (CICD-02)
            +-- Promote pipeline (CICD-03)

Architecture doc (DOC-01) ──enables──> Onboarding guide (DOC-03)
Runbook (DOC-02)          ──enables──> Incident response by 2nd dev
IT-request checklist (DOC-04) ──unblocks──> Everything Azure-side on day 1
```

### Dependency Notes

- **Azure AD app-registration is the single most upstream dependency.** Without it, neither Easy Auth on the app nor Auth.js on the portal works. Get this on day 1 via DOC-04 (IT-request checklist) so it isn't blocking on day 5.
- **Postgres is needed for portal but not for pilot-app.** Pilot-app (Meeting Agent) is stateless. Could defer Postgres provisioning if it slips, but portal can't ship without it.
- **Promote pipeline (CICD-03) depends on both portal role check AND pipeline-permissions** — defence-in-depth. Portal initiates, pipeline-permission verifies. Don't trust portal alone (Phase 2 might decouple).
- **Trivy gate (CICD-02) conflicts with "ship fast" instinct.** Acknowledge: first ship may be blocked by a HIGH CVE in a transitively-pulled base image. Mitigation: pre-bake an "approved" base image in ACR. Document in runbook.
- **Anti-feature collisions:** Self-service upload UI (Phase 2) *would* enable RCE if combined with Phase 1's permissive admin trust model. If/when self-service is built, the audit log + sandboxing must precede it.

## MVP Definition

### Launch With (v1 — the Phase 1 pilot, 2 weeks)

These are exactly the PROJECT.md Active items, restated as features to validate the hosting/deploy pattern with Meeting Agent.

- [ ] **SSO via Entra ID on portal + Meeting Agent** — without it the platform has no reason to exist
- [ ] **Admin + Developer roles, enforced server-side** — pattern for future RBAC scaling
- [ ] **Test + Prod Container Apps per app, per resource-limit spec** — proves the topology
- [ ] **Server-side secrets in Key Vault, accessed via managed identity** — single biggest security win over status quo
- [ ] **LLM-proxy with per-user rate-limit + $10/day cap** — proves cost-bounded vibecoding
- [ ] **Portal dashboard listing apps + status + URLs** — single pane of glass
- [ ] **Portal role-assignment page** — admin can grant Developer-role without Azure portal
- [ ] **Audit log: promote + role-change events** — compliance + traceability seed
- [ ] **Bicep for every resource, redeployable from zero** — proves reproducibility
- [ ] **Azure DevOps pipelines per artifact + Trivy gate + admin-gated promote** — proves release flow
- [ ] **Cost-alert on RG at €100/month** — financial safety net
- [ ] **HTTPS, CSP, secure cookies, no secrets in browser** — minimum security bar
- [ ] **Four docs (architecture, runbook, onboarding, IT-request) + README** — proves Phase 2 contributor can onboard

### Add After Validation (v1.x / Phase 1.5)

Add when the pilot has been used for 2-4 weeks and pain points emerge.

- [ ] **Custom domain `apps.tag-team.be`** — trigger: someone wants to share an app outside the dev team and the `*.azurecontainerapps.io` URL feels off
- [ ] **App-owner field surfaced on dashboard + share-link button** — trigger: a TAG colleague has to ask "who built this?"; cheap to add
- [ ] **Health-check green/red dot on dashboard** — trigger: admin has been asked "is it up?" more than twice
- [ ] **Email/Teams notification on promote + budget breach** — trigger: a budget breach is missed
- [ ] **Portal-rendered last-100-lines log per app** — trigger: dev opens Azure portal more than 3 times to debug

### Future Consideration (v2+)

- [ ] **Self-service app registration via portal** — defer: requires app templates + approval workflow design
- [ ] **"Drop your Claude artifact" repo template** — defer: high-leverage but needs Phase 1 pattern stabilised first; this is *the* Phase 2 differentiator
- [ ] **Per-app secret management UI** — defer: not needed until N apps > 2
- [ ] **Viewer role** — defer: no use case until non-dev TAG users want read-only access
- [ ] **Multi-LLM-provider proxy** — defer: only if Anthropic-only becomes a hard problem
- [ ] **Per-app cost/usage dashboard** — defer: needs proxy usage logging + cost API integration
- [ ] **Rollback button** — defer: rollback via pipeline rerun works for now
- [ ] **Approval workflow** — defer: single promote-gate is enough until > 5 devs
- [ ] **Templates for app shapes** (static / static+proxy / full-stack+DB) — defer: extract from real second/third app, not from Meeting Agent alone
- [ ] **App marketplace / discovery feed** — defer: not until ≥10 apps
- [ ] **Audit-log export to compliance** — defer: until TAG security asks
- [ ] **GitOps catalog (Backstage-style)** — defer: re-evaluate at ~25 apps

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| SSO via Entra ID (portal + apps) | HIGH | LOW (Easy Auth + Auth.js stock) | **P1** |
| Two-role RBAC | HIGH | LOW | **P1** |
| Test + Prod per app | HIGH | MEDIUM | **P1** |
| Server-side secrets (KV + MI) | HIGH | MEDIUM | **P1** |
| LLM-proxy with rate-limit + cap | HIGH | MEDIUM | **P1** |
| Portal dashboard | HIGH | MEDIUM | **P1** |
| Audit log (promote + role-change) | MEDIUM | LOW | **P1** |
| Role-assignment page | MEDIUM | LOW | **P1** |
| Bicep IaC | HIGH | MEDIUM | **P1** |
| CI/CD + Trivy + admin promote | HIGH | MEDIUM | **P1** |
| Cost alert | HIGH | LOW | **P1** |
| HTTPS/CSP/cookies | HIGH | LOW | **P1** |
| Four docs + README | HIGH | LOW | **P1** |
| App-owner field on dashboard | MEDIUM | LOW | **P1 stretch** |
| Share-link button | MEDIUM | LOW | **P1 stretch** |
| Custom domain | MEDIUM | LOW | **P2** |
| Health-check status display | MEDIUM | LOW | **P2** |
| Portal-rendered app logs | HIGH | MEDIUM | **P2** |
| Teams/Email notifications | MEDIUM | LOW | **P2** |
| Per-app secret mgmt UI | MEDIUM | MEDIUM | **P2** |
| Viewer role | LOW | LOW | **P2** |
| Rollback button | MEDIUM | MEDIUM | **P2** |
| Per-app cost dashboard | MEDIUM | MEDIUM | **P2** |
| "Drop your artifact" template | HIGH | MEDIUM | **P2** (top of v2) |
| Multi-LLM-provider proxy | LOW (pilot) | MEDIUM | **P3** |
| Self-service app registration | MEDIUM | HIGH | **P3** |
| Approval workflow | LOW (pilot) | MEDIUM | **P3** |
| App marketplace / discovery | LOW | MEDIUM | **P3** |
| GitOps catalog | LOW | HIGH | **P3** |

**Priority key:**
- **P1**: Must have in Phase 1 pilot (2 weeks). Pilot fails without these.
- **P1 stretch**: Cheap, high-trust-signal additions if time allows in pilot.
- **P2**: Add in Phase 1.5 / Phase 2 once pilot proves the pattern.
- **P3**: Defer indefinitely; re-evaluate when a real trigger appears.

## Competitor Feature Analysis

Framing: these are platforms TAG would *not* build because they exist or wouldn't fit, but they inform what TAG-developers-internally expect. We selectively borrow.

| Feature | Vercel | Render / Railway / fly.io | Heroku Enterprise | Backstage | Coolify / Dokploy | Azure App Service "1-button" | Our Approach |
|---------|--------|---------------------------|-------------------|-----------|-------------------|------------------------------|--------------|
| **SSO for end-users on deployed apps** | Deployment Protection (paid) | Limited / DIY | Yes (Heroku SSO + app-add-ons) | N/A (it's a portal, not a host) | Limited | Built-in (Easy Auth) | **Container Apps Easy Auth** — zero-code, Entra-native |
| **Preview deployments / test envs** | Per-branch preview URL (industry-leading) | Yes, per-branch | Review apps via pipelines | N/A | Yes per-app | Slots (App Service feature) | **Two long-lived envs per app** (test + prod). No per-PR previews in Phase 1 — overkill for 2 devs. |
| **Secrets management** | Env-var UI, scoped to env | Env-var UI | Config vars + add-ons | Plugins for Vault etc. | Built-in env UI + S3 backups | Key Vault references | **Key Vault + managed identity** (no env-var pasting) |
| **Image build & registry** | Built-in build pipeline | Build from Git | Slug compiler + Container Registry | N/A | Buildpacks + Docker | ACR + Web App for Containers | **ACR + DevOps Pipelines + explicit Dockerfile per app**. No Buildpacks in Phase 1. |
| **Role-based access** | Team roles | Team roles | Fine-grained per-app privileges + admin/member | Plugin-defined | Yes | Azure RBAC | **Two roles (Admin / Dev) in portal DB**; pilot scope |
| **Audit log** | Yes (enterprise) | Limited | Yes (enterprise) | Plugin | Limited | Azure Activity Log | **Portal-owned audit log** for promote + role-change; Azure Activity Log for infra |
| **App catalog / dashboard** | Project list | Service list | Apps list | **Software Catalog (core feature)** | Apps list | Resource group view | **Portal dashboard with status + URL** (less than Backstage, more than just a list) |
| **Templates / scaffolding** | Templates marketplace | Blueprint files | Buildpacks + button-deploy | **Software Templates (core feature)** | App templates | Quickstarts | **Repo template in Phase 2** ("drop your Claude artifact") |
| **Cost controls per app** | Spend limit (team) | Not first-class | Add-on quotas | N/A | None | Azure budgets per RG | **RG-level budget + per-app daily LLM cap in proxy** — cheap, sufficient |
| **Rate limiting** | Edge config | DIY | DIY | N/A | DIY | API Mgmt or DIY | **In-proxy rate-limit per user** — bare minimum, in scope |
| **Custom domains** | Yes, first-class | Yes | Yes | N/A | Yes | Yes | **Phase 1.5/2** — explicitly deferred |
| **Self-service UI app upload** | Push to git | Push to git | Push to git | Software templates trigger pipelines | Yes (UI Docker upload) | Yes (zip deploy in portal) | **Admin-managed in Phase 1**; revisit in Phase 2 with templates |
| **Documentation tooling** | Docs are external | External | External | **TechDocs (docs-like-code, core)** | External | External | **Markdown in `docs/` checked into repo** — Backstage-lite |

### Key takeaways from competitors

- **Backstage is what we'd grow into at 25+ apps, not what we start as.** Its core value (Software Catalog + TechDocs + Templates) maps directly onto our v2/v3, but its infrastructure cost (run Backstage itself, write entity YAMLs) doesn't pay back at 1-app scale.
- **Vercel's killer feature (per-PR preview deployments) is overkill for 2 devs.** Long-lived test+prod environments cover the same need with less Container Apps cost.
- **Coolify/Dokploy show the UI-upload temptation.** Both let devs upload a Docker Compose via UI. We deliberately reject this in Phase 1 because RCE risk + audit gap is real even internally.
- **Heroku Enterprise's "fine-grained per-app privileges" is over-engineering for pilot.** Two flat roles is sufficient until > 5 devs and > 5 apps.
- **Azure App Service's "Easy Auth + slots + ACR" pattern is what we're effectively rebuilding in Container Apps.** We chose Container Apps for scale-to-zero, container-native isolation, and per-app limits.

## Sources

- [Backstage — backstage.io](https://backstage.io/) — Software Catalog, Software Templates, TechDocs as core features (HIGH confidence, official)
- [Backstage GitHub](https://github.com/backstage/backstage) — feature scope and plugin model
- [Vercel Preview Deployments](https://vercel.com/docs/deployments/environments) — branch-scoped envs + deployment protection (HIGH confidence)
- [Vercel Deployment Protection](https://vercel.com/docs/deployments) — SSO for previews on enterprise tier
- [Promoting a preview deployment to production — Vercel](https://vercel.com/docs/deployments/promote-preview-to-production) — promote pattern
- [Coolify vs Dokploy 2026 comparison — Contabo blog](https://contabo.com/blog/blog-coolify-vs-dokploy-comparison/) — self-hosted PaaS feature surface (MEDIUM confidence)
- [Coolify vs Dokploy — Cherry Servers](https://www.cherryservers.com/blog/coolify-vs-dokploy) — UI-upload, buildpack, backup features
- [Dokploy vs Coolify — LogRocket](https://blog.logrocket.com/dokploy-vs-coolify-production/) — production-readiness gaps
- [Azure Container Apps Authentication — Microsoft Learn](https://learn.microsoft.com/en-us/azure/container-apps/authentication) — Easy Auth, /.auth endpoints (HIGH confidence, official)
- [Enable Authentication with Microsoft Entra ID — Microsoft Learn](https://learn.microsoft.com/en-us/azure/container-apps/authentication-entra) — Entra ID integration steps (HIGH confidence, official)
- [Heroku Enterprise Team Roles — Heroku Dev Center](https://devcenter.heroku.com/articles/managing-team-roles) — admin/member + per-app privileges (HIGH confidence, official)
- [Managing Apps and Users with Fine-Grained Access — Heroku](https://blog.heroku.com/managing_apps_and_users_with_fine_grained_access_controls) — RBAC patterns at scale
- [AI Agent Audit Logs — Maxim](https://www.getmaxim.ai/articles/ai-agent-audit-logs-full-visibility-over-tool-usage/) — immutable, time-stamped, identity-bound audit logs (MEDIUM confidence)
- [Advanced LLM security: secret leakage — Doppler](https://www.doppler.com/blog/advanced-llm-security) — never log API keys, scope per environment (MEDIUM confidence)
- PROJECT.md (`C:/Users/JasperPolleunis/vibecoding-platform/.planning/PROJECT.md`) — Active scope, Out of Scope, Key Decisions, Constraints (HIGH confidence, authoritative for this project)

---
*Feature research for: internal vibecoding platform / mini-PaaS / app-launcher (TAG-internal)*
*Researched: 2026-05-19*
