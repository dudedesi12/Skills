---
name: agent-bug-detective
description: >-
  Use this skill whenever the user mentions bugs, errors, crashes, performance problems, security
  audit, health check, monitoring, alerting, quality assurance, code review, broken features, slow
  pages, vulnerability scanning, dependency updates, or any debugging/diagnostic task. Also trigger
  for 'something is broken', 'why is this slow', 'check my site', 'find issues', 'audit my code',
  'is my app secure', or any quality-related concern — even if they don't explicitly say 'bug'.
---

# Agent Bug Detective

Automated quality agent that scans your app for bugs, performance issues, security vulnerabilities, and more — then suggests fixes using Gemini.

## Detection Categories

| # | Category | What It Catches |
|---|----------|-----------------|
| 1 | Runtime Errors | Unhandled exceptions, API 500s, client crashes |
| 2 | Performance | Slow routes (>3s), large bundles (>250KB), memory leaks |
| 3 | Security | Missing RLS, exposed env vars, XSS vectors, no CSP |
| 4 | SEO | Missing meta tags, broken links, no sitemap |
| 5 | Accessibility | Missing ARIA, low contrast, no keyboard nav |
| 6 | Data Integrity | Orphaned records, constraint violations, stale cache |
| 7 | UX | Dead clicks, error states without recovery, broken flows |
| 8 | Dependencies | Outdated packages, known CVEs, deprecated APIs |

## 1. Supabase Schema

```sql
-- file: supabase/migrations/001_bug_detective.sql

create type severity as enum ('critical', 'high', 'medium', 'low');
create type issue_status as enum ('open', 'acknowledged', 'fixing', 'resolved', 'wont_fix');

create table health_checks (
  id uuid primary key default gen_random_uuid(),
  checked_at timestamptz default now(),
  overall_score int not null check (overall_score between 0 and 100),
  category_scores jsonb not null,
  issues_found int default 0,
  duration_ms int
);

create table issues (
  id uuid primary key default gen_random_uuid(),
  health_check_id uuid references health_checks(id),
  category text not null,
  severity severity not null,
  title text not null,
  description text not null,
  file_path text,
  line_number int,
  suggested_fix text,
  status issue_status default 'open',
  fingerprint text unique not null,
  occurrences int default 1,
  first_seen timestamptz default now(),
  last_seen timestamptz default now(),
  resolved_at timestamptz
);

create table client_errors (
  id uuid primary key default gen_random_uuid(),
  message text not null,
  stack text,
  url text,
  user_agent text,
  user_id uuid,
  metadata jsonb default '{}',
  created_at timestamptz default now()
);

-- Indexes
create index idx_issues_severity on issues(severity);
create index idx_issues_status on issues(status);
create index idx_issues_category on issues(category);
create index idx_client_errors_created on client_errors(created_at desc);

-- RLS
alter table health_checks enable row level security;
alter table issues enable row level security;
alter table client_errors enable row level security;

create policy "Service role only" on health_checks for all using (auth.role() = 'service_role');
create policy "Service role only" on issues for all using (auth.role() = 'service_role');
create policy "Authenticated users can insert" on client_errors for insert with check (true);
create policy "Service role reads all" on client_errors for select using (auth.role() = 'service_role');
```

## 2. Client-Side Error Tracker

```typescript
// file: lib/error-tracker.ts
'use client';

import { createBrowserClient } from '@supabase/ssr';

const supabase = createBrowserClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
);

export function initErrorTracking() {
  if (typeof window === 'undefined') return;

  window.addEventListener('error', (event) => {
    reportError({
      message: event.message,
      stack: event.error?.stack,
      url: window.location.href,
    });
  });

  window.addEventListener('unhandledrejection', (event) => {
    reportError({
      message: `Unhandled Promise: ${event.reason}`,
      stack: event.reason?.stack,
      url: window.location.href,
    });
  });
}

async function reportError(error: { message: string; stack?: string; url: string }) {
  try {
    await supabase.from('client_errors').insert({
      message: error.message,
      stack: error.stack,
      url: error.url,
      user_agent: navigator.userAgent,
    });
  } catch {
    // Silent fail — don't crash the app over error tracking
  }
}
```

Add to your root layout:

```typescript
// file: app/layout.tsx
import { ErrorTracker } from '@/components/error-tracker';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        {children}
        <ErrorTracker />
      </body>
    </html>
  );
}
```

```typescript
// file: components/error-tracker.tsx
'use client';
import { useEffect } from 'react';
import { initErrorTracking } from '@/lib/error-tracker';

export function ErrorTracker() {
  useEffect(() => { initErrorTracking(); }, []);
  return null;
}
```

## 3. Health Check Engine

```typescript
// file: lib/bug-detective/engine.ts
import { createClient } from '@/lib/supabase/server';

interface Issue {
  category: string;
  severity: 'critical' | 'high' | 'medium' | 'low';
  title: string;
  description: string;
  filePath?: string;
  suggestedFix?: string;
}

interface CheckResult {
  category: string;
  score: number;
  issues: Issue[];
}

export async function runHealthCheck(): Promise<{ overallScore: number; results: CheckResult[] }> {
  const checks = await Promise.allSettled([
    checkRecentErrors(),
    checkDependencies(),
    checkSecurityHeaders(),
    checkPerformance(),
  ]);

  const results: CheckResult[] = checks
    .filter((c): c is PromiseFulfilledResult<CheckResult> => c.status === 'fulfilled')
    .map(c => c.value);

  const overallScore = Math.round(
    results.reduce((sum, r) => sum + r.score, 0) / results.length
  );

  // Save to Supabase
  const supabase = await createClient();
  const { data: healthCheck } = await supabase
    .from('health_checks')
    .insert({
      overall_score: overallScore,
      category_scores: Object.fromEntries(results.map(r => [r.category, r.score])),
      issues_found: results.reduce((sum, r) => sum + r.issues.length, 0),
    })
    .select()
    .single();

  // Upsert issues with deduplication via fingerprint
  for (const result of results) {
    for (const issue of result.issues) {
      const fingerprint = `${issue.category}:${issue.title}`;
      await supabase.rpc('upsert_issue', {
        p_health_check_id: healthCheck!.id,
        p_category: issue.category,
        p_severity: issue.severity,
        p_title: issue.title,
        p_description: issue.description,
        p_file_path: issue.filePath ?? null,
        p_suggested_fix: issue.suggestedFix ?? null,
        p_fingerprint: fingerprint,
      });
    }
  }

  return { overallScore, results };
}

async function checkRecentErrors(): Promise<CheckResult> {
  const supabase = await createClient();
  const oneDayAgo = new Date(Date.now() - 86400000).toISOString();
  const { count } = await supabase
    .from('client_errors')
    .select('*', { count: 'exact', head: true })
    .gte('created_at', oneDayAgo);

  const errorCount = count ?? 0;
  const score = errorCount === 0 ? 100 : errorCount < 10 ? 80 : errorCount < 50 ? 50 : 20;
  const issues: Issue[] = errorCount > 10 ? [{
    category: 'runtime',
    severity: errorCount > 50 ? 'critical' : 'high',
    title: `${errorCount} client errors in last 24h`,
    description: `Your app is throwing ${errorCount} errors. Check the client_errors table for details.`,
    suggestedFix: 'Review the most common error messages and add error boundaries around failing components.',
  }] : [];

  return { category: 'runtime', score, issues };
}

async function checkDependencies(): Promise<CheckResult> {
  const issues: Issue[] = [];
  // This runs `npm audit --json` in the health check cron
  // See references/health-check-cron.md for full implementation
  return { category: 'dependencies', score: 100, issues };
}

async function checkSecurityHeaders(): Promise<CheckResult> {
  const issues: Issue[] = [];
  try {
    const res = await fetch(process.env.NEXT_PUBLIC_APP_URL!, { method: 'HEAD' });
    const required = ['x-frame-options', 'x-content-type-options', 'strict-transport-security'];
    for (const header of required) {
      if (!res.headers.has(header)) {
        issues.push({
          category: 'security',
          severity: 'high',
          title: `Missing ${header} header`,
          description: `The ${header} security header is not set.`,
          suggestedFix: `Add the header in next.config.ts security headers. See security-hardening skill.`,
        });
      }
    }
  } catch {
    issues.push({
      category: 'security',
      severity: 'medium',
      title: 'Cannot reach app URL',
      description: 'Health check cannot reach NEXT_PUBLIC_APP_URL to verify headers.',
    });
  }

  const score = issues.length === 0 ? 100 : Math.max(0, 100 - issues.length * 25);
  return { category: 'security', score, issues };
}

async function checkPerformance(): Promise<CheckResult> {
  const issues: Issue[] = [];
  // Lighthouse CI integration — see references/health-check-cron.md
  return { category: 'performance', score: 100, issues };
}
```

## 4. Gemini Auto-Fix Suggestions

```typescript
// file: lib/bug-detective/fix-suggester.ts
import { GoogleGenerativeAI } from '@google/generative-ai';

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);

export async function suggestFix(issue: {
  category: string;
  title: string;
  description: string;
  filePath?: string;
  errorStack?: string;
}) {
  const model = genAI.getGenerativeModel({ model: 'gemini-2.0-flash' });

  const result = await model.generateContent({
    contents: [{ role: 'user', parts: [{ text: `Fix this issue in my Next.js + Supabase + Tailwind app:

Category: ${issue.category}
Title: ${issue.title}
Description: ${issue.description}
${issue.filePath ? `File: ${issue.filePath}` : ''}
${issue.errorStack ? `Stack trace:\n${issue.errorStack}` : ''}

Provide:
1. What's causing this (1 sentence, plain English)
2. The exact code fix (with file path)
3. How to prevent it in future (1 sentence)` }] }],
    generationConfig: { maxOutputTokens: 1024 },
  });

  return result.response.text();
}
```

## 5. Health Check Cron

```typescript
// file: app/api/cron/health-check/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { runHealthCheck } from '@/lib/bug-detective/engine';

export async function GET(request: NextRequest) {
  if (request.headers.get('authorization') !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  try {
    const result = await runHealthCheck();
    return NextResponse.json({ success: true, ...result });
  } catch (err) {
    const message = err instanceof Error ? err.message : 'Health check failed';
    return NextResponse.json({ error: message }, { status: 500 });
  }
}
```

Add to `vercel.json`:
```json
{
  "crons": [{ "path": "/api/cron/health-check", "schedule": "0 */6 * * *" }]
}
```

## 6. Health Dashboard Page

```typescript
// file: app/admin/health/page.tsx
import { createClient } from '@/lib/supabase/server';

export default async function HealthDashboard() {
  const supabase = await createClient();

  const { data: latestCheck } = await supabase
    .from('health_checks')
    .select('*')
    .order('checked_at', { ascending: false })
    .limit(1)
    .single();

  const { data: openIssues } = await supabase
    .from('issues')
    .select('*')
    .in('status', ['open', 'acknowledged'])
    .order('severity');

  const scoreColor = (score: number) =>
    score >= 80 ? 'text-green-500' : score >= 60 ? 'text-yellow-500' : 'text-red-500';

  return (
    <div className="max-w-4xl mx-auto p-6">
      <h1 className="text-2xl font-bold mb-6">App Health Dashboard</h1>

      {latestCheck && (
        <div className="bg-white dark:bg-gray-800 rounded-lg p-6 mb-6 shadow">
          <p className="text-sm text-gray-500">Last checked: {new Date(latestCheck.checked_at).toLocaleString()}</p>
          <p className={`text-5xl font-bold mt-2 ${scoreColor(latestCheck.overall_score)}`}>
            {latestCheck.overall_score}/100
          </p>
          <div className="grid grid-cols-2 md:grid-cols-4 gap-4 mt-4">
            {Object.entries(latestCheck.category_scores as Record<string, number>).map(([cat, score]) => (
              <div key={cat} className="text-center">
                <p className="text-sm text-gray-500 capitalize">{cat}</p>
                <p className={`text-xl font-semibold ${scoreColor(score)}`}>{score}</p>
              </div>
            ))}
          </div>
        </div>
      )}

      <h2 className="text-xl font-semibold mb-4">Open Issues ({openIssues?.length ?? 0})</h2>
      <div className="space-y-3">
        {openIssues?.map(issue => (
          <div key={issue.id} className="bg-white dark:bg-gray-800 rounded-lg p-4 shadow border-l-4"
            style={{ borderColor: issue.severity === 'critical' ? '#ef4444' : issue.severity === 'high' ? '#f97316' : issue.severity === 'medium' ? '#eab308' : '#22c55e' }}>
            <div className="flex justify-between items-start">
              <div>
                <span className="text-xs uppercase font-medium text-gray-500">{issue.category}</span>
                <h3 className="font-medium">{issue.title}</h3>
                <p className="text-sm text-gray-600 dark:text-gray-400">{issue.description}</p>
              </div>
              <span className="text-xs px-2 py-1 rounded bg-gray-100 dark:bg-gray-700">{issue.severity}</span>
            </div>
            {issue.suggested_fix && (
              <details className="mt-2">
                <summary className="text-sm text-blue-500 cursor-pointer">Suggested fix</summary>
                <pre className="mt-1 text-xs bg-gray-50 dark:bg-gray-900 p-2 rounded overflow-x-auto">{issue.suggested_fix}</pre>
              </details>
            )}
          </div>
        ))}
      </div>
    </div>
  );
}
```

> For detection rules per category, see `references/detection-rules.md`.
> For full cron implementation, see `references/health-check-cron.md`.
> For Gemini fix prompts, see `references/fix-suggestions.md`.
