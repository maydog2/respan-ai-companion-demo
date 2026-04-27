# Deployment Guide

## Local Reviewer Setup

For local review, run both apps on your machine:

- Frontend: `http://localhost:3000`
- Backend API: `http://127.0.0.1:8000`
- Swagger docs: `http://127.0.0.1:8000/docs`

The frontend is configured so `npm run dev` calls the local backend by default. You only need `NEXT_PUBLIC_API_URL` when explicitly testing a non-local backend.

### 1) Backend

From the repository root:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
copy .env.example .env
```

Set these values in `.env`:

```env
DB_URL=postgresql://app:app_pw_123@127.0.0.1:5433/companion
AUTH_TOKEN_SECRET=change-me
RESPAN_API_KEY=...
RESPAN_MODEL=gpt-4o
```

`DB_URL` can point to a local PostgreSQL instance, a Docker PostgreSQL container, or another PostgreSQL database you control. The Docker example below uses the same connection string.

Initialize a new empty database once:

```powershell
$env:PYTHONPATH = "$PWD\src"
python -m companion.infra.init_db
```

Start FastAPI:

```powershell
$env:PYTHONPATH = "$PWD\src"
uvicorn companion.api:app --reload --host 0.0.0.0 --port 8000
```

Open `http://127.0.0.1:8000/docs` to confirm the API is running.

### 2) Frontend

In a second terminal:

```powershell
cd frontend
copy .env.local.example .env.local
npm install
npm run dev
```

Open `http://localhost:3000`. The default `frontend/.env.local.example` points to `http://127.0.0.1:8000`, and `frontend/lib/api.ts` also forces local API usage during `next dev` unless `NEXT_PUBLIC_DEV_USE_REMOTE_API=1` is set.

### 3) Verify Respan Gateway

After backend `.env` is saved:

```powershell
$env:PYTHONPATH = "$PWD\src"
python -c "from dotenv import load_dotenv; load_dotenv(override=True); from companion.infra import llm; print(llm._base_url()); print(llm._main_model()); print(llm.get_reply([{'role':'user','content':'Reply with exactly: respan-ok'}]))"
```

Expected gateway base URL:

```text
https://api.respan.ai/api/
```

---

## Local PostgreSQL

The backend requires PostgreSQL. The fastest local setup is Docker:

```powershell
docker run --name companion-postgres `
  -e POSTGRES_USER=app `
  -e POSTGRES_PASSWORD=app_pw_123 `
  -e POSTGRES_DB=companion `
  -p 5433:5432 `
  -d postgres:16
```

Use this backend `.env` value:

```env
DB_URL=postgresql://app:app_pw_123@127.0.0.1:5433/companion
```

Check that the container is running:

```powershell
docker ps --filter "name=companion-postgres"
```

If the container already exists but is stopped:

```powershell
docker start companion-postgres
```

After PostgreSQL is running, create the app schema once:

```powershell
$env:PYTHONPATH = "$PWD\src"
python -m companion.infra.init_db
```

Do not run reset commands unless you intentionally want to wipe local data. If port `5433` is already in use, choose another host port and update `DB_URL` to match, for example `-p 5434:5432` with `DB_URL=postgresql://app:app_pw_123@127.0.0.1:5434/companion`.

---

## Environment Variables

Backend `.env` values:

| Variable | Required | Description |
|----------|----------|-------------|
| `DB_URL` | Yes | PostgreSQL connection string for local Docker/Postgres or a managed database |
| `AUTH_TOKEN_SECRET` | Yes | Long random secret used to sign auth tokens |
| `RESPAN_API_KEY` | Yes* | Preferred API key for chat through Respan Gateway |
| `RESPAN_MODEL` | Optional | Respan model id, e.g. `gpt-4o` or `gpt-4o-mini` |
| `RESPAN_BASE_URL` | Optional | Respan Gateway override; defaults to `https://api.respan.ai/api/` |
| `OPENAI_API_KEY` | Yes* | Fallback API key for direct OpenAI-compatible providers |
| `OPENAI_BASE_URL` | Optional | Fallback OpenAI-compatible base URL |
| `OPENAI_MODEL` | Optional | Fallback model id |

\*Set either `RESPAN_API_KEY` or `OPENAI_API_KEY` if you use endpoints that call the LLM.

Frontend `frontend/.env.local` values:

| Variable | Required | Description |
|----------|----------|-------------|
| `NEXT_PUBLIC_API_URL` | Optional for dev | Defaults to `http://127.0.0.1:8000` in `npm run dev` |
| `NEXT_PUBLIC_DEV_USE_REMOTE_API` | Optional | Set to `1` only if you want `npm run dev` to call a remote backend |

---

## Respan Notes

The backend is wired to prefer **Respan Gateway** for LLM calls. Respan's gateway is OpenAI-compatible, so the app uses the OpenAI Python SDK while pointing it at Respan's base URL.

By default, `RESPAN_API_KEY` sends chat completions to:

```text
https://api.respan.ai/api/
```

Optional overrides:

- `RESPAN_MODEL` sets the main chat model. If unset, the backend falls back to `OPENAI_MODEL` or `gpt-4o`.
- `RESPAN_BASE_URL` overrides the default Respan Gateway base URL.
- `RESPAN_TONE_MODEL` and `RESPAN_RELATIONSHIP_MODEL` can override the smaller auxiliary LLM calls used for tone hints and relationship trigger classification.

If you use OpenAI models through Respan, make sure the Respan project has either provider credentials configured in **Settings -> Providers** or credits available in **Settings -> Credits**. Otherwise Respan may return an upstream `401` for models such as `gpt-4o`.

---

## Local Troubleshooting

| Symptom | Likely cause | What to do |
|---------|--------------|------------|
| Frontend shows `Failed to fetch` | Backend is not running on port `8000` | Start FastAPI and open `http://127.0.0.1:8000/docs` |
| Login/register fails at startup | `AUTH_TOKEN_SECRET` is missing | Set `AUTH_TOKEN_SECRET` in backend `.env` and restart Uvicorn |
| DB connection errors | `DB_URL` is wrong or PostgreSQL is not running | Start your local/Docker database or update `DB_URL` |
| LLM returns `401 upstream_provider_error` | Respan key works, but the selected upstream model lacks provider credentials/credits | Configure provider credentials or credits in Respan, or use another available model |
| Frontend calls the wrong backend | `NEXT_PUBLIC_DEV_USE_REMOTE_API=1` is set | Remove it or set `NEXT_PUBLIC_API_URL=http://127.0.0.1:8000` |
