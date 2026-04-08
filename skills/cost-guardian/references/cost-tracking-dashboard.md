# Cost Tracking Dashboard

## Daily/Monthly Cost Queries

```sql
-- Daily AI costs for the last 30 days
SELECT
  date_trunc('day', created_at)::date AS day,
  model,
  COUNT(*) AS requests,
  SUM(total_tokens) AS total_tokens,
  ROUND(SUM(estimated_cost_usd)::numeric, 4) AS total_cost_usd
FROM token_usage
WHERE created_at > now() - interval '30 days'
GROUP BY day, model
ORDER BY day DESC, model;

-- Monthly cost by feature
SELECT
  feature,
  model,
  COUNT(*) AS requests,
  SUM(total_tokens) AS total_tokens,
  ROUND(SUM(estimated_cost_usd)::numeric, 4) AS total_cost_usd
FROM token_usage
WHERE created_at > date_trunc('month', now())
GROUP BY feature, model
ORDER BY total_cost_usd DESC;

-- Top 10 most expensive users this month
SELECT
  user_id,
  COUNT(*) AS requests,
  SUM(total_tokens) AS total_tokens,
  ROUND(SUM(estimated_cost_usd)::numeric, 4) AS total_cost_usd
FROM token_usage
WHERE created_at > date_trunc('month', now())
GROUP BY user_id
ORDER BY total_cost_usd DESC
LIMIT 10;
```

## API Route for Dashboard Data

```typescript
// app/api/admin/costs/route.ts
import { NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";

export async function GET() {
  const supabase = await createClient();

  // Verify admin
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  const { data: profile } = await supabase
    .from("profiles").select("role").eq("id", user.id).single();
  if (profile?.role !== "admin") return NextResponse.json({ error: "Forbidden" }, { status: 403 });

  // Get daily costs for last 30 days
  const { data: dailyCosts } = await supabase.rpc("get_daily_costs", { days: 30 });

  // Get current month total
  const { data: monthlyTotal } = await supabase
    .from("token_usage")
    .select("estimated_cost_usd")
    .gte("created_at", new Date(new Date().getFullYear(), new Date().getMonth(), 1).toISOString());

  const totalThisMonth = (monthlyTotal ?? []).reduce(
    (sum, r) => sum + Number(r.estimated_cost_usd), 0
  );

  return NextResponse.json({
    dailyCosts,
    totalThisMonth: Math.round(totalThisMonth * 100) / 100,
    updatedAt: new Date().toISOString(),
  });
}
```

## Simple Dashboard Component

```tsx
// app/admin/costs/page.tsx
import { createClient } from "@/lib/supabase/server";

export default async function CostDashboard() {
  const supabase = await createClient();

  const { data: recentUsage } = await supabase
    .from("token_usage")
    .select("model, feature, total_tokens, estimated_cost_usd, created_at")
    .order("created_at", { ascending: false })
    .limit(50);

  const totalToday = (recentUsage ?? [])
    .filter((r) => new Date(r.created_at).toDateString() === new Date().toDateString())
    .reduce((sum, r) => sum + Number(r.estimated_cost_usd), 0);

  return (
    <div className="mx-auto max-w-4xl p-8">
      <h1 className="text-2xl font-bold">Cost Dashboard</h1>

      <div className="mt-6 grid grid-cols-3 gap-4">
        <div className="rounded-lg border p-4">
          <p className="text-sm text-gray-500">Today</p>
          <p className="text-2xl font-bold">${totalToday.toFixed(4)}</p>
        </div>
      </div>

      <h2 className="mt-8 text-lg font-semibold">Recent API Calls</h2>
      <table className="mt-4 w-full text-sm">
        <thead>
          <tr className="border-b text-left text-gray-500">
            <th className="pb-2">Time</th>
            <th className="pb-2">Model</th>
            <th className="pb-2">Feature</th>
            <th className="pb-2">Tokens</th>
            <th className="pb-2">Cost</th>
          </tr>
        </thead>
        <tbody>
          {(recentUsage ?? []).map((row) => (
            <tr key={row.created_at} className="border-b">
              <td className="py-2">{new Date(row.created_at).toLocaleTimeString()}</td>
              <td className="py-2">{row.model}</td>
              <td className="py-2">{row.feature}</td>
              <td className="py-2">{row.total_tokens.toLocaleString()}</td>
              <td className="py-2">${Number(row.estimated_cost_usd).toFixed(4)}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```
