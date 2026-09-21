

> [!abstract] High-Level Diagnostic Flow
> 
> Triage performance systematically from outside to inside: **Network & Delivery (TTFB, Pre-flights, Payload)** $\rightarrow$ **Core Web Vitals & Rendering (LCP, INP, CLS)** $\rightarrow$ **Bundle Analysis (Code Splitting, Tree-Shaking)** $\rightarrow$ **Application & State (Rerenders, API Chaining, Virtualization)**. Never optimize blindly without profiling first.
> 
>   

## Key Concepts

- **Metrics Attribution**: Determine whether the latency is caused by **Server/Network** (high TTFB, waterfall requests), **Resource Delivery** (massive bundle size, blocking assets), or **Client Execution** (heavy JS execution, layout thrashing, unvirtualized tables).
    
      
    
- **Network Profiling**: Inspect Waterfall, Request Queuing, DNS/TLS handshake overhead, OPTIONS preflight latency, and gzip/brotli transfer compression.
    
      
    
- **Web Vitals Signals**:
    
      
    - **LCP high**: Slow server response, render-blocking resources, or late-discovered hero components.
        
          
        
    - **INP / TBT high**: Heavy main-thread execution blocking UI responsiveness during hydration or data ingestion.
        
          
        
    - **CLS high**: Dynamic charts or widgets injecting into DOM without reserved bounding boxes.
        
          
        
- **API Call Patterns**: Catch request waterfalls (`useEffect` inside child components), over-fetching, unbatched queries, and cache misses.
    
      
    
- **Render Bottlenecks**: Identify unbounded lists (e.g., thousands of rows/cards without virtualization) and excessive rerenders during dashboard state broadcasts.
    
      
    

## Common Interview Questions

- A user reports that the dashboard takes 8 seconds to load. Walk me through your step-by-step diagnostic workflow.
    
      
    
- How do you diagnose whether slow loading is caused by API latency versus front-end rendering performance?
    
      
    
- What causes excessive CORS pre-flight requests, and how can you eliminate or cache them?
    
      
    
- How do you detect and fix request waterfalls in a deeply nested React dashboard?
    
      
    
- What tools and strategies do you use to locate dead code or bloated third-party dependencies in production bundles?
    
      
    
- How do you optimize large analytics dashboards containing multiple real-time charts and data tables?
    
      
    

## Strong Answers / Talking Points

### 1. Step-by-Step Diagnostic Hierarchy

1. **Network Tab (The Outer Perimeter)**:
    
      
    - **TTFB (Time to First Byte)**: If HTML or core bootstrap API has >500ms TTFB, issue lies in backend processing, DB queries, or cold serverless starts.
        
          
        
    - **Pre-flight (OPTIONS) Overhead**: High latency on pre-flights indicates missing CORS cache headers (`Access-Control-Max-Age`) or non-simple headers triggering unneeded pre-flights on every request.
        
          
        
    - **Waterfall Analysis**: Look for serial sequential fetching (Parent renders $\rightarrow$ fetches $\rightarrow$ child renders $\rightarrow$ fetches).
        
          
        
    - **Asset Sizes**: Check `Content-Encoding` (is Brotli/Gzip active?) and check if JS vendor chunks exceed 300–500kB compressed.
        
          
        
2. **Performance & Profiler Tab (The Engine Room)**:
    
      
    - Run a CPU-throttled profile (4x slowdown).
        
          
        
    - Check **Long Tasks (>50ms)** blocking the main thread during hydration or initial mount.
        
          
        
    - Use React DevTools Profiler to detect expensive renders and cascaded updates caused by root context providers.
        
          
        
3. **Core Web Vitals & Lighthouse**:
    
      
    - Determine whether **LCP** is blocked by client-side data fetching or heavy charting dependencies (e.g., importing the entire `lodash` or `echarts` upfront).
        
          
        

### 2. Concrete Fixes & Remediation Strategies

- **For Slow APIs & Waterfalls**:
    
      
    - Parallelize independent queries via `Promise.all` or parallel React Query/RTK Query hooks.
        
          
        
    - Colocate prefetching at router level (e.g., TanStack Router loaders or Next.js route prefetching).
        
          
        
    - Enable HTTP caching via `ETag` / `Cache-Control: stale-while-revalidate`.
        
          
        
