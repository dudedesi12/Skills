---
name: supabase-fullstack
description: "Use this skill whenever the user mentions database, auth, users, login, signup, sign up, sign in, logout, RLS, row level security, Supabase, tables, migrations, policies, realtime, subscriptions, storage, buckets, edge functions, data modeling, schema, SQL, queries, foreign keys, indexes, OTP, magic link, OAuth, Google login, email verification, password reset, file upload, profile pictures, avatars, or ANY data persistence task — even if they don't explicitly say 'Supabase'. This skill handles ALL data layer operations."
---

# Supabase Fullstack

Supabase is your backend — it gives you a database (PostgreSQL), authentication, file storage, and real-time subscriptions. Think of it as "Firebase but with a real database."

## Supabase Client Setup

You need **three different clients** depending on where your code runs.

### 1. Server Component Client (for pages and layouts)

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
            // Server Components have read-only cookies — this is expected
          }
        },
      },
    }
  );
}
```

### 2. Browser Client (for Client Components)

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

### 3. Middleware Client (for auth session refresh)

```ts
// lib/supabase/middleware.ts
import { createServerClient } from "@supabase/ssr";
import { NextResponse, type NextRequest } from "next/server";

export async function updateSession(request: NextRequest) {
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
          cookiesToSet.forEach(({ name, value }) =>
            request.cookies.set(name, value)
          );
          response = NextResponse.next({ request });
          cookiesToSet.forEach(({ name, value, options }) =>
            response.cookies.set(name, value, options)
          );
        },
      },
    }
  );

  await supabase.auth.getUser();
  return response;
}
```

```ts
// middleware.ts
import { updateSession } from "@/lib/supabase/middleware";
import type { NextRequest } from "next/server";

export async function middleware(request: NextRequest) {
  return await updateSession(request);
}

export const config = {
  matcher: [
    "/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)",
  ],
};
```

## Authentication Flows

### Email/Password Sign Up

```tsx
// app/signup/page.tsx
import { createClient } from "@/lib/supabase/server";
import { redirect } from "next/navigation";

export default function SignupPage() {
  async function signup(formData: FormData) {
    "use server";

    const email = formData.get("email") as string;
    const password = formData.get("password") as string;

    if (!email || !password) {
      throw new Error("Email and password are required");
    }

    if (password.length < 8) {
      throw new Error("Password must be at least 8 characters");
    }

    const supabase = await createClient();
    const { error } = await supabase.auth.signUp({
      email,
      password,
    });

    if (error) {
      throw new Error(error.message);
    }

    redirect("/check-email");
  }

  return (
    <form action={signup} className="mx-auto max-w-sm space-y-4 p-8">
      <h1 className="text-2xl font-bold">Sign Up</h1>
      <input name="email" type="email" required placeholder="Email" className="w-full rounded border p-3" />
      <input name="password" type="password" required minLength={8} placeholder="Password (8+ characters)" className="w-full rounded border p-3" />
      <button type="submit" className="w-full rounded bg-blue-500 py-3 text-white">Create Account</button>
    </form>
  );
}
```

### Email/Password Login

```tsx
// app/login/page.tsx
import { createClient } from "@/lib/supabase/server";
import { redirect } from "next/navigation";

export default function LoginPage() {
  async function login(formData: FormData) {
    "use server";

    const email = formData.get("email") as string;
    const password = formData.get("password") as string;

    if (!email || !password) {
      throw new Error("Email and password are required");
    }

    const supabase = await createClient();
    const { error } = await supabase.auth.signInWithPassword({ email, password });

    if (error) {
      throw new Error("Invalid email or password");
    }

    redirect("/dashboard");
  }

  return (
    <form action={login} className="mx-auto max-w-sm space-y-4 p-8">
      <h1 className="text-2xl font-bold">Log In</h1>
      <input name="email" type="email" required placeholder="Email" className="w-full rounded border p-3" />
      <input name="password" type="password" required placeholder="Password" className="w-full rounded border p-3" />
      <button type="submit" className="w-full rounded bg-blue-500 py-3 text-white">Log In</button>
    </form>
  );
}
```

### OAuth (Google Login)

```tsx
// app/login/oauth-button.tsx
"use client";

import { createClient } from "@/lib/supabase/client";

export function GoogleLoginButton() {
  async function handleGoogleLogin() {
    const supabase = createClient();
    const { error } = await supabase.auth.signInWithOAuth({
      provider: "google",
      options: {
        redirectTo: `${window.location.origin}/auth/callback`,
      },
    });

    if (error) {
      alert("Failed to start Google login");
    }
  }

  return (
    <button onClick={handleGoogleLogin} className="w-full rounded border px-4 py-3 hover:bg-gray-50">
      Continue with Google
    </button>
  );
}
```

```ts
// app/auth/callback/route.ts
import { createClient } from "@/lib/supabase/server";
import { NextResponse } from "next/server";

export async function GET(request: Request) {
  const { searchParams, origin } = new URL(request.url);
  const code = searchParams.get("code");
  const next = searchParams.get("next") ?? "/dashboard";

  if (code) {
    const supabase = await createClient();
    const { error } = await supabase.auth.exchangeCodeForSession(code);

    if (!error) {
      return NextResponse.redirect(`${origin}${next}`);
    }
  }

  return NextResponse.redirect(`${origin}/login?error=auth_failed`);
}
```

### Magic Link (Passwordless)

```tsx
// app/login/magic-link-form.tsx
"use client";

import { useState } from "react";
import { createClient } from "@/lib/supabase/client";

export function MagicLinkForm() {
  const [email, setEmail] = useState("");
  const [sent, setSent] = useState(false);
  const [error, setError] = useState("");

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setError("");

    const supabase = createClient();
    const { error } = await supabase.auth.signInWithOtp({
      email,
      options: { emailRedirectTo: `${window.location.origin}/auth/callback` },
    });

    if (error) {
      setError(error.message);
    } else {
      setSent(true);
    }
  }

  if (sent) return <p className="text-green-600">Check your email for a login link.</p>;

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <input type="email" value={email} onChange={(e) => setEmail(e.target.value)} required placeholder="Email" className="w-full rounded border p-3" />
      {error && <p className="text-sm text-red-500">{error}</p>}
      <button type="submit" className="w-full rounded bg-blue-500 py-3 text-white">Send Magic Link</button>
    </form>
  );
}
```

### Logout

```ts
// app/actions/auth.ts
"use server";

import { createClient } from "@/lib/supabase/server";
import { redirect } from "next/navigation";

export async function logout() {
  const supabase = await createClient();
  await supabase.auth.signOut();
  redirect("/login");
}
```

### Get Current User

```tsx
// In Server Components
import { createClient } from "@/lib/supabase/server";

export default async function ProfilePage() {
  const supabase = await createClient();
  const { data: { user }, error } = await supabase.auth.getUser();

  if (!user) {
    redirect("/login");
  }

  return <p>Logged in as {user.email}</p>;
}
```

## Row Level Security (RLS)

RLS is how you protect your data. Every table should have RLS enabled. Without policies, NO ONE can access the data (not even authenticated users).

### Enable RLS and Create Policies

```sql
-- Always enable RLS on every table
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;

-- Users can read all published posts
CREATE POLICY "Anyone can read published posts"
ON posts FOR SELECT
USING (published = true);

-- Users can only insert their own posts
CREATE POLICY "Users can create own posts"
ON posts FOR INSERT
WITH CHECK (auth.uid() = author_id);

-- Users can only update their own posts
CREATE POLICY "Users can update own posts"
ON posts FOR UPDATE
USING (auth.uid() = author_id)
WITH CHECK (auth.uid() = author_id);

-- Users can only delete their own posts
CREATE POLICY "Users can delete own posts"
ON posts FOR DELETE
USING (auth.uid() = author_id);
```

## Database Schema Design

### Users and Profiles

```sql
-- Profiles table (extends auth.users with app-specific data)
CREATE TABLE profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  full_name TEXT NOT NULL DEFAULT '',
  username TEXT UNIQUE,
  avatar_url TEXT,
  bio TEXT DEFAULT '',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Anyone can view profiles"
ON profiles FOR SELECT USING (true);

CREATE POLICY "Users can update own profile"
ON profiles FOR UPDATE
USING (auth.uid() = id)
WITH CHECK (auth.uid() = id);

-- Auto-create profile when a new user signs up
CREATE OR REPLACE FUNCTION handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO profiles (id, full_name, avatar_url)
  VALUES (
    NEW.id,
    COALESCE(NEW.raw_user_meta_data->>'full_name', ''),
    COALESCE(NEW.raw_user_meta_data->>'avatar_url', '')
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
AFTER INSERT ON auth.users
FOR EACH ROW EXECUTE FUNCTION handle_new_user();
```

### Posts with Full-Text Search

```sql
CREATE TABLE posts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  body TEXT DEFAULT '',
  published BOOLEAN DEFAULT false,
  author_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),

  -- Full-text search column
  fts TSVECTOR GENERATED ALWAYS AS (
    setweight(to_tsvector('english', COALESCE(title, '')), 'A') ||
    setweight(to_tsvector('english', COALESCE(body, '')), 'B')
  ) STORED
);

CREATE INDEX posts_fts_idx ON posts USING GIN (fts);
CREATE INDEX posts_author_id_idx ON posts(author_id);
CREATE INDEX posts_slug_idx ON posts(slug);
CREATE INDEX posts_created_at_idx ON posts(created_at DESC);
```

Search query:

```tsx
// app/search/page.tsx
const { data, error } = await supabase
  .from("posts")
  .select("id, title, slug")
  .textSearch("fts", query, { type: "websearch" })
  .eq("published", true)
  .limit(20);
```

## Supabase Realtime

```tsx
// app/components/live-notifications.tsx
"use client";

import { useEffect, useState } from "react";
import { createClient } from "@/lib/supabase/client";

interface Notification {
  id: string;
  message: string;
  created_at: string;
}

