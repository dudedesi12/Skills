---
name: project-kickstarter
description: "Use this skill whenever the user says 'new project', 'start a project', 'create an app', 'build something new', 'kickstart', 'bootstrap', 'scaffold', 'init', 'setup', 'start fresh', 'new idea' (when it's a whole new product, not a feature), or mentions starting any new web application from scratch. Even if they just say 'I want to build X' where X is a new product — trigger this skill to set up the foundation before writing any feature code. Also trigger when the user says 'spin up', 'set up a repo', 'blank slate', 'from zero', 'greenfield', 'v2', 'rewrite', 'start over', or describes ANY new product concept without an existing codebase."
---

# Project Kickstarter

You take a user from zero to a deployed, working web app in a single session. No decisions required from them — you pick the best defaults, set up everything, and hand them a live URL with auth working, database connected, and CI/CD configured.

## Step 1: Pick the Template

Ask one question: **"What are you building?"** Then pick:

| Template | When to Use | What They Get |
|----------|------------|---------------|
| `saas` | Anything with users, accounts, payments | Auth + dashboard + Stripe + landing page |
| `content` | Blog, docs, SEO-driven site, content hub | MDX blog + SEO + sitemap + programmatic pages |
| `tool` | Calculator, analyzer, converter, tracker | Single-purpose UI + optional auth + share results |
| `api` | Backend service, webhook processor, data pipeline | Route handlers + cron + webhook receivers + queue |

If unclear, **default to `saas`**. See `references/setup-checklist.md` for template-specific steps.

## Step 2: Create the Project

Do not pause between steps. Do not ask for confirmation.

```bash
npx create-next-app@latest PROJECT_NAME \
  --typescript --tailwind --eslint --app --src-dir=false --import-alias="@/*" --use-npm
cd PROJECT_NAME && git init
```

## Step 3: Install Dependencies

```bash
# All templates
npm install @supabase/supabase-js @supabase/ssr
npm install -D prettier prettier-plugin-tailwindcss @types/node
```

**saas:** `npm install stripe @stripe/stripe-js @google/generative-ai lucide-react class-variance-authority clsx tailwind-merge`
**content:** `npm install @next/mdx @mdx-js/loader @mdx-js/react gray-matter reading-time @google/generative-ai lucide-react class-variance-authority clsx tailwind-merge`
**tool:** `npm install @google/generative-ai lucide-react class-variance-authority clsx tailwind-merge`
**api:** `npm install @google/generative-ai lucide-react class-variance-authority clsx tailwind-merge`

Then for all templates:
```bash
npx shadcn@latest init -d
```

shadcn components per template:
- **saas:** `npx shadcn@latest add button card input label separator toast dialog dropdown-menu avatar badge`
- **content:** `npx shadcn@latest add button card separator badge`
- **tool:** `npx shadcn@latest add button card input label toast progress`
- **api:** `npx shadcn@latest add button card badge`

## Step 4: Create the Folder Structure

See `references/folder-structure.md` for what each file does. For `saas`:

```
app/(auth)/login/page.tsx       app/(auth)/signup/page.tsx
app/auth/callback/route.ts      app/(dashboard)/layout.tsx
app/(dashboard)/page.tsx        app/(marketing)/page.tsx
app/api/health/route.ts         app/api/webhooks/stripe/route.ts
app/api/cron/route.ts           app/not-found.tsx
components/shared/navbar.tsx    components/shared/footer.tsx
components/shared/user-menu.tsx
lib/supabase/client.ts          lib/supabase/server.ts
lib/supabase/middleware.ts      lib/gemini/client.ts
lib/stripe/client.ts            lib/utils.ts
middleware.ts
```

## Step 5: Supabase Setup

```bash
npx supabase init && npx supabase start
```

### lib/supabase/client.ts — Browser client

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

### lib/supabase/server.ts — Server client

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
        getAll() { return cookieStore.getAll(); },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) => cookieStore.set(name, value, options));
          } catch { /* Called from Server Component — middleware handles refresh */ }
        },
      },
    }
  );
}
```

### lib/supabase/middleware.ts — Session refresh + route protection

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
        getAll() { return request.cookies.getAll(); },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value }) => request.cookies.set(name, value));
          supabaseResponse = NextResponse.next({ request });
          cookiesToSet.forEach(({ name, value, options }) => supabaseResponse.cookies.set(name, value, options));
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
  return supabaseResponse;
}
```

