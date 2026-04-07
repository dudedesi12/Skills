# Vercel Cron Patterns

Complete setup for scheduled jobs with Vercel Cron, including common schedules, idempotent patterns, and error recovery.

## Vercel Cron Configuration

```json
// vercel.json
{
  "crons": [
    {
      "path": "/api/cron/daily-digest",
      "schedule": "0 9 * * *"
    },
    {
      "path": "/api/cron/cleanup-expired",
      "schedule": "0 */6 * * *"
    },
    {
      "path": "/api/cron/sync-data",
      "schedule": "*/15 * * * *"
    },
    {
      "path": "/api/cron/weekly-report",
      "schedule": "0 10 * * 1"
    }
  ]
}
```

## Common Cron Schedules

| Schedule | Cron Expression | Notes |
|----------|----------------|-------|
| Every 15 minutes | `*/15 * * * *` | Minimum on Vercel Hobby |
| Every hour | `0 * * * *` | On the hour |
| Every 6 hours | `0 */6 * * *` | 4 times per day |
| Daily at 9 AM UTC | `0 9 * * *` | Morning jobs |
| Daily at midnight UTC | `0 0 * * *` | Nightly cleanup |
| Weekly Monday 10 AM UTC | `0 10 * * 1` | Weekly reports |
| First of month | `0 0 1 * *` | Monthly billing |

## Secured Cron Route Handler

Every cron route MUST verify the request comes from Vercel.

```typescript
// app/api/cron/daily-digest/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export async function GET(request: NextRequest) {
  // Verify the request is from Vercel Cron
  const authHeader = request.headers.get("authorization");
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const startTime = Date.now();

  try {
    // Your cron job logic here
    const { data: users, error } = await supabase
      .from("users")
      .select("id, email")
      .eq("daily_digest_enabled", true);

    if (error) throw error;

    let processed = 0;
    let failed = 0;

    for (const user of users ?? []) {
      try {
        // Process each user
        await sendDigestEmail(user.id, user.email);
        processed++;
      } catch (err) {
        console.error(`Failed for user ${user.id}:`, err);
        failed++;
      }
    }

    return NextResponse.json({
      success: true,
      processed,
      failed,
      durationMs: Date.now() - startTime,
    });
  } catch (err) {
    console.error("Cron job failed:", err);
    return NextResponse.json(
      { error: "Cron job failed", durationMs: Date.now() - startTime },
      { status: 500 }
    );
  }
}

async function sendDigestEmail(userId: string, email: string) {
  // Email sending logic
  console.log(`Sending digest to ${email}`);
}
```

## Idempotent Job Pattern

Cron jobs can run twice (Vercel may retry). Make sure running twice doesn't cause problems.

```typescript
// app/api/cron/process-payments/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export async function GET(request: NextRequest) {
  const authHeader = request.headers.get("authorization");
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  // Generate a unique run ID for this execution
  const runId = crypto.randomUUID();
  const today = new Date().toISOString().split("T")[0];

  // Check if this job already ran today (idempotency)
  const { data: existingRun } = await supabase
    .from("cron_runs")
    .select("id")
    .eq("job_name", "process-payments")
    .eq("run_date", today)
    .eq("status", "completed")
    .single();

  if (existingRun) {
    return NextResponse.json({
      skipped: true,
      reason: "Already ran today",
    });
  }

  // Record that we're starting
  await supabase.from("cron_runs").insert({
    id: runId,
    job_name: "process-payments",
    run_date: today,
    status: "running",
    started_at: new Date().toISOString(),
  });

  try {
    // Process payments that haven't been processed yet
    const { data: pendingPayments } = await supabase
      .from("payments")
      .select("*")
      .eq("status", "pending")
      .is("processed_at", null);

    for (const payment of pendingPayments ?? []) {
      // Use a transaction-like pattern: mark as processing first
      const { error: updateError } = await supabase
        .from("payments")
        .update({ status: "processing" })
        .eq("id", payment.id)
        .eq("status", "pending"); // Only update if still pending

      if (updateError) continue; // Another run already picked this up

      try {
        await processPayment(payment);
        await supabase
          .from("payments")
          .update({ status: "completed", processed_at: new Date().toISOString() })
          .eq("id", payment.id);
      } catch (err) {
        await supabase
          .from("payments")
          .update({ status: "failed", error: String(err) })
          .eq("id", payment.id);
      }
    }

    // Mark run as completed
    await supabase
      .from("cron_runs")
      .update({ status: "completed", finished_at: new Date().toISOString() })
      .eq("id", runId);

    return NextResponse.json({ success: true });
  } catch (err) {
    await supabase
      .from("cron_runs")
      .update({ status: "failed", error: String(err), finished_at: new Date().toISOString() })
      .eq("id", runId);

    return NextResponse.json({ error: "Job failed" }, { status: 500 });
  }
}

async function processPayment(payment: Record<string, unknown>) {
  console.log("Processing payment:", payment.id);
}
```

## Cleanup Job Pattern

```typescript
// app/api/cron/cleanup-expired/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export async function GET(request: NextRequest) {
  const authHeader = request.headers.get("authorization");
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  try {
    const thirtyDaysAgo = new Date(
      Date.now() - 30 * 24 * 60 * 60 * 1000
    ).toISOString();

    // Delete expired sessions
    const { count: sessionsDeleted } = await supabase
      .from("sessions")
      .delete({ count: "exact" })
      .lt("expires_at", new Date().toISOString());

    // Delete old API logs
    const { count: logsDeleted } = await supabase
      .from("api_logs")
      .delete({ count: "exact" })
      .lt("created_at", thirtyDaysAgo);

    // Delete unverified accounts older than 7 days
    const sevenDaysAgo = new Date(
      Date.now() - 7 * 24 * 60 * 60 * 1000
    ).toISOString();

    const { count: accountsDeleted } = await supabase
      .from("profiles")
      .delete({ count: "exact" })
      .eq("email_verified", false)
      .lt("created_at", sevenDaysAgo);

    return NextResponse.json({
      success: true,
      cleaned: {
        expiredSessions: sessionsDeleted ?? 0,
        oldLogs: logsDeleted ?? 0,
        unverifiedAccounts: accountsDeleted ?? 0,
      },
    });
  } catch (err) {
    console.error("Cleanup cron failed:", err);
    return NextResponse.json({ error: "Cleanup failed" }, { status: 500 });
  }
}
```

## Cron Runs Table (for tracking)

```sql
-- Run in Supabase SQL Editor
create table if not exists cron_runs (
  id uuid primary key,
  job_name text not null,
  run_date date not null,
  status text not null default 'running',
  error text,
  started_at timestamptz not null default now(),
  finished_at timestamptz,
  unique(job_name, run_date, status)
);

create index idx_cron_runs_job_date on cron_runs(job_name, run_date);
```

## Environment Variable

```bash
# .env.local
CRON_SECRET=your-random-secret-string-here

# Generate a good secret:
# node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

Add `CRON_SECRET` to Vercel environment variables for production.
