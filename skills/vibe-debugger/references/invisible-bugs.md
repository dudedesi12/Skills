# Invisible Bugs: 30+ "No Error But Broken" Scenarios

Every scenario: symptom, root cause, diagnostic steps, fix.

## Data Issues

### 1. Supabase Query Returns Empty Array

**Symptom:** Data exists in the database but the query returns `[]`. No error.

**Root cause:** RLS (Row Level Security) is enabled but no SELECT policy exists for the current user.

**Diagnose:**
```typescript
// Use service role to bypass RLS
const admin = createClient(url, process.env.SUPABASE_SERVICE_ROLE_KEY!);
const { data } = await admin.from("table").select("*");
// If this returns data → RLS is blocking
```

**Fix:**
```sql
CREATE POLICY "Users can read own data" ON public.your_table
  FOR SELECT USING (auth.uid() = user_id);
```

### 2. Data Shows But Is Stale

**Symptom:** The page shows old data even after updating the database.

**Root cause:** Next.js ISR cache is serving a stale version.

**Diagnose:** Check if the page uses static generation or ISR. Look for `revalidate` settings.

**Fix:**
```typescript
// Option 1: Revalidate after mutation
"use server";
import { revalidatePath } from "next/cache";
export async function updateData() {
  await db.update(...);
  revalidatePath("/dashboard");
}

// Option 2: Force dynamic rendering
export const dynamic = "force-dynamic";
```

### 3. Related Data Missing

**Symptom:** Main record loads but related records are empty.

**Root cause:** Supabase query does not include the join.

**Fix:**
```typescript
// Wrong — only gets the post
const { data } = await supabase.from("posts").select("*");

// Right — includes comments
const { data } = await supabase.from("posts").select("*, comments(*)");
```

### 4. Numbers Showing as Strings

**Symptom:** Calculations are wrong. `"5" + "3"` gives `"53"` instead of `8`.

**Root cause:** Data from API or form inputs comes as strings.

**Fix:**
```typescript
const total = Number(price) + Number(tax);
// Or
const total = parseInt(price, 10) + parseInt(tax, 10);
```

### 5. Dates Look Wrong or Show "Invalid Date"

**Symptom:** Dates display incorrectly or show NaN/Invalid Date.

**Root cause:** Timezone mismatch between server and client, or wrong date format.

**Fix:**
```typescript
// Parse ISO strings consistently
const date = new Date(row.created_at);
const formatted = date.toLocaleDateString("en-IN", { timeZone: "Asia/Kolkata" });
```

### 6. Null Values Crash the UI Silently

**Symptom:** Part of the page is missing. No error in console.

**Root cause:** A null value causes a component to render nothing.

**Fix:**
```typescript
// Add null checks
<p>{user?.name ?? "Anonymous"}</p>
<p>{profile?.bio || "No bio yet"}</p>
```

### 7. Supabase Update Silently Does Nothing

**Symptom:** Update query returns success but data does not change.

**Root cause:** RLS UPDATE policy is missing, or the WHERE clause matches nothing.

**Fix:**
```sql
CREATE POLICY "Users can update own data" ON public.profiles
  FOR UPDATE USING (auth.uid() = user_id);
```

### 8. Duplicate Data Appearing

**Symptom:** The same item appears multiple times in a list.

**Root cause:** Missing or wrong `key` prop in React, or the query returns duplicates from a join.

**Fix:**
```typescript
// Ensure unique keys
{items.map((item) => <Card key={item.id} {...item} />)}

// For duplicate join results, use .select() with distinct
const { data } = await supabase.from("posts").select("id, title").order("id");
```

## Styling Issues

### 9. Tailwind Classes Not Applying in Production

**Symptom:** Looks fine locally, styles missing in production.

**Root cause:** Dynamic class names get purged by Tailwind's tree-shaking.

**Fix:**
```typescript
// WRONG
<div className={`bg-${color}-500`} />

// RIGHT
const colorMap: Record<string, string> = {
  red: "bg-red-500",
  blue: "bg-blue-500",
  green: "bg-green-500",
};
<div className={colorMap[color]} />
```

### 10. Dark Mode Not Working

**Symptom:** Dark mode toggle does nothing.

**Root cause:** Tailwind dark mode is not configured, or the `dark` class is not on the html element.

**Fix:**
```typescript
// tailwind.config.ts
export default { darkMode: "class", /* ... */ };

// Toggle dark class on html
document.documentElement.classList.toggle("dark");
```

### 11. Layout Shifts on Page Load

**Symptom:** Content jumps around when the page loads.

**Root cause:** Images without dimensions, fonts loading late, or conditional rendering.

**Fix:**
```typescript
// Always set image dimensions
<Image src={url} width={400} height={300} alt="..." />

// Preload fonts in layout.tsx
import { Inter } from "next/font/google";
const inter = Inter({ subsets: ["latin"] });
```

### 12. Responsive Design Breaks at Specific Width

**Symptom:** Layout looks fine on desktop and mobile but breaks at a specific width.

**Root cause:** Missing breakpoint or conflicting CSS.

**Fix:** Use browser DevTools responsive mode. Drag the width slowly to find the exact breakpoint. Then add Tailwind classes for that range:
```typescript
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3">
```

## Auth Issues

### 13. User Logged In But Sees Unauthenticated State

**Symptom:** Auth works but the UI still shows "Login" button.

**Root cause:** Auth state not syncing between server and client, or cookie not being passed.

**Fix:** Make sure middleware refreshes the session:
```typescript
// middleware.ts
import { updateSession } from "@/lib/supabase/middleware";
export async function middleware(request: NextRequest) {
  return await updateSession(request);
}
```

### 14. Redirect Loop After Login

**Symptom:** Login succeeds but the page keeps redirecting back to login.

**Root cause:** Middleware checks auth, user is not recognized, redirects to login, which redirects to dashboard, which redirects to login...

**Fix:** Exclude auth routes from middleware:
```typescript
export const config = {
  matcher: ["/((?!login|signup|auth|_next/static|_next/image|favicon.ico).*)"],
};
```

### 15. OAuth Login Opens Blank Page

**Symptom:** Clicking "Login with Google" opens a new tab that stays blank.

**Root cause:** OAuth redirect URL not configured in Supabase or Google console.

**Fix:** Add the callback URL in both places:
- Supabase Dashboard > Auth > URL Configuration > Redirect URLs
- Google Cloud Console > OAuth > Authorized redirect URIs
- URL format: `https://your-project.supabase.co/auth/v1/callback`

## Performance Issues

### 16. Page Takes 5+ Seconds to Load

**Symptom:** No errors, just slow.

**Root cause:** Fetching too much data, no caching, or synchronous waterfall requests.

**Fix:**
```typescript
// Fetch in parallel, not sequentially
const [users, posts, stats] = await Promise.all([
  supabase.from("users").select("*"),
  supabase.from("posts").select("*").limit(10),
  supabase.from("stats").select("*").single(),
]);
```

### 17. First Load Is Slow, Then Fast

**Symptom:** First visit takes 2-3 seconds, subsequent visits are instant.

**Root cause:** Vercel serverless cold start. This is normal, not a bug.

**Fix:** Use Edge runtime for latency-sensitive routes:
```typescript
export const runtime = "edge";
```

### 18. Infinite Re-Renders

**Symptom:** Page freezes, fan spins, browser tab says "not responding."

**Root cause:** `useEffect` with an object/array dependency that recreates on every render.

**Fix:**
```typescript
// WRONG — object recreates every render, triggers useEffect infinitely
useEffect(() => { fetchData(filters); }, [filters]);

// RIGHT — depend on specific values
useEffect(() => { fetchData(filters); }, [filters.search, filters.page]);
```

### 19. Memory Leak Warning in Console

**Symptom:** "Can't perform a React state update on an unmounted component."

**Root cause:** Async operation completes after the component unmounts.

**Fix:**
```typescript
useEffect(() => {
  const controller = new AbortController();
  fetch("/api/data", { signal: controller.signal })
    .then((res) => res.json())
    .then(setData)
    .catch((err) => { if (err.name !== "AbortError") throw err; });
  return () => controller.abort();
}, []);
```

## Deployment Issues

### 20. Works Locally, 500 Error in Production

**Symptom:** Everything works in dev, production shows 500 error.

**Root cause:** Missing env var, case-sensitive file import, or Node.js API in Edge runtime.

**Diagnose:** Check Vercel function logs: Dashboard > Project > Logs.

### 21. API Route Returns HTML Instead of JSON

**Symptom:** Frontend shows parsing error. API returns a 404 HTML page.

**Root cause:** API route file is in the wrong location or has wrong export.

**Fix:** Verify the file is at `app/api/your-route/route.ts` (not `page.ts`) and exports named HTTP methods:
```typescript
export async function GET() {
  return Response.json({ status: "ok" });
}
```

### 22. Webhook Not Firing

**Symptom:** Stripe/Supabase webhook events are not being received.

**Root cause:** Webhook URL is wrong, or the endpoint is not handling POST requests.

**Fix:**
1. Check the webhook URL matches your deployed API route exactly
2. Make sure the route exports `POST`:
```typescript
export async function POST(request: Request) {
  const body = await request.text();
  // Process webhook
  return Response.json({ received: true });
}
```

### 23. Cron Job Not Running

**Symptom:** Scheduled task never executes.

**Root cause:** Missing `vercel.json` cron configuration, or the route does not return 200.

**Fix:**
```json
// vercel.json
{ "crons": [{ "path": "/api/cron", "schedule": "0 */6 * * *" }] }
```

### 24. Preview Deploy Shows Old Code

**Symptom:** Pushed new code but preview deployment shows the old version.

**Root cause:** Vercel build cache served old output.

**Fix:** Redeploy with "Override Build Cache" in Vercel dashboard.

## Realtime Issues

### 25. Supabase Realtime Not Updating

**Symptom:** Realtime subscription set up but changes do not appear.

**Root cause:** Table missing `REPLICA IDENTITY FULL` or Realtime not enabled.

**Fix:**
```sql
ALTER TABLE public.messages REPLICA IDENTITY FULL;
```
Then enable in Dashboard > Database > Replication.

### 26. Realtime Works Then Stops

**Symptom:** Updates come through initially, then stop after a few minutes.

**Root cause:** Connection dropped. No reconnect logic.

**Fix:**
```typescript
const channel = supabase.channel("room")
  .on("postgres_changes", { event: "*", schema: "public", table: "messages" }, handler)
  .subscribe((status) => {
    if (status === "CHANNEL_ERROR") {
      setTimeout(() => channel.subscribe(), 5000);
    }
  });
```

## Gemini API Issues

### 27. Gemini Returns Different Response Format

**Symptom:** Gemini API response parsing breaks intermittently.

**Root cause:** Different Gemini models return slightly different response structures.

**Fix:** Always check the response structure:
```typescript
const result = await model.generateContent(prompt);
const text = result.response?.candidates?.[0]?.content?.parts?.[0]?.text ?? "";
```

### 28. Gemini Response Truncated

**Symptom:** AI-generated content cuts off mid-sentence.

**Root cause:** Max output tokens limit hit.

**Fix:**
```typescript
const result = await model.generateContent({
  contents: [{ role: "user", parts: [{ text: prompt }] }],
  generationConfig: { maxOutputTokens: 4096 },
});
```

## Middleware Issues

### 29. Blank Flash Before Page Loads

**Symptom:** Page briefly shows blank before content appears.

**Root cause:** Middleware redirecting, then page loading — causes a visible flash.

**Fix:** Use `rewrite` instead of `redirect` for auth checks where possible, or add a loading state.

### 30. Middleware Runs on Static Assets

**Symptom:** Images, CSS, and JS files are slow to load.

**Root cause:** Middleware matcher is too broad.

**Fix:**
```typescript
export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)"],
};
```

### 31. Middleware Logs Show But Page Does Not Change

**Symptom:** Middleware console.log fires but redirect/rewrite has no effect.

**Root cause:** Not returning the response from middleware.

**Fix:**
```typescript
export async function middleware(request: NextRequest) {
  // Must RETURN the response
  return NextResponse.redirect(new URL("/login", request.url));
}
```
