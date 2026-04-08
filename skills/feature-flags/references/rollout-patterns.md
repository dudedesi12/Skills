# Rollout Patterns

## Gradual Percentage Rollout

The safest way to release a new feature:

```
Phase 1: Internal (0-1%)
  → Enable for your team (allowed_user_ids or role=admin)
  → Monitor for errors, test core flows

Phase 2: Canary (1-5%)
  → Small percentage of real users
  → Monitor error rates, performance, user feedback
  → Duration: 1-2 days

Phase 3: Early Adopter (10-25%)
  → Larger sample, statistically meaningful
  → Monitor conversion rates, engagement
  → Duration: 2-3 days

Phase 4: Majority (50%)
  → Half of users
  → Final validation before full rollout
  → Duration: 1-2 days

Phase 5: Full Rollout (100%)
  → All users
  → Keep flag for 2 weeks as a kill switch
  → Then clean up the flag
```

## Deterministic User Assignment

Users must always see the same variant. Use consistent hashing:

```typescript
function getUserBucket(flagName: string, userId: string): number {
  // MD5 hash of flag+user gives a deterministic number 0-99
  const hash = crypto.createHash("md5").update(`${flagName}:${userId}`).digest("hex");
  return parseInt(hash.substring(0, 8), 16) % 100;
}

function isUserInRollout(flagName: string, userId: string, percentage: number): boolean {
  const bucket = getUserBucket(flagName, userId);
  return bucket < percentage;
}

// At 10%: users in buckets 0-9 see the feature
// At 50%: users in buckets 0-49 see the feature
// At 100%: all users see the feature
// Increasing the percentage NEVER removes users who were already in
```

## Ring Deployment

Deploy to concentric rings of increasing risk:

```
Ring 0: Development     → All new features
Ring 1: Internal/Dogfood → Team members
Ring 2: Preview/Staging  → QA testers
Ring 3: Canary           → 1% of production
Ring 4: Early Majority   → 25% of production
Ring 5: Full Production  → 100%
```

```typescript
const RINGS: Record<string, { environments: string[]; percentage: number }> = {
  "ring-0": { environments: ["development"], percentage: 100 },
  "ring-1": { environments: ["development", "preview"], percentage: 100 },
  "ring-2": { environments: ["development", "preview"], percentage: 100 },
  "ring-3": { environments: ["development", "preview", "production"], percentage: 1 },
  "ring-4": { environments: ["development", "preview", "production"], percentage: 25 },
  "ring-5": { environments: ["development", "preview", "production"], percentage: 100 },
};
```

## Emergency Rollback

When a feature breaks in production:

```typescript
// 1. Kill the flag immediately
await killFeature("broken-feature", "Causing 500 errors on assessment page");

// 2. The flag evaluation returns false for everyone
// 3. Users see the old UI/behavior
// 4. Fix the bug
// 5. Re-enable gradually (start at 1% again)
```

## Monitoring During Rollout

Check these metrics at each rollout phase:

```
Error Rate: Compare treatment vs control group error rates
  → If treatment errors > control errors × 1.5 → STOP rollout

Performance: Compare p95 response times
  → If treatment p95 > control p95 × 1.3 → STOP rollout

Engagement: Compare key actions (assessments started, completed)
  → If treatment engagement < control × 0.8 → INVESTIGATE

Revenue: Compare conversion rates
  → If treatment conversion < control × 0.9 → INVESTIGATE
```
