# CLAUDE.md - Kalaradar Mono Repo

## ⚠️ Arendusreegel

**KÕIK muudatused tehakse ALATI lokaalselt repos, testitakse ja ALLES SIIS deployitakse serverisse.**
**Muudatused lähevad ESMALT staging'ule testimiseks, ALLES SEEJÄREL kasutaja käsuga prod'i.**

- ÄRA KUNAGI muuda faile otse serveris (ssh kaudu sed/echo jms)
- Muudatused jõuavad serverisse AINULT läbi GitHubi (push → pull)
- See kehtib ka nginx.conf, docker-compose.yml ja muude konfiguratsioonifailide kohta
- Kui serveris tehakse mõni muudatus (nt build-artifaktide kustutamine, konfigi parandus), KONTROLLI ALATI kas sama muudatus on vajalik ka lokaalselt repos. Serveris ja repos peavad olema sünkroonis
- Commit message'ite lõppu EI lisata käsitsi "Co-Authored-By" ega muid AI-genereeritud footer'eid (nt "🤖 Generated with Claude Code"). Erand: `gh pr merge --squash` võib automaatselt lisada `Co-authored-by:` trailer'i, kui lokaalse commit author email (`karl-eerik.park@vikk.ee`) erineb GitHub konto primaarsest emailist (`info@kalaradar.ee`) — see on tööriista artefakt, mitte reeglirikkumine. Soovi korral saab seda vältida lokaalselt `git config user.email info@kalaradar.ee` seadistamisega.
- Pärast IGAT deployd serverisse KONTROLLI, et `web-dist/` sisaldab AINULT uusi faile. `deploy.sh` kustutab vanad enne kopeerimist (`rm -rf` + `cp -r`), aga kui deployid käsitsi, tee alati: `rm -rf web-dist/app && mkdir -p web-dist/app && cp -r apps/pwa/dist/* web-dist/app/`. Vanad JS/CSS failid `web-dist/`-s põhjustavad SW cache probleeme kus kasutaja jääb vanale versioonile lõksu

## ⚠️ Issue-driven töövoog (KOHUSTUSLIK)

**Iga muudatus (ka triviaalne) algab GitHub Issue'ga.** Otse main'i ei commitita.

```bash
# 1. Loo GitHub Issue — kirjelda MIS ja MIKS
gh issue create --title "Kirjeldus" --body "..."

# 2. Loo branch (NR = issue number GitHubist)
git checkout -b feature/NR-lühikirjeldus    # või fix/NR-...

# 3. Tee muudatused lokaalselt, builda ja testi (vt "Enne commiti kontroll")
# 4. Kui PWA/Landing muutus, testi lokaalselt nginx'iga:
bash scripts/test-local.sh

# 5. Commit + push branch
git add -A && git commit -m "fix: #NR lühikirjeldus"
git push -u origin fix/NR-lühikirjeldus

# 6. Loo PR
gh pr create --title "fix: #NR lühikirjeldus" --body "Closes #NR"

# 7. Merge PR main'i
gh pr merge --squash

# 8. Deploy STAGING'ule (ALATI enne prod'i!)
ssh hetzner "cd /opt/kalaradar && bash deploy-staging.sh main"

# 9. Testi staging'us (vt "Deploy-järgne kontroll")
# 10. KASUTAJA KINNITAB → alles siis deploy PROD'i:
ssh hetzner "cd /opt/kalaradar && bash deploy.sh"

# 11. Pärast prod deployd kontrolli mõlemat lehte (vt "Deploy-järgne kontroll")
```

**⚠️ OLULINE: Prod'i EI deployita ilma kasutaja kinnituseta! Pärast staging deploy'd küsi alati: "Staging on uuendatud, kontrolli ja kinnita, et deployime prod'i."**

**Issue formaat:** issue body PEAB järgima template'e `.github/ISSUE_TEMPLATE/` kaustas (`feature.md` = User Story + Acceptance Criteria Given/When/Then; `bug.md` = lisaks Taastamise sammud). Loe template enne `gh issue create` koostamist.

**Branch nimetamine:**
- Feature: `feature/123-lühikirjeldus`
- Bug fix / refactoring / stiil / konfig: `fix/123-lühikirjeldus`

