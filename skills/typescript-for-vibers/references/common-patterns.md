# TypeScript Patterns for Next.js + Supabase

## Page Component Props

### Static Page (No Params)

```tsx
// app/about/page.tsx
export default function AboutPage() {
  return <h1>About</h1>;
}
```

### Dynamic Page with Params

```tsx
// app/posts/[slug]/page.tsx
export default async function PostPage({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;
  return <h1>Post: {slug}</h1>;
}
```

### Page with Multiple Dynamic Segments

```tsx
// app/teams/[teamId]/projects/[projectId]/page.tsx
export default async function ProjectPage({
  params,
}: {
  params: Promise<{ teamId: string; projectId: string }>;
}) {
  const { teamId, projectId } = await params;
  return <h1>Team {teamId} / Project {projectId}</h1>;
}
```

### Page with Search Params

```tsx
// app/search/page.tsx
export default async function SearchPage({
  searchParams,
}: {
  searchParams: Promise<{
    q?: string;
    page?: string;
    sort?: string;
    category?: string;
  }>;
}) {
  const { q, page, sort, category } = await searchParams;
  const currentPage = parseInt(page ?? "1", 10);

  return <h1>Search: {q}, Page: {currentPage}</h1>;
}
```

### Page with Both Params and Search Params

```tsx
// app/categories/[category]/page.tsx
export default async function CategoryPage({
  params,
  searchParams,
}: {
  params: Promise<{ category: string }>;
  searchParams: Promise<{ page?: string; sort?: string }>;
}) {
  const { category } = await params;
  const { page, sort } = await searchParams;

  return <h1>Category: {category}</h1>;
}
```

### Catch-All Route Params

```tsx
// app/docs/[...slug]/page.tsx
export default async function DocsPage({
  params,
}: {
  params: Promise<{ slug: string[] }>;
}) {
  const { slug } = await params;
  // /docs/intro → slug = ["intro"]
  // /docs/guides/setup → slug = ["guides", "setup"]

  return <h1>Docs: {slug.join(" / ")}</h1>;
}
```

### Optional Catch-All Route Params

```tsx
// app/shop/[[...categories]]/page.tsx
export default async function ShopPage({
  params,
}: {
  params: Promise<{ categories?: string[] }>;
}) {
  const { categories } = await params;
  // /shop → categories = undefined
  // /shop/shoes → categories = ["shoes"]

  return <h1>{categories ? categories.join(" > ") : "All Products"}</h1>;
}
```

## Layout Props

```tsx
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="flex">
      <aside className="w-64">Sidebar</aside>
      <main className="flex-1">{children}</main>
    </div>
  );
}
```

### Layout with Parallel Routes

```tsx
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children,
  stats,
  activity,
}: {
  children: React.ReactNode;
  stats: React.ReactNode;
  activity: React.ReactNode;
}) {
  return (
    <div>
      {children}
      <div className="grid grid-cols-2 gap-4">
        {stats}
        {activity}
      </div>
    </div>
  );
}
```

## Route Handler Types

### GET with Query Params

```ts
// app/api/posts/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";

export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url);
  const page = parseInt(searchParams.get("page") ?? "1", 10);
  const limit = Math.min(parseInt(searchParams.get("limit") ?? "20", 10), 100);
  const offset = (page - 1) * limit;

  const supabase = await createClient();
  const { data, error, count } = await supabase
    .from("posts")
    .select("*", { count: "exact" })
    .range(offset, offset + limit - 1);

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 });
  }

  return NextResponse.json({ data, total: count, page, limit });
}
```

### POST with Body Validation

```ts
// app/api/posts/route.ts
import { NextRequest, NextResponse } from "next/server";
import { z } from "zod";
import { createClient } from "@/lib/supabase/server";

const createPostSchema = z.object({
  title: z.string().min(1).max(200),
  body: z.string().optional().default(""),
  published: z.boolean().optional().default(false),
});

export async function POST(request: NextRequest) {
  let rawBody: unknown;
  try {
    rawBody = await request.json();
  } catch {
    return NextResponse.json({ error: "Invalid JSON" }, { status: 400 });
  }

  const parsed = createPostSchema.safeParse(rawBody);
  if (!parsed.success) {
    return NextResponse.json(
      { error: "Validation failed", details: parsed.error.flatten().fieldErrors },
      { status: 400 }
    );
  }

  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();

  if (!user) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const { data, error } = await supabase
    .from("posts")
    .insert({ ...parsed.data, author_id: user.id })
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
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";

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

  if (error || !data) {
    return NextResponse.json({ error: "Not found" }, { status: 404 });
  }

  return NextResponse.json({ data });
}

export async function PATCH(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;
  const body = await request.json();

  const supabase = await createClient();
  const { data, error } = await supabase
    .from("posts")
    .update(body)
    .eq("id", id)
    .select()
    .single();

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 });
  }

  return NextResponse.json({ data });
}

export async function DELETE(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;
  const supabase = await createClient();

  const { error } = await supabase.from("posts").delete().eq("id", id);

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 });
  }

  return new NextResponse(null, { status: 204 });
}
```

## Server Action Types

### Simple Action

```ts
// app/actions/posts.ts
"use server";

import { createClient } from "@/lib/supabase/server";
import { revalidatePath } from "next/cache";

export async function togglePublished(postId: string, published: boolean) {
  const supabase = await createClient();

  const { error } = await supabase
    .from("posts")
    .update({ published })
    .eq("id", postId);

  if (error) {
    return { error: error.message };
  }

  revalidatePath("/posts");
  return { success: true };
}
```

### Action with Discriminated Union Return

