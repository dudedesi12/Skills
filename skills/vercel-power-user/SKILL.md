---
name: vercel-power-user
description: >-
  Use this skill whenever the user mentions Vercel beyond basic deployment — edge functions, cron
  jobs, KV, Blob, middleware, ISR revalidation, build optimization, monorepo, preview deployments,
  vercel.json configuration, spending, cold starts, or any infrastructure concern. Even if they say
  'my site is slow', 'I need caching', 'schedule a task', or 'how do I handle file uploads' — use
  this skill.
---

# Vercel Power User

Advanced Vercel platform patterns — edge functions, KV, Blob, cron, middleware, and cost optimization for production Next.js apps.

## 1. Edge vs Serverless Functions

```
Edge Functions (runs at CDN edge, ~0ms cold start):
  ✅ Fast responses, no cold start
  ✅ Geo-routing, A/B testing, auth checks
  ❌ No Node.js APIs (fs, child_process)
  ❌ Limited to 128KB code size, 30s execution

Serverless Functions (runs in one region, ~250ms cold start):
  ✅ Full Node.js, up to 250MB, 60s execution (Pro)
  ✅ Heavy processing, database queries, file generation
  ❌ Cold starts, single region by default
```

### Set Runtime Per Route

```typescript
// file: app/api/fast-check/route.ts
export const runtime = 'edge'; // Runs at edge

export async function GET() {
  return Response.json({ status: 'ok', region: process.env.VERCEL_REGION });
}
```

```typescript
// file: app/api/heavy-task/route.ts
export const runtime = 'nodejs'; // Default — serverless
export const maxDuration = 60; // Pro plan: up to 300s

export async function POST() {
  // Heavy processing here
  return Response.json({ done: true });
}
```

## 2. Vercel Cron Jobs

```json
// file: vercel.json
{
  "crons": [
    { "path": "/api/cron/daily-cleanup", "schedule": "0 2 * * *" },
    { "path": "/api/cron/hourly-sync", "schedule": "0 * * * *" },
    { "path": "/api/cron/every-5-min", "schedule": "*/5 * * * *" }
  ]
}
```

```typescript
// file: app/api/cron/daily-cleanup/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function GET(request: NextRequest) {
  // Verify cron secret — prevents public access
  if (request.headers.get('authorization') !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  try {
    // Your cleanup logic here
    return NextResponse.json({ success: true, cleaned: 42 });
  } catch (err) {
    const message = err instanceof Error ? err.message : 'Cron failed';
    return NextResponse.json({ error: message }, { status: 500 });
  }
}
```

## 3. Vercel KV (Redis)

Install: `npm install @vercel/kv`

### Rate Limiting

```typescript
// file: lib/rate-limit.ts
import { kv } from '@vercel/kv';

export async function rateLimit(identifier: string, limit = 10, windowSeconds = 60) {
  const key = `rate:${identifier}`;
  const current = await kv.incr(key);

  if (current === 1) {
    await kv.expire(key, windowSeconds);
  }

  return {
    allowed: current <= limit,
    remaining: Math.max(0, limit - current),
    reset: windowSeconds,
  };
}
```

### Caching

```typescript
// file: lib/cache.ts
import { kv } from '@vercel/kv';

export async function cached<T>(key: string, fetcher: () => Promise<T>, ttlSeconds = 300): Promise<T> {
  const cached = await kv.get<T>(key);
  if (cached !== null) return cached;

  const fresh = await fetcher();
  await kv.set(key, fresh, { ex: ttlSeconds });
  return fresh;
}

// Usage in a route handler:
// const data = await cached('homepage-stats', () => fetchStats(), 600);
```

## 4. Vercel Blob (File Storage)

Install: `npm install @vercel/blob`

```typescript
// file: app/api/upload/route.ts
import { put, del } from '@vercel/blob';
import { NextRequest, NextResponse } from 'next/server';

export async function POST(request: NextRequest) {
  try {
    const form = await request.formData();
    const file = form.get('file') as File | null;

    if (!file) return NextResponse.json({ error: 'No file' }, { status: 400 });

    // Validate file
    const maxSize = 10 * 1024 * 1024; // 10MB
    if (file.size > maxSize) return NextResponse.json({ error: 'File too large' }, { status: 400 });

    const allowedTypes = ['image/jpeg', 'image/png', 'image/webp', 'application/pdf'];
    if (!allowedTypes.includes(file.type)) {
      return NextResponse.json({ error: 'Invalid file type' }, { status: 400 });
    }

    const blob = await put(`uploads/${Date.now()}-${file.name}`, file, {
      access: 'public',
      addRandomSuffix: true,
    });

    return NextResponse.json({ url: blob.url });
  } catch (err) {
    const message = err instanceof Error ? err.message : 'Upload failed';
    return NextResponse.json({ error: message }, { status: 500 });
  }
}
```

## 5. Advanced Middleware

```typescript
// file: middleware.ts
import { NextResponse, type NextRequest } from 'next/server';
import { createServerClient } from '@supabase/ssr';

export async function middleware(request: NextRequest) {
  const response = NextResponse.next();
  const { pathname } = request.nextUrl;

  // 1. Bot detection — block scrapers from API routes
  const ua = request.headers.get('user-agent') ?? '';
  if (pathname.startsWith('/api/') && /bot|crawler|spider/i.test(ua)) {
    return NextResponse.json({ error: 'Blocked' }, { status: 403 });
  }

  // 2. Geo-routing — redirect to country-specific page
  const country = request.geo?.country ?? 'US';
  if (pathname === '/' && country === 'AU') {
    return NextResponse.rewrite(new URL('/au', request.url));
  }

  // 3. Feature flags via cookie
  if (!request.cookies.has('ab-variant')) {
    const variant = Math.random() > 0.5 ? 'A' : 'B';
    response.cookies.set('ab-variant', variant, { maxAge: 86400 * 30 });
  }

  // 4. Auth check for protected routes
  if (pathname.startsWith('/dashboard') || pathname.startsWith('/admin')) {
    const supabase = createServerClient(
      process.env.NEXT_PUBLIC_SUPABASE_URL!,
      process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
      { cookies: { getAll: () => request.cookies.getAll(), setAll: (cookies) => cookies.forEach(({ name, value, options }) => response.cookies.set(name, value, options)) } }
    );
    const { data: { user } } = await supabase.auth.getUser();
    if (!user) return NextResponse.redirect(new URL('/login', request.url));
  }

  return response;
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)'],
};
```

## 6. ISR — On-Demand Revalidation

```typescript
// file: app/api/revalidate/route.ts
import { revalidatePath, revalidateTag } from 'next/cache';
import { NextRequest, NextResponse } from 'next/server';

export async function POST(request: NextRequest) {
  if (request.headers.get('authorization') !== `Bearer ${process.env.REVALIDATION_SECRET}`) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { path, tag } = await request.json();

  if (tag) {
    revalidateTag(tag); // Revalidate all fetches with this tag
    return NextResponse.json({ revalidated: true, tag });
  }

  if (path) {
    revalidatePath(path); // Revalidate specific path
    return NextResponse.json({ revalidated: true, path });
  }

  return NextResponse.json({ error: 'Provide path or tag' }, { status: 400 });
}
```

Use tags in data fetching:

```typescript
// file: app/blog/[slug]/page.tsx
async function getPost(slug: string) {
  const supabase = await createClient();
  const { data } = await supabase
    .from('posts')
    .select('*')
    .eq('slug', slug)
    .single();
  return data;
}

// Tag this page for on-demand revalidation
export default async function BlogPost({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const post = await getPost(slug);
  // ... render
}

// Revalidate with: POST /api/revalidate { "path": "/blog/my-post" }
```

## 7. Build Optimization

```typescript
// file: next.config.ts
import type { NextConfig } from 'next';
import bundleAnalyzer from '@next/bundle-analyzer';

const withBundleAnalyzer = bundleAnalyzer({ enabled: process.env.ANALYZE === 'true' });

const nextConfig: NextConfig = {
  // Tree shake unused code
  experimental: { optimizePackageImports: ['lucide-react', '@heroicons/react'] },

  // Compress images automatically
  images: {
    formats: ['image/avif', 'image/webp'],
    remotePatterns: [{ protocol: 'https', hostname: '**.supabase.co' }],
  },

  // Security headers
  headers: async () => [{
    source: '/(.*)',
    headers: [
      { key: 'X-Frame-Options', value: 'DENY' },
      { key: 'X-Content-Type-Options', value: 'nosniff' },
      { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
    ],
  }],
};

export default withBundleAnalyzer(nextConfig);
```

### Dynamic Imports for Heavy Components

```typescript
// file: app/dashboard/page.tsx
import dynamic from 'next/dynamic';

const HeavyChart = dynamic(() => import('@/components/charts/revenue-chart'), {
  loading: () => <div className="h-64 bg-gray-100 animate-pulse rounded-lg" />,
  ssr: false, // Client-only — no SSR for charts
});

export default function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
      <HeavyChart />
    </div>
  );
}
```

## 8. vercel.json Quick Reference

```json
// file: vercel.json
{
  "framework": "nextjs",
  "regions": ["syd1"],
  "crons": [
    { "path": "/api/cron/daily", "schedule": "0 2 * * *" }
  ],
  "redirects": [
    { "source": "/old-page", "destination": "/new-page", "permanent": true }
  ],
  "rewrites": [
    { "source": "/blog/:path*", "destination": "/posts/:path*" }
  ],
  "headers": [
    {
      "source": "/api/(.*)",
      "headers": [
        { "key": "Cache-Control", "value": "no-store" }
      ]
    }
  ]
}
```

## 9. Cost Optimization

| Resource | Hobby (Free) | Pro ($20/mo) |
|----------|-------------|--------------|
| Serverless execution | 100 GB-hrs | 1000 GB-hrs |
| Edge requests | 500K | 1M |
| Bandwidth | 100 GB | 1 TB |
| Cron jobs | 2 per day | Unlimited |
| KV | 3K reads/day | 150K reads/day |
| Blob | 500 MB | 10 GB |

**Tips to stay on budget:**
- Use ISR (revalidate every 60s) instead of SSR for semi-dynamic pages
- Cache API responses in KV instead of re-fetching
- Use `export const dynamic = 'force-static'` where possible
- Set `maxDuration` on functions to prevent runaway execution
- Use edge runtime for simple checks (auth, redirects)

> For edge vs serverless decision matrix, see `references/edge-vs-serverless.md`.
> For middleware recipes, see `references/middleware-recipes.md`.
> For full vercel.json reference, see `references/vercel-json-reference.md`.
