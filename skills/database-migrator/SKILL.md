---
name: database-migrator
description: "Use this skill whenever the user mentions database migration, schema migration, supabase migration, migration CLI, 'supabase db', schema diff, schema drift, migration testing, migration rollback, seed data, migration CI, migration review, 'change the database', 'add a table', 'alter a column', migration pipeline, data migration, migration workflow, 'add a column', 'rename a column', 'change column type', or ANY database schema change task — even if they don't explicitly say 'migration'. This skill guides safe schema changes from dev to production."
---

# Database Migrator

Safe database schema changes from dev to production. This skill focuses on workflows and CI automation. Cross-reference: supabase-power-user §8 for SQL migration patterns.

## 1. Supabase Migration CLI

```bash
# Create a new migration
supabase migration new add_visa_tracking

# Apply migrations to local database
supabase db reset    # Reset and replay all migrations

# Check migration status
supabase migration list

# Generate migration from schema changes made in Studio
supabase db diff --use-migra -f add_visa_tracking
```

## 2. Migration File Organization

```
supabase/
  migrations/
    20260101000000_initial_schema.sql
    20260115000000_add_profiles_table.sql
    20260201000000_add_assessments.sql
    20260315000000_add_visa_tracking.sql
    20260408000000_add_audit_logs.sql
  seed.sql                    ← Development seed data
  config.toml                 ← Supabase project config
```

**Naming convention:** `YYYYMMDDHHMMSS_description.sql`

## 3. Safe Schema Changes

### Safe Operations (No Downtime)

```sql
-- Adding a nullable column — always safe
ALTER TABLE profiles ADD COLUMN phone text;

-- Adding a column with a default — safe in PostgreSQL 11+
ALTER TABLE assessments ADD COLUMN version integer DEFAULT 1;

-- Creating a new table — always safe
CREATE TABLE visa_tracking (
  id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id uuid REFERENCES auth.users(id) ON DELETE CASCADE,
  visa_subclass text NOT NULL,
  status text DEFAULT 'pending',
  created_at timestamptz DEFAULT now()
);

-- Adding an index concurrently — safe, doesn't lock table
CREATE INDEX CONCURRENTLY idx_assessments_user ON assessments(user_id);
```

### Dangerous Operations (Requires Care)

```sql
-- DANGEROUS: Adding NOT NULL without default locks table
-- BAD:
ALTER TABLE profiles ADD COLUMN role text NOT NULL;

-- SAFE: Add nullable, backfill, then add constraint
ALTER TABLE profiles ADD COLUMN role text;
UPDATE profiles SET role = 'user' WHERE role IS NULL;
ALTER TABLE profiles ALTER COLUMN role SET NOT NULL;
ALTER TABLE profiles ALTER COLUMN role SET DEFAULT 'user';

-- DANGEROUS: Renaming a column breaks running code
-- Do it in 3 deployments:
-- Deploy 1: Add new column, write to both
-- Deploy 2: Read from new column, still write to both
-- Deploy 3: Drop old column

-- DANGEROUS: Changing column type
-- BAD:
ALTER TABLE assessments ALTER COLUMN score TYPE numeric;
-- SAFE: Add new column, migrate data, swap
ALTER TABLE assessments ADD COLUMN score_numeric numeric;
UPDATE assessments SET score_numeric = score::numeric;
-- After code deploys: ALTER TABLE assessments DROP COLUMN score;
```

## 4. Seed Data Management

```sql
-- supabase/seed.sql
-- Run with: supabase db reset (applies migrations then seed)

-- Test users (only for development)
INSERT INTO auth.users (id, email, encrypted_password, email_confirmed_at)
VALUES
  ('00000000-0000-0000-0000-000000000001', 'admin@test.com', crypt('password123', gen_salt('bf')), now()),
  ('00000000-0000-0000-0000-000000000002', 'user@test.com', crypt('password123', gen_salt('bf')), now());

INSERT INTO profiles (id, full_name, role, email)
VALUES
  ('00000000-0000-0000-0000-000000000001', 'Test Admin', 'admin', 'admin@test.com'),
  ('00000000-0000-0000-0000-000000000002', 'Test User', 'user', 'user@test.com');

-- Sample assessment data
INSERT INTO assessments (user_id, visa_subclass, points_score, result)
VALUES
  ('00000000-0000-0000-0000-000000000002', '189', 80, 'eligible'),
  ('00000000-0000-0000-0000-000000000002', '190', 75, 'eligible');
```

