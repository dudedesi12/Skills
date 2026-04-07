---
name: agent-coordinator-protocol
description: "Use this skill whenever the user mentions agent communication, message passing, agent protocol, inter-agent messaging, workflow state, error handling between agents, retry policies, circuit breakers, message queues, agent discovery, agent registration, heartbeats, saga patterns, or any concern about how agents coordinate and communicate. Also trigger for 'how do agents talk to each other', 'agent failed', 'workflow stuck', 'agent timeout', or any multi-agent reliability concern."
---

# Agent Coordinator Protocol

How to build reliable multi-agent communication in your Next.js app using Supabase as the message backbone. Every agent registers itself, sends typed messages, handles failures gracefully, and reports its health.

## Message Protocol

Every message between agents follows one strict shape. See `references/message-schema.md` for the full set of types.

```typescript
// lib/agents/types.ts
export type AgentMessageType = "request" | "response" | "event" | "error" | "heartbeat";
export type AgentPriority = "low" | "normal" | "high" | "critical";

export interface AgentMessage {
  id: string;
  timestamp: string;
  from: string;
  to: string;
  type: AgentMessageType;
  priority: AgentPriority;
  payload: Record<string, unknown>;
  context: {
    workflowId: string;
    stepIndex: number;
    totalSteps: number;
  };
  metadata: {
    correlationId: string;
    parentMessageId: string | null;
    ttl: number;
    retryCount: number;
    maxRetries: number;
    idempotencyKey: string;
  };
}

export interface AgentRegistration {
  agentId: string;
  name: string;
  capabilities: string[];
  endpoint: string;
  maxConcurrency: number;
  registeredAt: string;
  lastHeartbeat: string;
  status: "active" | "draining" | "offline";
}

export type AgentErrorCode =
  | "AGENT_UNAVAILABLE"
  | "TIMEOUT"
  | "INVALID_INPUT"
  | "RATE_LIMITED"
  | "MODEL_ERROR"
  | "QUOTA_EXCEEDED";

export interface AgentError {
  code: AgentErrorCode;
  message: string;
  agentId: string;
  timestamp: string;
  retryable: boolean;
  context?: Record<string, unknown>;
}
```

## Agent Lifecycle

Agents must register before sending or receiving messages. See `references/agent-registry.md` for the full registry implementation.

### Registration

When an agent starts, it announces itself to the master coordinator:

```typescript
// lib/agents/registry.ts
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export async function registerAgent(agent: {
  agentId: string;
  name: string;
  capabilities: string[];
  endpoint: string;
  maxConcurrency: number;
}): Promise<{ success: boolean; error: string | null }> {
  try {
    const { error } = await supabase.from("agent_registry").upsert(
      {
        agent_id: agent.agentId,
        name: agent.name,
        capabilities: agent.capabilities,
        endpoint: agent.endpoint,
        max_concurrency: agent.maxConcurrency,
        status: "active",
        registered_at: new Date().toISOString(),
        last_heartbeat: new Date().toISOString(),
      },
      { onConflict: "agent_id" }
    );

    if (error) {
      console.error(JSON.stringify({ level: "error", event: "agent_register_failed", agentId: agent.agentId, error: error.message }));
      return { success: false, error: error.message };
    }

    console.log(JSON.stringify({ level: "info", event: "agent_registered", agentId: agent.agentId, capabilities: agent.capabilities }));
    return { success: true, error: null };
  } catch (err) {
    const message = err instanceof Error ? err.message : "Unknown registration error";
    return { success: false, error: message };
  }
}
```

### Discovery

The master finds available agents by capability:

```typescript
// lib/agents/registry.ts
export async function discoverAgents(capability: string): Promise<{
  agents: AgentRegistration[];
  error: string | null;
}> {
  try {
    const { data, error } = await supabase
      .from("agent_registry")
      .select("*")
      .contains("capabilities", [capability])
      .eq("status", "active")
      .gte("last_heartbeat", new Date(Date.now() - 90_000).toISOString());

    if (error) {
      return { agents: [], error: error.message };
    }

    const agents: AgentRegistration[] = (data ?? []).map((row) => ({
      agentId: row.agent_id,
      name: row.name,
      capabilities: row.capabilities,
      endpoint: row.endpoint,
      maxConcurrency: row.max_concurrency,
      registeredAt: row.registered_at,
      lastHeartbeat: row.last_heartbeat,
      status: row.status,
    }));

    return { agents, error: null };
  } catch (err) {
    const message = err instanceof Error ? err.message : "Unknown discovery error";
    return { agents: [], error: message };
  }
}
```

