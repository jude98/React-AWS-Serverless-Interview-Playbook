# Strict Mode in JavaScript ("use strict")

## Key Concepts

> [!summary] What is Strict Mode?
>
> Introduced in ECMAScript 5 (ES5), **Strict Mode** is an opt-in mode that enforces a restricted variant of JavaScript. It converts silent errors into explicit runtime exceptions, disables deprecated/unsafe syntax, secures the `this` binding, and helps modern JavaScript engines optimize code execution.


> [!abstract] Enabling Strict Mode
>
> Activated using the string literal pragma `"use strict";` (or `'use strict';`):
>
> - **File-Level**: Placed at the very top of a script to apply to the entire file.
>
> - **Function-Level**: Placed as the first statement inside a function body to scope strictness only to that function.
>
> - **Automatic**: Implicitly enabled in **ECMAScript Modules (ESM)** (`import`/`export` or `<script type="module">`) and inside all **ES6 `class`** declarations/expressions.


> [!danger] Key Strictness Enforcements
>
> 1. **No Accidental Globals**: Assigning to an undeclared variable throws a `ReferenceError` instead of creating a global property on `window`/`global`.
>
> 2. **No Silent Failures**: Assigning to read-only properties, modifying frozen objects, or deleting non-configurable properties throws a `TypeError`.
>
> 3. **Secured `this` Context**: Standalone/free function invocations set `this` to `undefined` rather than the global object.
>
> 4. **No Duplicate Parameters**: Writing identical parameter names in a function declaration throws a `SyntaxError`.
>
> 5. **Banned Legacy Features**: Forbids the `with` statement and legacy octal literals (e.g., `010`).


> [!info] Optimization & Security Benefits
>
> Strict mode prevents variables from dynamically aliasing through deprecated features like `eval()` creating scope bindings or the `arguments.callee` pointer. Because scopes remain strictly lexical and predictable, JIT compilers (like V8) can perform aggressive optimizations (such as inline caching and dead-code elimination).


## Common Interview Questions

- "What is `'use strict'` and what problems does it solve compared to sloppy mode?"

- "What happens to the `this` keyword inside a regular, standalone function in strict mode?"

- "Can you name three operations that fail silently in sloppy mode but throw errors in strict mode?"

- "Why is `'use strict'` not required in modern React or Next.js codebases?"

- "How does strict mode change the behavior of the `arguments` object?"

- "What happens if you concatenate a strict-mode file with a non-strict-mode file in a legacy build pipeline?"

## Strong Answers / Talking Points

### 1. Eliminating Accidental Global Variables

- **Sloppy Mode**: Executing `count = 10;` without `let`, `const`, or `var` walks the entire scope chain. When reaching the global execution context without finding `count`, the engine creates a property `count` directly on the global object (`window.count = 10`). This creates severe namespace pollution and hard-to-trace bugs.

- **Strict Mode**: Halts execution and immediately throws `ReferenceError: count is not defined`.

### 2. Securing the `this` Context

- **Sloppy Mode**: Calling a free function (`fn()`) assigns `this` to the global object (`window` in browsers, `global` in Node.js). If the function attempts to modify `this.property`, it mutates global state.

- **Strict Mode**: `this` remains strictly `undefined`. If someone calls `this.property`, the engine halts with `TypeError: Cannot set properties of undefined`.

### 3. Turning Silent Failures into Throwing Errors

- **Read-Only / Non-Writable Properties**: Attempting to write to a property defined with `writable: false` or to global constants like `NaN = 5;` or `undefined = 5;` does nothing in sloppy mode. Strict mode throws `TypeError`.

- **Frozen / Sealed Objects**: Writing to an object frozen via `Object.freeze()` throws a `TypeError`.

- **Deleting Un-deletable Identifiers**: Calling `delete Object.prototype` or `delete myVar` throws a `TypeError` or `SyntaxError`.

### 4. Scoping & `eval` Isolation

- **Sloppy Mode**: Calling `eval("var secret = 42;")` injects `secret` directly into the enclosing lexical scope, creating unpredictable variable mutations.

- **Strict Mode**: `eval()` runs inside its own isolated lexical environment, preventing variables declared inside `eval` from leaking into the containing scope.

### 5. `arguments` Object Disconnection

- **Sloppy Mode**: Modifying named parameters modifies the `arguments` array entries and vice versa (aliasing).

- **Strict Mode**: Named parameters and `arguments[i]` are decoupled; changing one does not affect the other. Additionally, `arguments.callee` and `arguments.caller` are deprecated and throw errors if accessed.

## Code Snippets / Examples

### Declaring Strict Mode (Global vs. Local)

```javascript
// File-level strict mode (must be first line)
"use strict";

function testGlobalStrict() {
  // x = 10; // ReferenceError: x is not defined
}

// Function-level strict mode (when applied in non-strict scripts)
function specificStrictScope() {
  "use strict";
  // y = 20; // ReferenceError: y is not defined
}

function sloppyScope() {
  z = 30; // Works silently; creates window.z
}
```

### Sloppy Mode Silent Failures vs. Strict Mode Exceptions

```javascript
"use strict";

// 1. Assignment to non-writable property
const config = {};
Object.defineProperty(config, "apiKey", {
  value: "SECRET_KEY",
  writable: false
});
// config.apiKey = "NEW_KEY"; // TypeError: Cannot assign to read only property 'apiKey'

// 2. Modifying frozen object
const user = Object.freeze({ name: "Alice" });
// user.name = "Bob"; // TypeError: Cannot assign to read only property 'name'

// 3. Deleting non-configurable property
// delete Object.prototype; // TypeError: Cannot delete property 'prototype'

// 4. Duplicate parameter names
// function sum(a, a, b) {} // SyntaxError: Duplicate parameter name not allowed in this context
```

### Standalone Function `this` Resolution

```javascript
function showThisSloppy() {
  return this;
}

function showThisStrict() {
  "use strict";
  return this;
}

console.log(showThisSloppy() === window); // true (in browser)
console.log(showThisStrict());            // undefined
```

### Decoupling `arguments` from Named Parameters

```javascript
function sloppyAlias(a) {
  a = 42;
  return arguments[0]; // Returns 42 (aliased to parameter)
}

function strictNoAlias(a) {
  "use strict";
  a = 42;
  return arguments[0]; // Returns 10 (retains original argument passed)
}

console.log(sloppyAlias(10));   // 42
console.log(strictNoAlias(10)); // 10
```

## Comparison Matrix: Sloppy Mode vs. Strict Mode

|**Feature / Behavior**|**Sloppy Mode (Default)**|**Strict Mode ("use strict")**|
|---|---|---|
|**Undeclared Variable Assignment (`x = 1`)**|Creates global variable on `window`/`global`|Throws `ReferenceError`|
|**Standalone Function `this`**|Points to global object (`window`/`global`)|Evaluates strictly to `undefined`|
|**Writing to Read-Only / Frozen Property**|Fails silently|Throws `TypeError`|
|**Duplicate Function Parameter Names**|Allowed (last parameter shadows prior)|Throws `SyntaxError`|
|**`eval()` Variable Injection**|Leaks variables into enclosing scope|Contained within `eval` local scope|
|**`with` Statement**|Allowed|Throws `SyntaxError`|
|**Legacy Octal Literals (`010`)**|Allowed (evaluates to `8`)|Throws `SyntaxError`|
|**Modules (ESM) & Classes**|Not applicable|**Always enabled by default**|

## Related Topics

- [[JavaScript Execution Context, Memory Creation & Hoisting Mechanics]]

- [[JavaScript Variables, Scopes, and Execution Context|JavaScript Variable Declarations: var, let, and const]]

- [[The `this` Keyword & Execution Bindings|The this Keyword and Execution Bindings]]

- [[JavaScript Scope, Lexical Environment, and Shadowing]]

## Tags

#fullstack #interview #javascript #strict-mode #use-strict #runtime-rules #clean-code

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups
