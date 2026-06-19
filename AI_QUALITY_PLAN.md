# AI_QUALITY_PLAN.md — Kvaliteediväravate plaan (Kalaradar)

See dokument selgitab, milliste strateegiatega hoiab Kalaradari projekt koodikvaliteeti
AI-assistendiga arendades, ja vastab ülesande seitsmele küsimusele. AI reeglite fail ise
on [CLAUDE.md](CLAUDE.md); see plaan põhjendab selle valikuid ja kirjeldab tehnilist tuge,
ilma milleta reeglid üksi katkist koodi main harust eemal ei hoia.

---

## 1. Millise AI reeglite faili valisid ja miks?

Valisin **`CLAUDE.md`** projekti juurkaustas.

- Põhitööriist on **Claude Code**, mis loeb sessiooni alguses automaatselt projekti
  juurest `CLAUDE.md` ja kõik alamkaustade `CLAUDE.md` failid (Kalaradaril on lisaks
  juurfailile ka `apps/pwa/CLAUDE.md` i18n- ja stiilireeglite jaoks). Juhised laaditakse
  konteksti just selle tööriista oodatud asukohast.
- `CLAUDE.md` on **tööriistaülene de-facto standard**: GitHub Copilot coding agent loeb
  samuti `CLAUDE.md` (lisaks `AGENTS.md`, `.github/copilot-instructions.md`, `GEMINI.md`).
  Üks fail katab seega mitut assistenti, mida projektis kasutame (Claude Code + Copilot).
- Teadlik piir: `CLAUDE.md` on **kontekst, mitte tehniline sundmehhanism**. Anthropic
  rõhutab, et fail suunab käitumist, kuid ei blokeeri midagi — päris blokeering tuleb
  CI-st, branch protection'ist ja review'st (vt küsimused 4 ja 7). Seetõttu on reeglite
  fail vaid üks kiht — täiskaitse tekib koos tehniliste väravatega.

## 2. Milliseid 3–5 allikat kasutasid?

1. **Anthropic — Claude Code best practices** (anthropic.com/engineering/claude-code-best-practices):
   `CLAUDE.md` struktuur, konkreetsete käskude lisamine, "context not enforcement" põhimõte.
2. **Claude Code dokumentatsioon — Memory / CLAUDE.md ja Hooks**
   (docs.claude.com/en/docs/claude-code/memory ja .../hooks): faili automaatne laadimine,
   kihistamine kaustatasemetel, hook'id tegevuste päriseks blokeerimiseks.
3. **AGENTS.md spetsifikatsioon** (agents.md): agentidele mõeldud "README" formaat —
   build-, testimis-, stiili- ja turvajuhiste paigutus.
4. **GitHub Copilot coding agent / The GitHub Blog**: millised juhisefailid on
   tööriistaülesed (`AGENTS.md`, `CLAUDE.md`, `copilot-instructions.md`, `GEMINI.md`).
5. **GitHub Docs — Protected branches & required status checks**
   (docs.github.com/.../about-protected-branches): kuidas branch protection + rohelise CI
   nõue tehniliselt takistab katkise koodi jõudmist main'i.

## 3. Millised on projekti kõige suuremad riskid AI-ga arendamisel?

Kalaradar on monorepo (API + PWA + landing + Android admin eraldi repos) päris kasutajate,
maksete ja isikuandmetega. Suurimad riskid:

