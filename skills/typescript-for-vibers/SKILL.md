---
name: typescript-for-vibers
description: "Use this skill whenever the user encounters a TypeScript error, mentions types, interfaces, generics, Zod, validation, type safety, 'red squiggly lines', type errors, 'TS error', type definitions, enum, union types, Partial, Pick, Omit, Record, type narrowing, type guards, 'any' type, type assertion, generated types, database types, or asks to fix any TypeScript compilation issue — even if they just say 'it has errors' or 'something is broken'. This skill makes TypeScript work FOR the vibe coder."
---

# TypeScript for Vibers

TypeScript adds types to JavaScript so you get autocomplete and catch bugs before they happen. This skill makes TypeScript work for you.

## Turning JSON into a Type

```ts
// lib/types.ts
// From JSON: { "id": "abc", "title": "Hello", "tags": ["react"], "author": { "name": "Jane" } }
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

Rules: strings = `string`, numbers = `number`, true/false = `boolean`, arrays = `Type[]`, might be missing = `bio?: string`, might be null = `string | null`.

## Interface vs Type

Use `interface` for object shapes, `type` for unions and aliases.

```ts
interface User { id: string; name: string; email: string; }
type Status = "active" | "inactive" | "banned";
type UserWithStatus = User & { status: Status };
```

## Unions Instead of Enums

Prefer string unions. They are simpler and work better with databases.

```ts
// GOOD
type Status = "active" | "inactive" | "banned";

// BAD — enums add runtime code
enum Status { Active = "active" }
```

## Supabase Generated Types

```bash
npx supabase gen types typescript --project-id YOUR_ID > lib/database.types.ts
```

Type your client for full autocomplete:

```ts
// lib/supabase/server.ts
import type { Database } from "@/lib/database.types";

export async function createClient() {
  const cookieStore = await cookies();
  return createServerClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    { cookies: { getAll() { return cookieStore.getAll(); }, setAll(c) { try { c.forEach(({ name, value, options }) => cookieStore.set(name, value, options)); } catch {} } } }
  );
}
```

Extract helper types:

```ts
import type { Database } from "@/lib/database.types";
type Post = Database["public"]["Tables"]["posts"]["Row"];
type PostInsert = Database["public"]["Tables"]["posts"]["Insert"];
type PostUpdate = Database["public"]["Tables"]["posts"]["Update"];
```

## Utility Types

### Partial — Makes everything optional

```ts
// Use when updating (only send changed fields)
async function updateUser(id: string, changes: Partial<User>) {
  await supabase.from("users").update(changes).eq("id", id);
}
```

### Pick — Take only some fields

```ts
type PublicUser = Pick<User, "id" | "name" | "email">; // Only these fields
```

### Omit — Remove some fields

```ts
type SafeUser = Omit<User, "password_hash">; // Everything except password_hash
```

### Record — Object with known key types

```ts
const scores: Record<string, number> = { alice: 95, bob: 87 };
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
  searchParams: Promise<{ page?: string }>;
}) {
  const { slug } = await params;
  const { page } = await searchParams;
  return <div>Post: {slug}, Page: {page ?? "1"}</div>;
}
```

### Server Action Return Types

```ts
// app/actions/posts.ts
"use server";

type ActionResult = { success: true; data: { id: string } } | { success: false; error: string };

export async function createPost(formData: FormData): Promise<ActionResult> {
  const title = formData.get("title") as string;
  if (!title || title.length < 3) return { success: false, error: "Title too short" };

  const supabase = await createClient();
  const { data, error } = await supabase.from("posts").insert({ title }).select("id").single();
  if (error) return { success: false, error: error.message };
  return { success: true, data: { id: data.id } };
}
```

### Route Handler Types

```ts
// app/api/posts/route.ts
import { NextRequest, NextResponse } from "next/server";

export async function POST(request: NextRequest) {
  let body: { title?: string };
  try { body = await request.json(); } catch { return NextResponse.json({ error: "Invalid JSON" }, { status: 400 }); }
  if (!body.title) return NextResponse.json({ error: "title required" }, { status: 400 });

  const supabase = await createClient();
  const { data, error } = await supabase.from("posts").insert({ title: body.title }).select().single();
  if (error) return NextResponse.json({ error: error.message }, { status: 500 });
  return NextResponse.json(data, { status: 201 });
}
```

See `references/common-patterns.md` for layout props, dynamic route handlers, metadata types, and component prop patterns.

## Generics

Think of `<T>` as a placeholder that gets filled in when you use the function.

```ts
function getFirst<T>(items: T[]): T | undefined {
  return items[0];
}

