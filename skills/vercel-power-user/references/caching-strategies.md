# Caching Strategies — KV, ISR, SWR, CDN

## Caching Hierarchy

```
Request → CDN Cache (edge, free)
  └─ Miss → ISR Cache (server, revalidate)
       └─ Miss → KV Cache (Redis, fast)
            └─ Miss → Database (Supabase)
```

## 1. CDN Cache (Automatic)

Static assets are cached at the edge automatically. Control dynamic routes:

```typescript
// file: app/api/public-data/route.ts
export async function GET() {
  const data = await fetchData();
  return Response.json(data, {
    headers: { 'Cache-Control': 'public, s-maxage=60, stale-while-revalidate=300' },
  });
}
```

## 2. ISR (Incremental Static Regeneration)

```typescript
// file: app/blog/[slug]/page.tsx
// Time-based: regenerate every 60 seconds
export const revalidate = 60;

// On-demand: call revalidatePath('/blog/my-post') or revalidateTag('blog')
import { revalidatePath, revalidateTag } from 'next/cache';
```

### Tag-Based Revalidation

```typescript
// Fetch with tags
const posts = await fetch('https://api.example.com/posts', {
  next: { tags: ['posts'] },
});

// Revalidate all fetches tagged 'posts'
revalidateTag('posts');
```

## 3. Vercel KV (Redis)

```typescript
// file: lib/cache.ts
import { kv } from '@vercel/kv';

// Simple get/set
await kv.set('key', value, { ex: 300 }); // 5-min TTL
const cached = await kv.get<T>('key');

// Hash for structured data
await kv.hset('user:123', { name: 'Tanul', plan: 'pro' });
const user = await kv.hgetall('user:123');

// Sorted set for leaderboards
await kv.zadd('leaderboard', { score: 100, member: 'user:123' });
const top10 = await kv.zrange('leaderboard', 0, 9, { rev: true });

// List for queues
await kv.lpush('job-queue', JSON.stringify({ task: 'process' }));
const job = await kv.rpop('job-queue');
```

## 4. SWR (Client-Side)

```typescript
// file: hooks/use-data.ts
'use client';
import useSWR from 'swr';

const fetcher = (url: string) => fetch(url).then(r => r.json());

export function useStats() {
  const { data, error, isLoading, mutate } = useSWR('/api/stats', fetcher, {
    refreshInterval: 30000, // Refresh every 30s
    revalidateOnFocus: true,
    dedupingInterval: 5000,
  });

  return { stats: data, error, isLoading, refresh: mutate };
}
```

## Decision Guide

| Data Type | Strategy | TTL |
|-----------|----------|-----|
| Static pages | ISR | 3600s (1h) |
| User dashboard | SWR client-side | 30s |
| API rate limit counters | KV | 60s |
| Session data | KV | 86400s (24h) |
| Public API responses | CDN + ISR | 60s |
| Search results | KV | 300s (5m) |
| Real-time data | No cache (Supabase Realtime) | 0 |
