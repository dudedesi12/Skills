# Cache Busting Guide

Every cache layer in the Next.js + Supabase + Vercel stack and how to clear each one.

## Cache Layers (In Order of Likelihood)

### 1. Browser Cache

**What it caches:** Static assets (JS, CSS, images), API responses, HTML pages.

**How to clear:**
- **Hard refresh:** Ctrl+Shift+R (Windows/Linux) or Cmd+Shift+R (Mac)
- **Full clear:** F12 > Application > Storage > Clear site data
- **Incognito test:** Open in a private/incognito window to bypass all browser cache

**When to suspect:** You changed code, dev server shows the change, but the browser still shows the old version.

### 2. Next.js Build Cache (.next folder)

**What it caches:** Compiled pages, server components, static assets from previous builds.

**How to clear:**
```bash
# Stop the dev server first (Ctrl+C), then:
rm -rf .next
npm run dev
```

**When to suspect:** You changed code but `npm run dev` still shows old behavior. Especially common after changing `next.config.js`, `tailwind.config.ts`, or `middleware.ts`.

### 3. Next.js Data Cache (fetch cache)

**What it caches:** Results from `fetch()` calls in Server Components. Cached indefinitely by default.

**How to clear:**
```typescript
// Option 1: Opt out of caching for this fetch
const res = await fetch(url, { cache: "no-store" });

// Option 2: Set a revalidation interval
const res = await fetch(url, { next: { revalidate: 60 } }); // Re-fetch every 60s

// Option 3: Force the entire page to be dynamic
export const dynamic = "force-dynamic";

// Option 4: Revalidate after a mutation
"use server";
import { revalidatePath } from "next/cache";
export async function updateItem() {
  await db.update(...);
  revalidatePath("/items");
}
```

**When to suspect:** Data was updated in the database but the page still shows old data.

### 4. Next.js Full Route Cache (ISR)

**What it caches:** Entire pre-rendered HTML pages.

**How to clear:**
```typescript
// Option 1: Revalidate a specific path
import { revalidatePath } from "next/cache";
revalidatePath("/dashboard");

// Option 2: Revalidate by tag
import { revalidateTag } from "next/cache";
revalidateTag("posts");

// Option 3: Force dynamic rendering (no cache)
export const dynamic = "force-dynamic";
```

**When to suspect:** Page content is stale even after refreshing. New data appears only after redeploying.

### 5. Next.js Router Cache (Client-side)

**What it caches:** Previously visited pages in the browser. Navigating back shows cached version.

**How to clear:**
```typescript
import { useRouter } from "next/navigation";
const router = useRouter();

// Force refresh current page
router.refresh();
```

**When to suspect:** Navigating between pages shows stale content, but full page reload shows fresh content.

### 6. Vercel CDN Cache (Edge Network)

**What it caches:** Static pages and assets at Vercel's edge locations worldwide.

**How to clear:**
- **Redeploy:** Any new deployment clears the CDN cache for that project
- **Override build cache:** Vercel Dashboard > Deployments > Redeploy > Toggle "Override Build Cache"
- **Programmatic:** Use the Vercel API to purge specific paths

**When to suspect:** Local build works fine, production shows old content even after deploying.

### 7. Vercel Build Cache

**What it caches:** `node_modules`, `.next` build output from previous deployments.

**How to clear:**
- Vercel Dashboard > Project > Settings > General > Build Cache > Toggle off and redeploy
- Or: Redeploy with "Override Build Cache" toggled on

**When to suspect:** Build uses old dependencies or configurations. Especially after major dependency upgrades.

### 8. npm/Node Module Cache

**What it caches:** Downloaded packages in `node_modules`.

**How to clear:**
```bash
rm -rf node_modules package-lock.json
npm install
```

**When to suspect:** After upgrading a package but the old version is still being used.

### 9. Supabase Query Cache

**What it caches:** Supabase does not cache queries by default, but connection pooling (PgBouncer) may reuse connections with cached prepared statements.

**How to clear:**
```bash
# Reset the connection pool
# Go to Supabase Dashboard > Settings > Database > Restart server
```

**When to suspect:** Schema changes (new columns, new tables) are not reflected in query results.

### 10. DNS Cache

**What it caches:** Domain name to IP address mappings.

**How to clear:**
```bash
# macOS
sudo dscacheutil -flushcache

# Windows
ipconfig /flushdns

# Linux
sudo systemd-resolve --flush-caches
```

**When to suspect:** Just changed DNS records (custom domain) but old IP is still being used.

## The Nuclear Option (Clear Everything)

When nothing else works, clear ALL caches at once:

```bash
# Stop the dev server
# Then run:
rm -rf .next node_modules package-lock.json
npm install
npm run dev
```

For production:
1. Push a new commit (even a whitespace change)
2. In Vercel, redeploy with "Override Build Cache" toggled on
3. After deploy, hard refresh the browser (Ctrl+Shift+R)

## Cache Strategy Cheat Sheet

| Data Type | Cache Strategy | Revalidation |
|-----------|---------------|--------------|
| Static pages (marketing, docs) | ISR with long revalidate | `revalidate: 3600` (1 hour) |
| Dashboard / user data | Dynamic, no cache | `dynamic = "force-dynamic"` |
| Blog posts | ISR with medium revalidate | `revalidate: 60` (1 minute) |
| API responses | Short cache | `revalidate: 10` (10 seconds) |
| User-specific data | No cache | `cache: "no-store"` |
| Public reference data | Long cache | `revalidate: 86400` (1 day) |
