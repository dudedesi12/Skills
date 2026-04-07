# Scraper Patterns Reference

Complete patterns for production web scraping with Playwright.

## Page Navigation

```typescript
// lib/scraper/patterns/navigation.ts
import { type Page } from "playwright";

// Navigate and wait for content
export async function navigateAndWait(page: Page, url: string) {
  const response = await page.goto(url, {
    waitUntil: "domcontentloaded",
    timeout: 30000,
  });

  if (!response || response.status() >= 400) {
    throw new Error(`HTTP ${response?.status()} for ${url}`);
  }

  // Wait for dynamic content to load
  await page.waitForLoadState("networkidle");
  return response;
}

// Navigate through a multi-step flow
export async function navigateSteps(page: Page, steps: string[]) {
  const results: string[] = [];

  for (const step of steps) {
    await page.goto(step, { waitUntil: "domcontentloaded", timeout: 30000 });
    const html = await page.content();
    results.push(html);

    // Random delay between steps
    await page.waitForTimeout(1000 + Math.random() * 2000);
  }

  return results;
}

// Click through to a detail page and come back
export async function scrapeDetailPage(
  page: Page,
  linkSelector: string
): Promise<string> {
  const href = await page.getAttribute(linkSelector, "href");
  if (!href) throw new Error(`No href found for ${linkSelector}`);

  const detailPage = await page.context().newPage();
  try {
    await detailPage.goto(href, { waitUntil: "domcontentloaded", timeout: 30000 });
    return await detailPage.content();
  } finally {
    await detailPage.close();
  }
}
```

## Form Submission

```typescript
// lib/scraper/patterns/forms.ts
import { type Page } from "playwright";

// Fill and submit a search form
export async function submitSearchForm(
  page: Page,
  formSelector: string,
  fields: Record<string, string>
) {
  for (const [selector, value] of Object.entries(fields)) {
    const element = page.locator(selector);
    const tagName = await element.evaluate((el) => el.tagName.toLowerCase());

    if (tagName === "select") {
      await element.selectOption(value);
    } else if (tagName === "input") {
      const inputType = await element.getAttribute("type");
      if (inputType === "checkbox" || inputType === "radio") {
        if (value === "true") await element.check();
      } else {
        await element.fill(value);
      }
    } else {
      await element.fill(value);
    }

    // Small delay between fields to appear human
    await page.waitForTimeout(200 + Math.random() * 300);
  }

  // Submit the form
  const form = page.locator(formSelector);
  const submitButton = form.locator('button[type="submit"], input[type="submit"]');

  await Promise.all([
    page.waitForLoadState("networkidle"),
    submitButton.click(),
  ]);
}

// Handle login forms
export async function loginToSite(
  page: Page,
  loginUrl: string,
  credentials: { username: string; password: string }
) {
  await page.goto(loginUrl, { waitUntil: "domcontentloaded" });

  await page.fill(
    'input[type="email"], input[name="email"], input[name="username"]',
    credentials.username
  );
  await page.fill('input[type="password"]', credentials.password);

  await Promise.all([
    page.waitForNavigation({ timeout: 15000 }),
    page.click('button[type="submit"]'),
  ]);
}
```

## Pagination

```typescript
// lib/scraper/patterns/pagination.ts
import { type Page } from "playwright";

// Scrape all pages using "Next" button
export async function scrapeAllPages(
  page: Page,
  startUrl: string,
  itemSelector: string,
  nextButtonSelector: string,
  maxPages = 50
): Promise<string[][]> {
  const allItems: string[][] = [];

  await page.goto(startUrl, { waitUntil: "domcontentloaded" });

  for (let pageNum = 1; pageNum <= maxPages; pageNum++) {
    // Extract items from current page
    const items = await page.$$eval(itemSelector, (elements) =>
      elements.map((el) => el.outerHTML)
    );
    allItems.push(items);

    // Check if there is a next page
    const nextButton = page.locator(nextButtonSelector);
    const isDisabled = await nextButton.isDisabled().catch(() => true);
    const isVisible = await nextButton.isVisible().catch(() => false);

    if (!isVisible || isDisabled) {
      break; // No more pages
    }

    await Promise.all([
      page.waitForLoadState("networkidle"),
      nextButton.click(),
    ]);

    // Delay between pages
    await page.waitForTimeout(1000 + Math.random() * 2000);
  }

  return allItems;
}

// Scrape pages using URL-based pagination (?page=1, ?page=2, etc.)
export async function scrapeNumberedPages(
  page: Page,
  baseUrl: string,
  pageParam: string,
  itemSelector: string,
  maxPages = 50
): Promise<string[][]> {
  const allItems: string[][] = [];

  for (let pageNum = 1; pageNum <= maxPages; pageNum++) {
    const url = `${baseUrl}${baseUrl.includes("?") ? "&" : "?"}${pageParam}=${pageNum}`;
    await page.goto(url, { waitUntil: "domcontentloaded" });

    const items = await page.$$eval(itemSelector, (elements) =>
      elements.map((el) => el.outerHTML)
    );

    if (items.length === 0) break; // Empty page means we are done

    allItems.push(items);
    await page.waitForTimeout(1000 + Math.random() * 2000);
  }

  return allItems;
}
```

## Infinite Scroll

```typescript
// lib/scraper/patterns/infinite-scroll.ts
import { type Page } from "playwright";

export async function scrapeInfiniteScroll(
  page: Page,
  url: string,
  itemSelector: string,
  maxItems = 500,
  maxScrollAttempts = 100
): Promise<string[]> {
  await page.goto(url, { waitUntil: "domcontentloaded" });

  let previousItemCount = 0;
  let noNewItemsCount = 0;

  for (let attempt = 0; attempt < maxScrollAttempts; attempt++) {
    // Scroll to bottom
    await page.evaluate(() => window.scrollTo(0, document.body.scrollHeight));

    // Wait for new content to load
    await page.waitForTimeout(2000);

    const currentItemCount = await page.locator(itemSelector).count();

    if (currentItemCount >= maxItems) break;

    if (currentItemCount === previousItemCount) {
      noNewItemsCount++;
      if (noNewItemsCount >= 3) break; // No new items after 3 scrolls
    } else {
      noNewItemsCount = 0;
    }

    previousItemCount = currentItemCount;
  }

  // Extract all items
  const items = await page.$$eval(itemSelector, (elements) =>
    elements.map((el) => el.outerHTML)
  );

  return items;
}
```

## File Download

```typescript
// lib/scraper/patterns/download.ts
import { type Page } from "playwright";
import fs from "fs/promises";
import path from "path";

// Download a file by clicking a link
export async function downloadFile(
  page: Page,
  downloadSelector: string,
  savePath: string
): Promise<string> {
  // Ensure the download directory exists
  await fs.mkdir(path.dirname(savePath), { recursive: true });

  const [download] = await Promise.all([
    page.waitForEvent("download"),
    page.click(downloadSelector),
  ]);

  await download.saveAs(savePath);
  return savePath;
}

// Download a file from a direct URL
export async function downloadFromUrl(
  url: string,
  savePath: string
): Promise<string> {
  const response = await fetch(url, {
    headers: {
      "User-Agent":
        "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
    },
    signal: AbortSignal.timeout(60000),
  });

  if (!response.ok) {
    throw new Error(`Download failed: HTTP ${response.status}`);
  }

  const buffer = Buffer.from(await response.arrayBuffer());
  await fs.mkdir(path.dirname(savePath), { recursive: true });
  await fs.writeFile(savePath, buffer);

  return savePath;
}

// Download a PDF and return its text content
export async function downloadAndReadPdf(
  page: Page,
  pdfUrl: string
): Promise<string> {
  const response = await page.request.get(pdfUrl);
  const buffer = await response.body();

  // Save temporarily
  const tempPath = `/tmp/scraper-pdf-${Date.now()}.pdf`;
  await fs.writeFile(tempPath, buffer);

  // You would use a PDF parser here (like pdf-parse)
  // npm install pdf-parse
  // const pdfParse = require("pdf-parse");
  // const pdfData = await pdfParse(buffer);
  // return pdfData.text;

  return tempPath;
}
```

## Screenshot Capture

```typescript
// lib/scraper/patterns/screenshot.ts
import { type Page } from "playwright";
import fs from "fs/promises";

// Take a full-page screenshot
export async function captureFullPage(
  page: Page,
  savePath: string
): Promise<string> {
  await fs.mkdir(savePath.substring(0, savePath.lastIndexOf("/")), {
    recursive: true,
  });

  await page.screenshot({
    path: savePath,
    fullPage: true,
  });

  return savePath;
}

// Take a screenshot of a specific element
export async function captureElement(
  page: Page,
  selector: string,
  savePath: string
): Promise<string> {
  const element = page.locator(selector);
  await element.waitFor({ state: "visible", timeout: 10000 });

  await element.screenshot({ path: savePath });
  return savePath;
}

// Take before/after screenshots for change detection
export async function captureForComparison(
  page: Page,
  url: string,
  outputDir: string
): Promise<{ path: string; timestamp: string }> {
  await page.goto(url, { waitUntil: "networkidle" });

  const timestamp = new Date().toISOString().replace(/[:.]/g, "-");
  const filename = `${outputDir}/screenshot-${timestamp}.png`;

  await page.screenshot({ path: filename, fullPage: true });

  return { path: filename, timestamp };
}
```

## Cookie and Session Management

```typescript
// lib/scraper/patterns/session.ts
import { chromium, type BrowserContext } from "playwright";
import fs from "fs/promises";

const SESSION_FILE = "/tmp/scraper-session.json";

// Save session cookies to a file for reuse
export async function saveSession(context: BrowserContext): Promise<void> {
  const cookies = await context.cookies();
  await fs.writeFile(SESSION_FILE, JSON.stringify(cookies, null, 2));
}

// Load saved session cookies
export async function loadSession(context: BrowserContext): Promise<boolean> {
  try {
    const data = await fs.readFile(SESSION_FILE, "utf-8");
    const cookies = JSON.parse(data);
    await context.addCookies(cookies);
    return true;
  } catch {
    return false; // No saved session
  }
}

// Create a context with saved session or log in fresh
export async function getAuthenticatedContext(
  loginUrl: string,
  credentials: { username: string; password: string }
) {
  const browser = await chromium.launch({ headless: true });
  const context = await browser.newContext();

  // Try loading saved session
  const hasSession = await loadSession(context);

  if (!hasSession) {
    // Log in fresh
    const page = await context.newPage();
    await page.goto(loginUrl, { waitUntil: "domcontentloaded" });

    await page.fill('input[type="email"], input[name="email"]', credentials.username);
    await page.fill('input[type="password"]', credentials.password);

    await Promise.all([
      page.waitForNavigation(),
      page.click('button[type="submit"]'),
    ]);

    await saveSession(context);
    await page.close();
  }

  return { browser, context };
}
```

## Parallel Scraping

```typescript
// lib/scraper/patterns/parallel.ts
import { chromium } from "playwright";

interface ScrapeTask {
  url: string;
  selector: string;
}

export async function scrapeInParallel(
  tasks: ScrapeTask[],
  concurrency = 3
): Promise<Map<string, string>> {
  const browser = await chromium.launch({ headless: true });
  const results = new Map<string, string>();

  try {
    // Process tasks in batches
    for (let i = 0; i < tasks.length; i += concurrency) {
      const batch = tasks.slice(i, i + concurrency);

      const batchResults = await Promise.allSettled(
        batch.map(async (task) => {
          const context = await browser.newContext();
          const page = await context.newPage();

          try {
            await page.goto(task.url, {
              waitUntil: "domcontentloaded",
              timeout: 30000,
            });
            const content = await page.locator(task.selector).innerHTML();
            return { url: task.url, content };
          } finally {
            await context.close();
          }
        })
      );

      for (const result of batchResults) {
        if (result.status === "fulfilled") {
          results.set(result.value.url, result.value.content);
        }
      }

      // Delay between batches
      if (i + concurrency < tasks.length) {
        await new Promise((r) => setTimeout(r, 2000));
      }
    }
  } finally {
    await browser.close();
  }

  return results;
}
```

## Handling Dynamic Content

```typescript
// lib/scraper/patterns/dynamic.ts
import { type Page } from "playwright";

// Wait for AJAX content to load
export async function waitForAjaxContent(
  page: Page,
  selector: string,
  timeout = 15000
): Promise<void> {
  await page.waitForSelector(selector, { state: "visible", timeout });
}

// Wait for a loading spinner to disappear
export async function waitForSpinner(
  page: Page,
  spinnerSelector = ".loading, .spinner, [data-loading]",
  timeout = 15000
): Promise<void> {
  try {
    // Wait for spinner to appear (it might already be gone)
    await page.waitForSelector(spinnerSelector, { state: "visible", timeout: 2000 });
  } catch {
    return; // Spinner never appeared, content is already loaded
  }
  // Wait for spinner to disappear
  await page.waitForSelector(spinnerSelector, { state: "hidden", timeout });
}

// Click a "Load More" button until all content is loaded
export async function loadAllContent(
  page: Page,
  loadMoreSelector: string,
  itemSelector: string,
  maxClicks = 50
): Promise<number> {
  let clicks = 0;

  while (clicks < maxClicks) {
    const loadMore = page.locator(loadMoreSelector);
    const isVisible = await loadMore.isVisible().catch(() => false);

    if (!isVisible) break;

    await loadMore.click();
    await page.waitForTimeout(1500);
    clicks++;
  }

  return await page.locator(itemSelector).count();
}
```