**Commit/PR pealkiri:** `fix: #123 lühikirjeldus` või `feat: #123 lühikirjeldus` (eesti keeles; squash-merge teeb sellest main'i commit-pealkirja). PR body: `Closes #123` + lühike selgitus.

## ⚠️ Versioonihaldus (KOHUSTUSLIK enne igat commitit)

PWA versioon on `apps/pwa/package.json` `version` väljal (semver: `MAJOR.MINOR.PATCH`).
Vite süstib selle build-ajal `__APP_VERSION__` kaudu äppi, kasutaja näeb seadetes.

**Teiste workspace'ide versioonid ei muutu** — `apps/landing/package.json` (`1.0.0`) ja `apps/api/package.json` (`1.0.x`) on staatilised; landing'i ja API muudatused EI vaja versiooni tõusu. Ainult `apps/pwa/package.json` `version` on canonical ja jõuab kasutajani. Admin äpp on eraldi repo (`parkkarl/kalaradar-admin`), tal on oma Gradle `versionName` — selle monorepo CLAUDE.md reeglid ei kehti admin äpile.

**Enne IGAT commitit otsusta, kas version peab muutuma:**

- **MINOR tõus** (nt 2.1.0 → 2.2.0, PATCH nullitakse): uus kasutajale nähtav feature, uus leht/modaal, uus kalaliik, oluline UI muudatus
- **PATCH tõus** (nt 2.1.0 → 2.1.1): bug fix, stiiliparandus, teksti muudatus, konfigi muudatus, refactoring
- **Ei muutu**: ainult CLAUDE.md, README, dev-tööriistad, testid — midagi mis ei jõua kasutajani

Kui ühe sessiooni jooksul tehakse mitu commitit, tõsta versiooni AINULT esimeses commitist. Järgnevad sama sessiooni commitid ei pea uuesti tõstma.

### Paralleelsed PWA PR-id (#949)

Reegel "sama sessioon, ei pea tõstma" kehtib **commit'ide** kohta ühe branchi sees, **mitte PR-ide** kohta. Mitme paralleelse PWA PR-i puhul (näiteks #943, #944, #945 — kõik bumb'isid 2.51.24 → 2.51.25) tekib merge-time conflict.

**Töövoog:**
1. Esimene PR merge'b → main on `2.51.25`
2. Iga järgnev avatud PWA PR PEAB enne merge'i:
   - `git fetch origin && git rebase origin/main`
   - `cd apps/pwa && npm version patch --no-git-tag-version` (2.51.25 → 2.51.26)
   - `git add apps/pwa/package.json apps/pwa/package-lock.json package-lock.json && git commit --amend --no-edit`
   - `git push --force-with-lease`

**Kiir-audit avatud PR-ide versioonide kohta:** `bash scripts/audit-pr-versions.sh` (topelt-versioone näitab käsitsi visuaalne ülevaade).

## ⚠️ Enne commiti kontroll (KOHUSTUSLIK)

**Enne IGAT commitimist ja deploymist tuleb lokaalselt kontrollida, et KÕIK appid buildivad ja töötavad — mitte ainult see, mida muutsid!**

Isegi kui muutsid ainult üht appi (nt PWA), võivad muudatused kaudselt mõjutada teisi (nt uued npm paketid mõjutavad shared sõltuvusi, root `overrides` mõjutab kõiki workspace'e).

```bash
# 1. API TypeScript kontroll
cd apps/api && npx tsc --noEmit

# 2. PWA build
cd apps/pwa && npm run build

# 3. Landing build
cd apps/landing && npm run build

# 4. Lokaalne nginx test (buildib + serveerib + jooksutab scripts/verify-build.sh kontrolli)
bash scripts/test-local.sh
```

**Minimaalne kontroll kui `test-local.sh` pole võimalik:**
```bash
# Builda mõlemad frontendid ja kontrolli et 0 errorit
cd apps/pwa && npm run build && cd ../landing && npm run build
```

Kui mõni build ebaõnnestub → ÄRA COMMITI. Paranda enne.
(Mida `verify-build.sh` / `test-local.sh` täpselt kontrollivad: [docs/OPERATIONS.md](docs/OPERATIONS.md).)

## ⚠️ TDD / Testimine (KOHUSTUSLIK)

**See projekt on TDD-stiilis.** Iga uus feature või bugfix sisaldab testi **samas PR-is** — muudatus ilma testita pole valmis. Bugfix algab vigast taastootvast (red) testist; feature algab oodatud käitumist fikseerivast testist. Red → Green → Refactor.

**Eelista puhaste helperite eraldamist** (loogika `utils/`-i) — siis saab seda unit-testida ilma raske mockimiseta (nt `apps/api/src/utils/detectCountry.ts` resolverid).

**Testide jooksutamine:**
```bash
cd apps/api && npx vitest run    # API — DB/Redis mockitud, teenuseid pole vaja
cd apps/pwa && npx vitest run    # PWA — ⚠️ `npm test` on WATCH-režiim, kasuta `vitest run`
npx vitest run src/utils/hotRank.test.ts   # üksik fail (suhteline path workspace'i seest)
npm run test:e2e                 # juurest: Playwright integratsioonitestid (e2e/) päris
                                 # Postgres+API vastu — jooksuta kui muudad sharing-koodi;
                                 # setup: npm run test:e2e:setup, juhend: e2e/README.md
```

**Mustrid** (kopeeri neist): puhas loogika → `apps/api/src/__tests__/detectCountry.test.ts`; API route (supertest + mockitud DB) → `apps/api/src/__tests__/auth.routes.test.ts`; React komponent (testing-library + mockitud `react-i18next`) → `apps/pwa/src/components/CountrySwitcher.test.tsx`.

**CI:** `.github/workflows/test.yml` jooksutab mõlemad suite'd `pull_request → main` ja `push → main` peal (merge / squash-merge). Punased testid blokeerivad merge'i AINULT kui main'il on branch protection "Require status checks → `tests / unit tests`".

📖 Täisjuhend: [docs/TESTING.md](docs/TESTING.md)

## ⚠️ Andmebaasi migratsioonid

**Migratsioonifailid:** `apps/api/src/migrations/NNN_kirjeldus.sql`

**Nimetamine:**
- Prefiks on 3-kohaline number: `000`, `001`, `002`, ...
- Järgmine number = viimane olemasolev + 1 (ÄRA jäta numbreid vahele)
- Fail algab kommentaariga: `-- #ISSUE_NR: kirjeldus`

**Käsud:**
```bash
# Lokaalselt (dev DB):
cd apps/api && npm run db:migrate

# Eelvaade (mis jookseks):
cd apps/api && npm run db:migrate:dry

# Prodis (käsitsi, kui deploy.sh ei jooksnud):
ssh hetzner "docker exec kalaradar-api bun run src/scripts/runMigrations.ts"
```

**Turvareeglid:**
- ÄRA KUNAGI muuda juba rakendatud migratsiooni faili (checksum kontroll blokeerib)
- Iga migratsioon jookseb transaktsionis — vea korral rollback
- Advisory lock väldib samaaegset jooksutamist
- `deploy.sh` jooksutab migratsioonid automaatselt PÄRAST API konteineri rebuildi (`docker exec` uues konteineris) — uus kood võib hetke jooksta vana skeemi vastu, kirjuta migratsioonid ja kood tagasiühilduvalt
- Kirjuta SQL idempotentselt (`IF NOT EXISTS`, `DROP ... IF EXISTS` enne `ADD`)

## ⚠️ Deploy-järgne kontroll (KOHUSTUSLIK)

### Staging deploy kontroll

**Pärast staging deploy'd kontrolli:**
1. `https://staging.kalaradar.ee/app/` — PWA peab avanema
2. `https://staging-api.kalaradar.ee/health` — API peab olema healthy

**Kiirkontroll (serverist, Cloudflare bypass):**
```bash
ssh hetzner "curl -s http://localhost:3002/health"
ssh hetzner "curl -s -o /dev/null -w '%{http_code}' http://localhost:8083/app/"
```

**Pärast staging kontrolli küsi kasutajalt kinnitust enne prod deploy'd!**

### Prod deploy kontroll

**Pärast IGA prod deployd kontrolli MÕLEMAT lehte:**
1. `https://kalaradar.ee/` (landing) — peab avanema, mitte valge ekraan
2. `https://kalaradar.ee/app/` (PWA) — peab avanema, mitte valge ekraan

Kui leht on valge → JS runtime error. Kontrolli brauseri konsooli või API logisid.

**Kiirkontroll serverist:**
```bash
ssh hetzner "curl -s http://localhost:8082/ | head -3"        # Landing HTML
ssh hetzner "curl -s http://localhost:8082/app/ | head -3"    # PWA HTML
ssh hetzner "curl -s http://localhost:3000/health"            # API
```

## ⚠️ React versioonide reeglid (TÄHTIS)

Monorepos on **kaks erinevat React versiooni**:
- **PWA (`apps/pwa/`)** — React **18** (pinitud root `package.json` `overrides` kaudu)
- **Landing (`apps/landing/`)** — React **19**

**Reeglid:**
- Root `package.json` `overrides` sunnib React 18 PWA sõltuvustele. ÄRA eemalda neid override'e, muidu PWA läheb katki
- Landing `vite.config.ts` sisaldab `resolve.dedupe: ['react', 'react-dom']` — see tagab et landing buildib ainult ühe React'i koopia. ÄRA eemalda seda, muidu landing läheb katki (valge ekraan duplikaat React'i tõttu)
- Kui lisad uue npm paketi mis sõltub React'ist, kontrolli ALATI et mõlemad appid buildivad ja töötavad (vt "Enne commiti kontroll")

