# STATE — VibeCoding Platform

## Project Reference

See: `.planning/PROJECT.md` (updated 2026-05-30 — Sessie 2 scope reduction)

**Core value:** A TAG colleague can use a "vibecoded" internal app via a shareable URL after Microsoft login, without the developer leaking secrets or setting up cloud infra themselves.

**Current focus:** Phase 1 — Foundations (Container-Apps-only + self-host Postgres + Keycloak + Docker registry)

---

## Current Phase

**Phase 1** — Foundations
**Status:** Not started
**Plans:** 0/7 complete
**Blocking:** TAG IT request (RG + Storage Account + ACA Env quota + ADO project) — file day 1.

---

## Phase Index

| # | Phase | Status | Plans |
|---|-------|--------|-------|
| 1 | Foundations | Not started | 0/7 |
| 2 | Meeting Agent Live behind SSO | Not started | 0/4 |
| 3 | Daily-Cap + Rate-Limit | Not started | 0/3 |
| 4 | Portal Dashboard + Role Admin | Not started | 0/4 |
| 5 | CI/CD + Promote Flow + Audit | Not started | 0/5 |
| 6 | Documentation + DoD Validation | Not started | 0/3 |

---

## Open Decisions / Open Questions

Pending verification before Phase 1 starts:
- Azure Files Premium-or-Standard for pg-data (Premium recommended; verify minimum quota cost).
- Anthropic Sonnet 4 current pricing (affects $10/day cap maths).
- ACA Consumption pricing + free-tier numbers for May 2026.
- `distribution/distribution:3` Docker-OIDC auth via `oauth2-proxy` end-to-end smoke test.
- Keycloak 26 boot flow (`kc.sh start --optimized`) on Container Apps.
- Microsoft-tenant federation in Keycloak — out of Phase 1 scope but verify the federation flow before Phase 2 starts.

Pending TAG IT input:
- pgvector / Postgres-extensions needed in Phase 1? Standard `postgres:16-alpine` does not include pgvector. Add custom image if yes.
- Azure Files Premium quota minimum (100 GiB ≈ €7/month).
- Service Principal creation in Entra (or use ADO PAT for first deploy + create SP later).

---

## Last Updated

2026-05-30 — after Sessie 2 (scope reduction to Container-Apps-only + Python rebuild)
