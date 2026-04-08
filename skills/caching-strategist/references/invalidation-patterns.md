# Cache Invalidation Patterns

## Event-Driven Invalidation

The best approach: invalidate caches when the source data changes.

### Pattern: Invalidate on Write

```typescript
// lib/services/assessment-service.ts
export async function createAssessment(input: CreateAssessmentInput): Promise<Assessment> {
  const supabase = await createClient();
  const { data, error } = await supabase
    .from("assessments")
    .insert(input)
    .select()
    .single();

  if (error) throw new Error(error.message);

  // Invalidate related caches
  await invalidateAssessmentCaches(input.userId);

  return data;
}

async function invalidateAssessmentCaches(userId: string) {
  await Promise.all([
    // Redis caches
    redis.del(`app:assessments:user:${userId}`),
    redis.del(`app:assessments:count`),
    redis.del(`app:stats:daily:${new Date().toISOString().split("T")[0]}`),

    // Next.js caches
    revalidateTag("assessments"),
    revalidateTag(`user-${userId}`),
  ]);
}
```

### Pattern: Supabase Database Webhook

Trigger cache invalidation from database changes:

```typescript
// app/api/webhooks/supabase/route.ts
import { NextRequest, NextResponse } from "next/server";
import { revalidateTag } from "next/cache";

export async function POST(request: NextRequest) {
  const secret = request.headers.get("x-webhook-secret");
  if (secret !== process.env.SUPABASE_WEBHOOK_SECRET) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const payload = await request.json();
  const { table, type, record, old_record } = payload;

  // Invalidate based on table and operation
  switch (table) {
    case "assessments":
      await redis.del(`app:assessment:${record.id}`);
      await redis.del(`app:assessments:user:${record.user_id}`);
      revalidateTag("assessments");
      break;

    case "profiles":
      await redis.del(`app:user:${record.id}`);
      revalidateTag(`user-${record.id}`);
      break;

    case "occupations":
      await redis.del("app:occupations:all");
      revalidateTag("occupations");
      break;
  }

  return NextResponse.json({ processed: true });
}
```

## Tag-Based Invalidation (Next.js)

```typescript
// Tag your fetches
const assessments = await fetch(url, {
  next: { tags: ["assessments", `user-${userId}`, `subclass-${visaSubclass}`] },
});

// Invalidate precisely
revalidateTag("assessments");              // All assessments
revalidateTag(`user-${userId}`);           // One user's data
revalidateTag(`subclass-${visaSubclass}`); // One visa type
```

## Manual Invalidation Admin UI

```typescript
// app/api/admin/cache/route.ts
export async function DELETE(request: NextRequest) {
  const { key, pattern, tag } = await request.json();

  if (key) {
    await redis.del(key);
    return NextResponse.json({ deleted: key });
  }

  if (pattern) {
    // Delete by pattern
    let deleted = 0;
    let cursor = 0;
    do {
      const [next, keys] = await redis.scan(cursor, { match: pattern, count: 100 });
      cursor = Number(next);
      if (keys.length > 0) {
        await redis.del(...keys);
        deleted += keys.length;
      }
    } while (cursor !== 0);
    return NextResponse.json({ deleted, pattern });
  }

  if (tag) {
    revalidateTag(tag);
    return NextResponse.json({ revalidated: tag });
  }

  return NextResponse.json({ error: "Provide key, pattern, or tag" }, { status: 400 });
}
```

## Cache Invalidation Decision Tree

```
Data changed in the database?
├── YES → Is it user-specific data?
│         ├── YES → Invalidate: Redis key + Next.js user tag
│         └── NO  → Is it public/shared data?
│                   ├── YES → Invalidate: Redis key + Next.js resource tag + CDN purge
│                   └── NO  → Invalidate: Redis key only
└── NO  → Is the TTL appropriate?
          ├── YES → Let it expire naturally
          └── NO  → Adjust TTL in cache configuration
```

## Monitoring Cache Health

```typescript
// lib/cache/monitor.ts
export async function getCacheStats(): Promise<{
  hitRate: number;
  totalKeys: number;
  memoryUsage: string;
}> {
  const info = await redis.info();
  // Parse Redis INFO output for stats

  return {
    hitRate: 0, // Calculate from your hit/miss counters
    totalKeys: await redis.dbsize(),
    memoryUsage: "N/A", // From Redis INFO
  };
}

// Track hit/miss in your cached() function
export async function cachedWithMetrics<T>(key: string, fetcher: () => Promise<T>, ttl: number): Promise<T> {
  const hit = await redis.get<T>(key);
  if (hit !== null) {
    await redis.incr("cache:hits");
    return hit;
  }

  await redis.incr("cache:misses");
  const data = await fetcher();
  await redis.set(key, data, { ex: ttl });
  return data;
}
```
