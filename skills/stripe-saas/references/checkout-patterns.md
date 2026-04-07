# Checkout Patterns

All checkout variations: one-time, subscription, embedded, and custom with Elements.

## One-Time Payment Checkout

Redirect users to Stripe's hosted checkout page for a single purchase.

```typescript
// app/api/checkout/one-time/route.ts
import { NextRequest, NextResponse } from "next/server";
import { stripe } from "@/lib/stripe";
import { createClient } from "@/lib/supabase/server";

export async function POST(request: NextRequest) {
  try {
    const supabase = await createClient();
    const { data: { user } } = await supabase.auth.getUser();

    if (!user) {
      return NextResponse.json({ error: "Not authenticated" }, { status: 401 });
    }

    const { priceId, quantity } = await request.json();
    if (!priceId) {
      return NextResponse.json({ error: "Price ID required" }, { status: 400 });
    }

    const session = await stripe.checkout.sessions.create({
      customer_email: user.email,
      line_items: [
        {
          price: priceId,
          quantity: quantity ?? 1,
        },
      ],
      mode: "payment",
      success_url: `${process.env.NEXT_PUBLIC_APP_URL}/success?session_id={CHECKOUT_SESSION_ID}`,
      cancel_url: `${process.env.NEXT_PUBLIC_APP_URL}/pricing`,
      metadata: {
        userId: user.id,
        type: "one-time",
      },
      // Show what appears on bank statement
      payment_intent_data: {
        statement_descriptor: "MYAPP PURCHASE",
      },
    });

    return NextResponse.json({ url: session.url });
  } catch (err) {
    console.error("One-time checkout error:", err);
    return NextResponse.json({ error: "Checkout failed" }, { status: 500 });
  }
}
```

## Subscription Checkout

```typescript
// app/api/checkout/subscription/route.ts
import { NextRequest, NextResponse } from "next/server";
import { stripe } from "@/lib/stripe";
import { createClient } from "@/lib/supabase/server";

export async function POST(request: NextRequest) {
  try {
    const supabase = await createClient();
    const { data: { user } } = await supabase.auth.getUser();

    if (!user) {
      return NextResponse.json({ error: "Not authenticated" }, { status: 401 });
    }

    const { priceId, trialDays, couponId } = await request.json();
    if (!priceId) {
      return NextResponse.json({ error: "Price ID required" }, { status: 400 });
    }

    // Get or create Stripe customer
    const { data: profile } = await supabase
      .from("profiles")
      .select("stripe_customer_id")
      .eq("id", user.id)
      .single();

    let customerId = profile?.stripe_customer_id;

    if (!customerId) {
      const customer = await stripe.customers.create({
        email: user.email!,
        metadata: { supabase_user_id: user.id },
      });
      customerId = customer.id;

      await supabase
        .from("profiles")
        .update({ stripe_customer_id: customerId })
        .eq("id", user.id);
    }

    // Check for existing active subscription
    const existingSubscriptions = await stripe.subscriptions.list({
      customer: customerId,
      status: "active",
      limit: 1,
    });

    if (existingSubscriptions.data.length > 0) {
      // Redirect to customer portal instead
      const portalSession = await stripe.billingPortal.sessions.create({
        customer: customerId,
        return_url: `${process.env.NEXT_PUBLIC_APP_URL}/dashboard/settings`,
      });
      return NextResponse.json({ url: portalSession.url });
    }

    const sessionConfig: Record<string, unknown> = {
      customer: customerId,
      line_items: [{ price: priceId, quantity: 1 }],
      mode: "subscription",
      success_url: `${process.env.NEXT_PUBLIC_APP_URL}/dashboard?checkout=success`,
      cancel_url: `${process.env.NEXT_PUBLIC_APP_URL}/pricing`,
      metadata: { userId: user.id },
      allow_promotion_codes: true,
    };

    if (trialDays) {
      sessionConfig.subscription_data = {
        trial_period_days: trialDays,
      };
    }

    if (couponId) {
      sessionConfig.discounts = [{ coupon: couponId }];
      delete sessionConfig.allow_promotion_codes;
    }

    const session = await stripe.checkout.sessions.create(
      sessionConfig as Parameters<typeof stripe.checkout.sessions.create>[0]
    );

    return NextResponse.json({ url: session.url });
  } catch (err) {
    console.error("Subscription checkout error:", err);
    return NextResponse.json({ error: "Checkout failed" }, { status: 500 });
  }
}
```

## Embedded Checkout

Embed Stripe Checkout directly in your page instead of redirecting.

