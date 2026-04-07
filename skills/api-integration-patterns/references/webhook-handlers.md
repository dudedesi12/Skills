# Webhook Handler Patterns

Production-ready webhook handlers for Stripe, Supabase, and custom webhooks with signature verification.

## Generic Webhook Verifier

```typescript
// lib/webhooks/verify.ts
import crypto from "crypto";

export function verifyHmacSignature(
  payload: string,
  signature: string,
  secret: string,
  algorithm: string = "sha256"
): boolean {
  try {
    const expected = crypto
      .createHmac(algorithm, secret)
      .update(payload, "utf8")
      .digest("hex");

    return crypto.timingSafeEqual(
      Buffer.from(signature),
      Buffer.from(expected)
    );
  } catch {
    return false;
  }
}
```

## Stripe Webhook Handler

```typescript
// app/api/webhooks/stripe/route.ts
import { NextRequest, NextResponse } from "next/server";
import Stripe from "stripe";

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: "2024-12-18.acacia",
});

type EventHandler = (data: Stripe.Event.Data.Object) => Promise<void>;

const handlers: Record<string, EventHandler> = {
  "checkout.session.completed": async (data) => {
    const session = data as Stripe.Checkout.Session;
    console.log("Checkout completed:", session.id);
    // Update user subscription in database
  },
  "invoice.paid": async (data) => {
    const invoice = data as Stripe.Invoice;
    console.log("Invoice paid:", invoice.id);
    // Mark subscription as active
  },
  "invoice.payment_failed": async (data) => {
    const invoice = data as Stripe.Invoice;
    console.log("Payment failed:", invoice.id);
    // Mark subscription as past_due, notify user
  },
  "customer.subscription.updated": async (data) => {
    const subscription = data as Stripe.Subscription;
    console.log("Subscription updated:", subscription.id);
    // Sync plan changes to database
  },
  "customer.subscription.deleted": async (data) => {
    const subscription = data as Stripe.Subscription;
    console.log("Subscription canceled:", subscription.id);
    // Downgrade user to free plan
  },
};

export async function POST(request: NextRequest) {
  const body = await request.text();
  const signature = request.headers.get("stripe-signature");

  if (!signature) {
    return NextResponse.json({ error: "Missing signature" }, { status: 400 });
  }

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(
      body,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET!
    );
  } catch (err) {
    const message = err instanceof Error ? err.message : "Unknown error";
    console.error("Stripe signature verification failed:", message);
    return NextResponse.json({ error: "Invalid signature" }, { status: 400 });
  }

  const handler = handlers[event.type];
  if (handler) {
    try {
      await handler(event.data.object);
    } catch (err) {
      console.error(`Handler error for ${event.type}:`, err);
      return NextResponse.json({ error: "Handler failed" }, { status: 500 });
    }
  } else {
    console.log(`No handler for event type: ${event.type}`);
  }

  return NextResponse.json({ received: true });
}
```

## Supabase Database Webhook

Set up in Supabase Dashboard > Database > Webhooks.

```typescript
// app/api/webhooks/supabase/route.ts
import { NextRequest, NextResponse } from "next/server";
import { verifyHmacSignature } from "@/lib/webhooks/verify";

interface SupabaseWebhookPayload {
  type: "INSERT" | "UPDATE" | "DELETE";
  table: string;
  schema: string;
  record: Record<string, unknown> | null;
  old_record: Record<string, unknown> | null;
}

export async function POST(request: NextRequest) {
  const body = await request.text();
  const signature = request.headers.get("x-supabase-signature");

  if (!signature) {
    return NextResponse.json({ error: "Missing signature" }, { status: 400 });
  }

  const isValid = verifyHmacSignature(
    body,
    signature,
    process.env.SUPABASE_WEBHOOK_SECRET!
  );

  if (!isValid) {
    return NextResponse.json({ error: "Invalid signature" }, { status: 400 });
  }

  const payload: SupabaseWebhookPayload = JSON.parse(body);

  try {
    switch (payload.table) {
      case "profiles":
        if (payload.type === "UPDATE" && payload.record) {
          console.log("Profile updated:", payload.record.id);
        }
        break;
      case "orders":
        if (payload.type === "INSERT" && payload.record) {
          console.log("New order:", payload.record.id);
          // Send confirmation email, update inventory, etc.
        }
        break;
      default:
        console.log(`Unhandled table webhook: ${payload.table}`);
    }

    return NextResponse.json({ received: true });
  } catch (err) {
    console.error("Supabase webhook error:", err);
    return NextResponse.json({ error: "Handler failed" }, { status: 500 });
  }
}
```

## Custom Webhook Receiver

For any service that sends webhooks with HMAC signatures.

```typescript
// app/api/webhooks/custom/route.ts
import { NextRequest, NextResponse } from "next/server";
import { verifyHmacSignature } from "@/lib/webhooks/verify";

export async function POST(request: NextRequest) {
  const body = await request.text();

  // Different services use different header names
  const signature =
    request.headers.get("x-webhook-signature") ??
    request.headers.get("x-signature") ??
    request.headers.get("x-hub-signature-256");

  if (!signature) {
    return NextResponse.json({ error: "Missing signature" }, { status: 400 });
  }

  // Some services prefix with "sha256="
  const cleanSignature = signature.replace("sha256=", "");

  const isValid = verifyHmacSignature(
    body,
    cleanSignature,
    process.env.CUSTOM_WEBHOOK_SECRET!
  );

  if (!isValid) {
    return NextResponse.json({ error: "Invalid signature" }, { status: 400 });
  }

  const payload = JSON.parse(body);

  try {
    // Process the webhook event
    console.log("Received custom webhook:", payload.event);

    // Respond quickly — do heavy work asynchronously
    return NextResponse.json({ received: true });
  } catch (err) {
    console.error("Custom webhook error:", err);
    return NextResponse.json({ error: "Handler failed" }, { status: 500 });
  }
}
```

## Webhook Sender (Sending Webhooks FROM Your App)

```typescript
// lib/webhooks/send.ts
import crypto from "crypto";

interface WebhookDelivery {
  url: string;
  event: string;
  payload: Record<string, unknown>;
  secret: string;
}

export async function sendWebhook({
  url,
  event,
  payload,
  secret,
}: WebhookDelivery): Promise<boolean> {
  const body = JSON.stringify({ event, data: payload, timestamp: Date.now() });

  const signature = crypto
    .createHmac("sha256", secret)
    .update(body)
    .digest("hex");

  try {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "X-Webhook-Signature": signature,
        "X-Webhook-Event": event,
      },
      body,
      signal: AbortSignal.timeout(10000),
    });

    return response.ok;
  } catch (err) {
    console.error(`Webhook delivery failed to ${url}:`, err);
    return false;
  }
}
```

## Testing Webhooks Locally

```bash
# Install Stripe CLI
brew install stripe/stripe-cli/stripe

# Forward Stripe webhooks to your local server
stripe listen --forward-to localhost:3000/api/webhooks/stripe

# Trigger a test event
stripe trigger checkout.session.completed
```
