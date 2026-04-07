# Analytics Setup Reference

Complete setup guides for Vercel Analytics, PostHog, and Plausible with custom event schemas, consent management, and server-side tracking.

## Vercel Analytics — Full Setup

### Installation

```bash
npm install @vercel/analytics @vercel/speed-insights
```

### Root Layout Integration

```tsx
// app/layout.tsx
import { Analytics } from "@vercel/analytics/react";
import { SpeedInsights } from "@vercel/speed-insights/next";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        {children}
        <Analytics />
        <SpeedInsights />
      </body>
    </html>
  );
}
```

### Custom Event Schema

Define your events in one place so your whole team uses the same names and properties.

```tsx
// lib/events.ts
import { track } from "@vercel/analytics";

// Define all event types here
type AnalyticsEvent =
  | { name: "signup_completed"; properties: { method: "email" | "google" | "github" } }
  | { name: "purchase_completed"; properties: { plan: string; amount: string } }
  | { name: "feature_used"; properties: { feature: string; context: string } }
  | { name: "search_performed"; properties: { query_length: string; results_count: string } }
  | { name: "error_occurred"; properties: { error_type: string; page: string } }
  | { name: "cta_clicked"; properties: { cta_id: string; page: string } }
  | { name: "funnel_step"; properties: { funnel: string; step: string; step_number: string } };

export function trackEvent(event: AnalyticsEvent) {
  try {
    track(event.name, event.properties);
  } catch (error) {
    console.error(`Failed to track ${event.name}:`, error);
  }
}
```

Usage:

```tsx
// Anywhere in your app
import { trackEvent } from "@/lib/events";

trackEvent({
  name: "signup_completed",
  properties: { method: "google" },
});

trackEvent({
  name: "cta_clicked",
  properties: { cta_id: "hero-signup", page: "/landing" },
});
```

## PostHog — Full Setup

### Installation and Configuration

```bash
npm install posthog-js posthog-node
```

### Client-Side Provider

```tsx
// lib/posthog.ts
import posthog from "posthog-js";

export function initPostHog() {
  if (typeof window === "undefined") return;

  try {
    posthog.init(process.env.NEXT_PUBLIC_POSTHOG_KEY!, {
      api_host: process.env.NEXT_PUBLIC_POSTHOG_HOST || "https://us.i.posthog.com",
      person_profiles: "identified_only",
      capture_pageview: true,
      capture_pageleave: true,
      autocapture: false, // Manual tracking is more reliable
      persistence: "localStorage+cookie",
      loaded: (ph) => {
        if (process.env.NODE_ENV === "development") {
          ph.debug();
        }
      },
    });
  } catch (error) {
    console.error("PostHog initialization failed:", error);
  }
}

export function identifyUser(userId: string, traits?: Record<string, unknown>) {
  try {
    posthog.identify(userId, traits);
  } catch (error) {
    console.error("PostHog identify failed:", error);
  }
}

export function resetUser() {
  try {
    posthog.reset();
  } catch (error) {
    console.error("PostHog reset failed:", error);
  }
}

export function captureEvent(event: string, properties?: Record<string, unknown>) {
  try {
    posthog.capture(event, properties);
  } catch (error) {
    console.error(`PostHog capture failed for ${event}:`, error);
  }
}

export { posthog };
```

### Server-Side PostHog (for API Routes)

```tsx
// lib/posthog-server.ts
import { PostHog } from "posthog-node";

let client: PostHog | null = null;

export function getPostHogServer(): PostHog {
  if (!client) {
    client = new PostHog(process.env.POSTHOG_API_KEY!, {
      host: process.env.NEXT_PUBLIC_POSTHOG_HOST || "https://us.i.posthog.com",
      flushAt: 1,
      flushInterval: 0,
    });
  }
  return client;
}

export function serverCapture(
  distinctId: string,
  event: string,
  properties?: Record<string, unknown>
) {
  try {
    const ph = getPostHogServer();
    ph.capture({ distinctId, event, properties });
  } catch (error) {
    console.error(`Server-side PostHog capture failed for ${event}:`, error);
  }
}
```

### PostHog Feature Flags

```tsx
// lib/feature-flags.ts
import { posthog } from "@/lib/posthog";

export function isFeatureEnabled(flagName: string): boolean {
  try {
    return posthog.isFeatureEnabled(flagName) ?? false;
  } catch {
    return false;
  }
}

export function getFeatureFlag(flagName: string): string | boolean | undefined {
  try {
    return posthog.getFeatureFlag(flagName);
  } catch {
    return undefined;
  }
}
```

## Plausible — Full Setup

Plausible is a privacy-first analytics tool. No cookies needed, so no consent banner required.

### Script Setup

```tsx
// app/layout.tsx
import Script from "next/script";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        {children}
        <Script
          defer
          data-domain={process.env.NEXT_PUBLIC_PLAUSIBLE_DOMAIN}
          src="https://plausible.io/js/script.js"
          strategy="afterInteractive"
        />
      </body>
    </html>
  );
}
```

### Custom Events with Plausible

```tsx
// lib/plausible.ts
declare global {
  interface Window {
    plausible?: (event: string, options?: { props?: Record<string, string> }) => void;
  }
}

export function trackPlausibleEvent(event: string, props?: Record<string, string>) {
  try {
    if (typeof window !== "undefined" && window.plausible) {
      window.plausible(event, { props });
    }
  } catch (error) {
    console.error("Plausible tracking failed:", error);
  }
}
```

## Consent Management

### Full Consent Manager

```tsx
// lib/consent.ts
export type ConsentPreferences = {
  analytics: boolean;
  marketing: boolean;
  functional: boolean;
};

const DEFAULT_CONSENT: ConsentPreferences = {
  analytics: false,
  marketing: false,
  functional: true,
};

export function getConsent(): ConsentPreferences {
  if (typeof window === "undefined") return DEFAULT_CONSENT;

  try {
    const stored = localStorage.getItem("consent-preferences");
    if (!stored) return DEFAULT_CONSENT;
    return JSON.parse(stored) as ConsentPreferences;
  } catch {
    return DEFAULT_CONSENT;
  }
}

export function setConsent(preferences: ConsentPreferences) {
  try {
    localStorage.setItem("consent-preferences", JSON.stringify(preferences));
    localStorage.setItem("consent-date", new Date().toISOString());

    // Enable or disable tracking based on consent
    if (preferences.analytics) {
      enableAnalytics();
    } else {
      disableAnalytics();
    }
  } catch (error) {
    console.error("Failed to save consent:", error);
  }
}

export function hasConsentBeenGiven(): boolean {
  if (typeof window === "undefined") return false;
  return localStorage.getItem("consent-preferences") !== null;
}

function enableAnalytics() {
  // Re-initialize your analytics provider here
}

function disableAnalytics() {
  // Disable tracking, clear cookies if needed
}
```

### Consent-Aware Tracking Wrapper

```tsx
// lib/safe-track.ts
import { getConsent } from "@/lib/consent";
import { trackEvent } from "@/lib/events";

export function safeTrack(
  name: string,
  properties?: Record<string, string>
) {
  const consent = getConsent();
  if (!consent.analytics) return;

  // Strip any PII that might have been passed accidentally
  const cleanProperties: Record<string, string> = {};
  const piiKeys = ["email", "name", "phone", "address", "ip", "ssn"];

  if (properties) {
    for (const [key, value] of Object.entries(properties)) {
      if (!piiKeys.includes(key.toLowerCase())) {
        cleanProperties[key] = value;
      }
    }
  }

  trackEvent({ name: name as any, properties: cleanProperties as any });
}
```

## Server-Side Tracking

### Supabase Events Table

```sql
-- supabase/migrations/create_analytics_events.sql
CREATE TABLE analytics_events (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  event_name TEXT NOT NULL,
  properties JSONB DEFAULT '{}',
  user_agent TEXT,
  referrer TEXT,
  session_id TEXT,
  user_id UUID REFERENCES auth.users(id),
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Indexes for common queries
CREATE INDEX idx_events_name ON analytics_events(event_name);
CREATE INDEX idx_events_created ON analytics_events(created_at DESC);
CREATE INDEX idx_events_user ON analytics_events(user_id);
CREATE INDEX idx_events_session ON analytics_events(session_id);

-- Row Level Security
ALTER TABLE analytics_events ENABLE ROW LEVEL SECURITY;

-- Only admins can read events
CREATE POLICY "Admins can read events"
  ON analytics_events FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM user_roles
      WHERE user_roles.user_id = auth.uid()
      AND user_roles.role = 'admin'
    )
  );

-- Anyone can insert events (for tracking)
CREATE POLICY "Anyone can insert events"
  ON analytics_events FOR INSERT
  WITH CHECK (true);
```

### Server-Side Tracking API Route

```tsx
// app/api/events/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";

const ALLOWED_EVENTS = [
  "page_view",
  "signup_completed",
  "purchase_completed",
  "feature_used",
  "search_performed",
  "cta_clicked",
  "funnel_step",
];

export async function POST(request: NextRequest) {
  try {
    const body = await request.json();
    const { event, properties, sessionId } = body;

    // Validate event name
    if (!event || !ALLOWED_EVENTS.includes(event)) {
      return NextResponse.json(
        { error: "Invalid event name" },
        { status: 400 }
      );
    }

    // Validate properties size
    const propsString = JSON.stringify(properties || {});
    if (propsString.length > 10000) {
      return NextResponse.json(
        { error: "Properties too large" },
        { status: 400 }
      );
    }

    const supabase = await createClient();

    // Get the authenticated user if available
    const { data: { user } } = await supabase.auth.getUser();

    const { error } = await supabase.from("analytics_events").insert({
      event_name: event,
      properties: properties || {},
      session_id: sessionId || null,
      user_id: user?.id || null,
      user_agent: request.headers.get("user-agent") || "unknown",
      referrer: request.headers.get("referer") || null,
    });

    if (error) throw error;

    return NextResponse.json({ success: true });
  } catch (error) {
    console.error("Event tracking error:", error);
    return NextResponse.json(
      { error: "Failed to track event" },
      { status: 500 }
    );
  }
}
```

### Client-Side Beacon for Reliable Tracking

```tsx
// lib/beacon-track.ts
export function beaconTrack(event: string, properties?: Record<string, string>) {
  if (typeof navigator === "undefined" || !navigator.sendBeacon) return;

  try {
    const payload = JSON.stringify({ event, properties });
    const blob = new Blob([payload], { type: "application/json" });
    navigator.sendBeacon("/api/events", blob);
  } catch (error) {
    console.error("Beacon tracking failed:", error);
  }
}
```

This method uses `navigator.sendBeacon` which reliably sends data even when the user is leaving the page. Use this for page leave events and exit tracking.

## Environment Variables

```bash
# .env.local

# Vercel Analytics — automatic, no key needed if deployed on Vercel

# PostHog
NEXT_PUBLIC_POSTHOG_KEY=phc_your_key_here
NEXT_PUBLIC_POSTHOG_HOST=https://us.i.posthog.com
POSTHOG_API_KEY=phx_your_server_key_here

# Plausible
NEXT_PUBLIC_PLAUSIBLE_DOMAIN=yourdomain.com
```