```typescript
// app/api/checkout/embedded/route.ts
import { NextRequest, NextResponse } from "next/server";
import { stripe } from "@/lib/stripe";
import { createClient } from "@/lib/supabase/server";

export async function POST(request: NextRequest) {
  try {
    const supabase = await createClient();
    const { data: { user } } = await supabase.auth.getUser();

    if (!user) {
      return NextResponse.json({ error: "Not authenticated" }, { status: 401 });
    }

    const { priceId } = await request.json();

    const session = await stripe.checkout.sessions.create({
      customer_email: user.email,
      line_items: [{ price: priceId, quantity: 1 }],
      mode: "subscription",
      ui_mode: "embedded",
      return_url: `${process.env.NEXT_PUBLIC_APP_URL}/checkout/return?session_id={CHECKOUT_SESSION_ID}`,
      metadata: { userId: user.id },
    });

    return NextResponse.json({ clientSecret: session.client_secret });
  } catch (err) {
    console.error("Embedded checkout error:", err);
    return NextResponse.json({ error: "Checkout failed" }, { status: 500 });
  }
}
```

```typescript
// components/embedded-checkout.tsx
"use client";
import { useCallback, useState } from "react";
import {
  EmbeddedCheckoutProvider,
  EmbeddedCheckout,
} from "@stripe/react-stripe-js";
import { stripePromise } from "@/lib/stripe-client";

export function EmbeddedCheckoutForm({ priceId }: { priceId: string }) {
  const [error, setError] = useState<string | null>(null);

  const fetchClientSecret = useCallback(async () => {
    try {
      const res = await fetch("/api/checkout/embedded", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ priceId }),
      });

      const data = await res.json();
      if (data.error) throw new Error(data.error);
      return data.clientSecret;
    } catch (err) {
      setError(err instanceof Error ? err.message : "Checkout failed");
      throw err;
    }
  }, [priceId]);

  if (error) {
    return <div className="text-red-600">Error: {error}</div>;
  }

  return (
    <EmbeddedCheckoutProvider
      stripe={stripePromise}
      options={{ fetchClientSecret }}
    >
      <EmbeddedCheckout />
    </EmbeddedCheckoutProvider>
  );
}
```

## Checkout Return Page (for Embedded Checkout)

```typescript
// app/checkout/return/page.tsx
import { stripe } from "@/lib/stripe";
import { redirect } from "next/navigation";

export default async function CheckoutReturn({
  searchParams,
}: {
  searchParams: Promise<{ session_id?: string }>;
}) {
  const { session_id } = await searchParams;

  if (!session_id) {
    redirect("/pricing");
  }

  const session = await stripe.checkout.sessions.retrieve(session_id);

  if (session.status === "complete") {
    return (
      <div className="mx-auto max-w-lg py-16 text-center">
        <h1 className="text-3xl font-bold text-green-600">Payment Successful!</h1>
        <p className="mt-4 text-gray-600">
          Thank you for your purchase. You can now access all features.
        </p>
        <a
          href="/dashboard"
          className="mt-8 inline-block rounded-lg bg-blue-600 px-6 py-3 text-white"
        >
          Go to Dashboard
        </a>
      </div>
    );
  }

  return (
    <div className="mx-auto max-w-lg py-16 text-center">
      <h1 className="text-3xl font-bold text-red-600">Payment Failed</h1>
      <p className="mt-4 text-gray-600">
        Something went wrong. Please try again.
      </p>
      <a
        href="/pricing"
        className="mt-8 inline-block rounded-lg bg-gray-200 px-6 py-3"
      >
        Back to Pricing
      </a>
    </div>
  );
}
```

## Checkout Button Component

Reusable button that works with any checkout type.

```typescript
// components/checkout-button.tsx
"use client";
import { useState } from "react";

interface CheckoutButtonProps {
  priceId: string;
  mode: "payment" | "subscription";
  trialDays?: number;
  children: React.ReactNode;
  className?: string;
}

export function CheckoutButton({
  priceId,
  mode,
  trialDays,
  children,
  className = "",
}: CheckoutButtonProps) {
  const [loading, setLoading] = useState(false);

  async function handleClick() {
    setLoading(true);
    try {
      const endpoint =
        mode === "subscription"
          ? "/api/checkout/subscription"
          : "/api/checkout/one-time";

      const res = await fetch(endpoint, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ priceId, trialDays }),
      });

      const { url, error } = await res.json();
      if (error) throw new Error(error);
      if (url) window.location.href = url;
    } catch (err) {
      console.error("Checkout error:", err);
      alert("Checkout failed. Please try again.");
    } finally {
      setLoading(false);
    }
  }

  return (
    <button onClick={handleClick} disabled={loading} className={className}>
      {loading ? "Loading..." : children}
    </button>
  );
}
```
