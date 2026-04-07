# Middleware Recipes

## 1. Auth Guard

```typescript
if (pathname.startsWith('/dashboard')) {
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) return NextResponse.redirect(new URL('/login', request.url));
}
```

## 2. Admin Only

```typescript
if (pathname.startsWith('/admin')) {
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) return NextResponse.redirect(new URL('/login', request.url));
  const { data: profile } = await supabase.from('profiles').select('role').eq('id', user.id).single();
  if (profile?.role !== 'admin') return NextResponse.redirect(new URL('/unauthorized', request.url));
}
```

## 3. Geo-Redirect

```typescript
const country = request.geo?.country;
if (pathname === '/' && country === 'IN') {
  return NextResponse.redirect(new URL('/in', request.url));
}
```

## 4. Bot Blocking

```typescript
const ua = request.headers.get('user-agent') ?? '';
if (pathname.startsWith('/api/') && /bot|crawler|spider|scraper/i.test(ua)) {
  return new NextResponse('Forbidden', { status: 403 });
}
```

## 5. A/B Testing

```typescript
const variant = request.cookies.get('ab-variant')?.value;
if (!variant) {
  const newVariant = Math.random() > 0.5 ? 'A' : 'B';
  const response = NextResponse.next();
  response.cookies.set('ab-variant', newVariant, { maxAge: 30 * 86400 });
  return response;
}
if (pathname === '/pricing' && variant === 'B') {
  return NextResponse.rewrite(new URL('/pricing-v2', request.url));
}
```

## 6. Feature Flags

```typescript
const flags = request.cookies.get('feature-flags')?.value;
const parsed = flags ? JSON.parse(flags) : {};
if (pathname.startsWith('/new-dashboard') && !parsed.newDashboard) {
  return NextResponse.redirect(new URL('/dashboard', request.url));
}
```

## 7. Rate Limiting (with KV)

```typescript
import { kv } from '@vercel/kv';

if (pathname.startsWith('/api/')) {
  const ip = request.ip ?? request.headers.get('x-forwarded-for') ?? 'unknown';
  const key = `rate:${ip}`;
  const count = await kv.incr(key);
  if (count === 1) await kv.expire(key, 60);
  if (count > 100) {
    return NextResponse.json({ error: 'Rate limited' }, { status: 429 });
  }
}
```

## 8. Maintenance Mode

```typescript
const isMaintenanceMode = process.env.MAINTENANCE_MODE === 'true';
if (isMaintenanceMode && !pathname.startsWith('/maintenance') && !pathname.startsWith('/_next')) {
  return NextResponse.rewrite(new URL('/maintenance', request.url));
}
```

## 9. Redirect Trailing Slashes

```typescript
if (pathname !== '/' && pathname.endsWith('/')) {
  return NextResponse.redirect(new URL(pathname.slice(0, -1), request.url), 308);
}
```

## 10. Add Security Headers

```typescript
const response = NextResponse.next();
response.headers.set('X-Frame-Options', 'DENY');
response.headers.set('X-Content-Type-Options', 'nosniff');
response.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin');
return response;
```

## 11. CORS for API Routes

```typescript
if (pathname.startsWith('/api/public/')) {
  if (request.method === 'OPTIONS') {
    return new NextResponse(null, {
      headers: {
        'Access-Control-Allow-Origin': 'https://yourdomain.com',
        'Access-Control-Allow-Methods': 'GET, POST',
        'Access-Control-Allow-Headers': 'Content-Type, Authorization',
        'Access-Control-Max-Age': '86400',
      },
    });
  }
  const response = NextResponse.next();
  response.headers.set('Access-Control-Allow-Origin', 'https://yourdomain.com');
  return response;
}
```

## 12. Basic Auth for Preview

```typescript
if (process.env.VERCEL_ENV === 'preview') {
  const auth = request.headers.get('authorization');
  if (!auth || auth !== `Basic ${btoa(`admin:${process.env.PREVIEW_PASSWORD}`)}`) {
    return new NextResponse('Auth required', {
      status: 401,
      headers: { 'WWW-Authenticate': 'Basic realm="Preview"' },
    });
  }
}
```
