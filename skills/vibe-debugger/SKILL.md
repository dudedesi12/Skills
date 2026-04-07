---
name: vibe-debugger
description: "Use this skill whenever the user says 'it's not working right', 'something is off', 'it looks wrong', 'it works but...', 'it's weird', 'the data isn't showing', 'the page is blank', 'it works locally but not in production', 'it was working yesterday', 'nothing happens when I click', 'the styles are wrong', 'it's loading forever', or ANY description of unexpected behavior WITHOUT an explicit error message. Also trigger when user says 'I don't know what's wrong' or 'I can't figure it out'. This is the OPPOSITE of error-whisperer — it handles cases where there IS no error to read."
---

# Vibe Debugger

When something does not work but there is NO error message — the page loads but looks wrong, a feature "kinda works but not right", or the behavior is "weird" — this skill finds invisible bugs through systematic diagnosis.

## The Diagnosis Method

Every invisible bug gets solved the same way. Follow these 4 steps in order.

### Step 1: What SHOULD Happen?

Ask the user to describe the expected behavior in one sentence:

```
"When I click Submit, what SHOULD happen?"
"When the page loads, what SHOULD you see?"
"After login, where SHOULD the user end up?"
```

Get a clear picture of the desired outcome before diagnosing anything.

### Step 2: What ACTUALLY Happens?

Ask the user to describe the actual behavior:

```
"What do you actually see instead?"
"Does anything happen at all, or is it completely dead?"
"Is it partially working — like data shows but looks wrong?"
```

### Step 3: When Did It Last Work?

Narrow down the change that broke it:

```
"Was this ever working before?"
"What was the last thing you changed before it broke?"
"Did you install any new packages, change env vars, or modify config?"
```

If the user knows what changed, start there. If not, move to Step 4.

### Step 4: The 5 Usual Suspects

Check these in order. 90% of invisible bugs come from one of these:

#### Suspect 1: Cache

The most common invisible bug. You changed the code but the old version is still being served.

**Quick test:** Hard refresh (Ctrl+Shift+R or Cmd+Shift+R). If that fixes it, it was cache.

**Nuclear cache clear:**
```bash
# Stop the dev server first, then:
rm -rf .next
npm run dev
```

For production (Vercel): Redeploy with "Override Build Cache" toggled on.

See `references/cache-busting.md` for every cache layer and how to clear it.

#### Suspect 2: Environment

Wrong env vars, or dev environment behaves differently from production.

**Quick test:** Log the env var to confirm it has the right value:
```typescript
console.log("SUPABASE_URL:", process.env.NEXT_PUBLIC_SUPABASE_URL);
```

Common traps:
- `.env.local` has the value but Vercel does not
- Env var name is misspelled
- Missing `NEXT_PUBLIC_` prefix for client-side vars
- Changed `.env.local` but did not restart dev server

See `references/dev-vs-prod.md` for all environment differences.

#### Suspect 3: Data

The code is correct but the data is wrong, missing, or shaped differently than expected.

**Quick test:** Log the data right after fetching:
```typescript
const { data, error } = await supabase.from("profiles").select("*");
console.log("Data:", JSON.stringify(data, null, 2));
console.log("Error:", error);
```

Common traps:
- Supabase RLS silently returns empty arrays (no error, just `[]`)
- Data has `null` values where you expected strings
- Foreign key relationship not included in the query (need `.select("*, posts(*)")`)
- Column name is different in the database than in your code

#### Suspect 4: Timing

The code runs in the wrong order, or something is not ready yet when you try to use it.

**Quick test:** Add a loading state and see if the content appears after a delay:
```typescript
const [data, setData] = useState<Data | null>(null);
const [loading, setLoading] = useState(true);

useEffect(() => {
  fetchData().then(setData).finally(() => setLoading(false));
}, []);

if (loading) return <div>Loading...</div>;
if (!data) return <div>No data found</div>;
```

Common traps:
- Race condition: two fetches, wrong one finishes first
- Hydration: server renders one thing, client renders another
- Missing `await` on an async function
- `useEffect` runs after paint, so initial render has no data

#### Suspect 5: Scope

The code runs in the wrong context — server vs client, or middleware interferes.

**Quick test:** Add `console.log("Running on:", typeof window === "undefined" ? "server" : "client")` to see where the code executes.

Common traps:
- Using `useState` or `useEffect` in a Server Component (needs `"use client"`)
- Server Component trying to access `window`, `document`, or `localStorage`
- Middleware redirecting before the page loads
- API route using server-only imports on client

## Common Invisible Bugs

### Data Not Showing (No Error)