```ts
// app/actions/auth.ts
"use server";

type AuthResult =
  | { success: true }
  | { success: false; error: string; field?: "email" | "password" };

export async function login(formData: FormData): Promise<AuthResult> {
  const email = formData.get("email") as string;
  const password = formData.get("password") as string;

  if (!email) {
    return { success: false, error: "Email is required", field: "email" };
  }

  if (!password || password.length < 8) {
    return { success: false, error: "Password must be 8+ characters", field: "password" };
  }

  const supabase = await createClient();
  const { error } = await supabase.auth.signInWithPassword({ email, password });

  if (error) {
    return { success: false, error: "Invalid email or password" };
  }

  return { success: true };
}
```

### Using Action Result in Client Component

```tsx
// app/login/login-form.tsx
"use client";

import { useState } from "react";
import { login } from "@/app/actions/auth";

export function LoginForm() {
  const [error, setError] = useState("");
  const [errorField, setErrorField] = useState<string | undefined>();

  async function handleSubmit(formData: FormData) {
    setError("");
    setErrorField(undefined);

    const result = await login(formData);

    if (!result.success) {
      setError(result.error);
      setErrorField(result.field);
    }
    // If success, middleware will redirect
  }

  return (
    <form action={handleSubmit} className="space-y-4">
      <div>
        <input
          name="email"
          type="email"
          placeholder="Email"
          className={`w-full rounded border p-3 ${errorField === "email" ? "border-red-500" : ""}`}
        />
      </div>
      <div>
        <input
          name="password"
          type="password"
          placeholder="Password"
          className={`w-full rounded border p-3 ${errorField === "password" ? "border-red-500" : ""}`}
        />
      </div>
      {error && <p className="text-sm text-red-500">{error}</p>}
      <button type="submit" className="w-full rounded bg-blue-500 py-3 text-white">
        Log In
      </button>
    </form>
  );
}
```

## Component Props Patterns

### Simple Props

```tsx
interface ButtonProps {
  label: string;
  onClick: () => void;
  variant?: "primary" | "secondary" | "danger";
  disabled?: boolean;
}

export function Button({ label, onClick, variant = "primary", disabled = false }: ButtonProps) {
  const styles = {
    primary: "bg-blue-500 text-white",
    secondary: "bg-gray-200 text-gray-800",
    danger: "bg-red-500 text-white",
  };

  return (
    <button
      onClick={onClick}
      disabled={disabled}
      className={`rounded px-4 py-2 ${styles[variant]} disabled:opacity-50`}
    >
      {label}
    </button>
  );
}
```

### Props with Children

```tsx
interface CardProps {
  title: string;
  children: React.ReactNode;
  footer?: React.ReactNode;
}

export function Card({ title, children, footer }: CardProps) {
  return (
    <div className="rounded-lg border shadow-sm">
      <div className="border-b p-4">
        <h3 className="text-lg font-semibold">{title}</h3>
      </div>
      <div className="p-4">{children}</div>
      {footer && <div className="border-t bg-gray-50 p-4">{footer}</div>}
    </div>
  );
}
```

### Generic Component Props

```tsx
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
  emptyMessage?: string;
}

export function List<T extends { id: string }>({
  items,
  renderItem,
  emptyMessage = "No items",
}: ListProps<T>) {
  if (items.length === 0) {
    return <p className="text-gray-500">{emptyMessage}</p>;
  }

  return (
    <ul className="space-y-2">
      {items.map((item) => (
        <li key={item.id}>{renderItem(item)}</li>
      ))}
    </ul>
  );
}

// Usage
<List
  items={posts}
  renderItem={(post) => <span>{post.title}</span>}
  emptyMessage="No posts yet"
/>
```

## Supabase Query Types

### Type-Safe Queries with Generated Types

```ts
import type { Database } from "@/lib/database.types";

type Tables = Database["public"]["Tables"];

// Row type (what you get from SELECT)
type Post = Tables["posts"]["Row"];

// Insert type (what you send to INSERT — id and created_at are optional)
type NewPost = Tables["posts"]["Insert"];

// Update type (everything optional — you only send what changed)
type PostUpdate = Tables["posts"]["Update"];
```

### Typing Supabase Query Results with Joins

```ts
// When you use .select() with joins, the return type can be complex.
// Define the shape you expect:

interface PostWithAuthor {
  id: string;
  title: string;
  body: string;
  created_at: string;
  author: {
    full_name: string;
    avatar_url: string | null;
  } | null;
}

// Use in your component
const { data } = await supabase
  .from("posts")
  .select(`
    id,
    title,
    body,
    created_at,
    author:profiles!author_id (
      full_name,
      avatar_url
    )
  `)
  .returns<PostWithAuthor[]>();
```

## Metadata Types

```tsx
// app/posts/[slug]/page.tsx
import type { Metadata } from "next";
import { createClient } from "@/lib/supabase/server";

export async function generateMetadata({
  params,
}: {
  params: Promise<{ slug: string }>;
}): Promise<Metadata> {
  const { slug } = await params;
  const supabase = await createClient();
  const { data: post } = await supabase
    .from("posts")
    .select("title, excerpt")
    .eq("slug", slug)
    .single();

  if (!post) {
    return { title: "Not Found" };
  }

  return {
    title: post.title,
    description: post.excerpt,
    openGraph: { title: post.title, description: post.excerpt },
  };
}
```

## Middleware Types

```ts
// middleware.ts
import { type NextRequest, NextResponse } from "next/server";

export async function middleware(request: NextRequest): Promise<NextResponse> {
  // request.nextUrl — URL object
  // request.cookies — cookie store
  // request.headers — headers
  // request.geo — { city, country, region } (Vercel only)

  return NextResponse.next();
}
```
