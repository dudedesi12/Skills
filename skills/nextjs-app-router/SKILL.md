---
name: nextjs-app-router
description: "Use this skill whenever the user mentions Next.js, pages, routes, routing, layouts, navigation, server components, client components, SSR, SSG, ISR, streaming, loading states, error boundaries, middleware, app router, file conventions, page.tsx, layout.tsx, API routes, route handlers, server actions, forms, metadata, SEO tags, parallel routes, intercepting routes, catch-all routes, dynamic routes, or ANY web app structure task — even if they don't explicitly say 'Next.js'. This is the foundational skill for every web project."
---

# Next.js App Router

This skill covers the Next.js App Router — the way you build pages, layouts, and navigation in your web app. Every file you create inside the `app/` folder becomes a route (a page someone can visit).

## App Router File Conventions

Every folder inside `app/` can have special files. Here is what each one does:

### page.tsx — The Page Itself

This is the actual content someone sees when they visit a URL. A folder only becomes a visitable route if it has a `page.tsx`.

```tsx
// app/page.tsx
export default function HomePage() {
  return (
    <main className="flex min-h-screen flex-col items-center justify-center p-8">
      <h1 className="text-4xl font-bold">Welcome to My App</h1>
      <p className="mt-4 text-lg text-gray-600">This is the home page.</p>
    </main>
  );
}
```

### layout.tsx — Wraps Pages with Shared UI

Layouts wrap around pages. The root layout is required and wraps your entire app. Layouts do NOT re-render when you navigate between pages they wrap.

```tsx
// app/layout.tsx
import type { Metadata } from "next";
import "./globals.css";

export const metadata: Metadata = {
  title: "My App",
  description: "Built with Next.js and Supabase",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body className="bg-white text-gray-900 antialiased">
        {children}
      </body>
    </html>
  );
}
```

### loading.tsx — Shows While Page Loads

Drop this file in any route folder. Next.js automatically wraps your page in a Suspense boundary and shows this while the page loads.

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

### error.tsx — Catches Errors in That Route

This file catches JavaScript errors in the page or its children and shows a fallback UI instead of crashing the whole app. Must be a Client Component.

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
      <button
        onClick={reset}
        className="rounded bg-blue-500 px-4 py-2 text-white hover:bg-blue-600"
      >
        Try again
      </button>
    </div>
  );
}
```

### not-found.tsx — Custom 404 Page

```tsx
// app/not-found.tsx
import Link from "next/link";

export default function NotFound() {
  return (
    <div className="flex min-h-screen flex-col items-center justify-center gap-4">
      <h2 className="text-4xl font-bold">404 — Page Not Found</h2>
      <p className="text-gray-600">We could not find what you were looking for.</p>
      <Link href="/" className="text-blue-500 underline hover:text-blue-700">
        Go back home
      </Link>
    </div>
  );
}
```

### route.ts — API Route Handler

This replaces the old `pages/api/` pattern. It handles HTTP requests (GET, POST, PUT, DELETE) at a URL.

```ts
// app/api/health/route.ts
import { NextResponse } from "next/server";

export async function GET() {
  return NextResponse.json({ status: "ok", timestamp: new Date().toISOString() });
}
```

## Server Components vs Client Components

By default, every component in the App Router is a **Server Component**. It runs on the server, can access databases directly, and sends only HTML to the browser.

Add `"use client"` at the top of a file to make it a **Client Component** — it runs in the browser and can use React hooks like `useState`, `useEffect`, and event handlers like `onClick`.

**Decision tree:**
- Need `useState`, `useEffect`, `useRef`? → Client Component
- Need `onClick`, `onChange`, form interactions? → Client Component
- Need browser APIs (localStorage, window)? → Client Component
- Fetching data from Supabase? → Server Component (preferred)
- Just displaying content? → Server Component
- Using a third-party library that uses hooks internally? → Client Component

```tsx
// app/components/like-button.tsx
"use client";

import { useState } from "react";

