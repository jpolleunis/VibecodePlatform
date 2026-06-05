# Bouw een app voor VibeCoding — gids voor "vibecoders"

*Voor TAG-collega's die ideeën hebben en met AI willen vibecoden. Geen software-ingenieur-ervaring vereist.*

---

## TL;DR — in 60 seconden

1. **Start nooit blanco.** Clone het `vibecoding-app-template` repo. Alles dat het platform van je app verwacht (login, health-probes, secrets, logs) zit er al in — jij hoeft alleen je idee toe te voegen.
2. **Gebruik Claude Code, niet Claude in de chat.** Claude Code kan in je repo werken, tests draaien, commits maken. Veel productiever dan "kopieer-plak code uit de chat".
3. **Gebruik `superpowers:brainstorming` vóór je code schrijft.** Zelfs voor kleine ideeën. Het dwingt je om eerst te bepalen *wat* je wil, daarna *hoe*.
4. **Gebruik `/gsd:new-project` voor alles dat meer dan een dag werk is.** Het maakt PROJECT.md, ROADMAP.md en zorgt dat je in fasen werkt i.p.v. eindeloos vibecoden tot het werkt.
5. **Lees deze gids één keer volledig** vóór je begint. Daarna ben je drie uur sneller dan zonder.

---

## 1. Wat is een "platform-app"?

Een app die op VibeCoding draait, is altijd:

- Een **container** (Docker image) die we deployen op Azure Container Apps.
- **Achter Keycloak** (single sign-on); jij krijgt de identiteit van de gebruiker gratis aangeleverd.
- **Stateless** of gebruikmakend van **de gedeelde Postgres-database** (één tabel-schema per app).
- Roept **nooit** AI-providers direct aan — wel via `/api/llm` van het platform (zo werkt de daily-cap).

Het platform is niet: een Heroku-kloon waar je willekeurige binaries op kwakt. Het is een **opinionated** mini-PaaS met een paar harde regels. Als je de regels volgt, krijg je login + hosting + secrets + monitoring + cost-cap gratis.

---

## 2. De drie soorten apps die je kunt bouwen

| Type | Voorbeeld | Stack uit template | Complexiteit |
|------|-----------|---------------------|--------------|
| **Statisch** | HTML + JS dat Claude-artifact-stijl draait | Nginx + jouw `index.html` | XS |
| **Statisch + AI** | Idem maar roept LLM aan voor inhoud | Statisch + `/api/llm` van het platform | S |
| **Full backend** | DB-lookups, formulieren, multi-page | Python FastAPI + Postgres + Jinja2 + HTMX | M |

Begin nooit met "full backend" als statisch volstaat. **YAGNI**.

---

## 3. Workflow van idee tot productie

### Stap 0 — Voor je iets typt
Beantwoord deze drie vragen schriftelijk (in een Markdown-bestand of zelfs een Post-it):
1. Wie gaat dit gebruiken? (rol + ongeveer hoeveel mensen)
2. Welk probleem lost het op vandaag (concreet, niet "betere meetings")
3. Wat ziet de gebruiker als je app werkt? Beschrijf een typische sessie.

Geen antwoord = nog niet bouwen.

### Stap 1 — Brainstorm met Claude Code
In je terminal:
```bash
cd ~/my-vibecoded-app
claude
```
Dan in Claude Code: typ je idee + zeg "*gebruik de superpowers:brainstorming skill*".

Claude stelt je vragen, helpt scope te vinden, schrijft een spec naar `docs/superpowers/specs/`. Forceer jezelf om dit te doen, ook als het idee "duidelijk" lijkt.

### Stap 2 — Plan in fasen met GSD
Als de brainstorm meer dan een halve dag werk lijkt:
```
/gsd:new-project
```
Geeft je `PROJECT.md` (waarom) + `ROADMAP.md` (in welke volgorde) + `REQUIREMENTS.md` (wat moet er staan om "klaar" te zijn). Voor kleinere dingen mag je dit overslaan en direct naar Stap 3.

### Stap 3 — Clone de template
```bash
git clone https://dev.azure.com/tag/vibecoding/_git/app-template my-app
cd my-app
./bootstrap.sh   # maakt nieuwe app-naam, herzet git-history
```
De template levert: Dockerfile, `/healthz` + `/healthz/secrets`, OIDC-header parsing, JSON-logging, `pytest`-skelet, lokale `docker compose` met Keycloak + Postgres erbij.

