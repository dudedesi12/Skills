# Supabase Power User Blueprint

## Purpose

Provide production-ready patterns for advanced Supabase/PostgreSQL operations in Next.js App Router apps. The user has zero coding knowledge, so every example is complete and copy-pasteable.

## Architecture

```
supabase/
  migrations/          # Sequential SQL migration files
    001_multi_tenant_rls.sql
    002_role_hierarchy_rls.sql
    ...
src/
  lib/
    supabase/
      server.ts        # Server-side Supabase client
      client.ts        # Browser-side Supabase client
      env.ts           # Environment config
    db/
      pool.ts          # Direct pg pool for heavy queries
  app/
    api/
      team/invite/     # Team invite endpoint
      dashboard/stats/ # Dashboard stats endpoint
      search/          # Full-text search endpoint
```

## Key Patterns

1. **RLS First** — Every table has RLS. No exceptions. Use helper functions for DRY policies.
2. **Logic in the Database** — Business rules as PL/pgSQL functions for atomicity.
3. **Triggers for Automation** — Timestamps, soft deletes, audit logs run automatically.
4. **Materialized Views** — Pre-compute expensive dashboard queries, refresh with pg_cron.
5. **Full-Text Search** — Use tsvector with weighted columns instead of LIKE queries.
6. **Safe Migrations** — Add columns nullable first, backfill in batches, then add constraints.
7. **Branching** — Each Git branch gets its own database for safe testing.

## Stack

- Next.js App Router + TypeScript
- Supabase (PostgreSQL)
- Tailwind CSS
- Vercel deployment
- Gemini API for AI features (NEVER OpenAI)

## Reference Files

- `references/advanced-rls.md` — 20+ RLS policy templates
- `references/plpgsql-functions.md` — 15+ production database functions
- `references/triggers-patterns.md` — Common trigger patterns
- `references/performance.md` — Query optimization and indexing
- `references/migration-guide.md` — Safe migration workflow with rollback
