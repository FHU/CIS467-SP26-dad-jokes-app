# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Prerequisites

Before first run, create the secrets file the compose stack requires:

```bash
mkdir -p secrets
echo "dadjokes" > secrets/postgres_password.txt
```

## Running the App

```bash
# Start all services (dev targets — Vite dev server + ts-node)
docker compose up --build

# Hot-reload mode (syncs source changes without full rebuild)
docker compose watch

# Stop and clean up (remove volumes for a full DB reset)
docker compose down -v
```

| URL | Service |
|-----|---------|
| `http://localhost:8080` | Frontend (Vite dev server) |
| `http://localhost:3000` | Express API |
| `http://localhost:8081` | pgAdmin (admin@admin.com / admin) |
| `http://localhost:6379` | Redis |

In dev mode, the Vite dev server (not nginx) proxies `/api/*` to `http://api:3000`. The nginx reverse-proxy path is only used in the production (`prod`) Dockerfile stage.

## Local Development (without Docker)

Requires a running PostgreSQL instance.

```bash
# API (port 3000)
cd api && npm install
DB_HOST=localhost DB_USER=dadjokes DB_PASSWORD=dadjokes DB_NAME=dadjokes npx ts-node src/index.ts

# Frontend (port 5173)
cd frontend && npm install && npm run dev
# Before running, change the proxy target in vite.config.ts from http://api:3000 → http://localhost:3000
```

### Build commands

```bash
cd api && npm run build        # Compile TypeScript → dist/
cd frontend && npm run build   # tsc check + Vite production build
```

There are no lint or test scripts defined in either package.json.

## Architecture

Five-service stack: **React (Vite) → Express API → PostgreSQL**, plus **Redis** and **pgAdmin**.

- **[frontend/src/App.tsx](frontend/src/App.tsx)** — Single monolithic React component with all UI, state, and fetch logic. Three views: Random, Browse (with category filter), Add. Uses only React hooks (no external state library). All styles are inline.
- **[api/src/index.ts](api/src/index.ts)** — Single-file Express server with all 8 REST endpoints and direct `pg` pool queries. No ORM; uses parameterized queries. Redis is wired in `docker-compose.yml` but not yet integrated in the API code.
- **[db/init.sql](db/init.sql)** — Single `jokes` table schema + 20 seed rows across categories (science, food, work, animals, tech, general, nature, sports).
- **[frontend/vite.config.ts](frontend/vite.config.ts)** — Configures the `/api` proxy to `http://api:3000` for containerized dev; change target to `http://localhost:3000` for local-only development.
- **[frontend/nginx.conf](frontend/nginx.conf)** — Production only: serves the SPA and reverse-proxies `/api/*` to `http://api:3000`. Handles SPA client-side routing via `try_files`.

### API Endpoints

| Method | Path | Notes |
|--------|------|-------|
| GET | `/api/health` | DB connectivity check |
| GET | `/api/jokes` | Optional `?category=` filter |
| GET | `/api/jokes/random` | Increments `times_told` |
| GET | `/api/jokes/:id` | |
| POST | `/api/jokes` | Body: `{ setup, punchline, category }` |
| PATCH | `/api/jokes/:id/rate` | Body: `{ rating }` (0–5) |
| DELETE | `/api/jokes/:id` | |
| GET | `/api/categories` | Distinct categories from DB |

### Docker Setup

Both app services use multi-stage Dockerfiles (`base → dev → build → prod`). The compose file targets the `dev` stage: the API runs via `ts-node` and the frontend runs the Vite dev server. The `develop.watch` blocks in `docker-compose.yml` enable file sync without rebuilds when using `docker compose watch`. Docker Compose health checks gate startup order (`api` waits for `db` and `redis`; `pgadmin` waits for `db`).

### Database connection env vars

The API reads `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, and `DB_NAME`. In the compose stack the password is injected via the Docker secret at `./secrets/postgres_password.txt` (mounted as `DB_PASSWORD_FILE`); the API falls back to the hardcoded default `"dadjokes"` if `DB_PASSWORD` is unset.
