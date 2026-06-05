<!-- GSD:project-start source:PROJECT.md -->
## Project

**VibeCoding Platform**

Intern Azure-gehost platform van Tactical Advisory Group (TAG) waarmee medewerkers "vibecoded" apps (Claude artifacts, mini-tools, kleine internal apps) op een gestructureerde manier live kunnen zetten voor collega's. Levert SSO via een self-hosted Keycloak, applicatie-isolatie via Azure Container Apps, gescheiden test- en productie-omgevingen, en een management-portal voor overzicht en rolbeheer.

In Fase 1 dient het als pilot voor één bewezen pilot-app (Meeting Agent) — het doel is om het hosting- en deploy-patroon end-to-end te valideren voor toekomstige apps van TAG-collega's, mét minimale Azure-vendor-lock-in en lage maandelijkse infra-kost.

**Core Value:** Een TAG-collega kan een "vibecoded" interne app gebruiken via een deelbare URL na Microsoft-login, zonder dat de developer geheimen moet lekken of zelf cloud-infrastructuur moet opzetten.

### Constraints

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
<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->
## Technology Stack

## Executive verdict
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
## Container images (versions pinned)
| Image | Pin |
|-------|-----|
| `python:3.12-slim` | base for portal + proxy |
| `postgres:16-alpine` | self-host DB |
| `quay.io/keycloak/keycloak:26.0` | SSO |
| `distribution/distribution:3.0` | registry |
| `quay.io/oauth2-proxy/oauth2-proxy:v7.6.0` | OIDC sidecar |
| `nginx:1.27-alpine` | static |
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
## Open verification before Phase 1 start
- Anthropic Sonnet 4 pricing (affects $10/day cap maths).
- ACA Consumption pricing — verify free-tier numbers for May 2026.
- Keycloak 26 → check no breaking changes in `kc.sh start --optimized` flow.
- `distribution/distribution:3` → verify it supports OIDC token-auth backend or that `oauth2-proxy` sidecar pattern covers Docker auth flow.
- Azure Files file-locking behavior with Postgres (per-write fsync) — known sharp edge; consider Azure Files **Premium** for the pg-data share if performance is poor on Standard.
## Confidence
| Area | Level |
|------|-------|
| Container Apps pricing + features | HIGH |
| Python + FastAPI + Authlib + Keycloak | HIGH |
| Self-host Postgres on Azure Files | MEDIUM-HIGH (file-locking behavior of Azure Files = pending verification) |
| Self-host registry behind OIDC | MEDIUM (Docker CLI OIDC token-flow requires testing) |
| Trivy + Bicep single-file | HIGH |
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->
## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->



<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
