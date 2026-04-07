---
name: master-orchestrator-agent
description: >-
  Use this skill whenever the user mentions orchestrating multiple agents, coordinating AI tasks,
  building an agent system, workflow automation, task routing, multi-agent architecture, agent
  communication, or needs to build a system where different AI agents work together. Also trigger
  when user says 'coordinate', 'orchestrate', 'pipeline', 'workflow', 'agent team', 'multi-step
  AI task', or anything about agents working together — even if they don't use the word 'orchestrator'.
---

# Master Orchestrator Agent

The brain of your multi-agent system. Routes tasks to specialist agents, manages workflows, tracks state, and handles failures — all powered by Gemini.

## Architecture Overview

```
User Request → Master Agent (Gemini 2.0 Flash)
  ├── Classify task type
  ├── Select agent(s) from registry
  ├── Execute workflow (sequential / parallel / conditional)
  ├── Track state in Supabase
  └── Return aggregated result
```

**Model Selection:**
- `gemini-2.5-pro` → Complex reasoning, multi-step planning, quality review
- `gemini-2.0-flash` → Task routing, classification, user-facing responses
- `flash-lite` → Simple classification, intent detection

## 1. Supabase Schema

Run this migration to set up the orchestrator tables.

```sql
-- file: supabase/migrations/001_orchestrator.sql

-- Agent registry
create table agents (
  id uuid primary key default gen_random_uuid(),
  name text unique not null,
  description text not null,
  capabilities text[] not null default '{}',
  model_tier text not null check (model_tier in ('pro', 'flash', 'lite')),
  endpoint text not null,
  status text not null default 'active' check (status in ('active', 'inactive', 'error')),
  last_heartbeat timestamptz default now(),
  created_at timestamptz default now()
);

-- Workflows
create table workflows (
  id uuid primary key default gen_random_uuid(),
  type text not null check (type in ('sequential', 'parallel', 'conditional', 'iterative')),
  status text not null default 'pending'
    check (status in ('pending', 'running', 'completed', 'failed', 'cancelled')),
  input jsonb not null,
  output jsonb,
  error text,
  context jsonb default '{}',
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- Individual tasks within workflows
create table tasks (
  id uuid primary key default gen_random_uuid(),
  workflow_id uuid references workflows(id) on delete cascade,
  agent_name text references agents(name),
  status text not null default 'pending'
    check (status in ('pending', 'running', 'completed', 'failed', 'retrying')),
  input jsonb not null,
  output jsonb,
  error text,
  retry_count int default 0,
  max_retries int default 3,
  token_usage jsonb default '{"input": 0, "output": 0}',
  started_at timestamptz,
  completed_at timestamptz,
  created_at timestamptz default now()
);

-- Dead letter queue
create table dead_letter_queue (
  id uuid primary key default gen_random_uuid(),
  task_id uuid references tasks(id),
  workflow_id uuid references workflows(id),
  error text not null,
  original_input jsonb not null,
  created_at timestamptz default now()
);

-- RLS
alter table agents enable row level security;
alter table workflows enable row level security;
alter table tasks enable row level security;
alter table dead_letter_queue enable row level security;

-- Service role only (these are internal tables)
create policy "Service role only" on agents for all using (auth.role() = 'service_role');
create policy "Service role only" on workflows for all using (auth.role() = 'service_role');
create policy "Service role only" on tasks for all using (auth.role() = 'service_role');
create policy "Service role only" on dead_letter_queue for all using (auth.role() = 'service_role');
```

## 2. Agent Registry

```typescript
// file: lib/agents/registry.ts
import { createClient } from '@/lib/supabase/server';

export interface AgentConfig {
  name: string;
  description: string;
  capabilities: string[];
  modelTier: 'pro' | 'flash' | 'lite';
  endpoint: string;
}

export async function registerAgent(config: AgentConfig) {
  const supabase = await createClient();
  const { error } = await supabase
    .from('agents')
    .upsert({
      name: config.name,
      description: config.description,
      capabilities: config.capabilities,
      model_tier: config.modelTier,
      endpoint: config.endpoint,
      status: 'active',
      last_heartbeat: new Date().toISOString(),
    }, { onConflict: 'name' });

  if (error) throw new Error(`Failed to register agent: ${error.message}`);
}

export async function discoverAgents(capability?: string) {
  const supabase = await createClient();
  let query = supabase
    .from('agents')
    .select('*')
    .eq('status', 'active');

  if (capability) {
    query = query.contains('capabilities', [capability]);
  }

  const { data, error } = await query;
  if (error) throw new Error(`Failed to discover agents: ${error.message}`);
  return data ?? [];
}

export async function heartbeat(agentName: string) {
  const supabase = await createClient();
  await supabase
    .from('agents')
    .update({ last_heartbeat: new Date().toISOString() })
    .eq('name', agentName);
}
```

## 3. Task Router (The Brain)

```typescript
// file: lib/agents/router.ts
import { GoogleGenerativeAI } from '@google/generative-ai';
import { discoverAgents } from './registry';

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);

interface RoutingDecision {
  workflow_type: 'sequential' | 'parallel' | 'conditional';
  steps: { agent: string; input: Record<string, unknown> }[];
  reasoning: string;
}

export async function routeTask(userRequest: string): Promise<RoutingDecision> {
  const agents = await discoverAgents();
  const agentList = agents.map(a => `- ${a.name}: ${a.description} [${a.capabilities.join(', ')}]`).join('\n');

  const model = genAI.getGenerativeModel({ model: 'gemini-2.0-flash' });

  const result = await model.generateContent({
    contents: [{ role: 'user', parts: [{ text: userRequest }] }],
    systemInstruction: {
      role: 'system',
      parts: [{ text: `You are a task router. Given a user request and available agents, decide:
1. Which agents to use
2. Whether to run them sequentially, in parallel, or conditionally
3. What input each agent needs

Available agents:
${agentList}

Respond in JSON:
{
  "workflow_type": "sequential" | "parallel" | "conditional",
  "steps": [{ "agent": "agent-name", "input": { ... } }],
  "reasoning": "brief explanation"
}` }]
    },
    generationConfig: {
      responseMimeType: 'application/json',
    },
  });

  return JSON.parse(result.response.text()) as RoutingDecision;
}
```

## 4. Workflow Engine

```typescript
// file: lib/agents/workflow-engine.ts
import { createClient } from '@/lib/supabase/server';

interface TaskStep {
  agent: string;
  input: Record<string, unknown>;
}

export async function executeWorkflow(
  type: 'sequential' | 'parallel' | 'conditional',
  steps: TaskStep[],
  context: Record<string, unknown> = {}
) {
  const supabase = await createClient();

  // Create workflow record
  const { data: workflow, error: wfError } = await supabase
    .from('workflows')
    .insert({ type, status: 'running', input: { steps, context }, context })
    .select()
    .single();

  if (wfError || !workflow) throw new Error(`Workflow creation failed: ${wfError?.message}`);

  try {
    let results: Record<string, unknown>[] = [];

    if (type === 'parallel') {
      results = await executeParallel(workflow.id, steps, context, supabase);
    } else {
      results = await executeSequential(workflow.id, steps, context, supabase);
    }

    await supabase
      .from('workflows')
      .update({ status: 'completed', output: results, updated_at: new Date().toISOString() })
      .eq('id', workflow.id);

    return { workflowId: workflow.id, results };
  } catch (err) {
    const message = err instanceof Error ? err.message : 'Unknown error';
    await supabase
      .from('workflows')
      .update({ status: 'failed', error: message, updated_at: new Date().toISOString() })
      .eq('id', workflow.id);
    throw err;
  }
}

async function executeSequential(
  workflowId: string,
  steps: TaskStep[],
  context: Record<string, unknown>,
  supabase: Awaited<ReturnType<typeof createClient>>
) {
  const results: Record<string, unknown>[] = [];

  for (const step of steps) {
    const enrichedInput = { ...step.input, previousResults: results, context };
    const result = await executeTaskWithRetry(workflowId, step.agent, enrichedInput, supabase);
    results.push(result);
    context = { ...context, [`${step.agent}_result`]: result };
  }

  return results;
}

async function executeParallel(
  workflowId: string,
  steps: TaskStep[],
  context: Record<string, unknown>,
  supabase: Awaited<ReturnType<typeof createClient>>
) {
  const promises = steps.map(step =>
    executeTaskWithRetry(workflowId, step.agent, { ...step.input, context }, supabase)
  );
  return Promise.all(promises);
}

async function executeTaskWithRetry(
  workflowId: string,
  agentName: string,
  input: Record<string, unknown>,
  supabase: Awaited<ReturnType<typeof createClient>>,
  maxRetries = 3
): Promise<Record<string, unknown>> {
  const { data: task } = await supabase
    .from('tasks')
    .insert({ workflow_id: workflowId, agent_name: agentName, status: 'running', input, started_at: new Date().toISOString() })
    .select()
    .single();

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(`${process.env.NEXT_PUBLIC_APP_URL}/api/agents/${agentName}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${process.env.INTERNAL_API_KEY}` },
        body: JSON.stringify(input),
      });

      if (!response.ok) throw new Error(`Agent ${agentName} returned ${response.status}`);

      const result = await response.json();

      await supabase
        .from('tasks')
        .update({ status: 'completed', output: result, completed_at: new Date().toISOString(), retry_count: attempt })
        .eq('id', task!.id);

      return result;
    } catch (err) {
      if (attempt === maxRetries) {
        const message = err instanceof Error ? err.message : 'Unknown error';
        await supabase.from('tasks').update({ status: 'failed', error: message, retry_count: attempt }).eq('id', task!.id);
        await supabase.from('dead_letter_queue').insert({ task_id: task!.id, workflow_id: workflowId, error: message, original_input: input });
        throw err;
      }
      await new Promise(r => setTimeout(r, Math.pow(2, attempt) * 1000));
    }
  }
  throw new Error('Unreachable');
}
```

## 5. API Route — Orchestrator Entry Point

```typescript
// file: app/api/orchestrate/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { routeTask } from '@/lib/agents/router';
import { executeWorkflow } from '@/lib/agents/workflow-engine';

export async function POST(request: NextRequest) {
  try {
    const authHeader = request.headers.get('authorization');
    if (authHeader !== `Bearer ${process.env.INTERNAL_API_KEY}`) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    }

    const { task, context } = await request.json();
    if (!task || typeof task !== 'string') {
      return NextResponse.json({ error: 'Missing task string' }, { status: 400 });
    }

    // Step 1: Route the task
    const decision = await routeTask(task);

    // Step 2: Execute the workflow
    const result = await executeWorkflow(decision.workflow_type, decision.steps, context ?? {});

    return NextResponse.json({
      success: true,
      routing: { type: decision.workflow_type, reasoning: decision.reasoning },
      result,
    });
  } catch (err) {
    const message = err instanceof Error ? err.message : 'Internal error';
    return NextResponse.json({ error: message }, { status: 500 });
  }
}
```

## 6. Cost Tracking

```typescript
// file: lib/agents/cost-tracker.ts
import { createClient } from '@/lib/supabase/server';

const PRICING = {
  'gemini-2.5-pro': { input: 1.25 / 1_000_000, output: 10.0 / 1_000_000 },
  'gemini-2.0-flash': { input: 0.10 / 1_000_000, output: 0.40 / 1_000_000 },
  'flash-lite': { input: 0.02 / 1_000_000, output: 0.05 / 1_000_000 },
};

export function estimateCost(model: keyof typeof PRICING, inputTokens: number, outputTokens: number) {
  const rate = PRICING[model];
  return {
    inputCost: inputTokens * rate.input,
    outputCost: outputTokens * rate.output,
    totalCost: inputTokens * rate.input + outputTokens * rate.output,
  };
}

export async function logTokenUsage(taskId: string, model: string, inputTokens: number, outputTokens: number) {
  const supabase = await createClient();
  await supabase
    .from('tasks')
    .update({ token_usage: { model, input: inputTokens, output: outputTokens } })
    .eq('id', taskId);
}
```

## 7. Health Check Cron

```typescript
// file: app/api/cron/agent-health/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { createClient } from '@/lib/supabase/server';

export async function GET(request: NextRequest) {
  if (request.headers.get('authorization') !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const supabase = await createClient();
  const fiveMinutesAgo = new Date(Date.now() - 5 * 60 * 1000).toISOString();

  // Mark agents with stale heartbeats as inactive
  await supabase
    .from('agents')
    .update({ status: 'inactive' })
    .eq('status', 'active')
    .lt('last_heartbeat', fiveMinutesAgo);

  // Get current agent statuses
  const { data: agents } = await supabase.from('agents').select('name, status, last_heartbeat');

  return NextResponse.json({ checked_at: new Date().toISOString(), agents });
}
```

Add to `vercel.json`:
```json
{
  "crons": [{ "path": "/api/cron/agent-health", "schedule": "*/5 * * * *" }]
}
```

## Quick Reference

| Pattern | When to Use | Example |
|---------|-------------|---------|
| Sequential | Step B needs Step A's output | Scrape → Parse → Store |
| Parallel | Steps are independent | [Scrape Site A, Scrape Site B] → Merge |
| Conditional | Different paths based on input | If content → Content Agent, if data → Analysis Agent |
| Iterative | Quality loop until threshold | Generate → Review → Refine → Review... |
| Escalation | Fallback on failure | Flash → Pro → Human review |

> For detailed workflow code templates, see `references/workflow-patterns.md`.
> For the complete Supabase schema, see `references/state-schema.md`.
> For master agent system prompts, see `references/prompts.md`.
