# Browser and Runtime Errors (20+)

Every common browser/runtime error with plain English explanation and exact fix.

## CORS Errors

### 1. Access to fetch at 'X' has been blocked by CORS policy

**Why:** The external API does not allow requests from your website's domain. Browsers block this for security.

**Fix:** Create a Next.js API route as a proxy:
```typescript
// app/api/proxy/[...path]/route.ts
import { NextRequest } from "next/server";

export async function GET(request: NextRequest) {
  const url = new URL(request.url);
  const targetPath = url.pathname.replace("/api/proxy/", "");
  const res = await fetch(`https://external-api.com/${targetPath}`, {
    headers: { "Authorization": `Bearer ${process.env.API_KEY}` },
  });
  const data = await res.json();
  return Response.json(data);
}
```

### 2. CORS error with Supabase

**Why:** Supabase's URL or anon key is wrong, or the request is misconfigured.

**Fix:** Check that `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_ANON_KEY` are correct. Supabase handles CORS automatically when using the client library.

### 3. CORS preflight request fails

**Why:** The browser sends an OPTIONS request first, and the server is not handling it.

**Fix:**
```typescript
// app/api/webhook/route.ts
export async function OPTIONS() {
  return new Response(null, {
    status: 200,
    headers: {
      "Access-Control-Allow-Origin": "*",
      "Access-Control-Allow-Methods": "POST, OPTIONS",
      "Access-Control-Allow-Headers": "Content-Type, Authorization",
    },
  });
}
```

## SSR / Hydration Errors

### 4. ReferenceError: window is not defined

**Why:** You used `window` in a Server Component. Servers do not have a browser window.

**Fix:**
```typescript
// Option 1: Mark as client component
"use client";

// Option 2: Dynamic import with no SSR
import dynamic from "next/dynamic";
const Chart = dynamic(() => import("./Chart"), { ssr: false });

// Option 3: Guard with typeof check
if (typeof window !== "undefined") {
  window.scrollTo(0, 0);
}
```

### 5. ReferenceError: document is not defined

**Why:** Same as above — `document` only exists in the browser.

**Fix:** Same solutions as `window is not defined`. Use `"use client"` or dynamic import.

### 6. ReferenceError: localStorage is not defined

**Why:** `localStorage` only exists in the browser.

**Fix:**
```typescript
"use client";
import { useEffect, useState } from "react";

function useLocalStorage(key: string, initialValue: string) {
  const [value, setValue] = useState(initialValue);
  useEffect(() => {
    const stored = localStorage.getItem(key);
    if (stored) setValue(stored);
  }, [key]);
  return [value, setValue] as const;
}
```

### 7. Hydration error: Expected server HTML to contain a matching element

**Why:** Server and client rendered different HTML. Often caused by browser extensions or conditional rendering based on client state.

**Fix:**
```typescript
"use client";
import { useEffect, useState } from "react";

function ClientOnly({ children }: { children: React.ReactNode }) {
  const [mounted, setMounted] = useState(false);
  useEffect(() => setMounted(true), []);
  if (!mounted) return null;
  return <>{children}</>;
}
```

## Network Errors

### 8. TypeError: Failed to fetch

**Why:** Network request failed. Could be: server is down, no internet, wrong URL, or CORS.

**Fix:**
```typescript
try {
  const res = await fetch("/api/data");
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  const data = await res.json();
} catch (error) {
  if (error instanceof TypeError) {
    // Network error — show offline message
    console.error("Network error — check your connection");
  }
}
```

### 9. AbortError: The operation was aborted

**Why:** A fetch request was cancelled, usually by navigation or a timeout.

**Fix:** This is often intentional (component unmounted). Handle gracefully:
```typescript
useEffect(() => {
  const controller = new AbortController();
  fetch("/api/data", { signal: controller.signal })
    .then((res) => res.json())
    .then(setData)
    .catch((err) => {
      if (err.name !== "AbortError") console.error(err);
    });
  return () => controller.abort();
}, []);
```

### 10. SyntaxError: Unexpected token '<' (in JSON at position 0)

**Why:** You expected JSON but got HTML back (usually a 404 page or error page).

**Fix:** Check the API URL is correct. The server returned an HTML error page instead of JSON:
```typescript
const res = await fetch("/api/data");
if (!res.ok) {
  throw new Error(`API returned ${res.status}: ${res.statusText}`);
}
const data = await res.json(); // Only parse if response is OK
```

### 11. Error: 429 Too Many Requests

**Why:** You hit the rate limit of an API.

**Fix:** Add rate limiting to your client, or use caching:
```typescript
// Cache API responses
const res = await fetch("/api/data", { next: { revalidate: 60 } }); // Cache for 60s
```

## DOM Errors

### 12. TypeError: Cannot read properties of null (reading 'X')

**Why:** You tried to access a property on something that is null. Usually a DOM element that does not exist yet.

**Fix:**
```typescript
// Wrong
document.getElementById("my-element").textContent = "Hello";
// Right — check for null
const element = document.getElementById("my-element");
if (element) element.textContent = "Hello";
```

### 13. Error: Maximum update depth exceeded

**Why:** Infinite re-render loop in React. A state update triggers a re-render that triggers the same state update.

**Fix:**
```typescript
// Wrong — runs on every render
useEffect(() => {
  setCount(count + 1);
}); // Missing dependency array!

// Right — runs once
useEffect(() => {
  setCount((prev) => prev + 1);
}, []); // Empty array = run once
```

### 14. Warning: Each child in a list should have a unique "key" prop

**Why:** React needs unique keys to track list items efficiently.

**Fix:**
```typescript
// Wrong
{items.map((item) => <div>{item.name}</div>)}
// Right
{items.map((item) => <div key={item.id}>{item.name}</div>)}
// Never use index as key for dynamic lists
```

### 15. Warning: Can't perform a React state update on an unmounted component

**Why:** An async operation completed after the component was removed from the page.

**Fix:**
```typescript
useEffect(() => {
  const controller = new AbortController();
  fetchData(controller.signal).then(setData);
  return () => controller.abort(); // Cancel on unmount
}, []);
```

## Console Warnings

### 16. Warning: Invalid DOM property 'class'. Did you mean 'className'?

**Why:** React uses `className` instead of `class` for CSS classes.

**Fix:**
```typescript
// Wrong
<div class="container">
// Right
<div className="container">
```

### 17. Warning: Received 'true' for a non-boolean attribute

**Why:** Passing a boolean to an HTML attribute that expects a string.

**Fix:**
```typescript
// Wrong
<div hidden={true}>
// Right
<div hidden>
// Or for custom attributes
<div data-active={String(isActive)}>
```

## Performance Errors

### 18. Error: Out of memory — JavaScript heap

**Why:** Your app is using too much memory. Common causes: infinite loops, huge data sets, memory leaks.

**Fix:** Check for:
- Infinite loops in `useEffect`
- Storing huge arrays in state
- Not cleaning up subscriptions/intervals
```typescript
useEffect(() => {
  const interval = setInterval(fetchData, 5000);
  return () => clearInterval(interval); // Clean up!
}, []);
```

### 19. Uncaught RangeError: Maximum call stack size exceeded

**Why:** Infinite recursion — a function calls itself forever.

**Fix:** Check for recursive functions without a base case, or circular object references in JSON.stringify.

## Auth / Cookie Errors

### 20. Error: Cookies are blocked or not supported by your browser

**Why:** Third-party cookies are blocked (common in Safari and Firefox).

**Fix:** Use first-party cookies. Supabase SSR package handles this correctly:
```bash
npm install @supabase/ssr
```

### 21. Error: SecurityError — access to localStorage denied

**Why:** Browser is in private/incognito mode and localStorage is disabled.

**Fix:** Wrap localStorage access in try/catch:
```typescript
function safeGetItem(key: string): string | null {
  try {
    return localStorage.getItem(key);
  } catch {
    return null;
  }
}
```

### 22. Mixed content warning — loading HTTP resource on HTTPS page

**Why:** Your HTTPS site is trying to load something over HTTP.

**Fix:** Use HTTPS URLs everywhere. Check image sources, API endpoints, and script tags:
```typescript
// Wrong
const apiUrl = "http://api.example.com";
// Right
const apiUrl = "https://api.example.com";
```
