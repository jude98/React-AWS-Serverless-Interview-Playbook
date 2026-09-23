# Building a Multi-Team Enterprise Component Library

> [!abstract] Architectural Overview
> 
> Building a design system and component library across multiple teams is an **infrastructure, governance, contract design, and developer experience challenge** rather than a styling exercise. It requires a **headless/composition-first API**, **token-driven theme architecture**, **strict tree-shaking bundling**, **versioning governance (SemVer + Changesets)**, and **automated regression guardrails (visual, visual regression, accessibility, and bundle budgets)**.
> 
>   

## Key Concepts

- **Headless Core vs. Styled Implementations**: Decouple state, accessibility (ARIA, focus traps, keyboard bindings), and logic from visual styling. Use primitives like Radix UI, React Aria, or Ark UI under the hood.

- **Design Tokens Architecture**: Single source of truth (W3C Design Tokens Community Group format) for colors, typography, spacing, elevations, and animations exported to CSS custom properties, JS/TS, and Tailwind configs.

- **Packaging & Module Distribution**:

    - Distribute dual ESM/CJS outputs with dedicated `package.json` `"exports"` maps.

    - Enforce `sideEffects: false` for strict tree-shaking so consumer apps don't pay bundle penalties for unused components.

- **Styling Layer Strategy**: Prefer Zero-Runtime CSS (Tailwind CSS, Vanilla Extract, CSS Modules, or native CSS Custom Properties) over heavy runtime CSS-in-JS (like styled-components) to avoid hydration and runtime overhead.

- **Governance & Versioning**: Use Monorepo tooling (Turborepo/Nx) with automated changelogs via `@changesets/cli` and strict Semantic Versioning (`major.minor.patch`).

- **Living Documentation & Testing**: Storybook or Ladle with automated visual regression tests (Chromatic/Playwright), automated a11y testing (`axe-core`), and CI bundle-size monitors (bundlesize / size-limit).

## Common Interview Questions

- How do you structure the API of a component library to balance strict brand consistency with team-specific customization?

- Why is runtime CSS-in-JS (e.g., legacy Emotion or styled-components) generally avoided for modern component libraries?

- How do you guarantee tree-shaking so that importing one button doesn't bundle the entire library?

- How do you manage breaking changes across 15+ independent application repositories consuming the shared library?

- How do you integrate design tokens between Figma and production code in an automated pipeline?

- What is the difference between a monolithic package (`@company/ui`) versus multi-package scoped libraries (`@company/button`, `@company/modal`)?

## Strong Answers / Talking Points

### 1. API Design: Headless Primitives & Compound Patterns

- **Avoid Prop Bloat**:

    - _Anti-Pattern_: A single `<Button>` component with 40 props (`isRound`, `hasLeftIcon`, `iconSize`, `isLoading`, `dropdownArrow`, etc.).

    - _Solution_: Headless composition with Compound Components and slots. Consumers compose sub-components (`<Button.Icon>`, `<Button.Spinner>`) rather than toggling mutually exclusive booleans.

- **Polymorphism via `asChild` / Slot Pattern**:

    - Adopt Radix UI’s `asChild` pattern instead of naive `as="a"` props. Merges props, refs, and event handlers cleanly without TypeScript type gymnastics.

### 2. Styling Strategy: Zero-Runtime & Token Ingestion

- **Design Token Pipeline**:

    - Designers update tokens in Figma $\rightarrow$ Tokens synced via GitHub Actions using **Style Dictionary** $\rightarrow$ Compiled into CSS variables (`:root { --color-primary-500: #2563eb; }`) and TypeScript constants.

- **Avoid Runtime CSS-in-JS**:

    - Legacy CSS-in-JS parses styles, computes hashes, and injects `<style>` tags during JS execution on the main thread, causing poor INP and breaking React Server Components (RSC).

    - Use Vanilla Extract, CSS Modules, or Tailwind configured with design tokens for static zero-runtime CSS.

### 3. Monorepo Structure & Package Architecture

- **Single Monorepo (`@company/ui`) with Subpath Exports**:

    - Maintain a monorepo via Turborepo or pnpm workspaces.

    - Publish either a unified package with subpath exports (`@company/ui/button`, `@company/ui/dialog`) or discrete scoped packages (`@company/button`).

    - Subpath exports provide the clean ergonomics of one dependency while preserving isolateable bundle boundaries.

