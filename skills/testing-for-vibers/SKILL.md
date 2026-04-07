---
name: "Testing for Vibers"
description: "Use this skill whenever the user mentions tests, testing, E2E, end-to-end, Playwright, test automation, CI/CD, GitHub Actions, smoke tests, regression tests, Lighthouse, performance testing, 'does it work', 'is it broken', 'check if everything works', test coverage, integration tests, 'before I launch', QA, quality assurance, database seeding, test data, or ANY testing/quality task — even if they don't explicitly say 'test'. This skill auto-generates tests so you never ship broken code."
---

# Testing for Vibers

You don't need to be a QA engineer to ship reliable code. This skill gives you production-ready testing patterns that catch bugs before your users do.

## Playwright Setup for Next.js

Playwright is a browser automation tool that clicks through your app like a real user would.

### Install Playwright

```bash
npm install -D @playwright/test
npx playwright install chromium
```

### Playwright Config

```typescript
// playwright.config.ts
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "./tests/e2e",
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: process.env.CI ? "github" : "html",
  timeout: 30000,
  use: {
    baseURL: process.env.BASE_URL || "http://localhost:3000",
    trace: "on-first-retry",
    screenshot: "only-on-failure",
  },
  projects: [
    {
      name: "chromium",
      use: { ...devices["Desktop Chrome"] },
    },
  ],
  webServer: {
    command: "npm run dev",
    url: "http://localhost:3000",
    reuseExistingServer: !process.env.CI,
    timeout: 120000,
  },
});
```

### Your First Test

```typescript
// tests/e2e/homepage.spec.ts
import { test, expect } from "@playwright/test";

test("homepage loads successfully", async ({ page }) => {
  const response = await page.goto("/");

  if (!response) {
    throw new Error("Failed to navigate to homepage");
  }

  expect(response.status()).toBe(200);
  await expect(page.locator("body")).toBeVisible();
});
```

Run it with:

```bash
npx playwright test
```

## Auto-Generating E2E Tests from Page Descriptions

Describe what a page does and generate tests from that description:

```typescript
// tests/e2e/auth/signup.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Signup Page", () => {
  test.beforeEach(async ({ page }) => {
    await page.goto("/signup");
  });

  test("shows signup form with all required fields", async ({ page }) => {
    await expect(page.getByLabel("Email")).toBeVisible();
    await expect(page.getByLabel("Password")).toBeVisible();
    await expect(page.getByRole("button", { name: /sign up/i })).toBeVisible();
  });

  test("shows error for invalid email", async ({ page }) => {
    await page.getByLabel("Email").fill("not-an-email");
    await page.getByLabel("Password").fill("securePassword123!");
    await page.getByRole("button", { name: /sign up/i }).click();

    await expect(page.getByText(/invalid email/i)).toBeVisible({ timeout: 5000 });
  });

  test("shows error for weak password", async ({ page }) => {
    await page.getByLabel("Email").fill("test@example.com");
    await page.getByLabel("Password").fill("123");
    await page.getByRole("button", { name: /sign up/i }).click();

    await expect(page.getByText(/password/i)).toBeVisible({ timeout: 5000 });
  });

  test("successful signup redirects to dashboard", async ({ page }) => {
    const uniqueEmail = `test-${Date.now()}@example.com`;
    await page.getByLabel("Email").fill(uniqueEmail);
    await page.getByLabel("Password").fill("SecurePassword123!");
    await page.getByRole("button", { name: /sign up/i }).click();

    await page.waitForURL("**/dashboard", { timeout: 10000 });
    await expect(page).toHaveURL(/dashboard/);
  });
});
```

## API Route Testing Patterns

Test API routes directly without a browser:

```typescript
// tests/e2e/api/health.spec.ts
import { test, expect } from "@playwright/test";

test.describe("API Health Check", () => {
  test("GET /api/health returns 200", async ({ request }) => {
    const response = await request.get("/api/health");

    expect(response.status()).toBe(200);
    const body = await response.json();
    expect(body).toHaveProperty("status", "ok");
  });
});
```

For routes that need authentication:

```typescript
// tests/e2e/api/protected-route.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Protected API Routes", () => {
  test("returns 401 without auth token", async ({ request }) => {
    const response = await request.get("/api/user/profile");
    expect(response.status()).toBe(401);
  });

  test("returns user data with valid auth", async ({ request }) => {
    // First, log in to get a session
    const loginResponse = await request.post("/api/auth/login", {
      data: {
        email: process.env.TEST_USER_EMAIL || "test@example.com",
        password: process.env.TEST_USER_PASSWORD || "testpassword123",
      },
    });

    expect(loginResponse.status()).toBe(200);

    // The cookie is automatically stored by Playwright
    const profileResponse = await request.get("/api/user/profile");
    expect(profileResponse.status()).toBe(200);

    const profile = await profileResponse.json();
    expect(profile).toHaveProperty("email");
  });
});
```

## GitHub Actions CI/CD Pipeline

This runs your tests automatically whenever someone creates a Pull Request (PR). If tests pass and you merge to `main`, it deploys to Vercel.

```yaml
# .github/workflows/test-and-deploy.yml
name: Test and Deploy

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install chromium --with-deps

      - name: Run E2E tests
        run: npx playwright test
        env:
          BASE_URL: http://localhost:3000
          NEXT_PUBLIC_SUPABASE_URL: ${{ secrets.NEXT_PUBLIC_SUPABASE_URL }}
          NEXT_PUBLIC_SUPABASE_ANON_KEY: ${{ secrets.NEXT_PUBLIC_SUPABASE_ANON_KEY }}

      - name: Upload test results
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 7

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: "--prod"
```

## Smoke Tests for Critical User Flows

Smoke tests check the most important paths in your app. If these break, nothing else matters.

```typescript
// tests/e2e/smoke/critical-flows.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Critical User Flows", () => {
  test("user can sign up, verify email, and reach dashboard", async ({ page }) => {
    // Step 1: Sign up
    await page.goto("/signup");
    const email = `smoke-${Date.now()}@example.com`;
    await page.getByLabel("Email").fill(email);
    await page.getByLabel("Password").fill("SmokeTest123!");
    await page.getByRole("button", { name: /sign up/i }).click();

    // Step 2: Check for success message or redirect
    await expect(
      page.getByText(/check your email|verify|dashboard/i)
    ).toBeVisible({ timeout: 10000 });
  });

  test("user can log in and see their profile", async ({ page }) => {
    await page.goto("/login");
    await page.getByLabel("Email").fill(process.env.TEST_USER_EMAIL || "test@example.com");
    await page.getByLabel("Password").fill(process.env.TEST_USER_PASSWORD || "testpassword123");
    await page.getByRole("button", { name: /log in|sign in/i }).click();

    await page.waitForURL("**/dashboard", { timeout: 10000 });
    await expect(page).toHaveURL(/dashboard/);
  });

  test("logged-out user is redirected from protected pages", async ({ page }) => {
    await page.goto("/dashboard");
    await page.waitForURL("**/login", { timeout: 10000 });
    await expect(page).toHaveURL(/login/);
  });
});
```

## Lighthouse CI for Performance Regression

Lighthouse checks if your pages are fast, accessible, and SEO-friendly. This GitHub Action runs Lighthouse on every PR:

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
        with:
          node-version: 20
          cache: "npm"

      - run: npm ci
      - run: npm run build

      - name: Run Lighthouse
        uses: treosh/lighthouse-ci-action@v12
        with:
          configPath: "./lighthouserc.json"
          uploadArtifacts: true
```

Create the Lighthouse config:

```json
// lighthouserc.json
{
  "ci": {
    "collect": {
      "startServerCommand": "npm start",
      "startServerReadyPattern": "ready on",
      "url": [
        "http://localhost:3000/",
        "http://localhost:3000/login",
        "http://localhost:3000/signup"
      ],
      "numberOfRuns": 3
    },
    "assert": {
      "assertions": {
        "categories:performance": ["warn", { "minScore": 0.8 }],
        "categories:accessibility": ["error", { "minScore": 0.9 }],
        "categories:seo": ["warn", { "minScore": 0.9 }]
      }
    }
  }
}
```

## Database Seeding for Test Environments

Seeding means filling your database with fake data so tests have something to work with:

```typescript
// scripts/seed-test-data.ts
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

async function seedTestData() {
  console.log("Seeding test data...");

  try {
    // Clean up previous test data
    const { error: deleteError } = await supabase
      .from("profiles")
      .delete()
      .like("email", "%@test.example.com");

    if (deleteError) {
      throw new Error(`Cleanup failed: ${deleteError.message}`);
    }

    // Create test users
    const testUsers = [
      { email: "admin@test.example.com", role: "admin", full_name: "Test Admin" },
      { email: "user@test.example.com", role: "user", full_name: "Test User" },
    ];

    for (const user of testUsers) {
      const { data, error } = await supabase.auth.admin.createUser({
        email: user.email,
        password: "TestPassword123!",
        email_confirm: true,
        user_metadata: { full_name: user.full_name, role: user.role },
      });

      if (error) {
        throw new Error(`Failed to create user ${user.email}: ${error.message}`);
      }

      console.log(`Created user: ${user.email} (ID: ${data.user.id})`);
    }

    console.log("Seeding complete.");
  } catch (err) {
    console.error("Seeding failed:", err);
    process.exit(1);
  }
}

seedTestData();
```

Run it with:

```bash
npx tsx scripts/seed-test-data.ts
```

Add a seeding step to your CI pipeline by adding this before the test step:

```yaml
      - name: Seed test database
        run: npx tsx scripts/seed-test-data.ts
        env:
          NEXT_PUBLIC_SUPABASE_URL: ${{ secrets.TEST_SUPABASE_URL }}
          SUPABASE_SERVICE_ROLE_KEY: ${{ secrets.TEST_SUPABASE_SERVICE_ROLE_KEY }}
```

## Test Organization and Naming Conventions

Keep your tests organized like this:

```
tests/
  e2e/
    smoke/              ← Critical path tests (run first)
      critical-flows.spec.ts
    auth/               ← Authentication tests
      signup.spec.ts
      login.spec.ts
      password-reset.spec.ts
    api/                ← API route tests
      health.spec.ts
      user.spec.ts
    dashboard/          ← Feature-specific tests
      settings.spec.ts
      billing.spec.ts
```

**Naming rules:**
- Files end with `.spec.ts`
- Test names describe what the user does: `"user can sign up with valid email"`
- Group related tests with `test.describe("Feature Name", ...)`
- Put the most important tests in `smoke/`

## Running Tests Locally and in CI

**Locally (during development):**

```bash
# Run all tests (no browser window)
npx playwright test

# Run with a visible browser so you can watch
npx playwright test --headed

# Run one specific test file
npx playwright test tests/e2e/auth/signup.spec.ts

# Run tests matching a name
npx playwright test -g "signup"

# Open the visual test runner
npx playwright test --ui

# See the HTML report after tests run
npx playwright show-report
```

**In CI:** Tests run automatically via the GitHub Actions workflow above. When a test fails, download the `playwright-report` artifact from the GitHub Actions run to see screenshots and traces.

## Visual Regression Testing Basics

Visual regression testing takes screenshots of your pages and compares them to previous screenshots. If something looks different, the test fails so you can check if the change was intentional.

```typescript
// tests/e2e/visual/pages.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Visual Regression", () => {
  test("homepage looks correct", async ({ page }) => {
    await page.goto("/");
    // Wait for fonts and images to load
    await page.waitForLoadState("networkidle");

    await expect(page).toHaveScreenshot("homepage.png", {
      maxDiffPixelRatio: 0.01,
      fullPage: true,
    });
  });

  test("login page looks correct", async ({ page }) => {
    await page.goto("/login");
    await page.waitForLoadState("networkidle");

    await expect(page).toHaveScreenshot("login.png", {
      maxDiffPixelRatio: 0.01,
    });
  });
});
```

The first time you run these tests, Playwright saves "golden" screenshots. On subsequent runs, it compares new screenshots to the golden ones. To update the golden screenshots after an intentional design change, run `npx playwright test --update-snapshots`.
