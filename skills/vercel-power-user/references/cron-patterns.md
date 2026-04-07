# Cron Job Patterns

## Schedule Syntax

```
*    *    *    *    *
│    │    │    │    └── Day of week (0-7, 0=Sun)
│    │    │    └─────── Month (1-12)
│    │    └──────────── Day of month (1-31)
│    └───────────────── Hour (0-23)
└────────────────────── Minute (0-59)
```

## Common Schedules

| Schedule | Cron | Use Case |
|----------|------|----------|
| Every 5 min | `*/5 * * * *` | Health checks, cache refresh |
| Every hour | `0 * * * *` | Data sync, light scraping |
| Every 6 hours | `0 */6 * * *` | Content freshness check |
| Daily 2 AM | `0 2 * * *` | Cleanup, reports |
| Weekly Monday 8 AM | `0 8 * * 1` | Dependency audit, digest email |
| Monthly 1st at midnight | `0 0 1 * *` | Billing, monthly reports |

## Template: Idempotent Cron Job

```typescript
// file: app/api/cron/[jobName]/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { createClient } from '@/lib/supabase/server';

export const maxDuration = 60;

export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ jobName: string }> }
) {
  if (request.headers.get('authorization') !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { jobName } = await params;
  const supabase = await createClient();
  const runId = crypto.randomUUID();

  // Idempotency: check if already ran recently
  const { data: recentRun } = await supabase
    .from('cron_runs')
    .select('id')
    .eq('job_name', jobName)
    .eq('status', 'completed')
    .gte('completed_at', new Date(Date.now() - 4 * 60000).toISOString()) // 4-min window
    .limit(1)
    .single();

  if (recentRun) {
    return NextResponse.json({ skipped: true, reason: 'Already ran recently' });
  }

  // Record start
  await supabase.from('cron_runs').insert({ id: runId, job_name: jobName, status: 'running' });

  try {
    const result = await runJob(jobName);

    await supabase.from('cron_runs').update({
      status: 'completed',
      result,
      completed_at: new Date().toISOString(),
    }).eq('id', runId);

    return NextResponse.json({ success: true, runId, result });
  } catch (err) {
    const message = err instanceof Error ? err.message : 'Job failed';
    await supabase.from('cron_runs').update({
      status: 'failed',
      error: message,
      completed_at: new Date().toISOString(),
    }).eq('id', runId);

    return NextResponse.json({ error: message }, { status: 500 });
  }
}

async function runJob(jobName: string): Promise<Record<string, unknown>> {
  switch (jobName) {
    case 'cleanup': return { deleted: 42 };
    case 'sync': return { synced: 100 };
    default: throw new Error(`Unknown job: ${jobName}`);
  }
}
```

## Cron Runs Table

```sql
create table cron_runs (
  id uuid primary key,
  job_name text not null,
  status text not null default 'running',
  result jsonb,
  error text,
  started_at timestamptz default now(),
  completed_at timestamptz
);
create index idx_cron_runs_job on cron_runs(job_name, completed_at desc);
```
