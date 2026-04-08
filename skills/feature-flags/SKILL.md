---
name: feature-flags
description: "Use this skill whenever the user mentions feature flags, feature toggles, LaunchDarkly, Flagsmith, gradual rollout, percentage rollout, canary release, A/B testing flags, user segmentation, 'enable for some users', kill switch, 'turn feature on/off', environment flags, flag management, dark launch, beta features, 'only show to admins', experiment, experimentation, 'release gradually', or ANY feature flag/toggle task — even if they don't explicitly say 'feature flag'. This skill lets you release features safely and gradually."
---

# Feature Flags

Release features gradually, test with subsets of users, and kill-switch broken features instantly. This skill builds a custom Supabase-based flag system. Cross-reference: vercel-power-user §5 for middleware-based A/B testing.

## 1. Custom Feature Flag System

### Database Schema

```sql
-- supabase/migrations/xxx_feature_flags.sql
CREATE TABLE feature_flags (
  id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  name text NOT NULL UNIQUE,
  description text,
  enabled boolean DEFAULT false,
  rollout_percentage integer DEFAULT 0 CHECK (rollout_percentage >= 0 AND rollout_percentage <= 100),
  allowed_roles text[] DEFAULT '{}',
  allowed_user_ids uuid[] DEFAULT '{}',
  environments text[] DEFAULT '{production, preview, development}',
  metadata jsonb DEFAULT '{}',
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now()
);

-- Audit log for flag changes
CREATE TABLE flag_audit_log (
  id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  flag_name text NOT NULL,
  action text NOT NULL, -- 'created', 'updated', 'deleted'
  changed_by uuid REFERENCES auth.users(id),
  old_value jsonb,
  new_value jsonb,
  created_at timestamptz DEFAULT now()
);

-- Seed some flags
INSERT INTO feature_flags (name, description, enabled, rollout_percentage) VALUES
  ('new-assessment-ui', 'Redesigned assessment interface', true, 10),
  ('ai-recommendations', 'AI-powered visa recommendations', true, 100),
  ('dark-mode', 'Dark mode theme', false, 0),
  ('bulk-export', 'Bulk data export for admin', true, 100);

-- RLS: Anyone can read flags, only admins can modify
ALTER TABLE feature_flags ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Anyone can read flags" ON feature_flags FOR SELECT USING (true);
CREATE POLICY "Admins can manage flags" ON feature_flags FOR ALL
  USING (EXISTS (SELECT 1 FROM profiles WHERE id = auth.uid() AND role = 'admin'));
```

### Flag Evaluation

```typescript
// lib/flags/evaluate.ts
import { createClient } from "@/lib/supabase/server";
import crypto from "crypto";

interface FlagContext {
  userId?: string;
  role?: string;
  environment?: string;
}

export async function isFeatureEnabled(flagName: string, context: FlagContext = {}): Promise<boolean> {
  const supabase = await createClient();
  const { data: flag } = await supabase
    .from("feature_flags")
    .select("*")
    .eq("name", flagName)
    .single();

  if (!flag || !flag.enabled) return false;

  // Check environment
  const env = context.environment ?? process.env.VERCEL_ENV ?? "development";
  if (flag.environments.length > 0 && !flag.environments.includes(env)) return false;

  // Check allowed user IDs (explicit allowlist)
  if (flag.allowed_user_ids.length > 0 && context.userId) {
    if (flag.allowed_user_ids.includes(context.userId)) return true;
  }

  // Check allowed roles
  if (flag.allowed_roles.length > 0 && context.role) {
    if (flag.allowed_roles.includes(context.role)) return true;
    if (flag.allowed_roles.length > 0) return false; // Role required but doesn't match
  }

  // Percentage rollout (deterministic per user)
  if (flag.rollout_percentage < 100 && context.userId) {
    const hash = crypto.createHash("md5").update(`${flagName}:${context.userId}`).digest("hex");
    const bucket = parseInt(hash.substring(0, 8), 16) % 100;
    return bucket < flag.rollout_percentage;
  }

  return flag.rollout_percentage === 100;
}
```

## 2. React Components

### FeatureFlag Wrapper

```tsx
// components/feature-flag.tsx
import { isFeatureEnabled } from "@/lib/flags/evaluate";
import { createClient } from "@/lib/supabase/server";

interface FeatureFlagProps {
  name: string;
  children: React.ReactNode;
  fallback?: React.ReactNode;
}

export async function FeatureFlag({ name, children, fallback = null }: FeatureFlagProps) {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();

  let role: string | undefined;
  if (user) {
    const { data: profile } = await supabase
      .from("profiles").select("role").eq("id", user.id).single();
    role = profile?.role;
  }

  const enabled = await isFeatureEnabled(name, {
    userId: user?.id,
    role,
  });

  return enabled ? <>{children}</> : <>{fallback}</>;
}

// Usage in a server component:
<FeatureFlag name="new-assessment-ui" fallback={<OldAssessmentUI />}>
  <NewAssessmentUI />
</FeatureFlag>
```

### Client-Side Hook

```tsx
// hooks/use-feature-flag.ts
"use client";
import { useEffect, useState } from "react";

export function useFeatureFlag(flagName: string): { enabled: boolean; loading: boolean } {
  const [enabled, setEnabled] = useState(false);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch(`/api/flags/${flagName}`)
      .then((res) => res.json())
      .then((data) => { setEnabled(data.enabled); setLoading(false); })
      .catch(() => { setEnabled(false); setLoading(false); });
  }, [flagName]);

  return { enabled, loading };
}

// API route for client-side flag checks
// app/api/flags/[name]/route.ts
export async function GET(request: NextRequest, { params }: { params: Promise<{ name: string }> }) {
  const { name } = await params;
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();

  const enabled = await isFeatureEnabled(name, { userId: user?.id });
  return NextResponse.json({ enabled });
}
```

## 3. Percentage Rollouts

Gradually increase traffic to a new feature:

```
Day 1:  1% (internal testing)
Day 3:  10% (early adopters)
Day 5:  25%
Day 7:  50%
Day 10: 100% (full rollout)
```

```typescript
// Admin API to update rollout percentage
// app/api/admin/flags/[name]/route.ts
export async function PATCH(request: NextRequest, { params }: { params: Promise<{ name: string }> }) {
  const { name } = await params;
  const body = await request.json();

  const supabase = await createClient();

  // Get old value for audit log
  const { data: oldFlag } = await supabase
    .from("feature_flags").select("*").eq("name", name).single();

  // Update flag
  const { data, error } = await supabase
    .from("feature_flags")
    .update({ ...body, updated_at: new Date().toISOString() })
    .eq("name", name)
    .select()
    .single();

  if (error) return NextResponse.json({ error: error.message }, { status: 400 });

  // Audit log
  const { data: { user } } = await supabase.auth.getUser();
  await supabase.from("flag_audit_log").insert({
    flag_name: name,
    action: "updated",
    changed_by: user?.id,
    old_value: oldFlag,
    new_value: data,
  });

  return NextResponse.json(data);
}
```

## 4. Kill Switch

Instantly disable a broken feature:

```typescript
// lib/flags/kill-switch.ts
export async function killFeature(flagName: string, reason: string) {
  const supabase = await createClient();

  await supabase.from("feature_flags").update({
    enabled: false,
    metadata: { killed_at: new Date().toISOString(), kill_reason: reason },
    updated_at: new Date().toISOString(),
  }).eq("name", flagName);

  // Invalidate any cached flag values
  const { redis } = await import("@/lib/cache/redis");
  await redis.del(`flag:${flagName}`);

  // Alert the team
  const { sendAlert } = await import("@/lib/monitoring/alerts");
  await sendAlert("warning", `Feature killed: ${flagName}`, { reason });
}
```

## 5. Environment-Based Flags

```typescript
// Different flag values per environment
const FLAG_OVERRIDES: Record<string, Record<string, boolean>> = {
  development: {
    "debug-panel": true,
    "mock-payments": true,
  },
  preview: {
    "debug-panel": true,
    "mock-payments": true,
  },
  production: {
    "debug-panel": false,
    "mock-payments": false,
  },
};

export function getEnvironmentOverride(flagName: string): boolean | undefined {
  const env = process.env.VERCEL_ENV ?? "development";
  return FLAG_OVERRIDES[env]?.[flagName];
}
```

## 6. A/B Testing Integration

```typescript
// Track which variant a user sees
export async function trackFlagExposure(
  flagName: string,
  userId: string,
  variant: "control" | "treatment"
) {
  // Log to analytics (cross-reference: analytics-dashboard)
  await fetch("/api/analytics/track", {
    method: "POST",
    body: JSON.stringify({
      event: "flag_exposure",
      properties: { flag: flagName, variant, userId },
    }),
  });
}

// Usage with flag evaluation
const enabled = await isFeatureEnabled("new-assessment-ui", { userId });
await trackFlagExposure("new-assessment-ui", userId, enabled ? "treatment" : "control");
```

## 7. Flag Cleanup

Flags that have been at 100% for 2+ weeks should be removed:

```sql
-- Find stale flags (fully rolled out for 2+ weeks)
SELECT name, rollout_percentage, updated_at
FROM feature_flags
WHERE enabled = true
  AND rollout_percentage = 100
  AND updated_at < now() - interval '14 days'
ORDER BY updated_at;
```

**Cleanup process:**
1. Confirm the feature is stable (no errors)
2. Remove the `<FeatureFlag>` wrapper from code
3. Keep only the treatment code, delete the control
4. Delete the flag from the database
5. Commit: `refactor: remove feature flag for [feature-name]`

## Rules

1. **Every flag has an owner** — Someone is responsible for monitoring and cleaning up.
2. **Flags are temporary** — Set a cleanup date when creating the flag.
3. **Use deterministic rollouts** — Same user always sees the same variant (hash user ID).
4. **Log flag changes** — Audit log every enable/disable/percentage change.
5. **Kill switch everything** — Every feature behind a flag can be instantly disabled.
6. **Don't nest flags** — Flag A controlling whether Flag B is checked = debugging nightmare.
7. **Clean up after rollout** — At 100% for 2 weeks? Remove the flag and the conditional code.
8. **Test both paths** — When adding a flag, test both enabled and disabled states.

See `references/` for detailed guides:
- `custom-flag-system.md` — Full Supabase-based system with admin UI
- `rollout-patterns.md` — Gradual rollout strategies
- `ab-testing-guide.md` — Flag-driven A/B testing with analytics