### Heartbeat

Every agent sends a heartbeat every 30 seconds. If three consecutive heartbeats are missed (90 seconds), the agent is considered offline:

```typescript
// lib/agents/heartbeat.ts
export function startHeartbeat(agentId: string, intervalMs = 30_000): { stop: () => void } {
  const timer = setInterval(async () => {
    try {
      const { error } = await supabase
        .from("agent_registry")
        .update({ last_heartbeat: new Date().toISOString() })
        .eq("agent_id", agentId);

      if (error) {
        console.error(JSON.stringify({ level: "error", event: "heartbeat_failed", agentId, error: error.message }));
      }
    } catch (err) {
      console.error(JSON.stringify({
        level: "error",
        event: "heartbeat_exception",
        agentId,
        error: err instanceof Error ? err.message : "Unknown",
      }));
    }
  }, intervalMs);

  return {
    stop: () => {
      clearInterval(timer);
      supabase
        .from("agent_registry")
        .update({ status: "offline" })
        .eq("agent_id", agentId)
        .then(({ error }) => {
          if (error) {
            console.error(JSON.stringify({ level: "error", event: "deregister_failed", agentId }));
          }
        });
    },
  };
}
```

### Deregistration

Graceful shutdown marks the agent offline and drains in-flight work:

```typescript
// lib/agents/lifecycle.ts
export async function deregisterAgent(agentId: string): Promise<{ success: boolean; error: string | null }> {
  try {
    const { error: drainError } = await supabase
      .from("agent_registry")
      .update({ status: "draining" })
      .eq("agent_id", agentId);

    if (drainError) {
      return { success: false, error: drainError.message };
    }

    const { data: pendingMessages } = await supabase
      .from("agent_message_queue")
      .select("id")
      .eq("to_agent", agentId)
      .eq("status", "processing");

    if (pendingMessages && pendingMessages.length > 0) {
      await new Promise((resolve) => setTimeout(resolve, 5000));
    }

    const { error: offlineError } = await supabase
      .from("agent_registry")
      .update({ status: "offline" })
      .eq("agent_id", agentId);

    if (offlineError) {
      return { success: false, error: offlineError.message };
    }

    console.log(JSON.stringify({ level: "info", event: "agent_deregistered", agentId }));
    return { success: true, error: null };
  } catch (err) {
    const message = err instanceof Error ? err.message : "Unknown deregistration error";
    return { success: false, error: message };
  }
}
```

## Communication Patterns

### Request-Response (Synchronous)

Direct API call where the caller waits for a response. Use for fast operations (under 10 seconds).

```typescript
// app/api/agents/[agentId]/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@supabase/supabase-js";
import { randomUUID } from "crypto";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export async function POST(
  request: NextRequest,
  { params }: { params: Promise<{ agentId: string }> }
) {
  const { agentId } = await params;

  try {
    const body = await request.json();
    const correlationId = body.metadata?.correlationId ?? randomUUID();

    const { data: agent } = await supabase
      .from("agent_registry")
      .select("endpoint, status")
      .eq("agent_id", agentId)
      .single();

    if (!agent || agent.status !== "active") {
      return NextResponse.json(
        { code: "AGENT_UNAVAILABLE", message: `Agent ${agentId} is not available` },
        { status: 503 }
      );
    }

    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), 10_000);

    const response = await fetch(agent.endpoint, {
      method: "POST",
      headers: { "Content-Type": "application/json", "X-Correlation-Id": correlationId },
      body: JSON.stringify(body),
      signal: controller.signal,
    });

    clearTimeout(timeoutId);

    if (!response.ok) {
      const errorBody = await response.json().catch(() => ({}));
      return NextResponse.json(
        { code: "MODEL_ERROR", message: errorBody.message ?? `Agent returned ${response.status}` },
        { status: response.status }
      );
    }

    const result = await response.json();
    return NextResponse.json(result);
  } catch (err) {
    if (err instanceof DOMException && err.name === "AbortError") {
      return NextResponse.json({ code: "TIMEOUT", message: "Agent did not respond within 10s" }, { status: 504 });
    }
    const message = err instanceof Error ? err.message : "Unknown error";
    return NextResponse.json({ code: "MODEL_ERROR", message }, { status: 500 });
  }
}
```

### Fire-and-Forget (Async Queue)

Push a message to the Supabase queue and return immediately. The target agent polls or subscribes. See `references/supabase-queue-schema.md` for the table design.

```typescript
// lib/agents/queue.ts
import { createClient } from "@supabase/supabase-js";
import { randomUUID } from "crypto";
import type { AgentMessage } from "./types";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export async function enqueueMessage(message: Omit<AgentMessage, "id" | "timestamp">): Promise<{
  messageId: string | null;
  error: string | null;
}> {
  const messageId = randomUUID();

  try {
    const { error } = await supabase.from("agent_message_queue").insert({
      id: messageId,
      from_agent: message.from,
      to_agent: message.to,
      message_type: message.type,
      priority: message.priority,
      payload: message.payload,
      workflow_context: message.context,
      correlation_id: message.metadata.correlationId,
      parent_message_id: message.metadata.parentMessageId,
      ttl_seconds: message.metadata.ttl,
      retry_count: 0,
      max_retries: message.metadata.maxRetries,
      idempotency_key: message.metadata.idempotencyKey,
      status: "pending",
      created_at: new Date().toISOString(),
    });

    if (error) {
      console.error(JSON.stringify({ level: "error", event: "enqueue_failed", messageId, error: error.message }));
      return { messageId: null, error: error.message };
    }

    return { messageId, error: null };
  } catch (err) {
    const errMessage = err instanceof Error ? err.message : "Unknown enqueue error";
    return { messageId: null, error: errMessage };
  }
}

export async function dequeueMessages(agentId: string, limit = 10): Promise<{
  messages: AgentMessage[];
  error: string | null;
}> {
  try {
    const { data, error } = await supabase
      .from("agent_message_queue")
      .select("*")
      .eq("to_agent", agentId)
      .eq("status", "pending")
      .order("priority", { ascending: false })
      .order("created_at", { ascending: true })
      .limit(limit);

    if (error) {
      return { messages: [], error: error.message };
    }

    if (!data || data.length === 0) {
      return { messages: [], error: null };
    }

    const ids = data.map((row) => row.id);
    await supabase
      .from("agent_message_queue")
      .update({ status: "processing", processed_at: new Date().toISOString() })
      .in("id", ids);

    const messages: AgentMessage[] = data.map((row) => ({
      id: row.id,
      timestamp: row.created_at,
      from: row.from_agent,
      to: row.to_agent,
      type: row.message_type,
      priority: row.priority,
      payload: row.payload,
      context: row.workflow_context,
      metadata: {
        correlationId: row.correlation_id,
        parentMessageId: row.parent_message_id,
        ttl: row.ttl_seconds,
        retryCount: row.retry_count,
        maxRetries: row.max_retries,
        idempotencyKey: row.idempotency_key,
      },
    }));

    return { messages, error: null };
  } catch (err) {
    const errMessage = err instanceof Error ? err.message : "Unknown dequeue error";
    return { messages: [], error: errMessage };
  }
}
```

### Pub-Sub (Supabase Realtime)

Agents subscribe to channels for real-time event broadcasting:

