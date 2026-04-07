# Health Check Cron Implementation

## Vercel Cron Setup

```json
// file: vercel.json
{
  "crons": [
    { "path": "/api/cron/health-check", "schedule": "0 */6 * * *" },
    { "path": "/api/cron/dependency-audit", "schedule": "0 8 * * 1" }
  ]
}
```

## Extended Health Check Route

```typescript
// file: app/api/cron/health-check/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { runHealthCheck } from '@/lib/bug-detective/engine';
import { createClient } from '@/lib/supabase/server';

export const maxDuration = 60; // Allow up to 60s for full scan

export async function GET(request: NextRequest) {
  if (request.headers.get('authorization') !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const startTime = Date.now();

  try {
    const result = await runHealthCheck();
    const duration = Date.now() - startTime;

    // Update duration
    const supabase = await createClient();
    await supabase
      .from('health_checks')
      .update({ duration_ms: duration })
      .eq('overall_score', result.overallScore)
      .order('checked_at', { ascending: false })
      .limit(1);

    // Alert on critical score
    if (result.overallScore < 50) {
      await sendAlert('critical', `Health score dropped to ${result.overallScore}/100`, result);
    }

    return NextResponse.json({ success: true, duration, ...result });
  } catch (err) {
    const message = err instanceof Error ? err.message : 'Health check failed';
    return NextResponse.json({ error: message }, { status: 500 });
  }
}

async function sendAlert(
  severity: string,
  title: string,
  details: Record<string, unknown>
) {
  // Send via Resend (see email-transactional skill)
  await fetch(`${process.env.NEXT_PUBLIC_APP_URL}/api/email/alert`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${process.env.INTERNAL_API_KEY}`,
    },
    body: JSON.stringify({ severity, title, details }),
  });
}
```

## Dependency Audit Cron

```typescript
// file: app/api/cron/dependency-audit/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { exec } from 'child_process';
import { promisify } from 'util';
import { createClient } from '@/lib/supabase/server';

const execAsync = promisify(exec);

export async function GET(request: NextRequest) {
  if (request.headers.get('authorization') !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  try {
    const { stdout } = await execAsync('npm audit --json 2>/dev/null || true');
    const audit = JSON.parse(stdout);

    const vulnerabilities = audit.vulnerabilities ?? {};
    const critical = Object.values(vulnerabilities).filter((v: any) => v.severity === 'critical').length;
    const high = Object.values(vulnerabilities).filter((v: any) => v.severity === 'high').length;

    const supabase = await createClient();

    if (critical > 0 || high > 0) {
      for (const [name, vuln] of Object.entries(vulnerabilities) as [string, any][]) {
        if (vuln.severity === 'critical' || vuln.severity === 'high') {
          const fingerprint = `dep:${name}:${vuln.severity}`;
          await supabase.from('issues').upsert({
            category: 'dependencies',
            severity: vuln.severity,
            title: `Vulnerable package: ${name}`,
            description: `${vuln.severity} vulnerability in ${name}@${vuln.range}. ${vuln.via?.[0]?.title ?? ''}`,
            suggested_fix: vuln.fixAvailable ? `Run: npm audit fix` : 'Manual update required',
            fingerprint,
            last_seen: new Date().toISOString(),
          }, { onConflict: 'fingerprint' });
        }
      }
    }

    return NextResponse.json({ success: true, critical, high, total: Object.keys(vulnerabilities).length });
  } catch (err) {
    const message = err instanceof Error ? err.message : 'Audit failed';
    return NextResponse.json({ error: message }, { status: 500 });
  }
}
```

## Upsert Issue Database Function

```sql
-- file: supabase/migrations/002_upsert_issue.sql
create or replace function upsert_issue(
  p_health_check_id uuid,
  p_category text,
  p_severity severity,
  p_title text,
  p_description text,
  p_file_path text,
  p_suggested_fix text,
  p_fingerprint text
) returns void as $$
begin
  insert into issues (health_check_id, category, severity, title, description, file_path, suggested_fix, fingerprint)
  values (p_health_check_id, p_category, p_severity, p_title, p_description, p_file_path, p_suggested_fix, p_fingerprint)
  on conflict (fingerprint) do update set
    health_check_id = p_health_check_id,
    severity = p_severity,
    description = p_description,
    suggested_fix = p_suggested_fix,
    occurrences = issues.occurrences + 1,
    last_seen = now();
end;
$$ language plpgsql security definer;
```
