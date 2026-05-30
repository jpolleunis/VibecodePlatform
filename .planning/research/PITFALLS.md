# PITFALLS — Internal App-Hosting Platform on Azure Container Apps

**Project:** vibecoding-platform
**Researched:** 2026-05-30
**Confidence:** MEDIUM-HIGH (HIGH for stack-specific gotchas; verification pass recommended before treating as final)

> **Verification note.** Pitfalls below are grounded in well-known issues for this exact stack (ACA + Easy Auth + Auth.js v5 + Bicep + ADO + Anthropic). Run a follow-up Context7/WebSearch pass before locking pricing/version-specific numbers.

---

## Critical Pitfalls (cause rewrites, outages, or budget blowups)

### CRIT-1: ACA scale-to-zero kills your background workers and breaks the daily-spend cap

**Confidence:** HIGH

**What goes wrong:** With `minReplicas: 0` on the LLM-proxy container, ACA scales the app to zero after the cooldown (default 300s of no HTTP traffic). The next request takes 3-15s cold start. Worse: any *in-process* counter for the $10/day Claude cap is wiped because the replica is destroyed. The new replica starts with `spent_today = 0`.

**Consequences:**
- First user of the morning hits a 5-15s spinner — looks like the platform is broken.
- Daily-cap counter resets between scaler events — actual spend can be 2-3x the configured cap.
- Background log-flushers, audit-writers, retry queues are killed mid-flight → silent data loss.

**Prevention:**
1. **Daily-cap state MUST live outside the container** — put the counter in Postgres or Redis with `UPDATE ... RETURNING` for atomic decrement. Never in-memory.
2. For the LLM-proxy specifically, set `minReplicas: 1` (one B-tier replica idle costs ~€8-12/month — within €100 budget and worth it for cap-correctness alone).
3. For the Next.js portal, `minReplicas: 0` is fine if you accept cold start. Use a warmup ping only after Phase 1 ships.
4. Add `terminationGracePeriodSeconds: 30` so in-flight requests drain.

**Detection:** Spend reports exceed daily cap. App Insights `requests.duration` p95 > 3s on first request after idle window.

**Phase:** Phase 1 (LLM-proxy infra + cap-enforcement design).

---

### CRIT-2: Daily-spend cap race condition across replicas amplifies cost during traffic spikes

**Confidence:** HIGH

**What goes wrong:** Two replicas read `spent_today = $9.50`, both decide "I can take one more request" (each < $10 cap), both call Claude, both write $9.80. Cap was $10, actual spend is $10.10+ — and during burst, retries amplify this.

**Prevention:**
1. Use Postgres advisory lock or `SELECT ... FOR UPDATE` around cap check + decrement:
   ```sql
   BEGIN;
   SELECT spent_cents FROM daily_cap WHERE day = CURRENT_DATE FOR UPDATE;
   -- if spent + estimated_cost > cap then ROLLBACK and 429
   UPDATE daily_cap SET spent_cents = spent_cents + actual_cost_cents WHERE day = CURRENT_DATE;
   COMMIT;
   ```
2. **Account post-call from `usage` block in Anthropic response**, not pre-call estimate. `input_tokens` + `output_tokens` + `cache_creation_input_tokens` + `cache_read_input_tokens` all have different prices.
3. Disable client retries on your own cap-breach 429; only retry on Anthropic-origin 429/529 with capped exponential backoff (max 2 retries, jitter).
4. Reserve a safety margin — set the actual hard-stop at 90% of the configured cap.

**Detection:** End-of-day reconciliation between cap-counter and Anthropic's usage API. Delta > 5% → locking is broken.

**Phase:** Phase 1 (LLM-proxy daily-cap design — non-negotiable before any user can call Claude).

---

### CRIT-3: Easy Auth redirect-URI mismatch on first Bicep deploy locks you out of the portal

**Confidence:** HIGH

**What goes wrong:** Bicep creates the Container App with a generated FQDN like `portal--abc123.<region>.azurecontainerapps.io`, but the Entra ID app-registration's redirect URI was hardcoded. Easy Auth refuses login: `AADSTS50011: The redirect URI ... does not match`.

**Why it happens:**
- ACA appends a random suffix unless you pre-create the environment + custom domain.
- `/.auth/login/aad/callback` is the Easy Auth callback path — Auth.js v5 uses `/api/auth/callback/microsoft-entra-id` instead. Mixing them is the #1 issue.
- Bicep deploys the app-registration and the container app in parallel — redirect URI is set with a placeholder.

**Prevention:**
1. **Decide upfront: Easy Auth OR Auth.js per surface, not both on the same surface.** For pilot-apps with no per-user state: Easy Auth. For the portal with per-user roles + audit: Auth.js v5. Document the split clearly.
2. If Easy Auth: register **both** redirect URIs during transition: `https://<aca-fqdn>/.auth/login/aad/callback` and (later) `https://<custom-domain>/.auth/login/aad/callback`.
3. Set `accessTokenAcceptedVersion: 2` on the app-registration manifest. Set Easy Auth `allowedTokenAudiences` to include the exact `api://<client-id>` (case-sensitive, no trailing slash).
4. Bicep ordering: create the ACA environment + custom-domain binding before the app-registration redirect URIs. Use `dependsOn` explicitly.
5. Smoke test in pipeline: `curl -I https://portal/...` returns 302 to `login.microsoftonline.com` — fail the deploy if not.

**Detection:** `AADSTS50011`, `AADSTS50013`, `AADSTS700016` in App Insights traces. Browser network tab shows infinite redirect loop.

**Phase:** Phase 1 (auth setup, before any container is deployed).

---

### CRIT-4: Workload Identity Federation 401s from RG-scope misconfiguration

**Confidence:** HIGH

**What goes wrong:** ADO service connection uses WIF with subject claim `sc://<org>/<project>/<service-connection-name>`. Federated credential on the user-assigned MI is at subscription level, but RBAC (Contributor) is at RG level. Pipeline gets a token, then `az deployment group create` returns 401/403. The federated credential `issuer` mismatch gives a misleading "no permission" error.

**Prevention:**
1. **One MI per environment** (dev, prod), not per pipeline. Reduces federated-credential churn (20-credential limit).
2. RBAC at **resource-group scope** matching your Bicep deployment scope. If `targetScope = 'subscription'`, RBAC must be at subscription level.
3. Subject claim exact format: `sc://<org-name>/<project-name>/<service-connection-name>` — verify with `az identity federated-credential show`.
4. First pipeline step: `az account show` and `az group list --query "[?name=='<rg>']"`. If empty, fail fast.
5. Document the WIF setup as a runbook (6 steps; nobody remembers it after 3 months).

**Detection:** `AADSTS70021: No matching federated identity record found`. `AuthorizationFailed` on deployment.

**Phase:** Phase 1 (CI/CD bootstrap, day 1-2).

---

### CRIT-5: Key Vault references on ACA fail silently at cold start

**Confidence:** HIGH

**What goes wrong:** ACA resolves Key Vault references at **container start time**, not at revision-create time. If MI doesn't have `Key Vault Secrets User` RBAC yet (Bicep order or RBAC propagation lag), the container starts, env var is *empty string*, and the app crashes with a confusing "missing config" error — not "Key Vault access denied".

**Prevention:**
1. **Use RBAC, not access-policies.** Set `enableRbacAuthorization: true` on the KV resource and use `Key Vault Secrets User` role.
2. Bicep: explicit `dependsOn` from the Container App to the RBAC assignment. Add a deploy-time `deploymentScript` that polls `az containerapp show` until the revision is `Running`, retrying revision-create if needed.
3. For secret rotation, include secret version in the revision suffix so a new version forces a new revision automatically.
4. Add a `/healthz/secrets` endpoint that returns 503 if any required secret is empty/missing — surfaces the issue in the readiness probe, preventing traffic split to a broken revision.

**Detection:** App Insights: 500 errors immediately after deploy, env-var dump shows secrets as empty. ACA logs: `secret 'X' not found in keyvault`.

**Phase:** Phase 1 (KV + ACA wiring).

---

### CRIT-6: Log Analytics ingest cost blows past €100/month budget alert in week 1

**Confidence:** HIGH

**What goes wrong:** Default ACA log collection sends every stdout/stderr line to Log Analytics. A chatty Next.js dev build emits 50-200 lines per request. ContainerAppConsoleLogs + ContainerAppSystemLogs + Postgres slow-query log + App Insights default 100% sampling = €60-150/month easily from logs alone.

**Prevention:**
1. Set Log Analytics **daily cap** at 1 GB/day (LAW → Usage and estimated costs → Daily cap). Hard ceiling.
2. App Insights `samplingPercentage: 20` for dev, `5` for prod LLM-proxy. Adaptive sampling on.
3. Postgres: `log_min_duration_statement = 1000` (log only queries > 1s). Don't log all statements.
4. ACA: route only `ContainerAppConsoleLogs` to LAW. Send `ContainerAppSystemLogs` to a Storage Account.
5. **30-day retention** (default 90) for LAW.
6. Add a cost-alert at €50 (50% of budget) in addition to €100 — gives a week to react.

**Detection:** Cost Management → "Log Analytics workspace" line item > €30/month. LAW Usage chart > 500 MB/day.

**Phase:** Phase 1 (infra-bicep — must be set before first container starts emitting).

---

## High-Impact Pitfalls

### HIGH-1: ACA ingress revision-mode "Single" vs "Multiple" confusion blocks blue/green

**What goes wrong:** Default `activeRevisionsMode: 'Single'` — every new revision instantly takes 100% traffic. Devs want blue/green but think they have it because they see two revisions.

**Prevention:**
- Phase 1: stick with `Single`. Simpler, matches "ship fast".
- Phase 2+ blue/green: `activeRevisionsMode: 'Multiple'` **and** set `trafficWeights` explicitly. Bicep gotcha: omitting `trafficWeights` in Multiple mode = 0% to all revisions = downtime.
- Always pin `revisionSuffix` (e.g., git short SHA) so revisions are predictable.

**Phase:** Phase 1 (decide mode); Phase 2+ (implement blue/green).

---

### HIGH-2: Auth.js v5 + Entra ID — `trustHost`, session strategy, Edge runtime mistakes

**What goes wrong (multi-part):**
1. Behind ACA's reverse proxy, `AUTH_URL` (renamed from `NEXTAUTH_URL` in v5) must match the public URL exactly. → "URL mismatch".
2. v5 requires `trustHost: true` when behind a proxy (Vercel auto-detects, ACA does not). Without it: `UntrustedHost` error.
3. Default session strategy is `jwt` — fine for SSO. For server-side revocation: `database` strategy + Prisma adapter on Postgres.
4. Provider id is `microsoft-entra-id`, not `azure-ad` (renamed in v5). Old tutorials are wrong.
5. Edge runtime middleware can't use the database adapter — JWT-only there.
6. `tenantId` must be the actual tenant GUID for single-tenant apps, NOT `common` or `organizations` (those allow any tenant — security hole).

**Prevention:**
1. Hardcode `AUTH_URL` per environment in KV.
2. `trustHost: true` in `auth.ts` config.
3. `provider: 'microsoft-entra-id'` with `tenantId: process.env.AZURE_TENANT_ID` explicitly.
4. JWT sessions for Phase 1; move to DB sessions only when force-logout-on-revoke is needed.
5. Middleware check: `if (token.tid !== process.env.AZURE_TENANT_ID) return 403` — defense in depth.

**Phase:** Phase 1 (portal auth setup).

---

### HIGH-3: Bicep + AVM module version drift causes "works on my machine" deploys

**What goes wrong:**
- AVM modules pulled from `br/public:avm/res/...` without pinned version → next deploy gets a new minor with breaking parameter changes.
- AVM module idempotency: re-running a deploy can delete properties not specified (ARM PUT semantics) — e.g., re-deploying ACA wipes manual diagnostic settings added in the portal.

**Prevention:**
1. **Always pin AVM version**: `module foo 'br/public:avm/res/app/container-app:0.11.0' = { ... }`. Never `:latest` or omit.
2. Renovate-bot or quarterly review for AVM updates — read CHANGELOG before bumping.
3. Never click-ops in the portal for resources managed by Bicep. Bicep = single source of truth.
4. Use `az deployment group what-if` in CI before every deploy. Fail pipeline if unexpected deletes appear.

**Phase:** Phase 1 (Bicep skeleton); ongoing.

---

### HIGH-4: ACR + Trivy scanning produces false-fail noise from base-image CVEs

**What goes wrong:** Trivy scans every layer including the Node base. `node:20-alpine` has 10-30 medium CVEs at any time, mostly in OS packages you don't use. Pipeline fails on every push → devs disable scanning → real CVEs slip through.

**Prevention:**
1. `--severity HIGH,CRITICAL` only for fail condition. Report MEDIUM and below to a dashboard, don't fail.
2. `--ignore-unfixed` — nothing actionable about CVEs with no patch.
3. `.trivyignore` for known-acceptable CVEs with expiry date (force review every 90 days).
4. Use distroless or `node:20-slim` as base — smaller attack surface.
5. Trivy in `image` mode against the built image, not the Dockerfile.

**Phase:** Phase 1 (CI setup); revisit baselines monthly.

---

### HIGH-5: Postgres Flex B1ms — CPU credits and connection cap exhaustion

**What goes wrong:**
- B1ms is **burstable**: 20% baseline CPU credit accumulation. Sustained > 20% CPU drains credits in ~30 min, then performance collapses to baseline → 5-10x slower queries.
- B1ms `max_connections` default: ~85. Next.js + Auth.js + LLM-proxy + audit-log writers easily exhaust this.

**Prevention:**
1. **Add pgbouncer from day 1** — either Azure's built-in (`pgbouncer.enabled = true` Bicep param on Flex) or sidecar. Pool mode `transaction`.
2. App side: singleton DB-pool per process, max 5 connections per replica. 2 replicas of portal + 2 of LLM-proxy = 20 connections; under 85.
3. Monitor `cpu_credit_balance` — alert at < 200 remaining.
4. Upgrade trigger: as soon as Phase 2 hits, jump to D2ds_v5 (~€60/month). Don't run prod on burstable beyond MVP.
5. `idle_in_transaction_session_timeout = 30000` (30s) to kill leaked connections.

**Detection:** `SELECT count(*) FROM pg_stat_activity` consistently > 60. Query p95 latency suddenly 5x without traffic change = CPU credits depleted.

**Phase:** Phase 1 (DB setup); upgrade trigger Phase 2.

---

### HIGH-6: CSP headers break Claude-generated artifacts that use inline scripts

**Confidence:** MEDIUM

**What goes wrong:** Sensible CSP (`script-src 'self'`) blocks inline `<script>` in Claude-generated HTML artifacts. The artifact viewer shows a blank page. Devs add `'unsafe-inline'` everywhere → XSS protection gone.

**Prevention:**
1. **Sandbox the artifact viewer in an iframe** with `sandbox="allow-scripts"` and a **distinct origin** (e.g., `artifacts.vibecoding.tag-team.be` vs `portal.vibecoding.tag-team.be`). Artifact subdomain can have looser CSP because same-origin policy contains damage.
2. Strip `<script>` server-side if the artifact must render in the portal origin (DOMPurify with strict allowlist).
3. Portal: nonce-based CSP via middleware. Auth.js v5 docs cover this pattern.
4. Never `'unsafe-eval'` on the portal origin — Next.js doesn't need it in production.

**Phase:** Phase 2 (artifact viewer); CSP baseline in Phase 1.

---

## Moderate Pitfalls

### MOD-1: Audit log integrity — admins can edit Postgres rows

**Prevention:**
1. **Append-only at DB level**: revoke `UPDATE` and `DELETE` from the app role on the audit table. Only `INSERT` and `SELECT`.
2. Hash-chain rows: `prev_hash = sha256(prev_row.serialized + prev_row.prev_hash)`. Tampering breaks the chain.
3. Periodic export to **immutable Azure Storage** (Blob with `immutabilityPolicy` + legal hold). <€1/month.
4. DBA access via JIT (PIM) only, with mandatory ticket reference.

**Phase:** Phase 1 (basic append-only). Hash chain + immutable export → Phase 2.

---

### MOD-2: ACA replica idle billing — `minReplicas:1` × N apps × 24h is real money

**Prevention:**
- `minReplicas: 1` ONLY for: LLM-proxy (cap-state consistency) and portal (UX).
- Internal/admin tools: `minReplicas: 0`, accept cold start.
- Use **Consumption workload profile**, not Dedicated, until multiple high-traffic apps exist.
- CPU/memory at the minimum that passes the readiness probe — most Node apps run fine on 0.25 vCPU / 0.5 GiB.

**Phase:** Phase 1 (per-app sizing decision).

---

### MOD-3: ACR Basic SKU storage limit (10 GB) — image bloat from Next.js

**Prevention:**
1. **Multi-stage Dockerfile**: `output: 'standalone'` in `next.config.js`. Final image 150-300 MB.
2. ACR retention policy: delete untagged manifests after 7 days. `az acr config retention`.
3. Tag with git SHA + `latest`. Untag old SHAs after deploy succeeds.

**Phase:** Phase 1 (Dockerfile design + retention policy).

---

### MOD-4: Anthropic prompt-token vs completion-token vs cache-token accounting

**Prevention:**
1. Full pricing formula from `usage` response:
   `cost = input × $3/MTok + output × $15/MTok + cache_creation × $3.75/MTok + cache_read × $0.30/MTok` (Sonnet 4 rates — verify current).
2. Store the raw `usage` object in the audit table — recalculate cost at end-of-day for reconciliation.
3. Pin the model in code (`claude-sonnet-4-...`) — pricing differs by model.

**Phase:** Phase 1 (LLM-proxy implementation).

---

### MOD-5: Anthropic 529 (overloaded) retry storms

**Prevention:**
1. Exponential backoff with **jitter**: `delay = random(0, base * 2^attempt)`.
2. Max 2 retries total. After that, return 503 with "Claude is busy, try again".
3. Circuit breaker: 5 consecutive 529s in 1 min → stop calling Anthropic for 2 min.
4. Idempotency keys: pass `anthropic-idempotency-key` header when supported.

**Phase:** Phase 1 (LLM-proxy retry policy).

---

### MOD-6: Custom domain + managed cert provisioning timing

**Confidence:** MEDIUM

**What goes wrong:** ACA managed certificates require DNS CNAME + TXT validation. Provisioning takes 10-30 min. First deploy: Bicep "succeeds" but HTTPS returns cert error for 20 min.

**Prevention:**
1. Pre-create the custom domain binding **once, manually**, the day before first prod deploy. Cert provisioning is idempotent.
2. Document in runbook: "managed cert can take 30 min on first bind — wait, don't redeploy".
3. For dev environment, use the default ACA FQDN — skip custom domain.

**Phase:** Phase 1.5+ (custom domain not in Phase 1 scope).

---

## Minor Pitfalls

### MIN-1: Next.js 15 App Router — `cookies()` / `headers()` are now async

Next.js 15 made `cookies()`, `headers()`, `draftMode()` async. Auth.js v5 examples on the web are mixed. Sync usage shows deprecation in dev, **runtime error in prod**. Always `await` them.

**Phase:** Phase 1 (any server-component using auth).

---

### MIN-2: Bicep parameter file (`.bicepparam`) vs JSON parameter file confusion

`.bicepparam` is newer (2023+), supports KV references and expressions. `.parameters.json` is legacy. Pick `.bicepparam`, delete the rest.

**Phase:** Phase 1 (Bicep skeleton).

---

### MIN-3: Time zone in audit logs — UTC vs Europe/Brussels

Postgres `timestamp without time zone` + app inserting local time = ambiguous audit logs (especially across DST). Use `timestamptz` always. Display in Brussels TZ in the portal, store in UTC.

**Phase:** Phase 1 (schema design).

---

### MIN-4: Next.js standalone output forgets `public/` and `.next/static`

`output: 'standalone'` only copies server code. Must manually `COPY --from=build /app/public ./public` and `COPY --from=build /app/.next/static ./.next/static` in the Dockerfile, or static assets 404 in production.

**Phase:** Phase 1 (Dockerfile).

---

### MIN-5: ADO Service Connection scoping — too broad

By default, the WIF service connection is granted Contributor on the **subscription**. Scope to the RG only. Use a separate service connection per environment (dev/prod) for least privilege.

**Phase:** Phase 1 (CI setup).

---

## Anti-Recommendations (skip these multi-tenant SaaS patterns)

Internal, single-tenant platform — skip these despite their popularity in SaaS blogs:

| Pattern | Why skip |
|---------|---------|
| Row-Level Security (RLS) for tenant isolation | Single tenant. Schema-per-app at most. |
| Stripe-style metered billing | Internal users. Just track usage for chargeback Excel report. |
| Multi-region deployment | One Belgian team. Single region (West Europe). |
| `organizations` or `common` issuer in Entra | Single tenant — pin tenantId. `common` is a security regression. |
| Per-tenant Key Vault | One KV per environment is plenty. |
| Public-facing rate limiting (Cloudflare, etc.) | Behind Easy Auth. No anonymous traffic. |
| Sophisticated feature flags (LaunchDarkly) | 2 devs, 1 product. Env-var booleans suffice. |
| OpenTelemetry export to third-party APM | App Insights covers it. €30+/month for nothing. |
| Public status page (statuspage.io) | Internal — Teams channel suffices. |
| GDPR data-residency complexity | Internal employee data, EU tenant. Standard West Europe deployment satisfies. |
| CAPTCHA / bot protection | Behind SSO. No anonymous access. |

---

## Phase-Specific Warning Map

| Phase Topic | Critical Pitfalls Active | Mitigation Owner |
|-------------|--------------------------|------------------|
| Phase 1 — Auth bootstrap | CRIT-3, HIGH-2, MIN-1 | Dev A |
| Phase 1 — Infra Bicep | CRIT-5, HIGH-3, CRIT-6, MOD-2 | Dev B |
| Phase 1 — CI/CD WIF | CRIT-4, MIN-5, HIGH-4 | Dev A |
| Phase 1 — LLM-proxy + daily cap | CRIT-1, CRIT-2, MOD-4, MOD-5 | Dev B |
| Phase 1 — Postgres | HIGH-5, MOD-1, MIN-3 | Dev A |
| Phase 1 — Docker/ACR | MOD-3, MIN-4 | Dev B |
| Phase 2 — Artifact viewer | HIGH-6 | TBD |
| Phase 2 — Audit hardening | MOD-1 (hash chain + immutable export) | TBD |

---

## Recommendations for the Roadmap

1. **Treat CRIT-1/CRIT-2/CRIT-3 as Phase 1 blockers** — no production traffic until each is verified.
2. **Re-run a verification pass** before Phase 1 starts to lock down:
   - Current Anthropic pricing for Sonnet 4 (CRIT-2, MOD-4)
   - AVM `avm/res/app/container-app` latest version + breaking changes (HIGH-3)
   - Auth.js v5 stable release status (HIGH-2)
3. **Pre-deploy smoke tests in pipeline** for every CRIT pitfall — `/healthz/secrets`, redirect-302 check, `az account show`, `pg_stat_activity` count.
