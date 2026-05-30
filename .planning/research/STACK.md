# Stack Research — VibeCoding Platform

**Domain:** Internal Platform-as-a-Service / app-launcher on Azure Container Apps
**Researched:** 2026-05-19
**Confidence:** HIGH (versions verified against official sources May 2026)

## Executive verdict

The stack listed in `PROJECT.md` is **fundamentally correct and current**. It matches the 2026 mainstream pattern for "Bicep + Container Apps + Entra ID + Next.js + Auth.js + Prisma" internal platforms. The main corrections and additions:

- **Use Microsoft Entra ID provider, not Azure AD provider** in Auth.js v5 (Azure AD provider is legacy in Auth.js v5).
- **Add Application Insights** on top of Log Analytics — Log Analytics is the storage layer; App Insights is the developer-facing APM. Both share one workspace.
- **Use Workload Identity Federation (WIF)** for the Azure DevOps service connection, not a service principal secret. This is now the Microsoft default.
- **Use Azure Verified Modules (AVM) Bicep modules** as a base instead of writing every resource from scratch.
- **Add Renovate Bot** as the Dependabot equivalent for Azure DevOps (Dependabot-on-ADO via GHAS is paid and may not be approved by TAG IT for a pilot).
- **Pin Tailwind v4 + shadcn/ui via `shadcn@latest`** (Tailwind v4 + React 19 is stable in shadcn as of late 2025; no longer canary-only).
- **Consider ACR Standard** (not Basic) only if you hit throughput limits — for a pilot, Basic is fine.

Everything else (Express proxy, Key Vault + Managed Identity, Trivy in pipeline, cost alert) matches current best practice.

## Recommended Stack

### Core Technologies

| Technology | Version (May 2026) | Purpose | Why Recommended |
|------------|--------------------|---------|-----------------|
| **Azure Container Apps** | API `2026-01-01` | Serverless container hosting | Per-app isolation without managing K8s; native scale-to-zero; Easy Auth = zero-code SSO; per-app revisions = blue/green deploys built-in. AKS = overkill for 1-2 apps; App Service = no real container orchestration. |
| **Next.js** | **15.2.x (LTS line)** | Portal framework (App Router) | Current stable LTS as of May 2026. Next.js 16 exists but 15.x is the conservative, library-compatible LTS line through 2026. App Router + Server Components is the dominant 2026 pattern. |
| **React** | **19.x** | UI library | Bundled with Next.js 15; required for current shadcn/ui. |
| **TypeScript** | **5.6+** | Type-safe portal | Required by Next.js 15 / Prisma 6 / Auth.js v5 toolchains. |
| **Auth.js (NextAuth)** | **v5.0 (stable)** | Portal auth | v5 is the current line for Next.js 15/16 App Router. Use the **Microsoft Entra ID provider** (`@auth/core/providers/microsoft-entra-id`), not the legacy Azure AD provider. |
| **Prisma ORM** | **6.19.x** | DB access from portal | Current 6.x line (6.19.2 latest patch, Jan 2026). Rust-free engine in preview; query-plan-cache tunable; PostgreSQL JSON-list filter bugs fixed. v7 not yet GA. |
| **PostgreSQL** | **16** (Flexible Server) | Portal database | Default for Azure Database for PostgreSQL Flexible Server in 2026; matches Prisma 6 supported matrix. |
| **Azure Database for PostgreSQL — Flexible Server** | Burstable **B1ms** (1 vCore / 2 GiB) for pilot | Managed Postgres | Cheapest option (~€11/month); fits pilot. **Upgrade to General Purpose D2ds_v5** before real production — Burstable explicitly "for dev/test workloads" per Microsoft. |
| **Tailwind CSS** | **v4.x** | Styling | v4 is stable, 10× faster builds, zero-config content detection, native CSS variables. v3 still works but new shadcn projects default to v4. |
| **shadcn/ui** | latest (Tailwind v4 + React 19) | Component library | Use `npx shadcn@latest init`. The Tailwind v4 / React 19 generators graduated from `@canary` to the default `@latest` track in late 2025. |
| **Node.js** | **22 LTS** (Iron) | Runtime for portal + proxy | Active LTS until April 2027; matches Container Apps base images and Anthropic SDK requirements. |
| **Express** | **4.21.x** | LLM proxy framework | Minimal, well-known, fits one-endpoint proxy. Express 5 is GA but 4.x is what 99% of production middleware (rate-limit, helmet) targets — use 4.x for pilot. |
| **Bicep** | latest (Bicep CLI 0.31+) | IaC | Microsoft default IaC for Azure; AVM (Azure Verified Modules) GA February 2026; Microsoft retired classic ALZ-Bicep on 16 Feb 2026 in favor of AVM. |
| **Azure Container Registry (ACR)** | **Basic SKU** for pilot | Image registry | Basic supports MI auth + Entra ID + webhooks. Standard tier only needed if image throughput becomes a bottleneck. Premium (geo-replication, private endpoints) not needed for pilot. |
| **Azure Key Vault** | Standard SKU | Secret storage | Use Container Apps' `keyvaultref:` syntax + Managed Identity. Never put Claude key or DB password in app env vars or pipeline variables. |
| **Azure Log Analytics workspace** | — | Log sink | Auto-attached to Container Apps Environment; PAYG pricing. |
| **Azure Application Insights** | workspace-based | APM / traces / errors | **Missing from your spec — add it.** AI is the developer UX layer (transaction tracing, exceptions, performance). It writes into the same Log Analytics workspace, so no extra cost beyond ingestion. Container Apps does NOT support the AI auto-instrumentation agent — instrument the portal/proxy code with the AI SDK (`@azure/monitor-opentelemetry`). |
| **Azure DevOps Pipelines** | — | CI/CD | Matches your IT context (TAG is Microsoft-stack). Use multi-stage YAML pipelines. |
| **Workload Identity Federation (WIF)** | — | ADO → Azure auth | **Use WIF for the ADO service connection, not a service principal secret.** WIF is GA, recommended by Microsoft, and removes long-lived secrets from ADO. |
| **Trivy** | **0.58+** | Container image scanning | Open-source, fast, and — relevant context — Microsoft Defender for Containers internally uses Trivy. The official `aquasecurity/trivy-azure-pipelines-task` extension publishes scan results in the ADO UI. Set `--severity HIGH,CRITICAL --exit-code 1` to fail builds. |
| **Anthropic TypeScript SDK** | `@anthropic-ai/sdk` **0.97.x** | Claude API client | Official SDK; supports automatic retries, timeouts, strict types. Avoid hand-rolling fetch. |

### Supporting Libraries (portal)

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `@auth/prisma-adapter` | 2.x | Persist Auth.js sessions/users in Postgres | Always — needed for the audit-log + role-tables to share the DB. |
| `zod` | 3.23+ | Runtime schema validation | Validate all API route inputs (role-change, promote-to-prod). |
| `@tanstack/react-query` | 5.x | Server state in client components | Dashboard polling app status. Optional — Server Components cover most of it. |
| `lucide-react` | latest | Icons | Default icon set for shadcn/ui. |
| `next-safe-action` | 7.x | Type-safe server actions | Wraps RBAC checks around admin actions (promote, role-change). Optional but clean. |
| `pino` + `pino-http` | latest | Structured JSON logs | Container Apps logs JSON well in Log Analytics. Avoid `console.log` for prod. |

### Supporting Libraries (LLM proxy)

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `express` | 4.21.x | HTTP framework | Pilot proxy. |
| `helmet` | 8.x | Security headers | Auto-CSP, X-Frame-Options, etc. Pair with explicit CSP for inline-script needs of the static artifact. |
| `express-rate-limit` | **8.5.2** | Per-user / per-IP rate limit | The de-facto Express rate-limiter. Use a custom `keyGenerator` to key on the `x-ms-client-principal-id` header that Easy Auth injects → true per-user limits. |
| `rate-limiter-flexible` | 5.x | Alternative if you need a Postgres/Redis-backed counter for the **$10/day hard cap** | `express-rate-limit` is in-memory; the daily-spend cap must survive container restarts → back it with Postgres via `rate-limiter-flexible`'s PG store, or implement a simple `daily_spend` table. |
| `@anthropic-ai/sdk` | 0.97.x | Claude API | See above. |
| `pino-http` | latest | Request logging | Same as portal. |

### Azure Resources (Bicep modules)

Use **AVM (Azure Verified Modules)** from the public Bicep registry as the base — do not hand-roll these:

| Resource | AVM module reference |
|----------|---------------------|
| Container Apps Environment | `br/public:avm/res/app/managed-environment:<v>` |
| Container App | `br/public:avm/res/app/container-app:<v>` |
| Key Vault | `br/public:avm/res/key-vault/vault:<v>` |
| Container Registry | `br/public:avm/res/container-registry/registry:<v>` |
| Postgres Flexible Server | `br/public:avm/res/db-for-postgre-sql/flexible-server:<v>` |
| Log Analytics workspace | `br/public:avm/res/operational-insights/workspace:<v>` |
| Application Insights | `br/public:avm/res/insights/component:<v>` |
| User-assigned Managed Identity | `br/public:avm/res/managed-identity/user-assigned-identity:<v>` |
| Cost budget | `br/public:avm/res/consumption/budget:<v>` |

Pin `<v>` to a specific version (don't use `latest`) for reproducible builds.

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| **pnpm** | Package manager | Faster + disk-efficient vs npm; monorepo-friendly for portal + proxy in one repo. Add `packageManager` field to `package.json` to lock it. |
| **Biome** or **ESLint + Prettier** | Lint + format | Biome is faster and replaces both. Either is fine; pick one and don't run both. |
| **Vitest** | Unit tests | Faster than Jest; native TS/ESM; default for Next.js 15 projects in 2026. |
| **Playwright** | E2E tests | For the "login → see dashboard → see meeting agent URL" smoke test. Run in pipeline against the test slot. |
| **Bicep CLI 0.31+** | Lint / build Bicep | Run `bicep lint` and `bicep build` in pipeline. |
| **azd (Azure Developer CLI)** | Optional one-command deploy | Useful for local dev loops; not required if you have Bicep + ADO pipelines. |
| **Renovate Bot** | Dependency updates | **Recommended over Dependabot-on-ADO.** Renovate runs natively on ADO repos (no GHAS license needed), open-source, and is the de-facto standard for non-GitHub repos in 2026. |
| **Trivy CLI (local)** | Dev-time scan | Same engine the pipeline uses; lets devs scan locally before push. |

## Installation

```bash
# --- Portal (apps/portal) ---
pnpm create next-app@15 portal --typescript --tailwind --app --src-dir --import-alias "@/*"
cd portal
# Tailwind v4 is now the default of create-next-app; nothing extra to install.

# shadcn/ui
pnpm dlx shadcn@latest init

# Auth.js v5 + Entra ID provider + Prisma adapter
pnpm add next-auth@beta @auth/prisma-adapter

# Prisma + Postgres
pnpm add @prisma/client
pnpm add -D prisma
pnpm dlx prisma init --datasource-provider postgresql

# Supporting
pnpm add zod lucide-react pino pino-http
pnpm add @azure/monitor-opentelemetry @opentelemetry/api  # Application Insights via OTel
pnpm add @azure/identity @azure/keyvault-secrets         # Managed Identity → Key Vault (if portal needs it directly)

# --- LLM proxy (apps/llm-proxy) ---
pnpm add express helmet express-rate-limit rate-limiter-flexible
pnpm add @anthropic-ai/sdk@0.97
pnpm add pino pino-http
pnpm add @azure/identity @azure/keyvault-secrets
pnpm add -D @types/express tsx typescript

# --- Repo-wide dev tools ---
pnpm add -D vitest @playwright/test biome
```

For Bicep, install the CLI via Azure CLI:
```bash
az bicep install
az bicep upgrade   # ensure ≥ 0.31
```

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| Azure Container Apps | Azure App Service for Containers | If you need only HTTP web apps (no jobs, no scale-to-zero, no Dapr) and want App Service's slot-swap / deployment-center UX. Not better for a multi-app launcher. |
| Azure Container Apps | AKS | If you grow beyond ~20 apps OR need custom networking (multi-VNET peering, custom CNI, service mesh). Overkill at pilot scale. |
| Auth.js v5 | Microsoft Authentication Library (MSAL) for Node | If you need fine-grained token-cache control or on-behalf-of flow. For an SSO portal, Auth.js is far less code. |
| Prisma 6 | Drizzle ORM | If you want SQL-first / zero-runtime ORM. Prisma wins on DX + Azure tooling familiarity for a 2-dev team. |
| Tailwind v4 + shadcn/ui | Mantine, Chakra, Radix-only | shadcn copies components into your repo (no runtime dep), Tailwind v4 is the fastest. Mantine is fine but heavier and locks you into its theming system. |
| Trivy in pipeline | Microsoft Defender for Containers (registry scan) | If TAG already has Defender for Cloud licensed, enable registry scanning as a defense-in-depth layer. **Both can coexist** — Defender uses Trivy internally. Pilot doesn't need both. |
| Log Analytics + App Insights | Datadog / Grafana Cloud | If you have an existing observability stack. For an Azure-native pilot with €100 budget, native is cheaper and zero-config. |
| Workload Identity Federation | Service Principal with client secret | Only if WIF is blocked by tenant policy. WIF is strictly better — no secret to rotate. |
| Azure Container Registry Basic | ACR Standard | If pilot's image-pull throughput is hit (unlikely for 4 images). |
| Renovate | Dependabot on ADO (via GHAS) | If TAG already pays for GitHub Advanced Security for Azure DevOps. Otherwise Renovate is free + capable. |
| Direct Container Apps ingress | Azure Front Door in front | Only when you need: WAF, multi-region failover, CDN, or **a custom domain on `apps.tag-team.be`** (Front Door simplifies cert management). For pilot on `*.azurecontainerapps.io`, skip it. |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| **`AzureADProvider` from Auth.js** | Legacy provider, not OIDC-spec compliant with Auth.js v5; Microsoft renamed Azure AD → Entra ID. | `MicrosoftEntraIDProvider` from `next-auth/providers/microsoft-entra-id`. |
| **Service principal client secret in ADO service connection** | Long-lived secret = rotation pain + leak risk; ADO portal nags you to migrate. | Workload Identity Federation (WIF). |
| **Storing Claude key as a Container Apps env var directly** | Visible in portal, in Bicep state, and in `az containerapp show`. | `secretRef:` pointing to a Container Apps secret backed by `keyvaultref:` + the app's Managed Identity. |
| **`pages/` router in Next.js** | Frozen feature set; new docs and shadcn assume App Router. | App Router (`app/` directory) with React Server Components. |
| **`next-auth@4`** | EOL track; doesn't support App Router middleware properly. | `next-auth@5` (Auth.js v5). |
| **`prisma@5` or older** | Slower engine; missing PG-list-JSON fixes; query-plan-cache not tunable. | `prisma@6`. |
| **Tailwind v3 for a new project in 2026** | Slower builds; not the default of `create-next-app` anymore; shadcn defaults to v4 components. | Tailwind v4. |
| **Hand-rolled Azure RBAC checks in app code** | Easy Auth already validates the JWT and injects `x-ms-client-principal-*` headers — read those, don't re-validate. | Trust Easy Auth's headers in the proxy; do **role**-checks (admin vs developer) in the portal against your own DB. |
| **`console.log` for app logs** | Unstructured; harder to query in Log Analytics; loses request correlation. | `pino` JSON logs + Application Insights for traces. |
| **API keys in `pipeline-variables` (even "secret" ones)** | Logged in pipeline runs in some cases; not auditable. | Key Vault + WIF + `AzureKeyVault@2` task or Managed Identity at runtime. |
| **Postgres Single Server (Azure)** | Retired/retiring tier; "Single Server" was the legacy SKU. | Postgres **Flexible Server**, B1ms for pilot. |
| **`shadcn-ui` (old name)** | Package was renamed. | `shadcn` (`pnpm dlx shadcn@latest`). |
| **ACR `admin user` for image pulls** | Enables a built-in username/password — anti-pattern when Managed Identity is available. | Container App's Managed Identity granted `AcrPull` on the registry. |
| **Custom-built CSP via string concat in code** | Easy to break; hard to audit. | `helmet` with explicit `contentSecurityPolicy` config, tested against the static Meeting Agent artifact. |
| **OpenAI / Azure OpenAI SDK in proxy** | Pilot decision: Anthropic-direct. Don't add abstraction layers ("LLM router") in Phase 1. | `@anthropic-ai/sdk` directly. Add a router in Phase 2 if multi-provider becomes a requirement. |

## Stack Patterns by Variant

**If TAG IT denies WIF for ADO service connections:**
- Fall back to service principal **with certificate-based auth** stored in Key Vault, not a client secret string.
- Document the deviation in `docs/it-request.md` so it gets re-raised at Phase 2.

**If TAG already pays for Microsoft Defender for Cloud:**
- Enable **Defender for Containers** on the subscription → registry scans run automatically; ADO pipeline gets a Defender quality gate via the **Microsoft Security DevOps (MSDO)** extension (which itself orchestrates Trivy, Checkov, etc.).
- You can then drop the standalone Trivy task or keep both as defense-in-depth.

**If you need a custom domain (`apps.tag-team.be`) in Phase 1.5:**
- Add **Azure Front Door Standard** in front of the Container Apps Environment.
- Use one Front Door endpoint with multiple routes (one per app) → keeps cert management central and gives you a WAF.
- Reconfigure Easy Auth to accept the Front Door host (`forwardProxy` config).

**If the daily-spend cap needs to be enforceable across multiple replicas:**
- Don't use `express-rate-limit`'s in-memory store — back the daily counter with **Postgres** via `rate-limiter-flexible`'s `RateLimiterPostgres`, or a simple `INSERT … ON CONFLICT DO UPDATE` on a `daily_spend (date, total_usd)` row.

**If Prisma migrations conflict between test and prod app slots:**
- Run `prisma migrate deploy` as a **pipeline job that targets the database directly** (not in the container's startup script). This avoids two replicas racing on the same migration.

## Version Compatibility

| Package A | Compatible With | Notes |
|-----------|-----------------|-------|
| `next@15.2` | `react@19`, `react-dom@19` | React 19 required for current shadcn components. |
| `next-auth@5` | `next@15`, `next@16` | v5 supports App Router middleware out of the box; v4 doesn't. |
| `next-auth@5` | `@auth/prisma-adapter@2` | v1 adapter is for `next-auth@4` only. |
| `prisma@6.19` | `@prisma/client@6.19` | Always keep these versions in lock-step. |
| `@anthropic-ai/sdk@0.97` | `node@>=20` | Node 22 LTS recommended; SDK refuses Node 18. |
| `tailwindcss@4` | PostCSS 8 not required (Tailwind v4 has its own engine) | Don't add `autoprefixer` — v4 handles it. |
| `bicep CLI 0.31` | AVM modules `br/public:avm/...:0.x` | Pin module versions; don't use `latest` in module references. |
| `express-rate-limit@8` | `express@4` and `express@5` | Drop-in for both; verify the in-memory store is acceptable (multi-replica = not). |
| Container Apps API `2026-01-01` | Bicep CLI 0.30+ | Older Bicep versions don't know the latest auth-config schema fields. |

## Sources

- [Next.js 15 release blog](https://nextjs.org/blog/next-15) — HIGH confidence, official.
- [Next.js current version (Mar 2026: 15.2.4 stable)](https://www.abhs.in/blog/nextjs-current-version-march-2026-stable-release-whats-new) — MEDIUM (third-party but consistent with endoflife.date).
- [endoflife.date — Next.js](https://endoflife.date/nextjs) — HIGH, used for LTS line confirmation.
- [Auth.js — Microsoft Entra ID provider](https://authjs.dev/getting-started/providers/microsoft-entra-id) — HIGH, official, says legacy Azure AD provider "does not work out of the box with Microsoft Entra ID".
- [Auth.js v5 migration guide](https://authjs.dev/getting-started/migrating-to-v5) — HIGH, official.
- [Prisma 6.19.0 release](https://www.prisma.io/blog/announcing-prisma-6-19-0) — HIGH, official.
- [Prisma releases on GitHub](https://github.com/prisma/prisma/releases) — HIGH, version-of-truth.
- [Microsoft Learn — Authentication in Container Apps with Microsoft Entra ID](https://learn.microsoft.com/en-us/azure/container-apps/authentication-entra) — HIGH, official.
- [Microsoft Learn — Container Apps authentication overview](https://learn.microsoft.com/en-us/azure/container-apps/authentication) — HIGH.
- [shadcn/ui — Tailwind v4 docs](https://ui.shadcn.com/docs/tailwind-v4) — HIGH, official.
- [shadcn/ui — Next.js install docs](https://ui.shadcn.com/docs/installation/next) — HIGH.
- [Microsoft Learn — Compare Container Apps with other Azure options](https://learn.microsoft.com/en-us/azure/container-apps/compare-options) — HIGH.
- [Azure Container Apps vs AKS 2026 decision matrix](https://www.dataa.dev/2026/05/08/azure-container-apps-vs-aks-the-2026-decision-matrix/) — MEDIUM (independent, dated May 2026).
- [Azure Verified Modules — Bicep index](https://azure.github.io/Azure-Verified-Modules/indexes/bicep/) — HIGH, official.
- [AVM Container App module source](https://github.com/Azure/bicep-registry-modules/tree/main/avm/res/app/container-app) — HIGH, official.
- [Microsoft Community Hub — Release of Bicep AVM for Platform Landing Zone](https://techcommunity.microsoft.com/blog/azuretoolsblog/release-of-bicep-azure-verified-modules-for-platform-landing-zone/4487932) — HIGH, official, confirms ALZ-Bicep retirement on 16 Feb 2026.
- [Microsoft Learn — Microsoft.App/containerApps reference](https://learn.microsoft.com/en-us/azure/templates/microsoft.app/containerapps) — HIGH, confirms API version `2026-01-01`.
- [Trivy — Azure DevOps integration](https://trivy.dev/docs/latest/tutorials/integrations/azure-devops/) — HIGH, official.
- [Microsoft Learn — Microsoft Security DevOps Azure DevOps extension](https://learn.microsoft.com/en-us/azure/defender-for-cloud/configure-azure-devops-extension) — HIGH, confirms MSDO bundles Trivy.
- [Microsoft Learn — Observability in Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/observability) — HIGH.
- [Microsoft Learn — Container Apps does not support AI auto-instrumentation agent](https://learn.microsoft.com/en-us/azure/spring-apps/migration/migrate-to-azure-container-apps-monitoring) — HIGH, why we use the OTel SDK explicitly.
- [Microsoft Learn — Postgres Flexible Server compute options](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-compute) — HIGH, confirms Burstable is "for dev/test workloads".
- [Azure Pricing — Postgres Flexible Server](https://azure.microsoft.com/en-us/pricing/details/postgresql/flexible-server/) — HIGH, official.
- [npm — `@anthropic-ai/sdk`](https://www.npmjs.com/package/@anthropic-ai/sdk) — HIGH, latest 0.97.1.
- [npm — `express-rate-limit`](https://www.npmjs.com/package/express-rate-limit) — HIGH, latest 8.5.2.
- [Azure DevOps Blog — WIF GA announcement](https://devblogs.microsoft.com/devops/workload-identity-federation-for-azure-deployments-is-now-generally-available/) — HIGH, official.
- [Microsoft Learn — Set up Resource Manager WIF service connection](https://learn.microsoft.com/en-us/azure/devops/pipelines/release/configure-workload-identity?view=azure-devops) — HIGH.
- [Microsoft Learn — ACR SKU features and limits](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-skus) — HIGH.
- [Microsoft Learn — Dependabot on Azure DevOps roadmap](https://learn.microsoft.com/en-us/azure/devops/release-notes/roadmap/2024/ghazdo/dependabot) — HIGH, confirms Dependabot-on-ADO requires GHAS.
- [Renovate (Mend) docs — Azure DevOps support](https://rafter.so/blog/sca-tools-comparison) — MEDIUM, confirms Renovate as the ADO-friendly alternative.

---
*Stack research for: Internal Platform-as-a-Service on Azure Container Apps*
*Researched: 2026-05-19*
