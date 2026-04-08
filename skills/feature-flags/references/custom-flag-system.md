# Custom Feature Flag System with Supabase

## Admin UI Page

```tsx
// app/admin/flags/page.tsx
import { createClient } from "@/lib/supabase/server";
import { FlagList } from "./flag-list";

export default async function FlagsPage() {
  const supabase = await createClient();
  const { data: flags } = await supabase
    .from("feature_flags")
    .select("*")
    .order("name");

  return (
    <div className="mx-auto max-w-4xl p-8">
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-bold">Feature Flags</h1>
        <a href="/admin/flags/new" className="rounded bg-blue-600 px-4 py-2 text-white hover:bg-blue-700">
          New Flag
        </a>
      </div>
      <FlagList flags={flags ?? []} />
    </div>
  );
}
```

```tsx
// app/admin/flags/flag-list.tsx
"use client";
import { useState } from "react";

interface Flag {
  id: string;
  name: string;
  description: string;
  enabled: boolean;
  rollout_percentage: number;
  environments: string[];
  updated_at: string;
}

export function FlagList({ flags: initialFlags }: { flags: Flag[] }) {
  const [flags, setFlags] = useState(initialFlags);

  async function toggleFlag(name: string, enabled: boolean) {
    const res = await fetch(`/api/admin/flags/${name}`, {
      method: "PATCH",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ enabled }),
    });

    if (res.ok) {
      setFlags((prev) => prev.map((f) => f.name === name ? { ...f, enabled } : f));
    }
  }

  async function updateRollout(name: string, percentage: number) {
    await fetch(`/api/admin/flags/${name}`, {
      method: "PATCH",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ rollout_percentage: percentage }),
    });

    setFlags((prev) => prev.map((f) => f.name === name ? { ...f, rollout_percentage: percentage } : f));
  }

  return (
    <div className="mt-6 space-y-4">
      {flags.map((flag) => (
        <div key={flag.id} className="rounded border p-4">
          <div className="flex items-center justify-between">
            <div>
              <h3 className="font-semibold">{flag.name}</h3>
              <p className="text-sm text-gray-500">{flag.description}</p>
            </div>
            <label className="relative inline-flex cursor-pointer items-center">
              <input
                type="checkbox"
                checked={flag.enabled}
                onChange={(e) => toggleFlag(flag.name, e.target.checked)}
                className="peer sr-only"
              />
              <div className="h-6 w-11 rounded-full bg-gray-200 peer-checked:bg-blue-600 peer-focus:ring-4" />
            </label>
          </div>
          {flag.enabled && (
            <div className="mt-3">
              <label className="text-sm text-gray-600">
                Rollout: {flag.rollout_percentage}%
              </label>
              <input
                type="range"
                min={0}
                max={100}
                value={flag.rollout_percentage}
                onChange={(e) => updateRollout(flag.name, Number(e.target.value))}
                className="mt-1 w-full"
              />
            </div>
          )}
          <div className="mt-2 flex gap-2">
            {flag.environments.map((env) => (
              <span key={env} className="rounded bg-gray-100 px-2 py-0.5 text-xs text-gray-600">
                {env}
              </span>
            ))}
          </div>
        </div>
      ))}
    </div>
  );
}
```

## Flag with Redis Caching

Cache flag evaluations to avoid database queries on every request:

```typescript
// lib/flags/cached-evaluate.ts
import { redis } from "@/lib/cache/redis";
import { isFeatureEnabled } from "./evaluate";

export async function cachedIsFeatureEnabled(
  flagName: string,
  context: { userId?: string; role?: string }
): Promise<boolean> {
  const cacheKey = `flag:${flagName}:${context.userId ?? "anon"}`;
  const cached = await redis.get<boolean>(cacheKey);
  if (cached !== null) return cached;

  const enabled = await isFeatureEnabled(flagName, context);
  await redis.set(cacheKey, enabled, { ex: 60 }); // Cache for 1 minute
  return enabled;
}
```

## Create Flag API

```typescript
// app/api/admin/flags/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";
import { z } from "zod";

const createFlagSchema = z.object({
  name: z.string().regex(/^[a-z0-9-]+$/, "Use kebab-case"),
  description: z.string().optional(),
  enabled: z.boolean().default(false),
  rollout_percentage: z.number().int().min(0).max(100).default(0),
  environments: z.array(z.string()).default(["production", "preview", "development"]),
});

export async function POST(request: NextRequest) {
  const supabase = await createClient();

  const body = await request.json();
  const parsed = createFlagSchema.safeParse(body);
  if (!parsed.success) {
    return NextResponse.json({ error: parsed.error.flatten() }, { status: 400 });
  }

  const { data, error } = await supabase
    .from("feature_flags")
    .insert(parsed.data)
    .select()
    .single();

  if (error) return NextResponse.json({ error: error.message }, { status: 400 });

  // Audit log
  const { data: { user } } = await supabase.auth.getUser();
  await supabase.from("flag_audit_log").insert({
    flag_name: parsed.data.name,
    action: "created",
    changed_by: user?.id,
    new_value: data,
  });

  return NextResponse.json(data, { status: 201 });
}
```
