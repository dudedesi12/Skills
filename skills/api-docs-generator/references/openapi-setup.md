# OpenAPI Setup with zod-to-openapi

Complete pipeline for generating OpenAPI specs from your existing Zod schemas.

## Installation

```bash
npm install @asteasolutions/zod-to-openapi zod
```

## Registry Setup

```typescript
// lib/openapi/registry.ts
import { OpenAPIRegistry, OpenApiGeneratorV31 } from "@asteasolutions/zod-to-openapi";
import { z } from "zod";

export const registry = new OpenAPIRegistry();

// Generate the full OpenAPI document
export function generateOpenAPIDocument() {
  const generator = new OpenApiGeneratorV31(registry.definitions);
  return generator.generateDocument({
    openapi: "3.1.0",
    info: {
      title: "Immigration Platform API",
      version: "1.0.0",
      description: "API for visa eligibility assessments, user management, and immigration data.",
    },
    servers: [
      { url: "http://localhost:3000", description: "Development" },
      { url: "https://yourdomain.com", description: "Production" },
    ],
    security: [{ bearerAuth: [] }],
  });
}

// Register the bearer auth scheme
registry.registerComponent("securitySchemes", "bearerAuth", {
  type: "http",
  scheme: "bearer",
  bearerFormat: "JWT",
});
```

## Registering Schemas

```typescript
// lib/openapi/schemas/user.ts
import { z } from "zod";
import { registry } from "../registry";

export const UserSchema = registry.register(
  "User",
  z.object({
    id: z.string().uuid(),
    email: z.string().email(),
    fullName: z.string(),
    role: z.enum(["user", "admin"]),
    createdAt: z.string().datetime(),
  })
);

export const CreateUserSchema = registry.register(
  "CreateUser",
  z.object({
    email: z.string().email().openapi({ example: "user@example.com" }),
    fullName: z.string().min(2).openapi({ example: "Jane Smith" }),
    password: z.string().min(8).openapi({ example: "SecurePass123!" }),
  })
);

export const UpdateUserSchema = registry.register(
  "UpdateUser",
  CreateUserSchema.partial().omit({ password: true })
);
```

```typescript
// lib/openapi/schemas/assessment.ts
import { z } from "zod";
import { registry } from "../registry";

export const VisaAssessmentSchema = registry.register(
  "VisaAssessment",
  z.object({
    id: z.string().uuid(),
    userId: z.string().uuid(),
    visaSubclass: z.enum(["189", "190", "491"]),
    pointsScore: z.number().int().min(0).max(130),
    result: z.enum(["eligible", "not_eligible", "needs_review"]),
    createdAt: z.string().datetime(),
  })
);

export const CreateAssessmentSchema = registry.register(
  "CreateAssessment",
  z.object({
    occupation: z.string().openapi({ example: "Software Engineer" }),
    yearsExperience: z.number().int().min(0).max(50),
    englishLevel: z.enum(["superior", "proficient", "competent", "basic"]),
    ageRange: z.enum(["18-24", "25-32", "33-39", "40-44", "45+"]),
    qualificationLevel: z.enum(["phd", "masters", "bachelors", "diploma", "trade", "none"]),
  })
);
```

## Registering Route Paths

```typescript
// lib/openapi/paths/users.ts
import { registry } from "../registry";
import { UserSchema, CreateUserSchema, UpdateUserSchema } from "../schemas/user";
import { z } from "zod";

// Error response schema (reusable)
const ErrorSchema = registry.register(
  "Error",
  z.object({
    error: z.string(),
    details: z.record(z.array(z.string())).optional(),
  })
);

// GET /api/v1/users
registry.registerPath({
  method: "get",
  path: "/api/v1/users",
  tags: ["Users"],
  summary: "List all users",
  description: "Returns a paginated list of users. Requires admin role.",
  request: {
    query: z.object({
      page: z.coerce.number().int().min(1).default(1).openapi({ example: 1 }),
      limit: z.coerce.number().int().min(1).max(100).default(20),
      search: z.string().optional().openapi({ example: "jane" }),
    }),
  },
  responses: {
    200: {
      description: "List of users",
      content: { "application/json": { schema: z.array(UserSchema) } },
    },
    401: {
      description: "Unauthorized",
      content: { "application/json": { schema: ErrorSchema } },
    },
  },
  security: [{ bearerAuth: [] }],
});

// POST /api/v1/users
registry.registerPath({
  method: "post",
  path: "/api/v1/users",
  tags: ["Users"],
  summary: "Create a new user",
  request: {
    body: { content: { "application/json": { schema: CreateUserSchema } } },
  },
  responses: {
    201: {
      description: "User created",
      content: { "application/json": { schema: UserSchema } },
    },
    400: {
      description: "Validation error",
      content: { "application/json": { schema: ErrorSchema } },
    },
  },
});

// GET /api/v1/users/:id
registry.registerPath({
  method: "get",
  path: "/api/v1/users/{id}",
  tags: ["Users"],
  summary: "Get user by ID",
  request: {
    params: z.object({ id: z.string().uuid() }),
  },
  responses: {
    200: {
      description: "User found",
      content: { "application/json": { schema: UserSchema } },
    },
    404: {
      description: "User not found",
      content: { "application/json": { schema: ErrorSchema } },
    },
  },
});

// PATCH /api/v1/users/:id
registry.registerPath({
  method: "patch",
  path: "/api/v1/users/{id}",
  tags: ["Users"],
  summary: "Update user",
  request: {
    params: z.object({ id: z.string().uuid() }),
    body: { content: { "application/json": { schema: UpdateUserSchema } } },
  },
  responses: {
    200: {
      description: "User updated",
      content: { "application/json": { schema: UserSchema } },
    },
    404: {
      description: "User not found",
      content: { "application/json": { schema: ErrorSchema } },
    },
  },
});
```

## Serving the Spec

```typescript
// app/api/docs/spec/route.ts
import { NextResponse } from "next/server";
import { generateOpenAPIDocument } from "@/lib/openapi/registry";

// Import all path registrations (side effects)
import "@/lib/openapi/paths/users";
import "@/lib/openapi/paths/assessments";

export async function GET() {
  const doc = generateOpenAPIDocument();
  return NextResponse.json(doc);
}
```

## Collecting All Registrations

```typescript
// lib/openapi/index.ts
// Import this file to register all schemas and paths

// Schemas
import "./schemas/user";
import "./schemas/assessment";

// Paths
import "./paths/users";
import "./paths/assessments";

export { registry, generateOpenAPIDocument } from "./registry";
```
