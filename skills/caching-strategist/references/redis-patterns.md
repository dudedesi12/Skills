# Upstash Redis Caching Patterns

## Setup

```bash
npm install @upstash/redis
```

```typescript
// lib/cache/redis.ts
import { Redis } from "@upstash/redis";

export const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});
```

## Key Naming Convention

```
{app}:{resource}:{identifier}

Examples:
app:user:123                    → Single user
app:users:list:page=1           → Paginated list
app:assessment:456              → Single assessment
app:assessments:user:123        → User's assessments
app:occupations:all             → All occupations
app:ai:visa-check:a1b2c3       → AI response cache
app:stats:daily:2026-04-08     → Daily statistics
```

## Common Patterns

### Read-Through Cache

```typescript
export async function cached<T>(
  key: string,
  fetcher: () => Promise<T>,
  ttl = 3600
): Promise<T> {
  const hit = await redis.get<T>(key);
  if (hit !== null) return hit;

  const data = await fetcher();
  await redis.set(key, data, { ex: ttl });
  return data;
}

// Usage:
const user = await cached(`app:user:${id}`, () =>
  supabase.from("profiles").select("*").eq("id", id).single().then((r) => r.data)
);
```

### Write-Through Cache

```typescript
export async function updateAndCache<T>(
  key: string,
  updater: () => Promise<T>,
  ttl = 3600
): Promise<T> {
  const data = await updater();
  await redis.set(key, data, { ex: ttl });
  return data;
}
```

### Cache Aside (Manual)

```typescript
// Read: check cache, fall back to DB
async function getAssessment(id: string) {
  const cacheKey = `app:assessment:${id}`;
  const cached = await redis.get(cacheKey);
  if (cached) return cached;

  const { data } = await supabase.from("assessments").select("*").eq("id", id).single();
  if (data) await redis.set(cacheKey, data, { ex: 3600 });
  return data;
}

// Write: update DB, delete cache
async function updateAssessment(id: string, updates: Partial<Assessment>) {
  await supabase.from("assessments").update(updates).eq("id", id);
  await redis.del(`app:assessment:${id}`);
  // Related caches too:
  await redis.del(`app:assessments:user:${updates.userId}`);
}
```

## TTL Strategies

| Data Type | TTL | Reason |
|-----------|-----|--------|
| Occupation list | 24 hours | Changes rarely |
| User profile | 5 minutes | Changes occasionally |
| Assessment result | 1 hour | Stable after creation |
| AI response | 1 hour | Same input = same output |
| Daily statistics | Until midnight | Reset daily |
| Search results | 5 minutes | Frequently changing |
| Session data | 24 hours | Matches auth session |

## Bulk Operations

```typescript
// Get multiple keys at once
const keys = userIds.map((id) => `app:user:${id}`);
const users = await redis.mget<User[]>(...keys);

// Set multiple keys
const pipeline = redis.pipeline();
for (const user of users) {
  pipeline.set(`app:user:${user.id}`, user, { ex: 3600 });
}
await pipeline.exec();

// Delete by pattern (use scan, not keys)
async function deleteByPattern(pattern: string) {
  let cursor = 0;
  do {
    const [nextCursor, keys] = await redis.scan(cursor, { match: pattern, count: 100 });
    cursor = Number(nextCursor);
    if (keys.length > 0) {
      await redis.del(...keys);
    }
  } while (cursor !== 0);
}

// Delete all assessment caches
await deleteByPattern("app:assessment:*");
```

## Cache Warming

Pre-populate cache for frequently accessed data:

```typescript
// api/cron/warm-cache/route.ts
export async function GET() {
  // Warm the occupation list (accessed on every assessment page)
  const { data: occupations } = await supabase.from("occupations").select("*");
  await redis.set("app:occupations:all", occupations, { ex: 86400 });

  // Warm today's stats
  const stats = await calculateDailyStats();
  await redis.set(`app:stats:daily:${today}`, stats, { ex: 86400 });

  return NextResponse.json({ warmed: true });
}
```
