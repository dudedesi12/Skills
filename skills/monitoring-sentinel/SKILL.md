---
name: monitoring-sentinel
description: "Use this skill whenever the user mentions monitoring, Sentry, error tracking, error boundary, logging, structured logging, uptime, health check, alerts, alerting, PagerDuty, Slack alerts, dead letter queue, failed jobs, cron monitoring, latency tracking, observability, tracing, correlation ID, 'I need alerts', 'know when something breaks', 'production monitoring', log aggregation, incident response, 'my cron job failed', or ANY monitoring/alerting/observability task — even if they don't explicitly say 'monitoring'. This skill keeps you informed when things break."
---

# Monitoring Sentinel

Know when things break before your users tell you. This skill focuses on Sentry integration and alerting. Cross-reference: deployment-ops for basic logging and health checks.

## 1. Sentry Setup for Next.js

```bash
npx @sentry/wizard@latest -i nextjs
```

This creates `sentry.client.config.ts`, `sentry.server.config.ts`, `sentry.edge.config.ts`, and wraps your `next.config.ts`.

```typescript
// sentry.client.config.ts
import * as Sentry from "@sentry/nextjs";

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  environment: process.env.VERCEL_ENV ?? "development",
  tracesSampleRate: process.env.NODE_ENV === "production" ? 0.1 : 1.0,
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
  integrations: [Sentry.replayIntegration()],
});
```

```typescript
// sentry.server.config.ts
import * as Sentry from "@sentry/nextjs";

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.VERCEL_ENV ?? "development",
  tracesSampleRate: 0.1,
});
```

## 2. Error Boundaries with Sentry

```tsx
// app/error.tsx — Global error boundary
"use client";
import * as Sentry from "@sentry/nextjs";
import { useEffect } from "react";

export default function Error({ error, reset }: { error: Error & { digest?: string }; reset: () => void }) {
  useEffect(() => {
    Sentry.captureException(error);
  }, [error]);

  return (
    <div className="flex min-h-screen items-center justify-center">
      <div className="text-center">
        <h2 className="text-2xl font-bold">Something went wrong</h2>
        <p className="mt-2 text-gray-600">{error.message}</p>
        <button onClick={reset} className="mt-4 rounded bg-blue-600 px-4 py-2 text-white hover:bg-blue-700">
          Try again
        </button>
      </div>
    </div>
  );
}
```

### Attach User Context

```typescript
// lib/sentry-user.ts
import * as Sentry from "@sentry/nextjs";

export function setSentryUser(user: { id: string; email: string; role: string } | null) {
  if (user) {
    Sentry.setUser({ id: user.id, email: user.email, role: user.role });
  } else {
    Sentry.setUser(null);
  }
}
```

## 3. Structured Logging

```typescript
// lib/logger.ts
type LogLevel = "debug" | "info" | "warn" | "error";

interface LogContext {
  userId?: string;
  requestId?: string;
  route?: string;
  duration?: number;
  [key: string]: unknown;
}

function log(level: LogLevel, message: string, context: LogContext = {}) {
  const entry = {
    timestamp: new Date().toISOString(),
    level,
    message,
    ...context,
  };

  if (level === "error") console.error(JSON.stringify(entry));
  else if (level === "warn") console.warn(JSON.stringify(entry));
  else console.log(JSON.stringify(entry));
}

export const logger = {
  debug: (msg: string, ctx?: LogContext) => log("debug", msg, ctx),
  info: (msg: string, ctx?: LogContext) => log("info", msg, ctx),
  warn: (msg: string, ctx?: LogContext) => log("warn", msg, ctx),
  error: (msg: string, ctx?: LogContext) => log("error", msg, ctx),
};
```

### Request Correlation IDs

```typescript
// middleware.ts — add correlation ID to every request
import { NextResponse, type NextRequest } from "next/server";
import { randomUUID } from "crypto";

export function middleware(request: NextRequest) {
  const requestId = request.headers.get("x-request-id") ?? randomUUID();
  const response = NextResponse.next();
  response.headers.set("x-request-id", requestId);
  return response;
}
```

## 4. API Route Monitoring

```typescript
// lib/api/with-monitoring.ts
import { NextRequest, NextResponse } from "next/server";
import { logger } from "@/lib/logger";
import * as Sentry from "@sentry/nextjs";

type Handler = (request: NextRequest) => Promise<NextResponse>;

export function withMonitoring(handler: Handler): Handler {
  return async (request: NextRequest) => {
    const start = Date.now();
    const route = request.nextUrl.pathname;
    const requestId = request.headers.get("x-request-id") ?? "unknown";

    try {
      const response = await handler(request);
      const duration = Date.now() - start;

      logger.info("API request completed", { route, duration, requestId, status: response.status });

      if (duration > 5000) {
        logger.warn("Slow API request", { route, duration, requestId });
      }

      return response;
    } catch (error) {
      const duration = Date.now() - start;
      logger.error("API request failed", {
        route, duration, requestId,
        error: error instanceof Error ? error.message : "Unknown error",
      });
      Sentry.captureException(error);
      return NextResponse.json({ error: "Internal server error" }, { status: 500 });
    }
  };
}

// Usage:
export const GET = withMonitoring(async (request) => {
  // your route logic
  return NextResponse.json({ data: "ok" });
});
```

## 5. Cron Job Health Monitoring

```sql
-- supabase/migrations/xxx_cron_health.sql
CREATE TABLE cron_heartbeats (
  id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  job_name text NOT NULL,
  status text NOT NULL DEFAULT 'running', -- running, success, failed
  started_at timestamptz DEFAULT now(),
  completed_at timestamptz,
  duration_ms integer,
  error_message text,
  details jsonb DEFAULT '{}'
);

CREATE INDEX idx_cron_heartbeats_job ON cron_heartbeats(job_name, started_at DESC);
```

```typescript
// lib/monitoring/cron-health.ts
import { createClient } from "@/lib/supabase/server";

export async function startCronJob(jobName: string): Promise<string> {
  const supabase = await createClient();
  const { data } = await supabase
    .from("cron_heartbeats")
    .insert({ job_name: jobName, status: "running" })
    .select("id")
    .single();
  return data!.id;
}

export async function completeCronJob(heartbeatId: string, success: boolean, error?: string) {
  const supabase = await createClient();
  const { data: heartbeat } = await supabase
    .from("cron_heartbeats").select("started_at").eq("id", heartbeatId).single();

  const duration = heartbeat ? Date.now() - new Date(heartbeat.started_at).getTime() : 0;

  await supabase.from("cron_heartbeats").update({
    status: success ? "success" : "failed",
    completed_at: new Date().toISOString(),
    duration_ms: duration,
    error_message: error,
  }).eq("id", heartbeatId);

  if (!success) {
    logger.error(`Cron job failed: ${error}`, { jobName: heartbeatId });
  }
}

// Usage in a cron route:
export async function GET() {
  const id = await startCronJob("daily-cleanup");
  try {
    await performCleanup();
    await completeCronJob(id, true);
    return NextResponse.json({ success: true });
  } catch (error) {
    await completeCronJob(id, false, error instanceof Error ? error.message : "Unknown");
    return NextResponse.json({ error: "Cron failed" }, { status: 500 });
  }
}
```

## 6. Health Check Endpoint

```typescript
// app/api/health/route.ts
import { NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";

export async function GET() {
  const checks: Record<string, { status: string; latency?: number }> = {};

  // Check Supabase
  const dbStart = Date.now();
  try {
    const supabase = await createClient();
    await supabase.from("profiles").select("id").limit(1);
    checks.database = { status: "ok", latency: Date.now() - dbStart };
  } catch {
    checks.database = { status: "error", latency: Date.now() - dbStart };
  }

  const allOk = Object.values(checks).every((c) => c.status === "ok");
  return NextResponse.json(
    { status: allOk ? "healthy" : "degraded", checks, timestamp: new Date().toISOString() },
    { status: allOk ? 200 : 503 }
  );
}
```

## 7. Alert Routing

```typescript
// lib/monitoring/alerts.ts
type AlertSeverity = "info" | "warning" | "critical";

export async function sendAlert(severity: AlertSeverity, message: string, details?: Record<string, unknown>) {
  logger.warn(`Alert [${severity}]: ${message}`, details);

  if (severity === "critical" || severity === "warning") {
    await sendSlackAlert(severity, message, details);
  }
}

async function sendSlackAlert(severity: AlertSeverity, message: string, details?: Record<string, unknown>) {
  const webhookUrl = process.env.SLACK_WEBHOOK_URL;
  if (!webhookUrl) return;

  const color = severity === "critical" ? "#dc2626" : "#f59e0b";

  await fetch(webhookUrl, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      attachments: [{
        color,
        title: `[${severity.toUpperCase()}] ${message}`,
        text: details ? JSON.stringify(details, null, 2) : undefined,
        ts: Math.floor(Date.now() / 1000),
      }],
    }),
  });
}
```

## 8. Dead Letter Queue

```sql
CREATE TABLE dead_letter_queue (
  id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  source text NOT NULL,           -- 'webhook', 'cron', 'api'
  payload jsonb NOT NULL,
  error_message text,
  retry_count integer DEFAULT 0,
  max_retries integer DEFAULT 3,
  next_retry_at timestamptz,
  status text DEFAULT 'pending',  -- pending, retrying, resolved, abandoned
  created_at timestamptz DEFAULT now()
);
```

```typescript
// lib/monitoring/dead-letter.ts
export async function addToDeadLetter(source: string, payload: unknown, error: string) {
  const supabase = await createClient();
  await supabase.from("dead_letter_queue").insert({
    source,
    payload,
    error_message: error,
    next_retry_at: new Date(Date.now() + 60_000).toISOString(), // Retry in 1 min
  });
}
```

## Rules

1. **Set up Sentry on day one** — Don't wait for production bugs. Configure it before launch.
2. **Use structured logging** — JSON logs are searchable. String logs are not.
3. **Add correlation IDs** — Track a request across your entire system.
4. **Monitor every cron job** — Record start, end, duration, and status for each run.
5. **Set up Slack alerts** — Critical errors should notify you immediately.
6. **Use dead letter queues** — Don't lose failed webhooks or jobs. Queue them for retry.
7. **Track API latency** — Log response times. Alert on slow requests (> 5s).
8. **Health check dependencies** — Your health endpoint should check Supabase, not just return 200.
9. **Don't log PII** — Never log passwords, tokens, or sensitive user data.
10. **Review alerts weekly** — Tune thresholds to reduce noise and prevent alert fatigue.

See `references/` for detailed guides:
- `sentry-config.md` — Complete Sentry setup
- `logging-patterns.md` — Structured logging patterns
- `alerting-setup.md` — Alert routing and escalation
