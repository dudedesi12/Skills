# Data Fetching Patterns

## Three Ways to Fetch Data in App Router

1. **Server Components** — fetch directly in the component (best for page loads)
2. **Route Handlers** — traditional API endpoints (best for webhooks, external clients)
3. **Server Actions** — server-side functions called from forms/buttons (best for mutations)

## Supabase Client Setup

You need three different Supabase clients for different contexts.

### Server Component Client

```ts
// lib/supabase/server.ts
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function createClient() {
  const cookieStore = await cookies();

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            );
          } catch {
            // Ignore errors in Server Components (read-only cookies)
          }
        },
      },
    }
  );
}
```

### Browser Client

```ts
// lib/supabase/client.ts
import { createBrowserClient } from "@supabase/ssr";

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

## Pattern 1: Server Component Fetching

The simplest and most performant approach. The component is async, fetches on the server, and sends only HTML.

### Basic Query

```tsx
// app/products/page.tsx
import { createClient } from "@/lib/supabase/server";

export default async function ProductsPage() {
  const supabase = await createClient();

  const { data: products, error } = await supabase
    .from("products")
    .select("id, name, price, image_url")
    .eq("active", true)
    .order("name");

  if (error) {
    throw new Error(`Failed to load products: ${error.message}`);
  }

  return (
    <div className="grid grid-cols-3 gap-6 p-8">
      {products.map((product) => (
        <div key={product.id} className="rounded border p-4">
          <img src={product.image_url} alt={product.name} className="h-48 w-full rounded object-cover" />
          <h2 className="mt-2 text-lg font-semibold">{product.name}</h2>
          <p className="text-green-600 font-bold">${(product.price / 100).toFixed(2)}</p>
        </div>
      ))}
    </div>
  );
}
```

### With Related Data (Joins)

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
    .select(`
      id,
      title,
      body,
      created_at,
      author:profiles!author_id (
        full_name,
        avatar_url
      ),
      comments (
        id,
        body,
        created_at,
        commenter:profiles!user_id (full_name)
      )
    `)
    .eq("slug", slug)
    .single();

  if (error || !post) {
    notFound();
  }

  return (
    <article className="mx-auto max-w-2xl p-8">
      <h1 className="text-3xl font-bold">{post.title}</h1>
      <p className="mt-1 text-sm text-gray-500">
        by {post.author?.full_name} on {new Date(post.created_at).toLocaleDateString()}
      </p>
      <div className="mt-6">{post.body}</div>

      <section className="mt-10">
        <h2 className="text-xl font-semibold">Comments ({post.comments.length})</h2>
        {post.comments.map((comment) => (
          <div key={comment.id} className="mt-4 rounded border p-3">
            <p className="font-medium">{comment.commenter?.full_name}</p>
            <p className="text-gray-700">{comment.body}</p>
          </div>
        ))}
      </section>
    </article>
  );
}
```

### With Search Params (Filtering)

```tsx
// app/search/page.tsx
import { createClient } from "@/lib/supabase/server";

export default async function SearchPage({
  searchParams,
}: {
  searchParams: Promise<{ q?: string; category?: string; page?: string }>;
}) {
  const { q, category, page } = await searchParams;
  const currentPage = parseInt(page ?? "1", 10);
  const pageSize = 20;
  const offset = (currentPage - 1) * pageSize;

  const supabase = await createClient();

  let query = supabase
    .from("products")
    .select("*", { count: "exact" })
    .range(offset, offset + pageSize - 1)
    .order("created_at", { ascending: false });

  if (q) {
    query = query.ilike("name", `%${q}%`);
  }

  if (category) {
    query = query.eq("category", category);
  }

  const { data: products, count, error } = await query;

  if (error) {
    throw new Error("Search failed");
  }

  const totalPages = Math.ceil((count ?? 0) / pageSize);

  return (
    <div className="p-8">
      <p className="text-sm text-gray-500">{count} results found</p>
      <div className="mt-4 grid grid-cols-3 gap-4">
        {products?.map((p) => (
          <div key={p.id} className="rounded border p-4">{p.name}</div>
        ))}
      </div>
      <div className="mt-6 flex gap-2">
        {Array.from({ length: totalPages }, (_, i) => (
          <a
            key={i}
            href={`/search?q=${q ?? ""}&page=${i + 1}`}
            className={`rounded px-3 py-1 ${
              currentPage === i + 1 ? "bg-blue-500 text-white" : "bg-gray-100"
            }`}
          >
            {i + 1}
          </a>
        ))}
      </div>
    </div>
  );
}
```

## Pattern 2: Route Handlers

Traditional API endpoints. Use for webhooks, third-party integrations, or when you need fine-grained control over the response.

### CRUD API

```ts
// app/api/posts/route.ts
import { createClient } from "@/lib/supabase/server";
import { NextRequest, NextResponse } from "next/server";

export async function GET(request: NextRequest) {
  const supabase = await createClient();
  const { searchParams } = new URL(request.url);
  const limit = Math.min(parseInt(searchParams.get("limit") ?? "20", 10), 100);

  const { data, error } = await supabase
    .from("posts")
    .select("id, title, slug, created_at")
    .order("created_at", { ascending: false })
    .limit(limit);

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 });
  }

  return NextResponse.json({ data });
}

export async function POST(request: NextRequest) {
  const supabase = await createClient();

  const { data: { user } } = await supabase.auth.getUser();
  if (!user) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  let body: { title?: string; content?: string };
  try {
    body = await request.json();
  } catch {
    return NextResponse.json({ error: "Invalid JSON body" }, { status: 400 });
  }

  if (!body.title || body.title.trim().length < 3) {
    return NextResponse.json({ error: "Title must be at least 3 characters" }, { status: 400 });
  }

  const { data, error } = await supabase
    .from("posts")
    .insert({
      title: body.title.trim(),
      content: body.content?.trim() ?? "",
      author_id: user.id,
    })
    .select()
    .single();

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 });
  }

  return NextResponse.json({ data }, { status: 201 });
}
```

### Dynamic Route Handler

```ts
// app/api/posts/[id]/route.ts
import { createClient } from "@/lib/supabase/server";
import { NextRequest, NextResponse } from "next/server";

export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;
  const supabase = await createClient();

  const { data, error } = await supabase
    .from("posts")
    .select("*")
    .eq("id", id)
    .single();

  if (error) {
    return NextResponse.json({ error: "Post not found" }, { status: 404 });
  }

  return NextResponse.json({ data });
}

export async function DELETE(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;
  const supabase = await createClient();

  const { data: { user } } = await supabase.auth.getUser();
  if (!user) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const { error } = await supabase
    .from("posts")
    .delete()
    .eq("id", id)
    .eq("author_id", user.id);

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 });
  }

  return NextResponse.json({ success: true });
}
```

### Webhook Handler

```ts
// app/api/webhooks/stripe/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@supabase/supabase-js";

// Use service role for webhook handlers (no user session)
function getAdminClient() {
  const url = process.env.NEXT_PUBLIC_SUPABASE_URL;
  const key = process.env.SUPABASE_SERVICE_ROLE_KEY;
  if (!url || !key) throw new Error("Missing Supabase env vars");
  return createClient(url, key);
}

export async function POST(request: NextRequest) {
  const body = await request.text();
  const signature = request.headers.get("stripe-signature");

  if (!signature) {
    return NextResponse.json({ error: "Missing signature" }, { status: 400 });
  }

  // Verify webhook signature here (using Stripe SDK)
  // const event = stripe.webhooks.constructEvent(body, signature, webhookSecret);

  const supabase = getAdminClient();

  // Process the event
  // await supabase.from("payments").insert({ ... });

  return NextResponse.json({ received: true });
}
```

## Pattern 3: Server Actions

Server Actions are functions that run on the server but can be called from Client Components or forms. Best for mutations (creating, updating, deleting data).

### Form with Server Action

```tsx
// app/feedback/page.tsx
import { createClient } from "@/lib/supabase/server";
import { revalidatePath } from "next/cache";
import { redirect } from "next/navigation";

export default function FeedbackPage() {
  async function submitFeedback(formData: FormData) {
    "use server";

    const message = formData.get("message") as string;
    const rating = parseInt(formData.get("rating") as string, 10);

    if (!message || message.trim().length < 10) {
      throw new Error("Feedback must be at least 10 characters");
    }

    if (isNaN(rating) || rating < 1 || rating > 5) {
      throw new Error("Rating must be between 1 and 5");
    }

    const supabase = await createClient();
    const { data: { user } } = await supabase.auth.getUser();

    if (!user) {
      redirect("/login");
    }

    const { error } = await supabase.from("feedback").insert({
      message: message.trim(),
      rating,
      user_id: user.id,
    });

    if (error) {
      throw new Error("Failed to submit feedback");
    }

    revalidatePath("/feedback");
    redirect("/feedback/thanks");
  }

  return (
    <form action={submitFeedback} className="mx-auto max-w-md space-y-4 p-8">
      <textarea
        name="message"
        placeholder="Your feedback..."
        required
        minLength={10}
        rows={4}
        className="w-full rounded border p-3"
      />
      <select name="rating" required className="w-full rounded border p-3">
        <option value="">Select rating</option>
        {[1, 2, 3, 4, 5].map((n) => (
          <option key={n} value={n}>{n} star{n > 1 ? "s" : ""}</option>
        ))}
      </select>
      <button type="submit" className="w-full rounded bg-blue-500 py-3 text-white">
        Submit Feedback
      </button>
    </form>
  );
}
```

### Server Action in Separate File

```ts
// app/actions/posts.ts
"use server";

import { createClient } from "@/lib/supabase/server";
import { revalidatePath } from "next/cache";
import { redirect } from "next/navigation";

export async function createPost(formData: FormData) {
  const title = formData.get("title") as string;
  const body = formData.get("body") as string;

  if (!title || title.trim().length < 3) {
    return { error: "Title must be at least 3 characters" };
  }

  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();

  if (!user) {
    redirect("/login");
  }

  const slug = title
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, "-")
    .replace(/(^-|-$)/g, "");

  const { error } = await supabase.from("posts").insert({
    title: title.trim(),
    body: body?.trim() ?? "",
    slug,
    author_id: user.id,
  });

  if (error) {
    return { error: "Failed to create post. Title may already exist." };
  }

  revalidatePath("/posts");
  redirect(`/posts/${slug}`);
}

export async function deletePost(postId: string) {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();

  if (!user) {
    return { error: "Not authenticated" };
  }

  const { error } = await supabase
    .from("posts")
    .delete()
    .eq("id", postId)
    .eq("author_id", user.id);

  if (error) {
    return { error: "Failed to delete post" };
  }

  revalidatePath("/posts");
  redirect("/posts");
}
```

### Calling Server Action from Client Component

```tsx
// app/components/delete-button.tsx
"use client";

import { useState } from "react";
import { deletePost } from "@/app/actions/posts";

export function DeleteButton({ postId }: { postId: string }) {
  const [pending, setPending] = useState(false);

  async function handleDelete() {
    if (!confirm("Are you sure you want to delete this post?")) return;

    setPending(true);
    const result = await deletePost(postId);

    if (result?.error) {
      alert(result.error);
      setPending(false);
    }
  }

  return (
    <button
      onClick={handleDelete}
      disabled={pending}
      className="rounded bg-red-500 px-4 py-2 text-white disabled:opacity-50"
    >
      {pending ? "Deleting..." : "Delete"}
    </button>
  );
}
```

## Revalidation Patterns

### Time-Based Revalidation (ISR)

```tsx
// app/blog/page.tsx
// Re-fetch this page's data every 60 seconds
export const revalidate = 60;

export default async function BlogPage() {
  const supabase = await createClient();
  const { data } = await supabase.from("posts").select("*");
  return <div>{/* render posts */}</div>;
}
```

### On-Demand Revalidation

```ts
// app/actions/posts.ts
"use server";

import { revalidatePath } from "next/cache";
import { revalidateTag } from "next/cache";

export async function publishPost(formData: FormData) {
  // ... insert post ...

  // Revalidate a specific page
  revalidatePath("/blog");

  // Revalidate a specific dynamic page
  revalidatePath("/blog/my-new-post");

  // Revalidate by tag (if using fetch with tags)
  revalidateTag("posts");
}
```

## Parallel Data Fetching

When your page needs data from multiple sources, fetch them all at once instead of one after another.

```tsx
// app/dashboard/page.tsx
import { createClient } from "@/lib/supabase/server";

export default async function DashboardPage() {
  const supabase = await createClient();

  // Fetch everything in parallel — much faster
  const [postsResult, commentsResult, statsResult] = await Promise.all([
    supabase.from("posts").select("*").order("created_at", { ascending: false }).limit(5),
    supabase.from("comments").select("*").order("created_at", { ascending: false }).limit(10),
    supabase.from("posts").select("*", { count: "exact", head: true }),
  ]);

  if (postsResult.error || commentsResult.error) {
    throw new Error("Failed to load dashboard data");
  }

  return (
    <div className="grid grid-cols-2 gap-8 p-8">
      <section>
        <h2 className="text-xl font-bold">Recent Posts ({statsResult.count})</h2>
        {postsResult.data.map((post) => (
          <div key={post.id} className="mt-2 rounded border p-3">{post.title}</div>
        ))}
      </section>
      <section>
        <h2 className="text-xl font-bold">Recent Comments</h2>
        {commentsResult.data.map((comment) => (
          <div key={comment.id} className="mt-2 rounded border p-3">{comment.body}</div>
        ))}
      </section>
    </div>
  );
}
```