## 5. Schema Diffing

```bash
# Compare local schema to remote
supabase db diff --use-migra --linked

# Compare local schema to a specific migration
supabase db diff --use-migra

# Generate a migration from Studio changes
supabase db diff --use-migra -f my_changes
```

### Detecting Schema Drift

```bash
# Check if remote database matches your migrations
supabase db lint

# Pull the remote schema
supabase db pull

# Compare pulled schema with your local migrations
git diff supabase/migrations/
```

## 6. CI/CD Migration Pipeline

```yaml
# .github/workflows/migration-check.yml
name: Migration Validation

on:
  pull_request:
    paths: ["supabase/migrations/**"]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: supabase/setup-cli@v1
        with:
          version: latest

      - name: Start local Supabase
        run: supabase start

      - name: Apply migrations
        run: supabase db reset

      - name: Run schema lint
        run: supabase db lint

      - name: Verify seed data loads
        run: supabase db reset # Includes seed

      - name: Stop Supabase
        run: supabase stop
```

## 7. RLS Policy Versioning

Track RLS policy changes in migrations:

```sql
-- supabase/migrations/20260408000000_add_visa_tracking_rls.sql
-- Enable RLS
ALTER TABLE visa_tracking ENABLE ROW LEVEL SECURITY;

-- Users can read their own tracking data
CREATE POLICY "Users read own visa tracking"
  ON visa_tracking FOR SELECT
  USING (auth.uid() = user_id);

-- Users can insert their own tracking data
CREATE POLICY "Users insert own visa tracking"
  ON visa_tracking FOR INSERT
  WITH CHECK (auth.uid() = user_id);

-- Admins can read all tracking data
CREATE POLICY "Admins read all visa tracking"
  ON visa_tracking FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM profiles
      WHERE profiles.id = auth.uid()
      AND profiles.role = 'admin'
    )
  );
```

## 8. Rollback Strategies

```sql
-- Option 1: Reverse migration (manual)
-- For every migration, keep a mental (or written) rollback plan

-- Migration: add column
ALTER TABLE profiles ADD COLUMN bio text;
-- Rollback: drop column
ALTER TABLE profiles DROP COLUMN bio;

-- Migration: add table
CREATE TABLE visa_tracking (...);
-- Rollback: drop table
DROP TABLE IF EXISTS visa_tracking;
```

```bash
# Option 2: Repair migration history
# If a migration was applied but needs to be undone:
supabase migration repair --status reverted 20260408000000
# Then manually undo the SQL changes
```

## 9. Team Coordination

**Before merging a migration PR:**
- [ ] Migration file is named with correct timestamp
- [ ] SQL is reviewed by at least one person
- [ ] No destructive operations without a migration plan
- [ ] RLS policies included for new tables
- [ ] Indexes added for frequently queried columns
- [ ] Seed data updated if needed
- [ ] CI migration check passes

**Handling conflicts:**
- If two PRs add migrations with close timestamps, rename the later one
- Never reorder existing migrations
- If you need to fix a deployed migration, create a new migration to correct it

## Rules

1. **Never edit a deployed migration** — Create a new migration to fix issues.
2. **Always add RLS** — Every new table needs RLS enabled and policies defined.
3. **Add columns as nullable first** — Then backfill, then add NOT NULL constraint.
4. **Use CONCURRENTLY for indexes** — `CREATE INDEX CONCURRENTLY` doesn't lock the table.
5. **Test with `db reset`** — Run `supabase db reset` before pushing to verify the full migration chain.
6. **One concern per migration** — Don't mix table creation with data migration in one file.
7. **Keep a rollback plan** — For every migration, know how to reverse it.
8. **Update seed data** — When you add required columns, update seed.sql to match.
9. **Review migration PRs carefully** — Schema changes are the hardest to reverse.
10. **Never run migrations directly in production** — Always go through CI/CD pipeline.

See `references/` for detailed guides:
- `migration-workflow.md` — End-to-end migration lifecycle
- `safe-changes.md` — Safe vs dangerous DDL operations
- `ci-pipeline.md` — GitHub Actions for migration validation
