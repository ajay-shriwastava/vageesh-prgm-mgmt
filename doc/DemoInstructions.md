# Vageesh — Demo Instructions

---

## Setup

### Option A — Docker (recommended, single command)

```bash
cd vageesh-back-end
cp .env.example .env
# Edit .env — add ANTHROPIC_API_KEY (required) and Slack tokens (optional)
docker compose up --build
```

- **Frontend**: http://localhost
- **API docs**: http://localhost:8000/docs

Postgres is created, all migrations run, and all services start automatically.
Subsequent runs: `docker compose up` (no `--build` needed).

---

### Option B — Local Dev (two terminals)

**Prerequisites:** PostgreSQL running locally.

```bash
# One-time database setup
psql -U postgres -c "CREATE DATABASE symphony;"

# Terminal 1 — Backend
cd vageesh-back-end && workon symphony
alembic upgrade head
fastapi dev app/main.py        # → http://127.0.0.1:8000/docs

# Terminal 2 — Frontend
cd vageesh-front-end && npm install && npm run dev
# → http://localhost:5173/src/html/agents.html
```

> In local dev, set `BASE_URL = "http://localhost:8000"` in `vageesh-front-end/src/js/api.js`.

---

## URLs at a Glance

| | Docker | Local Dev |
|---|---|---|
| Frontend | http://localhost | http://localhost:5173 |
| Agents | http://localhost/agents | http://localhost:5173/agents |
| Workflows | http://localhost/workflows | http://localhost:5173/workflows |
| Messages | http://localhost/messages | http://localhost:5173/messages |
| Logs | http://localhost/logs | http://localhost:5173/logs |
| Agent Config | http://localhost/agent-config | http://localhost:5173/agent-config |
| Knowledge Base | http://localhost/knowledge | http://localhost:5173/knowledge |
| MCP Server | http://localhost/mcp | http://localhost:5173/mcp |
| API docs | http://localhost:8000/docs | http://127.0.0.1:8000/docs |

---

## pgAdmin — Connecting to the Vageesh Database

pgAdmin is the recommended GUI for inspecting database state during demos.

### Install (one-time)

```bash
brew install --cask pgadmin4
```

### Register the server (one-time)

1. Open pgAdmin → right-click **Servers → Register → Server**
2. **General tab** — Name: `Vageesh Local`
3. **Connection tab**:
   - Host: `localhost`
   - Port: `5432`
   - Maintenance database: `postgres`
   - Username: `postgres`
   - Password: `postgres`
4. Click **Save**

### Navigate to the Vageesh database

```
Servers → Vageesh Local → Databases → symphony → Schemas → public → Tables
```

Right-click any table → **View/Edit Data → All Rows**.

### Useful queries

```sql
-- All tables
SELECT table_name FROM information_schema.tables WHERE table_schema = 'public';

-- Agents
SELECT id, name, model, channels, status FROM agents;

-- Workflows
SELECT id, name, status, graph_definition FROM workflows;

-- Workflow runs
SELECT id, workflow_id, status, started_at, finished_at FROM workflow_runs ORDER BY started_at DESC;

-- Recent messages (including Slack)
SELECT id, agent_id, session_id, role, LEFT(content, 80) AS content, created_at
FROM messages ORDER BY created_at DESC LIMIT 20;

-- Logs
SELECT id, level, message, agent_id, created_at FROM logs ORDER BY created_at DESC LIMIT 20;

-- MCP audit log
SELECT caller_id, tool_name, params_summary, result_summary, created_at FROM mcp_audit_log ORDER BY created_at DESC LIMIT 20;

-- Reset a stuck workflow run
UPDATE workflow_runs SET status = 'failed', error = 'manually reset' WHERE status = 'running';
```

---

## Visual Workflow Builder — Testing

### Step 1 — Create a workflow
1. Open the Workflows page
2. Click **+ New Workflow**, enter a name, click **Create**

### Step 2 — Open the visual builder
Click **Edit**. The SVG canvas builder opens with:
- **Left palette** — Start, Agent, Condition, End node chips
- **Centre canvas** — empty SVG grid
- **Right config panel** — node properties

### Step 3 — Build a simple Start → End graph
1. Drag **Start** from the palette onto the canvas
2. Drag **End** to the right of Start
3. Hover Start until the purple output port dot appears on its right edge
4. Drag from the output port to End's input port (left edge) — a bezier arrow connects them

### Step 4 — Save and Run
- Click **Save** — "Workflow saved" toast confirms persistence
- Click **Run** — the run log panel opens and streams live WebSocket events:

```
→ node_enter: start
✓ node_complete: start
⟶ edge_traverse: e1
→ node_enter: end
✓ node_complete: end
✓ Run completed
```

### Step 5 — Check run history
The **Run History** section at the bottom lists the run with a `completed` badge and timestamps.

### Step 6 — Test a condition feedback loop
1. Drag **Start → Agent → Condition → End** onto the canvas
2. Click the **Agent** node → pick an agent from the config panel dropdown
3. Click the **Condition** node → set true/false labels
4. Draw edges: Condition `true` port → End, Condition `false` port → Agent (feedback loop)
5. Optionally set **Max Loops** in the toolbar (default: 20)
6. Save and Run — the loop iterates up to `max_loops` times then exits automatically

### Step 7 — Verify in pgAdmin

```sql
SELECT * FROM workflows;      -- graph_definition JSONB, status = 'draft'
SELECT * FROM workflow_runs;  -- status, output JSONB, started_at, finished_at
```

---

## Tests

195 tests (integration + unit + eval), all passing.

```bash
# One-time: create test database
psql -U postgres -c "CREATE DATABASE symphony_test;"

# Run all tests
workon symphony && pytest

# With coverage
pytest --cov=app --cov-report=term-missing
```

---

## Slack Integration — Testing

### Prerequisites
- A Slack workspace where you can install apps
- `SLACK_BOT_TOKEN` and `SLACK_APP_TOKEN` set in `.env`
- Backend running

### Step 1 — Configure an agent for Slack
1. Open the Agent Configuration page (`/agent-config`)
2. Select an agent from the dropdown
3. Under **Channels**, add `slack`
4. Save

### Step 2 — Verify bot connected
Backend startup logs should show:

```
INFO  Slack bot connected via Socket Mode.
```

### Step 3 — Send a direct message
In Slack, open a DM with your Vageesh bot and send any message. The bot replies using the configured agent's model and system prompt.

### Step 4 — Test @mention
In any channel where the bot is invited, type `@VageeshBot Hello`. The bot strips the mention prefix and replies.

### Step 5 — Verify message persistence
Messages are stored in the `messages` table and visible in the UI (messages.html) or via:

```sql
SELECT * FROM messages ORDER BY created_at DESC LIMIT 10;
```

### Known behaviour
- Routes to the **first** agent with `"slack"` in its channels — configure only one agent per workspace
- Slack channel ID is used as `session_id` — each channel maintains its own conversation context
- Falls back to `claude-haiku-4-5-20251001` with a generic prompt if no agent is configured for Slack
- Bot silently disables itself if tokens are missing or placeholder values
- When `MCP_API_KEY` is set, each reply is automatically enriched with agent memory and relevant knowledge chunks before the LLM is called — no extra configuration needed
- LLM replies containing `[REMEMBER key: value]` are automatically persisted to agent memory and stripped before sending to Slack

---

## MCP Server — Testing

### Prerequisites
- `MCP_API_KEY` set in `.env` (choose any strong random string)
- Backend running (`alembic upgrade head` run at least once to create the `mcp_audit_log` table)

### Step 1 — Open the MCP Server page
Navigate to `/mcp` in the UI. You will see:
- **Server Info** — the endpoint URL and a one-click copy button for the Claude Desktop `mcpServers` config JSON
- **Available Tools** — all 8 tools listed by category (memory / knowledge)
- **Audit Log** — paginated table of every MCP tool call with caller identity and timestamp

### Step 2 — Connect Claude Desktop
1. In the UI, click **Copy Config** in the Server Info card
2. Paste into your Claude Desktop `claude_desktop_config.json` under `mcpServers`
3. Restart Claude Desktop — Vageesh will appear as an MCP server in the tool list
4. Ask Claude Desktop: *"What do you know about [topic]?"* — it will call `search_knowledge`

### Step 3 — Test via curl
```bash
# List all memory for an agent
curl -X POST http://localhost:8000/mcp/ \
  -H "X-MCP-API-Key: <your-key>" \
  -H "Content-Type: application/json" \
  -d '{"method":"tools/call","params":{"name":"list_memory","arguments":{"agent_id":"<uuid>","caller_id":"curl:demo"}}}'

# Add a knowledge document
curl -X POST http://localhost:8000/mcp/ \
  -H "X-MCP-API-Key: <your-key>" \
  -H "Content-Type: application/json" \
  -d '{"method":"tools/call","params":{"name":"add_knowledge","arguments":{"title":"Test Doc","content":"Vageesh is an AI orchestration platform.","caller_id":"curl:demo"}}}'
```

### Step 4 — Verify audit log
```sql
SELECT caller_id, tool_name, params_summary, result_summary, created_at
FROM mcp_audit_log
ORDER BY created_at DESC LIMIT 20;
```

### Step 5 — Test context-aware Slack replies
With `MCP_API_KEY` set and an agent configured for Slack:
1. Add a memory entry via the Agent Config page (e.g. key: `preferred_currency`, value: `GBP`)
2. Upload a document on the Knowledge Base page
3. DM the Vageesh bot in Slack — the reply will be grounded in the agent memory and document content
4. Check the audit log: two new rows appear per message (`list_memory` + `search_knowledge`)

### Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `/mcp/` returns 503 | `MCP_API_KEY` not set | Add to `.env` and restart backend |
| `/mcp/` returns 401 | Wrong key in header | Check `X-MCP-API-Key` matches `.env` value |
| Slack replies not enriched | `MCP_API_KEY` missing | Set key — bot degrades gracefully without it |
| Migration error on startup | `mcp_audit_log` table missing | Run `alembic upgrade head` |

---

## LangSmith — Viewing Traces

### Prerequisites
- `LANGCHAIN_TRACING_V2=true`, `LANGCHAIN_API_KEY`, and `LANGCHAIN_PROJECT=symphony` set in `.env`
- Backend restarted after setting these values

### Steps

