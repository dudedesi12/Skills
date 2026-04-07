# Log Parsing — Vercel Runtime Logs

## Accessing Vercel Logs

Vercel exposes runtime logs via their API. Use the Vercel CLI or REST API.

### Via API Route (Self-Monitoring)

```typescript
// file: lib/bug-detective/log-parser.ts

interface ParsedLog {
  level: 'error' | 'warn' | 'info';
  message: string;
  route: string;
  timestamp: string;
  statusCode?: number;
}

export function parseVercelLog(rawLog: string): ParsedLog | null {
  // Match common Next.js error patterns
  const patterns = [
    { regex: /Error: (.+?) at (.+)/, level: 'error' as const },
    { regex: /WARN: (.+)/, level: 'warn' as const },
    { regex: /(\d{3}) (GET|POST|PUT|DELETE) (.+?) (\d+)ms/, level: 'info' as const },
  ];

  for (const { regex, level } of patterns) {
    const match = rawLog.match(regex);
    if (match) {
      return {
        level,
        message: match[1],
        route: match[3] ?? 'unknown',
        timestamp: new Date().toISOString(),
        statusCode: match[1] ? parseInt(match[1]) : undefined,
      };
    }
  }
  return null;
}

// Common error patterns to watch for
export const ERROR_PATTERNS = [
  { pattern: /NEXT_NOT_FOUND/, category: 'runtime', title: '404 in server component' },
  { pattern: /DYNAMIC_SERVER_USAGE/, category: 'runtime', title: 'Dynamic server usage in static page' },
  { pattern: /Hydration failed/, category: 'runtime', title: 'Hydration mismatch' },
  { pattern: /ECONNREFUSED/, category: 'runtime', title: 'Database connection refused' },
  { pattern: /JWT expired/, category: 'security', title: 'Expired auth token not refreshed' },
  { pattern: /rate limit/, category: 'performance', title: 'Rate limit hit on external API' },
  { pattern: /ENOMEM/, category: 'performance', title: 'Out of memory in serverless function' },
  { pattern: /FUNCTION_INVOCATION_TIMEOUT/, category: 'performance', title: 'Serverless function timeout' },
];
```

### Client-Side Error Aggregation

```typescript
// file: lib/bug-detective/error-aggregator.ts
import { createClient } from '@/lib/supabase/server';

export async function aggregateErrors(hours = 24) {
  const supabase = await createClient();
  const since = new Date(Date.now() - hours * 3600000).toISOString();

  const { data: errors } = await supabase
    .from('client_errors')
    .select('message, url, count(*)')
    .gte('created_at', since)
    .order('count', { ascending: false })
    .limit(20);

  return (errors ?? []).map(e => ({
    message: e.message,
    url: e.url,
    count: e.count,
    fingerprint: `client:${e.message.slice(0, 100)}`,
  }));
}
```
