# Workflow Patterns — Code Templates

## 1. Sequential Workflow

Tasks run one after another. Each task receives the previous task's output.

```typescript
// Example: Scrape website → Parse data → Store in Supabase
const result = await executeWorkflow('sequential', [
  { agent: 'scraper-agent', input: { url: 'https://example.gov.au/data' } },
  { agent: 'parser-agent', input: { format: 'immigration_visa' } },
  // parser-agent automatically receives scraper output via previousResults
]);
```

## 2. Parallel Workflow

Tasks run simultaneously. Use when tasks are independent.

```typescript
// Example: Scrape 3 different sites at once
const result = await executeWorkflow('parallel', [
  { agent: 'scraper-agent', input: { url: 'https://site-a.com', label: 'site-a' } },
  { agent: 'scraper-agent', input: { url: 'https://site-b.com', label: 'site-b' } },
  { agent: 'scraper-agent', input: { url: 'https://site-c.com', label: 'site-c' } },
]);
// results = [siteAData, siteBData, siteCData]
```

## 3. Conditional Workflow

```typescript
// file: lib/agents/conditional-workflow.ts
export async function executeConditional(
  workflowId: string,
  input: Record<string, unknown>,
  condition: (input: Record<string, unknown>) => string,
  branches: Record<string, TaskStep[]>
) {
  const branch = condition(input);
  const steps = branches[branch];

  if (!steps) throw new Error(`No branch found for condition: ${branch}`);

  return executeSequential(workflowId, steps, input, await createClient());
}

// Usage:
await executeConditional(
  workflowId,
  { contentType: 'pdf', url: '...' },
  (input) => input.contentType === 'pdf' ? 'pdf-path' : 'html-path',
  {
    'pdf-path': [
      { agent: 'parser-agent', input: { mode: 'pdf' } },
      { agent: 'content-agent', input: { action: 'summarize' } },
    ],
    'html-path': [
      { agent: 'scraper-agent', input: { mode: 'html' } },
      { agent: 'parser-agent', input: { mode: 'html' } },
    ],
  }
);
```

## 4. Iterative Workflow (Quality Loop)

```typescript
// file: lib/agents/iterative-workflow.ts
export async function executeIterative(
  workflowId: string,
  workerAgent: string,
  reviewerAgent: string,
  input: Record<string, unknown>,
  maxIterations = 3,
  qualityThreshold = 0.8
) {
  let currentInput = input;
  let iteration = 0;

  while (iteration < maxIterations) {
    // Worker produces output
    const work = await executeTaskWithRetry(workflowId, workerAgent, currentInput, await createClient());

    // Reviewer scores the output
    const review = await executeTaskWithRetry(workflowId, reviewerAgent, {
      original_request: input,
      output: work,
      iteration,
    }, await createClient());

    const score = (review as { quality_score: number }).quality_score;
    if (score >= qualityThreshold) return work;

    // Feed review back to worker
    currentInput = { ...input, previousAttempt: work, feedback: review, iteration: iteration + 1 };
    iteration++;
  }

  throw new Error(`Quality threshold not met after ${maxIterations} iterations`);
}

// Usage: Generate content, review, refine until quality >= 0.8
await executeIterative(workflowId, 'content-agent', 'moderation-agent', {
  task: 'Write a blog post about Australian visa changes',
});
```

## 5. Escalation Workflow (Fallback Chain)

```typescript
// file: lib/agents/escalation-workflow.ts
export async function executeWithEscalation(
  workflowId: string,
  agentChain: string[],
  input: Record<string, unknown>
) {
  const supabase = await createClient();

  for (const agent of agentChain) {
    try {
      return await executeTaskWithRetry(workflowId, agent, input, supabase, 1); // Only 1 retry per agent
    } catch (err) {
      console.warn(`Agent ${agent} failed, escalating...`, err);
      input = { ...input, escalatedFrom: agent, escalationReason: (err as Error).message };
    }
  }

  // All agents failed — notify human
  await supabase.from('dead_letter_queue').insert({
    workflow_id: workflowId,
    error: 'All agents in escalation chain failed',
    original_input: input,
  });

  throw new Error('Escalation chain exhausted — requires human intervention');
}

// Usage: Try Flash first (cheap), fall back to Pro (expensive)
await executeWithEscalation(workflowId, ['fast-parser', 'deep-parser', 'manual-review'], {
  document: pdfContent,
});
```
