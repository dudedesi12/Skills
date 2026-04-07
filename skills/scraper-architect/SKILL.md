---
name: scraper-architect
description: >-
  Use this skill whenever the user mentions scraping, web scraping, data extraction,
  crawling, Puppeteer, Playwright scraping, government data, 'get data from website',
  PDF parsing, data collection, anti-detection, headless browser, automated browsing,
  content extraction, data pipeline, 'pull data from', immigration data, regulatory data,
  'check for updates', robots.txt, HTML parsing, DOM extraction, scheduled scraping,
  proxy rotation, data normalization, 'monitor a website', 'watch for changes',
  or ANY web data collection task — even if they don't explicitly say 'scrape'.
  This skill builds production-grade data collection pipelines.
---

# Scraper Architect

Build production-grade web scraping and data extraction pipelines using Next.js, Supabase, Vercel Cron, and Gemini API for intelligent parsing.

## When to Use This Skill

- Extracting data from websites (government portals, blogs, directories)
- PDF parsing and structured data extraction
- Scheduled data collection with change detection
- Using Gemini to parse unstructured content into structured JSON
- Building multi-step scraping pipelines

## Core Architecture

Every scraper follows this pattern:
1. **Fetch** — Get the raw HTML/PDF
2. **Parse** — Extract structured data (selectors or Gemini)
3. **Validate** — Confirm data shape and quality
4. **Store** — Save to Supabase with change tracking
5. **Schedule** — Run on Vercel Cron with error recovery

## Supabase Schema for Scraped Data

```sql
-- supabase/migrations/001_scraper_tables.sql

create table scraped_sources (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  url text not null,
  scrape_interval text not null default '24h',
  last_scraped_at timestamptz,
  last_hash text,
  status text not null default 'active',
  created_at timestamptz not null default now()
);

create table scraped_data (
  id uuid primary key default gen_random_uuid(),
  source_id uuid references scraped_sources(id) on delete cascade,
  data jsonb not null,
  content_hash text not null,
  version int not null default 1,
  scraped_at timestamptz not null default now()
);

create index idx_scraped_data_source on scraped_data(source_id, scraped_at desc);
```

## Basic Scraper with Fetch API

For simple, static pages — no JavaScript rendering needed.

```typescript
// lib/scrapers/basic-scraper.ts
import * as cheerio from "cheerio";

interface ScrapeResult<T> {
  success: boolean;
  data: T | null;
  error?: string;
}

export async function scrapePage<T>(
  url: string,
  parser: (html: string) => T
): Promise<ScrapeResult<T>> {
  try {
    const response = await fetch(url, {
      headers: {
        "User-Agent":
          "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
        Accept: "text/html,application/xhtml+xml",
      },
      signal: AbortSignal.timeout(30000),
    });

    if (!response.ok) {
      return { success: false, data: null, error: `HTTP ${response.status}` };
    }

    const html = await response.text();
    const data = parser(html);
    return { success: true, data };
  } catch (error) {
    return {
      success: false,
      data: null,
      error: error instanceof Error ? error.message : "Unknown error",
    };
  }
}

// Example parser using cheerio
export function parseTableData(html: string) {
  const $ = cheerio.load(html);
  const rows: Record<string, string>[] = [];

  const headers: string[] = [];
  $("table thead th").each((_, el) => {
    headers.push($(el).text().trim());
  });

  $("table tbody tr").each((_, row) => {
    const rowData: Record<string, string> = {};
    $(row)
      .find("td")
      .each((i, cell) => {
        if (headers[i]) {
          rowData[headers[i]] = $(cell).text().trim();
        }
      });
    rows.push(rowData);
  });

  return rows;
}
```

## Puppeteer Scraper for JavaScript-Heavy Pages

```typescript
// lib/scrapers/puppeteer-scraper.ts
import puppeteer, { type Browser } from "puppeteer";

export async function scrapeWithBrowser<T>(
  url: string,
  extractor: (page: import("puppeteer").Page) => Promise<T>
): Promise<{ success: boolean; data: T | null; error?: string }> {
  let browser: Browser | null = null;

  try {
    browser = await puppeteer.launch({
      headless: true,
      args: ["--no-sandbox", "--disable-setuid-sandbox"],
    });

    const page = await browser.newPage();

    // Anti-detection basics
    await page.setUserAgent(
      "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
    );
    await page.setViewport({ width: 1920, height: 1080 });

    await page.goto(url, { waitUntil: "networkidle2", timeout: 30000 });

    // Random delay to appear human
    await new Promise((r) => setTimeout(r, 1000 + Math.random() * 2000));

    const data = await extractor(page);
    return { success: true, data };
  } catch (error) {
    return {
      success: false,
      data: null,
      error: error instanceof Error ? error.message : "Unknown error",
    };
  } finally {
    if (browser) await browser.close();
  }
}
```

## Gemini-Powered Data Extraction

Feed raw HTML or PDF text to Gemini and get structured JSON back. This is the most powerful pattern for unstructured data.

```typescript
// lib/scrapers/gemini-extractor.ts
import { GoogleGenerativeAI } from "@google/generative-ai";

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);

interface ExtractionResult<T> {
  success: boolean;
  data: T | null;
  error?: string;
}

export async function extractWithGemini<T>(
  content: string,
  schema: string,
  instructions: string
): Promise<ExtractionResult<T>> {
  try {
    // Use gemini-2.5-pro for heavy extraction tasks
    const model = genAI.getGenerativeModel({ model: "gemini-2.5-pro" });

    const prompt = `Extract structured data from the following content.

## Output Schema
${schema}

## Instructions
${instructions}

## Content
${content}

Respond ONLY with valid JSON matching the schema. No markdown, no explanation.`;

    const result = await model.generateContent(prompt);
    const text = result.response.text().trim();

    // Clean potential markdown wrapping
    const jsonStr = text.replace(/^```json?\n?/, "").replace(/\n?```$/, "");
    const parsed = JSON.parse(jsonStr) as T;

    return { success: true, data: parsed };
  } catch (error) {
    return {
      success: false,
      data: null,
      error: error instanceof Error ? error.message : "Extraction failed",
    };
  }
}
```

### Usage Example — Extract Immigration Policy Data

```typescript
// lib/scrapers/immigration-scraper.ts
import { scrapePage } from "./basic-scraper";
import { extractWithGemini } from "./gemini-extractor";

interface PolicyUpdate {
  title: string;
  effective_date: string;
  category: string;
  summary: string;
  source_url: string;
}

export async function scrapeImmigrationUpdates(url: string) {
  const result = await scrapePage(url, (html) => html);
  if (!result.success || !result.data) return [];

  const extraction = await extractWithGemini<PolicyUpdate[]>(
    result.data,
    `[{ "title": "string", "effective_date": "YYYY-MM-DD", "category": "string", "summary": "string", "source_url": "string" }]`,
    "Extract all policy updates. Parse dates into YYYY-MM-DD format. Categorize each as 'visa', 'citizenship', 'work-permit', or 'general'."
  );

  return extraction.data ?? [];
}
```

## Scheduled Scraping with Vercel Cron

```json
// vercel.json
{
  "crons": [
    {
      "path": "/api/cron/scrape",
      "schedule": "0 6 * * *"
    }
  ]
}
```

```typescript
// app/api/cron/scrape/route.ts
import { NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";
import { scrapePage, parseTableData } from "@/lib/scrapers/basic-scraper";
import crypto from "crypto";

export async function GET(request: Request) {
  // Verify cron secret
  const authHeader = request.headers.get("authorization");
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const supabase = await createClient();

  // Get all active sources
  const { data: sources } = await supabase
    .from("scraped_sources")
    .select("*")
    .eq("status", "active");

  if (!sources) return NextResponse.json({ scraped: 0 });

  const results = [];

  for (const source of sources) {
    try {
      const result = await scrapePage(source.url, parseTableData);
      if (!result.success || !result.data) continue;

      // Check if content changed via hash
      const contentHash = crypto
        .createHash("md5")
        .update(JSON.stringify(result.data))
        .digest("hex");

      if (contentHash === source.last_hash) {
        results.push({ source: source.name, status: "unchanged" });
        continue;
      }

      // Get current version
      const { count } = await supabase
        .from("scraped_data")
        .select("*", { count: "exact", head: true })
        .eq("source_id", source.id);

      // Store new version
      await supabase.from("scraped_data").insert({
        source_id: source.id,
        data: result.data,
        content_hash: contentHash,
        version: (count ?? 0) + 1,
      });

      // Update source
      await supabase
        .from("scraped_sources")
        .update({ last_scraped_at: new Date().toISOString(), last_hash: contentHash })
        .eq("id", source.id);

      results.push({ source: source.name, status: "updated" });
    } catch (error) {
      results.push({
        source: source.name,
        status: "error",
        error: error instanceof Error ? error.message : "Unknown",
      });
    }
  }

  return NextResponse.json({ scraped: results.length, results });
}
```

## robots.txt Compliance

Always check robots.txt before scraping. This is non-negotiable.

```typescript
// lib/scrapers/robots-check.ts
export async function isAllowedByRobots(
  url: string,
  userAgent = "*"
): Promise<boolean> {
  try {
    const parsedUrl = new URL(url);
    const robotsUrl = `${parsedUrl.origin}/robots.txt`;

    const response = await fetch(robotsUrl, {
      signal: AbortSignal.timeout(5000),
    });

    if (!response.ok) return true; // No robots.txt = allowed

    const text = await response.text();
    const path = parsedUrl.pathname;

    // Simple robots.txt parser
    let currentAgent = "";
    for (const line of text.split("\n")) {
      const trimmed = line.trim().toLowerCase();
      if (trimmed.startsWith("user-agent:")) {
        currentAgent = trimmed.replace("user-agent:", "").trim();
      }
      if (
        (currentAgent === userAgent.toLowerCase() || currentAgent === "*") &&
        trimmed.startsWith("disallow:")
      ) {
        const disallowed = trimmed.replace("disallow:", "").trim();
        if (disallowed && path.startsWith(disallowed)) return false;
      }
    }

    return true;
  } catch {
    return true; // If we can't fetch robots.txt, proceed
  }
}
```

## Multi-Agent Scraping Architecture (5-Agent Pattern)

For large-scale scraping, split responsibilities across 5 agents:

1. **Coordinator** — Manages the queue, assigns URLs, tracks progress
2. **Fetcher** — Downloads pages with retry and anti-detection
3. **Parser** — Extracts data using selectors or Gemini
4. **Validator** — Checks data quality and completeness
5. **Storer** — Saves to Supabase with deduplication and versioning

> See `references/scraper-patterns.md` for the full multi-agent implementation.

## Anti-Detection Best Practices

- Rotate User-Agent strings across requests
- Add random delays between requests (1-5 seconds)
- Respect rate limits and `Crawl-delay` in robots.txt
- Use residential proxies for sensitive targets (BrightData, Oxylabs)
- Don't scrape behind authentication without permission
- Cache aggressively — don't re-fetch unchanged pages

## Key Dependencies

```bash
npm install cheerio puppeteer @google/generative-ai
```

## Reference Files

- `references/scraper-patterns.md` — Full scraping patterns including pagination, infinite scroll, form submission, file downloads, multi-agent architecture
- `references/gemini-extraction.md` — Gemini prompt templates for data extraction, PDF parsing, validation patterns
