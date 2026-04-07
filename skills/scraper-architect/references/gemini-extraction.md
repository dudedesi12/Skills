# Gemini-Powered Data Extraction Reference

Using Gemini API to parse unstructured HTML and PDFs into structured JSON.

## Why Use Gemini for Extraction?

Traditional scraping uses CSS selectors to find data on a page. This breaks whenever the website changes its HTML structure. Gemini can understand the meaning of content regardless of how the HTML is structured, making your scraper much more resilient.

## Setup

```bash
npm install @google/generative-ai
```

```bash
# .env.local
GEMINI_API_KEY=your_api_key_here
```

Get your API key from [aistudio.google.com](https://aistudio.google.com/).

## Core Extraction Function

```typescript
// lib/scraper/gemini-extract.ts
import { GoogleGenerativeAI } from "@google/generative-ai";

if (!process.env.GEMINI_API_KEY) {
  throw new Error("GEMINI_API_KEY environment variable is not set");
}

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY);

export async function extractWithGemini<T>(
  content: string,
  prompt: string
): Promise<T> {
  const model = genAI.getGenerativeModel({ model: "gemini-2.0-flash" });

  // Clean HTML to reduce tokens
  const cleanContent = content
    .replace(/<script[\s\S]*?<\/script>/gi, "")
    .replace(/<style[\s\S]*?<\/style>/gi, "")
    .replace(/<!--[\s\S]*?-->/g, "")
    .replace(/<(nav|header|footer)[\s\S]*?<\/\1>/gi, "")
    .replace(/\s+/g, " ")
    .substring(0, 30000);

  const fullPrompt = `${prompt}

IMPORTANT RULES:
1. Return ONLY valid JSON — no markdown code fences, no explanation, no preamble
2. If a field cannot be found, use null instead of guessing
3. Dates should be in YYYY-MM-DD format
4. Prices should be numbers (no currency symbols)

CONTENT:
${cleanContent}`;

  try {
    const result = await model.generateContent(fullPrompt);
    const text = result.response.text().trim();

    // Strip code fences if present
    const jsonStr = text
      .replace(/^```json?\s*/i, "")
      .replace(/```\s*$/i, "")
      .trim();

    return JSON.parse(jsonStr) as T;
  } catch (err) {
    throw new Error(
      `Gemini extraction failed: ${err instanceof Error ? err.message : String(err)}`
    );
  }
}
```

## Prompt Templates for Common Extraction Tasks

### Extract Product Listings

```typescript
// lib/scraper/prompts/products.ts
export const PRODUCT_LISTING_PROMPT = `Extract all product listings from this page.

Return a JSON array where each item has:
{
  "name": "Product name",
  "price": 29.99,
  "currency": "AUD",
  "description": "Short description",
  "url": "Link to product page (relative URL is fine)",
  "image_url": "Image URL",
  "in_stock": true,
  "rating": 4.5,
  "review_count": 123
}

If a field is not available, use null.`;
```

### Extract News Articles

```typescript
// lib/scraper/prompts/articles.ts
export const NEWS_ARTICLES_PROMPT = `Extract all news articles from this page.

Return a JSON array where each item has:
{
  "title": "Article headline",
  "summary": "First paragraph or summary text",
  "author": "Author name",
  "published_date": "YYYY-MM-DD",
  "url": "Link to full article",
  "category": "Category or section name",
  "image_url": "Featured image URL"
}

If a field is not available, use null.`;
```

### Extract Contact Information

```typescript
// lib/scraper/prompts/contacts.ts
export const CONTACT_INFO_PROMPT = `Extract all contact information from this page.

Return a JSON object:
{
  "company_name": "Name of the organization",
  "phone_numbers": ["list of phone numbers"],
  "email_addresses": ["list of email addresses"],
  "physical_address": "Street address",
  "city": "City",
  "state": "State/Province",
  "postcode": "Postal/ZIP code",
  "country": "Country",
  "social_media": {
    "facebook": "URL or null",
    "twitter": "URL or null",
    "linkedin": "URL or null",
    "instagram": "URL or null"
  },
  "business_hours": "Operating hours text"
}

If a field is not available, use null.`;
```

### Extract Table Data

```typescript
// lib/scraper/prompts/tables.ts
export const TABLE_DATA_PROMPT = `Extract all data from the tables on this page.

Return a JSON object:
{
  "tables": [
    {
      "title": "Table title or caption if available",
      "headers": ["Column 1", "Column 2", "Column 3"],
      "rows": [
        ["value1", "value2", "value3"],
        ["value4", "value5", "value6"]
      ]
    }
  ]
}

Preserve the original data values. Convert dates to YYYY-MM-DD format. Convert currency values to plain numbers.`;
```

### Extract Government/Regulatory Data

```typescript
// lib/scraper/prompts/regulatory.ts
export const REGULATORY_DATA_PROMPT = `Extract regulatory or policy information from this government page.

Return a JSON array where each item has:
{
  "title": "Policy or regulation title",
  "reference_number": "Official reference or ID",
  "effective_date": "YYYY-MM-DD",
  "category": "Category of regulation",
  "status": "active | proposed | repealed | amended",
  "summary": "Brief summary of the regulation",
  "affected_parties": "Who this applies to",
  "source_url": "Link to full document",
  "last_updated": "YYYY-MM-DD"
}

If a field is not available, use null.`;
```

### Extract Job Listings

```typescript
// lib/scraper/prompts/jobs.ts
export const JOB_LISTINGS_PROMPT = `Extract all job listings from this page.

Return a JSON array where each item has:
{
  "title": "Job title",
  "company": "Company name",
  "location": "Location (city, state, or remote)",
  "salary_min": 80000,
  "salary_max": 120000,
  "salary_currency": "AUD",
  "employment_type": "full-time | part-time | contract",
  "description": "Brief job description (first 200 chars)",
  "posted_date": "YYYY-MM-DD",
  "url": "Link to full listing",
  "skills": ["skill1", "skill2"]
}

If salary is not listed, use null for salary fields.`;
```

## PDF Parsing with Gemini

For PDFs, first extract the text, then send it to Gemini:

```typescript
// lib/scraper/pdf-extract.ts
import { extractWithGemini } from "./gemini-extract";

// Using pdf-parse to extract text from PDFs
// npm install pdf-parse
export async function extractFromPdf<T>(
  pdfBuffer: Buffer,
  extractionPrompt: string
): Promise<T> {
  // Dynamic import for pdf-parse
  const pdfParse = (await import("pdf-parse")).default;

  const pdfData = await pdfParse(pdfBuffer);

  if (!pdfData.text || pdfData.text.trim().length === 0) {
    throw new Error("PDF contains no extractable text (might be scanned/image-based)");
  }

  return await extractWithGemini<T>(pdfData.text, extractionPrompt);
}

// Extract from a PDF URL
export async function extractFromPdfUrl<T>(
  url: string,
  extractionPrompt: string
): Promise<T> {
  const response = await fetch(url, {
    signal: AbortSignal.timeout(30000),
    headers: {
      "User-Agent":
        "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
    },
  });

  if (!response.ok) {
    throw new Error(`Failed to download PDF: HTTP ${response.status}`);
  }

  const buffer = Buffer.from(await response.arrayBuffer());
  return await extractFromPdf<T>(buffer, extractionPrompt);
}
```

### Example: Parse an Invoice PDF

```typescript
// Usage example
import { extractFromPdfUrl } from "@/lib/scraper/pdf-extract";

interface InvoiceData {
  invoice_number: string;
  date: string;
  due_date: string;
  vendor: string;
  total: number;
  currency: string;
  line_items: Array<{
    description: string;
    quantity: number;
    unit_price: number;
    total: number;
  }>;
}

const invoice = await extractFromPdfUrl<InvoiceData>(
  "https://example.com/invoice.pdf",
  `Extract invoice data from this document.
  Return JSON with: invoice_number, date (YYYY-MM-DD), due_date (YYYY-MM-DD),
  vendor name, total (number), currency (3-letter code),
  and line_items array with description, quantity, unit_price, total.`
);
```

## Validation of Extracted Data

Always validate what Gemini returns. It can hallucinate or misinterpret data:

```typescript
// lib/scraper/validate-extraction.ts

interface ValidationRule {
  field: string;
  required: boolean;
  type: "string" | "number" | "boolean" | "array" | "object" | "date";
  minLength?: number;
  min?: number;
  max?: number;
}

interface ValidationResult {
  valid: boolean;
  errors: string[];
  warnings: string[];
}

export function validateExtraction(
  data: Record<string, unknown>,
  rules: ValidationRule[]
): ValidationResult {
  const errors: string[] = [];
  const warnings: string[] = [];

  for (const rule of rules) {
    const value = data[rule.field];

    // Check required fields
    if (rule.required && (value === null || value === undefined)) {
      errors.push(`Required field "${rule.field}" is missing`);
      continue;
    }

    if (value === null || value === undefined) continue;

    // Type checks
    switch (rule.type) {
      case "string":
        if (typeof value !== "string") {
          errors.push(`Field "${rule.field}" should be a string, got ${typeof value}`);
        } else if (rule.minLength && value.length < rule.minLength) {
          warnings.push(
            `Field "${rule.field}" is suspiciously short (${value.length} chars)`
          );
        }
        break;

      case "number":
        if (typeof value !== "number" || isNaN(value)) {
          errors.push(`Field "${rule.field}" should be a number`);
        } else {
          if (rule.min !== undefined && value < rule.min) {
            errors.push(`Field "${rule.field}" is below minimum (${value} < ${rule.min})`);
          }
          if (rule.max !== undefined && value > rule.max) {
            errors.push(`Field "${rule.field}" is above maximum (${value} > ${rule.max})`);
          }
        }
        break;

      case "date":
        if (typeof value !== "string" || !/^\d{4}-\d{2}-\d{2}$/.test(value)) {
          errors.push(`Field "${rule.field}" should be a date in YYYY-MM-DD format`);
        } else {
          const date = new Date(value);
          if (isNaN(date.getTime())) {
            errors.push(`Field "${rule.field}" is not a valid date`);
          }
        }
        break;

      case "array":
        if (!Array.isArray(value)) {
          errors.push(`Field "${rule.field}" should be an array`);
        }
        break;

      case "boolean":
        if (typeof value !== "boolean") {
          errors.push(`Field "${rule.field}" should be a boolean`);
        }
        break;

      case "object":
        if (typeof value !== "object" || Array.isArray(value)) {
          errors.push(`Field "${rule.field}" should be an object`);
        }
        break;
    }
  }

  return { valid: errors.length === 0, errors, warnings };
}
```

### Usage

```typescript
import { validateExtraction } from "@/lib/scraper/validate-extraction";

const rules = [
  { field: "title", required: true, type: "string" as const, minLength: 3 },
  { field: "price", required: true, type: "number" as const, min: 0, max: 1000000 },
  { field: "published_date", required: false, type: "date" as const },
  { field: "tags", required: false, type: "array" as const },
];

const result = validateExtraction(extractedData, rules);

if (!result.valid) {
  console.error("Extraction validation failed:", result.errors);
  // Optionally retry with a more specific prompt
}

if (result.warnings.length > 0) {
  console.warn("Extraction warnings:", result.warnings);
}
```

## Re-Extraction on Failure

If Gemini returns invalid data, retry with a more specific prompt:

```typescript
// lib/scraper/retry-extract.ts
import { extractWithGemini } from "./gemini-extract";
import { validateExtraction, type ValidationRule } from "./validate-extraction";

export async function extractWithRetry<T>(
  content: string,
  prompt: string,
  rules: ValidationRule[],
  maxRetries = 2
): Promise<T> {
  let lastErrors: string[] = [];

  for (let attempt = 1; attempt <= maxRetries + 1; attempt++) {
    const currentPrompt =
      attempt === 1
        ? prompt
        : `${prompt}

PREVIOUS ATTEMPT FAILED WITH THESE ERRORS:
${lastErrors.map((e) => `- ${e}`).join("\n")}

Please fix these issues in your response.`;

    try {
      const data = await extractWithGemini<T>(content, currentPrompt);
      const validation = validateExtraction(
        data as Record<string, unknown>,
        rules
      );

      if (validation.valid) {
        return data;
      }

      lastErrors = validation.errors;
      console.warn(`Attempt ${attempt} validation failed:`, validation.errors);
    } catch (err) {
      lastErrors = [err instanceof Error ? err.message : String(err)];
      console.warn(`Attempt ${attempt} extraction error:`, lastErrors);
    }
  }

  throw new Error(
    `Extraction failed after ${maxRetries + 1} attempts. Last errors: ${lastErrors.join(", ")}`
  );
}
```

## Choosing the Right Gemini Model

| Task | Recommended Model | Why |
|------|------------------|-----|
| Simple table extraction | `gemini-2.0-flash` | Fast, cheap, good for structured data |
| Complex multi-page documents | `gemini-2.5-pro` | Better reasoning for complex layouts |
| Quick classification (is this page relevant?) | `gemini-2.0-flash` | Only needs a yes/no answer |
| PDF with mixed content (text + tables + images) | `gemini-2.5-pro` | Handles multi-modal content better |

## Cost Management

Gemini charges per token. Here is how to keep costs down:

1. **Clean HTML before sending** - Remove scripts, styles, nav, footer, comments
2. **Truncate to essentials** - Only send the relevant portion of the page (30,000 chars max)
3. **Cache results** - Use content hashing to avoid re-processing unchanged pages
4. **Use Flash for simple tasks** - `gemini-2.0-flash` is much cheaper than Pro
5. **Batch when possible** - Send multiple small extractions in one prompt if they share the same page
