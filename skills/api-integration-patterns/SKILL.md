---
name: api-integration-patterns
description: "Use this skill whenever the user mentions API, external service, integration, webhook, cron job, scheduled task, rate limiting, retry logic, Gemini, AI features, AI integration, fetch, HTTP client, REST API, third-party service, API key, secret management, caching, SWR, request logging, timeout, government API, external data source, 'connect to', 'call an API', 'use Gemini', or ANY external service connection task — even if they don't explicitly say 'API'. This skill wires up every external service correctly."
---

# API Integration Patterns

How to connect your Next.js app to any external service — APIs, webhooks, cron jobs, Gemini AI, and more.

## API Client Wrapper

Never call `fetch` directly. Wrap it with retry, timeout, and rate limiting. See `references/api-client.md` for the full class.

```typescript
// lib/api-client.ts
export class ApiClient {
  constructor(private config: {
    baseUrl: string; timeout?: number; maxRetries?: number;
    headers?: Record<string, string>;
  }) {}

  async request<T>(endpoint: string, options: RequestInit = {}): Promise<{
    data: T | null; error: string | null; status: number;
  }> {
    for (let attempt = 0; attempt <= (this.config.maxRetries ?? 3); attempt++) {
      try {
        const controller = new AbortController();
        const timeoutId = setTimeout(() => controller.abort(), this.config.timeout ?? 10000);
        const response = await fetch(`${this.config.baseUrl}${endpoint}`, {
          ...options, signal: controller.signal,
          headers: { ...this.config.headers, ...options.headers },
        });
        clearTimeout(timeoutId);

        if (response.status === 429) {
          const wait = parseInt(response.headers.get("Retry-After") ?? "0") * 1000 || 2 ** attempt * 1000;
          await new Promise((r) => setTimeout(r, wait));
          continue;
        }
        if (!response.ok) return { data: null, error: `HTTP ${response.status}`, status: response.status };
        return { data: (await response.json()) as T, error: null, status: response.status };
      } catch (err) {
        if (attempt === (this.config.maxRetries ?? 3))
          return { data: null, error: err instanceof Error ? err.message : "Unknown", status: 0 };
        await new Promise((r) => setTimeout(r, 2 ** attempt * 1000));
      }
    }
    return { data: null, error: "Max retries exceeded", status: 0 };
  }

  get<T>(endpoint: string) { return this.request<T>(endpoint, { method: "GET" }); }
  post<T>(endpoint: string, body: unknown) {
    return this.request<T>(endpoint, { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(body) });
  }
}
```

## Webhook Handlers

Webhooks are when an external service sends YOUR app a message. Always verify the signature. See `references/webhook-handlers.md` for Stripe, Supabase, and custom patterns.

```typescript
// app/api/webhooks/stripe/route.ts
import { NextRequest, NextResponse } from "next/server";
import Stripe from "stripe";

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, { apiVersion: "2024-12-18.acacia" });

export async function POST(request: NextRequest) {
  const body = await request.text();
  const signature = request.headers.get("stripe-signature");
  if (!signature) return NextResponse.json({ error: "No signature" }, { status: 400 });

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(body, signature, process.env.STRIPE_WEBHOOK_SECRET!);
  } catch (err) {
    console.error("Webhook signature failed:", err);
    return NextResponse.json({ error: "Invalid signature" }, { status: 400 });
  }

  try {
    switch (event.type) {
      case "checkout.session.completed": await handleCheckoutComplete(event.data.object); break;
      case "invoice.payment_failed": await handlePaymentFailed(event.data.object); break;
    }
    return NextResponse.json({ received: true });
  } catch (err) {
    console.error("Webhook handler error:", err);
    return NextResponse.json({ error: "Handler failed" }, { status: 500 });
  }
}
```

## Cron Jobs with Vercel

Cron jobs run on a schedule. See `references/cron-patterns.md` for idempotent patterns and error recovery.

```json
// vercel.json
{
  "crons": [
    { "path": "/api/cron/daily-digest", "schedule": "0 9 * * *" },
    { "path": "/api/cron/cleanup", "schedule": "0 */6 * * *" }
  ]
}
```

Common: `0 * * * *` = hourly, `0 9 * * *` = daily 9AM UTC, `0 0 * * 0` = weekly Sunday.

```typescript
// app/api/cron/daily-digest/route.ts
import { NextRequest, NextResponse } from "next/server";

export async function GET(request: NextRequest) {
  const authHeader = request.headers.get("authorization");
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }
  try {
    // Your job logic here
    return NextResponse.json({ success: true });
  } catch (err) {
    console.error("Cron job failed:", err);
    return NextResponse.json({ error: "Cron failed" }, { status: 500 });
  }
}
```

## Caching Strategies

### ISR (Incremental Static Regeneration)

```typescript
// app/blog/[slug]/page.tsx
export const revalidate = 3600; // Re-fetch every hour

export default async function BlogPost({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const post = await fetchPost(slug);
  if (!post) return <div>Post not found</div>;
  return <article><h1>{post.title}</h1><p>{post.content}</p></article>;
}
```

### SWR for Client-Side

```typescript
// hooks/use-profile.ts
"use client";
import useSWR from "swr";

const fetcher = (url: string) => fetch(url).then((r) => { if (!r.ok) throw new Error("Failed"); return r.json(); });

export function useProfile(userId: string) {
  const { data, error, isLoading, mutate } = useSWR(
    userId ? `/api/profile/${userId}` : null, fetcher,
    { revalidateOnFocus: false, dedupingInterval: 60000, errorRetryCount: 3 }
  );
  return { profile: data, error, isLoading, refresh: mutate };
}
```

## Secret Management

**NEVER put secrets in client-side code.** Only `NEXT_PUBLIC_` vars are visible to the browser.

```bash
# .env.local (never commit this)
STRIPE_SECRET_KEY=sk_live_...
SUPABASE_SERVICE_ROLE_KEY=eyJ...
GEMINI_API_KEY=AIza...
CRON_SECRET=my-cron-secret

# Safe for browser:
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
```

## Gemini API Integration

Three models for three jobs. See `references/gemini-integration.md` for streaming, structured output, function calling, and image analysis.

| Model | Use For | Speed | Cost |
|-------|---------|-------|------|
| `gemini-2.5-pro` | Heavy processing, PDF parsing, cron jobs | Slow | High |
| `gemini-2.0-flash` | User-facing features, chat, search | Fast | Medium |
| `gemini-2.0-flash-lite` | Classification, tagging, simple tasks | Fastest | Low |

```typescript
// lib/gemini.ts
import { GoogleGenerativeAI } from "@google/generative-ai";

if (!process.env.GEMINI_API_KEY) throw new Error("GEMINI_API_KEY is not set");
const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY);

export const geminiPro = genAI.getGenerativeModel({ model: "gemini-2.5-pro" });
export const geminiFlash = genAI.getGenerativeModel({ model: "gemini-2.0-flash" });
export const geminiLite = genAI.getGenerativeModel({ model: "gemini-2.0-flash-lite" });
```

### Streaming Response (User-Facing)

```typescript
// app/api/chat/route.ts
import { NextRequest } from "next/server";
import { geminiFlash } from "@/lib/gemini";

export async function POST(request: NextRequest) {
  try {
    const { message } = await request.json();
    if (!message || typeof message !== "string")
      return new Response(JSON.stringify({ error: "Invalid message" }), { status: 400 });

    const result = await geminiFlash.generateContentStream(message);
    const encoder = new TextEncoder();
    const stream = new ReadableStream({
      async start(controller) {
        try {
          for await (const chunk of result.stream) {
            controller.enqueue(encoder.encode(`data: ${JSON.stringify({ text: chunk.text() })}\n\n`));
          }
          controller.enqueue(encoder.encode("data: [DONE]\n\n"));
          controller.close();
        } catch (err) { controller.error(err); }
      },
    });
    return new Response(stream, { headers: { "Content-Type": "text/event-stream", "Cache-Control": "no-cache" } });
  } catch (err) {
    console.error("Chat API error:", err);
    return new Response(JSON.stringify({ error: "AI request failed" }), { status: 500 });
  }
}
```

### Classification (Fast + Cheap)

```typescript
// lib/ai/classifier.ts
import { geminiLite } from "@/lib/gemini";

export async function classifyTicket(text: string): Promise<"billing" | "technical" | "general" | "urgent"> {
  try {
    const result = await geminiLite.generateContent(
      `Classify this support ticket: billing, technical, general, urgent. Reply with ONLY the category.\n\nTicket: ${text}`
    );
    const category = result.response.text().trim().toLowerCase();
    const valid = ["billing", "technical", "general", "urgent"] as const;
    return valid.includes(category as any) ? (category as any) : "general";
  } catch { return "general"; }
}
```

## Request/Response Logging

```typescript
// lib/api-logger.ts
export async function trackedFetch<T>(service: string, endpoint: string, fetchFn: () => Promise<T>): Promise<T> {
  const start = Date.now();
  try {
    const result = await fetchFn();
    console.log(`[API] ${service} ${endpoint} OK ${Date.now() - start}ms`);
    return result;
  } catch (err) {
    console.error(`[API] ${service} ${endpoint} FAIL ${Date.now() - start}ms`, err);
    throw err;
  }
}
```

## Error Handling for External Services

```typescript
// lib/errors.ts
export class ExternalServiceError extends Error {
  constructor(public service: string, public originalError: unknown, public retryable = false) {
    super(`${service} failed: ${originalError instanceof Error ? originalError.message : "Unknown"}`);
  }
}
```

```typescript
// app/actions/generate-summary.ts
"use server";
import { ExternalServiceError } from "@/lib/errors";

export async function generateSummary(text: string) {
  try {
    const summary = await analyzeDocument(text);
    return { success: true, data: summary };
  } catch (err) {
    if (err instanceof ExternalServiceError && err.retryable)
      return { success: false, error: "Service busy. Please try again." };
    return { success: false, error: "Could not generate summary." };
  }
}
```

## Install Dependencies

```bash
npm install @google/generative-ai swr stripe
```

See `references/` for complete implementations:
- `api-client.md` — Full API client class with all HTTP methods
- `webhook-handlers.md` — Stripe, Supabase, custom webhook patterns
- `cron-patterns.md` — Vercel Cron setup, idempotent jobs, error recovery
- `gemini-integration.md` — All 3 model tiers, streaming, function calling, image analysis
