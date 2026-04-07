# Top 20 TypeScript Errors with Plain English Fixes

## 1. "Property 'x' does not exist on type 'y'"

**What it means:** You are trying to use a property that TypeScript does not know about on that object.

**Common cause:** Typo in property name, or the object type is too broad.

```ts
// BAD
const user = { name: "Jane", email: "jane@test.com" };
console.log(user.nmae); // Typo: 'nmae' doesn't exist

// FIX — correct the typo
console.log(user.name);
```

```ts
// BAD — data from Supabase might be typed as generic
const { data } = await supabase.from("posts").select("*");
console.log(data.title); // Error: data is an array, not a single object

// FIX — access the array correctly, or use .single()
const { data } = await supabase.from("posts").select("*").single();
if (data) console.log(data.title);
```

## 2. "Type 'string | undefined' is not assignable to type 'string'"

**What it means:** Something might be `undefined`, but you are using it where only `string` is allowed.

```ts
// BAD
const name: string = searchParams.get("name"); // Could be null

// FIX — provide a fallback
const name: string = searchParams.get("name") ?? "default";

// Or check first
const rawName = searchParams.get("name");
if (!rawName) throw new Error("name is required");
// Now rawName is string (not null)
```

## 3. "Argument of type 'X' is not assignable to parameter of type 'Y'"

**What it means:** You are passing the wrong type to a function.

```ts
// BAD
function greet(name: string) { return `Hello, ${name}`; }
greet(123); // number is not string

// FIX — pass the correct type
greet("Jane");
greet(String(123)); // Convert if needed
```

## 4. "Object is possibly 'null'"

**What it means:** The value might be null, and you need to check before using it.

```ts
// BAD
const { data } = await supabase.from("posts").select("*").single();
console.log(data.title); // data could be null

// FIX — check for null
const { data, error } = await supabase.from("posts").select("*").single();
if (error || !data) {
  throw new Error("Post not found");
}
console.log(data.title); // Now safe
```

## 5. "Cannot find module '@/lib/supabase/server'"

**What it means:** TypeScript cannot find the file you are importing. The `@/` path alias might not be configured.

