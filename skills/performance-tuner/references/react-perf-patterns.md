# React Performance Patterns

## Virtualized Lists

For lists with 100+ items, render only what's visible:

```bash
npm install @tanstack/react-virtual
```

```tsx
"use client";
import { useVirtualizer } from "@tanstack/react-virtual";
import { useRef } from "react";

interface VirtualListProps {
  items: { id: string; name: string; score: number }[];
}

export function VirtualList({ items }: VirtualListProps) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 64, // estimated row height in px
    overscan: 5, // render 5 extra items above/below viewport
  });

  return (
    <div ref={parentRef} className="h-96 overflow-auto rounded border">
      <div style={{ height: `${virtualizer.getTotalSize()}px`, position: "relative" }}>
        {virtualizer.getVirtualItems().map((virtualRow) => {
          const item = items[virtualRow.index];
          return (
            <div
              key={item.id}
              style={{
                position: "absolute",
                top: 0,
                left: 0,
                width: "100%",
                height: `${virtualRow.size}px`,
                transform: `translateY(${virtualRow.start}px)`,
              }}
              className="flex items-center justify-between border-b px-4"
            >
              <span>{item.name}</span>
              <span className="text-sm text-gray-500">{item.score} pts</span>
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

## Debounced Search Input

Prevent API calls on every keystroke:

```tsx
"use client";
import { useState, useEffect, useRef } from "react";

export function DebouncedSearch({ onSearch }: { onSearch: (query: string) => void }) {
  const [value, setValue] = useState("");
  const timerRef = useRef<NodeJS.Timeout>();

  useEffect(() => {
    clearTimeout(timerRef.current);
    timerRef.current = setTimeout(() => {
      onSearch(value);
    }, 300); // Wait 300ms after last keystroke

    return () => clearTimeout(timerRef.current);
  }, [value, onSearch]);

  return (
    <input
      value={value}
      onChange={(e) => setValue(e.target.value)}
      placeholder="Search occupations..."
      className="w-full rounded border px-3 py-2"
    />
  );
}
```

## Optimistic Updates

Show the result immediately, sync in background:

```tsx
"use client";
import { useState, useCallback } from "react";

export function BookmarkButton({ assessmentId, initialBookmarked }: {
  assessmentId: string;
  initialBookmarked: boolean;
}) {
  const [isBookmarked, setIsBookmarked] = useState(initialBookmarked);

  const toggle = useCallback(async () => {
    const newState = !isBookmarked;
    setIsBookmarked(newState); // Optimistic: update UI immediately

    try {
      const res = await fetch(`/api/bookmarks/${assessmentId}`, {
        method: newState ? "POST" : "DELETE",
      });
      if (!res.ok) setIsBookmarked(!newState); // Revert on failure
    } catch {
      setIsBookmarked(!newState); // Revert on error
    }
  }, [assessmentId, isBookmarked]);

  return (
    <button onClick={toggle} className="text-2xl">
      {isBookmarked ? "★" : "☆"}
    </button>
  );
}
```

## Anti-Patterns: When NOT to Optimize

### Don't wrap everything in memo

```tsx
// BAD: Unnecessary memo — this component is cheap
const Label = memo(({ text }: { text: string }) => (
  <span className="text-sm text-gray-500">{text}</span>
));

// GOOD: Just use the component directly
function Label({ text }: { text: string }) {
  return <span className="text-sm text-gray-500">{text}</span>;
}
```

### Don't useMemo for simple values

```tsx
// BAD: useMemo overhead > computation cost
const name = useMemo(() => `${firstName} ${lastName}`, [firstName, lastName]);

// GOOD: Just compute it
const name = `${firstName} ${lastName}`;
```

### Don't create new objects in render

```tsx
// BAD: New style object every render → children re-render even with memo
<UserCard user={user} style={{ padding: 16, margin: 8 }} />

// GOOD: Stable reference
const cardStyle = { padding: 16, margin: 8 }; // Outside component
<UserCard user={user} style={cardStyle} />

// BEST: Use Tailwind classes instead of style objects
<UserCard user={user} className="p-4 m-2" />
```

## Streaming with Suspense

```tsx
// app/dashboard/page.tsx
import { Suspense } from "react";

export default function DashboardPage() {
  return (
    <div>
      <h1>Dashboard</h1>

      {/* Fast data loads first */}
      <Suspense fallback={<ProfileSkeleton />}>
        <ProfileSection />
      </Suspense>

      {/* Slow data streams in later */}
      <Suspense fallback={<AssessmentsSkeleton />}>
        <RecentAssessments />
      </Suspense>

      {/* Even slower data streams last */}
      <Suspense fallback={<RecommendationsSkeleton />}>
        <AIRecommendations />
      </Suspense>
    </div>
  );
}

// Each component fetches its own data (server component)
async function RecentAssessments() {
  const supabase = await createClient();
  const { data } = await supabase
    .from("assessments")
    .select("id, visa_subclass, points_score")
    .order("created_at", { ascending: false })
    .limit(5);

  return <AssessmentList assessments={data ?? []} />;
}
```
