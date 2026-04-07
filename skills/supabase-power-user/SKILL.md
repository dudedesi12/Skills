---
name: supabase-power-user
description: >-
  Use this skill whenever the user mentions advanced database operations, complex RLS policies,
  database functions, triggers, materialized views, full-text search, connection pooling, pg_cron,
  database performance, query optimization, migrations, multi-tenant, role-based access, audit
  logging, or any complex Supabase operation beyond basic CRUD. Even if they just say 'my queries
  are slow' or 'how do I handle permissions' or 'I need audit logs' — use this skill.
---

# Supabase Power User

This skill covers advanced Supabase and PostgreSQL patterns for production Next.js App Router applications. Every example uses TypeScript, Supabase, Tailwind CSS, and deploys to Vercel. AI features use the Gemini API (NEVER OpenAI).

## Stack

- **Framework:** Next.js App Router (TypeScript)
- **Database:** Supabase (PostgreSQL)
- **Styling:** Tailwind CSS
- **Deployment:** Vercel
- **AI:** Gemini API

---

## 1. Advanced Row Level Security (RLS)

RLS is how you secure data in Supabase. Every table with user data MUST have RLS enabled. See `references/advanced-rls.md` for 20+ policy templates.

### Multi-Tenant Isolation

Every row belongs to an organization. Users can only see rows from their org.

```sql
-- file: supabase/migrations/001_multi_tenant_rls.sql

-- Enable RLS on the table
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

-- Helper function: get the user's org_id from their JWT
CREATE OR REPLACE FUNCTION public.get_user_org_id()
RETURNS uuid
LANGUAGE sql
STABLE
SECURITY DEFINER
SET search_path = public
AS $$
  SELECT org_id
  FROM public.profiles
  WHERE id = auth.uid();
$$;

-- Policy: users can only see rows from their own organization
CREATE POLICY "Tenant isolation: users see own org data"
  ON projects
  FOR SELECT
  USING (org_id = public.get_user_org_id());

-- Policy: users can only insert rows into their own org
CREATE POLICY "Tenant isolation: users insert own org data"
  ON projects
  FOR INSERT
  WITH CHECK (org_id = public.get_user_org_id());

-- Policy: users can only update rows in their own org
CREATE POLICY "Tenant isolation: users update own org data"
  ON projects
  FOR UPDATE
  USING (org_id = public.get_user_org_id())
  WITH CHECK (org_id = public.get_user_org_id());

-- Policy: users can only delete rows in their own org
CREATE POLICY "Tenant isolation: users delete own org data"
  ON projects
  FOR DELETE
  USING (org_id = public.get_user_org_id());
```

### Role Hierarchy (admin > mod > user)

```sql
-- file: supabase/migrations/002_role_hierarchy_rls.sql

-- Helper function: get user's role
CREATE OR REPLACE FUNCTION public.get_user_role()
RETURNS text
LANGUAGE sql
STABLE
SECURITY DEFINER
SET search_path = public
AS $$
  SELECT role
  FROM public.profiles
  WHERE id = auth.uid();
$$;

-- Helper function: check if user has at least a given role level
CREATE OR REPLACE FUNCTION public.has_role_level(required_role text)
RETURNS boolean
LANGUAGE plpgsql
STABLE
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  user_role text;
  role_levels jsonb := '{"user": 1, "mod": 2, "admin": 3}'::jsonb;
BEGIN
  SELECT role INTO user_role FROM public.profiles WHERE id = auth.uid();

  IF user_role IS NULL THEN
    RETURN false;
  END IF;

  RETURN (role_levels->>user_role)::int >= (role_levels->>required_role)::int;
END;
$$;

-- Admins can do everything
CREATE POLICY "Admins full access"
  ON projects
  FOR ALL
  USING (public.has_role_level('admin'));

-- Mods can read and update (but not delete)
CREATE POLICY "Mods can read"
  ON projects
  FOR SELECT
  USING (public.has_role_level('mod') AND org_id = public.get_user_org_id());

CREATE POLICY "Mods can update"
  ON projects
  FOR UPDATE
  USING (public.has_role_level('mod') AND org_id = public.get_user_org_id());

-- Regular users can only read their own rows
CREATE POLICY "Users read own rows"
  ON projects
  FOR SELECT
  USING (
    public.get_user_role() = 'user'
    AND org_id = public.get_user_org_id()
    AND created_by = auth.uid()
  );
```

