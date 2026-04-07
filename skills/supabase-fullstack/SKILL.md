---
name: supabase-fullstack
description: "Use this skill whenever the user mentions database, auth, users, login, signup, sign up, sign in, logout, RLS, row level security, Supabase, tables, migrations, policies, realtime, subscriptions, storage, buckets, edge functions, data modeling, schema, SQL, queries, foreign keys, indexes, OTP, magic link, OAuth, Google login, email verification, password reset, file upload, profile pictures, avatars, or ANY data persistence task — even if they don't explicitly say 'Supabase'. This skill handles ALL data layer operations."
---

# Supabase Fullstack

Supabase is your backend — database (PostgreSQL), authentication, file storage, and real-time subscriptions.

## Supabase Client Setup

You need three clients for different contexts.

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
        getAll() { return cookieStore.getAll(); },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) => cookieStore.set(name, value, options));
          } catch { /* Read-only in Server Components */ }
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

### Middleware Client

See `references/auth-patterns.md` for the full middleware setup with session refresh.

## Auth Flows

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
    if (!email || !password) throw new Error("Email and password required");
    if (password.length < 8) throw new Error("Password must be 8+ characters");

    const supabase = await createClient();
    const { error } = await supabase.auth.signUp({ email, password });
    if (error) throw new Error(error.message);
    redirect("/check-email");
  }

  return (
    <form action={signup} className="mx-auto max-w-sm space-y-4 p-8">
      <h1 className="text-2xl font-bold">Sign Up</h1>
      <input name="email" type="email" required placeholder="Email" className="w-full rounded border p-3" />
      <input name="password" type="password" required minLength={8} placeholder="Password" className="w-full rounded border p-3" />
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
    if (!email || !password) throw new Error("Email and password required");

    const supabase = await createClient();
    const { error } = await supabase.auth.signInWithPassword({ email, password });
    if (error) throw new Error("Invalid email or password");
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

### OAuth (Google)

```tsx
// app/login/google-button.tsx
"use client";
import { createClient } from "@/lib/supabase/client";

export function GoogleLoginButton() {
  async function handleClick() {
    const supabase = createClient();
    const { error } = await supabase.auth.signInWithOAuth({
      provider: "google",
      options: { redirectTo: `${window.location.origin}/auth/callback` },
    });
    if (error) alert("Failed to start Google login");
  }

  return <button onClick={handleClick} className="w-full rounded border px-4 py-3">Continue with Google</button>;
}
```

### Auth Callback Route

```ts
// app/auth/callback/route.ts
import { createClient } from "@/lib/supabase/server";
import { NextResponse } from "next/server";

export async function GET(request: Request) {
  const { searchParams, origin } = new URL(request.url);
  const code = searchParams.get("code");
  if (code) {
    const supabase = await createClient();
    const { error } = await supabase.auth.exchangeCodeForSession(code);
    if (!error) return NextResponse.redirect(`${origin}/dashboard`);
  }
  return NextResponse.redirect(`${origin}/login?error=auth_failed`);
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

See `references/auth-patterns.md` for magic link, password reset, and protected route patterns.

## Row Level Security (RLS)

Every table must have RLS enabled. Without policies, no one can access the data.

```sql
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Anyone can read published posts"
ON posts FOR SELECT USING (published = true);

CREATE POLICY "Users can create own posts"
ON posts FOR INSERT WITH CHECK (auth.uid() = author_id);

CREATE POLICY "Users can update own posts"
ON posts FOR UPDATE
USING (auth.uid() = author_id)
WITH CHECK (auth.uid() = author_id);

CREATE POLICY "Users can delete own posts"
ON posts FOR DELETE USING (auth.uid() = author_id);
```

See `references/rls-policies.md` for 15+ patterns (team-based, role-based, admin override, time-based, etc).

## Database Schema Design

### Users and Profiles

```sql
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
CREATE POLICY "Anyone can view profiles" ON profiles FOR SELECT USING (true);
CREATE POLICY "Users can update own profile" ON profiles FOR UPDATE USING (auth.uid() = id) WITH CHECK (auth.uid() = id);

-- Auto-create profile on signup
CREATE OR REPLACE FUNCTION handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO profiles (id, full_name, avatar_url)
  VALUES (NEW.id, COALESCE(NEW.raw_user_meta_data->>'full_name', ''), COALESCE(NEW.raw_user_meta_data->>'avatar_url', ''));
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
AFTER INSERT ON auth.users FOR EACH ROW EXECUTE FUNCTION handle_new_user();
```

See `references/schema-templates.md` for teams, subscriptions, comments, notifications, and more.

## Full-Text Search

```sql
CREATE TABLE posts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  body TEXT DEFAULT '',
  published BOOLEAN DEFAULT false,
  author_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  fts TSVECTOR GENERATED ALWAYS AS (
    setweight(to_tsvector('english', COALESCE(title, '')), 'A') ||
    setweight(to_tsvector('english', COALESCE(body, '')), 'B')
  ) STORED
);

