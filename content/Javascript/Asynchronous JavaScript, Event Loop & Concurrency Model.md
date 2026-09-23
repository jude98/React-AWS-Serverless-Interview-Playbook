# Asynchronous JavaScript, Event Loop & Concurrency Model

## Key Concepts

> [!summary] Single-Threaded Non-Blocking Architecture
>
> JavaScript's engine (V8, JavaScriptCore) has a single Call Stack and executes one instruction at a time. Concurrency is achieved not through multithreaded JS code execution, but via host environment APIs (Browser Web APIs or Node.js `libuv`), which offload asynchronous tasks (timers, network requests, disk I/O) and push callbacks to queues monitored by the **Event Loop**.


> [!abstract] Task Queues & Priority Hierarchies
>
> When the Call Stack clears, the Event Loop processes queues in strict priority order:
>
> 1. **Node.js-Only Priority Phase**: `process.nextTick` queue runs before any other microtask.
>
> 2. **Microtask Queue**: Promises (`.then`, `.catch`, `.finally`), `async/await` resumption steps, and `queueMicrotask` (or `MutationObserver` in browsers). The engine **completely empties** the microtask queue (including microtasks queued by running microtasks) before yielding.
>
> 3. **Macrotask / Task Queue**: `setTimeout`, `setInterval`, `setImmediate` (Node.js), DOM event listeners, and I/O callbacks.


> [!info] The Evolution of Async Flow
>
> - **Callbacks**: Passing a function as an argument to execute upon operation completion; led to inversion of control, poor error handling, and deeply nested pyramids of code (**Callback Hell**).
>
> - **Promises**: Concrete object state representation (`pending`, `fulfilled`, `rejected`) providing immutable resolution, chainability (`.then()`), and centralized error propagation (`.catch()`).
>
> - **Async/Await**: Syntactic sugar over Promises and Generators; pauses function execution linearly without blocking the Call Stack thread.


> [!tip] Node.js Timers: `setImmediate` vs. `process.nextTick` vs. `setTimeout`
>
> - `process.nextTick()`: Not part of the official libuv loop; runs immediately after current tick completes, starving I/O if called recursively.
>
> - `setImmediate()`: Executes in the **Check Phase** of the libuv event loop (designed to run after I/O callbacks).
>
> - `setTimeout(fn, 0)`: Executes in the **Timers Phase**; in Node.js, `0ms` is converted to minimum `1ms`.


## Common Interview Questions

- "Walk me through how the Event Loop coordinates the Call Stack, Microtask Queue, and Macrotask Queue."

- "Why does a resolved Promise callback execute before a `setTimeout(..., 0)` callback?"

- "What happens if a microtask recursively schedules another microtask? Does the browser render or run timers?"

- "What is the concrete execution order difference between `process.nextTick()` and `setImmediate()` in Node.js?"

- "How does `async/await` work under the hood using Promises and Generators?"

- "What is the difference between `Promise.all`, `Promise.allSettled`, `Promise.race`, and `Promise.any`?"

- "Predict the output of mixed synchronous logs, `setTimeout`, `Promise`, `process.nextTick`, and `async/await` code."

## Strong Answers / Talking Points

### 1. The Event Loop Algorithm Step-by-Step

1. Execute synchronous code from the **Call Stack** until empty.

2. Flush the **Microtask Queue** until completely empty.

3. In browsers: check if a render cycle is needed (run `requestAnimationFrame` and recalculate layout/paint).

4. Pull and execute **one single task** from the **Macrotask Queue**.

5. Immediately check and exhaust the **Microtask Queue** again.

6. Repeat the cycle indefinitely.

### 2. Starvation via Microtasks

- Because the engine drains the _entire_ microtask queue before moving to rendering or the macrotask queue, an infinite loop of microtasks (`function loop() { Promise.resolve().then(loop); }`) completely **starves the macrotask queue and UI rendering**, freezing the application.

### 3. Promise Combinators Comparison

- **`Promise.all(iterable)`**: Rejects immediately (**fail-fast**) if any promise rejects; resolves with an array of values when all resolve.

- **`Promise.allSettled(iterable)`**: Never rejects early; waits for every promise to either fulfill or reject, returning an array of `{ status, value | reason }` objects.

- **`Promise.race(iterable)`**: Settles as soon as the **first** promise settles (fulfills or rejects).

- **`Promise.any(iterable)`**: Ignores rejections and resolves with the **first successful fulfillment**; only rejects (with an `AggregateError`) if _all_ promises reject.

### 4. Node.js Libuv Loop Phases

1. **Timers**: Executes callbacks scheduled by `setTimeout` and `setInterval`.

2. **Pending Callbacks**: Executes I/O callbacks deferred to the next loop iteration.

3. **Idle, Prepare**: Used internally by libuv.

4. **Poll**: Retrieves new I/O events; executes I/O related callbacks.

5. **Check**: Executes `setImmediate()` callbacks.

6. **Close Callbacks**: Handles close events (e.g., `socket.on('close')`).

## Code Snippets / Examples

### 1. The Classic Output Prediction Puzzle (Browser & Node.js Common Core)

```javascript
console.log("1 - Sync Start");

setTimeout(() => {
  console.log("2 - setTimeout (Macrotask)");
}, 0);

Promise.resolve()
  .then(() => {
    console.log("3 - Promise 1 (Microtask)");
  })
  .then(() => {
    console.log("4 - Promise 2 (Microtask)");
  });

async function asyncDemo() {
  console.log("5 - Async Body (Sync)");
  await null; // Code after this point is queued as a microtask
  console.log("6 - Async After Await (Microtask)");
}
asyncDemo();

console.log("7 - Sync End");

// Output Order:
// 1 - Sync Start
// 5 - Async Body (Sync)
// 7 - Sync End
// 3 - Promise 1 (Microtask)
// 6 - Async After Await (Microtask)
// 4 - Promise 2 (Microtask)
// 2 - setTimeout (Macrotask)
```

### 2. Node.js Event Loop Hierarchy (`nextTick` vs `setImmediate`)

```javascript
// Run in Node.js environment
setTimeout(() => console.log("1 - setTimeout 0ms"), 0);
setImmediate(() => console.log("2 - setImmediate"));

process.nextTick(() => console.log("3 - process.nextTick"));
Promise.resolve().then(() => console.log("4 - Promise microtask"));

// Output within current tick:
// 3 - process.nextTick (Runs before standard microtasks)
// 4 - Promise microtask (Standard microtask)
//
// Following output depends on whether script is inside an I/O cycle:
// - If top-level: order of setTimeout(0) vs setImmediate can vary due to 1ms timer resolution.
// - Inside an fs.readFile (I/O cycle): setImmediate ALWAYS prints before setTimeout(0)!
```

### 3. Solving Callback Hell: Callbacks vs Promises vs Async/Await

```javascript
// 1. Callback Hell (Pyramid of Doom + Inversion of Control)
getUser(userId, (userErr, user) => {
  if (userErr) return handleError(userErr);
  getOrders(user.id, (orderErr, orders) => {
    if (orderErr) return handleError(orderErr);
    getOrderDetails(orders[0].id, (detailErr, details) => {
      if (detailErr) return handleError(detailErr);
      console.log(details);
    });
  });
});

// 2. Linear Promise Chain
getUser(userId)
  .then(user => getOrders(user.id))
  .then(orders => getOrderDetails(orders[0].id))
  .then(details => console.log(details))
  .catch(handleError);

// 3. Clean Async/Await with Localized try/catch
async function showOrderDetails(userId) {
  try {
    const user = await getUser(userId);
    const orders = await getOrders(user.id);
    const details = await getOrderDetails(orders[0].id);
    console.log(details);
  } catch (error) {
    handleError(error);
  }
}
```

### 4. Custom Promise Implementation Core (Mental Model)

```javascript
class SimplePromise {
  constructor(executor) {
    this.state = "pending";
    this.value = undefined;
    this.handlers = [];

    const resolve = (val) => {
      if (this.state !== "pending") return;
      this.state = "fulfilled";
      this.value = val;
      // Ensure callbacks execute asynchronously via microtask
      queueMicrotask(() => {
        this.handlers.forEach(h => h(this.value));
      });
    };

    try {
      executor(resolve);
    } catch (err) {
      // Rejection logic omitted for brevity
    }
  }

  then(onFulfilled) {
    return new SimplePromise((resolve) => {
      if (this.state === "fulfilled") {
        queueMicrotask(() => resolve(onFulfilled(this.value)));
      } else {
        this.handlers.push((val) => resolve(onFulfilled(val)));
      }
    });
  }
}
```

## Comparison Matrix: Asynchronous Queues

|**Queue / Mechanism**|**Environment**|**Processing Trigger**|**Priority Level**|
|---|---|---|---|
|**Call Stack**|All|Continuous synchronous execution|**Active execution thread**|
|**`process.nextTick`**|Node.js only|Immediately after current operation completes|Highest Priority Task|
|**Microtask Queue** (`Promise`, `await`, `queueMicrotask`)|All|Emptied completely whenever stack clears|High Priority|
|**Macrotask Queue** (`setTimeout`, `setInterval`)|All|Event Loop picks one task per loop cycle|Normal Priority|
|**`setImmediate`**|Node.js only|Check phase (after I/O poll phase)|Normal Priority (Check phase)|
|**`requestAnimationFrame`**|Browser only|Before visual repaint|Browser frame rendering|

## Related Topics

- [[JavaScript Execution Context, Memory Creation & Hoisting Mechanics]]

- [[JavaScript Exception Handling. Try-Catch-Finally, Error Objects & Global Error Boundaries|JavaScript Exception Handling: Try-Catch-Finally, Error Objects & Global Error Boundaries]]

- [[Asynchronous JavaScript, Event Loop & Concurrency Model|Node.js Runtime Architecture and Libuv]]

- [[JavaScript Loops, Iteration Protocols & Data Structure Traversal|Iterables, Iterators, and Generators]]

## Tags

#fullstack #interview #javascript #event-loop #promises #async-await #settimeout #microtasks #nodejs

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups
