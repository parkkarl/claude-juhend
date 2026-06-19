# Tõendus testimisest (Kalaradar)

See dokument tõendab, et vähemalt üks kvaliteedikontroll töötab. Kalaradaril töötavad
korraga **unit-testid (API + PWA)**, **tüübikontroll** ja **GitHub Actions CI**.
Toorväljundid on kaustas [`toendid/`](toendid/).

Kuupäev: 2026-06-19. Repo: `parkkarl/kalaradar` (monorepo, mille kohta [CLAUDE.md](CLAUDE.md) kehtib).

---

## 1. API unit-testid — 647 testi rohelist

Käsk: `cd apps/api && npx vitest run`  ·  toorväljund: [`toendid/api-vitest.txt`](toendid/api-vitest.txt)

```
 Test Files  67 passed (67)
      Tests  647 passed (647)
   Duration  7.19s
```

## 2. PWA unit-testid — 543 testi rohelist

Käsk: `cd apps/pwa && npx vitest run`  ·  toorväljund: [`toendid/pwa-vitest.txt`](toendid/pwa-vitest.txt)

```
 Test Files  65 passed (65)
      Tests  543 passed (543)
   Duration  12.03s
```

**Kokku 1190 testi rohelist (67 + 65 = 132 testfaili).**

## 3. GitHub Actions CI — roheline PR-i ja main'i peal

Workflow: `.github/workflows/test.yml` (jookseb `pull_request → main` ja `push → main`).
Toorväljund: [`toendid/ci-runs.txt`](toendid/ci-runs.txt)

```
completed  success  fix: #1299 ...  tests  main  push          27566290156
completed  success  fix: #1299 ...  tests  fix/1299-...  pull_request  27566130754
completed  success  feat: #1297 ... tests  main  push          27565695037
...
```

Üks konkreetne roheline run (mõlemad jobid läbisid):

```
✓ main tests · 27566290156
JOBS
✓ integration (postgres) in 1m11s
✓ unit tests in 2m14s
```

URL: https://github.com/parkkarl/kalaradar/actions/runs/27566290156

---

## Kuidas taasesitada

```bash
cd kalaradar-mono
npm install --no-audit --no-fund
cd apps/api && npx tsc --noEmit && npx vitest run      # tüübikontroll + 647 testi
cd ../pwa  && npx tsc --noEmit && npx vitest run       # tüübikontroll + 543 testi
# CI seisu vaatamiseks:
gh run list --workflow=test.yml --limit 8
```

Need on samad käsud, mis on CLAUDE.md "Enne commiti kontroll" / "TDD" peatükkides ja mis
jooksevad CI-s — lokaalne roheline tähendab CI roheline.
