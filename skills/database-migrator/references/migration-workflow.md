# End-to-End Migration Workflow

## The Full Lifecycle

```
1. Local Development
   └── Make changes in Supabase Studio or SQL
   └── Generate migration: supabase db diff -f description
   └── Test: supabase db reset

2. Code Review
   └── Create PR with migration file
   └── CI validates migration applies cleanly
   └── Team reviews SQL

3. Staging (Supabase Branch)
   └── supabase branches create staging-branch
   └── Apply migration to branch database
   └── Test application against branch

4. Production
   └── Merge PR to main
   └── Deploy migration: supabase db push
   └── Verify in production
   └── Monitor for errors
```

## Step-by-Step: Adding a New Feature

### 1. Start Local Supabase

```bash
supabase start
# Opens Studio at http://localhost:54323
```

### 2. Make Schema Changes

Option A: Use Supabase Studio (GUI)
- Navigate to Table Editor
- Create/modify tables
- Then generate migration:

```bash
supabase db diff --use-migra -f add_visa_notifications
```

Option B: Write SQL directly:

```bash
supabase migration new add_visa_notifications
# Edit: supabase/migrations/20260408120000_add_visa_notifications.sql
```

### 3. Test Locally

```bash
# Reset database and replay all migrations + seed
supabase db reset

# Start your app and test
npm run dev
```

### 4. Commit and Push

```bash
git add supabase/migrations/20260408120000_add_visa_notifications.sql
git commit -m "migration: add visa notifications table"
git push
```

### 5. CI Validates

GitHub Actions runs:
- Starts local Supabase
- Applies all migrations
- Runs lint
- Confirms clean apply

### 6. Deploy to Production

After PR merge:

```bash
# Link to your production project (one-time)
supabase link --project-ref your-project-ref

# Push migrations to production
supabase db push

# Verify
supabase migration list --linked
```

## Multi-Step Migration Example

Renaming a column safely (requires 3 separate migrations across deployments):

```sql
-- Migration 1: Add new column
-- File: 20260408000000_add_display_name.sql
ALTER TABLE profiles ADD COLUMN display_name text;
UPDATE profiles SET display_name = full_name;

-- Migration 2: (after code deploys reading from display_name)
-- File: 20260409000000_default_display_name.sql
ALTER TABLE profiles ALTER COLUMN display_name SET DEFAULT '';
ALTER TABLE profiles ALTER COLUMN display_name SET NOT NULL;

-- Migration 3: (after code stops using full_name)
-- File: 20260415000000_drop_full_name.sql
ALTER TABLE profiles DROP COLUMN full_name;
```

## Data Migration Script Template

For complex data transformations that don't fit in SQL:

```typescript
// scripts/migrate-data.ts
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

const BATCH_SIZE = 100;

async function migrateData() {
  console.log("Starting data migration...");

  let offset = 0;
  let totalMigrated = 0;

  while (true) {
    const { data: batch, error } = await supabase
      .from("assessments")
      .select("id, raw_data")
      .is("migrated_at", null)
      .range(offset, offset + BATCH_SIZE - 1);

    if (error) throw new Error(`Query failed: ${error.message}`);
    if (!batch || batch.length === 0) break;

    for (const row of batch) {
      const transformed = transformData(row.raw_data);
      const { error: updateError } = await supabase
        .from("assessments")
        .update({ processed_data: transformed, migrated_at: new Date().toISOString() })
        .eq("id", row.id);

      if (updateError) {
        console.error(`Failed to migrate ${row.id}: ${updateError.message}`);
        continue;
      }
      totalMigrated++;
    }

    console.log(`Migrated ${totalMigrated} rows...`);
    offset += BATCH_SIZE;
  }

  console.log(`Migration complete. Total: ${totalMigrated} rows.`);
}

function transformData(raw: unknown): unknown {
  // Your transformation logic here
  return raw;
}

migrateData().catch(console.error);
```

Run with: `npx tsx scripts/migrate-data.ts`
