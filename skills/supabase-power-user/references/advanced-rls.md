# Advanced RLS Policy Templates

## Setup

Always enable RLS on every table with user data:

```sql
ALTER TABLE your_table ENABLE ROW LEVEL SECURITY;
```

---

## 1. Basic Owner-Only Access

```sql
-- file: supabase/migrations/rls_owner_only.sql

CREATE POLICY "Owner select" ON documents
  FOR SELECT USING (user_id = auth.uid());

CREATE POLICY "Owner insert" ON documents
  FOR INSERT WITH CHECK (user_id = auth.uid());

CREATE POLICY "Owner update" ON documents
  FOR UPDATE USING (user_id = auth.uid()) WITH CHECK (user_id = auth.uid());

CREATE POLICY "Owner delete" ON documents
  FOR DELETE USING (user_id = auth.uid());
```

## 2. Multi-Tenant Isolation

```sql
-- file: supabase/migrations/rls_multi_tenant.sql

CREATE OR REPLACE FUNCTION public.get_user_org_id()
RETURNS uuid
LANGUAGE sql STABLE SECURITY DEFINER SET search_path = public
AS $$
  SELECT org_id FROM public.profiles WHERE id = auth.uid();
$$;

CREATE POLICY "Tenant select" ON projects
  FOR SELECT USING (org_id = public.get_user_org_id());

CREATE POLICY "Tenant insert" ON projects
  FOR INSERT WITH CHECK (org_id = public.get_user_org_id());

CREATE POLICY "Tenant update" ON projects
  FOR UPDATE USING (org_id = public.get_user_org_id())
  WITH CHECK (org_id = public.get_user_org_id());

CREATE POLICY "Tenant delete" ON projects
  FOR DELETE USING (org_id = public.get_user_org_id());
```

## 3. Role Hierarchy (admin > mod > user)

```sql
-- file: supabase/migrations/rls_role_hierarchy.sql

CREATE OR REPLACE FUNCTION public.has_role_level(required_role text)
RETURNS boolean
LANGUAGE plpgsql STABLE SECURITY DEFINER SET search_path = public
AS $$
DECLARE
  user_role text;
  role_levels jsonb := '{"user": 1, "mod": 2, "admin": 3}'::jsonb;
BEGIN
  SELECT role INTO user_role FROM public.profiles WHERE id = auth.uid();
  IF user_role IS NULL THEN RETURN false; END IF;
  RETURN (role_levels->>user_role)::int >= (role_levels->>required_role)::int;
END;
$$;

CREATE POLICY "Admin full access" ON projects
  FOR ALL USING (public.has_role_level('admin'));

CREATE POLICY "Mod read" ON projects
  FOR SELECT USING (public.has_role_level('mod') AND org_id = public.get_user_org_id());

CREATE POLICY "Mod update" ON projects
  FOR UPDATE USING (public.has_role_level('mod') AND org_id = public.get_user_org_id());
```

## 4. Team-Based Access

```sql
-- file: supabase/migrations/rls_team_based.sql

CREATE POLICY "Team members select" ON tasks
  FOR SELECT USING (
    EXISTS (
      SELECT 1 FROM team_members tm
      WHERE tm.team_id = tasks.team_id AND tm.user_id = auth.uid()
    )
  );

CREATE POLICY "Team members insert" ON tasks
  FOR INSERT WITH CHECK (
    EXISTS (
      SELECT 1 FROM team_members tm
      WHERE tm.team_id = tasks.team_id AND tm.user_id = auth.uid()
    )
  );
```

## 5. Public Read, Authenticated Write

```sql
-- file: supabase/migrations/rls_public_read.sql

CREATE POLICY "Public read" ON articles
  FOR SELECT USING (published = true);

CREATE POLICY "Authenticated insert" ON articles
  FOR INSERT WITH CHECK (auth.uid() IS NOT NULL AND user_id = auth.uid());

CREATE POLICY "Owner update" ON articles
  FOR UPDATE USING (user_id = auth.uid());
```

## 6. Time-Based Access

```sql
-- file: supabase/migrations/rls_time_based.sql

CREATE POLICY "Published content" ON articles
  FOR SELECT USING (published_at IS NOT NULL AND published_at <= now());

CREATE POLICY "Scheduled content for admins" ON articles
  FOR SELECT USING (
    public.has_role_level('admin') AND published_at IS NOT NULL
  );
```

## 7. IP-Based Restriction (via custom claims)

```sql
-- file: supabase/migrations/rls_ip_based.sql

CREATE POLICY "Internal network only" ON internal_reports
  FOR SELECT USING (
    (current_setting('request.headers', true)::json->>'x-forwarded-for')
    LIKE '10.0.%'
  );
```

## 8. Row-Level Approval Workflow

```sql
-- file: supabase/migrations/rls_approval.sql

-- Users can only see approved content, unless they created it
CREATE POLICY "Approved or own" ON submissions
  FOR SELECT USING (
    status = 'approved' OR created_by = auth.uid()
  );

-- Only mods+ can update status
CREATE POLICY "Mods approve" ON submissions
  FOR UPDATE USING (
    public.has_role_level('mod')
    AND org_id = public.get_user_org_id()
  );
```

## 9. Soft Delete Visibility

```sql
-- file: supabase/migrations/rls_soft_delete.sql

-- Hide soft-deleted rows from normal users
CREATE POLICY "Hide deleted" ON projects
  FOR SELECT USING (deleted_at IS NULL OR public.has_role_level('admin'));

-- Only admins can actually soft-delete
CREATE POLICY "Admin soft delete" ON projects
  FOR UPDATE USING (public.has_role_level('admin'))
  WITH CHECK (true);
```

## 10. Column-Level Security via Views

```sql
-- file: supabase/migrations/rls_column_level.sql

-- Create a view that hides sensitive columns
CREATE VIEW public.profiles_public AS
SELECT id, display_name, avatar_url, created_at
FROM public.profiles;

-- Grant access to the view
GRANT SELECT ON public.profiles_public TO anon, authenticated;
```

## 11. Invite-Only Access

```sql
-- file: supabase/migrations/rls_invite_only.sql

CREATE POLICY "Invited users only" ON workspaces
  FOR SELECT USING (
    EXISTS (
      SELECT 1 FROM workspace_members wm
      WHERE wm.workspace_id = workspaces.id
      AND wm.user_id = auth.uid()
      AND wm.accepted_at IS NOT NULL
    )
  );
```

## 12. Rate-Limited Write Access

```sql
-- file: supabase/migrations/rls_rate_limit.sql

CREATE POLICY "Rate limited insert" ON comments
  FOR INSERT WITH CHECK (
    auth.uid() IS NOT NULL
    AND (
      SELECT COUNT(*) FROM comments
      WHERE user_id = auth.uid()
      AND created_at > now() - interval '1 minute'
    ) < 5
  );
```

## 13. Hierarchical Access (parent-child)

```sql
-- file: supabase/migrations/rls_hierarchical.sql

-- Access to child if you have access to parent
CREATE POLICY "Parent access grants child access" ON subtasks
  FOR SELECT USING (
    EXISTS (
      SELECT 1 FROM tasks t
      JOIN team_members tm ON tm.team_id = t.team_id
      WHERE t.id = subtasks.task_id AND tm.user_id = auth.uid()
    )
  );
```

## 14. Feature Flag Based

```sql
-- file: supabase/migrations/rls_feature_flag.sql

CREATE POLICY "Beta feature access" ON beta_features
  FOR SELECT USING (
    EXISTS (
      SELECT 1 FROM user_feature_flags uff
      WHERE uff.user_id = auth.uid()
      AND uff.flag_name = 'beta_access'
      AND uff.enabled = true
    )
  );
```

## 15. Subscription Tier Based

```sql
-- file: supabase/migrations/rls_subscription.sql

CREATE OR REPLACE FUNCTION public.get_user_plan()
RETURNS text
LANGUAGE sql STABLE SECURITY DEFINER SET search_path = public
AS $$
  SELECT plan FROM public.subscriptions
  WHERE user_id = auth.uid() AND status = 'active'
  ORDER BY created_at DESC LIMIT 1;
$$;

CREATE POLICY "Pro features" ON advanced_reports
  FOR SELECT USING (public.get_user_plan() IN ('pro', 'enterprise'));
```

## 16. Geographic Restriction

```sql
-- file: supabase/migrations/rls_geo.sql

CREATE POLICY "EU data residency" ON eu_user_data
  FOR SELECT USING (
    EXISTS (
      SELECT 1 FROM profiles p
      WHERE p.id = auth.uid() AND p.region = 'eu'
    )
  );
```

## 17. Temporal Access (expiring permissions)

```sql
-- file: supabase/migrations/rls_temporal.sql

CREATE POLICY "Active permission" ON shared_documents
  FOR SELECT USING (
    EXISTS (
      SELECT 1 FROM document_shares ds
      WHERE ds.document_id = shared_documents.id
      AND ds.shared_with = auth.uid()
      AND ds.expires_at > now()
    )
  );
```

## 18. Service Role Bypass

```sql
-- file: supabase/migrations/rls_service_role.sql

-- Service role always bypasses RLS automatically.
-- For edge functions that need admin access, use supabaseAdmin:

-- In your edge function:
-- const supabaseAdmin = createClient(url, SERVICE_ROLE_KEY);
-- This client bypasses all RLS policies.
```

## 19. Conditional Insert with Limits

```sql
-- file: supabase/migrations/rls_conditional_insert.sql

-- Free users: max 5 projects. Pro: max 50. Enterprise: unlimited.
CREATE POLICY "Plan-based project limit" ON projects
  FOR INSERT WITH CHECK (
    CASE public.get_user_plan()
      WHEN 'enterprise' THEN true
      WHEN 'pro' THEN (
        SELECT COUNT(*) FROM projects WHERE created_by = auth.uid() AND deleted_at IS NULL
      ) < 50
      ELSE (
        SELECT COUNT(*) FROM projects WHERE created_by = auth.uid() AND deleted_at IS NULL
      ) < 5
    END
  );
```

## 20. Audit-Aware Policies

```sql
-- file: supabase/migrations/rls_audit_aware.sql

-- Allow reads on audit log only for records the user has access to
CREATE POLICY "Audit log access" ON audit_log
  FOR SELECT USING (
    CASE table_name
      WHEN 'projects' THEN EXISTS (
        SELECT 1 FROM projects p
        WHERE p.id = audit_log.record_id AND p.org_id = public.get_user_org_id()
      )
      WHEN 'tasks' THEN EXISTS (
        SELECT 1 FROM tasks t
        JOIN projects p ON p.id = t.project_id
        WHERE t.id = audit_log.record_id AND p.org_id = public.get_user_org_id()
      )
      ELSE false
    END
  );
```

## 21. Anonymous vs Authenticated Split

```sql
-- file: supabase/migrations/rls_anon_split.sql

-- Anonymous users: read-only on public data
CREATE POLICY "Anon read public" ON posts
  FOR SELECT TO anon
  USING (visibility = 'public');

-- Authenticated users: read public + their own private
CREATE POLICY "Auth read all accessible" ON posts
  FOR SELECT TO authenticated
  USING (visibility = 'public' OR user_id = auth.uid());

-- Authenticated users: full CRUD on own posts
CREATE POLICY "Auth write own" ON posts
  FOR INSERT TO authenticated
  WITH CHECK (user_id = auth.uid());
```
