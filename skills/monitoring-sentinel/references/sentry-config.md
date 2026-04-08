# Sentry Configuration for Next.js App Router

## Installation

```bash
npx @sentry/wizard@latest -i nextjs
```

This will:
1. Create `sentry.client.config.ts`, `sentry.server.config.ts`, `sentry.edge.config.ts`
2. Wrap `next.config.ts` with `withSentryConfig`
3. Create `instrumentation.ts`
4. Add `.env.sentry-build-plugin` for source maps

## Environment Variables

```bash
# .env.local
NEXT_PUBLIC_SENTRY_DSN=https://xxx@xxx.ingest.sentry.io/xxx
SENTRY_DSN=https://xxx@xxx.ingest.sentry.io/xxx
SENTRY_AUTH_TOKEN=sntrys_xxx  # For source maps upload
SENTRY_ORG=your-org
SENTRY_PROJECT=your-project
```

Add these to Vercel as well: Settings → Environment Variables.

## Client Configuration

```typescript
// sentry.client.config.ts
import * as Sentry from "@sentry/nextjs";

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  environment: process.env.NEXT_PUBLIC_VERCEL_ENV ?? "development",
  enabled: process.env.NODE_ENV === "production",

  // Performance monitoring
  tracesSampleRate: 0.1, // Sample 10% of transactions

  // Session replay — record user sessions on errors
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
  integrations: [
    Sentry.replayIntegration({
      maskAllText: true,        // Mask all text for privacy
      blockAllMedia: true,      // Block media for privacy
    }),
  ],

  // Filter out noise
  ignoreErrors: [
    "ResizeObserver loop limit exceeded",
    "Non-Error promise rejection captured",
    /Loading chunk \d+ failed/,
  ],

  beforeSend(event) {
    // Don't send errors in development
    if (process.env.NODE_ENV === "development") return null;
    return event;
  },
});
```

## Server Configuration

```typescript
// sentry.server.config.ts
import * as Sentry from "@sentry/nextjs";

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.VERCEL_ENV ?? "development",
  enabled: process.env.NODE_ENV === "production",
  tracesSampleRate: 0.1,
});
```

## Error Boundary Integration

```tsx
// app/global-error.tsx — catches errors in the root layout
"use client";
import * as Sentry from "@sentry/nextjs";
import { useEffect } from "react";

export default function GlobalError({ error, reset }: { error: Error; reset: () => void }) {
  useEffect(() => { Sentry.captureException(error); }, [error]);

  return (
    <html>
      <body>
        <div className="flex min-h-screen items-center justify-center">
          <div className="text-center">
            <h1 className="text-2xl font-bold">Something went wrong</h1>
            <button onClick={reset} className="mt-4 rounded bg-blue-600 px-4 py-2 text-white">
              Try again
            </button>
          </div>
        </div>
      </body>
    </html>
  );
}
```

## Manual Error Capture

```typescript
// In server actions or API routes
import * as Sentry from "@sentry/nextjs";

try {
  await riskyOperation();
} catch (error) {
  Sentry.captureException(error, {
    tags: { feature: "assessment", severity: "high" },
    extra: { userId: user.id, assessmentId },
  });
  throw error; // Re-throw after capturing
}
```

## Source Maps

Source maps let Sentry show you the original TypeScript code in error reports.

```typescript
// next.config.ts
import { withSentryConfig } from "@sentry/nextjs";

const nextConfig = { /* your config */ };

export default withSentryConfig(nextConfig, {
  org: process.env.SENTRY_ORG,
  project: process.env.SENTRY_PROJECT,
  silent: true, // Don't print upload logs
  hideSourceMaps: true, // Don't expose source maps to users
});
```
