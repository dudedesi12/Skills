---
name: i18n-localizer
description: "Use this skill whenever the user mentions i18n, internationalization, localization, l10n, translations, multi-language, multilingual, locale, language switcher, RTL, right-to-left, Arabic, Chinese, Hindi, next-intl, next-i18next, language routing, hreflang, date formatting by locale, number formatting, currency formatting, 'support multiple languages', 'translate my app', 'add language support', or ANY internationalization task — even if they don't explicitly say 'i18n'. This skill makes your app work for users worldwide."
---

# i18n Localizer

Make your Next.js immigration platform work for users in every language. This skill uses `next-intl` as the primary library because it works natively with the App Router, supports server components, and handles formatting out of the box.

The target audience is an Australian immigration platform serving applicants from China, India, Philippines, Pakistan, UK, and other countries.

## Stack Context

- **Next.js App Router** with TypeScript
- **next-intl** for translations, formatting, and locale routing
- **Tailwind CSS** for styling (including RTL utilities)
- **Supabase** for storing user locale preferences
- **Vercel** for deployment with edge middleware

## next-intl Setup

### Install

```bash
npm install next-intl
```

### Define Your Locales

```ts
// i18n/config.ts
export const locales = ["en", "zh", "hi", "tl", "ur", "ar"] as const;
export type Locale = (typeof locales)[number];

export const defaultLocale: Locale = "en";

// Human-readable names for the language switcher
export const localeNames: Record<Locale, string> = {
  en: "English",
  zh: "中文",
  hi: "हिन्दी",
  tl: "Filipino",
  ur: "اردو",
  ar: "العربية",
};

// RTL languages need special layout handling
export const rtlLocales: Locale[] = ["ar", "ur"];
```

This config is the single source of truth. Import it everywhere you need locale info.

### Plugin Config

```ts
// next.config.ts
import createNextIntlPlugin from "next-intl/plugin";

const withNextIntl = createNextIntlPlugin("./i18n/request.ts");

const nextConfig = {
  images: {
    remotePatterns: [
      { protocol: "https" as const, hostname: "*.supabase.co", pathname: "/storage/v1/object/public/**" },
    ],
  },
};

export default withNextIntl(nextConfig);
```

### Server Request Config

```ts
// i18n/request.ts
import { getRequestConfig } from "next-intl/server";
import { routing } from "./routing";

export default getRequestConfig(async ({ requestLocale }) => {
  let locale = await requestLocale;

  // Validate that the incoming locale is supported
  if (!locale || !routing.locales.includes(locale as any)) {
    locale = routing.defaultLocale;
  }

  return {
    locale,
    messages: (await import(`../messages/${locale}.json`)).default,
  };
});
```

### Routing Config

```ts
// i18n/routing.ts
import { defineRouting } from "next-intl/routing";
import { createNavigation } from "next-intl/navigation";
import { locales, defaultLocale } from "./config";

export const routing = defineRouting({
  locales,
  defaultLocale,
  localePrefix: "as-needed", // Only adds /zh/, /hi/, etc. English stays at /
});

// Drop-in replacements for Next.js navigation that handle locales
export const { Link, redirect, usePathname, useRouter } = createNavigation(routing);
```

### Middleware for Locale Detection

```ts
// middleware.ts
import createMiddleware from "next-intl/middleware";
import { routing } from "./i18n/routing";

export default createMiddleware(routing);

export const config = {
  matcher: [
    // Match all paths except static files and APIs
    "/((?!api|_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)",
  ],
};
```

This middleware reads the user's `Accept-Language` header, checks for a locale cookie, and redirects to the right URL. English users see `/about`, Chinese users see `/zh/about`.

### Layout with Providers

```tsx
// app/[locale]/layout.tsx
import { NextIntlClientProvider } from "next-intl";
import { getMessages, setRequestLocale } from "next-intl/server";
import { notFound } from "next/navigation";
import { routing } from "@/i18n/routing";
import { rtlLocales, type Locale } from "@/i18n/config";

export function generateStaticParams() {
  return routing.locales.map((locale) => ({ locale }));
}

export default async function LocaleLayout({
  children,
  params,
}: {
  children: React.ReactNode;
  params: Promise<{ locale: string }>;
}) {
  const { locale } = await params;

  // Validate locale
  if (!routing.locales.includes(locale as any)) {
    notFound();
  }

  setRequestLocale(locale);

  const messages = await getMessages();
  const dir = rtlLocales.includes(locale as Locale) ? "rtl" : "ltr";

  return (
    <html lang={locale} dir={dir}>
      <body className="bg-white text-gray-900 antialiased">
        <NextIntlClientProvider messages={messages}>
          {children}
        </NextIntlClientProvider>
      </body>
    </html>
  );
}
```

See `references/next-intl-setup.md` for the full setup including TypeScript types for translation keys.

## Translation File Structure

Organize translations as JSON files, one per locale. Use namespaces (nested objects) to group related strings.

```
messages/
  en.json
  zh.json
  hi.json
  tl.json
  ur.json
  ar.json
```

### Example Translation File

```json
// messages/en.json
{
  "common": {
    "submit": "Submit",
    "cancel": "Cancel",
    "loading": "Loading...",
    "error": "Something went wrong",
    "backToHome": "Back to home"
  },
  "nav": {
    "home": "Home",
    "visaFinder": "Visa Finder",
    "applications": "My Applications",
    "resources": "Resources",
    "contact": "Contact Us"
  },
  "hero": {
    "title": "Your Australian Immigration Journey Starts Here",
    "subtitle": "Find the right visa, check your eligibility, and apply with confidence.",
    "cta": "Check Your Eligibility"
  },
  "visaFinder": {
    "title": "Find Your Visa",
    "description": "Answer a few questions and we'll recommend the best visa for your situation.",
    "resultsCount": "We found {count, plural, =0 {no matching visas} =1 {one matching visa} other {# matching visas}}",
    "processingTime": "Processing time: {days} business days",
    "fee": "Application fee: {amount, number, ::currency/AUD}"
  },
  "application": {
    "status": {
      "draft": "Draft",
      "submitted": "Submitted",
      "inReview": "In Review",
      "approved": "Approved",
      "rejected": "Rejected"
    },
    "submittedOn": "Submitted on {date, date, long}",
    "documentsRequired": "You need to upload {count, plural, =1 {one document} other {# documents}}"
  }
}
```

```json
// messages/zh.json
{
  "common": {
    "submit": "提交",
    "cancel": "取消",
    "loading": "加载中...",
    "error": "出了点问题",
    "backToHome": "返回首页"
  },
  "nav": {
    "home": "首页",
    "visaFinder": "签证查找",
    "applications": "我的申请",
    "resources": "资源",
    "contact": "联系我们"
  },
  "hero": {
    "title": "您的澳大利亚移民之旅从这里开始",
    "subtitle": "找到合适的签证，检查您的资格，自信地申请。",
    "cta": "检查您的资格"
  }
}
```

See `references/translation-patterns.md` for pluralization, interpolation, rich text, and fallback handling.

## Locale Routing

next-intl handles URL-based locale routing. With `localePrefix: "as-needed"`:

- English (default): `/about`, `/visa-finder`, `/dashboard`
- Chinese: `/zh/about`, `/zh/visa-finder`, `/zh/dashboard`
- Hindi: `/hi/about`, `/hi/visa-finder`, `/hi/dashboard`
- Arabic: `/ar/about`, `/ar/visa-finder`, `/ar/dashboard`

### File Structure

```
app/
  [locale]/
    page.tsx              -> / or /zh or /hi
    layout.tsx            -> wraps all localized pages
    visa-finder/
      page.tsx            -> /visa-finder or /zh/visa-finder
    applications/
      page.tsx            -> /applications or /zh/applications
      [id]/
        page.tsx          -> /applications/123 or /zh/applications/123
```

### Using the Locale-Aware Link

```tsx
// Use the Link from i18n/routing.ts, NOT from next/link
import { Link } from "@/i18n/routing";

export function NavLink() {
  // This automatically adds the locale prefix
  // If user is on /zh, this links to /zh/visa-finder
  return <Link href="/visa-finder" className="text-blue-600 hover:underline">Visa Finder</Link>;
}
```

### Programmatic Navigation

```tsx
"use client";

import { useRouter } from "@/i18n/routing";

export function VisaButton() {
  const router = useRouter();

  function handleClick() {
    // Automatically navigates to the right locale path
    router.push("/visa-finder");
  }

  return (
    <button onClick={handleClick} className="rounded bg-blue-600 px-4 py-2 text-white">
      Find Your Visa
    </button>
  );
}
```

## Client Components Translation

Use the `useTranslations` hook in any client component. You must add `"use client"` at the top.

```tsx
// components/VisaCard.tsx
"use client";

import { useTranslations } from "next-intl";

export function VisaCard({ visaName, fee, processingDays }: {
  visaName: string;
  fee: number;
  processingDays: number;
}) {
  const t = useTranslations("visaFinder");

  return (
    <div className="rounded-lg border p-6 shadow-sm">
      <h3 className="text-xl font-semibold">{visaName}</h3>
      <p className="mt-2 text-gray-600">{t("processingTime", { days: processingDays })}</p>
      <p className="mt-1 font-medium">{t("fee", { amount: fee })}</p>
    </div>
  );
}
```

### Language Switcher Component

```tsx
// components/LanguageSwitcher.tsx
"use client";

import { useLocale } from "next-intl";
import { useRouter, usePathname } from "@/i18n/routing";
import { locales, localeNames, type Locale } from "@/i18n/config";

export function LanguageSwitcher() {
  const locale = useLocale();
  const router = useRouter();
  const pathname = usePathname();

  function switchLocale(newLocale: Locale) {
    router.replace(pathname, { locale: newLocale });
  }

  return (
    <select
      value={locale}
      onChange={(e) => switchLocale(e.target.value as Locale)}
      className="rounded border border-gray-300 bg-white px-3 py-1.5 text-sm"
      aria-label="Select language"
    >
      {locales.map((loc) => (
        <option key={loc} value={loc}>
          {localeNames[loc]}
        </option>
      ))}
    </select>
  );
}
```

## Server Components Translation

Server components use `getTranslations` instead of the hook. No `"use client"` needed.

```tsx
// app/[locale]/page.tsx
import { getTranslations, setRequestLocale } from "next-intl/server";

export default async function HomePage({ params }: { params: Promise<{ locale: string }> }) {
  const { locale } = await params;
  setRequestLocale(locale);

  const t = await getTranslations("hero");

  return (
    <main className="flex min-h-screen flex-col items-center justify-center p-8 text-center">
      <h1 className="text-4xl font-bold">{t("title")}</h1>
      <p className="mt-4 max-w-2xl text-lg text-gray-600">{t("subtitle")}</p>
      <a href="/visa-finder" className="mt-8 rounded-lg bg-blue-600 px-8 py-3 text-lg text-white hover:bg-blue-700">
        {t("cta")}
      </a>
    </main>
  );
}
```

### Translated Metadata for SEO

```tsx
// app/[locale]/visa-finder/page.tsx
import type { Metadata } from "next";
import { getTranslations, setRequestLocale } from "next-intl/server";

export async function generateMetadata({ params }: { params: Promise<{ locale: string }> }): Promise<Metadata> {
  const { locale } = await params;
  const t = await getTranslations({ locale, namespace: "visaFinder" });

  return {
    title: t("title"),
    description: t("description"),
  };
}

export default async function VisaFinderPage({ params }: { params: Promise<{ locale: string }> }) {
  const { locale } = await params;
  setRequestLocale(locale);
  const t = await getTranslations("visaFinder");

  return (
    <main className="mx-auto max-w-4xl p-8">
      <h1 className="text-3xl font-bold">{t("title")}</h1>
      <p className="mt-2 text-gray-600">{t("description")}</p>
    </main>
  );
}
```

## Date, Number, and Currency Formatting

Use `next-intl`'s formatting hooks or the `Intl` API directly. Both automatically respect the user's locale.

### With next-intl useFormatter Hook

```tsx
"use client";

import { useFormatter } from "next-intl";

export function ApplicationDetails({ submittedAt, fee }: { submittedAt: Date; fee: number }) {
  const format = useFormatter();

  return (
    <div className="space-y-2">
      <p>Submitted: {format.dateTime(submittedAt, { year: "numeric", month: "long", day: "numeric" })}</p>
      <p>Fee: {format.number(fee, { style: "currency", currency: "AUD" })}</p>
      <p>Relative: {format.relativeTime(submittedAt)}</p>
    </div>
  );
}
```

### Date Formatting Examples by Locale

The same date renders differently depending on locale:

```
en: 15 March 2025        (Australian English, day-first)
zh: 2025年3月15日          (year-month-day with Chinese characters)
hi: 15 मार्च 2025          (day-first with Hindi month name)
ar: ١٥ مارس ٢٠٢٥          (Arabic numerals, right-to-left)
```

### Currency Formatting

Always store amounts in cents (integers) and format at display time.

```tsx
// lib/format.ts
export function formatCurrency(amountInCents: number, currency: string, locale: string): string {
  return new Intl.NumberFormat(locale, {
    style: "currency",
    currency,
  }).format(amountInCents / 100);
}

// Usage examples:
// formatCurrency(63500, "AUD", "en-AU") -> "A$635.00"
// formatCurrency(63500, "AUD", "zh-CN") -> "A$635.00"
// formatCurrency(350000, "CNY", "zh-CN") -> "¥3,500.00"
// formatCurrency(2500000, "INR", "hi-IN") -> "₹25,000.00"
```

See `references/locale-formatting.md` for a complete reference of date, number, currency, phone, and address formats for all target markets.

## RTL Support

Arabic and Urdu are right-to-left (RTL) languages. The layout flips: navigation goes to the right, text aligns right, and margins/paddings swap sides.

### How It Works

The layout (above) sets `dir="rtl"` on the `<html>` tag based on the locale. Tailwind handles the rest with logical properties.

### Tailwind RTL Utilities

Use Tailwind's logical properties instead of `left`/`right`:

```tsx
// Instead of this (breaks in RTL):
<div className="ml-4 pl-6 text-left border-l-2">Content</div>

// Do this (works in both LTR and RTL):
<div className="ms-4 ps-6 text-start border-s-2">Content</div>
```

**Cheat sheet for swapping physical to logical:**

| Physical (avoid) | Logical (use this) | What it does |
|---|---|---|
| `ml-*` | `ms-*` | Margin at the start |
| `mr-*` | `me-*` | Margin at the end |
| `pl-*` | `ps-*` | Padding at the start |
| `pr-*` | `pe-*` | Padding at the end |
| `left-*` | `start-*` | Position from start |
| `right-*` | `end-*` | Position from end |
| `text-left` | `text-start` | Align text to start |
| `text-right` | `text-end` | Align text to end |
| `border-l-*` | `border-s-*` | Border on start side |
| `border-r-*` | `border-e-*` | Border on end side |
| `rounded-l-*` | `rounded-s-*` | Round start corners |
| `rounded-r-*` | `rounded-e-*` | Round end corners |

### RTL-Aware Component Example

```tsx
// components/StepIndicator.tsx
export function StepIndicator({ currentStep, totalSteps }: { currentStep: number; totalSteps: number }) {
  return (
    <div className="flex items-center gap-2">
      {Array.from({ length: totalSteps }, (_, i) => (
        <div key={i} className="flex items-center gap-2">
          <div
            className={`flex h-8 w-8 items-center justify-center rounded-full text-sm font-medium ${
              i + 1 <= currentStep
                ? "bg-blue-600 text-white"
                : "bg-gray-200 text-gray-500"
            }`}
          >
            {i + 1}
          </div>
          {/* Connector line between steps - uses logical border */}
          {i < totalSteps - 1 && (
            <div className={`h-0.5 w-8 ${i + 1 < currentStep ? "bg-blue-600" : "bg-gray-200"}`} />
          )}
        </div>
      ))}
    </div>
  );
}
```

### Font Loading for RTL Languages

```tsx
// app/[locale]/layout.tsx
import { Inter, Noto_Sans_Arabic, Noto_Nastaliq_Urdu } from "next/font/google";

const inter = Inter({ subsets: ["latin"], variable: "--font-sans" });
const notoArabic = Noto_Sans_Arabic({ subsets: ["arabic"], variable: "--font-arabic" });
const notoUrdu = Noto_Nastaliq_Urdu({ subsets: ["arabic"], variable: "--font-urdu" });

// In the layout, apply the right font class based on locale:
// locale === "ar" ? notoArabic.variable : locale === "ur" ? notoUrdu.variable : inter.variable
```

## Translation Workflow

### Adding a New Translation Key

1. Add the key to `messages/en.json` first (English is the source of truth)
2. Add the same key to every other locale file
3. If you skip a locale, `next-intl` falls back to the default locale (English)

### Finding Missing Translations

Create a script that compares all locale files against English:

```ts
// scripts/check-translations.ts
import fs from "fs";
import path from "path";

const messagesDir = path.join(process.cwd(), "messages");
const baseMessages = JSON.parse(fs.readFileSync(path.join(messagesDir, "en.json"), "utf-8"));

function getKeys(obj: Record<string, any>, prefix = ""): string[] {
  return Object.entries(obj).flatMap(([key, value]) => {
    const fullKey = prefix ? `${prefix}.${key}` : key;
    if (typeof value === "object" && value !== null) {
      return getKeys(value, fullKey);
    }
    return [fullKey];
  });
}

const baseKeys = getKeys(baseMessages);

const localeFiles = fs.readdirSync(messagesDir).filter((f) => f !== "en.json" && f.endsWith(".json"));

for (const file of localeFiles) {
  const locale = file.replace(".json", "");
  const messages = JSON.parse(fs.readFileSync(path.join(messagesDir, file), "utf-8"));
  const localeKeys = getKeys(messages);
  const missing = baseKeys.filter((key) => !localeKeys.includes(key));

  if (missing.length > 0) {
    console.log(`\n${locale}: ${missing.length} missing keys`);
    missing.forEach((key) => console.log(`  - ${key}`));
  } else {
    console.log(`${locale}: All keys present`);
  }
}
```

Run it with:

```bash
npx tsx scripts/check-translations.ts
```

### Storing User Locale Preference in Supabase

```sql
-- Add locale column to profiles table
alter table profiles add column preferred_locale text default 'en';
```

```tsx
// lib/locale.ts
import { createClient } from "@/lib/supabase/server";

export async function getUserLocale(): Promise<string> {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();

  if (!user) return "en";

  const { data: profile } = await supabase
    .from("profiles")
    .select("preferred_locale")
    .eq("id", user.id)
    .single();

  return profile?.preferred_locale ?? "en";
}

export async function setUserLocale(locale: string): Promise<void> {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();

  if (!user) return;

  await supabase
    .from("profiles")
    .update({ preferred_locale: locale })
    .eq("id", user.id);
}
```

## SEO for Multi-Language

### hreflang Tags

Tell search engines about all language versions of each page.

```tsx
// app/[locale]/layout.tsx
import type { Metadata } from "next";
import { locales } from "@/i18n/config";

export const metadata: Metadata = {
  alternates: {
    languages: Object.fromEntries(
      locales.map((locale) => [locale, `https://yourdomain.com.au/${locale}`])
    ),
  },
};
```

### Per-Page hreflang in generateMetadata

```tsx
export async function generateMetadata({ params }: { params: Promise<{ locale: string }> }): Promise<Metadata> {
  const { locale } = await params;
  const t = await getTranslations({ locale, namespace: "visaFinder" });

  return {
    title: t("title"),
    description: t("description"),
    alternates: {
      languages: Object.fromEntries(
        locales.map((loc) => [loc, `https://yourdomain.com.au/${loc}/visa-finder`])
      ),
    },
  };
}
```

### Locale-Specific Sitemap

```ts
// app/sitemap.ts
import type { MetadataRoute } from "next";
import { locales, defaultLocale } from "@/i18n/config";

export default function sitemap(): MetadataRoute.Sitemap {
  const baseUrl = "https://yourdomain.com.au";

  const pages = ["/", "/visa-finder", "/resources", "/contact"];

  return pages.flatMap((page) =>
    locales.map((locale) => ({
      url: `${baseUrl}${locale === defaultLocale ? "" : `/${locale}`}${page === "/" ? "" : page}`,
      lastModified: new Date(),
      changeFrequency: "weekly" as const,
      priority: page === "/" ? 1 : 0.8,
      alternates: {
        languages: Object.fromEntries(
          locales.map((loc) => [
            loc,
            `${baseUrl}${loc === defaultLocale ? "" : `/${loc}`}${page === "/" ? "" : page}`,
          ])
        ),
      },
    }))
  );
}
```

### Translated Open Graph Images

```tsx
// app/[locale]/opengraph-image.tsx
import { ImageResponse } from "next/og";
import { getTranslations } from "next-intl/server";

export const size = { width: 1200, height: 630 };
export const contentType = "image/png";

export default async function OGImage({ params }: { params: Promise<{ locale: string }> }) {
  const { locale } = await params;
  const t = await getTranslations({ locale, namespace: "hero" });

  return new ImageResponse(
    (
      <div style={{ display: "flex", flexDirection: "column", alignItems: "center", justifyContent: "center", width: "100%", height: "100%", backgroundColor: "#1e40af", color: "white", fontSize: 48, fontWeight: "bold", padding: 60 }}>
        {t("title")}
      </div>
    ),
    { ...size }
  );
}
```

## Rules

1. **English is the source of truth.** Always add new keys to `messages/en.json` first. Other locale files mirror its structure.

2. **Use next-intl's `Link`, `useRouter`, `usePathname`, and `redirect`** from `@/i18n/routing` instead of the Next.js equivalents. The next-intl versions handle locale prefixes automatically.

3. **Never hardcode user-facing strings.** Every string a user sees must come from a translation file. This includes error messages, button labels, placeholders, alt text, and ARIA labels.

4. **Use logical properties in Tailwind** (`ms-*`, `me-*`, `ps-*`, `pe-*`, `text-start`, `text-end`) instead of physical properties (`ml-*`, `mr-*`, `pl-*`, `pr-*`, `text-left`, `text-right`). This ensures RTL languages render correctly.

5. **Store amounts in cents, format at display time.** Never store pre-formatted currency strings. Use `Intl.NumberFormat` or `next-intl`'s `useFormatter` with the user's locale and the correct currency code.

6. **Always call `setRequestLocale(locale)` at the top of every server component page and layout** inside the `[locale]` segment. This enables static rendering for all locales.

7. **Keep translation keys descriptive and nested by feature.** Use `visaFinder.resultsCount` not `vf_rc`. Namespaces like `common`, `nav`, `errors` keep shared strings organized.

8. **Run the missing-translations check before every deploy.** Add `npx tsx scripts/check-translations.ts` to your CI pipeline. Missing translations silently fall back to English, which looks broken to non-English users.

9. **Set `lang` and `dir` attributes on `<html>`** in the locale layout. Screen readers and browsers use `lang` for pronunciation and `dir` for layout direction.

10. **Test RTL layouts manually.** Automated tests catch missing translations but not layout bugs. Switch to Arabic or Urdu and visually verify every page. Pay attention to icons with directional meaning (arrows, chevrons) -- they should flip in RTL.
