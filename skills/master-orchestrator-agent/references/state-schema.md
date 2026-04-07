# State Schema — Supabase Tables

## Entity Relationship

```
agents (1) ──< tasks (many)
workflows (1) ──< tasks (many)
tasks (1) ──< dead_letter_queue (0..1)
```

## Table: agents

| Column | Type | Description |
|--------|------|-------------|
| id | uuid | Primary key |
| name | text (unique) | Agent identifier (e.g., 'scraper-agent') |
| description | text | What this agent does |
| capabilities | text[] | Tags like ['scraping', 'pdf', 'html'] |
| model_tier | text | 'pro', 'flash', or 'lite' |
| endpoint | text | API route (e.g., '/api/agents/scraper') |
| status | text | 'active', 'inactive', 'error' |
| last_heartbeat | timestamptz | Last ping time |
| created_at | timestamptz | Registration time |

## Table: workflows

| Column | Type | Description |
|--------|------|-------------|
| id | uuid | Primary key |
| type | text | 'sequential', 'parallel', 'conditional', 'iterative' |
| status | text | 'pending', 'running', 'completed', 'failed', 'cancelled' |
| input | jsonb | Original request + steps |
| output | jsonb | Aggregated results |
| error | text | Error message if failed |
| context | jsonb | Shared context between tasks |
| created_at | timestamptz | When workflow started |
| updated_at | timestamptz | Last status change |

## Table: tasks

| Column | Type | Description |
|--------|------|-------------|
| id | uuid | Primary key |
| workflow_id | uuid | FK to workflows |
| agent_name | text | FK to agents.name |
| status | text | 'pending', 'running', 'completed', 'failed', 'retrying' |
| input | jsonb | What was sent to the agent |
| output | jsonb | Agent's response |
| error | text | Error message |
| retry_count | int | How many times retried |
| max_retries | int | Max retry attempts (default 3) |
| token_usage | jsonb | `{ model, input, output }` token counts |
| started_at | timestamptz | When task execution began |
| completed_at | timestamptz | When task finished |
| created_at | timestamptz | When task was created |

## Table: dead_letter_queue

| Column | Type | Description |
|--------|------|-------------|
| id | uuid | Primary key |
| task_id | uuid | FK to tasks |
| workflow_id | uuid | FK to workflows |
| error | text | Final error message |
| original_input | jsonb | What was attempted |
| created_at | timestamptz | When it was dead-lettered |

## Useful Queries

```sql
-- Active workflows
select * from workflows where status = 'running' order by created_at desc;

-- Failed tasks in last 24h
select t.*, w.type as workflow_type
from tasks t
join workflows w on t.workflow_id = w.id
where t.status = 'failed'
and t.created_at > now() - interval '24 hours';

-- Agent success rates
select
  agent_name,
  count(*) as total,
  count(*) filter (where status = 'completed') as succeeded,
  round(100.0 * count(*) filter (where status = 'completed') / count(*), 1) as success_rate
from tasks
where created_at > now() - interval '7 days'
group by agent_name
order by success_rate;

-- Token usage and estimated cost by agent
select
  agent_name,
  sum((token_usage->>'input')::int) as total_input_tokens,
  sum((token_usage->>'output')::int) as total_output_tokens
from tasks
where completed_at > now() - interval '30 days'
group by agent_name;

-- Dead letter queue items needing attention
select dlq.*, t.agent_name
from dead_letter_queue dlq
join tasks t on dlq.task_id = t.id
order by dlq.created_at desc
limit 20;
```
