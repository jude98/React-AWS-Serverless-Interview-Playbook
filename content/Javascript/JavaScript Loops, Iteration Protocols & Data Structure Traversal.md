# JavaScript Loops, Iteration Protocols & Data Structure Traversal

## Key Concepts

> [!summary] Iteration Protocols (Iterable vs. Iterator)
>
> An object is an **Iterable** if it implements the `[Symbol.iterator]` method, which returns an **Iterator**. An **Iterator** is an object with a `.next()` method returning `{ value: any, done: boolean }`.
>
> - **Built-in Iterables**: Arrays, Strings, Maps, Sets, `TypedArrays`, `NodeList`, `arguments`.
>
> - **Not Iterable**: Plain Objects (`{}`) do _not_ implement `[Symbol.iterator]` by default.


> [!abstract] Loop Constructs Comparison
>
> - `for`: Classic indexed counter loop; best for fine-grained index control, reverse iterations, or multi-step increments.
>
> - `while` / `do...while`: Condition-first or condition-last evaluation; used when iteration count is non-deterministic.
>
> - `for...of`: Traverses **values** of any **Iterable**; cleanly supports `break`, `continue`, `return`, and `await` (with `for await...of`).
>
> - `for...in`: Traverses **enumerable property keys** of an object (including inherited properties across the prototype chain); **never** recommended for arrays.
>
> - `Array.prototype.forEach`: Higher-order function; **cannot be broken or stopped early** via `break` or `continue`.


> [!danger] Control Flow: `break` vs. `continue` vs. `labeled statements`
>
> - `break`: Immediately terminates the innermost (or labeled) loop/switch construct.
>
> - `continue`: Skips the remainder of the current iteration body and evaluates the loop condition/increment step.
>
> - **Labeled Statements**: Identifiers preceding a loop (e.g., `outerLoop: for (...)`) enabling targeted `break` or `continue` from deeply nested loops.


## Common Interview Questions

- "What is the difference between `for...in` and `for...of`?"

- "Can you stop or break out of an `Array.prototype.forEach()` loop? What happens if you try to `return` inside it?"

- "Why shouldn't you use `for...in` to iterate over an Array?"

- "How do you make a plain JavaScript object iterable using `for...of`?"

- "What is the difference between `map()` and `forEach()`?"

- "How do you loop through an array asynchronously ensuring tasks run in sequence vs. in parallel?"

## Strong Answers / Talking Points

### 1. `for...in` vs. `for...of`

- **`for...in`**:

    - Iterates over **all enumerable keys/properties** (strings/symbols), traversing up the entire prototype chain unless filtered with `Object.hasOwn(obj, key)`.

    - Does not guarantee index order on arrays; array indices are treated as string property keys (`"0"`, `"1"`).

- **`for...of`**:

    - Leverages the `Symbol.iterator` protocol.

    - Iterates directly over **values**.

    - Does not touch prototype properties.

### 2. Breaking Out of Loops

- `break` and `continue` work with `for`, `for...of`, `for...in`, `while`, and `do...while`.

- Higher-order methods (`.forEach()`, `.map()`, `.filter()`, `.reduce()`) **cannot be stopped** via `break` or `continue` (throwing a `SyntaxError`). Returning from the callback simply exits that single iteration's callback function (functioning like `continue`).

- _Alternative_: To break early with functional semantics, use `.some()` (break on `true`), `.every()` (break on `false`), or standard `for...of`.

### 3. Iterating Over Plain Objects

- Plain objects are not iterables. To traverse an object, convert it into an iterable array via:

    - `Object.keys(obj)`: Array of own enumerable property names.

    - `Object.values(obj)`: Array of own enumerable property values.

    - `Object.entries(obj)`: Array of `[key, value]` tuples.

- Or use `for...of` on `Object.entries(obj)` with destructuring.

### 4. Async Traversal: Sequential vs. Parallel

- **Sequential**: `for...of` with `await` pauses execution of each step until the promise resolves.

- **Parallel**: `array.map(async ...)` launches all async operations concurrently, followed by `Promise.all()`.

- **Anti-pattern**: Using `await` inside `forEach` does _not_ pause the loop; `forEach` is not promise-aware and executes all iterations synchronously without waiting.

## Code Snippets / Examples

### Traversal by Data Structure

```javascript
// 1. Array Traversal
const nums = [10, 20, 30];

for (const [index, val] of nums.entries()) {
  if (val === 20) continue;
  console.log(`Index: ${index}, Value: ${val}`);
}

// 2. String Traversal (Handles Unicode code points correctly)
const text = "Hi 🚀";
for (const char of text) {
  console.log(char); // "H", "i", " ", "🚀"
}

// 3. Map Traversal
const userRoles = new Map([
  ["alice", "admin"],
  ["bob", "editor"]
]);

for (const [username, role] of userRoles) {
  console.log(`${username}: ${role}`);
}

// 4. Object Traversal (Clean modern approach)
const user = { id: 1, name: "Alice", active: true };

for (const [key, value] of Object.entries(user)) {
  console.log(`${key} => ${value}`);
}
```

### Labeled Statement & Break Traps

```javascript
// Breaking out of an outer loop using a label
outerLoop: for (let i = 0; i < 3; i++) {
  for (let j = 0; j < 3; j++) {
    if (i === 1 && j === 1) {
      break outerLoop; // Exits both loops immediately
    }
    console.log(`i=${i}, j=${j}`);
  }
}

// Breaking early with functional array methods (.some)
const items = [1, 2, 3, 4, 5];
items.some((item) => {
  if (item === 3) return true; // Acts like 'break'
  console.log(item); // Logs: 1, 2
  return false;
});
```

### Making a Plain Object Iterable

```javascript
const collection = {
  items: ["Alpha", "Beta", "Gamma"],
  [Symbol.iterator]() {
    let index = 0;
    return {
      next: () => {
        if (index < this.items.length) {
          return { value: this.items[index++], done: false };
        }
        return { value: undefined, done: true };
      }
    };
  }
};

for (const item of collection) {
  console.log(item); // Logs: "Alpha", "Beta", "Gamma"
}
```

### Asynchronous Loops: Sequential vs. Broken forEach

```javascript
const fetchItem = (id) => new Promise(res => setTimeout(() => res(id * 10), 100));

// 1. Sequential: for...of with await
async function processSequentially(ids) {
  for (const id of ids) {
    const data = await fetchItem(id); // Waits for completion before next iteration
    console.log(data);
  }
}

// 2. Parallel: Promise.all + map
async function processInParallel(ids) {
  const promises = ids.map(id => fetchItem(id));
  const results = await Promise.all(promises);
  console.log(results);
}

// 3. Pitfall: forEach does NOT wait for await!
async function brokenForEach(ids) {
  ids.forEach(async (id) => {
    const data = await fetchItem(id);
    console.log(data); // Will run fire-and-forget in parallel unhandled
  });
  console.log("Done"); // Logs BEFORE promises resolve!
}
```

## Comparison Matrix

|**Construct**|**Iterates Over**|**Target Data Structures**|**Supports break / continue**|**Async-safe (await)**|
|---|---|---|---|---|
|**`for`**|Index counter|Arrays, Indexables|Yes|Yes|
|**`while` / `do...while`**|Conditional state|Arbitrary expressions|Yes|Yes|
|**`for...of`**|Values|Iterables (Array, Map, Set, String)|Yes|Yes|
|**`for...in`**|Enumerable keys|Plain Objects|Yes|Yes|
|**`.forEach()`**|Values + Indices|Arrays, Maps, Sets|No|No (fire-and-forget)|

## Related Topics

- [[JavaScript Data Structures. Structured Data, Keyed & Indexed Collections|JavaScript Data Structures: Structured Data, Keyed & Indexed Collections]]

- [[JavaScript Loops, Iteration Protocols & Data Structure Traversal|Iterables, Iterators, and Generators]]

- [[JavaScript Promises & Async, Await. Architecture, Mechanics & Patterns|Asynchronous JavaScript: Promises, Async/Await and Event Loop]]

- [[JavaScript Expressions, Operators & Output Prediction|JavaScript Functional Array Methods: map, filter, and reduce]]

## Tags

#fullstack #interview #javascript #loops #iteration #iterables #control-flow

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups
