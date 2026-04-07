# Saga Pattern

Long-running workflows with compensation for rollback when steps fail.

## How It Works

A saga is a sequence of steps where each step has:
- **execute** — the forward action
- **compensate** — the undo action if a later step fails

If step 3 of 5 fails, the saga runs compensate for steps 2 and 1 (in reverse order).

## Full Implementation

```typescript
// lib/agents/saga.ts
import { createClient } from "@supabase/supabase-js";
import { randomUUID } from "crypto";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export type SagaStatus = "running" | "completed" | "failed" | "compensating";

export interface SagaStep {
  name: string;
  agentId: string;
  timeoutMs?: number;
  execute: (context: Record<string, unknown>) => Promise<Record<string, unknown>>;
  compensate: (context: Record<string, unknown>) => Promise<void>;
}

interface SagaState {
  sagaId: string;
  status: SagaStatus;
  currentStep: number;
  totalSteps: number;
  context: Record<string, unknown>;
  completedSteps: string[];
  failedStep: string | null;
  failureReason: string | null;
  compensationErrors: Array<{ step: string; error: string }>;
  startedAt: string;
  completedAt: string | null;
}

export class SagaOrchestrator {
  private sagaId: string;
  private state: SagaState;

  constructor(
    private steps: SagaStep[],
    private initialContext: Record<string, unknown>
  ) {
    this.sagaId = randomUUID();
    this.state = {
      sagaId: this.sagaId,
      status: "running",
      currentStep: 0,
      totalSteps: steps.length,
      context: { ...initialContext },
      completedSteps: [],
      failedStep: null,
      failureReason: null,
      compensationErrors: [],
      startedAt: new Date().toISOString(),
      completedAt: null,
    };
  }

  async run(): Promise<SagaState> {
    await this.persistState();

    for (let i = 0; i < this.steps.length; i++) {
      const step = this.steps[i];
      this.state.currentStep = i;
      await this.persistState();

      console.log(JSON.stringify({
        level: "info",
        event: "saga_step_start",
        sagaId: this.sagaId,
        step: step.name,
        agentId: step.agentId,
        stepIndex: i,
        totalSteps: this.steps.length,
      }));

      try {
        const result = await this.executeWithTimeout(step);
        this.state.context = { ...this.state.context, ...result };
        this.state.completedSteps.push(step.name);

        console.log(JSON.stringify({
          level: "info",
          event: "saga_step_complete",
          sagaId: this.sagaId,
          step: step.name,
          stepIndex: i,
        }));
      } catch (err) {
        const errorMessage = err instanceof Error ? err.message : "Unknown error";

        console.error(JSON.stringify({
          level: "error",
          event: "saga_step_failed",
          sagaId: this.sagaId,
          step: step.name,
          stepIndex: i,
          error: errorMessage,
        }));

        this.state.status = "compensating";
        this.state.failedStep = step.name;
        this.state.failureReason = errorMessage;
        await this.persistState();

        await this.compensate();

        this.state.status = "failed";
        this.state.completedAt = new Date().toISOString();
        await this.persistState();

        return this.state;
      }
    }

    this.state.status = "completed";
    this.state.completedAt = new Date().toISOString();
    await this.persistState();

    console.log(JSON.stringify({
      level: "info",
      event: "saga_completed",
      sagaId: this.sagaId,
      totalSteps: this.steps.length,
    }));

    return this.state;
  }

  private async executeWithTimeout(step: SagaStep): Promise<Record<string, unknown>> {
    const timeoutMs = step.timeoutMs ?? 30_000;

    return new Promise<Record<string, unknown>>((resolve, reject) => {
      const timer = setTimeout(() => {
        reject(new Error(`Step "${step.name}" timed out after ${timeoutMs}ms`));
      }, timeoutMs);

      step
        .execute(this.state.context)
        .then((result) => {
          clearTimeout(timer);
          resolve(result);
        })
        .catch((err) => {
          clearTimeout(timer);
          reject(err);
        });
    });
  }

  private async compensate(): Promise<void> {
    const stepsToCompensate = [...this.state.completedSteps].reverse();

    console.log(JSON.stringify({
      level: "info",
      event: "saga_compensation_start",
      sagaId: this.sagaId,
      stepsToCompensate,
    }));

    for (const stepName of stepsToCompensate) {
      const step = this.steps.find((s) => s.name === stepName);
      if (!step) continue;

      try {
        await step.compensate(this.state.context);

        console.log(JSON.stringify({
          level: "info",
          event: "saga_compensate_step_complete",
          sagaId: this.sagaId,
          step: stepName,
        }));
      } catch (err) {
        const errorMessage = err instanceof Error ? err.message : "Unknown compensation error";

        this.state.compensationErrors.push({ step: stepName, error: errorMessage });

        console.error(JSON.stringify({
          level: "error",
          event: "saga_compensate_step_failed",
          sagaId: this.sagaId,
          step: stepName,
          error: errorMessage,
        }));
      }
    }
  }

  private async persistState(): Promise<void> {
    try {
      await supabase.from("workflow_contexts").upsert(
        {
          workflow_id: this.sagaId,
          context: this.state.context,
          status: this.state.status,
          current_step: this.state.currentStep,
          total_steps: this.state.totalSteps,
          version: this.state.currentStep + 1,
          updated_at: new Date().toISOString(),
          completed_at: this.state.completedAt,
          error: this.state.failedStep
            ? { step: this.state.failedStep, reason: this.state.failureReason }
            : null,
        },
        { onConflict: "workflow_id" }
      );
    } catch (err) {
      console.error(JSON.stringify({
        level: "error",
        event: "saga_persist_failed",
        sagaId: this.sagaId,
        error: err instanceof Error ? err.message : "Unknown",
      }));
    }
  }

  getState(): SagaState {
    return { ...this.state };
  }
}
```

## Usage Example

A content generation workflow that researches, writes, reviews, and publishes:

```typescript
// lib/agents/workflows/content-generation.ts
import { SagaOrchestrator, SagaStep } from "../saga";
import { callAgent } from "../agent-caller";

export async function runContentGenerationWorkflow(topic: string): Promise<{
  success: boolean;
  content: string | null;
  error: string | null;
}> {
  const steps: SagaStep[] = [
    {
      name: "research",
      agentId: "research-agent",
      timeoutMs: 60_000,
      execute: async (ctx) => {
        const { result, error } = await callAgent(
          "research-agent",
          `${process.env.AGENT_BASE_URL}/api/agents/research`,
          { topic: ctx.topic },
          ctx.correlationId as string
        );
        if (error) throw new Error(error);
        return { researchData: result };
      },
      compensate: async () => {
        // Research is read-only, nothing to undo
      },
    },
    {
      name: "write-draft",
      agentId: "writer-agent",
      timeoutMs: 120_000,
      execute: async (ctx) => {
        const { result, error } = await callAgent(
          "writer-agent",
          `${process.env.AGENT_BASE_URL}/api/agents/writer`,
          { topic: ctx.topic, research: ctx.researchData },
          ctx.correlationId as string
        );
        if (error) throw new Error(error);
        return { draftId: (result as Record<string, unknown>).draftId, draft: result };
      },
      compensate: async (ctx) => {
        // Delete the draft
        await callAgent(
          "writer-agent",
          `${process.env.AGENT_BASE_URL}/api/agents/writer/delete`,
          { draftId: ctx.draftId },
          ctx.correlationId as string
        );
      },
    },
    {
      name: "review",
      agentId: "reviewer-agent",
      timeoutMs: 60_000,
      execute: async (ctx) => {
        const { result, error } = await callAgent(
          "reviewer-agent",
          `${process.env.AGENT_BASE_URL}/api/agents/reviewer`,
          { draft: ctx.draft },
          ctx.correlationId as string
        );
        if (error) throw new Error(error);
        const review = result as Record<string, unknown>;
        if (review.approved !== true) {
          throw new Error(`Review rejected: ${review.reason}`);
        }
        return { reviewResult: result };
      },
      compensate: async () => {
        // Review is read-only, nothing to undo
      },
    },
    {
      name: "publish",
      agentId: "publisher-agent",
      timeoutMs: 30_000,
      execute: async (ctx) => {
        const { result, error } = await callAgent(
          "publisher-agent",
          `${process.env.AGENT_BASE_URL}/api/agents/publisher`,
          { draft: ctx.draft, draftId: ctx.draftId },
          ctx.correlationId as string
        );
        if (error) throw new Error(error);
        return { publishedUrl: (result as Record<string, unknown>).url };
      },
      compensate: async (ctx) => {
        // Unpublish the content
        await callAgent(
          "publisher-agent",
          `${process.env.AGENT_BASE_URL}/api/agents/publisher/unpublish`,
          { draftId: ctx.draftId },
          ctx.correlationId as string
        );
      },
    },
  ];

  const saga = new SagaOrchestrator(steps, {
    topic,
    correlationId: crypto.randomUUID(),
  });

  const finalState = await saga.run();

  if (finalState.status === "completed") {
    return {
      success: true,
      content: finalState.context.publishedUrl as string,
      error: null,
    };
  }

  return {
    success: false,
    content: null,
    error: `Workflow failed at step "${finalState.failedStep}": ${finalState.failureReason}`,
  };
}
```
