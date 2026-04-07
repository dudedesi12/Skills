# Dashboard Components Reference

10+ Recharts dashboard components ready to use: LineChart, BarChart, PieChart, AreaChart, KPI Card, Metric Table, Funnel Chart, Date Range Picker, Real-time Counter, and Sparkline.

## Installation

```bash
npm install recharts
```

## 1. Line Chart

```tsx
// components/dashboard/line-chart.tsx
"use client";

import {
  LineChart,
  Line,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  Legend,
  ResponsiveContainer,
} from "recharts";

type LineChartProps = {
  data: { date: string; [key: string]: string | number }[];
  lines: { dataKey: string; color: string; label?: string }[];
  title: string;
  height?: number;
};

export function AnalyticsLineChart({ data, lines, title, height = 300 }: LineChartProps) {
  if (!data || data.length === 0) {
    return (
      <div className="bg-white rounded-lg border p-6">
        <h3 className="text-lg font-semibold mb-4">{title}</h3>
        <p className="text-gray-500 text-center py-8">No data available</p>
      </div>
    );
  }

  return (
    <div className="bg-white rounded-lg border p-6">
      <h3 className="text-lg font-semibold mb-4">{title}</h3>
      <ResponsiveContainer width="100%" height={height}>
        <LineChart data={data}>
          <CartesianGrid strokeDasharray="3 3" stroke="#f0f0f0" />
          <XAxis dataKey="date" fontSize={12} tickMargin={8} />
          <YAxis fontSize={12} tickMargin={8} />
          <Tooltip
            contentStyle={{
              borderRadius: "8px",
              border: "1px solid #e5e7eb",
              boxShadow: "0 2px 8px rgba(0,0,0,0.08)",
            }}
          />
          <Legend />
          {lines.map((line) => (
            <Line
              key={line.dataKey}
              type="monotone"
              dataKey={line.dataKey}
              stroke={line.color}
              strokeWidth={2}
              dot={false}
              name={line.label || line.dataKey}
            />
          ))}
        </LineChart>
      </ResponsiveContainer>
    </div>
  );
}
```

## 2. Bar Chart

```tsx
// components/dashboard/bar-chart.tsx
"use client";

import {
  BarChart,
  Bar,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  ResponsiveContainer,
} from "recharts";

type BarChartProps = {
  data: { label: string; value: number }[];
  title: string;
  color?: string;
  height?: number;
};

export function AnalyticsBarChart({
  data,
  title,
  color = "#3b82f6",
  height = 300,
}: BarChartProps) {
  if (!data || data.length === 0) {
    return (
      <div className="bg-white rounded-lg border p-6">
        <h3 className="text-lg font-semibold mb-4">{title}</h3>
        <p className="text-gray-500 text-center py-8">No data available</p>
      </div>
    );
  }

  return (
    <div className="bg-white rounded-lg border p-6">
      <h3 className="text-lg font-semibold mb-4">{title}</h3>
      <ResponsiveContainer width="100%" height={height}>
        <BarChart data={data}>
          <CartesianGrid strokeDasharray="3 3" stroke="#f0f0f0" />
          <XAxis dataKey="label" fontSize={12} />
          <YAxis fontSize={12} />
          <Tooltip />
          <Bar dataKey="value" fill={color} radius={[4, 4, 0, 0]} />
        </BarChart>
      </ResponsiveContainer>
    </div>
  );
}
```

## 3. Pie Chart

```tsx
// components/dashboard/pie-chart.tsx
"use client";

import { PieChart, Pie, Cell, Tooltip, ResponsiveContainer, Legend } from "recharts";

type PieChartProps = {
  data: { name: string; value: number }[];
  title: string;
  colors?: string[];
};

const DEFAULT_COLORS = ["#3b82f6", "#10b981", "#f59e0b", "#ef4444", "#8b5cf6", "#ec4899"];

export function AnalyticsPieChart({ data, title, colors = DEFAULT_COLORS }: PieChartProps) {
  if (!data || data.length === 0) {
    return (
      <div className="bg-white rounded-lg border p-6">
        <h3 className="text-lg font-semibold mb-4">{title}</h3>
        <p className="text-gray-500 text-center py-8">No data available</p>
      </div>
    );
  }

  return (
    <div className="bg-white rounded-lg border p-6">
      <h3 className="text-lg font-semibold mb-4">{title}</h3>
      <ResponsiveContainer width="100%" height={300}>
        <PieChart>
          <Pie
            data={data}
            cx="50%"
            cy="50%"
            innerRadius={60}
            outerRadius={100}
            paddingAngle={2}
            dataKey="value"
          >
            {data.map((_, index) => (
              <Cell key={`cell-${index}`} fill={colors[index % colors.length]} />
            ))}
          </Pie>
          <Tooltip />
          <Legend />
        </PieChart>
      </ResponsiveContainer>
    </div>
  );
}
```

## 4. Area Chart

```tsx
// components/dashboard/area-chart.tsx
"use client";

import {
  AreaChart,
  Area,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  ResponsiveContainer,
} from "recharts";

type AreaChartProps = {
  data: { date: string; value: number }[];
  title: string;
  color?: string;
  height?: number;
};

export function AnalyticsAreaChart({
  data,
  title,
  color = "#3b82f6",
  height = 300,
}: AreaChartProps) {
  if (!data || data.length === 0) {
    return (
      <div className="bg-white rounded-lg border p-6">
        <h3 className="text-lg font-semibold mb-4">{title}</h3>
        <p className="text-gray-500 text-center py-8">No data available</p>
      </div>
    );
  }

  return (
    <div className="bg-white rounded-lg border p-6">
      <h3 className="text-lg font-semibold mb-4">{title}</h3>
      <ResponsiveContainer width="100%" height={height}>
        <AreaChart data={data}>
          <CartesianGrid strokeDasharray="3 3" stroke="#f0f0f0" />
          <XAxis dataKey="date" fontSize={12} />
          <YAxis fontSize={12} />
          <Tooltip />
          <Area
            type="monotone"
            dataKey="value"
            stroke={color}
            fill={color}
            fillOpacity={0.1}
            strokeWidth={2}
          />
        </AreaChart>
      </ResponsiveContainer>
    </div>
  );
}
```

## 5. KPI Card

```tsx
// components/dashboard/kpi-card.tsx
type KPICardProps = {
  title: string;
  value: string | number;
  change?: number;
  changeLabel?: string;
  icon?: React.ReactNode;
  format?: "number" | "currency" | "percentage";
};

export function KPICard({ title, value, change, changeLabel, icon, format }: KPICardProps) {
  const formattedValue = formatValue(value, format);
  const isPositive = change !== undefined && change >= 0;

  return (
    <div className="bg-white rounded-lg border p-6 hover:shadow-sm transition-shadow">
      <div className="flex items-center justify-between">
        <p className="text-sm font-medium text-gray-500">{title}</p>
        {icon && <div className="text-gray-400">{icon}</div>}
      </div>
      <p className="text-3xl font-bold mt-2">{formattedValue}</p>
      {change !== undefined && (
        <div className="flex items-center gap-1 mt-2">
          <span
            className={`text-sm font-medium ${
              isPositive ? "text-green-600" : "text-red-600"
            }`}
          >
            {isPositive ? "+" : ""}{change.toFixed(1)}%
          </span>
          <span className="text-sm text-gray-500">
            {changeLabel || "vs last period"}
          </span>
        </div>
      )}
    </div>
  );
}

function formatValue(
  value: string | number,
  format?: "number" | "currency" | "percentage"
): string {
  if (typeof value === "string") return value;

  switch (format) {
    case "currency":
      return new Intl.NumberFormat("en-US", {
        style: "currency",
        currency: "USD",
        minimumFractionDigits: 0,
      }).format(value);
    case "percentage":
      return `${value.toFixed(1)}%`;
    case "number":
    default:
      return value.toLocaleString();
  }
}
```

## 6. Metric Table

```tsx
// components/dashboard/metric-table.tsx
type MetricRow = {
  label: string;
  value: number;
  change?: number;
};

type MetricTableProps = {
  title: string;
  rows: MetricRow[];
  valueLabel?: string;
};

export function MetricTable({ title, rows, valueLabel = "Value" }: MetricTableProps) {
  if (!rows || rows.length === 0) {
    return (
      <div className="bg-white rounded-lg border p-6">
        <h3 className="text-lg font-semibold mb-4">{title}</h3>
        <p className="text-gray-500 text-center py-4">No data available</p>
      </div>
    );
  }

  return (
    <div className="bg-white rounded-lg border p-6">
      <h3 className="text-lg font-semibold mb-4">{title}</h3>
      <table className="w-full">
        <thead>
          <tr className="border-b">
            <th className="text-left text-sm text-gray-500 pb-2">Name</th>
            <th className="text-right text-sm text-gray-500 pb-2">{valueLabel}</th>
            <th className="text-right text-sm text-gray-500 pb-2">Change</th>
          </tr>
        </thead>
        <tbody>
          {rows.map((row, i) => (
            <tr key={i} className="border-b last:border-0">
              <td className="py-3 text-sm">{row.label}</td>
              <td className="py-3 text-sm text-right font-medium">
                {row.value.toLocaleString()}
              </td>
              <td className="py-3 text-sm text-right">
                {row.change !== undefined && (
                  <span
                    className={row.change >= 0 ? "text-green-600" : "text-red-600"}
                  >
                    {row.change >= 0 ? "+" : ""}{row.change.toFixed(1)}%
                  </span>
                )}
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

## 7. Funnel Chart

```tsx
// components/dashboard/funnel-chart.tsx
type FunnelStep = {
  name: string;
  value: number;
};

type FunnelChartProps = {
  steps: FunnelStep[];
  title: string;
};

export function FunnelChart({ steps, title }: FunnelChartProps) {
  if (!steps || steps.length === 0) {
    return (
      <div className="bg-white rounded-lg border p-6">
        <h3 className="text-lg font-semibold mb-4">{title}</h3>
        <p className="text-gray-500 text-center py-4">No data available</p>
      </div>
    );
  }

  const maxValue = steps[0]?.value || 1;

  return (
    <div className="bg-white rounded-lg border p-6">
      <h3 className="text-lg font-semibold mb-6">{title}</h3>
      <div className="space-y-3">
        {steps.map((step, i) => {
          const widthPercent = (step.value / maxValue) * 100;
          const dropoff =
            i > 0 ? ((steps[i - 1].value - step.value) / steps[i - 1].value) * 100 : 0;

          return (
            <div key={i}>
              <div className="flex justify-between text-sm mb-1">
                <span className="font-medium">{step.name}</span>
                <span className="text-gray-500">
                  {step.value.toLocaleString()}
                  {i > 0 && (
                    <span className="text-red-500 ml-2">-{dropoff.toFixed(1)}%</span>
                  )}
                </span>
              </div>
              <div className="w-full bg-gray-100 rounded-full h-8">
                <div
                  className="bg-blue-500 rounded-full h-8 transition-all duration-500"
                  style={{ width: `${widthPercent}%` }}
                />
              </div>
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

## 8. Date Range Picker

```tsx
// components/dashboard/date-range-picker.tsx
"use client";

import { useState } from "react";

export type DateRange = { from: string; to: string };
type Preset = "today" | "7d" | "30d" | "90d" | "365d" | "custom";

const PRESETS: { value: Preset; label: string; days: number }[] = [
  { value: "today", label: "Today", days: 0 },
  { value: "7d", label: "7 Days", days: 7 },
  { value: "30d", label: "30 Days", days: 30 },
  { value: "90d", label: "90 Days", days: 90 },
  { value: "365d", label: "1 Year", days: 365 },
];

export function DateRangePicker({
  onChange,
  defaultPreset = "30d",
}: {
  onChange: (range: DateRange) => void;
  defaultPreset?: Preset;
}) {
  const [activePreset, setActivePreset] = useState<Preset>(defaultPreset);
  const [customFrom, setCustomFrom] = useState("");
  const [customTo, setCustomTo] = useState("");

  function handlePreset(preset: Preset, days: number) {
    setActivePreset(preset);

    if (preset === "custom") return;

    const to = new Date().toISOString().split("T")[0];
    const from =
      days === 0
        ? to
        : new Date(Date.now() - days * 86400000).toISOString().split("T")[0];

    onChange({ from, to });
  }

  function handleCustomApply() {
    if (customFrom && customTo) {
      setActivePreset("custom");
      onChange({ from: customFrom, to: customTo });
    }
  }

  return (
    <div className="flex flex-wrap items-center gap-2">
      {PRESETS.map((preset) => (
        <button
          key={preset.value}
          onClick={() => handlePreset(preset.value, preset.days)}
          className={`px-3 py-1.5 text-sm rounded-md transition-colors ${
            activePreset === preset.value
              ? "bg-blue-600 text-white"
              : "bg-gray-100 text-gray-700 hover:bg-gray-200"
          }`}
        >
          {preset.label}
        </button>
      ))}
      <div className="flex items-center gap-1 ml-2">
        <input
          type="date"
          value={customFrom}
          onChange={(e) => setCustomFrom(e.target.value)}
          className="px-2 py-1 text-sm border rounded"
        />
        <span className="text-gray-400">to</span>
        <input
          type="date"
          value={customTo}
          onChange={(e) => setCustomTo(e.target.value)}
          className="px-2 py-1 text-sm border rounded"
        />
        <button
          onClick={handleCustomApply}
          className="px-3 py-1 text-sm bg-gray-800 text-white rounded hover:bg-gray-900"
        >
          Apply
        </button>
      </div>
    </div>
  );
}
```

## 9. Real-time Counter

```tsx
// components/dashboard/realtime-counter.tsx
"use client";

import { useEffect, useState } from "react";
import { createClient } from "@/lib/supabase/client";

type RealtimeCounterProps = {
  eventName: string;
  label?: string;
  since?: string; // ISO date string for "since" filter
};

export function RealtimeCounter({ eventName, label, since }: RealtimeCounterProps) {
  const [count, setCount] = useState(0);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const supabase = createClient();

    async function fetchInitialCount() {
      try {
        let query = supabase
          .from("analytics_events")
          .select("*", { count: "exact", head: true })
          .eq("event_name", eventName);

        if (since) {
          query = query.gte("created_at", since);
        }

        const { count: total, error } = await query;
        if (error) throw error;
        setCount(total || 0);
      } catch (err) {
        console.error("Failed to fetch real-time count:", err);
      } finally {
        setLoading(false);
      }
    }

    fetchInitialCount();

    const channel = supabase
      .channel(`realtime-${eventName}`)
      .on(
        "postgres_changes",
        {
          event: "INSERT",
          schema: "public",
          table: "analytics_events",
          filter: `event_name=eq.${eventName}`,
        },
        () => {
          setCount((prev) => prev + 1);
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, [eventName, since]);

  return (
    <div className="bg-white rounded-lg border p-6 text-center">
      <p className="text-sm text-gray-500">{label || eventName}</p>
      {loading ? (
        <div className="h-10 mt-2 bg-gray-100 animate-pulse rounded" />
      ) : (
        <p className="text-4xl font-bold mt-2">{count.toLocaleString()}</p>
      )}
      <div className="flex items-center justify-center gap-1.5 mt-3">
        <span className="w-2 h-2 bg-green-500 rounded-full animate-pulse" />
        <span className="text-xs text-green-600 font-medium">Live</span>
      </div>
    </div>
  );
}
```

## 10. Sparkline

A tiny inline chart for use inside table rows or KPI cards.

```tsx
// components/dashboard/sparkline.tsx
"use client";

import { LineChart, Line, ResponsiveContainer } from "recharts";

type SparklineProps = {
  data: number[];
  color?: string;
  width?: number;
  height?: number;
};

export function Sparkline({
  data,
  color = "#3b82f6",
  width = 120,
  height = 32,
}: SparklineProps) {
  const chartData = data.map((value, index) => ({ index, value }));

  return (
    <div style={{ width, height }}>
      <ResponsiveContainer width="100%" height="100%">
        <LineChart data={chartData}>
          <Line
            type="monotone"
            dataKey="value"
            stroke={color}
            strokeWidth={1.5}
            dot={false}
          />
        </LineChart>
      </ResponsiveContainer>
    </div>
  );
}
```

## 11. Complete Dashboard Layout

Bringing all components together in a full analytics dashboard page.

```tsx
// app/dashboard/analytics/page.tsx
import { createClient } from "@/lib/supabase/server";
import { AnalyticsLineChart } from "@/components/dashboard/line-chart";
import { AnalyticsBarChart } from "@/components/dashboard/bar-chart";
import { AnalyticsPieChart } from "@/components/dashboard/pie-chart";
import { KPICard } from "@/components/dashboard/kpi-card";
import { MetricTable } from "@/components/dashboard/metric-table";
import { FunnelChart } from "@/components/dashboard/funnel-chart";
import { RealtimeCounter } from "@/components/dashboard/realtime-counter";
import { DashboardClient } from "./dashboard-client";

export default async function AnalyticsDashboard() {
  const supabase = await createClient();

  const thirtyDaysAgo = new Date(Date.now() - 30 * 86400000).toISOString();

  const { data: events, error } = await supabase
    .from("analytics_events")
    .select("event_name, properties, created_at")
    .gte("created_at", thirtyDaysAgo)
    .order("created_at", { ascending: true });

  if (error) {
    return (
      <div className="p-8">
        <p className="text-red-600">Failed to load analytics: {error.message}</p>
      </div>
    );
  }

  const safeEvents = events || [];

  // Calculate KPIs
  const pageViews = safeEvents.filter((e) => e.event_name === "page_view").length;
  const signups = safeEvents.filter((e) => e.event_name === "signup_completed").length;
  const purchases = safeEvents.filter((e) => e.event_name === "purchase_completed").length;
  const conversionRate = pageViews > 0 ? ((signups / pageViews) * 100) : 0;

  // Group page views by date for line chart
  const pvByDate = groupByDate(
    safeEvents.filter((e) => e.event_name === "page_view")
  );

  // Top pages for bar chart
  const topPages = getTopValues(
    safeEvents.filter((e) => e.event_name === "page_view"),
    "page"
  );

  // Traffic sources for pie chart
  const sources = getTopValues(safeEvents, "source");

  // Funnel data
  const funnelSteps = [
    { name: "Page View", value: pageViews },
    { name: "Signup Started", value: Math.round(pageViews * 0.3) },
    { name: "Signup Completed", value: signups },
    { name: "First Purchase", value: purchases },
  ];

  return (
    <div className="p-8 space-y-8 max-w-7xl mx-auto">
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-bold">Analytics Dashboard</h1>
        <DashboardClient />
      </div>

      {/* KPI Row */}
      <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4">
        <KPICard title="Page Views" value={pageViews} format="number" change={12.5} />
        <KPICard title="Signups" value={signups} format="number" change={8.3} />
        <KPICard title="Purchases" value={purchases} format="number" change={-2.1} />
        <KPICard title="Conversion" value={conversionRate} format="percentage" change={3.2} />
      </div>

      {/* Real-time counters */}
      <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
        <RealtimeCounter eventName="page_view" label="Live Page Views (Today)" />
        <RealtimeCounter eventName="signup_completed" label="Live Signups (Today)" />
      </div>

      {/* Charts Row */}
      <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
        <AnalyticsLineChart
          data={pvByDate.map((d) => ({ ...d, views: d.value }))}
          lines={[{ dataKey: "value", color: "#3b82f6", label: "Page Views" }]}
          title="Page Views Over Time"
        />
        <AnalyticsBarChart data={topPages} title="Top Pages" />
      </div>

      <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
        <AnalyticsPieChart
          data={sources.map((s) => ({ name: s.label, value: s.value }))}
          title="Traffic Sources"
        />
        <FunnelChart steps={funnelSteps} title="Conversion Funnel" />
      </div>

      <MetricTable
        title="Event Breakdown"
        rows={getEventCounts(safeEvents)}
        valueLabel="Count"
      />
    </div>
  );
}

// Helper functions

function groupByDate(events: { created_at: string }[]) {
  const grouped: Record<string, number> = {};
  for (const event of events) {
    const date = event.created_at.split("T")[0];
    grouped[date] = (grouped[date] || 0) + 1;
  }
  return Object.entries(grouped).map(([date, value]) => ({ date, value }));
}

function getTopValues(
  events: { properties: Record<string, unknown> }[],
  key: string
): { label: string; value: number }[] {
  const counts: Record<string, number> = {};
  for (const event of events) {
    const val = String(event.properties?.[key] || "unknown");
    counts[val] = (counts[val] || 0) + 1;
  }
  return Object.entries(counts)
    .sort((a, b) => b[1] - a[1])
    .slice(0, 10)
    .map(([label, value]) => ({ label, value }));
}

function getEventCounts(
  events: { event_name: string }[]
): { label: string; value: number }[] {
  const counts: Record<string, number> = {};
  for (const event of events) {
    counts[event.event_name] = (counts[event.event_name] || 0) + 1;
  }
  return Object.entries(counts)
    .sort((a, b) => b[1] - a[1])
    .map(([label, value]) => ({ label, value }));
}
```

### Client-Side Dashboard Controls

```tsx
// app/dashboard/analytics/dashboard-client.tsx
"use client";

import { DateRangePicker, DateRange } from "@/components/dashboard/date-range-picker";
import { useRouter, useSearchParams } from "next/navigation";

export function DashboardClient() {
  const router = useRouter();
  const searchParams = useSearchParams();

  function handleDateChange(range: DateRange) {
    const params = new URLSearchParams(searchParams.toString());
    params.set("from", range.from);
    params.set("to", range.to);
    router.push(`?${params.toString()}`);
  }

  return <DateRangePicker onChange={handleDateChange} />;
}
```
