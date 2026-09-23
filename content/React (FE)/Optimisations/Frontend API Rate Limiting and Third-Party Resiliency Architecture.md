# Frontend API Rate Limiting and Third-Party Resiliency Architecture

> [!abstract] Resiliency Principle
>
> The frontend must never be a passive victim of rate limits (`429 Too Many Requests`) or third-party downstream outages. The architecture must adopt **proactive client-side throttling/queuing**, **reactive backoff with full jitter**, **graceful UI degradation with blast-radius containment**, and **intelligent telemetry sampling** (never spamming monitoring systems with predictable rate-limit errors).


## Key Concepts

- **429 Anatomy & Headers**: Inspect `Retry-After` (seconds or HTTP date), `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset` to pace client dispatch loops.

- **Client-Side Throttling & Token Buckets**: Limit outbound request frequency before hitting the wire (e.g., token bucket or leaky bucket algorithm on autocomplete, analytics, or batch syncs).

- **Exponential Backoff with Full Jitter**: Prevents the client-side thundering herd problem:

    $$T = \text{random}(0, \min(M, B \times 2^{\text{attempt}}))$$

- **Request Coalescing & Deduplication**: Collapse identical in-flight promises into a single network call; debounce/throttle user-triggered operations.

- **Circuit Breaker Pattern**: If a third-party service fails repeatedly (consecutive 429s or 5xx), open the circuit to fast-fail subsequent requests on the client without touching the network.

- **Telemetry & Logging Hygiene**: Do not flood error aggregators (e.g., Sentry, Datadog) with expected rate-limit warnings; aggregate, sample, or log them as handled warnings with operational tags.

## Common Interview Questions

- How do you design an HTTP client interceptor to handle HTTP 429 errors automatically across an enterprise React application?

- Should frontend applications directly integrate third-party APIs (e.g., Stripe, Algolia, Google Maps) or route through a Backend-for-Frontend (BFF)?

- Do you log 429 errors to Sentry/monitoring tools? Why or why not, and how do you prevent telemetry exhaustion?

- How does the Circuit Breaker pattern function inside a browser environment?

- How do you communicate rate limiting to end-users without causing confusion or panic?

- What are the trade-offs between client-side request queues and optimistic UI updates when rate limits are active?

## Strong Answers / Talking Points

### 1. Direct Third-Party APIs vs. Backend-for-Frontend (BFF)

- **Direct Client Integration**:

    - _Use Case_: Specialized search (Algolia), client SDKs (Stripe Elements), or Maps where client IP/token authentication is native.

    - _Risk_: Exposes quota exhaustion, leaks usage patterns, and makes request throttling dependent on individual client behavior.

- **BFF / Reverse Proxy Pattern (Recommended)**:

    - Route third-party calls through an internal gateway/proxy (`/api/v1/third-party/...`).

    - Enables server-level Redis-backed caching, global request pooling, secret masking, and response sanitization.

### 2. The Multi-Tier Handling Strategy

1. **Preventative (Proactive)**:

    - Debounce search and autocomplete inputs (300–400ms).

    - Use TanStack Query with calibrated `staleTime` and deduplication so unmounted/remounted components reuse cached responses.

    - Run background telemetry through `navigator.sendBeacon` or a client-side queue that flushes batches periodically.

2. **Reactive (On 429 or Failure)**:

    - Read `Retry-After` response header. If present, pause outbound requests for that service until the timestamp passes.

    - If no header is present, fallback to exponential backoff with full jitter.

3. **Circuit Breaker**:

    - Track failure thresholds (e.g., 5 consecutive 429s or timeouts within 30 seconds).

    - Flip to `OPEN` state for a cooling-off window (e.g., 60 seconds). During this time, immediately return cached data, fallback mocks, or empty states without making network calls.

### 3. Do We Throw or Log Errors? (Telemetry Strategy)

- **Do NOT throw unhandled exceptions**:

    - Unhandled rejections trigger root-level error boundaries, causing whole-screen crashes over a rate-limited widget.

- **Do NOT log every 429 as a critical alert in Sentry**:

    - Expected 429s create alert fatigue and exhaust telemetry quotas.

    - **Pattern**: Catch 429s at the HTTP interceptor level. Downgrade them to `logger.warn` or sample them at 1–5% with contextual metadata (`endpoint`, `retryCount`, `resetTime`).

- **User-Facing UI State**:

    - Instead of generic toasts like `"Error: Request failed"`, show clear actionable feedback: _"High traffic: updates paused for 15 seconds. [Retry Now]"_.

    - Disable spam-prone trigger buttons and display a countdown timer matching `Retry-After`.

## Code Snippets / Examples

### Resilient Fetch Wrapper with `Retry-After` and Full Jitter

```typescript
interface RequestConfig extends RequestInit {
  maxRetries?: number;
  baseDelayMs?: number;
}

export async function resilientFetch<T>(
  url: string,
  config: RequestConfig = {}
): Promise<T> {
  const { maxRetries = 3, baseDelayMs = 500, ...fetchOptions } = config;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    const response = await fetch(url, fetchOptions);

    if (response.ok) {
      return (await response.json()) as T;
    }

    if (response.status === 429 && attempt < maxRetries) {
      // 1. Check for standard Retry-After header (in seconds or date string)
      const retryAfterHeader = response.headers.get('Retry-After');
      let delayMs: number;

      if (retryAfterHeader) {
        const parsedSeconds = parseInt(retryAfterHeader, 10);
        delayMs = isNaN(parsedSeconds)
          ? Math.max(0, new Date(retryAfterHeader).getTime() - Date.now())
          : parsedSeconds * 1000;
      } else {
        // 2. Fallback to exponential backoff with full jitter
        const maxBackoff = baseDelayMs * Math.pow(2, attempt);
        delayMs = Math.random() * maxBackoff;
      }

      // 3. Log warning to telemetry (sampled/sanitized, not logged as an unhandled error)
      if (Math.random() < 0.05) {
        console.warn(`[RateLimit] 429 on ${url}. Retrying in ${Math.round(delayMs)}ms`);
      }

      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    // Throw handled domain error for UI consumption
    throw new Error(`Request failed with status ${response.status}`);
  }

  throw new Error(`Exceeded max retries for ${url}`);
}
```

### Browser Client-Side Circuit Breaker

```typescript
type CircuitState = 'CLOSED' | 'OPEN' | 'HALF_OPEN';

export class ClientCircuitBreaker {
  private state: CircuitState = 'CLOSED';
  private failureCount = 0;
  private nextAttempt = Date.now();

  constructor(
    private threshold: number = 4,      // Number of 429s/failures to trip
    private timeoutMs: number = 30000    // Cooldown duration
  ) {}

  public canRequest(): boolean {
    if (this.state === 'OPEN') {
      if (Date.now() > this.nextAttempt) {
        this.state = 'HALF_OPEN';
        return true;
      }
      return false;
    }
    return true;
  }

  public recordSuccess(): void {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }

  public recordFailure(): void {
    this.failureCount += 1;
    if (this.failureCount >= this.threshold || this.state === 'HALF_OPEN') {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.timeoutMs;
      console.warn(`[CircuitBreaker] Circuit TRIPPED to OPEN until ${new Date(this.nextAttempt).toISOString()}`);
    }
  }
}
```

### Degraded UI State with Blast-Radius Isolation

```typescript
import React, { useState } from 'react';
import { useQuery } from '@tanstack/react-query';

export const ThirdPartyStockWidget = () => {
  const [isThrottled, setIsThrottled] = useState(false);

  const { data, refetch, isFetching } = useQuery({
    queryKey: ['live-feed'],
    queryFn: async () => {
      const res = await fetch('/api/proxy/stock-feed');
      if (res.status === 429) {
        setIsThrottled(true);
        throw new Error('RATE_LIMITED');
      }
      setIsThrottled(false);
      return res.json();
    },
    retry: false, // Handle retries through resilient interceptor or explicit user action
  });

  return (
    <div className="p-4 border rounded-md shadow-sm">
      <div className="flex justify-between items-center mb-2">
        <h4 className="font-semibold text-sm">Market Feed</h4>
        {isThrottled && (
          <span className="text-xs bg-amber-100 text-amber-800 px-2 py-0.5 rounded">
            Live updates paused (Rate limited)
          </span>
        )}
      </div

      {data ? (
        <div>Value: ${data.price}</div>
      ) : (
        <div className="text-gray-400 text-sm">Showing cached snapshot</div>
      )}

      <button
        disabled={isFetching || isThrottled}
        onClick={() => refetch()}
        className="mt-3 text-xs px-2.5 py-1 bg-gray-100 hover:bg-gray-200 rounded disabled:opacity-50"
      >
        {isFetching ? 'Refreshing...' : 'Retry Refresh'}
      </button>
    </div>
  );
};
```

## Related Topics

- [[Large-Scale Frontend System Design React at 10M to 1B Users]]

- [[TanStack Query Server State and Stale While Revalidate Patterns]]

- [[AWS Observability - CloudWatch, AWS X-Ray & CloudTrail|Distributed Tracing and Client Side RUM]]

- [[Frontend API Rate Limiting and Third-Party Resiliency Architecture|API Gateway Rate Limiting and Leaky Bucket Algorithms]]

- [[Clean Architecture, Directory Structure & DTOs|Backend for Frontend BFF Architecture]]

## Tags

#fullstack #interview #rate-limiting #resilience #circuit-breaker #api-design #error-handling

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups
