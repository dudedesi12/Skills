# next-intl Setup

Complete configuration guide for adding next-intl to a Next.js App Router project. Follow these steps in order.

## Installation

```bash
npm install next-intl
```

Your `package.json` should include:

```json
{
  "dependencies": {
    "next": "^15.0.0",
    "next-intl": "^3.22.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  }
}
```

## Step 1: Define Your Locales

This is the single source of truth for every locale your app supports. Import this file anywhere you need locale info.

```ts
// i18n/config.ts
export const locales = ["en", "zh", "hi", "tl", "ur", "ar"] as const;
export type Locale = (typeof locales)[number];

export const defaultLocale: Locale = "en";

// Human-readable names for the language switcher UI
export const localeNames: Record<Locale, string> = {
  en: "English",
  zh: "中文",
  hi: "हिन्दी",
  tl: "Filipino",
  ur: "اردو",
  ar: "العربية",
};

// These languages render right-to-left
export const rtlLocales: Locale[] = ["ar", "ur"];

// Map locale codes to full BCP 47 tags for Intl APIs
export const localeToTag: Record<Locale, string> = {
  en: "en-AU",    // Australian English
  zh: "zh-CN",    // Simplified Chinese
  hi: "hi-IN",    // Hindi (India)
  tl: "tl-PH",    // Filipino (Philippines)
  ur: "ur-PK",    // Urdu (Pakistan)
  ar: "ar-SA",    // Arabic (Saudi Arabia)
};
```

## Step 2: Configure Routing

This file defines how next-intl maps URLs to locales and creates locale-aware navigation helpers.

```ts
// i18n/routing.ts
import { defineRouting } from "next-intl/routing";
import { createNavigation } from "next-intl/navigation";
import { locales, defaultLocale } from "./config";

export const routing = defineRouting({
  locales,
  defaultLocale,

  // "as-needed" means the default locale (en) has no prefix:
  //   /about       -> English
  //   /zh/about    -> Chinese
  //   /hi/about    -> Hindi
  localePrefix: "as-needed",
});

// These are drop-in replacements for Next.js navigation.
// They automatically handle locale prefixes in URLs.
export const { Link, redirect, usePathname, useRouter } = createNavigation(routing);
```

**Why `localePrefix: "as-needed"`?** Your Australian users (the majority) see clean URLs without `/en/`. Non-English users get a locale prefix. Other options:
- `"always"` -- every URL has a prefix, including `/en/about`
- `"never"` -- no prefixes at all, locale comes from cookies/headers only

## Step 3: Server Request Config

This file tells next-intl how to load messages on the server for each request.

```ts
// i18n/request.ts
import { getRequestConfig } from "next-intl/server";
import { routing } from "./routing";

export default getRequestConfig(async ({ requestLocale }) => {
  // The locale comes from the URL segment matched by middleware
  let locale = await requestLocale;

  // Fallback to default if the locale is missing or unsupported
  if (!locale || !routing.locales.includes(locale as any)) {
    locale = routing.defaultLocale;
  }

  return {
    locale,
    messages: (await import(`../messages/${locale}.json`)).default,
  };
});
```

**How message loading works:** Each request loads only the messages for that locale. A Chinese user loading `/zh/visa-finder` only downloads `zh.json`, not all locale files.

## Step 4: Configure next.config.ts

The `createNextIntlPlugin` wraps your Next.js config with the i18n compiler plugin.

```ts
// next.config.ts
import createNextIntlPlugin from "next-intl/plugin";

// Point to your request config file
const withNextIntl = createNextIntlPlugin("./i18n/request.ts");

const nextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: "https" as const,
        hostname: "*.supabase.co",
        pathname: "/storage/v1/object/public/**",
      },
    ],
  },
};

export default withNextIntl(nextConfig);
```

## Step 5: Middleware for Locale Detection

The middleware runs before every request. It detects the user's preferred language and redirects to the correct locale URL.

```ts
// middleware.ts
import createMiddleware from "next-intl/middleware";
import { routing } from "./i18n/routing";

export default createMiddleware(routing);

export const config = {
  matcher: [
    // Match all routes except Next.js internals and static files
    "/((?!api|_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)",
  ],
};
```

**What the middleware does:**
1. Checks for a `NEXT_LOCALE` cookie (set when user switches language)
2. Falls back to the `Accept-Language` header from the browser
3. Falls back to the default locale (English)
4. Redirects if needed (e.g., a Chinese browser hitting `/about` gets sent to `/zh/about`)

### Combining with Supabase Auth Middleware

If you already have auth middleware, combine them:

```ts
// middleware.ts
import createMiddleware from "next-intl/middleware";
import { createServerClient } from "@supabase/ssr";
import { NextResponse, type NextRequest } from "next/server";
import { routing } from "./i18n/routing";

const intlMiddleware = createMiddleware(routing);

export async function middleware(request: NextRequest) {
  // Step 1: Run i18n middleware to handle locale detection
  const intlResponse = intlMiddleware(request);

  // Step 2: Create Supabase client with cookies from the response
  const response = intlResponse || NextResponse.next({ request });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll();
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value, options }) => {
            response.cookies.set(name, value, options);
          });
        },
      },
    }
  );

  // Step 3: Refresh the session (always call getUser, not getSession)
  const { data: { user } } = await supabase.auth.getUser();

  // Step 4: Protect routes that need auth
  const pathname = request.nextUrl.pathname;
  const isProtectedRoute = ["/dashboard", "/applications", "/settings"].some(
    (path) => pathname.includes(path)
  );

  if (!user && isProtectedRoute) {
    const locale = pathname.split("/")[1];
    const isLocalePrefix = routing.locales.includes(locale as any);
    const loginPath = isLocalePrefix ? `/${locale}/login` : "/login";
    return NextResponse.redirect(new URL(loginPath, request.url));
  }

  return response;
}

export const config = {
  matcher: [
    "/((?!api|_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)",
  ],
};
```

## Step 6: Layout with Providers

The locale layout wraps all pages with the translation provider and sets the HTML direction.

```tsx
// app/[locale]/layout.tsx
import { NextIntlClientProvider } from "next-intl";
import { getMessages, setRequestLocale } from "next-intl/server";
import { notFound } from "next/navigation";
import { routing } from "@/i18n/routing";
import { rtlLocales, type Locale } from "@/i18n/config";
import type { Metadata } from "next";

export function generateStaticParams() {
  return routing.locales.map((locale) => ({ locale }));
}

export async function generateMetadata({
  params,
}: {
  params: Promise<{ locale: string }>;
}): Promise<Metadata> {
  const { locale } = await params;
  // You can load locale-specific metadata here
  return {
    title: {
      template: "%s | Immigration Platform",
      default: "Immigration Platform",
    },
  };
}

export default async function LocaleLayout({
  children,
  params,
}: {
  children: React.ReactNode;
  params: Promise<{ locale: string }>;
}) {
  const { locale } = await params;

  // Show 404 for unsupported locales
  if (!routing.locales.includes(locale as any)) {
    notFound();
  }

  // Required for static rendering
  setRequestLocale(locale);

  // Load all messages for this locale
  const messages = await getMessages();

  // Determine text direction
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

**Key details:**
- `generateStaticParams` tells Next.js to pre-render pages for every locale at build time
- `setRequestLocale(locale)` enables static rendering -- call it at the top of every page and layout
- `NextIntlClientProvider` passes messages to all client components so `useTranslations` works
- `dir="rtl"` on `<html>` makes Arabic and Urdu render right-to-left

## Step 7: Create Translation Files

```
messages/
  en.json       <- English (source of truth)
  zh.json       <- Simplified Chinese
  hi.json       <- Hindi
  tl.json       <- Filipino
  ur.json       <- Urdu
  ar.json       <- Arabic
```

Minimal starter files:

```json
// messages/en.json
{
  "common": {
    "submit": "Submit",
    "cancel": "Cancel",
    "loading": "Loading...",
    "back": "Back",
    "next": "Next",
    "save": "Save",
    "delete": "Delete",
    "confirm": "Confirm",
    "error": "Something went wrong. Please try again.",
    "notFound": "Page not found"
  },
  "nav": {
    "home": "Home",
    "visaFinder": "Visa Finder",
    "applications": "My Applications",
    "resources": "Resources",
    "contact": "Contact Us",
    "login": "Log In",
    "signup": "Sign Up"
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
    "back": "返回",
    "next": "下一步",
    "save": "保存",
    "delete": "删除",
    "confirm": "确认",
    "error": "出了点问题。请重试。",
    "notFound": "页面未找到"
  },
  "nav": {
    "home": "首页",
    "visaFinder": "签证查找",
    "applications": "我的申请",
    "resources": "资源",
    "contact": "联系我们",
    "login": "登录",
    "signup": "注册"
  }
}
```

## Step 8: TypeScript Types for Translation Keys

next-intl can provide autocomplete for your translation keys. Create a type declaration file:

```ts
// types/next-intl.d.ts
import en from "../messages/en.json";

type Messages = typeof en;

declare module "next-intl" {
  interface AppConfig {
    Messages: Messages;
  }
}
```

Now when you type `t("common.`)`, your editor will autocomplete with all available keys from `en.json`. If you use a key that does not exist, TypeScript will show an error.

**Make sure** `tsconfig.json` includes the type declaration:

```json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["next-env.d.ts", "types/**/*.d.ts", "**/*.ts", "**/*.tsx"]
}
```

## File Structure Summary

After setup, your project structure should look like this:

```
project-root/
  i18n/
    config.ts          <- Locale list, names, RTL config
    routing.ts         <- Routing config + navigation exports
    request.ts         <- Server message loading
  messages/
    en.json            <- English translations
    zh.json            <- Chinese translations
    hi.json            <- Hindi translations
    tl.json            <- Filipino translations
    ur.json            <- Urdu translations
    ar.json            <- Arabic translations
  app/
    [locale]/
      layout.tsx       <- Locale layout with providers
      page.tsx         <- Home page
      visa-finder/
        page.tsx
      applications/
        page.tsx
        [id]/
          page.tsx
  middleware.ts        <- Locale detection + auth
  next.config.ts       <- With createNextIntlPlugin
  types/
    next-intl.d.ts     <- Type-safe translation keys
```

## Troubleshooting

### "Unable to find next-intl locale"

You forgot to call `setRequestLocale(locale)` in a server component page or layout. Add it as the first line after extracting locale from params.

### Translations not updating in development

next-intl caches message imports. Restart the dev server after editing JSON files, or use dynamic imports with cache busting in development.

### Middleware redirect loop

Check that your `matcher` in `middleware.ts` excludes static files and API routes. The pattern shown above handles this.

### Client component shows English instead of correct locale

Make sure the component is inside the `[locale]/layout.tsx` that wraps with `NextIntlClientProvider`. Components outside this layout do not have access to messages.

### TypeScript errors on translation keys

Rebuild your types after adding new keys to `en.json`. The type declaration reads from the JSON file, so changes are picked up on next compile.
