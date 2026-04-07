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

### Extracting Structured Data from a Page

```typescript
// lib/scraper/extract.ts
import { chromium } from "playwright";

interface ProductData {
  name: string;
  price: string;
  description: string;
  imageUrl: string;
}

export async function scrapeProductPage(url: string): Promise<ProductData> {
  const browser = await chromium.launch({ headless: true });

  try {
    const page = await browser.newPage();
    await page.goto(url, { waitUntil: "domcontentloaded", timeout: 30000 });

    const data = await page.evaluate(() => {
      const getText = (selector: string): string =>
        document.querySelector(selector)?.textContent?.trim() || "";
      const getAttr = (selector: string, attr: string): string =>
        document.querySelector(selector)?.getAttribute(attr) || "";

      return {
        name: getText("h1"),
        price: getText("[data-price], .price, .product-price"),
        description: getText("[data-description], .description, .product-description"),
        imageUrl: getAttr("img.product-image, .product img", "src"),
      };
    });

    return data;
  } catch (err) {
    console.error(`Product scrape failed:`, err);
    throw err;
  } finally {
    await browser.close();
  }
}
```

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
    geolocation: { latitude: -33.8688, longitude: 151.2093 },
    permissions: ["geolocation"],
  });

  // Remove the "webdriver" flag that identifies automated browsers
  await context.addInitScript(() => {
    Object.defineProperty(navigator, "webdriver", { get: () => false });
  });

  return { browser, context };
}
```

## Data Normalization and Cleaning

Raw scraped data is messy. Clean it before storing:

```typescript
// lib/scraper/normalize.ts
export function cleanText(raw: string): string {
  return raw
    .replace(/\s+/g, " ")
    .replace(/[\n\r\t]/g, " ")
    .trim();
}

export function extractPrice(raw: string): number | null {
  const match = raw.match(/[\d,]+\.?\d*/);
  if (!match) return null;
  return parseFloat(match[0].replace(/,/g, ""));
}

export function extractDate(raw: string): string | null {
  const patterns = [
    /(\d{1,2})\/(\d{1,2})\/(\d{4})/,
    /(\d{4})-(\d{2})-(\d{2})/,
    /(\d{1,2})\s+(Jan|Feb|Mar|Apr|May|Jun|Jul|Aug|Sep|Oct|Nov|Dec)\w*\s+(\d{4})/i,
  ];

  for (const pattern of patterns) {
    const match = raw.match(pattern);
    if (match) {
      try {
        const date = new Date(match[0]);
        if (!isNaN(date.getTime())) {
          return date.toISOString().split("T")[0];
        }
      } catch {
        continue;
      }
    }
  }
  return null;
}

export function normalizeUrl(url: string, baseUrl: string): string {
  try {
    return new URL(url, baseUrl).href;
  } catch {
    return url;
  }
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
      .from("scrape_sources")
      .select("*")
      .eq("active", true);

    if (error) throw new Error(`DB error: ${error.message}`);

    for (const source of sources || []) {
      try {
        const html = await scrapePage(source.url);
        const contentHash = await hashContent(html);

        if (contentHash === source.last_content_hash) {
          results.unchanged++;
          await supabase
            .from("scrape_sources")
            .update({ last_checked_at: new Date().toISOString() })
            .eq("id", source.id);
          continue;
        }

        const extracted = await extractWithGemini(html, source.extraction_prompt);

        const { error: insertError } = await supabase.from("scraped_data").insert({
          source_id: source.id,
          data: extracted,
          content_hash: contentHash,
          scraped_at: new Date().toISOString(),
        });

        if (insertError) throw new Error(`Insert error: ${insertError.message}`);

        await supabase
          .from("scrape_sources")
          .update({
            last_content_hash: contentHash,
            last_scraped_at: new Date().toISOString(),
            last_checked_at: new Date().toISOString(),
            consecutive_failures: 0,
          })
          .eq("id", source.id);

        results.scraped++;
      } catch (err) {
        results.failed++;
        console.error(`Failed to scrape ${source.url}:`, err);

        await supabase
          .from("scrape_sources")
          .update({
            consecutive_failures: (source.consecutive_failures || 0) + 1,
            last_error: err instanceof Error ? err.message : String(err),
            last_checked_at: new Date().toISOString(),
          })
          .eq("id", source.id);
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

**Ethical scraping rules:**
1. Always check `robots.txt` first
2. Respect `Crawl-Delay` headers
3. Do not scrape faster than one page per second
4. Identify yourself with a proper User-Agent when required
5. Cache pages and avoid re-scraping unchanged content
6. Stop immediately if the site blocks you

## Gemini-Powered Extraction

Instead of writing fragile CSS selectors, feed the HTML to Gemini and let it extract structured data:

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

For complex scraping pipelines, use a 5-agent pattern where each agent has one job:

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
      const html = await fetchAgent(job.url);
      const extracted = await parseAgent(html, job.extractionPrompt);

      const validation = await validateAgent(extracted);
      if (!validation.valid) {
        console.warn(`Validation failed for ${job.url}:`, validation.errors);
        results.skipped++;
        continue;
      }

      await storeAgent(job.sourceId, extracted);
      results.success++;
    } catch (err) {
      console.error(`Job failed for ${job.url}:`, err);
      results.failed++;
    }
  }

  return results;
}
```

```typescript
// lib/scraper/agents/fetcher.ts
import { chromium } from "playwright";
import { getRandomUserAgent } from "../user-agents";
import { randomDelay } from "../delays";
import { checkRobotsTxt } from "../robots";

export async function fetchAgent(url: string): Promise<string> {
  const { allowed, crawlDelay } = await checkRobotsTxt(url);
  if (!allowed) {
    throw new Error(`Scraping not allowed by robots.txt: ${url}`);
  }

  await randomDelay(crawlDelay * 1000, crawlDelay * 2000);

  const browser = await chromium.launch({ headless: true });
  try {
    const context = await browser.newContext({
      userAgent: getRandomUserAgent(),
      viewport: { width: 1920, height: 1080 },
    });

    const page = await context.newPage();
    await page.route("**/*.{png,jpg,jpeg,gif,svg,woff,woff2}", (route) => route.abort());

    const response = await page.goto(url, { waitUntil: "domcontentloaded", timeout: 30000 });

    if (!response || response.status() >= 400) {
      throw new Error(`HTTP ${response?.status()} for ${url}`);
    }

    return await page.content();
  } finally {
    await browser.close();
  }
}
```

```typescript
// lib/scraper/agents/parser.ts
import { extractWithGemini } from "../gemini-extract";

export async function parseAgent(
  html: string,
  extractionPrompt: string
): Promise<Record<string, unknown>> {
  return await extractWithGemini(html, extractionPrompt);
}
```

```typescript
// lib/scraper/agents/validator.ts
interface ValidationResult {
  valid: boolean;
  errors: string[];
}

export async function validateAgent(
  data: Record<string, unknown>
): Promise<ValidationResult> {
  const errors: string[] = [];

  if (!data || typeof data !== "object") {
    errors.push("Extracted data is not an object");
    return { valid: false, errors };
  }

  if (Object.keys(data).length === 0) {
    errors.push("Extracted data is empty");
    return { valid: false, errors };
  }

  for (const [key, value] of Object.entries(data)) {
    if (typeof value === "string" && value.length === 0) {
      errors.push(`Field "${key}" is empty`);
    }
  }

  return { valid: errors.length === 0, errors };
}
```

```typescript
// lib/scraper/agents/storer.ts
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export async function storeAgent(
  sourceId: string,
  data: Record<string, unknown>
): Promise<void> {
  const contentHash = await hashData(data);

  const { data: existing } = await supabase
    .from("scraped_data")
    .select("id")
    .eq("source_id", sourceId)
    .eq("content_hash", contentHash)
    .limit(1);

  if (existing && existing.length > 0) {
    await supabase
      .from("scrape_sources")
      .update({ last_checked_at: new Date().toISOString() })
      .eq("id", sourceId);
    return;
  }

  const { error } = await supabase.from("scraped_data").insert({
    source_id: sourceId,
    data,
    content_hash: contentHash,
    scraped_at: new Date().toISOString(),
  });

  if (error) throw new Error(`Store failed: ${error.message}`);

  await supabase
    .from("scrape_sources")
    .update({
      last_scraped_at: new Date().toISOString(),
      last_checked_at: new Date().toISOString(),
      last_content_hash: contentHash,
      consecutive_failures: 0,
    })
    .eq("id", sourceId);
}

async function hashData(data: Record<string, unknown>): Promise<string> {
  const encoder = new TextEncoder();
  const encoded = encoder.encode(JSON.stringify(data));
  const hashBuffer = await crypto.subtle.digest("SHA-256", encoded);
  return Array.from(new Uint8Array(hashBuffer))
    .map((b) => b.toString(16).padStart(2, "0"))
    .join("");
}
```

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

## Storing Scraped Data in Supabase with Change Tracking

```typescript
// lib/scraper/query.ts
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export async function getLatestData(sourceId: string) {
  const { data, error } = await supabase
    .from("scraped_data")
    .select("*")
    .eq("source_id", sourceId)
    .order("scraped_at", { ascending: false })
    .limit(1)
    .single();

  if (error) throw new Error(`Query failed: ${error.message}`);
  return data;
}

export async function getChangeHistory(sourceId: string, limit = 10) {
  const { data, error } = await supabase
    .from("scraped_data")
    .select("data, scraped_at, content_hash")
    .eq("source_id", sourceId)
    .order("scraped_at", { ascending: false })
    .limit(limit);

  if (error) throw new Error(`Query failed: ${error.message}`);
  return data;
}

export async function getStaleSources() {
  const oneDayAgo = new Date(Date.now() - 24 * 60 * 60 * 1000).toISOString();

  const { data, error } = await supabase
    .from("scrape_sources")
    .select("*")
    .eq("active", true)
    .or(`last_checked_at.is.null,last_checked_at.lt.${oneDayAgo}`);

  if (error) throw new Error(`Query failed: ${error.message}`);
  return data;
}
```
