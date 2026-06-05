# Research SUMMARY — VibeCoding Platform (Container-Apps-only)

**Synthesizes:** STACK.md + FEATURES.md + ARCHITECTURE.md + PITFALLS.md
**Date:** 2026-05-30 (Sessie 2 — scope reduction)

---

## TL;DR

Azure-surface is reduced to **Container Apps + Storage + Log Analytics**. Everything else is self-hosted: **Postgres + Keycloak + Docker registry** as Container Apps with Azure Files volumes, and the portal + LLM-proxy are rebuilt in **Python (FastAPI)**. Effective monthly Azure cost target ~€10-15. Phase 1 timeline realistic at ~5.5 weeks for 2 devs + Claude Code.

The 28 v1 requirements still cover the same product surface; what changed is the *technology choices*, not the *outcomes*.

---

## Stack — final picks (Sessie 2)

| Layer | Pick |
|-------|------|
| Hosting | Azure Container Apps (Consumption), single Env, suffix-naming |
| Storage | Azure Storage Account + Files (Premium for pg-data, Standard for others) |
| Logs | Azure Log Analytics workspace, daily-cap 500 MB |
| IaC | Bicep single-file (no AVM) |
| Portal | Python 3.12 + FastAPI + Jinja2 + HTMX + SQLAlchemy 2 (async) + Authlib |
| Pilot-app proxy | Python 3.12 + FastAPI + `anthropic` SDK + `slowapi` + `tenacity` + `structlog` |
| Pilot-app static | Nginx + `oauth2-proxy` sidecar |
| Database | Self-host `postgres:16-alpine` Container App + Azure Files Premium mount + pg_dump nightly ACA Job |
| SSO | Self-host `quay.io/keycloak/keycloak:26` Container App, Postgres-backed, realm-export in git |
| Container Registry | Self-host `distribution/distribution:3` Container App + `oauth2-proxy` sidecar + Azure Files mount |
| Secrets | Container Apps built-in `secrets:` |
| Image scan | Trivy `--severity HIGH,CRITICAL --ignore-unfixed` |
| CI/CD | Azure DevOps Pipelines + Service Principal client-secret service connection (90d rotation) |
| LLM | Anthropic Claude API direct |

---

## What was dropped vs Sessie 1

Azure DB for PostgreSQL Flex, Azure Key Vault, Azure Container Registry, Azure AD app-registration, Container Apps Easy Auth, Application Insights, Workload Identity Federation, Azure Verified Modules, Next.js, Auth.js, Prisma.

---

## Architecture decisions locked (ARCHITECTURE.md §11)

- **A1.** Single Container Apps Environment, suffix-naming.
- **A2.** Single Keycloak realm `vibecoding`, two clients (`portal` + `meeting-agent-proxy`), two roles (`Admin`, `Developer`).
- **A3.** Source of truth for roles = Postgres `portal.app_users` (seeded from Keycloak role-claim).
- **A4.** `oauth2-proxy` sidecar in front of static + registry; Authlib OIDC in portal + proxy.
- **A5.** Daily-cap = Postgres `SELECT … FOR UPDATE` + 90% safety margin.
- **A6.** Promote = retag in self-host registry (same digest).
- **A7.** Monorepo + path-triggered ADO pipelines.
- **A8.** SP-secret service connection, 90-day rotation.
- **A9.** Public ingress, no VNET.
- **A10.** structlog JSON to ACA stdout, no App Insights.

---

## Pitfalls that drive Phase 1 design

**Seven critical** — must be mitigated before any user traffic:

| # | Risk | Phase-1 mitigation |
|---|------|--------------------|
| CRIT-1 | ACA scale-to-zero kills cap counter | `minReplicas: 1` on LLM-proxy; cap state in Postgres |
| CRIT-2 | Daily-cap race condition across replicas | `SELECT … FOR UPDATE` + 90% safety margin |
| CRIT-3 | Self-host Postgres on Azure Files = file-locking bugs | Azure Files **Premium** for pg-data; nightly pg_dump; restore drill in DoD |
| CRIT-4 | Keycloak realm config lost on restart | `KC_DB=postgres` explicit; realm-export in git; pipeline applies it |
| CRIT-5 | Self-host registry leaks images if oauth2-proxy misconfigured | Sidecar pattern with strict `OAUTH2_PROXY_*` config; anon-401 smoke test |
| CRIT-6 | ACA `secrets:` have no rotation tooling | Documented procedure: pipeline-variable + revision-bump |
| CRIT-7 | Log Analytics ingest blows €30 budget | LAW daily cap 500 MB + structlog renderer + uvicorn --no-access-log |

**Six high-impact** (HIGH-1..6): revision-mode confusion, Python image bloat on registry, Keycloak realm drift between UI and git, Python cold-start, ACA Job reliability for pg_dump, CSP-vs-inline-scripts in Claude artifacts.

**Five minors** (MIN-1..5): Azure Files quota, Keycloak version pin, distribution filesystem driver, slowapi key-func, Postgres connection-limit.

---

## Build order (ARCHITECTURE.md §8)

1. Bicep skeleton (RG, Storage Account + Files shares, LAW, ACA Env)
2. Self-host registry + oauth2-proxy + Keycloak realm-export checked-in
3. Self-host Postgres + pg_dump ACA Job + restore-drill validated
4. Portal stub (login-only) wired to Keycloak via Authlib
5a + 5b (parallel). Portal real features ← + → LLM-proxy + daily-cap
6. Meeting Agent static (Nginx) + oauth2-proxy sidecar
7. All ADO pipelines path-triggered + Trivy + smoke tests
8. Docs (DOC-01..06)
9. Promote-test→prod pipeline + portal trigger + audit

---

## Open verification items (before Phase 1 starts)

- Anthropic Sonnet 4 current pricing.
- ACA Consumption pricing + free-tier numbers for May 2026.
- Keycloak 26 boot flow on Container Apps (`kc.sh start --optimized`).
- `distribution/distribution:3` Docker-OIDC auth via `oauth2-proxy` (test end-to-end before committing pattern).
- Azure Files Premium vs Standard for pg-data — Premium recommended for performance; verify minimum quota cost (~€7/100 GiB).

---

## Cost summary

Target: ~€10-15/month effective Azure cost after free-tier (~€16-20 gross), ~€120-180/year. See `docs/management.md` for the public-facing breakdown.

Trade-off: more dev work (~5.5 week timeline instead of 2) + ops burden (Postgres backups, Keycloak realm sync, registry GC) in exchange for cost + portability.
