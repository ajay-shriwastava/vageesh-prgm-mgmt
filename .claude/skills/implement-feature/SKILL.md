---
name: implement-feature
description: >
  Translates a feature name and description into a full-stack implementation by orchestrating
  a team of sub-agents: a tech lead who owns technical design and coordination, a frontend
  developer who builds the UI in React + TypeScript served by Vite, and a backend developer
  who builds the API with FastAPI + LangGraph + PostgreSQL.
  Use this skill whenever a user says "implement a feature", "build a feature", "add feature X",
  "develop feature Y", or provides a feature name and description and wants working code generated.
  Trigger even for partial descriptions — the skill will ask for missing details before proceeding.
---

# implement-feature

This skill takes a **feature name** and **feature description** from the user and produces
working frontend and backend code by running a coordinated multi-agent pipeline.

## Project Context

This skill targets an **AI Agent Orchestration Platform** — a system where users create,
configure, and connect AI agents into collaborative workflows. Agents run on a real runtime,
execute real tools, communicate with each other asynchronously, and can be reached through
external messaging channels (Telegram / Slack / WhatsApp). The platform includes a web UI
for managing everything visually.

**Fixed tech stack — do not deviate:**

| Layer | Technology |
|---|---|
| Frontend | **React 19 + TypeScript**, bundled and served by **Vite** |
| Backend | **FastAPI** (Python) |
| AI / Agent runtime | **LangGraph** (integrated into FastAPI services) |
| Database | **PostgreSQL** (accessed via SQLAlchemy async ORM) |
| Messaging channels | Telegram Bot API / Slack Bolt / Twilio WhatsApp |
| Real-time | WebSockets via FastAPI `WebSocket` endpoints |

---

## Pipeline

```
User Input
    │
    ▼
[Claude – Orchestrator]
  • Clarify inputs
  • Translate to TechRequirementsBlock
    │
    ▼
[vageesh-tech-lead]
  • Designs APIContract (REST + WS endpoints, Pydantic schemas, DB models)
  • Considers LangGraph graph design if the feature involves agent execution
  • Spawns and coordinates developer agents in parallel
    │
    ├──────────────────────────────────────────┐
    ▼                                          ▼
[vageesh-frontend-developer]      [vageesh-backend-developer]
  • React 19 + TypeScript pages/        • FastAPI routers + Pydantic schemas
    components (.tsx)                   • LangGraph graphs / nodes / edges
  • Vite project structure              • SQLAlchemy async models + Alembic migrations
  • apiFetch API client (api.ts)        • WebSocket broadcast logic (where needed)
  • WebSocket client (where needed)
    │                                          │
    └──────────────────┬────────────────────────┘
                       ▼
          [vageesh-tech-lead – review]
            • Validates API integration
            • Checks LangGraph ↔ FastAPI wiring
            • Returns final merged deliverable
```

---

## Step 1 — Gather & Validate Inputs

The user must supply:

| Parameter | Description |
|---|---|
| `feature_name` | Short identifier, e.g. "Agent Workflow Builder" |
| `feature_description` | Plain-language description of what the feature does and why |

If either is missing or too vague to proceed, ask a single focused question before continuing.

Also confirm or infer:
- Does this feature involve **agent execution** (LangGraph graph)? If yes, what is the
  graph's high-level flow (nodes, conditions, loops)?
- Does this feature require **real-time updates** (WebSocket)? e.g., live logs, inter-agent
  messages, token/cost tracking.
- Does this feature involve a **messaging channel** (Telegram/Slack/WhatsApp)?
- Does this feature require **authentication**? Default: JWT bearer token via FastAPI
  `Depends(get_current_user)`.

---

## Step 2 — Translate to Technical Requirements

Read `references/requirements-template.md` for the exact format.

Produce a **TechRequirementsBlock** covering:
- Feature scope and acceptance criteria
- User roles / permissions
- Data entities (PostgreSQL tables via SQLAlchemy)
- API surface: REST endpoints + WebSocket channels (if any)
- LangGraph graph design (if applicable): nodes, state schema, edges/conditions
- Frontend screens / components
- Non-functional requirements

---

## Step 3 — Spawn vageesh-tech-lead

Invoke the **`vageesh-tech-lead`** sub-agent directly. Do not look for a local agent
file — the sub-agent is pre-configured externally.

Pass it:
1. The **TechRequirementsBlock** from Step 2.
2. The fixed tech stack table from the "Project Context" section above.
3. The instruction: *"Design the APIContract, then coordinate vageesh-frontend-developer
   and vageesh-backend-developer in parallel to implement the feature. Validate
   integration consistency before returning the merged deliverable."*

---

## Step 4 — Present Deliverables

Once `vageesh-tech-lead` returns, present:

1. **APIContract** — Pydantic schemas, endpoint specs, WebSocket message envelopes.
2. **Backend files** — FastAPI routers, LangGraph graphs, SQLAlchemy models, Alembic migration.
3. **Frontend files** — HTML pages, CSS, JS modules, Vite config.
4. **Integration notes** — env vars, DB setup, `alembic upgrade head`, how to run.

Summarise what was built in 3–5 sentences, then list all files with one-line descriptions.

---

## Notes for the Orchestrator

- The TechRequirementsBlock is the single source of truth passed to all agents.
- LangGraph state schemas are part of the APIContract — the tech lead defines them; the
  backend developer implements them exactly.
- If a feature needs WebSockets, the tech lead must define the message envelope format
  so both sides agree before coding begins.
- Do not invent business logic not described by the user. Note assumptions explicitly.
- Generated code must be production-quality: typed Python (Pydantic + type hints),
  idiomatic vanilla JS (ES modules, async/await), with error handling throughout.