export function LikeButton({ initialCount }: { initialCount: number }) {
  const [count, setCount] = useState(initialCount);

  return (
    <button
      onClick={() => setCount((c) => c + 1)}
      className="rounded bg-pink-500 px-4 py-2 text-white"
    >
      ❤️ {count}
    </button>
  );
}
```

## Data Fetching Patterns

### Fetching in Server Components (Recommended)

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

  if (error) {
    throw new Error("Failed to load posts");
  }

  return (
    <ul className="space-y-4 p-8">
      {posts.map((post) => (
        <li key={post.id} className="rounded border p-4">
          <h2 className="text-xl font-semibold">{post.title}</h2>
          <p className="text-sm text-gray-500">
            {new Date(post.created_at).toLocaleDateString()}
          </p>
        </li>
      ))}
    </ul>
  );
}
```

### Server Actions (for Forms and Mutations)

Server Actions let you run server-side code when a form is submitted — no API route needed.

```tsx
// app/posts/new/page.tsx
import { createClient } from "@/lib/supabase/server";
import { redirect } from "next/navigation";
import { revalidatePath } from "next/cache";

export default function NewPostPage() {
  async function createPost(formData: FormData) {
    "use server";

    const title = formData.get("title") as string;
    const body = formData.get("body") as string;

    if (!title || title.length < 3) {
      throw new Error("Title must be at least 3 characters");
    }

    const supabase = await createClient();
    const { error } = await supabase.from("posts").insert({ title, body });

    if (error) {
      throw new Error("Failed to create post");
    }

    revalidatePath("/posts");
    redirect("/posts");
  }

  return (
    <form action={createPost} className="mx-auto max-w-lg space-y-4 p-8">
      <input
        name="title"
        placeholder="Post title"
        required
        minLength={3}
        className="w-full rounded border p-2"
      />
      <textarea
        name="body"
        placeholder="Write your post..."
        rows={6}
        className="w-full rounded border p-2"
      />
      <button type="submit" className="rounded bg-blue-500 px-6 py-2 text-white">
        Publish
      </button>
    </form>
  );
}
```

### Route Handlers (API Endpoints)

Use when you need a traditional API endpoint (for webhooks, external services, etc).

```ts
// app/api/posts/route.ts
import { createClient } from "@/lib/supabase/server";
import { NextRequest, NextResponse } from "next/server";

export async function GET(request: NextRequest) {
  const supabase = await createClient();

  const { searchParams } = new URL(request.url);
  const limit = parseInt(searchParams.get("limit") ?? "10", 10);

  const { data, error } = await supabase
    .from("posts")
    .select("*")
    .limit(limit)
    .order("created_at", { ascending: false });

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 });
  }

  return NextResponse.json(data);
}

export async function POST(request: NextRequest) {
  const supabase = await createClient();
  const body = await request.json();

  if (!body.title || typeof body.title !== "string") {
    return NextResponse.json({ error: "Title is required" }, { status: 400 });
  }

  const { data, error } = await supabase
    .from("posts")
    .insert({ title: body.title, body: body.body })
    .select()
    .single();

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 });
  }

  return NextResponse.json(data, { status: 201 });
}
```

## Dynamic Routes

Use square brackets in folder names to create dynamic segments.

```tsx
// app/posts/[slug]/page.tsx
import { createClient } from "@/lib/supabase/server";
import { notFound } from "next/navigation";

export default async function PostPage({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;
  const supabase = await createClient();

  const { data: post, error } = await supabase
    .from("posts")
    .select("*")
    .eq("slug", slug)
    .single();

  if (error || !post) {
    notFound();
  }

  return (
    <article className="mx-auto max-w-2xl p-8">
      <h1 className="text-3xl font-bold">{post.title}</h1>
      <p className="mt-4 text-gray-700">{post.body}</p>
    </article>
  );
}
```

## Middleware

Middleware runs before every request. Use it for auth checks, redirects, and headers.

```ts
// middleware.ts (at the project root, NOT inside app/)
import { createServerClient } from "@supabase/ssr";
import { NextResponse, type NextRequest } from "next/server";

export async function middleware(request: NextRequest) {
  let response = NextResponse.next({ request });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll();
        },
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
    const url = request.nextUrl.clone();
    url.pathname = "/login";
    return NextResponse.redirect(url);
  }

  return response;
}

export const config = {
  matcher: [
    "/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)",
  ],
};
```

