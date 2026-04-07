---
name: seo-programmatic
description: "Use this skill whenever the user mentions SEO, search engine, Google, meta tags, title tags, descriptions, Open Graph, OG image, Twitter card, sitemap, robots.txt, canonical URL, structured data, JSON-LD, schema markup, programmatic pages, landing pages, blog posts, internal linking, Core Web Vitals, page speed, indexing, crawling, search console, rich snippets, FAQ schema, breadcrumbs, 'how do I rank', 'get more traffic', or ANY search visibility task — even if they don't explicitly say 'SEO'. This skill drives organic traffic at scale."
---

# SEO & Programmatic Pages

Drive organic traffic with proper metadata, structured data, sitemaps, and programmatic page generation using Next.js App Router and Supabase.

## Next.js Metadata API

### Static Metadata

For pages where metadata does not change based on dynamic data.

```typescript
// app/page.tsx
import type { Metadata } from "next";

export const metadata: Metadata = {
  title: "MyApp — Build Faster",
  description: "The easiest way to build and ship web applications. Start free today.",
  openGraph: {
    title: "MyApp — Build Faster",
    description: "The easiest way to build and ship web applications.",
    url: "https://myapp.com",
    siteName: "MyApp",
    images: [
      {
        url: "https://myapp.com/og-image.png",
        width: 1200,
        height: 630,
        alt: "MyApp — Build Faster",
      },
    ],
    locale: "en_US",
    type: "website",
  },
  twitter: {
    card: "summary_large_image",
    title: "MyApp — Build Faster",
    description: "The easiest way to build and ship web applications.",
    images: ["https://myapp.com/og-image.png"],
  },
  alternates: {
    canonical: "https://myapp.com",
  },
};

export default function HomePage() {
  return <main>Your page content</main>;
}
```

### Metadata Template in Layout

Set a template so child pages inherit a consistent title pattern.

```typescript
// app/layout.tsx
import type { Metadata } from "next";

export const metadata: Metadata = {
  metadataBase: new URL("https://myapp.com"),
  title: {
    default: "MyApp — Build Faster",
    template: "%s | MyApp",
  },
  description: "The easiest way to build and ship web applications.",
  openGraph: {
    siteName: "MyApp",
    locale: "en_US",
    type: "website",
  },
  robots: {
    index: true,
    follow: true,
    googleBot: {
      index: true,
      follow: true,
      "max-video-preview": -1,
      "max-image-preview": "large",
      "max-snippet": -1,
    },
  },
};
```

Now child pages only need to set their own title and description:

```typescript
// app/pricing/page.tsx
import type { Metadata } from "next";

export const metadata: Metadata = {
  title: "Pricing", // Renders as "Pricing | MyApp"
  description: "Simple, transparent pricing. Start free, upgrade when you need to.",
  alternates: {
    canonical: "https://myapp.com/pricing",
  },
};
```

### Dynamic Metadata with generateMetadata

For pages where metadata depends on database data (blog posts, product pages, etc.).

```typescript
// app/blog/[slug]/page.tsx
import type { Metadata } from "next";
import { createClient } from "@/lib/supabase/server";
import { notFound } from "next/navigation";

interface PageProps {
  params: Promise<{ slug: string }>;
}

export async function generateMetadata({ params }: PageProps): Promise<Metadata> {
  const { slug } = await params;
  const supabase = await createClient();

  const { data: post, error } = await supabase
    .from("posts")
    .select("title, excerpt, og_image, published_at")
    .eq("slug", slug)
    .eq("published", true)
    .single();

  if (error || !post) {
    return { title: "Post Not Found" };
  }

  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      type: "article",
      publishedTime: post.published_at,
      url: `https://myapp.com/blog/${slug}`,
      images: post.og_image
        ? [{ url: post.og_image, width: 1200, height: 630, alt: post.title }]
        : [],
    },
    twitter: {
      card: "summary_large_image",
      title: post.title,
      description: post.excerpt,
      images: post.og_image ? [post.og_image] : [],
    },
    alternates: {
      canonical: `https://myapp.com/blog/${slug}`,
    },
  };
}

export default async function BlogPostPage({ params }: PageProps) {
  const { slug } = await params;
  const supabase = await createClient();

  const { data: post, error } = await supabase
    .from("posts")
    .select("*")
    .eq("slug", slug)
    .eq("published", true)
    .single();

  if (error || !post) {
    notFound();
  }

  return (
    <article>
      <h1>{post.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: post.content_html }} />
    </article>
  );
}
```

See `references/metadata-patterns.md` for more patterns.

## JSON-LD Structured Data

Structured data helps Google show rich results (FAQ dropdowns, breadcrumbs, star ratings, etc.).

### JSON-LD Component

```typescript
// components/JsonLd.tsx
export function JsonLd({ data }: { data: Record<string, unknown> }) {
  return (
    <script
      type="application/ld+json"
      dangerouslySetInnerHTML={{ __html: JSON.stringify(data) }}
    />
  );
}
```

### FAQ Schema

```typescript
// app/faq/page.tsx
import { JsonLd } from "@/components/JsonLd";

const faqs = [
  { question: "What is MyApp?", answer: "MyApp is a platform for building web applications quickly." },
  { question: "How much does it cost?", answer: "MyApp is free to start. Paid plans begin at $19/month." },
  { question: "Do I need coding experience?", answer: "No. MyApp is designed for beginners and experts alike." },
];

export default function FaqPage() {
  const faqSchema = {
    "@context": "https://schema.org",
    "@type": "FAQPage",
    mainEntity: faqs.map((faq) => ({
      "@type": "Question",
      name: faq.question,
      acceptedAnswer: {
        "@type": "Answer",
        text: faq.answer,
      },
    })),
  };

  return (
    <main>
      <JsonLd data={faqSchema} />
      <h1>Frequently Asked Questions</h1>
      {faqs.map((faq, i) => (
        <div key={i}>
          <h2>{faq.question}</h2>
          <p>{faq.answer}</p>
        </div>
      ))}
    </main>
  );
}
```

See `references/structured-data.md` for Article, BreadcrumbList, HowTo, Product, and Organization schemas with full examples.

## Sitemap Generation

### Static + Dynamic Sitemap

```typescript
// app/sitemap.ts
import type { MetadataRoute } from "next";
import { createClient } from "@/lib/supabase/server";

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const baseUrl = "https://myapp.com";

  // Static pages
  const staticPages: MetadataRoute.Sitemap = [
    { url: baseUrl, lastModified: new Date(), changeFrequency: "daily", priority: 1 },
    { url: `${baseUrl}/pricing`, lastModified: new Date(), changeFrequency: "weekly", priority: 0.8 },
    { url: `${baseUrl}/about`, lastModified: new Date(), changeFrequency: "monthly", priority: 0.5 },
    { url: `${baseUrl}/faq`, lastModified: new Date(), changeFrequency: "weekly", priority: 0.7 },
  ];

  // Dynamic pages from Supabase
  let dynamicPages: MetadataRoute.Sitemap = [];
  try {
    const supabase = await createClient();
    const { data: posts } = await supabase
      .from("posts")
      .select("slug, updated_at")
      .eq("published", true);

    dynamicPages = (posts ?? []).map((post) => ({
      url: `${baseUrl}/blog/${post.slug}`,
      lastModified: new Date(post.updated_at),
      changeFrequency: "weekly" as const,
      priority: 0.6,
    }));
  } catch (error) {
    console.error("Failed to fetch posts for sitemap:", error);
  }

  return [...staticPages, ...dynamicPages];
}
```

See `references/sitemap-generation.md` for large-site sitemap strategies.

## robots.txt

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
    sitemap: "https://myapp.com/sitemap.xml",
  };
}
```

## Canonical URLs and Duplicate Content

Every page should have a canonical URL. Set it in metadata:

```typescript
export const metadata: Metadata = {
  alternates: {
    canonical: "https://myapp.com/your-page",
  },
};
```

For dynamic pages, set it in `generateMetadata`:

