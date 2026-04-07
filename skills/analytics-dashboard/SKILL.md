---
name: analytics-dashboard
description: "Use this skill whenever the user mentions analytics, tracking, metrics, dashboard, charts, graphs, Recharts, PostHog, Plausible, Vercel Analytics, page views, conversions, funnels, A/B testing, experiments, user behavior, events, custom events, click tracking, 'how many users', 'track signups', 'conversion rate', KPIs, reports, data visualization, or ANY analytics/tracking/metrics task — even if they don't explicitly say 'analytics'. This skill gives you full visibility into how your app is performing."
---

# Analytics Dashboard

Add analytics, tracking, charts, and dashboards to your Next.js app. Track user actions, visualize data with Recharts, run A/B tests, and build real-time dashboards.

## Analytics Provider Setup

### Vercel Analytics

```bash
npm install @vercel/analytics
```

```tsx
// app/layout.tsx
import { Analytics } from "@vercel/analytics/react";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        {children}
        <Analytics />
      </body>
    </html>
  );
}
```

### Custom Events

```tsx
// lib/analytics.ts
import { track } from "@vercel/analytics";

export function trackSignup(method: string) {
  try {
    track("signup", { method });
  } catch (error) {
    console.error("Failed to track signup:", error);
  }
}

export function trackPurchase(productId: string, amount: number) {
  try {
    track("purchase", { productId, amount: String(amount) });
  } catch (error) {
    console.error("Failed to track purchase:", error);
  }
}
```

### PostHog Setup

```bash
npm install posthog-js
```

```tsx
// lib/posthog.ts
import posthog from "posthog-js";

let initialized = false;

export function initPostHog() {
  if (typeof window === "undefined" || initialized) return;
  try {
    posthog.init(process.env.NEXT_PUBLIC_POSTHOG_KEY!, {
      api_host: process.env.NEXT_PUBLIC_POSTHOG_HOST || "https://app.posthog.com",
      capture_pageview: true,
      persistence: "localStorage+cookie",
    });
    initialized = true;
  } catch (error) {
    console.error("PostHog init failed:", error);
  }
}
```

## Privacy-Compliant Tracking

```tsx
// components/cookie-consent.tsx
"use client";
import { useState, useEffect } from "react";

export function CookieConsent() {
  const [consent, setConsent] = useState<"pending" | "accepted" | "rejected">("pending");

  useEffect(() => {
    const stored = localStorage.getItem("cookie-consent");
    if (stored) setConsent(stored as typeof consent);
  }, []);

  if (consent !== "pending") return null;

  return (
    <div className="fixed bottom-0 left-0 right-0 bg-white border-t p-4 shadow-lg z-50">
      <div className="max-w-4xl mx-auto flex items-center justify-between gap-4">
        <p className="text-sm text-gray-700">We use analytics cookies to improve our site.</p>
        <div className="flex gap-2">
          <button onClick={() => { localStorage.setItem("cookie-consent", "rejected"); setConsent("rejected"); }}
            className="px-4 py-2 text-sm border rounded hover:bg-gray-50">Decline</button>
          <button onClick={() => { localStorage.setItem("cookie-consent", "accepted"); setConsent("accepted"); }}
            className="px-4 py-2 text-sm bg-blue-600 text-white rounded hover:bg-blue-700">Accept</button>
        </div>
      </div>
    </div>
  );
}
```

**Key GDPR rules**: Never put emails, names, or IPs in event properties. Use anonymous IDs only.

## Funnel Analysis

```sql
-- supabase/migrations/create_funnel_events.sql
CREATE TABLE funnel_events (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  session_id TEXT NOT NULL,
  funnel_name TEXT NOT NULL,
  step_name TEXT NOT NULL,
  step_order INTEGER NOT NULL,
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX idx_funnel_session ON funnel_events(session_id, funnel_name);
```

```tsx
// lib/funnel-tracking.ts
import { createClient } from "@/lib/supabase/client";

export async function trackFunnelStep(
  sessionId: string, funnelName: string, stepName: string, stepOrder: number
) {
  const supabase = createClient();
  try {
    const { error } = await supabase.from("funnel_events").insert({
      session_id: sessionId, funnel_name: funnelName,
      step_name: stepName, step_order: stepOrder,
    });
    if (error) throw error;
  } catch (err) {
    console.error("Failed to track funnel step:", err);
  }
}
```

## Dashboard Components with Recharts

```bash
npm install recharts
```

### Line Chart

```tsx
// components/dashboard/line-chart.tsx
"use client";
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } from "recharts";

export function AnalyticsLineChart({ data, title, color = "#3b82f6" }: {
  data: { date: string; value: number }[]; title: string; color?: string;
}) {
  if (!data?.length) return <p className="text-gray-500 text-center py-8">No data available</p>;
  return (
    <div className="bg-white rounded-lg border p-6">
      <h3 className="text-lg font-semibold mb-4">{title}</h3>
      <ResponsiveContainer width="100%" height={300}>
        <LineChart data={data}>
          <CartesianGrid strokeDasharray="3 3" />
          <XAxis dataKey="date" fontSize={12} />
          <YAxis fontSize={12} />
          <Tooltip />
          <Line type="monotone" dataKey="value" stroke={color} strokeWidth={2} dot={false} />
        </LineChart>
      </ResponsiveContainer>
    </div>
  );
}
```

### KPI Card

```tsx
// components/dashboard/kpi-card.tsx
export function KPICard({ title, value, change, changeLabel }: {
  title: string; value: string | number; change?: number; changeLabel?: string;
}) {
  const isPositive = change !== undefined && change >= 0;
  return (
    <div className="bg-white rounded-lg border p-6">
      <p className="text-sm text-gray-500">{title}</p>
      <p className="text-3xl font-bold mt-1">{value}</p>
      {change !== undefined && (
        <p className={`text-sm mt-2 ${isPositive ? "text-green-600" : "text-red-600"}`}>
          {isPositive ? "+" : ""}{change}% {changeLabel || "vs last period"}
        </p>
      )}
    </div>
  );
}
```

## A/B Testing

```tsx
// lib/ab-testing.ts
export type Variant = "control" | "variant_a" | "variant_b";

export function getVariant(experimentName: string, userId: string): Variant {
  const hash = simpleHash(`${experimentName}:${userId}`);
  const bucket = hash % 100;
  if (bucket < 50) return "control";
  if (bucket < 80) return "variant_a";
  return "variant_b";
}

function simpleHash(str: string): number {
  let hash = 0;
  for (let i = 0; i < str.length; i++) {
    hash = (hash << 5) - hash + str.charCodeAt(i);
    hash = hash & hash;
  }
  return Math.abs(hash);
}
```

## Server-Side Event Tracking

```tsx
// app/api/track/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";

export async function POST(request: NextRequest) {
  try {
    const { event, properties } = await request.json();
    if (!event || typeof event !== "string") {
      return NextResponse.json({ error: "Event name required" }, { status: 400 });
    }
    const supabase = await createClient();
    const { error } = await supabase.from("analytics_events").insert({
      event_name: event, properties: properties || {},
      user_agent: request.headers.get("user-agent") || "unknown",
      referrer: request.headers.get("referer") || null,
    });
    if (error) throw error;
    return NextResponse.json({ success: true });
  } catch (error) {
    console.error("Server tracking error:", error);
    return NextResponse.json({ error: "Tracking failed" }, { status: 500 });
  }
}
```

## Real-Time Dashboard with Supabase Realtime

```tsx
// components/dashboard/realtime-counter.tsx
"use client";
import { useEffect, useState } from "react";
import { createClient } from "@/lib/supabase/client";

export function RealtimeCounter({ eventName }: { eventName: string }) {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const supabase = createClient();
    supabase.from("analytics_events").select("*", { count: "exact", head: true })
      .eq("event_name", eventName).then(({ count: total }) => setCount(total || 0));

    const channel = supabase.channel("analytics-realtime")
      .on("postgres_changes", { event: "INSERT", schema: "public", table: "analytics_events",
        filter: `event_name=eq.${eventName}` }, () => setCount((p) => p + 1))
      .subscribe();
    return () => { supabase.removeChannel(channel); };
  }, [eventName]);

  return (
    <div className="bg-white rounded-lg border p-6 text-center">
      <p className="text-sm text-gray-500">Live {eventName}</p>
      <p className="text-4xl font-bold mt-2">{count.toLocaleString()}</p>
      <span className="inline-flex items-center gap-1 mt-2 text-green-600 text-sm">
        <span className="w-2 h-2 bg-green-500 rounded-full animate-pulse" /> Live
      </span>
    </div>
  );
}
```

## Date Range Filtering

```tsx
// components/dashboard/date-range-picker.tsx
"use client";
import { useState } from "react";

export function DateRangePicker({ onChange }: { onChange: (range: { from: string; to: string }) => void }) {
  const [preset, setPreset] = useState("30d");

  function handlePreset(p: string) {
    setPreset(p);
    const to = new Date().toISOString().split("T")[0];
    const days = p === "7d" ? 7 : p === "30d" ? 30 : 90;
    const from = new Date(Date.now() - days * 86400000).toISOString().split("T")[0];
    onChange({ from, to });
  }

  return (
    <div className="flex gap-2">
      {["7d", "30d", "90d"].map((p) => (
        <button key={p} onClick={() => handlePreset(p)}
          className={`px-3 py-1 text-sm rounded ${preset === p ? "bg-blue-600 text-white" : "bg-gray-100 hover:bg-gray-200"}`}>
          {p === "7d" ? "7 Days" : p === "30d" ? "30 Days" : "90 Days"}
        </button>
      ))}
    </div>
  );
}
```

See `references/analytics-setup.md` for full PostHog/Plausible setup, consent management, and server-side tracking. See `references/dashboard-components.md` for 10+ Recharts components including BarChart, PieChart, AreaChart, FunnelChart, MetricTable, and Sparkline.
