# Dev vs Production Differences

Common differences that cause "works locally but not in production" issues.

## File System

### 1. Case Sensitivity

**Local (Mac/Windows):** `Header.tsx` and `header.tsx` are the SAME file.
**Production (Linux/Vercel):** `Header.tsx` and `header.tsx` are DIFFERENT files.

**Symptom:** `Module not found` error only in production.

**Fix:**
```bash
# Check the exact import
import { Header } from "@/components/Header"; # Must match exactly
# Rename if needed
git mv components/header.tsx components/Header.tsx
```

### 2. File Path Length

**Local:** Usually no issues.
**Production:** Some systems have path length limits.

**Fix:** Keep folder nesting reasonable. Avoid deeply nested routes like `app/a/b/c/d/e/f/page.tsx`.

## Environment Variables

### 3. Missing Env Vars

**Local:** `.env.local` has all your variables.
**Production:** Env vars must be set in Vercel dashboard separately.

**Symptom:** Features work locally, return undefined/null in production.

**Fix:** Add every env var from `.env.local` to Vercel > Project > Settings > Environment Variables.

### 4. NEXT_PUBLIC_ Prefix

**Local:** Both prefixed and non-prefixed env vars might seem to work due to dev server behavior.
**Production:** Only `NEXT_PUBLIC_` prefixed vars are available on the client.

**Symptom:** Client-side code cannot read the env var in production.

**Fix:** Add `NEXT_PUBLIC_` prefix for any variable needed in the browser.

### 5. Env Var Values Differ

**Local:** `NEXT_PUBLIC_SUPABASE_URL` points to local or dev project.
**Production:** Should point to production Supabase project.

**Fix:** Use different Supabase projects for dev and production. Set the right URLs in each environment.

## Networking

### 6. Localhost URLs

**Local:** `http://localhost:3000/api/data` works.
**Production:** Localhost does not exist.

**Symptom:** API calls fail silently or return HTML error pages.

**Fix:**
```typescript
// Wrong
const res = await fetch("http://localhost:3000/api/data");

// Right — use relative URL
const res = await fetch("/api/data");

// Right — use env var for external services
const res = await fetch(`${process.env.NEXT_PUBLIC_APP_URL}/api/data`);
```

### 7. HTTP vs HTTPS

**Local:** Dev server runs on HTTP.
**Production:** Vercel serves over HTTPS.

**Symptom:** Mixed content warnings, cookies not being set (Secure flag).

**Fix:** Always use HTTPS URLs for external resources. Set cookie `Secure` flag conditionally:
```typescript
const isProduction = process.env.NODE_ENV === "production";
```

### 8. CORS Differences

**Local:** Same-origin requests (localhost to localhost) skip CORS.
**Production:** Cross-origin requests need CORS headers.

**Symptom:** API calls work locally but get CORS errors in production.

**Fix:** Use Next.js API routes as proxies for external APIs, or set CORS headers.

## Runtime

### 9. Node.js Version

**Local:** Whatever version you installed.
**Production:** Vercel uses a specific Node.js version.

**Symptom:** Syntax errors or missing APIs in production.

**Fix:** Match your local Node.js version to Vercel's. Set in `package.json`:
```json
{ "engines": { "node": ">=20.0.0" } }
```

### 10. Edge Runtime Limitations

**Local:** Everything runs on Node.js, so all APIs work.
**Production:** Edge functions cannot use `fs`, `path`, `crypto` (Node.js built-ins).

**Symptom:** Function works locally, crashes in production with "X is not defined."

**Fix:** Check if your route uses `export const runtime = "edge"`. If so, only use Web Standard APIs.

### 11. Serverless Function Size Limits

**Local:** No limits on function size.
**Production:** Vercel has a 50MB limit for serverless functions.

**Symptom:** Build fails with "Function exceeded size limit."

**Fix:** Use dynamic imports for large libraries. Check bundle size with:
```bash
npx @next/bundle-analyzer
```

## Caching

