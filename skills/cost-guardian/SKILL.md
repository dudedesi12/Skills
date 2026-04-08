---
name: cost-guardian
description: "Use this skill whenever the user mentions cost, costs, pricing, budget, spending, token usage, API costs, Gemini costs, AI costs, model costs, Supabase costs, Vercel costs, infrastructure costs, billing, 'how much does this cost', 'my bill is too high', 'reduce costs', 'save money', 'cost optimization', token counting, rate limiting for cost, model routing, 'which model should I use', tiered models, usage tracking, usage dashboard, or ANY cost management task — even if they don't explicitly say 'cost'. Running 3 Gemini models + Supabase + Vercel + multiple services without cost controls is a ticking time bomb."
---

# Cost Guardian

Track and control costs across all your services. Focus: AI token costs and holistic budgeting. Cross-reference: vercel-power-user §9 for Vercel plan limits.

## 1. Cost Landscape

| Service | Free Tier | Typical Monthly (Small App) | Cost Driver |
|---------|-----------|---------------------------|-------------|
| **Gemini 2.0 Flash** | 15 RPM free | $0.10-5 per 1M tokens | Token volume |
| **Gemini 2.5 Pro** | 2 RPM free | $1.25-10 per 1M tokens | Token volume |
| **Gemini 2.0 Flash-Lite** | 30 RPM free | $0.02-1 per 1M tokens | Token volume |
| **Supabase** | 500MB DB, 1GB storage | $25/mo (Pro) | Rows, storage, bandwidth |
| **Vercel** | 100GB bandwidth | $20/mo (Pro) | Function invocations, bandwidth |
| **Upstash Redis** | 10K commands/day | $0-10/mo | Command volume |
| **Stripe** | N/A | 1.75% + $0.30/txn (AU) | Transaction volume |
| **Resend** | 100 emails/day | $20/mo (5K/mo) | Email volume |

## 2. Gemini Model Tier Routing

Use the cheapest model that can handle the task:

```typescript
// lib/ai/model-router.ts
type TaskComplexity = "simple" | "medium" | "complex";

const MODEL_MAP: Record<TaskComplexity, string> = {
  simple: "gemini-2.0-flash-lite",   // Cheapest: classification, extraction
  medium: "gemini-2.0-flash",         // Mid-tier: summarization, Q&A
  complex: "gemini-2.5-pro",          // Most capable: analysis, reasoning
};

export function getModelForTask(task: string): { model: string; complexity: TaskComplexity } {
  const simplePatterns = ["classify", "extract", "format", "validate", "translate"];
  const complexPatterns = ["analyze", "recommend", "compare", "assess eligibility", "generate report"];

  const isSimple = simplePatterns.some((p) => task.toLowerCase().includes(p));
  const isComplex = complexPatterns.some((p) => task.toLowerCase().includes(p));

  const complexity: TaskComplexity = isComplex ? "complex" : isSimple ? "simple" : "medium";

  return { model: MODEL_MAP[complexity], complexity };
}
```

```typescript
// lib/ai/gemini-client.ts
import { GoogleGenerativeAI } from "@google/generative-ai";
import { getModelForTask } from "./model-router";
import { trackTokenUsage } from "./usage-tracker";

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);

export async function generateWithTracking(
  task: string,
  prompt: string,
  userId?: string
) {
  const { model: modelName, complexity } = getModelForTask(task);
  const model = genAI.getGenerativeModel({ model: modelName });

  const result = await model.generateContent(prompt);
  const response = result.response;

  // Track usage
  const usage = response.usageMetadata;
  if (usage) {
    await trackTokenUsage({
      userId,
      model: modelName,
      complexity,
      promptTokens: usage.promptTokenCount ?? 0,
      completionTokens: usage.candidatesTokenCount ?? 0,
      totalTokens: usage.totalTokenCount ?? 0,
      feature: task,
    });
  }

  return response.text();
}
```

## 3. Token Usage Tracking

```sql
-- supabase/migrations/xxx_token_usage.sql
CREATE TABLE token_usage (
  id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id uuid REFERENCES auth.users(id) ON DELETE SET NULL,
  model text NOT NULL,
  complexity text NOT NULL,
  prompt_tokens integer NOT NULL DEFAULT 0,
  completion_tokens integer NOT NULL DEFAULT 0,
  total_tokens integer NOT NULL DEFAULT 0,
  estimated_cost_usd numeric(10, 6) NOT NULL DEFAULT 0,
  feature text,
  created_at timestamptz DEFAULT now()
);

CREATE INDEX idx_token_usage_user ON token_usage(user_id, created_at DESC);
CREATE INDEX idx_token_usage_date ON token_usage(created_at DESC);
```

```typescript
// lib/ai/usage-tracker.ts
import { createClient } from "@/lib/supabase/server";

const COST_PER_1M_TOKENS: Record<string, { input: number; output: number }> = {
  "gemini-2.0-flash-lite": { input: 0.075, output: 0.30 },
  "gemini-2.0-flash": { input: 0.10, output: 0.40 },
  "gemini-2.5-pro": { input: 1.25, output: 10.00 },
};

interface UsageRecord {
  userId?: string;
  model: string;
  complexity: string;
  promptTokens: number;
  completionTokens: number;
  totalTokens: number;
  feature: string;
}

export async function trackTokenUsage(record: UsageRecord) {
  const costs = COST_PER_1M_TOKENS[record.model] ?? { input: 0, output: 0 };
  const estimatedCost =
    (record.promptTokens / 1_000_000) * costs.input +
    (record.completionTokens / 1_000_000) * costs.output;

  const supabase = await createClient();
  await supabase.from("token_usage").insert({
    user_id: record.userId,
    model: record.model,
    complexity: record.complexity,
    prompt_tokens: record.promptTokens,
    completion_tokens: record.completionTokens,
    total_tokens: record.totalTokens,
    estimated_cost_usd: estimatedCost,
    feature: record.feature,
  });
}
```

## 4. Supabase Cost Optimization

```typescript
// BAD: Fetching unnecessary data
const { data } = await supabase.from("profiles").select("*"); // All columns, all rows

// GOOD: Selective and paginated
const { data } = await supabase
  .from("profiles")
  .select("id, full_name, email")  // Only needed columns
  .range(0, 19)                    // Paginated
  .order("created_at", { ascending: false });
```

**Supabase cost drivers:**
- Database size (rows × columns) — Clean up old data
- Storage (uploaded files) — Compress images, delete unused
- Bandwidth (data transferred) — Select fewer columns
- Auth MAUs (monthly active users) — Included in plan
- Realtime connections — Close unused subscriptions

## 5. Budget Alerts

```sql
-- supabase/migrations/xxx_budget_config.sql
CREATE TABLE budget_config (
  id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  service text NOT NULL UNIQUE,
  daily_limit_usd numeric(10, 2),
  monthly_limit_usd numeric(10, 2),
  alert_threshold_pct integer DEFAULT 80,
  hard_limit boolean DEFAULT false,
  updated_at timestamptz DEFAULT now()
);

INSERT INTO budget_config (service, daily_limit_usd, monthly_limit_usd, alert_threshold_pct, hard_limit)
VALUES
  ('gemini', 5.00, 100.00, 80, true),
  ('supabase', null, 25.00, 90, false),
  ('vercel', null, 20.00, 90, false);
```

```typescript
// lib/costs/budget-check.ts
import { createClient } from "@/lib/supabase/server";

export async function checkBudget(service: string): Promise<{
  allowed: boolean;
  usage: number;
  limit: number;
  percentUsed: number;
}> {
  const supabase = await createClient();

  const { data: config } = await supabase
    .from("budget_config")
    .select("*")
    .eq("service", service)
    .single();

  if (!config?.daily_limit_usd) return { allowed: true, usage: 0, limit: 0, percentUsed: 0 };

  const today = new Date().toISOString().split("T")[0];
  const { data: usage } = await supabase
    .from("token_usage")
    .select("estimated_cost_usd")
    .gte("created_at", `${today}T00:00:00Z`)
    .lte("created_at", `${today}T23:59:59Z`);

  const totalUsage = (usage ?? []).reduce((sum, r) => sum + Number(r.estimated_cost_usd), 0);
  const percentUsed = Math.round((totalUsage / Number(config.daily_limit_usd)) * 100);

  return {
    allowed: !config.hard_limit || totalUsage < Number(config.daily_limit_usd),
    usage: totalUsage,
    limit: Number(config.daily_limit_usd),
    percentUsed,
  };
}
```

## 6. Cost-Aware Architecture Patterns

### Cache AI Responses

```typescript
// Don't call Gemini twice for the same input
import { Redis } from "@upstash/redis";
const redis = new Redis({ url: process.env.UPSTASH_REDIS_REST_URL!, token: process.env.UPSTASH_REDIS_REST_TOKEN! });

export async function generateWithCache(prompt: string, task: string) {
  const cacheKey = `ai:${task}:${hashPrompt(prompt)}`;
  const cached = await redis.get<string>(cacheKey);
  if (cached) return cached;

  const result = await generateWithTracking(task, prompt);
  await redis.set(cacheKey, result, { ex: 3600 }); // Cache 1 hour
  return result;
}
```

### Batch Operations

```typescript
// BAD: One API call per item
for (const item of items) {
  await generateWithTracking("classify", `Classify: ${item.text}`);
}

// GOOD: Batch into one call
const batchPrompt = items.map((item, i) => `${i + 1}. ${item.text}`).join("\n");
const result = await generateWithTracking(
  "classify",
  `Classify each item:\n${batchPrompt}\n\nReturn JSON array of classifications.`
);
```

## 7. Monthly Cost Review Template

```markdown
## Monthly Cost Review — [Month Year]

### Service Costs
| Service | Budget | Actual | % Used | Action |
|---------|--------|--------|--------|--------|
| Gemini API | $100 | $ | % | |
| Supabase | $25 | $ | % | |
| Vercel | $20 | $ | % | |
| Upstash | $10 | $ | % | |
| Resend | $20 | $ | % | |
| Stripe fees | N/A | $ | N/A | |
| **Total** | **$175** | **$** | | |

### Top Token Consumers
| Feature | Tokens Used | Cost | Model |
|---------|------------|------|-------|
| | | | |

### Action Items
- [ ] Review and clean up unused Supabase storage
- [ ] Check for expensive queries (pg_stat_statements)
- [ ] Verify model routing is using cheapest appropriate model
- [ ] Delete old token_usage records (> 90 days)
```

## Rules

1. **Track every AI call** — Log tokens, model, cost, and feature for every Gemini request.
2. **Use the cheapest model that works** — flash-lite for simple tasks, flash for medium, pro only when needed.
3. **Cache AI responses** — Same input = same output. Cache with Redis.
4. **Set hard limits** — Configure daily budget limits that reject requests when exceeded.
5. **Batch when possible** — One prompt with 10 items is cheaper than 10 separate prompts.
6. **Select specific columns** — Never `select("*")` in production queries.
7. **Clean up storage** — Delete unused files from Supabase Storage monthly.
8. **Monitor daily** — Check the cost dashboard or set up Slack alerts for budget thresholds.
9. **Review monthly** — Use the review template above to catch cost creep.
10. **Optimize prompts** — Shorter prompts = fewer input tokens = lower cost.

See `references/` for detailed guides:
- `gemini-cost-guide.md` — Pricing breakdown, token counting, prompt optimization
- `cost-tracking-dashboard.md` — Building a usage dashboard
- `optimization-patterns.md` — Caching, batching, deduplication patterns
