# PWA Configuration Reference

Complete PWA configuration: manifest.json, serwist setup, icon sizes, cache strategies, and offline page.

## Full manifest.json

```json
// public/manifest.json
{
  "name": "Your App Name",
  "short_name": "AppName",
  "description": "A brief description of what your app does",
  "start_url": "/",
  "scope": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#3b82f6",
  "orientation": "portrait-primary",
  "dir": "ltr",
  "lang": "en",
  "categories": ["productivity", "utilities"],
  "icons": [
    {
      "src": "/icons/icon-72.png",
      "sizes": "72x72",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-96.png",
      "sizes": "96x96",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-128.png",
      "sizes": "128x128",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-144.png",
      "sizes": "144x144",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-152.png",
      "sizes": "152x152",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-384.png",
      "sizes": "384x384",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-512.png",
      "sizes": "512x512",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-maskable-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "maskable"
    },
    {
      "src": "/icons/icon-maskable-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "maskable"
    }
  ],
  "screenshots": [
    {
      "src": "/screenshots/desktop-1.png",
      "sizes": "1280x720",
      "type": "image/png",
      "form_factor": "wide",
      "label": "Dashboard view"
    },
    {
      "src": "/screenshots/mobile-1.png",
      "sizes": "750x1334",
      "type": "image/png",
      "form_factor": "narrow",
      "label": "Mobile home screen"
    }
  ],
  "shortcuts": [
    {
      "name": "Dashboard",
      "short_name": "Dashboard",
      "description": "Go to your dashboard",
      "url": "/dashboard",
      "icons": [{ "src": "/icons/shortcut-dashboard.png", "sizes": "96x96" }]
    },
    {
      "name": "New Item",
      "short_name": "New",
      "description": "Create a new item",
      "url": "/new",
      "icons": [{ "src": "/icons/shortcut-new.png", "sizes": "96x96" }]
    }
  ]
}
```

## Serwist Full Setup

### next.config.ts

```ts
// next.config.ts
import withSerwistInit from "@serwist/next";

const withSerwist = withSerwistInit({
  swSrc: "app/sw.ts",
  swDest: "public/sw.js",
  reloadOnOnline: true,
  disable: process.env.NODE_ENV === "development",
});

const nextConfig = {
  reactStrictMode: true,
};

export default withSerwist(nextConfig);
```

### Service Worker with Custom Caching

```ts
// app/sw.ts
import type { PrecacheEntry, SerwistGlobalConfig } from "serwist";
import { CacheFirst, NetworkFirst, StaleWhileRevalidate, Serwist } from "serwist";

declare global {
  interface WorkerGlobalScope extends SerwistGlobalConfig {
    __SW_MANIFEST: (PrecacheEntry | string)[] | undefined;
  }
}

declare const self: ServiceWorkerGlobalScope;

const serwist = new Serwist({
  precacheEntries: self.__SW_MANIFEST,
  skipWaiting: true,
  clientsClaim: true,
  navigationPreload: true,
  runtimeCaching: [
    // Static assets — cache forever
    {
      urlPattern: /\.(?:js|css|woff2?)$/i,
      handler: new CacheFirst({
        cacheName: "static-assets",
        plugins: [
          {
            cacheWillUpdate: async ({ response }) => {
              return response && response.status === 200 ? response : null;
            },
          },
        ],
      }),
    },
    // Images — cache for 30 days
    {
      urlPattern: /\.(?:png|jpg|jpeg|svg|gif|webp|avif|ico)$/i,
      handler: new CacheFirst({
        cacheName: "images",
        plugins: [
          {
            cacheWillUpdate: async ({ response }) => {
              return response && response.status === 200 ? response : null;
            },
          },
        ],
      }),
    },
    // API calls — network first, fall back to cache
    {
      urlPattern: /^\/api\//,
      handler: new NetworkFirst({
        cacheName: "api-cache",
        networkTimeoutSeconds: 10,
      }),
    },
    // Pages — stale while revalidate
    {
      urlPattern: ({ request }) => request.destination === "document",
      handler: new StaleWhileRevalidate({
        cacheName: "pages",
      }),
    },
    // Google Fonts
    {
      urlPattern: /^https:\/\/fonts\.googleapis\.com\/.*/i,
      handler: new StaleWhileRevalidate({
        cacheName: "google-fonts-stylesheets",
      }),
    },
    {
      urlPattern: /^https:\/\/fonts\.gstatic\.com\/.*/i,
      handler: new CacheFirst({
        cacheName: "google-fonts-webfonts",
      }),
    },
  ],
  fallbacks: {
    entries: [
      {
        url: "/offline",
        matcher({ request }) {
          return request.destination === "document";
        },
      },
    ],
  },
});

serwist.addEventListeners();
```

### tsconfig additions

Make sure your `tsconfig.json` includes the service worker types:

```json
// tsconfig.json — add to compilerOptions.lib
{
  "compilerOptions": {
    "lib": ["dom", "dom.iterable", "esnext", "webworker"]
  }
}
```

## Icon Sizes Checklist

| File | Size | Purpose | Required |
|------|------|---------|----------|
| `favicon.ico` | 32x32 | Browser tab | Yes |
| `icon-72.png` | 72x72 | Old Android | No |
| `icon-96.png` | 96x96 | Android shortcut | No |
| `icon-128.png` | 128x128 | Chrome Web Store | No |
| `icon-144.png` | 144x144 | Windows tiles | No |
| `icon-152.png` | 152x152 | iPad | No |
| `icon-192.png` | 192x192 | Android home screen | Yes |
| `icon-384.png` | 384x384 | Android splash | No |
| `icon-512.png` | 512x512 | Android splash / install | Yes |
| `apple-touch-icon.png` | 180x180 | iOS home screen | Yes |
| `icon-maskable-192.png` | 192x192 | Android adaptive | Yes |
| `icon-maskable-512.png` | 512x512 | Android adaptive | Yes |

### Maskable Icon Safe Zone

Maskable icons can be cropped into circles, squares, or rounded shapes. Your logo must fit inside the inner 80% "safe zone." The outer 20% should be your brand color or a simple background.

## Offline Fallback Page

```tsx
// app/offline/page.tsx
export const metadata = {
  title: "Offline",
};

export default function OfflinePage() {
  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-50 p-4">
      <div className="text-center max-w-md">
        <div className="w-16 h-16 mx-auto mb-6 bg-gray-200 rounded-full flex items-center justify-center">
          <svg
            className="w-8 h-8 text-gray-400"
            fill="none"
            viewBox="0 0 24 24"
            stroke="currentColor"
          >
            <path
              strokeLinecap="round"
              strokeLinejoin="round"
              strokeWidth={2}
              d="M18.364 5.636a9 9 0 010 12.728M5.636 18.364a9 9 0 010-12.728"
            />
          </svg>
        </div>
        <h1 className="text-2xl font-bold text-gray-900">You are offline</h1>
        <p className="text-gray-600 mt-2 mb-6">
          Check your internet connection and try again. Some pages you have visited before
          may still be available.
        </p>
        <div className="space-y-3">
          <button
            onClick={() => window.location.reload()}
            className="w-full px-6 py-3 bg-blue-600 text-white rounded-lg hover:bg-blue-700 font-medium"
          >
            Retry Connection
          </button>
          <button
            onClick={() => window.history.back()}
            className="w-full px-6 py-3 border border-gray-300 rounded-lg hover:bg-gray-50 font-medium"
          >
            Go Back
          </button>
        </div>
      </div>
    </div>
  );
}
```

## iOS Splash Screen Setup

iOS requires specific `apple-touch-startup-image` links for splash screens.

```tsx
// app/layout.tsx — add to <head>
<head>
  <link rel="apple-touch-icon" href="/icons/apple-touch-icon.png" />
  <meta name="apple-mobile-web-app-capable" content="yes" />
  <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
  <meta name="apple-mobile-web-app-title" content="MyApp" />

  {/* iPhone 15 Pro Max */}
  <link
    rel="apple-touch-startup-image"
    media="screen and (device-width: 430px) and (device-height: 932px) and (-webkit-device-pixel-ratio: 3)"
    href="/splash/iphone-15-pro-max.png"
  />
  {/* iPhone 15 Pro */}
  <link
    rel="apple-touch-startup-image"
    media="screen and (device-width: 393px) and (device-height: 852px) and (-webkit-device-pixel-ratio: 3)"
    href="/splash/iphone-15-pro.png"
  />
  {/* iPhone SE */}
  <link
    rel="apple-touch-startup-image"
    media="screen and (device-width: 375px) and (device-height: 667px) and (-webkit-device-pixel-ratio: 2)"
    href="/splash/iphone-se.png"
  />
</head>
```

## Network Status Hook

```tsx
// hooks/use-network-status.ts
"use client";

import { useState, useEffect } from "react";

export function useNetworkStatus() {
  const [isOnline, setIsOnline] = useState(true);

  useEffect(() => {
    setIsOnline(navigator.onLine);

    function handleOnline() {
      setIsOnline(true);
    }
    function handleOffline() {
      setIsOnline(false);
    }

    window.addEventListener("online", handleOnline);
    window.addEventListener("offline", handleOffline);

    return () => {
      window.removeEventListener("online", handleOnline);
      window.removeEventListener("offline", handleOffline);
    };
  }, []);

  return { isOnline };
}
```

### Offline Banner Component

```tsx
// components/offline-banner.tsx
"use client";

import { useNetworkStatus } from "@/hooks/use-network-status";

export function OfflineBanner() {
  const { isOnline } = useNetworkStatus();

  if (isOnline) return null;

  return (
    <div className="fixed top-0 left-0 right-0 bg-yellow-500 text-yellow-900 text-center text-sm py-2 z-50 font-medium">
      You are offline. Some features may be unavailable.
    </div>
  );
}
```
