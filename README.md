# CloudTech HR GRC workspace

A working governance, risk and compliance (GRC) app for a fictional 50-person SaaS company that stores customer PII in AWS. It turns the five portfolio deliverables into one tool you can use: risk register, control testing, issue tracker, policy exceptions, audit evidence and a leadership dashboard.

> CloudTech HR is fictional. All names, figures, vendors and findings are invented. Framework references (ISO 27001:2022, NIST CSF 2.0, CIS Controls v8, NIST SP 800-53 Rev 5) are real; the mappings between them are analyst judgement.

Live version (claude.ai, shared workspace): https://claude.ai/artifact/28Jw6eqeSjJeyH7oxT4iwS

## Quick start

The whole app is one file: `dist/index.html`. No install, no server, no build step needed to run it.

1. Open `dist/index.html` in a browser (double-click it).
2. The sample data loads on first run and is saved in that browser.

To serve it locally instead:

```bash
python3 -m http.server 8080 --directory dist
# then open http://localhost:8080
```

## Deploy

It is a static page, so any static host works.

| Host | How |
|---|---|
| GitHub Pages | Push this folder to a GitHub repo, then Settings > Pages > Source: GitHub Actions. The included workflow (`.github/workflows/pages.yml`) runs the tests and publishes `dist/`. |
| Netlify / Cloudflare Pages | Drag the `dist` folder onto the dashboard, or connect the repo with publish directory `dist` (see `netlify.toml`). |
| AWS S3 static website | `aws s3 cp dist/index.html s3://YOUR-BUCKET/index.html --content-type text/html` and enable static website hosting. Put CloudFront in front for HTTPS. |
| claude.ai artifact | Publish `dist/index.html` as an artifact with the `db`, `user` (scope `profile`) and `downloads` capabilities. Data is then shared live between everyone you share it with. |

## Where the data lives

The app picks its storage automatically.

| Where it runs | Storage | Who sees changes |
|---|---|---|
| claude.ai published artifact | The artifact's shared database | Everyone with access, live. Viewers with view-only access cannot edit. |
| Anywhere else (file, GitHub Pages, S3...) | `localStorage` in that browser | Only that browser on that device |

Use **Settings > Export backup (JSON)** to keep a copy or move data between browsers, and **Import backup** to load it. **Restore sample data** puts the original CloudTech data back.

Browser mode is fine for a demo or a single analyst. For a real team on a static host you would add a backend (for example a small API with a database behind your SSO). `src/store.js` is the only file that would change.

## What the app enforces

Everything below is worked out by `src/logic.js` and covered by tests.

| Rule | Source | Behaviour |
|---|---|---|
| Rating bands 16-25 Critical, 10-15 High, 5-9 Medium, 1-4 Low | POL-RISK-001 | Scores and ratings are always calculated, never typed |
| Residual cannot exceed inherent | POL-RISK-001 | Save is blocked |
| Critical residual risk cannot be accepted | Risk appetite | Save is blocked |
| Only the CEO accepts a High residual risk, for at most 6 months | Risk appetite | Save is blocked |
| Every accepted risk needs an acceptance record or exception reference | POL-RISK-001 | Save is blocked |
| An issue closes only with closure evidence, a verifier and a date | Issue management | Save is blocked |
| Exceptions last at most 6 months | POL-EXC-001 | Save is blocked; status shows Expiring soon (30 days) and Expired |
| Head of Security approves every exception; the CEO also approves High residual risk and MFA or encryption gaps on customer-PII systems | POL-EXC-001 | Save is blocked |
| Leaver access removed within 24 hours; an extension counts only if approved before termination, up to 5 days | POL-IAM-001 s6.2, s6.4 | Lag hours, SLA check and result are calculated |
| A control passes only with adequate design and no unexplained deviations | Testing method | Result is calculated from the test you record |
| Overdue is derived from the due date and the as-at date | All registers | Change the as-at date in Settings to view any day |

Every change is stored in the record's own history (who, when, old value, new value; last 40 changes per record) and shown in **Activity**.

## Project layout

```text
cloudtech-grc-app/
├── dist/index.html          the built, self-contained app (this is what you deploy)
├── src/
│   ├── index.html           page template
│   ├── styles.css           design tokens, light and dark themes
│   ├── logic.js             rules engine: pure functions, no DOM (also used by tests)
│   ├── store.js             storage: shared database on claude.ai, localStorage elsewhere
│   └── app.js               user interface: routing, views, forms, heat map
├── seed/seed.json           sample data (107 records)
├── tools/
│   ├── portfolio_data.py    the portfolio's source data
│   └── build_seed.py        turns portfolio data into seed/seed.json
├── tests/
│   ├── logic.test.js        unit tests for the rules engine (Node)
│   └── e2e.py               browser tests (Playwright + Chromium)
├── build.py                 inlines CSS, JS and seed data into dist/index.html
├── package.json             npm scripts: build, test, test:e2e, serve
├── netlify.toml             Netlify config
└── .github/workflows/pages.yml   GitHub Pages: test, build, deploy
```

## Build and test

Requirements: Python 3.9+ and Node 18+. Browser tests also need Playwright.

```bash
python3 tools/build_seed.py      # only if you changed the sample data
python3 build.py                 # writes dist/index.html
node --test tests/logic.test.js  # 14 unit tests

pip install playwright && python3 -m playwright install chromium
python3 tests/e2e.py             # 46 browser checks, incl. shared-database and view-only modes
```

Or with npm: `npm run build`, `npm test`, `npm run test:e2e`, `npm run serve`.

The unit tests check that the app reproduces the portfolio's numbers exactly (41 open technical issues, 7 overdue, 6 of 10 controls passing, the REQ-07 leaver results). The browser tests drive the real page: blocked and allowed risk acceptances, issue closure, the 6-month exception cap, editing leaver timestamps, recording a control test, leadership decisions, persistence after reload, CSV export columns, restore, phone layout and dark mode. The shared-database mode is tested against an in-memory stand-in for `window.claude`.

## Using it for a real company

1. Replace the records in `tools/portfolio_data.py` (or edit `seed/seed.json` directly), or start from the sample and edit inside the app.
2. Update people and business units in the `meta/settings` record (`people`, `businessUnits`).
3. If your policies differ, change the constants in `src/logic.js` (`rating`, `EXC_MAX_MONTHS`, `SLA_HOURS`) and the matching tests.
4. Rebuild and redeploy.

Do not put real customer PII into a browser-mode deployment on a shared computer: localStorage is not encrypted and anyone using that browser profile can read it.

## Known limits

- Browser mode keeps data on one device only. Export a backup before clearing browser data.
- In shared mode, writes are last-writer-wins. If someone else saves a record you are editing, the form shows a banner and lets you load the latest version.
- Each record keeps its last 40 changes.
- The page loads the Public Sans font from Google Fonts; without internet access it falls back to the system font.
