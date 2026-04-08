# File Splitting Patterns for Next.js

Complete patterns for splitting every common file type in a Next.js + TypeScript + Supabase project.

## Splitting a Large `lib/` Utility File

### Before

```
lib/utils.ts (500+ lines)
  - cn() — className merger
  - generateId() — UUID generator
  - slugify() — string to URL slug
  - formatDate() — date formatting
  - formatCurrency() — money formatting
  - formatRelativeTime() — "2 hours ago"
  - validateEmail() — email check
  - validateABN() — Australian Business Number
  - validatePhone() — phone number check
  - calculatePoints() — scoring logic
  - weightedAverage() — math helper
  - debounce() — input debouncing
  - throttle() — rate limiting
  - sleep() — async delay
```

### After

```
lib/
  utils.ts           → cn(), generateId(), slugify(), sleep()
  formatting.ts      → formatDate(), formatCurrency(), formatRelativeTime()
  validation.ts      → validateEmail(), validateABN(), validatePhone()
  calculations.ts    → calculatePoints(), weightedAverage()
  timing.ts          → debounce(), throttle()
```

### Migration Steps

```typescript
// Step 1: Create lib/formatting.ts
export function formatDate(date: Date | string): string {
  const d = typeof date === "string" ? new Date(date) : date;
  return new Intl.DateTimeFormat("en-AU", {
    day: "numeric",
    month: "short",
    year: "numeric",
  }).format(d);
}

export function formatCurrency(amount: number, currency = "AUD"): string {
  return new Intl.NumberFormat("en-AU", {
    style: "currency",
    currency,
  }).format(amount);
}

export function formatRelativeTime(date: Date | string): string {
  const d = typeof date === "string" ? new Date(date) : date;
  const now = new Date();
  const diffMs = now.getTime() - d.getTime();
  const diffMins = Math.floor(diffMs / 60000);
  if (diffMins < 1) return "just now";
  if (diffMins < 60) return `${diffMins}m ago`;
  const diffHours = Math.floor(diffMins / 60);
  if (diffHours < 24) return `${diffHours}h ago`;
  const diffDays = Math.floor(diffHours / 24);
  if (diffDays < 30) return `${diffDays}d ago`;
  return formatDate(d);
}
```

```typescript
// Step 2: Add re-exports to lib/utils.ts (backward compat)
export { formatDate, formatCurrency, formatRelativeTime } from "./formatting";
```

```typescript
// Step 3: Update consumers one by one
// BEFORE
import { cn, formatDate, formatCurrency } from "@/lib/utils";
// AFTER
import { cn } from "@/lib/utils";
import { formatDate, formatCurrency } from "@/lib/formatting";
```

```bash
# Step 4: Verify
npm run build
```

```typescript
// Step 5: Remove re-exports from lib/utils.ts
// Delete the export { ... } from "./formatting" line
```

## Splitting a Large API Route Handler

### Before

```typescript
// app/api/users/route.ts (400 lines)
// Contains: auth check, validation, 4 different query patterns,
// response formatting, error handling, and type definitions
```

### After

```
app/api/users/
  route.ts           → thin handler, delegates to lib
lib/api/
  with-auth.ts       → auth middleware wrapper
  errors.ts          → standardized error responses
lib/services/
  user-service.ts    → all user business logic
types/
  user.types.ts      → User, CreateUserInput, UpdateUserInput
```

```typescript
// lib/api/errors.ts
import { NextResponse } from "next/server";

export function unauthorized(message = "Unauthorized") {
  return NextResponse.json({ error: message }, { status: 401 });
}

export function badRequest(message: string, details?: Record<string, string[]>) {
  return NextResponse.json({ error: message, details }, { status: 400 });
}

export function notFound(resource = "Resource") {
  return NextResponse.json({ error: `${resource} not found` }, { status: 404 });
}

export function serverError(message = "Internal server error") {
  console.error("Server error:", message);
  return NextResponse.json({ error: message }, { status: 500 });
}
```

```typescript
// lib/services/user-service.ts
import { SupabaseClient } from "@supabase/supabase-js";
import type { CreateUserInput, UpdateUserInput, User } from "@/types/user.types";

export class UserService {
  constructor(private supabase: SupabaseClient) {}

  async getAll(filters?: { role?: string; search?: string }): Promise<User[]> {
    let query = this.supabase.from("profiles").select("*");
    if (filters?.role) query = query.eq("role", filters.role);
    if (filters?.search) query = query.ilike("full_name", `%${filters.search}%`);
    const { data, error } = await query;
    if (error) throw new Error(error.message);
    return data;
  }

  async getById(id: string): Promise<User | null> {
    const { data, error } = await this.supabase
      .from("profiles")
      .select("*")
      .eq("id", id)
      .single();
    if (error) return null;
    return data;
  }

  async update(id: string, input: UpdateUserInput): Promise<User> {
    const { data, error } = await this.supabase
      .from("profiles")
      .update(input)
      .eq("id", id)
      .select()
      .single();
    if (error) throw new Error(error.message);
    return data;
  }
}
```

```typescript
// app/api/users/route.ts (now ~40 lines)
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";
import { UserService } from "@/lib/services/user-service";
import { unauthorized, badRequest, serverError } from "@/lib/api/errors";

export async function GET(request: NextRequest) {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) return unauthorized();

  try {
    const { searchParams } = new URL(request.url);
    const service = new UserService(supabase);
    const users = await service.getAll({
      role: searchParams.get("role") ?? undefined,
      search: searchParams.get("search") ?? undefined,
    });
    return NextResponse.json(users);
  } catch (error) {
    return serverError(error instanceof Error ? error.message : "Failed to fetch users");
  }
}
```

## Splitting a Large Component File

### Before

```
components/settings-page.tsx (800 lines)
  - 12 useState hooks
  - 5 useEffect hooks
  - 3 form sections (profile, notifications, billing)
  - sidebar navigation
  - modal for deleting account
```

### After

```
components/settings/
  index.ts                    → barrel export
  settings-page.tsx           → orchestrator (100 lines)
  settings-nav.tsx            → sidebar navigation
  profile-section.tsx         → profile form
  notification-section.tsx    → notification preferences
  billing-section.tsx         → billing info
  delete-account-modal.tsx    → danger zone modal
hooks/
  use-settings.ts             → all settings state and logic
types/
  settings.types.ts           → settings-related types
```

```typescript
// components/settings/settings-page.tsx (orchestrator)
"use client";
import { useState } from "react";
import { SettingsNav } from "./settings-nav";
import { ProfileSection } from "./profile-section";
import { NotificationSection } from "./notification-section";
import { BillingSection } from "./billing-section";

type SettingsTab = "profile" | "notifications" | "billing";

export function SettingsPage() {
  const [activeTab, setActiveTab] = useState<SettingsTab>("profile");

  return (
    <div className="flex gap-8">
      <SettingsNav activeTab={activeTab} onTabChange={setActiveTab} />
      <div className="flex-1">
        {activeTab === "profile" && <ProfileSection />}
        {activeTab === "notifications" && <NotificationSection />}
        {activeTab === "billing" && <BillingSection />}
      </div>
    </div>
  );
}
```

```typescript
// components/settings/index.ts
export { SettingsPage } from "./settings-page";
```

## Splitting a Large Types File

### Before

```typescript
// lib/types.ts (300 lines — every type in the app)
export interface User { /* ... */ }
export interface Post { /* ... */ }
export interface Comment { /* ... */ }
export interface Notification { /* ... */ }
export interface StripeCustomer { /* ... */ }
export interface StripeSubscription { /* ... */ }
export type VisaSubclass = "189" | "190" | "491";
export type AssessmentStatus = "pending" | "approved" | "rejected";
// ... 30 more types
```

### After

```
types/
  index.ts               → re-exports everything (barrel)
  user.types.ts           → User, UserProfile, UserRole
  post.types.ts           → Post, CreatePostInput
  notification.types.ts   → Notification, NotificationPrefs
  billing.types.ts        → StripeCustomer, StripeSubscription
  visa.types.ts           → VisaSubclass, AssessmentStatus
```

```typescript
// types/index.ts (barrel export)
export type { User, UserProfile, UserRole } from "./user.types";
export type { Post, CreatePostInput } from "./post.types";
export type { Notification, NotificationPrefs } from "./notification.types";
export type { StripeCustomer, StripeSubscription } from "./billing.types";
export type { VisaSubclass, AssessmentStatus } from "./visa.types";
```

**Important:** Always use `export type` for type-only re-exports. This ensures TypeScript erases them at compile time and doesn't cause bundling issues.

## Splitting Zod Schemas

### Before

```typescript
// lib/schemas.ts (400 lines — every Zod schema in the app)
export const userSchema = z.object({ /* ... */ });
export const postSchema = z.object({ /* ... */ });
export const contactSchema = z.object({ /* ... */ });
// ... 15 more schemas
```

### After

```
lib/schemas/
  index.ts            → barrel export
  user.schema.ts      → userSchema, createUserSchema, updateUserSchema
  post.schema.ts      → postSchema, createPostSchema
  contact.schema.ts   → contactSchema
```

**Naming convention:** `*.schema.ts` for Zod schema files. Each schema file exports both the schema and the inferred type:

```typescript
// lib/schemas/user.schema.ts
import { z } from "zod";

export const createUserSchema = z.object({
  email: z.string().email("Invalid email address"),
  fullName: z.string().min(2, "Name must be at least 2 characters").max(100),
  role: z.enum(["user", "admin"]).default("user"),
});

export type CreateUserInput = z.infer<typeof createUserSchema>;

export const updateUserSchema = createUserSchema.partial();
export type UpdateUserInput = z.infer<typeof updateUserSchema>;
```

## Splitting Supabase Queries

### Before

```typescript
// lib/supabase/queries.ts (500 lines — every database query)
export async function getUser(id: string) { /* ... */ }
export async function getUsers() { /* ... */ }
export async function createUser() { /* ... */ }
export async function getPosts() { /* ... */ }
export async function getPostBySlug() { /* ... */ }
// ... 20 more functions
```

### After

```
lib/services/
  user-service.ts     → all user queries
  post-service.ts     → all post queries
  notification-service.ts → all notification queries
```

Each service file groups related database operations. Use a class or plain functions — either works:

```typescript
// lib/services/post-service.ts
import { createClient } from "@/lib/supabase/server";
import type { Post } from "@/types/post.types";

export async function getPosts(limit = 20, offset = 0): Promise<Post[]> {
  const supabase = await createClient();
  const { data, error } = await supabase
    .from("posts")
    .select("*, author:profiles(full_name, avatar_url)")
    .order("created_at", { ascending: false })
    .range(offset, offset + limit - 1);
  if (error) throw new Error(error.message);
  return data;
}

export async function getPostBySlug(slug: string): Promise<Post | null> {
  const supabase = await createClient();
  const { data, error } = await supabase
    .from("posts")
    .select("*, author:profiles(full_name, avatar_url)")
    .eq("slug", slug)
    .single();
  if (error) return null;
  return data;
}
```

## Splitting Constants and Configuration

### Before

```typescript
// lib/constants.ts (200 lines)
export const API_URL = "...";
export const VISA_SUBCLASSES = [...];
export const OCCUPATION_LIST = [...];
export const STATE_OPTIONS = [...];
export const NAV_ITEMS = [...];
export const PLAN_LIMITS = { free: ..., pro: ... };
```

### After

```
lib/constants/
  index.ts           → barrel export
  visa.ts            → VISA_SUBCLASSES, OCCUPATION_LIST
  navigation.ts      → NAV_ITEMS
  plans.ts           → PLAN_LIMITS, FEATURE_FLAGS
  regions.ts         → STATE_OPTIONS, COUNTRY_LIST
config/
  api.ts             → API_URL, API_TIMEOUT, API_RETRY_COUNT
```

## General Rules for File Splits

1. **Each new file should have 50-300 lines.** If your split produces a 20-line file, it's probably too granular. Merge it back.
2. **Name files by what they contain**, not by where they came from. `formatting.ts` is better than `utils-part-2.ts`.
3. **Group by feature in `components/`**, group by type in `lib/`.
4. **Always add barrel exports** for directories with 3+ files that are imported together.
5. **Run `npm run build` after every single file split** — don't batch.