- **Otsene muutmine serveris** — AI võib `ssh` kaudu serveris faile parandada, mis tekitab
  repo ↔ server lahknevuse. CLAUDE.md keelab selle ("muudatused jõuavad serverisse AINULT
  läbi GitHubi: push → pull").
- **Kahe React-versiooni lõhkumine** — PWA on React 18 (root `overrides`), landing React 19
  (`dedupe`). Kui AI eemaldab override'id või dedupe'i, läheb leht valgeks (duplikaat-React).
- **Service Worker cache lõks** — kui `web-dist/`-i jäävad vanad JS/CSS failid, jäävad
  kasutajad vanale versioonile lõksu. Reegel nõuab `rm -rf` enne kopeerimist.
- **Prod-deploy ilma testimiseta** — AI võib deployda otse prodi. Reegel: staging-first,
  prod ainult kasutaja kinnitusega.
- **Rakendatud migratsiooni muutmine** — checksum-kontroll blokeerib; AI ei tohi vana
  migratsiooni faili muuta, ainult lisada uue (`NNN+1`).
- **Tundlike äriloogikate katkilõhkumine** — cancel-subscription flow (8 review-ringi,
  GDPR-invariandid) ja jagamise (entry_shares) loogika; CLAUDE.md suunab AI enne muutmist
  lugema `docs/CANCEL-FLOW.md` / `docs/SHARING-INVARIANTS.md`.
- **Versioonikonfliktid paralleelsetes PR-ides** — mitu PWA PR-i bump'ivad sama versiooni →
  merge-konflikt; reegel kirjeldab rebase + `npm version patch` töövoo.
- **Saladuste lekkimine** — `.env`, Stripe-võtmed, API-võtmed ei tohi commit'i sattuda.

## 4. Kuidas reeglid aitavad vältida katkise koodi jõudmist main harusse?

- **Issue-driven töövoog** — iga muudatus (ka triviaalne) algab GitHub Issue'ga ja eraldi
  feature/fix-branch'ist. **Otse main'i ei commitita kunagi.**
- **Pull request + squash-merge** — muudatus jõuab main'i ainult PR-i kaudu (`Closes #NR`).
- **CI värav** — `.github/workflows/test.yml` jooksutab API + PWA unit-testid ja
  tüübikontrolli `pull_request → main` ja `push → main` peal, pluss Postgres-integratsiooni-
  testid. Punased testid blokeerivad merge'i.
- **Lokaalne pre-commit kontroll** — enne commit'i buildivad KÕIK appid (mitte ainult
  muudetu), sest jagatud sõltuvused mõjutavad kõiki workspace'e.
- **Staging-first** — main → staging → (kasutaja kinnitus) → prod, nii ei jõua katkine
  build kunagi otse kasutajani.

> Oluline: reegel "ära commiti main'i" on **soovitus**, mille teeb jõustatavaks branch
> protection + required status checks GitHubis (vt küsimus 7).

## 5. Millised käsud peavad enne muudatuse lõpetamist õnnestuma?

Kõik allolevad peavad lõppema 0 veaga, enne kui töö märgitakse valmis:

```bash
cd apps/api && npx tsc --noEmit      # API tüübikontroll
cd apps/api && npx vitest run        # API unit-testid (DB/Redis mockitud)
cd apps/pwa && npx tsc --noEmit      # PWA tüübikontroll
cd apps/pwa && npx vitest run        # PWA testid (NB: `npm test` = watch, ära kasuta)
cd apps/pwa && npm run build         # PWA build
cd apps/landing && npm run build     # Landing build
bash scripts/test-local.sh           # nginx-test: buildib + serveerib + verify-build.sh
```

Lisaks **jagamis-koodi (entry_shares) puudutades:** `npm run test:e2e` (Playwright päris
Postgresi + API vastu). Kui mõni neist ebaõnnestub → **ÄRA COMMITI**, paranda enne.
Need samad käsud jooksevad ka CI-s, nii et lokaalne roheline = CI roheline.

## 6. Kuidas tagad, et vana töötav funktsionaalsus ei lähe katki?

- **TDD / regressioonitestid** — projekt on TDD-stiilis (Red → Green → Refactor). **Bugfix
  algab vigast taastootvast (red) testist samas PR-is**; feature algab oodatud käitumist
  fikseerivast testist. Muudatus ilma testita pole valmis.
- **Teste ei kustutata** põhjuseta — ainult kui need on selgelt aegunud ja põhjus on kirjas.
- **Täielik suite enne valmismärkimist** — kõik testid jooksevad, mitte ainult muudetud osa.
- **Integratsioonitestid** — plpgsql trigger-loogika (hot-ranking, migratsioonid 055–057)
  testitakse päris Postgresi vastu (mock ei jookse plpgsql-i). Just see värav püüaks
  #1209-tüüpi trigger-vea.
- **Subsüsteemi-dokumendid enne muutmist** — kriitiliste flow'de (cancel-subscription,
  jagamine) invariandid on dokumenteeritud ja CLAUDE.md sunnib need enne muutmist läbi
  lugema.
- **Tagasiühilduvad migratsioonid** — SQL idempotentne (`IF NOT EXISTS`), uus kood töötab
  hetkeks vana skeemi vastu.

## 7. Millised reeglid vajavad lisaks tehnilist tuge (CI / branch protection / PR review)?

| Reegel CLAUDE.md-s | Vajalik tehniline tugi |
|---|---|
| "Otse main'i ei commitita" | **Branch protection** main'il (push keelatud, ainult PR) |
| "Punased testid blokeerivad merge'i" | **Required status checks** → `tests / unit tests` peab roheline olema |
| "Iga muudatus läbi PR-i" | **Pull request review** (vajadusel required reviewer) |
| "Enne commit'i jooksevad testid/buildid" | **CI** — `.github/workflows/test.yml` (GitHub Actions) |
| "Ära commiti saladusi" | `.gitignore` (`.env`) + secret scanning / push protection |
| "Ära muuda rakendatud migratsiooni" | **Checksum-kontroll koodis** (`runMigrations.ts`) |
| "Päris blokeering, mitte ainult soovitus" | **Claude Code hooks** kõige kriitilisemateks keeldudeks |

Lühidalt: CLAUDE.md suunab AI käitumist, aga **jõustamine tuleb GitHubist (branch
protection + required CI), code review'st ja koodisisestest kontrollidest.** Maksimaalne
kvaliteet = juhised + testid + CI + branch protection + review + väikesed kontrollitavad
muudatused koosmõjus.

---

## Boonus: kuidas üks AI tehtud muudatus reeglite abil üle kontrolliti

Näide päris töövoost (issue **#1299**, "eemalda 'ahistamine/vihkamine' rida kogukonna-
nõusoleku modaalist"):

1. Loodi Issue #1299 ja branch `fix/1299-remove-harassment-rule`.
2. Muudatus tehti, lokaalselt jooksid testid + build (vt küsimus 5).
3. Avati PR → CI (`tests`) jooksis **rohelisena** PR-i peal
   (run `27566130754`, `pull_request`, ✓ success).
4. Squash-merge main'i → CI jooksis uuesti **rohelisena** push-il
   (run `27566290156`: ✓ unit tests + ✓ integration (postgres)).

See näitab, et reegel "katkine kood ei jõua main'i" on tegelikkuses jõustatud: nii PR kui
ka merge läbisid sama testivärava. Tõendid: [toendid/ci-runs.txt](toendid/ci-runs.txt).

---

## Enesehinnang

Kõige kasulikum reegel oli **"enne commit'i buildi KÕIK appid, mitte ainult muudetu"** koos
kahe React-versiooni hoiatusega — just see ennetab kõige salakavalamat viga, kus PWA-muudatus
lõhub `overrides` kaudu landing'u valgeks ekraaniks, mida testid alati kohe ei püüa. Teine
väga väärtuslik on **TDD-reegel**, mis nõuab bugfix'ile red-regressioonitesti samas PR-is —
nii ei tule vana viga tagasi. Suurim **alles jäänud risk** on inimfaktor: CLAUDE.md on
kontekst, mitte sundmehhanism, nii et kui keegi keelab branch protection'i või deployb
käsitsi serveris, hoiavad reeglid teoorias, aga praktikas mitte. Samuti ei kata unit-testid
kõike — service worker'i cache-lõks ja Cloudflare-kihi käitumine vajavad endiselt käsitsi
deploy-järgset kontrolli. Praegu toetub palju ka kasutaja kinnitusele prod-deploy ees, mis
on tahtlik, aga aeglustab töövoogu. **Järgmises versioonis** lisaksin: (1) Claude Code
hook'i, mis blokeerib `git commit` otse main'is ja `.env`-failide lisamise juba lokaalselt;
(2) eraldselguse, et `scripts/test-local.sh` on KOHUSTUSLIK enne iga frontend-PR-i, mitte
"kui võimalik"; (3) lühikese masinloetava checklist'i PR-template'i, et nii AI kui inimene
kinnitaks väravate läbimist. Kokkuvõttes annab fail + CI + branch protection + staging-first
juba praegu tugeva mitmekihilise kaitse, kus ükski üksik viga ei jõua otse kasutajani.