1. Go to [smith.langchain.com](https://smith.langchain.com) and sign in
2. Click **Projects** in the left sidebar → select **symphony**
3. Click the **Traces** tab — each row is one traced run (model, tokens, cost, latency)
4. Click any row to see the full call tree: each LangGraph node shows prompt, response, token breakdown

### Trigger traces

- **Workflow**: run any workflow with an Agent node
- **Slack**: send a DM or @mention to the Vageesh bot

### Filter traces

Use the **Filter** bar to narrow by status, model, latency, or date range.

---

## Data Ingestion Pipeline — Testing

### Prerequisites
- `DATASET_DIR` set in `.env` pointing to the `vageesh-prgm-mgmt/dataset` directory
- `SLACK_REPORT_CHANNEL` set in `.env` (e.g. `data-reports`) — bot must be invited to that channel
- `ANTHROPIC_API_KEY` set (the Report Agent node calls Claude)

### Step 1 — Instantiate the template
1. Open the Workflows page
2. In the **Templates** panel, find **Data Ingestion Pipeline**
3. Click **Use Template** — the workflow is created and its cron schedule (`* * * * *`) is registered automatically

### Step 2 — Drop a CSV into the input directory

```bash
ls $DATASET_DIR/          # should show: input/ output/ processed/ error/
cp $DATASET_DIR/input/real_estate.csv $DATASET_DIR/input/test_run.csv
```

A sample `real_estate.csv` is already included. The pipeline runs on a 1-minute cron.

### Step 3 — Watch backend logs

```
INFO  Tool 'csv_scanner' completed
INFO  Condition node: file_check — condition_result = True
INFO  Tool 'data_quality' completed
INFO  Tool 'db_ingestor' completed
INFO  Tool 'data_profiler' completed
INFO  Node 'report' completed — ↑NNN ↓NNN tokens, $0.XXXX
INFO  Tool 'report_publisher' completed
INFO  Workflow run <uuid> completed
```

### Step 4 — Verify output

```bash
ls $DATASET_DIR/output/     # ingested rows CSV + *_report_*.txt
ls $DATASET_DIR/processed/  # original CSV moved here
ls $DATASET_DIR/error/      # rejected rows (blanks/duplicates only)
```

### Step 5 — Verify in PostgreSQL

```sql
SELECT * FROM real_estate LIMIT 10;

SELECT id, status, started_at, finished_at, output->'usage' AS usage
FROM workflow_runs ORDER BY started_at DESC LIMIT 5;
```

### Step 6 — Check Slack
A formatted report summarising ingestion stats and data profile narrative appears in `#data-reports`.

### Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| No run triggered | Scheduler not started | Restart backend; check for `Scheduler started` log |
| `condition_result = False` | No CSV in input/ | Drop a file into `dataset/input/` |
| All rows in error/ | Data quality rejecting everything | Check CSV has at least some non-blank rows |
| Slack message missing | Token/channel misconfigured | Check `SLACK_BOT_TOKEN` and `SLACK_REPORT_CHANNEL` |

---

## SRE Job Summary — Testing

### Prerequisites
- Backend running
- `SLACK_BOT_TOKEN` set and bot invited to `#job-summary`
- At least a few workflow runs in the database

### Step 1 — Instantiate the template
1. Open the Workflows page → **Templates** panel → **SRE Job Summary** → **Use Template**

### Step 2 — Trigger manually (don't wait an hour)

```bash
# Get the workflow ID
curl http://localhost:8000/api/v1/workflows \
  -H "Authorization: Bearer test" | python3 -m json.tool | grep -A2 "SRE"

# Trigger a run
curl -X POST http://localhost:8000/api/v1/workflows/<workflow-id>/run \
  -H "Authorization: Bearer test" \
  -H "Content-Type: application/json" \
  -d '{}'
```

Or open the workflow in the UI and click **Run**.

### Step 3 — Check Slack `#job-summary`

```
*Vageesh Job Health Summary — Last 24h*
• Total runs: 8  |  Success rate: 87.5%
✅ Completed: 7  ❌ Failed: 1  🔄 Running: 0  ⏳ Pending: 0

*Per-workflow breakdown:*
✅ Data Ingestion Pipeline — 7 completed
❌ My Test Workflow — 1 failed  ⚠️ Needs attention
```

### Step 4 — Verify in pgAdmin

```sql
SELECT id, status, started_at, finished_at FROM workflow_runs ORDER BY started_at DESC LIMIT 5;
```

### Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| No Slack message | Bot not in #job-summary | `/invite @VageeshBot` in the channel |
| Empty stats | No workflow runs in DB | Run the Data Ingestion Pipeline a few times first |
| `SLACK_BOT_TOKEN` error | Token missing | Set in `.env` and restart |

---

## Known Behaviour

- Feedback loops exit after **max_loops** agent passes (default: 20, configurable per workflow in the builder toolbar) — `condition_result` is forced to `True` on the final iteration
- Agent nodes require `ANTHROPIC_API_KEY` to call Claude; Start / End / Condition nodes run without a key
- Any run stuck in `running` status can be reset:

```sql
UPDATE workflow_runs SET status = 'failed', error = 'manually reset' WHERE status = 'running';
```
