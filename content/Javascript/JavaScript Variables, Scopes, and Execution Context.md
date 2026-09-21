
## Key Concepts

> [!summary] Scope Boundaries
> 
>   
> 
> - `var` is **function-scoped** (or global). It ignores block delimiters (`if`, `for`, `while`, `{}`) and only respects boundaries created by functions.
>     
>       
>     
> - `let` and `const` are **block-scoped**. They are strictly confined to the closest enclosing pair of curly braces `{}`.
>     
>       
>     

> [!abstract] Hoisting & Initialization Behavior
> 
>   
> 
> - `var`: Registered in memory during Phase 1 (Creation Phase) and immediately initialized to `undefined`.
>     
>       
>     
> - `let` and `const`: Registered in memory during Phase 1, but left **uninitialized**. They enter the **Temporal Dead Zone (TDZ)** and throw a `ReferenceError` if accessed before the declaration line in Phase 2.
>     
>       
>     

> [!info] Re-declaration & Reassignment
> 
>   
> 
> - `var`: Can be re-declared and reassigned within the same scope without error.
>     
>       
>     
> - `let`: Can be reassigned, but **cannot** be re-declared within the same scope.
>     
>       
>     
> - `const`: Cannot be re-declared and **cannot be reassigned**; must be initialized immediately at the declaration statement.
>     
>       
>     

> [!tip] Global Object Pollution
> 
>   
> 
> - In global scope (outside any function), `var` attaches directly as a property on the global object (`window.x` in browser, `global.x` in Node.js non-module scripts).
>     
>       
>     
> - Top-level `let` and `const` live in the Declarative Environment Record and **never** attach to the global object.
>     
>       
>     

## Common Interview Questions

- "What are the three main differences between `var`, `let`, and `const`?"
    
      
    
- "Why does `console.log(a)` print `undefined` for `var a = 10`, but throws a `ReferenceError` for `let a = 10`?"
    
      
    
- "Does `const` make objects truly immutable? How do you make an object immutable in JavaScript?"
    
      
    
- "Explain the classic `for (var i = 0; i < 3; i++) setTimeout` bug and how `let` fixes it under the hood."
    
      
    
- "What happens if you re-declare a `let` variable in a child block vs. the same block?"
    
      
    
- "Why is global `var` considered bad practice in terms of the `window` object?"
    
      
    

## Strong Answers / Talking Points

### 1. Scope: Function Scope vs. Block Scope

- **`var` Scope Leakage**: Because `var` is function-scoped, loops and conditional blocks leak variables into the surrounding function or global environment, often causing unintentional variable shadowing or overwriting.
    
      
    
- **`let` / `const` Isolation**: Variables declared inside `if (true) { ... }` or `for (...) { ... }` cannot be referenced outside those braces, preserving encapsulation and reducing side effects.
    
      
    

### 2. The Mechanics of Hoisting & TDZ

- All three variable types are hoisted (registered in memory during Phase 1 of the Execution Context).
    
      
    
- **The Core Difference**:
    
      
    - `var` is automatically initialized with `undefined`.
        
          
        
    - `let` and `const` remain uninitialized. The engine actively checks access and throws `ReferenceError: Cannot access 'x' before initialization` until execution reaches the declaration line.
        
          
        

### 3. Mutability: Reassignment vs. Mutation (`const`)

- `const` enforces **immutable reference bindings**, not immutable value contents.
    
      
    
- Primitive types (`string`, `number`, `boolean`) stored in `const` cannot change because they are values.
    
      
    
- Complex types (`Object`, `Array`) stored in `const` hold a memory pointer/reference. The reference itself cannot be reassigned to another object or array, but internal properties/elements can be modified freely.
    
      
    
- To achieve shallow immutability, use `Object.freeze(obj)`. For deep immutability, use recursive freezing or libraries like Immutable.js/Immer.
    
      
    

### 4. The Loop Trap: `var` vs. `let` in Closures

- With `var i = 0`: A single variable `i` is shared across all loop iterations. When asynchronous callbacks (like `setTimeout`) run, they all read that single final value of `i`.
    
      
    
- With `let i = 0`: The engine creates a brand-new lexical scope and variable binding for **every single iteration**, preserving the specific value captured in each iteration's closure.
    
      
    

## Code Snippets / Examples

### Scope & Global Object Attachment

JavaScript

```
// Function vs. Block Scope
if (true) {
  var leakedVar = "I escaped the block";
  let scopedLet = "I am trapped inside";
  const scopedConst = "I am also trapped";
}
console.log(leakedVar);    // "I escaped the block"
// console.log(scopedLet); // ReferenceError: scopedLet is not defined

// Global Object Pollution
var globalVar = "Attached";
let globalLet = "Detached";

console.log(window.globalVar); // "Attached" (in browser)
console.log(window.globalLet); // undefined
```

### The Loop & Closure Behavior

JavaScript

```
// Problem with var: shared single binding
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(`var: ${i}`), 100);
}
// Output: var: 3, var: 3, var: 3

// Fixed with let: per-iteration lexical binding
for (let j = 0; j < 3; j++) {
  setTimeout(() => console.log(`let: ${j}`), 100);
}
// Output: let: 0, let: 1, let: 2
```

### `const` Reassignment vs. Mutation

JavaScript

```
const user = { name: "Alice" };

// Valid: mutating an interior property
user.name = "Bob";
console.log(user.name); // "Bob"

// Invalid: reassignment of the binding
// user = { name: "Charlie" }; // TypeError: Assignment to constant variable.

// Enforcing shallow immutability
const frozenUser = Object.freeze({ name: "Alice" });
// frozenUser.name = "Bob"; // Fails silently, or throws TypeError in strict mode
```

## Comparison Matrix

|**Feature**|**var**|**let**|**const**|
|---|---|---|---|
|**Scope**|Function scope|Block scope|Block scope|
|**Hoisting**|Yes (initialized to `undefined`)|Yes (uninitialized / TDZ)|Yes (uninitialized / TDZ)|
|**Re-declaration**|Allowed|Forbidden in same scope|Forbidden in same scope|
|**Reassignment**|Allowed|Allowed|Forbidden|
|**Initial Value Required**|No (defaults to `undefined`)|No (defaults to `undefined`)|Yes|
|**Attaches to `window`**|Yes (in global scope)|No|No|

## Related Topics

- [[JavaScript Execution Context, Memory Creation & Hoisting Mechanics]]
    
      
    
- [[JavaScript Closures and Scope Chains]]
    
      
    
- [[JavaScript Memory Management and Object Mutability]]
    
      
    
- [[The this Keyword and Execution Bindings]]
    
      
    

## Tags

#fullstack #interview #javascript #variables #scopes #es6

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups