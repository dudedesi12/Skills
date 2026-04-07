# Agent Team Builder — Blueprint

## Purpose
Guide users with zero coding knowledge through building production-ready AI agents for their SaaS. Covers 10 specialized agent types using Next.js App Router, Supabase, and the Gemini API. Every agent includes input validation, error handling, retries, rate limiting, cost budgets, logging, and health checks.

## Must Cover
- Shared BaseAgent utility class with Gemini integration, Zod validation, retry logic, rate limiting, and Supabase task queue
- All 10 agent types: Scraper, Parser, Content, Analysis, Notification, Moderation, Search, Report, Translation, Customer Support
- Dynamic API route at `app/api/agents/[agentName]/route.ts` with agent registry
- Health check endpoint per agent
- Agent versioning with A/B traffic routing
- Frontend client helper for calling agents
- Database schema for tasks, logs, and budgets

## Reference Files
1. `references/agent-configs.md` — Config templates (name, model, capabilities, I/O schema, rate limits) for all 10 agent types
2. `references/system-prompts.md` — Optimized system prompts engineered for each agent's specific task
3. `references/agent-api-routes.md` — Next.js API route templates including middleware patterns
4. `references/testing-agents.md` — Testing strategies, sample payloads, and curl commands for each agent
5. `references/cost-budgets.md` — Token usage estimates and monthly cost budgeting per agent type
