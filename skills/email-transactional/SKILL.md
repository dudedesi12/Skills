---
name: "Email Transactional"
description: "Use this skill whenever the user mentions email, send email, email template, welcome email, verification email, password reset email, notification email, Resend, React Email, transactional email, email queue, newsletter, email verification, 'notify users', 'send confirmation', 'email when', drip campaign, Australian Spam Act, unsubscribe, or ANY email communication task — even if they don't explicitly say 'email'. This skill handles all user communication via email."
---

# Transactional Email with Resend and React Email

Resend is an email sending service. React Email lets you build email templates using React components. Together they make sending beautiful, reliable emails simple.

## Resend Setup with Next.js

### Install Dependencies

```bash
npm install resend @react-email/components
npm install -D react-email
```

### Get Your API Key

1. Go to [resend.com](https://resend.com) and create an account
2. Go to **API Keys** and create a new key
3. Add it to `.env.local`:

```bash
# .env.local
RESEND_API_KEY=re_your_api_key_here
RESEND_FROM_EMAIL="Your App <noreply@yourdomain.com>"
```

To send from your own domain, verify it in Resend's dashboard under **Domains**. See `references/resend-setup.md` for full DNS setup instructions.

### Create the Email Client

```typescript
// lib/email.ts
import { Resend } from "resend";

if (!process.env.RESEND_API_KEY) {
  throw new Error("RESEND_API_KEY environment variable is not set");
}

export const resend = new Resend(process.env.RESEND_API_KEY);

export const FROM_EMAIL = process.env.RESEND_FROM_EMAIL || "onboarding@resend.dev";

export async function sendEmail({
  to,
  subject,
  react,
}: {
  to: string;
  subject: string;
  react: React.ReactElement;
}) {
  try {
    const { data, error } = await resend.emails.send({
      from: FROM_EMAIL,
      to,
      subject,
      react,
    });

    if (error) {
      console.error("Failed to send email:", error);
      throw new Error(`Email send failed: ${error.message}`);
    }

    return data;
  } catch (err) {
    console.error("Email service error:", err);
    throw err;
  }
}
```

## React Email Templates

React Email lets you write emails like React components. Here is a Welcome email example. See `references/email-templates.md` for 8 complete templates (Welcome, Verify, Password Reset, Order Confirmation, Subscription Changed, Weekly Digest, Alert, Invoice).

```tsx
// emails/welcome.tsx
import {
  Html, Head, Body, Container, Section,
  Text, Button, Heading, Hr, Preview,
} from "@react-email/components";

interface WelcomeEmailProps {
  userName: string;
  dashboardUrl: string;
}

export default function WelcomeEmail({ userName, dashboardUrl }: WelcomeEmailProps) {
  return (
    <Html>
      <Head />
      <Preview>Welcome to our platform, {userName}!</Preview>
      <Body style={{ backgroundColor: "#f6f9fc", fontFamily: "Arial, sans-serif" }}>
        <Container style={{ margin: "0 auto", padding: "40px 20px", maxWidth: "560px" }}>
          <Heading style={{ fontSize: "24px", color: "#1a1a1a" }}>Welcome, {userName}!</Heading>
          <Text style={{ fontSize: "16px", color: "#4a4a4a", lineHeight: "26px" }}>
            Thanks for signing up. Your account is ready to go.
          </Text>
          <Section style={{ textAlign: "center" as const, margin: "24px 0" }}>
            <Button href={dashboardUrl} style={{ backgroundColor: "#2563eb", borderRadius: "6px", color: "#fff", fontSize: "16px", padding: "12px 24px", textDecoration: "none" }}>
              Go to Dashboard
            </Button>
          </Section>
          <Hr style={{ borderColor: "#e5e7eb", margin: "24px 0" }} />
          <Text style={{ fontSize: "12px", color: "#9ca3af" }}>
            You received this because you signed up. If this was not you, ignore this email.
          </Text>
        </Container>
      </Body>
    </Html>
  );
}
```

## Transactional Email Flows

Here is how to wire up the signup-to-welcome flow:

### API Route: Send Verification Email

```typescript
// app/api/auth/send-verification/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";
import { sendEmail } from "@/lib/email";
import VerifyEmail from "@/emails/verify-email";
import crypto from "crypto";

export async function POST(request: NextRequest) {
  try {
    const { email } = await request.json();

    if (!email || typeof email !== "string") {
      return NextResponse.json({ error: "Email is required" }, { status: 400 });
    }

    const supabase = await createClient();

    // Generate a verification token
    const token = crypto.randomBytes(32).toString("hex");
    const expiresAt = new Date(Date.now() + 24 * 60 * 60 * 1000); // 24 hours

    // Store token in database
    const { error: dbError } = await supabase.from("email_verifications").insert({
      email,
      token,
      expires_at: expiresAt.toISOString(),
    });

    if (dbError) {
      console.error("Database error:", dbError);
      return NextResponse.json({ error: "Failed to create verification" }, { status: 500 });
    }

    // Send the email
    const verificationUrl = `${process.env.NEXT_PUBLIC_APP_URL}/auth/verify?token=${token}`;
    await sendEmail({
      to: email,
      subject: "Verify your email address",
      react: VerifyEmail({ verificationUrl, expiresInHours: 24 }),
    });

    return NextResponse.json({ message: "Verification email sent" });
  } catch (err) {
    console.error("Send verification error:", err);
    return NextResponse.json({ error: "Internal server error" }, { status: 500 });
  }
}
```

### API Route: Send Welcome Email After Verification

```typescript
// app/api/auth/verify-email/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";
import { sendEmail } from "@/lib/email";
import WelcomeEmail from "@/emails/welcome";

export async function GET(request: NextRequest) {
  try {
    const token = request.nextUrl.searchParams.get("token");

    if (!token) {
      return NextResponse.json({ error: "Token is required" }, { status: 400 });
    }

    const supabase = await createClient();

    // Look up the token
    const { data: verification, error } = await supabase
      .from("email_verifications")
      .select("*")
      .eq("token", token)
      .gt("expires_at", new Date().toISOString())
      .single();

    if (error || !verification) {
      return NextResponse.json({ error: "Invalid or expired token" }, { status: 400 });
    }

    // Mark email as verified
    const { error: updateError } = await supabase
      .from("profiles")
      .update({ email_verified: true })
      .eq("email", verification.email);

    if (updateError) {
      console.error("Profile update error:", updateError);
      return NextResponse.json({ error: "Verification failed" }, { status: 500 });
    }

    // Delete used token
    await supabase.from("email_verifications").delete().eq("token", token);

    // Send welcome email
    const dashboardUrl = `${process.env.NEXT_PUBLIC_APP_URL}/dashboard`;
    await sendEmail({
      to: verification.email,
      subject: "Welcome to our platform!",
      react: WelcomeEmail({ userName: verification.email.split("@")[0], dashboardUrl }),
    });

    return NextResponse.redirect(new URL("/dashboard?verified=true", request.url));
  } catch (err) {
    console.error("Verify email error:", err);
    return NextResponse.json({ error: "Internal server error" }, { status: 500 });
  }
}
```

## Queue Management for Bulk Emails

For sending many emails at once (like a newsletter), send them in batches to avoid rate limits:

```typescript
// lib/email-queue.ts
import { resend, FROM_EMAIL } from "./email";

interface QueuedEmail {
  to: string;
  subject: string;
  react: React.ReactElement;
}

export async function sendBulkEmails(emails: QueuedEmail[], batchSize = 10) {
  const results: { success: string[]; failed: string[] } = {
    success: [],
    failed: [],
  };

  // Process in batches
  for (let i = 0; i < emails.length; i += batchSize) {
    const batch = emails.slice(i, i + batchSize);

    const batchPromises = batch.map(async (email) => {
      try {
        const { error } = await resend.emails.send({
          from: FROM_EMAIL,
          to: email.to,
          subject: email.subject,
          react: email.react,
        });

        if (error) {
          results.failed.push(email.to);
          console.error(`Failed to send to ${email.to}:`, error);
        } else {
          results.success.push(email.to);
        }
      } catch (err) {
        results.failed.push(email.to);
        console.error(`Error sending to ${email.to}:`, err);
      }
    });

    await Promise.all(batchPromises);

    // Wait 1 second between batches to respect rate limits
    if (i + batchSize < emails.length) {
      await new Promise((resolve) => setTimeout(resolve, 1000));
    }
  }

  return results;
}
```

## Australian Spam Act Compliance

If you send emails to Australian users, you must follow the Spam Act 2003. Here is what you need:

1. **Consent**: Only send emails to people who opted in (signed up, checked a box, etc.)
2. **Sender Identity**: Every email must clearly identify who sent it
3. **Unsubscribe**: Every marketing email must have a working unsubscribe link

### Unsubscribe Link Implementation

```typescript
// app/api/email/unsubscribe/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";

export async function GET(request: NextRequest) {
  try {
    const token = request.nextUrl.searchParams.get("token");

    if (!token) {
      return NextResponse.json({ error: "Token required" }, { status: 400 });
    }

    const supabase = await createClient();

    // Look up the unsubscribe token to find the user
    const { data: subscription, error } = await supabase
      .from("email_subscriptions")
      .select("*")
      .eq("unsubscribe_token", token)
      .single();

    if (error || !subscription) {
      return NextResponse.json({ error: "Invalid token" }, { status: 400 });
    }

    // Unsubscribe the user
    const { error: updateError } = await supabase
      .from("email_subscriptions")
      .update({ subscribed: false, unsubscribed_at: new Date().toISOString() })
      .eq("id", subscription.id);

    if (updateError) {
      console.error("Unsubscribe error:", updateError);
      return NextResponse.json({ error: "Failed to unsubscribe" }, { status: 500 });
    }

    return NextResponse.redirect(new URL("/unsubscribed", request.url));
  } catch (err) {
    console.error("Unsubscribe error:", err);
    return NextResponse.json({ error: "Internal server error" }, { status: 500 });
  }
}
```

Add this footer to every marketing email template:

```tsx
// emails/components/compliance-footer.tsx
import { Text, Link, Section } from "@react-email/components";

interface ComplianceFooterProps {
  companyName: string;
  companyAddress: string;
  unsubscribeUrl: string;
}

export function ComplianceFooter({
  companyName,
  companyAddress,
  unsubscribeUrl,
}: ComplianceFooterProps) {
  return (
    <Section style={{ marginTop: "32px", borderTop: "1px solid #e5e7eb", paddingTop: "16px" }}>
      <Text style={{ fontSize: "12px", color: "#9ca3af", textAlign: "center" as const }}>
        Sent by {companyName}, {companyAddress}
      </Text>
      <Text style={{ fontSize: "12px", color: "#9ca3af", textAlign: "center" as const }}>
        <Link href={unsubscribeUrl} style={{ color: "#6b7280" }}>
          Unsubscribe
        </Link>{" "}
        from these emails.
      </Text>
    </Section>
  );
}
```

## Email Triggers from Supabase Database Webhooks

You can send emails automatically when something changes in your database. For example, send a notification when a new order is created:

```typescript
// app/api/webhooks/new-order/route.ts
import { NextRequest, NextResponse } from "next/server";
import { sendEmail } from "@/lib/email";
import OrderConfirmation from "@/emails/order-confirmation";

export async function POST(request: NextRequest) {
  try {
    // Verify the webhook is from Supabase
    const authHeader = request.headers.get("authorization");
    if (authHeader !== `Bearer ${process.env.SUPABASE_WEBHOOK_SECRET}`) {
      return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
    }

    const payload = await request.json();
    const { type, record } = payload;

    if (type !== "INSERT") {
      return NextResponse.json({ message: "Ignored" });
    }

    // Send order confirmation email
    await sendEmail({
      to: record.customer_email,
      subject: `Order #${record.id} confirmed`,
      react: OrderConfirmation({
        orderId: record.id,
        items: record.items,
        total: record.total,
      }),
    });

    return NextResponse.json({ message: "Email sent" });
  } catch (err) {
    console.error("Webhook error:", err);
    return NextResponse.json({ error: "Internal server error" }, { status: 500 });
  }
}
```

To set up the webhook in Supabase:
1. Go to your Supabase dashboard > **Database** > **Webhooks**
2. Create a new webhook
3. Table: `orders`, Events: `INSERT`
4. URL: `https://yourapp.com/api/webhooks/new-order`
5. Add an Authorization header with your webhook secret

## Error Handling and Retry for Failed Sends

```typescript
// lib/email-with-retry.ts
import { sendEmail } from "./email";

export async function sendEmailWithRetry(
  params: { to: string; subject: string; react: React.ReactElement },
  maxRetries = 3
) {
  let lastError: Error | null = null;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const result = await sendEmail(params);
      return result;
    } catch (err) {
      lastError = err instanceof Error ? err : new Error(String(err));
      console.error(`Email attempt ${attempt}/${maxRetries} failed:`, lastError.message);

      if (attempt < maxRetries) {
        // Wait longer between each retry: 1s, 2s, 4s
        const delay = Math.pow(2, attempt - 1) * 1000;
        await new Promise((resolve) => setTimeout(resolve, delay));
      }
    }
  }

  throw new Error(`Email failed after ${maxRetries} attempts: ${lastError?.message}`);
}
```

## Email Preview and Testing in Development

React Email includes a preview server so you can see your emails in a browser during development.

Add this script to your `package.json`:

```json
// package.json
{
  "scripts": {
    "email:dev": "email dev --dir emails --port 3001"
  }
}
```

Run `npm run email:dev` and open `http://localhost:3001` to preview your templates.

For testing, use Resend's test mode by sending to a special domain:

```typescript
// In your test environment, override the recipient
const testTo = process.env.NODE_ENV === "development"
  ? "delivered@resend.dev"
  : actualUserEmail;

await sendEmail({
  to: testTo,
  subject: "Test email",
  react: WelcomeEmail({ userName: "Test User", dashboardUrl: "/dashboard" }),
});
```

Resend also provides these test addresses:
- `delivered@resend.dev` - always succeeds
- `bounced@resend.dev` - simulates a bounce
- `complained@resend.dev` - simulates a spam complaint
