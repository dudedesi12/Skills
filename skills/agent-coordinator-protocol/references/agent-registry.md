# Agent Registry

How agents register, discover each other, monitor health, and handle stale agents.

## Registry Architecture

The agent registry is a Supabase table (`agent_registry`) that acts as a service directory. Every agent must:

1. **Register** on startup with its capabilities and endpoint
2. **Send heartbeats** every 30 seconds
3. **Deregister** on graceful shutdown (set status to "offline")

The coordinator uses the registry to discover which agents can handle a given capability.

## Full Registry Implementation

```typescript
// lib/agents/registry.ts
import { createClient } from "@supabase/supabase-js";
import type { AgentRegistration, AgentStatus } from "./types";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

const HEARTBEAT_INTERVAL_MS = 30_000;
const STALE_THRESHOLD_MS = 90_000;

// --- Registration ---

export async function registerAgent(config: {
  agentId: string;
  name: string;
  capabilities: string[];
  endpoint: string;
  maxConcurrency?: number;
  metadata?: { version: string; region: string; model?: string };
}): Promise<{ success: boolean; error: string | null }> {
  try {
    const { error } = await supabase.from("agent_registry").upsert(
      {
        agent_id: config.agentId,
        name: config.name,
        capabilities: config.capabilities,
        endpoint: config.endpoint,
        max_concurrency: config.maxConcurrency ?? 5,
        status: "active" as AgentStatus,
        registered_at: new Date().toISOString(),
        last_heartbeat: new Date().toISOString(),
        metadata: config.metadata ?? {},
      },
      { onConflict: "agent_id" }
    );

    if (error) {
      console.error(JSON.stringify({
        level: "error",
        event: "agent_register_failed",
        agentId: config.agentId,
        error: error.message,
      }));
      return { success: false, error: error.message };
    }

    console.log(JSON.stringify({
      level: "info",
      event: "agent_registered",
      agentId: config.agentId,
      name: config.name,
      capabilities: config.capabilities,
    }));

    return { success: true, error: null };
  } catch (err) {
    return { success: false, error: err instanceof Error ? err.message : "Unknown" };
  }
}

// --- Discovery ---

export async function discoverAgents(params: {
  capability?: string;
  status?: AgentStatus;
  excludeStale?: boolean;
}): Promise<{ agents: AgentRegistration[]; error: string | null }> {
  try {
    let query = supabase.from("agent_registry").select("*");

    if (params.capability) {
      query = query.contains("capabilities", [params.capability]);
    }

    if (params.status) {
      query = query.eq("status", params.status);
    }

    if (params.excludeStale !== false) {
      const staleThreshold = new Date(Date.now() - STALE_THRESHOLD_MS).toISOString();
      query = query.gte("last_heartbeat", staleThreshold);
    }

    const { data, error } = await query;

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
      metadata: row.metadata,
    }));

    return { agents, error: null };
  } catch (err) {
    return { agents: [], error: err instanceof Error ? err.message : "Unknown" };
  }
}

// --- Find best agent for a capability ---

export async function findBestAgent(capability: string): Promise<{
  agent: AgentRegistration | null;
  error: string | null;
}> {
  const { agents, error } = await discoverAgents({ capability, status: "active" });

  if (error) {
    return { agent: null, error };
  }

  if (agents.length === 0) {
    return { agent: null, error: `No active agent found with capability: ${capability}` };
  }

  // Pick the agent with the most recent heartbeat (healthiest)
  const sorted = agents.sort(
    (a, b) => new Date(b.lastHeartbeat).getTime() - new Date(a.lastHeartbeat).getTime()
  );

  return { agent: sorted[0], error: null };
}

// --- Get specific agent ---

export async function getAgent(agentId: string): Promise<{
  agent: AgentRegistration | null;
  error: string | null;
}> {
  try {
    const { data, error } = await supabase
      .from("agent_registry")
      .select("*")
      .eq("agent_id", agentId)
      .single();

    if (error || !data) {
      return { agent: null, error: error?.message ?? "Agent not found" };
    }

    return {
      agent: {
        agentId: data.agent_id,
        name: data.name,
        capabilities: data.capabilities,
        endpoint: data.endpoint,
        maxConcurrency: data.max_concurrency,
        registeredAt: data.registered_at,
        lastHeartbeat: data.last_heartbeat,
        status: data.status,
        metadata: data.metadata,
      },
      error: null,
    };
  } catch (err) {
    return { agent: null, error: err instanceof Error ? err.message : "Unknown" };
  }
}

// --- Heartbeat ---

export function startHeartbeat(agentId: string): { stop: () => void } {
  const timer = setInterval(async () => {
    try {
      const { error } = await supabase
        .from("agent_registry")
        .update({ last_heartbeat: new Date().toISOString() })
        .eq("agent_id", agentId)
        .eq("status", "active");

      if (error) {
        console.error(JSON.stringify({
          level: "error",
          event: "heartbeat_failed",
          agentId,
          error: error.message,
        }));
      }
    } catch (err) {
      console.error(JSON.stringify({
        level: "error",
        event: "heartbeat_exception",
        agentId,
        error: err instanceof Error ? err.message : "Unknown",
      }));
    }
  }, HEARTBEAT_INTERVAL_MS);

  return {
    stop: async () => {
      clearInterval(timer);
      try {
        await supabase
          .from("agent_registry")
          .update({ status: "offline" as AgentStatus })
          .eq("agent_id", agentId);

        console.log(JSON.stringify({
          level: "info",
          event: "agent_deregistered",
          agentId,
        }));
      } catch (err) {
        console.error(JSON.stringify({
          level: "error",
          event: "deregister_failed",
          agentId,
          error: err instanceof Error ? err.message : "Unknown",
        }));
      }
    },
  };
}

// --- Deregistration ---

export async function deregisterAgent(agentId: string): Promise<{
  success: boolean;
  error: string | null;
}> {
  try {
    // Set to draining first to stop new work
    const { error: drainError } = await supabase
      .from("agent_registry")
      .update({ status: "draining" as AgentStatus })
      .eq("agent_id", agentId);

    if (drainError) {
      return { success: false, error: drainError.message };
    }

    // Wait for in-flight messages to complete (max 10 seconds)
    const deadline = Date.now() + 10_000;
    while (Date.now() < deadline) {
      const { data: pending } = await supabase
        .from("agent_message_queue")
        .select("id")
        .eq("to_agent", agentId)
        .eq("status", "processing")
        .limit(1);

      if (!pending || pending.length === 0) break;
      await new Promise((resolve) => setTimeout(resolve, 1000));
    }

    // Mark as offline
    const { error: offlineError } = await supabase
      .from("agent_registry")
      .update({ status: "offline" as AgentStatus })
      .eq("agent_id", agentId);

    if (offlineError) {
      return { success: false, error: offlineError.message };
    }

    console.log(JSON.stringify({
      level: "info",
      event: "agent_deregistered",
      agentId,
    }));

    return { success: true, error: null };
  } catch (err) {
    return { success: false, error: err instanceof Error ? err.message : "Unknown" };
  }
}

// --- Stale Agent Cleanup ---

export async function cleanupStaleAgents(): Promise<{
  count: number;
  error: string | null;
}> {
  try {
    const staleThreshold = new Date(Date.now() - STALE_THRESHOLD_MS).toISOString();

    const { data, error } = await supabase
      .from("agent_registry")
      .update({ status: "offline" as AgentStatus })
      .eq("status", "active")
      .lt("last_heartbeat", staleThreshold)
      .select("agent_id");

    if (error) {
      return { count: 0, error: error.message };
    }

    const count = data?.length ?? 0;

    if (count > 0) {
      console.log(JSON.stringify({
        level: "warn",
        event: "stale_agents_cleaned",
        count,
        agentIds: data?.map((r) => r.agent_id),
      }));
    }

    return { count, error: null };
  } catch (err) {
    return { count: 0, error: err instanceof Error ? err.message : "Unknown" };
  }
}
```

## Agent Startup Pattern

How to wire everything together when an agent starts:

```typescript
// lib/agents/agent-bootstrap.ts
import { registerAgent, startHeartbeat, deregisterAgent } from "./registry";
import { subscribeToAgentEvents } from "./pubsub";
import { dequeueMessages } from "./queue";
import { createAgentLogger } from "./logger";
import { randomUUID } from "crypto";

export async function bootstrapAgent(config: {
  agentId: string;
  name: string;
  capabilities: string[];
  endpoint: string;
  maxConcurrency?: number;
  onMessage: (message: Record<string, unknown>) => Promise<void>;
  pollIntervalMs?: number;
}): Promise<{ shutdown: () => Promise<void> }> {
  const logger = createAgentLogger(config.agentId, randomUUID());

  // 1. Register with the coordinator
  const { success, error } = await registerAgent({
    agentId: config.agentId,
    name: config.name,
    capabilities: config.capabilities,
    endpoint: config.endpoint,
    maxConcurrency: config.maxConcurrency,
  });

  if (!success) {
    logger.error("bootstrap_failed", { error: error ?? "Registration failed" });
    throw new Error(`Agent registration failed: ${error}`);
  }

  // 2. Start heartbeat
  const heartbeat = startHeartbeat(config.agentId);

  // 3. Subscribe to real-time events
  const subscription = subscribeToAgentEvents(config.agentId, (message) => {
    config.onMessage(message).catch((err) => {
      logger.error("message_handler_error", {
        error: err instanceof Error ? err.message : "Unknown",
      });
    });
  });

  // 4. Start polling the queue
  const pollInterval = setInterval(async () => {
    try {
      const { messages, error: dequeueError } = await dequeueMessages(
        config.agentId,
        config.maxConcurrency ?? 5
      );

      if (dequeueError) {
        logger.warn("dequeue_error", { error: dequeueError });
        return;
      }

      for (const msg of messages) {
        config.onMessage(msg as unknown as Record<string, unknown>).catch((err) => {
          logger.error("queued_message_handler_error", {
            error: err instanceof Error ? err.message : "Unknown",
            metadata: { messageId: msg.id },
          });
        });
      }
    } catch (err) {
      logger.error("poll_error", {
        error: err instanceof Error ? err.message : "Unknown",
      });
    }
  }, config.pollIntervalMs ?? 5_000);

  logger.info("agent_bootstrapped", {
    metadata: {
      capabilities: config.capabilities,
      maxConcurrency: config.maxConcurrency ?? 5,
    },
  });

  // 5. Return shutdown handle
  return {
    shutdown: async () => {
      clearInterval(pollInterval);
      subscription.unsubscribe();
      heartbeat.stop();
      await deregisterAgent(config.agentId);
      logger.info("agent_shutdown_complete");
    },
  };
}
```
