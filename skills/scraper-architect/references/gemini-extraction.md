# Gemini-Powered Data Extraction Reference

Using Gemini API to parse unstructured HTML, PDFs, and documents into structured JSON.

## Why Gemini for Extraction?

Traditional scraping breaks when websites change their HTML structure. Gemini-powered extraction is resilient — it understands content semantically, not structurally. Feed it messy HTML and get clean JSON back.

## Model Selection

| Task | Model | Why |
|------|-------|-----|
| PDF parsing, large documents | `gemini-2.5-pro` | Handles long context, complex reasoning |
| Quick page extraction | `gemini-2.0-flash` | Fast, cost-effective for simple extractions |
| Classification (is page relevant?) | Flash Lite | Cheapest, fast yes/no decisions |

## Core Extraction Function

```typescript
// lib/scrapers/gemini-extractor.ts
import { GoogleGenerativeAI } from "@google/generative-ai";

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);

export async function extractStructuredData<T>(
  content: string,
  outputSchema: string,
  instructions: string,
  model: "gemini-2.5-pro" | "gemini-2.0-flash" = "gemini-2.0-flash"
): Promise<T> {
  const genModel = genAI.getGenerativeModel({ model });

  const prompt = `You are a data extraction specialist. Extract structured data from the provided content.

## Required Output Format (JSON)
${outputSchema}

## Extraction Rules
${instructions}

## Content to Process
${content}

IMPORTANT: Respond with ONLY valid JSON. No markdown code fences, no explanations.`;

  const result = await genModel.generateContent(prompt);
  const text = result.response.text().trim();

  // Clean markdown wrapping if present
  const cleaned = text.replace(/^```json?\n?/, "").replace(/\n?```$/, "");
  return JSON.parse(cleaned) as T;
}
```

## Prompt Templates

### Template 1: Table Data Extraction

```typescript
const schema = `{
  "headers": ["string"],
  "rows": [{"column_name": "value"}]
}`;

const instructions = `
- Extract ALL rows from any tables found in the content
- Use the table headers as JSON keys (lowercase, underscored)
- Convert dates to YYYY-MM-DD format
- Convert currencies to numbers (remove $ signs)
- If a cell is empty, use null`;
```

### Template 2: Contact Information

```typescript
const schema = `{
  "contacts": [{
    "name": "string",
    "email": "string | null",
    "phone": "string | null",
    "role": "string | null",
    "organization": "string | null"
  }]
}`;

const instructions = `
- Extract ALL people/contacts mentioned in the content
- Normalize phone numbers to E.164 format if possible
- If role or organization is not explicitly stated, infer from context or use null`;
```

### Template 3: Regulatory/Policy Updates

```typescript
const schema = `{
  "updates": [{
    "title": "string",
    "effective_date": "YYYY-MM-DD | null",
    "category": "visa | citizenship | work-permit | travel | general",
    "summary": "string (2-3 sentences)",
    "key_changes": ["string"],
    "affected_groups": ["string"],
    "source_section": "string"
  }]
}`;

const instructions = `
- Extract ALL policy or regulatory updates
- Categorize each update into the specified categories
- Summarize in plain English (no jargon)
- List who is affected by each change
- Note which section of the source page the info came from`;
```

### Template 4: Product/Service Listings

```typescript
const schema = `{
  "listings": [{
    "name": "string",
    "price": "number | null",
    "currency": "string",
    "description": "string",
    "features": ["string"],
    "url": "string | null",
    "availability": "in-stock | out-of-stock | unknown"
  }]
}`;

const instructions = `
- Extract ALL products or services listed
- Convert prices to numbers (e.g., "$49.99" → 49.99)
- Default currency to AUD unless otherwise specified
- Extract feature lists or bullet points as the features array`;
```

## PDF Extraction

```typescript
// lib/scrapers/pdf-extractor.ts
import { GoogleGenerativeAI } from "@google/generative-ai";

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);

export async function extractFromPDF(
  pdfBuffer: Buffer,
  instructions: string,
  outputSchema: string
) {
  // Use gemini-2.5-pro for PDF (large context)
  const model = genAI.getGenerativeModel({ model: "gemini-2.5-pro" });

  const pdfPart = {
    inlineData: {
      data: pdfBuffer.toString("base64"),
      mimeType: "application/pdf",
    },
  };

  const prompt = `Extract structured data from this PDF document.

## Output Schema
${outputSchema}

## Instructions
${instructions}

Respond with ONLY valid JSON.`;

  const result = await model.generateContent([prompt, pdfPart]);
  const text = result.response.text().trim();
  const cleaned = text.replace(/^```json?\n?/, "").replace(/\n?```$/, "");
  return JSON.parse(cleaned);
}
```

## Validation of Extracted Data

Always validate Gemini's output before storing:

```typescript
// lib/scrapers/validate-extraction.ts
import { z } from "zod";

// Define expected schema with Zod
const PolicyUpdateSchema = z.object({
  title: z.string().min(5),
  effective_date: z.string().regex(/^\d{4}-\d{2}-\d{2}$/).nullable(),
  category: z.enum(["visa", "citizenship", "work-permit", "travel", "general"]),
  summary: z.string().min(20),
  key_changes: z.array(z.string()).min(1),
  affected_groups: z.array(z.string()),
});

const ExtractionResultSchema = z.object({
  updates: z.array(PolicyUpdateSchema),
});

export function validateExtraction(data: unknown) {
  const result = ExtractionResultSchema.safeParse(data);

  if (!result.success) {
    console.error("Validation errors:", result.error.issues);
    return { valid: false, errors: result.error.issues, data: null };
  }

  return { valid: true, errors: [], data: result.data };
}
```

## Classification Before Extraction (Cost Optimization)

Use Flash Lite to pre-filter pages before expensive extraction:

```typescript
// lib/scrapers/classify-page.ts
export async function isPageRelevant(
  content: string,
  topic: string
): Promise<boolean> {
  const model = genAI.getGenerativeModel({ model: "gemini-2.0-flash" });

  const result = await model.generateContent(
    `Is this page about "${topic}"? Respond with only YES or NO.\n\n${content.slice(0, 2000)}`
  );

  return result.response.text().trim().toUpperCase() === "YES";
}
```

## Best Practices

1. **Trim content before sending** — Remove nav, footer, scripts to reduce tokens and cost
2. **Use the right model tier** — Flash for simple extractions, Pro for complex/long documents
3. **Always validate output** — Gemini can hallucinate data, always verify with Zod
4. **Cache extractions** — Store results in Supabase to avoid re-processing
5. **Handle rate limits** — Gemini has per-minute rate limits, add retry with backoff
6. **Chunk large documents** — Split very long content into sections for better accuracy