```typescript
// lib/agents/pubsub.ts
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export function subscribeToAgentEvents(
  agentId: string,
  onMessage: (message: Record<string, unknown>) => void
): { unsubscribe: () => void } {
  const channel = supabase
    .channel(`agent-${agentId}`)
    .on("broadcast", { event: "agent-message" }, (payload) => {
      try {
        onMessage(payload.payload as Record<string, unknown>);
      } catch (err) {
        console.error(JSON.stringify({
          level: "error",
          event: "pubsub_handler_error",
          agentId,
          error: err instanceof Error ? err.message : "Unknown",
        }));
      }
    })
    .subscribe((status) => {
      console.log(JSON.stringify({ level: "info", event: "pubsub_status", agentId, status }));
    });

  return {
    unsubscribe: () => {
      supabase.removeChannel(channel);
    },
  };
}

export async function publishAgentEvent(
  channel: string,
  event: Record<string, unknown>
): Promise<{ success: boolean; error: string | null }> {
  try {
    const ch = supabase.channel(channel);
    await ch.subscribe();
    await ch.send({ type: "broadcast", event: "agent-message", payload: event });
    supabase.removeChannel(ch);
    return { success: true, error: null };
  } catch (err) {
    const message = err instanceof Error ? err.message : "Unknown publish error";
    return { success: false, error: message };
  }
}
```

### Saga Pattern

For long-running workflows that span multiple agents. Each step has a compensation action for rollback. See `references/saga-pattern.md` for the full implementation.

```typescript
// lib/agents/saga.ts
export interface SagaStep {
  name: string;
  agentId: string;
  execute: (context: Record<string, unknown>) => Promise<Record<string, unknown>>;
  compensate: (context: Record<string, unknown>) => Promise<void>;
}

export async function executeSaga(
  workflowId: string,
  steps: SagaStep[],
  initialContext: Record<string, unknown>
): Promise<{ success: boolean; context: Record<string, unknown>; failedStep: string | null }> {
  const completedSteps: SagaStep[] = [];
  let context = { ...initialContext };

  for (const step of steps) {
    try {
      console.log(JSON.stringify({
        level: "info",
        event: "saga_step_start",
        workflowId,
        step: step.name,
        agentId: step.agentId,
      }));

      const result = await step.execute(context);
      context = { ...context, ...result };
      completedSteps.push(step);

      console.log(JSON.stringify({
        level: "info",
        event: "saga_step_complete",
        workflowId,
        step: step.name,
      }));
    } catch (err) {
      console.error(JSON.stringify({
        level: "error",
        event: "saga_step_failed",
        workflowId,
        step: step.name,
        error: err instanceof Error ? err.message : "Unknown",
      }));

      for (const completed of completedSteps.reverse()) {
        try {
          await completed.compensate(context);
          console.log(JSON.stringify({
            level: "info",
            event: "saga_compensate_complete",
            workflowId,
            step: completed.name,
          }));
        } catch (compErr) {
          console.error(JSON.stringify({
            level: "error",
            event: "saga_compensate_failed",
            workflowId,
            step: completed.name,
            error: compErr instanceof Error ? compErr.message : "Unknown",
          }));
        }
      }

      return { success: false, context, failedStep: step.name };
    }
  }

  return { success: true, context, failedStep: null };
}
```

## Error Handling Protocol

### Circuit Breaker

Prevents cascading failures by cutting off calls to unhealthy agents. See `references/circuit-breaker.md` for the full implementation with configurable thresholds.

```typescript
// lib/agents/circuit-breaker.ts
type CircuitState = "closed" | "open" | "half-open";

export class CircuitBreaker {
  private state: CircuitState = "closed";
  private failures = 0;
  private lastFailureTime = 0;
  private successesSinceHalfOpen = 0;

  constructor(
    private readonly threshold: number = 5,
    private readonly resetTimeoutMs: number = 30_000,
    private readonly halfOpenSuccesses: number = 3
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === "open") {
      if (Date.now() - this.lastFailureTime > this.resetTimeoutMs) {
        this.state = "half-open";
        this.successesSinceHalfOpen = 0;
      } else {
        throw new Error("Circuit breaker is OPEN — agent unavailable");
      }
    }

    try {
      const result = await fn();

      if (this.state === "half-open") {
        this.successesSinceHalfOpen++;
        if (this.successesSinceHalfOpen >= this.halfOpenSuccesses) {
          this.state = "closed";
          this.failures = 0;
        }
      } else {
        this.failures = 0;
      }

      return result;
    } catch (err) {
      this.failures++;
      this.lastFailureTime = Date.now();

      if (this.failures >= this.threshold) {
        this.state = "open";
        console.error(JSON.stringify({
          level: "error",
          event: "circuit_breaker_opened",
          failures: this.failures,
          threshold: this.threshold,
        }));
      }

      throw err;
    }
  }

  getState(): CircuitState {
    return this.state;
  }
}
```

