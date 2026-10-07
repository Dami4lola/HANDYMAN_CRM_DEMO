# HANDYMAN_CRM_DEMO
# Worker Portal

A workforce management web app for trades and contracting businesses. Field workers log timesheets and receipts from their phones, even offline. Managers schedule jobs, price estimates, issue invoices, run payroll and track how much each job is making.

## Features

- **Timesheets**: workers log hours, travel and materials per job. Billable hours are rounded to the quarter hour with a 4-hour minimum, and managers approve and mark them paid.
- **Jobs and scheduling**: jobs can have several workers, with a day-by-day crew schedule and a calendar view. Site distance is calculated automatically with Google Maps.
- **Estimates**: a calculator that prices labour, travel, equipment, scaffolding, materials (with Home Depot price lookup) and fees. It exports a PDF, and an accepted estimate converts into a job. Prospective clients can also request a quote on a public page.
- **Invoices**: auto-numbered `INV-YYYY-NNNN`, with HST applied and a PDF generated with ReportLab. Progress billing across several invoices per job is supported.
- **Job financials**: billed vs. cost vs. quote for each job and across all jobs, with gross profit and over-quote flags.
- **Payroll**: worker payouts covering labour, distance allowance and reimbursable materials, with subcontractor HST. Exports to PDF.
- **Inspections**: pre-job and post-job inspections with photo attachments stored in S3.
- **Receipts and purchase list**: receipt uploads to S3. Receipts paid with a company card are excluded from reimbursement automatically.
- **Time off and SMS reminders**: time-off requests, plus scheduled shift reminders sent with Twilio.
- **Offline PWA**: installable, and a Workbox service worker queues changes made without a signal.

## Roles

| Role | Access |
| --- | --- |
| Worker | Submits timesheets and receipts, views assigned jobs, completes inspections |
| Estimator | Worker access plus the estimate calculator; cannot create or manage jobs |
| Manager | Jobs, clients, all timesheets, invoices, payroll and financials |
| Admin | Everything, plus user and role management |

## Tech stack

| Layer | Stack |
| --- | --- |
| Frontend | React 18, TypeScript, Vite, Tailwind CSS, Radix UI, Zustand, TanStack Query, vite-plugin-pwa |
| Backend | FastAPI, Python 3.11, SQLModel, Alembic, APScheduler, ReportLab |
| Database | PostgreSQL in production, SQLite for local development and tests |
| Auth | JWT access and refresh tokens, bcrypt password hashing |
| Integrations | AWS S3, Google Maps Distance Matrix, Resend, Twilio, SerpApi |

## Project structure

```
backend/
  app/
    api/          route modules, one per resource (mounted under /api)
    models/       SQLModel tables
    schemas/      request and response models
    services/     pricing, payroll, PDFs, S3, email, SMS, distance
  alembic/        database migrations
  scripts/        dev-db.sh (local database bootstrap)
  tests/
  start.sh        production entrypoint
frontend/
  src/
    api/          typed API client per resource
    components/
    pages/
    store/        Zustand auth store
    types/
docker-compose.yml
```

## Getting started

### Prerequisites

- Python 3.11
- Node.js 18 or newer
- PostgreSQL, only if you are not using SQLite locally

### Backend

```bash
cd backend
python3.11 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env            # then set SECRET_KEY at minimum
./scripts/dev-db.sh             # create tables and stamp migrations
uvicorn app.main:app --reload
```

The API runs at `http://localhost:8000`, with interactive docs at `http://localhost:8000/docs`.

Use `scripts/dev-db.sh` on an empty database, not `alembic upgrade head`. Some early migrations fail with "duplicate column" on a fresh database. The script does what `start.sh` does in production: it creates the tables, then stamps the migrations as applied. Use `alembic upgrade head` only to apply new migrations to an existing database.

### Frontend

```bash
cd frontend
npm install
cp .env.example .env            # VITE_API_URL=http://localhost:8000/api
npm run dev
```

The app runs at `http://localhost:5173`.

### Docker

```bash
docker compose up --build
```

This starts Postgres, the API on port 8000 and the built frontend on port 80.

### First admin account

New sign-ups get the `worker` role. To promote your first account to admin, register through the app and then run this against the database:

```sql
INSERT INTO user_role_link (user_id, role_id)
SELECT u.id, r.id FROM "user" u, role r
WHERE u.username = 'your-username' AND r.name = 'admin';
```

After that, an admin can assign roles from the Admin page.

## Configuration

The backend reads `backend/.env`. See `backend/.env.example` for the full list.

| Variable | Required | Purpose |
| --- | --- | --- |
| `SECRET_KEY` | Yes | JWT signing key. Generate one with `python -c "import secrets; print(secrets.token_urlsafe(32))"` |
| `DATABASE_URL` | No | Defaults to a local SQLite file |
| `CORS_ORIGINS` | No | Comma-separated list of allowed frontend URLs |
| `FRONTEND_URL` | No | Base URL used in password-reset links |
| `OFFICE_ADDRESS` | No | Starting point for all job distance calculations |
| `COMPANY_CARD_DIGITS` | No | Last 4 digits of company cards, comma-separated; receipts paid with them are not reimbursed |
| `GOOGLE_MAPS_API_KEY` | No | Automatic job distances (enable the Distance Matrix API) |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_S3_BUCKET_NAME`, `AWS_S3_REGION` | No | Receipt and inspection photo storage |
| `RESEND_API_KEY` | No | Password-reset and notification emails |
| `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_PHONE_NUMBER` | No | SMS shift reminders |
| `SERPAPI_KEY` | No | Home Depot material price lookup in the estimator |

The frontend reads `VITE_API_URL` when it is built. If the API URL changes, rebuild the frontend.

### Adapting to your business

Some defaults belong to the business this app was first built for. Change them when you deploy for someone else:

- Set `OFFICE_ADDRESS`, `COMPANY_CARD_DIGITS` and `AWS_S3_BUCKET_NAME`. Each has a hard-coded default in `backend/app/config.py`.
- Replace the logo in `backend/app/services/` that is printed on invoice, estimate and payroll PDFs.
- The tax rate (13% Ontario HST), the 4-hour minimum and the default billing rates for labour, Red Seal labour and km are constants at the top of `backend/app/services/job_cost.py`.
- Managers can adjust the public quick-quote rates and the admin fee from inside the app.

## Testing

```bash
cd backend && pytest
cd frontend && npm run test:run
```

The backend tests use SQLite and need no external services.

## Deployment

- **Backend**: deployed on Railway with Postgres. The backend `Dockerfile` runs `start.sh`, which creates any missing tables, runs `alembic upgrade head` and then starts uvicorn on `$PORT`. The `Procfile` only starts uvicorn and skips migrations, so make sure your host runs `start.sh`.
- **Frontend**: deployed on Vercel (`vercel.json`), or with the frontend `Dockerfile`, which serves the build through nginx.

### Writing migrations

`create_all()` runs before Alembic on every deploy, so new tables and columns already exist when a migration runs. Every `upgrade()` must check what exists first, using `inspector.get_table_names()` or `inspector.get_columns()`, before it calls `op.create_table()` or `op.add_column()`. Wrap any `op.alter_column(type_=...)` in a check for `conn.dialect.name != "sqlite"`, because SQLite cannot change a column's type.

## Further reading

- `OBATEK_DEVELOPER_GUIDE.md`: architecture, data model, API endpoints and payout logic in depth
- `ESTIMATOR_FEATURE_SPEC.md`: the estimate calculator specification