export function LiveNotifications({ userId }: { userId: string }) {
  const [notifications, setNotifications] = useState<Notification[]>([]);

  useEffect(() => {
    const supabase = createClient();

    const channel = supabase
      .channel("notifications")
      .on(
        "postgres_changes",
        {
          event: "INSERT",
          schema: "public",
          table: "notifications",
          filter: `user_id=eq.${userId}`,
        },
        (payload) => {
          setNotifications((prev) => [payload.new as Notification, ...prev]);
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, [userId]);

  return (
    <div className="space-y-2">
      {notifications.map((n) => (
        <div key={n.id} className="rounded bg-blue-50 p-3 text-sm">
          {n.message}
        </div>
      ))}
    </div>
  );
}
```

## Storage (File Uploads)

### Create Bucket and Policies

```sql
-- Create a storage bucket
INSERT INTO storage.buckets (id, name, public)
VALUES ('avatars', 'avatars', true);

-- Allow authenticated users to upload their own avatar
CREATE POLICY "Users can upload own avatar"
ON storage.objects FOR INSERT
WITH CHECK (
  bucket_id = 'avatars' AND
  auth.uid()::text = (storage.foldername(name))[1]
);

-- Allow anyone to view avatars
CREATE POLICY "Anyone can view avatars"
ON storage.objects FOR SELECT
USING (bucket_id = 'avatars');

-- Allow users to update/delete their own avatar
CREATE POLICY "Users can update own avatar"
ON storage.objects FOR UPDATE
USING (bucket_id = 'avatars' AND auth.uid()::text = (storage.foldername(name))[1]);

CREATE POLICY "Users can delete own avatar"
ON storage.objects FOR DELETE
USING (bucket_id = 'avatars' AND auth.uid()::text = (storage.foldername(name))[1]);
```

### Upload File from Client

```tsx
// app/components/avatar-upload.tsx
"use client";

import { useState } from "react";
import { createClient } from "@/lib/supabase/client";

export function AvatarUpload({ userId }: { userId: string }) {
  const [uploading, setUploading] = useState(false);

  async function handleUpload(e: React.ChangeEvent<HTMLInputElement>) {
    const file = e.target.files?.[0];
    if (!file) return;

    if (file.size > 2 * 1024 * 1024) {
      alert("File must be under 2MB");
      return;
    }

    if (!file.type.startsWith("image/")) {
      alert("File must be an image");
      return;
    }

    setUploading(true);
    const supabase = createClient();
    const filePath = `${userId}/avatar.${file.name.split(".").pop()}`;

    const { error } = await supabase.storage
      .from("avatars")
      .upload(filePath, file, { upsert: true });

    if (error) {
      alert("Upload failed: " + error.message);
    } else {
      const { data } = supabase.storage.from("avatars").getPublicUrl(filePath);
      // Update profile with new avatar URL
      await supabase.from("profiles").update({ avatar_url: data.publicUrl }).eq("id", userId);
      window.location.reload();
    }

    setUploading(false);
  }

  return (
    <label className="cursor-pointer rounded bg-gray-100 px-4 py-2 hover:bg-gray-200">
      {uploading ? "Uploading..." : "Change Avatar"}
      <input type="file" accept="image/*" onChange={handleUpload} className="hidden" disabled={uploading} />
    </label>
  );
}
```

## Edge Functions (Deno)

Edge Functions run server-side code close to your users. Use for webhooks, scheduled tasks, or custom logic.

```ts
// supabase/functions/send-welcome-email/index.ts
import { serve } from "https://deno.land/std@0.177.0/http/server.ts";
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

serve(async (req) => {
  try {
    const { user_id } = await req.json();

    if (!user_id) {
      return new Response(JSON.stringify({ error: "user_id required" }), {
        status: 400,
        headers: { "Content-Type": "application/json" },
      });
    }

    const supabase = createClient(
      Deno.env.get("SUPABASE_URL")!,
      Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
    );

    const { data: profile } = await supabase
      .from("profiles")
      .select("full_name")
      .eq("id", user_id)
      .single();

    // Send email via your email provider here

    return new Response(JSON.stringify({ success: true }), {
      headers: { "Content-Type": "application/json" },
    });
  } catch (err) {
    return new Response(JSON.stringify({ error: "Internal error" }), {
      status: 500,
      headers: { "Content-Type": "application/json" },
    });
  }
});
```

Invoke from Next.js:

```ts
const { data, error } = await supabase.functions.invoke("send-welcome-email", {
  body: { user_id: user.id },
});
```

## Migrations

```bash
# Create a new migration
npx supabase migration new create_posts_table

# This creates: supabase/migrations/20240101000000_create_posts_table.sql
# Write your SQL in that file, then apply:

npx supabase db push          # Push to remote
npx supabase db reset         # Reset local DB and re-run all migrations
```

## Supabase CLI Commands

```bash
npx supabase init                          # Initialize Supabase in project
npx supabase start                         # Start local Supabase
npx supabase stop                          # Stop local Supabase
npx supabase db push                       # Push migrations to remote
npx supabase db pull                       # Pull remote schema as migration
npx supabase db reset                      # Reset local DB
npx supabase gen types typescript --local  # Generate TypeScript types from local DB
npx supabase gen types typescript --project-id YOUR_ID > lib/database.types.ts
npx supabase functions serve               # Run edge functions locally
npx supabase functions deploy my-function  # Deploy edge function
npx supabase migration new my_migration    # Create new migration file
```

## Database Functions and Triggers

```sql
-- Auto-update updated_at column
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_updated_at
BEFORE UPDATE ON posts
FOR EACH ROW EXECUTE FUNCTION update_updated_at();

-- Increment a counter safely
CREATE OR REPLACE FUNCTION increment_likes(post_id UUID)
RETURNS VOID AS $$
BEGIN
  UPDATE posts SET like_count = like_count + 1 WHERE id = post_id;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

## Environment Variables

```bash
# .env.local
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...your-anon-key
SUPABASE_SERVICE_ROLE_KEY=eyJ...your-service-role-key  # NEVER expose to browser
```

**Important:** The anon key is safe to expose (it is rate-limited and RLS protects data). The service role key bypasses RLS — NEVER use it in client-side code.
