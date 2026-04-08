---
name: performance-tuner
description: "Use this skill whenever the user mentions performance, slow, Core Web Vitals, LCP, FID, INP, CLS, render, re-render, memoization, React.memo, useMemo, useCallback, lazy loading, dynamic import, next/dynamic, next/image, image optimization, font optimization, next/font, bundle size, bundle analysis, profiling, React Profiler, 'why is my app slow', 'page takes too long', 'too many re-renders', virtualization, infinite scroll, debounce, throttle, 'optimize my app', or ANY performance optimization task — even if they don't explicitly say 'performance'. This skill makes your app fast."
---

# Performance Tuner

Make your Next.js app fast. This skill focuses on React-level performance and measurement. Cross-reference: vercel-power-user for ISR/caching and deployment-ops for build optimization.

## Stack Context

- Next.js App Router (TypeScript)
- Tailwind CSS
- Supabase (database, auth)
- Vercel (deployment)

## 1. Core Web Vitals

The three metrics Google uses to rank your site:

| Metric | What It Measures | Good | Needs Work | Poor |
|--------|-----------------|------|------------|------|
| **LCP** (Largest Contentful Paint) | How fast the main content loads | < 2.5s | 2.5-4s | > 4s |
| **INP** (Interaction to Next Paint) | How fast the app responds to clicks | < 200ms | 200-500ms | > 500ms |
| **CLS** (Cumulative Layout Shift) | How much the page jumps around | < 0.1 | 0.1-0.25 | > 0.25 |

### Measuring with web-vitals

```bash
npm install web-vitals
```

```typescript
// lib/vitals.ts
import { onLCP, onINP, onCLS } from "web-vitals";

export function reportWebVitals() {
  onLCP((metric) => {
    console.log("LCP:", metric.value, "ms");
    sendToAnalytics("LCP", metric.value);
  });
  onINP((metric) => {
    console.log("INP:", metric.value, "ms");
    sendToAnalytics("INP", metric.value);
  });
  onCLS((metric) => {
    console.log("CLS:", metric.value);
    sendToAnalytics("CLS", metric.value);
  });
}

function sendToAnalytics(name: string, value: number) {
  // Send to your analytics endpoint
  if (typeof navigator.sendBeacon === "function") {
    navigator.sendBeacon("/api/vitals", JSON.stringify({ name, value }));
  }
}
```

```tsx
// app/layout.tsx — client wrapper
"use client";
import { useEffect } from "react";
import { reportWebVitals } from "@/lib/vitals";

export function VitalsReporter() {
  useEffect(() => { reportWebVitals(); }, []);
  return null;
}
```

## 2. React Rendering Optimization

### React.memo — Prevent Re-renders of Unchanged Components

```tsx
// BEFORE: UserCard re-renders every time parent renders
function UserCard({ user }: { user: User }) {
  return <div className="rounded border p-4">{user.name}</div>;
}

// AFTER: Only re-renders when `user` prop changes
const UserCard = memo(function UserCard({ user }: { user: User }) {
  return <div className="rounded border p-4">{user.name}</div>;
});
```

**When to use React.memo:**
- Component renders often with the same props
- Component is expensive to render (large list, charts)
- Component is deep in the tree and parent re-renders frequently

**When NOT to use React.memo:**
- Component always receives new props (new objects/arrays each render)
- Component is cheap to render (simple text, small JSX)
- Component renders infrequently

### useMemo — Cache Expensive Computations

