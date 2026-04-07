# Resend Setup Reference

## Installation

```bash
npm install resend @react-email/components
npm install -D react-email
```

## API Key Management

### Create API Key

1. Sign up at [resend.com](https://resend.com)
2. Go to **API Keys** in the sidebar
3. Click **Create API Key**
4. Choose a name (e.g., "my-app-production")
5. Select permission: **Sending access** (least privilege)
6. Optionally restrict to a specific domain
7. Copy the key immediately (you cannot see it again)

### Store API Key Securely

```bash
# .env.local (local development — NEVER commit this file)
RESEND_API_KEY=re_123456789abcdef
RESEND_FROM_EMAIL="Your App <noreply@yourdomain.com>"
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

For production on Vercel:
1. Go to your Vercel project > **Settings** > **Environment Variables**
2. Add `RESEND_API_KEY` with the value from Resend
3. Set it for **Production** and **Preview** environments
4. Add `RESEND_FROM_EMAIL` similarly

### API Key Validation

```typescript
// lib/email.ts
import { Resend } from "resend";

if (!process.env.RESEND_API_KEY) {
  throw new Error(
    "RESEND_API_KEY is not set. Get one from https://resend.com/api-keys"
  );
}

export const resend = new Resend(process.env.RESEND_API_KEY);
```

## Domain Setup

### Why Verify a Domain?

Without domain verification, emails come from `onboarding@resend.dev` which looks unprofessional and often lands in spam. With a verified domain, emails come from `noreply@yourdomain.com`.

### Add Domain in Resend

1. Go to **Domains** in Resend dashboard
2. Click **Add Domain**
3. Enter your domain (e.g., `yourdomain.com`)
4. Select your DNS provider from the list (or choose "Other")

### DNS Records to Add

Resend will give you records like these. Add them in your domain's DNS settings (Cloudflare, Namecheap, GoDaddy, etc.):

**MX Record** (for receiving bounces):
```
Type: MX
Name: feedback
Value: feedback-smtp.resend.com
Priority: 10
```

**SPF Record** (proves you are allowed to send from this domain):
```
Type: TXT
Name: @
Value: v=spf1 include:amazonses.com ~all
```

**DKIM Records** (cryptographic signature to prove email is not forged):
```
Type: CNAME
Name: resend._domainkey
Value: (provided by Resend — a long string)
```

```
Type: CNAME
Name: resend2._domainkey
Value: (provided by Resend — a long string)
```

```
Type: CNAME
Name: resend3._domainkey
Value: (provided by Resend — a long string)
```

**DMARC Record** (tells email providers what to do with unverified emails):
```
Type: TXT
Name: _dmarc
Value: v=DMARC1; p=none;
```

### Verify Domain

After adding DNS records:
1. Go back to Resend > **Domains**
2. Click **Verify** next to your domain
3. Wait for all records to show green checkmarks
4. This usually takes 5-30 minutes but can take up to 48 hours for DNS propagation

### Check Verification Status Programmatically

```typescript
// scripts/check-domain.ts
import { Resend } from "resend";

const resend = new Resend(process.env.RESEND_API_KEY);

async function checkDomain() {
  try {
    const { data, error } = await resend.domains.list();

    if (error) {
      console.error("Error:", error);
      return;
    }

    for (const domain of data?.data || []) {
      console.log(`${domain.name}: ${domain.status}`);
    }
  } catch (err) {
    console.error("Failed to check domains:", err);
  }
}

checkDomain();
```

## Sending Limits

### Free Plan
- 100 emails/day
- 3,000 emails/month
- Single sender domain

### Pro Plan
- 50,000 emails/month (scales up from there)
- Multiple domains
- Dedicated IPs available

### Rate Limiting

Resend allows 10 requests per second on the free plan. For bulk sending, batch your requests:

```typescript
// lib/rate-limited-send.ts
import { resend, FROM_EMAIL } from "./email";

const RATE_LIMIT_PER_SECOND = 8; // stay under the 10/s limit
const BATCH_DELAY_MS = 1000;

export async function sendBatch(
  emails: Array<{ to: string; subject: string; react: React.ReactElement }>
) {
  const results = { sent: 0, failed: 0 };

  for (let i = 0; i < emails.length; i += RATE_LIMIT_PER_SECOND) {
    const batch = emails.slice(i, i + RATE_LIMIT_PER_SECOND);

    const promises = batch.map(async (email) => {
      try {
        const { error } = await resend.emails.send({
          from: FROM_EMAIL,
          to: email.to,
          subject: email.subject,
          react: email.react,
        });
        if (error) {
          results.failed++;
          console.error(`Failed: ${email.to}`, error);
        } else {
          results.sent++;
        }
      } catch (err) {
        results.failed++;
        console.error(`Error: ${email.to}`, err);
      }
    });

    await Promise.all(promises);

    // Wait between batches
    if (i + RATE_LIMIT_PER_SECOND < emails.length) {
      await new Promise((r) => setTimeout(r, BATCH_DELAY_MS));
    }
  }

  return results;
}
```

## Monitoring Emails

### Check Email Status

```typescript
// Check if a specific email was delivered
const { data, error } = await resend.emails.get("email-id-here");

if (data) {
  console.log(`Status: ${data.last_event}`);
  // Possible statuses: sent, delivered, bounced, complained, opened, clicked
}
```

### Webhook for Delivery Events

Set up a webhook in Resend to get notified about email events:

```typescript
// app/api/webhooks/resend/route.ts
import { NextRequest, NextResponse } from "next/server";

export async function POST(request: NextRequest) {
  try {
    const payload = await request.json();

    // Verify webhook signature (recommended)
    const svixId = request.headers.get("svix-id");
    const svixTimestamp = request.headers.get("svix-timestamp");
    const svixSignature = request.headers.get("svix-signature");

    if (!svixId || !svixTimestamp || !svixSignature) {
      return NextResponse.json({ error: "Missing headers" }, { status: 401 });
    }

    switch (payload.type) {
      case "email.delivered":
        console.log(`Email delivered to ${payload.data.to}`);
        break;
      case "email.bounced":
        console.error(`Email bounced: ${payload.data.to}`);
        // Mark this email address as invalid in your database
        break;
      case "email.complained":
        console.error(`Spam complaint from ${payload.data.to}`);
        // Immediately unsubscribe this user
        break;
    }

    return NextResponse.json({ received: true });
  } catch (err) {
    console.error("Webhook error:", err);
    return NextResponse.json({ error: "Internal error" }, { status: 500 });
  }
}
```

## Testing in Development

```typescript
// Use Resend's test addresses during development
const TEST_ADDRESSES = {
  success: "delivered@resend.dev",
  bounce: "bounced@resend.dev",
  complaint: "complained@resend.dev",
};

// In development, redirect all emails to the test address
function getRecipient(email: string): string {
  if (process.env.NODE_ENV !== "production") {
    return TEST_ADDRESSES.success;
  }
  return email;
}
```
