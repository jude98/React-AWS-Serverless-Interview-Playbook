# JavaScript Polyfills & Core Runtime Implementations

## Key Concepts

> [!summary] What is a Polyfill?
>
> A **polyfill** is runtime code (typically in pure JavaScript) that replicates the behavior, signature, and specification semantics of a built-in ECMAScript API or Web API on engines where that feature is missing, incomplete, or divergent. In frontend and full-stack interviews, writing polyfills evaluates whether an engineer understands prototype chains, internal ECMAScript algorithms, closures, execution contexts, and asynchronous scheduling (microtasks vs. macrotasks).


> [!abstract] ECMAScript Compliance vs. Toy Implementations
>
> A toy polyfill works on basic arrays or primitives but breaks on edge cases. A **production-grade interview polyfill** accounts for:
>
> 1. **Defensive Validation**: Strict checks for `null`, `undefined`, and non-callable parameters throwing proper `TypeError` instances.
>
> 2. **Sparse Arrays (Holes)**: Using `i in this` (or `Object.hasOwn`) rather than checking `this[i] !== undefined` to skip unallocated slots in arrays like `new Array(3)`.
>
> 3. **Context Binding (`thisArg`)**: Allowing custom `this` contexts to be passed to callbacks via `.call(thisArg, ...)`.
>
> 4. **Asynchronous Microtask Semantics**: Using `queueMicrotask()` (or MutationObserver) rather than macrotasks (`setTimeout`) when polyfilling Promise resolutions.
>
> 5. **Constructor Calling Semantics**: Detecting when a bound function is invoked via `new` (`this instanceof bound`) to avoid overriding newly allocated instance prototypes.


> [!tip] Prototype Mutability Strategy
>
> When augmenting global prototypes (`Array.prototype`, `Function.prototype`, `Promise`), always verify whether the native method already exists:
>
> ```javascript
> if (!Array.prototype.myMap) {
>   Object.defineProperty(Array.prototype, "myMap", {
>     value: function(...) { /* ... */ },
>     writable: true,
>     configurable: true,
>     enumerable: false, // CRITICAL: Never create enumerable prototype properties
>   });
> }
> ```
>
> Using `Object.defineProperty` with `enumerable: false` prevents your polyfill from corrupting `for...in` loops in legacy or third-party code.


## Common Interview Questions

- "Write a polyfill for `Array.prototype.map`, `filter`, and `reduce`. How do your implementations handle sparse arrays (holes)?"

- "How would you implement `Array.prototype.flat` supporting arbitrary recursion depth and `Infinity`?"

- "Implement `Function.prototype.call`, `apply`, and `bind` without using the built-in counterparts. How do you handle `new boundFunction()`?"

- "Write an advanced currying utility that supports fixed arity, infinite currying (`sum(1)(2)(3)()`), and placeholders (`curry.placeholder`)."

- "How do you implement a production-ready `debounce` function with `.cancel()` and immediate leading-edge execution, and how does it differ from `throttle`?"

- "Write a deep clone utility (`deepCopy`) that handles circular references, nested Arrays, Dates, RegExps, and preserves object prototypes."

- "Build a Promises/A+ compliant Promise polyfill (`MyPromise`) from scratch, including `.then()`, `.catch()`, `.finally()`, cycle detection, and static methods like `Promise.all` and `Promise.allSettled`."

- "How do you build a cancelable HTTP request wrapper with `AbortController` and timeout racing?"

---

## Strong Answers / Talking Points

### 1. Sparse Arrays: Why `i in this` Matters

In JavaScript, `[1, , 3]` is a sparse array. Index `1` does not exist in the array's property dictionary:
- `1 in [1, , 3]` evaluates to `false`.
- `[1, , 3][1]` evaluates to `undefined`, but index `1` is a **hole**, not an element whose value is `undefined`.

The official ECMAScript specification for `Array.prototype.map` states that callbacks must only be invoked for elements that actually exist on the object (`HasProperty(O, Pk)`). Checking `if (i in this)` ensures that holes are preserved in `map`, omitted in `filter`, and properly skipped when searching for an initial accumulator in `reduce`.

### 2. Microtasks vs. Macrotasks in Promise Polyfills

The Promises/A+ specification clause 2.2.4 mandates that `onFulfilled` and `onRejected` callbacks must not be called in the current execution context (they must be asynchronous). Using `setTimeout(fn, 0)` schedules the callback in the **Macrotask Queue**, which executes after DOM rendering, UI updates, and I/O callbacks. A compliant polyfill uses `queueMicrotask(fn)` (or `Promise.resolve().then(fn)`), ensuring callbacks execute at the end of the current microtask tick before yielding control back to the browser event loop.

### 3. Circular References in Deep Cloning

When cloning complex object graphs (e.g., DOM trees, linked lists, cyclic graphs where `obj.self = obj`), naive recursive traversals result in `RangeError: Maximum call stack size exceeded`. Employing a `WeakMap<object, object>` tracks every visited object and its newly instantiated clone. If a key is encountered again in the object traversal, the clone immediately returns the cached reference, completely resolving circularities without memory leaks.

---

## Complete Implementations & Code Blocks

### 1. Array Operations: `map`, `filter`, `reduce` & `flat`

```javascript
// ============================================================
// 1. Array.prototype.myMap
// ============================================================
Array.prototype.myMap = function(callback, thisArg) {
  if (this == null) {
    throw new TypeError("Array.prototype.myMap called on null or undefined");
  }
  if (typeof callback !== "function") {
    throw new TypeError(callback + " is not a function");
  }

  const O = Object(this);
  const len = O.length >>> 0; // Unsigned 32-bit integer conversion
  const result = new Array(len);

  for (let i = 0; i < len; i++) {
    // Only invoke callback for allocated slots (skip holes in sparse arrays)
    if (i in O) {
      result[i] = callback.call(thisArg, O[i], i, O);
    }
  }

  return result;
};

// ============================================================
// 2. Array.prototype.myFilter
// ============================================================
Array.prototype.myFilter = function(callback, thisArg) {
  if (this == null) {
    throw new TypeError("Array.prototype.myFilter called on null or undefined");
  }
  if (typeof callback !== "function") {
    throw new TypeError(callback + " is not a function");
  }

  const O = Object(this);
  const len = O.length >>> 0;
  const result = [];

  for (let i = 0; i < len; i++) {
    if (i in O) {
      const val = O[i];
      if (callback.call(thisArg, val, i, O)) {
        result.push(val);
      }
    }
  }

  return result;
};

// ============================================================
// 3. Array.prototype.myReduce
// ============================================================
Array.prototype.myReduce = function(callback, initialValue) {
  if (this == null) {
    throw new TypeError("Array.prototype.myReduce called on null or undefined");
  }
  if (typeof callback !== "function") {
    throw new TypeError(callback + " is not a function");
  }

  const O = Object(this);
  const len = O.length >>> 0;
  let accumulator;
  let startIndex = 0;

  if (arguments.length >= 2) {
    accumulator = initialValue;
  } else {
    // Locate the first defined, non-empty index
    let k = 0;
    while (k < len && !(k in O)) {
      k++;
    }

    if (k >= len) {
      throw new TypeError("Reduce of empty array with no initial value");
    }

    accumulator = O[k];
    startIndex = k + 1;
  }

  for (let i = startIndex; i < len; i++) {
    if (i in O) {
      accumulator = callback(accumulator, O[i], i, O);
    }
  }

  return accumulator;
};

// ============================================================
// 4. Array.prototype.myFlat
// ============================================================
Array.prototype.myFlat = function(depth = 1) {
  if (this == null) {
    throw new TypeError("Array.prototype.myFlat called on null or undefined");
  }

  const targetDepth = Number(depth);
  const O = Object(this);

  function flatten(array, currentDepth) {
    const output = [];

    for (let i = 0; i < array.length; i++) {
      if (i in array) {
        const item = array[i];
        if (Array.isArray(item) && currentDepth > 0) {
          output.push(...flatten(item, currentDepth - 1));
        } else {
          output.push(item);
        }
      }
    }

    return output;
  }

  return flatten(O, targetDepth);
};
```

---

### 2. Function Invocations & Context: `call`, `apply`, `bind`

```javascript
// ============================================================
// 1. Function.prototype.myCall
// ============================================================
Function.prototype.myCall = function(context, ...args) {
  if (typeof this !== "function") {
    throw new TypeError("Function.prototype.myCall - what is trying to be invoked is not callable");
  }

  // Fallback to global object if null or undefined passed (non-strict mode spec)
  const targetContext = context != null ? Object(context) : globalThis;
  const fnSymbol = Symbol("tempFn");

  targetContext[fnSymbol] = this;
  const result = targetContext[fnSymbol](...args);
  delete targetContext[fnSymbol];

  return result;
};

// ============================================================
// 2. Function.prototype.myApply
// ============================================================
Function.prototype.myApply = function(context, argsArray = []) {
  if (typeof this !== "function") {
    throw new TypeError("Function.prototype.myApply - what is trying to be invoked is not callable");
  }

  const targetContext = context != null ? Object(context) : globalThis;
  const fnSymbol = Symbol("tempFn");

  targetContext[fnSymbol] = this;
  const result = targetContext[fnSymbol](...argsArray);
  delete targetContext[fnSymbol];

  return result;
};

// ============================================================
// 3. Function.prototype.myBind (Supporting Constructor Calls)
// ============================================================
Function.prototype.myBind = function(context, ...boundArgs) {
  const originalFn = this;

  if (typeof originalFn !== "function") {
    throw new TypeError("Function.prototype.myBind - what is trying to be bound is not callable");
  }

  const boundFunction = function(...invokedArgs) {
    // If invoked with `new boundFunction()`, `this` is an instance of boundFunction
    // In that case, `this` must NOT be overridden with `context`
    const isNewCall = this instanceof boundFunction;
    const effectiveContext = isNewCall ? this : context;

    return originalFn.apply(effectiveContext, [...boundArgs, ...invokedArgs]);
  };

  // Maintain prototype chain for constructor calls
  if (originalFn.prototype) {
    boundFunction.prototype = Object.create(originalFn.prototype);
  }

  return boundFunction;
};
```

---

### 3. Advanced Currying Patterns

```javascript
// ============================================================
// 1. Fixed Arity Currying (Recursive Arity Check)
// ============================================================
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    }
    return function(...nextArgs) {
      return curried.apply(this, [...args, ...nextArgs]);
    };
  };
}

// Example usage:
const sum3 = (a, b, c) => a + b + c;
const curriedSum3 = curry(sum3);
console.log(curriedSum3(1)(2)(3)); // 6
console.log(curriedSum3(1, 2)(3)); // 6

// ============================================================
// 2. Infinite Currying (Terminated via Empty Invocation)
// ============================================================
function infiniteCurry(fn) {
  return function curried(...args) {
    return function(...nextArgs) {
      if (nextArgs.length === 0) {
        return args.reduce((acc, curr) => fn(acc, curr));
      }
      return curried(...args, ...nextArgs);
    };
  };
}

const add = infiniteCurry((a, b) => a + b);
console.log(add(1)(2)(3)(4)()); // 10

// ============================================================
// 3. Currying with Placeholder Support (Lodash-Style)
// ============================================================
function curryWithPlaceholder(fn) {
  function curried(...args) {
    // Check if enough arguments without placeholders are provided
    const complete = args.length >= fn.length &&
      !args.slice(0, fn.length).includes(curryWithPlaceholder.placeholder);

    if (complete) {
      return fn.apply(this, args);
    }

    return function(...nextArgs) {
      // Merge nextArgs into placeholders
      const mergedArgs = args.map((arg) =>
        arg === curryWithPlaceholder.placeholder && nextArgs.length > 0
          ? nextArgs.shift()
          : arg
      ).concat(nextArgs);

      return curried.apply(this, mergedArgs);
    };
  }

  return curried;
}

curryWithPlaceholder.placeholder = Symbol("curry_placeholder");

// Example usage:
const _ = curryWithPlaceholder.placeholder;
const greet = (greeting, title, name) => `${greeting}, ${title} ${name}!`;
const curriedGreet = curryWithPlaceholder(greet);

console.log(curriedGreet(_, "Dr.", _)("Hello")("Alice")); // "Hello, Dr. Alice!"
```

---

### 4. Async & Rate-Limiting: `debounce` & `throttle`

```javascript
// ============================================================
// 1. Debounce with .cancel() & Immediate Leading Edge Option
// ============================================================
function debounce(fn, delay, options = { leading: false }) {
  let timerId = null;
  let lastArgs = null;
  let lastThis = null;

  function debounced(...args) {
    lastArgs = args;
    lastThis = this;

    const callNow = options.leading && !timerId;

    if (timerId) {
      clearTimeout(timerId);
    }

    timerId = setTimeout(() => {
      timerId = null;
      if (!options.leading) {
        fn.apply(lastThis, lastArgs);
      }
    }, delay);

    if (callNow) {
      fn.apply(lastThis, lastArgs);
    }
  }

  debounced.cancel = function() {
    if (timerId) {
      clearTimeout(timerId);
      timerId = null;
    }
  };

  return debounced;
}

// ============================================================
// 2. Throttle (Leading & Trailing Edge Support)
// ============================================================
function throttle(fn, delay, options = { leading: true, trailing: true }) {
  let timerId = null;
  let lastArgs = null;
  let lastThis = null;
  let lastCallTime = 0;

  return function throttled(...args) {
    const now = Date.now();
    lastArgs = args;
    lastThis = this;

    if (!lastCallTime && !options.leading) {
      lastCallTime = now;
    }

    const remainingTime = delay - (now - lastCallTime);

    if (remainingTime <= 0 || remainingTime > delay) {
      if (timerId) {
        clearTimeout(timerId);
        timerId = null;
      }
      lastCallTime = now;
      fn.apply(lastThis, lastArgs);
    } else if (!timerId && options.trailing) {
      timerId = setTimeout(() => {
        lastCallTime = options.leading ? Date.now() : 0;
        timerId = null;
        fn.apply(lastThis, lastArgs);
      }, remainingTime);
    }
  };
}
```

---

### 5. Object Operations: Deep Clone with Circular References

```javascript
function deepClone(target, hash = new WeakMap()) {
  // 1. Primitives, functions, and null are returned as-is
  if (target === null || typeof target !== "object") {
    return target;
  }

  // 2. Handle circular references
  if (hash.has(target)) {
    return hash.get(target);
  }

  // 3. Handle Special Built-in Objects
  if (target instanceof Date) return new Date(target.getTime());
  if (target instanceof RegExp) return new RegExp(target.source, target.flags);
  if (target instanceof Map) {
    const mapCopy = new Map();
    hash.set(target, mapCopy);
    target.forEach((val, key) => mapCopy.set(deepClone(key, hash), deepClone(val, hash)));
    return mapCopy;
  }
  if (target instanceof Set) {
    const setCopy = new Set();
    hash.set(target, setCopy);
    target.forEach((val) => setCopy.add(deepClone(val, hash)));
    return setCopy;
  }

  // 4. Preserve prototype chain (Array vs. Custom Class vs. Plain Object)
  const clone = Array.isArray(target)
    ? []
    : Object.create(Object.getPrototypeOf(target));

  hash.set(target, clone);

  // 5. Recursively clone own enumerable Symbol and String properties
  const keys = [...Object.keys(target), ...Object.getOwnPropertySymbols(target)];
  for (const key of keys) {
    clone[key] = deepClone(target[key], hash);
  }

  return clone;
}

// Example handling circular reference:
const node = { id: 101 };
node.self = node;
const clonedNode = deepClone(node);
console.log(clonedNode !== node); // true
console.log(clonedNode.self === clonedNode); // true (circular reference retained cleanly)
```

---

### 6. Complete Promises/A+ Specification Polyfill

```javascript
class MyPromise {
  static PENDING = "pending";
  static FULFILLED = "fulfilled";
  static REJECTED = "rejected";

  constructor(executor) {
    this.state = MyPromise.PENDING;
    this.value = undefined;
    this.reason = undefined;
    this.onFulfilledCallbacks = [];
    this.onRejectedCallbacks = [];

    const resolve = (value) => {
      // If resolved with another promise/thenable, adopt its state
      if (value instanceof MyPromise) {
        value.then(resolve, reject);
        return;
      }
      if (this.state === MyPromise.PENDING) {
        this.state = MyPromise.FULFILLED;
        this.value = value;
        this.onFulfilledCallbacks.forEach((fn) => fn());
      }
    };

    const reject = (reason) => {
      if (this.state === MyPromise.PENDING) {
        this.state = MyPromise.REJECTED;
        this.reason = reason;
        this.onRejectedCallbacks.forEach((fn) => fn());
      }
    };

    try {
      executor(resolve, reject);
    } catch (err) {
      reject(err);
    }
  }

  then(onFulfilled, onRejected) {
    // 2.2.1: onFulfilled and onRejected are optional arguments
    const realOnFulfilled = typeof onFulfilled === "function" ? onFulfilled : (v) => v;
    const realOnRejected = typeof onRejected === "function" ? onRejected : (r) => { throw r; };

    const promise2 = new MyPromise((resolve, reject) => {
      const handleFulfilled = () => {
        queueMicrotask(() => {
          try {
            const x = realOnFulfilled(this.value);
            resolvePromise(promise2, x, resolve, reject);
          } catch (err) {
            reject(err);
          }
        });
      };

      const handleRejected = () => {
        queueMicrotask(() => {
          try {
            const x = realOnRejected(this.reason);
            resolvePromise(promise2, x, resolve, reject);
          } catch (err) {
            reject(err);
          }
        });
      };

      if (this.state === MyPromise.FULFILLED) {
        handleFulfilled();
      } else if (this.state === MyPromise.REJECTED) {
        handleRejected();
      } else {
        this.onFulfilledCallbacks.push(handleFulfilled);
        this.onRejectedCallbacks.push(handleRejected);
      }
    });

    return promise2;
  }

  catch(onRejected) {
    return this.then(null, onRejected);
  }

  finally(callback) {
    return this.then(
      (value) => MyPromise.resolve(callback()).then(() => value),
      (reason) => MyPromise.resolve(callback()).then(() => { throw reason; })
    );
  }

  // ============================================================
  // Static Combinators
  // ============================================================
  static resolve(value) {
    if (value instanceof MyPromise) return value;
    return new MyPromise((resolve) => resolve(value));
  }

  static reject(reason) {
    return new MyPromise((_, reject) => reject(reason));
  }

  static all(promises) {
    return new MyPromise((resolve, reject) => {
      const results = [];
      let completed = 0;
      const array = Array.from(promises);

      if (array.length === 0) return resolve(results);

      array.forEach((p, i) => {
        MyPromise.resolve(p).then((value) => {
          results[i] = value;
          completed++;
          if (completed === array.length) {
            resolve(results);
          }
        }, reject);
      });
    });
  }

  static allSettled(promises) {
    return new MyPromise((resolve) => {
      const results = [];
      let completed = 0;
      const array = Array.from(promises);

      if (array.length === 0) return resolve(results);

      array.forEach((p, i) => {
        MyPromise.resolve(p).then(
          (value) => {
            results[i] = { status: "fulfilled", value };
            completed++;
            if (completed === array.length) resolve(results);
          },
          (reason) => {
            results[i] = { status: "rejected", reason };
            completed++;
            if (completed === array.length) resolve(results);
          }
        );
      });
    });
  }

  static race(promises) {
    return new MyPromise((resolve, reject) => {
      Array.from(promises).forEach((p) => {
        MyPromise.resolve(p).then(resolve, reject);
      });
    });
  }

  static any(promises) {
    return new MyPromise((resolve, reject) => {
      const errors = [];
      let rejectedCount = 0;
      const array = Array.from(promises);

      if (array.length === 0) {
        return reject(new AggregateError([], "All promises were rejected"));
      }

      array.forEach((p, i) => {
        MyPromise.resolve(p).then(
          resolve,
          (reason) => {
            errors[i] = reason;
            rejectedCount++;
            if (rejectedCount === array.length) {
              reject(new AggregateError(errors, "All promises were rejected"));
            }
          }
        );
      });
    });
  }
}

/**
 * Promises/A+ 2.3: Promise Resolution Procedure
 */
function resolvePromise(promise2, x, resolve, reject) {
  // 2.3.1: If promise2 and x refer to the same object, reject with a TypeError
  if (promise2 === x) {
    return reject(new TypeError("Chaining cycle detected for promise"));
  }

  if (x !== null && (typeof x === "object" || typeof x === "function")) {
    let called = false;
    try {
      const then = x.then;
      if (typeof then === "function") {
        then.call(
          x,
          (y) => {
            if (called) return;
            called = true;
            resolvePromise(promise2, y, resolve, reject);
          },
          (r) => {
            if (called) return;
            called = true;
            reject(r);
          }
        );
      } else {
        resolve(x);
      }
    } catch (err) {
      if (called) return;
      called = true;
      reject(err);
    }
  } else {
    resolve(x);
  }
}
```

