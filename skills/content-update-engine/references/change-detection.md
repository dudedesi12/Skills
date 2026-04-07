# Change Detection Reference

Hash-based change detection, Gemini-powered semantic diff, notification triggers, monitoring dashboard, and alert escalation patterns.

## Hash-Based Change Detection

The simplest way to detect changes: compute a hash of the content and compare it to the stored hash. If they differ, something changed.

### Core Detection Library

```tsx
// lib/change-detection.ts
import crypto from "crypto";
import { createClient } from "@/lib/supabase/server";

export function computeHash(text: string): string {
  // Normalize before hashing:
  // 1. Collapse whitespace (so formatting changes don't trigger false positives)
  // 2. Lowercase (so case changes don't trigger)
  // 3. Remove HTML tags if present
  const normalized = text
    .replace(/<[^>]*>/g, " ")   // Strip HTML
    .replace(/\s+/g, " ")       // Collapse whitespace
    .trim()
    .toLowerCase();

  return crypto.createHash("sha256").update(normalized).digest("hex");
}

export function computeStructuralHash(text: string): string {
  // A looser hash that ignores minor text changes
  // Only captures structural elements (headings, lists, links)
  const structural = text
    .replace(/<[^>]*>/g, (tag) => {
      if (/^<(h[1-6]|ul|ol|li|table|tr|td|th|a )/.test(tag)) return tag;
      return "";
    })
    .replace(/\s+/g, " ")
    .trim();

  return crypto.createHash("sha256").update(structural).digest("hex");
}

type ChangeDetectionResult = {
  hasChanged: boolean;
  changeType: "none" | "minor" | "structural" | "major";
  contentId: string;
  title: string;
};

export async function detectChanges(
  contentId: string,
  newContent: string
): Promise<ChangeDetectionResult> {
  const supabase = await createClient();

  try {
    const { data: item, error } = await supabase
      .from("content_items")
      .select("content_hash, title, body")
      .eq("id", contentId)
      .single();

    if (error) throw error;

    const newHash = computeHash(newContent);
    const oldHash = item.content_hash;

    if (newHash === oldHash) {
      return { hasChanged: false, changeType: "none", contentId, title: item.title };
    }

    // Check if the change is structural or just text
    const oldStructural = computeStructuralHash(item.body);
    const newStructural = computeStructuralHash(newContent);

    let changeType: "minor" | "structural" | "major";

    if (oldStructural !== newStructural) {
      // Structure changed — this is significant
      changeType = "structural";
    } else {
      // Only text changed — check how much
      const similarity = calculateSimilarity(item.body, newContent);
      changeType = similarity > 0.9 ? "minor" : "major";
    }

    return { hasChanged: true, changeType, contentId, title: item.title };
  } catch (error) {
    console.error("Change detection failed:", error);
    return { hasChanged: false, changeType: "none", contentId, title: "Unknown" };
  }
}

// Simple word-level similarity (Jaccard index)
function calculateSimilarity(textA: string, textB: string): number {
  const wordsA = new Set(textA.toLowerCase().split(/\s+/));
  const wordsB = new Set(textB.toLowerCase().split(/\s+/));

  let intersection = 0;
  for (const word of wordsA) {
    if (wordsB.has(word)) intersection++;
  }

  const union = wordsA.size + wordsB.size - intersection;
  return union === 0 ? 1 : intersection / union;
}
```

## Gemini-Powered Semantic Diff

Use Gemini to understand what actually changed in meaning — not just which characters are different.

```tsx
// lib/semantic-diff.ts
import { GoogleGenerativeAI } from "@google/generative-ai";

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);

export type SemanticDiffResult = {
  summary: string;
  severity: "low" | "medium" | "high" | "critical";
  keyChanges: string[];
  affectedSections: string[];
  actionRequired: string;
};

export async function getSemanticDiff(
  oldContent: string,
  newContent: string,
  contentCategory?: string
): Promise<SemanticDiffResult> {
  try {
    const model = genAI.getGenerativeModel({ model: "gemini-2.0-flash" });

    // Truncate to avoid token limits
    const maxChars = 4000;
    const oldTruncated = oldContent.substring(0, maxChars);
    const newTruncated = newContent.substring(0, maxChars);

    const categoryContext = contentCategory
      ? `This is ${contentCategory} content, so pay special attention to changes that affect compliance, deadlines, or requirements.`
      : "";

    const prompt = `You are a content change analyzer. Compare these two versions and identify meaningful changes.

${categoryContext}

OLD VERSION:
---
${oldTruncated}
---

NEW VERSION:
---
${newTruncated}
---

Respond in this exact JSON format:
{
  "summary": "One sentence describing the overall change",
  "severity": "low|medium|high|critical",
  "keyChanges": ["List of specific changes, each as one sentence"],
  "affectedSections": ["List of section names/topics affected"],
  "actionRequired": "What should be done about this change"
}

Severity definitions:
- low: Typos, formatting, minor rewording with same meaning
- medium: New information added, facts updated, clarifications
- high: Policy changes, requirement changes, process updates
- critical: Legal changes, deadline changes, removal of important information, changes that could lead to wrong actions if users see old version`;

    const result = await model.generateContent(prompt);
    const text = result.response.text();

    const jsonMatch = text.match(/\{[\s\S]*\}/);
    if (!jsonMatch) {
      throw new Error("No JSON found in Gemini response");
    }

    return JSON.parse(jsonMatch[0]) as SemanticDiffResult;
  } catch (error) {
    console.error("Semantic diff failed:", error);
    return {
      summary: "Change detected but semantic analysis unavailable",
      severity: "medium",
      keyChanges: ["Content has been modified"],
      affectedSections: ["Unknown"],
      actionRequired: "Manual review required",
    };
  }
}

export async function batchSemanticDiff(
  changes: { contentId: string; oldContent: string; newContent: string; category: string }[]
): Promise<Map<string, SemanticDiffResult>> {
  const results = new Map<string, SemanticDiffResult>();

  // Process in parallel with a concurrency limit
  const batchSize = 3;
  for (let i = 0; i < changes.length; i += batchSize) {
    const batch = changes.slice(i, i + batchSize);
    const batchResults = await Promise.allSettled(
      batch.map((change) =>
        getSemanticDiff(change.oldContent, change.newContent, change.category)
      )
    );

    batchResults.forEach((result, index) => {
      const contentId = batch[index].contentId;
      if (result.status === "fulfilled") {
        results.set(contentId, result.value);
      } else {
        results.set(contentId, {
          summary: "Analysis failed",
          severity: "medium",
          keyChanges: ["Unable to analyze"],
          affectedSections: [],
          actionRequired: "Manual review required",
        });
      }
    });
  }

  return results;
}
```

## Notification Triggers

### Change Notification System

```tsx
// lib/change-notifications.ts
import { createClient } from "@/lib/supabase/server";
import type { SemanticDiffResult } from "@/lib/semantic-diff";

type NotificationPayload = {
  userId: string;
  title: string;
  body: string;
  url: string;
  priority: "low" | "medium" | "high" | "critical";
};

// In-app notification table
// CREATE TABLE notifications (
//   id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
//   user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
//   title TEXT NOT NULL,
//   body TEXT NOT NULL,
//   url TEXT,
//   priority TEXT DEFAULT 'medium',
//   read BOOLEAN DEFAULT false,
//   created_at TIMESTAMPTZ DEFAULT now()
// );

export async function notifyContentChange(
  contentId: string,
  contentTitle: string,
  diff: SemanticDiffResult
) {
  const supabase = await createClient();

  try {
    // Get content author and admins
    const { data: content } = await supabase
      .from("content_items")
      .select("created_by, slug")
      .eq("id", contentId)
      .single();

    const { data: admins } = await supabase
      .from("user_roles")
      .select("user_id")
      .eq("role", "admin");

    const recipients = new Set<string>();

    // Author always gets notified
    if (content?.created_by) {
      recipients.add(content.created_by);
    }

    // For high/critical, also notify admins
    if (diff.severity === "high" || diff.severity === "critical") {
      admins?.forEach((a) => recipients.add(a.user_id));
    }

    // Create notifications
    const notifications = Array.from(recipients).map((userId) => ({
      user_id: userId,
      title: `Content Update: ${contentTitle}`,
      body: diff.summary,
      url: `/admin/content/${contentId}/review`,
      priority: diff.severity,
    }));

    if (notifications.length > 0) {
      const { error } = await supabase
        .from("notifications")
        .insert(notifications);

      if (error) throw error;
    }

    return { notified: recipients.size };
  } catch (error) {
    console.error("Change notification failed:", error);
    return { notified: 0 };
  }
}
```

### Notification Bell Component

```tsx
// components/notification-bell.tsx
"use client";

import { useState, useEffect } from "react";
import { createClient } from "@/lib/supabase/client";

type Notification = {
  id: string;
  title: string;
  body: string;
  url: string;
  priority: string;
  read: boolean;
  created_at: string;
};

export function NotificationBell() {
  const [notifications, setNotifications] = useState<Notification[]>([]);
  const [isOpen, setIsOpen] = useState(false);
  const [unreadCount, setUnreadCount] = useState(0);

  useEffect(() => {
    const supabase = createClient();

    async function fetchNotifications() {
      try {
        const { data, error } = await supabase
          .from("notifications")
          .select("*")
          .order("created_at", { ascending: false })
          .limit(20);

        if (error) throw error;

        setNotifications(data || []);
        setUnreadCount(data?.filter((n) => !n.read).length || 0);
      } catch (err) {
        console.error("Failed to fetch notifications:", err);
      }
    }

    fetchNotifications();

    // Real-time updates
    const channel = supabase
      .channel("notifications")
      .on(
        "postgres_changes",
        { event: "INSERT", schema: "public", table: "notifications" },
        (payload) => {
          setNotifications((prev) => [payload.new as Notification, ...prev]);
          setUnreadCount((prev) => prev + 1);
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, []);

  async function markAsRead(id: string) {
    const supabase = createClient();
    try {
      await supabase.from("notifications").update({ read: true }).eq("id", id);
      setNotifications((prev) =>
        prev.map((n) => (n.id === id ? { ...n, read: true } : n))
      );
      setUnreadCount((prev) => Math.max(0, prev - 1));
    } catch (err) {
      console.error("Failed to mark as read:", err);
    }
  }

  const priorityColors: Record<string, string> = {
    low: "bg-gray-100",
    medium: "bg-blue-100",
    high: "bg-orange-100",
    critical: "bg-red-100",
  };

  return (
    <div className="relative">
      <button
        onClick={() => setIsOpen(!isOpen)}
        className="relative p-2 rounded-lg hover:bg-gray-100"
        aria-label="Notifications"
      >
        <svg className="w-6 h-6" fill="none" viewBox="0 0 24 24" stroke="currentColor">
          <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2}
            d="M15 17h5l-1.405-1.405A2.032 2.032 0 0118 14.158V11a6.002 6.002 0 00-4-5.659V5a2 2 0 10-4 0v.341C7.67 6.165 6 8.388 6 11v3.159c0 .538-.214 1.055-.595 1.436L4 17h5m6 0v1a3 3 0 11-6 0v-1m6 0H9" />
        </svg>
        {unreadCount > 0 && (
          <span className="absolute -top-1 -right-1 w-5 h-5 bg-red-500 text-white text-xs rounded-full flex items-center justify-center">
            {unreadCount > 9 ? "9+" : unreadCount}
          </span>
        )}
      </button>

      {isOpen && (
        <div className="absolute right-0 mt-2 w-80 max-h-96 overflow-y-auto bg-white rounded-xl shadow-2xl border z-50">
          <div className="p-3 border-b">
            <h3 className="font-semibold text-sm">Notifications</h3>
          </div>
          {notifications.length === 0 ? (
            <p className="p-4 text-sm text-gray-500 text-center">No notifications</p>
          ) : (
            notifications.map((n) => (
              <a
                key={n.id}
                href={n.url}
                onClick={() => markAsRead(n.id)}
                className={`block p-3 border-b hover:bg-gray-50 ${
                  !n.read ? "bg-blue-50" : ""
                }`}
              >
                <div className="flex items-start gap-2">
                  <span className={`w-2 h-2 rounded-full mt-1.5 flex-shrink-0 ${
                    priorityColors[n.priority] || "bg-gray-100"
                  }`} />
                  <div>
                    <p className="text-sm font-medium">{n.title}</p>
                    <p className="text-xs text-gray-500 mt-0.5">{n.body}</p>
                    <p className="text-xs text-gray-400 mt-1">
                      {new Date(n.created_at).toLocaleDateString()}
                    </p>
                  </div>
                </div>
              </a>
            ))
          )}
        </div>
      )}
    </div>
  );
}
```

## Monitoring Dashboard

### Source Monitoring Cron Job

```tsx
// app/api/cron/monitor-sources/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";
import { computeHash } from "@/lib/change-detection";
import { getSemanticDiff } from "@/lib/semantic-diff";
import { notifyContentChange } from "@/lib/change-notifications";

export async function GET(request: NextRequest) {
  const authHeader = request.headers.get("authorization");
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const supabase = await createClient();

  try {
    // Get sources that are due for checking
    const { data: sources, error } = await supabase
      .from("monitored_sources")
      .select(`
        id, content_id, source_url, last_hash, check_frequency_hours,
        last_checked_at, consecutive_failures,
        content_items (id, title, body, category)
      `)
      .eq("is_active", true)
      .order("last_checked_at", { ascending: true, nullsFirst: true })
      .limit(20); // Process 20 at a time

    if (error) throw error;
    if (!sources || sources.length === 0) {
      return NextResponse.json({ checked: 0, changes: 0 });
    }

    let checked = 0;
    let changes = 0;

    for (const source of sources) {
      try {
        // Check if it is time to check this source
        if (source.last_checked_at) {
          const lastCheck = new Date(source.last_checked_at).getTime();
          const intervalMs = source.check_frequency_hours * 60 * 60 * 1000;
          if (Date.now() - lastCheck < intervalMs) continue;
        }

        // Fetch the source
        const response = await fetch(source.source_url, {
          headers: { "User-Agent": "ContentMonitor/1.0" },
          signal: AbortSignal.timeout(15000), // 15 second timeout
        });

        if (!response.ok) {
          throw new Error(`HTTP ${response.status}`);
        }

        const newContent = await response.text();
        const newHash = computeHash(newContent);
        checked++;

        // Compare hashes
        if (source.last_hash && newHash !== source.last_hash) {
          changes++;

          // Get semantic diff
          const contentItem = source.content_items as any;
          const diff = await getSemanticDiff(
            contentItem.body,
            newContent,
            contentItem.category
          );

          // Log the change
          await supabase.from("content_audit_log").insert({
            content_id: source.content_id,
            action: "source_change_detected",
            details: {
              source_url: source.source_url,
              severity: diff.severity,
              summary: diff.summary,
              key_changes: diff.keyChanges,
            },
          });

          // Flag content for review
          await supabase
            .from("content_items")
            .update({
              status: "review",
              freshness_score: diff.severity === "critical" ? 0 : 20,
            })
            .eq("id", source.content_id);

          // Notify relevant people
          await notifyContentChange(source.content_id, contentItem.title, diff);
        }

        // Update source record
        await supabase
          .from("monitored_sources")
          .update({
            last_hash: newHash,
            last_checked_at: new Date().toISOString(),
            consecutive_failures: 0,
          })
          .eq("id", source.id);
      } catch (err) {
        // Track failures
        const failures = (source.consecutive_failures || 0) + 1;

        await supabase
          .from("monitored_sources")
          .update({
            consecutive_failures: failures,
            last_checked_at: new Date().toISOString(),
            // Disable after 5 consecutive failures
            ...(failures >= 5 ? { is_active: false } : {}),
          })
          .eq("id", source.id);

        console.error(`Source check failed for ${source.source_url}:`, err);
      }
    }

    return NextResponse.json({ checked, changes });
  } catch (error) {
    console.error("Source monitoring cron failed:", error);
    return NextResponse.json({ error: "Monitoring failed" }, { status: 500 });
  }
}
```

Add to `vercel.json`:

```json
{
  "crons": [
    {
      "path": "/api/cron/monitor-sources",
      "schedule": "0 */6 * * *"
    }
  ]
}
```

This checks sources every 6 hours.

## Alert Escalation Patterns

### Escalation Rules

```tsx
// lib/alert-escalation.ts
import { createClient } from "@/lib/supabase/server";

type EscalationLevel = {
  level: number;
  name: string;
  delayHours: number;
  recipients: "author" | "team_lead" | "admin" | "all_admins";
  channel: "in_app" | "email" | "both";
};

const ESCALATION_LEVELS: EscalationLevel[] = [
  { level: 1, name: "Initial Alert", delayHours: 0, recipients: "author", channel: "in_app" },
  { level: 2, name: "Follow-up", delayHours: 24, recipients: "author", channel: "both" },
  { level: 3, name: "Team Lead", delayHours: 48, recipients: "team_lead", channel: "both" },
  { level: 4, name: "Admin Escalation", delayHours: 72, recipients: "admin", channel: "both" },
  { level: 5, name: "Critical", delayHours: 96, recipients: "all_admins", channel: "both" },
];

export async function checkEscalations() {
  const supabase = await createClient();

  try {
    // Find unresolved review items
    const { data: pendingItems, error } = await supabase
      .from("content_items")
      .select("id, title, status, updated_at, created_by")
      .eq("status", "review")
      .order("updated_at", { ascending: true });

    if (error) throw error;
    if (!pendingItems) return { escalated: 0 };

    let escalated = 0;

    for (const item of pendingItems) {
      const hoursInReview =
        (Date.now() - new Date(item.updated_at).getTime()) / (1000 * 60 * 60);

      // Find the appropriate escalation level
      const level = ESCALATION_LEVELS
        .filter((l) => hoursInReview >= l.delayHours)
        .sort((a, b) => b.level - a.level)[0];

      if (!level) continue;

      // Check if we already sent this escalation level
      const { data: existing } = await supabase
        .from("content_audit_log")
        .select("id")
        .eq("content_id", item.id)
        .eq("action", `escalation_level_${level.level}`)
        .limit(1);

      if (existing && existing.length > 0) continue;

      // Send escalation
      await supabase.from("content_audit_log").insert({
        content_id: item.id,
        action: `escalation_level_${level.level}`,
        details: {
          level: level.level,
          name: level.name,
          hours_in_review: Math.round(hoursInReview),
          recipients: level.recipients,
        },
      });

      // Create notification for the appropriate recipient(s)
      await supabase.from("notifications").insert({
        user_id: item.created_by,
        title: `[${level.name}] Content needs review: ${item.title}`,
        body: `This content has been in review for ${Math.round(hoursInReview)} hours. Please address it.`,
        url: `/admin/content/${item.id}/review`,
        priority: level.level >= 4 ? "critical" : level.level >= 3 ? "high" : "medium",
      });

      escalated++;
    }

    return { escalated };
  } catch (error) {
    console.error("Escalation check failed:", error);
    return { escalated: 0 };
  }
}
```

### Monitoring Status Page

```tsx
// app/admin/monitoring/page.tsx
import { createClient } from "@/lib/supabase/server";

export default async function MonitoringPage() {
  const supabase = await createClient();

  const { data: sources, error } = await supabase
    .from("monitored_sources")
    .select(`
      id, source_url, last_checked_at, consecutive_failures, is_active,
      content_items (title, slug)
    `)
    .order("last_checked_at", { ascending: true });

  if (error) {
    return <p className="p-8 text-red-600">Failed to load: {error.message}</p>;
  }

  return (
    <div className="max-w-5xl mx-auto p-8">
      <h1 className="text-2xl font-bold mb-6">Source Monitoring</h1>

      <div className="bg-white rounded-lg border overflow-hidden">
        <table className="w-full">
          <thead>
            <tr className="bg-gray-50 border-b">
              <th className="text-left text-sm font-medium text-gray-500 px-4 py-3">Content</th>
              <th className="text-left text-sm font-medium text-gray-500 px-4 py-3">Source</th>
              <th className="text-left text-sm font-medium text-gray-500 px-4 py-3">Last Checked</th>
              <th className="text-left text-sm font-medium text-gray-500 px-4 py-3">Status</th>
            </tr>
          </thead>
          <tbody>
            {(sources || []).map((source) => {
              const content = source.content_items as any;
              const status = !source.is_active
                ? "disabled"
                : source.consecutive_failures > 0
                ? "failing"
                : "healthy";

              const statusColors: Record<string, string> = {
                healthy: "bg-green-100 text-green-800",
                failing: "bg-orange-100 text-orange-800",
                disabled: "bg-red-100 text-red-800",
              };

              return (
                <tr key={source.id} className="border-b">
                  <td className="px-4 py-3 text-sm font-medium">{content?.title}</td>
                  <td className="px-4 py-3 text-sm text-gray-500 truncate max-w-xs">
                    {source.source_url}
                  </td>
                  <td className="px-4 py-3 text-sm text-gray-500">
                    {source.last_checked_at
                      ? new Date(source.last_checked_at).toLocaleString()
                      : "Never"}
                  </td>
                  <td className="px-4 py-3">
                    <span className={`inline-flex px-2 py-1 text-xs font-medium rounded-full ${statusColors[status]}`}>
                      {status}
                      {source.consecutive_failures > 0 && ` (${source.consecutive_failures} failures)`}
                    </span>
                  </td>
                </tr>
              );
            })}
          </tbody>
        </table>
      </div>
    </div>
  );
}
```
