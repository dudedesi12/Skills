# Safe Migration Workflow with Rollback

## Principles

1. **Every migration must be reversible** — always write a rollback file
2. **Never modify an applied migration** — create a new one instead
3. **One change per migration** — small, atomic changes
4. **Test on a branch first** — use Supabase Branching
5. **Zero-downtime** — never lock tables for extended periods

---

## Workflow

### Step 1: Create a Migration

```bash
# Generate a timestamped migration file
supabase migration new add_priority_to_projects
# Creates: supabase/migrations/20250101000000_add_priority_to_projects.sql
```

### Step 2: Write the Migration

```sql
-- file: supabase/migrations/20250101000000_add_priority_to_projects.sql

-- Add column as nullable first (instant, no lock)
ALTER TABLE projects ADD COLUMN priority int;
```

### Step 3: Write the Rollback

```sql
-- file: supabase/migrations/20250101000000_add_priority_to_projects_rollback.sql

ALTER TABLE projects DROP COLUMN IF EXISTS priority;
```

### Step 4: Test Locally

```bash
# Reset local database and run all migrations
supabase db reset

# Check diff between local schema and migrations
supabase db diff

# Run the specific migration
supabase migration up
```

### Step 5: Push to Branch (Preview)

```bash
# Create a branch in Supabase (linked to Git branch)
git checkout -b feat/add-priority
git add supabase/migrations/
git commit -m "Add priority column to projects"
git push origin feat/add-priority

# Supabase Branching auto-applies the migration to the branch database
```

### Step 6: Merge to Production

```bash
# After testing on the preview branch
git checkout main
git merge feat/add-priority
git push origin main
# Migration runs on production automatically
```

---

## Zero-Downtime Patterns

### Adding a Column

```sql
-- file: supabase/migrations/safe_add_column.sql

-- SAFE: Add nullable column (instant, no lock)
ALTER TABLE projects ADD COLUMN priority int;

-- SAFE: Set default for new rows
ALTER TABLE projects ALTER COLUMN priority SET DEFAULT 0;

-- SAFE: Backfill in batches
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
    UPDATE projects SET priority = 0
    WHERE id IN (SELECT id FROM batch);
    GET DIAGNOSTICS rows_updated = ROW_COUNT;
    PERFORM pg_sleep(0.1);
  END LOOP;
END;
$$;

-- SAFE: Add NOT NULL after backfill completes
ALTER TABLE projects ALTER COLUMN priority SET NOT NULL;
```

**Rollback:**

```sql
-- file: supabase/migrations/safe_add_column_rollback.sql

ALTER TABLE projects DROP COLUMN IF EXISTS priority;
```

### Removing a Column

```sql
-- file: supabase/migrations/safe_remove_column.sql

-- Step 1: Stop writing to the column in your app code FIRST
-- Step 2: Deploy the code change
-- Step 3: Wait for all old requests to complete (a few minutes)
-- Step 4: Then drop the column

ALTER TABLE projects DROP COLUMN IF EXISTS legacy_field;
```

### Renaming a Column

Never rename directly. Use a 3-step approach:

```sql
-- file: supabase/migrations/rename_column_step1.sql
-- Step 1: Add new column
ALTER TABLE projects ADD COLUMN display_name text;

-- file: supabase/migrations/rename_column_step2.sql
-- Step 2: Copy data (after app writes to both columns)
UPDATE projects SET display_name = name WHERE display_name IS NULL;

-- file: supabase/migrations/rename_column_step3.sql
-- Step 3: Drop old column (after app only reads from new column)
ALTER TABLE projects DROP COLUMN name;
```

### Adding an Index

```sql
-- file: supabase/migrations/safe_add_index.sql

-- ALWAYS use CONCURRENTLY (does not lock table)
CREATE INDEX CONCURRENTLY idx_tasks_status
  ON tasks (status)
  WHERE deleted_at IS NULL;
```

**Rollback:**

```sql
-- file: supabase/migrations/safe_add_index_rollback.sql

DROP INDEX CONCURRENTLY IF EXISTS idx_tasks_status;
```

### Changing a Column Type

```sql
-- file: supabase/migrations/safe_change_type.sql

-- DANGER: ALTER COLUMN TYPE locks the table and rewrites it
-- Instead, use the add-copy-drop pattern:

-- Step 1: Add new column with desired type
ALTER TABLE tasks ADD COLUMN priority_new smallint;

-- Step 2: Backfill
UPDATE tasks SET priority_new = priority::smallint;

-- Step 3: Swap (requires brief lock)
ALTER TABLE tasks RENAME COLUMN priority TO priority_old;
ALTER TABLE tasks RENAME COLUMN priority_new TO priority;

-- Step 4: Drop old column later
ALTER TABLE tasks DROP COLUMN priority_old;
```

### Adding a NOT NULL Constraint

```sql
-- file: supabase/migrations/safe_add_not_null.sql

-- DANGER: ALTER COLUMN SET NOT NULL scans entire table
-- Instead, use a CHECK constraint (validated in background):

-- Step 1: Add as NOT VALID (instant, no scan)
ALTER TABLE projects
  ADD CONSTRAINT projects_priority_not_null
  CHECK (priority IS NOT NULL) NOT VALID;

-- Step 2: Validate in background (does not block writes)
ALTER TABLE projects
  VALIDATE CONSTRAINT projects_priority_not_null;

-- Step 3 (optional): Convert to NOT NULL
-- PostgreSQL recognizes the CHECK constraint and skips the scan
ALTER TABLE projects ALTER COLUMN priority SET NOT NULL;
ALTER TABLE projects DROP CONSTRAINT projects_priority_not_null;
```

---

## Adding a New Table

```sql
-- file: supabase/migrations/create_notifications.sql

CREATE TABLE IF NOT EXISTS public.notifications (
  id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  type text NOT NULL CHECK (type IN ('task_assigned', 'mention', 'comment', 'invite')),
  title text NOT NULL,
  body text,
  metadata jsonb DEFAULT '{}',
  read_at timestamptz,
  created_at timestamptz DEFAULT now()
);

-- Indexes
CREATE INDEX idx_notifications_user ON notifications (user_id, created_at DESC)
  WHERE read_at IS NULL;

-- RLS
ALTER TABLE notifications ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users see own notifications" ON notifications
  FOR SELECT USING (user_id = auth.uid());

CREATE POLICY "Users mark own as read" ON notifications
  FOR UPDATE USING (user_id = auth.uid())
  WITH CHECK (user_id = auth.uid());

-- Trigger for updated_at
CREATE TRIGGER set_updated_at
  BEFORE UPDATE ON notifications
  FOR EACH ROW
  EXECUTE FUNCTION public.handle_updated_at();
```

**Rollback:**

```sql
-- file: supabase/migrations/create_notifications_rollback.sql

DROP TABLE IF EXISTS public.notifications;
```

---

## Seeding Data

```sql
-- file: supabase/seed.sql

-- Only runs on local/branch databases, NOT production
INSERT INTO organizations (id, name, plan) VALUES
  ('00000000-0000-0000-0000-000000000001', 'Demo Org', 'pro')
ON CONFLICT DO NOTHING;

INSERT INTO profiles (id, org_id, display_name, email, role) VALUES
  ('00000000-0000-0000-0000-000000000010', '00000000-0000-0000-0000-000000000001', 'Admin User', 'admin@demo.com', 'admin'),
  ('00000000-0000-0000-0000-000000000011', '00000000-0000-0000-0000-000000000001', 'Regular User', 'user@demo.com', 'user')
ON CONFLICT DO NOTHING;
```

---

## Migration Checklist

Before merging any migration to production:

- [ ] Migration file has a rollback counterpart
- [ ] Tested on local database with `supabase db reset`
- [ ] No `ALTER COLUMN TYPE` on large tables (use add-copy-drop)
- [ ] All `CREATE INDEX` use `CONCURRENTLY`
- [ ] No `NOT NULL` added without backfilling first
- [ ] Columns added as nullable initially
- [ ] Deployed on a Supabase branch and tested with preview deployment
- [ ] App code handles both old and new schema during transition
- [ ] pg_stat_statements checked for query plan regressions
