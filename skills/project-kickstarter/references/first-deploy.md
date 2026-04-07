# First Deploy: Zero to Live URL

Get your project live on the internet in 10 minutes. This guide assumes you have completed the project setup from the main skill.

---

## Prerequisites

Before you start, you need accounts on these services (all have free tiers):

1. **GitHub** — [github.com](https://github.com) (for hosting your code)
2. **Vercel** — [vercel.com](https://vercel.com) (for hosting your app)
3. **Supabase** — [supabase.com](https://supabase.com) (for your database and auth)

You also need these CLI tools installed:

```bash
# Check if you have them
gh --version      # GitHub CLI
npx vercel --version  # Vercel CLI (runs via npx, no install needed)
```

If `gh` is not installed:

```bash
# macOS
brew install gh

# Linux
sudo apt install gh

# Windows
winget install GitHub.cli
```

Then authenticate:

```bash
gh auth login
```

---

## Step 1: Create a Production Supabase Project (2 minutes)

Your local Supabase instance only runs on your machine. You need a cloud project for production.

1. Go to [supabase.com/dashboard](https://supabase.com/dashboard)
2. Click "New Project"
3. Pick your organization (or create one)
4. Enter project details:
   - **Name:** Your project name
   - **Database Password:** Generate a strong password and save it somewhere safe
   - **Region:** Pick the one closest to your users
5. Click "Create new project"
6. Wait about 1 minute for the project to provision

Once ready, go to **Project Settings > API** and copy:
- **Project URL** (looks like `https://abcdefgh.supabase.co`)
- **anon public key** (starts with `eyJ...`)
- **service_role key** (starts with `eyJ...`)

### Push Your Database Schema to Production

```bash
# Link your local project to the remote one
npx supabase link --project-ref YOUR_PROJECT_REF

# The project ref is the random string in your Supabase URL
# https://abcdefgh.supabase.co → project ref is "abcdefgh"

# Push your migrations to production
npx supabase db push
```

This runs all migration files in `supabase/migrations/` against your production database.

### Configure Google OAuth in Production

1. Go to Supabase Dashboard > Authentication > Providers
2. Find Google and enable it
3. Paste your `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET`
4. Copy the "Callback URL" shown by Supabase
5. Go to Google Cloud Console > APIs & Services > Credentials
6. Edit your OAuth 2.0 Client ID
7. Add the Supabase callback URL to "Authorized redirect URIs"

---

## Step 2: Create a GitHub Repository (1 minute)

```bash
# Make sure you are in your project directory
cd PROJECT_NAME

# Create a private GitHub repo and push
git add -A
git commit -m "Initial project setup"
git branch -M main
gh repo create PROJECT_NAME --private --source=. --push
```

This creates a private repository on GitHub and pushes all your code. The `gh` CLI handles authentication automatically.

**Verify:** Open `https://github.com/YOUR_USERNAME/PROJECT_NAME` in your browser. You should see all your files.

---

## Step 3: Deploy to Vercel (3 minutes)

### Option A: Link via Vercel Dashboard (recommended for first-timers)

1. Go to [vercel.com/new](https://vercel.com/new)
2. Click "Import Git Repository"
3. Select your GitHub repository
4. Vercel auto-detects Next.js — no config needed
5. Before clicking "Deploy," expand "Environment Variables"
6. Add each production environment variable (see list below)
7. Click "Deploy"

### Option B: Link via CLI

```bash
# Link to Vercel (creates project if it doesn't exist)
npx vercel link

# Set environment variables
npx vercel env add NEXT_PUBLIC_SUPABASE_URL
# Paste your production Supabase URL when prompted

npx vercel env add NEXT_PUBLIC_SUPABASE_ANON_KEY
# Paste your anon key

npx vercel env add SUPABASE_SERVICE_ROLE_KEY
# Paste your service role key

npx vercel env add GEMINI_API_KEY
# Paste your Gemini API key

npx vercel env add NEXT_PUBLIC_APP_URL
# Enter your production URL (e.g., https://yourapp.vercel.app)

npx vercel env add CRON_SECRET
# Generate and paste: openssl rand -base64 32

# For saas template, also add:
npx vercel env add STRIPE_SECRET_KEY
npx vercel env add NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY
npx vercel env add STRIPE_WEBHOOK_SECRET
```

Then deploy:

```bash
# Deploy to production
npx vercel --prod
```

### Environment Variables to Set in Vercel

| Variable | Production Value |
|----------|-----------------|
| `NEXT_PUBLIC_SUPABASE_URL` | `https://YOUR_REF.supabase.co` |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Your production anon key |
| `SUPABASE_SERVICE_ROLE_KEY` | Your production service role key |
| `GEMINI_API_KEY` | Your Gemini API key |
| `NEXT_PUBLIC_APP_URL` | `https://your-project.vercel.app` (or custom domain) |
| `CRON_SECRET` | Random string from `openssl rand -base64 32` |
| `STRIPE_SECRET_KEY` | `sk_live_...` (or `sk_test_...` for staging) |
| `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` | `pk_live_...` (or `pk_test_...` for staging) |
| `STRIPE_WEBHOOK_SECRET` | From Stripe Dashboard webhook endpoint |

---

## Step 4: Verify the Deployment (2 minutes)

Once Vercel finishes building (usually under 60 seconds), it gives you a URL like `https://your-project.vercel.app`.

### Run these checks:

**1. Landing page loads:**
Visit `https://your-project.vercel.app` — you should see your marketing/landing page.

**2. Health check works:**
Visit `https://your-project.vercel.app/api/health` — you should see:
```json
{ "status": "ok", "timestamp": "...", "environment": "production" }
```

**3. Auth pages load:**
Visit `https://your-project.vercel.app/login` — you should see the login form.

**4. Auth redirect works:**
Visit `https://your-project.vercel.app/dashboard` — you should be redirected to `/login` because you are not authenticated.

**5. Build logs are clean:**
Go to Vercel Dashboard > Your Project > Deployments > latest deployment > "Building" tab. Check for warnings or errors.

---

## Step 5: Set Up a Custom Domain (2 minutes, optional)

### If you have a domain:

1. Go to Vercel Dashboard > Your Project > Settings > Domains
2. Enter your domain (e.g., `yourapp.com`)
3. Vercel shows you DNS records to add
4. Go to your domain registrar (Namecheap, Cloudflare, GoDaddy, etc.)
5. Add the DNS records Vercel shows:
   - For apex domain (`yourapp.com`): Add an `A` record pointing to `76.76.21.21`
   - For `www.yourapp.com`: Add a `CNAME` record pointing to `cname.vercel-dns.com`
6. Wait for DNS propagation (usually 1-5 minutes, can take up to 48 hours)
7. Vercel automatically provisions an SSL certificate

### Update your environment variables:

After adding a custom domain, update `NEXT_PUBLIC_APP_URL` in Vercel:

```
NEXT_PUBLIC_APP_URL=https://yourapp.com
```

Also update the Supabase redirect URLs:
1. Go to Supabase Dashboard > Authentication > URL Configuration
2. Set **Site URL** to `https://yourapp.com`
3. Add `https://yourapp.com/auth/callback` to **Redirect URLs**

---

## Step 6: Set Up CI/CD (automatic)

Vercel automatically sets up CI/CD when you connect a GitHub repository:

- **Every push to `main`** triggers a production deployment
- **Every pull request** gets a preview deployment with a unique URL
- **Preview deployments** use the same environment variables (or you can set separate ones for preview)

You can see all deployments in:
- Vercel Dashboard > Your Project > Deployments
- GitHub pull request comments (Vercel bot posts preview URLs)

### Vercel Cron Jobs

If your project has a `vercel.json` with cron configuration:

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

Cron jobs activate automatically on production deployments. They do NOT run on preview deployments.

Verify cron is set up: Go to Vercel Dashboard > Your Project > Settings > Cron Jobs.

---

## Step 7: Set Up Stripe Webhooks for Production (saas template only)

1. Go to [dashboard.stripe.com/webhooks](https://dashboard.stripe.com/webhooks)
2. Click "Add endpoint"
3. Enter your endpoint URL: `https://yourapp.com/api/webhooks/stripe`
4. Select events to listen for:
   - `checkout.session.completed`
   - `customer.subscription.created`
   - `customer.subscription.updated`
   - `customer.subscription.deleted`
   - `invoice.payment_succeeded`
   - `invoice.payment_failed`
5. Click "Add endpoint"
6. Copy the **Signing secret** (starts with `whsec_`)
7. Set it as `STRIPE_WEBHOOK_SECRET` in Vercel environment variables
8. Redeploy (or wait for next push)

### Test the webhook:

```bash
# From Stripe CLI (uses test mode)
stripe trigger checkout.session.completed
```

Check your Vercel function logs: Vercel Dashboard > Your Project > Deployments > latest > Functions tab.

---

## Troubleshooting

### "Build failed" on Vercel

**Check the build logs.** Go to Vercel Dashboard > Deployments > failed deployment > Building tab. Common causes:

- Missing environment variables (the build will show "NEXT_PUBLIC_SUPABASE_URL is not set")
- TypeScript errors (fix locally with `npm run build` before pushing)
- Missing dependencies (check `package.json`)

### Auth not working in production

1. Verify `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_ANON_KEY` are set in Vercel
2. Verify Supabase project Site URL matches your production URL
3. Verify redirect URLs include your production callback URL
4. For Google OAuth: verify the production Supabase callback URL is in your Google OAuth client redirect URIs

### Stripe webhooks returning 400

1. Verify `STRIPE_WEBHOOK_SECRET` in Vercel matches the signing secret from Stripe Dashboard
2. Ensure you are using the correct secret (production, not the CLI local one)
3. Check Vercel function logs for the exact error message

### "504 Gateway Timeout" on API routes

Vercel Serverless Functions have a 10-second timeout on the Hobby plan (60 seconds on Pro). If your Gemini API call takes too long:

1. Use the `flash` tier instead of `pro` for faster responses
2. Add streaming for long responses
3. Upgrade to Vercel Pro for longer timeouts

### Database migrations not applied

```bash
# Check migration status
npx supabase migration list --linked

# If migrations are pending, push them
npx supabase db push
```

---

## Post-Deploy Checklist

After your first deploy, confirm everything works end-to-end:

- [ ] Landing page loads on production URL
- [ ] `/api/health` returns 200
- [ ] Sign up with email works
- [ ] Sign in with email works
- [ ] Google OAuth sign in works
- [ ] Dashboard loads after sign in
- [ ] Dashboard redirects to login when not signed in
- [ ] Sign out works
- [ ] Stripe checkout works (test mode) — saas template
- [ ] Stripe webhook receives events — saas template
- [ ] Gemini API calls work from production
- [ ] Cron job is registered in Vercel
- [ ] Custom domain has SSL certificate (if applicable)
- [ ] All environment variables are set for production

Once all checks pass, your project is live and ready for feature development.
