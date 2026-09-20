
# Vageesh — Agentic AI Orchestration Platform

React + TypeScript frontend + FastAPI backend + PostgreSQL database.

---

## Architecture Diagram
![Architecture Diagram for Vageesh](doc/architecture.png)

---

## Repositories

| Repo | Link | Purpose |
|---|---|---|
| **vageesh-back-end** | [github.com/ajay-shriwastava/vageesh-back-end](https://github.com/ajay-shriwastava/vageesh-back-end) | FastAPI server — REST API, LangGraph workflow execution, Slack bot, APScheduler cron, Alembic migrations, WebSocket broadcast, Docker Compose entry point |
| **vageesh-front-end** | [github.com/ajay-shriwastava/vageesh-front-end](https://github.com/ajay-shriwastava/vageesh-front-end) | React + TypeScript SPA — visual workflow builder, agent management, live run panel, Vite dev server (local) / nginx (Docker) |

Both repos must be cloned side-by-side (`docker-compose.yml` in `vageesh-back-end` references `../vageesh-front-end`).

---

## Getting Started

### Prerequisites

| Tool | Version | Install |
|---|---|---|
| Git | any | https://git-scm.com |
| Docker Desktop | 4.x+ | https://www.docker.com/products/docker-desktop |
| An Anthropic API key | — | https://console.anthropic.com |

Docker Desktop must be running before you start.

### 1. Clone the repositories

Vageesh consists of two repos that must sit next to each other in the same parent directory:

```bash
mkdir vageesh && cd vageesh

git clone https://github.com/ajay-shriwastava/vageesh-back-end.git
git clone https://github.com/ajay-shriwastava/vageesh-front-end.git
```

Your directory structure should look like this:

```
vageesh/
  vageesh-back-end/     ← FastAPI backend (clone this first)
  vageesh-front-end/    ← Vite frontend
```

### 2. Configure environment variables

```bash
cd vageesh-back-end
cp .env.example .env
```

Open `.env` and set your values:

```
ANTHROPIC_API_KEY=sk-ant-...        # required — get from console.anthropic.com
SLACK_BOT_TOKEN=xoxb-...            # optional — Slack integration
SLACK_APP_TOKEN=xapp-...            # optional — Slack integration
SLACK_REPORT_CHANNEL=data-reports   # optional — Slack channel for reports
```

All other values in `.env.example` can be left at their defaults for a local run.

### 3. Start everything

```bash
docker compose up --build
```

This single command:
- Starts PostgreSQL and waits for it to be healthy
- Runs all database migrations automatically (`alembic upgrade head`)
- Builds and starts the FastAPI backend
- Builds the frontend and serves it via nginx

First build takes 2–4 minutes (downloading base images, installing dependencies). Subsequent runs: `docker compose up`.

### 4. Open the app

| URL | What you get |
|---|---|
| http://localhost | Vageesh UI |
| http://localhost:8000/docs | Interactive API docs (Swagger UI) |

### 5. Stop

```bash
docker compose down          # stops containers, keeps the database volume
docker compose down -v       # stops containers AND deletes the database
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 19 + TypeScript + Vite (dev) / nginx (Docker) |
| Backend | FastAPI (Python 3.11+), async SQLAlchemy 2, Alembic |
| Database | PostgreSQL (asyncpg driver) |
| Agent runtime | LangGraph + langchain-anthropic (integrated into FastAPI services) |
| Real-time | FastAPI WebSocket endpoints, ConnectionManager pattern (`app/ws_manager.py`) |
| Messaging | Slack Socket Mode bot (`app/slack_bot.py`) |
| Scheduling | APScheduler cron integration (`app/scheduler.py`) |
| MCP | FastMCP server at `POST /mcp` — agent memory + knowledge base tools with audit log |
| Observability | LangSmith tracing (opt-in via env vars) |
| Containerisation | Docker Compose (single-command setup) |

---

## Quick Start — Docker

```bash
cd vageesh-back-end
cp .env.example .env          # fill in ANTHROPIC_API_KEY (and optional Slack tokens)
docker compose up --build
```

- **Frontend**: http://localhost
- **API docs**: http://localhost:8000/docs

This single command starts Postgres, runs all Alembic migrations, and brings up the backend and frontend. Subsequent runs need only `docker compose up`.

---

## Local Dev Setup

### 1. PostgreSQL — create the database

```bash
brew services restart postgresql
psql -U postgres -c "CREATE DATABASE symphony;"
```

### 2. Backend (`vageesh-back-end`)

```bash
mkvirtualenv symphony
workon symphony
pip install -r requirements.txt
alembic upgrade head
fastapi dev app/main.py        # → http://127.0.0.1:8000/docs
```

### 3. Frontend (`vageesh-front-end`)

```bash
npm install
npm run dev                    # → http://localhost:5173
```

> In local dev without Docker, set `VITE_API_BASE=http://localhost:8000` in `vageesh-front-end/.env`.

---

## Features

### Agent Management
Create and configure AI agents with a name, model (Claude Sonnet / Haiku), system prompt, tools, and memory. Each agent can be independently scheduled, given skills, interaction rules, and guardrails, and assigned to messaging channels.

### Visual Workflow Builder
Drag-and-drop SVG canvas on the Workflows page. Supports Start, Agent, Condition, and End nodes connected by bezier edges. Condition nodes support branching (true/false) and feedback loops up to a configurable `max_loops` (default: 20).

### Workflow Execution
Workflows run as LangGraph graphs. Each run is tracked in the `workflow_runs` table with status, input/output, and token usage. Live execution events stream to the browser via WebSocket (`node_enter`, `node_complete`, `edge_traverse`, `run_complete`, `run_error`).

### Workflow Templates
Two pre-built templates available from the Workflows page:

| Template | Schedule | What it does |
|---|---|---|
| Data Ingestion Pipeline | Every minute | Scans for CSVs, checks quality, ingests to DB, profiles data, posts report to Slack |
| SRE Job Summary | Every hour | Queries 24h workflow run stats, writes a health summary, posts to Slack |

### Workflow Tool Configuration
Each workflow carries its own `tool_config` — a per-workflow override for operational parameters like `dataset_dir`, `slack_channel`, and `catalogue_path`. This creates a two-level hierarchy:

| Level | Where set | Purpose |
|---|---|---|
| `.env` | Server environment | Secrets and infra defaults (API keys, DB URLs, fallback paths) |
| `workflow.tool_config` | Vageesh UI | Operational params per workflow instance — override the env defaults |

**Where to configure:**
- **Workflow Builder** — click any tool or agent node in the canvas; param fields appear inline in the config panel (e.g. *Dataset Directory* under a `csv_scanner` node).
- **Config → Workflow Config** — flat audit view showing all tool params for a workflow in one place. Navigate via the Config page workflow dropdown.

**How it works at runtime:**
- Pipeline tool nodes: `tool_config[tool_name]` is merged into the state dict before `run()` is called. The tool reads the value from `state.get("dataset_dir") or DATASET_DIR`.
- LLM tool nodes (`@tool`): config is injected via a Python `ContextVar` scoped to the workflow execution coroutine. The `@tool` function reads `tool_config_var.get().get(tool_name, {})` — the parameter is never exposed as an LLM-visible argument.

**Templates pre-populate defaults** — instantiating a template fills `tool_config` with sensible placeholder values so you can see what's configurable immediately.

### Agent Messaging via Slack
A Socket Mode Slack bot starts automatically with the FastAPI server. It routes DMs and @mentions to the configured agent and persists all messages. Configurable per-agent via Agent Configuration → Channels.

### Agent Configuration
Per-agent settings managed via the UI (memory page):
- **Memory**: persistent key/value store per agent
- **Schedules**: cron-based automatic triggering
- **Skills**: capabilities the agent is allowed to use
- **Interaction Rules**: constraints on how the agent communicates
- **Guardrails**: safety boundaries
- **Channels**: messaging integrations (e.g. `slack`)

### Doc Store (Knowledge Base)
Upload PDF or plain text files, or paste raw text, to build a searchable vector knowledge base. Text is chunked, embedded via VoyageAI, and stored in PostgreSQL with pgvector. The Search tab performs semantic similarity search returning ranked chunks. File uploads accept `.pdf` (parsed with pypdf) and `.txt` (UTF-8 decode) via `POST /api/v1/knowledge/upload`.

### MCP Server
Vageesh exposes an MCP (Model Context Protocol) server at `POST /mcp` that provides controlled, audited access to agent memory and the knowledge base. All external clients — Slack bot, Claude Desktop, Cursor — go through this single gateway.

**8 tools across two categories:**

| Category | Tools |
|---|---|
| Memory | `get_memory`, `list_memory`, `set_memory`, `delete_memory` |
| Knowledge | `list_knowledge`, `add_knowledge`, `add_knowledge_file`, `search_knowledge` |

**Auth:** `X-MCP-API-Key` header. Set `MCP_API_KEY` in `.env`. If the key is absent, the endpoint returns 503.

**Audit log:** Every tool call is persisted to the `mcp_audit_log` table (fire-and-forget). View the audit log at `/mcp` in the UI.

**Slack enrichment:** When `MCP_API_KEY` is set, the Slack bot automatically:
- Injects agent memory and top-3 relevant knowledge chunks into each LLM system prompt
- Parses `[REMEMBER key: value]` markers in LLM replies and writes them back to agent memory via `set_memory`

**Claude Desktop integration:** Copy the `mcpServers` config JSON snippet from the MCP Server page in the UI.

### Observability
LangSmith tracing for all LLM calls — full prompt/response, token counts, cost, and per-node latency. Opt-in via environment variables, no code changes needed.

---

## Tests

195 tests (integration + unit + eval) covering all routers, the workflow runner, agent memory, and MCP endpoints.

```bash
# One-time: create the test database
psql -U postgres -c "CREATE DATABASE symphony_test;"

# Run all tests
workon symphony
pytest

# With coverage
pytest --cov=app --cov-report=term-missing
```

Tests use a dedicated `symphony_test` database (never touches the dev database). All tables are truncated between tests.

---

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `DATABASE_URL` | `postgresql+asyncpg://postgres:postgres@localhost/symphony` | PostgreSQL async connection string |
| `ANTHROPIC_API_KEY` | — | **Required** for LangGraph agent nodes and Slack bot |
| `SLACK_BOT_TOKEN` | — | Slack bot token (`xoxb-...`) for Socket Mode |
| `SLACK_APP_TOKEN` | — | Slack app-level token (`xapp-...`) for Socket Mode |
| `SLACK_REPORT_CHANNEL` | `data-reports` | Default Slack channel for pipeline reports. Can be overridden per workflow via `tool_config`. |
| `DATASET_DIR` | — | Default dataset directory for pipeline tools. Can be overridden per workflow via `tool_config`. |
| `LANGCHAIN_TRACING_V2` | `false` | Set to `true` to enable LangSmith tracing |
| `LANGCHAIN_API_KEY` | — | LangSmith API key |
| `LANGCHAIN_PROJECT` | `symphony` | LangSmith project name |
| `POSTGRES_PASSWORD` | `postgres` | Docker Compose only |
| `MESSAGE_LOG_LEVEL` | `MINIMAL` | Floor for message persistence: `MINIMAL` (agent role only), `STANDARD` (user + agent), `VERBOSE` (all roles). Per-agent setting can only raise above this floor. |
| `MCP_API_KEY` | — | API key for the MCP server. Required to activate the `/mcp` endpoint. If absent, the endpoint returns 503. |
| `VOYAGE_API_KEY` | — | **Required** for knowledge base embedding (VoyageAI voyage-3-lite). |

---

## Workflow Templates

### Template 1 — Data Ingestion Pipeline

```
Start → Scan CSV → File Found? (condition)
  [false] → End
  [true]  → Data Quality → Ingest to DB → Data Profile → Report Agent → Publish Report → End
```

**Required env vars:** `ANTHROPIC_API_KEY`

**Configurable via `tool_config`** (pre-populated on instantiation): `dataset_dir` (all pipeline tools), `slack_channel` (Publish Report)

### Template 2 — SRE Job Summary

```
Start → Collect Job Stats → SRE Report Agent → Post to Slack → End
```

**Required env vars:** `SLACK_BOT_TOKEN` (bot must be invited to `#job-summary`)

**Configurable via `tool_config`**: `slack_channel` (Post to Slack), pre-set to `job-summary`

### Template 3 — Portfolio Recommendation

**Configurable via `tool_config`**: `catalogue_path` (Product Universe Filter), `slack_channel` (RM Alert Publisher), pre-set to `portfolio-reco`

---

## Slack Integration

Vageesh includes a Socket Mode Slack bot that lets you chat with agents directly from Slack.

**Setup:**
1. Create a Slack app at [api.slack.com/apps](https://api.slack.com/apps) with **Socket Mode** enabled
2. Add bot scopes: `chat:write`, `im:history`, `app_mentions:read`
3. Subscribe to events: `message.im`, `app_mention`
4. Set `SLACK_BOT_TOKEN` and `SLACK_APP_TOKEN` in `.env`
5. In the Vageesh UI (**Agent Configuration → Channels**), add `slack` to the target agent

If tokens are not set, the bot silently disables itself and the rest of Vageesh runs normally.

---

## Observability — LangSmith Tracing

1. Sign up at [smith.langchain.com](https://smith.langchain.com) and create a project named `symphony`
2. Generate an API key under **Settings → API Keys**
3. Add to `vageesh-back-end/.env`:
   ```
   LANGCHAIN_TRACING_V2=true
   LANGCHAIN_API_KEY=lsv2_pt_...
   LANGCHAIN_PROJECT=symphony
   ```
4. Restart the backend — all subsequent LLM calls are traced automatically

---

## API Overview

All endpoints under `/api/v1`. Auth is a stub — any non-empty Bearer token is accepted.

| Resource | Endpoints |
|---|---|
| Agents | GET/POST `/agents`, GET/PUT/DELETE `/agents/{id}` |
| Agent Memory | GET/POST `/agents/{id}/memory`, GET/DELETE `/agents/{id}/memory/{key}` |
| Agent Schedules | GET/POST `/agents/{id}/schedules`, PUT/DELETE `/agents/{id}/schedules/{schedule_id}` |
| Agent Skills | PUT `/agents/{id}/skills` |
| Interaction Rules | PUT `/agents/{id}/interaction-rules` |
| Guardrails | PUT `/agents/{id}/guardrails` |
| Workflows | GET/POST `/workflows`, GET/PUT/DELETE `/workflows/{id}` |
| Workflow Runs | POST `/workflows/{id}/run`, GET `/workflows/{id}/runs`, GET `/workflows/{id}/runs/{run_id}` |
| Templates | GET `/templates`, POST `/templates/{id}/instantiate` |
| Tools | GET `/tools/params` |
| Knowledge Base | GET/POST `/knowledge`, POST `/knowledge/upload`, DELETE `/knowledge/{id}`, POST `/knowledge/search` |
| MCP | GET `/mcp/tools`, GET `/mcp/audit-log` |
| Messages | GET/POST `/messages`, GET/DELETE `/messages/{id}` |
| Logs | GET/POST `/logs`, GET `/logs/{id}` |
| WebSocket | `ws://localhost:8000/ws/workflows/{id}/runs/{run_id}?token=<token>` |

---

## References

- [FastAPI documentation](https://fastapi.tiangolo.com/tutorial/)
- [SQLAlchemy async](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)
- [Alembic documentation](https://alembic.sqlalchemy.org/)
- [Vite documentation](https://vite.dev/guide/)
- [LangSmith documentation](https://docs.smith.langchain.com/)
- [LangGraph documentation](https://langchain-ai.github.io/langgraph/)
- [Slack Bolt / Socket Mode](https://slack.dev/bolt-python/concepts)

---

Licensed under the Apache License 2.0. Copyright 2026 Ajay Shriwastava.