### Time-Based Access

```sql
-- file: supabase/migrations/003_time_based_rls.sql

-- Only allow access to content after its publish date
CREATE POLICY "Published content only"
  ON articles
  FOR SELECT
  USING (
    published_at IS NOT NULL
    AND published_at <= now()
  );

-- Allow access only during business hours (UTC)
CREATE POLICY "Business hours only"
  ON sensitive_reports
  FOR SELECT
  USING (
    EXTRACT(DOW FROM now()) BETWEEN 1 AND 5
    AND EXTRACT(HOUR FROM now()) BETWEEN 9 AND 17
  );
```

---

## 2. PL/pgSQL Database Functions

Move business logic into the database for atomicity and performance. See `references/plpgsql-functions.md` for 15+ production functions.

### Atomic Team Invite with Validation

```sql
-- file: supabase/migrations/004_invite_function.sql

CREATE OR REPLACE FUNCTION public.invite_team_member(
  p_org_id uuid,
  p_email text,
  p_role text DEFAULT 'user'
)
RETURNS jsonb
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_inviter_role text;
  v_existing_member uuid;
  v_invite_id uuid;
BEGIN
  -- Check the inviter has permission
  SELECT role INTO v_inviter_role
  FROM profiles
  WHERE id = auth.uid() AND org_id = p_org_id;

  IF v_inviter_role IS NULL THEN
    RETURN jsonb_build_object('error', 'You are not a member of this organization');
  END IF;

  IF v_inviter_role NOT IN ('admin', 'mod') THEN
    RETURN jsonb_build_object('error', 'Only admins and mods can invite members');
  END IF;

  -- Prevent mods from inviting admins
  IF v_inviter_role = 'mod' AND p_role = 'admin' THEN
    RETURN jsonb_build_object('error', 'Mods cannot invite admins');
  END IF;

  -- Check if already a member
  SELECT id INTO v_existing_member
  FROM profiles
  WHERE email = p_email AND org_id = p_org_id;

  IF v_existing_member IS NOT NULL THEN
    RETURN jsonb_build_object('error', 'User is already a member of this organization');
  END IF;

  -- Create the invite
  INSERT INTO invites (org_id, email, role, invited_by)
  VALUES (p_org_id, p_email, p_role, auth.uid())
  RETURNING id INTO v_invite_id;

  RETURN jsonb_build_object(
    'success', true,
    'invite_id', v_invite_id
  );
END;
$$;
```

Call it from Next.js:

```typescript
// file: src/app/api/team/invite/route.ts

import { createClient } from "@/lib/supabase/server";
import { NextRequest, NextResponse } from "next/server";

export async function POST(request: NextRequest) {
  try {
    const supabase = await createClient();
    const { org_id, email, role } = await request.json();

    if (!org_id || !email) {
      return NextResponse.json(
        { error: "org_id and email are required" },
        { status: 400 }
      );
    }

    const { data, error } = await supabase.rpc("invite_team_member", {
      p_org_id: org_id,
      p_email: email,
      p_role: role || "user",
    });

    if (error) {
      return NextResponse.json({ error: error.message }, { status: 500 });
    }

    if (data?.error) {
      return NextResponse.json({ error: data.error }, { status: 400 });
    }

    return NextResponse.json(data);
  } catch (err) {
    console.error("Invite error:", err);
    return NextResponse.json(
      { error: "Failed to send invite" },
      { status: 500 }
    );
  }
}
```

---

## 3. Triggers

Triggers run automatically when data changes. See `references/triggers-patterns.md` for all common patterns.

### Auto-Update Timestamps

```sql
-- file: supabase/migrations/005_timestamp_trigger.sql

-- Reusable function for updated_at
CREATE OR REPLACE FUNCTION public.handle_updated_at()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$;

-- Apply to any table
CREATE TRIGGER set_updated_at
  BEFORE UPDATE ON projects
  FOR EACH ROW
  EXECUTE FUNCTION public.handle_updated_at();

CREATE TRIGGER set_updated_at
  BEFORE UPDATE ON profiles
  FOR EACH ROW
  EXECUTE FUNCTION public.handle_updated_at();
```

### Cascade Soft Deletes

```sql
-- file: supabase/migrations/006_soft_delete_trigger.sql

CREATE OR REPLACE FUNCTION public.cascade_soft_delete()
RETURNS trigger
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
BEGIN
  -- When a project is soft-deleted, soft-delete all its tasks
  IF TG_TABLE_NAME = 'projects' THEN
    UPDATE tasks
    SET deleted_at = now(), deleted_by = auth.uid()
    WHERE project_id = NEW.id AND deleted_at IS NULL;

    -- Also soft-delete comments on those tasks
    UPDATE comments
    SET deleted_at = now(), deleted_by = auth.uid()
    WHERE task_id IN (
      SELECT id FROM tasks WHERE project_id = NEW.id
    ) AND deleted_at IS NULL;
  END IF;

  RETURN NEW;
END;
$$;

CREATE TRIGGER cascade_soft_delete_projects
  AFTER UPDATE OF deleted_at ON projects
  FOR EACH ROW
  WHEN (OLD.deleted_at IS NULL AND NEW.deleted_at IS NOT NULL)
  EXECUTE FUNCTION public.cascade_soft_delete();
```

### Audit Logging Trigger

```sql
-- file: supabase/migrations/007_audit_trigger.sql

-- Create audit log table
CREATE TABLE IF NOT EXISTS public.audit_log (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  table_name text NOT NULL,
  record_id uuid NOT NULL,
  action text NOT NULL CHECK (action IN ('INSERT', 'UPDATE', 'DELETE')),
  old_data jsonb,
  new_data jsonb,
  changed_by uuid DEFAULT auth.uid(),
  changed_at timestamptz DEFAULT now()
);

-- Index for fast lookups
CREATE INDEX idx_audit_log_record ON audit_log (table_name, record_id);
CREATE INDEX idx_audit_log_changed_at ON audit_log (changed_at);

-- Generic audit function
CREATE OR REPLACE FUNCTION public.audit_changes()
RETURNS trigger
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    INSERT INTO audit_log (table_name, record_id, action, new_data)
    VALUES (TG_TABLE_NAME, NEW.id, 'INSERT', to_jsonb(NEW));
    RETURN NEW;
  ELSIF TG_OP = 'UPDATE' THEN
    INSERT INTO audit_log (table_name, record_id, action, old_data, new_data)
    VALUES (TG_TABLE_NAME, NEW.id, 'UPDATE', to_jsonb(OLD), to_jsonb(NEW));
    RETURN NEW;
  ELSIF TG_OP = 'DELETE' THEN
    INSERT INTO audit_log (table_name, record_id, action, old_data)
    VALUES (TG_TABLE_NAME, OLD.id, 'DELETE', to_jsonb(OLD));
    RETURN OLD;
  END IF;
  RETURN NULL;
END;
$$;

-- Apply to any table
CREATE TRIGGER audit_projects
  AFTER INSERT OR UPDATE OR DELETE ON projects
  FOR EACH ROW
  EXECUTE FUNCTION public.audit_changes();
```

---

## 4. Materialized Views for Dashboards

Materialized views pre-compute expensive queries. Refresh them on a schedule with pg_cron.

```sql
-- file: supabase/migrations/008_materialized_views.sql

-- Dashboard stats per organization
CREATE MATERIALIZED VIEW IF NOT EXISTS public.org_dashboard_stats AS
SELECT
  o.id AS org_id,
  o.name AS org_name,
  COUNT(DISTINCT p.id) AS total_projects,
  COUNT(DISTINCT t.id) AS total_tasks,
  COUNT(DISTINCT t.id) FILTER (WHERE t.status = 'completed') AS completed_tasks,
  COUNT(DISTINCT t.id) FILTER (WHERE t.status = 'in_progress') AS in_progress_tasks,
  COUNT(DISTINCT m.id) AS total_members,
  ROUND(
    COUNT(DISTINCT t.id) FILTER (WHERE t.status = 'completed')::numeric /
    NULLIF(COUNT(DISTINCT t.id), 0) * 100, 1
  ) AS completion_rate
FROM organizations o
LEFT JOIN projects p ON p.org_id = o.id AND p.deleted_at IS NULL
LEFT JOIN tasks t ON t.project_id = p.id AND t.deleted_at IS NULL
LEFT JOIN profiles m ON m.org_id = o.id
GROUP BY o.id, o.name
WITH DATA;

-- Unique index required for CONCURRENTLY refresh
CREATE UNIQUE INDEX idx_org_dashboard_stats_org_id
  ON org_dashboard_stats (org_id);

-- Refresh function
CREATE OR REPLACE FUNCTION public.refresh_dashboard_stats()
RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
BEGIN
  REFRESH MATERIALIZED VIEW CONCURRENTLY public.org_dashboard_stats;
END;
$$;
```

Query it from Next.js:

```typescript
// file: src/app/api/dashboard/stats/route.ts

import { createClient } from "@/lib/supabase/server";
import { NextResponse } from "next/server";

export async function GET() {
  try {
    const supabase = await createClient();

    const {
      data: { user },
      error: authError,
    } = await supabase.auth.getUser();

    if (authError || !user) {
      return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
    }

    // Get user's org
    const { data: profile, error: profileError } = await supabase
      .from("profiles")
      .select("org_id")
      .eq("id", user.id)
      .single();

    if (profileError || !profile) {
      return NextResponse.json(
        { error: "Profile not found" },
        { status: 404 }
      );
    }

    // Fetch pre-computed stats
    const { data: stats, error: statsError } = await supabase
      .from("org_dashboard_stats")
      .select("*")
      .eq("org_id", profile.org_id)
      .single();

    if (statsError) {
      return NextResponse.json(
        { error: "Failed to load stats" },
        { status: 500 }
      );
    }

    return NextResponse.json(stats);
  } catch (err) {
    console.error("Dashboard stats error:", err);
    return NextResponse.json(
      { error: "Internal server error" },
      { status: 500 }
    );
  }
}
```

---

## 5. Full-Text Search

PostgreSQL has built-in full-text search. No need for Algolia or Elasticsearch for most apps.

```sql
-- file: supabase/migrations/009_full_text_search.sql

-- Add a tsvector column for search
ALTER TABLE articles ADD COLUMN IF NOT EXISTS search_vector tsvector;

-- Index it
CREATE INDEX IF NOT EXISTS idx_articles_search
  ON articles USING gin(search_vector);

-- Keep it updated automatically
CREATE OR REPLACE FUNCTION public.update_article_search_vector()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
  NEW.search_vector :=
    setweight(to_tsvector('english', COALESCE(NEW.title, '')), 'A') ||
    setweight(to_tsvector('english', COALESCE(NEW.subtitle, '')), 'B') ||
    setweight(to_tsvector('english', COALESCE(NEW.body, '')), 'C') ||
    setweight(to_tsvector('english', COALESCE(NEW.tags::text, '')), 'D');
  RETURN NEW;
END;
$$;

CREATE TRIGGER update_article_search
  BEFORE INSERT OR UPDATE ON articles
  FOR EACH ROW
  EXECUTE FUNCTION public.update_article_search_vector();

-- Search function with ranking
CREATE OR REPLACE FUNCTION public.search_articles(
  p_query text,
  p_limit int DEFAULT 20,
  p_offset int DEFAULT 0
)
RETURNS TABLE(
  id uuid,
  title text,
  subtitle text,
  published_at timestamptz,
  rank real
)
LANGUAGE plpgsql
STABLE
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_tsquery tsquery;
BEGIN
  -- Convert user input to tsquery (handles typos with :* prefix matching)
  v_tsquery := websearch_to_tsquery('english', p_query);

  RETURN QUERY
  SELECT
    a.id,
    a.title,
    a.subtitle,
    a.published_at,
    ts_rank(a.search_vector, v_tsquery) AS rank
  FROM articles a
  WHERE
    a.search_vector @@ v_tsquery
    AND a.deleted_at IS NULL
    AND a.published_at <= now()
  ORDER BY rank DESC
  LIMIT p_limit
  OFFSET p_offset;
END;
$$;
```

Call it from Next.js:

```typescript
// file: src/app/api/search/route.ts

import { createClient } from "@/lib/supabase/server";
import { NextRequest, NextResponse } from "next/server";

export async function GET(request: NextRequest) {
  try {
    const supabase = await createClient();
    const { searchParams } = new URL(request.url);
    const query = searchParams.get("q");
    const limit = parseInt(searchParams.get("limit") || "20", 10);
    const offset = parseInt(searchParams.get("offset") || "0", 10);

    if (!query || query.trim().length < 2) {
      return NextResponse.json(
        { error: "Query must be at least 2 characters" },
        { status: 400 }
      );
    }

    const { data, error } = await supabase.rpc("search_articles", {
      p_query: query.trim(),
      p_limit: Math.min(limit, 50),
      p_offset: Math.max(offset, 0),
    });

    if (error) {
      console.error("Search error:", error);
      return NextResponse.json(
        { error: "Search failed" },
        { status: 500 }
      );
    }

    return NextResponse.json({ results: data, query });
  } catch (err) {
    console.error("Search error:", err);
    return NextResponse.json(
      { error: "Internal server error" },
      { status: 500 }
    );
  }
}
```

---

## 6. Connection Pooling and Query Optimization

See `references/performance.md` for the full optimization guide.

### Supabase Client Setup with Pooling

```typescript
// file: src/lib/supabase/server.ts

import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function createClient() {
  const cookieStore = await cookies();

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            );
          } catch {
            // Ignore in Server Components (read-only)
          }
        },
      },
    }
  );
}
```

### Direct Pool Connection for Heavy Queries

```typescript
// file: src/lib/db/pool.ts

import { Pool } from "pg";

let pool: Pool | null = null;

export function getPool(): Pool {
  if (!pool) {
    pool = new Pool({
      connectionString: process.env.DATABASE_URL,
      max: 10,
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 5000,
    });

    pool.on("error", (err) => {
      console.error("Unexpected pool error:", err);
      pool = null;
    });
  }

  return pool;
}

export async function query<T>(
  text: string,
  params?: unknown[]
): Promise<T[]> {
  const client = await getPool().connect();
  try {
    const result = await client.query(text, params);
    return result.rows as T[];
  } catch (err) {
    console.error("Query error:", { text, err });
    throw err;
  } finally {
    client.release();
  }
}
```

---

## 7. pg_cron for Scheduled Database Jobs

Enable pg_cron in the Supabase dashboard under Database > Extensions.

```sql
-- file: supabase/migrations/010_pg_cron.sql

-- Refresh materialized views every 15 minutes
SELECT cron.schedule(
  'refresh-dashboard-stats',
  '*/15 * * * *',
  $$SELECT public.refresh_dashboard_stats()$$
);

-- Clean up expired invites daily at 3am UTC
SELECT cron.schedule(
  'cleanup-expired-invites',
  '0 3 * * *',
  $$DELETE FROM public.invites WHERE expires_at < now() - interval '7 days'$$
);

-- Archive old audit logs monthly (move to cold storage table)
SELECT cron.schedule(
  'archive-old-audit-logs',
  '0 2 1 * *',
  $$
  WITH moved AS (
    DELETE FROM public.audit_log
    WHERE changed_at < now() - interval '90 days'
    RETURNING *
  )
  INSERT INTO public.audit_log_archive
  SELECT * FROM moved;
  $$
);

-- Vacuum analyze for performance (weekly on Sunday at 4am)
SELECT cron.schedule(
  'vacuum-analyze',
  '0 4 * * 0',
  $$VACUUM ANALYZE projects; VACUUM ANALYZE tasks; VACUUM ANALYZE profiles;$$
);
```

### Monitor pg_cron Jobs

```sql
-- View scheduled jobs
SELECT * FROM cron.job ORDER BY jobid;

-- View job run history
SELECT * FROM cron.job_run_details ORDER BY start_time DESC LIMIT 20;

-- Unschedule a job
SELECT cron.unschedule('refresh-dashboard-stats');
```

---

## 8. Migration Best Practices

See `references/migration-guide.md` for the full safe migration workflow.

### Zero-Downtime Column Addition

```sql
-- file: supabase/migrations/011_add_column_safe.sql

-- Step 1: Add column as nullable (instant, no lock)
ALTER TABLE projects ADD COLUMN priority int;

-- Step 2: Backfill in batches (no long lock)
DO $$
DECLARE
  batch_size int := 1000;
  rows_updated int := 1;
BEGIN
  WHILE rows_updated > 0 LOOP
    WITH batch AS (
      SELECT id FROM projects
      WHERE priority IS NULL
      LIMIT batch_size
      FOR UPDATE SKIP LOCKED
    )
    UPDATE projects
    SET priority = 0
    WHERE id IN (SELECT id FROM batch);

    GET DIAGNOSTICS rows_updated = ROW_COUNT;
    PERFORM pg_sleep(0.1);  -- Brief pause to reduce load
  END LOOP;
END;
$$;

-- Step 3: Add default for new rows
ALTER TABLE projects ALTER COLUMN priority SET DEFAULT 0;

-- Step 4: Add NOT NULL constraint (only after all rows filled)
ALTER TABLE projects ALTER COLUMN priority SET NOT NULL;

-- Step 5: Add index concurrently (does not lock table)
CREATE INDEX CONCURRENTLY idx_projects_priority ON projects (priority);
```

### Rollback Pattern

```sql
-- file: supabase/migrations/011_add_column_safe_rollback.sql

-- Reverse the migration
DROP INDEX IF EXISTS idx_projects_priority;
ALTER TABLE projects DROP COLUMN IF EXISTS priority;
```

---

## 9. Supabase Branching

Supabase Branching creates isolated database environments per Git branch. Enable it in the Supabase dashboard under Settings > Branching.

### How It Works

1. Each Git branch gets its own Supabase database instance
2. Migrations from `supabase/migrations/` run automatically on branch creation
3. Preview deployments on Vercel connect to the branch database
4. Merging to main applies migrations to production

### Branch-Aware Environment Setup

```typescript
// file: src/lib/supabase/env.ts

export function getSupabaseConfig() {
  // Supabase Branching sets these automatically on preview deployments
  const url = process.env.NEXT_PUBLIC_SUPABASE_URL;
  const anonKey = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY;

  if (!url || !anonKey) {
    throw new Error(
      "Missing Supabase environment variables. " +
        "Ensure NEXT_PUBLIC_SUPABASE_URL and NEXT_PUBLIC_SUPABASE_ANON_KEY are set."
    );
  }

  return { url, anonKey };
}
```

### Best Practices

- Keep migrations small and atomic (one change per file)
- Always include a rollback migration
- Test migrations on a branch before merging to main
- Use `supabase db diff` to generate migrations from schema changes
- Never modify a migration that has already been applied to production

---

## Quick Reference

| Task | Where to look |
|------|---------------|
| RLS policy templates | `references/advanced-rls.md` |
| Database functions | `references/plpgsql-functions.md` |
| Trigger patterns | `references/triggers-patterns.md` |
| Performance tuning | `references/performance.md` |
| Migration workflow | `references/migration-guide.md` |
