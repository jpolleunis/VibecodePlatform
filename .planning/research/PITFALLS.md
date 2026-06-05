# PITFALLS — Container-Apps-only + Self-host Python rebuild

**Project:** vibecoding-platform
**Researched:** 2026-05-30 (Sessie 2 — scope reduction)
**Confidence:** MEDIUM-HIGH (HIGH for ACA + self-host gotchas; verification pass recommended before treating as final)

---

## Critical Pitfalls

### CRIT-1: ACA scale-to-zero kills your in-process daily-cap counter (unchanged)

**Confidence:** HIGH

**What goes wrong:** Same as in the Azure-heavy version. With `minReplicas: 0` on the LLM-proxy, ACA scales to zero, and any in-process counter for the $10/day cap resets on cold start.

**Prevention:**
1. **Daily-cap state MUST live in Postgres**, not in-process. `SELECT … FOR UPDATE` + 90% safety margin.
2. LLM-proxy `minReplicas: 1` in both test and prod (one B-tier replica idle costs ~€2-3/month — well within the €30 budget).
3. `terminationGracePeriodSeconds: 30` so in-flight requests drain.

**Phase:** Phase 1 (LLM-proxy infra + cap-enforcement design).

---

### CRIT-2: Daily-cap race condition across replicas (unchanged)

Same mitigation as in the original stack — `SELECT … FOR UPDATE` + cost-from-`usage` post-call + 90% safety margin. See ARCHITECTURE.md §5 for the transaction pattern.

**Phase:** Phase 1.

---

### CRIT-3: Self-host Postgres on Azure Files = file-locking surprises

**Confidence:** HIGH

**What goes wrong:** Azure Files (SMB) doesn't honor every `fsync`/`flock` semantic that Postgres assumes. Result: occasional WAL corruption on restart, very slow checkpoint writes, or "could not write to relation X" errors under load. This is a well-documented sharp edge of running Postgres on SMB-mounted volumes.

**Consequences:**
- Random WAL corruption requiring restore.
- Sustained query latency 5-20× higher than local disk.
- Postgres refuses to start if it detects stale lock file from a previous run.

**Prevention:**
1. **Use Azure Files Premium (NFS-backed)** for the `pg-data` share, not Standard SMB. ~€7/month for the minimum 100 GiB quota — still cheaper than managed Postgres.
2. If Standard MUST be used, mount with `nobrl,noserverino,actimeo=0` and set Postgres `wal_sync_method=fsync` (explicit, don't rely on default).
3. **Nightly pg_dump to second share is mandatory**, not optional. 14-day retention.
4. **Restore drill in Phase 1 DoD** — corrupt pg-data share, restore from latest dump, confirm portal works within 1 hour.
5. Use the external Python recovery platform (user's existing tooling) as a second safety net for catastrophic loss.

**Phase:** Phase 1 (Postgres infra setup).

---

### CRIT-4: Keycloak realm config not persisted = login broken after restart

**Confidence:** HIGH

**What goes wrong:** Keycloak stores its realm/users/clients in the Postgres DB. If `JAVA_OPTS` or `KC_DB_*` env vars are misconfigured, Keycloak boots with an in-memory H2 dev DB. Restart = all users + clients lost. Worse: silent — login still seems to work briefly with whatever was bootstrapped.

**Consequences:**
- On every container restart, all role assignments + audit go away.
- Even worse if production users were added manually — gone with no recovery.

**Prevention:**
1. **Always set `KC_DB=postgres` + `KC_DB_URL` + `KC_DB_USERNAME` + `KC_DB_PASSWORD`** explicitly. Never rely on defaults.
2. **Realm-export checked into git** (`infra/keycloak/realm-export.json`). Pipeline `keycloak-realm-sync.yml` applies it on every deploy via `kcadm.sh`.
3. Boot Keycloak with `kc.sh start --optimized` only after first warm-up build (`kc.sh build`) — both must run with the same DB env vars.
4. Smoke test in `infra.yml` pipeline: `curl https://keycloak…/realms/vibecoding/.well-known/openid-configuration` returns 200 with `issuer` matching expected URL.
5. Bootstrap admin password in ACA-secret; rotate after first login.

**Detection:** Keycloak logs `Using H2 database` on boot. `/realms/master` returns instead of `/realms/vibecoding`.

**Phase:** Phase 1 (Keycloak setup, before any other auth).

---

### CRIT-5: Self-host Docker registry exposes images publicly if oauth2-proxy is bypassed

**Confidence:** HIGH

**What goes wrong:** `distribution/distribution:3` listens on `:5000` with no auth by default. ACA ingress maps that to public HTTPS. Without `oauth2-proxy` sidecar in front, anyone with the URL can `docker pull` any image. With it in front but misconfigured (wrong upstream port, missing `--reverse-proxy=true`), the Docker CLI flow breaks — devs disable auth in frustration.

**Consequences:**
- Internal images leaked publicly (source code / secrets baked in images).
- Or: no one can pull, builds fail, devs disable security.

**Prevention:**
1. **oauth2-proxy sidecar pattern**: oauth2-proxy on port `4180` is the only ingress target; it proxies to `localhost:5000` (registry) after auth. Bicep ingress points to `4180`.
2. Set `OAUTH2_PROXY_SKIP_AUTH_REGEX = ^/v2/$` (Docker pings unauthenticated `/v2/` for capability discovery).
3. Set `OAUTH2_PROXY_PASS_AUTHORIZATION_HEADER=true` + `OAUTH2_PROXY_REVERSE_PROXY=true` for `docker login` flow with bearer-tokens.
4. **Test from a clean machine**: `docker login <registry-url>` should redirect to Keycloak; anonymous `curl -X GET <registry-url>/v2/_catalog` should return 401.
5. Disable v1 registry API entirely (`distribution/distribution:3` already does, but verify).

**Phase:** Phase 1 (registry setup).

---

### CRIT-6: ACA `secrets:` have no rotation tooling — manual procedure required

**Confidence:** HIGH

**What goes wrong:** ACA `secrets:` are revision-scoped — updating a secret value does NOT update existing revisions. Apps continue using the old value until a new revision is created. Without a documented procedure, secrets get "rotated" but apps keep using the old value silently.

**Prevention:**
1. **Document in `docs/runbook.md`:** "To rotate secret X: (a) update Bicep `secrets:` value or pipeline-variable, (b) run `az containerapp update` with `--set-env-vars` to force new revision, (c) verify via `az containerapp revision list`."
2. Use **pipeline variables** for rotatable secrets (Anthropic key, SP-secret) so rotation = pipeline run; immutable secrets (Keycloak admin password) stay in Bicep.
3. **`/healthz/secrets` endpoint** on portal + proxy returns 503 if any required secret is empty/missing or test-validates as expired — surfaces a failed rotation in readiness probe.
4. Bicep-managed secrets: changing the value in `.bicepparam` and re-deploying creates a new revision automatically (`identifier`-based revisions).

**Phase:** Phase 1 (KV-equivalent setup), runbook in Phase 5.

---

### CRIT-7: Log Analytics ingest from chatty Python apps blows the budget alert

**Confidence:** HIGH

**Same risk class as the original** — but now Python `structlog` JSON is the source. Default Python loggers + uvicorn access logs emit 5-50 lines per request.

**Prevention:**
1. LAW **daily cap 500 MB** (lower than original 1 GB).
2. `structlog` JSON renderer; **drop noisy keys** before emit.
3. Uvicorn: `--no-access-log` in production; rely on FastAPI middleware to log only error+warn paths.
4. Postgres: `log_min_duration_statement = 1000`.
5. Cost-alert at €15 (50% of €30 budget) in addition to €30.

**Phase:** Phase 1.

---

## High-Impact Pitfalls

### HIGH-1: ACA revision-mode confusion (unchanged)

Default `Single` revision mode is fine for Phase 1; for blue/green move to `Multiple` with explicit `trafficWeights`. Same advice as before.

---

### HIGH-2: Python image bloat balloons self-host registry storage

**What goes wrong:** `python:3.12` is ~120 MB; `python:3.12-slim` is ~50 MB. With 50 builds/dev/week + no retention, registry storage on Azure Files fills up.

**Prevention:**
1. Multi-stage Dockerfiles; final image uses `python:3.12-slim` + only runtime deps.
2. Registry retention: registry garbage-collect job (`registry garbage-collect /etc/docker/registry/config.yml`) runs weekly via ACA Job.
3. Image-tagging discipline: `git-sha` tags immutable; `latest` mutable alias only.
4. Trivy multi-stage scan: only the final stage.

**Phase:** Phase 1 (Dockerfile design + retention job).

---

### HIGH-3: Keycloak realm-export drift between staging and prod

**What goes wrong:** Admins click in the Keycloak admin UI to make changes; those changes don't end up in `realm-export.json` in git. Next pipeline run overwrites the live config with the stale checked-in version → lost role assignments + user accounts.

**Prevention:**
1. **One-way sync only**: pipeline applies `realm-export.json` to Keycloak. Never the reverse.
2. Admin UI access disabled in production (Keycloak `--features-disabled=admin-ui` or restricted via Keycloak role).
3. All realm changes go via PR editing `realm-export.json` + `kcadm.sh` validation in CI.
4. Keycloak users data (passwords etc.) lives in Postgres — exported separately if needed; not in `realm-export.json`.

**Phase:** Phase 1 (Keycloak setup).

---

### HIGH-4: Python startup time + cold starts (FastAPI)

**What goes wrong:** Python FastAPI cold-start is 2-5s (vs Node ~0.5s). Combined with ACA scale-to-zero (which we keep on for static + registry), some endpoints feel slow first-touch.

**Prevention:**
1. Portal and LLM-proxy: `minReplicas: 1`. Cost-acceptable.
2. Pre-warm Python via `uvicorn --workers 1` + lazy-load Anthropic SDK only on first request.
3. Use `--app-dir` + skip auto-reload.
4. For scale-to-zero apps (static + registry): cold-start measured but acceptable (< 3s).

**Phase:** Phase 1.

---

### HIGH-5: ACA Job scheduling reliability for nightly pg_dump

**What goes wrong:** ACA Jobs run on cron — if the cron expression is off (timezone confusion), or if the previous run was still running, the new run skips silently. pg_dump-job not running for a week = no backups.

**Prevention:**
1. Cron expression in UTC, verify with `crontab.guru`.
2. Job sets `replicaTimeout: 1800` (30 min max) and `replicaRetryLimit: 1`.
3. **Job last-run audit row** written to `portal.audit_log` by the job itself; portal dashboard shows "last backup: X hours ago" with red badge if > 36h.
4. Alert rule in Azure Monitor on ACA Job `replicaFailures` > 0.

**Phase:** Phase 1.

---

### HIGH-6: CSP for Claude artifacts (unchanged)

Same as before — Claude artifacts often use inline `<script>`. Sandbox the artifact viewer in an iframe on a distinct subdomain with relaxed CSP, or strip server-side.

**Phase:** Phase 2 (artifact viewer); CSP baseline in Phase 1 for the static-app served via Nginx.

---

## Moderate Pitfalls

### MOD-1: Audit log integrity (unchanged)

Append-only at DB level (revoke `UPDATE`, `DELETE` on app role). Hash-chain rows in Phase 2.

---

### MOD-2: ACA replica idle billing scaled down

**Now-relevant counts:** portal + proxy + Postgres + Keycloak = 4 always-on replicas × ~€3 each = €12 idle compute baseline. Plus scale-to-zero apps. Total ~€15-20 net.

**Prevention:** sizing per ARCHITECTURE.md; resist `minReplicas: 1` for non-stateful apps.

---

### MOD-3: Anthropic prompt-token accounting (unchanged)

Use the full `usage` block from Anthropic responses: `input + output + cache_creation + cache_read` with current pricing.

---

### MOD-4: Anthropic 529 retry storms (unchanged)

Exponential backoff with jitter via `tenacity`. Max 2 retries. Circuit breaker on 5 consecutive 529s.

---

### MOD-5: SP-secret rotation procedure missed

**What goes wrong:** ADO service connection uses a Service Principal client-secret. Default expiry 12 months. Without rotation reminder, deploys fail one day with "AADSTS7000222: The provided client secret keys are expired".

**Prevention:**
1. SP-secret expiry 90 days; calendar reminder + audit-log row "SP-secret rotated by X on Y".
2. Rotation procedure in `docs/runbook.md`: create new secret → update ADO service connection → run dummy pipeline → revoke old secret.
3. Optional: Phase 2 migrate to WIF after first proven Phase 1 run.

**Phase:** Phase 1 (initial SP creation); rotation procedure in Phase 5.

---

## Minor Pitfalls

### MIN-1: Azure Files SMB quota auto-grow

Standard Files default share quota = 5 TiB; provisioned tier is what costs money. Ensure `quota: 100` (GiB) on the share — not 5120.

---

### MIN-2: Keycloak version pin

`quay.io/keycloak/keycloak:26` pulls latest 26.x. Pin to `:26.0.x` exact.

---

### MIN-3: `distribution/distribution:3` config

Default config file uses `inmemory` storage driver. Override with `filesystem` driver pointing to mounted volume — else images vanish on restart.

---

### MIN-4: Python `slowapi` keying

`slowapi` defaults to keying on IP. We want per-user keying. Use a custom `key_func` reading the Keycloak `sub` from the validated JWT.

---

### MIN-5: Postgres connection-limit at low resource sizing

`postgres:16-alpine` default `max_connections = 100`. With portal + 2 proxies (test+prod) + Keycloak + admin sessions, this is tight. Either bump to 200 in postgres.conf or add pgbouncer container. Recommended: pgbouncer container (`edoburu/pgbouncer:latest`) in transaction-pool mode.

**Phase:** Phase 1 (Postgres bootstrap).

---

## Anti-Recommendations (skip)

Same as before. Internal, single-tenant. Skip RLS, multi-region, statuspage, etc.

---

## Phase-Specific Warning Map

| Phase Topic | Critical Pitfalls Active | Mitigation Owner |
|-------------|--------------------------|------------------|
| Phase 1 — Bicep + ACA Env + Storage | CRIT-3 (Azure Files for Postgres), CRIT-7 (LAW ingest) | Dev B |
| Phase 1 — Postgres + pg_dump | CRIT-3, HIGH-5 (job reliability), MIN-5 (connection-limit) | Dev A |
| Phase 1 — Keycloak | CRIT-4 (DB persistence), HIGH-3 (realm drift), MIN-2 (version pin) | Dev A |
| Phase 1 — Self-host registry | CRIT-5 (oauth2-proxy bypass), HIGH-2 (image bloat), MIN-3 (filesystem driver) | Dev B |
| Phase 1 — LLM-proxy + cap | CRIT-1, CRIT-2, MOD-3, MOD-4 | Dev B |
| Phase 1 — Secrets via ACA | CRIT-6 (no rotation tooling) | Dev A |
| Phase 1 — Python apps | HIGH-4 (cold start), MIN-4 (slowapi keying) | Dev A |
| Phase 5 — CI/CD | MOD-5 (SP-secret rotation) | Dev A |
| Phase 2 — Artifact viewer | HIGH-6 (CSP/inline scripts) | TBD |

---

## Recommendations for the Roadmap

1. Treat CRIT-1/CRIT-2/CRIT-3/CRIT-4 as Phase 1 blockers — no production traffic until each is verified.
2. Postgres-on-Azure-Files-Premium decision **before** Phase 1 day 1 — if Standard chosen, plan for slower queries.
3. **Restore drill in DoD** — corrupting the pg-data share and restoring from dump must be exercised at least once.
4. Pre-deploy smoke tests in pipeline for every CRIT — `/healthz/secrets`, registry-anon-401, Keycloak well-known, daily-cap-transaction.
