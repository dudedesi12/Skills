---
name: "Scraper Architect"
description: "Use this skill whenever the user mentions scraping, web scraping, data extraction, crawling, Puppeteer, Playwright scraping, government data, 'get data from website', PDF parsing, data collection, anti-detection, headless browser, automated browsing, content extraction, data pipeline, 'pull data from', immigration data, regulatory data, 'check for updates', robots.txt, or ANY web data collection task — even if they don't explicitly say 'scrape'. This skill builds production-grade data collection pipelines."
---

# Scraper Architect

Build production-grade web scraping pipelines that collect, clean, and store data reliably. This skill covers everything from basic page scraping to multi-agent architectures powered by Gemini for intelligent data extraction.

## Puppeteer/Playwright Scraping Patterns

Puppeteer and Playwright both control a headless browser (a browser with no visible window). Playwright is recommended because it supports multiple browsers and has better auto-waiting.

### Install Playwright for Scraping

```bash
npm install playwright
npx playwright install chromium
```

### Basic Page Scraping

```typescript
// lib/scraper/basic.ts
import { chromium, type Browser, type Page } from "playwright";

export async function scrapePage(url: string): Promise<string> {
  let browser: Browser | null = null;

  try {
    browser = await chromium.launch({ headless: true });
    const context = await browser.newContext({
      userAgent:
        "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
    });

    const page = await context.newPage();

    // Block images, fonts, and CSS to speed things up
    await page.route("**/*.{png,jpg,jpeg,gif,svg,woff,woff2,css}", (route) =>
      route.abort()
    );

    const response = await page.goto(url, {
      waitUntil: "domcontentloaded",
      timeout: 30000,
    });

    if (!response || response.status() >= 400) {
      throw new Error(`Failed to load ${url}: HTTP ${response?.status()}`);
    }

    await page.waitForSelector("body", { timeout: 10000 });

    const html = await page.content();
    return html;
  } catch (err) {
    console.error(`Scrape failed for ${url}:`, err);
    throw err;
  } finally {
    if (browser) {
      await browser.close();
    }
  }
}
```

For extracting structured data, use `page.evaluate()` with CSS selectors, or feed the HTML to Gemini for intelligent extraction (see Gemini-Powered Extraction below). See `references/scraper-patterns.md` for pagination, infinite scroll, form submission, and file download patterns.

## Anti-Detection Techniques

Websites try to block scrapers. These techniques help your scraper look like a real user.

### User Agent Rotation

```typescript
// lib/scraper/user-agents.ts
const USER_AGENTS = [
  "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
  "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
  "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:121.0) Gecko/20100101 Firefox/121.0",
  "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.2 Safari/605.1.15",
];

export function getRandomUserAgent(): string {
  return USER_AGENTS[Math.floor(Math.random() * USER_AGENTS.length)];
}
```

### Random Delays Between Requests

```typescript
// lib/scraper/delays.ts
export function randomDelay(minMs: number, maxMs: number): Promise<void> {
  const delay = Math.floor(Math.random() * (maxMs - minMs + 1)) + minMs;
  return new Promise((resolve) => setTimeout(resolve, delay));
}

// Usage: wait 2-5 seconds between pages
await randomDelay(2000, 5000);
```

### Stealth Browser Context

```typescript
// lib/scraper/stealth.ts
import { chromium } from "playwright";
import { getRandomUserAgent } from "./user-agents";

export async function createStealthBrowser() {
  const browser = await chromium.launch({ headless: true });
  const context = await browser.newContext({
    userAgent: getRandomUserAgent(),
    viewport: { width: 1920, height: 1080 },
    locale: "en-AU",
    timezoneId: "Australia/Sydney",
  });
  // Remove the "webdriver" flag that identifies automated browsers
  await context.addInitScript(() => {
    Object.defineProperty(navigator, "webdriver", { get: () => false });
  });
  return { browser, context };
}
```

## Data Normalization and Cleaning

```typescript
// lib/scraper/normalize.ts
export function cleanText(raw: string): string {
  return raw.replace(/\s+/g, " ").replace(/[\n\r\t]/g, " ").trim();
}

export function extractPrice(raw: string): number | null {
  const match = raw.match(/[\d,]+\.?\d*/);
  if (!match) return null;
  return parseFloat(match[0].replace(/,/g, ""));
}

export function normalizeUrl(url: string, baseUrl: string): string {
  try { return new URL(url, baseUrl).href; } catch { return url; }
}
```

## Scheduled Scraping with Vercel Cron

Vercel Cron lets you run scraping jobs on a schedule. Create an API route and configure it as a cron job:

```typescript
// app/api/cron/scrape/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@supabase/supabase-js";
import { scrapePage } from "@/lib/scraper/basic";
import { extractWithGemini } from "@/lib/scraper/gemini-extract";

function verifyCronSecret(request: NextRequest): boolean {
  const authHeader = request.headers.get("authorization");
  return authHeader === `Bearer ${process.env.CRON_SECRET}`;
}

export async function GET(request: NextRequest) {
  if (!verifyCronSecret(request)) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const supabase = createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!
  );

  const results = { scraped: 0, failed: 0, unchanged: 0 };

  try {
    const { data: sources, error } = await supabase
      .from("scrape_sources").select("*").eq("active", true);
    if (error) throw new Error(`DB error: ${error.message}`);

    for (const source of sources || []) {
      try {
        const html = await scrapePage(source.url);
        const contentHash = await hashContent(html);

        // Skip if content unchanged
        if (contentHash === source.last_content_hash) {
          results.unchanged++;
          await supabase.from("scrape_sources")
            .update({ last_checked_at: new Date().toISOString() }).eq("id", source.id);
          continue;
        }

        const extracted = await extractWithGemini(html, source.extraction_prompt);
        await supabase.from("scraped_data").insert({
          source_id: source.id, data: extracted,
          content_hash: contentHash, scraped_at: new Date().toISOString(),
        });
        await supabase.from("scrape_sources").update({
          last_content_hash: contentHash, last_scraped_at: new Date().toISOString(),
          last_checked_at: new Date().toISOString(), consecutive_failures: 0,
        }).eq("id", source.id);
        results.scraped++;
      } catch (err) {
        results.failed++;
        await supabase.from("scrape_sources").update({
          consecutive_failures: (source.consecutive_failures || 0) + 1,
          last_error: err instanceof Error ? err.message : String(err),
          last_checked_at: new Date().toISOString(),
        }).eq("id", source.id);
      }
    }

    return NextResponse.json({ message: "Scrape complete", results });
  } catch (err) {
    console.error("Cron scrape error:", err);
    return NextResponse.json({ error: "Scrape failed" }, { status: 500 });
  }
}

async function hashContent(content: string): Promise<string> {
  const encoder = new TextEncoder();
  const data = encoder.encode(content);
  const hashBuffer = await crypto.subtle.digest("SHA-256", data);
  const hashArray = Array.from(new Uint8Array(hashBuffer));
  return hashArray.map((b) => b.toString(16).padStart(2, "0")).join("");
}
```

Configure the cron schedule in `vercel.json`:

```json
// vercel.json
{
  "crons": [
    {
      "path": "/api/cron/scrape",
      "schedule": "0 */6 * * *"
    }
  ]
}
```

Common schedules: `0 * * * *` (every hour), `0 */6 * * *` (every 6 hours), `0 0 * * *` (daily at midnight), `0 9 * * 1` (Mondays at 9am).

## Freshness Tracking

Track when data was last scraped and whether it has changed:

```sql
-- Supabase SQL: Create the scrape tracking tables
CREATE TABLE scrape_sources (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  url TEXT NOT NULL,
  name TEXT NOT NULL,
  extraction_prompt TEXT,
  active BOOLEAN DEFAULT true,
  last_scraped_at TIMESTAMPTZ,
  last_checked_at TIMESTAMPTZ,
  last_content_hash TEXT,
  last_error TEXT,
  consecutive_failures INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE scraped_data (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  source_id UUID REFERENCES scrape_sources(id),
  data JSONB NOT NULL,
  content_hash TEXT NOT NULL,
  scraped_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_scraped_data_source ON scraped_data(source_id, scraped_at DESC);
```

## robots.txt Compliance and Ethical Scraping

Always check `robots.txt` before scraping. This file tells you what pages the website owner allows bots to access:

```typescript
// lib/scraper/robots.ts
export async function checkRobotsTxt(
  url: string
): Promise<{ allowed: boolean; crawlDelay: number }> {
  try {
    const parsedUrl = new URL(url);
    const robotsUrl = `${parsedUrl.origin}/robots.txt`;

    const response = await fetch(robotsUrl, { signal: AbortSignal.timeout(10000) });

    if (!response.ok) {
      return { allowed: true, crawlDelay: 1 };
    }

    const text = await response.text();
    const lines = text.split("\n");

    let inUserAgentBlock = false;
    let crawlDelay = 1;
    const disallowedPaths: string[] = [];

    for (const line of lines) {
      const trimmed = line.trim().toLowerCase();

      if (trimmed.startsWith("user-agent: *") || trimmed.startsWith("user-agent:*")) {
        inUserAgentBlock = true;
        continue;
      }

      if (trimmed.startsWith("user-agent:") && inUserAgentBlock) {
        break;
      }

      if (inUserAgentBlock) {
        if (trimmed.startsWith("disallow:")) {
          const path = trimmed.replace("disallow:", "").trim();
          if (path) disallowedPaths.push(path);
        }
        if (trimmed.startsWith("crawl-delay:")) {
          crawlDelay = parseInt(trimmed.replace("crawl-delay:", "").trim()) || 1;
        }
      }
    }

    const pathname = parsedUrl.pathname;
    const allowed = !disallowedPaths.some((dp) => pathname.startsWith(dp));

    return { allowed, crawlDelay };
  } catch (err) {
    console.error("robots.txt check failed:", err);
    return { allowed: true, crawlDelay: 1 };
  }
}
```

**Ethical scraping rules:** Always check `robots.txt` first. Respect `Crawl-Delay`. Never scrape faster than one page per second. Cache pages to avoid re-fetching unchanged content. Stop if the site blocks you.

## Gemini-Powered Extraction

Instead of writing fragile CSS selectors, feed the HTML to Gemini and get structured data back. See `references/gemini-extraction.md` for prompt templates and validation patterns.

```typescript
// lib/scraper/gemini-extract.ts
import { GoogleGenerativeAI } from "@google/generative-ai";

if (!process.env.GEMINI_API_KEY) {
  throw new Error("GEMINI_API_KEY environment variable is not set");
}

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY);

export async function extractWithGemini(
  html: string,
  extractionPrompt: string
): Promise<Record<string, unknown>> {
  const model = genAI.getGenerativeModel({ model: "gemini-2.0-flash" });

  const cleanHtml = html
    .replace(/<script[\s\S]*?<\/script>/gi, "")
    .replace(/<style[\s\S]*?<\/style>/gi, "")
    .replace(/<!--[\s\S]*?-->/g, "")
    .replace(/\s+/g, " ")
    .substring(0, 30000);

  const prompt = `You are a data extraction assistant. Extract structured data from the following HTML.

${extractionPrompt}

IMPORTANT: Return ONLY valid JSON, no markdown, no code fences, no explanation.

HTML:
${cleanHtml}`;

  try {
    const result = await model.generateContent(prompt);
    const responseText = result.response.text().trim();

    const jsonStr = responseText
      .replace(/^```json?\s*/i, "")
      .replace(/```\s*$/i, "")
      .trim();

    const parsed = JSON.parse(jsonStr);
    return parsed;
  } catch (err) {
    console.error("Gemini extraction failed:", err);
    throw new Error(`Gemini extraction failed: ${err instanceof Error ? err.message : String(err)}`);
  }
}
```

## Multi-Agent Scraping Architecture

For complex scraping pipelines, split responsibilities across 5 agents: **Coordinator** (manages queue), **Fetcher** (downloads pages with anti-detection), **Parser** (extracts data via Gemini), **Validator** (checks data quality), **Storer** (saves to Supabase with deduplication).

```typescript
// lib/scraper/agents/coordinator.ts
import { fetchAgent } from "./fetcher";
import { parseAgent } from "./parser";
import { validateAgent } from "./validator";
import { storeAgent } from "./storer";

interface ScrapeJob {
  url: string;
  extractionPrompt: string;
  sourceId: string;
}

export async function coordinatorAgent(jobs: ScrapeJob[]) {
  const results = { success: 0, failed: 0, skipped: 0 };

  for (const job of jobs) {
    try {
      // Each agent has a single responsibility
      const html = await fetchAgent(job.url);         // 1. Fetch with anti-detection
      const extracted = await parseAgent(html, job.extractionPrompt); // 2. Parse via Gemini
      const validation = await validateAgent(extracted);              // 3. Validate shape
      if (!validation.valid) { results.skipped++; continue; }
      await storeAgent(job.sourceId, extracted);      // 4. Store with dedup
      results.success++;
    } catch (err) {
      console.error(`Job failed for ${job.url}:`, err);
      results.failed++;
    }
  }
  return results;
}
```

Each agent (fetcher, parser, validator, storer) is a separate file in `lib/scraper/agents/`. The fetcher checks `robots.txt` and applies random delays. The parser calls `extractWithGemini`. The validator checks for empty/missing fields. The storer hashes content to detect duplicates before inserting into Supabase.

## Error Recovery and Retry Patterns

```typescript
// lib/scraper/retry.ts
export async function withRetry<T>(
  fn: () => Promise<T>,
  maxRetries = 3,
  baseDelayMs = 1000
): Promise<T> {
  let lastError: Error | null = null;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      lastError = err instanceof Error ? err : new Error(String(err));
      console.warn(`Attempt ${attempt}/${maxRetries} failed:`, lastError.message);

      if (attempt < maxRetries) {
        const delay = baseDelayMs * Math.pow(2, attempt - 1) + Math.random() * 1000;
        await new Promise((resolve) => setTimeout(resolve, delay));
      }
    }
  }

  throw new Error(`All ${maxRetries} attempts failed. Last error: ${lastError?.message}`);
}
```

## Querying Scraped Data

```typescript
// lib/scraper/query.ts
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

// Get latest scraped data for a source
export async function getLatestData(sourceId: string) {
  const { data, error } = await supabase
    .from("scraped_data").select("*")
    .eq("source_id", sourceId).order("scraped_at", { ascending: false })
    .limit(1).single();
  if (error) throw new Error(`Query failed: ${error.message}`);
  return data;
}

// Get sources not checked in 24+ hours
export async function getStaleSources() {
  const oneDayAgo = new Date(Date.now() - 24 * 60 * 60 * 1000).toISOString();
  const { data, error } = await supabase
    .from("scrape_sources").select("*").eq("active", true)
    .or(`last_checked_at.is.null,last_checked_at.lt.${oneDayAgo}`);
  if (error) throw new Error(`Query failed: ${error.message}`);
  return data;
}
```
