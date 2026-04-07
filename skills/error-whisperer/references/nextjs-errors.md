# Next.js Errors (50+)

Every common Next.js error with plain English explanation and exact fix.

## Build Errors

### 1. Module not found: Can't resolve '@/components/X'

**Why:** The import path is wrong or the file does not exist at that location.

**Fix:**
```typescript
// Wrong
import { Button } from "@/components/Button";
// Right — check exact file name and path
import { Button } from "@/components/ui/button";
```

**Prevent:** Use your editor's auto-import. Never type import paths manually.

### 2. Type error: Property 'X' does not exist on type 'Y'

**Why:** TypeScript found a mismatch between what you wrote and what the type expects.

**Fix:** Check the type definition and add the missing property or fix the name.

**Prevent:** Let TypeScript guide you — hover over variables to see their types.

### 3. SyntaxError: Unexpected token

**Why:** There is a typo, missing bracket, or wrong syntax in your code.

**Fix:** Check the file and line number in the error. Look for missing `}`, `)`, or `;`.

**Prevent:** Use ESLint and Prettier to auto-format your code.

### 4. Module not found: Can't resolve 'fs' (or 'path', 'crypto', etc.)

**Why:** You are using a Node.js module in a client component. Browser cannot use `fs`.

**Fix:**
```typescript
// Add 'use server' or move to a Server Component / API route
// Or use dynamic import:
const data = await import("fs").then((fs) => fs.readFileSync(path));
```

**Prevent:** Node.js built-in modules only work in Server Components, API routes, or server actions.

### 5. Error: 'X' is not a valid Next.js page export

**Why:** You exported something Next.js does not recognize from a page file.

**Fix:** Pages can only export: `default` (the component), `metadata`, `generateMetadata`, `generateStaticParams`, `revalidate`, `dynamic`, `runtime`.

### 6. Error: Cannot find module 'next/headers'

**Why:** Your Next.js version is outdated. `next/headers` was added in Next.js 13.

**Fix:**
```bash
npm install next@latest
```

### 7. TypeError: Cannot read properties of undefined (reading 'map')

**Why:** You are calling `.map()` on something that is `undefined` instead of an array.

**Fix:**
```typescript
// Add a fallback
{(items ?? []).map((item) => <div key={item.id}>{item.name}</div>)}
```

**Prevent:** Always initialize arrays and check for null/undefined before mapping.

## Runtime Errors

### 8. Hydration mismatch: Server HTML and client HTML differ

**Why:** The server rendered different HTML than the client. Common causes: dates, random numbers, browser-only APIs.

**Fix:**
```typescript
// Option 1: Mark as client component
"use client";

// Option 2: Use useEffect for browser-only values
const [time, setTime] = useState("");
useEffect(() => setTime(new Date().toLocaleString()), []);

// Option 3: Suppress for known safe cases
<time suppressHydrationWarning>{date}</time>
```

**Prevent:** Never use `Date.now()`, `Math.random()`, or `window` in the initial render of a Server Component.

### 9. Error: Invariant: headers() expects to have requestAsyncStorage

**Why:** You called `headers()`, `cookies()`, or another dynamic API outside of a request context.

**Fix:** Make sure these are called inside a Server Component, server action, or route handler — not at module level.

### 10. Error: Dynamic server usage: headers

**Why:** A static page is trying to use dynamic APIs like `headers()` or `cookies()`.

**Fix:**
```typescript
// Add to the page file:
export const dynamic = "force-dynamic";
```

### 11. Error: Unhandled Runtime Error — Text content mismatch

**Why:** Same as hydration mismatch. Server sent "X" but client rendered "Y".

**Fix:** See fix #8 above. Most common cause: date formatting that differs between server and client timezone.

### 12. Error: Event handlers cannot be passed to Client Component props from Server Components

**Why:** You passed an `onClick` or other event handler from a Server Component to a Client Component without marking it.

**Fix:**
```typescript
// The child component needs "use client" at the top
"use client";
export function Button({ onClick }: { onClick: () => void }) {
  return <button onClick={onClick}>Click</button>;
}
```

### 13. Error: Functions cannot be passed directly to Client Components

**Why:** Server Components cannot pass functions as props to Client Components (they can not be serialized).

**Fix:** Move the function into the Client Component, or use a server action.

```typescript
// Server action approach
"use server";
export async function handleSubmit(formData: FormData) {
  // server logic here
}
```

### 14. Error: Cannot access 'X' before initialization

**Why:** Circular dependency between modules or using a variable before it is declared.

**Fix:** Check your import chain. File A imports from File B which imports from File A — break the cycle.

### 15. Error: Unsupported Server Component type: undefined

**Why:** Your page or layout is not exporting a valid React component as default.

**Fix:**
```typescript
// Make sure you have a default export
export default function Page() {
  return <div>Content</div>;
}
```

## Routing Errors

### 16. 404 — Page not found

**Why:** The file is not in the right folder, or the folder name does not match the URL.

**Fix:** Check that your file is at `app/{route}/page.tsx`. The folder structure IS the URL structure.

```
app/dashboard/page.tsx → /dashboard
app/blog/[slug]/page.tsx → /blog/my-post
```

### 17. Error: You cannot have two parallel pages resolving to the same path

**Why:** Two `page.tsx` files resolve to the same URL.

**Fix:** Check for conflicting routes. Route groups `(groupName)` share the same URL space.

### 18. Error: Middleware redirected too many times

**Why:** Your middleware creates an infinite redirect loop.

**Fix:**
```typescript
// middleware.ts — exclude the redirect target from the matcher
export const config = {
  matcher: ["/((?!login|_next/static|_next/image|favicon.ico).*)"],
};
```

### 19. Error: generateStaticParams must return an array

**Why:** Your `generateStaticParams` function returned something other than an array.

**Fix:**
```typescript
export async function generateStaticParams() {
  const posts = await getPosts();
  return posts.map((post) => ({ slug: post.slug }));
}
```

### 20. Error: Route "X" used `cookies` but is configured with `force-static`

**Why:** Conflict — you told Next.js the page is static but used a dynamic API.

**Fix:** Remove `force-static` or stop using cookies/headers in that route.

## Data Fetching Errors

### 21. Error: fetch failed — ECONNREFUSED

**Why:** The API you are trying to reach is not running or the URL is wrong.

**Fix:** Check the URL in your fetch call. If calling your own API routes during build, use absolute URLs.

### 22. Error: Fetch API cannot load 'X' due to access control checks

**Why:** CORS issue — the API does not allow requests from your domain.

**Fix:** Create a Next.js API route as a proxy:
```typescript
// app/api/proxy/route.ts
export async function GET() {
  const res = await fetch("https://external-api.com/data");
  const data = await res.json();
  return Response.json(data);
}
```

### 23. Error: revalidatePath/revalidateTag can only be called in a server action or route handler

**Why:** You tried to revalidate cache from a Client Component.

**Fix:** Move the revalidation into a server action:
```typescript
"use server";
import { revalidatePath } from "next/cache";

export async function refreshData() {
  revalidatePath("/dashboard");
}
```

### 24. Data is stale after mutation

**Why:** Next.js cached the old data. You need to revalidate after writing.

**Fix:**
```typescript
"use server";
import { revalidatePath } from "next/cache";

export async function createPost(formData: FormData) {
  await db.insert(posts).values({ title: formData.get("title") });
  revalidatePath("/blog");
}
```

### 25. Error: Failed to collect page data for /X

**Why:** The page threw an error during static generation (build time).

**Fix:** Check the full stack trace. Common cause: API call fails during build because env vars are missing.

## Middleware Errors

### 26. Error: Middleware cannot use Node.js APIs

**Why:** Next.js Middleware runs on the Edge runtime. No `fs`, `path`, `crypto` Node modules.

**Fix:** Use Web APIs instead. For crypto: `crypto.subtle`. For encoding: `TextEncoder`.

### 27. Error: Middleware returned a response body

**Why:** Edge middleware cannot return response bodies directly in older versions.

**Fix:**
```typescript
// Use NextResponse.json() in route handlers instead
// In middleware, only redirect or rewrite:
return NextResponse.redirect(new URL("/login", request.url));
```

### 28. Middleware runs on every request including static files

**Why:** Default matcher catches everything.

**Fix:**
```typescript
export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)"],
};
```

## Image Errors

### 29. Error: Invalid src prop on `next/image`

**Why:** The image URL domain is not in the allowed list.

**Fix:**
```typescript
// next.config.js
const nextConfig = {
  images: {
    remotePatterns: [
      { protocol: "https", hostname: "**.supabase.co" },
      { protocol: "https", hostname: "lh3.googleusercontent.com" },
    ],
  },
};
export default nextConfig;
```

### 30. Error: Image with src "X" has width or height of 0

**Why:** The image dimensions are missing or zero.

**Fix:** Always specify `width` and `height` or use `fill` prop with a sized container.

## API Route Errors

### 31. Error: Route handler body size exceeds the limit

**Why:** The response body is larger than the default limit (4MB on Vercel).

**Fix:** Stream the response or paginate the data.

### 32. Error: No response is returned from route handler

**Why:** Your route handler does not return a `Response` object.

**Fix:**
```typescript
export async function GET() {
  return Response.json({ message: "ok" });
}
```

### 33. Error: Method 'X' is not supported

**Why:** You exported a function with the wrong HTTP method name.

**Fix:** Export functions named `GET`, `POST`, `PUT`, `PATCH`, `DELETE` (uppercase).

## Config Errors

### 34. Error: next.config.js is not a valid configuration

