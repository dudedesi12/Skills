---
name: mobile-pwa
description: "Use this skill whenever the user mentions PWA, progressive web app, mobile app, app-like, offline, service worker, push notifications, web manifest, install prompt, 'add to home screen', mobile experience, iOS app, Android app, 'works offline', app icon, splash screen, 'native feel', background sync, cache strategy, or ANY mobile/app-like experience task — even if they don't explicitly say 'PWA'. This skill gives your web app a native app experience without the App Store."
---

# Mobile PWA

This skill covers everything you need to turn your Next.js app into a Progressive Web App. Users can install it on their phone, use it offline, and receive push notifications — all without going through the App Store.

## Next.js PWA Configuration with Serwist

Serwist is the modern replacement for next-pwa. It gives you a service worker with zero hassle.

### Installation

```bash
npm install @serwist/next
npm install -D serwist
```

### Next.js Config

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
  // Your existing Next.js config
};

export default withSerwist(nextConfig);
```

### Service Worker File

```ts
// app/sw.ts
import { defaultCache } from "@serwist/next/worker";
import type { PrecacheEntry, SerwistGlobalConfig } from "serwist";
import { Serwist } from "serwist";

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
  runtimeCaching: defaultCache,
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

## Web Manifest

The manifest tells the browser how your app should look and behave when installed.

```json
// public/manifest.json
{
  "name": "My App",
  "short_name": "MyApp",
  "description": "A description of your app",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#3b82f6",
  "orientation": "portrait-primary",
  "icons": [
    {
      "src": "/icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-512.png",
      "sizes": "512x512",
      "type": "image/png"
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
      "src": "/screenshots/mobile.png",
      "sizes": "750x1334",
      "type": "image/png",
      "form_factor": "narrow"
    }
  ]
}
```

### Link the Manifest in Your Layout

```tsx
// app/layout.tsx
import type { Metadata, Viewport } from "next";

export const metadata: Metadata = {
  title: "My App",
  description: "A description of your app",
  manifest: "/manifest.json",
  appleWebApp: {
    capable: true,
    statusBarStyle: "default",
    title: "My App",
  },
};

export const viewport: Viewport = {
  themeColor: "#3b82f6",
  width: "device-width",
  initialScale: 1,
  maximumScale: 1,
  userScalable: false,
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <head>
        <link rel="apple-touch-icon" href="/icons/apple-touch-icon.png" />
        <meta name="apple-mobile-web-app-capable" content="yes" />
        <meta name="apple-mobile-web-app-status-bar-style" content="default" />
      </head>
      <body>{children}</body>
    </html>
  );
}
```

## Service Worker Strategies

Different types of content need different caching strategies.

### Cache-First (Static Assets)

Good for images, fonts, and CSS that rarely change. The browser loads from cache instantly and only checks the network if the cache is empty.

### Network-First (API Data)

Good for API responses and dynamic content. The browser tries the network first and falls back to cache if offline.

### Stale-While-Revalidate (Semi-Dynamic)

Good for content that changes sometimes. Shows cached version immediately, then updates the cache in the background.

### Custom Runtime Caching Config

```ts
// lib/cache-config.ts
import type { RuntimeCaching } from "serwist";

export const runtimeCaching: RuntimeCaching[] = [
  // Cache images for 30 days
  {
    urlPattern: /\.(?:png|jpg|jpeg|svg|gif|webp|avif)$/i,
    handler: "CacheFirst",
    options: {
      cacheName: "images",
      expiration: {
        maxEntries: 100,
        maxAgeSeconds: 30 * 24 * 60 * 60, // 30 days
      },
    },
  },
  // Cache API responses with network-first
  {
    urlPattern: /^\/api\//,
    handler: "NetworkFirst",
    options: {
      cacheName: "api-responses",
      expiration: {
        maxEntries: 50,
        maxAgeSeconds: 5 * 60, // 5 minutes
      },
      networkTimeoutSeconds: 10,
    },
  },
  // Cache fonts forever (they never change)
  {
    urlPattern: /\.(?:woff|woff2|ttf|otf|eot)$/i,
    handler: "CacheFirst",
    options: {
      cacheName: "fonts",
      expiration: {
        maxEntries: 20,
        maxAgeSeconds: 365 * 24 * 60 * 60, // 1 year
      },
    },
  },
  // Stale-while-revalidate for pages
  {
    urlPattern: ({ request }) => request.destination === "document",
    handler: "StaleWhileRevalidate",
    options: {
      cacheName: "pages",
      expiration: {
        maxEntries: 50,
        maxAgeSeconds: 24 * 60 * 60, // 1 day
      },
    },
  },
];
```

## Offline Fallback Page

