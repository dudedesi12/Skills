# Zod Patterns for Next.js + Supabase

Zod validates data at runtime. TypeScript types disappear when your code runs — Zod schemas actually check the data and throw errors if it is wrong.

## Installation

```bash
npm install zod
```

## Basic Schema Patterns

### Primitives

```ts
import { z } from "zod";

const stringSchema = z.string();
const numberSchema = z.number();
const booleanSchema = z.boolean();
const dateSchema = z.date();

// With constraints
const email = z.string().email("Invalid email address");
const age = z.number().int().min(13, "Must be at least 13").max(120);
const name = z.string().min(1, "Name is required").max(100, "Name too long");
const url = z.string().url("Must be a valid URL");
const uuid = z.string().uuid("Must be a valid UUID");
```

### Objects

```ts
const userSchema = z.object({
  name: z.string().min(1, "Name is required"),
  email: z.string().email("Invalid email"),
  age: z.number().int().min(13).optional(),
  bio: z.string().max(500).optional().default(""),
});

// Get the TypeScript type from the schema
type User = z.infer<typeof userSchema>;
// { name: string; email: string; age?: number; bio?: string }
```

### Arrays

```ts
const tagsSchema = z.array(z.string()).min(1, "At least one tag required").max(10);
const scoresSchema = z.array(z.number().min(0).max(100));
```

### Enums / Unions

```ts
const statusSchema = z.enum(["active", "inactive", "banned"]);
type Status = z.infer<typeof statusSchema>; // "active" | "inactive" | "banned"

const roleSchema = z.union([
  z.literal("admin"),
  z.literal("moderator"),
  z.literal("user"),
]);
```

## Form Validation

### Server Action with Zod

```ts
// app/actions/posts.ts
"use server";

import { z } from "zod";
import { createClient } from "@/lib/supabase/server";
import { revalidatePath } from "next/cache";
import { redirect } from "next/navigation";

const createPostSchema = z.object({
  title: z
    .string()
    .min(3, "Title must be at least 3 characters")
    .max(200, "Title must be under 200 characters"),
  body: z
    .string()
    .min(10, "Body must be at least 10 characters")
    .max(50000, "Body is too long"),
  published: z
    .string()
    .optional()
    .transform((val) => val === "on" || val === "true"),
});

interface FormState {
  errors?: {
    title?: string[];
    body?: string[];
    _form?: string[];
  };
}

export async function createPost(
  prevState: FormState,
  formData: FormData
): Promise<FormState> {
  const raw = {
    title: formData.get("title"),
    body: formData.get("body"),
    published: formData.get("published"),
  };

  const result = createPostSchema.safeParse(raw);

  if (!result.success) {
    return { errors: result.error.flatten().fieldErrors };
  }

  const supabase = await createClient();
  const {
    data: { user },
  } = await supabase.auth.getUser();

  if (!user) {
    return { errors: { _form: ["You must be logged in"] } };
  }

  const { error } = await supabase.from("posts").insert({
    title: result.data.title,
    body: result.data.body,
    published: result.data.published,
    author_id: user.id,
  });

  if (error) {
    return { errors: { _form: [error.message] } };
  }

  revalidatePath("/posts");
  redirect("/posts");
}
```

### Client Component Form with useActionState

```tsx
// app/posts/new/post-form.tsx
"use client";

import { useActionState } from "react";
import { createPost } from "@/app/actions/posts";

export function PostForm() {
  const [state, formAction, pending] = useActionState(createPost, { errors: undefined });

  return (
    <form action={formAction} className="mx-auto max-w-lg space-y-4 p-8">
      <div>
        <label htmlFor="title" className="block text-sm font-medium">
          Title
        </label>
        <input
          id="title"
          name="title"
          required
          className={`mt-1 w-full rounded border p-3 ${
            state.errors?.title ? "border-red-500" : ""
          }`}
        />
        {state.errors?.title && (
          <p className="mt-1 text-sm text-red-500">{state.errors.title[0]}</p>
        )}
      </div>

      <div>
        <label htmlFor="body" className="block text-sm font-medium">
          Body
        </label>
        <textarea
          id="body"
          name="body"
          rows={8}
          required
          className={`mt-1 w-full rounded border p-3 ${
            state.errors?.body ? "border-red-500" : ""
          }`}
        />
        {state.errors?.body && (
          <p className="mt-1 text-sm text-red-500">{state.errors.body[0]}</p>
        )}
      </div>

      <div className="flex items-center gap-2">
        <input id="published" name="published" type="checkbox" />
        <label htmlFor="published">Publish immediately</label>
      </div>

      {state.errors?._form && (
        <p className="text-sm text-red-500">{state.errors._form[0]}</p>
      )}

      <button
        type="submit"
        disabled={pending}
        className="w-full rounded bg-blue-500 py-3 text-white disabled:opacity-50"
      >
        {pending ? "Creating..." : "Create Post"}
      </button>
    </form>
  );
}
```

