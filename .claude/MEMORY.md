# Vageesh Program Management — Memory

## Project Structure
- `vageesh-prgm-mgmt` — umbrella/planning repo (this repo)
- `vageesh-front-end` — React 19 + TypeScript + Vite SPA at `~/tech/vageesh/vageesh-front-end`
- `vageesh-back-end` — FastAPI + LangGraph + PostgreSQL backend at `~/tech/vageesh/vageesh-back-end`

## .claude Directory (read at startup)
- `.claude/agents/vageesh-tech-lead.md` — Tech lead agent: decomposes features, defines API contracts, delegates to sub-agents. Works in `vageesh-prgm-mgmt`. Does NOT write code itself.
- `.claude/agents/vageesh-frontend-dev.md` — Frontend agent: React 19 + TypeScript, Vite, React Router v7, custom hooks. Works in `vageesh-front-end`.
- `.claude/agents/vageesh-backend-dev.md` — Backend agent: FastAPI, LangGraph, PostgreSQL, async SQLAlchemy. Works in `vageesh-back-end`. Layers: router → service → repository.
- `.claude/skills/implement-feature/SKILL.md` — Multi-agent pipeline skill: Orchestrator → tech-lead → (frontend-dev + backend-dev in parallel) → tech-lead review.
- `.claude/skills/implement-feature/requirements-template.md` — TechRequirementsBlock template used in Step 2 of the implement-feature skill.

## Agent Memory Directories
Each agent has persistent memory at `.claude/agent-memory/<agent-name>/` inside `vageesh-prgm-mgmt`.
- `.claude/agent-memory/vageesh-tech-lead/`
- `.claude/agent-memory/vageesh-frontend-dev/`
- `.claude/agent-memory/vageesh-backend-dev/`

## Key Conventions
- Frontend: React 19 + TypeScript, JSX, React Router v7, Vite; all API calls in `src/js/api.ts`; pages in `src/pages/`; shared hooks in `src/hooks/`; constants in `src/config.ts`
- Backend: async FastAPI, Pydantic models, parameterized SQL, virtualenv named `symphony`
- API base URL: `/api/v1`
- Auth: JWT bearer token via `Depends(get_current_user)`
- Agents never write code for each other's layers
- "Subject Matter Not Known" = the signal when something is unclear (never guess)

## Product Principles
- **Prompt engineering is out of scope for Vageesh.** No CoT, Few-Shot, or other prompt engineering techniques as platform features. Vageesh's responsibility is orchestration (workflows, tools, channels, memory, guardrails). The system prompt textarea is the right boundary — agent authors handle their own prompts.

## implement-feature Skill Flow
1. Gather inputs (feature_name, feature_description)
2. Produce TechRequirementsBlock
3. Spawn vageesh-tech-lead with the block
4. Tech lead designs APIContract, spawns frontend-dev + backend-dev in parallel
5. Tech lead validates integration, returns merged deliverable
