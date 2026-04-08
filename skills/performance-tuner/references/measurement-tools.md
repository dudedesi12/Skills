# Performance Measurement Tools

## Lighthouse CI

Run Lighthouse automatically on every PR.

```bash
npm install -D @lhci/cli
```

```json
// lighthouserc.json
{
  "ci": {
    "collect": {
      "startServerCommand": "npm start",
      "startServerReadyPattern": "ready on",
      "url": ["http://localhost:3000/", "http://localhost:3000/login"],
      "numberOfRuns": 3
    },
    "assert": {
      "assertions": {
        "categories:performance": ["warn", { "minScore": 0.8 }],
        "categories:accessibility": ["error", { "minScore": 0.9 }],
        "categories:seo": ["warn", { "minScore": 0.9 }],
        "first-contentful-paint": ["warn", { "maxNumericValue": 2000 }],
        "largest-contentful-paint": ["warn", { "maxNumericValue": 2500 }],
        "cumulative-layout-shift": ["warn", { "maxNumericValue": 0.1 }]
      }
    }
  }
}
```

```yaml
# .github/workflows/lighthouse.yml
name: Lighthouse CI
on:
  pull_request:
    branches: [main]

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - run: npm run build
      - run: npx @lhci/cli autorun
```

## web-vitals Library

### API Route for Collecting Vitals

```typescript
// app/api/vitals/route.ts
import { NextRequest, NextResponse } from "next/server";

export async function POST(request: NextRequest) {
  const body = await request.json();
  const { name, value } = body;

  // Log to console in development
  console.log(`[Web Vital] ${name}: ${value}`);

  // In production, send to your analytics or monitoring service
  // await sendToAnalytics({ name, value, timestamp: Date.now() });

  return NextResponse.json({ received: true });
}
```

### PerformanceObserver for Custom Metrics

```typescript
// lib/custom-metrics.ts
"use client";

export function observeLongTasks() {
  if (typeof PerformanceObserver === "undefined") return;

  const observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      if (entry.duration > 50) { // Tasks longer than 50ms
        console.warn(`Long task detected: ${entry.duration}ms`, entry);
      }
    }
  });

  observer.observe({ type: "longtask", buffered: true });
}

export function measureComponentRender(name: string) {
  const start = performance.now();
  return () => {
    const duration = performance.now() - start;
    if (duration > 16) { // Longer than one frame (60fps)
      console.warn(`Slow render: ${name} took ${duration.toFixed(1)}ms`);
    }
  };
}
```

## Chrome DevTools Performance Tab

### Recording a Performance Trace

1. Open Chrome DevTools (F12)
2. Go to the **Performance** tab
3. Click the record button (circle icon)
4. Interact with your page (navigate, click, scroll)
5. Click stop
6. Analyze the flame chart

### What to Look For

- **Long Tasks** (red bars) — JavaScript blocking the main thread > 50ms
- **Layout Shifts** — Pink/purple bars indicating CLS events
- **Paint events** — How often the browser repaints
- **Network waterfall** — Are resources loading in parallel or sequentially?

## Performance Budget Monitoring

```typescript
// scripts/check-bundle-size.ts
import { readFileSync, readdirSync, statSync } from "fs";
import { join } from "path";

const BUDGET = {
  totalJs: 300 * 1024,    // 300KB total JS
  singleChunk: 100 * 1024, // 100KB per chunk
  totalCss: 50 * 1024,     // 50KB total CSS
};

function getFileSize(dir: string, ext: string): number {
  let total = 0;
  const files = readdirSync(dir, { recursive: true });
  for (const file of files) {
    const filePath = join(dir, file.toString());
    if (filePath.endsWith(ext) && statSync(filePath).isFile()) {
      total += statSync(filePath).size;
    }
  }
  return total;
}

const jsSize = getFileSize(".next/static", ".js");
const cssSize = getFileSize(".next/static", ".css");

console.log(`JS: ${(jsSize / 1024).toFixed(1)}KB / ${(BUDGET.totalJs / 1024).toFixed(0)}KB`);
console.log(`CSS: ${(cssSize / 1024).toFixed(1)}KB / ${(BUDGET.totalCss / 1024).toFixed(0)}KB`);

if (jsSize > BUDGET.totalJs) {
  console.error("JS budget exceeded!");
  process.exit(1);
}
if (cssSize > BUDGET.totalCss) {
  console.error("CSS budget exceeded!");
  process.exit(1);
}
console.log("All budgets within limits.");
```