## API Route Validation

```ts
// app/api/contacts/route.ts
import { NextRequest, NextResponse } from "next/server";
import { z } from "zod";
import { createClient } from "@/lib/supabase/server";

const contactSchema = z.object({
  name: z.string().min(1, "Name is required").max(100),
  email: z.string().email("Invalid email"),
  message: z.string().min(10, "Message must be at least 10 characters").max(5000),
  phone: z
    .string()
    .regex(/^\+?[\d\s\-()]{7,15}$/, "Invalid phone number")
    .optional(),
});

export async function POST(request: NextRequest) {
  let rawBody: unknown;
  try {
    rawBody = await request.json();
  } catch {
    return NextResponse.json({ error: "Invalid JSON" }, { status: 400 });
  }

  const result = contactSchema.safeParse(rawBody);

  if (!result.success) {
    return NextResponse.json(
      {
        error: "Validation failed",
        details: result.error.flatten().fieldErrors,
      },
      { status: 400 }
    );
  }

  const supabase = await createClient();
  const { error } = await supabase.from("contacts").insert(result.data);

  if (error) {
    return NextResponse.json({ error: "Failed to save" }, { status: 500 });
  }

  return NextResponse.json({ success: true }, { status: 201 });
}
```

## Environment Variable Validation

Never let your app run with missing environment variables. Validate them at startup.

```ts
// lib/env.ts
import { z } from "zod";

const envSchema = z.object({
  // Public variables (available in browser)
  NEXT_PUBLIC_SUPABASE_URL: z.string().url("SUPABASE_URL must be a valid URL"),
  NEXT_PUBLIC_SUPABASE_ANON_KEY: z.string().min(1, "SUPABASE_ANON_KEY is required"),
  NEXT_PUBLIC_SITE_URL: z.string().url().optional().default("http://localhost:3000"),

  // Server-only variables
  SUPABASE_SERVICE_ROLE_KEY: z.string().min(1, "SERVICE_ROLE_KEY is required"),
  GEMINI_API_KEY: z.string().min(1, "GEMINI_API_KEY is required"),

  // Optional with defaults
  NODE_ENV: z
    .enum(["development", "production", "test"])
    .optional()
    .default("development"),
});

// Parse once at module load — throws if invalid
export const env = envSchema.parse(process.env);

// Usage anywhere in your server code:
// import { env } from "@/lib/env";
// const supabase = createClient(env.NEXT_PUBLIC_SUPABASE_URL, env.SUPABASE_SERVICE_ROLE_KEY);
```

## Common Zod Patterns

### Transform (Convert Data)

```ts
// Convert string "true"/"false" from form data to boolean
const formBooleanSchema = z
  .string()
  .optional()
  .transform((val) => val === "true" || val === "on");

// Convert string to number
const pageSchema = z
  .string()
  .optional()
  .default("1")
  .transform((val) => parseInt(val, 10))
  .pipe(z.number().int().min(1));

// Trim and lowercase email
const emailSchema = z
  .string()
  .email()
  .transform((val) => val.trim().toLowerCase());
```

### Refine (Custom Validation)

```ts
const passwordSchema = z
  .string()
  .min(8, "Password must be at least 8 characters")
  .refine((val) => /[A-Z]/.test(val), "Must contain an uppercase letter")
  .refine((val) => /[0-9]/.test(val), "Must contain a number");

// Cross-field validation
const signupSchema = z
  .object({
    password: z.string().min(8),
    confirmPassword: z.string(),
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: "Passwords do not match",
    path: ["confirmPassword"], // Shows error on confirmPassword field
  });
```

### Discriminated Unions

```ts
// Different shapes based on a "type" field
const notificationSchema = z.discriminatedUnion("type", [
  z.object({
    type: z.literal("email"),
    email: z.string().email(),
    subject: z.string(),
    body: z.string(),
  }),
  z.object({
    type: z.literal("sms"),
    phone: z.string(),
    message: z.string().max(160),
  }),
  z.object({
    type: z.literal("push"),
    title: z.string(),
    body: z.string(),
    token: z.string(),
  }),
]);

type Notification = z.infer<typeof notificationSchema>;
```

### Coerce (Auto-Convert Types)

```ts
// Automatically converts strings to the right type
const querySchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  active: z.coerce.boolean().default(true),
});

// "5" becomes 5, "true" becomes true
const parsed = querySchema.parse({ page: "5", limit: "20", active: "true" });
```

### Nullable vs Optional

```ts
const schema = z.object({
  // Optional: field can be missing entirely
  bio: z.string().optional(), // string | undefined

  // Nullable: field must exist but can be null
  avatar_url: z.string().url().nullable(), // string | null

  // Both: can be missing, undefined, or null
  nickname: z.string().optional().nullable(), // string | null | undefined
});
```

