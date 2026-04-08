# Swagger UI Integration

## Install

```bash
npm install swagger-ui-react
npm install -D @types/swagger-ui-react
```

## API Docs Page

```tsx
// app/api-docs/page.tsx
"use client";
import dynamic from "next/dynamic";
import "swagger-ui-react/swagger-ui.css";

const SwaggerUI = dynamic(() => import("swagger-ui-react"), { ssr: false });

export default function ApiDocsPage() {
  return (
    <div className="min-h-screen bg-white">
      <div className="mx-auto max-w-6xl px-4 py-8">
        <h1 className="mb-4 text-3xl font-bold">API Documentation</h1>
        <p className="mb-8 text-gray-600">
          Interactive API reference. Expand any endpoint to see request/response schemas and try it out.
        </p>
        <SwaggerUI
          url="/api/docs/spec"
          docExpansion="list"
          defaultModelsExpandDepth={2}
          persistAuthorization
          tryItOutEnabled
        />
      </div>
    </div>
  );
}
```

## Adding Auth Token for Try-It-Out

```tsx
// app/api-docs/page.tsx — with auth
"use client";
import dynamic from "next/dynamic";
import { useState } from "react";
import "swagger-ui-react/swagger-ui.css";

const SwaggerUI = dynamic(() => import("swagger-ui-react"), { ssr: false });

export default function ApiDocsPage() {
  const [token, setToken] = useState("");

  return (
    <div className="min-h-screen bg-white">
      <div className="mx-auto max-w-6xl px-4 py-8">
        <h1 className="mb-4 text-3xl font-bold">API Documentation</h1>

        <div className="mb-6 rounded border bg-gray-50 p-4">
          <label htmlFor="auth-token" className="block text-sm font-medium text-gray-700">
            Bearer Token (for Try It Out)
          </label>
          <input
            id="auth-token"
            type="text"
            value={token}
            onChange={(e) => setToken(e.target.value)}
            placeholder="Paste your JWT token here"
            className="mt-1 w-full rounded border px-3 py-2 font-mono text-sm"
          />
        </div>

        <SwaggerUI
          url="/api/docs/spec"
          docExpansion="list"
          persistAuthorization
          requestInterceptor={(req) => {
            if (token) {
              req.headers.Authorization = `Bearer ${token}`;
            }
            return req;
          }}
        />
      </div>
    </div>
  );
}
```

## Using Scalar Instead (Modern Alternative)

Scalar is a modern, better-looking alternative to Swagger UI.

```bash
npm install @scalar/nextjs-api-reference
```

```tsx
// app/api-docs/page.tsx
import { ApiReference } from "@scalar/nextjs-api-reference";

export default function ApiDocsPage() {
  return (
    <ApiReference
      configuration={{
        spec: { url: "/api/docs/spec" },
        theme: "default",
        layout: "modern",
        hideModels: false,
        hideDownloadButton: false,
      }}
    />
  );
}
```

## Environment-Based Visibility

Show docs in dev/preview, optionally hide in production:

```tsx
// app/api-docs/page.tsx
import { notFound } from "next/navigation";

export default function ApiDocsPage() {
  // Hide in production unless explicitly enabled
  if (process.env.NODE_ENV === "production" && !process.env.SHOW_API_DOCS) {
    notFound();
  }

  return (
    // ... Swagger UI or Scalar
  );
}
```

## Custom CSS Theming

```css
/* app/api-docs/swagger-overrides.css */
.swagger-ui .topbar { display: none; }
.swagger-ui .info .title { color: #1e293b; }
.swagger-ui .opblock.opblock-get .opblock-summary { border-color: #3b82f6; }
.swagger-ui .opblock.opblock-post .opblock-summary { border-color: #22c55e; }
.swagger-ui .opblock.opblock-put .opblock-summary { border-color: #f59e0b; }
.swagger-ui .opblock.opblock-delete .opblock-summary { border-color: #ef4444; }
.swagger-ui .btn.execute { background-color: #3b82f6; }
```
