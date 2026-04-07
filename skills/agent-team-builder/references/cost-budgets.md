# Token Usage Estimates and Cost Budgets

Cost estimates per agent type based on Gemini API pricing and typical usage patterns.

## Gemini API Pricing (as of 2026)

| Model | Input (per 1M tokens) | Output (per 1M tokens) |
|-------|----------------------|----------------------|
| gemini-2.0-flash | $0.10 | $0.40 |
| gemini-2.5-pro | $1.25 | $10.00 |

## Per-Request Token Estimates

| Agent | Model | Avg Input Tokens | Avg Output Tokens | Cost per Request |
|-------|-------|-----------------|-------------------|-----------------|
| Scraper | flash | 800 | 400 | $0.00024 |
| Parser | flash | 1,200 | 600 | $0.00036 |
| Content | pro | 500 | 2,500 | $0.02563 |
| Analysis | pro | 2,000 | 1,500 | $0.01750 |
| Notification | flash | 300 | 200 | $0.00011 |
| Moderation | flash | 500 | 200 | $0.00013 |
| Search | flash | 400 | 500 | $0.00024 |
| Report | pro | 3,000 | 4,000 | $0.04375 |
| Translation | flash | 600 | 700 | $0.00034 |
| Customer Support | pro | 2,000 | 800 | $0.01050 |

## Monthly Cost Projections

Assumes a SaaS with moderate usage. Adjust multipliers for your actual traffic.

### Low Usage (100 requests/day per agent)

| Agent | Requests/Month | Monthly Cost |
|-------|---------------|-------------|
| Scraper | 3,000 | $0.72 |
| Parser | 3,000 | $1.08 |
| Content | 3,000 | $76.88 |
| Analysis | 3,000 | $52.50 |
| Notification | 3,000 | $0.33 |
| Moderation | 3,000 | $0.39 |
| Search | 3,000 | $0.72 |
| Report | 3,000 | $131.25 |
| Translation | 3,000 | $1.02 |
| Customer Support | 3,000 | $31.50 |
| **Total** | **30,000** | **$296.39** |

### Medium Usage (1,000 requests/day per agent)

| Agent | Requests/Month | Monthly Cost |
|-------|---------------|-------------|
| Scraper | 30,000 | $7.20 |
| Parser | 30,000 | $10.80 |
| Content | 30,000 | $768.75 |
| Analysis | 30,000 | $525.00 |
| Notification | 30,000 | $3.30 |
| Moderation | 30,000 | $3.90 |
| Search | 30,000 | $7.20 |
| Report | 30,000 | $1,312.50 |
| Translation | 30,000 | $10.20 |
| Customer Support | 30,000 | $315.00 |
| **Total** | **300,000** | **$2,963.85** |

## Recommended Budget Limits

Set these in the `agent_budgets` table:

```sql
-- Conservative budgets for a startup
INSERT INTO agent_budgets (agent_name, daily_limit_usd, monthly_limit_usd) VALUES
  ('scraper', 1.00, 20.00),
  ('parser', 1.00, 20.00),
  ('content', 10.00, 200.00),
  ('analysis', 8.00, 150.00),
  ('notification', 0.50, 10.00),
  ('moderation', 0.50, 10.00),
  ('search', 1.00, 20.00),
  ('report', 15.00, 350.00),
  ('translation', 1.00, 20.00),
  ('customer-support', 5.00, 100.00);
```

## Cost Optimization Strategies

### 1. Use Flash for Simple Tasks
Agents that do classification, short-form output, or template-based work should use `gemini-2.0-flash`. Reserve `gemini-2.5-pro` for agents that need deep reasoning (content, analysis, reports, support).

### 2. Cache Common Requests
For the Search and Customer Support agents, cache results in Supabase:

```typescript
// lib/agents/cache.ts
import { createClient } from "@supabase/supabase-js";
import crypto from "crypto";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export async function getCachedResult(
  agentName: string,
  input: Record<string, unknown>,
  maxAgeMs: number = 3600000
): Promise<unknown | null> {
  const inputHash = crypto
    .createHash("sha256")
    .update(JSON.stringify(input))
    .digest("hex");

  const { data } = await supabase
    .from("agent_tasks")
    .select("output, created_at")
    .eq("agent_name", agentName)
    .eq("status", "completed")
    .order("created_at", { ascending: false })
    .limit(1);

  if (!data || data.length === 0) return null;

  const age = Date.now() - new Date(data[0].created_at).getTime();
  if (age > maxAgeMs) return null;

  return data[0].output;
}
```

### 3. Batch Requests
For the Parser and Moderation agents, batch multiple items into a single request instead of sending them one by one. This reduces overhead tokens (system prompt is sent once instead of per-item).

### 4. Set Token Limits
Configure `maxOutputTokens` in the Gemini API call to prevent runaway costs:

```typescript
const result = await model.generateContent({
  contents: [{ role: "user", parts: [{ text: prompt }] }],
  generationConfig: {
    maxOutputTokens: 2048, // Cap output length
    temperature: 0.7,
  },
});
```

### 5. Monitor and Alert
Query the `agent_tasks` table daily to track spending:

```sql
-- Daily cost report
SELECT
  agent_name,
  COUNT(*) as total_tasks,
  SUM(token_usage) as total_tokens,
  SUM(cost_usd) as total_cost,
  AVG(cost_usd) as avg_cost_per_task
FROM agent_tasks
WHERE created_at >= NOW() - INTERVAL '24 hours'
GROUP BY agent_name
ORDER BY total_cost DESC;
```
