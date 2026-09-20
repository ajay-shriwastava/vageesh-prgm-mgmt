# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is the program management repository for **Vageesh** — an Agentic AI Orchestration Platform. It serves as the planning, documentation, and architecture hub for the Vageesh system, which consists of two sibling repositories:

- `vageesh-front-end` — React 19 + TypeScript SPA (Vite dev server)
- `vageesh-back-end` — FastAPI backend running LangGraph agents

## Architecture

Vageesh is a multi-agent AI platform. The tech stack decisions recorded here:

- **Frontend**: React 19 + TypeScript + React Router v7 + Vite; lazy-loaded routes, custom hooks, centralized config
- **Backend**: FastAPI + LangGraph agents, running in a Python virtualenv named `symphony`
- **Agent system**: Agents are defined with a structured template covering system prompt, responsibilities, decision framework, quality checks, output format, edge cases, and persistent memory (`Memory.md`)

## Development Commands

### Frontend (`vageesh-front-end`)
```bash
npm run dev        # Start Vite dev server at http://localhost:5173
npm test           # Run Vitest tests
npm run lint       # ESLint
# Quit: Ctrl+C
```

### Backend (`vageesh-back-end`)
```bash
mkvirtualenv symphony   # One-time setup
workon symphony
pip install fastapi uvicorn

workon symphony
fastapi dev              # Start server at http://127.0.0.1:8000
                         # API docs at http://127.0.0.1:8000/docs
# Quit: Ctrl+C
```

## Agent Definition Structure

When designing or documenting agents for Vageesh, use this template (from `doc/AgentDefinition.md`):

```
- Name:
- Description:
- Model: (Sonnet / Haiku / Inherit from Parent)
- Memory:

## Project Context
## Your responsibilities
## Decision Framework
## Quality Checks
## Output Format
## Edge cases to handle
## Persistent Agent Memory
## Memory.md
```