## Projekt

Kalaradar - kalastusprognoosi rakenduse monorepo. Sisaldab kõiki rakendusi ja teenuseid.

**Struktuur:**
- `apps/api/` - Node.js backend API
  - `src/routes/` - REST endpointid; `routes/admin/` = Android admin äpi API
  - `src/jobs/` - node-cron taustatööd (lukud: `jobs/locks.ts`)
  - `src/services/` - storageService (R2/lokaalne failihoidla), emailService (Resend), redis, webhookQueueService
  - `src/utils/` - puhtad helperid (unit-testitavad)
- `apps/pwa/` - Progressive Web App (React 18, Vite) — vt ka `apps/pwa/CLAUDE.md` (i18n, country-data, stiilireeglid)
- `apps/landing/` - Koduleht (React 19)
- `apps/admin/` - ⚠️ legacy prebuilt dist-artifakt (ainult `dist/`, source puudub, nginx EI serveeri) — päris admin on eraldi Android repo (vt "Admin äpp")

**Env muutujad:**
- API kanooniline loend: `apps/api/.env.example` (dev väärtused: `dev/.env.dev`)
- PWA build-aja muutujad (template: `apps/pwa/.env.example`): `VITE_API_DEV_URL`, `VITE_API_PROD_URL`, `VITE_API_TIMEOUT`, `VITE_STRIPE_PUBLISHABLE_KEY`, `VITE_TILE_URL`, `VITE_TILE_ATTRIBUTION`

## Arenduskäsud

```bash
npm run api:dev       # API dev server
npm run pwa:dev       # PWA dev server
npm run landing:dev   # Landing dev server
npm run build:all     # Ehita kõik
```

## Suuremad subsüsteemid (loe ENNE muutmist)

