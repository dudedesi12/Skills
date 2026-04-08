# Cache-Control Headers Guide

## Directive Reference

| Directive | Scope | Meaning |
|-----------|-------|---------|
| `public` | CDN + browser | Any cache can store this |
| `private` | Browser only | Only the user's browser caches this |
| `no-cache` | Both | Must revalidate before using cached version |
| `no-store` | Both | Never cache this response |
| `max-age=N` | Browser | Browser cache duration (seconds) |
| `s-maxage=N` | CDN | CDN cache duration (seconds) |
| `stale-while-revalidate=N` | CDN | Serve stale for N seconds while refreshing |
| `stale-if-error=N` | CDN | Serve stale for N seconds if origin errors |
| `must-revalidate` | Both | Don't serve stale, always check origin |
| `immutable` | Both | Content will never change (for hashed assets) |

## Common Patterns

### Static Assets (fonts, hashed JS/CSS)

```
Cache-Control: public, max-age=31536000, immutable
```
One year cache. The filename hash changes when content changes, so this is safe.

### Public API (changes infrequently)

```
Cache-Control: public, s-maxage=86400, stale-while-revalidate=604800
```
CDN caches for 1 day. Serves stale for up to 7 days while refreshing.

### Semi-Dynamic Public Content

```
Cache-Control: public, s-maxage=60, stale-while-revalidate=3600
```
CDN caches for 1 minute. Serves stale for 1 hour while refreshing.

### Authenticated API Response

```
Cache-Control: private, max-age=60
```
Browser-only cache for 1 minute. CDN never caches.

### Real-Time Data (never cache)

```
Cache-Control: no-store
```
No caching at all. Every request hits the origin.

### Payment/Sensitive Data

```
Cache-Control: no-store, no-cache, must-revalidate
```
Triple protection: never store, always revalidate, no stale.

## Setting Headers in Next.js

### API Routes

```typescript
export async function GET() {
  const data = await fetchData();
  return NextResponse.json(data, {
    headers: {
      "Cache-Control": "public, s-maxage=3600, stale-while-revalidate=86400",
    },
  });
}
```

### vercel.json (Global)

```json
{
  "headers": [
    {
      "source": "/api/public/(.*)",
      "headers": [
        { "key": "Cache-Control", "value": "public, s-maxage=3600, stale-while-revalidate=86400" }
      ]
    },
    {
      "source": "/api/user/(.*)",
      "headers": [
        { "key": "Cache-Control", "value": "private, no-cache" }
      ]
    }
  ]
}
```

### next.config.ts

```typescript
const nextConfig = {
  async headers() {
    return [
      {
        source: "/:path*.woff2",
        headers: [
          { key: "Cache-Control", value: "public, max-age=31536000, immutable" },
        ],
      },
    ];
  },
};
```

## Vary Header

Use `Vary` when the same URL returns different content based on request headers:

```typescript
// Response varies by Accept-Language
response.headers.set("Vary", "Accept-Language");

// Response varies by cookie (authenticated vs not)
response.headers.set("Vary", "Cookie");
```

**Warning:** `Vary: Cookie` effectively disables CDN caching for authenticated users. Use `Cache-Control: private` instead.

## Debug Headers

Add these to verify caching is working:

```typescript
response.headers.set("X-Cache-Status", cacheHit ? "HIT" : "MISS");
response.headers.set("X-Cache-TTL", `${ttl}s`);
```

Vercel automatically adds `X-Vercel-Cache` header: `HIT`, `MISS`, `STALE`, `PRERENDER`.
