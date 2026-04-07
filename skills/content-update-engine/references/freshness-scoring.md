# Freshness Scoring Reference

Freshness scoring algorithm, decay functions, source weighting, urgency levels, auto-flagging rules, and Supabase schema for tracking.

## Freshness Scoring Algorithm

The freshness score tells you how current a piece of content is. It starts at 100 when content is verified and decays toward 0 over time. The decay speed depends on the content category and source reliability.

### Core Algorithm

```tsx
// lib/freshness-scoring.ts

type FreshnessConfig = {
  maxAgeDays: number;
  decayRate: number;
  sourceWeight: number;
  sensitivityMultiplier: number;
};

// How fast each content category goes stale
const CATEGORY_CONFIG: Record<string, FreshnessConfig> = {
  // Regulatory content changes frequently — stale after 30 days
  regulatory: {
    maxAgeDays: 30,
    decayRate: 3.0,
    sourceWeight: 0.9,
    sensitivityMultiplier: 2.0,
  },
  // Immigration/legal content — stale after 60 days
  immigration: {
    maxAgeDays: 45,
    decayRate: 2.5,
    sourceWeight: 0.85,
    sensitivityMultiplier: 1.8,
  },
  // Legal content — stale after 60 days
  legal: {
    maxAgeDays: 60,
    decayRate: 2.0,
    sourceWeight: 0.8,
    sensitivityMultiplier: 1.5,
  },
  // News/current events — stale after 7 days
  news: {
    maxAgeDays: 7,
    decayRate: 5.0,
    sourceWeight: 0.7,
    sensitivityMultiplier: 3.0,
  },
  // Product/pricing — stale after 90 days
  product: {
    maxAgeDays: 90,
    decayRate: 1.5,
    sourceWeight: 0.6,
    sensitivityMultiplier: 1.2,
  },
  // General content — stale after 180 days
  general: {
    maxAgeDays: 180,
    decayRate: 1.0,
    sourceWeight: 0.5,
    sensitivityMultiplier: 1.0,
  },
  // Evergreen content (how-to guides, tutorials) — stale after 1 year
  evergreen: {
    maxAgeDays: 365,
    decayRate: 0.5,
    sourceWeight: 0.3,
    sensitivityMultiplier: 0.5,
  },
};

// How much to trust the source
const SOURCE_RELIABILITY_WEIGHTS: Record<string, number> = {
  official: 1.0,   // Government websites, official documentation
  high: 0.8,       // Reputable news sources, established organizations
  medium: 0.6,     // Industry blogs, community resources
  low: 0.4,        // User-generated content, forums
};

export function calculateFreshnessScore(
  lastVerifiedAt: Date,
  category: string,
  sourceReliability: string
): number {
  const config = CATEGORY_CONFIG[category] || CATEGORY_CONFIG.general;
  const sourceWeight = SOURCE_RELIABILITY_WEIGHTS[sourceReliability] || 0.5;

  // Calculate age in days
  const ageInDays = (Date.now() - lastVerifiedAt.getTime()) / (1000 * 60 * 60 * 24);

  // Exponential decay: score = 100 * e^(-rate * age / maxAge)
  const rawScore =
    100 *
    Math.exp(
      (-config.decayRate * config.sensitivityMultiplier * ageInDays) /
        config.maxAgeDays
    );

  // More reliable sources decay slower
  // Formula: adjustedScore = rawScore * (0.5 + 0.5 * sourceWeight)
  // This means official sources keep 100% of their score,
  // while low-reliability sources only keep 70%
  const adjustedScore = rawScore * (0.5 + 0.5 * sourceWeight);

  return Math.max(0, Math.min(100, Math.round(adjustedScore)));
}
```

### Decay Curve Examples

For regulatory content (maxAgeDays=30, decayRate=3.0):
- Day 0: Score = 100
- Day 5: Score = 78
- Day 10: Score = 55
- Day 15: Score = 37
- Day 20: Score = 22
- Day 30: Score = 7

For evergreen content (maxAgeDays=365, decayRate=0.5):
- Day 0: Score = 100
- Day 30: Score = 96
- Day 90: Score = 88
- Day 180: Score = 78
- Day 365: Score = 61

## Urgency Levels

```tsx
// lib/freshness-levels.ts

export type FreshnessLevel = {
  label: string;
  color: string;
  bgColor: string;
  textColor: string;
  urgent: boolean;
  action: string;
};

export function getFreshnessLevel(score: number): FreshnessLevel {
  if (score >= 80) {
    return {
      label: "Fresh",
      color: "green",
      bgColor: "bg-green-100",
      textColor: "text-green-800",
      urgent: false,
      action: "No action needed",
    };
  }
  if (score >= 60) {
    return {
      label: "Aging",
      color: "yellow",
      bgColor: "bg-yellow-100",
      textColor: "text-yellow-800",
      urgent: false,
      action: "Schedule review within 2 weeks",
    };
  }
  if (score >= 40) {
    return {
      label: "Stale",
      color: "orange",
      bgColor: "bg-orange-100",
      textColor: "text-orange-800",
      urgent: true,
      action: "Review this week",
    };
  }
  if (score >= 20) {
    return {
      label: "Outdated",
      color: "red",
      bgColor: "bg-red-100",
      textColor: "text-red-800",
      urgent: true,
      action: "Review immediately",
    };
  }
  return {
    label: "Critical",
    color: "red",
    bgColor: "bg-red-200",
    textColor: "text-red-900",
    urgent: true,
    action: "Remove or update immediately — may contain wrong information",
  };
}
```

## Auto-Flagging Rules

```tsx
// lib/auto-flag.ts
import { createClient } from "@/lib/supabase/server";
import { calculateFreshnessScore, getFreshnessLevel } from "@/lib/freshness-scoring";

type FlagResult = {
  contentId: string;
  title: string;
  oldScore: number;
  newScore: number;
  action: string;
};

export async function runAutoFlagging(): Promise<FlagResult[]> {
  const supabase = await createClient();
  const results: FlagResult[] = [];

  try {
    const { data: items, error } = await supabase
      .from("content_items")
      .select("id, title, category, source_reliability, last_verified_at, freshness_score, status")
      .eq("status", "published");

    if (error) throw error;
    if (!items) return results;

    for (const item of items) {
      const newScore = calculateFreshnessScore(
        new Date(item.last_verified_at),
        item.category,
        item.source_reliability
      );

      // Only act if score changed significantly
      if (Math.abs(newScore - item.freshness_score) < 2) continue;

      const level = getFreshnessLevel(newScore);

      // Update the score
      await supabase
        .from("content_items")
        .update({
          freshness_score: newScore,
          ...(level.urgent && item.freshness_score >= 40 ? { status: "review" } : {}),
        })
        .eq("id", item.id);

      // Log if it crossed a threshold
      if (
        (newScore < 40 && item.freshness_score >= 40) ||
        (newScore < 20 && item.freshness_score >= 20)
      ) {
        await supabase.from("content_audit_log").insert({
          content_id: item.id,
          action: "auto_flagged",
          details: {
            old_score: item.freshness_score,
            new_score: newScore,
            level: level.label,
            recommended_action: level.action,
          },
        });

        results.push({
          contentId: item.id,
          title: item.title,
          oldScore: item.freshness_score,
          newScore,
          action: level.action,
        });
      }
    }

    return results;
  } catch (error) {
    console.error("Auto-flagging failed:", error);
    return results;
  }
}
```

## Supabase Schema for Freshness Tracking

### Complete Schema

```sql
-- supabase/migrations/create_freshness_tracking.sql

-- Freshness check history — keeps a log of every freshness calculation
CREATE TABLE freshness_checks (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  content_id UUID REFERENCES content_items(id) ON DELETE CASCADE,
  score_before FLOAT NOT NULL,
  score_after FLOAT NOT NULL,
  category TEXT NOT NULL,
  source_reliability TEXT NOT NULL,
  age_days FLOAT NOT NULL,
  flagged BOOLEAN DEFAULT false,
  checked_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_freshness_checks_content ON freshness_checks(content_id, checked_at DESC);
CREATE INDEX idx_freshness_checks_flagged ON freshness_checks(flagged) WHERE flagged = true;

-- Source monitoring configuration
CREATE TABLE monitored_sources (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  content_id UUID REFERENCES content_items(id) ON DELETE CASCADE,
  source_url TEXT NOT NULL,
  check_frequency_hours INTEGER DEFAULT 24,
  last_checked_at TIMESTAMPTZ,
  last_hash TEXT,
  consecutive_failures INTEGER DEFAULT 0,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_monitored_sources_active ON monitored_sources(is_active, last_checked_at);

-- Content review assignments
CREATE TABLE review_assignments (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  content_id UUID REFERENCES content_items(id) ON DELETE CASCADE,
  assignee_id UUID REFERENCES auth.users(id),
  priority TEXT DEFAULT 'medium' CHECK (priority IN ('low', 'medium', 'high', 'critical')),
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'in_progress', 'completed', 'skipped')),
  due_date TIMESTAMPTZ,
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  completed_at TIMESTAMPTZ
);

CREATE INDEX idx_review_assignments_assignee ON review_assignments(assignee_id, status);
CREATE INDEX idx_review_assignments_priority ON review_assignments(priority, status);

-- RLS policies
ALTER TABLE freshness_checks ENABLE ROW LEVEL SECURITY;
ALTER TABLE monitored_sources ENABLE ROW LEVEL SECURITY;
ALTER TABLE review_assignments ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Admins read freshness checks" ON freshness_checks
  FOR SELECT USING (
    EXISTS (SELECT 1 FROM user_roles WHERE user_id = auth.uid() AND role = 'admin')
  );

CREATE POLICY "Admins manage monitored sources" ON monitored_sources
  FOR ALL USING (
    EXISTS (SELECT 1 FROM user_roles WHERE user_id = auth.uid() AND role = 'admin')
  );

CREATE POLICY "Users read own assignments" ON review_assignments
  FOR SELECT USING (assignee_id = auth.uid());

CREATE POLICY "Admins manage assignments" ON review_assignments
  FOR ALL USING (
    EXISTS (SELECT 1 FROM user_roles WHERE user_id = auth.uid() AND role = 'admin')
  );
```

### Freshness Dashboard Query

```tsx
// lib/freshness-dashboard.ts
import { createClient } from "@/lib/supabase/server";

export async function getFreshnessDashboardData() {
  const supabase = await createClient();

  try {
    // Get content by freshness level
    const { data: items, error } = await supabase
      .from("content_items")
      .select("id, title, slug, category, freshness_score, last_verified_at, status")
      .eq("status", "published")
      .order("freshness_score", { ascending: true });

    if (error) throw error;

    const safeItems = items || [];

    // Calculate summary stats
    const total = safeItems.length;
    const fresh = safeItems.filter((i) => i.freshness_score >= 80).length;
    const aging = safeItems.filter((i) => i.freshness_score >= 60 && i.freshness_score < 80).length;
    const stale = safeItems.filter((i) => i.freshness_score >= 40 && i.freshness_score < 60).length;
    const outdated = safeItems.filter((i) => i.freshness_score < 40).length;

    const averageScore =
      total > 0
        ? Math.round(safeItems.reduce((sum, i) => sum + i.freshness_score, 0) / total)
        : 0;

    // Get items needing review (lowest scores)
    const needsReview = safeItems.filter((i) => i.freshness_score < 40).slice(0, 20);

    // Get pending review count
    const { count: pendingReviews } = await supabase
      .from("content_items")
      .select("*", { count: "exact", head: true })
      .eq("status", "review");

    return {
      summary: { total, fresh, aging, stale, outdated, averageScore },
      needsReview,
      pendingReviews: pendingReviews || 0,
    };
  } catch (error) {
    console.error("Dashboard data fetch failed:", error);
    return {
      summary: { total: 0, fresh: 0, aging: 0, stale: 0, outdated: 0, averageScore: 0 },
      needsReview: [],
      pendingReviews: 0,
    };
  }
}
```

### Freshness Trend Tracking

```tsx
// lib/freshness-trend.ts
import { createClient } from "@/lib/supabase/server";

export async function logFreshnessCheck(
  contentId: string,
  scoreBefore: number,
  scoreAfter: number,
  category: string,
  sourceReliability: string,
  ageDays: number
) {
  const supabase = await createClient();

  try {
    const { error } = await supabase.from("freshness_checks").insert({
      content_id: contentId,
      score_before: scoreBefore,
      score_after: scoreAfter,
      category,
      source_reliability: sourceReliability,
      age_days: ageDays,
      flagged: scoreAfter < 40,
    });

    if (error) throw error;
  } catch (error) {
    console.error("Failed to log freshness check:", error);
  }
}

export async function getFreshnessTrend(contentId: string, days: number = 30) {
  const supabase = await createClient();

  try {
    const since = new Date(Date.now() - days * 86400000).toISOString();

    const { data, error } = await supabase
      .from("freshness_checks")
      .select("score_after, checked_at")
      .eq("content_id", contentId)
      .gte("checked_at", since)
      .order("checked_at", { ascending: true });

    if (error) throw error;

    return (data || []).map((row) => ({
      date: row.checked_at.split("T")[0],
      score: row.score_after,
    }));
  } catch (error) {
    console.error("Failed to get freshness trend:", error);
    return [];
  }
}
```

## Notification Rules

```tsx
// lib/freshness-notifications.ts

type NotificationRule = {
  threshold: number;
  direction: "below" | "crossed";
  channel: "email" | "in_app" | "both";
  recipients: "author" | "admins" | "both";
  message: string;
};

export const NOTIFICATION_RULES: NotificationRule[] = [
  {
    threshold: 60,
    direction: "crossed",
    channel: "in_app",
    recipients: "author",
    message: "Your content \"{title}\" is aging. Consider reviewing it soon.",
  },
  {
    threshold: 40,
    direction: "crossed",
    channel: "both",
    recipients: "both",
    message: "Content \"{title}\" is now stale and needs review this week.",
  },
  {
    threshold: 20,
    direction: "crossed",
    channel: "both",
    recipients: "admins",
    message: "URGENT: Content \"{title}\" is outdated and may contain incorrect information.",
  },
];

export function getTriggeredRules(
  oldScore: number,
  newScore: number
): NotificationRule[] {
  return NOTIFICATION_RULES.filter((rule) => {
    if (rule.direction === "crossed") {
      return oldScore >= rule.threshold && newScore < rule.threshold;
    }
    return newScore < rule.threshold;
  });
}
```
