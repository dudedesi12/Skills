# vercel.json — Complete Reference

## Full Example

```json
{
  "framework": "nextjs",
  "regions": ["syd1"],
  "buildCommand": "next build",
  "installCommand": "npm install",

  "crons": [
    { "path": "/api/cron/daily-cleanup", "schedule": "0 2 * * *" },
    { "path": "/api/cron/hourly-sync", "schedule": "0 * * * *" }
  ],

  "redirects": [
    { "source": "/old-blog/:slug", "destination": "/blog/:slug", "permanent": true },
    { "source": "/app", "destination": "https://app.yourdomain.com", "permanent": false }
  ],

  "rewrites": [
    { "source": "/api/proxy/:path*", "destination": "https://api.external.com/:path*" },
    { "source": "/blog", "destination": "/posts" }
  ],

  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Frame-Options", "value": "DENY" },
        { "key": "X-Content-Type-Options", "value": "nosniff" },
        { "key": "Strict-Transport-Security", "value": "max-age=63072000; includeSubDomains; preload" }
      ]
    },
    {
      "source": "/api/(.*)",
      "headers": [
        { "key": "Cache-Control", "value": "no-store, no-cache" }
      ]
    },
    {
      "source": "/(.*\\.(?:js|css|woff2|png|jpg|svg)$)",
      "headers": [
        { "key": "Cache-Control", "value": "public, max-age=31536000, immutable" }
      ]
    }
  ],

  "functions": {
    "app/api/heavy/**": {
      "maxDuration": 60,
      "memory": 1024
    },
    "app/api/edge/**": {
      "runtime": "edge"
    }
  }
}
```

## Key Properties

| Property | Description |
|----------|-------------|
| `framework` | Auto-detected, but can force `nextjs` |
| `regions` | Deploy functions to specific regions. `syd1` for Australia |
| `crons` | Scheduled jobs (Pro: unlimited, Hobby: 2/day) |
| `redirects` | URL redirects (301/308 permanent, 302/307 temporary) |
| `rewrites` | URL rewrites (user sees original URL) |
| `headers` | Custom HTTP headers per route pattern |
| `functions` | Per-route function configuration |

## Redirect vs Rewrite

- **Redirect**: Changes the URL in the browser. Use for moved pages, external links.
- **Rewrite**: Keeps the URL the same but serves different content. Use for proxying, A/B tests, path aliasing.

## Region Codes

| Code | Location |
|------|----------|
| syd1 | Sydney, Australia |
| sin1 | Singapore |
| hnd1 | Tokyo, Japan |
| iad1 | Washington DC, US (default) |
| sfo1 | San Francisco, US |
| lhr1 | London, UK |
| fra1 | Frankfurt, Germany |