```tsx
"use client";
import { useMemo, useState } from "react";

export function AssessmentResults({ assessments }: { assessments: Assessment[] }) {
  const [filter, setFilter] = useState("");

  // WITHOUT useMemo: recalculates on EVERY render (even typing in search)
  // const stats = calculateStats(assessments);

  // WITH useMemo: only recalculates when assessments array changes
  const stats = useMemo(() => calculateStats(assessments), [assessments]);

  const filtered = useMemo(
    () => assessments.filter((a) => a.occupation.toLowerCase().includes(filter.toLowerCase())),
    [assessments, filter]
  );

  return (
    <div>
      <p>Average score: {stats.averageScore}</p>
      <input value={filter} onChange={(e) => setFilter(e.target.value)} placeholder="Filter..." />
      {filtered.map((a) => <AssessmentCard key={a.id} assessment={a} />)}
    </div>
  );
}

function calculateStats(assessments: Assessment[]) {
  // Expensive: iterates all assessments
  const total = assessments.reduce((sum, a) => sum + a.pointsScore, 0);
  return { averageScore: Math.round(total / assessments.length), total: assessments.length };
}
```

### useCallback — Stable Function References

```tsx
"use client";
import { useCallback, useState } from "react";

export function UserList({ users }: { users: User[] }) {
  const [selected, setSelected] = useState<string | null>(null);

  // WITHOUT useCallback: new function on every render → children re-render
  // const handleSelect = (id: string) => setSelected(id);

  // WITH useCallback: same function reference → children skip re-render
  const handleSelect = useCallback((id: string) => {
    setSelected(id);
  }, []);

  return (
    <ul>
      {users.map((user) => (
        <UserRow key={user.id} user={user} onSelect={handleSelect} isSelected={selected === user.id} />
      ))}
    </ul>
  );
}

// Combine with React.memo for maximum effect
const UserRow = memo(function UserRow({ user, onSelect, isSelected }: {
  user: User;
  onSelect: (id: string) => void;
  isSelected: boolean;
}) {
  return (
    <li
      onClick={() => onSelect(user.id)}
      className={`cursor-pointer p-3 ${isSelected ? "bg-blue-50" : ""}`}
    >
      {user.name}
    </li>
  );
});
```

See `references/react-perf-patterns.md` for virtualization, debouncing, and anti-patterns.

## 3. Component Profiling

### React DevTools Profiler

1. Install the React DevTools browser extension
2. Open DevTools → Profiler tab
3. Click "Start profiling" (record button)
4. Interact with your app
5. Click "Stop profiling"
6. Look for components that render too often or take too long

**What to look for:**
- Components rendering when they shouldn't (no prop changes)
- Components taking > 16ms to render (causes dropped frames)
- Deep component trees re-rendering from the top

### why-did-you-render (Development Only)

```bash
npm install -D @welldone-software/why-did-you-render
```

```typescript
// lib/wdyr.ts — import in layout.tsx in dev only
import React from "react";

if (process.env.NODE_ENV === "development") {
  const whyDidYouRender = require("@welldone-software/why-did-you-render");
  whyDidYouRender(React, {
    trackAllPureComponents: true,
    logOnDifferentValues: true,
  });
}
```

## 4. Dynamic Imports and Code Splitting

```tsx
import dynamic from "next/dynamic";

// Heavy component loaded only when needed
const Chart = dynamic(() => import("@/components/chart"), {
  loading: () => <div className="h-64 animate-pulse rounded bg-gray-100" />,
  ssr: false, // Don't render on server (e.g., for browser-only libraries)
});

// Conditionally loaded based on user action
const PDFExporter = dynamic(() => import("@/components/pdf-exporter"), {
  loading: () => <p>Loading exporter...</p>,
});

export function Dashboard() {
  const [showExport, setShowExport] = useState(false);

  return (
    <div>
      <Chart data={chartData} />
      <button onClick={() => setShowExport(true)}>Export PDF</button>
      {showExport && <PDFExporter />}
    </div>
  );
}
```

## 5. Image Optimization

```tsx
import Image from "next/image";

// Hero image — priority loading (above the fold)
<Image
  src="/hero.jpg"
  alt="Immigration pathway illustration"
  width={1200}
  height={600}
  priority
  className="rounded-lg"
/>

// Thumbnail — lazy loaded with blur placeholder
<Image
  src="/avatar.jpg"
  alt="User avatar"
  width={48}
  height={48}
  placeholder="blur"
  blurDataURL="data:image/jpeg;base64,/9j/4AAQSkZJRg..."
  className="rounded-full"
/>

// Full-width responsive image
<div className="relative aspect-video">
  <Image
    src="/banner.jpg"
    alt="Assessment results banner"
    fill
    sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
    className="object-cover"
  />
</div>
```

See `references/next-image-guide.md` for the complete image optimization reference.

## 6. Font Optimization

```tsx
// app/layout.tsx
import { Inter } from "next/font/google";

const inter = Inter({
  subsets: ["latin"],
  display: "swap", // Show fallback font until custom font loads
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={inter.className}>
      <body>{children}</body>
    </html>
  );
}
```

**Font display strategies:**
- `swap` — Show fallback font immediately, swap when loaded (best for body text)
- `optional` — Use fallback if font doesn't load fast enough (best for less critical text)
- `block` — Hide text until font loads (use sparingly — bad for LCP)

## 7. Database Query Performance

```typescript
// BAD: Select everything
const { data } = await supabase.from("profiles").select("*");

// GOOD: Select only what you need
const { data } = await supabase.from("profiles").select("id, full_name, avatar_url");

// BAD: Fetch all, then filter in JavaScript
const { data } = await supabase.from("assessments").select("*");
const recent = data?.filter((a) => new Date(a.created_at) > lastWeek);

// GOOD: Filter in the database
const { data } = await supabase
  .from("assessments")
  .select("id, visa_subclass, points_score, created_at")
  .gte("created_at", lastWeek.toISOString())
  .order("created_at", { ascending: false })
  .limit(20);

// GOOD: Parallel fetching
const [profileResult, assessmentsResult, notificationsResult] = await Promise.all([
  supabase.from("profiles").select("id, full_name").eq("id", userId).single(),
  supabase.from("assessments").select("id, result").eq("user_id", userId).limit(5),
  supabase.from("notifications").select("id, message").eq("user_id", userId).eq("read", false),
]);
```

## 8. Bundle Analysis

```bash
npm install -D @next/bundle-analyzer
```

```typescript
// next.config.ts
import withBundleAnalyzer from "@next/bundle-analyzer";

const config = withBundleAnalyzer({
  enabled: process.env.ANALYZE === "true",
})({
  // your existing next config
});

export default config;
```

```bash
ANALYZE=true npm run build
# Opens a treemap showing bundle sizes
```

**Common findings and fixes:**
- Large icon library → Import specific icons: `import { Check } from "lucide-react"`
- Moment.js / date-fns → Use `Intl.DateTimeFormat` or import specific functions
- Full lodash → Use `lodash-es` or individual imports

## 9. Performance Budget

Set limits and alert when exceeded:

```json
// package.json
{
  "scripts": {
    "perf:check": "npx bundlesize",
    "perf:lighthouse": "npx lighthouse http://localhost:3000 --output json --chrome-flags='--headless'"
  }
}
```

## Rules

1. **Measure before optimizing** — Use React DevTools Profiler to find the actual bottleneck. Don't guess.
2. **Don't memoize everything** — React.memo/useMemo/useCallback have overhead. Only use when profiling shows a problem.
3. **Select only needed columns** — `select("*")` transfers unnecessary data. List the columns you need.
4. **Fetch in parallel** — Use `Promise.all` for independent data fetches. Never await sequentially.
5. **Use `priority` on hero images** — The first image the user sees should have `priority` prop.
6. **Use `display: swap` for fonts** — Users see content immediately instead of a blank page.
7. **Dynamic import heavy libraries** — Charts, PDF generators, editors — load them on demand.
8. **Set image `sizes`** — Without `sizes`, the browser downloads the full-width image on mobile.
9. **Profile production builds** — Development mode is slower. Always test performance with `npm run build && npm start`.
10. **Run bundle analyzer monthly** — Dependencies creep in. Check your bundle size regularly.

See `references/` for detailed patterns:
- `react-perf-patterns.md` — Virtualization, debouncing, render optimization
- `measurement-tools.md` — Lighthouse CI, web-vitals, PerformanceObserver
- `next-image-guide.md` — Complete next/image reference
