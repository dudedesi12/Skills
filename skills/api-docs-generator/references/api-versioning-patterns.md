# API Versioning Patterns

## URL-Based Versioning (Recommended)

The simplest approach. Version is in the URL path.

```
/api/v1/users     ← version 1
/api/v2/users     ← version 2
```

### Directory Structure

```
app/api/
  v1/
    users/route.ts
    assessments/route.ts
  v2/
    users/route.ts          ← new version
    assessments/route.ts    ← can re-export v1 if unchanged
```

### Re-exporting Unchanged Routes

```typescript
// app/api/v2/assessments/route.ts
// No changes in v2 — just re-export v1
export { GET, POST } from "../../v1/assessments/route";
```

### Shared Logic Between Versions

```typescript
// lib/services/user-service.ts — shared business logic
export class UserService {
  async getAll() { /* ... */ }
  async getById(id: string) { /* ... */ }
}

// app/api/v1/users/route.ts — v1 response shape
import { UserService } from "@/lib/services/user-service";

export async function GET() {
  const service = new UserService(supabase);
  const users = await service.getAll();
  return NextResponse.json(users); // v1 shape
}

// app/api/v2/users/route.ts — v2 response shape (added pagination)
import { UserService } from "@/lib/services/user-service";

export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url);
  const page = Number(searchParams.get("page") ?? 1);
  const limit = Number(searchParams.get("limit") ?? 20);

  const service = new UserService(supabase);
  const users = await service.getAll();

  return NextResponse.json({
    data: users.slice((page - 1) * limit, page * limit),
    meta: { page, limit, total: users.length },
  }); // v2 shape — wrapped with pagination
}
```

## Header-Based Versioning

Version specified via request header. URLs stay clean but harder to test.

```typescript
// middleware.ts — route to correct version based on header
import { NextResponse, type NextRequest } from "next/server";

export function middleware(request: NextRequest) {
  if (!request.nextUrl.pathname.startsWith("/api/")) return NextResponse.next();

  const version = request.headers.get("Accept-Version") ?? "1";

  // Rewrite /api/users → /api/v{version}/users
  const newPath = request.nextUrl.pathname.replace("/api/", `/api/v${version}/`);
  return NextResponse.rewrite(new URL(newPath, request.url));
}
```

## Deprecation Headers

When sunsetting an old API version:

```typescript
// lib/api/deprecation.ts
import { NextResponse } from "next/server";

export function withDeprecation(
  response: NextResponse,
  sunsetDate: string,
  newVersionUrl: string
): NextResponse {
  response.headers.set("Deprecation", "true");
  response.headers.set("Sunset", sunsetDate); // RFC 7231 date
  response.headers.set("Link", `<${newVersionUrl}>; rel="successor-version"`);
  return response;
}

// Usage in v1 route after v2 is released:
export async function GET() {
  const data = await getUsers();
  const response = NextResponse.json(data);
  return withDeprecation(
    response,
    "Sat, 01 Jan 2027 00:00:00 GMT",
    "/api/v2/users"
  );
}
```

## Breaking vs Non-Breaking Changes

**Non-breaking (no new version needed):**
- Adding a new optional field to a response
- Adding a new endpoint
- Adding a new optional query parameter
- Improving error messages

**Breaking (new version required):**
- Removing or renaming a field
- Changing a field's type
- Changing the response structure (e.g., wrapping in `{ data: ... }`)
- Removing an endpoint
- Making a previously optional parameter required
- Changing authentication method

## Changelog Template

```markdown
# API Changelog

## v2 (2026-04-01)

### Breaking Changes
- **GET /api/v2/users**: Response is now paginated. Previously returned a flat array.
  - Old: `[{ id, name, email }]`
  - New: `{ data: [{ id, name, email }], meta: { page, limit, total } }`

### New Endpoints
- **POST /api/v2/assessments/batch**: Submit multiple assessments at once.

### Deprecations
- **v1 sunset date: January 1, 2027.** All v1 endpoints now return `Deprecation: true` header.

---

## v1 (2025-06-01)

### Initial Release
- User CRUD: GET, POST, PATCH, DELETE /api/v1/users
- Assessments: GET, POST /api/v1/assessments
- Auth: POST /api/v1/auth/login, POST /api/v1/auth/signup
```
