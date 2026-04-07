---
name: security-hardening
description: "Use this skill whenever the user mentions security, CAPTCHA, Turnstile, rate limiting, XSS, CSRF, CSP, headers, input validation, sanitization, SQL injection, auth security, session management, admin routes, protected routes, CORS, dependency scanning, vulnerability, penetration testing, 'is my app secure', 'before launch', 'production checklist', security audit, or ANY security/protection task — even if they don't explicitly say 'security'. This skill hardens your app before it goes live."
---

# Security Hardening

How to protect your Next.js + Supabase app from attacks. Do all of these before launching.

## Cloudflare Turnstile CAPTCHA

Free, invisible CAPTCHA that stops bots. Get keys from [Cloudflare Turnstile](https://dash.cloudflare.com/?to=/:account/turnstile).

```bash
# .env.local
NEXT_PUBLIC_TURNSTILE_SITE_KEY=0x4AAAAAAA...
TURNSTILE_SECRET_KEY=0x4AAAAAAA...
```

```typescript
// components/turnstile.tsx
"use client";
import { useEffect, useRef } from "react";

export function Turnstile({ onVerify }: { onVerify: (token: string) => void }) {
  const ref = useRef<HTMLDivElement>(null);
  useEffect(() => {
    const script = document.createElement("script");
    script.src = "https://challenges.cloudflare.com/turnstile/v0/api.js";
    script.async = true;
    document.head.appendChild(script);
    script.onload = () => {
      if (ref.current && window.turnstile) {
        window.turnstile.render(ref.current, {
          sitekey: process.env.NEXT_PUBLIC_TURNSTILE_SITE_KEY!,
          callback: onVerify,
        });
      }
    };
    return () => { document.head.removeChild(script); };
  }, [onVerify]);
  return <div ref={ref} />;
}
```

```typescript
// lib/turnstile.ts — Server verification
export async function verifyTurnstile(token: string): Promise<boolean> {
  try {
    const res = await fetch("https://challenges.cloudflare.com/turnstile/v0/siteverify", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ secret: process.env.TURNSTILE_SECRET_KEY, response: token }),
    });
    const data = await res.json();
    return data.success === true;
  } catch { return false; }
}
```

## Rate Limiting

### With Upstash Redis (Production)

```bash
npm install @upstash/ratelimit @upstash/redis
```

```typescript
// lib/rate-limit.ts
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";

export const rateLimiter = new Ratelimit({
  redis: new Redis({ url: process.env.UPSTASH_REDIS_REST_URL!, token: process.env.UPSTASH_REDIS_REST_TOKEN! }),
  limiter: Ratelimit.slidingWindow(10, "10 s"),
});
```

```typescript
// Usage in any API route:
const ip = request.headers.get("x-forwarded-for") ?? "anonymous";
const { success } = await rateLimiter.limit(ip);
if (!success) return NextResponse.json({ error: "Too many requests" }, { status: 429 });
```

### Simple In-Memory (Small Apps)

```typescript
// lib/simple-rate-limit.ts
const requests = new Map<string, { count: number; resetAt: number }>();

export function simpleRateLimit(key: string, max = 10, windowMs = 60000): boolean {
  const now = Date.now();
  const entry = requests.get(key);
  if (!entry || now > entry.resetAt) { requests.set(key, { count: 1, resetAt: now + windowMs }); return true; }
  if (entry.count >= max) return false;
  entry.count++;
  return true;
}
```

## Input Validation with Zod

Use Zod on EVERY form and API route.

```typescript
// lib/validations.ts
import { z } from "zod";

export const contactSchema = z.object({
  name: z.string().min(1).max(100).regex(/^[a-zA-Z\s'-]+$/, "Invalid characters"),
  email: z.string().email(),
  message: z.string().min(10).max(5000),
});
```

```typescript
// In any API route:
const parsed = contactSchema.safeParse(await request.json());
if (!parsed.success)
  return NextResponse.json({ error: "Invalid input", details: parsed.error.flatten().fieldErrors }, { status: 400 });
// parsed.data is now typed and safe
```

## XSS Prevention

React escapes HTML by default. **Never** use `dangerouslySetInnerHTML` with user input. If you must render HTML, sanitize first:

```bash
npm install isomorphic-dompurify
```

```typescript
// lib/sanitize.ts
import DOMPurify from "isomorphic-dompurify";

export function sanitizeHtml(dirty: string): string {
  return DOMPurify.sanitize(dirty, {
    ALLOWED_TAGS: ["b", "i", "em", "strong", "a", "p", "br", "ul", "ol", "li"],
    ALLOWED_ATTR: ["href", "target", "rel"],
  });
}
```

## Content Security Policy

See `references/headers-config.md` for the complete config with all variations.

```typescript
// next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  async headers() {
    return [{
      source: "/(.*)",
      headers: [
        { key: "Content-Security-Policy", value: [
          "default-src 'self'",
          "script-src 'self' 'unsafe-inline' 'unsafe-eval' https://challenges.cloudflare.com",
          "style-src 'self' 'unsafe-inline'",
          "img-src 'self' data: https:",
          "connect-src 'self' https://*.supabase.co",
          "frame-src https://challenges.cloudflare.com https://js.stripe.com",
        ].join("; ") },
        { key: "X-Frame-Options", value: "DENY" },
        { key: "X-Content-Type-Options", value: "nosniff" },
        { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
        { key: "Permissions-Policy", value: "camera=(), microphone=(), geolocation=()" },
        { key: "Strict-Transport-Security", value: "max-age=31536000; includeSubDomains" },
      ],
    }];
  },
};
export default nextConfig;
```

## CSRF Protection

Next.js Server Actions have built-in CSRF protection. For custom API routes:

```typescript
// lib/csrf.ts
import crypto from "crypto";

export function generateCsrfToken(): string { return crypto.randomBytes(32).toString("hex"); }
export function validateCsrfToken(token: string, expected: string): boolean {
  if (!token || !expected) return false;
  return crypto.timingSafeEqual(Buffer.from(token), Buffer.from(expected));
}
```

## Auth Session Security

See `references/auth-security.md` for full middleware, OAuth, and RLS patterns.

```typescript
// middleware.ts
import { NextResponse, type NextRequest } from "next/server";
import { createServerClient } from "@supabase/ssr";

export async function middleware(request: NextRequest) {
  let response = NextResponse.next({ request });
  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!, process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    { cookies: {
      getAll() { return request.cookies.getAll(); },
      setAll(cookiesToSet) {
        cookiesToSet.forEach(({ name, value, options }) => {
          request.cookies.set(name, value);
          response.cookies.set(name, value, { ...options, httpOnly: true, secure: process.env.NODE_ENV === "production", sameSite: "lax" });
        });
        response = NextResponse.next({ request });
      },
    } }
  );

  // ALWAYS use getUser() not getSession() — getSession() doesn't verify the JWT
  const { data: { user } } = await supabase.auth.getUser();

  if (request.nextUrl.pathname.startsWith("/dashboard") && !user)
    return NextResponse.redirect(new URL("/login", request.url));
  return response;
}

export const config = { matcher: ["/dashboard/:path*", "/admin/:path*"] };
```

## Admin Route Protection

```typescript
// lib/auth/require-admin.ts
import { createClient } from "@/lib/supabase/server";
import { redirect } from "next/navigation";

export async function requireAdmin() {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect("/login");

  const { data: profile } = await supabase.from("profiles").select("role").eq("id", user.id).single();
  if (profile?.role !== "admin") redirect("/unauthorized");
  return user;
}
```

## CORS Configuration

```typescript
// app/api/public/route.ts
import { NextRequest, NextResponse } from "next/server";

const ALLOWED_ORIGINS = ["https://yourdomain.com", "https://app.yourdomain.com"];

export async function GET(request: NextRequest) {
  const origin = request.headers.get("origin") ?? "";
  const response = NextResponse.json({ data: "public" });
  if (ALLOWED_ORIGINS.includes(origin)) {
    response.headers.set("Access-Control-Allow-Origin", origin);
    response.headers.set("Access-Control-Allow-Methods", "GET, POST, OPTIONS");
    response.headers.set("Access-Control-Allow-Headers", "Content-Type, Authorization");
  }
  return response;
}
```

## Environment Variable Audit

```typescript
// lib/env-check.ts
export function auditEnvVars() {
  const required = ["NEXT_PUBLIC_SUPABASE_URL", "SUPABASE_SERVICE_ROLE_KEY", "STRIPE_SECRET_KEY", "GEMINI_API_KEY"];
  const missing = required.filter((k) => !process.env[k]);
  if (missing.length) throw new Error(`Missing env vars: ${missing.join(", ")}`);

  for (const [key, value] of Object.entries(process.env)) {
    if (key.startsWith("NEXT_PUBLIC_") && value?.startsWith("sk_"))
      throw new Error(`SECRET LEAK: ${key} contains a secret key!`);
  }
}
```

## Dependency Scanning

```bash
npm audit          # Check for vulnerabilities
npm audit fix      # Auto-fix where possible
npx snyk test      # Deeper scanning (free tier)
```

See `references/` for complete configs and checklists:
- `headers-config.md` — Full security headers with CSP variations
- `auth-security.md` — Session management, OAuth, RLS patterns
- `audit-checklist.md` — 30+ item pre-launch security checklist
