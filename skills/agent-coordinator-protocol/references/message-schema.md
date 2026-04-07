# Agent Message Schema

Complete TypeScript types for the agent communication protocol.

## Core Message Types

```typescript
// lib/agents/types.ts

// --- Enums and Literals ---

export type AgentMessageType = "request" | "response" | "event" | "error" | "heartbeat";
export type AgentPriority = "low" | "normal" | "high" | "critical";
export type AgentStatus = "active" | "draining" | "offline";

export type AgentErrorCode =
  | "AGENT_UNAVAILABLE"
  | "TIMEOUT"
  | "INVALID_INPUT"
  | "RATE_LIMITED"
  | "MODEL_ERROR"
  | "QUOTA_EXCEEDED";

// --- Core Message Interface ---

export interface AgentMessage {
  id: string;
  timestamp: string;
  from: string;
  to: string;
  type: AgentMessageType;
  priority: AgentPriority;
  payload: Record<string, unknown>;
  context: WorkflowContext;
  metadata: MessageMetadata;
}

export interface WorkflowContext {
  workflowId: string;
  stepIndex: number;
  totalSteps: number;
}

export interface MessageMetadata {
  correlationId: string;
  parentMessageId: string | null;
  ttl: number;
  retryCount: number;
  maxRetries: number;
  idempotencyKey: string;
}

// --- Specialized Message Payloads ---

export interface RequestPayload {
  action: string;
  input: Record<string, unknown>;
  expectedOutputSchema?: Record<string, unknown>;
  timeoutMs?: number;
}

export interface ResponsePayload {
  action: string;
  output: Record<string, unknown>;
  durationMs: number;
  tokenUsage?: TokenUsage;
}

export interface EventPayload {
  eventType: string;
  data: Record<string, unknown>;
  source: string;
}

export interface ErrorPayload {
  code: AgentErrorCode;
  message: string;
  retryable: boolean;
  details?: Record<string, unknown>;
  stack?: string;
}

export interface HeartbeatPayload {
  agentId: string;
  status: AgentStatus;
  load: {
    activeRequests: number;
    maxConcurrency: number;
    memoryUsageMb: number;
  };
  uptime: number;
}

// --- Token Usage ---

export interface TokenUsage {
  input: number;
  output: number;
  total: number;
  model: string;
}

// --- Agent Registration ---

export interface AgentRegistration {
  agentId: string;
  name: string;
  capabilities: string[];
  endpoint: string;
  maxConcurrency: number;
  registeredAt: string;
  lastHeartbeat: string;
  status: AgentStatus;
  metadata?: {
    version: string;
    region: string;
    model?: string;
  };
}

// --- Agent Error ---

export interface AgentError {
  code: AgentErrorCode;
  message: string;
  agentId: string;
  timestamp: string;
  retryable: boolean;
  context?: Record<string, unknown>;
}

// --- Workflow State ---

export interface WorkflowState {
  workflowId: string;
  status: "running" | "completed" | "failed" | "compensating";
  currentStep: number;
  totalSteps: number;
  context: Record<string, unknown>;
  version: number;
  startedAt: string;
  updatedAt: string;
  completedAt: string | null;
  error: AgentError | null;
}
```

## Message Factory

Helper to create properly formatted messages:

```typescript
// lib/agents/message-factory.ts
import { randomUUID } from "crypto";
import type { AgentMessage, AgentMessageType, AgentPriority, WorkflowContext, MessageMetadata } from "./types";

export function createMessage(params: {
  from: string;
  to: string;
  type: AgentMessageType;
  priority?: AgentPriority;
  payload: Record<string, unknown>;
  context: WorkflowContext;
  correlationId?: string;
  parentMessageId?: string | null;
  ttl?: number;
  maxRetries?: number;
}): AgentMessage {
  const now = new Date().toISOString();
  const correlationId = params.correlationId ?? randomUUID();
  const idempotencyKey = `${params.from}-${params.to}-${params.type}-${correlationId}-${params.context.stepIndex}`;

  return {
    id: randomUUID(),
    timestamp: now,
    from: params.from,
    to: params.to,
    type: params.type,
    priority: params.priority ?? "normal",
    payload: params.payload,
    context: params.context,
    metadata: {
      correlationId,
      parentMessageId: params.parentMessageId ?? null,
      ttl: params.ttl ?? 300,
      retryCount: 0,
      maxRetries: params.maxRetries ?? 3,
      idempotencyKey,
    },
  };
}

export function createErrorMessage(params: {
  from: string;
  to: string;
  code: import("./types").AgentErrorCode;
  message: string;
  retryable: boolean;
  correlationId: string;
  context: WorkflowContext;
}): AgentMessage {
  return createMessage({
    from: params.from,
    to: params.to,
    type: "error",
    priority: "high",
    payload: {
      code: params.code,
      message: params.message,
      retryable: params.retryable,
    },
    context: params.context,
    correlationId: params.correlationId,
  });
}

export function createHeartbeat(agentId: string, load: {
  activeRequests: number;
  maxConcurrency: number;
  memoryUsageMb: number;
}, uptime: number): AgentMessage {
  return createMessage({
    from: agentId,
    to: "master",
    type: "heartbeat",
    priority: "low",
    payload: {
      agentId,
      status: "active",
      load,
      uptime,
    },
    context: { workflowId: "system", stepIndex: 0, totalSteps: 0 },
    ttl: 60,
    maxRetries: 0,
  });
}
```

## Message Validation

Always validate incoming messages before processing:

```typescript
// lib/agents/validation.ts
import type { AgentMessage, AgentMessageType, AgentPriority } from "./types";

const VALID_TYPES: AgentMessageType[] = ["request", "response", "event", "error", "heartbeat"];
const VALID_PRIORITIES: AgentPriority[] = ["low", "normal", "high", "critical"];

export function validateMessage(msg: unknown): { valid: boolean; errors: string[] } {
  const errors: string[] = [];

  if (!msg || typeof msg !== "object") {
    return { valid: false, errors: ["Message must be a non-null object"] };
  }

  const m = msg as Record<string, unknown>;

  if (typeof m.id !== "string" || m.id.length === 0) {
    errors.push("id must be a non-empty string");
  }

  if (typeof m.timestamp !== "string" || isNaN(Date.parse(m.timestamp as string))) {
    errors.push("timestamp must be a valid ISO date string");
  }

  if (typeof m.from !== "string" || m.from.length === 0) {
    errors.push("from must be a non-empty string");
  }

  if (typeof m.to !== "string" || m.to.length === 0) {
    errors.push("to must be a non-empty string");
  }

  if (!VALID_TYPES.includes(m.type as AgentMessageType)) {
    errors.push(`type must be one of: ${VALID_TYPES.join(", ")}`);
  }

  if (!VALID_PRIORITIES.includes(m.priority as AgentPriority)) {
    errors.push(`priority must be one of: ${VALID_PRIORITIES.join(", ")}`);
  }

  if (!m.payload || typeof m.payload !== "object") {
    errors.push("payload must be a non-null object");
  }

  if (!m.context || typeof m.context !== "object") {
    errors.push("context must be a non-null object");
  } else {
    const ctx = m.context as Record<string, unknown>;
    if (typeof ctx.workflowId !== "string") errors.push("context.workflowId must be a string");
    if (typeof ctx.stepIndex !== "number") errors.push("context.stepIndex must be a number");
    if (typeof ctx.totalSteps !== "number") errors.push("context.totalSteps must be a number");
  }

  if (!m.metadata || typeof m.metadata !== "object") {
    errors.push("metadata must be a non-null object");
  } else {
    const meta = m.metadata as Record<string, unknown>;
    if (typeof meta.correlationId !== "string") errors.push("metadata.correlationId must be a string");
    if (typeof meta.ttl !== "number" || (meta.ttl as number) <= 0) errors.push("metadata.ttl must be a positive number");
    if (typeof meta.retryCount !== "number") errors.push("metadata.retryCount must be a number");
    if (typeof meta.maxRetries !== "number") errors.push("metadata.maxRetries must be a number");
    if (typeof meta.idempotencyKey !== "string") errors.push("metadata.idempotencyKey must be a string");
  }

  return { valid: errors.length === 0, errors };
}
```
