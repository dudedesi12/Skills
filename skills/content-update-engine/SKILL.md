---
name: content-update-engine
description: "Use this skill whenever the user mentions content freshness, content updates, regulatory changes, immigration updates, data staleness, 'is this still current', content versioning, audit trail, review queue, change detection, freshness scoring, 'check for changes', 'monitor updates', content calendar, scheduled updates, notifications for changes, 'data is outdated', compliance updates, or ANY content maintenance/freshness task — even if they don't explicitly say 'content update'. This skill keeps your content accurate and current automatically."
---

# Content Update Engine

This skill covers everything you need to keep your content accurate and current. It includes content versioning, freshness scoring, change detection, review queues, and automated monitoring — so stale content gets flagged before users notice.

## Content Versioning System

### Supabase Schema

Every piece of content gets a version history. When content changes, the old version is preserved.

```sql
-- supabase/migrations/create_content_versioning.sql

-- Main content table
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

-- Version history
CREATE TABLE content_versions (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  content_id UUID REFERENCES content_items(id) ON DELETE CASCADE,
  version_number INTEGER NOT NULL,
  title TEXT NOT NULL,
  body TEXT NOT NULL,
  change_summary TEXT,
  changed_by UUID REFERENCES auth.users(id),
  change_reason TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Audit trail
CREATE TABLE content_audit_log (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  content_id UUID REFERENCES content_items(id) ON DELETE CASCADE,
  action TEXT NOT NULL,
  actor_id UUID REFERENCES auth.users(id),
  details JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Indexes
CREATE INDEX idx_content_slug ON content_items(slug);
CREATE INDEX idx_content_status ON content_items(status);
CREATE INDEX idx_content_freshness ON content_items(freshness_score);
CREATE INDEX idx_content_verified ON content_items(last_verified_at);
CREATE INDEX idx_versions_content ON content_versions(content_id, version_number DESC);
CREATE INDEX idx_audit_content ON content_audit_log(content_id, created_at DESC);

-- RLS
ALTER TABLE content_items ENABLE ROW LEVEL SECURITY;
ALTER TABLE content_versions ENABLE ROW LEVEL SECURITY;
ALTER TABLE content_audit_log ENABLE ROW LEVEL SECURITY;

-- Public can read published content
CREATE POLICY "Public read published" ON content_items
  FOR SELECT USING (status = 'published');

-- Admins can do everything
CREATE POLICY "Admins full access" ON content_items
  FOR ALL USING (
    EXISTS (SELECT 1 FROM user_roles WHERE user_id = auth.uid() AND role = 'admin')
  );

CREATE POLICY "Admins read versions" ON content_versions
  FOR SELECT USING (
    EXISTS (SELECT 1 FROM user_roles WHERE user_id = auth.uid() AND role = 'admin')
  );

CREATE POLICY "Admins read audit" ON content_audit_log
  FOR SELECT USING (
    EXISTS (SELECT 1 FROM user_roles WHERE user_id = auth.uid() AND role = 'admin')
  );
```

### Version Management Functions

```tsx
// lib/content-versioning.ts
import { createClient } from "@/lib/supabase/server";
import crypto from "crypto";

export function hashContent(content: string): string {
  return crypto.createHash("sha256").update(content).digest("hex");
}

export async function updateContent(
  contentId: string,
  updates: { title: string; body: string },
  userId: string,
  changeReason: string
) {
  const supabase = await createClient();

  try {
    // Get current content
    const { data: current, error: fetchError } = await supabase
      .from("content_items")
      .select("title, body")
      .eq("id", contentId)
      .single();

    if (fetchError) throw fetchError;

    // Get current version number
    const { count } = await supabase
      .from("content_versions")
      .select("*", { count: "exact", head: true })
      .eq("content_id", contentId);

    const nextVersion = (count || 0) + 1;

    // Save current as a version
    const { error: versionError } = await supabase
      .from("content_versions")
      .insert({
        content_id: contentId,
        version_number: nextVersion,
        title: current.title,
        body: current.body,
        changed_by: userId,
        change_reason: changeReason,
      });

    if (versionError) throw versionError;

    // Update the content
    const newHash = hashContent(updates.body);
    const { error: updateError } = await supabase
      .from("content_items")
      .update({
        title: updates.title,
        body: updates.body,
        content_hash: newHash,
        updated_by: userId,
        updated_at: new Date().toISOString(),
        last_verified_at: new Date().toISOString(),
        freshness_score: 100,
      })
      .eq("id", contentId);

    if (updateError) throw updateError;

    // Log the change
    await supabase.from("content_audit_log").insert({
      content_id: contentId,
      action: "content_updated",
      actor_id: userId,
      details: { version: nextVersion, reason: changeReason },
    });

    return { success: true, version: nextVersion };
  } catch (error) {
    console.error("Content update failed:", error);
    return { success: false, error };
  }
}
```

## Freshness Scoring Algorithm

Content gets a freshness score from 0 to 100. The score decays over time based on the content category and source reliability.

```tsx
// lib/freshness-scoring.ts

type FreshnessConfig = {
  maxAgeDays: number;      // After this many days, score hits 0
  decayRate: number;       // How fast the score drops (higher = faster)
  sourceWeight: number;    // 0-1, how much to trust the source
  sensitivityMultiplier: number; // Higher = decays faster
};

const CATEGORY_CONFIG: Record<string, FreshnessConfig> = {
  regulatory: {
    maxAgeDays: 30,
    decayRate: 3.0,
    sourceWeight: 0.9,
    sensitivityMultiplier: 2.0,
  },
  legal: {
    maxAgeDays: 60,
    decayRate: 2.0,
    sourceWeight: 0.8,
    sensitivityMultiplier: 1.5,
  },
  general: {
    maxAgeDays: 180,
    decayRate: 1.0,
    sourceWeight: 0.5,
    sensitivityMultiplier: 1.0,
  },
  evergreen: {
    maxAgeDays: 365,
    decayRate: 0.5,
    sourceWeight: 0.3,
    sensitivityMultiplier: 0.5,
  },
};

const SOURCE_RELIABILITY_WEIGHTS: Record<string, number> = {
  official: 1.0,
  high: 0.8,
  medium: 0.6,
  low: 0.4,
};

export function calculateFreshnessScore(
  lastVerifiedAt: Date,
  category: string,
  sourceReliability: string
): number {
  const config = CATEGORY_CONFIG[category] || CATEGORY_CONFIG.general;
  const sourceWeight = SOURCE_RELIABILITY_WEIGHTS[sourceReliability] || 0.5;

  const ageInDays = (Date.now() - lastVerifiedAt.getTime()) / (1000 * 60 * 60 * 24);

  // Exponential decay formula
  const rawScore = 100 * Math.exp(
    (-config.decayRate * config.sensitivityMultiplier * ageInDays) / config.maxAgeDays
  );

  // Adjust by source reliability (more reliable sources decay slower)
  const adjustedScore = rawScore * (0.5 + 0.5 * sourceWeight);

  return Math.max(0, Math.min(100, Math.round(adjustedScore)));
}

export function getFreshnessLevel(score: number): {
  label: string;
  color: string;
  urgent: boolean;
} {
  if (score >= 80) return { label: "Fresh", color: "green", urgent: false };
  if (score >= 60) return { label: "Aging", color: "yellow", urgent: false };
  if (score >= 40) return { label: "Stale", color: "orange", urgent: true };
  return { label: "Outdated", color: "red", urgent: true };
}
```

## Change Detection

### Hash-Based Change Detection

Compare content hashes to detect when source material has changed.

```tsx
// lib/change-detection.ts
import crypto from "crypto";
import { createClient } from "@/lib/supabase/server";

export function computeHash(text: string): string {
  // Normalize whitespace before hashing so formatting changes don't trigger alerts
  const normalized = text.replace(/\s+/g, " ").trim().toLowerCase();
  return crypto.createHash("sha256").update(normalized).digest("hex");
}

export async function checkForChanges(contentId: string, newContent: string) {
  const supabase = await createClient();

  try {
    const { data: existing, error } = await supabase
      .from("content_items")
      .select("content_hash, title, slug")
      .eq("id", contentId)
      .single();

    if (error) throw error;

    const newHash = computeHash(newContent);
    const hasChanged = existing.content_hash !== newHash;

    if (hasChanged) {
      // Log the detection
      await supabase.from("content_audit_log").insert({
        content_id: contentId,
        action: "change_detected",
        details: {
          old_hash: existing.content_hash,
          new_hash: newHash,
        },
      });

      // Flag for review
      await supabase
        .from("content_items")
        .update({ status: "review", freshness_score: 20 })
        .eq("id", contentId);
    }

    return { hasChanged, contentId, title: existing.title, slug: existing.slug };
  } catch (error) {
    console.error("Change detection failed:", error);
    return { hasChanged: false, error };
  }
}
```

### Gemini-Powered Semantic Diff

Use the Gemini API to understand what actually changed in meaning, not just text.

```tsx
// lib/semantic-diff.ts
import { GoogleGenerativeAI } from "@google/generative-ai";

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);

export async function getSemanticDiff(
  oldContent: string,
  newContent: string
): Promise<{
  summary: string;
  severity: "low" | "medium" | "high" | "critical";
  keyChanges: string[];
}> {
  try {
    const model = genAI.getGenerativeModel({ model: "gemini-2.0-flash" });

    const prompt = `Compare these two versions of content and analyze the changes.

OLD VERSION:
${oldContent.substring(0, 3000)}

NEW VERSION:
${newContent.substring(0, 3000)}

Respond in JSON format:
{
  "summary": "Brief summary of what changed",
  "severity": "low|medium|high|critical",
  "keyChanges": ["change 1", "change 2"]
}

Severity guide:
- low: typos, formatting, minor wording
- medium: updated facts, new information added
- high: significant policy or requirement changes
- critical: legal changes, deadline changes, process changes that affect users`;

    const result = await model.generateContent(prompt);
    const text = result.response.text();

    // Parse JSON from response
    const jsonMatch = text.match(/\{[\s\S]*\}/);
    if (!jsonMatch) {
      return {
        summary: "Unable to analyze changes",
        severity: "medium",
        keyChanges: ["Content has been modified"],
      };
    }

    return JSON.parse(jsonMatch[0]);
  } catch (error) {
    console.error("Semantic diff failed:", error);
    return {
      summary: "Analysis unavailable",
      severity: "medium",
      keyChanges: ["Content has been modified but analysis failed"],
    };
  }
}
```

## User Notification System

### Freshness Badge Component

Show users when content was last verified.

```tsx
// components/freshness-badge.tsx
import { getFreshnessLevel } from "@/lib/freshness-scoring";

type FreshnessBadgeProps = {
  score: number;
  lastVerifiedAt: string;
};

export function FreshnessBadge({ score, lastVerifiedAt }: FreshnessBadgeProps) {
  const level = getFreshnessLevel(score);
  const date = new Date(lastVerifiedAt).toLocaleDateString("en-US", {
    month: "short",
    day: "numeric",
    year: "numeric",
  });

  const colorClasses: Record<string, string> = {
    green: "bg-green-100 text-green-800 border-green-200",
    yellow: "bg-yellow-100 text-yellow-800 border-yellow-200",
    orange: "bg-orange-100 text-orange-800 border-orange-200",
    red: "bg-red-100 text-red-800 border-red-200",
  };

  return (
    <div className={`inline-flex items-center gap-1.5 px-2.5 py-1 rounded-full text-xs font-medium border ${colorClasses[level.color]}`}>
      <span className={`w-1.5 h-1.5 rounded-full ${
        level.color === "green" ? "bg-green-500" :
        level.color === "yellow" ? "bg-yellow-500" :
        level.color === "orange" ? "bg-orange-500" : "bg-red-500"
      }`} />
      {level.label} — Verified {date}
    </div>
  );
}
```

## Review Queue

### Admin Review Dashboard

```tsx
// app/admin/review-queue/page.tsx
import { createClient } from "@/lib/supabase/server";
import { FreshnessBadge } from "@/components/freshness-badge";

export default async function ReviewQueuePage() {
  const supabase = await createClient();

  const { data: items, error } = await supabase
    .from("content_items")
    .select("id, title, slug, category, freshness_score, last_verified_at, status, updated_at")
    .or("status.eq.review,freshness_score.lt.40")
    .order("freshness_score", { ascending: true });

  if (error) {
    return <p className="p-8 text-red-600">Failed to load review queue: {error.message}</p>;
  }

  return (
    <div className="max-w-5xl mx-auto p-8">
      <div className="flex items-center justify-between mb-6">
        <h1 className="text-2xl font-bold">Content Review Queue</h1>
        <span className="text-sm text-gray-500">
          {items?.length || 0} items need attention
        </span>
      </div>

      {!items || items.length === 0 ? (
        <div className="text-center py-12 bg-white rounded-lg border">
          <p className="text-gray-500">All content is up to date.</p>
        </div>
      ) : (
        <div className="space-y-3">
          {items.map((item) => (
            <div
              key={item.id}
              className="bg-white rounded-lg border p-4 hover:shadow-sm transition-shadow"
            >
              <div className="flex items-start justify-between gap-4">
                <div className="flex-1">
                  <h3 className="font-semibold">{item.title}</h3>
                  <p className="text-sm text-gray-500 mt-1">
                    /{item.slug} — {item.category}
                  </p>
                </div>
                <FreshnessBadge
                  score={item.freshness_score}
                  lastVerifiedAt={item.last_verified_at}
                />
              </div>
              <div className="flex gap-2 mt-4">
                <a
                  href={`/admin/content/${item.id}/edit`}
                  className="px-3 py-1.5 text-sm bg-blue-600 text-white rounded hover:bg-blue-700"
                >
                  Review & Update
                </a>
                <form action={`/api/content/${item.id}/verify`} method="POST">
                  <button
                    type="submit"
                    className="px-3 py-1.5 text-sm border rounded hover:bg-gray-50"
                  >
                    Mark as Verified
                  </button>
                </form>
              </div>
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
```

### Verify Content API

```tsx
// app/api/content/[id]/verify/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";

export async function POST(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  try {
    const { id } = await params;
    const supabase = await createClient();
    const { data: { user } } = await supabase.auth.getUser();

    if (!user) {
      return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
    }

    const { error } = await supabase
      .from("content_items")
      .update({
        last_verified_at: new Date().toISOString(),
        freshness_score: 100,
        status: "published",
        updated_by: user.id,
      })
      .eq("id", id);

    if (error) throw error;

    // Audit log
    await supabase.from("content_audit_log").insert({
      content_id: id,
      action: "verified",
      actor_id: user.id,
      details: { verified_at: new Date().toISOString() },
    });

    return NextResponse.json({ success: true });
  } catch (error) {
    console.error("Verify failed:", error);
    return NextResponse.json({ error: "Verification failed" }, { status: 500 });
  }
}
```

## Automated Freshness Checks with Vercel Cron

```ts
// app/api/cron/freshness-check/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";
import { calculateFreshnessScore } from "@/lib/freshness-scoring";

export async function GET(request: NextRequest) {
  // Verify cron secret
  const authHeader = request.headers.get("authorization");
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const supabase = await createClient();

  try {
    const { data: items, error } = await supabase
      .from("content_items")
      .select("id, category, source_reliability, last_verified_at, freshness_score")
      .eq("status", "published");

    if (error) throw error;
    if (!items) return NextResponse.json({ updated: 0 });

    let updated = 0;
    let flagged = 0;

    for (const item of items) {
      const newScore = calculateFreshnessScore(
        new Date(item.last_verified_at),
        item.category,
        item.source_reliability
      );

      if (newScore !== item.freshness_score) {
        await supabase
          .from("content_items")
          .update({ freshness_score: newScore })
          .eq("id", item.id);
        updated++;

        // Flag content that has become stale
        if (newScore < 40 && item.freshness_score >= 40) {
          await supabase
            .from("content_items")
            .update({ status: "review" })
            .eq("id", item.id);
          flagged++;
        }
      }
    }

    return NextResponse.json({ updated, flagged, total: items.length });
  } catch (error) {
    console.error("Freshness check failed:", error);
    return NextResponse.json({ error: "Check failed" }, { status: 500 });
  }
}
```

Add the cron schedule to your `vercel.json`:

```json
// vercel.json
{
  "crons": [
    {
      "path": "/api/cron/freshness-check",
      "schedule": "0 6 * * *"
    }
  ]
}
```

This runs the freshness check every day at 6 AM UTC.

## Regulatory Change Monitoring

### Source Monitor

```tsx
// lib/source-monitor.ts
import { createClient } from "@/lib/supabase/server";
import { computeHash } from "@/lib/change-detection";
import { getSemanticDiff } from "@/lib/semantic-diff";

export async function monitorSource(sourceUrl: string, contentId: string) {
  try {
    // Fetch the source page
    const response = await fetch(sourceUrl, {
      headers: { "User-Agent": "ContentMonitor/1.0" },
    });

    if (!response.ok) {
      throw new Error(`Source returned ${response.status}`);
    }

    const newContent = await response.text();
    const newHash = computeHash(newContent);

    const supabase = await createClient();

    // Get stored hash
    const { data: item, error } = await supabase
      .from("content_items")
      .select("content_hash, body")
      .eq("id", contentId)
      .single();

    if (error) throw error;

    if (item.content_hash !== newHash) {
      // Content changed — analyze the diff
      const diff = await getSemanticDiff(item.body, newContent);

      // Log the detection
      await supabase.from("content_audit_log").insert({
        content_id: contentId,
        action: "source_change_detected",
        details: {
          source_url: sourceUrl,
          severity: diff.severity,
          summary: diff.summary,
          key_changes: diff.keyChanges,
        },
      });

      // Flag for review if significant
      if (diff.severity === "high" || diff.severity === "critical") {
        await supabase
          .from("content_items")
          .update({ status: "review", freshness_score: 10 })
          .eq("id", contentId);
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

export async function POST(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  try {
    const { id } = await params;
    const supabase = await createClient();
    const { data: { user } } = await supabase.auth.getUser();

    if (!user) {
      return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
    }

    const body = await request.json();
    const { action, reason } = body;

    if (!["approve", "reject", "request_changes"].includes(action)) {
      return NextResponse.json({ error: "Invalid action" }, { status: 400 });
    }

    const statusMap: Record<string, string> = {
      approve: "published",
      reject: "draft",
      request_changes: "draft",
    };

    const { error } = await supabase
      .from("content_items")
      .update({
        status: statusMap[action],
        updated_by: user.id,
        updated_at: new Date().toISOString(),
        ...(action === "approve" ? {
          published_at: new Date().toISOString(),
          last_verified_at: new Date().toISOString(),
          freshness_score: 100,
        } : {}),
      })
      .eq("id", id);

    if (error) throw error;

    await supabase.from("content_audit_log").insert({
      content_id: id,
      action: `content_${action}d`,
      actor_id: user.id,
      details: { reason: reason || null },
    });

    return NextResponse.json({ success: true, status: statusMap[action] });
  } catch (error) {
    console.error("Approval action failed:", error);
    return NextResponse.json({ error: "Action failed" }, { status: 500 });
  }
}
```

See the `references/` folder for the complete freshness scoring algorithm and change detection patterns.
