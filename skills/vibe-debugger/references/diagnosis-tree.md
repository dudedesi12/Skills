# Diagnosis Tree

A decision tree for narrowing down invisible bugs. Start at the top and follow the branches.

## Start Here

```
Something is not working right (no error message)
│
├── Does it work LOCALLY?
│   │
│   ├── YES (works locally, broken in production)
│   │   │
│   │   ├── Check: Environment variables set in Vercel?
│   │   │   ├── NO → Add env vars in Vercel dashboard, redeploy
│   │   │   └── YES ↓
│   │   │
│   │   ├── Check: File name case sensitivity?
│   │   │   ├── Mismatch found → Rename file to match import exactly
│   │   │   └── All match ↓
│   │   │
│   │   ├── Check: Using Node.js APIs in Edge runtime?
│   │   │   ├── YES → Move to Node runtime or use Web APIs
│   │   │   └── NO ↓
│   │   │
│   │   ├── Check: Hardcoded localhost URLs?
│   │   │   ├── YES → Use relative URLs or env-based URLs
│   │   │   └── NO ↓
│   │   │
│   │   └── Check: Tailwind classes being purged?
│   │       ├── YES → Use full class names, check content config
│   │       └── NO → Check Vercel function logs for hidden errors
│   │
│   └── NO (broken locally too)
│       │
│       ├── Is the page COMPLETELY BLANK?
│       │   │
│       │   ├── YES
│       │   │   ├── Check browser console (F12) for errors
│       │   │   ├── Check: Does the component have a default export?
│       │   │   ├── Check: Is error.tsx catching a silent error?
│       │   │   └── Check: Is middleware redirecting?
│       │   │
│       │   └── NO (page loads but something is wrong)
│       │       │
│       │       ├── DATA issue (wrong data, missing data, stale data)
│       │       │   │
│       │       │   ├── No data at all?
│       │       │   │   ├── Check: Supabase RLS policies exist?
│       │       │   │   ├── Check: Query has correct table/column names?
│       │       │   │   └── Check: User is authenticated?
│       │       │   │
│       │       │   ├── Wrong data?
│       │       │   │   ├── Check: Query filters correct?
│       │       │   │   ├── Check: Data types match (string vs number)?
│       │       │   │   └── Check: Joins included in select?
│       │       │   │
│       │       │   └── Stale data?
│       │       │       ├── Clear .next cache: rm -rf .next && npm run dev
│       │       │       ├── Check: revalidatePath called after mutations?
│       │       │       └── Check: ISR revalidate interval too long?
│       │       │
│       │       ├── STYLE issue (looks wrong, layout broken)
│       │       │   │
│       │       │   ├── Check: Dynamic Tailwind classes?
│       │       │   │   └── Use full class names, not string templates
│       │       │   │
│       │       │   ├── Check: Missing responsive breakpoints?
│       │       │   │   └── Test at all widths with DevTools
│       │       │   │
│       │       │   └── Check: CSS specificity conflict?
│       │       │       └── Use DevTools > Elements > Styles to see applied rules
│       │       │
│       │       ├── INTERACTION issue (clicks do nothing, forms don't submit)
│       │       │   │
│       │       │   ├── Check: "use client" directive on interactive components?
│       │       │   ├── Check: onClick handler exists and is spelled correctly?
│       │       │   ├── Check: Element covered by invisible overlay (z-index)?
│       │       │   ├── Check: Form action/onSubmit wired up?
│       │       │   └── Check: Button type="submit" vs type="button"?
│       │       │
│       │       └── PERFORMANCE issue (slow, freezing, loading forever)
│       │           │
│       │           ├── Loading never finishes?
│       │           │   ├── Check: Missing await on async function?
│       │           │   ├── Check: .finally() to clear loading state?
│       │           │   └── Check: API endpoint returning response?
│       │           │
│       │           ├── Page freezes?
│       │           │   ├── Check: Infinite useEffect loop?
│       │           │   ├── Check: Huge data set rendered without pagination?
│       │           │   └── Check: Recursive function without base case?
│       │           │
│       │           └── Slow but works eventually?
│       │               ├── Check: Sequential fetches → use Promise.all
│       │               ├── Check: No caching → add revalidate
│       │               └── Check: Cold start → use Edge runtime
│
└── SPECIAL CASES
    │
    ├── "It was working yesterday"
    │   ├── Check: git log — what changed since yesterday?
    │   ├── Check: Did a dependency update? (check package-lock.json diff)
    │   ├── Check: Did Supabase/Vercel have an incident?
    │   └── Check: Did a free-tier limit get hit?
    │
    ├── "It works sometimes but not always"
    │   ├── Check: Race condition (timing-dependent)?
    │   ├── Check: Supabase connection pool exhausted?
    │   ├── Check: Rate limiting on external API?
    │   └── Check: Edge function cold start (first request slow)?
    │
    └── "It works for me but not for others"
        ├── Check: Auth role differences?
        ├── Check: RLS policies filter by user_id?
        ├── Check: Browser-specific issues (Safari vs Chrome)?
        └── Check: Ad blocker or browser extension interference?
```

## Quick Diagnostic Commands

### Check if it is a cache problem
```bash
rm -rf .next && npm run dev
```

### Check if it is an env var problem
```typescript
console.log("ENV:", {
  supabaseUrl: !!process.env.NEXT_PUBLIC_SUPABASE_URL,
  supabaseKey: !!process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY,
  serviceRole: !!process.env.SUPABASE_SERVICE_ROLE_KEY,
});
```

### Check if it is a data/RLS problem
```typescript
// Bypass RLS with service role
const admin = createClient(url, process.env.SUPABASE_SERVICE_ROLE_KEY!);
const { data, error } = await admin.from("table").select("*");
console.log("Admin data:", data, "Error:", error);
```

### Check if it is a timing problem
```typescript
console.time("fetch");
const data = await fetchData();
console.timeEnd("fetch");
console.log("Data received:", !!data);
```

### Check if it is a scope problem
```typescript
console.log("Context:", typeof window === "undefined" ? "SERVER" : "CLIENT");
```
