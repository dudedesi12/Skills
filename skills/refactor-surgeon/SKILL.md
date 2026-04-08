---
name: refactor-surgeon
description: "Use this skill whenever the user mentions refactoring, code splitting, breaking up files, extracting components, modular code, barrel exports, 'file is too big', 'too many lines', monolithic files, dead code, unused code, duplicate code, DRY, code organization, import rewiring, module extraction, 'clean up this file', 'split this into smaller files', component extraction, custom hooks extraction, utility extraction, 'reduce complexity', cyclomatic complexity, code smell, tech debt, technical debt, 'any' types, type safety cleanup, or ANY task involving restructuring existing code without changing behavior. This skill guides you through surgical refactoring — breaking large files into focused modules without breaking imports or functionality."
---

# Refactor Surgeon

You are a senior engineer performing precision refactoring. Your job is to break monolithic files into focused, well-organized modules — without breaking a single import, test, or feature. The user is vibe coding and may not understand module systems deeply, so you handle all the wiring.

## Stack Context

Every project uses this stack unless stated otherwise:
- Next.js App Router (not Pages Router)
- TypeScript
- Tailwind CSS
- Supabase (auth, database, storage, realtime)
- Vercel (deployment)

## Response Format

For EVERY refactoring task, follow this format:

```
DIAGNOSIS: [what's wrong with the current structure — be specific]
RISK: [low/medium/high — what could break]
PLAN: [numbered steps you'll take]
CHANGES: [the actual code changes, file by file]
VERIFY: [how to confirm nothing broke]
```

## Before You Touch Anything

### Step 1: Analyze the File

Before refactoring any file, answer these questions:

1. **How many lines?** Files over 300 lines are candidates for splitting.
2. **How many responsibilities?** Count distinct concerns (data fetching, UI rendering, business logic, types, utilities).
3. **How many exports?** Files with 5+ exports often contain unrelated code.
4. **Who imports this file?** Run a search to find every file that depends on it.
5. **Are there `any` types?** Flag them for replacement during refactoring.

```bash
# Count lines in a file
wc -l path/to/file.ts

# Find all files that import from this module
grep -r "from.*['\"].*module-name" --include="*.ts" --include="*.tsx" src/ app/ lib/

# Count exports
grep -c "^export" path/to/file.ts

# Find 'any' types
grep -n ": any" path/to/file.ts
```

### Step 2: Map Dependencies

Before splitting, understand what depends on what:

```typescript
// Draw this mental map:
// file.ts exports: [FunctionA, FunctionB, TypeC, ConstD, ComponentE]
//   → FunctionA is used by: page1.tsx, page2.tsx
//   → FunctionB is used by: api/route.ts
//   → TypeC is used by: FunctionA, FunctionB, page3.tsx
//   → ConstD is used by: FunctionA only (internal)
//   → ComponentE is used by: page1.tsx, layout.tsx
```

**Rule: Never remove an export until you've confirmed nothing depends on it.**

## File Splitting Strategies

### Strategy 1: Split by Responsibility

The most common refactor. One file does too many things.

**Before — a 600-line `lib/utils.ts`:**
```
lib/utils.ts (600 lines)
  ├── formatDate(), formatCurrency(), formatPhone()    → formatting
  ├── validateEmail(), validatePhone(), validateABN()   → validation
  ├── calculateScore(), weightedAverage()               → calculations
  ├── cn(), generateId(), slugify()                     → general helpers
  └── type Prettify, type DeepPartial                   → utility types
```

**After — focused modules:**
```
lib/
  ├── utils.ts              → cn(), generateId(), slugify() (kept here)
  ├── formatting.ts         → formatDate(), formatCurrency(), formatPhone()
  ├── validation.ts         → validateEmail(), validatePhone(), validateABN()
  ├── calculations.ts       → calculateScore(), weightedAverage()
  └── types/
      └── utility-types.ts  → Prettify, DeepPartial
```

### Strategy 2: Extract Custom Hooks

Components bloated with state logic, effects, and data fetching.

**Before — a 500-line component:**
```tsx
// components/dashboard.tsx (500 lines)
"use client";

export function Dashboard() {
  // 80 lines of state declarations
  const [users, setUsers] = useState([]);
  const [filters, setFilters] = useState({});
  const [search, setSearch] = useState("");
  // ... 20 more useState calls

  // 100 lines of data fetching and effects
  useEffect(() => { /* fetch users */ }, []);
  useEffect(() => { /* filter logic */ }, [filters]);
  useEffect(() => { /* search logic */ }, [search]);

  // 50 lines of handler functions
  const handleFilter = () => { /* ... */ };
  const handleSearch = () => { /* ... */ };
  const handleExport = () => { /* ... */ };

  // 270 lines of JSX
  return <div>...</div>;
}
```

**After — hook + component:**
```tsx
// hooks/use-dashboard.ts
"use client";
import { useState, useEffect, useMemo } from "react";

interface UseDashboardOptions {
  initialFilters?: Record<string, string>;
}

export function useDashboard(options: UseDashboardOptions = {}) {
  const [users, setUsers] = useState<User[]>([]);
  const [filters, setFilters] = useState(options.initialFilters ?? {});
  const [search, setSearch] = useState("");
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    async function fetchUsers() {
      setIsLoading(true);
      const { data } = await supabase.from("users").select("*");
      setUsers(data ?? []);
      setIsLoading(false);
    }
    fetchUsers();
  }, []);

  const filteredUsers = useMemo(() => {
    return users.filter((user) => {
      const matchesSearch = !search || user.name.toLowerCase().includes(search.toLowerCase());
      const matchesFilters = Object.entries(filters).every(
        ([key, value]) => !value || user[key as keyof User] === value
      );
      return matchesSearch && matchesFilters;
    });
  }, [users, search, filters]);

  return {
    users: filteredUsers,
    isLoading,
    search,
    setSearch,
    filters,
    setFilters,
    totalCount: users.length,
    filteredCount: filteredUsers.length,
  };
}
```

```tsx
// components/dashboard.tsx (now ~200 lines — just UI)
"use client";
import { useDashboard } from "@/hooks/use-dashboard";
import { UserTable } from "./dashboard/user-table";
import { DashboardFilters } from "./dashboard/dashboard-filters";
import { SearchBar } from "./dashboard/search-bar";

export function Dashboard() {
  const { users, isLoading, search, setSearch, filters, setFilters } = useDashboard();

  if (isLoading) return <DashboardSkeleton />;

  return (
    <div>
      <SearchBar value={search} onChange={setSearch} />
      <DashboardFilters filters={filters} onChange={setFilters} />
      <UserTable users={users} />
    </div>
  );
}
```

### Strategy 3: Extract Sub-Components

A component renders too many distinct UI sections.

**Rule of thumb:** If a section of JSX has its own heading, its own data, or could be reused — extract it.

```
components/
  settings-page.tsx (800 lines)
    → components/settings/
        ├── settings-page.tsx        (orchestrator — 80 lines)
        ├── profile-section.tsx      (profile form)
        ├── notification-section.tsx  (notification prefs)
        ├── billing-section.tsx      (billing info)
        ├── danger-zone.tsx          (delete account)
        └── settings-nav.tsx         (sidebar nav)
```

### Strategy 4: Extract Types to Dedicated Files

When types are scattered across implementation files.

```
Before:
  lib/api.ts         → has 15 interfaces mixed with functions
  components/form.tsx → has 8 types mixed with component code

After:
  types/
    ├── api.types.ts    → all API-related types
    └── form.types.ts   → all form-related types
  lib/api.ts            → imports types from types/api.types.ts
  components/form.tsx   → imports types from types/form.types.ts
```

**Naming convention:** Always suffix type files with `.types.ts` so they're easy to find.

## Import Rewiring

This is the most error-prone part of refactoring. Follow this process exactly.

### Step 1: Create the New File with Exports

```typescript
// lib/formatting.ts (NEW)
export function formatDate(date: Date): string {
  return new Intl.DateTimeFormat("en-AU", { dateStyle: "medium" }).format(date);
}

export function formatCurrency(amount: number): string {
  return new Intl.NumberFormat("en-AU", { style: "currency", currency: "AUD" }).format(amount);
}
```

### Step 2: Update the Old File — Re-export First

Don't delete from the old file yet. Re-export so existing imports don't break:

```typescript
// lib/utils.ts (OLD) — add re-exports at the bottom
// These re-exports maintain backward compatibility while consumers migrate
export { formatDate, formatCurrency } from "./formatting";
```

### Step 3: Update All Consumers

Find and update every file that imports from the old location:

```typescript
// BEFORE
import { cn, formatDate, formatCurrency } from "@/lib/utils";

// AFTER
import { cn } from "@/lib/utils";
import { formatDate, formatCurrency } from "@/lib/formatting";
```

### Step 4: Remove Re-exports from Old File

Once ALL consumers are updated, remove the re-exports from the old file.

### Step 5: Verify

```bash
# Build the project — TypeScript will catch any broken imports
npm run build

# Run tests if you have them
npm test

# Search for any remaining imports from the old path
grep -r "from.*utils.*format" --include="*.ts" --include="*.tsx" src/ app/ lib/
```

## Barrel Exports (index.ts)

Use barrel exports when a directory has multiple related modules that consumers import together.

```typescript
// components/dashboard/index.ts
export { Dashboard } from "./dashboard";
export { DashboardFilters } from "./dashboard-filters";
export { UserTable } from "./user-table";
export { SearchBar } from "./search-bar";
export { DashboardSkeleton } from "./dashboard-skeleton";
```

Now consumers can import cleanly:

```typescript
// BEFORE (importing from deep paths)
import { Dashboard } from "@/components/dashboard/dashboard";
import { UserTable } from "@/components/dashboard/user-table";

// AFTER (importing from barrel)
import { Dashboard, UserTable } from "@/components/dashboard";
```

**When NOT to use barrel exports:**
- Don't barrel-export server components and client components together — it can cause bundling issues in Next.js
- Don't create barrel exports for directories with 1-2 files — it's overhead for no benefit
- Don't re-export everything — only export the public API

## Detecting and Removing Dead Code

### Find Unused Exports

```bash
# Install ts-prune for dead export detection
npx ts-prune | grep -v "(used in module)"
```

Or do it manually:

```bash
# For each export, search if it's imported anywhere
grep -r "importName" --include="*.ts" --include="*.tsx" src/ app/ lib/ components/
# If zero results (besides the definition), it's dead code
```

### Find Unused Dependencies

```bash
# Check for packages in package.json that aren't imported anywhere
npx depcheck
```

### Common Dead Code Patterns

1. **Commented-out code** — Delete it. Git has history.
2. **Unused imports** — TypeScript/ESLint catches these. Fix all warnings.
3. **Unreachable code** — Code after `return`, `throw`, or `process.exit()`.
4. **Feature flags that shipped** — If a feature flag has been `true` in production for weeks, remove the flag and the old code path.
5. **Unused function parameters** — Prefix with underscore (`_unused`) or remove.

## Fixing `any` Types

Replace `any` types during refactoring. Never leave them behind.

### Common `any` Replacements

```typescript
// BAD: API response typed as any
const data: any = await response.json();

// GOOD: Define the shape
interface ApiResponse {
  users: User[];
  total: number;
  page: number;
}
const data: ApiResponse = await response.json();
```

```typescript
// BAD: Event handler with any
const handleChange = (e: any) => { setValue(e.target.value); };

// GOOD: Use React's event types
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => { setValue(e.target.value); };
```

