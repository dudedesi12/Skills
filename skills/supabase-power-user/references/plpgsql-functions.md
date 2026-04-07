# Production PL/pgSQL Functions

All functions use `SECURITY DEFINER` and `SET search_path = public` for safety.

---

## 1. Invite Team Member (with validation)

```sql
-- file: supabase/migrations/fn_invite_team_member.sql

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
  v_existing uuid;
  v_invite_id uuid;
BEGIN
  SELECT role INTO v_inviter_role FROM profiles WHERE id = auth.uid() AND org_id = p_org_id;

  IF v_inviter_role IS NULL THEN
    RETURN jsonb_build_object('error', 'Not a member of this organization');
  END IF;

  IF v_inviter_role NOT IN ('admin', 'mod') THEN
    RETURN jsonb_build_object('error', 'Insufficient permissions');
  END IF;

  IF v_inviter_role = 'mod' AND p_role = 'admin' THEN
    RETURN jsonb_build_object('error', 'Mods cannot assign admin role');
  END IF;

  SELECT id INTO v_existing FROM profiles WHERE email = p_email AND org_id = p_org_id;
  IF v_existing IS NOT NULL THEN
    RETURN jsonb_build_object('error', 'Already a member');
  END IF;

  INSERT INTO invites (org_id, email, role, invited_by, expires_at)
  VALUES (p_org_id, p_email, p_role, auth.uid(), now() + interval '7 days')
  RETURNING id INTO v_invite_id;

  RETURN jsonb_build_object('success', true, 'invite_id', v_invite_id);
END;
$$;
```

## 2. Accept Invite

```sql
-- file: supabase/migrations/fn_accept_invite.sql

CREATE OR REPLACE FUNCTION public.accept_invite(p_invite_id uuid)
RETURNS jsonb
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_invite record;
  v_user_email text;
BEGIN
  SELECT email INTO v_user_email FROM auth.users WHERE id = auth.uid();

  SELECT * INTO v_invite FROM invites
  WHERE id = p_invite_id AND email = v_user_email AND accepted_at IS NULL;

  IF v_invite IS NULL THEN
    RETURN jsonb_build_object('error', 'Invalid or expired invite');
  END IF;

  IF v_invite.expires_at < now() THEN
    RETURN jsonb_build_object('error', 'Invite has expired');
  END IF;

  -- Update user profile with org and role
  UPDATE profiles
  SET org_id = v_invite.org_id, role = v_invite.role
  WHERE id = auth.uid();

  -- Mark invite as accepted
  UPDATE invites SET accepted_at = now() WHERE id = p_invite_id;

  RETURN jsonb_build_object('success', true, 'org_id', v_invite.org_id);
END;
$$;
```

## 3. Transfer Ownership

```sql
-- file: supabase/migrations/fn_transfer_ownership.sql

CREATE OR REPLACE FUNCTION public.transfer_ownership(
  p_org_id uuid,
  p_new_owner_id uuid
)
RETURNS jsonb
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_current_role text;
  v_new_owner_role text;
BEGIN
  SELECT role INTO v_current_role FROM profiles WHERE id = auth.uid() AND org_id = p_org_id;
  IF v_current_role != 'admin' THEN
    RETURN jsonb_build_object('error', 'Only admins can transfer ownership');
  END IF;

  SELECT role INTO v_new_owner_role FROM profiles WHERE id = p_new_owner_id AND org_id = p_org_id;
  IF v_new_owner_role IS NULL THEN
    RETURN jsonb_build_object('error', 'User is not a member of this organization');
  END IF;

  -- Promote new owner, demote current
  UPDATE profiles SET role = 'admin' WHERE id = p_new_owner_id AND org_id = p_org_id;
  UPDATE organizations SET owner_id = p_new_owner_id WHERE id = p_org_id;

  RETURN jsonb_build_object('success', true);
END;
$$;
```

## 4. Bulk Status Update with Validation

```sql
-- file: supabase/migrations/fn_bulk_status_update.sql

CREATE OR REPLACE FUNCTION public.bulk_update_task_status(
  p_task_ids uuid[],
  p_new_status text
)
RETURNS jsonb
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_valid_statuses text[] := ARRAY['todo', 'in_progress', 'review', 'completed'];
  v_updated_count int;
BEGIN
  IF NOT p_new_status = ANY(v_valid_statuses) THEN
    RETURN jsonb_build_object('error', 'Invalid status: ' || p_new_status);
  END IF;

  IF array_length(p_task_ids, 1) IS NULL OR array_length(p_task_ids, 1) > 100 THEN
    RETURN jsonb_build_object('error', 'Provide between 1 and 100 task IDs');
  END IF;

  UPDATE tasks
  SET status = p_new_status, updated_at = now()
  WHERE id = ANY(p_task_ids)
    AND deleted_at IS NULL
    AND project_id IN (
      SELECT p.id FROM projects p WHERE p.org_id = public.get_user_org_id()
    );

  GET DIAGNOSTICS v_updated_count = ROW_COUNT;

  RETURN jsonb_build_object('success', true, 'updated_count', v_updated_count);
END;
$$;
```

## 5. Generate Unique Slug

```sql
-- file: supabase/migrations/fn_generate_slug.sql

CREATE OR REPLACE FUNCTION public.generate_unique_slug(
  p_table_name text,
  p_title text
)
RETURNS text
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_slug text;
  v_count int;
  v_exists boolean;
BEGIN
  -- Convert title to slug
  v_slug := lower(regexp_replace(trim(p_title), '[^a-zA-Z0-9]+', '-', 'g'));
  v_slug := trim(both '-' from v_slug);

  IF length(v_slug) < 3 THEN
    v_slug := v_slug || '-' || substr(gen_random_uuid()::text, 1, 8);
  END IF;

  -- Check uniqueness
  EXECUTE format('SELECT EXISTS(SELECT 1 FROM %I WHERE slug = $1)', p_table_name)
  INTO v_exists USING v_slug;

  IF v_exists THEN
    v_slug := v_slug || '-' || substr(gen_random_uuid()::text, 1, 8);
  END IF;

  RETURN v_slug;
END;
$$;
```

## 6. Paginated Query with Cursor

```sql
-- file: supabase/migrations/fn_paginated_tasks.sql

CREATE OR REPLACE FUNCTION public.get_tasks_paginated(
  p_project_id uuid,
  p_cursor timestamptz DEFAULT null,
  p_limit int DEFAULT 20,
  p_status text DEFAULT null
)
RETURNS TABLE(
  id uuid,
  title text,
  status text,
  priority int,
  assigned_to uuid,
  created_at timestamptz,
  has_more boolean
)
LANGUAGE plpgsql
STABLE
SECURITY DEFINER
SET search_path = public
AS $$
BEGIN
  RETURN QUERY
  WITH filtered AS (
    SELECT t.id, t.title, t.status, t.priority, t.assigned_to, t.created_at
    FROM tasks t
    WHERE t.project_id = p_project_id
      AND t.deleted_at IS NULL
      AND (p_cursor IS NULL OR t.created_at < p_cursor)
      AND (p_status IS NULL OR t.status = p_status)
    ORDER BY t.created_at DESC
    LIMIT p_limit + 1
  )
  SELECT
    f.id, f.title, f.status, f.priority, f.assigned_to, f.created_at,
    (SELECT COUNT(*) FROM filtered) > p_limit AS has_more
  FROM filtered f
  LIMIT p_limit;
END;
$$;
```

## 7. Increment Counter Atomically

```sql
-- file: supabase/migrations/fn_increment_counter.sql

CREATE OR REPLACE FUNCTION public.increment_view_count(p_article_id uuid)
RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
BEGIN
  UPDATE articles
  SET view_count = view_count + 1
  WHERE id = p_article_id AND deleted_at IS NULL;

  IF NOT FOUND THEN
    RAISE EXCEPTION 'Article not found: %', p_article_id;
  END IF;
END;
$$;
```

## 8. Safe Delete with Dependency Check

```sql
-- file: supabase/migrations/fn_safe_delete.sql

CREATE OR REPLACE FUNCTION public.safe_delete_project(p_project_id uuid)
RETURNS jsonb
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_active_tasks int;
BEGIN
  IF NOT public.has_role_level('admin') THEN
    RETURN jsonb_build_object('error', 'Only admins can delete projects');
  END IF;

  SELECT COUNT(*) INTO v_active_tasks
  FROM tasks
  WHERE project_id = p_project_id AND status != 'completed' AND deleted_at IS NULL;

  IF v_active_tasks > 0 THEN
    RETURN jsonb_build_object(
      'error', format('%s active tasks must be completed or removed first', v_active_tasks)
    );
  END IF;

  -- Soft delete
  UPDATE projects SET deleted_at = now(), deleted_by = auth.uid()
  WHERE id = p_project_id;

  RETURN jsonb_build_object('success', true);
END;
$$;
```

## 9. User Stats Aggregation

```sql
-- file: supabase/migrations/fn_user_stats.sql

CREATE OR REPLACE FUNCTION public.get_user_stats(p_user_id uuid DEFAULT null)
RETURNS jsonb
LANGUAGE plpgsql
STABLE
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_user_id uuid;
  v_result jsonb;
BEGIN
  v_user_id := COALESCE(p_user_id, auth.uid());

  SELECT jsonb_build_object(
    'total_tasks', COUNT(*),
    'completed_tasks', COUNT(*) FILTER (WHERE status = 'completed'),
    'in_progress_tasks', COUNT(*) FILTER (WHERE status = 'in_progress'),
    'overdue_tasks', COUNT(*) FILTER (WHERE due_date < now() AND status != 'completed'),
    'completion_rate', ROUND(
      COUNT(*) FILTER (WHERE status = 'completed')::numeric /
      NULLIF(COUNT(*), 0) * 100, 1
    )
  ) INTO v_result
  FROM tasks
  WHERE assigned_to = v_user_id AND deleted_at IS NULL;

  RETURN v_result;
END;
$$;
```

## 10. Duplicate Record

```sql
-- file: supabase/migrations/fn_duplicate_project.sql

CREATE OR REPLACE FUNCTION public.duplicate_project(p_project_id uuid)
RETURNS jsonb
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_new_project_id uuid;
  v_task_count int;
BEGIN
  -- Duplicate the project
  INSERT INTO projects (name, description, org_id, created_by, priority)
  SELECT
    name || ' (Copy)',
    description,
    org_id,
    auth.uid(),
    priority
  FROM projects
  WHERE id = p_project_id AND org_id = public.get_user_org_id()
  RETURNING id INTO v_new_project_id;

  IF v_new_project_id IS NULL THEN
    RETURN jsonb_build_object('error', 'Project not found or no access');
  END IF;

  -- Duplicate all tasks
  INSERT INTO tasks (title, description, status, priority, project_id, created_by)
  SELECT title, description, 'todo', priority, v_new_project_id, auth.uid()
  FROM tasks
  WHERE project_id = p_project_id AND deleted_at IS NULL;

  GET DIAGNOSTICS v_task_count = ROW_COUNT;

  RETURN jsonb_build_object(
    'success', true,
    'new_project_id', v_new_project_id,
    'tasks_copied', v_task_count
  );
END;
$$;
```

## 11. Search with Autocomplete

```sql
-- file: supabase/migrations/fn_autocomplete.sql

CREATE OR REPLACE FUNCTION public.autocomplete_search(
  p_query text,
  p_table text DEFAULT 'articles',
  p_limit int DEFAULT 10
)
RETURNS TABLE(id uuid, title text, match_type text)
LANGUAGE plpgsql
STABLE
SECURITY DEFINER
SET search_path = public
AS $$
BEGIN
  IF p_table = 'articles' THEN
    RETURN QUERY
    SELECT a.id, a.title,
      CASE
        WHEN a.title ILIKE p_query || '%' THEN 'prefix'
        WHEN a.title ILIKE '%' || p_query || '%' THEN 'contains'
        ELSE 'fuzzy'
      END AS match_type
    FROM articles a
    WHERE a.deleted_at IS NULL
      AND (
        a.title ILIKE '%' || p_query || '%'
        OR a.search_vector @@ websearch_to_tsquery('english', p_query)
      )
    ORDER BY
      CASE WHEN a.title ILIKE p_query || '%' THEN 0 ELSE 1 END,
      ts_rank(a.search_vector, websearch_to_tsquery('english', p_query)) DESC
    LIMIT p_limit;
  END IF;
END;
$$;
```

## 12. Org Usage Limits Check

```sql
-- file: supabase/migrations/fn_check_usage.sql

CREATE OR REPLACE FUNCTION public.check_org_usage(p_resource text)
RETURNS jsonb
LANGUAGE plpgsql
STABLE
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_org_id uuid;
  v_plan text;
  v_current int;
  v_limit int;
  v_limits jsonb := '{
    "free":       {"projects": 5,   "members": 3,  "storage_mb": 100},
    "pro":        {"projects": 50,  "members": 20, "storage_mb": 5000},
    "enterprise": {"projects": 999, "members": 999, "storage_mb": 50000}
  }'::jsonb;
BEGIN
  v_org_id := public.get_user_org_id();
  SELECT plan INTO v_plan FROM organizations WHERE id = v_org_id;
  v_plan := COALESCE(v_plan, 'free');

  v_limit := (v_limits->v_plan->>p_resource)::int;

  IF p_resource = 'projects' THEN
    SELECT COUNT(*) INTO v_current FROM projects WHERE org_id = v_org_id AND deleted_at IS NULL;
  ELSIF p_resource = 'members' THEN
    SELECT COUNT(*) INTO v_current FROM profiles WHERE org_id = v_org_id;
  ELSE
    RETURN jsonb_build_object('error', 'Unknown resource: ' || p_resource);
  END IF;

  RETURN jsonb_build_object(
    'current', v_current,
    'limit', v_limit,
    'remaining', v_limit - v_current,
    'can_create', v_current < v_limit
  );
END;
$$;
```

## 13. Batch Upsert

```sql
-- file: supabase/migrations/fn_batch_upsert.sql

CREATE OR REPLACE FUNCTION public.batch_upsert_tags(p_tags jsonb)
RETURNS jsonb
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_upserted int;
BEGIN
  INSERT INTO tags (name, slug, color)
  SELECT
    t->>'name',
    lower(regexp_replace(trim(t->>'name'), '[^a-zA-Z0-9]+', '-', 'g')),
    COALESCE(t->>'color', '#6B7280')
  FROM jsonb_array_elements(p_tags) AS t
  ON CONFLICT (slug) DO UPDATE SET
    name = EXCLUDED.name,
    color = EXCLUDED.color,
    updated_at = now();

  GET DIAGNOSTICS v_upserted = ROW_COUNT;

  RETURN jsonb_build_object('success', true, 'upserted', v_upserted);
END;
$$;
```

## 14. Activity Feed

```sql
-- file: supabase/migrations/fn_activity_feed.sql

CREATE OR REPLACE FUNCTION public.get_activity_feed(
  p_limit int DEFAULT 20,
  p_offset int DEFAULT 0
)
RETURNS TABLE(
  id bigint,
  action text,
  table_name text,
  record_id uuid,
  actor_name text,
  actor_avatar text,
  changed_at timestamptz,
  summary text
)
LANGUAGE plpgsql
STABLE
SECURITY DEFINER
SET search_path = public
AS $$
BEGIN
  RETURN QUERY
  SELECT
    al.id,
    al.action,
    al.table_name,
    al.record_id,
    p.display_name AS actor_name,
    p.avatar_url AS actor_avatar,
    al.changed_at,
    CASE
      WHEN al.action = 'INSERT' AND al.table_name = 'projects'
        THEN p.display_name || ' created project "' || (al.new_data->>'name') || '"'
      WHEN al.action = 'UPDATE' AND al.table_name = 'tasks'
        THEN p.display_name || ' updated task "' || (al.new_data->>'title') || '"'
      WHEN al.action = 'DELETE'
        THEN p.display_name || ' deleted a ' || al.table_name
      ELSE p.display_name || ' ' || lower(al.action) || 'd a ' || al.table_name
    END AS summary
  FROM audit_log al
  LEFT JOIN profiles p ON p.id = al.changed_by
  WHERE al.table_name IN (
    SELECT t.table_name::text FROM information_schema.tables t
    WHERE t.table_schema = 'public'
  )
  ORDER BY al.changed_at DESC
  LIMIT p_limit OFFSET p_offset;
END;
$$;
```

## 15. Data Export (CSV-ready)

```sql
-- file: supabase/migrations/fn_export_data.sql

CREATE OR REPLACE FUNCTION public.export_org_data(p_table text)
RETURNS jsonb
LANGUAGE plpgsql
STABLE
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_org_id uuid;
  v_result jsonb;
BEGIN
  v_org_id := public.get_user_org_id();

  IF NOT public.has_role_level('admin') THEN
    RETURN jsonb_build_object('error', 'Only admins can export data');
  END IF;

  IF p_table = 'projects' THEN
    SELECT jsonb_agg(row_to_json(p)) INTO v_result
    FROM projects p WHERE p.org_id = v_org_id AND p.deleted_at IS NULL;
  ELSIF p_table = 'tasks' THEN
    SELECT jsonb_agg(row_to_json(t)) INTO v_result
    FROM tasks t
    JOIN projects p ON p.id = t.project_id
    WHERE p.org_id = v_org_id AND t.deleted_at IS NULL;
  ELSIF p_table = 'members' THEN
    SELECT jsonb_agg(jsonb_build_object(
      'id', pr.id, 'display_name', pr.display_name,
      'email', pr.email, 'role', pr.role, 'joined_at', pr.created_at
    )) INTO v_result
    FROM profiles pr WHERE pr.org_id = v_org_id;
  ELSE
    RETURN jsonb_build_object('error', 'Unsupported table: ' || p_table);
  END IF;

  RETURN jsonb_build_object('success', true, 'data', COALESCE(v_result, '[]'::jsonb));
END;
$$;
```
