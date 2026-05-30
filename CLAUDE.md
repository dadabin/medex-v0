# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MeDEX is a medical AI agent application. One project = one patient. The agent analyzes patient data, creates subtask plans, executes them, and generates medical analysis reports. Currently in design/prototyping phase — only `doc/` exists; backend and frontend code has not been written yet.

## Current State

The repo contains documentation and a UI mockup only. No backend or frontend code exists yet.

```
medex/
├── doc/
│   ├── implementation_plan.md    # Full technical spec: architecture, data model, API, tools, dev plan
│   ├── ui/
│   │   ├── index.html            # Interactive UI mockup (single-file, vanilla HTML/CSS/JS)
│   │   └── ui_design.md          # UI style guide (dark theme, colors, typography, components)
│   └── agentscope-v2/            # AgentScope v2 reference docs (00-16)
├── .claude/settings.local.json
└── CLAUDE.md
```

## Key Documents

- **`doc/implementation_plan.md`** — Authoritative spec for the entire system: architecture, data model, API routes, SSE events, agent tools, system prompt, directory structure, and phased dev plan. Read this before implementing anything.
- **`doc/ui/ui_design.md`** — UI style tokens (colors, fonts, spacing, border-radius, component specs). The mockup follows this guide.
- **`doc/ui/index.html`** — ~2500-line single-file mockup with working interactions: model selector, skills selector, file upload, message sending, demo animation, ask blocks, file panel with preview. Uses CSS custom properties for dark/light themes.

## Planned Architecture

From implementation_plan.md:

- **Backend**: FastAPI + AgentScope v2 Agent (ReAct loop)
- **Frontend**: Vanilla HTML/CSS/JS (no framework)
- **Agent**: 5 custom tools — FileManager, SubtaskManager, WebSearch, UserInteraction, MedicalDocGen
- **Storage**: Local filesystem (JSON/JSONL), abstracted via `StorageBase` for future MySQL migration
- **Streaming**: SSE with event types: text_delta, thinking_delta, tool_call, tool_result, task_update, ask_user, file_created, message_end
- **API prefix**: `/api/projects/{id}/...` for chat, files, tasks, interact

## Key Conventions

- All user-facing strings and comments in Chinese
- AgentScope v2 Agent class, not v1 APIs — reference docs in `doc/agentscope-v2/`
- UI follows extreme minimalist dark theme per `ui_design.md`: no emojis in production, no colored icons, no borders on AI reply components, gray-scale hierarchy with accent green only for interactive states
- Storage layer abstracted via `StorageBase` interface — `LocalStorage` now, `MySQLStorage` later

## Running the Mockup

```bash
# UI mockup (static)
open doc/ui/index.html
# or
cd doc/ui && python -m http.server 8080
```

## Planned Running Commands

```bash
# Backend (not yet implemented)
cd backend && pip install -r requirements.txt && python main.py

# Frontend (not yet implemented)
cd frontend && python -m http.server 8080
```

## Planned Directory Structure

See `doc/implementation_plan.md` section 9 for the full tree. Key paths:

- `backend/main.py` — FastAPI entry
- `backend/agent/medex_agent.py` — Agent definition
- `backend/tools/` — 5 custom tools
- `backend/storage/` — `base.py` (interface) + `local_storage.py`
- `backend/models/` — Pydantic models for Project, Message, Task, File
- `frontend/js/sse.js` — SSE stream handler
- `data/projects/{id}/` — meta.json, messages.jsonl, tasks.json, files/uploads/, files/generated/
