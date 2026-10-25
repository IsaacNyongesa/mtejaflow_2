# MtejaFlow

Customer, quotation, follow-up and payment tracking for small service businesses (electricians, plumbers, appliance repairers, cleaners and similar independent professionals). *Mteja* is Swahili for "customer".

A real full-stack application: **React + Vite + TypeScript + Tailwind** frontend, **Node + Express + TypeScript** API, **PostgreSQL** with **Prisma**, Zod validation, HTTP-only cookie sessions.

## What it does

| Area | Capabilities |
|---|---|
| Accounts | Register, log in/out, persistent session, business onboarding (name, category, phone, email, currency, language, timezone), settings |
| Customers | Add, edit, view, search (name or phone), filter, sort, archive/restore, full history page |
| Enquiries | Six-stage pipeline (New, Contacted, Quotation Sent, Follow-up Required, Confirmed, Lost) as a board (drag and drop, or a status menu) and as a table |
| Quotations | Line items, discount, server-side decimal totals, unique numbers, validity dates, status, preview/print/download **PDF**, create a job from an accepted quotation |
| Follow-ups | Schedule with date, time, purpose and notes; today / overdue / upcoming / completed; complete, reopen, reschedule |
| Message assistant | Editable WhatsApp drafts (5 kinds, English and Kiswahili) with a copy button and a `wa.me` link |
| Jobs | Table and detail views, filters, statuses, technician, linked to quotations/enquiries |
| Payments | Manual records (cash, bank transfer, M-Pesa, other), balances, overpayment protection, void with reason |
| Dashboard and reports | Live figures from the database; date-range reports with charts |
| Billing structure | Free Trial / Starter / Pro with enforced limits; **no checkout** |

### What it deliberately does **not** do

These are limits, not bugs. The interface says so wherever it matters.

- **It never sends anything.** No WhatsApp, SMS or e-mail is sent. The message assistant opens WhatsApp with a draft; the user presses Send. Follow-up reminders appear inside the app only.
- **No M-Pesa or bank integration.** Payments are manual records. M-Pesa codes are stored as typed and are never verified. Every payment is flagged `verified: false`.
- **No subscription payments.** The plans and limits exist, but nothing can be purchased or switched in the app. The prices (KSh 500 and KSh 1,000 per month) are *proposed launch prices*, not validated market rates. See `docs/BILLING_INTEGRATION.md`.
- **No password reset or e-mail verification.** Not implemented yet; see `docs/SECURITY.md`.
- Kiswahili: message templates are complete; the interface translation is a **partial starter set**. Neither has been reviewed by a native speaker.

---

## Requirements

| Software | Version | Notes |
|---|---|---|
| Node.js | 20 or newer (developed and tested on 22) | includes npm |
| PostgreSQL | 14 or newer (developed and tested on 16) | local install, or Docker |
| Git | any | optional |

### Windows PowerShell: install the tools

```powershell
winget install OpenJS.NodeJS.LTS
winget install PostgreSQL.PostgreSQL.16      # or use Docker, below
node --version ; npm --version
```

Close and reopen PowerShell after installing so `node` is on your PATH.

If PowerShell says *"npm.ps1 cannot be loaded because running scripts is disabled"*, run once:

```powershell
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
```

## Quick start (Windows PowerShell)

All commands run from the project folder (the one containing this README).

### 1. Install dependencies

```powershell
npm run install:all
```

### 2. Create the database

**Option A, PostgreSQL installed locally** (use the password you chose during installation):

```powershell
psql -U postgres -c "CREATE USER mtejaflow WITH PASSWORD 'choose-a-password' CREATEDB;"
psql -U postgres -c "CREATE DATABASE mtejaflow OWNER mtejaflow;"
psql -U postgres -c "CREATE DATABASE mtejaflow_test OWNER mtejaflow;"
```

If `psql` is not found, add `C:\Program Files\PostgreSQL\16\bin` to your PATH, or use pgAdmin to run the same SQL.

**Option B, Docker** (needs Docker Desktop):

