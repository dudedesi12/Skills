# TypeScript Errors (40+)

Every common TypeScript error with plain English explanation and exact fix.

## Type Assignment Errors

### 1. Type 'string' is not assignable to type 'number'

**Why:** You passed a string where a number was expected.

**Fix:**
```typescript
// Wrong
const count: number = "5";
// Right
const count: number = 5;
// Or convert
const count: number = parseInt("5", 10);
```

### 2. Type 'X' is not assignable to type 'Y'

**Why:** Two types look similar but have different shapes.

**Fix:** Compare the types. Add missing properties or remove extras:
```typescript
interface User { name: string; email: string; }
// Wrong — missing email
const user: User = { name: "Ali" };
// Right
const user: User = { name: "Ali", email: "ali@example.com" };
```

### 3. Type 'string | undefined' is not assignable to type 'string'

**Why:** The value might be undefined, but you are treating it as always defined.

**Fix:**
```typescript
// Option 1: Non-null assertion (only if you are SURE it exists)
const name = user.name!;
// Option 2: Default value (safer)
const name = user.name ?? "Anonymous";
// Option 3: Guard check
if (user.name) {
  doSomething(user.name); // TypeScript knows it's string here
}
```

### 4. Type 'null' is not assignable to type 'X'

**Why:** A value could be null but the type does not allow it.

**Fix:**
```typescript
// Allow null in the type
const [user, setUser] = useState<User | null>(null);
// Or provide a default
const [user, setUser] = useState<User>({ name: "", email: "" });
```

### 5. Type 'X[]' is not assignable to type 'readonly X[]'

**Why:** Passing a mutable array where a readonly one is expected.

**Fix:**
```typescript
const items: readonly string[] = ["a", "b"] as const;
// Or use the satisfies keyword
const items = ["a", "b"] as const satisfies readonly string[];
```

### 6. Argument of type 'string' is not assignable to parameter of type '"active" | "inactive"'

**Why:** A literal union type expects specific strings, not just any string.

**Fix:**
```typescript
// Wrong
const status: string = "active";
updateStatus(status); // Error
// Right
const status = "active" as const;
updateStatus(status); // Works
// Or type the variable
const status: "active" | "inactive" = "active";
```

## Property Errors

### 7. Property 'X' does not exist on type 'Y'

**Why:** You are accessing a property that TypeScript does not know about.

**Fix:**
```typescript
// Add it to the interface
interface User {
  name: string;
  email: string;
  avatar?: string; // Add the missing property
}
```

### 8. Property 'X' is optional in type 'Y' but required in type 'Z'

**Why:** One type has the property as optional (`?`) but another requires it.

**Fix:** Make it required in both or optional in both, or provide a default.

### 9. Object literal may only specify known properties

**Why:** You added a property that does not exist on the target type.

**Fix:** Check for typos in property names. Remove extra properties or add them to the type.

### 10. Property 'X' has no initializer and is not assigned in the constructor

**Why:** A class property is declared but never given a value.

**Fix:**
```typescript
class User {
  name: string = ""; // Add default value
  // Or use the ! operator if you set it elsewhere
  email!: string;
}
```

## Function Errors

### 11. Expected X arguments, but got Y

**Why:** You called a function with the wrong number of arguments.

**Fix:** Check the function signature and pass all required parameters.

### 12. Parameter 'X' implicitly has an 'any' type

**Why:** TypeScript strict mode requires explicit types on function parameters.

**Fix:**
```typescript
// Wrong
function greet(name) { return `Hello ${name}`; }
// Right
function greet(name: string) { return `Hello ${name}`; }
```

### 13. Not all code paths return a value

**Why:** A function with a return type might not return in every case.

**Fix:**
```typescript
function getStatus(code: number): string {
  if (code === 200) return "OK";
  if (code === 404) return "Not Found";
  return "Unknown"; // Add a catch-all return
}
```

### 14. A function whose declared type is neither 'void' nor 'any' must return a value

**Why:** The function says it returns something but has no return statement.

**Fix:** Add a return statement, or change the return type to `void`.

### 15. Void function return value is used

**Why:** You tried to use the return value of a function that returns nothing.

**Fix:** The function does not return anything useful. Remove the assignment.

## Generic Errors

### 16. Type 'X' does not satisfy the constraint 'Y'

**Why:** A generic type parameter does not meet the required constraint.

**Fix:**
```typescript
// The constraint
function getProperty<T extends { id: string }>(obj: T) { return obj.id; }
// Wrong — number has no 'id'
getProperty(42);
// Right — object has 'id'
getProperty({ id: "abc", name: "Ali" });
```

### 17. Generic type 'X' requires Y type argument(s)

**Why:** You used a generic type without specifying the type parameter.

**Fix:**
```typescript
// Wrong
const result: Array = [];
// Right
const result: Array<string> = [];
// Or shorthand
const result: string[] = [];
```

### 18. Type parameter 'T' has a circular constraint

**Why:** A type references itself in a way that creates an infinite loop.

**Fix:** Break the cycle by restructuring the type constraints.

## Module Errors

### 19. Cannot find module 'X' or its corresponding type declarations

**Why:** The package is not installed, or types are missing.

**Fix:**
```bash
# Install the package
npm install package-name
# If types are separate
npm install -D @types/package-name
```

### 20. Could not find a declaration file for module 'X'

**Why:** The package exists but has no TypeScript types.

**Fix:**
```typescript
// Option 1: Install types
npm install -D @types/package-name
// Option 2: Declare the module (create a .d.ts file)
// types/package-name.d.ts
declare module "package-name";
```

### 21. Module has no default export

**Why:** You used `import X` but the module uses named exports.

**Fix:**
```typescript
// Wrong
import Button from "./Button";
// Right
import { Button } from "./Button";
```

### 22. Module has no exported member 'X'

**Why:** The named export does not exist in that module.

**Fix:** Check what the module actually exports. Might be renamed or removed in a newer version.

### 23. Cannot use import statement outside a module

**Why:** The file is being treated as a CommonJS file, not an ES module.

**Fix:** Make sure `"type": "module"` is in `package.json` or use `.mts` extension.

## React + TypeScript Errors

### 24. 'X' cannot be used as a JSX component

**Why:** The component does not return a valid React element type.

**Fix:**
```typescript
// Make sure the component returns JSX
export default function Card(): React.ReactNode {
  return <div>Content</div>;
}
```

### 25. Type 'X' is not assignable to type 'IntrinsicAttributes & Props'

**Why:** You passed a prop that the component does not accept.

**Fix:** Check the component's prop types and pass only what it expects.

### 26. Type 'Element[]' is not assignable to type 'ReactNode'

**Why:** Returning an array of elements where a single node is expected.

**Fix:** Wrap in a fragment:
```typescript
return <>{items.map((item) => <div key={item.id}>{item.name}</div>)}</>;
```

### 27. Property 'children' does not exist on type 'X'

**Why:** Your component does not accept children in its props type.

**Fix:**
```typescript
interface Props {
  title: string;
  children: React.ReactNode; // Add children
}
```

### 28. Type 'EventTarget' is missing properties

**Why:** Event types in React are more specific than plain DOM types.

**Fix:**
```typescript
function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
  const value = e.target.value; // TypeScript now knows this is an input
}
```

## Utility Type Errors

### 29. Type 'Partial<X>' makes all properties optional

**Why:** Using Partial when you need some properties to remain required.

**Fix:**
```typescript
// Only make some optional
type UpdateUser = Pick<User, "id"> & Partial<Omit<User, "id">>;
```

### 30. Cannot assign to 'X' because it is a read-only property

**Why:** The property is marked as `readonly` and cannot be changed.

**Fix:** Create a new object instead of mutating:
```typescript
const updated = { ...original, name: "New Name" };
```

## Async Errors

### 31. Property 'then' does not exist on type 'X'

**Why:** You are trying to await something that is not a Promise.

**Fix:** Check if the function is actually async. Remove the `await` if it is not.

### 32. Type 'Promise<X>' is not assignable to type 'X'

**Why:** You forgot to `await` the promise.

**Fix:**
```typescript
// Wrong
const user: User = getUser(); // Returns Promise<User>
// Right
const user: User = await getUser();
```

### 33. 'await' expressions are only allowed within async functions

**Why:** You used `await` in a non-async function.

**Fix:** Add `async` to the function declaration.

## Enum Errors

### 34. Enum member must have initializer

**Why:** String enums require values for each member.

**Fix:**
```typescript
enum Status {
  Active = "active",
  Inactive = "inactive",
}
```

### 35. Type 'string' is not assignable to type 'Status'

**Why:** A plain string is not automatically a valid enum value.

**Fix:**
```typescript
const status = "active" as Status;
// Or use a type guard
function isStatus(value: string): value is Status {
  return Object.values(Status).includes(value as Status);
}
```

## Config Errors

### 36. Cannot find name 'process'

**Why:** Node.js types are not installed.

**Fix:**
```bash
npm install -D @types/node
```

### 37. Include/exclude paths in tsconfig.json not working

**Why:** Path patterns need to match the actual file structure.

**Fix:**
```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["./*"] }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

### 38. Duplicate identifier 'X'

**Why:** The same name is declared in multiple places.

**Fix:** Check for conflicting imports or multiple type declarations. Use namespaces or rename one.

## Narrowing Errors

### 39. Object is possibly 'null' or 'undefined'

**Why:** TypeScript is not sure the value exists.

**Fix:**
```typescript
// Narrow with a check
if (user) {
  console.log(user.name); // Safe
}
// Or optional chaining
console.log(user?.name ?? "Unknown");
```

### 40. Type 'X | Y' is not assignable — need to narrow

**Why:** A union type needs to be narrowed before you can use type-specific properties.

**Fix:**
```typescript
function handle(input: string | number) {
  if (typeof input === "string") {
    return input.toUpperCase(); // Safe — TypeScript knows it's string
  }
  return input.toFixed(2); // Safe — TypeScript knows it's number
}
```

### 41. This expression is not callable — no call signatures

**Why:** You are trying to call something that is not a function.

**Fix:** Check the type. You might have a value where you expected a function.
