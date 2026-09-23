# Monorepo and Micro-Frontend Architecture Evaluation

> [!abstract] Architectural Thesis
> 
> **Monorepo** and **Micro-Frontends (MFEs)** solve two completely different problems and are not mutually exclusive. A Monorepo is a **code management, build, and dependency strategy** (handling _how code is stored and built_), whereas Micro-Frontends represent an **organizational and deployment strategy** (handling _how independent teams deploy and isolate runtime ownership_). You can have a Monolithic Repo with a Monolithic SPA, a Monorepo containing Micro-Frontends, or Poly-repo Micro-Frontends.
> 
>   

## Key Concepts

- **Monorepo (Build & Storage Architecture)**: A single version-controlled repository containing multiple distinct projects, apps, and shared libraries with unified tooling (e.g., Turborepo, Nx, pnpm workspaces). Enforces atomic commits, unified dependency versions, and instant refactoring visibility across packages.

- **Micro-Frontends (Deployment & Runtime Architecture)**: Decomposing a large frontend into independently deliverable, loosely coupled sub-applications owned by autonomous, cross-functional domain teams (e.g., Checkout, Catalog, Account).

- **Integration Vectors for Micro-Frontends**:

    - **Build-time Integration**: Sub-apps published as npm packages and bundled into a host shell at build time (tight coupling, slow releases, but zero runtime overhead).

    - **Runtime Client-Side Integration**: Module Federation (Webpack/Rspack/Vite) or Native ES Modules dynamically fetching remote bundles over the wire into a container shell.

    - **Runtime Routing / Edge Integration**: Reverse proxy (Nginx, Cloudflare Workers) routing distinct paths (`/checkout` vs. `/search`) to completely independent apps/SPAs.

    - **Server-Side Integration**: Edge-side fragment assembly (Edge-Side Includes / SSI / Server Component streaming).

- **The Micro-Frontend Tax**: Increased total bundle size (potential duplicate dependencies), complex cross-app state sharing, CSS isolation challenges, global navigation/routing synchronization, and testing/observability overhead.

## Common Interview Questions

- What is the difference between a Monorepo and a Micro-Frontend architecture? Can you use both together?

- Under what organizational or technical conditions should an engineering team adopt Micro-Frontends? When is it an anti-pattern?

- How does Webpack/Rspack Module Federation resolve shared singletons like React and React-DOM across independent micro-apps?

- How do you handle cross-micro-frontend communication and state synchronization without tightly coupling teams?

- How do you isolate CSS and prevent style collisions between independently developed micro-frontends?

- Why do many engineering organizations regret adopting Micro-Frontends, and what are the primary failure modes?

## Strong Answers / Talking Points

### 1. When to Choose a Monorepo

- **Indicators for Monorepo**:

    - Multiple applications share internal code (design systems, API contracts, utility types, auth helpers).

    - You want **atomic cross-project commits**: changing an API client interface and updating all consuming apps in a single atomic Pull Request.

    - Fast-moving teams needing single-source-of-truth dependency versions to prevent version drift.

    - Optimized CI/CD via build caches (Turborepo/Nx affected graphs) where unchanged projects are skipped during compilation and test runs.

- **When NOT to use Monorepo**:

    - Teams require strict access control/security firewalls preventing developers from viewing certain parts of the codebase.

    - Tooling complexity is an issue: monorepos require dedicated engineers to maintain CI pipelines, caching, and git performance at scale.

### 2. When to Choose Micro-Frontends (and When NOT to)

- **Indicators for Micro-Frontends (The Organizational Scaling Problem)**:

    - **Team Autonomy**: You have 100+ frontend engineers divided into 10+ autonomous squads. Coordination meetings and release trains for a single monolithic SPA create organizational gridlock.

    - **Independent Deployability**: Squad A (Payments) must deploy 10 times a day without waiting for Squad B (Search) to finish QA or unblock a failed integration test.

    - **Incremental Technology Migration**: Migrating a legacy Angular or AngularJS application to React incrementally using a Strangler Fig pattern route-by-route.

- **When Micro-Frontends are an Anti-Pattern**:

    - Small to mid-sized teams (<30 engineers). Adopting MFEs solely for "clean architecture" introduces massive infrastructure overhead without any organizational gain.

    - Highly interdependent UI: If changing a checkout item requires mutating state across 4 different micro-apps on the same screen, coupling makes MFEs brittle.

### 3. Combining Monorepo + Micro-Frontends (The Golden Standard)

- Instead of using 15 separate git repositories (Poly-repo MFEs) which causes dependency drift and nightmare integration debugging:

- Use a **Monorepo to house all Micro-Frontends**:

    - Shared design tokens and UI components live in `packages/ui`.

    - The Shell/Host container lives in `apps/shell`.

    - Independent domain teams own `apps/checkout`, `apps/catalog`.

    - CI independently builds and deploys only the affected micro-frontend using Module Federation artifacts stored on S3/CDN.

### 4. Technical Guardrails for Runtime Micro-Frontends

- **Dependency Sharing**: Configure Module Federation `shared` options strictly with `singleton: true` and `strictVersion: true` on libraries that fail if duplicated (`react`, `react-dom`, global state stores).

- **CSS Isolation**: Use CSS Modules, Tailwind with strict scope wrappers, or Shadow DOM to prevent global stylesheet bleed.

- **Cross-App Communication**: Never use shared mutable global state objects. Use loose event contracts: native `CustomEvent` dispatched on `window` or URL query parameters as the single source of truth.

## Code Snippets / Examples

### Module Federation Host Configuration (Rspack / Webpack)

```javascript
// apps/shell/rspack.config.js
const { ModuleFederationPlugin } = require('@rspack/core').container;

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell',
      remotes: {
        // Points to independently deployed remote micro-apps over CDN
        checkout: 'checkout@https://cdn.platform.com/apps/checkout/remoteEntry.js',
        catalog: 'catalog@https://cdn.platform.com/apps/catalog/remoteEntry.js',
      },
      shared: {
        react: { singleton: true, eager: true, requiredVersion: '^18.0.0' },
        'react-dom': { singleton: true, eager: true, requiredVersion: '^18.0.0' },
      },
    }),
  ],
};
```

### Module Federation Remote Configuration (Checkout App)

```javascript
// apps/checkout/rspack.config.js
const { ModuleFederationPlugin } = require('@rspack/core').container;

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'checkout',
      filename: 'remoteEntry.js',
      exposes: {
        // Expose specific component/widget boundary to host
        './CheckoutWidget': './src/components/CheckoutWidget.tsx',
      },
      shared: {
        react: { singleton: true, requiredVersion: '^18.0.0' },
        'react-dom': { singleton: true, requiredVersion: '^18.0.0' },
      },
    }),
  ],
};
```

### Decoupled Cross-MFE Event Bus

```typescript
// packages/shared-events/src/eventBus.ts
export interface CartItemAddedDetail {
  productId: string;
  quantity: number;
}

// Type-safe custom event contract
export function publishEvent<T>(eventName: string, detail: T): void {
  const event = new CustomEvent(eventName, { detail, bubbles: true });
  window.dispatchEvent(event);
}

export function subscribeToEvent<T>(
  eventName: string,
  callback: (detail: T) => void
): () => void {
  const handler = (e: Event) => {
    const customEvent = e as CustomEvent<T>;
    callback(customEvent.detail);
  };

  window.addEventListener(eventName, handler);
  return () => window.removeEventListener(eventName, handler);
}
```

## Related Topics

- [[Building a Multi-Team Enterprise Component Library]]

- [[Large-Scale Frontend System Design React at 10M to 1B Users]]

- [[Framework Evaluation and Migration Architectural Decision Framework]]

- [[Bundle Size Optimization and Build Analysis Architecture in React (Vite & Rollup)|Webpack and Vite Asset Bundling]]

- [[AWS Observability - CloudWatch, AWS X-Ray & CloudTrail|Distributed Tracing and Client Side RUM]]

## Tags

#fullstack #interview #monorepo #micro-frontends #module-federation #turborepo #system-architecture

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups