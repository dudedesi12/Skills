# Agent Coordinator Protocol — Blueprint

## Purpose

Provide a complete protocol for multi-agent communication, coordination, and reliability in a Next.js + Supabase stack. Covers message formats, agent lifecycle, communication patterns, error recovery, and observability so that any number of agents can work together without losing messages or failing silently.

## Must Cover

- **Message protocol** — Typed AgentMessage interface with correlation IDs, TTL, retry metadata, and idempotency keys
- **Agent lifecycle** — Registration, capability-based discovery, heartbeat health checks (30s interval), graceful deregistration with drain
- **Communication patterns** — Request-response (sync API routes), fire-and-forget (Supabase queue), pub-sub (Supabase Realtime channels), saga pattern (multi-step with compensation)
- **Error handling** — Standard error codes (AGENT_UNAVAILABLE, TIMEOUT, INVALID_INPUT, RATE_LIMITED, MODEL_ERROR, QUOTA_EXCEEDED), circuit breaker with configurable thresholds, fallback chains, dead letter queue
- **Shared state** — Workflow context with optimistic concurrency (version field), Supabase table design for message queue, idempotency keys, TTL-based cron cleanup
- **Observability** — Structured JSON logging, trace ID propagation across agents, metrics tracking (latency, success/failure rate, token usage)

## Reference Files

- `references/message-schema.md` — Complete TypeScript types for all message types, payloads, and metadata
- `references/supabase-queue-schema.md` — Database tables (agent_message_queue, agent_dead_letter_queue, workflow_contexts, agent_metrics) with indexes and RLS policies
- `references/circuit-breaker.md` — Full CircuitBreaker class with closed/open/half-open states, configurable failure thresholds, reset timeouts, and monitoring hooks
- `references/saga-pattern.md` — SagaOrchestrator with step execution, compensation rollback, persistent saga state, and timeout handling
- `references/logging-format.md` — Structured log schema, trace reconstruction queries, and log aggregation patterns
- `references/agent-registry.md` — Agent registration, capability-based discovery, heartbeat monitoring, and stale agent cleanup
