# Vageesh Tech Lead Memory

## Architecture Decisions

- Sub-agents cannot be invoked as nested Claude Code processes (CLAUDECODE env var blocks nesting). Tech Lead implements all code directly when nested invocation fails.
- Backend virtualenv: `symphony` (mkvirtualenv/workon)
- DB driver: `asyncpg` — DATABASE_URL format: `postgresql+asyncpg://user:pass@localhost/symphony`
- Auth: JWT stub in `app/dependencies.py` — `get_current_user` returns `{"id": "stub-user"}` for any bearer token. Token stored in `localStorage["symphony_token"]`, fallback `"dev-token"`.
- CORS: allow all origins in dev (fastapi CORSMiddleware).

## Backend Structure (vageesh-back-end)

```
app/
  main.py            — FastAPI app + router registration + CORS + MCP mount
  mcp_server.py      — FastMCP server with 8 tools + McpAuthMiddleware; mcp_asgi_app exported
  database.py        — async engine, AsyncSessionLocal, Base, get_db()
  dependencies.py    — get_current_user (JWT stub)
  models/            — SQLAlchemy ORM models, __init__.py imports all for Alembic
  schemas/           — Pydantic request/response schemas
  routers/           — APIRouter handlers (agents, agent_config, workflows, messages, logs, workflow_runs, mcp)
  ws_manager.py      — ConnectionManager: connect/disconnect/broadcast per run_id
  workflow_runner.py — WorkflowRunner.compile() + run_workflow() async function
alembic/
  env.py             — async run_migrations pattern, imports app.models
  versions/          — migration files
alembic.ini
requirements.txt
```

## Key Patterns

- Log ORM uses `metadata_` (column alias `metadata`) to avoid SQLAlchemy reserved name conflict. Router manually maps it to `LogOut`.
- AgentMemory upsert: PostgreSQL `INSERT ... ON CONFLICT (agent_id, key) DO UPDATE` via `sqlalchemy.dialects.postgresql.insert`.
- All list endpoints return `{ items, total, skip, limit }` envelope.
- Alembic: `0001_initial_schema.py` creates all 5 original tables. `0003` adds workflow.status + workflow_runs table. `0004` adds skills/interaction_rules/guardrails JSONB columns to agents + creates agent_schedules table. Latest migration is `0016_mcp_audit_log.py` — always check `alembic/versions/` before naming a new migration, as the spec-provided name may be stale.
- WebSocket routes need a SEPARATE APIRouter with no prefix (`ws_router`). Export it from the router module and register it in main.py separately. Do NOT put `@router.websocket` on a prefixed APIRouter.
- Background tasks needing DB access: use `asyncio.create_task()` with a new `async with AsyncSessionLocal() as bg_db` — never reuse the request-scoped `db` session after it closes.
- workflow_runs router exports two objects: `router` (prefix `/api/v1/workflows`) and `ws_router` (no prefix, WebSocket at `/ws/workflows/{wf_id}/runs/{run_id}`).
- MCP server: `mcp_asgi_app` is a `McpAuthMiddleware`-wrapped FastMCP ASGI app; mounted at `/mcp` in main.py via `app.mount()`. Auth middleware checks `X-MCP-API-Key` header against `MCP_API_KEY` env var; returns 503 if key not configured, 401 if wrong.
- MCP audit log writes are fire-and-forget: `asyncio.create_task(_audit(...))`. Failures swallowed with `logger.warning` so they never block tool responses.
- MCP tools never store raw content/file bytes in `params_summary` — only metadata (title, content_length, key, top_k, file_type, encoded_length, etc.).

## Frontend Structure (vageesh-front-end/src)

- React 19 + TypeScript SPA, React Router v7, Vite
- `js/api.ts` — all TypeScript interfaces + `apiFetch<T>()` + `WS_BASE`; one function per API endpoint
- `config.ts` — `PAGE_SIZE`, `MODEL_OPTIONS`, `CHANNELS`, `CHANNEL_LABELS`, `AUTH_TOKEN_KEY`, `DEV_TOKEN`, `TOAST_DURATION_MS`
- `App.tsx` — all routes lazy-loaded via `React.lazy()`, wrapped in `ErrorBoundary` + `Suspense`
- `pages/` — one `.tsx` file (or subfolder) per route: `Agents.tsx`, `Workflows.tsx`, `Messages.tsx`, `Logs.tsx`, `AgentConfig.tsx`, `McpServer.tsx`
- `pages/workflows/` — `WorkflowBuilder.tsx`, `NodeConfigPanel.tsx`, `TemplatesSection.tsx`, `graph-helpers.ts`
- `pages/agent-config/` — `MemoryTab.tsx`, `SchedulesTab.tsx`, `SkillsTab.tsx`, `InteractionRulesTab.tsx`, `GuardrailsTab.tsx`, `types.ts`
- `components/` — `Nav.tsx`, `Pagination.tsx`, `LoadingRows.tsx`, `ErrorBoundary.tsx`
- `hooks/useApiList.ts` — generic paginated list hook: `useApiList<T>(fetcher, limit)` → `{ items, total, skip, loading, setSkip, reload }`
- `context/ToastContext.tsx` — `ToastProvider` + `useToast()` hook
- `utils/truncate.ts` — `truncate(str, n)`
- JSX escaping prevents XSS by default; no `dangerouslySetInnerHTML`

## Completed Features

- **persistence-layer** (2026-05-27): Full CRUD for Agent, Workflow, Message, Log, AgentMemory. PostgreSQL + Alembic + async SQLAlchemy. Frontend tabular views with pagination, inline forms, toast errors.
- **visual-workflow-builder** (2026-05-28): SVG canvas with drag-drop nodes (start/agent/condition/end), cubic bezier edges with arrowheads, config panel, LangGraph runner, WebSocket run streaming, run history panel. Migration 0003 adds `workflow.status` column + `workflow_runs` table.
- **agent-configuration** (2026-05-29): Expanded memory.html into 5-tab Agent Config page (Memory, Schedules, Skills, Interaction Rules, Guardrails). Migration 0004 adds 3 JSONB columns to agents + agent_schedules table. New router: `app/routers/agent_config.py`. Nav label "Memory" renamed to "Config". AgentOut schema extended with skills/interaction_rules/guardrails fields.
- **agent-to-agent-handoffs** (2026-05-31): No DB migration. `messages.role` Literal extended to include `"agent"`. `GET /api/v1/messages` gains `role` filter query param. `workflow_runner.py` `_make_agent_node()` persists agent output to `messages` table (role=agent, session_id=run_id, agent_id=source agent) after each agent node completes — fire-and-forget with exception swallowing. `messages.html` gains "Agent Handoffs" tab with styled cards (agent name resolved from agent map, session truncated, timestamp). `api.js` `getMessages` updated to pass `role` param.
- **full-message-capture** (2026-08-04): Migration 0011 adds `destination_type VARCHAR(50)` + `destination_ref TEXT` (both nullable) to messages. `MessageCreate` gains optional `destination_type: Optional[Literal[...]]` + `destination_ref`. `MessageOut` gains same fields. `_make_agent_node()` gains `next_node_type` + `next_agent_id` params; old single-record agent handoff block replaced with full 4-record block (system/user/assistant-or-tool/agent) in one AsyncSessionLocal. `WorkflowRunner.compile()` computes next-node type from `out_edges` before calling `_make_agent_node`. Frontend: `Message` interface gains `destination_type` + `destination_ref`. Messages page AllMessages tab gains Dest Type (badge) + Dest Ref (truncated) columns; colSpan updated 6→8. DEST_BADGE map added.
- **message-log-level** (2026-08-04): Migration 0012 adds `message_log_level VARCHAR(20)` nullable to agents. `AgentCreate`, `AgentUpdate`, `AgentOut` schemas gain `message_log_level: Optional[Literal['MINIMAL','STANDARD','VERBOSE']] = None`. `workflow_runner.py` adds `_LEVEL_RANK` dict + `_effective_log_level(agent_obj)` (reads `MESSAGE_LOG_LEVEL` env var as floor, returns max(floor, agent level)). Message persistence block in `_make_agent_node` gated by effective level: MINIMAL→role=agent only; STANDARD→user+agent; VERBOSE→all roles. Frontend: `Agent` interface + `AgentCreatePayload` gain `message_log_level` field. `InteractionRulesTab.tsx` gains a Message Log Level dropdown (Inherit/MINIMAL/STANDARD/VERBOSE); save calls `updateInteractionRules` then `updateAgent` sequentially.
- **workflow-tool-config** (2026-08-11): Migration 0013 adds `tool_config JSONB NOT NULL DEFAULT '{}'` to workflows. Two-level config hierarchy: `.env` (infra/secrets) → `workflow.tool_config` (operational params per workflow instance). `TOOL_PARAMS` dict in `app/tools/__init__.py` is the single source of truth for configurable params, served via `GET /api/v1/tools/params`. Pipeline tools receive config via state merge in `_make_tool_node`. LLM tools receive config via `ContextVar[dict]` in `app/tools/tool_context.py`, set/reset around `react_agent.ainvoke()` with token pattern. Templates pre-populate `tool_config_defaults` on instantiation. New route: `/config/workflows/:id` (WorkflowConfig page — flat audit view). Workflow Builder NodeConfigPanel shows param fields inline when a node is selected. `WorkflowRunner.compile()` and `run_workflow()` both accept `tool_config: dict | None = None`.

