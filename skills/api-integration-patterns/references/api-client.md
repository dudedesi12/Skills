# API Client Wrapper

Complete production-ready API client with retry, timeout, rate limiting, and typed responses.

## Full Implementation

```typescript
// lib/api-client.ts
interface ApiClientConfig {
  baseUrl: string;
  timeout?: number;
  maxRetries?: number;
  headers?: Record<string, string>;
  rateLimitPerSecond?: number;
}

interface ApiResponse<T> {
  data: T | null;
  error: string | null;
  status: number;
  headers: Record<string, string>;
  durationMs: number;
}

interface RequestOptions extends RequestInit {
  params?: Record<string, string>;
  timeout?: number;
}

export class ApiClient {
  private baseUrl: string;
  private timeout: number;
  private maxRetries: number;
  private defaultHeaders: Record<string, string>;
  private rateLimitPerSecond: number;
  private requestTimestamps: number[] = [];

  constructor(config: ApiClientConfig) {
    this.baseUrl = config.baseUrl.replace(/\/$/, "");
    this.timeout = config.timeout ?? 10000;
    this.maxRetries = config.maxRetries ?? 3;
    this.defaultHeaders = config.headers ?? {};
    this.rateLimitPerSecond = config.rateLimitPerSecond ?? 0;
  }

  private async waitForRateLimit(): Promise<void> {
    if (this.rateLimitPerSecond <= 0) return;

    const now = Date.now();
    this.requestTimestamps = this.requestTimestamps.filter(
      (t) => now - t < 1000
    );

    if (this.requestTimestamps.length >= this.rateLimitPerSecond) {
      const oldest = this.requestTimestamps[0]!;
      const waitMs = 1000 - (now - oldest);
      if (waitMs > 0) {
        await new Promise((resolve) => setTimeout(resolve, waitMs));
      }
    }

    this.requestTimestamps.push(Date.now());
  }

  private buildUrl(endpoint: string, params?: Record<string, string>): string {
    const url = new URL(`${this.baseUrl}${endpoint}`);
    if (params) {
      Object.entries(params).forEach(([key, value]) => {
        url.searchParams.set(key, value);
      });
    }
    return url.toString();
  }

  async request<T>(
    endpoint: string,
    options: RequestOptions = {}
  ): Promise<ApiResponse<T>> {
    const { params, timeout: requestTimeout, ...fetchOptions } = options;
    const url = this.buildUrl(endpoint, params);
    const effectiveTimeout = requestTimeout ?? this.timeout;
    const start = Date.now();

    for (let attempt = 0; attempt <= this.maxRetries; attempt++) {
      try {
        await this.waitForRateLimit();

        const controller = new AbortController();
        const timeoutId = setTimeout(() => controller.abort(), effectiveTimeout);

        const response = await fetch(url, {
          ...fetchOptions,
          signal: controller.signal,
          headers: {
            ...this.defaultHeaders,
            ...(fetchOptions.headers as Record<string, string>),
          },
        });

        clearTimeout(timeoutId);

        const responseHeaders: Record<string, string> = {};
        response.headers.forEach((value, key) => {
          responseHeaders[key] = value;
        });

        // Rate limited — wait and retry
        if (response.status === 429) {
          const retryAfter = response.headers.get("Retry-After");
          const waitMs = retryAfter
            ? parseInt(retryAfter) * 1000
            : Math.min(2 ** attempt * 1000, 30000);
          console.warn(`Rate limited on ${endpoint}, waiting ${waitMs}ms`);
          await new Promise((r) => setTimeout(r, waitMs));
          continue;
        }

        // Server error — retry
        if (response.status >= 500 && attempt < this.maxRetries) {
          const waitMs = 2 ** attempt * 1000;
          console.warn(`Server error ${response.status} on ${endpoint}, retrying in ${waitMs}ms`);
          await new Promise((r) => setTimeout(r, waitMs));
          continue;
        }

        // Client error — don't retry
        if (!response.ok) {
          let errorBody = "";
          try {
            errorBody = await response.text();
          } catch {}
          return {
            data: null,
            error: `HTTP ${response.status}: ${errorBody || response.statusText}`,
            status: response.status,
            headers: responseHeaders,
            durationMs: Date.now() - start,
          };
        }

        // Success
        const contentType = response.headers.get("content-type") ?? "";
        let data: T;
        if (contentType.includes("application/json")) {
          data = (await response.json()) as T;
        } else {
          data = (await response.text()) as unknown as T;
        }

        return {
          data,
          error: null,
          status: response.status,
          headers: responseHeaders,
          durationMs: Date.now() - start,
        };
      } catch (err) {
        if (attempt === this.maxRetries) {
          const message =
            err instanceof Error
              ? err.name === "AbortError"
                ? `Request timed out after ${effectiveTimeout}ms`
                : err.message
              : "Unknown error";
          return {
            data: null,
            error: message,
            status: 0,
            headers: {},
            durationMs: Date.now() - start,
          };
        }
        const waitMs = 2 ** attempt * 1000;
        await new Promise((r) => setTimeout(r, waitMs));
      }
    }

    return {
      data: null,
      error: "Max retries exceeded",
      status: 0,
      headers: {},
      durationMs: Date.now() - start,
    };
  }

  async get<T>(endpoint: string, params?: Record<string, string>) {
    return this.request<T>(endpoint, { method: "GET", params });
  }

  async post<T>(endpoint: string, body: unknown) {
    return this.request<T>(endpoint, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(body),
    });
  }

  async put<T>(endpoint: string, body: unknown) {
    return this.request<T>(endpoint, {
      method: "PUT",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(body),
    });
  }

  async patch<T>(endpoint: string, body: unknown) {
    return this.request<T>(endpoint, {
      method: "PATCH",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(body),
    });
  }

  async delete<T>(endpoint: string) {
    return this.request<T>(endpoint, { method: "DELETE" });
  }
}
```

## Usage Examples

```typescript
// lib/clients/github-api.ts
import { ApiClient } from "@/lib/api-client";

const github = new ApiClient({
  baseUrl: "https://api.github.com",
  timeout: 8000,
  maxRetries: 2,
  rateLimitPerSecond: 10,
  headers: {
    Authorization: `Bearer ${process.env.GITHUB_TOKEN}`,
    Accept: "application/vnd.github.v3+json",
  },
});

interface GitHubRepo {
  id: number;
  name: string;
  full_name: string;
  description: string;
  stargazers_count: number;
}

export async function getRepo(owner: string, repo: string) {
  const result = await github.get<GitHubRepo>(`/repos/${owner}/${repo}`);
  if (result.error) {
    throw new Error(`GitHub API error: ${result.error}`);
  }
  return result.data!;
}

export async function searchRepos(query: string) {
  const result = await github.get<{ items: GitHubRepo[] }>("/search/repositories", {
    q: query,
    sort: "stars",
    per_page: "10",
  });
  if (result.error) {
    throw new Error(`GitHub search failed: ${result.error}`);
  }
  return result.data!.items;
}
```
