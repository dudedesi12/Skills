# axe-core Setup for Automated Accessibility Testing

## Installation

```bash
npm install -D @axe-core/playwright @playwright/test
```

## Basic Playwright Integration

```typescript
// tests/e2e/accessibility/a11y.spec.ts
import { test, expect } from "@playwright/test";
import AxeBuilder from "@axe-core/playwright";

test.describe("Accessibility Checks", () => {
  test("homepage passes WCAG AA", async ({ page }) => {
    await page.goto("/");
    await page.waitForLoadState("networkidle");

    const results = await new AxeBuilder({ page })
      .withTags(["wcag2a", "wcag2aa", "wcag22aa"])
      .analyze();

    // Log violations for debugging
    if (results.violations.length > 0) {
      console.log("Accessibility violations:");
      results.violations.forEach((v) => {
        console.log(`  ${v.id}: ${v.description}`);
        console.log(`  Impact: ${v.impact}`);
        console.log(`  Elements: ${v.nodes.map((n) => n.html).join(", ")}`);
      });
    }

    expect(results.violations).toEqual([]);
  });
});
```

## Testing Multiple Pages

```typescript
// tests/e2e/accessibility/full-audit.spec.ts
import { test, expect } from "@playwright/test";
import AxeBuilder from "@axe-core/playwright";

const PUBLIC_PAGES = ["/", "/login", "/signup", "/about", "/pricing", "/privacy"];
const AUTH_PAGES = ["/dashboard", "/assessment", "/settings", "/profile"];

test.describe("Public Pages Accessibility", () => {
  for (const path of PUBLIC_PAGES) {
    test(`${path} has no WCAG AA violations`, async ({ page }) => {
      await page.goto(path);
      await page.waitForLoadState("networkidle");

      const results = await new AxeBuilder({ page })
        .withTags(["wcag2a", "wcag2aa"])
        .analyze();

      expect(results.violations).toEqual([]);
    });
  }
});

test.describe("Authenticated Pages Accessibility", () => {
  test.beforeEach(async ({ page }) => {
    // Log in first
    await page.goto("/login");
    await page.getByLabel("Email").fill(process.env.TEST_USER_EMAIL || "test@example.com");
    await page.getByLabel("Password").fill(process.env.TEST_USER_PASSWORD || "testpassword123");
    await page.getByRole("button", { name: /log in|sign in/i }).click();
    await page.waitForURL("**/dashboard", { timeout: 10000 });
  });

  for (const path of AUTH_PAGES) {
    test(`${path} has no WCAG AA violations`, async ({ page }) => {
      await page.goto(path);
      await page.waitForLoadState("networkidle");

      const results = await new AxeBuilder({ page })
        .withTags(["wcag2a", "wcag2aa"])
        .analyze();

      expect(results.violations).toEqual([]);
    });
  }
});
```

## Excluding Known Issues

Sometimes third-party components have issues you can't fix immediately. Exclude specific rules while you work on fixes:

```typescript
const results = await new AxeBuilder({ page })
  .withTags(["wcag2a", "wcag2aa"])
  .disableRules(["color-contrast"]) // Temporary — remove after fixing
  .exclude(".third-party-widget") // Exclude elements you don't control
  .analyze();
```

## Testing Component States

Test interactive components in different states:

```typescript
test("modal is accessible when open", async ({ page }) => {
  await page.goto("/dashboard");
  await page.getByRole("button", { name: "New Assessment" }).click();

  // Wait for modal to appear
  await expect(page.getByRole("dialog")).toBeVisible();

  const results = await new AxeBuilder({ page })
    .include('[role="dialog"]') // Only test the modal
    .withTags(["wcag2a", "wcag2aa"])
    .analyze();

  expect(results.violations).toEqual([]);
});

test("form shows accessible error states", async ({ page }) => {
  await page.goto("/signup");
  await page.getByRole("button", { name: /sign up/i }).click(); // Submit empty form

  // Wait for error messages
  await expect(page.getByRole("alert")).toBeVisible();

  const results = await new AxeBuilder({ page })
    .withTags(["wcag2a", "wcag2aa"])
    .analyze();

  expect(results.violations).toEqual([]);
});
```

## GitHub Actions Integration

```yaml
# .github/workflows/accessibility.yml
name: Accessibility Tests

on:
  pull_request:
    branches: [main]

jobs:
  a11y:
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "npm"

      - run: npm ci
      - run: npx playwright install chromium --with-deps

      - name: Run accessibility tests
        run: npx playwright test tests/e2e/accessibility/
        env:
          BASE_URL: http://localhost:3000
          NEXT_PUBLIC_SUPABASE_URL: ${{ secrets.NEXT_PUBLIC_SUPABASE_URL }}
          NEXT_PUBLIC_SUPABASE_ANON_KEY: ${{ secrets.NEXT_PUBLIC_SUPABASE_ANON_KEY }}

      - name: Upload report
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: a11y-report
          path: playwright-report/
          retention-days: 7
```

## Custom Reporter for Accessibility Results

```typescript
// tests/e2e/accessibility/helpers.ts
import type { Result } from "axe-core";

export function formatViolations(violations: Result[]): string {
  if (violations.length === 0) return "No violations found.";

  return violations
    .map((v) => {
      const nodes = v.nodes
        .map((n) => `    - ${n.html}\n      Fix: ${n.failureSummary}`)
        .join("\n");
      return `[${v.impact?.toUpperCase()}] ${v.id}: ${v.description}\n${nodes}`;
    })
    .join("\n\n");
}
```

## Development-Time Checking

For instant feedback during development, add axe-core to the browser:

```typescript
// lib/a11y-dev.ts — Only import in development
export async function initA11yDevTools() {
  if (process.env.NODE_ENV !== "development") return;

  const axe = await import("@axe-core/react");
  const React = await import("react");
  const ReactDOM = await import("react-dom");
  axe.default(React.default, ReactDOM.default, 1000); // Check every 1 second
}
```

```tsx
// app/layout.tsx — Client component wrapper for dev only
"use client";
import { useEffect } from "react";

export function A11yDevChecker() {
  useEffect(() => {
    if (process.env.NODE_ENV === "development") {
      import("@/lib/a11y-dev").then((m) => m.initA11yDevTools());
    }
  }, []);
  return null;
}
```

This logs accessibility violations directly to the browser console as you develop.
