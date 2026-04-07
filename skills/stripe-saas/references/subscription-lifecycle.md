# Subscription Lifecycle

Full lifecycle from trial to cancellation: trial, active, past_due, canceled, upgrade, downgrade, proration, and cancel at period end.

## Lifecycle States

```
trial → active → (past_due → active) → canceled
                       ↓
                   canceled (if payment never recovers)

Upgrade/Downgrade can happen at any "active" or "trialing" state.
```

| Status | Meaning | User Access |
|--------|---------|------------|
| `trialing` | Free trial period, no charge yet | Full access |
| `active` | Paying and current | Full access |
| `past_due` | Payment failed, Stripe is retrying | Grace period (your choice) |
| `canceled` | Subscription ended | Free tier only |
| `incomplete` | Initial payment failed | No access |
| `incomplete_expired` | Initial payment never succeeded | No access |

## Check Subscription Status

```typescript
// lib/subscription.ts
import { createClient } from "@/lib/supabase/server";

export type SubscriptionStatus =
  | "trialing"
  | "active"
  | "past_due"
  | "canceled"
  | "inactive";

export interface SubscriptionInfo {
  plan: string;
  status: SubscriptionStatus;
  isActive: boolean;
  isTrial: boolean;
  isPastDue: boolean;
  cancelAtPeriodEnd: boolean;
  currentPeriodEnd: string | null;
  trialEnd: string | null;
}

export async function getSubscription(): Promise<SubscriptionInfo> {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();

  if (!user) {
    return {
      plan: "free",
      status: "inactive",
      isActive: false,
      isTrial: false,
      isPastDue: false,
      cancelAtPeriodEnd: false,
      currentPeriodEnd: null,
      trialEnd: null,
    };
  }

  const { data: profile } = await supabase
    .from("profiles")
    .select(
      "plan, subscription_status, current_period_end, trial_end, cancel_at_period_end"
    )
    .eq("id", user.id)
    .single();

  const status = (profile?.subscription_status ?? "inactive") as SubscriptionStatus;

  return {
    plan: profile?.plan ?? "free",
    status,
    isActive: ["active", "trialing"].includes(status),
    isTrial: status === "trialing",
    isPastDue: status === "past_due",
    cancelAtPeriodEnd: profile?.cancel_at_period_end ?? false,
    currentPeriodEnd: profile?.current_period_end ?? null,
    trialEnd: profile?.trial_end ?? null,
  };
}
```

## Upgrade Plan

When a user upgrades (e.g., Pro to Enterprise), Stripe prorates automatically.

```typescript
// app/api/billing/upgrade/route.ts
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

    const { newPriceId } = await request.json();
    if (!newPriceId) {
      return NextResponse.json({ error: "New price ID required" }, { status: 400 });
    }

    const { data: profile } = await supabase
      .from("profiles")
      .select("stripe_subscription_id")
      .eq("id", user.id)
      .single();

    if (!profile?.stripe_subscription_id) {
      return NextResponse.json({ error: "No active subscription" }, { status: 400 });
    }

    // Get current subscription
    const subscription = await stripe.subscriptions.retrieve(
      profile.stripe_subscription_id
    );

    // Update to new price with proration
    const updatedSubscription = await stripe.subscriptions.update(
      profile.stripe_subscription_id,
      {
        items: [
          {
            id: subscription.items.data[0]!.id,
            price: newPriceId,
          },
        ],
        proration_behavior: "create_prorations", // Charge the difference immediately
      }
    );

    return NextResponse.json({
      success: true,
      status: updatedSubscription.status,
    });
  } catch (err) {
    console.error("Upgrade error:", err);
    return NextResponse.json({ error: "Upgrade failed" }, { status: 500 });
  }
}
```

## Downgrade Plan

Downgrade takes effect at the end of the current billing period (no refund needed).

```typescript
// app/api/billing/downgrade/route.ts
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

    const { newPriceId } = await request.json();
    if (!newPriceId) {
      return NextResponse.json({ error: "New price ID required" }, { status: 400 });
    }

    const { data: profile } = await supabase
      .from("profiles")
      .select("stripe_subscription_id")
      .eq("id", user.id)
      .single();

    if (!profile?.stripe_subscription_id) {
      return NextResponse.json({ error: "No active subscription" }, { status: 400 });
    }

    const subscription = await stripe.subscriptions.retrieve(
      profile.stripe_subscription_id
    );

    // Downgrade at end of period — no proration, user keeps current plan until renewal
    const updatedSubscription = await stripe.subscriptions.update(
      profile.stripe_subscription_id,
      {
        items: [
          {
            id: subscription.items.data[0]!.id,
            price: newPriceId,
          },
        ],
        proration_behavior: "none", // Don't charge or refund anything now
      }
    );

    return NextResponse.json({
      success: true,
      effectiveDate: new Date(
        updatedSubscription.current_period_end * 1000
      ).toISOString(),
    });
  } catch (err) {
    console.error("Downgrade error:", err);
    return NextResponse.json({ error: "Downgrade failed" }, { status: 500 });
  }
}
```

## Cancel at Period End

Don't cancel immediately. Let the user keep access until their paid period ends.

```typescript
// app/api/billing/cancel/route.ts
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

    const { data: profile } = await supabase
      .from("profiles")
      .select("stripe_subscription_id")
      .eq("id", user.id)
      .single();

    if (!profile?.stripe_subscription_id) {
      return NextResponse.json({ error: "No active subscription" }, { status: 400 });
    }

    // Cancel at end of current period
    const subscription = await stripe.subscriptions.update(
      profile.stripe_subscription_id,
      { cancel_at_period_end: true }
    );

    // Update database
    await supabase
      .from("profiles")
      .update({ cancel_at_period_end: true })
      .eq("id", user.id);

    return NextResponse.json({
      success: true,
      accessUntil: new Date(
        subscription.current_period_end * 1000
      ).toISOString(),
    });
  } catch (err) {
    console.error("Cancel error:", err);
    return NextResponse.json({ error: "Cancel failed" }, { status: 500 });
  }
}
```

## Resume (Undo Cancellation)

If a user changes their mind before the period ends, let them resume.

```typescript
// app/api/billing/resume/route.ts
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

    const { data: profile } = await supabase
      .from("profiles")
      .select("stripe_subscription_id")
      .eq("id", user.id)
      .single();

    if (!profile?.stripe_subscription_id) {
      return NextResponse.json({ error: "No subscription to resume" }, { status: 400 });
    }

    // Remove the scheduled cancellation
    await stripe.subscriptions.update(profile.stripe_subscription_id, {
      cancel_at_period_end: false,
    });

    await supabase
      .from("profiles")
      .update({ cancel_at_period_end: false })
      .eq("id", user.id);

    return NextResponse.json({ success: true });
  } catch (err) {
    console.error("Resume error:", err);
    return NextResponse.json({ error: "Resume failed" }, { status: 500 });
  }
}
```

## Subscription Status Banner Component

Show users their subscription state clearly.

```typescript
// components/subscription-banner.tsx
import { getSubscription } from "@/lib/subscription";

export async function SubscriptionBanner() {
  const sub = await getSubscription();

  if (sub.status === "inactive" || sub.plan === "free") {
    return null;
  }

  // Trial ending soon
  if (sub.isTrial && sub.trialEnd) {
    const daysLeft = Math.ceil(
      (new Date(sub.trialEnd).getTime() - Date.now()) / (1000 * 60 * 60 * 24)
    );

    if (daysLeft <= 3) {
      return (
        <div className="bg-yellow-50 border-b border-yellow-200 px-4 py-2 text-center text-sm text-yellow-800">
          Your trial ends in {daysLeft} day{daysLeft !== 1 ? "s" : ""}.{" "}
          <a href="/dashboard/settings" className="font-medium underline">
            Add a payment method
          </a>{" "}
          to keep your access.
        </div>
      );
    }
  }

  // Past due
  if (sub.isPastDue) {
    return (
      <div className="bg-red-50 border-b border-red-200 px-4 py-2 text-center text-sm text-red-800">
        Your payment failed. Please{" "}
        <a href="/dashboard/settings" className="font-medium underline">
          update your payment method
        </a>{" "}
        to avoid losing access.
      </div>
    );
  }

  // Scheduled to cancel
  if (sub.cancelAtPeriodEnd && sub.currentPeriodEnd) {
    const endDate = new Date(sub.currentPeriodEnd).toLocaleDateString();
    return (
      <div className="bg-orange-50 border-b border-orange-200 px-4 py-2 text-center text-sm text-orange-800">
        Your subscription will end on {endDate}.{" "}
        <form action="/api/billing/resume" method="POST" className="inline">
          <button type="submit" className="font-medium underline">
            Resume subscription
          </button>
        </form>
      </div>
    );
  }

  return null;
}
```

## Database Schema for Full Lifecycle

```sql
-- Add lifecycle columns to profiles
alter table profiles add column if not exists trial_end timestamptz;
alter table profiles add column if not exists cancel_at_period_end boolean default false;
alter table profiles add column if not exists cancel_at timestamptz;

-- Invoices table
create table if not exists invoices (
  id uuid primary key default gen_random_uuid(),
  stripe_invoice_id text unique not null,
  user_id uuid references auth.users(id) on delete cascade,
  amount integer not null,
  currency text not null default 'usd',
  status text not null,
  paid_at timestamptz,
  invoice_url text,
  invoice_pdf text,
  created_at timestamptz default now()
);

alter table invoices enable row level security;
create policy "Users can view own invoices"
  on invoices for select using (auth.uid() = user_id);

-- Payment events table (for tracking failures, retries)
create table if not exists payment_events (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id) on delete cascade,
  event_type text not null,
  stripe_invoice_id text,
  amount integer,
  attempt_count integer,
  created_at timestamptz default now()
);

alter table payment_events enable row level security;
create policy "Users can view own payment events"
  on payment_events for select using (auth.uid() = user_id);
```