```typescript
// BAD: Catch block with any
catch (error: any) { console.error(error.message); }

// GOOD: Use unknown and narrow
catch (error: unknown) {
  const message = error instanceof Error ? error.message : "Unknown error";
  console.error(message);
}
```

```typescript
// BAD: Dynamic object with any
const config: any = {};

// GOOD: Use Record or a proper interface
const config: Record<string, string | number | boolean> = {};
// OR
interface AppConfig { apiUrl: string; timeout: number; debug: boolean; }
const config: AppConfig = { apiUrl: "", timeout: 5000, debug: false };
```

See `references/file-splitting-patterns.md` for complete patterns for every file type in a Next.js project.

## Duplicate Code Detection

### Spotting Duplicates

Look for these patterns:

1. **Copy-pasted functions** — Same logic with different variable names
2. **Similar API route handlers** — Routes that all do auth check → validate → query → respond
3. **Repeated UI patterns** — The same card/list/form layout in multiple components
4. **Shared validation logic** — Same Zod schemas defined in multiple places

### Extracting Shared Logic

```typescript
// BEFORE: Same auth check in 10 API routes
export async function GET(request: NextRequest) {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  // ... route-specific logic
}

// AFTER: Shared auth wrapper
// lib/api/with-auth.ts
import { createClient } from "@/lib/supabase/server";
import { NextRequest, NextResponse } from "next/server";

type AuthHandler = (
  request: NextRequest,
  user: User,
  supabase: SupabaseClient
) => Promise<NextResponse>;

export function withAuth(handler: AuthHandler) {
  return async (request: NextRequest) => {
    const supabase = await createClient();
    const { data: { user } } = await supabase.auth.getUser();
    if (!user) {
      return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
    }
    return handler(request, user, supabase);
  };
}

// Usage in any route:
export const GET = withAuth(async (request, user, supabase) => {
  const { data } = await supabase.from("posts").select("*").eq("user_id", user.id);
  return NextResponse.json(data);
});
```

## Refactoring Decision Tree

Use this to decide what to refactor and how:

```
Is the file > 300 lines?
  ├─ YES → Does it have multiple responsibilities?
  │         ├─ YES → Split by responsibility (Strategy 1)
  │         └─ NO  → Is it a component with lots of state logic?
  │                   ├─ YES → Extract custom hooks (Strategy 2)
  │                   └─ NO  → Extract sub-components (Strategy 3)
  └─ NO  → Does it have 'any' types?
            ├─ YES → Fix types in place (no splitting needed)
            └─ NO  → Does it duplicate logic from another file?
                      ├─ YES → Extract shared utility
                      └─ NO  → Leave it alone. Don't refactor for the sake of it.
```

## Rules

1. **Never refactor and add features at the same time** — Refactoring changes structure, not behavior. Do one or the other.
2. **Build after every split** — Run `npm run build` after each file you extract. Don't batch.
3. **One file at a time** — Split one file, verify it works, then move to the next.
4. **Re-export before removing** — Always add re-exports in the old file before updating consumers.
5. **Keep git commits small** — One commit per file split. Makes it easy to revert if something breaks.
6. **Don't create premature abstractions** — If code is only used once, don't extract it into a utility. Three similar instances = time to extract.
7. **Preserve the public API** — Consumers of your module shouldn't need to change their imports unless you're intentionally improving the API.
8. **Test after every change** — If you have tests, run them. If you don't, at minimum do `npm run build`.
9. **Never delete code you're unsure about** — Comment it out first, build, test. Delete in a separate commit.
10. **Fix `any` types as you go** — If you're already touching a file, fix its `any` types. Don't leave them for later.

See `references/` for detailed patterns and checklists:
- `file-splitting-patterns.md` — Complete patterns for splitting every file type in Next.js
- `component-extraction.md` — Step-by-step component and hook extraction with examples
- `refactoring-checklist.md` — Pre/post refactoring verification checklist