```powershell
Copy-Item .env.example .env          # then edit .env: set POSTGRES_PASSWORD and JWT_SECRET
docker compose -f docker-compose.yml -f docker-compose.dev-db.yml up -d db
```

The `dev-db` add-on publishes the database on `127.0.0.1:5432`; if that port is already used on your PC, set `$env:DB_PORT = "5433"` first and use 5433 in `DATABASE_URL`. The compose file creates the `mtejaflow` database. Create the test database once:

```powershell
docker compose exec db psql -U mtejaflow -c "CREATE DATABASE mtejaflow_test;"
```

### 3. Configure the API

```powershell
Copy-Item backend\.env.example backend\.env
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"    # copy the output
notepad backend\.env
```

In `backend\.env` set `DATABASE_URL`, `TEST_DATABASE_URL` (user, password and database names from step 2) and `JWT_SECRET` (the value you just generated). Never commit this file; it is already in `.gitignore`.

### 4. Generate the Prisma client and create the tables

```powershell
npm run db:generate
npm run db:migrate
```

### 5. (Optional) Load fictional demo data

```powershell
npm run db:seed
```

Creates **demo@mtejaflow.test** with the password `DemoPass-12345` (override with `$env:SEED_DEMO_PASSWORD`), eight clearly fictional customers and linked enquiries, quotations, jobs, payments and follow-ups. Refuses to run in production. Re-create it with `npm run db:seed -- --reset`.

### 6. Start the app

```powershell
npm run dev
```

Open **http://localhost:5173**. The API runs on port 4000 and the dev server forwards `/api` to it, so the browser only ever sees one address (no CORS or cookie problems).

To run the two parts in separate terminals instead: `npm --prefix backend run dev` and `npm --prefix frontend run dev`.

### 7. Try the full workflow

Register, finish business setup, add a customer, record an enquiry with a follow-up, create a quotation and download its PDF, mark it accepted, create a job, record a payment (see the outstanding balance change), open the message assistant, then look at the dashboard and Reports.

---

## Quick start with Docker only (Windows PowerShell)

No Node.js or PostgreSQL install needed, just Docker Desktop (running). Run from the project folder.

```powershell
# 1. Create your settings file with fresh random secrets
function New-Secret($bytes) { $b = New-Object byte[] $bytes; [Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($b); -join ($b | ForEach-Object { $_.ToString("x2") }) }
Copy-Item .env.example .env
(Get-Content .env) -replace 'REPLACE_WITH_A_STRONG_PASSWORD', (New-Secret 24) -replace 'REPLACE_WITH_LONG_RANDOM_SECRET', (New-Secret 48) | Set-Content .env
Get-Content .env                      # check: no REPLACE_WITH_... left

# 2. Build and start (the first build takes a few minutes)
docker compose up --build -d
docker compose ps                     # both services should say "running" / "healthy"

# 3. Check it is alive, then open the app
Invoke-RestMethod http://localhost:8080/api/health
Start-Process http://localhost:8080

# 4. (Optional) load fictional demo data: demo@mtejaflow.test / DemoPass-12345
docker compose exec -e ALLOW_PRODUCTION_SEED=true app node dist/scripts/seed.js
```

Useful afterwards: `docker compose logs -f app` (live logs), `docker compose down` (stop, keeps your data), `docker compose down -v` (stop **and delete all data**), `docker compose up -d` (start again). If you edit `.env` by hand, keep `POSTGRES_PASSWORD` to letters and digits only; symbols such as `@` or `#` break the database address. The database password cannot be changed later without `docker compose down -v`.

## Put the project on GitHub

```powershell
git init -b main
git add .
git status                            # check: no ".env" listed (only *.example files are expected)
git commit -m "Initial commit"
```

Then either create the repository on github.com (New repository, **private** is a good start, leave "Add a README" unchecked) and run what it shows:

```powershell
git remote add origin https://github.com/YOUR-USERNAME/mtejaflow.git
git push -u origin main
```

or, with the GitHub CLI (`winget install GitHub.cli`, then `gh auth login`):

```powershell
gh repo create mtejaflow --private --source . --remote origin --push
```

