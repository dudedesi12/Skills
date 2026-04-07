# Dashboard Schema — Issue Tracking

## Core Tables

See the main SKILL.md for the full migration. Key tables:

- `health_checks` — One row per scan (score, timestamp, category breakdown)
- `issues` — Deduplicated issues with fingerprints, severity, status
- `client_errors` — Raw error reports from the browser

## Dashboard Queries

### Health Score Trend (Last 30 Days)

```sql
select
  date_trunc('day', checked_at) as day,
  round(avg(overall_score)) as avg_score,
  sum(issues_found) as total_issues
from health_checks
where checked_at > now() - interval '30 days'
group by day
order by day;
```

### Issues by Category and Severity

```sql
select
  category,
  count(*) filter (where severity = 'critical') as critical,
  count(*) filter (where severity = 'high') as high,
  count(*) filter (where severity = 'medium') as medium,
  count(*) filter (where severity = 'low') as low
from issues
where status in ('open', 'acknowledged')
group by category
order by critical desc, high desc;
```

### Top Recurring Issues

```sql
select title, category, severity, occurrences, first_seen, last_seen
from issues
where status = 'open'
order by occurrences desc
limit 10;
```

### Fix Rate (Issues Resolved vs Opened)

```sql
select
  date_trunc('week', first_seen) as week,
  count(*) filter (where status = 'open') as opened,
  count(*) filter (where status = 'resolved') as resolved
from issues
where first_seen > now() - interval '90 days'
group by week
order by week;
```

### Client Error Hotspots (Most Erroring Pages)

```sql
select
  url,
  count(*) as error_count,
  count(distinct message) as unique_errors,
  max(created_at) as last_error
from client_errors
where created_at > now() - interval '7 days'
group by url
order by error_count desc
limit 10;
```

## Dashboard Component (Recharts)

```typescript
// file: components/admin/health-trend-chart.tsx
'use client';

import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } from 'recharts';

interface DataPoint {
  day: string;
  avg_score: number;
  total_issues: number;
}

export function HealthTrendChart({ data }: { data: DataPoint[] }) {
  return (
    <ResponsiveContainer width="100%" height={300}>
      <LineChart data={data}>
        <CartesianGrid strokeDasharray="3 3" />
        <XAxis dataKey="day" tickFormatter={(d) => new Date(d).toLocaleDateString('en-AU', { month: 'short', day: 'numeric' })} />
        <YAxis domain={[0, 100]} />
        <Tooltip />
        <Line type="monotone" dataKey="avg_score" stroke="#22c55e" strokeWidth={2} name="Health Score" />
      </LineChart>
    </ResponsiveContainer>
  );
}
```
