# React Strict Mode and Double-Invocation Behavior

> [!note] Strict Mode Core Purpose
>
> `<React.StrictMode>` is a developer-only tool that helps catch side-effect bugs, impure functions, missing effect cleanups, and deprecated patterns by **intentionally double-invoking** specific functions and simulating mount-unmount-remount cycles in development. It does nothing in production.


> [!abstract] Why React Executes Things Twice
>
> React's concurrent architecture requires components to be resilient to being mounted, destroyed, and re-mounted multiple times. Strict Mode forces this behavior early to expose latent bugs—such as uncleaned event listeners, memory leaks, or race conditions—before they hit production.


## Key Concepts

- **Environment Scope**: Strict Mode only alters execution behavior in **development mode**. Production builds run normally.

- **Component Function Double-Invocation (Initial Mount)**: React intentionally calls your component function **twice** on initial load to check if it is a pure function.

- **Simulated Unmount/Remount Cycle (Effects)**: Introduced heavily in React 18+, effects are mounted, immediately unmounted (cleanup runs), and remounted on the initial load to verify cleanup correctness.

- **Purity Enforcement**: Rendering should be pure (`UI = f(state)`). If a component mutates global state, arrays, or objects directly inside the render body, the double-execution exposes the inconsistency immediately.

## What Executes Twice on Initial Mount (Development Only)?

1. **The React Component Function Body**:

    - The entire function block runs twice back-to-back to ensure that rendering produces deterministic output without side effects.

2. **State Initializers and Hooks**:

    - `useState(() => initialValue)` (lazy initializers) run twice.

    - `useMemo` and `useReducer` initializer/calculation callbacks are invoked twice.

3. **`useEffect` Mounting Simulation Sequence**:

    - **Step 1**: Component mounts $\rightarrow$ Effect runs for the first time.

    - **Step 2**: React _simulates_ an immediate unmount $\rightarrow$ Effect cleanup function runs.

    - **Step 3**: React _simulates_ a remount $\rightarrow$ Effect runs for the second time.

    - _Net result in console_: `Effect ran` $\rightarrow$ `Cleanup ran` $\rightarrow$ `Effect ran` (totaling 2 executions of the effect, 1 execution of the cleanup).

## What Happens During Re-Renders?

- **Re-render Component Function**: When a state or prop update triggers a re-render, the component function executes **twice** in development under Strict Mode for the same purity-checking reasons.

- **Effect Behavior on Re-render**:

    - When dependencies change, the previous effect's cleanup function runs, followed by the new effect running **once** per standard lifecycle update (unless Strict Mode triggers its dev-only mount check again during structural layout changes).

## Common Interview Questions

- What is the purpose of `React.StrictMode`, and does it affect production performance?

- Why does React intentionally invoke component functions and effects twice during mounting in development?

- What is the exact execution sequence of `useEffect` and its cleanup under Strict Mode during an initial mount?

- What makes a component "impure" during the render phase, and how does Strict Mode expose it?

- How do you fix data fetching or WebSocket subscription bugs caused by Strict Mode double-invocations?

## Strong Answers / Talking Points

- **The Impure Render Problem**:

    - If a component function modifies a variable declared outside its scope (e.g., `let globalCounter = 0; function Comp() { globalCounter++; ... }`), running it twice increments the counter unexpectedly. Strict Mode exposes this non-deterministic behavior.

- **The Missing Cleanup Problem (`useEffect`)**:

    - Developers often forget to return a cleanup function for event listeners or timers. When Strict Mode unmounts and remounts the component, a duplicate listener or timer is registered. If the code breaks or duplicates data, it exposes a missing cleanup bug.

- **Handling Data Fetching in Strict Mode**:

    - _Pitfall_: Seeing two network requests in the network tab on page load.

    - _Solution_: Use an `AbortController` inside the effect cleanup, or adopt modern asynchronous caching layers like React Query / RTK Query, which gracefully handle duplicate or cancelled mount requests.

## Code Snippets / Examples

```javascript
import { useState, useEffect } from 'react';

export function StrictModeDemo() {
  const [count, setCount] = useState(() => {
    console.log('1. useState lazy initializer runs (Executes twice in Strict Mode)');
    return 0;
  });

  // Component Function Body (Executes twice on mount/re-render in dev)
  console.log('2. Component function body rendered');

  useEffect(() => {
    console.log('3. Effect mounted / ran');

    const subscription = window.addEventListener('resize', () => {});

    // Cleanup function: Essential for Strict Mode's simulated unmount
    return () => {
      console.log('4. Effect cleanup executed (Simulated unmount)');
      window.removeEventListener('resize', () => {});
    };
  }, []);

  return (
    <button onClick={() => setCount(c => c + 1)}>
      Count: {count}
    </button>
  );
}
```

_Console Output on Initial Mount under Strict Mode:_

```text
> 1. useState lazy initializer runs
> 2. useState lazy initializer runs
> 3. Component function body rendered
> 4. Component function body rendered
> 5. Effect mounted / ran
> 6. Effect cleanup executed (Simulated unmount)
> 7. Effect mounted / ran
```

## Related Topics

- [[React Lifecycle and Execution Flow|React Component Lifecycle]]

- [[React useEffect and Synchronization Architecture|React useEffect and Side Effects]]

- [[React Reconciliation and Diffing Algorithm|Virtual DOM and Reconciliation]]

- [[React Fiber Architecture and Non-Blocking Rendering|React Fiber Architecture]]

## Tags

#fullstack #interview #react-strictmode #react-fundamentals

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups
