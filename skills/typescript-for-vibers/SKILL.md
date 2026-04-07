---
name: typescript-for-vibers
description: "Use this skill whenever the user encounters a TypeScript error, mentions types, interfaces, generics, Zod, validation, type safety, 'red squiggly lines', type errors, 'TS error', type definitions, enum, union types, Partial, Pick, Omit, Record, type narrowing, type guards, 'any' type, type assertion, generated types, database types, or asks to fix any TypeScript compilation issue — even if they just say 'it has errors' or 'something is broken'. This skill makes TypeScript work FOR the vibe coder."
---

# TypeScript for Vibers

TypeScript adds types to JavaScript. Types tell you (and your editor) what shape your data has — so you get autocomplete and catch bugs before they happen. This skill covers everything you need to make TypeScript work for you instead of against you.

## Turning JSON into a Type

Got some JSON data? Here is how to turn it into a TypeScript type.

### Step 1: Look at the data

```json
{
  "id": "abc123",
  "title": "My First Post",
  "published": true,
  "tags": ["react", "nextjs"],
  "author": {
    "name": "Jane",
    "email": "jane@example.com"
  }
}
```

### Step 2: Write the interface

```ts
// lib/types.ts
interface Author {
  name: string;
  email: string;
}

interface Post {
  id: string;
  title: string;
  published: boolean;
  tags: string[];
  author: Author;
}
```

### Rules of thumb
- Strings → `string`
- Numbers → `number`
- true/false → `boolean`
- Arrays of strings → `string[]`
- Arrays of objects → `SomeType[]`
- Nested objects → Create another interface
- Could be missing → Add `?` like `bio?: string`
- Could be null → `string | null`

## Interface vs Type

Both work. Use `interface` for object shapes. Use `type` for unions, intersections, and simpler aliases.

```ts
// Interface — for object shapes (most common)
interface User {
  id: string;
  name: string;
  email: string;
}

// Type — for unions, aliases, computed types
type Status = "active" | "inactive" | "banned";
type UserWithStatus = User & { status: Status };
```

## Unions Instead of Enums

Prefer string unions over enums. They are simpler and work better with databases.

```ts
// BAD — enums add runtime code and are harder to use with Supabase
enum Status {
  Active = "active",
  Inactive = "inactive",
}

// GOOD — string unions are simpler
type Status = "active" | "inactive" | "banned";

// Use in an interface
interface User {
  id: string;
  status: Status;
}

// TypeScript will autocomplete and validate
const user: User = {
  id: "123",
  status: "active", // Only "active" | "inactive" | "banned" allowed
};
```

## Supabase Generated Types

Supabase can generate TypeScript types from your database schema. This means your types always match your database.

### Generate Types

```bash
npx supabase gen types typescript --project-id YOUR_PROJECT_ID > lib/database.types.ts
# Or from local:
npx supabase gen types typescript --local > lib/database.types.ts
```

### Use Generated Types

```ts
// lib/database.types.ts (auto-generated — do NOT edit by hand)
export type Database = {
  public: {
    Tables: {
      posts: {
        Row: {
          id: string;
          title: string;
          body: string;
          published: boolean;
          author_id: string;
          created_at: string;
        };
        Insert: {
          id?: string;
          title: string;
          body?: string;
          published?: boolean;
          author_id: string;
          created_at?: string;
        };
        Update: {
          id?: string;
          title?: string;
          body?: string;
          published?: boolean;
          author_id?: string;
          created_at?: string;
        };
      };
    };
  };
};
```

### Type Your Supabase Client

```ts
// lib/supabase/server.ts
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";
import type { Database } from "@/lib/database.types";

export async function createClient() {
  const cookieStore = await cookies();

  return createServerClient<Database>(
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
            // Read-only in Server Components
          }
        },
      },
    }
  );
}
```

Now all your queries are fully typed:

```tsx
// app/posts/page.tsx
const supabase = await createClient();

// TypeScript knows the exact shape of posts!
const { data: posts, error } = await supabase
  .from("posts")            // autocomplete table names
  .select("id, title")      // autocomplete column names
  .eq("published", true);   // type-checked filter values

// posts is typed as { id: string; title: string }[] | null
```

### Helper Types from Generated Types

```ts
// lib/types.ts
import type { Database } from "@/lib/database.types";

// Extract the Row type for a table
type Post = Database["public"]["Tables"]["posts"]["Row"];
type PostInsert = Database["public"]["Tables"]["posts"]["Insert"];
type PostUpdate = Database["public"]["Tables"]["posts"]["Update"];

// Use them in your components
interface PostCardProps {
  post: Post;
}
```

## Utility Types Explained Simply

### Partial — Makes everything optional

```ts
interface User {
  id: string;
  name: string;
  email: string;
}

// Partial<User> is the same as:
// { id?: string; name?: string; email?: string }

// Use it when updating (you might only change some fields)
async function updateUser(id: string, changes: Partial<User>) {
  const { error } = await supabase
    .from("users")
    .update(changes) // Can pass { name: "New Name" } without email
    .eq("id", id);

  if (error) throw new Error(error.message);
}
```