CREATE INDEX posts_fts_idx ON posts USING GIN (fts);
```

Query: `await supabase.from("posts").select("*").textSearch("fts", query, { type: "websearch" })`

## Realtime Subscriptions

```tsx
// app/components/live-notifications.tsx
"use client";
import { useEffect, useState } from "react";
import { createClient } from "@/lib/supabase/client";

export function LiveNotifications({ userId }: { userId: string }) {
  const [items, setItems] = useState<{ id: string; message: string }[]>([]);

  useEffect(() => {
    const supabase = createClient();
    const channel = supabase
      .channel("notifications")
      .on("postgres_changes", {
        event: "INSERT", schema: "public", table: "notifications",
        filter: `user_id=eq.${userId}`,
      }, (payload) => {
        setItems((prev) => [payload.new as any, ...prev]);
      })
      .subscribe();

    return () => { supabase.removeChannel(channel); };
  }, [userId]);

  return (
    <div className="space-y-2">
      {items.map((n) => <div key={n.id} className="rounded bg-blue-50 p-3 text-sm">{n.message}</div>)}
    </div>
  );
}
```

See `references/realtime.md` for broadcast, presence, and chat examples.

## Storage (File Uploads)

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
    if (file.size > 2 * 1024 * 1024) { alert("File must be under 2MB"); return; }
    if (!file.type.startsWith("image/")) { alert("Must be an image"); return; }

    setUploading(true);
    const supabase = createClient();
    const filePath = `${userId}/avatar.${file.name.split(".").pop()}`;
    const { error } = await supabase.storage.from("avatars").upload(filePath, file, { upsert: true });

    if (error) { alert("Upload failed"); }
    else {
      const { data } = supabase.storage.from("avatars").getPublicUrl(filePath);
      await supabase.from("profiles").update({ avatar_url: data.publicUrl }).eq("id", userId);
    }
    setUploading(false);
  }

  return (
    <label className="cursor-pointer rounded bg-gray-100 px-4 py-2">
      {uploading ? "Uploading..." : "Change Avatar"}
      <input type="file" accept="image/*" onChange={handleUpload} className="hidden" disabled={uploading} />
    </label>
  );
}
```

## Database Functions and Triggers

```sql
-- Auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN NEW.updated_at = now(); RETURN NEW; END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_updated_at BEFORE UPDATE ON posts
FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

## Edge Functions

See `references/edge-functions.md` for Deno edge function patterns and invocation from Next.js.

## Migrations and CLI

```bash
npx supabase init                          # Initialize
npx supabase start                         # Start local
npx supabase migration new my_migration    # Create migration
npx supabase db push                       # Push to remote
npx supabase db pull                       # Pull remote schema
npx supabase gen types typescript --local > lib/database.types.ts
npx supabase functions deploy my-function  # Deploy edge function
```

## Environment Variables

```bash
# .env.local
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...       # Safe for browser (RLS protects data)
SUPABASE_SERVICE_ROLE_KEY=eyJ...           # Bypasses RLS — NEVER expose to browser
```
