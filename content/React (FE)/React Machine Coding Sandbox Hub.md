# React Machine Coding Sandbox Hub

> [!abstract] Overview
>
> A centralized hub linking directly to live implementations of essential frontend machine coding challenges, custom hooks, and performance patterns. This sandbox serves as a playground and reference implementation for common live-coding interview rounds.


## Key Concepts

- **Component Architecture**: Building composable, accessible (WAI-ARIA compliant), and headless UI primitives without external component libraries.

- **Performance Primitives**: Utilizing `IntersectionObserver` for lazy loading and custom virtualization for unblocking the DOM render queue.

- **State Partitioning**: Implementing compound components and Context API patterns with split state/dispatch boundaries to eliminate cascading re-renders.

- **Timer and Event Precision**: Managing `requestAnimationFrame`, `setInterval` cleanups, debounce/throttle queues, and focus management across complex inputs.

## Common Interview Questions

- How do you implement an accessible, keyboard-navigable OTP input that handles copy-paste across inputs?

- How does your custom Virtualization hook calculate slice offsets without layout thrashing?

- How do you design a Toast notification system using Context and Portals without coupling to parent component render cycles?

- What are the trade-offs of implementing a Stopwatch using `setInterval` vs. delta timestamps with `requestAnimationFrame`?

- How do you structure a compound Tabs and Combo Tab component to allow headless consumption?

- How do you avoid unnecessary re-renders in a multi-step form built entirely with the React Context API?

## Strong Answers / Talking Points

### 1. The Machine Coding Evaluation Criteria

- **Separation of Concerns**: Keep business/state logic inside custom hooks (`useVirtualizer`, `useStopwatch`, `useToast`) and UI presentation strictly in stateless/compound views.

- **Edge Cases & Accessibility**:

    - _OTP Input_: Support arrow navigation, backspace boundary jumping, non-numeric character rejection, and multi-digit clipboard paste.

    - _Toast_: Stacking context via `createPortal`, auto-dismiss cleanup timers, and accessibility announcements via `role="alert"`.

    - _Infinite Scroll / IntersectionObserver_: Disconnecting observers on unmount, handling zero-height sentinel edges, and aborting concurrent fetches.

- **State Management Trade-offs (Form with Context)**:

    - Mitigate Context performance traps by utilizing component composition (`children`), memoizing context values, and decoupling dispatch callbacks from form data stores.

### 2. Sandbox Feature Catalog

- **Performance & Data**:

    - `Virtualization`: Windowed viewport rendering of continuous list nodes.

    - `Intersection Observer`: Infinite scroll sentinel detection and lazy image loading.

    - `Search Bar`: Debounced search inputs with keyboard-navigable autocomplete dropdown.

- **Input & Feedback Controls**:

    - `OTP Input`: Multi-cell auto-advancing focus input with paste parsing.

    - `Sliding Bar / Range Slider`: Controlled coordinate dragging with touch and mouse event listeners.

    - `Progress Bar`: Animated width transitions and dynamic ARIA value binding (`aria-valuenow`).

    - `Toast System`: Queue-managed portal notifications with auto-dismiss timers.

- **State & Navigation Primitives**:

    - `Counter`: Reducer-driven state updates with boundary validation.

    - `Stopwatch`: High-precision timer using timestamp deltas (`performance.now()`).

    - `Tabs & Combo Tab`: Compound components managing active tab indices and synced panel views.

    - `Context Form`: Multi-field form built with compound composition and segregated dispatch contexts.

## Code Snippets / Examples

### Sandbox Embed & Quick Launch Index

```markdown
<!-- Sandbox Direct Link -->
[🚀 Open Interactive Machine Coding Sandbox](https://codesandbox.io/dashboard/sandboxes/React?workspace=ws_BVcztAHnDihGMUDM732RKY)
```

```typescript
import React from 'react';

export const SandboxFeatureDirectory = () => {
  const components = [
    { title: 'Virtualization', path: '/virtual-list', tag: 'Performance' },
    { title: 'Intersection Observer', path: '/infinite-scroll', tag: 'Performance' },
    { title: 'OTP Input', path: '/otp', tag: 'Forms' },
    { title: 'Context Form Composition', path: '/context-form', tag: 'State' },
    { title: 'Toast Notification Portal', path: '/toast', tag: 'Overlay' },
    { title: 'Stopwatch (High Precision)', path: '/stopwatch', tag: 'Hooks' },
    { title: 'Combo Tabs', path: '/combo-tabs', tag: 'Compound UI' },
    { title: 'Sliding Bar', path: '/slider', tag: 'UI Controls' },
    { title: 'Progress Bar', path: '/progress', tag: 'UI Controls' },
    { title: 'Debounced Search Bar', path: '/search', tag: 'Hooks' },
  ];

  return (
    <div className="p-6 max-w-4xl mx-auto">
      <h1 className="text-2xl font-bold mb-4">Machine Coding Implementation Index</h1>
      <div className="grid grid-cols-2 gap-4">
        {components.map((item) => (
          <div key={item.path} className="p-4 border rounded-lg hover:shadow-md transition">
            <span className="text-xs uppercase font-semibold text-blue-600 tracking-wider">
              {item.tag}
            </span>
            <h3 className="text-lg font-medium mt-1">{item.title}</h3>
            <code className="text-xs text-gray-500">{item.path}</code>
          </div>
        ))}
      </div>
    </div>
  );
};
```

## Related Topics

- [[High-Scale Data Table Architecture Handling Millions of Records]]

- [[Advanced React Performance Optimization Patterns]]

- [[Comprehensive Performance Optimization Architecture in React|React Performance Optimization]]

- [[Browser Architecture. High-Level Components, Rendering Engines & HTML Parsing|Browser Rendering Engine and Critical Rendering Path]]

- [[Combining React Context and useReducer Architecture|React Context API and Performance Anti Patterns]]

## Tags

#fullstack #interview #machine-coding #react-components #code-sandbox #custom-hooks #ui-patterns

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups
