---
name: deployment-ops
description: "Use this skill whenever the user mentions deploy, deployment, Vercel, hosting, production, preview, build errors, build fails, DNS, domain, custom domain, GoDaddy, Cloudflare, environment variables, env vars, .env, serverless, edge runtime, serverless functions, timeout, cold start, rollback, CI/CD, monitoring, logs, build cache, 'my site is down', 'deploy failed', '500 error in production', or ANY shipping/hosting task — even if they don't explicitly say 'deploy'. This skill gets your app live and keeps it running."
---

# Deployment & Operations

This skill covers everything needed to deploy, monitor, and maintain a Next.js App Router application on Vercel with Supabase.

## Vercel Deployment Configuration

Every project can have a `vercel.json` at the root to control build and routing behavior.

```json
// vercel.json
{
  "buildCommand": "next build",
  "installCommand": "npm install",
  "framework": "nextjs",
  "regions": ["iad1"],
  "headers": [
    {
      "source": "/api/(.*)",
      "headers": [
        { "key": "Cache-Control", "value": "no-store, max-age=0" }
      ]
    },
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Frame-Options", "value": "DENY" },
        { "key": "X-Content-Type-Options", "value": "nosniff" },
        { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" }
      ]
    }
  ],
  "redirects": [
    { "source": "/old-page", "destination": "/new-page", "permanent": true }
  ],
  "rewrites": [
    { "source": "/blog/:slug", "destination": "/posts/:slug" }
  ]
}
```

## Environment Variables

Vercel has three environments: **Development**, **Preview**, and **Production**. Each can have different values for the same variable.

### Setting Environment Variables in Vercel Dashboard

1. Go to your project in Vercel
2. Click **Settings** then **Environment Variables**
3. Add each variable and check which environments it applies to

### Required Variables for Next.js + Supabase

```bash
# .env.local (for local development only — NEVER commit this file)
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...your-anon-key
SUPABASE_SERVICE_ROLE_KEY=eyJ...your-service-role-key
GEMINI_API_KEY=your-gemini-api-key
NEXT_PUBLIC_SITE_URL=http://localhost:3000
```

**Important rules:**
- Variables starting with `NEXT_PUBLIC_` are visible to the browser — never put secrets there
- `SUPABASE_SERVICE_ROLE_KEY` and `GEMINI_API_KEY` must NEVER start with `NEXT_PUBLIC_`
- In Vercel, set `NEXT_PUBLIC_SITE_URL` to `https://yourdomain.com` for production and `https://*.vercel.app` for preview

### Accessing Environment Variables Safely

```typescript
// lib/env.ts
function getRequiredEnv(name: string): string {
  const value = process.env[name];
  if (!value) {
    throw new Error(`Missing required environment variable: ${name}`);
  }
  return value;
}

export const env = {
  supabaseUrl: getRequiredEnv("NEXT_PUBLIC_SUPABASE_URL"),
  supabaseAnonKey: getRequiredEnv("NEXT_PUBLIC_SUPABASE_ANON_KEY"),
  supabaseServiceKey: getRequiredEnv("SUPABASE_SERVICE_ROLE_KEY"),
  geminiApiKey: getRequiredEnv("GEMINI_API_KEY"),
  siteUrl: getRequiredEnv("NEXT_PUBLIC_SITE_URL"),
};
```

## Domain Configuration

### Connecting a Custom Domain in Vercel

1. Go to your Vercel project **Settings** then **Domains**
2. Type your domain (e.g., `myapp.com`) and click **Add**
3. Vercel shows you DNS records to add at your registrar

### DNS Records You Need

| Type | Name | Value |
|------|------|-------|
| A | @ | 76.76.21.21 |
| CNAME | www | cname.vercel-dns.com |

See `references/dns-setup.md` for registrar-specific steps (GoDaddy, Cloudflare, Namecheap).

## Build Error Troubleshooting

Below are the most common build errors. See `references/build-errors.md` for the full top-20 list with exact fixes.

### Error: Module not found

```
Module not found: Can't resolve 'some-package'
```

**Fix:** The package is missing from `package.json`. Run:

```bash
npm install some-package
```

### Error: Type errors during build

```
Type error: Property 'x' does not exist on type 'y'
```

**Fix:** Next.js runs TypeScript checks during build. Fix the type error or, as a last resort, add to `next.config.ts`:

```typescript
// next.config.ts
const nextConfig = {
  typescript: {
    ignoreBuildErrors: false, // Keep this false in production
  },
};
export default nextConfig;
```

### Error: Missing environment variable at build time

If your code reads `process.env.SOMETHING` during build (in a Server Component or `generateStaticParams`), that variable must exist in Vercel's environment settings.

### Error: Edge runtime + Node.js API

```
Dynamic Code Evaluation not allowed in Edge Runtime
```

**Fix:** Remove `export const runtime = 'edge'` or replace Node.js-only APIs with edge-compatible alternatives.

## Serverless Function Limits

| Plan | Max Duration | Memory | Payload Size |
|------|-------------|--------|-------------|
| Hobby | 10 seconds | 1024 MB | 4.5 MB |
| Pro | 60 seconds | 3008 MB | 4.5 MB |
| Enterprise | 900 seconds | 3008 MB | 4.5 MB |

### Optimizing Serverless Functions

```typescript
// app/api/heavy-task/route.ts
import { NextResponse } from "next/server";

// Set max duration for this route (Pro plan)
export const maxDuration = 60;

export async function POST(request: Request) {
  try {
    const body = await request.json();

    // Do your work here — keep it focused
    const result = await processData(body);

    return NextResponse.json({ success: true, data: result });
  } catch (error) {
    console.error("API error:", error);
    return NextResponse.json(
      { success: false, error: "Internal server error" },
      { status: 500 }
    );
  }
}
```

### Avoiding Timeouts

1. **Break large jobs into smaller pieces** — use a queue or batch approach
2. **Move heavy work to background jobs** — use Supabase Edge Functions or a queue
3. **Cache expensive computations** — use `unstable_cache` or Supabase

```typescript
// app/api/batch/route.ts
import { NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";

export const maxDuration = 60;

export async function POST(request: Request) {
  try {
    const { items } = await request.json();

    if (!Array.isArray(items) || items.length > 100) {
      return NextResponse.json(
        { error: "Send 100 items or fewer per batch" },
        { status: 400 }
      );
    }

    const supabase = await createClient();
    const { data, error } = await supabase.from("results").insert(items).select();

    if (error) {
      console.error("Supabase error:", error);
      return NextResponse.json({ error: "Database error" }, { status: 500 });
    }

    return NextResponse.json({ success: true, count: data.length });
  } catch (error) {
    console.error("Batch error:", error);
    return NextResponse.json({ error: "Internal server error" }, { status: 500 });
  }
}
```

## Edge vs Serverless Runtime

| Feature | Serverless (default) | Edge |
|---------|---------------------|------|
| Cold start | ~250ms | ~0ms |
| Max duration | 10-900s | 30s |
| Node.js APIs | Full support | Limited subset |
| npm packages | All | Must be edge-compatible |
| Regions | One region | All regions |
| Best for | Heavy computation, DB queries | Auth checks, redirects, A/B tests |

### Choosing Edge Runtime

```typescript
// app/api/fast-check/route.ts
export const runtime = "edge";

export async function GET(request: Request) {
  try {
    const token = request.headers.get("authorization");
    if (!token) {
      return new Response(JSON.stringify({ error: "Unauthorized" }), {
        status: 401,
        headers: { "Content-Type": "application/json" },
      });
    }
    return new Response(JSON.stringify({ valid: true }), {
      headers: { "Content-Type": "application/json" },
    });
  } catch (error) {
    return new Response(JSON.stringify({ error: "Server error" }), {
      status: 500,
      headers: { "Content-Type": "application/json" },
    });
  }
}
```

## Preview Deployments

Every push to a non-production branch creates a preview deployment automatically.

- Preview URL format: `project-git-branch-name-team.vercel.app`
- Each pull request gets its own preview
- Preview deployments use **Preview** environment variables

### Protecting Preview Deployments

In Vercel project settings, enable **Vercel Authentication** to require login for preview URLs. This prevents anyone from seeing unfinished work.

## Rollback Procedures

### Instant Rollback via Dashboard

1. Go to your Vercel project **Deployments** tab
2. Find the last working deployment
3. Click the three-dot menu and select **Promote to Production**

### Rollback via CLI

```bash
# List recent deployments
npx vercel ls

# Promote a specific deployment to production
npx vercel promote <deployment-url>
```

## Build Caching and Optimization

### Next.js Build Cache

Vercel automatically caches `.next/cache` between builds. To optimize:

```typescript
// next.config.ts
const nextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: "https",
        hostname: "your-project.supabase.co",
        pathname: "/storage/v1/object/public/**",
      },
    ],
  },
  experimental: {
    optimizePackageImports: ["lucide-react", "@supabase/supabase-js"],
  },
};
export default nextConfig;
```

### Speeding Up Builds

1. **Use `optimizePackageImports`** for large icon/component libraries
2. **Avoid importing entire libraries** — use specific imports: `import { Button } from "@/components/ui/button"` not `import { Button } from "@/components/ui"`
3. **Check bundle size** with `npx @next/bundle-analyzer` to find bloated imports

## Monitoring and Logging

### Vercel Function Logs

1. Go to your Vercel project
2. Click **Logs** tab
3. Filter by **Function** to see serverless/edge function output

All `console.log` and `console.error` calls in API routes and Server Components appear here.

### Adding Structured Logging

```typescript
// lib/logger.ts
export function log(
  level: "info" | "warn" | "error",
  message: string,
  data?: Record<string, unknown>
) {
  const entry = {
    timestamp: new Date().toISOString(),
    level,
    message,
    ...data,
  };
  if (level === "error") {
    console.error(JSON.stringify(entry));
  } else {
    console.log(JSON.stringify(entry));
  }
}
```

```typescript
// app/api/example/route.ts
import { log } from "@/lib/logger";
import { NextResponse } from "next/server";

export async function GET() {
  try {
    log("info", "Processing request", { route: "/api/example" });
    return NextResponse.json({ ok: true });
  } catch (error) {
    log("error", "Request failed", {
      route: "/api/example",
      error: error instanceof Error ? error.message : "Unknown error",
    });
    return NextResponse.json({ error: "Server error" }, { status: 500 });
  }
}
```

### Health Check Endpoint

```typescript
// app/api/health/route.ts
import { createClient } from "@/lib/supabase/server";
import { NextResponse } from "next/server";

export const dynamic = "force-dynamic";

export async function GET() {
  const checks: Record<string, string> = {};

  try {
    const supabase = await createClient();
    const { error } = await supabase.from("_health").select("*").limit(1);
    checks.database = error ? "unhealthy" : "healthy";
  } catch {
    checks.database = "unreachable";
  }

  const allHealthy = Object.values(checks).every((s) => s === "healthy");

  return NextResponse.json(
    { status: allHealthy ? "healthy" : "degraded", checks },
    { status: allHealthy ? 200 : 503 }
  );
}
```

## Production Debugging Checklist

When something breaks in production:

1. **Check Vercel Logs** — look for error messages in the Functions tab
2. **Check deployment status** — was a recent deploy the cause? Rollback if so
3. **Check environment variables** — are all required vars set for Production?
4. **Check Supabase** — is the database up? Check the Supabase dashboard
5. **Check DNS** — if the site is unreachable, verify DNS records haven't changed
6. **Check rate limits** — Supabase free tier has connection limits
7. **Test locally** — pull production env vars and run `next build && next start`

See `references/build-errors.md` for specific error messages and fixes, and `references/optimization.md` for performance tuning.
