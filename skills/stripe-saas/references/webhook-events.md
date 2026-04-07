# Stripe Webhook Events

All critical webhook events with handler code for syncing Stripe with Supabase.

## Webhook Route Setup

```typescript
// app/api/webhooks/stripe/route.ts
import { NextRequest, NextResponse } from "next/server";
import { stripe } from "@/lib/stripe";
import { createClient } from "@supabase/supabase-js";
import Stripe from "stripe";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export async function POST(request: NextRequest) {
  const body = await request.text();
  const signature = request.headers.get("stripe-signature");

  if (!signature) {
    return NextResponse.json({ error: "No signature" }, { status: 400 });
  }

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(
      body,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET!
    );
  } catch (err) {
    console.error("Stripe webhook signature failed:", err);
    return NextResponse.json({ error: "Invalid signature" }, { status: 400 });
  }

  try {
    await handleEvent(event);
    return NextResponse.json({ received: true });
  } catch (err) {
    console.error(`Webhook error for ${event.type}:`, err);
    return NextResponse.json({ error: "Handler failed" }, { status: 500 });
  }
}

async function handleEvent(event: Stripe.Event) {
  switch (event.type) {
    case "checkout.session.completed":
      await onCheckoutCompleted(event.data.object as Stripe.Checkout.Session);
      break;
    case "invoice.paid":
      await onInvoicePaid(event.data.object as Stripe.Invoice);
      break;
    case "invoice.payment_failed":
      await onInvoicePaymentFailed(event.data.object as Stripe.Invoice);
      break;
    case "customer.subscription.updated":
      await onSubscriptionUpdated(event.data.object as Stripe.Subscription);
      break;
    case "customer.subscription.deleted":
      await onSubscriptionDeleted(event.data.object as Stripe.Subscription);
      break;
    case "customer.subscription.trial_will_end":
      await onTrialWillEnd(event.data.object as Stripe.Subscription);
      break;
    default:
      console.log(`Unhandled Stripe event: ${event.type}`);
  }
}
```

## Event: checkout.session.completed

Fires when a customer completes a Checkout Session. This is your main "purchase successful" event.

```typescript
async function onCheckoutCompleted(session: Stripe.Checkout.Session) {
  const userId = session.metadata?.userId;
  if (!userId) {
    console.error("No userId in checkout session metadata");
    return;
  }

  // Subscription checkout
  if (session.mode === "subscription" && session.subscription) {
    const subscription = await stripe.subscriptions.retrieve(
      session.subscription as string
    );

    const priceId = subscription.items.data[0]?.price?.id;
    const lookupKey = subscription.items.data[0]?.price?.lookup_key;

    await supabase
      .from("profiles")
      .update({
        stripe_customer_id: session.customer as string,
        stripe_subscription_id: subscription.id,
        subscription_status: subscription.status,
        plan: lookupKey ?? getPlanFromPriceId(priceId ?? ""),
        current_period_end: new Date(
          subscription.current_period_end * 1000
        ).toISOString(),
        trial_end: subscription.trial_end
          ? new Date(subscription.trial_end * 1000).toISOString()
          : null,
      })
      .eq("id", userId);

    console.log(`Subscription created for user ${userId}: ${subscription.id}`);
  }

  // One-time payment
  if (session.mode === "payment") {
    await supabase.from("purchases").insert({
      user_id: userId,
      stripe_session_id: session.id,
      stripe_payment_intent: session.payment_intent as string,
      amount: session.amount_total,
      currency: session.currency,
      status: "completed",
    });

    console.log(`One-time payment recorded for user ${userId}`);
  }
}

function getPlanFromPriceId(priceId: string): string {
  const planMap: Record<string, string> = {
    [process.env.STRIPE_PRO_MONTHLY_PRICE_ID ?? ""]: "pro",
    [process.env.STRIPE_PRO_YEARLY_PRICE_ID ?? ""]: "pro",
    [process.env.STRIPE_ENTERPRISE_MONTHLY_PRICE_ID ?? ""]: "enterprise",
    [process.env.STRIPE_ENTERPRISE_YEARLY_PRICE_ID ?? ""]: "enterprise",
  };
  return planMap[priceId] ?? "pro";
}
```

## Event: invoice.paid

Fires when an invoice is successfully paid. Happens on each billing cycle.

```typescript
async function onInvoicePaid(invoice: Stripe.Invoice) {
  const customerId = invoice.customer as string;

  const { data: profile } = await supabase
    .from("profiles")
    .select("id")
    .eq("stripe_customer_id", customerId)
    .single();

  if (!profile) {
    console.error(`No profile found for Stripe customer ${customerId}`);
    return;
  }

  // Ensure subscription is marked active
  await supabase
    .from("profiles")
    .update({ subscription_status: "active" })
    .eq("id", profile.id);

  // Record the invoice
  await supabase.from("invoices").upsert({
    stripe_invoice_id: invoice.id,
    user_id: profile.id,
    amount: invoice.amount_paid,
    currency: invoice.currency,
    status: "paid",
    paid_at: new Date((invoice.status_transitions?.paid_at ?? 0) * 1000).toISOString(),
    invoice_url: invoice.hosted_invoice_url,
    invoice_pdf: invoice.invoice_pdf,
  });

  console.log(`Invoice paid for user ${profile.id}: ${invoice.id}`);
}
```

## Event: invoice.payment_failed

Fires when a payment attempt fails (expired card, insufficient funds, etc.).

```typescript
async function onInvoicePaymentFailed(invoice: Stripe.Invoice) {
  const customerId = invoice.customer as string;

  const { data: profile } = await supabase
    .from("profiles")
    .select("id, email")
    .eq("stripe_customer_id", customerId)
    .single();

  if (!profile) return;

  // Mark subscription as past_due
  await supabase
    .from("profiles")
    .update({ subscription_status: "past_due" })
    .eq("id", profile.id);

  // Record failed payment attempt
  await supabase.from("payment_events").insert({
    user_id: profile.id,
    event_type: "payment_failed",
    stripe_invoice_id: invoice.id,
    amount: invoice.amount_due,
    attempt_count: invoice.attempt_count,
  });

  console.log(
    `Payment failed for user ${profile.id}, attempt ${invoice.attempt_count}`
  );

  // Send notification to user (email, in-app, etc.)
  // await sendPaymentFailedEmail(profile.email, invoice.hosted_invoice_url);
}
```

## Event: customer.subscription.updated

Fires when a subscription changes — plan upgrade/downgrade, status change, cancellation scheduled.

```typescript
async function onSubscriptionUpdated(subscription: Stripe.Subscription) {
  const customerId = subscription.customer as string;

  const { data: profile } = await supabase
    .from("profiles")
    .select("id, plan")
    .eq("stripe_customer_id", customerId)
    .single();

  if (!profile) return;

  const priceId = subscription.items.data[0]?.price?.id;
  const lookupKey = subscription.items.data[0]?.price?.lookup_key;
  const newPlan = lookupKey ?? getPlanFromPriceId(priceId ?? "");

  const updateData: Record<string, unknown> = {
    subscription_status: subscription.status,
    plan: newPlan,
    current_period_end: new Date(
      subscription.current_period_end * 1000
    ).toISOString(),
    cancel_at_period_end: subscription.cancel_at_period_end,
  };

  if (subscription.cancel_at) {
    updateData.cancel_at = new Date(subscription.cancel_at * 1000).toISOString();
  }

  await supabase.from("profiles").update(updateData).eq("id", profile.id);

  // Log the change
  if (profile.plan !== newPlan) {
    console.log(
      `Plan changed for user ${profile.id}: ${profile.plan} → ${newPlan}`
    );
  }

  if (subscription.cancel_at_period_end) {
    console.log(
      `Subscription scheduled to cancel for user ${profile.id} at period end`
    );
  }
}
```

## Event: customer.subscription.deleted

Fires when a subscription is fully canceled (not just scheduled to cancel).

```typescript
async function onSubscriptionDeleted(subscription: Stripe.Subscription) {
  const customerId = subscription.customer as string;

  const { data: profile } = await supabase
    .from("profiles")
    .select("id")
    .eq("stripe_customer_id", customerId)
    .single();

  if (!profile) return;

  await supabase
    .from("profiles")
    .update({
      subscription_status: "canceled",
      plan: "free",
      stripe_subscription_id: null,
      current_period_end: null,
      cancel_at_period_end: false,
      cancel_at: null,
    })
    .eq("id", profile.id);

  console.log(`Subscription deleted for user ${profile.id}`);

  // Optionally record cancellation reason
  // await supabase.from("cancellations").insert({ ... });
}
```

## Event: customer.subscription.trial_will_end

Fires 3 days before a trial ends. Good time to remind users to add a payment method.

```typescript
async function onTrialWillEnd(subscription: Stripe.Subscription) {
  const customerId = subscription.customer as string;

  const { data: profile } = await supabase
    .from("profiles")
    .select("id, email")
    .eq("stripe_customer_id", customerId)
    .single();

  if (!profile) return;

  console.log(`Trial ending soon for user ${profile.id}`);

  // Send trial ending notification
  // await sendTrialEndingEmail(profile.email);
}
```

## Stripe CLI for Local Testing

```bash
# Listen for webhook events locally
stripe listen --forward-to localhost:3000/api/webhooks/stripe

# Trigger specific test events
stripe trigger checkout.session.completed
stripe trigger invoice.paid
stripe trigger invoice.payment_failed
stripe trigger customer.subscription.updated
stripe trigger customer.subscription.deleted
```

## Required Webhook Events in Stripe Dashboard

Go to Stripe Dashboard > Developers > Webhooks > Add Endpoint and select these events:

1. `checkout.session.completed`
2. `invoice.paid`
3. `invoice.payment_failed`
4. `customer.subscription.created`
5. `customer.subscription.updated`
6. `customer.subscription.deleted`
7. `customer.subscription.trial_will_end`
