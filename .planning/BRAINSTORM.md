# Brainstorm — VibeCoding Platform (Q&A trail)

This file captures the full brainstorming dialogue that led to PROJECT.md and ROADMAP.md. It exists so future contributors (and future-you) can understand *why* each decision was made, not just *what* the decision was.

**Date:** 2026-05-30
**Participants:** Jasper Polleunis (TAG), Claude (sonnet-4.5, caveman-mode)
**Workflow:** `superpowers:brainstorming` → `gsd-new-project`

---

## 0. Starting point — first-draft notes

Pasted in by Jasper, verbatim:

> Dit document beschrijft de volledige implementatie van een intern platform waarmee medewerkers hun VibeCoding-projecten kunnen indienen, reviewen, testen en officieel lanceren. Alle projecten worden gehost op een centrale Azure-infrastructuur met automatische deployment, isolatie en toegangsbeheer. Core features: Security, IAM (SSO), Navigatie en front-end design, aparte modules voor elke app. Test- en productieomgeving.

The notes also contained an ASCII architecture sketch (Azure AD → Management Portal → TEST/PROD containers → ACR → Azure DevOps Pipeline), a resource-limit table (test 0.5 vCPU / 1 GB / scale-to-zero, prod 2 vCPU / 4 GB / min 1 replica / auto-scale CPU>70%), and a list of pilot-phase considerations:

- Test with one app (Meeting Agent) in Azure Container Apps
- Admin role (manage, upload, test, prod) + Viewer role (upload, prod)
- One Container App per app for isolation, one URL per app
- Frontend React + .NET backend, but also explore Alpine.js
- Replace some Azure tools with own code (e.g. Container Registry) — *later challenged*
- PostgreSQL
- Dashboard with apps, test/prod split, app intake, admin page, login, app detail
- Per-app CPU analytics for admin
- Phase 2: DB sync + Nimbus (master)

---

## 1. Initial framing & scope challenge

Claude flagged the scope as too broad for a single design (≥6 independent subsystems). The user had already implicitly decomposed via "Fase 1 / Fase 2"; the brainstorm focused Phase 1 = pilot one app end-to-end.

---

## 2. Question chain (sequential, one at a time)

### Q1 — Primary user of Phase 1?
**A — Chosen: B.** Jasper + 1-2 collega-developers as testers.
**Effect:** Scope of auth limited to internal-dev rollout. Full external-user UX polish deferred.

### Q2 — Status of the Meeting Agent app?
**A — Chosen: A.** Works locally, no Dockerfile.
**Effect:** Containerization is in Phase 1 scope.

### Q3 — Tech stack of Meeting Agent?
**Response (freeform):** "Meeting agent is gewoon gevibecodet artifact in Claude" — it's a Claude artifact, not a real backend app.
**Effect:** Re-framed the pilot: static HTML+JS artifact, not a server app.

### Q4 — What does the artifact need?
**A — Chosen: B + remark "but we also want apps with backend".**
The artifact is static + calls the LLM directly from browser (API-key-in-browser leak risk). The platform must also support real backend apps later.

### Q5 — How to handle LLM API key for Phase 1?
**A — Chosen: B.** Build a minimal backend LLM-proxy (Node Express, one endpoint) for Phase 1.
**Rationale logged:**
- Platform vision includes backend apps — proxy pattern must work from day 1.
- Container App is only meaningful with a backend; static-only could live on Static Web Apps.
- Key Vault + Managed Identity = security foundation reused by every future app.
- Extra effort ~½ day.

### Q6 — Azure environment status?
**A — Chosen: B with twist.** TAG has tenant, Jasper has limited rights, must request RG + Service Principal from IT. *User initially proposed self-coded login without Azure AD — Claude pushed back on security grounds.*

### Q7 — Custom login vs Azure AD?
**Security pushback from Claude:** Self-coded login = bcrypt/argon2 hashing, session+CSRF, brute-force protection, password-reset flow with email+token, 2FA, audit log, account lockout. Single bug compromises every app on the platform.
**A — Chosen: A.** Azure AD app-registration in TAG tenant. No new subscription needed.

### Q8 — Roles?
**Initial answer C, then corrected to B.** Admin (all) + Developer (upload + test, requires admin-approval for prod). No Viewer role in Phase 1; pattern stays available.

### Q9 — How does an app get to production in Phase 1?
**A — Chosen: A.** Hardcoded pilot. Admin deploys Meeting Agent + proxy manually via Azure CLI / GitHub Action. Portal shows "running / not running" + URL. No upload UI. Developer-flow = Phase 2.

### Q10 — Frontend portal stack?
**Initial Q11 — Repo + CI/CD?**
- User noted "Ik zit op Bitbucket. Denk je op dit of beter Azure DevOps?"
- Claude recommended **Azure DevOps** (native ACR + Container Apps tasks, service connections, free tier sufficient).
- User accepted ADO.

### Q11 — Test/Prod isolation strategy?
**A — Chosen: A.** Single Container Apps Environment, suffix-naming (e.g. `meeting-agent-test`, `meeting-agent-prod`). Cheapest. Per-app resource-limits give enough isolation for pilot. Migration to two-environment topology is later if needed.

### Q12 — LLM provider for the proxy?
**A — Chosen: A.** Anthropic Claude API direct. Fits TAG's existing Claude usage (Claude Code, skills).

---

## 3. Big challenge — "is this Phase 1 actually logical?"

User explicitly asked Claude to challenge the scope: *"Heb je goed nagedacht of dit een logische eerste fase was? ... Gebruik zeker goede brainstorm om me te challengen of er nog iets extra in scope moet."*

Claude pushed back with four problems and missing items:

**Problem 1 — Too big for a pilot.** Original Phase 1 ≈ 10-15 dev-days = 3-4 weeks. Pilots should be 1-2 weeks.

**Problem 2 — Portal is too early.** A dashboard for 1 app shows one row; portal's real value emerges with multiple apps + self-service. Building it first is the biggest time-sink.

**Problem 3 — Azure Container Apps Easy Auth was missing from the notes.** Container Apps has zero-code Azure AD authentication for the app itself; no Next.js portal needed to put Meeting Agent behind SSO.

**Problem 4 — Test+Prod for the *portal* is overkill.** Apps need test/prod (something to test). Portal has no end-users yet — single instance is enough.

**Gaps not in the original notes (added as Phase 1 requirements):**
- Cost-budget alert (Azure costs are sneaky, ~€60-150/month pilot)
- IaC (Bicep) from day 1 — without it, every redeploy = portal-clicks
- Log Analytics workspace — no logs = blind debugging
- Rate-limit on LLM proxy — without it, one accidental loop burns Claude credits
- Daily Claude-API budget cap (Anthropic console + proxy-side)
- Audit logging in portal
- CSP headers on every web endpoint
- Image scanning in CI (Trivy, free)
- Secret-rotation procedure (documented, not necessarily executed)
- Naming + tagging conventions for cost attribution
- Backup policy for Postgres

**Proposed restructure:**
- Phase 1A — Infra foundation + 1 app live behind SSO (1-2 weeks)
- Phase 1B — Portal (1-2 weeks)
- Phase 1C — Promotion flow + 2nd app blueprint (~1 week)

**User decision:** *"Ik ga Claude Code gebruiken en we zijn met twee. Dus het kan wel sneller gaan. Alle gaten moeten gedekt zijn."*

→ Keep Phase 1 broad (don't split into 1A/1B/1C across milestones) but treat them as **internal milestones**. All gaps in scope.

---

## 4. Final stack + scope confirmed

| Area | Decision |
|------|---------|
| Hosting | Azure Container Apps, single Environment, suffix-naming for test/prod |
| Auth (pilot apps) | Azure AD app-registration + Container Apps Easy Auth (no custom login code) |
| Auth (portal) | Auth.js v5 with Microsoft Entra ID provider |
| Portal | Next.js 15 + TypeScript + Tailwind + shadcn/ui + Prisma + Postgres |
| Pilot-app proxy | Node Express, one endpoint `/api/llm`, server-side Claude key |
| Container Registry | Azure Container Registry (Basic SKU) |
| Secrets | Azure Key Vault + Managed Identity |
| Logging | Azure Log Analytics + Application Insights |
| IaC | Bicep (with Azure Verified Modules) |
| CI/CD | Azure DevOps Repos + Pipelines, Workload Identity Federation (no SP secrets) |
| Image scan | Trivy in pipeline; fail on HIGH+ |
| LLM provider | Anthropic Claude API direct |
| Budget alert | €100/month on the pilot Resource Group |
| Claude API cap | $10/day hard-cap in the proxy (Postgres-backed counter — `express-rate-limit` is in-memory only) |
| Roles | Admin + Developer (no Viewer in Phase 1) |

**Out of scope in Phase 1:** developer self-service upload, review/approval workflow beyond a single test→prod promote, Nimbus + DB sync, custom domain, VNET isolation, multi-region, multi-LLM-provider abstraction, Viewer role, native mobile.

---

## 5. Project-init questions (gsd-new-project Q13-Q16)

### Q13 — Timeline target Phase 1?
**A — B.** ~2 weeks (10 working days). Aggressive but feasible with 2 devs + Claude Code, provided IT request is filed day 1.

### Q14 — Cost limits?
**A — A + E.** Azure budget alert at €100/month. Claude API daily cap $10/day.

### Q15 — Azure access at TAG?
**A — B.** Tenant exists, limited rights, need IT ticket. **Risk-flagged:** IT lead time 1-5 days against a 2-week target.

**IT request shopping list (from the brainstorm, written into `docs/it-request.md` in Phase 1):**
- Resource Group `rg-vibecoding-pilot` in West Europe
- Contributor on that RG for Jasper + 2nd dev
- Azure AD app-registration permission (TAG tenant, app name `vibecoding-platform`)
- ACR Basic SKU
- Postgres Flexible Server quota in West Europe (B1ms vCore)
- Azure DevOps organization + project `VibeCoding`
- Workload Identity Federation for ADO → Azure (no long-lived service principal secret)

### Q16 — Documentation requirement
User explicitly asked: *"Ik wil ook dat alles ergens goed gedocumenteerd is over hoe dit platform werkt zodat dit duidelijk is voor de toekomst. Ik moet ook een duidelijk projectblad hebben met een roadmap."*

→ Documentation became a first-class Phase 1 requirement: `DOC-01` architecture, `DOC-02` runbook, `DOC-03` onboarding, `DOC-04` IT-request checklist, `DOC-05` repo README.

---

## 6. Project workflow config (gsd-new-project settings)

| Setting | Value |
|---------|------|
| Mode | Interactive |
| Granularity | Standard (5-8 phases, 3-5 plans each) |
| Execution | Parallel |
| Git tracking | Yes (`.planning/` in repo) |
| Research before plan-phase | Yes |
| Plan-check | Yes |
| Verifier | Yes |
| AI models | Balanced (Sonnet default) |
| PR body extras | All seeded `enabled: false` (can be flipped later) |

---

## 7. What this trail unlocks

This file plus `PROJECT.md` is the ground truth for *why* each decision was made. When someone in 3 months says "why didn't we use Bitbucket / why did we pick Easy Auth / why is the portal not in two environments", the answer is here, with date and reasoning.

The downstream artifacts that build on this trail:

- `.planning/PROJECT.md` — locked project context
- `.planning/config.json` — GSD workflow settings
- `.planning/research/STACK.md` — validated stack + version pins
- `.planning/research/FEATURES.md` — feature scope (table stakes / differentiators / anti-features)
- `.planning/research/ARCHITECTURE.md` — component layout + data flow
- `.planning/research/PITFALLS.md` — Azure-specific gotchas
- `.planning/research/SUMMARY.md` — synthesis of the above
- `.planning/REQUIREMENTS.md` — REQ-IDs, ready for roadmap mapping
- `.planning/ROADMAP.md` — phased execution plan

---

*Captured at the end of the brainstorming session, before requirements + roadmap generation.*
