# Security Headers Configuration

Complete security headers for next.config.ts — CSP, HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy, Permissions-Policy.

## Full next.config.ts

```typescript
// next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  async headers() {
    return [
      {
        // Apply to all routes
        source: "/(.*)",
        headers: [
          // Content Security Policy — controls what can load on your page
          {
            key: "Content-Security-Policy",
            value: [
              // Default: only allow resources from your own domain
              "default-src 'self'",

              // Scripts: your domain + inline scripts (needed for Next.js) + Cloudflare Turnstile
              "script-src 'self' 'unsafe-inline' 'unsafe-eval' https://challenges.cloudflare.com",

              // Styles: your domain + inline styles (needed for Tailwind)
              "style-src 'self' 'unsafe-inline'",

              // Images: your domain + data URIs + any HTTPS source
              "img-src 'self' data: https:",

              // Fonts: your domain only
              "font-src 'self'",

              // API calls: your domain + Supabase
              "connect-src 'self' https://*.supabase.co wss://*.supabase.co",

              // Iframes: Turnstile + Stripe
              "frame-src https://challenges.cloudflare.com https://js.stripe.com",

              // Forms: only submit to your own domain
              "form-action 'self'",

              // Base URI: prevent base tag hijacking
              "base-uri 'self'",

              // Don't allow your site to be embedded in objects
              "object-src 'none'",

              // Upgrade HTTP requests to HTTPS
              "upgrade-insecure-requests",
            ].join("; "),
          },

          // Prevent clickjacking — don't allow your site in iframes
          {
            key: "X-Frame-Options",
            value: "DENY",
          },

          // Prevent MIME type sniffing — browser must trust the Content-Type header
          {
            key: "X-Content-Type-Options",
            value: "nosniff",
          },

          // Control what information is sent in the Referer header
          {
            key: "Referrer-Policy",
            value: "strict-origin-when-cross-origin",
          },

          // Disable browser features you don't need
          {
            key: "Permissions-Policy",
            value: [
              "camera=()",
              "microphone=()",
              "geolocation=()",
              "interest-cohort=()",
              "payment=(self)",
              "usb=()",
              "magnetometer=()",
              "gyroscope=()",
              "accelerometer=()",
            ].join(", "),
          },

          // Force HTTPS for 1 year, including subdomains
          {
            key: "Strict-Transport-Security",
            value: "max-age=31536000; includeSubDomains; preload",
          },

          // Prevent XSS attacks (legacy browsers)
          {
            key: "X-XSS-Protection",
            value: "1; mode=block",
          },

          // Prevent DNS prefetching for external domains
          {
            key: "X-DNS-Prefetch-Control",
            value: "on",
          },
        ],
      },
    ];
  },
};

export default nextConfig;
```

## CSP Variations

### Minimal CSP (Start Here)

```typescript
{
  key: "Content-Security-Policy",
  value: "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; connect-src 'self' https://*.supabase.co"
}
```

### CSP with Google Fonts

```typescript
{
  key: "Content-Security-Policy",
  value: [
    "default-src 'self'",
    "script-src 'self' 'unsafe-inline' 'unsafe-eval'",
    "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
    "font-src 'self' https://fonts.gstatic.com",
    "img-src 'self' data: https:",
    "connect-src 'self' https://*.supabase.co",
  ].join("; ")
}
```

### CSP with Analytics (Vercel Analytics, Plausible)

```typescript
{
  key: "Content-Security-Policy",
  value: [
    "default-src 'self'",
    "script-src 'self' 'unsafe-inline' 'unsafe-eval' https://va.vercel-scripts.com https://plausible.io",
    "style-src 'self' 'unsafe-inline'",
    "img-src 'self' data: https:",
    "connect-src 'self' https://*.supabase.co https://vitals.vercel-insights.com https://plausible.io",
  ].join("; ")
}
```

## Testing Your Headers

After deploying, verify headers are correct:

```bash
# Check headers on your deployed site
curl -I https://yourdomain.com

# Or use SecurityHeaders.com to get a grade
# https://securityheaders.com/?q=yourdomain.com
```

## Report-Only Mode (Test Before Enforcing)

If you're worried about breaking things, start with report-only mode:

```typescript
{
  key: "Content-Security-Policy-Report-Only",
  value: "default-src 'self'; report-uri /api/csp-report"
}
```

```typescript
// app/api/csp-report/route.ts
import { NextRequest, NextResponse } from "next/server";

export async function POST(request: NextRequest) {
  const report = await request.json();
  console.warn("CSP Violation:", JSON.stringify(report, null, 2));
  return NextResponse.json({ received: true });
}
```

This logs violations without blocking anything, so you can see what would break before enforcing the policy.