## Metadata API for SEO

```tsx
// app/posts/[slug]/page.tsx — add this above or alongside your component
import type { Metadata } from "next";

export async function generateMetadata({
  params,
}: {
  params: Promise<{ slug: string }>;
}): Promise<Metadata> {
  const { slug } = await params;
  const supabase = await createClient();
  const { data: post } = await supabase
    .from("posts")
    .select("title, description")
    .eq("slug", slug)
    .single();

  if (!post) {
    return { title: "Post Not Found" };
  }

  return {
    title: post.title,
    description: post.description,
    openGraph: {
      title: post.title,
      description: post.description,
    },
  };
}
```

## Streaming and Suspense

Wrap slow parts of your page in Suspense to stream them in as they load.

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
      <Suspense fallback={<p className="text-gray-500">Loading activity...</p>}>
        <RecentActivity />
      </Suspense>
    </div>
  );
}

async function DashboardStats() {
  const supabase = await createClient();
  const { count } = await supabase
    .from("posts")
    .select("*", { count: "exact", head: true });

  return <p className="mt-4 text-2xl">Total posts: {count}</p>;
}

async function RecentActivity() {
  const supabase = await createClient();
  const { data } = await supabase
    .from("activity_log")
    .select("*")
    .order("created_at", { ascending: false })
    .limit(5);

  return (
    <ul className="mt-4 space-y-2">
      {data?.map((item) => (
        <li key={item.id} className="text-sm text-gray-600">{item.description}</li>
      ))}
    </ul>
  );
}
```

## Environment Variables

- `NEXT_PUBLIC_` prefix: exposed to the browser (use for Supabase URL, anon key)
- No prefix: server-only (use for secrets, service role keys)

```bash
# .env.local
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_ROLE_KEY=eyJ...  # NEVER expose this to the browser
GEMINI_API_KEY=AIza...            # Server-only
```

Always validate env vars at startup:

```ts
// lib/env.ts
function getEnvVar(name: string): string {
  const value = process.env[name];
  if (!value) {
    throw new Error(`Missing environment variable: ${name}`);
  }
  return value;
}

export const env = {
  supabaseUrl: getEnvVar("NEXT_PUBLIC_SUPABASE_URL"),
  supabaseAnonKey: getEnvVar("NEXT_PUBLIC_SUPABASE_ANON_KEY"),
  supabaseServiceRoleKey: getEnvVar("SUPABASE_SERVICE_ROLE_KEY"),
};
```

## next.config.ts Common Patterns

```ts
// next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: "https",
        hostname: "*.supabase.co",
        pathname: "/storage/v1/object/public/**",
      },
    ],
  },
  experimental: {
    serverActions: {
      bodySizeLimit: "2mb",
    },
  },
  async redirects() {
    return [
      {
        source: "/old-page",
        destination: "/new-page",
        permanent: true,
      },
    ];
  },
};

export default nextConfig;
```

## ISR / SSG / SSR Selection

- **SSR (default)**: Page re-renders on every request. Use for personalized/auth content.
- **SSG**: Page built once at build time. Use `generateStaticParams`.
- **ISR**: Page regenerates after a time interval. Add `revalidate`.

```tsx
// Static generation at build time
export async function generateStaticParams() {
  const supabase = await createClient();
  const { data: posts } = await supabase.from("posts").select("slug");
  return (posts ?? []).map((post) => ({ slug: post.slug }));
}

// ISR — regenerate every 60 seconds
export const revalidate = 60;
```

## Route Groups

Folders wrapped in parentheses `(name)` organize routes without affecting the URL.

```
app/
  (marketing)/
    page.tsx          → /
    about/page.tsx    → /about
    layout.tsx        → shared marketing layout
  (app)/
    dashboard/page.tsx → /dashboard
    settings/page.tsx  → /settings
    layout.tsx         → shared app layout (with sidebar)
```
