# Scraper Patterns Reference

Complete patterns for production web scraping with Next.js, Puppeteer, and Supabase.

## Pagination Handling

```typescript
// lib/scrapers/paginated-scraper.ts
import * as cheerio from "cheerio";

interface PaginatedResult<T> {
  data: T[];
  totalPages: number;
}

export async function scrapePaginated<T>(
  baseUrl: string,
  pageParam: string,
  maxPages: number,
  parser: (html: string) => T[]
): Promise<PaginatedResult<T>> {
  const allData: T[] = [];
  let currentPage = 1;

  while (currentPage <= maxPages) {
    const url = `${baseUrl}?${pageParam}=${currentPage}`;

    const response = await fetch(url, {
      headers: {
        "User-Agent":
          "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
      },
      signal: AbortSignal.timeout(30000),
    });

    if (!response.ok) break;

    const html = await response.text();
    const pageData = parser(html);

    if (pageData.length === 0) break; // No more data

    allData.push(...pageData);
    currentPage++;

    // Respectful delay between pages
    await new Promise((r) => setTimeout(r, 1500 + Math.random() * 1500));
  }

  return { data: allData, totalPages: currentPage - 1 };
}
```

## Infinite Scroll Handling (Puppeteer)

```typescript
// lib/scrapers/infinite-scroll.ts
import puppeteer from "puppeteer";

export async function scrapeInfiniteScroll(
  url: string,
  itemSelector: string,
  maxScrolls = 10
): Promise<string[]> {
  const browser = await puppeteer.launch({
    headless: true,
    args: ["--no-sandbox"],
  });

  try {
    const page = await browser.newPage();
    await page.goto(url, { waitUntil: "networkidle2" });

    let previousHeight = 0;
    let scrollCount = 0;

    while (scrollCount < maxScrolls) {
      const currentHeight = await page.evaluate(() => document.body.scrollHeight);

      if (currentHeight === previousHeight) break; // No more content

      previousHeight = currentHeight;
      await page.evaluate(() => window.scrollTo(0, document.body.scrollHeight));

      // Wait for new content to load
      await new Promise((r) => setTimeout(r, 2000));
      scrollCount++;
    }

    // Extract all items
    const items = await page.$$eval(itemSelector, (elements) =>
      elements.map((el) => el.textContent?.trim() ?? "")
    );

    return items;
  } finally {
    await browser.close();
  }
}
```

## Form Submission Scraping

```typescript
// lib/scrapers/form-scraper.ts
import puppeteer from "puppeteer";

export async function scrapeWithFormSubmission(
  url: string,
  formData: Record<string, string>,
  resultSelector: string
) {
  const browser = await puppeteer.launch({
    headless: true,
    args: ["--no-sandbox"],
  });

  try {
    const page = await browser.newPage();
    await page.goto(url, { waitUntil: "networkidle2" });

    // Fill form fields
    for (const [selector, value] of Object.entries(formData)) {
      await page.waitForSelector(selector, { timeout: 5000 });
      await page.type(selector, value, { delay: 50 }); // Human-like typing
    }

    // Submit and wait for results
    await Promise.all([
      page.waitForNavigation({ waitUntil: "networkidle2" }),
      page.click('button[type="submit"]'),
    ]);

    // Extract results
    const results = await page.$$eval(resultSelector, (elements) =>
      elements.map((el) => el.innerHTML)
    );

    return { success: true, data: results };
  } catch (error) {
    return {
      success: false,
      data: null,
      error: error instanceof Error ? error.message : "Unknown",
    };
  } finally {
    await browser.close();
  }
}
```

## File Download Scraping

```typescript
// lib/scrapers/file-downloader.ts
import { createClient } from "@/lib/supabase/server";

export async function downloadAndStore(
  fileUrl: string,
  bucket: string,
  filePath: string
) {
  const response = await fetch(fileUrl, {
    signal: AbortSignal.timeout(60000),
  });

  if (!response.ok) {
    throw new Error(`Failed to download: HTTP ${response.status}`);
  }

  const buffer = Buffer.from(await response.arrayBuffer());
  const supabase = await createClient();

  const { error } = await supabase.storage
    .from(bucket)
    .upload(filePath, buffer, {
      contentType: response.headers.get("content-type") ?? "application/octet-stream",
      upsert: true,
    });

  if (error) throw error;

  return { size: buffer.length, path: filePath };
}
```

## Cookie and Session Management

```typescript
// lib/scrapers/session-scraper.ts
import puppeteer from "puppeteer";

export async function scrapeWithLogin(
  loginUrl: string,
  credentials: { username: string; password: string },
  targetUrl: string,
  extractor: (page: import("puppeteer").Page) => Promise<unknown>
) {
  const browser = await puppeteer.launch({
    headless: true,
    args: ["--no-sandbox"],
  });

  try {
    const page = await browser.newPage();

    // Login
    await page.goto(loginUrl, { waitUntil: "networkidle2" });
    await page.type('input[name="username"]', credentials.username);
    await page.type('input[name="password"]', credentials.password);
    await Promise.all([
      page.waitForNavigation(),
      page.click('button[type="submit"]'),
    ]);

    // Navigate to target with session cookies
    await page.goto(targetUrl, { waitUntil: "networkidle2" });

    const data = await extractor(page);
    return { success: true, data };
  } catch (error) {
    return {
      success: false,
      data: null,
      error: error instanceof Error ? error.message : "Unknown",
    };
  } finally {
    await browser.close();
  }
}
```

## User-Agent Rotation

```typescript
// lib/scrapers/user-agents.ts
const USER_AGENTS = [
  "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
  "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
  "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:121.0) Gecko/20100101 Firefox/121.0",
  "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.2 Safari/605.1.15",
  "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
];

export function getRandomUserAgent(): string {
  return USER_AGENTS[Math.floor(Math.random() * USER_AGENTS.length)];
}
```

## Multi-Agent Architecture (5-Agent Pattern)

For large-scale scraping, decompose into independent agents communicating via a Supabase queue:

```sql
-- supabase/migrations/002_scrape_queue.sql

create type job_status as enum ('pending', 'fetching', 'parsing', 'validating', 'storing', 'done', 'failed');

create table scrape_jobs (
  id uuid primary key default gen_random_uuid(),
  url text not null,
  status job_status not null default 'pending',
  raw_html text,
  parsed_data jsonb,
  validated boolean default false,
  error text,
  attempts int default 0,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);
```

### Agent 1: Coordinator

```typescript
// app/api/cron/scrape-coordinator/route.ts
// Runs on cron, picks pending URLs, assigns to fetcher queue
```

### Agent 2: Fetcher

```typescript
// app/api/cron/scrape-fetch/route.ts
// Picks jobs with status 'pending', fetches HTML, updates to 'fetching' → 'parsing'
```

### Agent 3: Parser

```typescript
// app/api/cron/scrape-parse/route.ts
// Picks jobs with status 'parsing', extracts data with Gemini, updates to 'validating'
```

### Agent 4: Validator

```typescript
// app/api/cron/scrape-validate/route.ts
// Picks jobs with status 'validating', checks data quality, updates to 'storing'
```

### Agent 5: Storer

```typescript
// app/api/cron/scrape-store/route.ts
// Picks jobs with status 'storing', deduplicates, saves to final tables, marks 'done'
```

## Error Recovery Pattern

```typescript
// lib/scrapers/retry.ts
export async function withRetry<T>(
  fn: () => Promise<T>,
  maxAttempts = 3,
  baseDelay = 1000
): Promise<T> {
  let lastError: Error | undefined;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error instanceof Error ? error : new Error(String(error));

      if (attempt < maxAttempts) {
        const delay = baseDelay * Math.pow(2, attempt - 1) + Math.random() * 1000;
        await new Promise((r) => setTimeout(r, delay));
      }
    }
  }

  throw lastError;
}
```
