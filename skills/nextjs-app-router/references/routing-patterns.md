# Next.js App Router — Routing Patterns

## Basic Routes

Every folder inside `app/` with a `page.tsx` becomes a route:

```
app/
  page.tsx              → /
  about/page.tsx        → /about
  contact/page.tsx      → /contact
  blog/page.tsx         → /blog
```

## Dynamic Routes — `[param]`

Square brackets create a dynamic segment. The value is passed as a param.

```tsx
// app/posts/[slug]/page.tsx
// Matches: /posts/hello-world, /posts/my-first-post, etc.

export default async function PostPage({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;

  return <h1>Post: {slug}</h1>;
}
```

### Nested Dynamic Routes

```tsx
// app/users/[userId]/posts/[postId]/page.tsx
// Matches: /users/123/posts/456

export default async function UserPostPage({
  params,
}: {
  params: Promise<{ userId: string; postId: string }>;
}) {
  const { userId, postId } = await params;

  return (
    <div>
      <p>User: {userId}</p>
      <p>Post: {postId}</p>
    </div>
  );
}
```

## Catch-All Routes — `[...slug]`

Three dots before the name catches all remaining segments as an array.

```tsx
// app/docs/[...slug]/page.tsx
// Matches: /docs/intro, /docs/guides/setup, /docs/api/auth/login

export default async function DocsPage({
  params,
}: {
  params: Promise<{ slug: string[] }>;
}) {
  const { slug } = await params;
  // /docs/guides/setup → slug = ["guides", "setup"]

  const path = slug.join("/");

  return <h1>Docs: {path}</h1>;
}
```

## Optional Catch-All Routes — `[[...slug]]`

Double brackets make the catch-all optional — it also matches the base path.

```tsx
// app/shop/[[...categories]]/page.tsx
// Matches: /shop, /shop/shoes, /shop/shoes/running

export default async function ShopPage({
  params,
}: {
  params: Promise<{ categories?: string[] }>;
}) {
  const { categories } = await params;

  if (!categories || categories.length === 0) {
    return <h1>All Products</h1>;
  }

  return <h1>Category: {categories.join(" > ")}</h1>;
}
```

## Route Groups — `(name)`

Parentheses create organizational folders that do NOT appear in the URL. Use them to apply different layouts to different sections.

```
app/
  (marketing)/
    layout.tsx           → Marketing layout (no sidebar)
    page.tsx             → /
    about/page.tsx       → /about
    pricing/page.tsx     → /pricing

  (dashboard)/
    layout.tsx           → Dashboard layout (with sidebar + nav)
    dashboard/page.tsx   → /dashboard
    settings/page.tsx    → /settings
    billing/page.tsx     → /billing
```

### Marketing Layout

```tsx
// app/(marketing)/layout.tsx
export default function MarketingLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div>
      <header className="border-b p-4">
        <nav className="flex items-center gap-6">
          <a href="/" className="text-xl font-bold">MyApp</a>
          <a href="/about" className="text-gray-600 hover:text-gray-900">About</a>
          <a href="/pricing" className="text-gray-600 hover:text-gray-900">Pricing</a>
          <a href="/dashboard" className="ml-auto rounded bg-blue-500 px-4 py-2 text-white">
            Dashboard
          </a>
        </nav>
      </header>
      <main>{children}</main>
    </div>
  );
}
```

### Dashboard Layout

```tsx
// app/(dashboard)/layout.tsx
import { createClient } from "@/lib/supabase/server";
import { redirect } from "next/navigation";

export default async function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();

  if (!user) {
    redirect("/login");
  }

  return (
    <div className="flex min-h-screen">
      <aside className="w-64 border-r bg-gray-50 p-4">
        <nav className="space-y-2">
          <a href="/dashboard" className="block rounded px-3 py-2 hover:bg-gray-200">Dashboard</a>
          <a href="/settings" className="block rounded px-3 py-2 hover:bg-gray-200">Settings</a>
          <a href="/billing" className="block rounded px-3 py-2 hover:bg-gray-200">Billing</a>
        </nav>
      </aside>
      <main className="flex-1 p-8">{children}</main>
    </div>
  );
}
```

## Parallel Routes — `@slot`

Parallel routes let you render multiple pages in the same layout simultaneously. Each `@folder` is a "slot" passed as a prop to the layout.

```
app/
  @analytics/page.tsx    → analytics slot
  @team/page.tsx         → team slot
  layout.tsx             → receives both slots
  page.tsx               → main content
```

```tsx
// app/layout.tsx (or a nested layout)
export default function Layout({
  children,
  analytics,
  team,
}: {
  children: React.ReactNode;
  analytics: React.ReactNode;
  team: React.ReactNode;
}) {
  return (
    <div className="grid grid-cols-2 gap-4 p-8">
      <div className="col-span-2">{children}</div>
      <div>{analytics}</div>
      <div>{team}</div>
    </div>
  );
}
```

### Parallel Route Default

When a parallel route does not have a matching page for the current URL, create a `default.tsx`:

```tsx
// app/@analytics/default.tsx
export default function AnalyticsDefault() {
  return null; // or a fallback UI
}
```

## Intercepting Routes

Intercepting routes let you load a route from another part of your app within the current layout. Commonly used for modals.

Convention:
- `(.)` — same level
- `(..)` — one level up
- `(..)(..)` — two levels up
- `(...)` — from root

### Example: Photo Modal

```
app/
  feed/
    page.tsx                    → /feed (shows grid of photos)
    @modal/
      (..)photo/[id]/page.tsx   → intercepts /photo/[id] when navigating from /feed
      default.tsx
  photo/
    [id]/
      page.tsx                  → /photo/[id] (full page, direct URL access)
```

```tsx
// app/feed/page.tsx
import Link from "next/link";

export default function FeedPage() {
  const photos = [
    { id: "1", src: "/photo1.jpg" },
    { id: "2", src: "/photo2.jpg" },
  ];

  return (
    <div className="grid grid-cols-3 gap-4 p-8">
      {photos.map((photo) => (
        <Link key={photo.id} href={`/photo/${photo.id}`}>
          <img src={photo.src} alt="" className="rounded" />
        </Link>
      ))}
    </div>
  );
}
```

```tsx
// app/feed/@modal/(..)photo/[id]/page.tsx
// This shows as a MODAL when navigating from /feed
export default async function PhotoModal({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;

  return (
    <div className="fixed inset-0 flex items-center justify-center bg-black/50">
      <div className="rounded-lg bg-white p-6">
        <h2>Photo {id}</h2>
        <img src={`/photo${id}.jpg`} alt="" className="max-w-lg rounded" />
      </div>
    </div>
  );
}
```

```tsx
// app/feed/@modal/default.tsx
export default function Default() {
  return null;
}
```

```tsx
// app/photo/[id]/page.tsx
// This shows as a FULL PAGE when accessed directly
export default async function PhotoPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;

  return (
    <div className="flex min-h-screen items-center justify-center">
      <img src={`/photo${id}.jpg`} alt="" className="max-w-2xl rounded" />
    </div>
  );
}
```

## Private Folders — `_folder`

Underscore prefix opts a folder out of routing entirely. Use for components, utilities, etc.

```
app/
  _components/
    header.tsx          → NOT a route, just a component
    footer.tsx
  _lib/
    utils.ts
  page.tsx              → /
```

## Complete Route Structure Example

```
app/
  (marketing)/
    layout.tsx
    page.tsx                          → /
    about/page.tsx                    → /about
    pricing/page.tsx                  → /pricing
    blog/
      page.tsx                        → /blog
      [slug]/page.tsx                 → /blog/my-post

  (app)/
    layout.tsx
    dashboard/
      page.tsx                        → /dashboard
      loading.tsx
      error.tsx
      @stats/page.tsx                 → parallel slot
      @activity/page.tsx              → parallel slot

    settings/
      page.tsx                        → /settings
      profile/page.tsx                → /settings/profile
      billing/page.tsx                → /settings/billing

    projects/
      page.tsx                        → /projects
      [projectId]/
        page.tsx                      → /projects/abc123
        settings/page.tsx             → /projects/abc123/settings

  docs/
    [[...slug]]/page.tsx              → /docs, /docs/intro, /docs/api/auth

  api/
    health/route.ts                   → GET /api/health
    webhooks/stripe/route.ts          → POST /api/webhooks/stripe
```
