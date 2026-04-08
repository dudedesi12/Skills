# Cost Optimization Patterns

## Cache AI Responses

Same input should never call the API twice.

```typescript
// lib/ai/cached-generate.ts
import { Redis } from "@upstash/redis";
import crypto from "crypto";

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

function hashPrompt(prompt: string): string {
  return crypto.createHash("sha256").update(prompt).digest("hex").substring(0, 16);
}

export async function cachedGenerate(
  task: string,
  prompt: string,
  options: { ttl?: number; userId?: string } = {}
) {
  const { ttl = 3600, userId } = options;
  const cacheKey = `ai:${task}:${hashPrompt(prompt)}`;

  // Check cache first
  const cached = await redis.get<string>(cacheKey);
  if (cached) {
    // Track as cached (zero cost)
    await trackTokenUsage({
      userId, model: "cache", complexity: "cached",
      promptTokens: 0, completionTokens: 0, totalTokens: 0, feature: task,
    });
    return cached;
  }

  // Generate and cache
  const result = await generateWithTracking(task, prompt, userId);
  await redis.set(cacheKey, result, { ex: ttl });
  return result;
}
```

## Request Deduplication

Prevent duplicate requests within a short window:

```typescript
// lib/ai/dedup.ts
const inFlight = new Map<string, Promise<string>>();

export async function deduplicatedGenerate(task: string, prompt: string): Promise<string> {
  const key = `${task}:${hashPrompt(prompt)}`;

  if (inFlight.has(key)) {
    return inFlight.get(key)!; // Return the same promise
  }

  const promise = generateWithTracking(task, prompt).finally(() => {
    inFlight.delete(key);
  });

  inFlight.set(key, promise);
  return promise;
}
```

## Batch Processing

Combine multiple items into one API call:

```typescript
export async function batchClassify(items: string[]): Promise<string[]> {
  if (items.length === 0) return [];
  if (items.length === 1) {
    const result = await generateWithTracking("classify", `Classify: ${items[0]}`);
    return [result];
  }

  // Batch: one call for all items
  const prompt = `Classify each occupation into ANZSCO category. Return JSON array.
Items:
${items.map((item, i) => `${i + 1}. ${item}`).join("\n")}`;

  const result = await generateWithTracking("classify-batch", prompt);
  return JSON.parse(result);
}

// 10 items × 1 call = ~$0.001
// vs 10 items × 10 calls = ~$0.01 (10x more expensive)
```

## Tiered Service Limits

Different limits for free vs paid users:

```typescript
// lib/costs/user-limits.ts
const LIMITS: Record<string, { dailyTokens: number; model: string }> = {
  free: { dailyTokens: 10_000, model: "gemini-2.0-flash-lite" },
  pro: { dailyTokens: 100_000, model: "gemini-2.0-flash" },
  enterprise: { dailyTokens: 1_000_000, model: "gemini-2.5-pro" },
};

export async function checkUserLimit(userId: string, plan: string): Promise<boolean> {
  const limit = LIMITS[plan] ?? LIMITS.free;
  const supabase = await createClient();

  const today = new Date().toISOString().split("T")[0];
  const { data } = await supabase
    .from("token_usage")
    .select("total_tokens")
    .eq("user_id", userId)
    .gte("created_at", `${today}T00:00:00Z`);

  const used = (data ?? []).reduce((sum, r) => sum + r.total_tokens, 0);
  return used < limit.dailyTokens;
}
```

## Off-Peak Processing

Schedule expensive AI tasks during off-peak hours:

```typescript
// Vercel Cron: run expensive analysis at 3 AM AEST (off-peak for Gemini)
// vercel.json
{
  "crons": [{
    "path": "/api/cron/batch-analysis",
    "schedule": "0 17 * * *"  // 5 PM UTC = 3 AM AEST
  }]
}
```

## Reduce Supabase Costs

```typescript
// 1. Use count queries instead of fetching all rows
const { count } = await supabase
  .from("assessments")
  .select("*", { count: "exact", head: true }); // head: true = don't return data

// 2. Paginate everything
const PAGE_SIZE = 20;
const { data } = await supabase
  .from("profiles")
  .select("id, full_name")
  .range(page * PAGE_SIZE, (page + 1) * PAGE_SIZE - 1);

// 3. Unsubscribe from realtime when component unmounts
useEffect(() => {
  const channel = supabase.channel("changes").on("postgres_changes", { ... }, handler).subscribe();
  return () => { supabase.removeChannel(channel); }; // Clean up!
}, []);
```