### Stap 4 — Vibecode (in Claude Code, niet in browser)
- Stel je verandering voor in 1 zin: *"Voeg een formulier toe waarmee user meeting-notes plakt en samenvattingen krijgt"*.
- Claude Code maakt een plan, schrijft tests eerst (TDD via `superpowers:test-driven-development` skill), implementeert, commit.
- Verifieer **altijd** dat tests slagen voor je een PR maakt: `pytest` lokaal.

### Stap 5 — Push naar feature-branch
```bash
git push origin feat/my-feature
```
ADO pipeline draait: lint → test → bouw image → Trivy-scan → deploy naar `my-app-test` Container App. Je krijgt een test-URL.

### Stap 6 — Smoke test in test-omgeving
Open je test-URL, log in via Keycloak, klik door je flow. Iets fout? Terug naar Stap 4.

### Stap 7 — Vraag admin om promote naar prod
Stuur Jasper of de admin een Teams-bericht. Hij klikt "Promote" in het portal. Audit-log onthoudt wie + wanneer.

---

## 4. Platform-contract (wat je app **moet** doen)

Deze regels staan al in de template. Breek ze niet.

| Regel | Waarom |
|-------|--------|
| Luister op de poort uit env-var `PORT` (default 8080) | ACA injecteert dit; vaste poort breekt deploy |
| Bind aan `0.0.0.0`, niet `localhost`/`127.0.0.1` | Container krijgt traffic vanaf buiten anders niet |
| Expose `GET /healthz` → HTTP 200 | ACA readiness-probe; faalt = geen traffic |
| Expose `GET /healthz/secrets` → 200 als alle vereiste secrets gezet zijn, 503 anders | Voorkomt dat een revision live gaat met lege secrets |
| Lees secrets **enkel** uit env-vars (ACA injecteert ze) | Geen hardcoded keys; geen `.env` in repo |
| Log naar stdout in JSON (één event per regel, `structlog` doet dit) | Log Analytics begrijpt JSON; queryen wordt makkelijk |
| Honor SIGTERM (drain 30s) | ACA stuurt SIGTERM bij scale-down; abrupt exit = data-loss |
| Run als non-root user in container (`USER 1000`) | Standaard security-baseline |
| Image < 200 MB (multi-stage Dockerfile, `python:3.12-slim` base) | Sneller deploys, minder Trivy-noise |
| Voor wie-is-gebruiker: lees HTTP-header `x-forwarded-user` of de OIDC-cookie (template doet dit voor je) | Keycloak/oauth2-proxy injecteert dit; nooit zelf token verifiëren tenzij je weet wat je doet |
| LLM-calls **altijd** via `POST /api/llm` van het platform, **nooit** rechtstreeks Anthropic | Anders gaat de daily-cap eraan |
| Database-toegang via connection-string in env-var `DATABASE_URL` | Geen hardcoded credentials, geen eigen DB-server |

---

## 5. Wat je app **niet** mag doen

- **Eigen login bouwen.** Keycloak doet alles. Als je een "Login"-knop hebt, gebruik je Keycloak verkeerd.
- **API-keys in browser-code zetten.** Ooit, voor niks, om welke reden ook. (Dit is de hele reden dat we de proxy hebben.)
- **De gedeelde Postgres rechtstreeks vanuit de browser benaderen.** Browser → jouw FastAPI → Postgres. Drie hops, niet twee.
- **`websockets` of long-polling** zonder eerst te checken (in template-docs) of het ondersteund is. ACA heeft beperkingen.
- **Custom domain** instellen zonder admin. Pilot draait op `*.azurecontainerapps.io`.
- **Multi-megabyte assets in je image bakken.** Zet ze op een aparte Azure Files-share of CDN (vraag admin).

---

## 6. Skills + commands cheat-sheet

| Wanneer | Tool |
|---------|------|
| "Ik weet niet wat ik wil bouwen" | `superpowers:brainstorming` |
| "Idee is helder, ik wil structuur" | `/gsd:new-project` |
| "Ik wil dit kleine ding gewoon doen" | `/gsd:fast` of `/gsd:quick` |
| "Ik wil iteratief plannen + uitvoeren per fase" | `/gsd:plan-phase 1` → `/gsd:execute-phase 1` |
| "Ik snap iets niet in de bestaande code" | `superpowers:systematic-debugging` |
| "Ik wil een test eerst schrijven" | `superpowers:test-driven-development` |
| "Ik wil controleren of mijn werk af is" | `superpowers:verification-before-completion` |
| "Ik wil mijn PR laten reviewen door Claude" | `superpowers:requesting-code-review` |
| "Ik hou van GSD maar wil dat alles vanzelf gaat" | `/gsd:autonomous` (let op: dit doet veel zonder check-in) |
| "Ik wil weten wat de skills nog meer kunnen" | `/superpowers:using-superpowers` (lijst alle skills) |
| "Ik moet stoppen, hoe pak ik morgen weer op?" | `/gsd:pause-work` + morgen `/gsd:resume-work` |
| "Ik wil snel een UI-schets" | `/gsd:sketch` of `/gsd:ui-phase` |

