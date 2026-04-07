# Vercel Configuration Reference

## Full vercel.json Structure

```json
// vercel.json
{
  "buildCommand": "next build",
  "installCommand": "npm install",
  "framework": "nextjs",
  "regions": ["iad1"],
  "crons": [
    {
      "path": "/api/cron/daily-cleanup",
      "schedule": "0 0 * * *"
    },
    {
      "path": "/api/cron/hourly-sync",
      "schedule": "0 * * * *"
    }
  ],
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Frame-Options", "value": "DENY" },
        { "key": "X-Content-Type-Options", "value": "nosniff" },
        { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" },
        { "key": "Permissions-Policy", "value": "camera=(), microphone=(), geolocation=()" },
        {
          "key": "Content-Security-Policy",
          "value": "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data:; connect-src 'self' https://*.supabase.co https://generativelanguage.googleapis.com"
        }
      ]
    },
    {
      "source": "/api/(.*)",
      "headers": [
        { "key": "Cache-Control", "value": "no-store, max-age=0" }
      ]
    },
    {
      "source": "/_next/static/(.*)",
      "headers": [
        { "key": "Cache-Control", "value": "public, max-age=31536000, immutable" }
      ]
    }
  ],
  "redirects": [
    {
      "source": "/old-path",
      "destination": "/new-path",
      "permanent": true
    },
    {
      "source": "/blog/old-slug",
      "destination": "/blog/new-slug",
      "permanent": true
    },
    {
      "source": "/app",
      "destination": "https://app.yourdomain.com",
      "permanent": false
    }
  ],
  "rewrites": [
    {
      "source": "/api/proxy/:path*",
      "destination": "https://external-api.com/:path*"
    },
    {
      "source": "/sitemap.xml",
      "destination": "/api/sitemap"
    }
  ]
}
```

## Cron Jobs

Cron jobs run on a schedule. They call a route in your app.

```typescript
// app/api/cron/daily-cleanup/route.ts
import { NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";

export const maxDuration = 60;

export async function GET(request: Request) {
  // Verify the request is from Vercel Cron
  const authHeader = request.headers.get("authorization");
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  try {
    const supabase = await createClient();
    const thirtyDaysAgo = new Date();
    thirtyDaysAgo.setDate(thirtyDaysAgo.getDate() - 30);

    const { error } = await supabase
      .from("temp_data")
      .delete()
      .lt("created_at", thirtyDaysAgo.toISOString());

    if (error) {
      console.error("Cleanup failed:", error);
      return NextResponse.json({ error: "Cleanup failed" }, { status: 500 });
    }

    return NextResponse.json({ success: true });
  } catch (error) {
    console.error("Cron error:", error);
    return NextResponse.json({ error: "Internal error" }, { status: 500 });
  }
}
```

Set `CRON_SECRET` in Vercel environment variables to secure cron endpoints.

## Common Redirect Patterns

```json
// Redirect www to non-www
{ "source": "/:path*", "has": [{ "type": "host", "value": "www.yourdomain.com" }], "destination": "https://yourdomain.com/:path*", "permanent": true }

// Redirect HTTP to HTTPS (Vercel does this automatically)

// Trailing slash removal (Next.js handles this via next.config.ts trailingSlash option)
```

## Region Selection

Common region codes:
- `iad1` — Washington, D.C. (US East) — closest to many US users
- `sfo1` — San Francisco (US West)
- `lhr1` — London
- `hnd1` — Tokyo
- `cdg1` — Paris
- `sin1` — Singapore
- `syd1` — Sydney

Choose the region closest to your Supabase project's region to minimize latency.

## Function Configuration per Route

```typescript
// app/api/long-task/route.ts

// Maximum execution time in seconds
export const maxDuration = 60; // Pro plan: up to 60s

// Choose runtime
export const runtime = "nodejs"; // or "edge"

// Dynamic behavior
export const dynamic = "force-dynamic"; // never cache this route
```
