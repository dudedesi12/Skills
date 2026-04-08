# CI Pipeline for Database Migrations

## GitHub Actions: Migration Validation

```yaml
# .github/workflows/migration-check.yml
name: Validate Migrations

on:
  pull_request:
    paths:
      - "supabase/migrations/**"
      - "supabase/seed.sql"

jobs:
  validate-migrations:
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - uses: actions/checkout@v4

      - uses: supabase/setup-cli@v1
        with:
          version: latest

      - name: Start local Supabase
        run: supabase start -x realtime,storage-api,imgproxy,inbucket,postgrest,pgadmin-schema-diff,migra,studio,edge-runtime,logflare,vector,supavisor

      - name: Apply all migrations
        run: supabase db reset
        id: migrations

      - name: Run database linter
        run: supabase db lint
        if: always()

      - name: Check for schema drift
        run: |
          supabase db diff --use-migra > /tmp/drift.sql
          if [ -s /tmp/drift.sql ]; then
            echo "Schema drift detected!"
            cat /tmp/drift.sql
            exit 1
          fi
          echo "No schema drift."

      - name: Stop Supabase
        if: always()
        run: supabase stop
```

## Production Deployment Workflow

```yaml
# .github/workflows/deploy-migrations.yml
name: Deploy Migrations

on:
  push:
    branches: [main]
    paths:
      - "supabase/migrations/**"

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # Requires approval

    steps:
      - uses: actions/checkout@v4

      - uses: supabase/setup-cli@v1
        with:
          version: latest

      - name: Link to production
        run: supabase link --project-ref ${{ secrets.SUPABASE_PROJECT_REF }}
        env:
          SUPABASE_ACCESS_TOKEN: ${{ secrets.SUPABASE_ACCESS_TOKEN }}

      - name: Push migrations
        run: supabase db push
        env:
          SUPABASE_ACCESS_TOKEN: ${{ secrets.SUPABASE_ACCESS_TOKEN }}

      - name: Verify migration status
        run: supabase migration list --linked
        env:
          SUPABASE_ACCESS_TOKEN: ${{ secrets.SUPABASE_ACCESS_TOKEN }}
```

## Required Secrets

Add these to GitHub Settings → Secrets:

| Secret | Description | Where to Find |
|--------|-------------|---------------|
| SUPABASE_ACCESS_TOKEN | Personal access token | Supabase Dashboard → Account → Access Tokens |
| SUPABASE_PROJECT_REF | Project reference ID | Supabase Dashboard → Settings → General |
| SUPABASE_DB_PASSWORD | Database password | Supabase Dashboard → Settings → Database |

## PR Review Checklist

Add this as a PR template for migration PRs:

```markdown
## Migration Review Checklist

### Safety
- [ ] No direct `ALTER TABLE ... NOT NULL` without backfill
- [ ] Indexes use `CREATE INDEX CONCURRENTLY`
- [ ] Foreign keys use `NOT VALID` pattern
- [ ] No column renames (use add/migrate/drop pattern)
- [ ] No column type changes (use add/migrate/drop pattern)

### Completeness
- [ ] RLS enabled on new tables
- [ ] RLS policies defined for all operations
- [ ] Indexes added for frequently queried columns
- [ ] Seed data updated for new tables/columns
- [ ] Migration file has a descriptive name

### Testing
- [ ] `supabase db reset` runs cleanly
- [ ] Application works with the new schema
- [ ] Rollback plan documented (even if informal)
```
