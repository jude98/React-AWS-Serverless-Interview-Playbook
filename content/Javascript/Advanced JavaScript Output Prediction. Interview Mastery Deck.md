# Advanced JavaScript Output Prediction: Interview Mastery Deck

A curated collection of tricky, high-signal JavaScript output prediction puzzles covering runtime mechanics, edge cases, scope, coercion, and concurrency.

---

## 1. Concurrency, Event Loop & Microtasks

### Puzzle 1.1: Mixed Async/Await, Microtasks, and Macrotasks

```javascript
console.log("1");

setTimeout(() => {
  console.log("2");
  Promise.resolve().then(() => console.log("3"));
}, 0);

new Promise((resolve) => {
  console.log("4");
  resolve();
  console.log("5");
}).then(() => {
  console.log("6");
});

async function asyncFn() {
  console.log("7");
  await null;
  console.log("8");
  await Promise.resolve();
  console.log("9");
}

asyncFn();
console.log("10");
```

**Output:**

```text
1
4
5
7
10
6
8
9
2
3
```

**Walkthrough:**

1. `console.log("1")` runs synchronously $\to$ logs `1`.
2. `setTimeout` registers a callback to the **Macrotask Queue** (Task Queue).
3. The `new Promise` executor runs **synchronously**:
* Logs `4`.
* `resolve()` moves the promise to fulfilled and enqueues `.then()` to the **Microtask Queue**.
* Logs `5`.

4. `asyncFn()` is called:
* Logs `7` synchronously.
* Hits `await null`. The expression is evaluated, and the remainder of `asyncFn` is queued as a **microtask**.

5. `console.log("10")` runs synchronously $\to$ logs `10`.
6. **Call stack is now empty.** The engine drains the **Microtask Queue**:
* First microtask: Promise `.then()` logs `6`.
* Second microtask: Post-`await null` of `asyncFn()` resumes $\to$ logs `8`.
* Hits `await Promise.resolve()`. The rest of `asyncFn()` is enqueued as a new microtask.
* Third microtask: The resumed `asyncFn()` logs `9`.

7. **Microtask queue is empty.** The engine takes **one macrotask** (`setTimeout`):
* Logs `2`.
* `Promise.resolve().then(...)` schedules a new microtask.
* The current macrotask finishes.

8. The engine drains the microtask queue before picking another macrotask $\to$ logs `3`.

---

### Puzzle 1.2: Node.js vs. Browser Environment Hierarchy

```javascript
// Run in Node.js vs. modern browser
setTimeout(() => console.log("setTimeout"), 0);
Promise.resolve().then(() => console.log("Promise"));
queueMicrotask(() => console.log("queueMicrotask"));

if (typeof process !== "undefined" && process.nextTick) {
  process.nextTick(() => console.log("nextTick"));
}
```

**In Node.js:**

```text
nextTick
Promise
queueMicrotask
setTimeout
```

**In Modern Browser:**

```text
Promise
queueMicrotask
setTimeout
```

**Walkthrough:**

* In Node.js, `process.nextTick` maintains its own dedicated queue that executes **immediately after the current tick** of the Call Stack, running *before* standard microtasks (Promises and `queueMicrotask`).
* In both runtimes, all microtasks drain completely before the macrotask (`setTimeout`) executes.

---

## 2. Scope, Closures, Hoisting & Temporal Dead Zone (TDZ)

### Puzzle 2.1: Hoisting with Functions vs. Variables

```javascript
var a = 1;

function b() {
  a = 10;
  return;
  function a() {}
}

b();
console.log(a);
```

**Output:**

```text
1
```

**Walkthrough:**
Inside function `b()`, function declarations are hoisted to the top of the function's local scope:

```javascript
function b() {
  function a() {} // 'a' is a local variable pointing to a function!
  a = 10;         // Mutates the LOCAL variable 'a', NOT the outer global 'a'
  return;
}
```

The mutation `a = 10` affects only the local binding. The global `a` remains `1`.

---

### Puzzle 2.2: Temporal Dead Zone (TDZ) & Shadowing

```javascript
let x = 10;

function test(a = x, b = () => x) {
  let x = 20;
  console.log(a);
  console.log(b());
}

test();
```

**Output:**

```text
10
20
```

**Walkthrough:**

1. Parameters have their own scope, evaluated left-to-right between the outer scope and the function body.
2. `a = x`: The local `let x = 20` inside the function body does not exist yet. `x` resolves to the outer `let x = 10`. So `a = 10`.
3. `b = () => x`: The arrow function captures `x` by reference. When `b()` executes inside the body, the function body's local environment record has initialized `let x = 20`. The closure resolves `x` as `20`.

*Variation with TDZ Error:*

```javascript
let y = 1;
function fail(y = y) {} // ReferenceError: Cannot access 'y' before initialization
```

---

### Puzzle 2.3: `var` in Loops vs. `let`

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log("var:", i), 0);
}

for (let j = 0; j < 3; j++) {
  setTimeout(() => console.log("let:", j), 0);
}
```

**Output:**

```text
var: 3
var: 3
var: 3
let: 0
let: 1
let: 2
```

**Walkthrough:**

* `var` is function/globally scoped. A single shared variable `i` is mutated. By the time the timers fire in the macrotask queue, the loop has completed and `i === 3`.
* `let` is block-scoped. The ECMAScript specification mandates that for `for (let ...)`, a **new lexical environment binding** is created for each iteration, preserving values `0`, `1`, and `2` within each callback's closure.

---

## 3. The `this` Keyword & Arrow Functions

### Puzzle 3.1: Implicit Binding vs. Arrow Lexical `this`

```javascript
const obj = {
  name: "Outer",
  regularFn: function () {
    return this.name;
  },
  arrowFn: () => {
    return this.name;
  },
  nested: {
    name: "Inner",
    getNames() {
      const arrow = () => this.name;
      return [this.regularFn ? this.regularFn() : "N/A", arrow()];
    }
  }
};

const extracted = obj.regularFn;

console.log(obj.regularFn());
console.log(extracted());
console.log(obj.arrowFn());
console.log(obj.nested.getNames());
```

**Output (Non-Strict Mode Browser):**

```text
Outer
undefined (or window.name if set)
undefined (or window.name if set)
["N/A", "Inner"]
```

**Output (Strict Mode `'use strict';`):**

```text
Outer
TypeError: Cannot read properties of undefined (reading 'name')
```

**Walkthrough:**

1. `obj.regularFn()`: Invoked with implicit context `obj` $\to$ `this` is `obj` $\to$ `"Outer"`.
2. `extracted()`: Standalone invocation. In non-strict mode, `this` falls back to `window` (or `global`). In strict mode, `this` is `undefined`, throwing a `TypeError`.
3. `obj.arrowFn()`: Arrow functions do not bind their own `this`. They capture `this` from their enclosing lexical context at creation time. Here, the outer context is the global/module scope, not `obj` (object literals do not introduce a new lexical scope).
4. `obj.nested.getNames()`: Inside `getNames`, `this` is `obj.nested`. `this.regularFn` is `undefined` on `nested`. The arrow function captures `this` from `getNames`, resolving to `obj.nested.name` $\to$ `"Inner"`.

---

### Puzzle 3.2: `this` with Class Fields and Inheritance

```javascript
class Parent {
  name = "Parent";

  getName = () => {
    return this.name;
  };
}

class Child extends Parent {
  name = "Child";
}

