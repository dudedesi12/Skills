# Structured Logging Patterns

## Logger with Correlation ID

```typescript
// lib/logger.ts
type LogLevel = "debug" | "info" | "warn" | "error";

interface LogEntry {
  timestamp: string;
  level: LogLevel;
  message: string;
  requestId?: string;
  userId?: string;
  route?: string;
  duration?: number;
  error?: string;
  [key: string]: unknown;
}

class Logger {
  private context: Record<string, unknown> = {};

  withContext(ctx: Record<string, unknown>): Logger {
    const child = new Logger();
    child.context = { ...this.context, ...ctx };
    return child;
  }

  private log(level: LogLevel, message: string, meta?: Record<string, unknown>) {
    const entry: LogEntry = {
      timestamp: new Date().toISOString(),
      level,
      message,
      ...this.context,
      ...meta,
    };

    const output = JSON.stringify(entry);
    if (level === "error") console.error(output);
    else if (level === "warn") console.warn(output);
    else console.log(output);
  }

  debug(msg: string, meta?: Record<string, unknown>) { this.log("debug", msg, meta); }
  info(msg: string, meta?: Record<string, unknown>) { this.log("info", msg, meta); }
  warn(msg: string, meta?: Record<string, unknown>) { this.log("warn", msg, meta); }
  error(msg: string, meta?: Record<string, unknown>) { this.log("error", msg, meta); }
}

export const logger = new Logger();
```

### Usage in API Routes

```typescript
export async function GET(request: NextRequest) {
  const requestId = request.headers.get("x-request-id") ?? randomUUID();
  const log = logger.withContext({ requestId, route: "/api/users" });

  log.info("Request started");

  const start = Date.now();
  const { data, error } = await supabase.from("profiles").select("*");
  log.info("Database query completed", { duration: Date.now() - start });

  if (error) {
    log.error("Database error", { error: error.message });
    return NextResponse.json({ error: "Database error" }, { status: 500 });
  }

  log.info("Request completed", { duration: Date.now() - start, resultCount: data.length });
  return NextResponse.json(data);
}
```

## Log Drain to Axiom

Vercel can forward all logs to Axiom for search and analysis.

1. Create an Axiom account (free tier: 500MB/month)
2. In Vercel Dashboard → Settings → Log Drains
3. Add Axiom as a log drain with your Axiom ingest token
4. All `console.log`/`console.error` calls are automatically forwarded

## Query Timing Wrapper

```typescript
// lib/db/timed-query.ts
import { logger } from "@/lib/logger";

export async function timedQuery<T>(
  name: string,
  queryFn: () => Promise<{ data: T | null; error: { message: string } | null }>
): Promise<T | null> {
  const start = Date.now();
  const { data, error } = await queryFn();
  const duration = Date.now() - start;

  if (error) {
    logger.error(`Query failed: ${name}`, { duration, error: error.message });
    throw new Error(error.message);
  }

  if (duration > 1000) {
    logger.warn(`Slow query: ${name}`, { duration });
  } else {
    logger.debug(`Query: ${name}`, { duration });
  }

  return data;
}

// Usage:
const users = await timedQuery("get-users", () =>
  supabase.from("profiles").select("id, full_name").limit(20)
);
```