### middleware.ts (project root)

```typescript
import { type NextRequest } from "next/server";
import { updateSession } from "@/lib/supabase/middleware";

export async function middleware(request: NextRequest) {
  return await updateSession(request);
}

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)"],
};
```

### Database migration — supabase/migrations/00001_create_users.sql

```sql
create table public.profiles (
  id uuid references auth.users on delete cascade not null primary key,
  email text, full_name text, avatar_url text,
  created_at timestamptz default now() not null,
  updated_at timestamptz default now() not null
);
alter table public.profiles enable row level security;

create policy "Users can view own profile" on public.profiles
  for select using (auth.uid() = id);
create policy "Users can update own profile" on public.profiles
  for update using (auth.uid() = id);

create or replace function public.handle_new_user() returns trigger
language plpgsql security definer set search_path = '' as $$
begin
  insert into public.profiles (id, email, full_name, avatar_url)
  values (new.id, new.email, new.raw_user_meta_data ->> 'full_name', new.raw_user_meta_data ->> 'avatar_url');
  return new;
end; $$;

create trigger on_auth_user_created after insert on auth.users
  for each row execute procedure public.handle_new_user();

create or replace function public.handle_updated_at() returns trigger
language plpgsql as $$
begin new.updated_at = now(); return new; end; $$;

create trigger on_profile_updated before update on public.profiles
  for each row execute procedure public.handle_updated_at();
```

### Auth config in supabase/config.toml

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

## Step 6: Auth Pages

Create `app/(auth)/login/page.tsx` with email + Google OAuth login using shadcn Card, Input, Label, Button, Separator. The page calls `supabase.auth.signInWithPassword()` for email and `supabase.auth.signInWithOAuth({ provider: "google" })` for Google. On success, redirect to `/dashboard`.

Create `app/(auth)/signup/page.tsx` with the same pattern using `supabase.auth.signUp()`.

Create `app/auth/callback/route.ts` to handle OAuth redirects:

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
    if (!error) return NextResponse.redirect(`${origin}${next}`);
  }
  return NextResponse.redirect(`${origin}/login?error=auth_failed`);
}
```

## Step 7: Gemini API Client

```typescript
// lib/gemini/client.ts
import { GoogleGenerativeAI } from "@google/generative-ai";

if (!process.env.GEMINI_API_KEY) throw new Error("GEMINI_API_KEY is not set");
const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY);

export const gemini = {
  flash: genAI.getGenerativeModel({ model: "gemini-2.0-flash" }),           // Fast: classification, extraction
  pro: genAI.getGenerativeModel({ model: "gemini-2.5-pro-preview-06-05" }), // Best: content gen, analysis
  thinking: genAI.getGenerativeModel({ model: "gemini-2.5-flash-preview-05-20" }), // Reasoning: math, code
};

export async function generateText(prompt: string, tier: "flash" | "pro" | "thinking" = "flash") {
  const result = await gemini[tier].generateContent(prompt);
  return result.response.text();
}

export async function generateJSON<T>(prompt: string, tier: "flash" | "pro" | "thinking" = "flash"): Promise<T> {
  const result = await gemini[tier].generateContent(`${prompt}\n\nRespond ONLY with valid JSON.`);
  return JSON.parse(result.response.text()) as T;
}
```

## Step 8: Stripe (saas only)

```typescript
// lib/stripe/client.ts
import Stripe from "stripe";
if (!process.env.STRIPE_SECRET_KEY) throw new Error("STRIPE_SECRET_KEY is not set");

export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY, {
  apiVersion: "2024-12-18.acacia", typescript: true,
});

export async function createCheckoutSession(params: {
  priceId: string; userId: string; successUrl: string; cancelUrl: string;
}) {
  return stripe.checkout.sessions.create({
    mode: "subscription", payment_method_types: ["card"],
    line_items: [{ price: params.priceId, quantity: 1 }],
    success_url: params.successUrl, cancel_url: params.cancelUrl,
    client_reference_id: params.userId, metadata: { userId: params.userId },
  });
}
```

Create `app/api/webhooks/stripe/route.ts` that verifies the Stripe signature with `stripe.webhooks.constructEvent()`, then handles `checkout.session.completed` (save customer ID to profiles) and `customer.subscription.updated/deleted` events.

## Step 9: API Routes

**app/api/health/route.ts** — Returns `{ status: "ok", timestamp, environment }`.

**app/api/cron/route.ts** — Validates `Bearer CRON_SECRET` auth header, runs scheduled tasks.

## Step 10: Environment Variables

Create `.env.local.example` then `cp .env.local.example .env.local`. See `references/env-vars.md` for all variables. Key ones:

```bash
NEXT_PUBLIC_SUPABASE_URL=http://localhost:54321
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
GEMINI_API_KEY=your-gemini-api-key
STRIPE_SECRET_KEY=sk_test_...                      # saas only
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...     # saas only
STRIPE_WEBHOOK_SECRET=whsec_...                    # saas only
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
CRON_SECRET=your-cron-secret
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

## Step 11: ESLint + Prettier

Create `.prettierrc`:
```json
{ "semi": true, "singleQuote": false, "tabWidth": 2, "trailingComma": "es5", "printWidth": 100, "plugins": ["prettier-plugin-tailwindcss"] }
```

Add to `package.json` scripts: `"format": "prettier --write ."` and `"format:check": "prettier --check ."`.

## Step 12: Utilities

```typescript
// lib/utils.ts
import { type ClassValue, clsx } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) { return twMerge(clsx(inputs)); }

export function formatDate(date: string | Date): string {
  return new Intl.DateTimeFormat("en-US", { month: "short", day: "numeric", year: "numeric" }).format(new Date(date));
}

export function absoluteUrl(path: string): string {
  return `${process.env.NEXT_PUBLIC_APP_URL}${path}`;
}
```

## Step 13: .gitignore

Ensure it includes: `node_modules/`, `.next/`, `.env*.local`, `.vercel`, `*.tsbuildinfo`, `next-env.d.ts`, `.supabase/`.

## Step 14: IDEAS.md

Create `IDEAS.md` with the idea-parking-lot table format:
```markdown
# Ideas Parking Lot
| # | Idea | Impact | Confidence | Ease | ICE Score | Status |
|---|------|--------|------------|------|-----------|--------|
```

## Step 15: README

Generate with: project name, one-line description, tech stack badges, getting started steps, folder structure overview, deployment link.

## Step 16: Deploy

See `references/first-deploy.md` for the full walkthrough.

```bash
git add -A && git commit -m "Initial project setup"
git branch -M main
gh repo create PROJECT_NAME --private --source=. --push
npx vercel link && npx vercel --prod
```

## Step 17: Verify

1. `npm run dev` starts without errors
2. Landing page loads at `localhost:3000`
3. `/login` and `/signup` render
4. `/dashboard` redirects to `/login` when not authed
5. `/api/health` returns `{ status: "ok" }`
6. `npm run build` completes without errors
7. Supabase Studio accessible at `localhost:54323`

Tell the user: **"Your project is live. Here's what you have..."** and list everything with links.

## Template Variations

### content
Replace `(dashboard)` with `(blog)` containing `page.tsx` and `[slug]/page.tsx`. Add `content/` dir for MDX, `lib/mdx.ts` for parsing, `app/sitemap.ts`, `app/robots.ts`, `app/feed.xml/route.ts`. Skip Stripe. See `references/setup-checklist.md`.

### tool
Replace `(dashboard)` with `(tool)/page.tsx` and `(tool)/results/page.tsx`. Add `lib/calculations.ts`. Skip Stripe unless monetization mentioned. See `references/setup-checklist.md`.

### api
Remove all route groups. Keep `app/api/` routes and minimal `app/page.tsx` status page. Add `app/api/v1/` versioned routes, `lib/queue.ts`, `lib/validators.ts`. See `references/setup-checklist.md`.
