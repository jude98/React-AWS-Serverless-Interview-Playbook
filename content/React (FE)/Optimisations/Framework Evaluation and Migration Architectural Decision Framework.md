# Framework Evaluation and Migration Architectural Decision Framework

> [!abstract] Architectural Decision Matrix
> 
> Choosing or migrating a frontend framework is not merely a syntax preference; it is a fundamental trade-off across **Rendering Architecture (CSR vs. SSR/SSG/ISR/RSC)**, **Bundle Budget & Runtime Overhead**, **Ecosystem & Community Longevity**, **Hiring & Developer Experience (DX)**, and **Total Cost of Ownership (Infra & Operational Complexity)**.
> 
>   

## Key Concepts

- **Mental Model & Component Model**: React/Preact use functional unidirectional data flow with synthetic VDOM reconciliation; Angular uses a TypeScript-first, object-oriented, decorator-driven architecture with zone-less or signals-based change detection and dependency injection.

- **Rendering Paradigms**:

    - Pure React (Vite/SPA): Exclusively Client-Side Rendering (CSR). Empty initial HTML payload; JavaScript hydrates and renders everything on the client.

    - Next.js: Hybrid framework supporting Server Components (RSC), Server-Side Rendering (SSR), Static Site Generation (SSG), and Incremental Static Regeneration (ISR) at the route and component levels.

- **Runtime Footprint**:

    - React + ReactDOM: ~45–50kB gzip baseline runtime before application code.

    - Preact: ~3kB gzip dropped-in replacement with identical VDOM semantics via `preact/compat`.

    - Angular: Highly batteries-included (Router, Forms, HTTP client, RxJS/Signals) resulting in a larger base runtime, offset by integrated build-time tree-shaking and compilation.

- **Infrastructure Coupling**: SPA React deployable on cheap static object stores (AWS S3, Cloudflare Pages); Next.js requires Node.js servers, serverless/edge runtimes (Vercel, AWS Lambda), or custom cache storage adapters for dynamic features.

## Common Interview Questions

- Under what concrete technical constraints would you migrate a React SPA (Vite) to Next.js?

- Why would an engineering team choose Preact over React, and what subtle edge cases break when using `preact/compat`?

- Contrast the state management and change detection philosophies of modern Angular (Signals) versus React.

- How does the total cost of ownership (TCO) and infrastructure footprint change when moving from a pure React SPA to Next.js?

- If an enterprise organization wants to convert a large React codebase to Angular, what are the primary architectural migration patterns (e.g., Module Federation, Web Components, strangler fig)?

## Strong Answers / Talking Points

### 1. React SPA vs. Next.js (Why Pick Next.js?)

- **When to choose Next.js**:

    - **SEO & Social Crawlers**: Public e-commerce, content platforms, and marketing pages where search indexing and dynamic Open Graph images directly determine revenue.

    - **Core Web Vitals & FCP**: Moving data fetching and initial rendering to the server eliminates client-side network waterfalls and allows streaming HTML directly to the browser.

    - **React Server Components (RSC)**: Heavy dependencies (markdown parsers, sanitization libraries, date formatters) remain on the server and send 0kB of JavaScript down to the client.

- **When to stick with pure React (Vite)**:

    - **B2B / Auth-Gated Dashboards**: Applications behind a login screen where SEO is irrelevant and the entire app lifecycle is long-running and state-heavy.

    - **Low Infrastructure Complexity**: SPAs are statically deployable to any CDN edge/bucket at negligible hosting cost without managing Node runtimes, serverless cold starts, or cache invalidation layers.

### 2. React to Preact (Why and What Breaks?)

- **Why Migrate**:

    - **Extreme Bundle Constraints**: Embedded widgets, third-party script integrations, low-end mobile devices in emerging markets, or micro-frontends where every kilobyte directly hurts conversion rates.

    - Drops base runtime from ~45kB to ~3kB.

- **Considerations & Trade-offs**:

    - **Synthetic Events**: Preact uses native browser DOM events instead of React’s synthetic event system. Subtle differences in event bubbling order (e.g., `onChange` vs. `onInput`) can introduce regressions.

    - **Ecosystem Compatibility**: While `preact/compat` aliases standard React libraries, complex third-party packages relying on private React internals (e.g., deep React Fiber inspection, advanced React 18/19 concurrent primitives) can fail.

### 3. React to Angular (Paradigm Shift & Trade-offs)

- **Why Choose Angular**:

    - **Enterprise Standardization**: Angular is an opinionated, complete platform. It ships built-in routing, forms validation, HTTP clients, and dependency injection out of the box. Teams spend zero time debating state libraries, linting setups, or directory topologies.

    - **Strict Architectural Boundaries**: Angular’s class-based services and dependency injection make it well-suited for large, distributed enterprise teams needing strict contract enforcement and test isolation.

- **Considerations & Migration Friction**:

    - Complete mental model overhaul: JSX/hooks are replaced by Angular templates, signals/RxJS observables, and dependency injection tokens.

    - Cannot be done via a simple drop-in bridge; requires either a Strangler Fig pattern using Micro-Frontends (Module Federation) or wrapping isolated components inside framework-agnostic Web Components (`Custom Elements`).

### 4. High-Level Decision Matrix

|**Dimension**|**Pure React (Vite SPA)**|**Next.js (App Router)**|**Preact**|**Angular**|
|---|---|---|---|---|
|**Primary Use Case**|Auth-gated SaaS dashboards, internal tooling|SEO-critical apps, e-commerce, content portals|Performance-critical widgets, mobile-first sites|Large enterprise systems, standardized teams|
|**Base Bundle Overhead**|~45kB gzip|Variable (Server-driven, 0kB RSC potential)|~3kB gzip|~60kB–100kB+ gzip (Full platform baseline)|
|**Rendering Strategy**|CSR only|SSR, SSG, ISR, RSC, Streaming|CSR (can SSR with custom node setups)|SSR (Angular Universal), SSG, CSR|
|**Hosting & Infra**|Static CDN / S3|Node.js runtime / Serverless / Edge|Static CDN / S3|Static CDN (CSR) or Node.js (SSR)|
|**Learning Curve**|Low / Moderate|Moderate / High (RSC mental model)|Low (identical to React)|Steep (TypeScript, DI, RxJS, Templates)|

## Code Snippets / Examples

### Preact Alias Configuration (Vite)

```typescript
// vite.config.ts: Drop-in replacement of React with Preact
import { defineConfig } from 'vite';
import preact from '@preact/preset-vite';

export default defineConfig({
  plugins: [preact()],
  resolve: {
    alias: {
      // Direct all React imports to preact/compat bridge
      react: 'preact/compat',
      'react-dom': 'preact/compat',
      'react/jsx-runtime': 'preact/jsx-runtime',
    },
  },
});
```

### Next.js Server Component (Zero-Bundle Shipping)

```typescript
// app/metrics/page.tsx - React Server Component (RSC)
// Heavy libraries (marked, db-client) run on server and NEVER enter the client JS bundle.
import { marked } from 'marked';
import db from '@/lib/db';

export default async function MetricsPage() {
  // Direct DB call on server - no client-side waterfall, no public API exposure
  const report = await db.reports.findFirst({ orderBy: { createdAt: 'desc' } });
  const renderedContent = marked(report?.content || '');

  return (
    <article className="prose max-w-4xl mx-auto p-6">
      <h1>Server-Rendered Analysis</h1>
      {/* Ships pure static HTML to browser with zero client JS overhead */}
      <div dangerouslySetInnerHTML={{ __html: renderedContent }} />
    </article>
  );
}
```

## Related Topics

- [[Web Vitals Optimization LCP INP and FCP|Core Web Vitals LCP FID CLS]]

- [[Comprehensive Performance Optimization Architecture in React|React Performance Optimization]]

- [[Web Vitals Optimization LCP INP and FCP]]

- [[Large-Scale Frontend System Design React at 10M to 1B Users]]

- [[React State Management Architecture, Context vs External Stores vs React Query|Client State vs Server State Architecture]]

## Tags

#fullstack #interview #framework-selection #nextjs #react #preact #angular #software-architecture

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups