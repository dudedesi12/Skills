---
name: api-docs-generator
description: "Use this skill whenever the user mentions API documentation, OpenAPI, Swagger, API spec, API reference, documenting endpoints, auto-generate docs, API playground, API explorer, 'document my API', 'API docs page', REST documentation, route documentation, endpoint documentation, zod-to-openapi, API versioning, API changelog, 'how do I document my API', or ANY task related to creating or maintaining API documentation — even if they don't explicitly say 'docs'. This skill auto-generates beautiful API documentation from your existing code."
---

# API Docs Generator

Generate beautiful, auto-updating API documentation from your Next.js + TypeScript + Supabase codebase. Uses your existing Zod schemas as the single source of truth so your docs never drift from your actual API.

## Stack Context

This skill is built for projects using:

- **Next.js 14+** with the App Router (`app/api/` route handlers)
- **TypeScript** for type safety
- **Zod** for runtime validation (already in your project)
- **Supabase** for auth and database
- **Tailwind CSS** for styling the docs page
- **Vercel** for deployment

The core idea: you already write Zod schemas to validate your API inputs. With `zod-to-openapi`, those same schemas become your OpenAPI spec, which powers a Swagger UI page. One source of truth, zero manual doc writing.

## OpenAPI Spec from Zod Schemas

The `zod-to-openapi` library lets you take any Zod schema and turn it into an OpenAPI 3.x component. Here is the full setup.

### Install the packages

```bash
npm install @asteasolutions/zod-to-openapi
```

You already have `zod` installed. That is the only new dependency.

### Create the OpenAPI registry

The registry is where you register all your schemas and route definitions. Create a single file that your whole app imports from:

```typescript
// lib/openapi/registry.ts
import {
  OpenAPIRegistry,
  OpenApiGeneratorV3,
} from "@asteasolutions/zod-to-openapi";
import { z } from "zod";

// One registry for the whole app
export const registry = new OpenAPIRegistry();

// Helper to generate the full spec from whatever has been registered
export function generateOpenAPISpec() {
  const generator = new OpenApiGeneratorV3(registry.definitions);

  return generator.generateDocument({
    openapi: "3.0.3",
    info: {
      title: "My App API",
      version: "1.0.0",
      description:
        "Auto-generated API documentation. Every schema here is derived from the Zod validators used in production code.",
    },
    servers: [
      {
        url: process.env.NEXT_PUBLIC_APP_URL || "http://localhost:3000",
        description: "Current environment",
      },
    ],
    security: [
      {
        BearerAuth: [],
      },
    ],
  });
}
```

### Register Zod schemas as OpenAPI components

Take your existing Zod schemas and register them. The key method is `registry.register()`:

```typescript
// lib/schemas/user.ts
import { z } from "zod";
import { registry } from "@/lib/openapi/registry";

// Your existing Zod schema — no changes needed
export const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  name: z.string().min(1).max(100),
  role: z.enum(["admin", "member", "viewer"]),
  created_at: z.string().datetime(),
});

// Register it so it appears in the OpenAPI spec
registry.register("User", UserSchema);

// Input schema for creating a user
export const CreateUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
  role: z.enum(["admin", "member", "viewer"]).default("member"),
});

registry.register("CreateUser", CreateUserSchema);

// Input schema for updating a user
export const UpdateUserSchema = CreateUserSchema.partial();

registry.register("UpdateUser", UpdateUserSchema);

// Derive TypeScript types from Zod (you probably already do this)
export type User = z.infer<typeof UserSchema>;
export type CreateUser = z.infer<typeof CreateUserSchema>;
export type UpdateUser = z.infer<typeof UpdateUserSchema>;
```

### Register route definitions

Each API route gets a `registerPath` call that describes the method, path, request body, query params, and response shapes:

```typescript
// lib/openapi/routes/users.ts
import { registry } from "@/lib/openapi/registry";
import { UserSchema, CreateUserSchema, UpdateUserSchema } from "@/lib/schemas/user";
import { z } from "zod";

// GET /api/users — list all users
registry.registerPath({
  method: "get",
  path: "/api/users",
  summary: "List users",
  description: "Returns a paginated list of users. Requires authentication.",
  tags: ["Users"],
  request: {
    query: z.object({
      page: z.coerce.number().int().min(1).default(1).openapi({
        description: "Page number (1-indexed)",
        example: 1,
      }),
      limit: z.coerce.number().int().min(1).max(100).default(20).openapi({
        description: "Items per page",
        example: 20,
      }),
      role: z.enum(["admin", "member", "viewer"]).optional().openapi({
        description: "Filter by role",
      }),
    }),
  },
  responses: {
    200: {
      description: "Paginated list of users",
      content: {
        "application/json": {
          schema: z.object({
            data: z.array(UserSchema),
            total: z.number(),
            page: z.number(),
            limit: z.number(),
          }),
        },
      },
    },
    401: {
      description: "Missing or invalid authentication token",
    },
  },
});

// POST /api/users — create a user
registry.registerPath({
  method: "post",
  path: "/api/users",
  summary: "Create a user",
  description: "Creates a new user account. Requires admin role.",
  tags: ["Users"],
  request: {
    body: {
      content: {
        "application/json": {
          schema: CreateUserSchema,
        },
      },
    },
  },
  responses: {
    201: {
      description: "User created successfully",
      content: {
        "application/json": {
          schema: UserSchema,
        },
      },
    },
    400: {
      description: "Validation error — check the request body",
    },
    401: {
      description: "Missing or invalid authentication token",
    },
    403: {
      description: "Only admins can create users",
    },
  },
});

// PATCH /api/users/[id] — update a user
registry.registerPath({
  method: "patch",
  path: "/api/users/{id}",
  summary: "Update a user",
  description: "Partially updates a user. Send only the fields you want to change.",
  tags: ["Users"],
  request: {
    params: z.object({
      id: z.string().uuid().openapi({ description: "User ID" }),
    }),
    body: {
      content: {
        "application/json": {
          schema: UpdateUserSchema,
        },
      },
    },
  },
  responses: {
    200: {
      description: "Updated user",
      content: {
        "application/json": {
          schema: UserSchema,
        },
      },
    },
    404: {
      description: "User not found",
    },
  },
});
```

### Serve the spec as a JSON endpoint

Create an API route that returns the generated OpenAPI JSON. Swagger UI will fetch from this endpoint:

```typescript
// app/api/openapi/route.ts
import { generateOpenAPISpec } from "@/lib/openapi/registry";

// Import all route registration files so they execute
import "@/lib/openapi/routes/users";
// import "@/lib/openapi/routes/posts";
// import "@/lib/openapi/routes/billing";

export async function GET() {
  const spec = generateOpenAPISpec();
  return Response.json(spec);
}
```

Hit `http://localhost:3000/api/openapi` and you get the full OpenAPI 3.0 JSON spec. That is the entire pipeline: Zod schemas -> OpenAPI spec -> JSON endpoint.

## Auto-Generating Route Documentation

If you have many routes and want to scan them automatically rather than writing registration files by hand, you can create a script that discovers routes from `app/api/`:

```typescript
// scripts/scan-api-routes.ts
import fs from "fs";
import path from "path";

interface RouteInfo {
  filePath: string;
  apiPath: string;
  methods: string[];
}

const API_DIR = path.join(process.cwd(), "app", "api");
const HTTP_METHODS = ["GET", "POST", "PUT", "PATCH", "DELETE", "HEAD", "OPTIONS"];

function scanRoutes(dir: string, basePath: string = "/api"): RouteInfo[] {
  const routes: RouteInfo[] = [];

  if (!fs.existsSync(dir)) return routes;

  const entries = fs.readdirSync(dir, { withFileTypes: true });

  for (const entry of entries) {
    const fullPath = path.join(dir, entry.name);

    if (entry.isDirectory()) {
      // Convert [param] to {param} for OpenAPI
      const segment = entry.name.replace(/\[([^\]]+)\]/g, "{$1}");
      routes.push(...scanRoutes(fullPath, `${basePath}/${segment}`));
    }

    if (entry.name === "route.ts" || entry.name === "route.js") {
      const content = fs.readFileSync(fullPath, "utf-8");
      const methods = HTTP_METHODS.filter((method) => {
        // Match "export async function GET" or "export function GET"
        const regex = new RegExp(`export\\s+(async\\s+)?function\\s+${method}\\b`);
        return regex.test(content);
      });

      if (methods.length > 0) {
        routes.push({
          filePath: fullPath,
          apiPath: basePath,
          methods,
        });
      }
    }
  }

  return routes;
}

const routes = scanRoutes(API_DIR);

console.log("\nDiscovered API Routes:\n");
for (const route of routes) {
  for (const method of route.methods) {
    console.log(`  ${method.padEnd(7)} ${route.apiPath}`);
  }
}
console.log(`\nTotal: ${routes.reduce((sum, r) => sum + r.methods.length, 0)} endpoints`);
```

Run it with:

```bash
npx tsx scripts/scan-api-routes.ts
```

This gives you a quick inventory. For each discovered route, you still want to write the `registerPath` call with proper schemas — automation finds the routes, but you describe the contracts.

## Swagger UI Integration

Swagger UI gives you an interactive docs page where developers can read your API docs and test endpoints directly in the browser.

### Install swagger-ui-react

```bash
npm install swagger-ui-react
npm install -D @types/swagger-ui-react
```

### Create the docs page

```typescript
// app/api-docs/page.tsx
"use client";

import dynamic from "next/dynamic";
import "swagger-ui-react/swagger-ui.css";

// Dynamic import because Swagger UI uses browser APIs
const SwaggerUI = dynamic(() => import("swagger-ui-react"), {
  ssr: false,
  loading: () => (
    <div className="flex items-center justify-center min-h-screen">
      <div className="text-gray-500 text-lg">Loading API docs...</div>
    </div>
  ),
});

export default function ApiDocsPage() {
  return (
    <div className="min-h-screen bg-white">
      <div className="max-w-6xl mx-auto py-8 px-4">
        <div className="mb-8">
          <h1 className="text-3xl font-bold text-gray-900">API Reference</h1>
          <p className="mt-2 text-gray-600">
            Interactive documentation for all API endpoints. Schemas are
            auto-generated from the Zod validators used in production.
          </p>
        </div>
        <SwaggerUI
          url="/api/openapi"
          docExpansion="list"
          defaultModelsExpandDepth={2}
          persistAuthorization={true}
        />
      </div>
    </div>
  );
}
```

### Add auth token support for testing

To let developers paste their Supabase JWT and test endpoints right from the docs page, register the security scheme:

```typescript
// lib/openapi/registry.ts — add this to the existing file
registry.registerComponent("securitySchemes", "BearerAuth", {
  type: "http",
  scheme: "bearer",
  bearerFormat: "JWT",
  description:
    "Supabase access token. Get it from supabase.auth.getSession() or the Authorization header after login.",
});
```

Now Swagger UI shows an "Authorize" button. Paste a JWT, and all subsequent requests include the `Authorization: Bearer <token>` header.

### Environment-based visibility

You might want docs visible in development and preview deploys but hidden in production:

```typescript
// middleware.ts — add to your existing middleware
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export function middleware(request: NextRequest) {
  // Block API docs in production
  if (
    request.nextUrl.pathname.startsWith("/api-docs") &&
    process.env.NODE_ENV === "production" &&
    process.env.SHOW_API_DOCS !== "true"
  ) {
    return NextResponse.rewrite(new URL("/404", request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ["/api-docs/:path*"],
};
```

Set `SHOW_API_DOCS=true` in your Vercel preview environment if you want docs there too.

## API Versioning Strategy

When your API evolves, you need a plan for versioning so existing consumers do not break.

### URL-based versioning (recommended for most projects)

The simplest approach. Your routes look like this:

```
app/
  api/
    v1/
      users/
        route.ts      # /api/v1/users
      posts/
        route.ts      # /api/v1/posts
    v2/
      users/
        route.ts      # /api/v2/users (new response shape)
```

This is straightforward in Next.js — just nest folders. See the versioning patterns reference for full implementation details.

### Header-based versioning

Use the `Accept-Version` header to let clients choose a version without changing the URL:

```typescript
// app/api/users/route.ts
export async function GET(request: Request) {
  const version = request.headers.get("Accept-Version") || "1";

  if (version === "2") {
    // New response shape
    return Response.json({ users: [], meta: { total: 0, page: 1 } });
  }

  // Default v1 response
  return Response.json({ data: [], total: 0 });
}
```

### Document which version an endpoint uses

In your OpenAPI route registrations, use tags and descriptions to make the version obvious:

```typescript
registry.registerPath({
  method: "get",
  path: "/api/v1/users",
  summary: "List users (v1)",
  description: "**Version 1** — Returns a flat array. Deprecated: use v2 for paginated responses.",
  tags: ["Users", "v1 (deprecated)"],
  // ... schemas
});

registry.registerPath({
  method: "get",
  path: "/api/v2/users",
  summary: "List users (v2)",
  description: "**Version 2** — Returns paginated response with metadata.",
  tags: ["Users", "v2"],
  // ... schemas
});
```

## Rate Limit Documentation

If your API has rate limits, document them so consumers know what to expect. Add rate limit info to each route description:

```typescript
registry.registerPath({
  method: "post",
  path: "/api/v1/auth/login",
  summary: "Log in",
  description: `Authenticate with email and password.

**Rate Limits:**
- 5 requests per minute per IP
- 20 requests per hour per IP
- Returns 429 Too Many Requests when exceeded

**Headers in response:**
- \`X-RateLimit-Limit\`: Max requests in the window
- \`X-RateLimit-Remaining\`: Requests left in the window
- \`X-RateLimit-Reset\`: Unix timestamp when the window resets`,
  tags: ["Auth"],
  responses: {
    200: {
      description: "Login successful. Returns access and refresh tokens.",
      content: {
        "application/json": {
          schema: z.object({
            access_token: z.string(),
            refresh_token: z.string(),
            expires_in: z.number().openapi({
              description: "Seconds until the access token expires",
              example: 3600,
            }),
          }),
        },
      },
    },
    429: {
      description: "Rate limit exceeded. Wait and retry.",
      content: {
        "application/json": {
          schema: z.object({
            error: z.literal("rate_limit_exceeded"),
            retry_after: z.number().openapi({
              description: "Seconds to wait before retrying",
              example: 60,
            }),
          }),
        },
      },
    },
  },
});
```

## Authentication Documentation

Document which endpoints need auth and what kind. Use OpenAPI security at the route level:

```typescript
// Public endpoint — no auth required
registry.registerPath({
  method: "get",
  path: "/api/v1/health",
  summary: "Health check",
  description: "Returns 200 if the API is running. No authentication required.",
  tags: ["System"],
  security: [], // Empty array = no auth needed
  responses: {
    200: {
      description: "API is healthy",
      content: {
        "application/json": {
          schema: z.object({
            status: z.literal("ok"),
            timestamp: z.string().datetime(),
          }),
        },
      },
    },
  },
});

// Protected endpoint — requires JWT
registry.registerPath({
  method: "get",
  path: "/api/v1/me",
  summary: "Get current user",
  description: `Returns the authenticated user's profile.

**Authentication:** Requires a valid Supabase JWT in the Authorization header.

How to get a token:
1. Call \`POST /api/v1/auth/login\` with email and password
2. Use the \`access_token\` from the response
3. Send it as \`Authorization: Bearer <token>\``,
  tags: ["Users"],
  security: [{ BearerAuth: [] }],
  responses: {
    200: {
      description: "Current user profile",
      content: {
        "application/json": {
          schema: UserSchema,
        },
      },
    },
    401: {
      description: "Token is missing, expired, or invalid",
    },
  },
});

// Admin-only endpoint
registry.registerPath({
  method: "delete",
  path: "/api/v1/users/{id}",
  summary: "Delete a user",
  description: `Permanently deletes a user account.

**Authentication:** Requires JWT with \`admin\` role.
**Authorization:** Only users with \`role: "admin"\` can call this endpoint.`,
  tags: ["Users", "Admin"],
  security: [{ BearerAuth: [] }],
  // ... params and responses
});
```

## Response Schema Documentation

Define consistent response shapes so every endpoint returns data in a predictable format. Register shared schemas once:

```typescript
// lib/schemas/api-responses.ts
import { z } from "zod";
import { registry } from "@/lib/openapi/registry";

// Standard success wrapper
export function createPaginatedResponse<T extends z.ZodTypeAny>(itemSchema: T) {
  return z.object({
    data: z.array(itemSchema),
    pagination: z.object({
      total: z.number().int().openapi({ example: 142 }),
      page: z.number().int().openapi({ example: 1 }),
      limit: z.number().int().openapi({ example: 20 }),
      total_pages: z.number().int().openapi({ example: 8 }),
    }),
  });
}

// Standard error shape — every error endpoint returns this
export const ApiErrorSchema = z.object({
  error: z.object({
    code: z.string().openapi({
      description: "Machine-readable error code",
      example: "validation_error",
    }),
    message: z.string().openapi({
      description: "Human-readable error message",
      example: "Email is required",
    }),
    details: z
      .array(
        z.object({
          field: z.string().openapi({ example: "email" }),
          message: z.string().openapi({ example: "Required" }),
        })
      )
      .optional()
      .openapi({
        description: "Field-level errors (for validation failures)",
      }),
  }),
});

registry.register("ApiError", ApiErrorSchema);

// Use it in route definitions like this:
// responses: {
//   400: {
//     description: "Validation error",
//     content: {
//       "application/json": { schema: ApiErrorSchema },
//     },
//   },
// }
```

This means every 400/401/403/404/500 response has the same shape. Consumers can write one error handler that works for every endpoint.

## API Changelog Practices

Keep a changelog so your API consumers know what changed and when. Create a simple markdown file:

```markdown
<!-- docs/API_CHANGELOG.md -->

# API Changelog

## 2025-01-15 — v2.1.0

### Added
- `GET /api/v2/users` now supports `sort_by` and `sort_order` query parameters
- New endpoint: `GET /api/v2/users/{id}/activity`

### Changed
- `POST /api/v2/posts` now returns the full post object instead of just the ID

### Fixed
- `PATCH /api/v2/users/{id}` no longer returns 500 when updating only the name field

---

## 2025-01-02 — v2.0.0 (Breaking)

### Breaking Changes
- Response envelope changed from `{ data, total }` to `{ data, pagination: { total, page, limit, total_pages } }`
- `GET /api/v1/users` is now deprecated. Use `GET /api/v2/users` instead.
- Removed `DELETE /api/v1/posts/{id}/hard` — use `DELETE /api/v2/posts/{id}?permanent=true`

### Migration Guide
1. Update your response parsing to use `response.pagination.total` instead of `response.total`
2. Change all `/api/v1/users` calls to `/api/v2/users`
3. If you used hard delete, pass `?permanent=true` to the v2 delete endpoint
```

You can also expose the changelog inside the OpenAPI spec description:

```typescript
// lib/openapi/registry.ts — update the info block
info: {
  title: "My App API",
  version: "2.1.0",
  description: `Auto-generated API documentation.

## Changelog

**2025-01-15 — v2.1.0**
- Added sort parameters to GET /api/v2/users
- New endpoint: GET /api/v2/users/{id}/activity

**2025-01-02 — v2.0.0 (Breaking)**
- New response envelope with pagination object
- Deprecated v1 endpoints

[Full changelog →](/docs/API_CHANGELOG.md)`,
},
```

## Rules

1. **Zod is the single source of truth.** Every request body, query param, and response shape must be defined as a Zod schema first, then registered with the OpenAPI registry. Never write raw OpenAPI JSON by hand.

2. **Register routes next to the schemas they use.** Keep your `registerPath` calls in `lib/openapi/routes/` files organized by resource (users.ts, posts.ts, billing.ts). Import the Zod schemas from `lib/schemas/`.

3. **Import all route files in the OpenAPI endpoint.** The `app/api/openapi/route.ts` handler must import every `lib/openapi/routes/*.ts` file so the registrations execute. If you add a new resource, add the import.

4. **Use consistent tags.** Every route must have at least one tag matching its resource name (Users, Posts, Billing). Use additional tags for version (v1, v2) or access level (Admin). Tags organize the Swagger UI sidebar.

5. **Document every response code.** If a route can return 200, 400, 401, 403, 404, or 500, each one needs an entry in `responses`. Include the schema for error responses using the shared `ApiErrorSchema`.

6. **Add `.openapi()` metadata to Zod fields.** Use `.openapi({ description, example })` on fields to make the docs more useful. Especially important for IDs, enums, and date fields.

7. **Version your API from day one.** Start with `/api/v1/` even if you only have one version. It costs nothing and saves a painful migration later. See the versioning patterns reference for implementation.

8. **Keep the changelog updated.** Every PR that changes an API endpoint must update `docs/API_CHANGELOG.md`. Include the date, version, what changed, and whether it is breaking.

9. **Gate docs visibility per environment.** Use middleware to hide `/api-docs` in production unless explicitly enabled. Preview deploys and local dev should always show docs.

10. **Test the spec is valid.** Add a test that hits `GET /api/openapi` and parses the response as valid OpenAPI 3.0. If someone registers a route with invalid schemas, the test catches it before deploy.