- **For Pre-flight Overhead**:
    
      
    - Set `Access-Control-Max-Age: 86400` on the API gateway/backend to cache OPTIONS requests for 24 hours.
        
          
        
    - Serve frontend and API behind the same reverse proxy / domain (e.g., `/api/*`) to eliminate cross-origin requests entirely.
        
          
        
- **For Massive Bundles**:
    
      
    - Analyze using `rollup-plugin-visualizer` or `@next/bundle-analyzer`.
        
          
        
    - Lazy load heavy charting libraries (`react-window`, `recharts`, `chart.js`) via `React.lazy()` and `Suspense`.
        
          
        
    - Check imports: switch default monolithic imports (`import { map } from 'lodash'`) to path-based or tree-shakeable modular imports (`lodash-es`).
        
          
        
- **For Client-Side DOM Bloat**:
    
      
    - Virtualize large tables and activity feeds using `@tanstack/react-virtual`.
        
          
        
    - Skeleton screens with fixed dimensions to hold space and eliminate CLS.
        
          
        
    - Defer rendering non-critical off-screen widgets until scrolled into view using `IntersectionObserver`.
        
          
        

## Code Snippets / Examples

### Eliminating Waterfalls via Parallel Prefetching

```TypeScript
// ❌ Bad: Waterfalling useEffects in parent and child
// Parent renders, awaits API, then mounts Child, which awaits another API.

// ✅ Good: Parallel orchestration using TanStack Query / Promise.all
import { useQueries } from '@tanstack/react-query';
import React, { Suspense } from 'react';

export const DashboardLoader = () => {
  const [metricsQuery, feedQuery] = useQueries({
    queries: [
      { queryKey: ['metrics'], queryFn: fetchDashboardMetrics, staleTime: 60_000 },
      { queryKey: ['feed'], queryFn: fetchActivityFeed, staleTime: 30_000 },
    ],
  });

  if (metricsQuery.isLoading || feedQuery.isLoading) {
    return <DashboardSkeleton />;
  }

  return (
    <div className="grid grid-cols-2 gap-4">
      <MetricsWidget data={metricsQuery.data} />
      <FeedWidget data={feedQuery.data} />
    </div>
  );
};
```

### Lazy-Loading Heavy Charting Packages with Suspense

```TypeScript
import React, { lazy, Suspense } from 'react';

// Defer heavy chart bundle until requested
const AnalyticsChart = lazy(() => 
  import(/* webpackChunkName: "analytics-chart" */ './AnalyticsChart')
);

export const DashboardView = () => {
  return (
    <div className="dashboard-container">
      <header className="h-16">Fast-rendering Summary Cards</header>
      
      {/* Chart chunk is fetched on-demand without blocking initial page interactive time */}
      <Suspense fallback={<div className="h-64 animate-pulse bg-gray-200" />}>
        <AnalyticsChart />
      </Suspense>
    </div>
  );
};
```

### CORS Pre-flight Caching (Node / Express Header Fix)

```TypeScript
// Backend remediation: Cache preflight to avoid OPTIONS per API request
import express from 'express';
const app = express();

app.use((req, res, next) => {
  res.header('Access-Control-Allow-Origin', 'https://app.dashboard.com');
  res.header('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
  res.header('Access-Control-Allow-Headers', 'Content-Type, Authorization');
  
  // Cache preflight OPTIONS response for 24 hours in the browser
  res.header('Access-Control-Max-Age', '86400');

  if (req.method === 'OPTIONS') {
    return res.sendStatus(204);
  }
  next();
});
```

## Related Topics

- [[React Performance Optimization]]
    
      
    
- [[Core Web Vitals LCP FID CLS]]
    
      
    
- [[HTTP Caching and CDN Architecture]]
    
      
    
- [[Code Splitting and Dynamic Imports]]
    
      
    
- [[Virtualization and Large Data Rendering]]
    
      
    

## Tags

#fullstack #interview #dashboard-performance #debugging #network-optimization

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups