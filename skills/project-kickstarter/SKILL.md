---
name: project-kickstarter
description: "Use this skill whenever the user says 'new project', 'start a project', 'create an app', 'build something new', 'kickstart', 'bootstrap', 'scaffold', 'init', 'setup', 'start fresh', 'new idea' (when it's a whole new product, not a feature), or mentions starting any new web application from scratch. Even if they just say 'I want to build X' where X is a new product — trigger this skill to set up the foundation before writing any feature code. Also trigger when the user says 'spin up', 'set up a repo', 'blank slate', 'from zero', 'greenfield', 'v2', 'rewrite', 'start over', or describes ANY new product concept without an existing codebase."
---

# Project Kickstarter

You take a user from zero to a deployed, working web app in a single session. No decisions required from them — you pick the best defaults, set up everything, and hand them a live URL with auth working, database connected, and CI/CD configured.

## Step 1: Pick the Template

Ask the user one question: **"What are you building?"**

Based on their answer, pick the template:

| Template | When to Use | What They Get |
|----------|------------|---------------|
| `saas` | Anything with users, accounts, payments | Auth + dashboard + Stripe + landing page |
| `content` | Blog, docs, SEO-driven site, content hub | MDX blog + SEO + sitemap + programmatic pages |
| `tool` | Calculator, analyzer, converter, tracker | Single-purpose UI + optional auth + share results |
| `api` | Backend service, webhook processor, data pipeline | Route handlers + cron + webhook receivers + queue |

If unclear, **default to `saas`** — it covers the most ground and pieces can be removed later.

## Step 2: Create the Project

Run the full setup. Do not pause between steps. Do not ask for confirmation.

```bash
npx create-next-app@latest PROJECT_NAME \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --src-dir=false \
  --import-alias="@/*" \
  --use-npm
```

Then immediately:

```bash
cd PROJECT_NAME
git init
```

## Step 3: Install Dependencies

All templates share a base set of dependencies:

```bash
npm install @supabase/supabase-js @supabase/ssr
npm install -D prettier prettier-plugin-tailwindcss @types/node
```

Template-specific additions:

**saas:**
```bash
npm install stripe @stripe/stripe-js
npm install @google/generative-ai
npm install lucide-react class-variance-authority clsx tailwind-merge
npx shadcn@latest init -d
npx shadcn@latest add button card input label separator toast dialog dropdown-menu avatar badge
```

**content:**
```bash
npm install @next/mdx @mdx-js/loader @mdx-js/react gray-matter reading-time
npm install @google/generative-ai
npm install lucide-react class-variance-authority clsx tailwind-merge
npx shadcn@latest init -d
npx shadcn@latest add button card separator badge
```

**tool:**
```bash
npm install @google/generative-ai
npm install lucide-react class-variance-authority clsx tailwind-merge
npx shadcn@latest init -d
npx shadcn@latest add button card input label toast progress
```

**api:**
```bash
npm install @google/generative-ai
npm install lucide-react class-variance-authority clsx tailwind-merge
npx shadcn@latest init -d
npx shadcn@latest add button card badge
```

## Step 4: Create the Folder Structure

See `references/folder-structure.md` for detailed explanations of every file and folder.

Create the full structure based on the chosen template. For `saas` (the default), create:

```
app/
├── (auth)/
│   ├── login/page.tsx
│   └── signup/page.tsx
├── (dashboard)/
│   ├── layout.tsx
│   └── page.tsx
├── (marketing)/
│   └── page.tsx
├── api/
│   ├── health/route.ts
│   ├── webhooks/stripe/route.ts
│   └── cron/route.ts
├── layout.tsx
├── globals.css
├── not-found.tsx
components/
├── ui/                        # managed by shadcn
├── shared/
│   ├── navbar.tsx
│   ├── footer.tsx
│   └── user-menu.tsx
lib/
├── supabase/
│   ├── client.ts
│   ├── server.ts
│   └── middleware.ts
├── gemini/
│   └── client.ts
├── stripe/
│   └── client.ts
└── utils.ts
middleware.ts
```

## Step 5: Configure Supabase

### 5a: Initialize Supabase locally

```bash
npx supabase init
npx supabase start
```

### 5b: Create the Supabase client files

**lib/supabase/client.ts** — Browser client (used in Client Components):

```typescript
"use client";

import { createBrowserClient } from "@supabase/ssr";

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

**lib/supabase/server.ts** — Server client (used in Server Components, Route Handlers, Server Actions):

```typescript
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
            // The `setAll` method is called from a Server Component.
            // This can be ignored if you have middleware refreshing sessions.
          }
        },
      },
    }
  );
}
```

**lib/supabase/middleware.ts** — Middleware helper for session refresh:

```typescript
import { createServerClient } from "@supabase/ssr";
import { NextResponse, type NextRequest } from "next/server";

export async function updateSession(request: NextRequest) {
  let supabaseResponse = NextResponse.next({ request });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll();
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value, options }) =>
            request.cookies.set(name, value)
          );
          supabaseResponse = NextResponse.next({ request });
          cookiesToSet.forEach(({ name, value, options }) =>
            supabaseResponse.cookies.set(name, value, options)
          );
        },
      },
    }
  );

  const {
    data: { user },
  } = await supabase.auth.getUser();

  // Redirect unauthenticated users away from protected routes
  if (
    !user &&
    request.nextUrl.pathname.startsWith("/dashboard")
  ) {
    const url = request.nextUrl.clone();
    url.pathname = "/login";
    return NextResponse.redirect(url);
  }

  return supabaseResponse;
}
```

### 5c: Create the root middleware

**middleware.ts** (project root):

```typescript
import { type NextRequest } from "next/server";
import { updateSession } from "@/lib/supabase/middleware";

export async function middleware(request: NextRequest) {
  return await updateSession(request);
}

export const config = {
  matcher: [
    "/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)",
  ],
};
```

### 5d: Create the users table with RLS

Create `supabase/migrations/00001_create_users.sql`:

```sql
-- Create a public profiles table that mirrors auth.users
create table public.profiles (
  id uuid references auth.users on delete cascade not null primary key,
  email text,
  full_name text,
  avatar_url text,
  created_at timestamptz default now() not null,
  updated_at timestamptz default now() not null
);

-- Enable RLS
alter table public.profiles enable row level security;

-- Users can read their own profile
create policy "Users can view own profile"
  on public.profiles for select
  using (auth.uid() = id);

-- Users can update their own profile
create policy "Users can update own profile"
  on public.profiles for update
  using (auth.uid() = id);

-- Auto-create profile on signup
create or replace function public.handle_new_user()
returns trigger
language plpgsql
security definer set search_path = ''
as $$
begin
  insert into public.profiles (id, email, full_name, avatar_url)
  values (
    new.id,
    new.email,
    new.raw_user_meta_data ->> 'full_name',
    new.raw_user_meta_data ->> 'avatar_url'
  );
  return new;
end;
$$;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute procedure public.handle_new_user();

-- Auto-update updated_at
create or replace function public.handle_updated_at()
returns trigger
language plpgsql
as $$
begin
  new.updated_at = now();
  return new;
end;
$$;

create trigger on_profile_updated
  before update on public.profiles
  for each row execute procedure public.handle_updated_at();
```

### 5e: Configure Auth Providers

In `supabase/config.toml`, ensure:

```toml
[auth]
enabled = true
site_url = "http://localhost:3000"
additional_redirect_urls = ["http://localhost:3000/auth/callback"]

[auth.email]
enable_signup = true
enable_confirmations = false

[auth.external.google]
enabled = true
client_id = "env(GOOGLE_CLIENT_ID)"
secret = "env(GOOGLE_CLIENT_SECRET)"
redirect_uri = "http://localhost:54321/auth/v1/callback"
```

## Step 6: Create Auth Pages

**app/(auth)/login/page.tsx:**

```tsx
"use client";

import { createClient } from "@/lib/supabase/client";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from "@/components/ui/card";
import { Separator } from "@/components/ui/separator";
import Link from "next/link";
import { useRouter } from "next/navigation";
import { useState } from "react";

export default function LoginPage() {
  const router = useRouter();
  const supabase = createClient();
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(false);

  async function handleEmailLogin(e: React.FormEvent) {
    e.preventDefault();
    setLoading(true);
    setError(null);
    const { error } = await supabase.auth.signInWithPassword({ email, password });
    if (error) {
      setError(error.message);
      setLoading(false);
    } else {
      router.push("/dashboard");
      router.refresh();
    }
  }

  async function handleGoogleLogin() {
    await supabase.auth.signInWithOAuth({
      provider: "google",
      options: { redirectTo: `${window.location.origin}/auth/callback` },
    });
  }

  return (
    <div className="flex min-h-screen items-center justify-center p-4">
      <Card className="w-full max-w-md">
        <CardHeader className="text-center">
          <CardTitle className="text-2xl">Welcome back</CardTitle>
          <CardDescription>Sign in to your account</CardDescription>
        </CardHeader>
        <CardContent className="space-y-4">
          <Button variant="outline" className="w-full" onClick={handleGoogleLogin}>
            Continue with Google
          </Button>
          <Separator />
          <form onSubmit={handleEmailLogin} className="space-y-4">
            <div className="space-y-2">
              <Label htmlFor="email">Email</Label>
              <Input id="email" type="email" value={email} onChange={(e) => setEmail(e.target.value)} required />
            </div>
            <div className="space-y-2">
              <Label htmlFor="password">Password</Label>
              <Input id="password" type="password" value={password} onChange={(e) => setPassword(e.target.value)} required />
            </div>
            {error && <p className="text-sm text-red-500">{error}</p>}
            <Button type="submit" className="w-full" disabled={loading}>
              {loading ? "Signing in..." : "Sign in"}
            </Button>
          </form>
          <p className="text-center text-sm text-muted-foreground">
            No account? <Link href="/signup" className="underline">Sign up</Link>
          </p>
        </CardContent>
      </Card>
    </div>
  );
}
```

**app/(auth)/signup/page.tsx** follows the same pattern but calls `supabase.auth.signUp` instead.

**app/auth/callback/route.ts** — OAuth callback handler:

```typescript
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

## Step 7: Configure Gemini API Client

**lib/gemini/client.ts:**

```typescript
import { GoogleGenerativeAI } from "@google/generative-ai";

if (!process.env.GEMINI_API_KEY) {
  throw new Error("GEMINI_API_KEY is not set");
}

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY);

// Model tiering — pick the right model for the job
export const gemini = {
  // Fast and cheap — use for classification, extraction, simple Q&A
  flash: genAI.getGenerativeModel({ model: "gemini-2.0-flash" }),

  // Balanced — use for summarization, content generation, analysis
  pro: genAI.getGenerativeModel({ model: "gemini-2.5-pro-preview-06-05" }),

  // Thinking model — use for complex reasoning, math, code generation
  thinking: genAI.getGenerativeModel({ model: "gemini-2.5-flash-preview-05-20" }),
};

// Helper for simple text generation
export async function generateText(
  prompt: string,
  tier: "flash" | "pro" | "thinking" = "flash"
) {
  const model = gemini[tier];
  const result = await model.generateContent(prompt);
  return result.response.text();
}

// Helper for structured JSON output
export async function generateJSON<T>(
  prompt: string,
  tier: "flash" | "pro" | "thinking" = "flash"
): Promise<T> {
  const model = gemini[tier];
  const result = await model.generateContent(
    `${prompt}\n\nRespond ONLY with valid JSON. No markdown, no explanation.`
  );
  const text = result.response.text();
  return JSON.parse(text) as T;
}
```

## Step 8: Configure Stripe Client (saas template only)

**lib/stripe/client.ts:**

```typescript
import Stripe from "stripe";

if (!process.env.STRIPE_SECRET_KEY) {
  throw new Error("STRIPE_SECRET_KEY is not set");
}

export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY, {
  apiVersion: "2024-12-18.acacia",
  typescript: true,
});

// Common helpers
export async function createCheckoutSession(params: {
  priceId: string;
  userId: string;
  successUrl: string;
  cancelUrl: string;
}) {
  return stripe.checkout.sessions.create({
    mode: "subscription",
    payment_method_types: ["card"],
    line_items: [{ price: params.priceId, quantity: 1 }],
    success_url: params.successUrl,
    cancel_url: params.cancelUrl,
    client_reference_id: params.userId,
    metadata: { userId: params.userId },
  });
}

export async function createCustomerPortalSession(customerId: string, returnUrl: string) {
  return stripe.billingPortal.sessions.create({
    customer: customerId,
    return_url: returnUrl,
  });
}
```

**app/api/webhooks/stripe/route.ts:**

```typescript
import { stripe } from "@/lib/stripe/client";
import { createClient } from "@/lib/supabase/server";
import { headers } from "next/headers";
import { NextResponse } from "next/server";

export async function POST(request: Request) {
  const body = await request.text();
  const headersList = await headers();
  const signature = headersList.get("stripe-signature")!;

  let event;
  try {
    event = stripe.webhooks.constructEvent(
      body,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET!
    );
  } catch (err) {
    console.error("Webhook signature verification failed:", err);
    return NextResponse.json({ error: "Invalid signature" }, { status: 400 });
  }

  const supabase = await createClient();

  switch (event.type) {
    case "checkout.session.completed": {
      const session = event.data.object;
      const userId = session.client_reference_id;
      if (userId) {
        await supabase
          .from("profiles")
          .update({ stripe_customer_id: session.customer as string })
          .eq("id", userId);
      }
      break;
    }
    case "customer.subscription.updated":
    case "customer.subscription.deleted": {
      // Handle subscription status changes
      const subscription = event.data.object;
      console.log("Subscription update:", subscription.id, subscription.status);
      break;
    }
  }

  return NextResponse.json({ received: true });
}
```

## Step 9: Create API Routes

**app/api/health/route.ts:**