**Social feed (#1057):** `apps/api/src/routes/social.ts` + `routes/admin/social.ts` (Android admin moderatsioon), `apps/pwa/src/pages/SocialFeedPage.tsx`, `apps/api/src/utils/hotRank.ts`; migratsioonid 046–056. Hot-ranking: `hot_score`/`rank_key` uuenevad engagement-deltadega (like +1, kommentaar +3, unlike −1, kustutus −3; half-life 48h), sort=hot|new (default hot).

**Cancel-subscription email flow:** läbinud 8 review ringi. ⚠️ ENNE kui muudad `stripe.ts` cancel-handler'eid (webhook dispatch, `handleSubscriptionUpdate`, `handleSubscriptionDeleted`), `sendCancelAdminEmail.ts`, `cancelReasonLookup.ts`, `cancelSubscriptionBody.ts` või `notificationSettingsStore.ts` → loe [docs/CANCEL-FLOW.md](docs/CANCEL-FLOW.md) (event taxonomy, email dedup, webhook idempotency, GDPR invariandid).

**Jagamine (entry_shares):** ⚠️ ENNE kui muudad `sharedLogs.ts`, `calendarShares.ts`, `friends.ts` või `shareProjection.ts` → loe [docs/SHARING-INVARIANTS.md](docs/SHARING-INVARIANTS.md) (re-share read_at, unfriend kaskaad, wire-format, M10 location-resolutsioon — kanooniline definitsioon ka `shareProjection.ts` header'is).

## Production server (Hetzner)

> **Süsteemitasandi config** (mida Docker Compose ei halda — journald cap, fail2ban): vt [docs/SERVER-PROVISIONING.md](docs/SERVER-PROVISIONING.md). Kanoonilised failid `infra/server/`-is, rakendatakse `git pull` + dokumenteeritud käskudega — ÄRA muuda `/etc/...` otse serveris.

Ühendus: `ssh hetzner` (alias ~/.ssh/config-is). Repo serveris: `/opt/kalaradar/`.

**⚠️ API deploy: ALATI `npm run build` ENNE `docker compose up -d --build api`!**
Docker konteiner kasutab kompileeritud `dist/` kausta. Kui TypeScripti ei kompileeri, deployitakse VANA kood, isegi kui `src/` failid on uued. `deploy.sh` teeb seda automaatselt, aga käsitsi API deployl PEAB alati enne buildima.

```bash
# Täielik prod deploy (testid + buildid + konteinerid + migratsioonid + health-check):
ssh hetzner "cd /opt/kalaradar && bash deploy.sh"

# Ainult API rebuild + restart (ALATI builda TS enne Docker rebuildi!):
ssh hetzner "cd /opt/kalaradar/apps/api && npm run build && cd /opt/kalaradar && docker compose up -d --build api"

# Logid ja DB:
ssh hetzner "docker logs kalaradar-api --tail 50"
ssh hetzner "docker exec -it kalaradar-db psql -U kalamees_user -d kalamees"
```

📖 Täielik runbook (serveri struktuur, konteinerite tabelid, kõik deploy/restart/logi/DB käsud, dev-keskkonna tabelid): [docs/OPERATIONS.md](docs/OPERATIONS.md)

## Lokaalne arendus

### Eeldused
- **podman** (konteinerid) - juba installitud
- **Node.js** + **npm** - API ja frontendide jooksutamiseks

### Käivitamine

```bash
# 1. Käivita dev DB + Redis (ainult esimesel korral või pärast reset'i)
cd dev && bash start.sh    # Postgres port 5433, Redis port 6380 (dev/ on gitignored)

# 2. Käivita API (eraldi terminalis)
cd apps/api && npm run dev    # tsx watch, port 3000

# 3. Käivita PWA (eraldi terminalis, vajadusel)
cd apps/pwa && npm run dev    # vite, port 3000 (kui API hõivab 3000, valib vite järgmise vaba pordi — vaata terminali väljundit)
```

📖 dev/ failide tabel, pordid, admin API key, Android emulaatori IP-d: [docs/OPERATIONS.md](docs/OPERATIONS.md)

## Staging server

**Staging-first reegel:** kõik muudatused lähevad ESMALT staging'ule, prod alles kasutaja kinnitusega (vt "Issue-driven töövoog").

- **Staging PWA:** https://staging.kalaradar.ee/app/ | **Staging API:** https://staging-api.kalaradar.ee (health: `/health`)
- Staging serveerib AINULT PWA-d (landing'ut staging'us testida EI saa — kasuta `bash scripts/test-local.sh`)
- Staging DB on eraldi (seed data, mitte prod koopia); Stripe **test** keys; PWA build-ajal `VITE_API_PROD_URL` override

```bash
# Deploy staging'ule (main või konkreetne branch):
ssh hetzner "cd /opt/kalaradar && bash deploy-staging.sh main"
```

📖 Staging konteinerid, logid, restart, DB-reset, konfiguratsioonifailid: [docs/OPERATIONS.md](docs/OPERATIONS.md)

## Seotud projektid

- **Production API:** https://api.kalaradar.ee
- **Production PWA:** https://kalaradar.ee

### Admin äpp (eraldi repo)

- **Repo:** `/home/park/kalaradar-admin-android/` (GitHub: `parkkarl/kalaradar-admin`)
- **Branch:** `main` (Kotlin/Jetpack Compose Android äpp). NB: `master` branch on vana Expo versioon, MITTE kasutada
- **Tehnoloogia:** Kotlin, Jetpack Compose, Hilt DI, Retrofit + Moshi, Material 3
- **Build:** `JAVA_HOME=/home/park/java/jdk-21.0.6 ./gradlew assembleDebug` (JDK 25 ei ühildu Kotlin compileriga, kasuta JDK 21)
- **API:** kasutab selle monorepo API-t (`apps/api/src/routes/admin/`)
- **Seos:** admin äpi Kotlin mudelid peavad vastama API response'idele. Kui muudad admin API endpointi, kontrolli ka admin äpi mudeleid (`data/model/`) ja vastupidi

## Branding

- **Nimi:** Kalaradar (mitte "Kalamees")
- **Domeen:** kalaradar.ee
- **API:** api.kalaradar.ee
- **Tehnilised ID-d:** kalamees (tagasiühilduvuse jaoks)
