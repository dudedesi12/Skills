# Edge vs Serverless — Decision Matrix

## Quick Decision

```
Need full Node.js APIs? → Serverless
Need <50ms cold start? → Edge
Heavy computation? → Serverless
Auth/redirect check? → Edge
Database queries? → Serverless (use connection pooling)
File processing? → Serverless
Simple JSON response? → Edge
```

## Comparison Table

| Feature | Edge | Serverless |
|---------|------|------------|
| Cold start | ~0ms | ~250ms |
| Max execution | 30s | 60s (Pro: 300s) |
| Max code size | 128KB (4MB compressed) | 250MB |
| Node.js APIs | Limited (no fs, child_process) | Full |
| Runtime | V8 isolates | Node.js |
| Regions | All CDN edges | Single region (configurable) |
| Memory | 128MB | 1024MB (Pro: 3008MB) |
| Streaming | Yes | Yes |
| Cost | Lower per request | Higher per request |

## Setting Runtime

```typescript
// Edge — add to any route.ts or page.tsx
export const runtime = 'edge';

// Serverless (default) — explicit for clarity
export const runtime = 'nodejs';
export const maxDuration = 60;

// Prefer specific region for DB proximity
export const preferredRegion = 'syd1'; // Sydney for Australian users
```

## When to Use Edge

1. **Middleware** — Auth checks, redirects, geo-routing
2. **Simple API routes** — Health checks, feature flags
3. **Streaming** — Real-time responses, SSE
4. **A/B testing** — Cookie-based variant assignment

## When to Use Serverless

1. **Database operations** — Supabase queries, complex joins
2. **File processing** — PDF generation, image manipulation
3. **External API calls** — Stripe, Gemini, long-running fetches
4. **Cron jobs** — Scheduled tasks that may run >30s
