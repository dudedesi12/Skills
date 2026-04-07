# Architecture — Master Orchestrator Agent

## System Diagram

```
┌─────────────────────────────────────────────────┐
│                   User / Client                  │
└───────────────────────┬─────────────────────────┘
                        │ POST /api/orchestrate
                        ▼
┌─────────────────────────────────────────────────┐
│              Master Orchestrator                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │  Router   │→│ Workflow  │→│ Cost Tracker  │  │
│  │(Flash)    │  │ Engine   │  │              │  │
│  └──────────┘  └──────────┘  └──────────────┘  │
└───────┬───────────┬───────────┬─────────────────┘
        │           │           │
   ┌────▼────┐ ┌────▼────┐ ┌───▼─────┐
   │ Agent A │ │ Agent B │ │ Agent C │
   │(Scraper)│ │(Parser) │ │(Content)│
   └────┬────┘ └────┬────┘ └───┬─────┘
        │           │           │
        └───────────┼───────────┘
                    ▼
          ┌─────────────────┐
          │    Supabase     │
          │  - workflows    │
          │  - tasks        │
          │  - agents       │
          │  - dead_letter  │
          └─────────────────┘
```

## Data Flow

1. **Request arrives** at `/api/orchestrate` with a task description
2. **Router** (Gemini Flash) classifies the task and picks agents
3. **Workflow Engine** creates a workflow record, then executes tasks
4. **Each task** calls the agent's API endpoint via internal fetch
5. **Results** flow back through the engine, aggregated for the caller
6. **Failures** retry with exponential backoff, then go to dead letter queue
7. **Cost tracker** logs token usage per task

## Component Responsibilities

| Component | Purpose | Model |
|-----------|---------|-------|
| Router | Classify task, select agents, choose workflow type | gemini-2.0-flash |
| Workflow Engine | Execute sequential/parallel/conditional flows | N/A (orchestration logic) |
| Agent Registry | Track available agents and their health | N/A (Supabase table) |
| Cost Tracker | Log token usage, estimate costs | N/A (calculation) |
| Health Cron | Check agent heartbeats every 5 minutes | N/A (cron job) |

## Security Model

- All orchestrator routes require `INTERNAL_API_KEY` in Authorization header
- Agent-to-agent calls use the same internal key
- Supabase tables use service_role only (no client access)
- Cron routes validate `CRON_SECRET`
