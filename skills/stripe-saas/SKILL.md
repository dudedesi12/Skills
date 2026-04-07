---
name: stripe-saas
description: "Use this skill whenever the user mentions Stripe, payments, subscriptions, billing, checkout, pricing page, pricing table, free trial, paid plan, pro plan, premium, upgrade, downgrade, cancel subscription, refund, invoice, customer portal, webhook, payment link, GST, tax, revenue, monetization, SaaS, 'charge users', 'accept payments', 'make money', or ANY payment/billing task — even if they don't explicitly say 'Stripe'. This skill handles all revenue operations."
---

# Stripe SaaS Integration

Payments and subscriptions for Next.js + Supabase using Stripe.

## Setup

```bash
npm install stripe @stripe/stripe-js @stripe/react-stripe-js
```

```typescript
// lib/stripe.ts
import Stripe from "stripe";
if (!process.env.STRIPE_SECRET_KEY) throw new Error("STRIPE_SECRET_KEY is not set");
export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY, { apiVersion: "2024-12-18.acacia", typescript: true });
```

```typescript
// lib/stripe-client.ts
"use client";
import { loadStripe } from "@stripe/stripe-js";
export const stripePromise = loadStripe(process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY!);
```

```bash
# .env.local
STRIPE_SECRET_KEY=sk_test_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
```

## Subscription Checkout

See `references/checkout-patterns.md` for one-time, embedded, and custom Element checkouts.

```typescript
// app/api/checkout/subscription/route.ts
import { NextRequest, NextResponse } from "next/server";
import { stripe } from "@/lib/stripe";
import { createClient } from "@/lib/supabase/server";

export async function POST(request: NextRequest) {
  try {
    const supabase = await createClient();
    const { data: { user } } = await supabase.auth.getUser();
    if (!user) return NextResponse.json({ error: "Not authenticated" }, { status: 401 });

    const { priceId, trialDays } = await request.json();
    if (!priceId) return NextResponse.json({ error: "Price ID required" }, { status: 400 });

    // Find or create Stripe customer
    const { data: profile } = await supabase.from("profiles").select("stripe_customer_id").eq("id", user.id).single();
    let customerId = profile?.stripe_customer_id;
    if (!customerId) {
      const customer = await stripe.customers.create({ email: user.email!, metadata: { supabase_user_id: user.id } });
      customerId = customer.id;
      await supabase.from("profiles").update({ stripe_customer_id: customerId }).eq("id", user.id);
    }

    const session = await stripe.checkout.sessions.create({
      customer: customerId,
      line_items: [{ price: priceId, quantity: 1 }],
      mode: "subscription",
      subscription_data: trialDays ? { trial_period_days: trialDays } : undefined,
      success_url: `${process.env.NEXT_PUBLIC_APP_URL}/dashboard?checkout=success`,
      cancel_url: `${process.env.NEXT_PUBLIC_APP_URL}/pricing`,
      metadata: { userId: user.id },
    });
    return NextResponse.json({ url: session.url });
  } catch (err) {
    console.error("Checkout error:", err);
    return NextResponse.json({ error: "Checkout failed" }, { status: 500 });
  }
}
```

## Webhook Handler

See `references/webhook-events.md` for all event handlers with full Supabase sync code.

```typescript
// app/api/webhooks/stripe/route.ts
import { NextRequest, NextResponse } from "next/server";
import { stripe } from "@/lib/stripe";
import { createClient } from "@supabase/supabase-js";
import Stripe from "stripe";

const supabase = createClient(process.env.NEXT_PUBLIC_SUPABASE_URL!, process.env.SUPABASE_SERVICE_ROLE_KEY!);

export async function POST(request: NextRequest) {
  const body = await request.text();
  const signature = request.headers.get("stripe-signature");
  if (!signature) return NextResponse.json({ error: "No signature" }, { status: 400 });

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(body, signature, process.env.STRIPE_WEBHOOK_SECRET!);
  } catch (err) {
    return NextResponse.json({ error: "Invalid signature" }, { status: 400 });
  }

  try {
    switch (event.type) {
      case "checkout.session.completed": {
        const session = event.data.object as Stripe.Checkout.Session;
        const userId = session.metadata?.userId;
        if (userId && session.subscription) {
          const sub = await stripe.subscriptions.retrieve(session.subscription as string);
          await supabase.from("profiles").update({
            stripe_subscription_id: sub.id, subscription_status: sub.status,
            plan: sub.items.data[0]?.price?.lookup_key ?? "pro",
            current_period_end: new Date(sub.current_period_end * 1000).toISOString(),
          }).eq("id", userId);
        }
        break;
      }
      case "invoice.paid": {
        const inv = event.data.object as Stripe.Invoice;
        await supabase.from("profiles").update({ subscription_status: "active" }).eq("stripe_customer_id", inv.customer as string);
        break;
      }
      case "invoice.payment_failed": {
        const inv = event.data.object as Stripe.Invoice;
        await supabase.from("profiles").update({ subscription_status: "past_due" }).eq("stripe_customer_id", inv.customer as string);
        break;
      }
      case "customer.subscription.deleted": {
        const sub = event.data.object as Stripe.Subscription;
        await supabase.from("profiles").update({ subscription_status: "canceled", plan: "free" }).eq("stripe_customer_id", sub.customer as string);
        break;
      }
    }
    return NextResponse.json({ received: true });
  } catch (err) {
    console.error(`Webhook error ${event.type}:`, err);
    return NextResponse.json({ error: "Handler failed" }, { status: 500 });
  }
}
```

## Customer Portal

```typescript
// app/api/billing/portal/route.ts
import { NextRequest, NextResponse } from "next/server";
import { stripe } from "@/lib/stripe";
import { createClient } from "@/lib/supabase/server";

export async function POST(request: NextRequest) {
  try {
    const supabase = await createClient();
    const { data: { user } } = await supabase.auth.getUser();
    if (!user) return NextResponse.json({ error: "Not authenticated" }, { status: 401 });

    const { data: profile } = await supabase.from("profiles").select("stripe_customer_id").eq("id", user.id).single();
    if (!profile?.stripe_customer_id) return NextResponse.json({ error: "No billing account" }, { status: 400 });

    const portal = await stripe.billingPortal.sessions.create({
      customer: profile.stripe_customer_id,
      return_url: `${process.env.NEXT_PUBLIC_APP_URL}/dashboard/settings`,
    });
    return NextResponse.json({ url: portal.url });
  } catch (err) {
    console.error("Portal error:", err);
    return NextResponse.json({ error: "Portal failed" }, { status: 500 });
  }
}
```

## Pricing Page

```typescript
// components/pricing-card.tsx
"use client";
import { useState } from "react";

export function PricingCard({ name, price, period, features, priceId, highlighted }: {
  name: string; price: string; period: string; features: string[];
  priceId: string | null; highlighted: boolean;
}) {
  const [loading, setLoading] = useState(false);

  async function handleSubscribe() {
    if (!priceId) { window.location.href = "/signup"; return; }
    setLoading(true);
    try {
      const res = await fetch("/api/checkout/subscription", {
        method: "POST", headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ priceId }),
      });
      const { url, error } = await res.json();
      if (error) throw new Error(error);
      if (url) window.location.href = url;
    } catch { alert("Something went wrong. Please try again."); }
    finally { setLoading(false); }
  }

  return (
    <div className={`rounded-2xl border p-8 ${highlighted ? "border-blue-500 bg-blue-50 shadow-lg" : "border-gray-200"}`}>
      <h2 className="text-xl font-semibold">{name}</h2>
      <div className="mt-4"><span className="text-4xl font-bold">{price}</span><span className="text-gray-500">{period}</span></div>
      <ul className="mt-6 space-y-3">
        {features.map((f) => <li key={f} className="flex items-center gap-2">&#10003; {f}</li>)}
      </ul>
      <button onClick={handleSubscribe} disabled={loading}
        className={`mt-8 w-full rounded-lg px-4 py-3 font-medium ${highlighted ? "bg-blue-600 text-white hover:bg-blue-700" : "bg-gray-100 hover:bg-gray-200"} disabled:opacity-50`}>
        {loading ? "Loading..." : priceId ? "Subscribe" : "Get Started"}
      </button>
    </div>
  );
}
```

## Free Trial Flows

```typescript
// Trial without payment method (lower conversion, friendlier)
subscription_data: { trial_period_days: 14 },
payment_method_collection: "if_required",

// Trial with payment method (higher conversion)
subscription_data: { trial_period_days: 14 },
payment_method_collection: "always",
```

## Australian GST

```typescript
// Set tax_behavior when creating prices:
const price = await stripe.prices.create({
  unit_amount: 2900, currency: "aud", recurring: { interval: "month" },
  product: "prod_xxx", tax_behavior: "inclusive", // GST included in $29
});

// Enable automatic tax in checkout:
const session = await stripe.checkout.sessions.create({
  // ...other config
  automatic_tax: { enabled: true },
  customer_update: { address: "auto" },
});
```

## Statement Descriptor

```typescript
payment_intent_data: {
  statement_descriptor: "MYAPP PRO",      // Max 22 chars, appears on bank statement
  statement_descriptor_suffix: "MONTHLY",
},
```

## Subscription Status in Middleware

```typescript
// middleware.ts — protect pro-only routes
if (request.nextUrl.pathname.startsWith("/dashboard/pro")) {
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) return NextResponse.redirect(new URL("/login", request.url));

  const { data: profile } = await supabase.from("profiles").select("plan, subscription_status").eq("id", user.id).single();
  const hasAccess = profile?.plan !== "free" && ["active", "trialing"].includes(profile?.subscription_status ?? "");
  if (!hasAccess) return NextResponse.redirect(new URL("/pricing", request.url));
}
```

## Supabase Schema

```sql
alter table profiles add column if not exists stripe_customer_id text;
alter table profiles add column if not exists stripe_subscription_id text;
alter table profiles add column if not exists subscription_status text default 'inactive';
alter table profiles add column if not exists plan text default 'free';
alter table profiles add column if not exists current_period_end timestamptz;
```

## Check Subscription Server-Side

```typescript
// lib/subscription.ts
import { createClient } from "@/lib/supabase/server";

export async function getSubscription() {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) return { plan: "free", isActive: false };

  const { data: profile } = await supabase.from("profiles")
    .select("plan, subscription_status").eq("id", user.id).single();

  return {
    plan: profile?.plan ?? "free",
    isActive: profile?.plan !== "free" && ["active", "trialing"].includes(profile?.subscription_status ?? ""),
  };
}
```

See `references/` for detailed patterns:
- `checkout-patterns.md` — One-time, subscription, embedded, custom checkouts
- `webhook-events.md` — All critical events with handler code
- `subscription-lifecycle.md` — Trial, active, past_due, canceled, upgrade/downgrade
