# Component & Hook Extraction Guide

Step-by-step patterns for extracting components and custom hooks from monolithic React files.

## When to Extract a Component

Extract when any of these are true:

- A section of JSX is **50+ lines** with its own visual boundary (card, section, modal)
- A section **manages its own state** (has useState/useEffect that doesn't affect siblings)
- A section is **reused** in 2+ places (or could be)
- A section has its own **loading or error state**
- The parent component is **hard to read** because of too many nested elements

**Don't extract** when:
- The JSX is under 20 lines and won't be reused
- Extracting would require passing 8+ props (sign of a wrong split boundary)
- The component would only be used in one place and has no independent state

## Component Extraction: Step-by-Step

### Step 1: Identify the Boundary

Look for natural boundaries in the JSX. Common ones:

```tsx
// These sections are natural extraction candidates:
return (
  <div>
    {/* === HEADER SECTION === */}
    <header>...</header>           {/* → HeaderSection */}

    {/* === FILTER BAR === */}
    <div className="filters">      {/* → FilterBar */}
      <input ... />
      <select ... />
    </div>

    {/* === DATA TABLE === */}
    <table>                        {/* → DataTable */}
      <thead>...</thead>
      <tbody>
        {items.map(item => (
          <tr key={item.id}>       {/* → DataTableRow */}
            <td>...</td>
          </tr>
        ))}
      </tbody>
    </table>

    {/* === PAGINATION === */}
    <nav>                          {/* → Pagination */}
      <button>Previous</button>
      <span>Page 1 of 10</span>
      <button>Next</button>
    </nav>
  </div>
);
```

### Step 2: Define the Props Interface

Before writing the component, define what data and callbacks it needs:

```typescript
// Ask: What does this section NEED from the parent?
// - Data it displays → props
// - Actions it triggers → callback props
// - Configuration → optional props with defaults

interface DataTableProps {
  // Data
  items: User[];
  isLoading: boolean;

  // Actions
  onRowClick: (id: string) => void;
  onDelete: (id: string) => void;

  // Configuration (optional with defaults)
  pageSize?: number;
  showActions?: boolean;
}
```

**Prop count rule:** If you need more than 6-7 props, consider:
1. Grouping related props into an object
2. Using React Context instead
3. Splitting at a different boundary

### Step 3: Extract the Component

```tsx
// components/users/data-table.tsx
"use client";

interface DataTableProps {
  items: User[];
  isLoading: boolean;
  onRowClick: (id: string) => void;
  onDelete: (id: string) => void;
}

export function DataTable({ items, isLoading, onRowClick, onDelete }: DataTableProps) {
  if (isLoading) {
    return <DataTableSkeleton />;
  }

  if (items.length === 0) {
    return (
      <div className="py-12 text-center text-gray-500">
        No users found.
      </div>
    );
  }

  return (
    <table className="w-full">
      <thead>
        <tr className="border-b text-left text-sm text-gray-500">
          <th className="pb-2">Name</th>
          <th className="pb-2">Email</th>
          <th className="pb-2">Role</th>
          <th className="pb-2" />
        </tr>
      </thead>
      <tbody>
        {items.map((item) => (
          <tr
            key={item.id}
            onClick={() => onRowClick(item.id)}
            className="cursor-pointer border-b hover:bg-gray-50"
          >
            <td className="py-3">{item.full_name}</td>
            <td className="py-3">{item.email}</td>
            <td className="py-3">{item.role}</td>
            <td className="py-3">
              <button
                onClick={(e) => {
                  e.stopPropagation();
                  onDelete(item.id);
                }}
                className="text-red-500 hover:text-red-700"
              >
                Delete
              </button>
            </td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}

function DataTableSkeleton() {
  return (
    <div className="space-y-3">
      {Array.from({ length: 5 }).map((_, i) => (
        <div key={i} className="h-12 animate-pulse rounded bg-gray-100" />
      ))}
    </div>
  );
}
```

### Step 4: Update the Parent

```tsx
// BEFORE (parent had 200 lines of table JSX)
export function UsersPage() {
  const { users, isLoading } = useUsers();
  // ... lots of table JSX here
}

// AFTER (parent delegates to DataTable)
import { DataTable } from "./data-table";

export function UsersPage() {
  const { users, isLoading } = useUsers();

  return (
    <div>
      <h1>Users</h1>
      <DataTable
        items={users}
        isLoading={isLoading}
        onRowClick={(id) => router.push(`/users/${id}`)}
        onDelete={handleDelete}
      />
    </div>
  );
}
```

## Custom Hook Extraction: Step-by-Step

### When to Extract a Hook

Extract a custom hook when:
- A component has **3+ useState** calls that work together
- A component has **useEffect** that fetches data
- The same **state + effect pattern** appears in 2+ components
- Business logic is **mixed into** the component rendering

### Step 1: Identify the State Cluster

Look for groups of state that change together:

```tsx
// These 6 lines form a cluster — they all relate to "user search"
const [users, setUsers] = useState<User[]>([]);
const [search, setSearch] = useState("");
const [isLoading, setIsLoading] = useState(true);
const [error, setError] = useState<string | null>(null);
const [page, setPage] = useState(1);
const [totalPages, setTotalPages] = useState(1);

// This effect uses the cluster
useEffect(() => {
  async function fetchUsers() {
    setIsLoading(true);
    setError(null);
    const { data, error, count } = await supabase
      .from("profiles")
      .select("*", { count: "exact" })
      .ilike("full_name", `%${search}%`)
      .range((page - 1) * 20, page * 20 - 1);

    if (error) {
      setError(error.message);
    } else {
      setUsers(data ?? []);
      setTotalPages(Math.ceil((count ?? 0) / 20));
    }
    setIsLoading(false);
  }
  fetchUsers();
}, [search, page]);
```

### Step 2: Define the Hook's Return Type

```typescript
interface UseUsersReturn {
  // Data
  users: User[];
  totalPages: number;
  isLoading: boolean;
  error: string | null;

  // State setters the component needs
  search: string;
  setSearch: (search: string) => void;
  page: number;
  setPage: (page: number) => void;

  // Actions
  refresh: () => void;
  deleteUser: (id: string) => Promise<void>;
}
```

### Step 3: Extract the Hook

```typescript
// hooks/use-users.ts
"use client";
import { useState, useEffect, useCallback } from "react";
import { createClient } from "@/lib/supabase/client";
import type { User } from "@/types/user.types";

const PAGE_SIZE = 20;

interface UseUsersOptions {
  initialSearch?: string;
  pageSize?: number;
}

export function useUsers(options: UseUsersOptions = {}) {
  const { initialSearch = "", pageSize = PAGE_SIZE } = options;
  const [users, setUsers] = useState<User[]>([]);
  const [search, setSearch] = useState(initialSearch);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const [page, setPage] = useState(1);
  const [totalPages, setTotalPages] = useState(1);

  const fetchUsers = useCallback(async () => {
    const supabase = createClient();
    setIsLoading(true);
    setError(null);

    const from = (page - 1) * pageSize;
    const to = from + pageSize - 1;

    const { data, error: fetchError, count } = await supabase
      .from("profiles")
      .select("*", { count: "exact" })
      .ilike("full_name", `%${search}%`)
      .range(from, to);

    if (fetchError) {
      setError(fetchError.message);
      setUsers([]);
    } else {
      setUsers(data ?? []);
      setTotalPages(Math.ceil((count ?? 0) / pageSize));
    }
    setIsLoading(false);
  }, [search, page, pageSize]);

  useEffect(() => {
    fetchUsers();
  }, [fetchUsers]);

  // Reset to page 1 when search changes
  useEffect(() => {
    setPage(1);
  }, [search]);

  const deleteUser = useCallback(async (id: string) => {
    const supabase = createClient();
    const { error: deleteError } = await supabase.from("profiles").delete().eq("id", id);
    if (deleteError) {
      setError(deleteError.message);
      return;
    }
    // Refresh the list after deletion
    await fetchUsers();
  }, [fetchUsers]);

  return {
    users,
    totalPages,
    isLoading,
    error,
    search,
    setSearch,
    page,
    setPage,
    refresh: fetchUsers,
    deleteUser,
  };
}
```

### Step 4: Use the Hook in the Component

```tsx
// components/users-page.tsx
"use client";
import { useUsers } from "@/hooks/use-users";
import { DataTable } from "./users/data-table";
import { SearchBar } from "./users/search-bar";
import { Pagination } from "./users/pagination";
import { ErrorAlert } from "./ui/error-alert";

export function UsersPage() {
  const {
    users,
    isLoading,
    error,
    search,
    setSearch,
    page,
    setPage,
    totalPages,
    deleteUser,
  } = useUsers();

  return (
    <div className="space-y-4">
      <h1 className="text-2xl font-bold">Users</h1>
      <SearchBar value={search} onChange={setSearch} />
      {error && <ErrorAlert message={error} />}
      <DataTable items={users} isLoading={isLoading} onDelete={deleteUser} />
      <Pagination page={page} totalPages={totalPages} onPageChange={setPage} />
    </div>
  );
}
```

## Extracting Form Components

Forms are the most common extraction target. Here's the pattern:

### Before: 300-line Component with Inline Form

```tsx
// One massive component with form state, validation, submission, and JSX
export function CreateUserPage() {
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");
  const [role, setRole] = useState("user");
  const [errors, setErrors] = useState({});
  const [isSubmitting, setIsSubmitting] = useState(false);
  // ... 50 more lines of state and handlers
  // ... 200 lines of form JSX
}
```

### After: Form Component + Hook + Validation Schema

```typescript
// lib/schemas/user.schema.ts
import { z } from "zod";

export const createUserSchema = z.object({
  fullName: z.string().min(2, "Name is required"),
  email: z.string().email("Invalid email"),
  role: z.enum(["user", "admin"]),
});

export type CreateUserInput = z.infer<typeof createUserSchema>;
```

```typescript
// hooks/use-create-user-form.ts
"use client";
import { useState } from "react";
import { createUserSchema, type CreateUserInput } from "@/lib/schemas/user.schema";

export function useCreateUserForm() {
  const [errors, setErrors] = useState<Record<string, string[]>>({});
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [submitError, setSubmitError] = useState<string | null>(null);

  async function handleSubmit(formData: FormData) {
    setErrors({});
    setSubmitError(null);

    const raw = {
      fullName: formData.get("fullName") as string,
      email: formData.get("email") as string,
      role: formData.get("role") as string,
    };

    const result = createUserSchema.safeParse(raw);
    if (!result.success) {
      setErrors(result.error.flatten().fieldErrors as Record<string, string[]>);
      return;
    }

    setIsSubmitting(true);
    try {
      const response = await fetch("/api/users", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(result.data),
      });

      if (!response.ok) {
        const body = await response.json();
        setSubmitError(body.error || "Failed to create user");
        return;
      }

      // Success — redirect or show message
      window.location.href = "/users";
    } catch {
      setSubmitError("Network error. Please try again.");
    } finally {
      setIsSubmitting(false);
    }
  }

  return { errors, isSubmitting, submitError, handleSubmit };
}
```

```tsx
// components/users/create-user-form.tsx
"use client";
import { useCreateUserForm } from "@/hooks/use-create-user-form";

export function CreateUserForm() {
  const { errors, isSubmitting, submitError, handleSubmit } = useCreateUserForm();

  return (
    <form action={handleSubmit} className="space-y-4">
      {submitError && (
        <div className="rounded bg-red-50 p-3 text-red-700">{submitError}</div>
      )}

      <div>
        <label htmlFor="fullName" className="block text-sm font-medium">
          Full Name
        </label>
        <input
          id="fullName"
          name="fullName"
          type="text"
          className="mt-1 w-full rounded border px-3 py-2"
        />
        {errors.fullName && (
          <p className="mt-1 text-sm text-red-500">{errors.fullName[0]}</p>
        )}
      </div>

      <div>
        <label htmlFor="email" className="block text-sm font-medium">
          Email
        </label>
        <input
          id="email"
          name="email"
          type="email"
          className="mt-1 w-full rounded border px-3 py-2"
        />
        {errors.email && (
          <p className="mt-1 text-sm text-red-500">{errors.email[0]}</p>
        )}
      </div>

      <div>
        <label htmlFor="role" className="block text-sm font-medium">
          Role
        </label>
        <select
          id="role"
          name="role"
          className="mt-1 w-full rounded border px-3 py-2"
        >
          <option value="user">User</option>
          <option value="admin">Admin</option>
        </select>
      </div>

      <button
        type="submit"
        disabled={isSubmitting}
        className="rounded bg-blue-600 px-4 py-2 text-white hover:bg-blue-700 disabled:opacity-50"
      >
        {isSubmitting ? "Creating..." : "Create User"}
      </button>
    </form>
  );
}
```

## Extracting Modal Components

Modals are another common extraction. Keep the trigger and the modal separate:

```tsx
// components/ui/confirm-modal.tsx
"use client";

interface ConfirmModalProps {
  isOpen: boolean;
  title: string;
  message: string;
  confirmLabel?: string;
  cancelLabel?: string;
  isDestructive?: boolean;
  isLoading?: boolean;
  onConfirm: () => void;
  onCancel: () => void;
}

export function ConfirmModal({
  isOpen,
  title,
  message,
  confirmLabel = "Confirm",
  cancelLabel = "Cancel",
  isDestructive = false,
  isLoading = false,
  onConfirm,
  onCancel,
}: ConfirmModalProps) {
  if (!isOpen) return null;

  return (
    <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/50">
      <div className="w-full max-w-md rounded-lg bg-white p-6 shadow-xl">
        <h2 className="text-lg font-semibold">{title}</h2>
        <p className="mt-2 text-gray-600">{message}</p>
        <div className="mt-4 flex justify-end gap-3">
          <button
            onClick={onCancel}
            disabled={isLoading}
            className="rounded px-4 py-2 text-gray-700 hover:bg-gray-100"
          >
            {cancelLabel}
          </button>
          <button
            onClick={onConfirm}
            disabled={isLoading}
            className={`rounded px-4 py-2 text-white ${
              isDestructive
                ? "bg-red-600 hover:bg-red-700"
                : "bg-blue-600 hover:bg-blue-700"
            } disabled:opacity-50`}
          >
            {isLoading ? "..." : confirmLabel}
          </button>
        </div>
      </div>
    </div>
  );
}
```

Usage in parent:

```tsx
const [showDeleteModal, setShowDeleteModal] = useState(false);
const [deletingId, setDeletingId] = useState<string | null>(null);

<ConfirmModal
  isOpen={showDeleteModal}
  title="Delete User"
  message="Are you sure? This action cannot be undone."
  confirmLabel="Delete"
  isDestructive
  isLoading={isDeleting}
  onConfirm={() => deletingId && handleDelete(deletingId)}
  onCancel={() => setShowDeleteModal(false)}
/>
```

## Server vs Client Component Extraction

In Next.js App Router, be intentional about the client boundary:

```
page.tsx (Server Component — fetches data)
  └── client-section.tsx (Client Component — handles interactivity)
        ├── search-bar.tsx (Client — user input)
        ├── data-table.tsx (Client — click handlers)
        └── pagination.tsx (Client — page state)
```

**Rules:**
1. Keep data fetching in Server Components (pages, layouts)
2. Push `"use client"` as low as possible in the component tree
3. Pass server-fetched data down as props to client components
4. Never import a Server Component inside a Client Component

```tsx
// app/users/page.tsx (Server Component — NO "use client")
import { createClient } from "@/lib/supabase/server";
import { UsersClient } from "./users-client";

export default async function UsersPage() {
  const supabase = await createClient();
  const { data: users } = await supabase.from("profiles").select("*").limit(20);

  return (
    <div>
      <h1 className="text-2xl font-bold">Users</h1>
      <UsersClient initialUsers={users ?? []} />
    </div>
  );
}
```

```tsx
// app/users/users-client.tsx (Client Component)
"use client";
import { useState } from "react";
import type { User } from "@/types/user.types";

interface UsersClientProps {
  initialUsers: User[];
}

export function UsersClient({ initialUsers }: UsersClientProps) {
  const [users] = useState(initialUsers);
  const [search, setSearch] = useState("");

  const filtered = users.filter((u) =>
    u.full_name.toLowerCase().includes(search.toLowerCase())
  );

  return (
    <>
      <input
        value={search}
        onChange={(e) => setSearch(e.target.value)}
        placeholder="Search users..."
        className="mt-4 w-full rounded border px-3 py-2"
      />
      <ul className="mt-4 space-y-2">
        {filtered.map((user) => (
          <li key={user.id} className="rounded border p-3">
            {user.full_name} — {user.email}
          </li>
        ))}
      </ul>
    </>
  );
}
```
