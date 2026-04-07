# Push Notifications Reference

Web Push API setup, VAPID keys, Supabase subscription storage, sending notifications from route handlers, and notification UI.

## VAPID Key Generation

VAPID (Voluntary Application Server Identification) keys let your server send push notifications without needing a third-party service.

```bash
# Install web-push globally
npm install -g web-push

# Generate keys (run once, save the output)
web-push generate-vapid-keys
```

Save the output to your `.env.local`:

```bash
# .env.local
NEXT_PUBLIC_VAPID_PUBLIC_KEY=BEl62iUYgUivx...your_public_key
VAPID_PRIVATE_KEY=UGkG...your_private_key
VAPID_EMAIL=mailto:admin@yourdomain.com
```

Also install web-push as a project dependency:

```bash
npm install web-push
npm install -D @types/web-push
```

## Supabase Subscription Storage

### Database Schema

```sql
-- supabase/migrations/create_push_subscriptions.sql
CREATE TABLE push_subscriptions (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  endpoint TEXT NOT NULL,
  p256dh TEXT NOT NULL,
  auth TEXT NOT NULL,
  device_name TEXT,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(endpoint)
);

-- Index for looking up subscriptions by user
CREATE INDEX idx_push_subs_user ON push_subscriptions(user_id) WHERE is_active = true;

-- Row Level Security
ALTER TABLE push_subscriptions ENABLE ROW LEVEL SECURITY;

-- Users can manage their own subscriptions
CREATE POLICY "Users can read own subscriptions"
  ON push_subscriptions FOR SELECT
  USING (auth.uid() = user_id);

CREATE POLICY "Users can insert own subscriptions"
  ON push_subscriptions FOR INSERT
  WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own subscriptions"
  ON push_subscriptions FOR UPDATE
  USING (auth.uid() = user_id);

CREATE POLICY "Users can delete own subscriptions"
  ON push_subscriptions FOR DELETE
  USING (auth.uid() = user_id);

-- Notification log table
CREATE TABLE notification_log (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID REFERENCES auth.users(id),
  title TEXT NOT NULL,
  body TEXT NOT NULL,
  url TEXT,
  status TEXT NOT NULL DEFAULT 'pending',
  error_message TEXT,
  sent_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_notification_log_user ON notification_log(user_id, created_at DESC);
```

## Client-Side Subscription

### Permission Request and Subscribe

```tsx
// lib/push-notifications.ts

export async function requestNotificationPermission(): Promise<NotificationPermission> {
  try {
    if (!("Notification" in window)) {
      console.warn("Notifications not supported in this browser");
      return "denied";
    }
    return await Notification.requestPermission();
  } catch (error) {
    console.error("Permission request failed:", error);
    return "denied";
  }
}

export async function subscribeToPushNotifications(): Promise<boolean> {
  try {
    // Check browser support
    if (!("serviceWorker" in navigator) || !("PushManager" in window)) {
      console.warn("Push notifications not supported");
      return false;
    }

    // Request permission
    const permission = await requestNotificationPermission();
    if (permission !== "granted") {
      return false;
    }

    // Get service worker registration
    const registration = await navigator.serviceWorker.ready;

    // Check for existing subscription
    let subscription = await registration.pushManager.getSubscription();

    if (!subscription) {
      // Create new subscription
      subscription = await registration.pushManager.subscribe({
        userVisibleOnly: true,
        applicationServerKey: urlBase64ToUint8Array(
          process.env.NEXT_PUBLIC_VAPID_PUBLIC_KEY!
        ),
      });
    }

    // Send subscription to server
    const response = await fetch("/api/push/subscribe", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        endpoint: subscription.endpoint,
        keys: {
          p256dh: arrayBufferToBase64(subscription.getKey("p256dh")!),
          auth: arrayBufferToBase64(subscription.getKey("auth")!),
        },
        deviceName: getDeviceName(),
      }),
    });

    if (!response.ok) {
      throw new Error("Failed to save subscription on server");
    }

    return true;
  } catch (error) {
    console.error("Push subscription failed:", error);
    return false;
  }
}

export async function unsubscribeFromPush(): Promise<boolean> {
  try {
    const registration = await navigator.serviceWorker.ready;
    const subscription = await registration.pushManager.getSubscription();

    if (subscription) {
      // Unsubscribe from browser
      await subscription.unsubscribe();

      // Remove from server
      await fetch("/api/push/unsubscribe", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ endpoint: subscription.endpoint }),
      });
    }

    return true;
  } catch (error) {
    console.error("Unsubscribe failed:", error);
    return false;
  }
}

// Helper: Convert VAPID key to Uint8Array
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

// Helper: Convert ArrayBuffer to base64
function arrayBufferToBase64(buffer: ArrayBuffer): string {
  const bytes = new Uint8Array(buffer);
  let binary = "";
  for (let i = 0; i < bytes.length; i++) {
    binary += String.fromCharCode(bytes[i]);
  }
  return window.btoa(binary);
}

// Helper: Get device name
function getDeviceName(): string {
  const ua = navigator.userAgent;
  if (/iPhone/.test(ua)) return "iPhone";
  if (/iPad/.test(ua)) return "iPad";
  if (/Android/.test(ua)) return "Android";
  if (/Mac/.test(ua)) return "Mac";
  if (/Windows/.test(ua)) return "Windows";
  return "Unknown";
}
```

## Server-Side API Routes

### Subscribe Route

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

    const { endpoint, keys, deviceName } = await request.json();

    if (!endpoint || !keys?.p256dh || !keys?.auth) {
      return NextResponse.json(
        { error: "Invalid subscription data" },
        { status: 400 }
      );
    }

    const { error } = await supabase.from("push_subscriptions").upsert(
      {
        user_id: user.id,
        endpoint,
        p256dh: keys.p256dh,
        auth: keys.auth,
        device_name: deviceName || "Unknown",
        is_active: true,
        updated_at: new Date().toISOString(),
      },
      { onConflict: "endpoint" }
    );

    if (error) throw error;

    return NextResponse.json({ success: true });
  } catch (error) {
    console.error("Subscribe error:", error);
    return NextResponse.json(
      { error: "Failed to save subscription" },
      { status: 500 }
    );
  }
}
```

### Unsubscribe Route

```tsx
// app/api/push/unsubscribe/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";

export async function POST(request: NextRequest) {
  try {
    const supabase = await createClient();
    const { data: { user } } = await supabase.auth.getUser();

    if (!user) {
      return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
    }

    const { endpoint } = await request.json();

    if (!endpoint) {
      return NextResponse.json({ error: "Endpoint required" }, { status: 400 });
    }

    const { error } = await supabase
      .from("push_subscriptions")
      .update({ is_active: false, updated_at: new Date().toISOString() })
      .eq("endpoint", endpoint)
      .eq("user_id", user.id);

    if (error) throw error;

    return NextResponse.json({ success: true });
  } catch (error) {
    console.error("Unsubscribe error:", error);
    return NextResponse.json(
      { error: "Failed to unsubscribe" },
      { status: 500 }
    );
  }
}
```

### Send Notification Route

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

type SendRequest = {
  userId: string;
  title: string;
  body: string;
  url?: string;
  icon?: string;
  badge?: string;
};

export async function POST(request: NextRequest) {
  try {
    const supabase = await createClient();
    const body: SendRequest = await request.json();

    // Validate input
    if (!body.userId || !body.title || !body.body) {
      return NextResponse.json(
        { error: "userId, title, and body are required" },
        { status: 400 }
      );
    }

    // Get active subscriptions for user
    const { data: subscriptions, error: fetchError } = await supabase
      .from("push_subscriptions")
      .select("id, endpoint, p256dh, auth")
      .eq("user_id", body.userId)
      .eq("is_active", true);

    if (fetchError) throw fetchError;

    if (!subscriptions || subscriptions.length === 0) {
      return NextResponse.json(
        { error: "No active subscriptions found" },
        { status: 404 }
      );
    }

    // Build notification payload
    const payload = JSON.stringify({
      title: body.title,
      body: body.body,
      url: body.url || "/",
      icon: body.icon || "/icons/icon-192.png",
      badge: body.badge || "/icons/badge-72.png",
      timestamp: Date.now(),
    });

    // Send to all subscriptions
    const results = await Promise.allSettled(
      subscriptions.map(async (sub) => {
        try {
          await webpush.sendNotification(
            {
              endpoint: sub.endpoint,
              keys: { p256dh: sub.p256dh, auth: sub.auth },
            },
            payload
          );
          return { id: sub.id, status: "sent" as const };
        } catch (err: any) {
          // If subscription is expired (410 Gone), mark inactive
          if (err.statusCode === 410 || err.statusCode === 404) {
            await supabase
              .from("push_subscriptions")
              .update({ is_active: false })
              .eq("id", sub.id);
          }
          return { id: sub.id, status: "failed" as const, error: err.message };
        }
      })
    );

    const sent = results.filter(
      (r) => r.status === "fulfilled" && r.value.status === "sent"
    ).length;
    const failed = results.length - sent;

    // Log the notification
    await supabase.from("notification_log").insert({
      user_id: body.userId,
      title: body.title,
      body: body.body,
      url: body.url,
      status: failed === 0 ? "sent" : "partial",
      sent_at: new Date().toISOString(),
    });

    return NextResponse.json({ sent, failed, total: results.length });
  } catch (error) {
    console.error("Send notification error:", error);
    return NextResponse.json(
      { error: "Failed to send notifications" },
      { status: 500 }
    );
  }
}
```

## Service Worker Push Event Handler

Add this to your service worker to handle incoming push messages.

```ts
// Add to app/sw.ts — push event handling

self.addEventListener("push", (event: PushEvent) => {
  if (!event.data) return;

  try {
    const data = event.data.json();

    const options: NotificationOptions = {
      body: data.body,
      icon: data.icon || "/icons/icon-192.png",
      badge: data.badge || "/icons/badge-72.png",
      data: { url: data.url || "/" },
      actions: [
        { action: "open", title: "Open" },
        { action: "dismiss", title: "Dismiss" },
      ],
      vibrate: [100, 50, 100],
      tag: data.tag || "default",
      renotify: true,
    };

    event.waitUntil(self.registration.showNotification(data.title, options));
  } catch (error) {
    console.error("Push event handling failed:", error);
  }
});

self.addEventListener("notificationclick", (event: NotificationEvent) => {
  event.notification.close();

  if (event.action === "dismiss") return;

  const url = event.notification.data?.url || "/";

  event.waitUntil(
    self.clients.matchAll({ type: "window" }).then((clientList) => {
      // Focus existing window if open
      for (const client of clientList) {
        if (client.url.includes(url) && "focus" in client) {
          return client.focus();
        }
      }
      // Otherwise open new window
      return self.clients.openWindow(url);
    })
  );
});
```

## Notification UI Components

### Notification Permission Button

```tsx
// components/notification-button.tsx
"use client";

import { useState, useEffect } from "react";
import {
  subscribeToPushNotifications,
  unsubscribeFromPush,
} from "@/lib/push-notifications";

export function NotificationButton() {
  const [permission, setPermission] = useState<NotificationPermission>("default");
  const [isSubscribed, setIsSubscribed] = useState(false);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    if ("Notification" in window) {
      setPermission(Notification.permission);
    }

    // Check existing subscription
    if ("serviceWorker" in navigator) {
      navigator.serviceWorker.ready.then(async (reg) => {
        const sub = await reg.pushManager.getSubscription();
        setIsSubscribed(!!sub);
      });
    }
  }, []);

  async function handleToggle() {
    setLoading(true);
    try {
      if (isSubscribed) {
        const success = await unsubscribeFromPush();
        if (success) setIsSubscribed(false);
      } else {
        const success = await subscribeToPushNotifications();
        if (success) {
          setIsSubscribed(true);
          setPermission("granted");
        }
      }
    } catch (error) {
      console.error("Toggle notification failed:", error);
    } finally {
      setLoading(false);
    }
  }

  if (permission === "denied") {
    return (
      <div className="text-sm text-gray-500">
        Notifications are blocked. Enable them in your browser settings.
      </div>
    );
  }

  return (
    <button
      onClick={handleToggle}
      disabled={loading}
      className={`px-4 py-2 rounded-lg text-sm font-medium transition-colors disabled:opacity-50 ${
        isSubscribed
          ? "bg-gray-200 text-gray-700 hover:bg-gray-300"
          : "bg-blue-600 text-white hover:bg-blue-700"
      }`}
    >
      {loading
        ? "Processing..."
        : isSubscribed
        ? "Disable Notifications"
        : "Enable Notifications"}
    </button>
  );
}
```

### Notification Preferences Page

```tsx
// app/settings/notifications/page.tsx
import { createClient } from "@/lib/supabase/server";
import { NotificationButton } from "@/components/notification-button";
import { DeviceList } from "./device-list";

export default async function NotificationSettings() {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();

  if (!user) {
    return <p className="p-8 text-red-600">Please sign in to manage notifications.</p>;
  }

  const { data: subscriptions, error } = await supabase
    .from("push_subscriptions")
    .select("id, device_name, is_active, created_at")
    .eq("user_id", user.id)
    .order("created_at", { ascending: false });

  if (error) {
    return <p className="p-8 text-red-600">Failed to load settings: {error.message}</p>;
  }

  return (
    <div className="max-w-2xl mx-auto p-8 space-y-8">
      <div>
        <h1 className="text-2xl font-bold">Notification Settings</h1>
        <p className="text-gray-600 mt-1">Manage how you receive notifications.</p>
      </div>

      <div className="bg-white rounded-lg border p-6">
        <h2 className="text-lg font-semibold mb-4">Push Notifications</h2>
        <NotificationButton />
      </div>

      <div className="bg-white rounded-lg border p-6">
        <h2 className="text-lg font-semibold mb-4">Registered Devices</h2>
        {subscriptions && subscriptions.length > 0 ? (
          <DeviceList devices={subscriptions} />
        ) : (
          <p className="text-gray-500 text-sm">No devices registered.</p>
        )}
      </div>
    </div>
  );
}
```

### Sending Notifications from Server Actions

```tsx
// lib/send-notification.ts
import webpush from "web-push";
import { createClient } from "@/lib/supabase/server";

webpush.setVapidDetails(
  process.env.VAPID_EMAIL!,
  process.env.NEXT_PUBLIC_VAPID_PUBLIC_KEY!,
  process.env.VAPID_PRIVATE_KEY!
);

export async function sendNotificationToUser(
  userId: string,
  title: string,
  body: string,
  url?: string
) {
  const supabase = await createClient();

  try {
    const { data: subs, error } = await supabase
      .from("push_subscriptions")
      .select("endpoint, p256dh, auth")
      .eq("user_id", userId)
      .eq("is_active", true);

    if (error) throw error;
    if (!subs || subs.length === 0) return { sent: 0, failed: 0 };

    const payload = JSON.stringify({
      title,
      body,
      url: url || "/",
      icon: "/icons/icon-192.png",
    });

    let sent = 0;
    let failed = 0;

    for (const sub of subs) {
      try {
        await webpush.sendNotification(
          { endpoint: sub.endpoint, keys: { p256dh: sub.p256dh, auth: sub.auth } },
          payload
        );
        sent++;
      } catch {
        failed++;
      }
    }

    return { sent, failed };
  } catch (error) {
    console.error("sendNotificationToUser failed:", error);
    return { sent: 0, failed: 0 };
  }
}
```

Use this function from anywhere on the server — API routes, server actions, cron jobs, webhooks, etc.