```typescript
export async function generateMetadata({ params }: PageProps): Promise<Metadata> {
  const { slug } = await params;
  return {
    alternates: {
      canonical: `https://myapp.com/blog/${slug}`,
    },
  };
}
```

This tells Google which URL is the "real" one, preventing duplicate content issues from www vs non-www, trailing slashes, or query parameters.

## OG Images

### Static OG Image

Place a file at `app/opengraph-image.png` (1200x630 pixels). Next.js serves it automatically.

### Dynamic OG Images with ImageResponse

Create `app/blog/[slug]/opengraph-image.tsx` with `export const runtime = "edge"`, `export const size = { width: 1200, height: 630 }`, and a default export function that fetches the post title from Supabase and returns `new ImageResponse(...)` with JSX-like syntax for the image layout. Next.js automatically links this as the OG image for that route.

## Internal Linking Strategy

Internal links help Google discover pages and distribute ranking authority.

Key internal linking patterns:

1. **Related Posts component** — Query Supabase for posts in the same category, excluding the current post, and render as links below the article.
2. **Breadcrumb navigation** — Show the page hierarchy (Home / Blog / Post Title) with both visible links and BreadcrumbList JSON-LD schema. Use `aria-label="Breadcrumb"` on the `<nav>` and `aria-current="page"` on the last item.

## Core Web Vitals Optimization

### 1. Largest Contentful Paint (LCP)

- Use `priority` on above-the-fold images: `<Image priority ... />`
- Preload critical fonts with `next/font`
- Avoid layout shifts from loading content

### 2. Cumulative Layout Shift (CLS)

- Always set `width` and `height` on images
- Use skeleton loaders with fixed dimensions
- Avoid injecting content above existing content

### 3. Interaction to Next Paint (INP)

- Keep event handlers fast — offload heavy work to Web Workers or server
- Use `useTransition` for non-urgent updates
- Avoid blocking the main thread

## Programmatic Page Templates

Generate hundreds of SEO-optimized pages from data.

### City Pages

```typescript
// app/[city]/page.tsx
import type { Metadata } from "next";
import { createClient } from "@/lib/supabase/server";
import { notFound } from "next/navigation";
import { JsonLd } from "@/components/JsonLd";

interface PageProps {
  params: Promise<{ city: string }>;
}

export async function generateStaticParams() {
  try {
    const supabase = await createClient();
    const { data } = await supabase.from("cities").select("slug");
    return (data ?? []).map((city) => ({ city: city.slug }));
  } catch {
    return [];
  }
}

export async function generateMetadata({ params }: PageProps): Promise<Metadata> {
  const { city } = await params;
  const supabase = await createClient();
  const { data } = await supabase.from("cities").select("name").eq("slug", city).single();

  if (!data) return { title: "Not Found" };

  return {
    title: `Best Services in ${data.name}`,
    description: `Find the top-rated services in ${data.name}. Verified reviews, transparent pricing.`,
    alternates: { canonical: `https://myapp.com/${city}` },
  };
}

export default async function CityPage({ params }: PageProps) {
  const { city } = await params;
  const supabase = await createClient();

  const { data: cityData, error } = await supabase
    .from("cities")
    .select("name, state, description, population")
    .eq("slug", city)
    .single();

  if (error || !cityData) notFound();

  const localBusinessSchema = {
    "@context": "https://schema.org",
    "@type": "WebPage",
    name: `Best Services in ${cityData.name}`,
    description: `Find top-rated services in ${cityData.name}, ${cityData.state}.`,
    url: `https://myapp.com/${city}`,
  };

  return (
    <main>
      <JsonLd data={localBusinessSchema} />
      <h1 className="text-3xl font-bold mb-4">
        Best Services in {cityData.name}, {cityData.state}
      </h1>
      <p className="text-gray-600 dark:text-gray-400 mb-8">
        {cityData.description}
      </p>
      {/* List services, reviews, etc. */}
    </main>
  );
}
```

Category pages and tool pages follow the same pattern as City Pages above: use `generateStaticParams` to list all slugs, `generateMetadata` for dynamic SEO, and fetch content from Supabase in the page component. See `references/structured-data.md` and `references/sitemap-generation.md` for supporting these programmatic pages with proper schema and sitemaps.