The `.gitignore` files keep `.env`, `node_modules` and build output out of the repository. If a real `.env` is ever committed by mistake, change every secret in it (database password, `JWT_SECRET`), because Git history keeps old files.

## Everyday commands

Run from the project folder.

| Task | Command |
|---|---|
| Install everything | `npm run install:all` |
| Start API and web app (development) | `npm run dev` |
| Generate Prisma client | `npm run db:generate` |
| Apply database migrations | `npm run db:migrate` |
| Reset the development database (**deletes all data**) | `npm run db:reset` |
| Load demo data | `npm run db:seed` |
| Type-check | `npm run typecheck` |
| Lint | `npm run lint` |
| Run all tests | `npm test` |
| Production build | `npm run build` |
| Run the production build | `npm start` (see *Production*) |
| Regenerate API docs | `npm run docs:openapi` |

Backend-only and frontend-only variants: `npm --prefix backend run <script>` and `npm --prefix frontend run <script>`.

### Changing the database schema

Edit `backend/prisma/schema.prisma`, then write a new folder under `backend/prisma/migrations/<timestamp>_<name>/migration.sql` containing the SQL change (`npx prisma migrate diff` can draft it), run `npm run db:migrate`, then `npm run db:generate`. Never edit a migration that has already been applied; the runner refuses to continue if you do.

## Testing

```powershell
npm test
```

Backend tests run against a **real PostgreSQL database** (`TEST_DATABASE_URL`), which is migrated automatically before the run and cleared between tests. **Do not point `TEST_DATABASE_URL` at a database you care about.**

The backend suite covers registration/login, unauthorized access, cross-account isolation, customer CRUD, enquiry status changes, quotation arithmetic, PDF generation and contents, follow-up scheduling and timezones, payment calculations, concurrency (simultaneous payments cannot overpay), outstanding balances, reports, security headers/CORS/CSRF/rate limiting/log hygiene, migrations, seed data and an OpenAPI route-drift check. The frontend suite covers money maths, phone handling, the API client, translations, modal accessibility and the registration form.

### Browser workflow test (optional)

`e2e/` contains a script that drives the real app in a browser through the whole workflow: register, onboarding, customers, enquiry plus follow-up, pipeline, quotation with live totals and a real PDF download, job, M-Pesa payment (overpayment and duplicate-code blocking), message assistant, dashboard and report figures, settings, billing, dark mode, logout/login, **a second account that must not see the first account's data**, and a phone-sized screen checked for sideways scrolling. It creates throwaway accounts, so use a development database.

```powershell
npm --prefix e2e install
npm run dev                                     # in one window
$env:BASE = "http://localhost:5173"             # in another
npm --prefix e2e run test
```

It launches your installed Google Chrome (`$env:PW_CHANNEL = "msedge"` for Edge). Screenshots are written to `e2e\screenshots`. Sign-in is rate limited (20 per 15 minutes); a run uses about 8, so restart the API if you run it many times in a row.

## Configuration reference

### `backend/.env`

| Variable | Required | Description |
|---|---|---|
| `DATABASE_URL` | yes | PostgreSQL connection string |
| `TEST_DATABASE_URL` | for tests | A **separate** database used by `npm test` |
| `JWT_SECRET` | yes | At least 32 random characters. Signs session tokens. Keep secret |
| `CLIENT_ORIGIN` | no | Origin allowed by CORS (default `http://localhost:5173`) |
| `PORT` | no | API port (default 4000) |
| `NODE_ENV` | no | `development`, `test` or `production` |
| `FRONTEND_DIST` | no | Folder with the built web app; when set the API serves it too (single-process deployment) |
| `COOKIE_SECURE` | no | Defaults to `true` in production. Set `false` only to test a production build over plain http |

### `frontend/.env` (optional)

| Variable | Description |
|---|---|
| `VITE_CONTACT_EMAIL`, `VITE_CONTACT_WHATSAPP` | Shown in the landing page's Contact section. In production the section is hidden until one is set |
| `VITE_API_TARGET` | Dev/preview only: where to forward `/api` (default `http://localhost:4000`) |

## API

Interactive-friendly OpenAPI 3 description: `docs/openapi.json` (also served at `/api/openapi.json`). It is generated from the same Zod schemas the server validates with, and a test fails if it ever disagrees with the real routes.

Conventions: cookie session (`mf_token`, HTTP-only); every non-GET request needs the header `X-Requested-With: mtejaflow` (CSRF defence); errors are always `{ "error": { "code", "message", "details?" } }`; money is a decimal **string**; every record belongs to one business and another account's ids return 404.

## Production

### Single process

```powershell
npm run build
$env:NODE_ENV = "production"
$env:DATABASE_URL = "postgresql://..."
$env:JWT_SECRET = "..."
$env:CLIENT_ORIGIN = "https://your-domain"
$env:FRONTEND_DIST = (Resolve-Path frontend\dist).Path
npm --prefix backend run db:deploy
npm start
```

Serve it **over HTTPS** (for example behind Caddy, nginx or your host's TLS). Session cookies are `Secure` in production.

### Docker

```powershell
Copy-Item .env.example .env     # fill in POSTGRES_PASSWORD and JWT_SECRET
docker compose up --build
```

Open **http://localhost:8080**. The container applies migrations on start. More in `docs/DEPLOYMENT.md`.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `Invalid environment configuration` and a list of fields | A required variable in `backend\.env` is missing; `JWT_SECRET` needs 32+ characters |
| `ECONNREFUSED 127.0.0.1:5432` | PostgreSQL is not running. Start the service (`services.msc`, or `docker compose up -d db`) |
| `password authentication failed` | The user/password in `DATABASE_URL` does not match what you created in step 2 |
| `database "mtejaflow" does not exist` | Run the `CREATE DATABASE` commands again |
| `npm run db:generate` fails to download | Prisma downloads its tooling on first use; check your internet/proxy and retry |
| Tests say `TEST_DATABASE_URL is not set` | Add it to `backend\.env` |
| `port is already allocated` when starting Docker | Another program uses that port (often a local PostgreSQL on 5432). The default compose file no longer publishes the database port; for the app port change `APP_PORT` in `.env` |
| `EADDRINUSE` on 4000 or 5173 | Another program uses the port. Find it: `netstat -ano \| findstr :4000`, then end it or change `PORT` |
| Logged in, but you are sent back to the login page on a deployed site | Your site is on http but `COOKIE_SECURE` is true: use HTTPS, or set `COOKIE_SECURE=false` for a local test only |
| 403 `CSRF` on every action | Something is calling the API without `X-Requested-With: mtejaflow`; use the web app or add the header |
| PDF preview is blank | Use **Download PDF** or **Open in new tab**. Some phone browsers cannot show PDFs inside a page |
| Reports say the range is too long | Report length depends on the plan (31 / 92 / 366 days) |
| Too many sign-in attempts | Sign-in and registration are rate limited; wait a few minutes |

## Project layout

```
backend/    Express API (src/), Prisma schema and SQL migrations (prisma/), tests (tests/)
frontend/   React app (src/features/* per area, src/components, src/lib)
e2e/        optional browser workflow test
docs/       openapi.json, SECURITY.md, DEPLOYMENT.md, BILLING_INTEGRATION.md
Dockerfile, docker-compose.yml, package.json (root scripts)
```

## Verification status

Run and passing in the development environment this was built in (Linux, Node 22, PostgreSQL 16, headless Chromium):

- Backend: type-check, lint, build, **115 integration tests** against real PostgreSQL.
- Frontend: type-check, lint, production build, **40 unit/component tests**.
- A scripted **end-to-end run in a real browser** (`e2e/`, 22 steps) covering the main workflow, account isolation, dark mode and phone-sized layouts, run against both the development setup and the **production build served by the API with its strict security headers**.

**Not verified, so please check on your own machine:** Windows and PowerShell specifically; the Prisma CLI commands (`npm run db:generate`; the migration runner was verified instead); the Docker image and compose file (Docker was not available); Safari, Firefox and real phones; the PDF preview inside Chrome's built-in viewer (the PDF file itself was rendered and inspected); Kiswahili wording.
# mtejaflow_2