const firstPost = getFirst(posts);   // returns Post | undefined
const firstUser = getFirst(users);   // returns User | undefined
```

### Practical Generic: API Response

```ts
interface ApiResponse<T> { data: T | null; error: string | null; }

async function fetchApi<T>(url: string): Promise<ApiResponse<T>> {
  try {
    const res = await fetch(url);
    if (!res.ok) return { data: null, error: `HTTP ${res.status}` };
    return { data: await res.json(), error: null };
  } catch { return { data: null, error: "Network error" }; }
}

const result = await fetchApi<Post[]>("/api/posts");
```

## Type Narrowing

TypeScript figures out more specific types based on your checks.

```ts
function processValue(value: string | number | null) {
  if (value === null) return "No value";
  if (typeof value === "string") return value.toUpperCase();
  return value * 2; // TypeScript knows: number
}
```

### Custom Type Guard

```ts
function isPost(content: Content): content is Post {
  return content.type === "post";
}

if (isPost(content)) {
  console.log(content.title); // TypeScript knows it's a Post
}
```

## Zod for Runtime Validation

TypeScript types disappear at runtime. Zod validates data when your app actually runs.

```ts
import { z } from "zod";

const createPostSchema = z.object({
  title: z.string().min(3, "Title must be 3+ characters").max(200),
  body: z.string().optional().default(""),
  published: z.boolean().optional().default(false),
});

type CreatePostInput = z.infer<typeof createPostSchema>; // Get TS type from schema

// Use in server action
export async function createPost(formData: FormData) {
  "use server";
  const result = createPostSchema.safeParse({
    title: formData.get("title"),
    body: formData.get("body"),
  });

  if (!result.success) return { error: result.error.flatten().fieldErrors };

  const supabase = await createClient();
  const { error } = await supabase.from("posts").insert(result.data);
  if (error) return { error: { _form: [error.message] } };
}
```

### Validate Environment Variables

```ts
// lib/env.ts
import { z } from "zod";

const envSchema = z.object({
  NEXT_PUBLIC_SUPABASE_URL: z.string().url(),
  NEXT_PUBLIC_SUPABASE_ANON_KEY: z.string().min(1),
  SUPABASE_SERVICE_ROLE_KEY: z.string().min(1),
  GEMINI_API_KEY: z.string().min(1),
});

export const env = envSchema.parse(process.env); // Throws at startup if missing
```

See `references/zod-patterns.md` for form validation, API validation, transforms, and Supabase integration.

## Event Handler Types

```tsx
"use client";

function handleChange(e: React.ChangeEvent<HTMLInputElement>) { /* input */ }
function handleSubmit(e: React.FormEvent<HTMLFormElement>) { e.preventDefault(); }
function handleClick(e: React.MouseEvent<HTMLButtonElement>) { /* button */ }
function handleTextarea(e: React.ChangeEvent<HTMLTextAreaElement>) { /* textarea */ }
function handleSelect(e: React.ChangeEvent<HTMLSelectElement>) { /* select */ }
```

## The Non-Null Assertion (!)

Tells TypeScript "trust me, this is not null." Use sparingly.

```ts
const url = process.env.NEXT_PUBLIC_SUPABASE_URL!; // I'm sure it exists

// BETTER: validate and throw helpful error
function getEnv(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing env var: ${name}`);
  return value;
}
```

## Common Mistakes

### Using `any`

```ts
// BAD
function process(data: any) { return data.name; }

// GOOD — use unknown and narrow
function process(data: unknown) {
  if (typeof data === "object" && data !== null && "name" in data) {
    return (data as { name: string }).name;
  }
  throw new Error("Invalid data");
}
```

### Forgetting to handle null from Supabase

```ts
// BAD
const { data } = await supabase.from("posts").select("*");
console.log(data.length); // data might be null!

// GOOD
const { data, error } = await supabase.from("posts").select("*");
if (error || !data) throw new Error("Failed to load");
console.log(data.length); // Safe
```

See `references/error-fixes.md` for the top 20 TypeScript errors with plain English fixes.
