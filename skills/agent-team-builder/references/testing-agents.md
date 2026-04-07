# Testing Agents

How to test each agent type with curl commands, sample payloads, and automated test patterns.

## Prerequisites

Start your dev server:

```bash
npm run dev
```

Set your base URL:

```bash
BASE_URL="http://localhost:3000"
```

## 1. Scraper Agent

```bash
curl -X POST "$BASE_URL/api/agents/scraper" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example.com/products",
    "selectors": ["title", "price", "description"],
    "format": "json"
  }'
```

**Expected response:**
```json
{
  "success": true,
  "data": {
    "url": "https://example.com/products",
    "extractedData": [{ "title": "...", "price": "...", "description": "..." }],
    "fieldCount": 3,
    "scrapedAt": "2026-04-07T..."
  },
  "tokenUsage": 450,
  "costUsd": 0.000045,
  "taskId": "uuid-here"
}
```

## 2. Parser Agent

```bash
curl -X POST "$BASE_URL/api/agents/parser" \
  -H "Content-Type: application/json" \
  -d '{
    "rawData": "Invoice #1234\nDate: March 15, 2026\nTotal: Rs. 5,499.00\nItem: Premium Plan (Annual)\nCustomer: Rahul Sharma, rahul@example.com",
    "dataType": "invoice"
  }'
```

## 3. Content Agent

```bash
curl -X POST "$BASE_URL/api/agents/content" \
  -H "Content-Type: application/json" \
  -d '{
    "topic": "How to build a SaaS MVP in 30 days",
    "keywords": ["SaaS MVP", "startup", "product launch", "minimum viable product"],
    "tone": "professional",
    "wordCount": 1500,
    "contentType": "blog-post"
  }'
```

## 4. Analysis Agent

```bash
curl -X POST "$BASE_URL/api/agents/analysis" \
  -H "Content-Type: application/json" \
  -d '{
    "data": [
      { "month": "Jan", "revenue": 12000, "users": 150 },
      { "month": "Feb", "revenue": 15000, "users": 200 },
      { "month": "Mar", "revenue": 13500, "users": 180 }
    ],
    "analysisType": "trend",
    "metrics": ["revenue", "users"],
    "timeField": "month"
  }'
```

## 5. Notification Agent

```bash
curl -X POST "$BASE_URL/api/agents/notification" \
  -H "Content-Type: application/json" \
  -d '{
    "channel": "email",
    "eventType": "subscription-expiring",
    "recipientContext": {
      "name": "Priya",
      "preferences": { "language": "en" }
    },
    "data": {
      "planName": "Pro",
      "expiresIn": "3 days",
      "renewalUrl": "/billing/renew"
    }
  }'
```

## 6. Moderation Agent

```bash
curl -X POST "$BASE_URL/api/agents/moderation" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "This product is absolutely terrible! Worst purchase ever. The company is run by scammers!!!",
    "contentType": "review",
    "context": { "userId": "user-123", "previousViolations": 0 }
  }'
```

## 7. Search Agent

```bash
curl -X POST "$BASE_URL/api/agents/search" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "how to reset my password",
    "maxResults": 5,
    "includeGrounding": true
  }'
```

## 8. Report Agent

```bash
curl -X POST "$BASE_URL/api/agents/report" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Monthly Revenue Report - March 2026",
    "data": {
      "totalRevenue": 150000,
      "newCustomers": 45,
      "churnRate": 2.1,
      "topPlan": "Pro",
      "mrr": 148500
    },
    "reportType": "monthly",
    "sections": ["overview", "revenue", "customers", "churn"],
    "includeChartDescriptions": true
  }'
```

## 9. Translation Agent

```bash
curl -X POST "$BASE_URL/api/agents/translation" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Welcome to our platform! Your account has been created successfully. You can start exploring features right away.",
    "sourceLanguage": "en",
    "targetLanguage": "hinglish",
    "tone": "casual",
    "preserveFormatting": true
  }'
```

## 10. Customer Support Agent

```bash
curl -X POST "$BASE_URL/api/agents/customer-support" \
  -H "Content-Type: application/json" \
  -d '{
    "question": "How do I upgrade my plan?",
    "knowledgeBase": [
      {
        "title": "Billing & Plans",
        "content": "To upgrade your plan, go to Settings > Billing > Change Plan. Select your new plan and confirm. Changes take effect immediately. You will be charged the prorated difference.",
        "category": "billing"
      },
      {
        "title": "Account Settings",
        "content": "Manage your profile, notifications, and security settings from the Settings page.",
        "category": "account"
      }
    ],
    "userId": "user-456"
  }'
```

## Health Check (Any Agent)

```bash
curl "$BASE_URL/api/agents/scraper/health"
curl "$BASE_URL/api/agents/content/health"
curl "$BASE_URL/api/agents/moderation/health"
```

## Task Status Lookup

After any agent call, use the returned `taskId`:

```bash
curl "$BASE_URL/api/agents/tasks/YOUR_TASK_ID_HERE"
```

## Testing Version Routing

Send requests with a specific version header:

```bash
# Force v2
curl -X POST "$BASE_URL/api/agents/content" \
  -H "Content-Type: application/json" \
  -H "x-agent-version: v2" \
  -d '{ "topic": "test", "keywords": ["test"], "wordCount": 100, "contentType": "blog-post" }'
```

## Automated Test File

```typescript
// __tests__/agents/scraper.test.ts
import { ScraperAgent } from "@/lib/agents/scraper-agent";

describe("ScraperAgent", () => {
  const agent = new ScraperAgent();

  it("rejects invalid input", async () => {
    const result = await agent.run({ url: "not-a-url", selectors: [] });
    expect(result.success).toBe(false);
    expect(result.error).toContain("Invalid input");
  });

  it("accepts valid input", async () => {
    const result = await agent.run({
      url: "https://example.com",
      selectors: ["title"],
      format: "json",
    });
    // Result depends on Gemini API availability
    expect(result.taskId).toBeDefined();
  });
});
```

## Error Scenarios to Test

| Scenario | How to Trigger | Expected Result |
|----------|---------------|-----------------|
| Invalid input | Omit required fields | 422 with validation error |
| Unknown agent | POST to `/api/agents/fake` | 404 with available agents list |
| Empty body | Send no JSON body | 400 with "must be valid JSON" |
| Rate limit | Send 100+ requests in 1 minute | 429 with retry-after header |
| Budget exceeded | Set daily limit to $0.001 in DB | 422 with budget error |
