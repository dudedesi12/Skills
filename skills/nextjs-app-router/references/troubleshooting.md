# Next.js Troubleshooting — Top 20 Errors

## 1. Hydration Mismatch

**Error:** `Text content does not match server-rendered HTML` or `Hydration failed because the initial UI does not match`

**Why:** The HTML generated on the server is different from what React renders in the browser. Common causes: using `Date.now()`, `Math.random()`, browser-only values, or wrapping elements incorrectly (e.g., `<p>` inside `<p>`).

**Fix:**

```tsx
// BAD — different output on server vs client
export default function Page() {
  return <p>Time: {Date.now()}</p>; // Different on server vs browser
}

// GOOD — use useEffect for browser-only values
"use client";
import { useState, useEffect } from "react";

export default function Page() {
  const [time, setTime] = useState<number | null>(null);

  useEffect(() => {
    setTime(Date.now());
  }, []);

  return <p>Time: {time ?? "Loading..."}</p>;
}
```

Also check for invalid HTML nesting:
```tsx
// BAD — <div> cannot be inside <p>
<p><div>Hello</div></p>

// GOOD
<div><div>Hello</div></div>
```

## 2. "use client" Missing

**Error:** `useState only works in Client Components. Add the "use client" directive.`

**Why:** You used a React hook (`useState`, `useEffect`, etc.) in a Server Component.

**Fix:** Add `"use client"` as the very first line of the file:

```tsx
"use client"; // Must be the FIRST line, before all imports

import { useState } from "react";
```

## 3. Cannot Use async in Client Components

**Error:** `async/await is not yet supported in Client Components`

**Why:** Client Components cannot be async functions.

**Fix:** Use `useEffect` to fetch data, or fetch in a Server Component and pass data as props:

```tsx
// GOOD — fetch in Server Component parent, pass to Client Component child
// app/page.tsx (Server Component)
export default async function Page() {
  const data = await fetchData();
  return <InteractiveChart data={data} />;
}
```

## 4. Module Not Found: Can't Resolve 'fs'

**Error:** `Module not found: Can't resolve 'fs'`

**Why:** You are importing a Node.js module (`fs`, `path`, `crypto`) in a Client Component or a file imported by one.

**Fix:** Move the code to a Server Component, server action, or route handler. If you need the module in a file that is also imported client-side, use dynamic imports:

```ts
// Only runs on the server
if (typeof window === "undefined") {
  const fs = await import("fs");
}
```

## 5. "window is not defined"

**Error:** `ReferenceError: window is not defined`

**Why:** Accessing `window`, `document`, or `localStorage` during server-side rendering.

**Fix:**

```tsx
"use client";
import { useEffect, useState } from "react";

export function ThemeToggle() {
  const [theme, setTheme] = useState("light");

  useEffect(() => {
    // Safe — only runs in the browser
    const saved = localStorage.getItem("theme") ?? "light";
    setTheme(saved);
  }, []);

  return <button onClick={() => setTheme(theme === "light" ? "dark" : "light")}>{theme}</button>;
}
```

## 6. generateStaticParams Required for Dynamic Routes in Static Export

**Error:** `Dynamic server usage` or `Page couldn't be rendered statically`

**Why:** You have a dynamic route but did not provide `generateStaticParams`, or you are using dynamic features (cookies, headers) in a page that Next.js is trying to statically generate.

**Fix:**

```tsx
// Option 1: Provide static params
export async function generateStaticParams() {
  const supabase = await createClient();
  const { data } = await supabase.from("posts").select("slug");
  return (data ?? []).map((post) => ({ slug: post.slug }));
}

// Option 2: Force dynamic rendering
export const dynamic = "force-dynamic";
```

## 7. Params Must Be Awaited

**Error:** `Type 'Promise<{ slug: string }>' is not assignable to type '{ slug: string }'`

**Why:** In newer Next.js versions, `params` and `searchParams` are Promises that must be awaited.

**Fix:**

```tsx
// BAD (old pattern)
export default function Page({ params }: { params: { slug: string } }) {
  return <h1>{params.slug}</h1>;
}

// GOOD (current pattern)
export default async function Page({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;
  return <h1>{slug}</h1>;
}
```

## 8. Cannot Read Properties of null (Supabase Query)

**Error:** `Cannot read properties of null (reading 'id')` after a Supabase query

**Why:** The query returned null but you did not check for it.

**Fix:**

```tsx
const { data, error } = await supabase.from("posts").select("*").eq("slug", slug).single();

// Always check for errors AND null data
if (error || !data) {
  notFound(); // or throw an error
}

// Now TypeScript knows data is not null
console.log(data.id);
```

## 9. NEXT_REDIRECT Error in Server Actions

**Error:** `NEXT_REDIRECT` thrown as an error in try/catch

**Why:** `redirect()` works by throwing a special error. If you wrap it in try/catch, you catch the redirect.

**Fix:**

```tsx
"use server";
import { redirect } from "next/navigation";

export async function createPost(formData: FormData) {
  // Do your work FIRST
  const { error } = await supabase.from("posts").insert({ ... });

  if (error) {
    return { error: error.message }; // Return errors, don't throw
  }

  // Call redirect OUTSIDE of try/catch
  redirect("/posts");
}
```

## 10. Images Not Loading (next/image)

**Error:** `Invalid src prop` or images showing broken

**Why:** External image domains must be configured in `next.config.ts`.

**Fix:**

```ts
// next.config.ts
const nextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: "https",
        hostname: "*.supabase.co",
        pathname: "/storage/v1/object/public/**",
      },
      {
        protocol: "https",
        hostname: "lh3.googleusercontent.com", // Google OAuth avatars
      },
    ],
  },
};

export default nextConfig;
```

## 11. Cookies Cannot Be Set in Server Components

**Error:** `Cookies can only be modified in a Server Action or Route Handler`

**Why:** Server Components are read-only for cookies. You can read cookies but cannot set them.

**Fix:** Use a Server Action or Route Handler to set cookies:

```ts
// app/actions/theme.ts
"use server";
import { cookies } from "next/headers";

export async function setTheme(theme: string) {
  const cookieStore = await cookies();
  cookieStore.set("theme", theme, {
    httpOnly: true,
    secure: true,
    sameSite: "lax",
    maxAge: 60 * 60 * 24 * 365, // 1 year
  });
}
```

## 12. "Failed to compile" — Import from Server-Only Module

**Error:** `You're importing a component that imports server-only`

**Why:** A Client Component is importing a module marked with `import "server-only"`.

**Fix:** Restructure your imports. Pass server data as props to Client Components instead of importing server modules directly.

## 13. Metadata Export Not Working

**Error:** Metadata/SEO tags not appearing in the page

**Why:** `metadata` or `generateMetadata` can only be exported from `page.tsx` or `layout.tsx` — NOT from Client Components.

**Fix:** Export metadata from the page file (which must be a Server Component):

```tsx
// app/about/page.tsx — Server Component
import type { Metadata } from "next";

export const metadata: Metadata = {
  title: "About Us",
  description: "Learn more about our company",
};

export default function AboutPage() {
  return <h1>About Us</h1>;
}
```

## 14. Route Handler Returns HTML Instead of JSON

**Error:** API route returns an HTML page instead of JSON

**Why:** You have both a `page.tsx` and `route.ts` in the same folder. `page.tsx` takes priority.

**Fix:** Put route handlers in a separate folder (usually under `app/api/`):

```
app/
  posts/
    page.tsx          → /posts (renders HTML page)
  api/
    posts/
      route.ts        → /api/posts (returns JSON)
```

## 15. "use server" Not Working

**Error:** Server action not executing, or `"use server" is not allowed in client modules`

**Why:** `"use server"` must be at the top of a standalone file OR inside an async function in a Server Component. It cannot be at the top of a file that also has `"use client"`.

**Fix:**

```ts
// app/actions.ts — Separate file for server actions
"use server";

export async function myAction() {
  // This runs on the server
}
```

## 16. Layout Not Re-rendering on Navigation

**This is expected behavior**, not a bug. Layouts are shared UI that persist between pages. They do NOT re-render when you navigate to a child page.

If you need something to update on every navigation, put it in the page, not the layout.

## 17. Environment Variables Undefined in Browser

**Error:** `process.env.MY_VAR` is `undefined` on the client

**Why:** Only environment variables prefixed with `NEXT_PUBLIC_` are available in the browser.

**Fix:**

```bash
# .env.local
NEXT_PUBLIC_SUPABASE_URL=https://...   # Available in browser
SUPABASE_SERVICE_ROLE_KEY=eyJ...        # Server-only (no prefix)
```

## 18. Infinite Loop in useEffect

**Error:** Component keeps re-rendering, network tab shows endless requests

**Why:** Missing or incorrect dependency array in `useEffect`, or creating objects/functions inside the component that change on every render.

**Fix:**

```tsx
"use client";
import { useEffect, useState, useCallback } from "react";

export function DataFetcher({ id }: { id: string }) {
  const [data, setData] = useState(null);

  // BAD — missing dependency array causes infinite loop
  // useEffect(() => { fetch(`/api/${id}`).then(...) });

  // GOOD — only runs when id changes
  useEffect(() => {
    let cancelled = false;
    fetch(`/api/${id}`)
      .then((r) => r.json())
      .then((d) => { if (!cancelled) setData(d); })
      .catch(console.error);
    return () => { cancelled = true; };
  }, [id]);

  return <div>{JSON.stringify(data)}</div>;
}
```

## 19. Build Fails with "prerendered" Error

**Error:** `Error: Page "/xyz" is missing "generateStaticParams()" so it cannot be used with "output: export"`

**Why:** Using `output: "export"` (static site) but have dynamic routes without `generateStaticParams`.

**Fix:** Either add `generateStaticParams` to all dynamic routes, or switch to server rendering:

```ts
// next.config.ts — remove output: "export" if you need dynamic routes
const nextConfig = {
  // output: "export",  // Remove this for dynamic routes
};
```

## 20. CORS Error on API Routes

**Error:** `Access to fetch has been blocked by CORS policy`

**Why:** Browser is blocking cross-origin requests to your API routes.

**Fix:**

```ts
// app/api/data/route.ts
import { NextRequest, NextResponse } from "next/server";

export async function OPTIONS() {
  return new NextResponse(null, {
    status: 204,
    headers: {
      "Access-Control-Allow-Origin": "https://your-frontend.com",
      "Access-Control-Allow-Methods": "GET, POST, PUT, DELETE, OPTIONS",
      "Access-Control-Allow-Headers": "Content-Type, Authorization",
      "Access-Control-Max-Age": "86400",
    },
  });
}

export async function GET(request: NextRequest) {
  const data = { message: "Hello" };

  return NextResponse.json(data, {
    headers: {
      "Access-Control-Allow-Origin": "https://your-frontend.com",
    },
  });
}
```
