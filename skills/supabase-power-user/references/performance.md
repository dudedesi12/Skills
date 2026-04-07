# Query Optimization and Performance

## Index Strategy

### When to Add Indexes

Add indexes for columns used in:
- `WHERE` clauses
- `JOIN` conditions
- `ORDER BY` clauses
- Foreign keys (PostgreSQL does NOT auto-index foreign keys)

### Essential Indexes

```sql
-- file: supabase/migrations/indexes.sql

-- Foreign keys (always index these)
CREATE INDEX CONCURRENTLY idx_tasks_project_id ON tasks (project_id);
CREATE INDEX CONCURRENTLY idx_tasks_assigned_to ON tasks (assigned_to);
CREATE INDEX CONCURRENTLY idx_profiles_org_id ON profiles (org_id);
CREATE INDEX CONCURRENTLY idx_comments_task_id ON comments (task_id);

-- Common query patterns
CREATE INDEX CONCURRENTLY idx_tasks_status ON tasks (status) WHERE deleted_at IS NULL;
CREATE INDEX CONCURRENTLY idx_projects_org_created ON projects (org_id, created_at DESC) WHERE deleted_at IS NULL;
CREATE INDEX CONCURRENTLY idx_tasks_project_status ON tasks (project_id, status) WHERE deleted_at IS NULL;

-- Full-text search
CREATE INDEX CONCURRENTLY idx_articles_search ON articles USING gin(search_vector);

-- JSONB columns
CREATE INDEX CONCURRENTLY idx_tasks_metadata ON tasks USING gin(metadata);
```

### Partial Indexes (Smaller, Faster)

```sql
-- file: supabase/migrations/partial_indexes.sql

-- Only index non-deleted rows
CREATE INDEX CONCURRENTLY idx_active_projects
  ON projects (org_id, created_at DESC)
  WHERE deleted_at IS NULL;

-- Only index pending invites
CREATE INDEX CONCURRENTLY idx_pending_invites
  ON invites (email, org_id)
  WHERE accepted_at IS NULL AND expires_at > now();

-- Only index high-priority tasks
CREATE INDEX CONCURRENTLY idx_high_priority_tasks
  ON tasks (project_id, created_at DESC)
  WHERE priority >= 4 AND deleted_at IS NULL;
```

### Composite Indexes

```sql
-- file: supabase/migrations/composite_indexes.sql

-- Order matters: most selective column first for equality, then range/sort
CREATE INDEX CONCURRENTLY idx_tasks_lookup
  ON tasks (project_id, status, created_at DESC)
  WHERE deleted_at IS NULL;

-- Covering index (includes columns needed by SELECT to avoid table lookup)
CREATE INDEX CONCURRENTLY idx_tasks_list
  ON tasks (project_id, status, created_at DESC)
  INCLUDE (title, priority, assigned_to)
  WHERE deleted_at IS NULL;
```

---

## EXPLAIN ANALYZE

Always check your query plans.

### How to Use

```sql
-- Show the full execution plan with actual timings
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT *
FROM tasks
WHERE project_id = 'some-uuid'
  AND status = 'in_progress'
  AND deleted_at IS NULL
ORDER BY created_at DESC
LIMIT 20;
```

### What to Look For

| Bad Sign | Meaning | Fix |
|----------|---------|-----|
| `Seq Scan` on large table | No index used | Add an index |
| `Sort` with high cost | Sorting in memory | Add index on ORDER BY column |
| `Nested Loop` with high rows | Expensive join | Check join conditions, add indexes |
| `Hash Join` with large build | Large hash table | Filter earlier, add indexes |
| `Rows Removed by Filter` is high | Index not selective enough | Use partial index or composite index |
| `Buffers: shared read` is high | Many disk reads | Increase shared_buffers or add index |

### Running EXPLAIN from Next.js (Dev Only)

```typescript
// file: src/lib/db/explain.ts

import { getPool } from "./pool";

export async function explainQuery(
  query: string,
  params?: unknown[]
): Promise<string> {
  if (process.env.NODE_ENV !== "development") {
    throw new Error("EXPLAIN only available in development");
  }

  const pool = getPool();
  const client = await pool.connect();
  try {
    const result = await client.query(
      `EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) ${query}`,
      params
    );
    return result.rows.map((r: Record<string, string>) => r["QUERY PLAN"]).join("\n");
  } catch (err) {
    console.error("EXPLAIN error:", err);
    throw err;
  } finally {
    client.release();
  }
}
```

---

## Connection Pooling

### Supabase Connection Types

| Connection | Port | Use Case |
|-----------|------|----------|
| Direct | 5432 | Migrations, long-running queries |
| Pooler (Transaction mode) | 6543 | App queries (default) |
| Pooler (Session mode) | 5432 | Prepared statements, LISTEN/NOTIFY |

### Environment Setup

```bash
# file: .env.local

# For app queries (transaction pooling via Supavisor)
DATABASE_URL="postgresql://postgres.[ref]:[password]@aws-0-us-east-1.pooler.supabase.com:6543/postgres?pgbouncer=true"

# For migrations (direct connection)
DIRECT_URL="postgresql://postgres.[ref]:[password]@aws-0-us-east-1.pooler.supabase.com:5432/postgres"
```

### Pool Configuration

```typescript
// file: src/lib/db/pool.ts

import { Pool } from "pg";

let pool: Pool | null = null;

export function getPool(): Pool {
  if (!pool) {
    pool = new Pool({
      connectionString: process.env.DATABASE_URL,
      max: 10,                    // Max connections in pool
      idleTimeoutMillis: 30000,   // Close idle connections after 30s
      connectionTimeoutMillis: 5000, // Fail if can't connect in 5s
      statement_timeout: 30000,   // Kill queries after 30s
    });

    pool.on("error", (err) => {
      console.error("Pool error:", err);
      pool = null;
    });
  }

  return pool;
}
```

---

## Query Optimization Patterns

### 1. Avoid SELECT *

```typescript
// file: src/app/api/tasks/route.ts

// BAD: fetches all columns including large text fields
const { data } = await supabase.from("tasks").select("*");

// GOOD: only fetch what you need
const { data } = await supabase
  .from("tasks")
  .select("id, title, status, priority, created_at")
  .eq("project_id", projectId)
  .eq("deleted_at", null)
  .order("created_at", { ascending: false })
  .limit(20);
```

### 2. Use Cursor Pagination (Not OFFSET)

```typescript
// file: src/app/api/tasks/paginated/route.ts

import { createClient } from "@/lib/supabase/server";
import { NextRequest, NextResponse } from "next/server";

export async function GET(request: NextRequest) {
  try {
    const supabase = await createClient();
    const { searchParams } = new URL(request.url);
    const cursor = searchParams.get("cursor");
    const limit = Math.min(parseInt(searchParams.get("limit") || "20", 10), 50);
    const projectId = searchParams.get("project_id");

    if (!projectId) {
      return NextResponse.json({ error: "project_id required" }, { status: 400 });
    }

    let query = supabase
      .from("tasks")
      .select("id, title, status, priority, created_at")
      .eq("project_id", projectId)
      .is("deleted_at", null)
      .order("created_at", { ascending: false })
      .limit(limit + 1);

    if (cursor) {
      query = query.lt("created_at", cursor);
    }

    const { data, error } = await query;

    if (error) {
      return NextResponse.json({ error: error.message }, { status: 500 });
    }

    const hasMore = (data?.length || 0) > limit;
    const items = data?.slice(0, limit) || [];
    const nextCursor = hasMore ? items[items.length - 1]?.created_at : null;

    return NextResponse.json({ items, nextCursor, hasMore });
  } catch (err) {
    console.error("Pagination error:", err);
    return NextResponse.json({ error: "Internal server error" }, { status: 500 });
  }
}
```

### 3. Batch Operations

```typescript
// file: src/lib/db/batch.ts

import { createClient } from "@/lib/supabase/server";

export async function batchInsertTasks(
  tasks: { title: string; project_id: string; priority: number }[]
) {
  const supabase = await createClient();
  const BATCH_SIZE = 100;
  const results = [];

  for (let i = 0; i < tasks.length; i += BATCH_SIZE) {
    const batch = tasks.slice(i, i + BATCH_SIZE);
    const { data, error } = await supabase
      .from("tasks")
      .insert(batch)
      .select("id");

    if (error) {
      console.error(`Batch ${i / BATCH_SIZE} failed:`, error);
      throw error;
    }

    results.push(...(data || []));
  }

  return results;
}
```

### 4. Use Materialized Views for Dashboards

```sql
-- file: supabase/migrations/mat_view_dashboard.sql

CREATE MATERIALIZED VIEW org_dashboard_stats AS
SELECT
  o.id AS org_id,
  COUNT(DISTINCT p.id) AS total_projects,
  COUNT(DISTINCT t.id) AS total_tasks,
  COUNT(DISTINCT t.id) FILTER (WHERE t.status = 'completed') AS completed_tasks,
  ROUND(
    COUNT(DISTINCT t.id) FILTER (WHERE t.status = 'completed')::numeric /
    NULLIF(COUNT(DISTINCT t.id), 0) * 100, 1
  ) AS completion_rate
FROM organizations o
LEFT JOIN projects p ON p.org_id = o.id AND p.deleted_at IS NULL
LEFT JOIN tasks t ON t.project_id = p.id AND t.deleted_at IS NULL
GROUP BY o.id
WITH DATA;

CREATE UNIQUE INDEX idx_org_stats ON org_dashboard_stats (org_id);

-- Refresh every 15 minutes via pg_cron
SELECT cron.schedule('refresh-dashboard', '*/15 * * * *',
  'REFRESH MATERIALIZED VIEW CONCURRENTLY org_dashboard_stats');
```

### 5. Table Partitioning for Large Tables

```sql
-- file: supabase/migrations/partitioning.sql

-- Partition audit_log by month for fast queries and easy archival
CREATE TABLE audit_log_partitioned (
  id bigint GENERATED ALWAYS AS IDENTITY,
  table_name text NOT NULL,
  record_id uuid NOT NULL,
  action text NOT NULL,
  old_data jsonb,
  new_data jsonb,
  changed_by uuid,
  changed_at timestamptz DEFAULT now()
) PARTITION BY RANGE (changed_at);

-- Create partitions for each month
CREATE TABLE audit_log_2025_01 PARTITION OF audit_log_partitioned
  FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
CREATE TABLE audit_log_2025_02 PARTITION OF audit_log_partitioned
  FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
-- Continue for each month...

-- Auto-create future partitions with pg_cron
SELECT cron.schedule('create-audit-partition', '0 0 25 * *', $$
  SELECT public.create_next_audit_partition();
$$);
```

---

## Monitoring Queries

### Find Slow Queries

```sql
-- Enable pg_stat_statements (in Supabase dashboard > Extensions)
SELECT
  query,
  calls,
  mean_exec_time::numeric(10,2) AS avg_ms,
  total_exec_time::numeric(10,2) AS total_ms,
  rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;
```

### Find Missing Indexes

```sql
SELECT
  schemaname,
  relname AS table_name,
  seq_scan,
  seq_tup_read,
  idx_scan,
  idx_tup_fetch,
  seq_scan - idx_scan AS too_many_seq_scans
FROM pg_stat_user_tables
WHERE seq_scan > idx_scan
  AND seq_scan > 100
ORDER BY too_many_seq_scans DESC;
```

### Table Bloat Check

```sql
SELECT
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size,
  n_dead_tup,
  n_live_tup,
  ROUND(n_dead_tup::numeric / NULLIF(n_live_tup, 0) * 100, 2) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```
