# Pre-Launch Security Checklist

30+ items to verify before going to production. Go through every single one.

## Authentication (8 items)

- [ ] **Use `getUser()` not `getSession()`** — `getSession()` doesn't verify the JWT. Always use `getUser()` for auth decisions.
- [ ] **Middleware protects all private routes** — Dashboard, settings, admin, and API routes all redirect unauthenticated users.
- [ ] **Admin routes check user role** — Not just authentication, but authorization. Verify the user has the `admin` role.
- [ ] **OAuth callback validates redirect URL** — Prevent open redirect attacks by checking the redirect origin matches your domain.
- [ ] **Password policy enforced** — Minimum 8 characters, requires uppercase, lowercase, and a number.
- [ ] **Email verification enabled** — In Supabase Dashboard > Authentication > Settings, enable "Confirm email."
- [ ] **Rate limiting on auth endpoints** — Limit login attempts to prevent brute force (e.g., 5 attempts per minute per IP).
- [ ] **Sign-out clears session properly** — Call `supabase.auth.signOut()` and redirect to login page.

## Data Protection (7 items)

- [ ] **Row Level Security (RLS) enabled on ALL tables** — Every single table in Supabase must have RLS enabled. No exceptions.
- [ ] **RLS policies are restrictive** — Users can only read/write their own data. Test by trying to access another user's data.
- [ ] **Input validation on every API route** — Use Zod schemas to validate all incoming data. Never trust client input.
- [ ] **SQL injection prevention** — Use Supabase client (parameterized queries). Never concatenate user input into SQL strings.
- [ ] **File upload validation** — Check file type, file size, and sanitize file names. Don't trust the `Content-Type` header alone.
- [ ] **Sensitive data not in client-side code** — No API keys, secrets, or tokens in components, hooks, or anything that ships to the browser.
- [ ] **Personal data handling** — Ensure GDPR/privacy compliance. Allow users to export and delete their data.

## Security Headers (6 items)

- [ ] **Content-Security-Policy set** — Restricts which scripts, styles, and resources can load. See `headers-config.md`.
- [ ] **X-Frame-Options: DENY** — Prevents your site from being embedded in iframes (clickjacking protection).
- [ ] **X-Content-Type-Options: nosniff** — Prevents browsers from MIME-sniffing the content type.
- [ ] **Strict-Transport-Security set** — Forces HTTPS with `max-age=31536000; includeSubDomains`.
- [ ] **Referrer-Policy set** — Use `strict-origin-when-cross-origin` to limit referer information leakage.
- [ ] **Permissions-Policy set** — Disable camera, microphone, geolocation, and other APIs you don't use.

## Environment Variables (5 items)

- [ ] **No secrets in `NEXT_PUBLIC_` variables** — Only the Supabase URL and anon key should be public. Everything else is server-only.
- [ ] **`.env.local` in `.gitignore`** — Never commit your local environment file.
- [ ] **All production secrets in Vercel** — Use `vercel env add` for each secret. Don't hardcode anything.
- [ ] **Service role key is server-only** — `SUPABASE_SERVICE_ROLE_KEY` must NEVER start with `NEXT_PUBLIC_`.
- [ ] **Secrets rotated after any leak** — If a secret was ever in a commit, public repo, or log, rotate it immediately.

## API Routes (5 items)

- [ ] **All API routes check authentication** — Unless intentionally public, every route must verify the user is logged in.
- [ ] **Rate limiting on public API routes** — Prevent abuse with Upstash Redis rate limiting.
- [ ] **CORS configured correctly** — Only allow requests from your own domain(s). Don't use `Access-Control-Allow-Origin: *`.
- [ ] **Webhook endpoints verify signatures** — Stripe, Supabase, and any other webhook must verify the HMAC signature.
- [ ] **Cron endpoints verify `CRON_SECRET`** — All Vercel Cron routes must check `Authorization: Bearer $CRON_SECRET`.

## XSS and CSRF (4 items)

- [ ] **No `dangerouslySetInnerHTML` with user input** — If you must render HTML, sanitize it with DOMPurify first.
- [ ] **User-generated content is escaped** — React does this by default, but verify any custom rendering.
- [ ] **CSRF protection on state-changing routes** — Server Actions have built-in CSRF protection. For custom API routes, use tokens.
- [ ] **Cookies set with correct flags** — `httpOnly`, `secure` (in production), `sameSite: 'lax'`.

## Dependencies (3 items)

- [ ] **`npm audit` shows no critical vulnerabilities** — Run `npm audit` and fix any critical or high severity issues.
- [ ] **Dependencies are up to date** — Run `npm outdated` and update packages, especially security-related ones.
- [ ] **No unnecessary dependencies** — Remove packages you're not using. Fewer dependencies = fewer attack vectors.

## Monitoring (3 items)

- [ ] **Error logging in place** — Errors in API routes and server actions are logged (console.error at minimum, Sentry for production).
- [ ] **Failed auth attempts are logged** — Track failed logins to detect brute force attacks.
- [ ] **API rate limit violations are logged** — Know when someone is hitting your rate limits.

## Quick Verification Commands

```bash
# Check for vulnerabilities
npm audit

# Check for outdated packages
npm outdated

# Check security headers (after deploying)
curl -I https://yourdomain.com

# Verify .env.local is gitignored
git status --ignored | grep .env

# Search for hardcoded secrets in your code
grep -r "sk_live\|sk_test\|password\|secret" --include="*.ts" --include="*.tsx" src/ app/ lib/

# Check that NEXT_PUBLIC_ vars don't contain secrets
grep -r "NEXT_PUBLIC_.*SECRET\|NEXT_PUBLIC_.*KEY.*sk_" .env*
```

## After Launch

- [ ] Test your site with [SecurityHeaders.com](https://securityheaders.com)
- [ ] Run a basic scan with [Mozilla Observatory](https://observatory.mozilla.org)
- [ ] Set up Dependabot or Renovate for automatic dependency updates
- [ ] Schedule monthly security reviews (check this list again)
