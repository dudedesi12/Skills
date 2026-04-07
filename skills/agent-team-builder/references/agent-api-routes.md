# Agent API Routes — Next.js Templates

## Directory Structure

```
app/
  api/
    agents/
      [agentName]/
        route.ts          # Main POST handler (all agents)
        health/
          route.ts        # GET health check per agent
      list/
        route.ts          # GET list of available agents
      tasks/
        [taskId]/
          route.ts        # GET task status by ID
```

## Agent List Endpoint

```typescript
// app/api/agents/list/route.ts
import { NextResponse } from "next/server";

const agents = [
  { name: "scraper", description: "Extracts structured data from web pages", model: "gemini-2.0-flash" },
  { name: "parser", description: "Converts raw data to structured JSON", model: "gemini-2.0-flash" },
  { name: "content", description: "Generates SEO blog posts and copy", model: "gemini-2.5-pro" },
  { name: "analysis", description: "Calculations, predictions, scoring", model: "gemini-2.5-pro" },
  { name: "notification", description: "Email, push, in-app alert content", model: "gemini-2.0-flash" },
  { name: "moderation", description: "Spam/abuse/quality detection", model: "gemini-2.0-flash" },
  { name: "search", description: "Intelligent semantic search", model: "gemini-2.0-flash" },
  { name: "report", description: "PDF reports and summaries", model: "gemini-2.5-pro" },
  { name: "translation", description: "English/Hindi/Hinglish translation", model: "gemini-2.0-flash" },
  { name: "customer-support", description: "Knowledge base Q&A", model: "gemini-2.5-pro" },
];

export async function GET() {
  return NextResponse.json({ agents, count: agents.length });
}
```

## Task Status Endpoint

Look up the result of any agent task by its ID:

```typescript
// app/api/agents/tasks/[taskId]/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export async function GET(
  _request: NextRequest,
  { params }: { params: Promise<{ taskId: string }> }
) {
  try {
    const { taskId } = await params;

    const { data, error } = await supabase
      .from("agent_tasks")
      .select("*")
      .eq("id", taskId)
      .single();

    if (error || !data) {
      return NextResponse.json(
        { success: false, error: "Task not found" },
        { status: 404 }
      );
    }

    return NextResponse.json({
      success: true,
      task: {
        id: data.id,
        agentName: data.agent_name,
        version: data.agent_version,
        status: data.status,
        input: data.input,
        output: data.output,
        error: data.error,
        tokenUsage: data.token_usage,
        costUsd: data.cost_usd,
        createdAt: data.created_at,
        completedAt: data.completed_at,
      },
    });
  } catch (err) {
    const message = err instanceof Error ? err.message : "Failed to fetch task";
    return NextResponse.json({ success: false, error: message }, { status: 500 });
  }
}
```

## Rate Limiting Middleware

Apply rate limiting before the agent processes the request:

```typescript
// lib/agents/rate-limit-middleware.ts
import { NextRequest, NextResponse } from "next/server";

const requestCounts = new Map<string, { count: number; resetAt: number }>();

export function rateLimitMiddleware(
  request: NextRequest,
  agentName: string,
  maxPerMinute: number
): NextResponse | null {
  const clientIp = request.headers.get("x-forwarded-for") ?? "unknown";
  const key = `${agentName}:${clientIp}`;
  const now = Date.now();

  const entry = requestCounts.get(key);

  if (!entry || now > entry.resetAt) {
    requestCounts.set(key, { count: 1, resetAt: now + 60_000 });
    return null;
  }

  if (entry.count >= maxPerMinute) {
    return NextResponse.json(
      {
        success: false,
        error: "Rate limit exceeded. Try again shortly.",
        retryAfterMs: entry.resetAt - now,
      },
      {
        status: 429,
        headers: {
          "Retry-After": String(Math.ceil((entry.resetAt - now) / 1000)),
        },
      }
    );
  }

  entry.count++;
  return null;
}
```

## Authentication Middleware

Protect agent routes with Supabase auth:

```typescript
// lib/agents/auth-middleware.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export async function authMiddleware(
  request: NextRequest
): Promise<{ userId: string } | NextResponse> {
  const authHeader = request.headers.get("authorization");

  if (!authHeader || !authHeader.startsWith("Bearer ")) {
    return NextResponse.json(
      { success: false, error: "Missing authorization header" },
      { status: 401 }
    );
  }

  const token = authHeader.replace("Bearer ", "");

  const { data, error } = await supabase.auth.getUser(token);

  if (error || !data.user) {
    return NextResponse.json(
      { success: false, error: "Invalid or expired token" },
      { status: 401 }
    );
  }

  return { userId: data.user.id };
}
```

## Full Route with Middleware

Combining auth and rate limiting in the main route:

```typescript
// app/api/agents/[agentName]/route.ts (enhanced version)
import { NextRequest, NextResponse } from "next/server";
import { authMiddleware } from "@/lib/agents/auth-middleware";
import { rateLimitMiddleware } from "@/lib/agents/rate-limit-middleware";

// ... agent registry imports from SKILL.md ...

const rateLimits: Record<string, number> = {
  scraper: 30,
  parser: 60,
  content: 10,
  analysis: 15,
  notification: 100,
  moderation: 120,
  search: 60,
  report: 5,
  translation: 60,
  "customer-support": 30,
};

export async function POST(
  request: NextRequest,
  { params }: { params: Promise<{ agentName: string }> }
) {
  try {
    const { agentName } = await params;

    // Auth check
    const authResult = await authMiddleware(request);
    if (authResult instanceof NextResponse) return authResult;

    // Rate limit check
    const limit = rateLimits[agentName] ?? 30;
    const rateLimitResult = rateLimitMiddleware(request, agentName, limit);
    if (rateLimitResult) return rateLimitResult;

    // Agent lookup
    const createAgent = agentRegistry[agentName];
    if (!createAgent) {
      return NextResponse.json(
        { success: false, error: `Unknown agent: ${agentName}` },
        { status: 404 }
      );
    }

    const body = await request.json().catch(() => null);
    if (!body) {
      return NextResponse.json(
        { success: false, error: "Request body must be valid JSON" },
        { status: 400 }
      );
    }

    const version = request.headers.get("x-agent-version") ?? "v1";
    const agent = createAgent();
    const result = await agent.run(body, version);

    return NextResponse.json(result, {
      status: result.success ? 200 : 422,
    });
  } catch (err) {
    const message = err instanceof Error ? err.message : "Internal server error";
    return NextResponse.json(
      { success: false, error: message },
      { status: 500 }
    );
  }
}
```
