---
name: mobile-pwa
description: "Use this skill whenever the user mentions PWA, progressive web app, mobile app, app-like, offline, service worker, push notifications, web manifest, install prompt, 'add to home screen', mobile experience, iOS app, Android app, 'works offline', app icon, splash screen, 'native feel', background sync, cache strategy, or ANY mobile/app-like experience task — even if they don't explicitly say 'PWA'. This skill gives your web app a native app experience without the App Store."
---

# Mobile PWA

Turn your Next.js app into a Progressive Web App. Users can install it on their phone, use it offline, and receive push notifications — no App Store needed.

## Next.js PWA Configuration with Serwist

```bash
npm install @serwist/next && npm install -D serwist
```

```ts
// next.config.ts
import withSerwistInit from "@serwist/next";

const withSerwist = withSerwistInit({
  swSrc: "app/sw.ts",
  swDest: "public/sw.js",
  reloadOnOnline: true,
  disable: process.env.NODE_ENV === "development",
});

export default withSerwist({});
```

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
    entries: [{ url: "/offline", matcher({ request }) { return request.destination === "document"; } }],
  },
});
serwist.addEventListeners();
```

## Web Manifest

```json
// public/manifest.json
{
  "name": "My App",
  "short_name": "MyApp",
  "description": "Your app description",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#3b82f6",
  "icons": [
    { "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png" },
    { "src": "/icons/icon-maskable-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ]
}
```

```tsx
// app/layout.tsx
import type { Metadata, Viewport } from "next";

export const metadata: Metadata = {
  title: "My App",
  manifest: "/manifest.json",
  appleWebApp: { capable: true, statusBarStyle: "default", title: "My App" },
};

export const viewport: Viewport = {
  themeColor: "#3b82f6", width: "device-width", initialScale: 1, maximumScale: 1,
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <head>
        <link rel="apple-touch-icon" href="/icons/apple-touch-icon.png" />
        <meta name="apple-mobile-web-app-capable" content="yes" />
      </head>
      <body>{children}</body>
    </html>
  );
}
```

## Service Worker Cache Strategies

- **Cache-First**: Static assets (images, fonts, CSS). Loads instantly from cache.
- **Network-First**: API data. Tries network, falls back to cache when offline.
- **Stale-While-Revalidate**: Semi-dynamic content. Shows cached version, updates in background.

## Offline Fallback Page

```tsx
// app/offline/page.tsx
export default function OfflinePage() {
  return (
    <div className="min-h-screen flex items-center justify-center p-4 bg-gray-50">
      <div className="text-center max-w-md">
        <h1 className="text-2xl font-bold mb-2">You are offline</h1>
        <p className="text-gray-600 mb-6">Check your internet connection and try again.</p>
        <button onClick={() => window.location.reload()}
          className="px-6 py-3 bg-blue-600 text-white rounded-lg hover:bg-blue-700">Try Again</button>
      </div>
    </div>
  );
}
```

## Push Notifications

Generate VAPID keys once: `npx web-push generate-vapid-keys`

```bash
# .env.local
NEXT_PUBLIC_VAPID_PUBLIC_KEY=your_public_key
VAPID_PRIVATE_KEY=your_private_key
VAPID_EMAIL=mailto:you@example.com
```

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
CREATE POLICY "Users manage own" ON push_subscriptions FOR ALL USING (auth.uid() = user_id);
```

```tsx
// lib/push-notifications.ts
export async function subscribeToPush(): Promise<PushSubscription | null> {
  try {
    if (!("serviceWorker" in navigator) || !("PushManager" in window)) return null;
    const permission = await Notification.requestPermission();
    if (permission !== "granted") return null;

    const registration = await navigator.serviceWorker.ready;
    const subscription = await registration.pushManager.subscribe({
      userVisibleOnly: true,
      applicationServerKey: urlBase64ToUint8Array(process.env.NEXT_PUBLIC_VAPID_PUBLIC_KEY!),
    });

    const res = await fetch("/api/push/subscribe", {
      method: "POST", headers: { "Content-Type": "application/json" },
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
  return Uint8Array.from(rawData, (c) => c.charCodeAt(0));
}
```

```tsx
// app/api/push/subscribe/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";

export async function POST(request: NextRequest) {
  try {
    const supabase = await createClient();
    const { data: { user } } = await supabase.auth.getUser();
    if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

    const subscription = await request.json();
    const { error } = await supabase.from("push_subscriptions").upsert({
      user_id: user.id, endpoint: subscription.endpoint,
      p256dh: subscription.keys.p256dh, auth: subscription.keys.auth,
    }, { onConflict: "endpoint" });
    if (error) throw error;
    return NextResponse.json({ success: true });
  } catch (error) {
    console.error("Subscribe error:", error);
    return NextResponse.json({ error: "Failed to save" }, { status: 500 });
  }
}
```

## Install Prompt

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

  useEffect(() => {
    if (window.matchMedia("(display-mode: standalone)").matches) return;
    function handler(e: Event) { e.preventDefault(); setDeferredPrompt(e as BeforeInstallPromptEvent); setShowPrompt(true); }
    window.addEventListener("beforeinstallprompt", handler);
    return () => window.removeEventListener("beforeinstallprompt", handler);
  }, []);

  async function handleInstall() {
    if (!deferredPrompt) return;
    try { await deferredPrompt.prompt(); } catch (e) { console.error("Install failed:", e); }
    finally { setDeferredPrompt(null); setShowPrompt(false); }
  }

  if (!showPrompt) return null;
  return (
    <div className="fixed bottom-4 right-4 w-80 bg-white rounded-xl shadow-2xl border p-4 z-50">
      <h3 className="font-semibold">Install Our App</h3>
      <p className="text-sm text-gray-600 mt-1">Add to your home screen for a native experience.</p>
      <div className="flex gap-2 mt-4">
        <button onClick={() => setShowPrompt(false)} className="flex-1 px-4 py-2 text-sm border rounded-lg">Not Now</button>
        <button onClick={handleInstall} className="flex-1 px-4 py-2 text-sm bg-blue-600 text-white rounded-lg">Install</button>
      </div>
    </div>
  );
}
```

## iOS-Specific Optimizations

iOS Safari does not support `beforeinstallprompt`. Show manual instructions instead.

```tsx
// components/ios-install-guide.tsx
"use client";
import { useState, useEffect } from "react";

export function IOSInstallGuide() {
  const [show, setShow] = useState(false);

  useEffect(() => {
    const isIOS = /iPad|iPhone|iPod/.test(navigator.userAgent);
    const isStandalone = window.matchMedia("(display-mode: standalone)").matches;
    const dismissed = localStorage.getItem("ios-install-dismissed") === "true";
    setShow(isIOS && !isStandalone && !dismissed);
  }, []);

  if (!show) return null;
  return (
    <div className="fixed bottom-0 left-0 right-0 bg-white border-t shadow-lg p-4 z-50">
      <button onClick={() => { localStorage.setItem("ios-install-dismissed", "true"); setShow(false); }}
        className="absolute top-2 right-2 text-gray-400" aria-label="Close">x</button>
      <p className="text-sm font-medium">Install this app on your iPhone</p>
      <p className="text-sm text-gray-600 mt-1">Tap Share, then &quot;Add to Home Screen&quot;.</p>
    </div>
  );
}
```

## Background Sync for Offline Forms

```tsx
// lib/offline-queue.ts
type QueuedRequest = { id: string; url: string; method: string; body: string; timestamp: number };
const QUEUE_KEY = "offline-request-queue";

export function queueRequest(url: string, method: string, body: unknown) {
  try {
    const queue = getQueue();
    queue.push({ id: crypto.randomUUID(), url, method, body: JSON.stringify(body), timestamp: Date.now() });
    localStorage.setItem(QUEUE_KEY, JSON.stringify(queue));
  } catch (error) { console.error("Failed to queue:", error); }
}

export function getQueue(): QueuedRequest[] {
  try { return JSON.parse(localStorage.getItem(QUEUE_KEY) || "[]"); } catch { return []; }
}

export async function processQueue() {
  const queue = getQueue();
  const remaining: QueuedRequest[] = [];
  for (const req of queue) {
    try {
      const res = await fetch(req.url, { method: req.method, headers: { "Content-Type": "application/json" }, body: req.body });
      if (!res.ok) remaining.push(req);
    } catch { remaining.push(req); }
  }
  localStorage.setItem(QUEUE_KEY, JSON.stringify(remaining));
}
```

```tsx
// components/online-sync.tsx
"use client";
import { useEffect } from "react";
import { processQueue } from "@/lib/offline-queue";

export function OnlineSync() {
  useEffect(() => {
    window.addEventListener("online", processQueue);
    if (navigator.onLine) processQueue();
    return () => window.removeEventListener("online", processQueue);
  }, []);
  return null;
}
```

## App Icon Sizes

| File | Size | Purpose |
|------|------|---------|
| `icon-192.png` | 192x192 | Android home screen |
| `icon-512.png` | 512x512 | Android splash |
| `icon-maskable-512.png` | 512x512 | Android adaptive (keep logo in center 80%) |
| `apple-touch-icon.png` | 180x180 | iOS home screen |
| `favicon.ico` | 32x32 | Browser tab |

## Cache Invalidation

```tsx
// components/update-prompt.tsx
"use client";
import { useEffect, useState } from "react";

export function UpdatePrompt() {
  const [available, setAvailable] = useState(false);

  useEffect(() => {
    if (!("serviceWorker" in navigator)) return;
    navigator.serviceWorker.ready.then((reg) => {
      reg.addEventListener("updatefound", () => {
        const sw = reg.installing;
        sw?.addEventListener("statechange", () => {
          if (sw.state === "installed" && navigator.serviceWorker.controller) setAvailable(true);
        });
      });
    });
  }, []);

  if (!available) return null;
  return (
    <div className="fixed top-4 right-4 w-80 bg-blue-600 text-white rounded-xl shadow-2xl p-4 z-50">
      <p className="font-medium">Update Available</p>
      <button onClick={() => window.location.reload()}
        className="mt-3 w-full px-4 py-2 bg-white text-blue-600 rounded-lg font-medium">Update Now</button>
    </div>
  );
}
```

See `references/pwa-config.md` for complete manifest, serwist config, icon checklist, and iOS splash screens. See `references/push-notifications.md` for full Web Push API, VAPID setup, send notifications from server, and notification UI components.
