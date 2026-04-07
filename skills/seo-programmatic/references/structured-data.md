# Structured Data (JSON-LD)

## JSON-LD Helper Component

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

## FAQ Schema

Shows expandable question/answer pairs directly in Google search results.

```typescript
// Usage in a page
import { JsonLd } from "@/components/JsonLd";

const faqs = [
  {
    question: "How do I get started?",
    answer: "Sign up for a free account at myapp.com/signup. No credit card required.",
  },
  {
    question: "What payment methods do you accept?",
    answer: "We accept all major credit cards, PayPal, and bank transfers.",
  },
  {
    question: "Can I cancel anytime?",
    answer: "Yes, you can cancel your subscription at any time from your account settings.",
  },
];

export default function FaqPage() {
  return (
    <main>
      <JsonLd
        data={{
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
        }}
      />
      <h1>FAQ</h1>
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

## Article Schema

For blog posts and articles.

```typescript
<JsonLd
  data={{
    "@context": "https://schema.org",
    "@type": "Article",
    headline: "How to Build a SaaS with Next.js",
    description: "A step-by-step guide to building a SaaS application.",
    url: "https://myapp.com/blog/build-saas-nextjs",
    image: "https://myapp.com/blog/build-saas-nextjs/og.png",
    datePublished: "2025-01-15T08:00:00Z",
    dateModified: "2025-02-01T10:00:00Z",
    author: {
      "@type": "Person",
      name: "Jane Smith",
      url: "https://myapp.com/authors/jane-smith",
    },
    publisher: {
      "@type": "Organization",
      name: "MyApp",
      logo: {
        "@type": "ImageObject",
        url: "https://myapp.com/logo.png",
        width: 200,
        height: 200,
      },
    },
    mainEntityOfPage: {
      "@type": "WebPage",
      "@id": "https://myapp.com/blog/build-saas-nextjs",
    },
  }}
/>
```

## HowTo Schema

Shows step-by-step instructions in Google search results.

```typescript
<JsonLd
  data={{
    "@context": "https://schema.org",
    "@type": "HowTo",
    name: "How to Deploy a Next.js App to Vercel",
    description: "Deploy your Next.js application to Vercel in 3 simple steps.",
    totalTime: "PT5M",
    estimatedCost: {
      "@type": "MonetaryAmount",
      currency: "USD",
      value: "0",
    },
    step: [
      {
        "@type": "HowToStep",
        position: 1,
        name: "Connect your repository",
        text: "Go to vercel.com and click 'Import Project'. Connect your GitHub repository.",
        url: "https://myapp.com/guides/deploy-vercel#step-1",
      },
      {
        "@type": "HowToStep",
        position: 2,
        name: "Configure settings",
        text: "Select the framework as Next.js and add your environment variables.",
        url: "https://myapp.com/guides/deploy-vercel#step-2",
      },
      {
        "@type": "HowToStep",
        position: 3,
        name: "Deploy",
        text: "Click 'Deploy' and wait for the build to complete. Your site is now live.",
        url: "https://myapp.com/guides/deploy-vercel#step-3",
      },
    ],
  }}
/>
```

## Product Schema

For product pages or tool listings.

```typescript
<JsonLd
  data={{
    "@context": "https://schema.org",
    "@type": "Product",
    name: "MyApp Pro Plan",
    description: "Full-featured plan for growing teams.",
    url: "https://myapp.com/pricing",
    brand: {
      "@type": "Brand",
      name: "MyApp",
    },
    offers: {
      "@type": "Offer",
      price: "19.00",
      priceCurrency: "USD",
      availability: "https://schema.org/InStock",
      url: "https://myapp.com/pricing",
      priceValidUntil: "2026-12-31",
    },
    aggregateRating: {
      "@type": "AggregateRating",
      ratingValue: "4.8",
      reviewCount: "142",
      bestRating: "5",
      worstRating: "1",
    },
  }}
/>
```

## BreadcrumbList Schema

Shows breadcrumb trail in search results.

```typescript
<JsonLd
  data={{
    "@context": "https://schema.org",
    "@type": "BreadcrumbList",
    itemListElement: [
      {
        "@type": "ListItem",
        position: 1,
        name: "Home",
        item: "https://myapp.com",
      },
      {
        "@type": "ListItem",
        position: 2,
        name: "Blog",
        item: "https://myapp.com/blog",
      },
      {
        "@type": "ListItem",
        position: 3,
        name: "How to Build a SaaS",
        item: "https://myapp.com/blog/build-saas-nextjs",
      },
    ],
  }}
/>
```

## Organization Schema

Add to your homepage for brand recognition in search results.

```typescript
// app/page.tsx
<JsonLd
  data={{
    "@context": "https://schema.org",
    "@type": "Organization",
    name: "MyApp",
    url: "https://myapp.com",
    logo: "https://myapp.com/logo.png",
    description: "The easiest way to build and ship web applications.",
    foundingDate: "2024",
    sameAs: [
      "https://twitter.com/myapp",
      "https://github.com/myapp",
      "https://linkedin.com/company/myapp",
    ],
    contactPoint: {
      "@type": "ContactPoint",
      email: "support@myapp.com",
      contactType: "customer support",
      availableLanguage: "English",
    },
  }}
/>
```

## Dynamic Schema from Supabase

Generate structured data from your database.

```typescript
// app/blog/[slug]/page.tsx
import { createClient } from "@/lib/supabase/server";
import { JsonLd } from "@/components/JsonLd";
import { notFound } from "next/navigation";

export default async function BlogPost({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;
  const supabase = await createClient();

  const { data: post, error } = await supabase
    .from("posts")
    .select("title, excerpt, content_html, slug, og_image, published_at, updated_at, author_name")
    .eq("slug", slug)
    .eq("published", true)
    .single();

  if (error || !post) notFound();

  const articleSchema = {
    "@context": "https://schema.org",
    "@type": "Article",
    headline: post.title,
    description: post.excerpt,
    url: `https://myapp.com/blog/${post.slug}`,
    image: post.og_image || "https://myapp.com/og-default.png",
    datePublished: post.published_at,
    dateModified: post.updated_at,
    author: {
      "@type": "Person",
      name: post.author_name,
    },
    publisher: {
      "@type": "Organization",
      name: "MyApp",
      logo: { "@type": "ImageObject", url: "https://myapp.com/logo.png" },
    },
  };

  const breadcrumbSchema = {
    "@context": "https://schema.org",
    "@type": "BreadcrumbList",
    itemListElement: [
      { "@type": "ListItem", position: 1, name: "Home", item: "https://myapp.com" },
      { "@type": "ListItem", position: 2, name: "Blog", item: "https://myapp.com/blog" },
      { "@type": "ListItem", position: 3, name: post.title, item: `https://myapp.com/blog/${post.slug}` },
    ],
  };

  return (
    <main>
      <JsonLd data={articleSchema} />
      <JsonLd data={breadcrumbSchema} />
      <h1>{post.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: post.content_html }} />
    </main>
  );
}
```

## Testing Structured Data

1. Use Google's Rich Results Test: https://search.google.com/test/rich-results
2. Paste your page URL or HTML
3. Check for errors and warnings
4. Preview how your page appears in search results
