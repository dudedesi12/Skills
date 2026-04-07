# Optimization Reference

## Bundle Size Analysis

### Setting Up Bundle Analyzer

```bash
npm install @next/bundle-analyzer
```

```typescript
// next.config.ts
import bundleAnalyzer from "@next/bundle-analyzer";

const withBundleAnalyzer = bundleAnalyzer({
  enabled: process.env.ANALYZE === "true",
});

const nextConfig = {
  experimental: {
    optimizePackageImports: [
      "lucide-react",
      "@supabase/supabase-js",
      "date-fns",
      "lodash",
    ],
  },
};

export default withBundleAnalyzer(nextConfig);
```

Run analysis:
```bash
ANALYZE=true npm run build
```

### Reducing Bundle Size

1. **Use specific imports:**
```typescript
// Bad — imports the entire library
import { format } from "date-fns";

// Good — imports only what you need
import format from "date-fns/format";
```

2. **Dynamic imports for heavy components:**
```typescript
// app/dashboard/page.tsx
import dynamic from "next/dynamic";

const HeavyChart = dynamic(() => import("@/components/Chart"), {
  loading: () => <div className="h-64 animate-pulse bg-gray-200 rounded" />,
  ssr: false,
});

export default function DashboardPage() {
  return (
    <main>
      <h1>Dashboard</h1>
      <HeavyChart />
    </main>
  );
}
```

3. **Tree-shake with barrel file optimization:**
```typescript
// next.config.ts
const nextConfig = {
  experimental: {
    optimizePackageImports: ["your-icon-library", "your-component-library"],
  },
};
```

## Image Optimization

### Using next/image Correctly

```typescript
// components/OptimizedImage.tsx
import Image from "next/image";

export function OptimizedImage({
  src,
  alt,
  width,
  height,
}: {
  src: string;
  alt: string;
  width: number;
  height: number;
}) {
  return (
    <Image
      src={src}
      alt={alt}
      width={width}
      height={height}
      quality={80}
      placeholder="blur"
      blurDataURL="data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mN88P/BfwAJhAPk9NlCkQAAAABJRU5ErkJggg=="
      sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
    />
  );
}
```

### Supabase Storage Images

```typescript
// lib/image-url.ts
export function getOptimizedImageUrl(
  path: string,
  options: { width?: number; height?: number; quality?: number } = {}
): string {
  const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL;
  const { width = 800, height, quality = 80 } = options;

  let url = `${supabaseUrl}/storage/v1/render/image/public/${path}?width=${width}&quality=${quality}`;
  if (height) url += `&height=${height}`;

  return url;
}
```

## Edge Caching

### Cache-Control Headers for API Routes

```typescript
// app/api/public-data/route.ts
import { NextResponse } from "next/server";

export async function GET() {
  try {
    const data = await fetchPublicData();

    return NextResponse.json(data, {
      headers: {
        // Cache for 60 seconds, allow stale for 300 seconds while revalidating
        "Cache-Control": "public, s-maxage=60, stale-while-revalidate=300",
      },
    });
  } catch (error) {
    return NextResponse.json({ error: "Failed to load" }, { status: 500 });
  }
}
```

### Page-Level Caching with ISR

```typescript
// app/blog/[slug]/page.tsx
export const revalidate = 3600; // Revalidate every hour

export default async function BlogPost({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;
  const post = await getPost(slug);

  if (!post) {
    return <div>Post not found</div>;
  }

  return (
    <article>
      <h1>{post.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: post.content_html }} />
    </article>
  );
}
```

### unstable_cache for Data Fetching

```typescript
// lib/data.ts
import { unstable_cache } from "next/cache";
import { createClient } from "@/lib/supabase/server";

export const getCachedPosts = unstable_cache(
  async () => {
    const supabase = await createClient();
    const { data, error } = await supabase
      .from("posts")
      .select("id, title, slug, created_at")
      .eq("published", true)
      .order("created_at", { ascending: false })
      .limit(50);

    if (error) {
      console.error("Failed to fetch posts:", error);
      return [];
    }

    return data;
  },
  ["posts-list"],
  { revalidate: 300, tags: ["posts"] }
);
```

Invalidate the cache when data changes:

```typescript
// app/api/posts/route.ts
import { revalidateTag } from "next/cache";
import { NextResponse } from "next/server";

export async function POST(request: Request) {
  try {
    // ... create the post ...
    revalidateTag("posts");
    return NextResponse.json({ success: true });
  } catch (error) {
    return NextResponse.json({ error: "Failed" }, { status: 500 });
  }
}
```

## Cold Start Reduction

### 1. Keep Functions Small

Each API route becomes its own serverless function. Smaller functions = faster cold starts.

```typescript
// Bad: importing everything in one route
import { heavyLib } from "heavy-lib";
import { anotherLib } from "another-lib";

// Good: import only what this route needs
import { specificFunction } from "heavy-lib/specificFunction";
```

### 2. Use Edge Runtime for Latency-Sensitive Routes

```typescript
// app/api/auth/check/route.ts
export const runtime = "edge"; // Near-zero cold start

export async function GET(request: Request) {
  const token = request.headers.get("authorization");
  if (!token) {
    return new Response(JSON.stringify({ authenticated: false }), {
      status: 401,
      headers: { "Content-Type": "application/json" },
    });
  }
  return new Response(JSON.stringify({ authenticated: true }), {
    headers: { "Content-Type": "application/json" },
  });
}
```

### 3. Warm Critical Functions

For Pro plan users, use Vercel's cron to keep critical functions warm:

```json
// vercel.json
{
  "crons": [
    {
      "path": "/api/health",
      "schedule": "*/5 * * * *"
    }
  ]
}
```

### 4. Optimize Dependencies

```bash
# Find large dependencies
npx depcheck
npm ls --all | head -50

# Remove unused packages
npm uninstall unused-package
```

## Performance Monitoring

### Web Vitals Tracking

```typescript
// app/layout.tsx
import { SpeedInsights } from "@vercel/speed-insights/next";
import { Analytics } from "@vercel/analytics/next";

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        {children}
        <SpeedInsights />
        <Analytics />
      </body>
    </html>
  );
}
```

Install the packages:
```bash
npm install @vercel/speed-insights @vercel/analytics
```

This gives you Core Web Vitals (LCP, FID, CLS) data in the Vercel dashboard under **Speed Insights**.
