# Research SUMMARY — VibeCoding Platform

**Synthesizes:** STACK.md + FEATURES.md + ARCHITECTURE.md + PITFALLS.md
**Date:** 2026-05-30

---

## TL;DR

The stack in PROJECT.md is **fundamentally correct** with five non-obvious corrections (logged below). The 20 Phase-1 features map 1:1 to PROJECT.md Active items — no gaps, no fluff. Six critical pitfalls **must be mitigated in Phase 1** or production traffic will burn money or break security. Phase 1 is feasible in ~2 weeks for 2 devs + Claude Code provided the TAG IT request lands on day 1.

---

## Stack — final picks

| Layer | Pick | Version (May 2026) |
|-------|------|---------------------|
| Hosting | Azure Container Apps, single Env, suffix-naming for test/prod | API `2026-01-01` |
| Portal | Next.js (App Router) + TS + Tailwind + shadcn/ui + Prisma | 15.2.x LTS / 5.6+ / v4 / latest / 6.19.x |
| Portal auth | Auth.js v5 with **`microsoft-entra-id` provider** (not legacy Azure AD) | 5.0 |
| Pilot-app auth | Container Apps **Easy Auth** (Entra ID) | n/a |
| Pilot-app proxy | Node 22 LTS + Express 4 + `@anthropic-ai/sdk` | 4.21.x / 0.97.x |
| Rate-limit & daily cap | `rate-limiter-flexible` with Postgres store (`express-rate-limit` is in-memory only — won't survive replica restarts) | 5.x |
| DB | Postgres Flex B1ms + **pgbouncer ON** | 16 |
| Registry | ACR Basic | n/a |
| Secrets | Key Vault, RBAC (not access-policies), Managed Identity | n/a |
| IaC | Bicep + **Azure Verified Modules** (`br/public:avm/...`), version-pinned | Bicep CLI 0.31+ |
| CI/CD | Azure DevOps Pipelines, **Workload Identity Federation** (no SP secrets) | n/a |
| Image scan | Trivy `--severity HIGH,CRITICAL --ignore-unfixed` | 0.58+ |
| Observability | Log Analytics workspace **+ Application Insights** (workspace-based) | n/a |
| Dep updates | **Renovate Bot** (Dependabot-on-ADO needs paid GHAS) | n/a |
| LLM | Anthropic Claude API direct | Sonnet 4 |

**Five corrections vs. original PROJECT.md notes:**
1. `MicrosoftEntraIDProvider`, not legacy `AzureADProvider` in Auth.js v5.
2. Add Application Insights (PROJECT.md only mentioned Log Analytics).
3. Use WIF for ADO → Azure (replaces long-lived service principal secret).
4. Use AVM modules (classic ALZ-Bicep retired 16 Feb 2026).
5. Daily-spend cap **must be Postgres-backed**, not in-memory (`express-rate-limit` is per-process).

---

## Feature priorities

**Phase 1 — 20 P1 features**, all map 1:1 to PROJECT.md Active list:
- Auth (3): AUTH-01..03
- Infra (5): INFRA-01..05
- Pilot app (4): APP-01..04
- Portal (4): PORTAL-01..04
- CI/CD (3): CICD-01..03
- Security (4): SEC-01..04
- Documentation (5): DOC-01..05 — **first-class P1, not afterthought**

**Phase 2 candidates** (deferred but signposted): custom domain → app-owner field + share button → health-check display → portal-rendered logs → Teams notifications → self-service app registration → multi-LLM abstraction → rollback button.

**12 anti-features** explicitly to NOT build in Phase 1:
- UI-upload of Dockerfiles (RCE risk)
- Custom domain
- Multi-LLM-provider abstraction (3× proxy code, zero pilot value)
- Self-hosting Coolify/Dokploy instead of ACA
- Per-PR preview deploys (Vercel-style)
- Backstage Software Catalog (right for ~25 apps, not 1)
- Stripe-style metered billing
- RLS / multi-tenant isolation patterns
- Public status page
- LaunchDarkly / sophisticated feature flags
- OpenTelemetry export to third-party APM
- CAPTCHA / bot protection

---

## Architecture decisions locked (see ARCHITECTURE.md §13)

- **A1.** Single Container Apps Environment, suffix-naming for test/prod.
- **A2.** Single Entra app-registration, app-roles `Admin` + `Developer`.
- **A3.** Source of truth for roles = Postgres `app_users` (seeded from Entra claim on first login).
- **A4.** Easy Auth for pilot static + proxy; Auth.js v5 for portal.
- **A5.** Daily cap counter = Postgres `SELECT … FOR UPDATE` + 90% safety margin.
- **A6.** Promote = ACR retag (same digest), never rebuild.
- **A7.** Monorepo + path-triggered ADO pipelines.
- **A8.** WIF for ADO → Azure.
- **A9.** Public ingress, no VNET in Phase 1.
- **A10.** App Insights workspace-based, sampling (20% dev / 5% proxy prod).

---

## Pitfalls that drive Phase 1 design

**Six critical** — must be mitigated before any user traffic:

| # | Risk | Phase-1 mitigation |
|---|------|---------------------|
| CRIT-1 | ACA scale-to-zero kills in-memory cap counter | `minReplicas: 1` on LLM-proxy; cap state in Postgres |
| CRIT-2 | Daily-cap race condition across replicas | `SELECT … FOR UPDATE` + 90% safety margin |
| CRIT-3 | Easy Auth redirect-URI mismatch locks devs out | Decide Easy Auth vs Auth.js per-surface upfront; v2 audience; pre-bind redirect URIs in Bicep with `dependsOn` |
| CRIT-4 | WIF 401s from scope mismatch | One MI per environment; RBAC at RG-scope matching Bicep target scope; exact subject-claim format |
| CRIT-5 | Key Vault refs fail silently at container cold start | RBAC (not access-policies); explicit Bicep `dependsOn`; `/healthz/secrets` readiness probe |
| CRIT-6 | Log Analytics ingest blows €100/month in week 1 | LAW daily cap 1 GB; App Insights sampling 20%/5%; cost alert at €50 + €100 |

**Six high-impact** (HIGH-1..6): revision-mode confusion, Auth.js v5 `trustHost`/Edge runtime gotchas, AVM version drift, Trivy false-fail noise, Postgres B1ms credit/connection exhaustion, CSP-vs-inline-scripts in Claude artifacts.

**Five Next.js-specific minors** (MIN-1..5): async `cookies()`/`headers()`, `.bicepparam` vs `.parameters.json`, `timestamptz` for audit, standalone-output forgets `public/`, WIF service-connection scoped too broad by default.

---

## Build order (from ARCHITECTURE.md §9)

1. Bicep skeleton (RG, KV empty, LAW, AI, ACR, ACA Env)
2. Entra app-reg + WIF service connection
3. Postgres Flex + pgbouncer + initial schema
4. Portal stub (login only) + Easy Auth on dummy app — **validates SSO contract**
5a + 5b (parallel). Portal real features ← + → LLM-proxy + daily-cap
6. Meeting Agent static (Nginx) + wire to proxy
7. All ADO pipelines path-triggered + Trivy + smoke tests
8. Docs (DOC-01..05)
9. Promote-test→prod pipeline + portal trigger + audit

Steps 5a + 5b are explicitly parallel — two developers, one each.

---

## Open verification items (re-check before Phase 1 starts)

- Current Anthropic Sonnet 4 pricing (affects CRIT-2 and MOD-4 numbers).
- Latest AVM `avm/res/app/container-app` version + breaking-change CHANGELOG (HIGH-3).
- TAG IT decision: GHAS-on-ADO yes/no (determines Dependabot vs Renovate).
- TAG IT decision: Defender for Cloud subscription state (affects whether ACR scanning replaces Trivy or augments it).

---

## Roadmap implications

- **Phase structure** maps to build order in §9 above. Granularity "standard" (5-8 phases) per config: likely Phase 1 = single milestone, decomposed into ~6 phases (Foundation, Auth, Data, Portal, Pilot App, Promotion+Docs).
- **Critical-path dependency: TAG IT request.** File day 1; without RG + WIF + app-reg the 2-week timeline collapses. IT-request checklist (DOC-04) is therefore the highest-leverage doc in Phase 1.
- **Documentation is in scope from day 1** (DOC-01..05). Code without docs = not "done".