**Why:** Syntax error or invalid option in the config file.

**Fix:** Check for typos. Use `next.config.mjs` for ES module syntax.

### 35. Warning: Invalid next.config.js options detected — unrecognized key 'X'

**Why:** You used an option that does not exist or was renamed.

**Fix:** Check the Next.js docs for the current config options.

## Server Action Errors

### 36. Error: Server actions must be async functions

**Why:** Server actions require the `async` keyword.

**Fix:**
```typescript
"use server";
export async function submitForm(formData: FormData) {
  // must be async
}
```

### 37. Error: Only plain objects can be passed to Client Components from Server Components

**Why:** You are trying to pass a class instance, Date object, or function.

**Fix:** Serialize to plain objects:
```typescript
const data = { ...result, createdAt: result.createdAt.toISOString() };
```

### 38. Error: Server action not found

**Why:** The server action file is missing `"use server"` at the top.

**Fix:** Add `"use server";` as the first line of the file or before the function.

## Suspense and Loading Errors

### 39. Error: Suspense boundary received an update before it finished hydrating

**Why:** A Suspense boundary was updated before initial hydration completed.

**Fix:** Ensure `loading.tsx` files exist for routes that use async data fetching.

### 40. Error: Missing Suspense boundary with streaming

**Why:** You used streaming (async Server Components) without a Suspense wrapper.

**Fix:** Add `loading.tsx` to the route folder, or wrap with `<Suspense>`.

## Metadata Errors

### 41. Error: Metadata export is only supported in Server Components

**Why:** You put `export const metadata` in a Client Component file.

**Fix:** Move metadata to a Server Component (remove `"use client"`), or use `generateMetadata`.

### 42. Error: generateMetadata and metadata cannot both be exported

**Why:** You exported both the static object and the function in the same file.

**Fix:** Use only one — `generateMetadata` for dynamic, `metadata` for static.

## CSS and Styling Errors

### 43. Error: Global CSS cannot be imported from within node_modules

**Why:** Importing CSS from a package in the wrong location.

**Fix:** Import global CSS only in `app/layout.tsx` or `app/globals.css`.

### 44. Tailwind classes not applying

**Why:** Tailwind is not scanning the right files, or class is dynamically constructed.

**Fix:**
```typescript
// tailwind.config.ts — make sure content paths are correct:
content: ["./app/**/*.{ts,tsx}", "./components/**/*.{ts,tsx}"],

// WRONG: dynamic class names get purged
<div className={`text-${color}-500`} />

// RIGHT: use full class names
<div className={color === "red" ? "text-red-500" : "text-blue-500"} />
```

### 45. PostCSS plugin error

**Why:** PostCSS config is misconfigured or a plugin is missing.

**Fix:**
```bash
npm install -D tailwindcss postcss autoprefixer
```

## TypeScript Integration Errors

### 46. Error: 'X' cannot be used as a JSX component

**Why:** The component's return type does not match what React expects.

**Fix:** Make sure the component returns `JSX.Element` or `React.ReactNode`. Check for missing `return` statements.

### 47. Error: Parameter 'X' implicitly has an 'any' type

**Why:** TypeScript strict mode requires explicit types.

**Fix:**
```typescript
// Add the type
export default function Page({ params }: { params: { slug: string } }) {
  return <div>{params.slug}</div>;
}
```

## Environment Variable Errors

### 48. Error: Missing required environment variable: X

**Why:** The env var is not set in `.env.local` or the deployment platform.

**Fix:** Add it to `.env.local` for local dev, and to Vercel/hosting dashboard for production.

### 49. NEXT_PUBLIC_ variable is undefined on the client

**Why:** Client-side env vars must start with `NEXT_PUBLIC_`.

**Fix:**
```bash
# .env.local
# Server only (not accessible in browser):
SUPABASE_SERVICE_ROLE_KEY=xxx
# Client accessible:
NEXT_PUBLIC_SUPABASE_URL=xxx
```

### 50. Environment variable changes not taking effect

**Why:** You changed `.env.local` but did not restart the dev server.

**Fix:** Stop the dev server (Ctrl+C) and restart with `npm run dev`. Env vars are loaded at startup.

## Catch-All Patterns

### 51. White screen with no error

**Why:** Usually a rendering error caught by a boundary, or a missing `"use client"` directive.

**Fix:** Check browser console (F12). Add `error.tsx` to your route for error boundaries.

### 52. Infinite re-render loop

**Why:** A `useEffect` dependency causes itself to re-trigger.

**Fix:** Check `useEffect` dependencies. Remove objects/arrays from deps, use specific values instead.

### 53. Build succeeds locally but fails on Vercel

**Why:** Case-sensitive file system on Vercel vs case-insensitive locally.

**Fix:** Check file name casing. `Component.tsx` and `component.tsx` are the same locally but different on Vercel.