### 4. Enterprise Governance & Adoption Lifecycle

- **Strict Semantic Versioning (SemVer)**:

    - Breaking changes require a major version bump, documented migration guides, and ideally automated **Codemods** (via `jscodeshift`) to automate 80% of consumer upgrades.

- **Automated CI Gates**:

    - `size-limit`: Enforce strict budgets (e.g., `<Button>` cannot exceed 2.5kB gzip).

    - `axe-core`: Enforce WCAG 2.1 AA accessibility standards on all components.

    - `Chromatic / Playwright`: Block PRs that introduce unexpected visual regressions across responsive viewports and dark/light modes.

- **Component Lifecycle Stages**:

    - Label components explicitly in Storybook: `Experimental` $\rightarrow$ `Stable` $\rightarrow$ `Deprecated`.

## Code Snippets / Examples

### Package Configuration for Deep Tree-Shaking (`package.json`)

```json
{
  "name": "@acme/ui",
  "version": "2.4.0",
  "type": "module",
  "sideEffects": false,
  "exports": {
    "./button": {
      "types": "./dist/components/Button/index.d.ts",
      "import": "./dist/components/Button/index.js",
      "require": "./dist/components/Button/index.cjs"
    },
    "./modal": {
      "types": "./dist/components/Modal/index.d.ts",
      "import": "./dist/components/Modal/index.js",
      "require": "./dist/components/Modal/index.cjs"
    },
    "./tokens.css": "./dist/tokens.css"
  },
  "peerDependencies": {
    "react": "^18.0.0 || ^19.0.0",
    "react-dom": "^18.0.0 || ^19.0.0"
  },
  "scripts": {
    "build": "tsup",
    "check:size": "size-limit"
  }
}
```

### Composable Component Pattern with `asChild` (Slot Pattern)

```typescript
import React, { forwardRef } from 'react';
import { Slot } from '@radix-ui/react-slot';
import { cva, type VariantProps } from 'class-variance-authority';

const buttonVariants = cva(
  'inline-flex items-center justify-center font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 disabled:pointer-events-none disabled:opacity-50 rounded-md',
  {
    variants: {
      variant: {
        primary: 'bg-[var(--color-primary)] text-white hover:bg-[var(--color-primary-hover)]',
        secondary: 'bg-[var(--color-surface)] text-[var(--color-text)] border border-[var(--color-border)]',
        ghost: 'hover:bg-[var(--color-surface-hover)] text-[var(--color-text)]',
      },
      size: {
        sm: 'h-8 px-3 text-xs',
        md: 'h-10 px-4 text-sm',
        lg: 'h-12 px-6 text-base',
      },
    },
    defaultVariants: {
      variant: 'primary',
      size: 'md',
    },
  }
);

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean;
}

export const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, ...props }, ref) => {
    // Slot pattern passes all props and merged refs to direct child (e.g. Next.js Link)
    const Comp = asChild ? Slot : 'button';
    return (
      <Comp
        ref={ref}
        className={buttonVariants({ variant, size, className })}
        {...props}
      />
    );
  }
);
Button.displayName = 'Button';
```

### Automated CI Size Guard (`.size-limit.json`)

```json
[
  {
    "name": "Button (Standalone Tree-Shaken)",
    "path": "dist/components/Button/index.js",
    "limit": "3 KB"
  },
  {
    "name": "Full Bundle Export",
    "path": "dist/index.js",
    "limit": "30 KB"
  }
]
```

## Related Topics

- [[Advanced React Performance Optimization Patterns]]

- [[Large-Scale Frontend System Design React at 10M to 1B Users]]

- [[Bundle Size Optimization and Build Analysis Architecture in React (Vite & Rollup)|Webpack and Vite Asset Bundling]]

- [[Component Isolation and Independent State Architecture|Browser Accessibility ARIA and W3C Standards]]

- [[Framework Evaluation and Migration Architectural Decision Framework]]

## Tags

#fullstack #interview #component-library #design-systems #monorepo #tree-shaking #frontend-architecture

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups