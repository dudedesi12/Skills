# Circuit Breaker Pattern

Full implementation with configurable thresholds, state transitions, and monitoring.

## How It Works

The circuit breaker has three states:

1. **Closed** (normal) — requests pass through. Failures are counted. When failures hit the threshold, the circuit opens.
2. **Open** (blocked) — all requests are immediately rejected with an error. After a timeout period, the circuit moves to half-open.
3. **Half-Open** (testing) — a limited number of requests pass through. If they succeed, the circuit closes. If any fail, it opens again.

## Full Implementation

```typescript
// lib/agents/circuit-breaker.ts

type CircuitState = "closed" | "open" | "half-open";

interface CircuitBreakerConfig {
  failureThreshold: number;
  resetTimeoutMs: number;
  halfOpenMaxAttempts: number;
  onStateChange?: (from: CircuitState, to: CircuitState, stats: CircuitStats) => void;
}

interface CircuitStats {
  totalRequests: number;
  totalFailures: number;
  consecutiveFailures: number;
  lastFailureTime: number | null;
  lastSuccessTime: number | null;
  state: CircuitState;
}

export class CircuitBreaker {
  private state: CircuitState = "closed";
  private consecutiveFailures = 0;
  private totalRequests = 0;
  private totalFailures = 0;
  private lastFailureTime: number | null = null;
  private lastSuccessTime: number | null = null;
  private halfOpenAttempts = 0;
  private halfOpenSuccesses = 0;

  private readonly failureThreshold: number;
  private readonly resetTimeoutMs: number;
  private readonly halfOpenMaxAttempts: number;
  private readonly onStateChange?: CircuitBreakerConfig["onStateChange"];

  constructor(config: Partial<CircuitBreakerConfig> = {}) {
    this.failureThreshold = config.failureThreshold ?? 5;
    this.resetTimeoutMs = config.resetTimeoutMs ?? 30_000;
    this.halfOpenMaxAttempts = config.halfOpenMaxAttempts ?? 3;
    this.onStateChange = config.onStateChange;
  }

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    this.totalRequests++;

    if (this.state === "open") {
      if (this.lastFailureTime && Date.now() - this.lastFailureTime > this.resetTimeoutMs) {
        this.transitionTo("half-open");
        this.halfOpenAttempts = 0;
        this.halfOpenSuccesses = 0;
      } else {
        throw new CircuitBreakerError(
          "Circuit breaker is OPEN — requests are blocked",
          this.getStats()
        );
      }
    }

    if (this.state === "half-open") {
      this.halfOpenAttempts++;
      if (this.halfOpenAttempts > this.halfOpenMaxAttempts) {
        this.transitionTo("open");
        this.lastFailureTime = Date.now();
        throw new CircuitBreakerError(
          "Circuit breaker exceeded half-open attempt limit",
          this.getStats()
        );
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (err) {
      this.onFailure();
      throw err;
    }
  }

  private onSuccess(): void {
    this.lastSuccessTime = Date.now();

    if (this.state === "half-open") {
      this.halfOpenSuccesses++;
      if (this.halfOpenSuccesses >= this.halfOpenMaxAttempts) {
        this.transitionTo("closed");
        this.consecutiveFailures = 0;
      }
    } else {
      this.consecutiveFailures = 0;
    }
  }

  private onFailure(): void {
    this.totalFailures++;
    this.consecutiveFailures++;
    this.lastFailureTime = Date.now();

    if (this.state === "half-open") {
      this.transitionTo("open");
    } else if (this.consecutiveFailures >= this.failureThreshold) {
      this.transitionTo("open");
    }
  }

  private transitionTo(newState: CircuitState): void {
    const oldState = this.state;
    if (oldState === newState) return;

    this.state = newState;

    console.log(JSON.stringify({
      level: "info",
      event: "circuit_breaker_state_change",
      from: oldState,
      to: newState,
      stats: this.getStats(),
    }));

    if (this.onStateChange) {
      try {
        this.onStateChange(oldState, newState, this.getStats());
      } catch {
        // Swallow callback errors to avoid breaking the circuit breaker
      }
    }
  }

  getState(): CircuitState {
    return this.state;
  }

  getStats(): CircuitStats {
    return {
      totalRequests: this.totalRequests,
      totalFailures: this.totalFailures,
      consecutiveFailures: this.consecutiveFailures,
      lastFailureTime: this.lastFailureTime,
      lastSuccessTime: this.lastSuccessTime,
      state: this.state,
    };
  }

  reset(): void {
    this.state = "closed";
    this.consecutiveFailures = 0;
    this.halfOpenAttempts = 0;
    this.halfOpenSuccesses = 0;
  }
}

export class CircuitBreakerError extends Error {
  constructor(
    message: string,
    public readonly stats: CircuitStats
  ) {
    super(message);
    this.name = "CircuitBreakerError";
  }
}
```

## Per-Agent Circuit Breaker Registry

Manage one circuit breaker per agent:

```typescript
// lib/agents/circuit-registry.ts
import { CircuitBreaker } from "./circuit-breaker";

const breakers = new Map<string, CircuitBreaker>();

export function getCircuitBreaker(agentId: string, config?: {
  failureThreshold?: number;
  resetTimeoutMs?: number;
  halfOpenMaxAttempts?: number;
}): CircuitBreaker {
  const existing = breakers.get(agentId);
  if (existing) return existing;

  const breaker = new CircuitBreaker({
    failureThreshold: config?.failureThreshold ?? 5,
    resetTimeoutMs: config?.resetTimeoutMs ?? 30_000,
    halfOpenMaxAttempts: config?.halfOpenMaxAttempts ?? 3,
    onStateChange: (from, to, stats) => {
      console.log(JSON.stringify({
        level: to === "open" ? "error" : "info",
        event: "circuit_breaker_change",
        agentId,
        from,
        to,
        consecutiveFailures: stats.consecutiveFailures,
      }));
    },
  });

  breakers.set(agentId, breaker);
  return breaker;
}

export function getAllCircuitStates(): Record<string, { state: string; stats: ReturnType<CircuitBreaker["getStats"]> }> {
  const result: Record<string, { state: string; stats: ReturnType<CircuitBreaker["getStats"]> }> = {};
  for (const [agentId, breaker] of breakers) {
    result[agentId] = { state: breaker.getState(), stats: breaker.getStats() };
  }
  return result;
}
```

## Usage with Agent Calls

```typescript
// lib/agents/agent-caller.ts
import { getCircuitBreaker } from "./circuit-registry";
import { CircuitBreakerError } from "./circuit-breaker";

export async function callAgent(
  agentId: string,
  endpoint: string,
  payload: Record<string, unknown>,
  correlationId: string
): Promise<{ result: Record<string, unknown> | null; error: string | null }> {
  const breaker = getCircuitBreaker(agentId);

  try {
    const result = await breaker.execute(async () => {
      const controller = new AbortController();
      const timeoutId = setTimeout(() => controller.abort(), 10_000);

      const response = await fetch(endpoint, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "X-Correlation-Id": correlationId,
        },
        body: JSON.stringify(payload),
        signal: controller.signal,
      });

      clearTimeout(timeoutId);

      if (!response.ok) {
        throw new Error(`Agent ${agentId} returned HTTP ${response.status}`);
      }

      return response.json();
    });

    return { result, error: null };
  } catch (err) {
    if (err instanceof CircuitBreakerError) {
      return { result: null, error: `Circuit breaker open for agent ${agentId}: ${err.message}` };
    }
    if (err instanceof DOMException && err.name === "AbortError") {
      return { result: null, error: `Agent ${agentId} timed out after 10s` };
    }
    return { result: null, error: err instanceof Error ? err.message : "Unknown error" };
  }
}
```
