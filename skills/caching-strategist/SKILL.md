---
name: caching-strategist
description: "Use this skill whenever the user mentions caching, cache, Redis, Upstash, cache invalidation, stale data, stale-while-revalidate, ISR strategy, CDN caching, edge caching, cache warming, cache stampede, thundering herd, 'data is stale', 'cache is wrong', 'when to cache', cache-control headers, TTL, cache busting, multi-layer cache, 'my data is outdated', or ANY caching strategy/design task — even if they don't explicitly say 'cache'. This skill designs coherent caching strategies across all layers."
---

# Caching Strategist

Design a coherent caching strategy across all layers. This skill focuses on multi-layer architecture and cache invalidation. Cross-reference: vercel-power-user §3 for KV basics, §6 for ISR basics.

## 1. Caching Mental Model

Your app has 5 cache layers. A request checks each layer from top to bottom:

```
Browser Cache  →  CDN Edge  →  Next.js Data Cache  →  Redis (Upstash)  →  Database
  (client)       (Vercel)      (server)               (app layer)         (Supabase)
```

| Layer | TTL | Invalidation | Best For |
|-------|-----|-------------|----------|
| Browser | seconds-hours | Cache-Control headers | Static assets, fonts |
| CDN Edge | seconds-days | `s-maxage`, deploy | Public pages, images |
| Next.js Data Cache | seconds-hours | `revalidateTag`, `revalidatePath` | fetch() results |
| Redis (Upstash) | seconds-days | Manual delete, TTL | API responses, computed data |
| Database | N/A | Source of truth | Everything |

## 2. Next.js Data Cache

```typescript
// Server component — cached for 60 seconds
async function getAssessments() {
  const res = await fetch(`${process.env.API_URL}/assessments`, {
    next: { revalidate: 60 }, // Cache for 60 seconds
  });
  return res.json();
}

// Server component — cached with a tag
async function getOccupationList() {
  const res = await fetch(`${process.env.API_URL}/occupations`, {
    next: { tags: ["occupations"] }, // Tagged for invalidation
  });
  return res.json();
}

// Invalidate by tag when data changes
import { revalidateTag } from "next/cache";
revalidateTag("occupations"); // All fetches tagged "occupations" refetch
```

## 3. ISR Strategies

### Time-Based (Simple)

```typescript
// app/blog/[slug]/page.tsx
export const revalidate = 3600; // Revalidate every hour
```

### On-Demand (Precise)

```typescript
// app/api/revalidate/route.ts
import { revalidatePath, revalidateTag } from "next/cache";
import { NextRequest, NextResponse } from "next/server";

export async function POST(request: NextRequest) {
  const { secret, path, tag } = await request.json();

  if (secret !== process.env.REVALIDATION_SECRET) {
    return NextResponse.json({ error: "Invalid secret" }, { status: 401 });
  }

  if (tag) revalidateTag(tag);
  if (path) revalidatePath(path);

  return NextResponse.json({ revalidated: true });
}
```

## 4. Upstash Redis Caching

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

// Read-through cache pattern
export async function cached<T>(
  key: string,
  fetcher: () => Promise<T>,
  ttlSeconds = 3600
): Promise<T> {
  const hit = await redis.get<T>(key);
  if (hit !== null) return hit;

  const data = await fetcher();
  await redis.set(key, data, { ex: ttlSeconds });
  return data;
}
```

### Key Naming Convention

```
app:{resource}:{id}           → app:user:123
app:{resource}:list:{params}  → app:assessments:list:page=1&limit=20
app:{resource}:count           → app:assessments:count
app:ai:{hash}                  → app:ai:a1b2c3d4
```

## 5. CDN Cache Headers

```typescript
// app/api/public/occupations/route.ts
import { NextResponse } from "next/server";

export async function GET() {
  const data = await getOccupations();

  return NextResponse.json(data, {
    headers: {
      // CDN caches for 1 hour, browser caches for 5 min
      "Cache-Control": "public, s-maxage=3600, max-age=300, stale-while-revalidate=86400",
    },
  });
}
```

**Header anatomy:**
```
Cache-Control: public, s-maxage=3600, max-age=300, stale-while-revalidate=86400

public              → CDN can cache this
s-maxage=3600       → CDN keeps it for 1 hour
max-age=300         → Browser keeps it for 5 minutes
stale-while-revalidate=86400  → Serve stale for 24h while refreshing
```

**Per-route strategy:**

| Route | Cache-Control | Why |
|-------|--------------|-----|
| `/api/public/occupations` | `public, s-maxage=86400` | Changes rarely |
| `/api/assessments` | `private, no-cache` | User-specific data |
| `/api/visa-news` | `public, s-maxage=3600, stale-while-revalidate=86400` | Updates hourly |
| `/api/user/profile` | `private, max-age=60` | User data, short cache |

## 6. Cache Invalidation Strategies

### Event-Driven (Best)

```typescript
// When data changes, invalidate related caches
export async function updateAssessment(id: string, data: UpdateAssessmentInput) {
  const supabase = await createClient();
  const { error } = await supabase.from("assessments").update(data).eq("id", id);
  if (error) throw new Error(error.message);

  // Invalidate all related caches
  await Promise.all([
    redis.del(`app:assessment:${id}`),
    redis.del(`app:assessments:list:user=${data.userId}`),
    revalidateTag("assessments"),
  ]);
}
```

### Tag-Based (Next.js)

```typescript
// Tag fetches when creating them
const data = await fetch(url, { next: { tags: ["assessments", `user-${userId}`] } });

// Invalidate by tag
revalidateTag("assessments");        // All assessment data
revalidateTag(`user-${userId}`);     // All data for one user
```

### TTL-Based (Simplest)

```typescript
// Data expires automatically after TTL
await redis.set("app:stats:daily", stats, { ex: 3600 }); // Expires in 1 hour
```

## 7. Cache Stampede Prevention

When the cache expires, many requests hit the database at once (thundering herd):

```typescript
// lib/cache/stampede-proof.ts
import { redis } from "./redis";

export async function cachedWithLock<T>(
  key: string,
  fetcher: () => Promise<T>,
  ttl: number = 3600
): Promise<T> {
  // Check cache
  const cached = await redis.get<T>(key);
  if (cached !== null) return cached;

  // Try to acquire lock
  const lockKey = `lock:${key}`;
  const acquired = await redis.set(lockKey, "1", { nx: true, ex: 30 }); // 30s lock

  if (!acquired) {
    // Another request is fetching — wait and retry
    await new Promise((r) => setTimeout(r, 100));
    return cachedWithLock(key, fetcher, ttl);
  }

  try {
    const data = await fetcher();
    await redis.set(key, data, { ex: ttl });
    return data;
  } finally {
    await redis.del(lockKey);
  }
}
```

## 8. Caching Anti-Patterns

**Don't cache:**
- User authentication state (security risk)
- Payment/billing data (must be real-time)
- Data during active editing (causes conflicts)
- Rapidly changing data with short TTLs (cache churn)

**Don't:**
- Cache without a TTL (stale forever)
- Cache without an invalidation strategy
- Cache large objects (> 1MB in Redis)
- Use the same TTL for everything

## Rules

1. **Always set a TTL** — Even if it's 24 hours. Data without TTL goes stale silently.
2. **Invalidate on write** — When data changes, delete related cache keys immediately.
3. **Use `stale-while-revalidate`** — Users get fast responses while fresh data loads in the background.
4. **Cache at the right layer** — Public data at CDN, user data in Redis, computed data in application.
5. **Never cache user-specific data in CDN** — Use `private` in Cache-Control for authenticated responses.
6. **Name keys consistently** — `app:{resource}:{id}` makes bulk invalidation easy.
7. **Log cache hits/misses** — Track hit rate to know if your caching strategy is working.
8. **Use tags for Next.js cache** — Tag-based invalidation is more precise than path-based.
9. **Prevent stampedes** — Use locking for expensive cache rebuilds.
10. **Profile before caching** — Only cache what's actually slow. Don't add complexity for 5ms queries.

See `references/` for detailed guides:
- `cache-headers-guide.md` — Complete Cache-Control reference
- `redis-patterns.md` — Upstash Redis caching patterns
- `invalidation-patterns.md` — Event-driven invalidation strategies