const instance = new Child();
const { getName } = instance;
console.log(instance.getName());
console.log(getName());
```

**Output:**

```text
Child
Child
```

**Walkthrough:**

* Public class arrow fields (`getName = () => ...`) are created on each **instance** during constructor execution, not on the prototype.
* In `Child`, `super()` runs the `Parent` constructor, which binds the arrow function with `this` anchored to the newly created instance.
* Even when destructured (`const { getName } = instance`), the arrow function permanently retains its lexical `this` reference to the instance. When accessed, `instance.name` has been overwritten by `Child` to `"Child"`.

---

## 4. Type Coercion, Equality Operators (`==` vs `===`), and Objects

### Puzzle 4.1: The Equality Gauntlet

```javascript
console.log([] == ![]);
console.log([] == 0);
console.log([""] == false);
console.log(null == undefined);
console.log(null === undefined);
console.log(null == 0);
console.log(null > 0);
console.log(null >= 0);
console.log(NaN == NaN);
console.log(Object.is(NaN, NaN));
```

**Output:**

```text
true
true
true
true
false
false
false
true
false
true
```

**Walkthrough:**

1. `[] == ![]`:
* `!` has higher precedence than `==`. `Boolean([])` is truthy, so `![]` evaluates to `false`.
* `[] == false` triggers coercion: `false` converts to number `0`.
* `[] == 0`: Array coerced via `ToPrimitive([])` $\to$ `""` $\to$ Number `0`.
* `0 == 0` evaluates to `true`.

2. `[] == 0`: `ToPrimitive([])` is `""`, and `Number("")` is `0` $\to$ `0 == 0` is `true`.
3. `[""] == false`: `[""].toString()` is `""`. `"" == false` $\to$ `0 == 0` $\to$ `true`.
4. `null == undefined`: Specified by the ECMAScript spec as loose equal (`true`), but strict inequality (`false`).
5. Relational comparison vs Equality for `null`:
* Equality (`==`) does not convert `null` to a number (it only loosely equals `undefined`). Thus, `null == 0` is `false`.
* Relational operators (`>`, `<`, `>=`, `<=`) coerce operands using `ToNumeric`. `null` converts to `+0`.
* `null > 0` $\to$ `0 > 0` (`false`).
* `null >= 0` $\to$ `0 >= 0` (`true`).

6. `NaN == NaN`: `NaN` is not equal to any value, including itself. `Object.is(NaN, NaN)` uses SameValue equality $\to$ `true`.

---

### Puzzle 4.2: Object Keys and Coercion

```javascript
const a = {};
const b = { key: "b" };
const c = { key: "c" };

a[b] = 123;
a[c] = 456;

console.log(a[b]);
```

**Output:**

```text
456
```

**Walkthrough:**

* Plain JavaScript object keys must be either `String` or `Symbol`.
* When an object (`b`) is used as a property key, it is coerced to a string via `b.toString()`, which returns `"[object Object]"`.
* `a[b] = 123` sets `a["[object Object]"] = 123`.
* `a[c] = 456` overwrites `a["[object Object]"] = 456`.
* `a[b]` retrieves `a["[object Object]"]`, which is `456`.
* *Fix*: Use `Map` if you need object references as keys (`const map = new Map()`).

---

## 5. Operators, Precedence, and Evaluation Order

### Puzzle 5.1: Increment/Decrement and Chained Assignment

```javascript
let a = 1;
let b = (a++) + (++a) * (a--);
console.log("a:", a);
console.log("b:", b);

let x = 1;
let y = x = typeof z;
console.log("x:", x);
console.log("y:", y);
```

**Output:**

```text
a: 2
b: 10
x: "undefined"
y: "undefined"
```

**Walkthrough:**

1. Step-by-step evaluation of `(a++) + (++a) * (a--)`:
* `(a++)`: Evaluates to `1`, but `a` becomes `2`.
* `(++a)`: `a` increments to `3`, evaluates to `3`.
* `(a--)`: Evaluates to `3`, then `a` decrements to `2`.
* Multiplication has higher precedence: `3 * 3 = 9`.
* Addition: `1 + 9 = 10`.
* Final `a` is `2`; `b` is `10`.

2. `typeof z`: `typeof` on an undeclared variable is safe and returns `"undefined"`.
* Assignment operators group right-to-left: `x = "undefined"`, then `y = "undefined"`.

---

### Puzzle 5.2: Nullish Coalescing (`??`) vs. Logical OR (`||`)

```javascript
const config = {
  zero: 0,
  blank: "",
  falsy: false,
  nil: null,
  undef: undefined
};

console.log(config.zero || 10);
console.log(config.zero ?? 10);

console.log(config.blank || "default");
console.log(config.blank ?? "default");

console.log(config.falsy || true);
console.log(config.falsy ?? true);

console.log(config.nil ?? "fallback");
console.log(config.undef ?? "fallback");
```

**Output:**

```text
10
0
default
""
true
false
fallback
fallback
```

**Walkthrough:**

* Logical OR (`||`) returns the right-hand operand if the left operand is **any falsy value** (`0`, `""`, `false`, `null`, `undefined`, `NaN`).
* Nullish Coalescing (`??`) returns the right-hand operand **only** if the left operand is **nullish** (`null` or `undefined`). `0`, `""`, and `false` are considered defined values and are preserved.

---

## 6. Strict Mode vs. Sloppy Mode

### Puzzle 6.1: Octal Literals, Arguments Shadowing, and Read-Only Props

```javascript
function nonStrict(a) {
  arguments[0] = 99;
  console.log(a);
}
nonStrict(10);

function strict(a) {
  "use strict";
  arguments[0] = 99;
  console.log(a);
}
strict(10);
```

**Output:**

```text
99
10
```

**Walkthrough:**

* In non-strict mode, the `arguments` object is **aliased** to the function's named parameters. Mutating `arguments[0]` directly updates `a`.
* In strict mode (`"use strict"`), `arguments` maintains an unlinked snapshot of parameters passed at call time. Modifying `arguments[0]` does **not** mutate `a`.

---

### Puzzle 6.2: Writing to Non-Writable Properties

```javascript
const user = {};
Object.defineProperty(user, "role", {
  value: "admin",
  writable: false
});

function mutateUser() {
  user.role = "guest";
  return user.role;
}

console.log(mutateUser());
```

**In Non-Strict Mode:**

```text
admin
```

*(Fails silently; assignment is ignored).*

**In Strict Mode (`"use strict"`):**

```text
TypeError: Cannot assign to read only property 'role' of object '#<Object>'
```

---

## 7. Generator, Promise & Async Iteration Interleaving

### Puzzle 7.1: Generators Interleaved with Microtasks

```javascript
function* numberGen() {
  console.log("G1");
  yield 1;
  console.log("G2");
  yield 2;
  console.log("G3");
}

console.log("Start");

const gen = numberGen();

Promise.resolve().then(() => {
  console.log("P1");
  console.log("Gen in Promise:", gen.next().value);
});

console.log("Gen sync:", gen.next().value);

Promise.resolve().then(() => {
  console.log("P2");
});

console.log("End");
```

**Output:**

```text
Start
G1
Gen sync: 1
End
P1
G2
Gen in Promise: 2
P2
```

**Walkthrough:**

1. `Start` is logged synchronously.
2. `numberGen()` returns a generator iterator object (does not run code inside the body yet).
3. First Promise schedules `.then(...)` into the Microtask Queue.
4. `gen.next().value`: Runs generator up to first `yield`:
* Logs `G1`.
* Yields `1`.
* Synchronous log: `Gen sync: 1`.

5. Second Promise schedules its `.then(...)` into the Microtask Queue.
6. Synchronous log: `End`.
7. **Stack clears; microtasks drain:**
* First microtask runs: Logs `P1`. Calls `gen.next().value`, resuming generator:
* Logs `G2`.
* Yields `2`.
* Logs `Gen in Promise: 2`.

* Second microtask runs: Logs `P2`.

---

## 8. Quick-Fire Cheat Sheet for Interviews

| Pattern / Gotcha | Key Takeaway |
| --- | --- |
| `[] == ![]` | Evaluates to `true` (Logical NOT converts `[]` to `false`, then coerced to `0 == 0`). |
| `typeof null` | Returns `"object"` (historical bug in JS from the first version). |
| `0.1 + 0.2 === 0.3` | Evaluates to `false` (IEEE 754 floating-point precision error; equals `0.30000000000000004`). |
| `[1, 2, 3] + [4, 5, 6]` | Returns `"1,2,34,5,6"` (Array coercion calls `.toString()` and concatenates strings). |
| `await` in loops | `Array.prototype.forEach` does **not** wait for `await`; use `for...of` for sequential execution. |
| Object key ordering | Integer-like keys are sorted numerically first; all other strings/symbols follow insertion order. |
| Arrow function `this` | Determined **lexically at declaration**, never dynamically altered by `.call()`, `.apply()`, or `.bind()`. |

## Related Topics

- [[JavaScript Expressions, Operators & Output Prediction]]
- [[JavaScript Type Casting Coercion vs. Conversion & Predict-the-Output]]
- [[The `this` Keyword & Execution Bindings]]
- [[JavaScript Execution Context, Memory Creation & Hoisting Mechanics]]
- [[JavaScript Closures. Encapsulation, Currying & Output Puzzles]]
