# Cost Optimization — Staying on Budget

## Vercel Plan Limits

| Resource | Hobby (Free) | Pro ($20/mo) |
|----------|-------------|--------------|
| Bandwidth | 100 GB | 1 TB |
| Serverless execution | 100 GB-hrs | 1000 GB-hrs |
| Edge requests | 500K | 1M |
| Builds | 6000 min/mo | 24000 min/mo |
| Cron invocations | 2/day total | Unlimited |
| KV reads | 3K/day | 150K/day |
| KV writes | 1K/day | 50K/day |
| Blob storage | 500 MB | 10 GB |
| Concurrent builds | 1 | 3 |

## Top Cost Reduction Strategies

### 1. Use ISR Instead of SSR

```typescript
// BAD: Re-renders on EVERY request (costs serverless time)
export const dynamic = 'force-dynamic';

// GOOD: Renders once, caches for 60s (serves from CDN for free)
export const revalidate = 60;
```

### 2. Use Edge for Simple Routes

```typescript
// Edge is cheaper per invocation and has no cold start
export const runtime = 'edge';
```

### 3. Set maxDuration to Prevent Runaway

```typescript
// Don't let a stuck function run for 300 seconds
export const maxDuration = 10; // Fail fast
```

### 4. Cache API Responses in KV

```typescript
import { kv } from '@vercel/kv';

export async function GET() {
  const cached = await kv.get('expensive-query');
  if (cached) return Response.json(cached);

  const data = await expensiveQuery();
  await kv.set('expensive-query', data, { ex: 300 });
  return Response.json(data);
}
```

### 5. Optimize Images

```typescript
// next.config.ts
export default {
  images: {
    formats: ['image/avif', 'image/webp'], // Smaller files
    deviceSizes: [640, 750, 828, 1080, 1200], // Don't generate too many sizes
  },
};
```

### 6. Dynamic Imports for Heavy Libraries

```typescript
// Only load chart library when needed
const Chart = dynamic(() => import('@/components/chart'), { ssr: false });
```

### 7. Limit Cron Frequency

```json
// Instead of every minute, run every 5 minutes
{ "path": "/api/cron/check", "schedule": "*/5 * * * *" }
```

## Monitoring Usage

Check your usage at: `vercel.com/[team]/~/usage`

Set up spending alerts:
1. Go to Settings → Billing
2. Set monthly budget alert threshold
3. Get notified before overages hit

## Cost per Feature Estimate

| Feature | Monthly Cost (Pro) |
|---------|-------------------|
| Static marketing site | ~$0 (CDN cache) |
| Dashboard with ISR | ~$2-5 |
| API with 10K requests/day | ~$5-10 |
| 5 cron jobs (hourly) | ~$1-3 |
| KV cache (50K reads/day) | ~$5 |
| Blob storage (5GB) | ~$2 |
| Heavy AI processing (Gemini calls) | Gemini API costs, not Vercel |
