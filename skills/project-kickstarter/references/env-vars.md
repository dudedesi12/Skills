# Environment Variables

Every environment variable needed for the project, grouped by service. Copy `.env.local.example` to `.env.local` and fill in each value.

---

## Supabase

| Variable | Value | Where to Find |
|----------|-------|---------------|
| `NEXT_PUBLIC_SUPABASE_URL` | `http://localhost:54321` (local) or `https://xxxx.supabase.co` (prod) | Run `npx supabase status` locally. In production, go to Supabase Dashboard > Project Settings > API. |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | `eyJhbG...` | Same as above. The `anon` key is safe for the browser — it respects RLS policies. |
| `SUPABASE_SERVICE_ROLE_KEY` | `eyJhbG...` | Same as above. The `service_role` key bypasses RLS. Server-only. Never expose to the client. |

**The `NEXT_PUBLIC_` prefix matters.** Variables with this prefix are bundled into client-side JavaScript and visible to anyone. Variables without it are server-only.

**Local development:** When you run `npx supabase start`, it prints all these values. Copy them directly.

**Production:** Create a Supabase project at [supabase.com](https://supabase.com), then go to Project Settings > API to find the URL and keys.

---

## Gemini AI

| Variable | Value | Where to Find |
|----------|-------|---------------|
| `GEMINI_API_KEY` | `AIza...` | Go to [aistudio.google.com/apikey](https://aistudio.google.com/apikey) and create an API key. |

**Free tier:** Gemini offers a generous free tier with rate limits. For production, set up billing in Google Cloud Console.

**Server-only.** This key is never exposed to the client. All Gemini calls happen in Route Handlers or Server Actions.

---

## Stripe (saas template only)

| Variable | Value | Where to Find |
|----------|-------|---------------|
| `STRIPE_SECRET_KEY` | `sk_test_...` (test) or `sk_live_...` (prod) | Go to [dashboard.stripe.com/apikeys](https://dashboard.stripe.com/apikeys). Use test mode keys during development. |
| `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` | `pk_test_...` (test) or `pk_live_...` (prod) | Same page. The publishable key is safe for the browser. |
| `STRIPE_WEBHOOK_SECRET` | `whsec_...` | For local: run `stripe listen --forward-to localhost:3000/api/webhooks/stripe` and copy the signing secret it prints. For production: go to Stripe Dashboard > Developers > Webhooks > Add endpoint > copy signing secret. |

**Test vs Live:** Always use `sk_test_` and `pk_test_` keys during development. Stripe test mode uses fake credit card numbers (4242 4242 4242 4242). Switch to live keys only when deploying to production.

**Webhook setup for local dev:**

```bash
# Install Stripe CLI
brew install stripe/stripe-cli/stripe   # macOS
# or download from https://stripe.com/docs/stripe-cli

# Login to Stripe
stripe login

# Forward webhooks to your local app
stripe listen --forward-to localhost:3000/api/webhooks/stripe
```

The `stripe listen` command prints a webhook signing secret (starts with `whsec_`). Use that as `STRIPE_WEBHOOK_SECRET` in `.env.local`.

---

## Google OAuth

| Variable | Value | Where to Find |
|----------|-------|---------------|
| `GOOGLE_CLIENT_ID` | `xxxx.apps.googleusercontent.com` | Google Cloud Console > APIs & Services > Credentials > OAuth 2.0 Client ID |
| `GOOGLE_CLIENT_SECRET` | `GOCSPX-...` | Same location as above. |

**Setup steps:**

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Create a project (or select an existing one)
3. Go to APIs & Services > OAuth consent screen
4. Configure the consent screen (External, add your app name and email)
5. Go to APIs & Services > Credentials
6. Click "Create Credentials" > "OAuth client ID"
7. Application type: "Web application"
8. Add authorized redirect URIs:
   - Local: `http://localhost:54321/auth/v1/callback`
   - Production: `https://YOUR_SUPABASE_PROJECT_ID.supabase.co/auth/v1/callback`
9. Copy the Client ID and Client Secret

**These go in Supabase, not directly in your app.** Set them in `supabase/config.toml` for local dev. In production, go to Supabase Dashboard > Authentication > Providers > Google and paste them there.

---

## Cron Jobs

| Variable | Value | Where to Find |
|----------|-------|---------------|
| `CRON_SECRET` | Any random string | Generate one: `openssl rand -base64 32` |

**Purpose:** Protects the `/api/cron` endpoint from unauthorized access. Vercel Cron sends this as a Bearer token in the Authorization header.

**Setting up Vercel Cron:** Add a `vercel.json` at the project root:

```json
{
  "crons": [
    {
      "path": "/api/cron",
      "schedule": "0 0 * * *"
    }
  ]
}
```

Then set `CRON_SECRET` as an environment variable in Vercel Dashboard > Project > Settings > Environment Variables.

---

## App Configuration

| Variable | Value | Where to Find |
|----------|-------|---------------|
| `NEXT_PUBLIC_APP_URL` | `http://localhost:3000` (local) or `https://yourdomain.com` (prod) | Your app's public URL. Used for generating absolute URLs in emails, Open Graph tags, and OAuth callbacks. |

---

## API Template Additional Variables

| Variable | Value | Where to Find |
|----------|-------|---------------|
| `API_SECRET_KEY` | Any random string | Generate one: `openssl rand -base64 32`. Used as the `x-api-key` header for authenticating API requests. |

---

## Complete .env.local.example

```bash
# =============================================================================
# Supabase
# Run `npx supabase status` to get local values
# Production: Supabase Dashboard > Project Settings > API
# =============================================================================
NEXT_PUBLIC_SUPABASE_URL=http://localhost:54321
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key-here
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key-here

# =============================================================================
# Gemini AI
# Get your key at https://aistudio.google.com/apikey
# =============================================================================
GEMINI_API_KEY=your-gemini-api-key-here

# =============================================================================
# Stripe (saas template only)
# Dashboard: https://dashboard.stripe.com/apikeys
# Webhook: run `stripe listen --forward-to localhost:3000/api/webhooks/stripe`
# =============================================================================
STRIPE_SECRET_KEY=sk_test_your-key-here
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_your-key-here
STRIPE_WEBHOOK_SECRET=whsec_your-secret-here

# =============================================================================
# Google OAuth
# These go in supabase/config.toml for local dev
# For production: Supabase Dashboard > Authentication > Providers > Google
# =============================================================================
GOOGLE_CLIENT_ID=your-google-client-id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=GOCSPX-your-google-client-secret

# =============================================================================
# Cron Jobs
# Generate: openssl rand -base64 32
# =============================================================================
CRON_SECRET=your-cron-secret-here

# =============================================================================
# App
# =============================================================================
NEXT_PUBLIC_APP_URL=http://localhost:3000

# =============================================================================
# API (api template only)
# Generate: openssl rand -base64 32
# =============================================================================
# API_SECRET_KEY=your-api-secret-key-here
```

---

## Production Environment Variables in Vercel

When deploying to Vercel, set all non-`NEXT_PUBLIC_` variables as environment variables in:

**Vercel Dashboard > Project > Settings > Environment Variables**

For each variable:
1. Enter the name (e.g., `STRIPE_SECRET_KEY`)
2. Enter the production value (live keys, not test keys)
3. Select which environments it applies to (Production, Preview, Development)
4. Save

**Recommended per-environment values:**

| Variable | Development | Preview | Production |
|----------|------------|---------|------------|
| `NEXT_PUBLIC_SUPABASE_URL` | Local Supabase | Staging Supabase | Production Supabase |
| `STRIPE_SECRET_KEY` | `sk_test_...` | `sk_test_...` | `sk_live_...` |
| `NEXT_PUBLIC_APP_URL` | `http://localhost:3000` | Vercel preview URL | `https://yourdomain.com` |

Vercel automatically injects `NEXT_PUBLIC_` variables into the client bundle at build time. Server-only variables are available via `process.env` at runtime.
