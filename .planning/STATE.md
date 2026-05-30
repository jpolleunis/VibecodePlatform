# STATE — VibeCoding Platform

## Project Reference

See: `.planning/PROJECT.md` (updated 2026-05-30)

**Core value:** A TAG colleague can use a "vibecoded" internal app via a shareable URL after Microsoft login, without the developer leaking secrets or setting up cloud infra themselves.

**Current focus:** Phase 1 — Foundations + Empty Dashboard

---

## Current Phase

**Phase 1** — Foundations + Empty Dashboard
**Status:** Not started
**Plans:** 0/5 complete
**Blocking:** TAG IT request (Resource Group, Workload Identity Federation, Entra app-registration) — file day 1

---

## Phase Index

| # | Phase | Status | Plans |
|---|-------|--------|-------|
| 1 | Foundations + Empty Dashboard | Not started | 0/5 |
| 2 | Meeting Agent Live behind SSO | Not started | 0/4 |
| 3 | Daily-Cap + Rate-Limit | Not started | 0/3 |
| 4 | Portal Dashboard + Role Admin | Not started | 0/4 |
| 5 | CI/CD + Promote Flow + Audit | Not started | 0/5 |
| 6 | Documentation + DoD Validation | Not started | 0/3 |

---

## Open Decisions / Open Questions

Pending TAG IT input (will become Key Decisions in PROJECT.md):
- GHAS-on-ADO licensing → determines Dependabot vs Renovate (default Renovate).
- Defender for Cloud subscription state → ACR scanning could replace Trivy (default keep Trivy + Defender if free).

Pending verification before Phase 1 start (research recommends a quick re-check):
- Current Anthropic Sonnet 4 pricing (affects daily-cap math).
- Latest AVM `avm/res/app/container-app` version + breaking-change CHANGELOG.
- Auth.js v5 stable release status (any post-v5.0 patches to track).

---

## Last Updated

2026-05-30 — after `/gsd:new-project` initialization