**Most likely cause:** Supabase RLS. RLS returns empty arrays instead of errors.

**Fix:**
```sql
-- Check if RLS is enabled
SELECT tablename, policyname FROM pg_policies WHERE tablename = 'your_table';

-- Quick fix: add a SELECT policy
CREATE POLICY "Allow authenticated reads" ON public.your_table
  FOR SELECT USING (auth.uid() = user_id);
```

**Verify with service role (bypasses RLS):**
```typescript
import { createClient } from "@supabase/supabase-js";
const admin = createClient(url, process.env.SUPABASE_SERVICE_ROLE_KEY!);
const { data } = await admin.from("your_table").select("*");
console.log("Admin query:", data); // If this returns data, it's RLS
```

### Page Is Blank (No Error)

**Check in order:**
1. Browser console (F12) — there might be an error you missed
2. Does the component have a default export?
3. Is it wrapped in a layout that might be hiding content?
4. Is middleware redirecting before the page loads?
5. Does `error.tsx` exist in the route? If not, errors get swallowed silently.

### Styles Look Wrong in Production

**Most likely cause:** Tailwind classes getting purged.

**Fix:** Check that dynamic class names use full strings:
```typescript
// WRONG — gets purged in production
<div className={`text-${color}-500`} />

// RIGHT — full class names survive purging
<div className={color === "red" ? "text-red-500" : "text-blue-500"} />
```

Also check `tailwind.config.ts` content paths include all your files:
```typescript
content: ["./app/**/*.{ts,tsx}", "./components/**/*.{ts,tsx}"],
```

### Works Locally, Breaks in Production

See `references/dev-vs-prod.md` for the full list. Most common causes:
1. **Case sensitivity** — Mac is case-insensitive, Linux (Vercel) is not
2. **Missing env vars** — set in `.env.local` but not in Vercel
3. **Hardcoded localhost URLs** — `http://localhost:3000` does not work in production
4. **Node.js API in Edge runtime** — works in dev, fails in production Edge functions

### Click Does Nothing

**Check in order:**
1. Is there an `onClick` handler? Check spelling: `onClick` not `onclick`
2. Is the handler an async function that is silently failing? Add try/catch
3. Is a parent element with `pointer-events: none` or `z-index` blocking it?
4. Is the button disabled or covered by an invisible overlay?
5. Is the event handler in a Server Component? Events need `"use client"`

### Loading Forever (Spinner Never Goes Away)

**Check in order:**
1. Is the fetch actually completing? Add a `.finally()` to set loading false
2. Is there an `await` missing? The fetch returned a Promise, not data
3. Is the API endpoint correct? Check network tab for 404s
4. Is the Supabase query hanging? Add a timeout
5. Is it an infinite re-render? Check `useEffect` dependencies

## Diagnostic Code Snippets

### Log Everything About a Supabase Query

```typescript
const { data, error, count, status, statusText } = await supabase
  .from("your_table")
  .select("*", { count: "exact" });

console.log("Status:", status, statusText);
console.log("Error:", error);
console.log("Count:", count);
console.log("Data:", JSON.stringify(data, null, 2));
```

### Check Auth State

```typescript
const { data: { user }, error } = await supabase.auth.getUser();
console.log("User:", user?.id, user?.email);
console.log("Auth error:", error);
console.log("Session:", await supabase.auth.getSession());
```

### Check All Environment Variables

```typescript
// Server-side only — NEVER expose in client code
console.log("ENV CHECK:", {
  supabaseUrl: !!process.env.NEXT_PUBLIC_SUPABASE_URL,
  supabaseAnon: !!process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY,
  supabaseService: !!process.env.SUPABASE_SERVICE_ROLE_KEY,
  stripeSecret: !!process.env.STRIPE_SECRET_KEY,
  geminiKey: !!process.env.GEMINI_API_KEY,
});
```

### Network Request Inspector

```typescript
const res = await fetch(url);
console.log("Fetch result:", {
  ok: res.ok,
  status: res.status,
  statusText: res.statusText,
  headers: Object.fromEntries(res.headers.entries()),
  url: res.url,
});
```

## Vercel Log Tailing

Check production logs in real time:

```bash
# Install Vercel CLI
npm i -g vercel

# Tail logs
vercel logs --follow

# Or check in dashboard:
# Vercel > Project > Logs > Functions
```

## The 3-Question Rule

If you cannot figure it out in 3 targeted questions, escalate to the full diagnosis tree in `references/diagnosis-tree.md`. Do not dump a 20-item checklist on the user — ask smart questions that narrow down the problem fast.
