# Folder Structure

Detailed explanation of every folder and file in the project. This covers the `saas` template (the most comprehensive). Other templates use subsets of this structure.

---

## Root Directory

```
PROJECT_NAME/
├── app/                    # All routes, layouts, and API endpoints
├── components/             # Reusable React components
├── lib/                    # Shared utilities, clients, and business logic
├── public/                 # Static assets served at root URL
├── supabase/               # Supabase config, migrations, seed data
├── .env.local              # Environment variables (never committed)
├── .env.local.example      # Template for environment variables (committed)
├── .gitignore              # Files excluded from git
├── .prettierrc             # Prettier formatting configuration
├── IDEAS.md                # Idea parking lot (managed by idea-parking-lot skill)
├── README.md               # Project documentation
├── middleware.ts            # Next.js middleware (auth, redirects)
├── next.config.ts          # Next.js configuration
├── package.json            # Dependencies and scripts
├── tailwind.config.ts      # Tailwind CSS configuration
└── tsconfig.json           # TypeScript configuration
```

---

## app/ Directory

The `app/` directory uses Next.js App Router conventions. Every folder can become a route. Special files (`page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx`) have specific roles.

### Route Groups — Parentheses Folders

Folders wrapped in parentheses are **route groups**. They organize code without affecting the URL.

```
app/
├── (auth)/                 # Auth-related pages — URL: /login, /signup (not /auth/login)
├── (dashboard)/            # Protected pages — URL: /dashboard
├── (marketing)/            # Public pages — URL: / (landing page)
```

**Why route groups exist:** They let you apply different layouts to different sections. The marketing pages get a public navbar with "Sign in" buttons. The dashboard pages get a sidebar with navigation. The auth pages get a minimal centered layout. Each group has its own `layout.tsx`.

### app/(auth)/

```
(auth)/
├── login/
│   └── page.tsx            # Login form with email + Google OAuth
└── signup/
    └── page.tsx            # Signup form with email + Google OAuth
```

**What goes here:** Any page related to authentication. Login, signup, forgot password, reset password, email verification, magic link confirmation.

**URL mapping:** `/login`, `/signup` (the `(auth)` wrapper is invisible in the URL).

**Key patterns:**
- These pages use `"use client"` because they have form state and event handlers.
- They call `supabase.auth.signInWithPassword()` or `supabase.auth.signUp()` directly.
- Google OAuth uses `supabase.auth.signInWithOAuth({ provider: "google" })`.
- On success, they redirect to `/dashboard` using `router.push()` and `router.refresh()`.

### app/auth/callback/

```
auth/
└── callback/
    └── route.ts            # OAuth callback handler (NOT in a route group)
```

**Why this is separate from (auth):** This is NOT a page — it is a Route Handler. Supabase redirects here after Google OAuth. It exchanges the auth code for a session, then redirects the user to `/dashboard`. It lives at `/auth/callback` in the URL because it needs to match the redirect URL configured in Supabase and Google Cloud Console.

### app/(dashboard)/

```
(dashboard)/
├── layout.tsx              # Dashboard layout with sidebar, nav, user menu
└── page.tsx                # Dashboard home — main view after login
```

**What goes here:** Everything behind authentication. Dashboard, settings, profile, billing, any page that requires a logged-in user.

**Layout behavior:** The `layout.tsx` in this group fetches the user session on the server side using `createClient()` from `lib/supabase/server.ts`. It renders a sidebar, top navigation, and user menu. All child pages inherit this layout.

**Protection:** The middleware in `middleware.ts` checks if the user is authenticated before allowing access to any `/dashboard` route. Unauthenticated users get redirected to `/login`.

### app/(marketing)/

```
(marketing)/
└── page.tsx                # Landing page — hero, features, pricing, CTA
```

**What goes here:** Public-facing pages. Landing page, pricing page, about page, terms, privacy policy. Anything a visitor sees before signing up.

**URL mapping:** The `page.tsx` directly inside `(marketing)` maps to `/` (the root URL) because this is the index page of the route group.

### app/api/

```
api/
├── health/
│   └── route.ts            # GET /api/health — returns status, timestamp, environment
├── webhooks/
│   └── stripe/
│       └── route.ts        # POST /api/webhooks/stripe — handles Stripe events
└── cron/
    └── route.ts            # GET /api/cron — scheduled tasks triggered by Vercel Cron
```

**health/route.ts:** A simple endpoint that confirms the app is running. Useful for uptime monitoring (Vercel, UptimeRobot, Betterstack). Returns `{ status: "ok", timestamp: "...", environment: "..." }`.

**webhooks/stripe/route.ts:** Receives Stripe webhook events (checkout completed, subscription updated, subscription deleted). Verifies the webhook signature using `stripe.webhooks.constructEvent()` before processing. This is a POST-only endpoint.

**cron/route.ts:** Executed on a schedule by Vercel Cron Jobs. Protected by a `CRON_SECRET` Bearer token. Use this for daily cleanups, sending digest emails, syncing data, or any recurring task. Configure the schedule in `vercel.json`:

```json
{
  "crons": [
    {
      "path": "/api/cron",
      "schedule": "0 0 * * *"
    }
  ]
}
```

### app/layout.tsx

The root layout. Required by Next.js. Wraps every page in the entire app.

**Responsibilities:**
- Sets `<html>` and `<body>` tags
- Loads fonts (typically Inter from `next/font/google`)
- Applies global CSS (`globals.css`)
- Sets default metadata (title, description, Open Graph)
- Wraps children in any global providers (theme, toast)

### app/globals.css

Tailwind CSS directives and any global styles. Generated by `create-next-app` and extended by `shadcn/ui init`. Contains the `@tailwind` directives and CSS custom properties for the shadcn theme.

### app/not-found.tsx

Custom 404 page. Shown when a user navigates to a route that does not exist. Should have a friendly message and a link back to the home page.

---

## components/ Directory

```
components/
├── ui/                     # shadcn/ui components (auto-managed)
│   ├── button.tsx
│   ├── card.tsx
│   ├── input.tsx
│   ├── label.tsx
│   └── ...
└── shared/                 # Hand-written shared components
    ├── navbar.tsx
    ├── footer.tsx
    └── user-menu.tsx
```

### components/ui/

**Managed by shadcn/ui.** Do not create files here manually. Use `npx shadcn@latest add <component>` to add new components. You CAN edit these files to customize styling — shadcn copies the source code into your project, so they are yours to modify.

### components/shared/

**Hand-written components** used across multiple pages.

**navbar.tsx:** Top navigation bar. Shows different content based on auth state — "Sign in / Get started" for anonymous visitors, user avatar and dropdown for authenticated users. Used in marketing and dashboard layouts.

**footer.tsx:** Site footer with links to legal pages, social media, and copyright notice. Used in the marketing layout.

**user-menu.tsx:** Dropdown menu showing the user's avatar, name, email, and options (Settings, Billing, Sign out). Used in the dashboard layout. Calls `supabase.auth.signOut()` on sign out.

---

## lib/ Directory

```
lib/
├── supabase/
│   ├── client.ts           # Browser Supabase client (Client Components)
│   ├── server.ts           # Server Supabase client (Server Components, Route Handlers)
│   └── middleware.ts        # Middleware helper for session refresh
├── gemini/
│   └── client.ts           # Google Gemini AI client with model tiering
├── stripe/
│   └── client.ts           # Stripe server client with checkout helpers
└── utils.ts                # General utility functions
```

### lib/supabase/

Three separate client files because Supabase requires different client configurations depending on where it runs.

**client.ts** — Used in Client Components (files with `"use client"`). Creates a browser client using `createBrowserClient()`. The browser client reads and writes auth cookies automatically.

**server.ts** — Used in Server Components, Route Handlers, and Server Actions. Creates a server client using `createServerClient()` with the cookies API from `next/headers`. Must handle cookie get/set through the Next.js cookies interface.

**middleware.ts** — Used exclusively by the root `middleware.ts`. Refreshes the auth session on every request so tokens stay valid. Also handles redirecting unauthenticated users away from protected routes.

**Why three files?** Next.js has different execution contexts (browser, server, edge/middleware). Each has different APIs for reading cookies. Supabase needs cookies for auth, so it needs different client configurations for each context.

### lib/gemini/

**client.ts** — Wraps the Google Generative AI SDK with a tiered model system:

- **flash** (`gemini-2.0-flash`): Fastest and cheapest. Use for simple tasks like classification, extraction, formatting, and quick Q&A. Response time: ~500ms.
- **pro** (`gemini-2.5-pro-preview-06-05`): Most capable. Use for complex content generation, detailed analysis, and nuanced understanding. Response time: ~2-5s.
- **thinking** (`gemini-2.5-flash-preview-05-20`): Best for complex reasoning. Use for math, logic, code generation, multi-step reasoning. Response time: ~1-3s.

Also exports `generateText()` and `generateJSON()` helper functions for the most common patterns.

### lib/stripe/

**client.ts** — Initializes the Stripe server SDK and exports helper functions:

- `createCheckoutSession()` — Creates a Stripe Checkout session for subscriptions
- `createCustomerPortalSession()` — Creates a billing portal session for self-service management

Only used in the `saas` template. The Stripe client is server-only — never import this in Client Components.

### lib/utils.ts

General-purpose utility functions used across the project:

- `cn()` — Merges Tailwind CSS class names using `clsx` and `tailwind-merge`. Resolves conflicts (e.g., `cn("px-4", "px-6")` outputs `"px-6"`). Used by every shadcn component.
- `formatDate()` — Formats dates into human-readable strings using `Intl.DateTimeFormat`.
- `absoluteUrl()` — Prepends the app URL to a path. Useful for generating URLs for emails, Open Graph metadata, and API callbacks.

---

## middleware.ts (Root)

The root middleware runs on every request before it reaches a page or API route. It does two things:

1. **Refreshes the Supabase auth session** — Ensures the auth token stays valid by calling `supabase.auth.getUser()` on every request.
2. **Protects routes** — Redirects unauthenticated users from `/dashboard/*` to `/login`.

The `matcher` config excludes static files and images from middleware processing for performance.

---

## supabase/ Directory

```
supabase/
├── config.toml             # Supabase local dev configuration
├── migrations/
│   └── 00001_create_users.sql  # Initial schema with profiles + RLS
└── seed.sql                # Optional seed data for development
```

**config.toml:** Configuration for the local Supabase instance. Sets auth providers, site URL, redirect URLs, and feature flags. Committed to git so all developers share the same local config.

**migrations/:** SQL files that define your database schema. Run in order by filename. The first migration creates the `profiles` table, RLS policies, and triggers for auto-creating profiles on signup.

**seed.sql:** Optional file for inserting test data into the local database. Runs when you call `npx supabase db reset`.

---

## public/ Directory

```
public/
├── favicon.ico             # Browser tab icon
├── og-image.png            # Open Graph image for social sharing
└── ...                     # Any other static assets
```

Files in `public/` are served at the root URL. `public/og-image.png` is accessible at `https://yourapp.com/og-image.png`. Use this for images, fonts, or any static files that do not need processing.

---

## Configuration Files

**next.config.ts:** Next.js configuration. Set image domains, redirects, headers, and experimental features here.

**tailwind.config.ts:** Tailwind CSS configuration. Extended by shadcn/ui with custom colors, border radius, and animations. Add custom theme extensions here.

**tsconfig.json:** TypeScript configuration. The `@/*` path alias maps to the project root, so `import { cn } from "@/lib/utils"` works from any file.

**package.json:** Dependencies and npm scripts. Key scripts:
- `dev` — Start development server
- `build` — Production build
- `lint` — ESLint check
- `format` — Prettier format all files
- `format:check` — Prettier check without writing

**.prettierrc:** Prettier configuration with the Tailwind CSS plugin for automatic class sorting.

**.gitignore:** Excludes `node_modules/`, `.next/`, `.env*.local`, `.vercel`, `.supabase/`, and build artifacts.

---

## Template-Specific Variations

### Content Template Additions

```
content/
└── posts/                  # MDX blog posts
    └── hello-world.mdx     # Each file = one blog post with frontmatter
lib/
└── mdx.ts                  # MDX parsing with gray-matter and reading-time
app/
├── (blog)/
│   ├── page.tsx            # Blog index listing all posts
│   └── [slug]/
│       └── page.tsx        # Individual blog post page
├── sitemap.ts              # Dynamic sitemap for SEO
├── robots.ts               # Robots.txt configuration
└── feed.xml/
    └── route.ts            # RSS feed generator
components/
└── blog/
    ├── post-header.tsx     # Post title, date, reading time
    └── post-body.tsx       # MDX content renderer
```

### Tool Template Additions

```
app/
├── (tool)/
│   ├── page.tsx            # Main tool interface (input form)
│   └── results/
│       └── page.tsx        # Shareable results page
├── api/
│   └── analyze/
│       └── route.ts        # Processing endpoint
lib/
└── calculations.ts         # Core tool logic (separated from UI)
components/
└── tool/                   # Tool-specific components
```

### API Template Additions

```
app/
├── api/
│   └── v1/                 # Versioned API routes
│       └── example/
│           └── route.ts    # Example CRUD endpoint
├── api/
│   └── webhooks/
│       └── incoming/
│           └── route.ts    # Generic webhook receiver
lib/
├── validators/
│   └── index.ts            # Request validation and error handling
└── queue/
    └── index.ts            # Background job queue using Supabase
```
