# Server Components vs Client Components

## The Rule

Every component in the App Router is a **Server Component** by default. It runs on the server, never ships JavaScript to the browser, and can directly access databases and secrets.

Add `"use client"` at the very top of a file to make it a **Client Component**. It runs in the browser and can use React hooks and event handlers.

## Decision Tree

Ask these questions in order:

### 1. Does this component need interactivity?
- `onClick`, `onChange`, `onSubmit` handlers → **Client Component**
- `useState`, `useReducer` for local state → **Client Component**
- No interactivity at all → **Server Component**

### 2. Does this component need browser APIs?
- `window`, `document`, `localStorage` → **Client Component**
- `navigator.geolocation` → **Client Component**
- `IntersectionObserver`, `ResizeObserver` → **Client Component**
- None of these → **Server Component**

### 3. Does this component need React lifecycle hooks?
- `useEffect` → **Client Component**
- `useRef` (for DOM manipulation) → **Client Component**
- `useContext` → **Client Component**
- No hooks → **Server Component**

### 4. Does this component use a third-party library that requires hooks?
- Framer Motion, react-hook-form, react-select → **Client Component**
- Headless UI, Radix UI dialogs/dropdowns → **Client Component**
- Libraries that just export functions (date-fns, zod) → **Server Component is fine**

### 5. Does this component fetch data?
- Fetching from Supabase to display → **Server Component** (preferred)
- Fetching + needs real-time updates → **Client Component** (with Supabase realtime)

## Server Component Examples

### Fetching and Displaying Data

```tsx
// app/users/page.tsx
// NO "use client" — this is a Server Component
import { createClient } from "@/lib/supabase/server";

export default async function UsersPage() {
  const supabase = await createClient();
  const { data: users, error } = await supabase
    .from("profiles")
    .select("id, full_name, avatar_url")
    .order("full_name");

  if (error) {
    throw new Error("Failed to load users");
  }

  return (
    <ul className="space-y-2 p-8">
      {users.map((user) => (
        <li key={user.id} className="flex items-center gap-3">
          <img
            src={user.avatar_url ?? "/default-avatar.png"}
            alt=""
            className="h-10 w-10 rounded-full"
          />
          <span>{user.full_name}</span>
        </li>
      ))}
    </ul>
  );
}
```

### Accessing Environment Variables (Server Secrets)

```tsx
// app/admin/page.tsx
// Server Component — safe to use secret env vars
import { createClient } from "@supabase/supabase-js";

export default async function AdminPage() {
  // Service role key is NEVER exposed to the browser
  const supabase = createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!
  );

  const { data: { users }, error } = await supabase.auth.admin.listUsers();

  if (error) {
    throw new Error("Failed to list users");
  }

  return (
    <div className="p-8">
      <h1 className="text-2xl font-bold">Admin: {users?.length} users</h1>
    </div>
  );
}
```

## Client Component Examples

### Interactive Form

```tsx
// app/components/search-bar.tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";

export function SearchBar() {
  const [query, setQuery] = useState("");
  const router = useRouter();

  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    if (query.trim()) {
      router.push(`/search?q=${encodeURIComponent(query.trim())}`);
    }
  }

  return (
    <form onSubmit={handleSubmit} className="flex gap-2">
      <input
        type="text"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search..."
        className="rounded border px-3 py-2"
      />
      <button type="submit" className="rounded bg-blue-500 px-4 py-2 text-white">
        Search
      </button>
    </form>
  );
}
```

### Real-Time Subscription

```tsx
// app/components/live-comments.tsx
"use client";

import { useEffect, useState } from "react";
import { createClient } from "@/lib/supabase/client";

interface Comment {
  id: string;
  body: string;
  created_at: string;
}

export function LiveComments({ postId }: { postId: string }) {
  const [comments, setComments] = useState<Comment[]>([]);
  const supabase = createClient();

  useEffect(() => {
    // Initial fetch
    supabase
      .from("comments")
      .select("id, body, created_at")
      .eq("post_id", postId)
      .order("created_at", { ascending: true })
      .then(({ data }) => {
        if (data) setComments(data);
      });

    // Real-time subscription
    const channel = supabase
      .channel(`comments:${postId}`)
      .on(
        "postgres_changes",
        {
          event: "INSERT",
          schema: "public",
          table: "comments",
          filter: `post_id=eq.${postId}`,
        },
        (payload) => {
          setComments((prev) => [...prev, payload.new as Comment]);
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, [postId, supabase]);

  return (
    <ul className="space-y-2">
      {comments.map((comment) => (
        <li key={comment.id} className="rounded border p-3">
          {comment.body}
        </li>
      ))}
    </ul>
  );
}
```

## The Composition Pattern (Best Practice)

Keep the page as a Server Component. Pass data DOWN to small Client Components.

```tsx
// app/posts/[slug]/page.tsx — SERVER Component (fetches data)
import { createClient } from "@/lib/supabase/server";
import { notFound } from "next/navigation";
import { LikeButton } from "@/app/components/like-button";
import { ShareButton } from "@/app/components/share-button";

export default async function PostPage({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;
  const supabase = await createClient();

  const { data: post } = await supabase
    .from("posts")
    .select("*")
    .eq("slug", slug)
    .single();

  if (!post) notFound();

  return (
    <article className="mx-auto max-w-2xl p-8">
      <h1 className="text-3xl font-bold">{post.title}</h1>
      <p className="mt-4">{post.body}</p>
      <div className="mt-6 flex gap-4">
        {/* Small Client Components for interactivity */}
        <LikeButton postId={post.id} initialCount={post.like_count} />
        <ShareButton url={`/posts/${slug}`} title={post.title} />
      </div>
    </article>
  );
}
```

```tsx
// app/components/like-button.tsx — CLIENT Component (handles clicks)
"use client";

import { useState } from "react";
import { createClient } from "@/lib/supabase/client";

export function LikeButton({
  postId,
  initialCount,
}: {
  postId: string;
  initialCount: number;
}) {
  const [count, setCount] = useState(initialCount);
  const [isLiked, setIsLiked] = useState(false);

  async function handleLike() {
    if (isLiked) return;
    setIsLiked(true);
    setCount((c) => c + 1);

    const supabase = createClient();
    const { error } = await supabase.rpc("increment_likes", { post_id: postId });

    if (error) {
      setIsLiked(false);
      setCount((c) => c - 1);
    }
  }

  return (
    <button
      onClick={handleLike}
      disabled={isLiked}
      className={`rounded px-4 py-2 ${
        isLiked ? "bg-pink-200 text-pink-700" : "bg-gray-100 hover:bg-gray-200"
      }`}
    >
      {isLiked ? "Liked" : "Like"} ({count})
    </button>
  );
}
```

## Common Mistakes

### Mistake 1: Making the whole page a Client Component

```tsx
// BAD — entire page is a Client Component, loses SSR benefits
"use client";

import { useEffect, useState } from "react";

export default function PostsPage() {
  const [posts, setPosts] = useState([]);

  useEffect(() => {
    fetch("/api/posts").then((r) => r.json()).then(setPosts);
  }, []);

  return <div>{/* ... */}</div>;
}
```

```tsx
// GOOD — Server Component page, only interactive parts are Client Components
import { createClient } from "@/lib/supabase/server";
import { SearchBar } from "@/app/components/search-bar"; // "use client"

export default async function PostsPage() {
  const supabase = await createClient();
  const { data: posts } = await supabase.from("posts").select("*");

  return (
    <div>
      <SearchBar />
      {posts?.map((post) => <div key={post.id}>{post.title}</div>)}
    </div>
  );
}
```

### Mistake 2: Importing a Server Component inside a Client Component

```tsx
// BAD — ServerChart will be forced to become a Client Component
"use client";
import { ServerChart } from "./server-chart"; // This won't work as expected

export function Dashboard() {
  return <ServerChart />;
}
```

```tsx
// GOOD — Pass Server Components as children
"use client";

export function DashboardShell({ children }: { children: React.ReactNode }) {
  const [sidebarOpen, setSidebarOpen] = useState(true);
  return (
    <div className="flex">
      <aside className={sidebarOpen ? "w-64" : "w-0"}>{/* sidebar */}</aside>
      <main>{children}</main> {/* Server Components can go here */}
    </div>
  );
}
```

### Mistake 3: Forgetting "use client" when using hooks

```
Error: useState only works in Client Components. Add the "use client" directive at the top of the file.
```

Fix: Add `"use client"` as the very first line of the file (before any imports).

### Mistake 4: Trying to use async in Client Components

```tsx
// BAD — Client Components cannot be async
"use client";

export default async function MyComponent() { // ERROR
  const data = await fetch("...");
}
```

```tsx
// GOOD — Use useEffect or server actions instead
"use client";

import { useEffect, useState } from "react";

export default function MyComponent() {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetch("/api/data")
      .then((r) => r.json())
      .then(setData)
      .catch(console.error);
  }, []);

  return <div>{/* render data */}</div>;
}
```
