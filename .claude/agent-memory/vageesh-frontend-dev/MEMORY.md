# Vageesh Frontend Dev Memory

## Project Structure (vageesh-front-end/src)

- React 19 + TypeScript SPA, React Router v7, Vite
- `js/api.ts` — all TypeScript interfaces + `apiFetch<T>()` + `WS_BASE`; one function per API endpoint
- `config.ts` — `PAGE_SIZE`, `MODEL_OPTIONS`, `CHANNELS`, `CHANNEL_LABELS`, `AUTH_TOKEN_KEY`, `DEV_TOKEN`, `TOAST_DURATION_MS`, `AGENT_DROPDOWN_LIMIT`
- `App.tsx` — all routes lazy-loaded via `React.lazy()`, wrapped in `ErrorBoundary` + `Suspense`
- `pages/` — one `.tsx` file (or subfolder) per route
- `pages/workflows/` — `WorkflowBuilder.tsx`, `NodeConfigPanel.tsx`, `TemplatesSection.tsx`, `graph-helpers.ts`
- `pages/agent-config/` — `MemoryTab.tsx`, `SchedulesTab.tsx`, `SkillsTab.tsx`, `InteractionRulesTab.tsx`, `GuardrailsTab.tsx`, `types.ts`
- `components/` — `Nav.tsx`, `Pagination.tsx`, `LoadingRows.tsx`, `ErrorBoundary.tsx`
- `hooks/useApiList.ts` — generic paginated list hook: `useApiList<T>(fetcher, limit)` → `{ items, total, skip, loading, setSkip, reload }`
- `context/ToastContext.tsx` — `ToastProvider` + `useToast()` hook
- `css/symphony.css` — single global stylesheet; all component styles live here with BEM-like prefixes

## Key Conventions

- All API calls in `src/js/api.ts` — never fetch directly in components
- CSS: no CSS modules; use class prefix per page/component (e.g. `.wfc-` for WorkflowConfig, `.nb-` for node panel)
- No `dangerouslySetInnerHTML` — JSX escaping prevents XSS by default
- `useToast()` for all error and success messages
- Routes are lazy-loaded; add `React.lazy()` + `<Route>` in `App.tsx` for every new page

## Workflow Tool Configuration (workflow-tool-config feature)

`workflow.tool_config` (shape: `Record<string, Record<string, string>>`) holds per-workflow operational params.

### State management in WorkflowBuilder
- `toolConfig` state initialized from `workflow.tool_config ?? {}`
- `toolParams` state fetched once on mount via `getToolParams()` → `GET /api/v1/tools/params`
- `handleUpdateToolConfig(toolName, paramName, value)` updates nested state immutably
- `handleSave()` includes `tool_config: toolConfig` in the PATCH payload

### NodeConfigPanel props for tool config
```tsx
toolConfig: Record<string, Record<string, string>>
onUpdateToolConfig: (toolName: string, paramName: string, value: string) => void
toolParams: Record<string, ToolParam[]>
agentsList: Agent[]
```
- `type="tool"` node: renders param inputs using `toolParams[node.tool_name]`
- `type="agent"` node: finds agent by `node.agent_id`, iterates `agent.tools`, renders params for each

### WorkflowConfig page (`/config/workflows/:workflowId`)
- Flat audit view — all configurable params for a workflow in one place
- Two sections: Pipeline Tools (nodes where `type === "tool"`) and LLM Tools (agent nodes → agent lookup → agent.tools)
- CSS prefix: `.wfc-*`; param rows use `display: grid; grid-template-columns: 160px 1fr` — label and input on the same line
- Input width constrained to `320px max-width: 100%` (not full-width)
- Accessible from Config page via workflow dropdown → navigate to `/config/workflows/:id`

### API additions (api.ts)
- `ToolParam` interface: `{ name, label, type, required }`
- `getToolParams()` → `GET /api/v1/tools/params` → `Record<string, ToolParam[]>`
- `getWorkflow(id)` → `GET /api/v1/workflows/:id` → `Workflow`
- `Workflow.tool_config: Record<string, Record<string, string>>`
- `WorkflowUpdatePayload.tool_config?: Record<string, Record<string, string>>`