**Fix:** Make sure `tsconfig.json` has the path alias:

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./*"]
    }
  }
}
```

And make sure the file actually exists at `lib/supabase/server.ts`.

## 6. "Type 'Promise<{ slug: string }>' is not assignable to type '{ slug: string }'"

**What it means:** In newer Next.js versions, `params` is a Promise that needs to be awaited.

```ts
// BAD (old pattern)
export default function Page({ params }: { params: { slug: string } }) {
  return <h1>{params.slug}</h1>;
}

// FIX (current pattern)
export default async function Page({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;
  return <h1>{slug}</h1>;
}
```

## 7. "Type 'string' is not assignable to type '"active" | "inactive"'"

**What it means:** You have a union type but are passing a plain string.

```ts
type Status = "active" | "inactive";

// BAD
const status: Status = someVariable; // someVariable is typed as string

// FIX — validate first
function isStatus(value: string): value is Status {
  return value === "active" || value === "inactive";
}

if (isStatus(someVariable)) {
  const status: Status = someVariable; // Now safe
}

// Or use 'as' if you are sure
const status = someVariable as Status;
```

## 8. "Property 'x' is missing in type '{}' but required in type 'Y'"

**What it means:** You created an object but forgot a required property.

```ts
interface User {
  id: string;
  name: string;
  email: string;
}

// BAD — missing 'email'
const user: User = { id: "1", name: "Jane" };

// FIX — include all required fields
const user: User = { id: "1", name: "Jane", email: "jane@test.com" };

// Or make email optional in the interface
interface User {
  id: string;
  name: string;
  email?: string; // Now optional
}
```

## 9. "No overload matches this call"

**What it means:** You are calling a function with the wrong arguments. The function has specific ways it can be called, and none of them match what you wrote.

```ts
// BAD — addEventListener expects specific event types
document.addEventListener("clck", () => {}); // Typo

// FIX
document.addEventListener("click", () => {});
```

This also happens with Supabase methods when column names are wrong or filter values have the wrong type.

## 10. "'X' refers to a value, but is being used as a type here"

**What it means:** You are using a variable name where TypeScript expects a type name.

```ts
// BAD
const User = { name: "Jane" };
function greet(user: User) {} // User is a variable, not a type

// FIX — use interface/type for types, variables for values
interface User {
  name: string;
}
const user: User = { name: "Jane" };
```

## 11. "Type 'unknown' is not assignable to type 'X'"

**What it means:** The value has type `unknown` (could be anything), and you need to narrow it down before using it.

```ts
// BAD
function handleError(error: unknown) {
  console.log(error.message); // Can't access .message on unknown
}

// FIX — narrow the type first
function handleError(error: unknown) {
  if (error instanceof Error) {
    console.log(error.message); // Now TypeScript knows it's an Error
  } else {
    console.log(String(error));
  }
}
```

## 12. "Element implicitly has an 'any' type because expression of type 'string' can't be used to index type 'X'"

**What it means:** You are using a string to access an object property, but TypeScript does not know if that key exists.

```ts
// BAD
const colors = { red: "#ff0000", blue: "#0000ff" };
const key = "red"; // TypeScript sees this as type 'string'
const color = colors[key]; // Error

// FIX — type the key properly
const key: keyof typeof colors = "red";
const color = colors[key];

// Or use Record
const colors: Record<string, string> = { red: "#ff0000", blue: "#0000ff" };
const color = colors["red"]; // Works
```

## 13. "JSX element type 'X' does not have any construct or call signatures"

**What it means:** You are trying to use something as a React component, but it is not one.

```ts
// BAD — importing a type instead of a component
import { User } from "@/lib/types"; // This is an interface
return <User />; // Can't render a type as a component

// FIX — import the component
import { UserCard } from "@/components/user-card";
return <UserCard />;
```

## 14. "Type 'void' is not assignable to type 'X'"

**What it means:** A function returns nothing (void) but you are trying to use its return value.

```ts
// BAD
function logMessage(msg: string): void {
  console.log(msg);
}
const result: string = logMessage("hello"); // void is not string

// FIX — make the function return a value
function formatMessage(msg: string): string {
  return `[LOG] ${msg}`;
}
const result: string = formatMessage("hello");
```

## 15. "Unsafe assignment of an 'any' value"

**What it means:** (ESLint rule) You are assigning something typed as `any` to a typed variable. TypeScript cannot check the type.

```ts
// BAD
const data: any = await response.json();
const name: string = data.name; // Unsafe — data could be anything

// FIX — type the response
interface ApiResponse {
  name: string;
  email: string;
}
const data: ApiResponse = await response.json();
const name: string = data.name; // Safe

// Or validate with Zod
const schema = z.object({ name: z.string(), email: z.string() });
const data = schema.parse(await response.json());
```

## 16. "Cannot use import statement outside a module"

**What it means:** The file is not being treated as a module. Usually a config issue.

**Fix:** Make sure your `tsconfig.json` has:
```json
{
  "compilerOptions": {
    "module": "esnext",
    "moduleResolution": "bundler"
  }
}
```

## 17. "Type 'X[]' is not assignable to type 'readonly X[]'"

**What it means:** You are passing a mutable array where an immutable one is expected.

```ts
// FIX — use 'as const' or match the expected type
const items = ["a", "b", "c"] as const;
// Or
const items: readonly string[] = ["a", "b", "c"];
```

## 18. "Unused variable 'x'" / "Declared but never used"

**What it means:** You declared a variable or imported something but never used it.

```ts
// BAD
import { useState, useEffect } from "react"; // useEffect unused
const [count, setCount] = useState(0); // setCount unused

// FIX — remove unused imports and variables
import { useState } from "react";
const [count] = useState(0);

// Or prefix with _ to indicate intentionally unused
const [_count, setCount] = useState(0);
```

## 19. "The left-hand side of an arithmetic operation must be of type 'any', 'number', 'bigint' or an enum type"

**What it means:** You are trying to do math on a string.

```ts
// BAD
const price = searchParams.get("price"); // string | null
const total = price * 2; // Can't multiply a string

// FIX — parse the number first
const price = parseInt(searchParams.get("price") ?? "0", 10);
const total = price * 2; // Now it's a number
```

## 20. "Type instantiation is excessively deep and possibly infinite"

**What it means:** TypeScript is trying to figure out a type that is too complex or recursive.

**Common cause:** Complex Supabase query types with many joins.

```ts
// FIX — use .returns<T>() to manually specify the return type
interface PostWithRelations {
  id: string;
  title: string;
  author: { full_name: string } | null;
  comments: { id: string; body: string }[];
}

const { data } = await supabase
  .from("posts")
  .select("id, title, author:profiles(full_name), comments(id, body)")
  .returns<PostWithRelations[]>();
```

## Quick Reference: Type Error Cheat Sheet

| Error Pattern | Likely Fix |
|---|---|
| "possibly null" | Add `if (!x)` check or `?? fallback` |
| "possibly undefined" | Add `if (!x)` check or `?? fallback` |
| "not assignable to" | Check types match, add type assertion or conversion |
| "missing property" | Add the missing field to the object |
| "cannot find module" | Check import path and tsconfig paths |
| "no overload matches" | Check function arguments match expected types |
| "implicitly has 'any'" | Add explicit type annotation |
| "unknown" | Narrow with `instanceof`, `typeof`, or type guard |
