# JavaScript Fundamentals & Module Systems

## Key Concepts

> [!summary] JavaScript Core Nature
>
> A single-threaded, synchronous-by-default runtime language with a non-blocking, asynchronous event-driven architecture powered by the Event Loop.


> [!abstract] ECMAScript vs. JavaScript
>
> ECMAScript (ES) is the open standard specification (ECMA-262); JavaScript is the dialect implementation containing standard ECMAScript plus host environment APIs (DOM, Node.js core modules).


> [!info] ESNext
>
> The dynamic label referring to whatever proposed features are currently in pipeline stages (TC39 process) destined for the upcoming ECMAScript release.


> [!example] CommonJS (CJS) vs. ECMAScript Modules (ESM)
>
> - **CommonJS (CJS)**: Legacy Node.js module system (`require` / `module.exports`); loaded synchronously and dynamically evaluated at runtime.
>
> - **ECMAScript Modules (ESM)**: Official language standard module system (`import` / `export`); parsed and resolved statically at compile time, enabling dead-code elimination (tree-shaking).


> [!tip] Static vs. Dynamic Resolution
>
> Static imports resolve module specifiers at parse/compile-time before script execution; dynamic imports (`import()`) evaluate specifiers at runtime and return promises, enabling on-demand code-splitting.


> [!important] SemVer & Dependency Management
>
> - **Semantic Versioning (SemVer)**: `MAJOR.MINOR.PATCH` format indicating breaking changes, backward-compatible features, and backward-compatible bug fixes respectively.
>
> - **`package-lock.json`**: Deterministic snapshot of the exact dependency graph tree, ensuring byte-for-byte identical installations across environments.
>
> - **Peer Dependencies**: Dependencies that a plugin or library expects the host consuming application to install directly, preventing duplicate multi-version instances in memory.


## Common Interview Questions

- "JavaScript is single-threaded, so how does it handle non-blocking asynchronous I/O?"

- "What is the concrete difference between ECMAScript and JavaScript, and what is ESNext?"

- "Compare CommonJS and ESM. Why can't you easily use `require()` inside pure ESM or top-level `await` in CJS?"

- "What is the difference between static resolution and dynamic imports, and how does static analysis enable tree-shaking?"

- "Explain the difference between `^1.2.3`, `~1.2.3`, and `1.2.3` in `package.json`."

- "What problem does `package-lock.json` solve, and why should it always be committed to source control?"

- "What is a `peerDependency` in `package.json`, and how does it differ from a standard `dependency` or `devDependency`?"

## Strong Answers / Talking Points

### 1. JavaScript Engine & Execution Model

- **Single-Threaded Call Stack**: Executes one stack frame at a time; heavy CPU computations will block execution.

- **Concurrency via Environment**: Web APIs (browser) or libuv (Node.js) offload asynchronous tasks (timers, file I/O, network requests), pushing callbacks into Task and Microtask Queues for the Event Loop to process.

### 2. CommonJS (CJS) vs. ECMAScript Modules (ESM)

- **Evaluation Mechanism**:

    - **CJS**: Loaded dynamically on demand when execution reaches `require()`. It evaluates synchronously and returns an object copy/reference of `module.exports`.

    - **ESM**: Multi-phase lifecycle (Construction -> Instantiation -> Evaluation). Imports establish live, read-only bindings to exported memory locations.

- **Tree-Shaking**:

    - CJS cannot be reliably tree-shaken because `require()` paths can be dynamically constructed at runtime.

    - ESM enforces static paths at top level, allowing bundlers (Vite, Rollup, Webpack) to parse the AST and eliminate dead code.

- **Top-Level Await**: ESM supports top-level `await` natively; CJS does not because `require()` is fundamentally synchronous.

### 3. SemVer & Version Specifiers in `package.json`

- **Format**: `MAJOR.MINOR.PATCH`

    - `MAJOR`: Breaking breaking changes.

    - `MINOR`: Backward-compatible new features.

    - `PATCH`: Backward-compatible bug fixes.

- **Specifiers**:

    - `~1.2.3` (Tilde): Allows PATCH upgrades only (`>=1.2.3 <1.3.0`).

    - `^1.2.3` (Caret): Allows MINOR and PATCH upgrades (`>=1.2.3 <2.0.0`). _Note: for `0.x.x`, caret locks to patch/minor accordingly due to zero-version instability._

    - `1.2.3` (Exact): Disables automatic updates.

    - `*` or `latest`: Accepts any version (dangerous in production).

### 4. Dependency Categories & `package-lock.json`

- **`dependencies` vs `devDependencies` vs `peerDependencies`**:

    - `dependencies`: Needed in production runtime (e.g., Express, React).

    - `devDependencies`: Needed only during development/build phase (e.g., TypeScript, ESLint, Vitest).

    - `peerDependencies`: Expresses compatibility without nesting. Example: Plugin libraries (`react-router`) require the consumer to supply a single matching instance of `react` to avoid multiple React copies in memory breaking hooks.

- **Why `package-lock.json` is Critical**:

    - Prevents the "works on my machine" issue where caret ranges resolve different minor versions across machines.

    - Stores integrity hashes (SHA-512) to protect against tampered packages.

    - Always commit this file; use `npm ci` in CI/CD pipelines instead of `npm install` for deterministic installs.

## Code Snippets / Examples

### Static vs. Dynamic Resolution (ESM & CJS)

```javascript
// Static Resolution (ESM) - parsed before execution; cannot be conditional
import { render } from './renderer.js';

// Dynamic Resolution (ESM) - runtime evaluation, returns a Promise
if (condition) {
  const { heavyFeature } = await import('./heavyFeature.js');
  heavyFeature();
}

// CommonJS Dynamic Behavior - evaluated imperatively at runtime
const moduleName = condition ? './moduleA.js' : './moduleB.js';
const loadedModule = require(moduleName);
```

### package.json Dependency Configuration

```json
{
  "name": "ui-component-library",
  "version": "1.0.0",
  "type": "module",
  "dependencies": {
    "lodash-es": "^4.17.21"
  },
  "devDependencies": {
    "typescript": "~5.3.3"
  },
  "peerDependencies": {
    "react": ">=18.0.0 <19.0.0"
  }
}
```

## Related Topics

- [[Asynchronous JavaScript, Event Loop & Concurrency Model|JavaScript Event Loop and Concurrency Model]]

- [[Bundle Size Optimization and Build Analysis Architecture in React (Vite & Rollup)|Bundlers and Build Tools: Webpack, Vite, and Rollup]]

- [[Asynchronous JavaScript, Event Loop & Concurrency Model|Node.js Runtime Architecture and Libuv]]

- [[Monorepo and Micro-Frontend Architecture Evaluation|Monorepos and Package Management: NPM, PNPM, and Yarn]]

## Tags

#fullstack #interview #javascript #nodejs #npm

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups
