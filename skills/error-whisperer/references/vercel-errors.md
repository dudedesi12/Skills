# Vercel Deployment Errors (20+)

Every common Vercel error with plain English explanation and exact fix.

## Build Errors

### 1. Error: Build failed — Module not found

**Why:** A dependency is missing or the import path is wrong. Vercel uses a fresh install, so anything not in `package.json` will not be there.

**Fix:**
```bash
# Make sure the dependency is in package.json (not just locally installed)
npm install the-missing-package
# Commit package.json AND package-lock.json
git add package.json package-lock.json
git commit -m "fix: add missing dependency"
```

### 2. Error: Build failed — case sensitivity

**Why:** Your Mac or Windows treats `File.tsx` and `file.tsx` as the same file. Linux (Vercel) does not.

**Fix:** Rename the file to match the import exactly:
```bash
# Check the import
import { Header } from "@/components/Header"; # Capital H
# File must be: components/Header.tsx (not header.tsx)
git mv components/header.tsx components/Header.tsx
```

### 3. Error: Function exceeded 50MB size limit

**Why:** Your serverless function bundle is too large. Usually caused by importing heavy libraries.

**Fix:**
```typescript
// Use dynamic imports for heavy libraries
const pdf = await import("pdf-lib");

// Or move to Edge runtime for smaller bundles
export const runtime = "edge";
```

### 4. Error: No output directory "X" found after build

**Why:** The build output directory does not match what Vercel expects.

**Fix:** In Vercel project settings > Build & Development Settings, set the output directory. For Next.js, it should be `.next`.

### 5. Error: Command "build" not found

**Why:** Missing build script in `package.json`.

**Fix:**
```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  }
}
```

### 6. Error: TypeScript compilation errors

**Why:** Your code has type errors. Next.js by default does not fail on type errors in dev mode, but build mode is strict.

**Fix:** Run `npm run build` locally first to catch all type errors. Fix them before deploying.

### 7. Error: ESLint errors during build

**Why:** ESLint rules are failing during the production build.

**Fix:** Fix the lint errors, or (not recommended) ignore during build:
```typescript
// next.config.js — only as a temporary measure
const nextConfig = { eslint: { ignoreDuringBuilds: true } };
```

## Runtime Errors

### 8. Error: FUNCTION_INVOCATION_TIMEOUT (10s or 60s limit)

**Why:** Your serverless function took too long. Free plan: 10s, Pro: 60s.

**Fix:**
```typescript
// Option 1: Optimize the function
// Option 2: Switch to Edge runtime (no timeout but CPU limits)
export const runtime = "edge";
// Option 3: Use streaming for long operations
export async function GET() {
  const stream = new ReadableStream({ /* ... */ });
  return new Response(stream);
}
```

### 9. Error: FUNCTION_INVOCATION_FAILED

**Why:** Your function crashed at runtime. Could be an unhandled error, missing env var, or import error.

**Fix:** Check Vercel logs: Dashboard > Project > Logs > Functions. The error will be in the log output.

### 10. Error: EDGE_FUNCTION_INVOCATION_FAILED

**Why:** Edge function crashed. Edge has limitations — no Node.js APIs, no fs, limited packages.

**Fix:** Make sure you only use Web Standard APIs in Edge functions. No `fs`, `path`, `crypto` (use `crypto.subtle` instead).

### 11. Error: 504 Gateway Timeout

**Why:** The function or upstream API took too long to respond.

**Fix:** Add timeouts to external API calls:
```typescript
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 8000);
const res = await fetch(url, { signal: controller.signal });
clearTimeout(timeout);
```

### 12. Error: 500 Internal Server Error (no details)

**Why:** The function threw an error but it was not caught.

**Fix:** Add error handling and check Vercel function logs:
```typescript
export async function GET() {
  try {
    const data = await fetchData();
    return Response.json(data);
  } catch (error) {
    console.error("API error:", error);
    return Response.json({ error: "Internal server error" }, { status: 500 });
  }
}
```

## Environment Variable Errors

### 13. Environment variable undefined in production

**Why:** The env var is in `.env.local` (local only) but not configured in Vercel.

**Fix:** Add it in Vercel Dashboard > Project > Settings > Environment Variables. Make sure the right environments are checked (Production, Preview, Development).

### 14. NEXT_PUBLIC_ variable not available on client

**Why:** You added the env var to Vercel but forgot the `NEXT_PUBLIC_` prefix for client-side access.

**Fix:** In Vercel, rename the variable with the `NEXT_PUBLIC_` prefix. Redeploy after changing.

### 15. Environment variables changed but not taking effect

**Why:** Env vars are baked in at build time. Changing them requires a redeploy.

**Fix:** After changing env vars in Vercel dashboard, trigger a new deployment.

## Domain and DNS Errors

### 16. Error: DNS_PROBE_FINISHED_NXDOMAIN

**Why:** Your custom domain's DNS is not pointing to Vercel.

**Fix:** Add the DNS records shown in Vercel Dashboard > Project > Settings > Domains:
- A record: `76.76.21.21`
- Or CNAME: `cname.vercel-dns.com`

### 17. Error: SSL certificate pending

**Why:** Vercel needs DNS to propagate before it can issue an SSL certificate.

**Fix:** Wait 5-30 minutes after configuring DNS. If it takes longer, check that DNS records are correct.

### 18. Error: Domain is already in use by another Vercel project

**Why:** The domain is configured on a different Vercel project.

**Fix:** Remove it from the other project first: Other Project > Settings > Domains > Remove.

## Deployment Config Errors

### 19. Error: Root directory setting is incorrect

**Why:** Vercel is looking for the project in the wrong folder (common in monorepos).

**Fix:** In Vercel Dashboard > Project > Settings > Root Directory, set the path to your Next.js app.

### 20. Error: Node.js version mismatch

**Why:** Your project requires a Node.js version that Vercel is not using.

**Fix:** Set the Node.js version in Vercel project settings or in `package.json`:
```json
{ "engines": { "node": ">=20.0.0" } }
```

### 21. Error: Build cache is stale

**Why:** Cached dependencies or build artifacts are causing issues.

**Fix:** Trigger a fresh deploy: Vercel Dashboard > Deployments > Redeploy > Toggle "Override Build Cache."

### 22. Error: Cron job not triggering

**Why:** Cron configuration is missing or incorrect.

**Fix:**
```json
// vercel.json
{
  "crons": [
    { "path": "/api/cron", "schedule": "0 */6 * * *" }
  ]
}
```
Make sure the API route exists and returns a 200 status code.

### 23. Preview deployments not working

**Why:** Preview deployments only trigger for branches pushed to the connected Git provider.

**Fix:** Make sure the branch is pushed to GitHub, and the Vercel project is connected to the repo.

### 24. Error: Serverless function cold start too slow

**Why:** Not an error per se — serverless functions take 200-500ms to start after being idle.

**Fix:**
```typescript
// Option 1: Switch to Edge runtime (near-instant cold starts)
export const runtime = "edge";
// Option 2: Use Vercel's cron to keep functions warm
// vercel.json
{ "crons": [{ "path": "/api/health", "schedule": "*/5 * * * *" }] }
```
