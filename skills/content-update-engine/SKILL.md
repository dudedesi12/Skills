---
name: content-update-engine
description: "Use this skill whenever the user mentions content freshness, content updates, regulatory changes, immigration updates, data staleness, 'is this still current', content versioning, audit trail, review queue, change detection, freshness scoring, 'check for changes', 'monitor updates', content calendar, scheduled updates, notifications for changes, 'data is outdated', compliance updates, or ANY content maintenance/freshness task — even if they don't explicitly say 'content update'. This skill keeps your content accurate and current automatically."
---

# Content Update Engine

Keep your content accurate and current with versioning, freshness scoring, change detection, review queues, and automated monitoring.

## Content Versioning System

```sql
-- supabase/migrations/create_content_versioning.sql
CREATE TABLE content_items (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  slug TEXT NOT NULL UNIQUE,
  title TEXT NOT NULL,
  body TEXT NOT NULL,
  category TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'published', 'archived', 'review')),
  source_url TEXT,
  source_reliability TEXT DEFAULT 'medium' CHECK (source_reliability IN ('low', 'medium', 'high', 'official')),
  content_hash TEXT NOT NULL,
  freshness_score FLOAT DEFAULT 100,
  last_verified_at TIMESTAMPTZ DEFAULT now(),
  published_at TIMESTAMPTZ,
  created_by UUID REFERENCES auth.users(id),
  updated_by UUID REFERENCES auth.users(id),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE content_versions (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  content_id UUID REFERENCES content_items(id) ON DELETE CASCADE,
  version_number INTEGER NOT NULL,
  title TEXT NOT NULL,
  body TEXT NOT NULL,
  changed_by UUID REFERENCES auth.users(id),
  change_reason TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE content_audit_log (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  content_id UUID REFERENCES content_items(id) ON DELETE CASCADE,
  action TEXT NOT NULL,
  actor_id UUID REFERENCES auth.users(id),
  details JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_content_freshness ON content_items(freshness_score);
CREATE INDEX idx_versions_content ON content_versions(content_id, version_number DESC);
CREATE INDEX idx_audit_content ON content_audit_log(content_id, created_at DESC);

ALTER TABLE content_items ENABLE ROW LEVEL SECURITY;
ALTER TABLE content_versions ENABLE ROW LEVEL SECURITY;
ALTER TABLE content_audit_log ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Public read published" ON content_items FOR SELECT USING (status = 'published');
CREATE POLICY "Admins full access" ON content_items FOR ALL
  USING (EXISTS (SELECT 1 FROM user_roles WHERE user_id = auth.uid() AND role = 'admin'));
```

### Version Management

```tsx
// lib/content-versioning.ts
import { createClient } from "@/lib/supabase/server";
import crypto from "crypto";

export function hashContent(content: string): string {
  return crypto.createHash("sha256").update(content).digest("hex");
}

export async function updateContent(
  contentId: string, updates: { title: string; body: string },
  userId: string, changeReason: string
) {
  const supabase = await createClient();
  try {
    const { data: current, error: fetchError } = await supabase
      .from("content_items").select("title, body").eq("id", contentId).single();
    if (fetchError) throw fetchError;

    const { count } = await supabase.from("content_versions")
      .select("*", { count: "exact", head: true }).eq("content_id", contentId);

    await supabase.from("content_versions").insert({
      content_id: contentId, version_number: (count || 0) + 1,
      title: current.title, body: current.body, changed_by: userId, change_reason: changeReason,
    });

    await supabase.from("content_items").update({
      ...updates, content_hash: hashContent(updates.body), updated_by: userId,
      updated_at: new Date().toISOString(), last_verified_at: new Date().toISOString(), freshness_score: 100,
    }).eq("id", contentId);

    await supabase.from("content_audit_log").insert({
      content_id: contentId, action: "content_updated", actor_id: userId,
      details: { version: (count || 0) + 1, reason: changeReason },
    });
    return { success: true };
  } catch (error) {
    console.error("Content update failed:", error);
    return { success: false, error };
  }
}
```

## Freshness Scoring Algorithm

Content gets a score from 0-100 that decays over time based on category and source reliability.

```tsx
// lib/freshness-scoring.ts
const CATEGORY_CONFIG: Record<string, { maxAgeDays: number; decayRate: number; sensitivityMultiplier: number }> = {
  regulatory: { maxAgeDays: 30, decayRate: 3.0, sensitivityMultiplier: 2.0 },
  legal: { maxAgeDays: 60, decayRate: 2.0, sensitivityMultiplier: 1.5 },
  general: { maxAgeDays: 180, decayRate: 1.0, sensitivityMultiplier: 1.0 },
  evergreen: { maxAgeDays: 365, decayRate: 0.5, sensitivityMultiplier: 0.5 },
};

const SOURCE_WEIGHTS: Record<string, number> = { official: 1.0, high: 0.8, medium: 0.6, low: 0.4 };

export function calculateFreshnessScore(lastVerifiedAt: Date, category: string, sourceReliability: string): number {
  const config = CATEGORY_CONFIG[category] || CATEGORY_CONFIG.general;
  const sourceWeight = SOURCE_WEIGHTS[sourceReliability] || 0.5;
  const ageInDays = (Date.now() - lastVerifiedAt.getTime()) / (1000 * 60 * 60 * 24);
  const rawScore = 100 * Math.exp((-config.decayRate * config.sensitivityMultiplier * ageInDays) / config.maxAgeDays);
  return Math.max(0, Math.min(100, Math.round(rawScore * (0.5 + 0.5 * sourceWeight))));
}

export function getFreshnessLevel(score: number) {
  if (score >= 80) return { label: "Fresh", color: "green", urgent: false };
  if (score >= 60) return { label: "Aging", color: "yellow", urgent: false };
  if (score >= 40) return { label: "Stale", color: "orange", urgent: true };
  return { label: "Outdated", color: "red", urgent: true };
}
```

## Change Detection

```tsx
// lib/change-detection.ts
import crypto from "crypto";
import { createClient } from "@/lib/supabase/server";

export function computeHash(text: string): string {
  return crypto.createHash("sha256").update(text.replace(/\s+/g, " ").trim().toLowerCase()).digest("hex");
}

export async function checkForChanges(contentId: string, newContent: string) {
  const supabase = await createClient();
  try {
    const { data: existing, error } = await supabase
      .from("content_items").select("content_hash, title").eq("id", contentId).single();
    if (error) throw error;

    const hasChanged = existing.content_hash !== computeHash(newContent);
    if (hasChanged) {
      await supabase.from("content_audit_log").insert({ content_id: contentId, action: "change_detected" });
      await supabase.from("content_items").update({ status: "review", freshness_score: 20 }).eq("id", contentId);
    }
    return { hasChanged, title: existing.title };
  } catch (error) {
    console.error("Change detection failed:", error);
    return { hasChanged: false, error };
  }
}
```

### Gemini-Powered Semantic Diff

```tsx
// lib/semantic-diff.ts
import { GoogleGenerativeAI } from "@google/generative-ai";

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);

export async function getSemanticDiff(oldContent: string, newContent: string) {
  try {
    const model = genAI.getGenerativeModel({ model: "gemini-2.0-flash" });
    const prompt = `Compare these content versions. Respond in JSON: {"summary":"...","severity":"low|medium|high|critical","keyChanges":["..."]}\n\nOLD:\n${oldContent.substring(0, 3000)}\n\nNEW:\n${newContent.substring(0, 3000)}`;
    const result = await model.generateContent(prompt);
    const match = result.response.text().match(/\{[\s\S]*\}/);
    return match ? JSON.parse(match[0]) : { summary: "Analysis unavailable", severity: "medium", keyChanges: [] };
  } catch (error) {
    console.error("Semantic diff failed:", error);
    return { summary: "Analysis failed", severity: "medium", keyChanges: ["Content modified"] };
  }
}
```

## Freshness Badge Component

```tsx
// components/freshness-badge.tsx
import { getFreshnessLevel } from "@/lib/freshness-scoring";

export function FreshnessBadge({ score, lastVerifiedAt }: { score: number; lastVerifiedAt: string }) {
  const level = getFreshnessLevel(score);
  const date = new Date(lastVerifiedAt).toLocaleDateString("en-US", { month: "short", day: "numeric", year: "numeric" });
  const colors: Record<string, string> = {
    green: "bg-green-100 text-green-800", yellow: "bg-yellow-100 text-yellow-800",
    orange: "bg-orange-100 text-orange-800", red: "bg-red-100 text-red-800",
  };
  return (
    <span className={`inline-flex items-center gap-1.5 px-2.5 py-1 rounded-full text-xs font-medium ${colors[level.color]}`}>
      {level.label} — Verified {date}
    </span>
  );
}
```

## Review Queue

```tsx
// app/admin/review-queue/page.tsx
import { createClient } from "@/lib/supabase/server";
import { FreshnessBadge } from "@/components/freshness-badge";

export default async function ReviewQueuePage() {
  const supabase = await createClient();
  const { data: items, error } = await supabase
    .from("content_items")
    .select("id, title, slug, category, freshness_score, last_verified_at, status")
    .or("status.eq.review,freshness_score.lt.40")
    .order("freshness_score", { ascending: true });

  if (error) return <p className="p-8 text-red-600">Failed to load: {error.message}</p>;

  return (
    <div className="max-w-5xl mx-auto p-8">
      <h1 className="text-2xl font-bold mb-6">Content Review Queue</h1>
      <div className="space-y-3">
        {(items || []).map((item) => (
          <div key={item.id} className="bg-white rounded-lg border p-4">
            <div className="flex items-start justify-between gap-4">
              <div>
                <h3 className="font-semibold">{item.title}</h3>
                <p className="text-sm text-gray-500">/{item.slug} — {item.category}</p>
              </div>
              <FreshnessBadge score={item.freshness_score} lastVerifiedAt={item.last_verified_at} />
            </div>
            <div className="flex gap-2 mt-4">
              <a href={`/admin/content/${item.id}/edit`}
                className="px-3 py-1.5 text-sm bg-blue-600 text-white rounded hover:bg-blue-700">Review</a>
            </div>
          </div>
        ))}
      </div>
    </div>
  );
}
```

## Automated Freshness Checks with Vercel Cron

```ts
// app/api/cron/freshness-check/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";
import { calculateFreshnessScore } from "@/lib/freshness-scoring";

export async function GET(request: NextRequest) {
  if (request.headers.get("authorization") !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }
  const supabase = await createClient();
  try {
    const { data: items, error } = await supabase.from("content_items")
      .select("id, category, source_reliability, last_verified_at, freshness_score")
      .eq("status", "published");
    if (error) throw error;

    let updated = 0, flagged = 0;
    for (const item of items || []) {
      const newScore = calculateFreshnessScore(new Date(item.last_verified_at), item.category, item.source_reliability);
      if (newScore !== item.freshness_score) {
        await supabase.from("content_items").update({
          freshness_score: newScore,
          ...(newScore < 40 && item.freshness_score >= 40 ? { status: "review" } : {}),
        }).eq("id", item.id);
        updated++;
        if (newScore < 40 && item.freshness_score >= 40) flagged++;
      }
    }
    return NextResponse.json({ updated, flagged });
  } catch (error) {
    console.error("Freshness check failed:", error);
    return NextResponse.json({ error: "Check failed" }, { status: 500 });
  }
}
```

```json
// vercel.json
{ "crons": [{ "path": "/api/cron/freshness-check", "schedule": "0 6 * * *" }] }
```

## Regulatory Change Monitoring

```tsx
// lib/source-monitor.ts
import { createClient } from "@/lib/supabase/server";
import { computeHash } from "@/lib/change-detection";
import { getSemanticDiff } from "@/lib/semantic-diff";

export async function monitorSource(sourceUrl: string, contentId: string) {
  try {
    const response = await fetch(sourceUrl, { headers: { "User-Agent": "ContentMonitor/1.0" } });
    if (!response.ok) throw new Error(`Source returned ${response.status}`);
    const newContent = await response.text();
    const supabase = await createClient();

    const { data: item, error } = await supabase.from("content_items")
      .select("content_hash, body").eq("id", contentId).single();
    if (error) throw error;

    if (item.content_hash !== computeHash(newContent)) {
      const diff = await getSemanticDiff(item.body, newContent);
      await supabase.from("content_audit_log").insert({
        content_id: contentId, action: "source_change_detected",
        details: { source_url: sourceUrl, severity: diff.severity, summary: diff.summary },
      });
      if (diff.severity === "high" || diff.severity === "critical") {
        await supabase.from("content_items").update({ status: "review", freshness_score: 10 }).eq("id", contentId);
      }
      return { changed: true, diff };
    }
    return { changed: false };
  } catch (error) {
    console.error("Source monitoring failed:", error);
    return { changed: false, error };
  }
}
```

## Content Approval Workflow

```tsx
// app/api/content/[id]/approve/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";

export async function POST(request: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  try {
    const { id } = await params;
    const supabase = await createClient();
    const { data: { user } } = await supabase.auth.getUser();
    if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

    const { action, reason } = await request.json();
    const statusMap: Record<string, string> = { approve: "published", reject: "draft", request_changes: "draft" };
    if (!statusMap[action]) return NextResponse.json({ error: "Invalid action" }, { status: 400 });

    await supabase.from("content_items").update({
      status: statusMap[action], updated_by: user.id,
      ...(action === "approve" ? { published_at: new Date().toISOString(), freshness_score: 100 } : {}),
    }).eq("id", id);

    await supabase.from("content_audit_log").insert({
      content_id: id, action: `content_${action}d`, actor_id: user.id, details: { reason },
    });
    return NextResponse.json({ success: true });
  } catch (error) {
    console.error("Approval failed:", error);
    return NextResponse.json({ error: "Action failed" }, { status: 500 });
  }
}
```

See `references/freshness-scoring.md` for decay curves, urgency levels, auto-flagging rules, and full Supabase schema. See `references/change-detection.md` for hash algorithms, semantic diff, notification triggers, and alert escalation patterns.
