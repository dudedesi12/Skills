# Top 20 Build Errors and Fixes

## 1. Module not found

```
Module not found: Can't resolve 'package-name'
```

**Fix:** Install the missing package:
```bash
npm install package-name
```

## 2. TypeScript type error

```
Type error: Property 'x' does not exist on type 'y'
```

**Fix:** Fix the type. Common causes:
- Missing property in an interface — add it
- Wrong variable type — check your data shapes
- Nullable value — use optional chaining: `data?.property`

## 3. Missing environment variable at build time

```
Error: Missing required environment variable: NEXT_PUBLIC_SUPABASE_URL
```

**Fix:** Add the variable in Vercel dashboard under **Settings** then **Environment Variables**. Make sure it is enabled for the **Production** environment.

## 4. Cannot find module or its type declarations

```
Cannot find module '@/components/Button' or its corresponding type declarations
```

**Fix:** Check the file path. Verify `tsconfig.json` has the path alias:
```json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```
Make sure the file exists at the exact path.

## 5. Dynamic server usage without dynamic flag

```
Error: Dynamic server usage: Page couldn't be rendered statically because it used `cookies`.
```

**Fix:** Add at the top of the page or layout:
```typescript
export const dynamic = "force-dynamic";
```

## 6. Edge runtime incompatibility

```
Dynamic Code Evaluation (e.g., 'eval', 'new Function') not allowed in Edge Runtime
```

**Fix:** Either remove `export const runtime = 'edge'` or replace the incompatible code. Common culprits: `bcrypt` (use `bcryptjs`), some Supabase operations.

## 7. ESLint errors blocking build

```
ESLint: error  Unexpected any  @typescript-eslint/no-explicit-any
```

**Fix:** Fix the lint errors, or if needed temporarily, add to `next.config.ts`:
```typescript
const nextConfig = {
  eslint: {
    ignoreDuringBuilds: false, // Set true only as last resort
  },
};
```

## 8. Image optimization error

```
Error: Invalid src prop on `next/image`
```

**Fix:** Add the domain to `next.config.ts`:
```typescript
const nextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: "https",
        hostname: "your-project.supabase.co",
        pathname: "/storage/v1/object/public/**",
      },
    ],
  },
};
```

## 9. Out of memory during build

```
FATAL ERROR: Reached heap limit Allocation failed - JavaScript heap out of memory
```

**Fix:** Add to `package.json` scripts:
```json
{
  "scripts": {
    "build": "NODE_OPTIONS='--max-old-space-size=4096' next build"
  }
}
```

## 10. Serverless function too large

```
Error: Serverless Function size exceeds limit (50MB)
```

**Fix:**
- Check for large dependencies — remove unused packages
- Use dynamic imports: `const lib = await import('heavy-lib')`
- Add large packages to `serverExternalPackages` in next.config.ts

## 11. generateStaticParams error

```
Error: Failed to collect page data for /blog/[slug]
```

**Fix:** Check that `generateStaticParams` returns valid data and handles errors:
```typescript
export async function generateStaticParams() {
  try {
    const supabase = createClient();
    const { data } = await supabase.from("posts").select("slug");
    return (data ?? []).map((post) => ({ slug: post.slug }));
  } catch {
    return [];
  }
}
```

## 12. Metadata export error

```
Error: "metadata" is not a valid export from a Client Component
```

**Fix:** Remove `"use client"` from the page. Metadata must be in Server Components. If you need client-side interactivity, extract it to a separate client component and import it.

## 13. Hydration mismatch

```
Error: Text content does not match server-rendered HTML
```

**Fix:** Common causes:
- Using `Date.now()` or `Math.random()` in render — move to useEffect
- Browser extensions modifying HTML
- Conditional rendering based on `window` — use `useEffect` + state

## 14. Import cycle / circular dependency

```
Warning: Circular dependency detected
```

**Fix:** Restructure imports. Extract shared types/utils into a separate file that both modules import from, rather than importing from each other.

## 15. PostCSS / Tailwind error

```
Error: Cannot find module 'tailwindcss'
```

**Fix:** Make sure Tailwind is installed:
```bash
npm install tailwindcss @tailwindcss/postcss postcss autoprefixer
```

## 16. Prisma / ORM generation error

```
Error: @prisma/client did not initialize yet
```

**Fix:** Add to `package.json`:
```json
{
  "scripts": {
    "postinstall": "prisma generate"
  }
}
```

## 17. API route returning HTML instead of JSON

```
Unexpected token < in JSON at position 0
```

**Fix:** The API route is hitting a 404 or error page. Check:
- Route file is named `route.ts` (not `page.ts`)
- File is in the correct `app/api/` directory
- No conflicting `page.ts` in the same folder

## 18. CORS errors in production

```
Access to fetch at 'https://api...' has been blocked by CORS policy
```

**Fix:** Add CORS headers in your API route:
```typescript
export async function OPTIONS() {
  return new Response(null, {
    headers: {
      "Access-Control-Allow-Origin": process.env.NEXT_PUBLIC_SITE_URL || "*",
      "Access-Control-Allow-Methods": "GET, POST, OPTIONS",
      "Access-Control-Allow-Headers": "Content-Type, Authorization",
    },
  });
}
```

## 19. Sharp module error

```
Error: Could not load the "sharp" module
```

**Fix:** Vercel handles Sharp automatically. If you see this error, check:
- Remove `sharp` from `dependencies` if manually added
- Or pin the version: `npm install sharp@0.33.2`

## 20. Middleware matcher error

```
Error: Invalid middleware matcher pattern
```

**Fix:** Check `middleware.ts` matcher syntax:
```typescript
// middleware.ts
export const config = {
  matcher: [
    "/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)",
  ],
};
```