## Supabase Integration Patterns

### Validate Before Insert

```ts
// lib/schemas.ts
import { z } from "zod";

export const createCommentSchema = z.object({
  body: z.string().min(1, "Comment cannot be empty").max(2000, "Comment is too long"),
  post_id: z.string().uuid("Invalid post ID"),
});

export type CreateCommentInput = z.infer<typeof createCommentSchema>;
```

```ts
// app/actions/comments.ts
"use server";

import { createCommentSchema } from "@/lib/schemas";
import { createClient } from "@/lib/supabase/server";
import { revalidatePath } from "next/cache";

export async function createComment(formData: FormData) {
  const raw = {
    body: formData.get("body"),
    post_id: formData.get("post_id"),
  };

  const result = createCommentSchema.safeParse(raw);

  if (!result.success) {
    return { error: result.error.flatten().fieldErrors };
  }

  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();

  if (!user) {
    return { error: { _form: ["Must be logged in"] } };
  }

  const { error } = await supabase.from("comments").insert({
    body: result.data.body,
    post_id: result.data.post_id,
    user_id: user.id,
  });

  if (error) {
    return { error: { _form: [error.message] } };
  }

  revalidatePath(`/posts`);
  return { success: true };
}
```

### Validate Search/Filter Params

```ts
// lib/schemas.ts
export const searchParamsSchema = z.object({
  q: z.string().optional().default(""),
  page: z.coerce.number().int().min(1).optional().default(1),
  limit: z.coerce.number().int().min(1).max(100).optional().default(20),
  sort: z.enum(["newest", "oldest", "popular"]).optional().default("newest"),
  category: z.string().optional(),
});

export type SearchParams = z.infer<typeof searchParamsSchema>;
```

```tsx
// app/search/page.tsx
import { searchParamsSchema } from "@/lib/schemas";
import { createClient } from "@/lib/supabase/server";

export default async function SearchPage({
  searchParams,
}: {
  searchParams: Promise<Record<string, string | string[] | undefined>>;
}) {
  const raw = await searchParams;
  const params = searchParamsSchema.parse(raw);

  const supabase = await createClient();
  const offset = (params.page - 1) * params.limit;

  let query = supabase
    .from("posts")
    .select("*", { count: "exact" })
    .range(offset, offset + params.limit - 1);

  if (params.q) {
    query = query.textSearch("fts", params.q);
  }

  if (params.category) {
    query = query.eq("category", params.category);
  }

  const sortMap = {
    newest: { column: "created_at", ascending: false },
    oldest: { column: "created_at", ascending: true },
    popular: { column: "like_count", ascending: false },
  } as const;

  const sort = sortMap[params.sort];
  query = query.order(sort.column, { ascending: sort.ascending });

  const { data, count, error } = await query;

  if (error) throw new Error("Search failed");

  return (
    <div className="p-8">
      <p>{count} results for "{params.q}"</p>
      {data?.map((post) => (
        <div key={post.id} className="mt-4 rounded border p-4">
          {post.title}
        </div>
      ))}
    </div>
  );
}
```

### Validate Webhook Payloads

```ts
// app/api/webhooks/stripe/route.ts
import { NextRequest, NextResponse } from "next/server";
import { z } from "zod";

const checkoutSessionSchema = z.object({
  id: z.string(),
  customer: z.string(),
  subscription: z.string().nullable(),
  metadata: z.object({
    team_id: z.string().uuid(),
    plan_id: z.string().uuid(),
  }),
});

export async function POST(request: NextRequest) {
  const body = await request.json();

  // Validate the webhook event data
  if (body.type === "checkout.session.completed") {
    const result = checkoutSessionSchema.safeParse(body.data.object);

    if (!result.success) {
      console.error("Invalid webhook data:", result.error);
      return NextResponse.json({ error: "Invalid data" }, { status: 400 });
    }

    const session = result.data;
    // Now session is fully typed and validated
    console.log(`Team ${session.metadata.team_id} subscribed to plan ${session.metadata.plan_id}`);
  }

  return NextResponse.json({ received: true });
}
```

## Error Formatting

### Flatten Errors for Forms

```ts
const result = schema.safeParse(data);

if (!result.success) {
  const errors = result.error.flatten();

  // errors.fieldErrors = { title: ["Too short"], email: ["Invalid email"] }
  // errors.formErrors = ["Passwords don't match"] (from .refine())

  return {
    fieldErrors: errors.fieldErrors,
    formErrors: errors.formErrors,
  };
}
```

### Format Errors as a Simple Object

```ts
const result = schema.safeParse(data);

if (!result.success) {
  const errors: Record<string, string> = {};

  for (const issue of result.error.issues) {
    const path = issue.path.join(".");
    errors[path] = issue.message;
  }

  // errors = { title: "Too short", "address.zip": "Invalid zip" }
  return errors;
}
```
