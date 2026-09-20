# Vageesh Backend Dev Memory

## Tool Architecture — Dual-Wrapper Pattern

Every file in `app/tools/` exposes the same logic as **both** a Pipeline Tool and an LLM Tool.
The two wrappers always remain separate — their contracts are fundamentally different.

### The three layers in each tool file

```python
async def _core_logic() -> dict:
    """Private. No LangChain or LangGraph coupling. Returns a plain dict."""
    ...

@tool
async def scan_csv() -> str:
    """LLM Tool. Docstring = LLM API spec. Always returns json.dumps(result)."""
    return json.dumps(await _core_logic())

async def run(state: dict) -> dict:
    """Pipeline Tool. Always custom — maps core result into specific state keys."""
    r = await _core_logic()
    return {**state, "condition_result": r["found"], "csv_file_path": r["file_path"], ...}
```

### Rules
- Extract `_core_logic()` whenever `scan_csv()` and `run()` duplicate logic in the same file.
- Create `app/tools/core/` only if logic is shared across multiple tool files.
- The LLM Tool wrapper is always trivial (`json.dumps`). The Pipeline Tool wrapper is always custom (state key mapping is tool-specific — cannot be genericized).
- The `@tool` docstring is the LLM's API contract — write it carefully, it affects agent behaviour.

### Registration (app/tools/__init__.py)
- `TOOL_REGISTRY` — keyed by `@tool` function name (e.g. `"scan_csv"`) → used by `create_react_agent`
- `PIPELINE_TOOLS` — keyed by tool file name (e.g. `"csv_scanner"`) → used by `_make_tool_node`
- `CHANNEL_TOOLS` — subset of `TOOL_REGISTRY` auto-injected when a channel is active on an agent
- `TOOL_PARAMS` — maps tool name → list of `{name, label, type, required}` dicts; served via `GET /api/v1/tools/params`; single source of truth for what's configurable per tool

### Per-Workflow Tool Configuration (tool_context.py)

`workflow.tool_config` (JSONB, shape `{tool_name: {param_name: value}}`) provides per-workflow overrides for operational params like `dataset_dir`, `slack_channel`, `catalogue_path`. Two-level hierarchy: `.env` (infra defaults) → `workflow.tool_config` (per-workflow override).

**LLM tools** — injected via `ContextVar[dict]` in `app/tools/tool_context.py`. The runner sets the var around `react_agent.ainvoke()` using the token pattern (set/reset in try/finally). The `@tool` function reads it directly — the param is never exposed as an LLM-visible argument:
```python
from app.tools.tool_context import tool_config as _tool_config_var

@tool
async def scan_csv() -> str:
    cfg = _tool_config_var.get().get("scan_csv", {})
    dataset_dir = cfg.get("dataset_dir") or DATASET_DIR
    ...
```

**Pipeline tools** — `_make_tool_node()` merges `tool_config[tool_name]` into state before calling `run()`. Merge priority: `tool_cfg (baseline) → node_params → state (wins)`. The `run()` function just reads `state.get("dataset_dir") or DATASET_DIR` — no ContextVar needed.

**Router** — `app/routers/tools.py` exposes `GET /api/v1/tools/params` returning `TOOL_PARAMS`.

### Why a generic Pipeline wrapper doesn't work
State merging is tool-specific. `csv_scanner` sets `csv_file_path`, `table_name`, `condition_result`.
`data_quality` sets `clean_csv_path`, `row_count`, `warnings`. No generic wrapper can know this.
Blindly spreading `{**state, **result}` sacrifices type safety and makes state shape unpredictable.

## Backend Structure

```
app/
  main.py            — FastAPI app + router registration + CORS
  database.py        — async engine, AsyncSessionLocal, Base, get_db()
  dependencies.py    — get_current_user (JWT stub)
  models/            — SQLAlchemy ORM models
  schemas/           — Pydantic request/response schemas
  routers/           — APIRouter handlers
  tools/             — Pipeline Tools (run) + LLM Tools (@tool) + __init__.py registries
  workflow_runner.py — WorkflowRunner.compile() + run_workflow()
alembic/versions/    — always check latest migration number before naming a new one
```

## Key Conventions
- Layers: router → service → repository
- All FastAPI handlers: async/await throughout
- All SQL: parameterized queries, never string interpolation
- DB sessions: `Depends(get_db)` in routes; `async with AsyncSessionLocal()` in background tasks
- Auth: JWT stub — `get_current_user` returns `{"id": "stub-user"}` for any bearer token
- API base: `/api/v1`; WebSocket base: `/ws`
- List endpoints return `{ items, total, skip, limit }` envelope
