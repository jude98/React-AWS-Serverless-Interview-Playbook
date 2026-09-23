# Large-Scale Frontend System Design React at 10M to 1B Users

> [!abstract] High-Level Architectural Thesis
> 
> Scaling a frontend application to tens of millions of daily active users (and hundreds of millions to a billion total users) is not about micro-optimizing component renders; it is an **infrastructure, delivery, resilience, and operational observability challenge**. The system must shift execution to the edge, treat client-side bundles as versioned distributed deployments, aggressively eliminate main-thread bottlenecks, isolate failures, and protect backend origins from thundering herds.
> 
>   

## Key Concepts

- **Edge-First Delivery Topology**: Cache static content and HTML shells at CDN points of presence (PoPs). Dynamic data uses Edge Workers (Cloudflare Workers, Fastly Compute) for geo-routing, header transformations, and personalized edge rendering/streaming (RSC).

- **Zero-Origin CDN Strategy**: Long-lived asset hashing (`Cache-Control: public, max-age=31536000, immutable`), Stale-While-Revalidate (`SWR`), and origin shields prevent traffic spikes from reaching your origin servers.

- **Hierarchical Code Splitting**: Granular chunking by route, heavy feature (e.g., charts, editors), user permission level, and locale rather than monolithic application bundles.

- **Traffic Shaping & Ingestion Resilience**: Client-side circuit breakers, exponential backoff with full jitter, request batching/deduplication, and progressive degradation during backend degradation.

- **Fault Domain Isolation**: Multi-tiered Error Boundaries with blast-radius containment, feature flags, kill switches, and graceful fallbacks (micro-frontends or modular sub-apps).

- **DOM Virtualization & Memory Management**: Virtualized DOM trees for large lists, Web Workers for expensive compute (data transforms, parsing), and explicit cleanup of event listeners and abortable network signals.

- **Observability at Scale**: RUM (Real User Monitoring) sampled telemetry (0.1%–1% at scale) tracking Core Web Vitals, custom user action timings, error budgets, and network failure distributions.

## Common Interview Questions

- How do you design an asset delivery and caching architecture to handle 10M+ DAU without taking down your origin servers?

- How do you structure route-level and component-level code splitting to prevent high initial load times and cache invalidation churn?

- How do you prevent a cascading backend outage when millions of client applications reconnect simultaneously after a network blip (thundering herd)?

- How would you design an Error Boundary architecture to guarantee zero full-page white screens?

- How do you manage real-time updates and high-frequency data ingestion (e.g., WebSocket feeds) in React without locking up the browser main thread?

- How do you implement Real User Monitoring (RUM) and client logging at billion-user scale without degrading performance or blowing telemetry budgets?

## Strong Answers / Talking Points

### 1. Delivery & Caching Infrastructure (Edge & CDN)

- **Asset Stratification**:

    - `index.html`: `Cache-Control: public, max-age=0, must-revalidate` (or edge-rendered with brief `s-maxage` and `stale-while-revalidate`). Validated via `ETag`.

    - JS/CSS/Media chunks: Bundled with content hashes (e.g., `app.[contenthash].js`). Set to `public, max-age=31536000, immutable`.

- **Multi-CDN & Anycast Routing**:

    - Use Anycast DNS to route users to the nearest PoP.

    - Multi-CDN routing (e.g., Route 53 latency-based routing between Cloudflare and Fastly) provides vendor failover.

    - Origin Shielding: Sits between CDN edge PoPs and application origins to collapse redundant cache misses into a single origin request.

- **Edge Compute (Workers)**:

    - Perform A/B test bucket assignment, user-agent parsing, geo-routing, and security authentication checks at the CDN edge before requests hit core microservices.

### 2. Code Splitting & Chunking Strategy

- **Granular Chunk Split Points**:

    1. _Route-Level_: Each top-level route is loaded via dynamic `import()` wrapped in `React.lazy()` and `Suspense`.

    2. _Vendor Chunk Separation_: Isolate core frameworks (`react`, `react-dom`, `scheduler`) into a single persistent vendor chunk that invalidates rarely.

    3. _Feature-Based Dynamic Imports_: Heavy dependencies (Monaco Editor, Chart.js, PDF renderers) are only imported when user triggers an explicit UI intent (hover, click, or intersecting viewport).

    4. _Locale-Based Dynamic Imports_: Translation dictionaries (`i18n`) split per language/region and loaded on demand.

- **Avoid Over-Splitting**: Too many micro-chunks causes HTTP/2 or HTTP/3 multiplexing overhead and waterfall requests. Optimize chunk size boundaries to a sweet spot (50kB–150kB compressed).

### 3. API Ingestion, Traffic Shaping & Network Resilience

- **Request Deduplication & SWR**:

    - Global query client (e.g., TanStack Query) with `staleTime` tuned to avoid redundant fetches on window refocus.

    - Coalesce identical in-flight network requests into a single promise.

- **Thundering Herd Mitigation**:

    - Implement full jitter with exponential backoff on retries: $T = \text{random}(0, \min(M, B \times 2^{\text{attempt}}))$.

    - WebSockets: Do not reconnect all clients at the same millisecond when a gateway resets; introduce random reconnection spreads across seconds/minutes.

- **Offline & Optimistic Mutations**:

    - Use standard IndexedDB via lightweight wrappers (e.g., `idb`) for persistent offline write-ahead logs and background sync.

### 4. Blast Radius Containment & Error Architecture

- **Hierarchical Error Boundaries**:

    - _Root Level_: Displays global recovery page with telemetry reporting and hard reload options.

    - _Layout / Navigation Level_: Retains navigation bars, menus, and sidebars intact while resetting content areas.

    - _Widget Level_: Individual dashboard cards, tables, or feeds display isolated error states with individual "Retry" buttons, preventing one broken widget from taking down the screen.

- **Stale Chunk Error Handling**:

    - When a new deployment invalidates old chunks, users on older sessions may trigger `ChunkLoadError` upon navigation. Intercept dynamic import failures to trigger a controlled background page update.

- **Remote Kill Switches**:

    - Use feature flagging (LaunchDarkly, Statsig) with edge-cached evaluations to turn off broken or heavy modules instantly without running a full CI/CD deployment.

### 5. Large-Scale Data Rendering & Virtualization

- **Windowing / Virtualization**:

    - Render only the DOM nodes currently visible within the viewport plus an overscan buffer (using `@tanstack/react-virtual` or `react-window`).

    - Keeps DOM node count bounded (<1,500 nodes) regardless of whether the dataset contains 10 rows or 1,000,000 rows.

- **Offloading Computations**:

    - Web Workers handle heavy data aggregation, mathematical operations, CSV/JSON parsing, and search indexing.

    - `SharedArrayBuffer` or Transferable objects eliminate copy overhead for large memory transfers between worker and main thread.

### 6. Client Observability & Telemetry at Hyper-Scale

- **Dynamic Sampling**:

    - At 10M+ DAU, logging 100% of events will overwhelm logging backends (Elastic, Datadog) and drain user bandwidth.

    - Sample successful transactions at 0.1%–1%; sample fatal errors at 100%.

- **Beacon API (`navigator.sendBeacon`)**:

    - Offload analytics payloads asynchronously during page unloading without delaying user transitions.

## Code Snippets / Examples

### Resilience: Jittered Retry with Exponential Backoff

```typescript
interface RetryOptions {
  maxRetries?: number;
  baseDelayMs?: number;
  maxDelayMs?: number;
}

export async function fetchWithJitter<T>(
  url: string,
  options: RequestInit = {},
  retryOpts: RetryOptions = {}
): Promise<T> {
  const { maxRetries = 3, baseDelayMs = 300, maxDelayMs = 5000 } = retryOpts;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(url, options);
      if (!response.ok) {
        // Only retry on server errors or rate limits
        if (response.status >= 500 || response.status === 429) {
          throw new Error(`Server error: ${response.status}`);
        }
        throw new Error(`Client error: ${response.status}`);
      }
      return (await response.json()) as T;
    } catch (error) {
      if (attempt === maxRetries) throw error;

      // Exponential backoff with full jitter to avoid thundering herd
      const exponentialDelay = Math.min(maxDelayMs, baseDelayMs * Math.pow(2, attempt));
      const jitteredDelay = Math.random() * exponentialDelay;

      await new Promise((resolve) => setTimeout(resolve, jitteredDelay));
    }
  }
  throw new Error('Unreachable retry state');
}
```

### Hierarchical Error Boundary with Chunk Recovery

```typescript
import React, { Component, ErrorInfo, ReactNode } from 'react';

interface Props {
  fallbackTitle: string;
  children: ReactNode;
}

interface State {
  hasError: boolean;
  isChunkError: boolean;
}

export class ResilientErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false, isChunkError: false };

  static getDerivedStateFromError(error: Error): State {
    const isChunk =
      error.name === 'ChunkLoadError' ||
      /Loading chunk [\d]+ failed/i.test(error.message);
    return { hasError: true, isChunkError: isChunk };
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    // Report to sampled RUM collector via non-blocking Beacon
    if (Math.random() < 0.1) {
      navigator.sendBeacon?.(
        '/api/telemetry/errors',
        JSON.stringify({ error: error.message, stack: info.componentStack })
      );
    }
  }

  handleRecovery = () => {
    if (this.state.isChunkError) {
      // Force reload to pull new index.html referencing active build hashes
      window.location.reload();
      return;
    }
    this.setState({ hasError: false, isChunkError: false });
  };

  render() {
    if (this.state.hasError) {
      return (
        <div className="rounded-lg border border-red-200 bg-red-50 p-4 text-red-900">
          <h3 className="font-semibold">{this.props.fallbackTitle}</h3>
          <p className="text-sm text-red-700 mt-1">
            {this.state.isChunkError
              ? 'A newer version of the application is available.'
              : 'This section failed to load.'}
          </p>
          <button
            onClick={this.handleRecovery}
            className="mt-3 px-3 py-1.5 text-xs bg-red-600 text-white rounded hover:bg-red-700"
          >
            {this.state.isChunkError ? 'Refresh Application' : 'Retry'}
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}
```

### High-Volume Viewport Virtualization

```typescript
import React, { useRef } from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';

interface VirtualListProps<T> {
  items: T[];
  renderItem: (item: T, index: number) => React.ReactNode;
  estimatedItemHeight?: number;
}

export function HighScaleVirtualList<T>({
  items,
  renderItem,
  estimatedItemHeight = 48,
}: VirtualListProps<T>) {
  const parentRef = useRef<HTMLDivElement | null>(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => estimatedItemHeight,
    overscan: 5, // Render 5 off-screen nodes to prevent scrolling blanks
  });

  return (
    <div
      ref={parentRef}
      className="h-[600px] overflow-auto border border-gray-200 rounded"
      style={{ contain: 'strict' }} // CSS containment isolates layout recalculation
    >
      <div
        className="w-full relative"
        style={{ height: `${virtualizer.getTotalSize()}px` }}
      >
        {virtualizer.getVirtualItems().map((virtualRow) => (
          <div
            key={virtualRow.key}
            className="absolute top-0 left-0 w-full"
            style={{
              height: `${virtualRow.size}px`,
              transform: `translateY(${virtualRow.start}px)`,
            }}
          >
            {renderItem(items[virtualRow.index], virtualRow.index)}
          </div>
        ))}
      </div>
    </div>
  );
}
```

## Related Topics

- [[Caching Architecture, Eviction Policies & Invalidation Pitfalls|CDN and Edge Architecture Cloudflare Fastly]]

- [[Comprehensive Performance Optimization Architecture in React|React Performance Optimization]]

- [[High-Scale Data Table Architecture Handling Millions of Records|Virtualization and Large Data Rendering]]

- [[Browser Workers Architecture. Dedicated, Shared, Service & Worklets|Web Workers and Off-Main-Thread Processing]]

- [[AWS Observability - CloudWatch, AWS X-Ray & CloudTrail|Distributed Tracing and Client Side RUM]]

- [[Monorepo and Micro-Frontend Architecture Evaluation|Resilient Micro Frontends Architecture]]

## Tags

#fullstack #interview #system-design #frontend-scale #react-architecture #high-availability

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups