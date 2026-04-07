---
name: nextjs-app-router
description: "Use this skill whenever the user mentions Next.js, pages, routes, routing, layouts, navigation, server components, client components, SSR, SSG, ISR, streaming, loading states, error boundaries, middleware, app router, file conventions, page.tsx, layout.tsx, API routes, route handlers, server actions, forms, metadata, SEO tags, parallel routes, intercepting routes, catch-all routes, dynamic routes, or ANY web app structure task — even if they don't explicitly say 'Next.js'. This is the foundational skill for every web project."
---

# Next.js App Router

Every file you create inside the `app/` folder becomes a route. This skill covers all file conventions, data fetching, and patterns.

## App Router File Conventions

### page.tsx — The Page Itself

A folder only becomes a visitable route if it has a `page.tsx`.

```tsx
// app/page.tsx
export default function HomePage() {
  return (
    <main className="flex min-h-screen flex-col items-center justify-center p-8">
      <h1 className="text-4xl font-bold">Welcome to My App</h1>
    </main>
  );
}
```

### layout.tsx — Wraps Pages with Shared UI

Layouts persist across page navigations. The root layout is required.

```tsx
// app/layout.tsx
import type { Metadata } from "next";
import "./globals.css";

export const metadata: Metadata = {
  title: "My App",
  description: "Built with Next.js and Supabase",
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body className="bg-white text-gray-900 antialiased">{children}</body>
    </html>
  );
}
```

### loading.tsx — Shows While Page Loads

```tsx
// app/dashboard/loading.tsx
export default function DashboardLoading() {
  return (
    <div className="flex min-h-screen items-center justify-center">
      <div className="h-8 w-8 animate-spin rounded-full border-4 border-blue-500 border-t-transparent" />
    </div>
  );
}
```

### error.tsx — Catches Errors (Must Be Client Component)

```tsx
// app/dashboard/error.tsx
"use client";

export default function DashboardError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div className="flex min-h-screen flex-col items-center justify-center gap-4">
      <h2 className="text-2xl font-bold text-red-600">Something went wrong</h2>
      <p className="text-gray-600">{error.message}</p>
      <button onClick={reset} className="rounded bg-blue-500 px-4 py-2 text-white">Try again</button>
    </div>
  );
}
```

### not-found.tsx — Custom 404

```tsx
// app/not-found.tsx
import Link from "next/link";

export default function NotFound() {
  return (
    <div className="flex min-h-screen flex-col items-center justify-center gap-4">
      <h2 className="text-4xl font-bold">404 — Page Not Found</h2>
      <Link href="/" className="text-blue-500 underline">Go back home</Link>
    </div>
  );
}
```

### route.ts — API Route Handler

```ts
// app/api/health/route.ts
import { NextResponse } from "next/server";

export async function GET() {
  return NextResponse.json({ status: "ok", timestamp: new Date().toISOString() });
}
```

## Server vs Client Components

By default, every component is a **Server Component** (runs on server, no JS sent to browser). Add `"use client"` at the top to make it a **Client Component** (runs in browser, can use hooks).

**Use Client Component when you need:** `useState`, `useEffect`, `onClick`, `onChange`, browser APIs (`localStorage`, `window`), or third-party libraries that use hooks.

**Keep as Server Component when:** fetching data, accessing secrets, displaying content without interactivity.

```tsx
// app/components/like-button.tsx
"use client";

import { useState } from "react";

export function LikeButton({ initialCount }: { initialCount: number }) {
  const [count, setCount] = useState(initialCount);
  return (
    <button onClick={() => setCount((c) => c + 1)} className="rounded bg-pink-500 px-4 py-2 text-white">
      {count} likes
    </button>
  );
}
```

See `references/server-vs-client.md` for the full decision tree and common mistakes.

## Data Fetching in Server Components

Server Components can be async. Fetch data directly — no `useEffect` needed.

```tsx
// app/posts/page.tsx
import { createClient } from "@/lib/supabase/server";

export default async function PostsPage() {
  const supabase = await createClient();
  const { data: posts, error } = await supabase
    .from("posts")
    .select("id, title, created_at")
    .order("created_at", { ascending: false });

  if (error) throw new Error("Failed to load posts");

  return (
    <ul className="space-y-4 p-8">
      {posts.map((post) => (
        <li key={post.id} className="rounded border p-4">
          <h2 className="text-xl font-semibold">{post.title}</h2>
        </li>
      ))}
    </ul>
  );
}
```

## Server Actions (Forms and Mutations)

```tsx
// app/posts/new/page.tsx
import { createClient } from "@/lib/supabase/server";
import { redirect } from "next/navigation";
import { revalidatePath } from "next/cache";

export default function NewPostPage() {
  async function createPost(formData: FormData) {
    "use server";
    const title = formData.get("title") as string;
    if (!title || title.length < 3) throw new Error("Title must be at least 3 characters");

    const supabase = await createClient();
    const { error } = await supabase.from("posts").insert({ title });
    if (error) throw new Error("Failed to create post");

    revalidatePath("/posts");
    redirect("/posts");
  }

  return (
    <form action={createPost} className="mx-auto max-w-lg space-y-4 p-8">
      <input name="title" required minLength={3} placeholder="Post title" className="w-full rounded border p-2" />
      <button type="submit" className="rounded bg-blue-500 px-6 py-2 text-white">Publish</button>
    </form>
  );
}
```

See `references/data-fetching.md` for route handlers, revalidation patterns, and parallel fetching.

## Dynamic Routes

```tsx
// app/posts/[slug]/page.tsx
import { createClient } from "@/lib/supabase/server";
import { notFound } from "next/navigation";

export default async function PostPage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const supabase = await createClient();
  const { data: post, error } = await supabase.from("posts").select("*").eq("slug", slug).single();

  if (error || !post) notFound();

  return (
    <article className="mx-auto max-w-2xl p-8">
      <h1 className="text-3xl font-bold">{post.title}</h1>
      <p className="mt-4 text-gray-700">{post.body}</p>
    </article>
  );
}
```

See `references/routing-patterns.md` for catch-all routes, route groups, parallel routes, and intercepting routes.

## Middleware

Middleware runs before every request. Place it at the project root (not inside `app/`).

```ts
// middleware.ts
import { createServerClient } from "@supabase/ssr";
import { NextResponse, type NextRequest } from "next/server";

export async function middleware(request: NextRequest) {
  let response = NextResponse.next({ request });
  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() { return request.cookies.getAll(); },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value, options }) => {
            request.cookies.set(name, value);
            response.cookies.set(name, value, options);
          });
        },
      },
    }
  );

  const { data: { user } } = await supabase.auth.getUser();

  if (!user && request.nextUrl.pathname.startsWith("/dashboard")) {
    return NextResponse.redirect(new URL("/login", request.url));
  }

  return response;
}

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)"],
};
```

See `references/middleware.md` for redirects, CORS, geo-routing, and rate limiting.

## Metadata API for SEO

```tsx
// app/posts/[slug]/page.tsx
import type { Metadata } from "next";

export async function generateMetadata({ params }: { params: Promise<{ slug: string }> }): Promise<Metadata> {
  const { slug } = await params;
  const supabase = await createClient();
  const { data: post } = await supabase.from("posts").select("title, description").eq("slug", slug).single();

  if (!post) return { title: "Post Not Found" };

  return {
    title: post.title,
    description: post.description,
    openGraph: { title: post.title, description: post.description },
  };
}
```

## Streaming and Suspense

Wrap slow components in Suspense to stream them in as they load.

```tsx
// app/dashboard/page.tsx
import { Suspense } from "react";

export default function DashboardPage() {
  return (
    <div className="p-8">
      <h1 className="text-3xl font-bold">Dashboard</h1>
      <Suspense fallback={<p className="text-gray-500">Loading stats...</p>}>
        <DashboardStats />
      </Suspense>
    </div>
  );
}

async function DashboardStats() {
  const supabase = await createClient();
  const { count } = await supabase.from("posts").select("*", { count: "exact", head: true });
  return <p className="mt-4 text-2xl">Total posts: {count}</p>;
}
```

## Environment Variables

- `NEXT_PUBLIC_` prefix: exposed to the browser (Supabase URL, anon key)
- No prefix: server-only (service role key, API keys)

```bash
# .env.local
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_ROLE_KEY=eyJ...  # NEVER expose to browser
GEMINI_API_KEY=AIza...            # Server-only
```

## ISR / SSG / SSR Selection

- **SSR (default)**: Re-renders every request. Use for auth/personalized content.
- **SSG**: Built at build time. Add `generateStaticParams`.
- **ISR**: Regenerates on interval. Add `export const revalidate = 60;`.

```tsx
export async function generateStaticParams() {
  const supabase = await createClient();
  const { data: posts } = await supabase.from("posts").select("slug");
  return (posts ?? []).map((post) => ({ slug: post.slug }));
}

export const revalidate = 60;
```

## next.config.ts

```ts
// next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  images: {
    remotePatterns: [{ protocol: "https", hostname: "*.supabase.co", pathname: "/storage/v1/object/public/**" }],
  },
  async redirects() {
    return [{ source: "/old-page", destination: "/new-page", permanent: true }];
  },
};

export default nextConfig;
```

## Route Groups

Organize routes without affecting the URL using parentheses.

```
app/
  (marketing)/
    page.tsx          → /
    about/page.tsx    → /about
    layout.tsx        → marketing layout
  (app)/
    dashboard/page.tsx → /dashboard
    settings/page.tsx  → /settings
    layout.tsx         → app layout (with sidebar)
```

See `references/troubleshooting.md` for the top 20 Next.js errors and their fixes.
