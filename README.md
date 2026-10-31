# Competitive Intelligence Monitor

Watches competitor websites, detects meaningful changes, scores and classifies
them with an LLM, and turns them into reviewed briefings and sales battlecards.

## Features

- **Multi-page monitoring** — each competitor is tracked through a small set of
  role-selected pages (pricing, product, etc.) with text and visual diffing.
- **LLM scoring** — every change gets a materiality score and a classification;
  cross-competitor synthesis uses embeddings (pgvector).
- **Human approval gate** — briefings and battlecard updates are drafts until a
  team member approves them; nothing is delivered without sign-off.
- **Delivery** — Slack, email and CRM connectors.
- **Teams and governance** — workspaces, roles, audit log, per-workspace LLM
  budget, GDPR export and delete.
- **Benchmarking** — compare your own site against competitors, plus traffic and
  category-pricing views.
- **Public demo** — a read-only workspace reachable from the login page
  (see [docs/DEMO.md](docs/DEMO.md)).

## Architecture

| Part | Stack |
| --- | --- |
| API | FastAPI, SQLAlchemy, Alembic |
| Database | PostgreSQL with the `pgvector` extension |
| Background jobs | arq on Redis, separate worker service ([docs](docs/QUEUE_AND_WORKER.md)) |
| Scheduler | APScheduler in the web process |
| Rendering | Playwright (Chromium) |
| LLM | NVIDIA NIM through its OpenAI-compatible API |
| Frontend | Next.js (App Router), React, Tailwind CSS |

Postgres is the source of truth for every job; Redis only carries messages.

```
backend/    FastAPI app: routers, services, models, schemas, migrations, tests
frontend/   Next.js app
docs/       Queue/worker design and demo setup
```

## Getting started

### Prerequisites

Python 3.13, Node.js 22, PostgreSQL 16 with `pgvector`, and (for background
jobs) Redis.

### Backend

```bash
cd backend
python -m venv venv
venv\Scripts\activate          # Windows; use `source venv/bin/activate` elsewhere
pip install -r requirements.txt
playwright install chromium
```

Create `backend/.env`:

```env
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/ci_monitor
JWT_SECRET_KEY=change-me
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=60

# Optional
NVIDIA_API_KEY=
REDIS_URL=
```

Enable the extension, migrate, and start the API:

```bash
psql -d ci_monitor -c "CREATE EXTENSION IF NOT EXISTS vector;"
alembic upgrade head
uvicorn app.main:app --reload --port 8000
```

Run the worker as a second process (it needs the same environment):

```bash
arq app.worker.WorkerSettings
```

### Frontend

```bash
cd frontend
npm ci
echo NEXT_PUBLIC_API_URL=http://localhost:8000 > .env.local
npm run dev
```

### Both at once

From the repository root: `npm install && npm run dev`.

## Configuration

All settings are read from environment variables (see
`backend/app/core/config.py`).

| Variable | Purpose |
| --- | --- |
| `DATABASE_URL`, `JWT_SECRET_KEY`, `JWT_ALGORITHM`, `ACCESS_TOKEN_EXPIRE_MINUTES` | Required |
| `NVIDIA_API_KEY`, `NIM_CHAT_MODEL`, `NIM_EMBED_MODEL` | LLM scoring and embeddings |
| `REDIS_URL` | Job queue |
| `ENABLE_BROWSER_RENDERING`, `ENABLE_BROWSER_DISCOVERY` | Chromium rendering of pages |
| `MAX_ACTIVE_SURFACES_PER_COMPETITOR` | Watched pages per competitor (default 3) |
| `SMTP_*` | Email delivery |
| `SIMILARWEB_API_KEY` | Traffic data |
| `DEMO_USER_EMAIL`, `DEMO_USER_PASSWORD` | Public demo sign-in |
| `NEXT_PUBLIC_API_URL` | Backend URL used by the frontend |

## Testing

```bash
cd backend && pytest -q
cd frontend && npm run lint && npm test && npm run build
```

CI runs both suites, plus an Alembic downgrade/upgrade round trip, on every push
and pull request.

## Deployment

The backend ships as one Docker image (`backend/Dockerfile`) run as two
services: the web process (default `CMD`) and the worker
(`arq app.worker.WorkerSettings`). Both must receive the same `NIM_*`
variables. See [docs/QUEUE_AND_WORKER.md](docs/QUEUE_AND_WORKER.md).