### Fallback Chains

Try agents in priority order. If the primary fails, fall back to the next:

```typescript
// lib/agents/fallback.ts
import { CircuitBreaker } from "./circuit-breaker";

interface FallbackAgent {
  agentId: string;
  execute: (payload: Record<string, unknown>) => Promise<Record<string, unknown>>;
  circuitBreaker: CircuitBreaker;
}

export async function executeWithFallback(
  agents: FallbackAgent[],
  payload: Record<string, unknown>,
  correlationId: string
): Promise<{ result: Record<string, unknown> | null; agentUsed: string | null; error: string | null }> {
  for (const agent of agents) {
    try {
      const result = await agent.circuitBreaker.execute(() => agent.execute(payload));
      console.log(JSON.stringify({
        level: "info",
        event: "fallback_success",
        agentId: agent.agentId,
        correlationId,
      }));
      return { result, agentUsed: agent.agentId, error: null };
    } catch (err) {
      console.warn(JSON.stringify({
        level: "warn",
        event: "fallback_agent_failed",
        agentId: agent.agentId,
        correlationId,
        error: err instanceof Error ? err.message : "Unknown",
      }));
      continue;
    }
  }

  return { result: null, agentUsed: null, error: "All fallback agents exhausted" };
}
```

### Dead Letter Queue

Messages that exceed max retries go to the dead letter queue for manual inspection:

```typescript
// lib/agents/dead-letter.ts
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export async function moveToDeadLetter(
  messageId: string,
  reason: string
): Promise<{ success: boolean; error: string | null }> {
  try {
    const { data: message, error: fetchError } = await supabase
      .from("agent_message_queue")
      .select("*")
      .eq("id", messageId)
      .single();

    if (fetchError || !message) {
      return { success: false, error: fetchError?.message ?? "Message not found" };
    }

    const { error: insertError } = await supabase.from("agent_dead_letter_queue").insert({
      original_message_id: message.id,
      from_agent: message.from_agent,
      to_agent: message.to_agent,
      message_type: message.message_type,
      payload: message.payload,
      failure_reason: reason,
      retry_count: message.retry_count,
      original_created_at: message.created_at,
      moved_at: new Date().toISOString(),
    });

    if (insertError) {
      return { success: false, error: insertError.message };
    }

    await supabase.from("agent_message_queue").update({ status: "dead_letter" }).eq("id", messageId);

    console.log(JSON.stringify({
      level: "warn",
      event: "message_dead_lettered",
      messageId,
      reason,
    }));

    return { success: true, error: null };
  } catch (err) {
    const errMessage = err instanceof Error ? err.message : "Unknown dead letter error";
    return { success: false, error: errMessage };
  }
}
```

## Shared State Management

### Workflow Context

Every workflow carries a shared context that agents read from and write to:

```typescript
// lib/agents/workflow-context.ts
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export async function getWorkflowContext(workflowId: string): Promise<{
  context: Record<string, unknown> | null;
  error: string | null;
}> {
  try {
    const { data, error } = await supabase
      .from("workflow_contexts")
      .select("context, version")
      .eq("workflow_id", workflowId)
      .single();

    if (error) {
      return { context: null, error: error.message };
    }

    return { context: data?.context ?? null, error: null };
  } catch (err) {
    return { context: null, error: err instanceof Error ? err.message : "Unknown" };
  }
}

export async function updateWorkflowContext(
  workflowId: string,
  updates: Record<string, unknown>,
  expectedVersion: number
): Promise<{ success: boolean; error: string | null }> {
  try {
    const { data: current, error: fetchError } = await supabase
      .from("workflow_contexts")
      .select("context, version")
      .eq("workflow_id", workflowId)
      .single();

    if (fetchError) {
      return { success: false, error: fetchError.message };
    }

    if (current.version !== expectedVersion) {
      return { success: false, error: "Version conflict — another agent updated the context" };
    }

    const mergedContext = { ...current.context, ...updates };

    const { error: updateError } = await supabase
      .from("workflow_contexts")
      .update({ context: mergedContext, version: expectedVersion + 1, updated_at: new Date().toISOString() })
      .eq("workflow_id", workflowId)
      .eq("version", expectedVersion);

    if (updateError) {
      return { success: false, error: updateError.message };
    }

    return { success: true, error: null };
  } catch (err) {
    return { success: false, error: err instanceof Error ? err.message : "Unknown" };
  }
}
```

### TTL-Based Cleanup

A cron job that removes expired messages and stale agent registrations:

```typescript
// app/api/cron/agent-cleanup/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export async function GET(request: NextRequest) {
  const authHeader = request.headers.get("authorization");
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  try {
    const { count: expiredMessages, error: msgError } = await supabase
      .from("agent_message_queue")
      .delete({ count: "exact" })
      .lt("expires_at", new Date().toISOString())
      .eq("status", "pending");

    const { count: staleAgents, error: agentError } = await supabase
      .from("agent_registry")
      .update({ status: "offline" })
      .lt("last_heartbeat", new Date(Date.now() - 90_000).toISOString())
      .eq("status", "active");

    console.log(JSON.stringify({
      level: "info",
      event: "agent_cleanup_complete",
      expiredMessages: expiredMessages ?? 0,
      staleAgents: staleAgents ?? 0,
      errors: [msgError?.message, agentError?.message].filter(Boolean),
    }));

    return NextResponse.json({
      expiredMessages: expiredMessages ?? 0,
      staleAgents: staleAgents ?? 0,
    });
  } catch (err) {
    console.error(JSON.stringify({
      level: "error",
      event: "agent_cleanup_failed",
      error: err instanceof Error ? err.message : "Unknown",
    }));
    return NextResponse.json({ error: "Cleanup failed" }, { status: 500 });
  }
}
```

Add this to `vercel.json`:

```json
// vercel.json
{
  "crons": [
    { "path": "/api/cron/agent-cleanup", "schedule": "*/5 * * * *" }
  ]
}
```

## Observability

Structured logging, trace propagation, and metrics tracking. See `references/logging-format.md` for the full format specification.

```typescript
// lib/agents/logger.ts
export interface AgentLogEntry {
  level: "debug" | "info" | "warn" | "error";
  timestamp: string;
  traceId: string;
  spanId: string;
  agentId: string;
  event: string;
  durationMs?: number;
  tokenUsage?: { input: number; output: number; total: number };
  error?: string;
  metadata?: Record<string, unknown>;
}

export function createAgentLogger(agentId: string, traceId: string) {
  const spanId = crypto.randomUUID().slice(0, 8);

  function log(level: AgentLogEntry["level"], event: string, extra?: Partial<AgentLogEntry>) {
    const entry: AgentLogEntry = {
      level,
      timestamp: new Date().toISOString(),
      traceId,
      spanId,
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
    }
  }

  return {
    debug: (event: string, extra?: Partial<AgentLogEntry>) => log("debug", event, extra),
    info: (event: string, extra?: Partial<AgentLogEntry>) => log("info", event, extra),
    warn: (event: string, extra?: Partial<AgentLogEntry>) => log("warn", event, extra),
    error: (event: string, extra?: Partial<AgentLogEntry>) => log("error", event, extra),
  };
}
```

Track metrics per agent:

```typescript
// lib/agents/metrics.ts
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export async function recordAgentMetric(metric: {
  agentId: string;
  operation: string;
  durationMs: number;
  success: boolean;
  tokenUsage?: { input: number; output: number };
  traceId: string;
}): Promise<void> {
  try {
    await supabase.from("agent_metrics").insert({
      agent_id: metric.agentId,
      operation: metric.operation,
      duration_ms: metric.durationMs,
      success: metric.success,
      input_tokens: metric.tokenUsage?.input ?? 0,
      output_tokens: metric.tokenUsage?.output ?? 0,
      trace_id: metric.traceId,
      recorded_at: new Date().toISOString(),
    });
  } catch (err) {
    console.error(JSON.stringify({
      level: "error",
      event: "metric_record_failed",
      agentId: metric.agentId,
      error: err instanceof Error ? err.message : "Unknown",
    }));
  }
}
```