### 12. No Cache Locally, Aggressive Cache in Production

**Local:** Dev server does not cache pages by default.
**Production:** Vercel aggressively caches static pages and ISR output.

**Symptom:** Changes appear instantly locally but take minutes/hours in production.

**Fix:** Use `dynamic = "force-dynamic"` for pages that must always be fresh, or set appropriate `revalidate` intervals.

### 13. ISR Not Available Locally

**Local:** ISR revalidation runs differently in dev mode (always fresh).
**Production:** ISR serves cached pages and revalidates in the background.

**Symptom:** Data is always fresh locally but stale in production.

**Fix:** Test with `npm run build && npm run start` locally to simulate production caching.

## Database

### 14. Different Supabase Projects

**Local:** Using dev/staging Supabase project with test data.
**Production:** Using production Supabase project with real data.

**Symptom:** Data exists locally but not in production, or vice versa.

**Fix:** Make sure migrations are run on both projects:
```bash
supabase db push --linked  # Push to linked project
```

### 15. RLS Policies Not Migrated

**Local:** Created RLS policies through dashboard on dev project.
**Production:** Policies do not exist on production project.

**Symptom:** Queries return data locally but empty arrays in production.

**Fix:** Write RLS policies in migration files so they deploy to all environments:
```sql
-- In a migration file
CREATE POLICY "Users read own data" ON public.profiles
  FOR SELECT USING (auth.uid() = user_id);
```

### 16. Different Data Shape

**Local:** Test data has all fields populated.
**Production:** Real user data has nulls, empty strings, missing fields.

**Symptom:** UI works with test data but crashes with real data.

**Fix:** Always handle null/undefined:
```typescript
<p>{user?.name ?? "Anonymous"}</p>
{(items ?? []).map((item) => ...)}
```

## Build Process

### 17. TypeScript Strictness

**Local:** `npm run dev` does not fail on type errors by default.
**Production:** `npm run build` fails on type errors.

**Symptom:** Dev works fine, build/deploy fails with type errors.

**Fix:** Run `npm run build` locally before pushing to catch errors early.

### 18. ESLint Rules

**Local:** ESLint warnings do not block dev server.
**Production:** Vercel build fails on ESLint errors.

**Fix:** Fix lint errors, or temporarily disable in `next.config.js` (not recommended long-term):
```typescript
const nextConfig = { eslint: { ignoreDuringBuilds: true } };
```

### 19. Missing Dependencies

**Local:** A globally installed package works.
**Production:** Only packages in `package.json` are available.

**Symptom:** `Module not found` only in production.

**Fix:** Install the package as a project dependency:
```bash
npm install the-missing-package
```

## Timing and Performance

### 20. Cold Starts

**Local:** Dev server is always running, no cold start.
**Production:** Serverless functions spin up on demand, first request is slow.

**Symptom:** First API call takes 1-3 seconds, subsequent ones are fast.

**Fix:** This is normal for serverless. Use Edge runtime for faster cold starts, or use cron to keep functions warm.

### 21. Timeout Limits

**Local:** No timeout on dev server.
**Production:** Vercel free tier: 10s, Pro: 60s function timeout.

**Symptom:** Long-running operations work locally but timeout in production.

**Fix:** Optimize the function, use streaming, or break into smaller functions.

### 22. Concurrent Request Limits

**Local:** Single user, no concurrency issues.
**Production:** Multiple users hitting the same endpoint simultaneously.

**Symptom:** Works for one user, fails under load.

**Fix:** Use connection pooling for database, add rate limiting, optimize queries.

## Quick Pre-Deploy Checklist

Before deploying, verify:

1. `npm run build` succeeds locally
2. All env vars are in Vercel dashboard
3. No hardcoded `localhost` URLs
4. No Node.js built-in imports in Edge routes
5. File name casing matches imports exactly
6. RLS policies exist on production Supabase
7. All dependencies in `package.json` (not just global)
8. Dynamic class names use full Tailwind strings