```typescript
import { NextResponse } from "next/server";

export async function GET() {
  return NextResponse.json({
    status: "ok",
    timestamp: new Date().toISOString(),
    environment: process.env.NODE_ENV,
  });
}
```

**app/api/cron/route.ts:**

```typescript
import { NextResponse } from "next/server";

export async function GET(request: Request) {
  const authHeader = request.headers.get("authorization");
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  // Add your scheduled tasks here
  console.log("Cron job executed at:", new Date().toISOString());

  return NextResponse.json({ success: true });
}
```

## Step 10: Create Environment Variables Template

Create `.env.local.example` — see `references/env-vars.md` for full details:

```bash
# Supabase
NEXT_PUBLIC_SUPABASE_URL=http://localhost:54321
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key

# Gemini AI
GEMINI_API_KEY=your-gemini-api-key

# Stripe (saas template only)
STRIPE_SECRET_KEY=sk_test_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Google OAuth
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret

# Cron
CRON_SECRET=your-cron-secret

# App
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

Then copy it:

```bash
cp .env.local.example .env.local
```

## Step 11: Configure ESLint + Prettier

**.prettierrc:**

```json
{
  "semi": true,
  "singleQuote": false,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 100,
  "plugins": ["prettier-plugin-tailwindcss"]
}
```

Update `package.json` scripts:

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  }
}
```

## Step 12: Set Up .gitignore

Ensure `.gitignore` includes:

```
node_modules/
.next/
.env*.local
.vercel
*.tsbuildinfo
next-env.d.ts
.supabase/
```

## Step 13: Create Utility Files

**lib/utils.ts:**

```typescript
import { type ClassValue, clsx } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}

export function formatDate(date: string | Date): string {
  return new Intl.DateTimeFormat("en-US", {
    month: "short",
    day: "numeric",
    year: "numeric",
  }).format(new Date(date));
}

export function absoluteUrl(path: string): string {
  return `${process.env.NEXT_PUBLIC_APP_URL}${path}`;
}
```

## Step 14: Create IDEAS.md

```markdown
# Ideas Parking Lot

| # | Idea | Impact | Confidence | Ease | ICE Score | Status |
|---|------|--------|------------|------|-----------|--------|
| 1 | (Your first idea goes here) | - | - | - | - | Parked |
```

This file is managed by the `idea-parking-lot` skill.

## Step 15: Create README

Generate a README.md with:
- Project name and one-line description
- Tech stack badges (Next.js, TypeScript, Tailwind, Supabase, Vercel)
- Getting started steps (clone, install, env vars, supabase start, dev)
- Folder structure overview
- Deployment link

## Step 16: Link Vercel and Deploy

See `references/first-deploy.md` for the complete walkthrough.

```bash
npx vercel link
npx vercel env pull .env.local
```

Then push to GitHub and let Vercel auto-deploy:

```bash
git add -A
git commit -m "Initial project setup"
git branch -M main
gh repo create PROJECT_NAME --private --source=. --push
```

Vercel will detect the push and deploy automatically if the project is linked.

## Step 17: Verify Everything Works

Run through this checklist:
1. `npm run dev` starts without errors
2. Landing page loads at `localhost:3000`
3. `/login` and `/signup` pages render
4. `/dashboard` redirects to `/login` when not authenticated
5. `/api/health` returns `{ status: "ok" }`
6. `npm run build` completes without errors
7. `npm run lint` passes
8. Supabase Studio is accessible at `localhost:54323`

Tell the user: **"Your project is live. Here's what you have..."** and list everything that was set up with links to each piece.

## Template Variations

### content template
- Replace `(dashboard)` route group with `(blog)` containing `page.tsx` (blog index) and `[slug]/page.tsx` (individual posts)
- Add `content/` directory at project root for MDX files
- Add `lib/mdx.ts` with frontmatter parsing using gray-matter
- Add `app/sitemap.ts` and `app/robots.ts`
- Skip Stripe setup
- Add RSS feed route at `app/feed.xml/route.ts`

### tool template
- Remove `(dashboard)` route group, replace with single `app/(tool)/page.tsx`
- Add `app/(tool)/results/page.tsx` for shareable results
- Skip Stripe unless user mentions monetization
- Focus shadcn components on inputs and feedback (progress, toast)
- Add `lib/calculations.ts` or `lib/analysis.ts` depending on tool type

### api template
- Remove all `(auth)`, `(dashboard)`, `(marketing)` route groups
- Keep only `app/api/` routes and a minimal `app/page.tsx` status page
- Add `app/api/v1/` versioned route structure
- Add `lib/queue.ts` for background job patterns
- Add `lib/validators.ts` with request validation helpers
- Focus on webhook receivers and cron jobs