---

## 7. Voorbeelden — drie typische apps

### 7.1 "Meeting Agent" (statisch + AI)
- One-pager HTML.
- JavaScript-knop "Vat samen" → roept `POST /api/llm` met user-input.
- Geen DB, geen state.
- Tijd om te bouwen: 1-2 uur met Claude Code.
- Template: `app-template/examples/static-ai/`.

### 7.2 "Project Status Tracker" (full backend, klein)
- FastAPI + Jinja2 + HTMX.
- Eén tabel `projects (id, name, owner, status, updated_at)`.
- Pagina's: lijst, detail, bewerk.
- Iedereen kan lezen, alleen project-owner of admin kan bewerken (rol uit Keycloak).
- Tijd om te bouwen: 1 dag met Claude Code + GSD.
- Template: `app-template/examples/crud-app/`.

### 7.3 "RFP Helper" (AI + DB + uploads)
- Upload PDF van klant-RFP.
- Roept `/api/llm` aan om vragen te extraheren.
- Bewaart vragen + jouw antwoorden in DB.
- Genereert response-doc.
- Tijd om te bouwen: 2-3 dagen.
- Combineer 7.1 + 7.2 patterns; vraag admin om Azure Files share voor de PDFs.

---

## 8. Wanneer iets niet werkt

| Symptoom | Eerste check |
|----------|-------------|
| App start niet in ACA, wel lokaal | Luistert hij op `$PORT`? Bindt hij aan `0.0.0.0`? `/healthz` geeft 200? |
| 503 op alle endpoints | `/healthz/secrets` faalt — een vereiste env-var is leeg. Vraag admin om secrets te zetten. |
| Login redirect-loop | Redirect-URI mismatch in Keycloak realm. Niet zelf knoeien — admin vragen. |
| LLM-aanroep geeft 429 | Daily-cap of rate-limit gehaald. Wacht of vraag admin om cap te verhogen. |
| Pipeline faalt op Trivy | HIGH CVE in jouw image. Update base-image of voeg `.trivyignore` toe met expiry-date. |
| Image > 200 MB | Multi-stage Dockerfile + `python:3.12-slim` base + `--no-cache-dir` bij `pip install`. |
| "Het werkt op mijn machine" | Test met `docker compose` uit de template — die simuleert ACA correct. |

Vraag voor je 30 minuten verloren bent: Teams-channel `#vibecoding-help` of direct aan Jasper.

---

## 9. Anti-patronen (echt ooit gezien)

- "Ik gebruik geen template, ik begin liever blanco." → 3 uur later bezig met OIDC-tokens. Doe dit niet.
- "Ik kopieer mijn API-key in de HTML, het is toch intern." → Geen.
- "Ik installeer eigen Postgres in mijn container." → Gebruik de gedeelde DB; één Postgres voor de hele platform.
- "Ik laat Claude in de chat code schrijven en kopieer alles." → Gebruik Claude Code. Het is gratis met je TAG-account.
- "Ik commit secrets per ongeluk, ik haal ze er later wel uit." → Ze blijven in git history. Force-push of bel admin.
- "Ik test alleen lokaal en push direct naar prod." → Er is geen "direct naar prod"-pad — admin moet promote. Sluit gat al in test.

---

## 10. Volgende stappen

1. Vraag toegang tot het template-repo via admin.
2. Lees `docs/onboarding.md` voor lokale dev-setup (Python 3.12 + Docker Desktop + Claude Code).
3. Probeer als eerste eens een **statisch + AI**-mini-app (Stap 7.1) — dat is in 1-2 uur klaar en je begrijpt het patroon.
4. Houd `docs/runbook.md` open in een tabblad voor "oh shit"-momenten.

---

*Vragen? `#vibecoding-help` in Teams, of `j.polleunis@tag-team.be`.*
