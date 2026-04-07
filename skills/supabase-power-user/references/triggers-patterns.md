# Common Trigger Patterns

Production-ready PostgreSQL triggers for Supabase projects.

---

## 1. Auto-Update `updated_at` Timestamp

```sql
-- file: supabase/migrations/trigger_updated_at.sql

CREATE OR REPLACE FUNCTION public.handle_updated_at()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$;

-- Apply to any table (repeat for each table)
CREATE TRIGGER set_updated_at
  BEFORE UPDATE ON projects
  FOR EACH ROW
  EXECUTE FUNCTION public.handle_updated_at();

CREATE TRIGGER set_updated_at
  BEFORE UPDATE ON tasks
  FOR EACH ROW
  EXECUTE FUNCTION public.handle_updated_at();

CREATE TRIGGER set_updated_at
  BEFORE UPDATE ON profiles
  FOR EACH ROW
  EXECUTE FUNCTION public.handle_updated_at();
```

## 2. Cascade Soft Deletes

```sql
-- file: supabase/migrations/trigger_cascade_soft_delete.sql

CREATE OR REPLACE FUNCTION public.cascade_soft_delete()
RETURNS trigger
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
BEGIN
  IF TG_TABLE_NAME = 'projects' THEN
    UPDATE tasks SET deleted_at = now(), deleted_by = auth.uid()
    WHERE project_id = NEW.id AND deleted_at IS NULL;

    UPDATE comments SET deleted_at = now(), deleted_by = auth.uid()
    WHERE task_id IN (SELECT id FROM tasks WHERE project_id = NEW.id)
    AND deleted_at IS NULL;
  END IF;

  IF TG_TABLE_NAME = 'tasks' THEN
    UPDATE comments SET deleted_at = now(), deleted_by = auth.uid()
    WHERE task_id = NEW.id AND deleted_at IS NULL;
  END IF;

  RETURN NEW;
END;
$$;

CREATE TRIGGER cascade_soft_delete_projects
  AFTER UPDATE OF deleted_at ON projects
  FOR EACH ROW
  WHEN (OLD.deleted_at IS NULL AND NEW.deleted_at IS NOT NULL)
  EXECUTE FUNCTION public.cascade_soft_delete();

CREATE TRIGGER cascade_soft_delete_tasks
  AFTER UPDATE OF deleted_at ON tasks
  FOR EACH ROW
  WHEN (OLD.deleted_at IS NULL AND NEW.deleted_at IS NOT NULL)
  EXECUTE FUNCTION public.cascade_soft_delete();
```

## 3. Audit Log

```sql
-- file: supabase/migrations/trigger_audit_log.sql

CREATE TABLE IF NOT EXISTS public.audit_log (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  table_name text NOT NULL,
  record_id uuid NOT NULL,
  action text NOT NULL CHECK (action IN ('INSERT', 'UPDATE', 'DELETE')),
  old_data jsonb,
  new_data jsonb,
  changed_fields text[],
  changed_by uuid DEFAULT auth.uid(),
  changed_at timestamptz DEFAULT now()
);

CREATE INDEX idx_audit_log_record ON audit_log (table_name, record_id);
CREATE INDEX idx_audit_log_time ON audit_log (changed_at DESC);

CREATE OR REPLACE FUNCTION public.audit_changes()
RETURNS trigger
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_changed_fields text[];
  v_old jsonb;
  v_new jsonb;
  key text;
BEGIN
  IF TG_OP = 'INSERT' THEN
    INSERT INTO audit_log (table_name, record_id, action, new_data)
    VALUES (TG_TABLE_NAME, NEW.id, 'INSERT', to_jsonb(NEW));
    RETURN NEW;

  ELSIF TG_OP = 'UPDATE' THEN
    v_old := to_jsonb(OLD);
    v_new := to_jsonb(NEW);

    -- Track which fields actually changed
    FOR key IN SELECT jsonb_object_keys(v_new)
    LOOP
      IF v_old->key IS DISTINCT FROM v_new->key THEN
        v_changed_fields := array_append(v_changed_fields, key);
      END IF;
    END LOOP;

    -- Only log if something actually changed (besides updated_at)
    v_changed_fields := array_remove(v_changed_fields, 'updated_at');
    IF array_length(v_changed_fields, 1) > 0 THEN
      INSERT INTO audit_log (table_name, record_id, action, old_data, new_data, changed_fields)
      VALUES (TG_TABLE_NAME, NEW.id, 'UPDATE', v_old, v_new, v_changed_fields);
    END IF;
    RETURN NEW;

  ELSIF TG_OP = 'DELETE' THEN
    INSERT INTO audit_log (table_name, record_id, action, old_data)
    VALUES (TG_TABLE_NAME, OLD.id, 'DELETE', to_jsonb(OLD));
    RETURN OLD;
  END IF;

  RETURN NULL;
END;
$$;

-- Apply to tables
CREATE TRIGGER audit_projects
  AFTER INSERT OR UPDATE OR DELETE ON projects
  FOR EACH ROW EXECUTE FUNCTION public.audit_changes();

CREATE TRIGGER audit_tasks
  AFTER INSERT OR UPDATE OR DELETE ON tasks
  FOR EACH ROW EXECUTE FUNCTION public.audit_changes();
```

## 4. Auto-Generate Slug on Insert

```sql
-- file: supabase/migrations/trigger_auto_slug.sql

CREATE OR REPLACE FUNCTION public.auto_generate_slug()
RETURNS trigger
LANGUAGE plpgsql
AS $$
DECLARE
  v_slug text;
  v_exists boolean;
BEGIN
  IF NEW.slug IS NULL OR NEW.slug = '' THEN
    v_slug := lower(regexp_replace(trim(NEW.title), '[^a-zA-Z0-9]+', '-', 'g'));
    v_slug := trim(both '-' from v_slug);

    EXECUTE format('SELECT EXISTS(SELECT 1 FROM %I WHERE slug = $1)', TG_TABLE_NAME)
    INTO v_exists USING v_slug;

    IF v_exists THEN
      v_slug := v_slug || '-' || substr(gen_random_uuid()::text, 1, 8);
    END IF;

    NEW.slug := v_slug;
  END IF;

  RETURN NEW;
END;
$$;

CREATE TRIGGER auto_slug_articles
  BEFORE INSERT ON articles
  FOR EACH ROW
  EXECUTE FUNCTION public.auto_generate_slug();
```

## 5. Validate JSON Schema

```sql
-- file: supabase/migrations/trigger_validate_json.sql

CREATE OR REPLACE FUNCTION public.validate_task_metadata()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
  -- Ensure metadata has required fields if present
  IF NEW.metadata IS NOT NULL THEN
    IF NEW.metadata ? 'priority' AND NOT (
      (NEW.metadata->>'priority')::int BETWEEN 1 AND 5
    ) THEN
      RAISE EXCEPTION 'metadata.priority must be between 1 and 5';
    END IF;

    IF NEW.metadata ? 'due_date' THEN
      BEGIN
        PERFORM (NEW.metadata->>'due_date')::timestamptz;
      EXCEPTION WHEN OTHERS THEN
        RAISE EXCEPTION 'metadata.due_date must be a valid timestamp';
      END;
    END IF;
  END IF;

  RETURN NEW;
END;
$$;

CREATE TRIGGER validate_task_metadata
  BEFORE INSERT OR UPDATE ON tasks
  FOR EACH ROW
  EXECUTE FUNCTION public.validate_task_metadata();
```

## 6. Enforce Row Limits per User

```sql
-- file: supabase/migrations/trigger_row_limit.sql

CREATE OR REPLACE FUNCTION public.enforce_row_limit()
RETURNS trigger
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_count int;
  v_limit int;
  v_plan text;
BEGIN
  SELECT COALESCE(s.plan, 'free') INTO v_plan
  FROM subscriptions s WHERE s.user_id = auth.uid() AND s.status = 'active'
  ORDER BY s.created_at DESC LIMIT 1;

  v_limit := CASE v_plan
    WHEN 'pro' THEN 1000
    WHEN 'enterprise' THEN 10000
    ELSE 100
  END;

  EXECUTE format(
    'SELECT COUNT(*) FROM %I WHERE created_by = $1 AND deleted_at IS NULL',
    TG_TABLE_NAME
  ) INTO v_count USING auth.uid();

  IF v_count >= v_limit THEN
    RAISE EXCEPTION 'Row limit reached (% of % for % plan)', v_count, v_limit, v_plan;
  END IF;

  RETURN NEW;
END;
$$;

CREATE TRIGGER enforce_project_limit
  BEFORE INSERT ON projects
  FOR EACH ROW
  EXECUTE FUNCTION public.enforce_row_limit();
```

## 7. Notify on Important Changes (pg_notify for Realtime)

```sql
-- file: supabase/migrations/trigger_notify.sql

CREATE OR REPLACE FUNCTION public.notify_assignment()
RETURNS trigger
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
BEGIN
  IF NEW.assigned_to IS DISTINCT FROM OLD.assigned_to AND NEW.assigned_to IS NOT NULL THEN
    INSERT INTO notifications (user_id, type, title, body, metadata)
    VALUES (
      NEW.assigned_to,
      'task_assigned',
      'New task assigned',
      'You have been assigned to: ' || NEW.title,
      jsonb_build_object('task_id', NEW.id, 'assigned_by', auth.uid())
    );
  END IF;

  RETURN NEW;
END;
$$;

CREATE TRIGGER notify_task_assignment
  AFTER UPDATE OF assigned_to ON tasks
  FOR EACH ROW
  EXECUTE FUNCTION public.notify_assignment();
```

## 8. Maintain Denormalized Counters

```sql
-- file: supabase/migrations/trigger_counter_cache.sql

CREATE OR REPLACE FUNCTION public.update_task_count()
RETURNS trigger
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    UPDATE projects SET task_count = task_count + 1 WHERE id = NEW.project_id;
    RETURN NEW;
  ELSIF TG_OP = 'DELETE' THEN
    UPDATE projects SET task_count = task_count - 1 WHERE id = OLD.project_id;
    RETURN OLD;
  ELSIF TG_OP = 'UPDATE' AND OLD.project_id != NEW.project_id THEN
    UPDATE projects SET task_count = task_count - 1 WHERE id = OLD.project_id;
    UPDATE projects SET task_count = task_count + 1 WHERE id = NEW.project_id;
    RETURN NEW;
  END IF;
  RETURN NULL;
END;
$$;

CREATE TRIGGER update_project_task_count
  AFTER INSERT OR UPDATE OF project_id OR DELETE ON tasks
  FOR EACH ROW
  EXECUTE FUNCTION public.update_task_count();
```

## 9. Prevent Accidental Data Loss

```sql
-- file: supabase/migrations/trigger_prevent_mass_delete.sql

CREATE OR REPLACE FUNCTION public.prevent_mass_delete()
RETURNS trigger
LANGUAGE plpgsql
AS $$
DECLARE
  v_total int;
  v_deleting int;
BEGIN
  EXECUTE format('SELECT COUNT(*) FROM %I', TG_TABLE_NAME) INTO v_total;

  -- Block if deleting more than 50% of rows in a single transaction
  IF v_total > 10 THEN
    EXECUTE format(
      'SELECT COUNT(*) FROM %I WHERE id IN (SELECT id FROM old_table)',
      TG_TABLE_NAME
    ) INTO v_deleting;

    IF v_deleting > v_total * 0.5 THEN
      RAISE EXCEPTION 'Mass delete blocked: attempting to delete % of % rows', v_deleting, v_total;
    END IF;
  END IF;

  RETURN OLD;
END;
$$;
```

## 10. Set Default Values from User Profile

```sql
-- file: supabase/migrations/trigger_set_defaults.sql

CREATE OR REPLACE FUNCTION public.set_user_defaults()
RETURNS trigger
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_profile record;
BEGIN
  SELECT org_id, timezone INTO v_profile
  FROM profiles WHERE id = auth.uid();

  IF NEW.org_id IS NULL THEN
    NEW.org_id := v_profile.org_id;
  END IF;

  IF NEW.created_by IS NULL THEN
    NEW.created_by := auth.uid();
  END IF;

  RETURN NEW;
END;
$$;

CREATE TRIGGER set_defaults_projects
  BEFORE INSERT ON projects
  FOR EACH ROW
  EXECUTE FUNCTION public.set_user_defaults();

CREATE TRIGGER set_defaults_tasks
  BEFORE INSERT ON tasks
  FOR EACH ROW
  EXECUTE FUNCTION public.set_user_defaults();
```
