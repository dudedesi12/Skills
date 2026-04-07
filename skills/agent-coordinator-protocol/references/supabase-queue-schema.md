# Supabase Queue Schema

Database tables for the agent message queue, dead letter queue, workflow contexts, and metrics.

## Migration SQL

Run this in the Supabase SQL Editor:

```sql
-- supabase/migrations/001_agent_coordinator.sql

-- Agent Registry
create table if not exists agent_registry (
  agent_id text primary key,
  name text not null,
  capabilities text[] not null default '{}',
  endpoint text not null,
  max_concurrency integer not null default 5,
  status text not null default 'active' check (status in ('active', 'draining', 'offline')),
  registered_at timestamptz not null default now(),
  last_heartbeat timestamptz not null default now(),
  metadata jsonb default '{}'
);

create index idx_agent_registry_status on agent_registry(status);
create index idx_agent_registry_capabilities on agent_registry using gin(capabilities);
create index idx_agent_registry_heartbeat on agent_registry(last_heartbeat);

-- Agent Message Queue
create table if not exists agent_message_queue (
  id uuid primary key default gen_random_uuid(),
  from_agent text not null references agent_registry(agent_id),
  to_agent text not null references agent_registry(agent_id),
  message_type text not null check (message_type in ('request', 'response', 'event', 'error', 'heartbeat')),
  priority text not null default 'normal' check (priority in ('low', 'normal', 'high', 'critical')),
  payload jsonb not null default '{}',
  workflow_context jsonb not null default '{}',
  correlation_id uuid not null,
  parent_message_id uuid,
  ttl_seconds integer not null default 300,
  retry_count integer not null default 0,
  max_retries integer not null default 3,
  idempotency_key text not null,
  status text not null default 'pending' check (status in ('pending', 'processing', 'completed', 'failed', 'dead_letter')),
  created_at timestamptz not null default now(),
  processed_at timestamptz,
  completed_at timestamptz,
  expires_at timestamptz generated always as (created_at + (ttl_seconds || ' seconds')::interval) stored,
  error_message text
);

create index idx_queue_to_agent_status on agent_message_queue(to_agent, status);
create index idx_queue_correlation on agent_message_queue(correlation_id);
create index idx_queue_idempotency on agent_message_queue(idempotency_key);
create index idx_queue_priority_created on agent_message_queue(priority desc, created_at asc)
  where status = 'pending';
create index idx_queue_expires on agent_message_queue(expires_at)
  where status = 'pending';

-- Unique constraint for idempotency within a time window
create unique index idx_queue_idempotency_unique on agent_message_queue(idempotency_key)
  where status in ('pending', 'processing');

-- Dead Letter Queue
create table if not exists agent_dead_letter_queue (
  id uuid primary key default gen_random_uuid(),
  original_message_id uuid not null,
  from_agent text not null,
  to_agent text not null,
  message_type text not null,
  payload jsonb not null default '{}',
  failure_reason text not null,
  retry_count integer not null default 0,
  original_created_at timestamptz not null,
  moved_at timestamptz not null default now(),
  reprocessed boolean not null default false,
  reprocessed_at timestamptz
);

create index idx_dlq_moved_at on agent_dead_letter_queue(moved_at desc);
create index idx_dlq_reprocessed on agent_dead_letter_queue(reprocessed)
  where reprocessed = false;

-- Workflow Contexts
create table if not exists workflow_contexts (
  workflow_id uuid primary key default gen_random_uuid(),
  context jsonb not null default '{}',
  status text not null default 'running' check (status in ('running', 'completed', 'failed', 'compensating')),
  current_step integer not null default 0,
  total_steps integer not null default 0,
  version integer not null default 1,
  started_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  completed_at timestamptz,
  error jsonb
);

create index idx_workflow_status on workflow_contexts(status);

-- Agent Metrics
create table if not exists agent_metrics (
  id uuid primary key default gen_random_uuid(),
  agent_id text not null references agent_registry(agent_id),
  operation text not null,
  duration_ms integer not null,
  success boolean not null,
  input_tokens integer not null default 0,
  output_tokens integer not null default 0,
  trace_id text not null,
  recorded_at timestamptz not null default now()
);

create index idx_metrics_agent on agent_metrics(agent_id, recorded_at desc);
create index idx_metrics_trace on agent_metrics(trace_id);

-- Partition metrics by month for performance (optional, for high-volume systems)
-- create table agent_metrics_partitioned (like agent_metrics) partition by range (recorded_at);
```

## Row Level Security

```sql
-- supabase/migrations/002_agent_rls.sql

-- Enable RLS on all tables
alter table agent_registry enable row level security;
alter table agent_message_queue enable row level security;
alter table agent_dead_letter_queue enable row level security;
alter table workflow_contexts enable row level security;
alter table agent_metrics enable row level security;

-- Service role has full access (used by server-side API routes)
create policy "Service role full access on agent_registry"
  on agent_registry for all
  using (auth.role() = 'service_role');

create policy "Service role full access on agent_message_queue"
  on agent_message_queue for all
  using (auth.role() = 'service_role');

create policy "Service role full access on agent_dead_letter_queue"
  on agent_dead_letter_queue for all
  using (auth.role() = 'service_role');

create policy "Service role full access on workflow_contexts"
  on workflow_contexts for all
  using (auth.role() = 'service_role');

create policy "Service role full access on agent_metrics"
  on agent_metrics for all
  using (auth.role() = 'service_role');
```

## Helper Functions

```sql
-- supabase/migrations/003_agent_functions.sql

-- Atomic dequeue: claim the next N pending messages for an agent
create or replace function dequeue_messages(
  p_agent_id text,
  p_limit integer default 10
)
returns setof agent_message_queue
language plpgsql
as $$
begin
  return query
  update agent_message_queue
  set status = 'processing', processed_at = now()
  where id in (
    select id from agent_message_queue
    where to_agent = p_agent_id
      and status = 'pending'
      and expires_at > now()
    order by
      case priority
        when 'critical' then 0
        when 'high' then 1
        when 'normal' then 2
        when 'low' then 3
      end,
      created_at asc
    limit p_limit
    for update skip locked
  )
  returning *;
end;
$$;

-- Mark message as completed
create or replace function complete_message(
  p_message_id uuid
)
returns void
language plpgsql
as $$
begin
  update agent_message_queue
  set status = 'completed', completed_at = now()
  where id = p_message_id;
end;
$$;

-- Retry a failed message
create or replace function retry_message(
  p_message_id uuid,
  p_error_message text
)
returns boolean
language plpgsql
as $$
declare
  v_retry_count integer;
  v_max_retries integer;
begin
  select retry_count, max_retries into v_retry_count, v_max_retries
  from agent_message_queue
  where id = p_message_id;

  if v_retry_count >= v_max_retries then
    update agent_message_queue
    set status = 'failed', error_message = p_error_message
    where id = p_message_id;
    return false;
  end if;

  update agent_message_queue
  set status = 'pending',
      retry_count = retry_count + 1,
      processed_at = null,
      error_message = p_error_message
  where id = p_message_id;

  return true;
end;
$$;

-- Cleanup expired messages
create or replace function cleanup_expired_messages()
returns integer
language plpgsql
as $$
declare
  v_count integer;
begin
  with deleted as (
    delete from agent_message_queue
    where expires_at < now() and status = 'pending'
    returning id
  )
  select count(*) into v_count from deleted;

  return v_count;
end;
$$;

-- Get agent metrics summary
create or replace function agent_metrics_summary(
  p_agent_id text,
  p_since timestamptz default now() - interval '1 hour'
)
returns table(
  total_requests bigint,
  success_count bigint,
  failure_count bigint,
  avg_duration_ms numeric,
  p95_duration_ms numeric,
  total_input_tokens bigint,
  total_output_tokens bigint
)
language plpgsql
as $$
begin
  return query
  select
    count(*)::bigint as total_requests,
    count(*) filter (where success)::bigint as success_count,
    count(*) filter (where not success)::bigint as failure_count,
    round(avg(duration_ms)::numeric, 2) as avg_duration_ms,
    round(percentile_cont(0.95) within group (order by duration_ms)::numeric, 2) as p95_duration_ms,
    coalesce(sum(input_tokens), 0)::bigint as total_input_tokens,
    coalesce(sum(output_tokens), 0)::bigint as total_output_tokens
  from agent_metrics
  where agent_id = p_agent_id
    and recorded_at >= p_since;
end;
$$;
```