When a user is offline and tries to visit a page that is not cached, show them a friendly offline page.

```tsx
// app/offline/page.tsx
export default function OfflinePage() {
  return (
    <div className="min-h-screen flex items-center justify-center p-4 bg-gray-50">
      <div className="text-center max-w-md">
        <div className="text-6xl mb-4">📡</div>
        <h1 className="text-2xl font-bold mb-2">You are offline</h1>
        <p className="text-gray-600 mb-6">
          It looks like you have lost your internet connection.
          The page you are trying to visit is not available offline.
        </p>
        <button
          onClick={() => window.location.reload()}
          className="px-6 py-3 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition-colors"
        >
          Try Again
        </button>
      </div>
    </div>
  );
}
```

## Push Notifications

### VAPID Key Generation

First, generate your VAPID keys. Run this once.

```bash
npx web-push generate-vapid-keys
```

Store the keys in your `.env.local`:

```bash
# .env.local
NEXT_PUBLIC_VAPID_PUBLIC_KEY=your_public_key_here
VAPID_PRIVATE_KEY=your_private_key_here
VAPID_EMAIL=mailto:you@example.com
```

### Supabase Table for Subscriptions

```sql
-- supabase/migrations/create_push_subscriptions.sql
CREATE TABLE push_subscriptions (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  endpoint TEXT NOT NULL UNIQUE,
  p256dh TEXT NOT NULL,
  auth TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);

ALTER TABLE push_subscriptions ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users manage own subscriptions"
  ON push_subscriptions FOR ALL
  USING (auth.uid() = user_id);
```

### Subscribe to Push Notifications (Client)

```tsx
// lib/push-notifications.ts
export async function subscribeToPush(): Promise<PushSubscription | null> {
  try {
    if (!("serviceWorker" in navigator) || !("PushManager" in window)) {
      console.warn("Push notifications not supported");
      return null;
    }

    const permission = await Notification.requestPermission();
    if (permission !== "granted") {
      console.warn("Notification permission denied");
      return null;
    }

    const registration = await navigator.serviceWorker.ready;

    const subscription = await registration.pushManager.subscribe({
      userVisibleOnly: true,
      applicationServerKey: urlBase64ToUint8Array(
        process.env.NEXT_PUBLIC_VAPID_PUBLIC_KEY!
      ),
    });

    // Send subscription to your server
    const res = await fetch("/api/push/subscribe", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(subscription),
    });

    if (!res.ok) throw new Error("Failed to save subscription");

    return subscription;
  } catch (error) {
    console.error("Push subscription failed:", error);
    return null;
  }
}

function urlBase64ToUint8Array(base64String: string): Uint8Array {
  const padding = "=".repeat((4 - (base64String.length % 4)) % 4);
  const base64 = (base64String + padding).replace(/-/g, "+").replace(/_/g, "/");
  const rawData = window.atob(base64);
  const outputArray = new Uint8Array(rawData.length);
  for (let i = 0; i < rawData.length; i++) {
    outputArray[i] = rawData.charCodeAt(i);
  }
  return outputArray;
}
```

### Save Subscription (API Route)

```tsx
// app/api/push/subscribe/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";

export async function POST(request: NextRequest) {
  try {
    const supabase = await createClient();
    const { data: { user } } = await supabase.auth.getUser();

    if (!user) {
      return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
    }

    const subscription = await request.json();

    const { error } = await supabase.from("push_subscriptions").upsert(
      {
        user_id: user.id,
        endpoint: subscription.endpoint,
        p256dh: subscription.keys.p256dh,
        auth: subscription.keys.auth,
      },
      { onConflict: "endpoint" }
    );

    if (error) throw error;

    return NextResponse.json({ success: true });
  } catch (error) {
    console.error("Subscription save failed:", error);
    return NextResponse.json({ error: "Failed to save" }, { status: 500 });
  }
}
```

### Send Notification (API Route)

```tsx
// app/api/push/send/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";
import webpush from "web-push";

webpush.setVapidDetails(
  process.env.VAPID_EMAIL!,
  process.env.NEXT_PUBLIC_VAPID_PUBLIC_KEY!,
  process.env.VAPID_PRIVATE_KEY!
);

export async function POST(request: NextRequest) {
  try {
    const supabase = await createClient();
    const { userId, title, body, url } = await request.json();

    if (!title || !body) {
      return NextResponse.json({ error: "Title and body required" }, { status: 400 });
    }

    const { data: subscriptions, error } = await supabase
      .from("push_subscriptions")
      .select("endpoint, p256dh, auth")
      .eq("user_id", userId);

    if (error) throw error;
    if (!subscriptions?.length) {
      return NextResponse.json({ error: "No subscriptions found" }, { status: 404 });
    }

    const payload = JSON.stringify({ title, body, url: url || "/" });

    const results = await Promise.allSettled(
      subscriptions.map((sub) =>
        webpush.sendNotification(
          {
            endpoint: sub.endpoint,
            keys: { p256dh: sub.p256dh, auth: sub.auth },
          },
          payload
        )
      )
    );

    const sent = results.filter((r) => r.status === "fulfilled").length;
    const failed = results.filter((r) => r.status === "rejected").length;

    return NextResponse.json({ sent, failed });
  } catch (error) {
    console.error("Push send failed:", error);
    return NextResponse.json({ error: "Send failed" }, { status: 500 });
  }
}
```

## Install Prompt

Capture the browser's install prompt and show your own custom UI.

```tsx
// components/install-prompt.tsx
"use client";

import { useState, useEffect } from "react";

interface BeforeInstallPromptEvent extends Event {
  prompt: () => Promise<void>;
  userChoice: Promise<{ outcome: "accepted" | "dismissed" }>;
}

export function InstallPrompt() {
  const [deferredPrompt, setDeferredPrompt] = useState<BeforeInstallPromptEvent | null>(null);
  const [showPrompt, setShowPrompt] = useState(false);
  const [isInstalled, setIsInstalled] = useState(false);

  useEffect(() => {
    // Check if already installed
    if (window.matchMedia("(display-mode: standalone)").matches) {
      setIsInstalled(true);
      return;
    }

    function handleBeforeInstall(e: Event) {
      e.preventDefault();
      setDeferredPrompt(e as BeforeInstallPromptEvent);
      setShowPrompt(true);
    }

    window.addEventListener("beforeinstallprompt", handleBeforeInstall);

    return () => {
      window.removeEventListener("beforeinstallprompt", handleBeforeInstall);
    };
  }, []);

  async function handleInstall() {
    if (!deferredPrompt) return;

    try {
      await deferredPrompt.prompt();
      const { outcome } = await deferredPrompt.userChoice;

      if (outcome === "accepted") {
        setIsInstalled(true);
      }
    } catch (error) {
      console.error("Install prompt failed:", error);
    } finally {
      setDeferredPrompt(null);
      setShowPrompt(false);
    }
  }

  if (isInstalled || !showPrompt) return null;

  return (
    <div className="fixed bottom-4 left-4 right-4 md:left-auto md:right-4 md:w-80 bg-white rounded-xl shadow-2xl border p-4 z-50">
      <h3 className="font-semibold text-lg">Install Our App</h3>
      <p className="text-sm text-gray-600 mt-1">
        Add this app to your home screen for a faster, native-like experience.
      </p>
      <div className="flex gap-2 mt-4">
        <button
          onClick={() => setShowPrompt(false)}
          className="flex-1 px-4 py-2 text-sm border rounded-lg hover:bg-gray-50"
        >
          Not Now
        </button>
        <button
          onClick={handleInstall}
          className="flex-1 px-4 py-2 text-sm bg-blue-600 text-white rounded-lg hover:bg-blue-700"
        >
          Install
        </button>
      </div>
    </div>
  );
}
```

## iOS-Specific Optimizations

iOS Safari does not support the `beforeinstallprompt` event. You need to show manual instructions.

```tsx
// components/ios-install-guide.tsx
"use client";

import { useState, useEffect } from "react";

export function IOSInstallGuide() {
  const [isIOS, setIsIOS] = useState(false);
  const [isStandalone, setIsStandalone] = useState(false);
  const [dismissed, setDismissed] = useState(false);

  useEffect(() => {
    const ua = navigator.userAgent;
    const ios = /iPad|iPhone|iPod/.test(ua);
    const standalone = window.matchMedia("(display-mode: standalone)").matches
      || ("standalone" in navigator && (navigator as any).standalone);

    setIsIOS(ios);
    setIsStandalone(!!standalone);
    setDismissed(localStorage.getItem("ios-install-dismissed") === "true");
  }, []);

  if (!isIOS || isStandalone || dismissed) return null;

  function handleDismiss() {
    localStorage.setItem("ios-install-dismissed", "true");
    setDismissed(true);
  }

  return (
    <div className="fixed bottom-0 left-0 right-0 bg-white border-t shadow-lg p-4 z-50">
      <button
        onClick={handleDismiss}
        className="absolute top-2 right-2 text-gray-400 hover:text-gray-600"
        aria-label="Close"
      >
        x
      </button>
      <p className="text-sm font-medium">Install this app on your iPhone</p>
      <p className="text-sm text-gray-600 mt-1">
        Tap the Share button in Safari, then tap &quot;Add to Home Screen&quot;.
      </p>
    </div>
  );
}
```

## Background Sync for Offline Forms

When users submit a form while offline, queue it and sync when back online.

```tsx
// lib/offline-queue.ts
type QueuedRequest = {
  id: string;
  url: string;
  method: string;
  body: string;
  timestamp: number;
};

const QUEUE_KEY = "offline-request-queue";

export function queueRequest(url: string, method: string, body: unknown) {
  try {
    const queue = getQueue();
    queue.push({
      id: crypto.randomUUID(),
      url,
      method,
      body: JSON.stringify(body),
      timestamp: Date.now(),
    });
    localStorage.setItem(QUEUE_KEY, JSON.stringify(queue));
  } catch (error) {
    console.error("Failed to queue request:", error);
  }
}

export function getQueue(): QueuedRequest[] {
  try {
    const stored = localStorage.getItem(QUEUE_KEY);
    return stored ? JSON.parse(stored) : [];
  } catch {
    return [];
  }
}

export async function processQueue() {
  const queue = getQueue();
  if (queue.length === 0) return;

  const remaining: QueuedRequest[] = [];

  for (const request of queue) {
    try {
      const res = await fetch(request.url, {
        method: request.method,
        headers: { "Content-Type": "application/json" },
        body: request.body,
      });

      if (!res.ok) {
        remaining.push(request);
      }
    } catch {
      remaining.push(request);
    }
  }

  localStorage.setItem(QUEUE_KEY, JSON.stringify(remaining));
}
```

### Auto-Sync When Online

```tsx
// components/online-sync.tsx
"use client";

import { useEffect } from "react";
import { processQueue } from "@/lib/offline-queue";

export function OnlineSync() {
  useEffect(() => {
    function handleOnline() {
      processQueue();
    }

    window.addEventListener("online", handleOnline);

    // Also process on mount if online
    if (navigator.onLine) {
      processQueue();
    }

    return () => {
      window.removeEventListener("online", handleOnline);
    };
  }, []);

  return null;
}
```

## App Icon Generation Guide

You need these icon sizes at minimum:

| File | Size | Purpose |
|------|------|---------|
| `icon-192.png` | 192x192 | Android home screen |
| `icon-512.png` | 512x512 | Android splash screen |
| `icon-maskable-512.png` | 512x512 | Android adaptive icon (safe zone: center 80%) |
| `apple-touch-icon.png` | 180x180 | iOS home screen |
| `favicon.ico` | 32x32 | Browser tab |

Place all icons in the `public/icons/` folder. For the maskable icon, keep your logo centered in the inner 80% of the image (the outer 20% can be cropped to different shapes).

## Cache Invalidation

When you deploy a new version, you want users to get the latest content.

```tsx
// lib/cache-utils.ts
export async function clearAllCaches() {
  try {
    if ("caches" in window) {
      const cacheNames = await caches.keys();
      await Promise.all(cacheNames.map((name) => caches.delete(name)));
    }
  } catch (error) {
    console.error("Cache clear failed:", error);
  }
}

export async function clearCacheByName(cacheName: string) {
  try {
    if ("caches" in window) {
      await caches.delete(cacheName);
    }
  } catch (error) {
    console.error(`Failed to clear cache ${cacheName}:`, error);
  }
}
```

### Version-Based Cache Busting

```tsx
// components/update-prompt.tsx
"use client";

import { useEffect, useState } from "react";

export function UpdatePrompt() {
  const [updateAvailable, setUpdateAvailable] = useState(false);

  useEffect(() => {
    if (!("serviceWorker" in navigator)) return;

    navigator.serviceWorker.ready.then((registration) => {
      registration.addEventListener("updatefound", () => {
        const newWorker = registration.installing;
        if (!newWorker) return;

        newWorker.addEventListener("statechange", () => {
          if (newWorker.state === "installed" && navigator.serviceWorker.controller) {
            setUpdateAvailable(true);
          }
        });
      });
    });
  }, []);

  function handleUpdate() {
    window.location.reload();
  }

  if (!updateAvailable) return null;

  return (
    <div className="fixed top-4 left-4 right-4 md:left-auto md:right-4 md:w-80 bg-blue-600 text-white rounded-xl shadow-2xl p-4 z-50">
      <p className="font-medium">Update Available</p>
      <p className="text-sm text-blue-100 mt-1">A new version is ready.</p>
      <button
        onClick={handleUpdate}
        className="mt-3 w-full px-4 py-2 bg-white text-blue-600 rounded-lg font-medium hover:bg-blue-50"
      >
        Update Now
      </button>
    </div>
  );
}
```

See the `references/` folder for complete PWA config and push notification setup guides.
