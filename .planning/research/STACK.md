# Stack Research — VibeCoding Platform (Container-Apps-only)

**Domain:** Internal app-launcher on Azure Container Apps with everything else self-hosted in Python or via open-source containers.
**Researched:** 2026-05-30 (Sessie 2 — scope reduction)
**Confidence:** HIGH

## Executive verdict

Drop Azure-managed PaaS where possible; keep only Azure Container Apps + Storage + Log Analytics. Build the rest from open-source container images and Python (FastAPI). Effective monthly cost: ~€10-15 after free tier.

---

## Core Technologies

| Technology | Version (May 2026) | Purpose | Why |
|-----------|---------------------|---------|-----|
| **Azure Container Apps** | API `2026-01-01`, Consumption profile | Compute | Only Azure-managed PaaS in scope. Per-app isolation, scale-to-zero, built-in secrets. |
| **Azure Storage Account + Files** | Standard LRS | Persistent volumes for Postgres data, Keycloak data, Docker registry data, pg_dump backups | Required for stateful containers; minimal Azure footprint. |
| **Azure Log Analytics workspace** | — | ACA log sink (required by ACA Env) | Free tier 5 GB/month; daily-cap set to 500 MB. |
| **Bicep** | CLI 0.31+, single-file | IaC | No AVM modules — keep one readable `infra/main.bicep` per the user's "as little Azure as possible" stance. |
| **Python** | 3.12 | Language for portal + LLM-proxy + helpers | Fastest MVP; smallest images via `python:3.12-slim`; strong Anthropic SDK. |
| **FastAPI** | 0.115+ | HTTP framework for portal + proxy | Async, simple, fits Container Apps health-probe model. |
| **Jinja2** | 3.1+ | Server-rendered templates (portal) | Replaces Next.js. Paired with HTMX for interactivity. |
| **HTMX** | 2.x | Sprinkle of client-side interactivity | No SPA build pipeline. |
| **SQLAlchemy** | 2.x (async) + `asyncpg` | DB layer | Replaces Prisma. |
| **Alembic** | latest | DB migrations | |
| **Authlib** | 1.3+ | OIDC client (against Keycloak) | Replaces Auth.js. |
| **`anthropic` Python SDK** | 0.40+ | Claude API | Server-side; key from ACA-secret env-var. |
| **`structlog`** | 24.x | Structured JSON logs | Replaces App Insights — logs end up in Log Analytics via ACA stdout. |
| **`uvicorn`** | latest | ASGI server | Behind nothing — ACA terminates TLS. |
| **`httpx`** | 0.27+ | HTTP client | Inter-service calls. |
| **Keycloak** | 26.x official image | Self-hosted SSO/OIDC | Replaces Azure AD. Postgres-backed. |
| **Postgres** | `postgres:16-alpine` | Self-hosted DB | Replaces Azure DB for Postgres Flex. |
| **Docker Registry** | `distribution/distribution:3` | Self-hosted registry | Replaces ACR. |
| **`oauth2-proxy`** | 7.6+ | Sidecar OIDC gatekeeper in front of Nginx static + Docker registry | Cookie + bearer-token flow against Keycloak. |
| **Nginx** | `nginx:alpine` | Serves Meeting Agent static artifact | Same as before. |
| **Trivy** | 0.58+ | Image scan in pipeline | `--severity HIGH,CRITICAL --ignore-unfixed`. |
| **Anthropic API** | Sonnet 4 | LLM | Direct. |
| **Azure DevOps Pipelines** | — | CI/CD | SP-secret service connection (no WIF in this strategy). |

---

## Supporting Python libraries

| Library | Purpose |
|---------|--------|
| `python-jose` (via Authlib) | JWT validation for Keycloak tokens |
| `slowapi` | FastAPI rate-limit middleware (per-user keying) |
| `psycopg[binary]` | Sync driver for migrations |
| `cryptography` | Secret-handling primitives |
| `tenacity` | Anthropic API retry-with-jitter |
| `pydantic-settings` | Env-var validation at boot |
| `pytest`, `pytest-asyncio`, `httpx` (test) | Test stack |
| `ruff`, `mypy` | Lint + type-check |

---

## Container images (versions pinned)

| Image | Pin |
|-------|-----|
| `python:3.12-slim` | base for portal + proxy |
| `postgres:16-alpine` | self-host DB |
| `quay.io/keycloak/keycloak:26.0` | SSO |
| `distribution/distribution:3.0` | registry |
| `quay.io/oauth2-proxy/oauth2-proxy:v7.6.0` | OIDC sidecar |
| `nginx:1.27-alpine` | static |

---

## Dropped from previous stack

| Was | Why dropped | Replaced by |
|-----|-------------|-------------|
| Next.js + Auth.js + Prisma | JS stack not required; portal is small server-rendered HTML | FastAPI + Jinja2 + HTMX + SQLAlchemy + Authlib |
| Azure DB for PostgreSQL Flex | Cost + Azure lock-in | `postgres:16-alpine` Container App + Azure Files volume |
| Azure Key Vault | Cost + extra resource | Container Apps built-in `secrets:` |
| Azure Container Registry | Cost | `distribution/distribution:3` Container App + Azure Files |
| Azure AD app-registration / Entra ID | TAG tenant complexity + lock-in | Self-hosted Keycloak (optional Microsoft federation in Phase 2) |
| Container Apps Easy Auth | Bound to Azure AD | `oauth2-proxy` sidecar in front of static + registry |
| Application Insights | Cost + auto-instrumentation gaps on ACA | `structlog` JSON to ACA stdout → Log Analytics |
| Workload Identity Federation | Mismatch with self-host strategy | ADO Service Principal client-secret |
| Azure Verified Modules (AVM) | Vendor binding to AVM version drift | Single-file Bicep |

---

## Open verification before Phase 1 start

- Anthropic Sonnet 4 pricing (affects $10/day cap maths).
- ACA Consumption pricing — verify free-tier numbers for May 2026.
- Keycloak 26 → check no breaking changes in `kc.sh start --optimized` flow.
- `distribution/distribution:3` → verify it supports OIDC token-auth backend or that `oauth2-proxy` sidecar pattern covers Docker auth flow.
- Azure Files file-locking behavior with Postgres (per-write fsync) — known sharp edge; consider Azure Files **Premium** for the pg-data share if performance is poor on Standard.

---

## Confidence

| Area | Level |
|------|-------|
| Container Apps pricing + features | HIGH |
| Python + FastAPI + Authlib + Keycloak | HIGH |
| Self-host Postgres on Azure Files | MEDIUM-HIGH (file-locking behavior of Azure Files = pending verification) |
| Self-host registry behind OIDC | MEDIUM (Docker CLI OIDC token-flow requires testing) |
| Trivy + Bicep single-file | HIGH |