### Pick — Take only some fields

```ts
interface User {
  id: string;
  name: string;
  email: string;
  password_hash: string;
  created_at: string;
}

// Pick<User, "id" | "name" | "email"> is the same as:
// { id: string; name: string; email: string }

// Use for API responses (never expose password_hash)
type PublicUser = Pick<User, "id" | "name" | "email">;
```

### Omit — Remove some fields

```ts
// Omit<User, "password_hash" | "created_at"> is the same as:
// { id: string; name: string; email: string }

type SafeUser = Omit<User, "password_hash">;
```

### Record — Object with known key types

```ts
// Record<string, number> means an object where keys are strings and values are numbers
const scores: Record<string, number> = {
  alice: 95,
  bob: 87,
  charlie: 92,
};

// Record is great for lookup objects
type PermissionMap = Record<string, boolean>;
const permissions: PermissionMap = {
  canEdit: true,
  canDelete: false,
  canInvite: true,
};
```

## Common Next.js TypeScript Patterns

### Page Props

```tsx
// app/posts/[slug]/page.tsx
export default async function PostPage({
  params,
  searchParams,
}: {
  params: Promise<{ slug: string }>;
  searchParams: Promise<{ page?: string; sort?: string }>;
}) {
  const { slug } = await params;
  const { page, sort } = await searchParams;

  return <div>Post: {slug}, Page: {page ?? "1"}</div>;
}
```

### Layout Props

```tsx
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="flex">
      <aside>Sidebar</aside>
      <main>{children}</main>
    </div>
  );
}
```

### Server Action Types

```ts
// app/actions/posts.ts
"use server";

// Return type for actions that might fail
type ActionResult =
  | { success: true; data: { id: string } }
  | { success: false; error: string };

export async function createPost(formData: FormData): Promise<ActionResult> {
  const title = formData.get("title") as string;

  if (!title || title.length < 3) {
    return { success: false, error: "Title must be at least 3 characters" };
  }

  const supabase = await createClient();
  const { data, error } = await supabase
    .from("posts")
    .insert({ title })
    .select("id")
    .single();

  if (error) {
    return { success: false, error: error.message };
  }

  return { success: true, data: { id: data.id } };
}
```

### Route Handler Types

```ts
// app/api/posts/route.ts
import { NextRequest, NextResponse } from "next/server";

interface CreatePostBody {
  title: string;
  body?: string;
}

export async function POST(request: NextRequest) {
  let body: CreatePostBody;

  try {
    body = await request.json();
  } catch {
    return NextResponse.json({ error: "Invalid JSON" }, { status: 400 });
  }

  if (!body.title || typeof body.title !== "string") {
    return NextResponse.json({ error: "title is required" }, { status: 400 });
  }

  // body.title is now guaranteed to be a string
  const supabase = await createClient();
  const { data, error } = await supabase
    .from("posts")
    .insert({ title: body.title, body: body.body ?? "" })
    .select()
    .single();

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 });
  }

  return NextResponse.json(data, { status: 201 });
}
```

## Generics — The Simple Version

Generics let you write a function that works with different types. Think of `<T>` as a placeholder that gets filled in when you use the function.

```ts
// Without generics — you'd need a separate function for each type
function getFirstPost(items: Post[]): Post | undefined {
  return items[0];
}
function getFirstUser(items: User[]): User | undefined {
  return items[0];
}

// With generics — one function works for any type
function getFirst<T>(items: T[]): T | undefined {
  return items[0];
}

// TypeScript figures out T automatically
const firstPost = getFirst(posts);   // T = Post, returns Post | undefined
const firstUser = getFirst(users);   // T = User, returns User | undefined
```

### Practical Generic: API Response Wrapper

```ts
// lib/types.ts
interface ApiResponse<T> {
  data: T | null;
  error: string | null;
}

// Works with any data type
async function fetchFromApi<T>(url: string): Promise<ApiResponse<T>> {
  try {
    const response = await fetch(url);

    if (!response.ok) {
      return { data: null, error: `HTTP ${response.status}` };
    }

    const data: T = await response.json();
    return { data, error: null };
  } catch (err) {
    return { data: null, error: "Network error" };
  }
}

// Usage — TypeScript knows the exact return type
const result = await fetchFromApi<Post[]>("/api/posts");
if (result.data) {
  // result.data is Post[]
  console.log(result.data[0].title);
}
```

## Type Narrowing and Guards

Type narrowing is when TypeScript figures out a more specific type based on a check you do.

```ts
// TypeScript narrows the type after each check
function processValue(value: string | number | null) {
  if (value === null) {
    // TypeScript knows: value is null
    return "No value";
  }

  if (typeof value === "string") {
    // TypeScript knows: value is string
    return value.toUpperCase();
  }

  // TypeScript knows: value is number
  return value * 2;
}
```

### Custom Type Guard

```ts
interface Post {
  type: "post";
  title: string;
  body: string;
}

interface Comment {
  type: "comment";
  body: string;
  postId: string;
}

type Content = Post | Comment;

// Type guard function — the "is Post" tells TypeScript what the check means
function isPost(content: Content): content is Post {
  return content.type === "post";
}

function renderContent(content: Content) {
  if (isPost(content)) {
    // TypeScript knows: content is Post
    return <h1>{content.title}</h1>;
  }

  // TypeScript knows: content is Comment
  return <p>Comment on post {content.postId}</p>;
}
```

## Zod for Runtime Validation

TypeScript types only exist at compile time — they disappear when your code runs. Zod validates data at runtime (when your app is actually running).

```ts
// lib/schemas.ts
import { z } from "zod";

// Define the schema
const createPostSchema = z.object({
  title: z.string().min(3, "Title must be at least 3 characters").max(200),
  body: z.string().optional().default(""),
  published: z.boolean().optional().default(false),
  tags: z.array(z.string()).optional().default([]),
});

// Get the TypeScript type from the schema (no duplication!)
type CreatePostInput = z.infer<typeof createPostSchema>;

// Use in a server action
export async function createPost(formData: FormData) {
  "use server";

  const raw = {
    title: formData.get("title"),
    body: formData.get("body"),
    published: formData.get("published") === "true",
  };

  const result = createPostSchema.safeParse(raw);

  if (!result.success) {
    // result.error.flatten() gives you field-by-field errors
    return { error: result.error.flatten().fieldErrors };
  }

  // result.data is fully typed and validated
  const { title, body, published } = result.data;

  const supabase = await createClient();
  const { error } = await supabase.from("posts").insert({ title, body, published });

  if (error) {
    return { error: { _form: [error.message] } };
  }
}
```

### Validate Environment Variables with Zod

```ts
// lib/env.ts
import { z } from "zod";

const envSchema = z.object({
  NEXT_PUBLIC_SUPABASE_URL: z.string().url(),
  NEXT_PUBLIC_SUPABASE_ANON_KEY: z.string().min(1),
  SUPABASE_SERVICE_ROLE_KEY: z.string().min(1),
  GEMINI_API_KEY: z.string().min(1),
});

// This will throw at startup if any env var is missing
export const env = envSchema.parse(process.env);
```

## The Non-Null Assertion Operator (!)

The `!` after a value tells TypeScript "trust me, this is not null/undefined." Use it sparingly and only when you are sure.

```ts
// TypeScript says: might be undefined
const url = process.env.NEXT_PUBLIC_SUPABASE_URL; // string | undefined

// The ! says: I'm sure this exists
const supabase = createClient(process.env.NEXT_PUBLIC_SUPABASE_URL!);

// BETTER: validate and throw a helpful error
function getEnv(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing env var: ${name}`);
  return value;
}
```

## Type Assertion (as)

Use `as` to tell TypeScript to treat a value as a specific type. Use carefully.

```ts
// FormData.get() returns FormDataEntryValue | null
const title = formData.get("title") as string;

// Better: validate first
const rawTitle = formData.get("title");
if (typeof rawTitle !== "string" || rawTitle.length === 0) {
  throw new Error("Title is required");
}
// Now TypeScript knows rawTitle is a string
```

## Event Handler Types

```tsx
"use client";

import { useState } from "react";

export function MyForm() {
  const [value, setValue] = useState("");

  // Input change
  function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
    setValue(e.target.value);
  }

  // Form submit
  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    console.log(value);
  }

  // Button click
  function handleClick(e: React.MouseEvent<HTMLButtonElement>) {
    console.log("Clicked");
  }

  // Textarea change
  function handleTextarea(e: React.ChangeEvent<HTMLTextAreaElement>) {
    console.log(e.target.value);
  }

  // Select change
  function handleSelect(e: React.ChangeEvent<HTMLSelectElement>) {
    console.log(e.target.value);
  }

  return (
    <form onSubmit={handleSubmit}>
      <input value={value} onChange={handleChange} />
      <button type="submit" onClick={handleClick}>Submit</button>
    </form>
  );
}
```

## Common Mistakes

### Using `any` instead of proper types

```ts
// BAD — defeats the purpose of TypeScript
function processData(data: any) {
  return data.name; // No type checking at all
}

// GOOD — use unknown and narrow
function processData(data: unknown) {
  if (typeof data === "object" && data !== null && "name" in data) {
    return (data as { name: string }).name;
  }
  throw new Error("Invalid data");
}
```

### Forgetting to handle null from Supabase

```ts
// BAD — data could be null
const { data } = await supabase.from("posts").select("*");
console.log(data.length); // TypeScript error: data is possibly null

// GOOD — check first
const { data, error } = await supabase.from("posts").select("*");
if (error || !data) {
  throw new Error("Failed to load posts");
}
console.log(data.length); // Now TypeScript knows data is not null
```
