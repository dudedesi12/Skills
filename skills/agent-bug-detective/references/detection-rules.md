# Detection Rules — By Category

## 1. Runtime Errors

| Rule | Severity | Detection Method |
|------|----------|-----------------|
| >50 client errors/24h | Critical | Query client_errors table |
| >10 client errors/24h | High | Query client_errors table |
| Same error repeating >5x | High | Group by message fingerprint |
| API route returning 500 | Critical | Fetch test all /api/* routes |
| Unhandled promise rejection | Medium | Client error tracker |

## 2. Performance

| Rule | Severity | Detection Method |
|------|----------|-----------------|
| Page load >3s | High | Lighthouse CI |
| API response >2s | High | Timed fetch to /api/* routes |
| Bundle size >250KB (gzipped) | Medium | Build output analysis |
| No loading.tsx for dynamic pages | Low | File system check |
| Images without width/height | Medium | Lighthouse CI |
| No Suspense boundaries | Low | Static analysis |

## 3. Security

| Rule | Severity | Detection Method |
|------|----------|-----------------|
| Missing CSP header | Critical | HEAD request check |
| Missing X-Frame-Options | High | HEAD request check |
| Missing HSTS | High | HEAD request check |
| Tables without RLS | Critical | Supabase query: `select * from pg_tables where rowsecurity = false` |
| Exposed env var in client bundle | Critical | Search NEXT_PUBLIC_ for secrets |
| No rate limiting on auth routes | High | Config check |
| npm audit critical findings | Critical | `npm audit --json` |

## 4. SEO

| Rule | Severity | Detection Method |
|------|----------|-----------------|
| Missing title tag | High | Fetch page, parse HTML |
| Missing meta description | Medium | Fetch page, parse HTML |
| Missing OG image | Medium | Fetch page, check meta tags |
| No sitemap.xml | High | Fetch /sitemap.xml |
| No robots.txt | Medium | Fetch /robots.txt |
| Broken internal links | High | Crawl all internal <a> tags |
| Missing canonical URL | Medium | Parse HTML head |

## 5. Accessibility

| Rule | Severity | Detection Method |
|------|----------|-----------------|
| Missing alt on images | High | Lighthouse audit |
| Contrast ratio <4.5:1 | Medium | Lighthouse audit |
| Missing form labels | High | Lighthouse audit |
| No skip-to-content link | Low | Parse HTML |
| Missing lang attribute | Medium | Parse HTML |
| Interactive elements without focus styles | Medium | Lighthouse audit |

## 6. Data Integrity

| Rule | Severity | Detection Method |
|------|----------|-----------------|
| Orphaned records (FK violations) | Medium | SQL query for dangling refs |
| Null values in required fields | High | SQL check constraints |
| Duplicate entries | Medium | SQL group by + having count > 1 |
| Stale cache (>24h old) | Low | Check cache timestamps |

## 7. UX

| Rule | Severity | Detection Method |
|------|----------|-----------------|
| Error page without recovery action | Medium | Manual + Lighthouse |
| Form without validation feedback | Medium | Lighthouse audit |
| No loading indicators on async actions | Low | Code analysis |
| 404 page returns 200 status | High | Fetch known bad URL |

## 8. Dependencies

| Rule | Severity | Detection Method |
|------|----------|-----------------|
| npm audit critical | Critical | `npm audit --json` |
| npm audit high | High | `npm audit --json` |
| Major version behind on Next.js | Medium | Compare installed vs latest |
| Deprecated package | Low | `npm outdated --json` |
