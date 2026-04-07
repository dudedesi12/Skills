# Metadata Patterns

## Static Metadata

For pages where content does not change dynamically.

```typescript
// app/pricing/page.tsx
import type { Metadata } from "next";

export const metadata: Metadata = {
  title: "Pricing",
  description: "Simple pricing for teams of all sizes. Free tier included.",
  openGraph: {
    title: "Pricing — MyApp",
    description: "Simple pricing for teams of all sizes.",
    url: "https://myapp.com/pricing",
    images: [
      {
        url: "https://myapp.com/og/pricing.png",
        width: 1200,
        height: 630,
        alt: "MyApp Pricing Plans",
      },
    ],
  },
  twitter: {
    card: "summary_large_image",
    title: "Pricing — MyApp",
    description: "Simple pricing for teams of all sizes.",
  },
  alternates: {
    canonical: "https://myapp.com/pricing",
  },
};
```

## Layout-Level Template

Set a title template in your root layout so all child pages get a consistent suffix.

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

With this template, a child page that sets `title: "Pricing"` renders as `Pricing | MyApp`.

## Dynamic generateMetadata

For pages like blog posts, product pages, or user profiles where metadata depends on data.

```typescript
// app/blog/[slug]/page.tsx
import type { Metadata } from "next";
import { createClient } from "@/lib/supabase/server";

interface PageProps {
  params: Promise<{ slug: string }>;
}

export async function generateMetadata({ params }: PageProps): Promise<Metadata> {
  const { slug } = await params;
  const supabase = await createClient();

  const { data: post, error } = await supabase
    .from("posts")
    .select("title, excerpt, og_image, published_at, updated_at, author_name")
    .eq("slug", slug)
    .eq("published", true)
    .single();

  if (error || !post) {
    return {
      title: "Post Not Found",
      robots: { index: false },
    };
  }

  const url = `https://myapp.com/blog/${slug}`;

  return {
    title: post.title,
    description: post.excerpt,
    authors: [{ name: post.author_name }],
    openGraph: {
      title: post.title,
      description: post.excerpt,
      type: "article",
      publishedTime: post.published_at,
      modifiedTime: post.updated_at,
      url,
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
      canonical: url,
    },
  };
}
```

## Per-Route Metadata for Different Sections

### Marketing Pages

```typescript
// app/(marketing)/layout.tsx
import type { Metadata } from "next";

export const metadata: Metadata = {
  title: {
    template: "%s — MyApp",
    default: "MyApp",
  },
};
```

### Dashboard (No Indexing)

```typescript
// app/(dashboard)/layout.tsx
import type { Metadata } from "next";

export const metadata: Metadata = {
  title: {
    template: "%s — Dashboard",
    default: "Dashboard",
  },
  robots: {
    index: false,
    follow: false,
  },
};
```

### Blog Section

```typescript
// app/blog/layout.tsx
import type { Metadata } from "next";

export const metadata: Metadata = {
  title: {
    template: "%s — MyApp Blog",
    default: "Blog — MyApp",
  },
  openGraph: {
    type: "article",
  },
};
```

## Metadata for Programmatic Pages

When generating hundreds of pages, use a helper to keep metadata consistent.

```typescript
// lib/seo.ts
import type { Metadata } from "next";

interface SeoConfig {
  title: string;
  description: string;
  path: string;
  image?: string;
  noIndex?: boolean;
}

export function buildMetadata({ title, description, path, image, noIndex }: SeoConfig): Metadata {
  const url = `https://myapp.com${path}`;
  const ogImage = image || "https://myapp.com/og-default.png";

  return {
    title,
    description,
    openGraph: {
      title,
      description,
      url,
      images: [{ url: ogImage, width: 1200, height: 630, alt: title }],
    },
    twitter: {
      card: "summary_large_image",
      title,
      description,
      images: [ogImage],
    },
    alternates: { canonical: url },
    robots: noIndex ? { index: false, follow: false } : undefined,
  };
}
```

Usage in a page:

```typescript
// app/tools/[slug]/page.tsx
import { buildMetadata } from "@/lib/seo";
import { createClient } from "@/lib/supabase/server";

export async function generateMetadata({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const supabase = await createClient();
  const { data } = await supabase.from("tools").select("name, description").eq("slug", slug).single();

  if (!data) return { title: "Not Found", robots: { index: false } };

  return buildMetadata({
    title: `${data.name} — Free Online Tool`,
    description: data.description,
    path: `/tools/${slug}`,
  });
}
```

## Metadata for Search Actions

Allow Google to show a search box in search results:

```typescript
// app/layout.tsx
export const metadata: Metadata = {
  // ... other metadata
  other: {
    "google-site-verification": "your-verification-code",
  },
};
```

## Favicon and App Icons

Place these files in the `app/` directory and Next.js serves them automatically:

- `app/favicon.ico` — browser tab icon
- `app/icon.png` — modern icon (512x512)
- `app/apple-icon.png` — iOS home screen icon (180x180)
- `app/opengraph-image.png` — default OG image (1200x630)
- `app/twitter-image.png` — default Twitter card image (1200x630)
