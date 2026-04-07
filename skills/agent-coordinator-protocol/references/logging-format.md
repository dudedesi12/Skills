# Structured Logging Format

JSON-based structured logging with trace ID propagation for multi-agent observability.

## Log Schema

Every log entry follows this shape:

```typescript
// lib/agents/logger.ts

export interface AgentLogEntry {
  level: "debug" | "info" | "warn" | "error";
  timestamp: string;
  traceId: string;
  spanId: string;
  parentSpanId?: string;
  agentId: string;
  event: string;
  durationMs?: number;
  tokenUsage?: {
    input: number;
    output: number;
    total: number;
    model: string;
  };
  error?: string;
  errorCode?: string;
  metadata?: Record<string, unknown>;
}
```

## Logger Implementation

```typescript
// lib/agents/logger.ts
import { randomUUID } from "crypto";

export function createAgentLogger(agentId: string, traceId: string, parentSpanId?: string) {
  const spanId = randomUUID().slice(0, 8);

  function log(
    level: AgentLogEntry["level"],
    event: string,
    extra?: Partial<Omit<AgentLogEntry, "level" | "timestamp" | "traceId" | "spanId" | "agentId" | "event">>
  ): void {
    const entry: AgentLogEntry = {
      level,
      timestamp: new Date().toISOString(),
      traceId,
      spanId,
      parentSpanId,
      agentId,
      event,
      ...extra,
    };

    const output = JSON.stringify(entry);

    switch (level) {
      case "error":
        console.error(output);
        break;
      case "warn":
        console.warn(output);
        break;
      default:
        console.log(output);
        break;
    }
  }

  return {
    debug: (event: string, extra?: Partial<AgentLogEntry>) => log("debug", event, extra),
    info: (event: string, extra?: Partial<AgentLogEntry>) => log("info", event, extra),
    warn: (event: string, extra?: Partial<AgentLogEntry>) => log("warn", event, extra),
    error: (event: string, extra?: Partial<AgentLogEntry>) => log("error", event, extra),
    child: (childAgentId: string) => createAgentLogger(childAgentId, traceId, spanId),
    getSpanId: () => spanId,
    getTraceId: () => traceId,
  };
}
```

## Trace Propagation

Pass trace IDs through HTTP headers between agents:

```typescript
// lib/agents/trace.ts
import { NextRequest } from "next/server";
import { randomUUID } from "crypto";

export function extractTraceContext(request: NextRequest): {
  traceId: string;
  parentSpanId: string | undefined;
  correlationId: string;
} {
  return {
    traceId: request.headers.get("X-Trace-Id") ?? randomUUID(),
    parentSpanId: request.headers.get("X-Parent-Span-Id") ?? undefined,
    correlationId: request.headers.get("X-Correlation-Id") ?? randomUUID(),
  };
}

export function injectTraceHeaders(
  traceId: string,
  spanId: string,
  correlationId: string
): Record<string, string> {
  return {
    "X-Trace-Id": traceId,
    "X-Parent-Span-Id": spanId,
    "X-Correlation-Id": correlationId,
  };
}
```

## Usage in an Agent Handler

```typescript
// app/api/agents/writer/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createAgentLogger } from "@/lib/agents/logger";
import { extractTraceContext } from "@/lib/agents/trace";

export async function POST(request: NextRequest) {
  const { traceId, parentSpanId, correlationId } = extractTraceContext(request);
  const logger = createAgentLogger("writer-agent", traceId, parentSpanId);

  const startTime = Date.now();

  try {
    const body = await request.json();
    logger.info("request_received", { metadata: { correlationId, action: body.action } });

    // --- Agent logic here ---
    const result = { content: "Generated content", wordCount: 500 };
    // --- End agent logic ---

    const durationMs = Date.now() - startTime;

    logger.info("request_complete", {
      durationMs,
      tokenUsage: { input: 1200, output: 800, total: 2000, model: "gemini-2.0-flash" },
      metadata: { correlationId },
    });

    return NextResponse.json(result);
  } catch (err) {
    const durationMs = Date.now() - startTime;
    const errorMessage = err instanceof Error ? err.message : "Unknown error";

    logger.error("request_failed", {
      durationMs,
      error: errorMessage,
      errorCode: "MODEL_ERROR",
      metadata: { correlationId },
    });

    return NextResponse.json(
      { code: "MODEL_ERROR", message: errorMessage },
      { status: 500 }
    );
  }
}
```

## Standard Events

Use these event names consistently across all agents:

| Event | Level | When |
|---|---|---|
| `agent_registered` | info | Agent registers with the coordinator |
| `agent_deregistered` | info | Agent shuts down gracefully |
| `heartbeat_sent` | debug | Agent sends a heartbeat |
| `heartbeat_failed` | error | Heartbeat update fails |
| `request_received` | info | Agent receives a request |
| `request_complete` | info | Agent finishes processing |
| `request_failed` | error | Agent fails to process |
| `saga_step_start` | info | Saga step begins |
| `saga_step_complete` | info | Saga step finishes |
| `saga_step_failed` | error | Saga step fails |
| `saga_compensation_start` | info | Compensation begins |
| `saga_compensate_step_complete` | info | Compensation step finishes |
| `saga_compensate_step_failed` | error | Compensation step fails |
| `circuit_breaker_state_change` | info/error | Circuit breaker transitions |
| `message_enqueued` | info | Message added to queue |
| `message_dequeued` | info | Message picked up from queue |
| `message_dead_lettered` | warn | Message moved to DLQ |
| `fallback_agent_failed` | warn | Primary agent failed, trying fallback |
| `fallback_success` | info | Fallback agent succeeded |

## Trace Reconstruction Queries

Find all logs for a single workflow:

```sql
-- Find all log entries for a trace (requires logs stored in a table or log aggregator)
-- If using Supabase for log storage:
select *
from agent_logs
where trace_id = 'your-trace-id-here'
order by timestamp asc;

-- Find slow operations
select agent_id, event, duration_ms, timestamp
from agent_logs
where duration_ms > 5000
  and recorded_at > now() - interval '1 hour'
order by duration_ms desc;

-- Error rate per agent
select
  agent_id,
  count(*) filter (where level = 'error') as errors,
  count(*) as total,
  round(100.0 * count(*) filter (where level = 'error') / count(*), 2) as error_pct
from agent_logs
where recorded_at > now() - interval '1 hour'
group by agent_id
order by error_pct desc;
```

## Log Storage Table (Optional)

If you want to persist logs in Supabase for querying:

```sql
-- supabase/migrations/004_agent_logs.sql

create table if not exists agent_logs (
  id uuid primary key default gen_random_uuid(),
  level text not null,
  timestamp timestamptz not null,
  trace_id text not null,
  span_id text not null,
  parent_span_id text,
  agent_id text not null,
  event text not null,
  duration_ms integer,
  token_usage jsonb,
  error text,
  error_code text,
  metadata jsonb,
  recorded_at timestamptz not null default now()
);

create index idx_logs_trace on agent_logs(trace_id);
create index idx_logs_agent on agent_logs(agent_id, recorded_at desc);
create index idx_logs_level on agent_logs(level, recorded_at desc)
  where level in ('error', 'warn');
create index idx_logs_event on agent_logs(event);
```
