
## Key Concepts

> [!summary] Strict Mode (`"use strict"`)
> 
> An opt-in pragma introduced in ECMAScript 5 (ES5) that enforces stricter parsing and error handling rules at runtime. It eliminates silent failures by turning them into explicit runtime exceptions, disables features that confuse JavaScript engines (aiding engine optimization), and prevents unsafe global variable leakage.
> 
>   

> [!abstract] Automatic Strict Mode Contexts
> 
> You do **not** need to declare `"use strict"` manually in modern JavaScript if your code runs inside:
> 
>   
> 
> - **ECMAScript Modules (ESM)**: Modules (`import` / `export` or `<script type="module">`) are strictly evaluated in strict mode by default.
>     
>       
>     
> - **ES6 Classes**: All code written inside class declarations and class expressions (constructors and methods) executes automatically in strict mode.
>     
>       
>     

> [!info] The Global Object Across Runtimes
> 
> The root container providing global variables, built-in functions (`parseInt`, `fetch`), and runtime-specific APIs:
> 
>   
> 
> - **Browser**: Represented by `window` (and `self` inside Web Workers).
>     
>       
>     
> - **Node.js**: Represented by `global`.
>     
>       
>     
> - **Universal Standard**: `globalThis` (ES2020) provides a unified, cross-platform pointer to the global object regardless of environment.
>     
>       
>     

> [!danger] Default `this` Binding Behavior
> 
> In standard (sloppy) mode, calling a standalone function (`foo()`) binds `this` to the global object (`window` or `global`). In strict mode, `this` in standalone functions defaults strictly to `undefined`, preventing accidental mutation of global state.
> 
>   

## Common Interview Questions

- "What is `'use strict'` and what specific problems does it solve in legacy JavaScript?"
    
      
    
- "What happens to the `this` keyword inside a free/standalone function in strict mode vs. sloppy mode?"
    
      
    
- "Why does assignment to an undeclared variable behave differently in strict mode?"
    
      
    
- "Where is strict mode enabled automatically without writing the string pragma?"
    
      
    
- "How does the global object differ between the browser, Node.js, and Web Workers, and why was `globalThis` introduced?"
    
      
    
- "Can you apply `'use strict'` to only a single function instead of the entire file?"
    
      
    

## Strong Answers / Talking Points

### 1. Key Behavioral Shifts Enforced by `"use strict"`

- **Prevents Accidental Globals**: In sloppy mode, assigning to an undeclared variable (`x = 42`) implicitly attaches `x` to `window`/`global`. Strict mode throws an explicit `ReferenceError: x is not defined`.
    
      
    
- **Eliminates Silent Assignment Failures**: Attempting to write to read-only properties (e.g., modifying `NaN` or a property marked `writable: false`, or writing to an `Object.freeze()` object) fails silently in sloppy mode, but throws a `TypeError` in strict mode.
    
      
    
- **De-duplication of Function Parameter Names**: Writing `function sum(a, a, c) {}` is permitted in sloppy mode (the latter shadows the former), but throws a `SyntaxError` in strict mode.
    
      
    
- **Securing `this`**: Prevents accidental leakage or modification of `window` by setting default function call `this` to `undefined`.
    
      
    
- **Disables Deprecated Syntax**: Disallows the `with` statement and disallows octal numeric literals using legacy zero prefixes (`010`).
    
      
    

### 2. Scope & Application Scenarios of `"use strict"`

- **File-Level**: Placed at the very top of a script before any code, applying to all statements in that file.
    
      
    
- **Function-Level**: Placed at the very top of a specific function body, scoping strictness only to that function and its inner functions.
    
      
    
- **Implicit Strictness**: Modern full-stack codebases (Vite, Next.js, Webpack, NestJS, TypeScript) transpile or bundle into ESM modules or ES6 classes, making manual `"use strict"` declarations largely unnecessary in modern projects.
    
      
    

### 3. Global Object Evolution: `window` vs. `global` vs. `globalThis`

- **Browser (`window`)**: Combines JavaScript's runtime global state with the Browser Object Model (BOM) and Document Object Model (DOM) APIs (`window.document`, `window.localStorage`, `window.location`).
    
      
    
- **Node.js (`global`)**: Contains server-side primitives like `process`, `Buffer`, and timer functions (`setImmediate`), but has no DOM or UI APIs.
    
      
    
- **Web Workers (`self`)**: Have no access to `window` or DOM, relying on `DedicatedWorkerGlobalScope` accessible via `self`.
    
      
    
- **`globalThis`**: Eliminates fragile environment sniffing (e.g., `typeof window !== 'undefined' ? window : global`) by providing a standardized cross-platform reference.
    
      
    

## Code Snippets / Examples

### Strict Mode vs. Sloppy Mode Violations

JavaScript

```
// Function-level strict mode example
function sloppyFunction() {
  leakedGlobal = "I pollute the global scope"; // Allowed in sloppy mode
}
sloppyFunction();
console.log(window.leakedGlobal); // "I pollute the global scope"

function strictFunction() {
  "use strict";
  // strictVariable = "Error thrown"; // Uncaught ReferenceError: strictVariable is not defined
}
strictFunction();
```

### Standalone Function `this` Resolution

JavaScript

```
function checkThisSloppy() {
  return this;
}

function checkThisStrict() {
  "use strict";
  return this;
}

console.log(checkThisSloppy() === window); // true (in browser)
console.log(checkThisStrict());            // undefined
```

### Silent Failures Turned to Explicit Errors

JavaScript

```
"use strict";

// 1. Assignment to non-writable property
const user = {};
Object.defineProperty(user, "id", { value: 101, writable: false });
// user.id = 202; // Uncaught TypeError: Cannot assign to read only property 'id'

// 2. Modifying a frozen object
const config = Object.freeze({ env: "production" });
// config.env = "staging"; // Uncaught TypeError: Cannot assign to read only property 'env'

// 3. Deleting un-deletable properties
// delete Object.prototype; // Uncaught TypeError: Cannot delete property 'prototype'
```

### Cross-Environment Global Access (`globalThis`)

JavaScript

```
// Universal way to access the global scope across Node.js and Browser
function getGlobal() {
  return globalThis;
}

console.log(globalThis === window); // true in browser main thread
// console.log(globalThis === global); // true in Node.js
```

## Related Topics

- [[JavaScript Execution Context, Memory Creation & Hoisting Mechanics]]
    
      
    
- [[JavaScript Variable Declarations: var, let, and const]]
    
      
    
- [[The this Keyword and Execution Bindings]]
    
      
    
- [[JavaScript Modules: CommonJS vs ECMAScript Modules]]
    
      
    

## Tags

#fullstack #interview #javascript #strict-mode #global-object #globalthis

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups