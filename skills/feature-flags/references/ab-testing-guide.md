# A/B Testing with Feature Flags

## How It Works

1. Feature flag randomly assigns users to Control (old) or Treatment (new)
2. Both groups are tracked via analytics events
3. After enough data, compare conversion rates
4. If treatment wins, roll out to 100%

## Setting Up an A/B Test

### Step 1: Create the Flag

```sql
INSERT INTO feature_flags (name, description, enabled, rollout_percentage)
VALUES ('test-new-cta-button', 'A/B test: new CTA button on assessment page', true, 50);
```

### Step 2: Implement Both Variants

```tsx
// app/assessment/page.tsx
import { FeatureFlag } from "@/components/feature-flag";

export default async function AssessmentPage() {
  return (
    <div>
      <h1>Visa Assessment</h1>
      <FeatureFlag
        name="test-new-cta-button"
        fallback={<CTAButtonControl />}
      >
        <CTAButtonTreatment />
      </FeatureFlag>
    </div>
  );
}

function CTAButtonControl() {
  return (
    <button className="rounded bg-gray-600 px-6 py-3 text-white">
      Start Assessment
    </button>
  );
}

function CTAButtonTreatment() {
  return (
    <button className="rounded bg-blue-600 px-8 py-4 text-lg font-bold text-white shadow-lg">
      Check My Eligibility Now
    </button>
  );
}
```

### Step 3: Track Exposure and Conversion

```typescript
// Track when user sees the variant (exposure)
async function trackExposure(flagName: string, userId: string, variant: string) {
  await fetch("/api/analytics/track", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      event: "ab_test_exposure",
      properties: { test: flagName, variant, userId },
    }),
  });
}

// Track when user completes the desired action (conversion)
async function trackConversion(flagName: string, userId: string, variant: string) {
  await fetch("/api/analytics/track", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      event: "ab_test_conversion",
      properties: { test: flagName, variant, userId },
    }),
  });
}
```

### Step 4: Analyze Results

```sql
-- Get exposure counts per variant
SELECT
  properties->>'variant' AS variant,
  COUNT(DISTINCT properties->>'userId') AS users
FROM analytics_events
WHERE event = 'ab_test_exposure'
  AND properties->>'test' = 'test-new-cta-button'
GROUP BY variant;

-- Get conversion rates
WITH exposures AS (
  SELECT DISTINCT
    properties->>'userId' AS user_id,
    properties->>'variant' AS variant
  FROM analytics_events
  WHERE event = 'ab_test_exposure'
    AND properties->>'test' = 'test-new-cta-button'
),
conversions AS (
  SELECT DISTINCT properties->>'userId' AS user_id
  FROM analytics_events
  WHERE event = 'ab_test_conversion'
    AND properties->>'test' = 'test-new-cta-button'
)
SELECT
  e.variant,
  COUNT(e.user_id) AS exposed,
  COUNT(c.user_id) AS converted,
  ROUND(COUNT(c.user_id)::numeric / NULLIF(COUNT(e.user_id), 0) * 100, 2) AS conversion_rate
FROM exposures e
LEFT JOIN conversions c ON e.user_id = c.user_id
GROUP BY e.variant;
```

### Step 5: Make a Decision

| Metric | Control | Treatment | Winner |
|--------|---------|-----------|--------|
| Users exposed | 500 | 500 | — |
| Conversions | 50 | 75 | Treatment |
| Conversion rate | 10% | 15% | Treatment (+50%) |

If treatment wins → set rollout to 100%
If control wins → set rollout to 0%, remove treatment code
If inconclusive → run longer or try a different approach

## Sample Size Calculator

You need enough users to be statistically confident:

```typescript
// Minimum sample size per variant for 95% confidence, 80% power
function minSampleSize(baseRate: number, minDetectableEffect: number): number {
  // Simplified formula
  const p1 = baseRate;
  const p2 = baseRate * (1 + minDetectableEffect);
  const pooledP = (p1 + p2) / 2;
  const z_alpha = 1.96; // 95% confidence
  const z_beta = 0.84;  // 80% power
  const n = (2 * pooledP * (1 - pooledP) * (z_alpha + z_beta) ** 2) / (p2 - p1) ** 2;
  return Math.ceil(n);
}

// Example: 10% base conversion, want to detect 20% improvement
const needed = minSampleSize(0.10, 0.20);
// ~3,900 users per variant
```

## Rules for A/B Tests

1. **Define success metric before starting** — What counts as a "conversion"?
2. **Run for at least 1 week** — Weekday vs weekend behavior differs
3. **Don't peek at results early** — Wait for the planned sample size
4. **One test per page** — Multiple tests on the same page contaminate results
5. **Document everything** — Hypothesis, variants, results, decision