- **knowledge-base-file-upload** (2026-09-17): No DB migration. New endpoint `POST /api/v1/knowledge/upload` added to `app/routers/knowledge.py` (above the `/{entry_id}` DELETE route to avoid path conflicts). Accepts `multipart/form-data` with `file: UploadFile` + `title: str = Form(...)`. PDF parsed with `pypdf.PdfReader(io.BytesIO(raw_bytes))`; .txt decoded as UTF-8. Both paths feed into existing `chunk_text` → `embed` → raw SQL INSERT pipeline. 400 on unsupported ext or empty extraction; 500 on parse exception. `pypdf>=4.0.0` added to `requirements.txt` (import is local inside the endpoint to keep it optional). Frontend: `uploadKnowledge(file, title)` added to `api.ts` using raw `fetch()` with `FormData` (no Content-Type header — browser sets multipart boundary). `KnowledgeBase.tsx` gains "Upload File" section above existing "Ingest Document" form; file input has `accept=".pdf,.txt"` and an associated `<label>`; Upload button disabled while uploading or when file/title missing; `fileInputRef.current.value = ""` used to clear the file input after success.
- **mcp-server** (2026-09-18): Migration `0016_mcp_audit_log.py` adds `mcp_audit_log` table (caller_id VARCHAR(255), tool_name VARCHAR(100), agent_id UUID nullable no-FK, params_summary JSONB, result_summary TEXT, created_at TIMESTAMPTZ). `app/mcp_server.py` — FastMCP server with 8 tools (4 memory + 4 knowledge), `McpAuthMiddleware` wraps ASGI app, mounted at `/mcp`. `app/routers/mcp.py` — `GET /api/v1/mcp/tools` (static list) + `GET /api/v1/mcp/audit-log` (paginated, caller_id filter). `app/slack_bot.py` enhanced: `_run_agent_direct` accepts `slack_user_id`, calls `list_memory` + `search_knowledge` concurrently via FastMCP Client before LLM, parses `[REMEMBER key: value]` post-response + calls `set_memory`, graceful degradation when `MCP_API_KEY` absent. `fastmcp>=2.0.0` added to `requirements.txt`. Frontend: `McpServer.tsx` at `/mcp` (Server Info card with Claude Desktop config copy button, Available Tools table grouped by category, Audit Log table with caller_id filter + pagination). `getMcpTools` + `getMcpAuditLog` added to `api.ts`. Route at `/mcp` + nav link ("MCP Server", `ti-server`) registered.

## API Naming Convention

- Base path: `/api/v1`
- WebSocket base path: `/ws`
- MCP endpoint: `/mcp` (Streamable HTTP — FastMCP ASGI mount, NOT under `/api/v1`)
- All REST endpoints require `Authorization: Bearer <token>`
- WebSocket auth: `?token=<jwt>` query param
- MCP auth: `X-MCP-API-Key` header
- 404 on missing resource, 422 on Pydantic validation failure, 204 on successful delete

## Dual-Wrapper Tool Pattern

Every tool in `app/tools/` can be exposed as **both** a Pipeline Tool and an LLM Tool simultaneously. The two wrappers always remain separate because their contracts differ:

| | LLM Tool (`@tool`) | Pipeline Tool (`run`) |
|---|---|---|
| Input | typed args or none | `state: dict` |
| Output | JSON string for LLM to read | updated `state: dict` for LangGraph |
| Caller | agent at inference time | workflow runner as a fixed graph node |

**When to extract shared logic:**
- **Same file, private helper** (e.g. `_find_latest_csv()`) — when both wrappers in the same file duplicate logic. This is the common case.
- **Separate module** (e.g. `app/tools/core/`) — only when the logic is shared across multiple tool files.

The wrappers never collapse into one. Even with a shared core, `scan_csv()` returns JSON and `run()` merges into state.

**Example structure:**
```python
async def _core_logic() -> dict:          # shared, no LangChain/LangGraph coupling
    ...

@tool
async def scan_csv() -> str:              # LLM Tool — JSON out
    return json.dumps(await _core_logic())

async def run(state: dict) -> dict:       # Pipeline Tool — state dict out
    result = await _core_logic()
    return {**state, "condition_result": result["found"], ...}
```

## LangGraph Pattern

- `WorkflowRunner.compile(graph_definition, agents_map, run_id)` returns an uncompiled `StateGraph`
- Call `.compile()` on the returned graph before `.ainvoke(state)`
- State dict keys: `messages` (List[str]), `current_output` (Any), `condition_result` (bool), `run_id` (str)
- Condition nodes use `add_conditional_edges` with a router function reading `state["condition_result"]`
- Agent nodes call `ChatAnthropic(model="claude-haiku-4-5")` with the agent's system_prompt
- `ANTHROPIC_API_KEY` env var required for agent node execution
