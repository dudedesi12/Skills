# Safe vs Dangerous DDL Operations

## Quick Reference

| Operation | Safe? | Notes |
|-----------|-------|-------|
| Add nullable column | Yes | No lock, instant |
| Add column with default | Yes | PostgreSQL 11+ handles this efficiently |
| Add NOT NULL column | **No** | Add nullable → backfill → set NOT NULL |
| Drop column | Yes* | Ensure no code references it first |
| Rename column | **No** | Add new → migrate → drop old (3 deploys) |
| Change column type | **No** | Add new column → migrate → swap |
| Add table | Yes | Always safe |
| Drop table | Yes* | Ensure no code references it |
| Add index | **Careful** | Use `CREATE INDEX CONCURRENTLY` |
| Drop index | Yes | But verify query plans first |
| Add foreign key | **Careful** | Validates existing data, can lock |
| Add CHECK constraint | **Careful** | Use `NOT VALID` then `VALIDATE` separately |
| Enable RLS | Yes | But add policies first! |
| Add RLS policy | Yes | Instant |

## Detailed Patterns

### Adding NOT NULL Column Safely

```sql
-- Step 1: Add as nullable (instant, no lock)
ALTER TABLE profiles ADD COLUMN timezone text;

-- Step 2: Backfill existing rows (can be slow on large tables)
UPDATE profiles SET timezone = 'Australia/Sydney' WHERE timezone IS NULL;

-- Step 3: Add NOT NULL constraint (instant after backfill)
ALTER TABLE profiles ALTER COLUMN timezone SET NOT NULL;
ALTER TABLE profiles ALTER COLUMN timezone SET DEFAULT 'Australia/Sydney';
```

### Adding Foreign Key Safely

```sql
-- BAD: Validates all existing data, can lock for a long time
ALTER TABLE assessments ADD CONSTRAINT fk_user FOREIGN KEY (user_id) REFERENCES profiles(id);

-- GOOD: Add as NOT VALID (instant), then validate separately
ALTER TABLE assessments ADD CONSTRAINT fk_user
  FOREIGN KEY (user_id) REFERENCES profiles(id) NOT VALID;

-- Validate in a separate migration (runs in background)
ALTER TABLE assessments VALIDATE CONSTRAINT fk_user;
```

### Adding CHECK Constraint Safely

```sql
-- Same pattern: NOT VALID then VALIDATE
ALTER TABLE assessments ADD CONSTRAINT check_score
  CHECK (points_score >= 0 AND points_score <= 130) NOT VALID;

ALTER TABLE assessments VALIDATE CONSTRAINT check_score;
```

### Creating Indexes Without Downtime

```sql
-- BAD: Locks the table during index creation
CREATE INDEX idx_assessments_user ON assessments(user_id);

-- GOOD: Creates index without locking (takes longer but no downtime)
CREATE INDEX CONCURRENTLY idx_assessments_user ON assessments(user_id);

-- Note: CONCURRENTLY can't run inside a transaction
-- So this must be the ONLY statement in the migration file
```

### Dropping a Column Safely

```sql
-- Step 1: Remove all code references to the column
-- Step 2: Deploy code changes
-- Step 3: Drop the column in a migration
ALTER TABLE profiles DROP COLUMN IF EXISTS legacy_field;
```

### Changing Column Type Safely

```sql
-- Step 1: Add new column with desired type
ALTER TABLE assessments ADD COLUMN score_v2 numeric;

-- Step 2: Backfill
UPDATE assessments SET score_v2 = score::numeric;

-- Step 3: Deploy code to read from score_v2
-- Step 4: Drop old column
ALTER TABLE assessments DROP COLUMN score;

-- Step 5: Rename (optional)
ALTER TABLE assessments RENAME COLUMN score_v2 TO score;
```
