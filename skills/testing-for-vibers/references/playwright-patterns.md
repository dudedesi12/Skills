# Playwright Patterns Reference

## Page Navigation

```typescript
// Basic navigation
await page.goto("/dashboard");
await page.goto("https://example.com/page");

// Wait for navigation after clicking a link
await Promise.all([
  page.waitForURL("**/dashboard"),
  page.click('a[href="/dashboard"]'),
]);

// Go back/forward
await page.goBack();
await page.goForward();

// Reload
await page.reload();

// Wait for specific load state
await page.waitForLoadState("networkidle"); // all network requests done
await page.waitForLoadState("domcontentloaded"); // DOM ready
```

## Finding Elements (Locators)

```typescript
// By role (preferred — most reliable)
page.getByRole("button", { name: "Submit" });
page.getByRole("link", { name: "Dashboard" });
page.getByRole("heading", { name: "Welcome" });
page.getByRole("textbox", { name: "Email" });

// By label (for form fields)
page.getByLabel("Email");
page.getByLabel("Password");

// By placeholder
page.getByPlaceholder("Enter your email");

// By text content
page.getByText("Sign up");
page.getByText(/sign up/i); // case-insensitive regex

// By test ID (add data-testid to your HTML)
page.getByTestId("submit-button");

// CSS selector (last resort)
page.locator(".my-class");
page.locator("#my-id");
page.locator('input[type="email"]');
```

## Form Filling

```typescript
// Fill a text input (clears existing text first)
await page.getByLabel("Email").fill("user@example.com");

// Type character by character (for inputs that react to each keystroke)
await page.getByLabel("Search").pressSequentially("search term", { delay: 50 });

// Click a button
await page.getByRole("button", { name: "Submit" }).click();

// Select from a dropdown
await page.getByLabel("Country").selectOption("AU");
await page.getByLabel("Country").selectOption({ label: "Australia" });

// Check/uncheck a checkbox
await page.getByLabel("I agree to terms").check();
await page.getByLabel("I agree to terms").uncheck();

// Radio button
await page.getByLabel("Monthly").check();

// File upload
await page.getByLabel("Upload file").setInputFiles("./test-files/document.pdf");

// Clear an input
await page.getByLabel("Email").clear();

// Complete form example
async function fillSignupForm(page, email: string, password: string) {
  await page.getByLabel("Email").fill(email);
  await page.getByLabel("Password").fill(password);
  await page.getByLabel("I agree to terms").check();
  await page.getByRole("button", { name: /sign up/i }).click();
}
```

## Assertions

```typescript
// Element is visible
await expect(page.getByText("Welcome")).toBeVisible();
await expect(page.getByText("Welcome")).toBeVisible({ timeout: 10000 });

// Element is hidden
await expect(page.getByText("Loading")).toBeHidden();

// Element has specific text
await expect(page.getByRole("heading")).toHaveText("Dashboard");
await expect(page.getByRole("heading")).toContainText("Dash");

// URL checks
await expect(page).toHaveURL("/dashboard");
await expect(page).toHaveURL(/dashboard/);

// Page title
await expect(page).toHaveTitle("My App - Dashboard");

// Input value
await expect(page.getByLabel("Email")).toHaveValue("user@example.com");

// Element count
await expect(page.getByRole("listitem")).toHaveCount(5);

// Element has CSS class
await expect(page.getByTestId("alert")).toHaveClass(/error/);

// Element has attribute
await expect(page.getByRole("link")).toHaveAttribute("href", "/about");

// Element is enabled/disabled
await expect(page.getByRole("button")).toBeEnabled();
await expect(page.getByRole("button")).toBeDisabled();

// Negate any assertion with .not
await expect(page.getByText("Error")).not.toBeVisible();
```

## Auth State Reuse

Log in once, save the session, and reuse it across tests so you don't waste time logging in for every test:

```typescript
// tests/e2e/auth.setup.ts
import { test as setup, expect } from "@playwright/test";
import path from "path";

const authFile = path.join(__dirname, "../.auth/user.json");

setup("authenticate", async ({ page }) => {
  await page.goto("/login");
  await page.getByLabel("Email").fill(process.env.TEST_USER_EMAIL || "test@example.com");
  await page.getByLabel("Password").fill(process.env.TEST_USER_PASSWORD || "testpassword123");
  await page.getByRole("button", { name: /log in|sign in/i }).click();

  await page.waitForURL("**/dashboard", { timeout: 15000 });
  await expect(page).toHaveURL(/dashboard/);

  // Save the logged-in state to a file
  await page.context().storageState({ path: authFile });
});
```

Update `playwright.config.ts` to use the saved auth state:

```typescript
// playwright.config.ts (projects section)
projects: [
  // Setup project runs first and saves auth state
  { name: "setup", testMatch: /.*\.setup\.ts/ },

  // All other tests reuse the saved auth state
  {
    name: "chromium",
    use: {
      ...devices["Desktop Chrome"],
      storageState: "tests/.auth/user.json",
    },
    dependencies: ["setup"],
  },
],
```

Add `tests/.auth/` to your `.gitignore` so auth tokens are never committed.

## Fixtures (Reusable Test Helpers)

Fixtures let you create reusable setup code. Think of them as helper functions that run before each test:

```typescript
// tests/e2e/fixtures.ts
import { test as base, expect } from "@playwright/test";

type TestFixtures = {
  authenticatedPage: ReturnType<typeof base["page"]>;
  testUser: { email: string; password: string };
};

export const test = base.extend<TestFixtures>({
  testUser: async ({}, use) => {
    await use({
      email: `test-${Date.now()}@example.com`,
      password: "TestPassword123!",
    });
  },

  authenticatedPage: async ({ page }, use) => {
    await page.goto("/login");
    await page.getByLabel("Email").fill("test@example.com");
    await page.getByLabel("Password").fill("testpassword123");
    await page.getByRole("button", { name: /sign in/i }).click();
    await page.waitForURL("**/dashboard");
    await use(page);
  },
});

export { expect };
```

Use the fixture in tests:

```typescript
// tests/e2e/dashboard/settings.spec.ts
import { test, expect } from "../fixtures";

test("user can update their name", async ({ authenticatedPage }) => {
  await authenticatedPage.goto("/dashboard/settings");
  await authenticatedPage.getByLabel("Full Name").fill("New Name");
  await authenticatedPage.getByRole("button", { name: "Save" }).click();
  await expect(authenticatedPage.getByText("Settings saved")).toBeVisible();
});
```

## Parallel Tests

By default, Playwright runs test files in parallel. Each file gets its own browser. Tests within the same file run sequentially.

```typescript
// Force sequential execution for tests that share state
test.describe.serial("Checkout Flow", () => {
  test("add item to cart", async ({ page }) => {
    // ...
  });

  test("proceed to checkout", async ({ page }) => {
    // ...
  });

  test("complete payment", async ({ page }) => {
    // ...
  });
});

// Run tests within a file in parallel
test.describe.parallel("Independent Tests", () => {
  test("test A", async ({ page }) => { /* ... */ });
  test("test B", async ({ page }) => { /* ... */ });
});
```

Control parallelism in config:

```typescript
// playwright.config.ts
export default defineConfig({
  fullyParallel: true,  // Run all tests in parallel
  workers: 4,           // Use 4 parallel workers
  // workers: process.env.CI ? 1 : undefined, // 1 in CI, auto locally
});
```

## Screenshot Comparison

```typescript
// Take a full-page screenshot and compare to baseline
await expect(page).toHaveScreenshot("dashboard.png", {
  fullPage: true,
  maxDiffPixelRatio: 0.01,
});

// Screenshot a specific element
const card = page.getByTestId("pricing-card");
await expect(card).toHaveScreenshot("pricing-card.png");

// Mask dynamic content (like dates or avatars)
await expect(page).toHaveScreenshot("profile.png", {
  mask: [page.getByTestId("timestamp"), page.getByTestId("avatar")],
});

// Update baseline screenshots when design changes intentionally
// Run: npx playwright test --update-snapshots
```

## Waiting for Things

```typescript
// Wait for an element to appear
await page.waitForSelector(".loading-spinner", { state: "hidden" });

// Wait for a network request to finish
await page.waitForResponse("**/api/data");

// Wait for a specific API call
const responsePromise = page.waitForResponse(
  (response) => response.url().includes("/api/users") && response.status() === 200
);
await page.getByRole("button", { name: "Load Users" }).click();
const response = await responsePromise;
const data = await response.json();

// Wait for navigation
await page.waitForURL("**/success");

// Wait a fixed amount of time (use sparingly)
await page.waitForTimeout(1000);
```

## Network Interception (Mocking API Responses)

```typescript
// Mock an API response
await page.route("**/api/users", async (route) => {
  await route.fulfill({
    status: 200,
    contentType: "application/json",
    body: JSON.stringify([
      { id: 1, name: "Test User", email: "test@example.com" },
    ]),
  });
});

// Simulate a network error
await page.route("**/api/data", async (route) => {
  await route.abort("failed");
});

// Simulate slow network
await page.route("**/api/slow", async (route) => {
  await new Promise((resolve) => setTimeout(resolve, 3000));
  await route.continue();
});

// Intercept and modify a request
await page.route("**/api/data", async (route) => {
  const response = await route.fetch();
  const json = await response.json();
  json.modified = true;
  await route.fulfill({ response, json });
});
```

## Multi-Page and Multi-Tab Tests

```typescript
// Handle a new tab opening
const [newPage] = await Promise.all([
  page.context().waitForEvent("page"),
  page.getByRole("link", { name: "Open in new tab" }).click(),
]);
await newPage.waitForLoadState();
await expect(newPage).toHaveURL(/external-page/);

// Work with multiple browser contexts (like incognito)
const context2 = await page.context().browser()!.newContext();
const page2 = await context2.newPage();
await page2.goto("/admin");
await context2.close();
```
