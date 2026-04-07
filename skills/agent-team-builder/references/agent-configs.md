# Agent Config Templates

Complete configuration objects for all 10 agent types. Each config is passed to `BaseAgent` constructor.

## 1. Scraper Agent

```typescript
// lib/agents/scraper-agent.ts
import { z } from "zod";

const scraperConfig: AgentConfig = {
  name: "scraper",
  description: "Extracts structured data from web pages",
  capabilities: ["html-parsing", "data-extraction", "css-selectors"],
  modelId: "gemini-2.0-flash",
  systemPrompt: "See references/system-prompts.md",
  inputSchema: z.object({
    url: z.string().url("Must be a valid URL"),
    selectors: z.array(z.string()).min(1, "At least one selector required"),
    format: z.enum(["json", "csv", "markdown"]).default("json"),
  }),
  maxRetries: 3,
  rateLimitPerMinute: 30,
  costPerMillionTokens: 0.10,
};
```

## 2. Parser Agent

```typescript
// lib/agents/parser-agent.ts
import { z } from "zod";

const parserConfig: AgentConfig = {
  name: "parser",
  description: "Converts raw unstructured data into structured JSON",
  capabilities: ["text-parsing", "json-structuring", "schema-mapping"],
  modelId: "gemini-2.0-flash",
  systemPrompt: "See references/system-prompts.md",
  inputSchema: z.object({
    rawData: z.string().min(1, "Raw data cannot be empty"),
    targetSchema: z.record(z.string(), z.string()).optional(),
    dataType: z.enum(["email", "invoice", "receipt", "form", "log", "custom"]).default("custom"),
  }),
  maxRetries: 3,
  rateLimitPerMinute: 60,
  costPerMillionTokens: 0.10,
};
```

## 3. Content Agent

```typescript
// lib/agents/content-agent.ts
import { z } from "zod";

const contentConfig: AgentConfig = {
  name: "content",
  description: "Generates SEO-optimized blog posts and marketing copy",
  capabilities: ["seo-writing", "blog-generation", "copy-creation", "keyword-targeting"],
  modelId: "gemini-2.5-pro",
  systemPrompt: "See references/system-prompts.md",
  inputSchema: z.object({
    topic: z.string().min(3, "Topic must be at least 3 characters"),
    keywords: z.array(z.string()).min(1, "At least one keyword required"),
    tone: z.enum(["professional", "casual", "technical", "friendly"]).default("professional"),
    wordCount: z.number().min(100).max(5000).default(1000),
    contentType: z.enum(["blog-post", "landing-page", "email", "social-media", "product-description"]).default("blog-post"),
  }),
  maxRetries: 2,
  rateLimitPerMinute: 10,
  costPerMillionTokens: 1.25,
};
```

## 4. Analysis Agent

```typescript
// lib/agents/analysis-agent.ts
import { z } from "zod";

const analysisConfig: AgentConfig = {
  name: "analysis",
  description: "Performs calculations, predictions, and scoring on datasets",
  capabilities: ["data-analysis", "trend-detection", "scoring", "predictions"],
  modelId: "gemini-2.5-pro",
  systemPrompt: "See references/system-prompts.md",
  inputSchema: z.object({
    data: z.array(z.record(z.string(), z.unknown())).min(1, "Data array cannot be empty"),
    analysisType: z.enum(["trend", "anomaly", "scoring", "prediction", "summary"]),
    metrics: z.array(z.string()).optional(),
    timeField: z.string().optional(),
  }),
  maxRetries: 2,
  rateLimitPerMinute: 15,
  costPerMillionTokens: 1.25,
};
```

## 5. Notification Agent

```typescript
// lib/agents/notification-agent.ts
import { z } from "zod";

const notificationConfig: AgentConfig = {
  name: "notification",
  description: "Generates and manages email, push, and in-app alert content",
  capabilities: ["email-drafting", "push-notifications", "in-app-alerts", "personalization"],
  modelId: "gemini-2.0-flash",
  systemPrompt: "See references/system-prompts.md",
  inputSchema: z.object({
    channel: z.enum(["email", "push", "in-app"]),
    eventType: z.string().min(1, "Event type required"),
    recipientContext: z.object({
      name: z.string().optional(),
      preferences: z.record(z.string(), z.unknown()).optional(),
    }),
    data: z.record(z.string(), z.unknown()),
  }),
  maxRetries: 2,
  rateLimitPerMinute: 100,
  costPerMillionTokens: 0.10,
};
```

## 6. Moderation Agent

```typescript
// lib/agents/moderation-agent.ts
import { z } from "zod";

const moderationConfig: AgentConfig = {
  name: "moderation",
  description: "Detects spam, abuse, and low-quality content",
  capabilities: ["spam-detection", "abuse-detection", "quality-scoring", "content-classification"],
  modelId: "gemini-2.0-flash",
  systemPrompt: "See references/system-prompts.md",
  inputSchema: z.object({
    content: z.string().min(1, "Content cannot be empty"),
    contentType: z.enum(["comment", "review", "post", "message", "profile"]),
    context: z.object({
      userId: z.string().optional(),
      previousViolations: z.number().default(0),
    }).optional(),
  }),
  maxRetries: 1,
  rateLimitPerMinute: 120,
  costPerMillionTokens: 0.10,
};
```

## 7. Search Agent

```typescript
// lib/agents/search-agent.ts
import { z } from "zod";

const searchConfig: AgentConfig = {
  name: "search",
  description: "Intelligent search with Gemini grounded search capabilities",
  capabilities: ["semantic-search", "grounded-search", "query-expansion", "result-ranking"],
  modelId: "gemini-2.0-flash",
  systemPrompt: "See references/system-prompts.md",
  inputSchema: z.object({
    query: z.string().min(1, "Search query cannot be empty"),
    filters: z.record(z.string(), z.unknown()).optional(),
    maxResults: z.number().min(1).max(50).default(10),
    includeGrounding: z.boolean().default(true),
  }),
  maxRetries: 2,
  rateLimitPerMinute: 60,
  costPerMillionTokens: 0.10,
};
```

## 8. Report Agent

```typescript
// lib/agents/report-agent.ts
import { z } from "zod";

const reportConfig: AgentConfig = {
  name: "report",
  description: "Generates PDF reports, dashboard summaries, and data narratives",
  capabilities: ["report-generation", "data-summarization", "chart-descriptions", "executive-summaries"],
  modelId: "gemini-2.5-pro",
  systemPrompt: "See references/system-prompts.md",
  inputSchema: z.object({
    title: z.string().min(1, "Report title required"),
    data: z.record(z.string(), z.unknown()),
    reportType: z.enum(["executive-summary", "detailed", "dashboard", "weekly", "monthly"]),
    sections: z.array(z.string()).optional(),
    includeChartDescriptions: z.boolean().default(true),
  }),
  maxRetries: 2,
  rateLimitPerMinute: 5,
  costPerMillionTokens: 1.25,
};
```

## 9. Translation Agent

```typescript
// lib/agents/translation-agent.ts
import { z } from "zod";

const translationConfig: AgentConfig = {
  name: "translation",
  description: "Translates content between English and Hindi with Hinglish support",
  capabilities: ["en-to-hi", "hi-to-en", "hinglish-adaptation", "cultural-localization"],
  modelId: "gemini-2.0-flash",
  systemPrompt: "See references/system-prompts.md",
  inputSchema: z.object({
    text: z.string().min(1, "Text cannot be empty"),
    sourceLanguage: z.enum(["en", "hi", "hinglish"]),
    targetLanguage: z.enum(["en", "hi", "hinglish"]),
    tone: z.enum(["formal", "casual", "business"]).default("casual"),
    preserveFormatting: z.boolean().default(true),
  }),
  maxRetries: 2,
  rateLimitPerMinute: 60,
  costPerMillionTokens: 0.10,
};
```

## 10. Customer Support Agent

```typescript
// lib/agents/customer-support-agent.ts
import { z } from "zod";

const customerSupportConfig: AgentConfig = {
  name: "customer-support",
  description: "Answers user questions from a knowledge base with fallback escalation",
  capabilities: ["kb-search", "answer-generation", "escalation", "ticket-classification"],
  modelId: "gemini-2.5-pro",
  systemPrompt: "See references/system-prompts.md",
  inputSchema: z.object({
    question: z.string().min(3, "Question must be at least 3 characters"),
    knowledgeBase: z.array(
      z.object({
        title: z.string(),
        content: z.string(),
        category: z.string().optional(),
      })
    ).min(1, "At least one knowledge base article required"),
    conversationHistory: z.array(
      z.object({
        role: z.enum(["user", "assistant"]),
        content: z.string(),
      })
    ).optional(),
    userId: z.string().optional(),
  }),
  maxRetries: 2,
  rateLimitPerMinute: 30,
  costPerMillionTokens: 1.25,
};
```
