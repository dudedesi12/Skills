---
name: agent-team-builder
description: "Use this skill whenever the user wants to create an AI agent, build a specialist bot, set up an AI-powered feature, create an automated worker, or needs any individual agent for scraping, parsing, content generation, analysis, notifications, moderation, search, reports, translation, or customer support. Also trigger for 'build a bot', 'automate this with AI', 'I need an agent that…', 'AI worker', or any request to create an autonomous AI component."
---

# Agent Team Builder

Build production-ready AI agents for your SaaS using Next.js, Supabase, and the Gemini API. This skill walks you through creating 10 specialized agent types — from web scrapers to customer support bots — with proper error handling, rate limiting, cost budgets, and task queuing.

**Stack:** Next.js App Router, Supabase, Vercel, Tailwind CSS, TypeScript, Gemini API

## Quick Start: Install Dependencies

```bash
npm install @google/generative-ai zod uuid
npm install -D @types/uuid
```

Set your environment variable in `.env.local`:

```env
# .env.local
GEMINI_API_KEY=your_gemini_api_key_here
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key
```

## Database Setup: Supabase Tables

Run this SQL in your Supabase SQL Editor to create the task queue and results storage:

```sql
-- Agent task queue
CREATE TABLE agent_tasks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_name TEXT NOT NULL,
  agent_version TEXT NOT NULL DEFAULT 'v1',
  status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'processing', 'completed', 'failed')),
  input JSONB NOT NULL,
  output JSONB,
  error TEXT,
  token_usage INTEGER DEFAULT 0,
  cost_usd NUMERIC(10, 6) DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT now(),
  completed_at TIMESTAMPTZ,
  retry_count INTEGER DEFAULT 0,
  max_retries INTEGER DEFAULT 3
);

-- Agent health and usage logs
CREATE TABLE agent_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_name TEXT NOT NULL,
  level TEXT NOT NULL CHECK (level IN ('info', 'warn', 'error')),
  message TEXT NOT NULL,
  metadata JSONB,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Cost budget tracking per agent
CREATE TABLE agent_budgets (
  agent_name TEXT PRIMARY KEY,
  daily_limit_usd NUMERIC(10, 4) NOT NULL DEFAULT 5.0,
  monthly_limit_usd NUMERIC(10, 4) NOT NULL DEFAULT 100.0,
  total_spent_today_usd NUMERIC(10, 6) DEFAULT 0,
  total_spent_month_usd NUMERIC(10, 6) DEFAULT 0,
  last_reset_daily TIMESTAMPTZ DEFAULT now(),
  last_reset_monthly TIMESTAMPTZ DEFAULT now()
);

-- Indexes for performance
CREATE INDEX idx_agent_tasks_status ON agent_tasks(status);
CREATE INDEX idx_agent_tasks_agent ON agent_tasks(agent_name);
CREATE INDEX idx_agent_logs_agent ON agent_logs(agent_name);
```

## The 10 Agent Types

| # | Agent | What It Does | Gemini Model |
|---|-------|-------------|--------------|
| 1 | **Scraper** | Extracts data from websites | gemini-2.0-flash |
| 2 | **Parser** | Converts raw data to structured JSON | gemini-2.0-flash |
| 3 | **Content** | Generates SEO blog posts, copy | gemini-2.5-pro |
| 4 | **Analysis** | Calculations, predictions, scoring | gemini-2.5-pro |
| 5 | **Notification** | Sends email, push, in-app alerts | gemini-2.0-flash |
| 6 | **Moderation** | Detects spam, abuse, quality issues | gemini-2.0-flash |
| 7 | **Search** | Intelligent search with grounded results | gemini-2.0-flash |
| 8 | **Report** | PDF reports, dashboards, summaries | gemini-2.5-pro |
| 9 | **Translation** | English to Hindi (Hinglish audience) | gemini-2.0-flash |
| 10 | **Customer Support** | Answers from your knowledge base | gemini-2.5-pro |

See `references/agent-configs.md` for full config templates for each agent.

## Shared Agent Base Utility

Every agent shares this base class. It handles Gemini calls, retries, logging, rate limiting, and cost tracking.

```typescript
// lib/agents/base-agent.ts
import { GoogleGenerativeAI, GenerativeModel } from "@google/generative-ai";
import { z, ZodSchema } from "zod";
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);

export interface AgentConfig {
  name: string;
  description: string;
  capabilities: string[];
  modelId: string;
  systemPrompt: string;
  inputSchema: ZodSchema;
  maxRetries: number;
  rateLimitPerMinute: number;
  costPerMillionTokens: number;
}

interface AgentResult<T> {
  success: boolean;
  data: T | null;
  error: string | null;
  tokenUsage: number;
  costUsd: number;
  taskId: string;
}

const rateLimitMap = new Map<string, { count: number; resetAt: number }>();

export class BaseAgent<TInput, TOutput> {
  protected config: AgentConfig;
  protected model: GenerativeModel;

  constructor(config: AgentConfig) {
    this.config = config;
    this.model = genAI.getGenerativeModel({
      model: config.modelId,
      systemInstruction: config.systemPrompt,
    });
  }

  private checkRateLimit(): boolean {
    const now = Date.now();
    const entry = rateLimitMap.get(this.config.name);

    if (!entry || now > entry.resetAt) {
      rateLimitMap.set(this.config.name, {
        count: 1,
        resetAt: now + 60_000,
      });
      return true;
    }

    if (entry.count >= this.config.rateLimitPerMinute) {
      return false;
    }

    entry.count++;
    return true;
  }

  private async checkBudget(estimatedCost: number): Promise<boolean> {
    const { data } = await supabase.from("agent_budgets").select("*").eq("agent_name", this.config.name).single();
    if (!data) return true;
    const now = new Date();
    let todaySpent = now.toDateString() !== new Date(data.last_reset_daily).toDateString() ? 0 : data.total_spent_today_usd;
    let monthSpent = now.getMonth() !== new Date(data.last_reset_monthly).getMonth() ? 0 : data.total_spent_month_usd;
    if (todaySpent + estimatedCost > data.daily_limit_usd) return false;
    if (monthSpent + estimatedCost > data.monthly_limit_usd) return false;
    return true;
  }

  private async log(level: "info" | "warn" | "error", message: string, metadata?: Record<string, unknown>): Promise<void> {
    try {
      await supabase.from("agent_logs").insert({ agent_name: this.config.name, level, message, metadata: metadata ?? null });
    } catch { console.error(`[${this.config.name}] Log failed: ${message}`); }
  }

  private async updateBudget(cost: number): Promise<void> {
    const { data } = await supabase.from("agent_budgets").select("total_spent_today_usd, total_spent_month_usd").eq("agent_name", this.config.name).single();
    if (data) {
      await supabase.from("agent_budgets").update({
        total_spent_today_usd: data.total_spent_today_usd + cost,
        total_spent_month_usd: data.total_spent_month_usd + cost,
      }).eq("agent_name", this.config.name);
    }
  }

  async run(
    rawInput: unknown,
    version: string = "v1"
  ): Promise<AgentResult<TOutput>> {
    const taskId = crypto.randomUUID();

    const parseResult = this.config.inputSchema.safeParse(rawInput);
    if (!parseResult.success) {
      await this.log("error", "Input validation failed", { errors: parseResult.error.flatten() });
      return { success: false, data: null, error: `Invalid input: ${parseResult.error.issues.map((i) => i.message).join(", ")}`, tokenUsage: 0, costUsd: 0, taskId };
    }

    if (!this.checkRateLimit()) {
      await this.log("warn", "Rate limit exceeded");
      return { success: false, data: null, error: "Rate limit exceeded. Try again in a moment.", tokenUsage: 0, costUsd: 0, taskId };
    }

    if (!(await this.checkBudget(0.001))) {
      await this.log("warn", "Budget limit reached");
      return { success: false, data: null, error: "Agent budget limit reached for this period.", tokenUsage: 0, costUsd: 0, taskId };
    }

    await supabase.from("agent_tasks").insert({ id: taskId, agent_name: this.config.name, agent_version: version, status: "processing", input: parseResult.data });

    // Execute with retries
    let lastError: string = "";
    for (let attempt = 0; attempt <= this.config.maxRetries; attempt++) {
      try {
        await this.log("info", `Attempt ${attempt + 1}`, { version });

        const prompt = this.buildPrompt(parseResult.data as TInput);
        const result = await this.model.generateContent(prompt);
        const response = result.response;
        const text = response.text();

        const tokenUsage = response.usageMetadata?.totalTokenCount ?? 0;
        const costUsd = (tokenUsage / 1_000_000) * this.config.costPerMillionTokens;
        const output = this.parseOutput(text);

        await supabase.from("agent_tasks").update({ status: "completed", output, token_usage: tokenUsage, cost_usd: costUsd, completed_at: new Date().toISOString() }).eq("id", taskId);
        await this.updateBudget(costUsd);
        await this.log("info", "Task completed", { tokenUsage, costUsd });

        return { success: true, data: output as TOutput, error: null, tokenUsage, costUsd, taskId };
      } catch (err) {
        lastError = err instanceof Error ? err.message : String(err);
        await this.log("error", `Attempt ${attempt + 1} failed: ${lastError}`);

        if (attempt < this.config.maxRetries) {
          await new Promise((r) => setTimeout(r, 1000 * (attempt + 1)));
        }
      }
    }

    await supabase.from("agent_tasks").update({ status: "failed", error: lastError, retry_count: this.config.maxRetries }).eq("id", taskId);
    return { success: false, data: null, error: `Agent failed after ${this.config.maxRetries + 1} attempts: ${lastError}`, tokenUsage: 0, costUsd: 0, taskId };
  }

  protected buildPrompt(input: TInput): string {
    return JSON.stringify(input);
  }

  protected parseOutput(text: string): unknown {
    try {
      const jsonMatch = text.match(/```json\s*([\s\S]*?)```/);
      if (jsonMatch) return JSON.parse(jsonMatch[1].trim());
      return JSON.parse(text);
    } catch {
      return { rawText: text };
    }
  }
}
```

## Creating an Agent: Scraper Example

Every agent extends `BaseAgent` with three things: a Zod input schema, a `buildPrompt` method, and a `parseOutput` method. Here is the Scraper agent — all other agents follow this same pattern (see `references/agent-configs.md` for all 10 configs):

```typescript
// lib/agents/scraper-agent.ts
import { z } from "zod";
import { BaseAgent, AgentConfig } from "./base-agent";

const ScraperInputSchema = z.object({
  url: z.string().url("Must be a valid URL"),
  selectors: z.array(z.string()).min(1, "At least one selector required"),
  format: z.enum(["json", "csv", "markdown"]).default("json"),
});

type ScraperInput = z.infer<typeof ScraperInputSchema>;

interface ScraperOutput {
  url: string;
  extractedData: Record<string, string | string[]>[];
  fieldCount: number;
  scrapedAt: string;
}

export class ScraperAgent extends BaseAgent<ScraperInput, ScraperOutput> {
  constructor() {
    super({
      name: "scraper",
      description: "Extracts structured data from web pages",
      capabilities: ["html-parsing", "data-extraction", "css-selectors"],
      modelId: "gemini-2.0-flash",
      systemPrompt: `You are a web scraping assistant. Given HTML content and target selectors/fields, extract the requested data into clean structured JSON. Always return valid JSON with an "extractedData" array. If a field cannot be found, set its value to null. Never fabricate data.`,
      inputSchema: ScraperInputSchema,
      maxRetries: 3,
      rateLimitPerMinute: 30,
      costPerMillionTokens: 0.10,
    });
  }

  protected buildPrompt(input: ScraperInput): string {
    return `Extract these fields from ${input.url}: ${input.selectors.join(", ")}
Format: ${input.format}
Return JSON: { "url": "...", "extractedData": [...], "fieldCount": N, "scrapedAt": "ISO" }`;
  }

  protected parseOutput(text: string): ScraperOutput {
    try {
      const jsonMatch = text.match(/```json\s*([\s\S]*?)```/);
      const parsed = JSON.parse(jsonMatch ? jsonMatch[1].trim() : text);
      return {
        url: parsed.url ?? "",
        extractedData: parsed.extractedData ?? [],
        fieldCount: parsed.fieldCount ?? 0,
        scrapedAt: parsed.scrapedAt ?? new Date().toISOString(),
      };
    } catch {
      return { url: "", extractedData: [], fieldCount: 0, scrapedAt: new Date().toISOString() };
    }
  }
}
```

## Dynamic API Route for All Agents

A single dynamic route handles every agent. The `[agentName]` param selects the right agent class. See `references/agent-api-routes.md` for the full version with auth and rate-limit middleware.

```typescript
// app/api/agents/[agentName]/route.ts
import { NextRequest, NextResponse } from "next/server";
import { ScraperAgent } from "@/lib/agents/scraper-agent";
// Import all other agents the same way...
import { BaseAgent } from "@/lib/agents/base-agent";

const agentRegistry: Record<string, () => BaseAgent<unknown, unknown>> = {
  scraper: () => new ScraperAgent() as BaseAgent<unknown, unknown>,
  // parser, content, analysis, notification, moderation,
  // search, report, translation, customer-support — same pattern
};

export async function POST(
  request: NextRequest,
  { params }: { params: Promise<{ agentName: string }> }
) {
  try {
    const { agentName } = await params;
    const version = request.headers.get("x-agent-version") ?? "v1";

    const createAgent = agentRegistry[agentName];
    if (!createAgent) {
      return NextResponse.json(
        { success: false, error: `Unknown agent: ${agentName}. Available: ${Object.keys(agentRegistry).join(", ")}` },
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

    const agent = createAgent();
    const result = await agent.run(body, version);
    return NextResponse.json(result, { status: result.success ? 200 : 422 });
  } catch (err) {
    const message = err instanceof Error ? err.message : "Internal server error";
    return NextResponse.json({ success: false, error: message }, { status: 500 });
  }
}
```

## Agent Health Check Endpoint

Each agent gets a health check at `GET /api/agents/[agentName]/health`. See `references/agent-api-routes.md` for the full implementation. It returns:

```json
{
  "agent": "scraper",
  "status": "healthy",
  "last24h": { "totalTasks": 142, "totalErrors": 3, "errorRate": "2.1%" },
  "budget": { "dailySpent": 0.34, "dailyLimit": 1.0, "monthlySpent": 8.12, "monthlyLimit": 20.0 }
}
```

## Agent Versioning with A/B Routing

Support running v1 and v2 of an agent simultaneously with weighted traffic routing:

```typescript
// lib/agents/version-router.ts
const versionConfigs: Record<string, { v1Weight: number }> = {
  content: { v1Weight: 80 },   // 80% v1, 20% v2
  analysis: { v1Weight: 50 },  // 50/50 split
};

export function selectVersion(agentName: string): string {
  const config = versionConfigs[agentName];
  if (!config) return "v1";
  return Math.random() * 100 < config.v1Weight ? "v1" : "v2";
}
```

In the API route, replace the version line: `const version = request.headers.get("x-agent-version") ?? selectVersion(agentName);`

## Calling Agents from Your Frontend

```typescript
// lib/agents/client.ts
export async function callAgent<T>(
  agentName: string,
  input: Record<string, unknown>,
  version?: string
): Promise<{ success: boolean; data: T | null; error: string | null; tokenUsage: number; costUsd: number; taskId: string }> {
  try {
    const headers: Record<string, string> = { "Content-Type": "application/json" };
    if (version) headers["x-agent-version"] = version;

    const response = await fetch(`/api/agents/${agentName}`, {
      method: "POST",
      headers,
      body: JSON.stringify(input),
    });
    return await response.json();
  } catch (err) {
    return { success: false, data: null, error: err instanceof Error ? err.message : "Network error", tokenUsage: 0, costUsd: 0, taskId: "" };
  }
}
```

Usage example:

```typescript
const result = await callAgent("scraper", {
  url: "https://example.com/products",
  selectors: ["title", "price"],
  format: "json",
});
if (result.success) console.log(result.data);
else console.error(result.error);
```

## Reference Files

For detailed configs, prompts, and implementation guides for all 10 agents:

- **`references/agent-configs.md`** — Full config objects for every agent type
- **`references/system-prompts.md`** — Optimized system prompts for each agent
- **`references/agent-api-routes.md`** — Complete API route patterns and middleware
- **`references/testing-agents.md`** — How to test each agent type
- **`references/cost-budgets.md`** — Token usage estimates and monthly budgets