---

### 7. Network Abort & Cancellation Pattern

```javascript
/**
 * Custom fetch wrapper with configurable timeout and abort signal
 */
async function fetchWithTimeout(url, options = {}) {
  const { timeout = 8000, ...fetchOptions } = options;

  const controller = new AbortController();
  const timeoutId = setTimeout(() => {
    controller.abort(new DOMException("Request timed out", "TimeoutError"));
  }, timeout);

  // If the caller already provided a signal, merge them
  if (options.signal) {
    options.signal.addEventListener("abort", () => {
      controller.abort(options.signal.reason);
    });
  }

  try {
    const response = await fetch(url, {
      ...fetchOptions,
      signal: controller.signal,
    });
    return response;
  } catch (error) {
    if (error.name === "AbortError" || error.name === "TimeoutError") {
      console.warn(`[Network Aborted]: ${error.message}`);
    }
    throw error;
  } finally {
    clearTimeout(timeoutId);
  }
}
```

---

## Comparison Matrix: Polyfill Edge Cases & Specification Subtleties

| Built-In API | Key Interview Gotchas | Specification Requirement | Recommended Defense |
|---|---|---|---|
| **`Array.prototype.map`** | Sparse arrays (`[1, , 3]`) | `HasProperty(O, Pk)` | Check `if (i in O)` before invoking callback |
| **`Array.prototype.reduce`** | Empty array without initial value | Must throw `TypeError` | Check if initial index exists; throw if array is empty |
| **`Function.prototype.bind`** | Invocation via `new boundFn()` | `this` must bind to the newly allocated instance | Check `this instanceof boundFn` and preserve prototype chain |
| **`Function.prototype.call`** | Collision with existing object keys | Temporary property deletion | Use `const key = Symbol("tempFn")` and `delete target[key]` |
| **`deepClone`** | Infinite recursion on cyclic structures | Retain reference parity | Track visited references in a `WeakMap` |
| **`MyPromise`** | Sync resolution starvation | Promises/A+ 2.2.4 Asynchrony | Use `queueMicrotask()` to defer callback execution |
| **`Promise.all`** | Out-of-order resolution index | Preserves input order | Store by index `results[i] = val` and count completions |

---

## Related Topics

- [[Function Borrowing, Explicit Binding & Function Currying|Function Borrowing, Explicit Binding & Function Currying]]

- [[JavaScript Promises & Async, Await. Architecture, Mechanics & Patterns|JavaScript Promises & Async/Await: Architecture, Mechanics & Patterns]]

- [[JavaScript Closures. Encapsulation, Currying & Output Puzzles|JavaScript Closures: Encapsulation, Currying & Output Puzzles]]

- [[The `this` Keyword & Execution Bindings|The this Keyword and Execution Bindings]]

- [[JavaScript Data Structures. Structured Data, Keyed & Indexed Collections|JavaScript Data Structures: Structured Data, Keyed & Indexed Collections]]

- [[AbortController & Multi-Signal Timeout Architectures|AbortController & Multi-Signal Timeout Architectures]]

- [[Event Loop Starvation. Causes, Mechanics & Mitigation Strategies|Event Loop Starvation: Causes, Mechanics & Mitigation Strategies]]

## Tags

#fullstack #interview #javascript #polyfills #promises #currying #call-apply-bind #debounce-throttle #deep-copy #abortcontroller

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups
