# Supabase Edge Functions

Edge Functions run server-side Deno code close to your users. Use them for webhooks, scheduled tasks, integrations with external APIs, and logic that should not run in the browser.

## Creating an Edge Function

```bash
# Create a new function
npx supabase functions new my-function

# This creates: supabase/functions/my-function/index.ts
```

## Basic Edge Function

```ts
// supabase/functions/hello/index.ts
import { serve } from "https://deno.land/std@0.177.0/http/server.ts";

serve(async (req) => {
  try {
    const { name } = await req.json();

    if (!name || typeof name !== "string") {
      return new Response(
        JSON.stringify({ error: "name is required" }),
        { status: 400, headers: { "Content-Type": "application/json" } }
      );
    }

    return new Response(
      JSON.stringify({ message: `Hello, ${name}!` }),
      { status: 200, headers: { "Content-Type": "application/json" } }
    );
  } catch (err) {
    return new Response(
      JSON.stringify({ error: "Internal server error" }),
      { status: 500, headers: { "Content-Type": "application/json" } }
    );
  }
});
```

## Edge Function with Supabase Client

```ts
// supabase/functions/process-order/index.ts
import { serve } from "https://deno.land/std@0.177.0/http/server.ts";
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

serve(async (req) => {
  try {
    // Get the auth header from the request (passed automatically when using supabase.functions.invoke)
    const authHeader = req.headers.get("Authorization");

    // Create client with service role for admin operations
    const supabaseAdmin = createClient(
      Deno.env.get("SUPABASE_URL")!,
      Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
    );

    // Create client with user's token for user-scoped operations
    const supabaseUser = createClient(
      Deno.env.get("SUPABASE_URL")!,
      Deno.env.get("SUPABASE_ANON_KEY")!,
      {
        global: {
          headers: { Authorization: authHeader! },
        },
      }
    );

    // Get the authenticated user
    const { data: { user }, error: authError } = await supabaseUser.auth.getUser();

    if (authError || !user) {
      return new Response(
        JSON.stringify({ error: "Unauthorized" }),
        { status: 401, headers: { "Content-Type": "application/json" } }
      );
    }

    const { order_id } = await req.json();

    if (!order_id) {
      return new Response(
        JSON.stringify({ error: "order_id is required" }),
        { status: 400, headers: { "Content-Type": "application/json" } }
      );
    }

    // Use admin client for operations that need to bypass RLS
    const { data: order, error } = await supabaseAdmin
      .from("orders")
      .update({ status: "processing", processed_at: new Date().toISOString() })
      .eq("id", order_id)
      .eq("user_id", user.id) // Still verify ownership
      .select()
      .single();

    if (error) {
      return new Response(
        JSON.stringify({ error: "Failed to process order" }),
        { status: 500, headers: { "Content-Type": "application/json" } }
      );
    }

    return new Response(
      JSON.stringify({ data: order }),
      { status: 200, headers: { "Content-Type": "application/json" } }
    );
  } catch (err) {
    return new Response(
      JSON.stringify({ error: "Internal server error" }),
      { status: 500, headers: { "Content-Type": "application/json" } }
    );
  }
});
```

## Calling Edge Functions from Next.js

### From a Server Component or Server Action

```ts
// app/actions/orders.ts
"use server";

import { createClient } from "@/lib/supabase/server";

export async function processOrder(orderId: string) {
  const supabase = await createClient();

  const { data, error } = await supabase.functions.invoke("process-order", {
    body: { order_id: orderId },
  });

  if (error) {
    return { error: "Failed to process order" };
  }

  return { data };
}
```

### From a Client Component

```tsx
// app/components/process-button.tsx
"use client";

import { useState } from "react";
import { createClient } from "@/lib/supabase/client";

export function ProcessButton({ orderId }: { orderId: string }) {
  const [loading, setLoading] = useState(false);

  async function handleProcess() {
    setLoading(true);
    const supabase = createClient();

    const { data, error } = await supabase.functions.invoke("process-order", {
      body: { order_id: orderId },
    });

    if (error) {
      alert("Failed to process order");
    } else {
      alert("Order processed!");
    }

    setLoading(false);
  }

  return (
    <button
      onClick={handleProcess}
      disabled={loading}
      className="rounded bg-green-500 px-4 py-2 text-white disabled:opacity-50"
    >
      {loading ? "Processing..." : "Process Order"}
    </button>
  );
}
```

## Webhook Handler

