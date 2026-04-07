# Sitemap Generation

## Basic Static Sitemap

For small sites with known pages.

```typescript
// app/sitemap.ts
import type { MetadataRoute } from "next";

export default function sitemap(): MetadataRoute.Sitemap {
  const baseUrl = "https://myapp.com";

  return [
    { url: baseUrl, lastModified: new Date(), changeFrequency: "daily", priority: 1 },
    { url: `${baseUrl}/pricing`, lastModified: new Date(), changeFrequency: "weekly", priority: 0.8 },
    { url: `${baseUrl}/about`, lastModified: new Date(), changeFrequency: "monthly", priority: 0.5 },
    { url: `${baseUrl}/blog`, lastModified: new Date(), changeFrequency: "daily", priority: 0.9 },
    { url: `${baseUrl}/faq`, lastModified: new Date(), changeFrequency: "weekly", priority: 0.7 },
    { url: `${baseUrl}/contact`, lastModified: new Date(), changeFrequency: "monthly", priority: 0.4 },
  ];
}
```

## Dynamic Sitemap from Supabase

For sites with content stored in Supabase.

```typescript
// app/sitemap.ts
import type { MetadataRoute } from "next";
import { createClient } from "@/lib/supabase/server";

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const baseUrl = "https://myapp.com";
  const supabase = await createClient();

  // Static pages
  const staticPages: MetadataRoute.Sitemap = [
    { url: baseUrl, lastModified: new Date(), changeFrequency: "daily", priority: 1 },
    { url: `${baseUrl}/pricing`, lastModified: new Date(), changeFrequency: "weekly", priority: 0.8 },
    { url: `${baseUrl}/about`, lastModified: new Date(), changeFrequency: "monthly", priority: 0.5 },
  ];

  // Blog posts
  let blogPages: MetadataRoute.Sitemap = [];
  try {
    const { data: posts } = await supabase
      .from("posts")
      .select("slug, updated_at")
      .eq("published", true)
      .order("updated_at", { ascending: false });

    blogPages = (posts ?? []).map((post) => ({
      url: `${baseUrl}/blog/${post.slug}`,
      lastModified: new Date(post.updated_at),
      changeFrequency: "weekly" as const,
      priority: 0.6,
    }));
  } catch (error) {
    console.error("Failed to fetch blog posts for sitemap:", error);
  }

  // Category pages
  let categoryPages: MetadataRoute.Sitemap = [];
  try {
    const { data: categories } = await supabase
      .from("categories")
      .select("slug, updated_at");

    categoryPages = (categories ?? []).map((cat) => ({
      url: `${baseUrl}/categories/${cat.slug}`,
      lastModified: new Date(cat.updated_at),
      changeFrequency: "weekly" as const,
      priority: 0.7,
    }));
  } catch (error) {
    console.error("Failed to fetch categories for sitemap:", error);
  }

  // City/location pages
  let cityPages: MetadataRoute.Sitemap = [];
  try {
    const { data: cities } = await supabase
      .from("cities")
      .select("slug, updated_at");

    cityPages = (cities ?? []).map((city) => ({
      url: `${baseUrl}/${city.slug}`,
      lastModified: new Date(city.updated_at),
      changeFrequency: "weekly" as const,
      priority: 0.7,
    }));
  } catch (error) {
    console.error("Failed to fetch cities for sitemap:", error);
  }

  return [...staticPages, ...blogPages, ...categoryPages, ...cityPages];
}
```

## Sitemap Index for Large Sites (50,000+ URLs)

Google limits sitemaps to 50,000 URLs each. For large sites, use a sitemap index that points to multiple sitemaps.

### Step 1: Create the Sitemap Index

```typescript
// app/sitemap.ts
import type { MetadataRoute } from "next";

export default function sitemap(): MetadataRoute.Sitemap {
  // This creates a sitemap index when you return sitemap IDs
  // Next.js handles the index automatically when you use multiple sitemap files
  const baseUrl = "https://myapp.com";

  return [
    { url: baseUrl, lastModified: new Date(), priority: 1 },
    { url: `${baseUrl}/pricing`, lastModified: new Date(), priority: 0.8 },
  ];
}
```

### Step 2: Create Numbered Sitemaps via Route Handler

For very large sites, use an API route to generate sitemaps dynamically.

```typescript
// app/api/sitemap/[page]/route.ts
import { createClient } from "@/lib/supabase/server";
import { NextResponse } from "next/server";

const URLS_PER_PAGE = 10000;

export async function GET(
  _request: Request,
  { params }: { params: Promise<{ page: string }> }
) {
  const { page } = await params;
  const pageNum = parseInt(page, 10);

  if (isNaN(pageNum) || pageNum < 1) {
    return NextResponse.json({ error: "Invalid page" }, { status: 400 });
  }

  try {
    const supabase = await createClient();
    const offset = (pageNum - 1) * URLS_PER_PAGE;

    const { data: posts, error } = await supabase
      .from("posts")
      .select("slug, updated_at")
      .eq("published", true)
      .order("created_at", { ascending: true })
      .range(offset, offset + URLS_PER_PAGE - 1);

    if (error) {
      console.error("Sitemap query error:", error);
      return NextResponse.json({ error: "Database error" }, { status: 500 });
    }

    const baseUrl = "https://myapp.com";

    const xml = `<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
${(posts ?? [])
  .map(
    (post) => `  <url>
    <loc>${baseUrl}/blog/${post.slug}</loc>
    <lastmod>${new Date(post.updated_at).toISOString()}</lastmod>
    <changefreq>weekly</changefreq>
    <priority>0.6</priority>
  </url>`
  )
  .join("\n")}
</urlset>`;

    return new Response(xml, {
      headers: {
        "Content-Type": "application/xml",
        "Cache-Control": "public, s-maxage=3600, stale-while-revalidate=600",
      },
    });
  } catch (error) {
    console.error("Sitemap generation error:", error);
    return NextResponse.json({ error: "Server error" }, { status: 500 });
  }
}
```

### Step 3: Sitemap Index Route

```typescript
// app/api/sitemap-index/route.ts
import { createClient } from "@/lib/supabase/server";

const URLS_PER_PAGE = 10000;

export async function GET() {
  try {
    const supabase = await createClient();

    const { count, error } = await supabase
      .from("posts")
      .select("*", { count: "exact", head: true })
      .eq("published", true);

    if (error) {
      console.error("Sitemap index count error:", error);
      return new Response("Error", { status: 500 });
    }

    const totalPages = Math.ceil((count ?? 0) / URLS_PER_PAGE);
    const baseUrl = "https://myapp.com";

    const xml = `<?xml version="1.0" encoding="UTF-8"?>
<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <sitemap>
    <loc>${baseUrl}/sitemap.xml</loc>
    <lastmod>${new Date().toISOString()}</lastmod>
  </sitemap>
${Array.from({ length: totalPages }, (_, i) => i + 1)
  .map(
    (page) => `  <sitemap>
    <loc>${baseUrl}/api/sitemap/${page}</loc>
    <lastmod>${new Date().toISOString()}</lastmod>
  </sitemap>`
  )
  .join("\n")}
</sitemapindex>`;

    return new Response(xml, {
      headers: {
        "Content-Type": "application/xml",
        "Cache-Control": "public, s-maxage=3600, stale-while-revalidate=600",
      },
    });
  } catch (error) {
    console.error("Sitemap index error:", error);
    return new Response("Error", { status: 500 });
  }
}
```

### Step 4: Update robots.txt

```typescript
// app/robots.ts
import type { MetadataRoute } from "next";

export default function robots(): MetadataRoute.Robots {
  return {
    rules: [
      {
        userAgent: "*",
        allow: "/",
        disallow: ["/api/", "/admin/", "/dashboard/"],
      },
    ],
    sitemap: [
      "https://myapp.com/sitemap.xml",
      "https://myapp.com/api/sitemap-index",
    ],
  };
}
```

## Revalidating the Sitemap

### On-Demand Revalidation

When new content is published, revalidate the sitemap path:

```typescript
// app/api/webhook/content-update/route.ts
import { revalidatePath } from "next/cache";
import { NextResponse } from "next/server";

export async function POST(request: Request) {
  try {
    const body = await request.json();
    const secret = request.headers.get("x-webhook-secret");

    if (secret !== process.env.WEBHOOK_SECRET) {
      return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
    }

    // Revalidate the sitemap
    revalidatePath("/sitemap.xml");

    // Revalidate the specific page
    if (body.slug) {
      revalidatePath(`/blog/${body.slug}`);
    }

    return NextResponse.json({ revalidated: true });
  } catch (error) {
    console.error("Revalidation error:", error);
    return NextResponse.json({ error: "Failed" }, { status: 500 });
  }
}
```

### Time-Based Revalidation

Set a revalidation interval on the sitemap:

```typescript
// app/sitemap.ts
export const revalidate = 3600; // Regenerate sitemap every hour

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  // ... sitemap generation
}
```

## Submitting to Google Search Console

1. Go to https://search.google.com/search-console
2. Add your property (domain or URL prefix)
3. Verify ownership (DNS TXT record or HTML file)
4. Go to **Sitemaps** in the left menu
5. Enter your sitemap URL: `https://myapp.com/sitemap.xml`
6. Click **Submit**

Google will crawl your sitemap and report any issues. Check back in a few days to see indexing status.