```ts
// supabase/functions/stripe-webhook/index.ts
import { serve } from "https://deno.land/std@0.177.0/http/server.ts";
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";
import Stripe from "https://esm.sh/stripe@13?target=deno";

const stripe = new Stripe(Deno.env.get("STRIPE_SECRET_KEY")!, {
  apiVersion: "2023-10-16",
  httpClient: Stripe.createFetchHttpClient(),
});

const webhookSecret = Deno.env.get("STRIPE_WEBHOOK_SECRET")!;

serve(async (req) => {
  const body = await req.text();
  const signature = req.headers.get("stripe-signature");

  if (!signature) {
    return new Response("Missing signature", { status: 400 });
  }

  let event: Stripe.Event;
  try {
    event = await stripe.webhooks.constructEventAsync(
      body,
      signature,
      webhookSecret
    );
  } catch (err) {
    return new Response(`Webhook signature verification failed`, { status: 400 });
  }

  const supabase = createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
  );

  switch (event.type) {
    case "checkout.session.completed": {
      const session = event.data.object as Stripe.Checkout.Session;
      const { error } = await supabase
        .from("subscriptions")
        .update({
          status: "active",
          stripe_subscription_id: session.subscription as string,
        })
        .eq("stripe_customer_id", session.customer as string);

      if (error) {
        console.error("Failed to update subscription:", error);
        return new Response("Database error", { status: 500 });
      }
      break;
    }

    case "customer.subscription.deleted": {
      const subscription = event.data.object as Stripe.Subscription;
      await supabase
        .from("subscriptions")
        .update({ status: "canceled" })
        .eq("stripe_subscription_id", subscription.id);
      break;
    }

    default:
      console.log(`Unhandled event type: ${event.type}`);
  }

  return new Response(JSON.stringify({ received: true }), {
    headers: { "Content-Type": "application/json" },
  });
});
```

## Calling External APIs (Gemini)

```ts
// supabase/functions/generate-summary/index.ts
import { serve } from "https://deno.land/std@0.177.0/http/server.ts";
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

serve(async (req) => {
  try {
    const { post_id } = await req.json();

    if (!post_id) {
      return new Response(
        JSON.stringify({ error: "post_id required" }),
        { status: 400, headers: { "Content-Type": "application/json" } }
      );
    }

    const supabase = createClient(
      Deno.env.get("SUPABASE_URL")!,
      Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
    );

    const { data: post, error } = await supabase
      .from("posts")
      .select("title, body")
      .eq("id", post_id)
      .single();

    if (error || !post) {
      return new Response(
        JSON.stringify({ error: "Post not found" }),
        { status: 404, headers: { "Content-Type": "application/json" } }
      );
    }

    // Call Gemini API
    const geminiKey = Deno.env.get("GEMINI_API_KEY");
    if (!geminiKey) {
      return new Response(
        JSON.stringify({ error: "Gemini API key not configured" }),
        { status: 500, headers: { "Content-Type": "application/json" } }
      );
    }

    const geminiResponse = await fetch(
      `https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent?key=${geminiKey}`,
      {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          contents: [
            {
              parts: [
                {
                  text: `Summarize this blog post in 2-3 sentences:\n\nTitle: ${post.title}\n\n${post.body}`,
                },
              ],
            },
          ],
        }),
      }
    );

    if (!geminiResponse.ok) {
      return new Response(
        JSON.stringify({ error: "Gemini API call failed" }),
        { status: 502, headers: { "Content-Type": "application/json" } }
      );
    }

    const geminiData = await geminiResponse.json();
    const summary = geminiData.candidates?.[0]?.content?.parts?.[0]?.text ?? "";

    // Save summary
    await supabase
      .from("posts")
      .update({ excerpt: summary })
      .eq("id", post_id);

    return new Response(
      JSON.stringify({ summary }),
      { status: 200, headers: { "Content-Type": "application/json" } }
    );
  } catch (err) {
    return new Response(
      JSON.stringify({ error: "Internal error" }),
      { status: 500, headers: { "Content-Type": "application/json" } }
    );
  }
});
```

## CORS Handling

If you call edge functions directly from the browser (not via supabase.functions.invoke), you need CORS headers:

```ts
// supabase/functions/_shared/cors.ts
export const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Headers": "authorization, x-client-info, apikey, content-type",
};
```

```ts
// supabase/functions/my-function/index.ts
import { serve } from "https://deno.land/std@0.177.0/http/server.ts";
import { corsHeaders } from "../_shared/cors.ts";

serve(async (req) => {
  // Handle CORS preflight
  if (req.method === "OPTIONS") {
    return new Response("ok", { headers: corsHeaders });
  }

  try {
    const data = { message: "Hello" };

    return new Response(JSON.stringify(data), {
      headers: { ...corsHeaders, "Content-Type": "application/json" },
    });
  } catch (err) {
    return new Response(JSON.stringify({ error: "Internal error" }), {
      status: 500,
      headers: { ...corsHeaders, "Content-Type": "application/json" },
    });
  }
});
```

## Deploying Edge Functions

```bash
# Deploy a specific function
npx supabase functions deploy my-function

# Deploy all functions
npx supabase functions deploy

# Set secrets for your function
npx supabase secrets set GEMINI_API_KEY=AIza...
npx supabase secrets set STRIPE_SECRET_KEY=sk_live_...

# List secrets
npx supabase secrets list

# Run locally for testing
npx supabase functions serve my-function
```

## Testing Locally

```bash
# Start Supabase locally
npx supabase start

# Serve functions locally
npx supabase functions serve

# Test with curl
curl -X POST http://localhost:54321/functions/v1/hello \
  -H "Authorization: Bearer YOUR_ANON_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "World"}'
```
