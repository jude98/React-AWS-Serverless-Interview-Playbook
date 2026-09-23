# JavaScript Scope, Lexical Environment, and Shadowing

## Key Concepts

> [!summary] Scope & Lexical Scoping
> 
> **Scope** dictates the accessibility and visibility of variables and functions at different parts of your code during runtime. JavaScript uses **Lexical Scoping** (Static Scoping), meaning a function's scope is determined by where it is physically written in the source code, _not_ where it is called.
> 
>   

> [!abstract] The Lexical Environment
> 
> The internal engine construct created during the Execution Context's memory phase. It consists of two parts:
> 
>   
> 
> 1. **Environment Record**: The actual memory space storing local variable/function declarations (`var`, `let`, `const`, `function`).
>     
>       
>     
> 2. **Outer Lexical Reference**: A pointer to the Lexical Environment of its parent (the scope where the function was lexically declared).
>     
>       
>     

> [!info] The Scope Chain
> 
> The mechanism the JavaScript engine uses to resolve variable values. If an identifier isn't found in the local Environment Record, the engine follows the Outer Lexical Reference to search the parent scope, continuing upwards until it hits the Global Lexical Environment. If it's still not found, it throws a `ReferenceError`.
> 
>   

> [!danger] Shadowing and Illegal Shadowing
> 
> **Shadowing** occurs when a variable declared in an inner scope shares the same name as a variable in an outer scope, overriding it within that inner block. **Illegal Shadowing** happens when an inner `var` attempts to shadow an outer `let` or `const` across a block boundary, causing a `SyntaxError` because `var` ignores blocks and attempts to illegally redeclare the existing block-scoped variable.
> 
>   

## Common Interview Questions

- "What is a Lexical Environment, and how does it differ from the Execution Context?"
    
      
    
- "Explain the Scope Chain. How does the engine resolve a variable that isn't defined locally?"
    
      
    
- "What does it mean that JavaScript is 'lexically scoped'?"
    
      
    
- "What is variable shadowing? Can you give an example?"
    
      
    
- "What is illegal shadowing, and why does `let a = 1; { var a = 2; }` throw a `SyntaxError`?"
    
      
    

## Strong Answers / Talking Points

### 1. Lexical Environment vs. Execution Context

- An **Execution Context (EC)** is the overall wrapper managing the execution of code.
    
      
    
- The **Lexical Environment** is a _component_ of the EC. Whenever a new EC is created (like when a function is invoked), a new Lexical Environment is instantiated to hold its specific local memory bindings and the reference to its physical parent in the code.
    
      
    

### 2. Lexical Scoping (Static Scoping) Mechanics

- "Lexical" relates to the compilation phase. The scope chain is defined before the code even executes.
    
      
    
- Even if `Function B` is passed as a callback and executed inside `Function C`, `Function B` will only have access to its local variables, the global variables, and the variables of `Function A` (where it was originally authored). It will _not_ have access to `Function C`'s variables. This forms the fundamental basis of **Closures**.
    
      
    

### 3. Resolving Variables via the Scope Chain

- **Step 1**: Engine looks in the current Environment Record. If found, use it.
    
      
    
- **Step 2**: If not found, follow the Outer Lexical Reference to the parent's Environment Record.
    
      
    
- **Step 3**: Repeat until reaching the Global Execution Context (whose outer reference is `null`).
    
      
    
- **Step 4**: If it reaches `null` and the variable is still unresolved, throw a `ReferenceError` (in strict mode).
    
      
    

### 4. Shadowing & Illegal Shadowing

- **Legal Shadowing**: You can shadow a `var` with a `let`, a `let` with a `let`, or a `var` with a `var`. The inner variable safely masks the outer one.
    
      
    
- **Illegal Shadowing**: You **cannot** shadow an outer `let` or `const` with an inner `var` inside a block.
    
      
    - _Why?_ Because `var` is function-scoped. It ignores the block `{}` and tries to attach itself to the nearest function or global environment. In doing so, it collides with the `let` variable already residing in that same lexical space, violating the rule that `let` cannot be redeclared.
        
          
        

## Code Snippets / Examples

### The Scope Chain & Lexical Environment

```javascript
const globalVar = "Global";

function outer() {
  const outerVar = "Outer";

  function inner() {
    const innerVar = "Inner";
    // Scope Chain Lookup: innerVar (local) -> outerVar (parent) -> globalVar (grandparent)
    console.log(`${innerVar} scopes to ${outerVar} scopes to ${globalVar}`);
  }
  
  inner();
}

outer(); 
```

### Valid Variable Shadowing

```javascript
let count = 10;
var score = 100;

if (true) {
  let count = 20; // Valid: inner 'let' shadows outer 'let'
  let score = 200; // Valid: inner 'let' shadows outer 'var'
  console.log(count, score); // 20, 200
}

console.log(count, score); // 10, 100 (outer scope remains unaffected)
```

### Illegal Shadowing

```javascript
let name = "Alice";

if (true) {
  // SyntaxError: Identifier 'name' has already been declared
  // var tries to escape the block and redefine the outer 'let'
  var name = "Bob"; 
}

// NOTE: The reverse IS completely legal because 'let' stays safely in the block:
var age = 25;
if (true) {
  let age = 30; // Perfectly valid
}
```

## Related Topics

- [[JavaScript Execution Context, Memory Creation & Hoisting Mechanics]]
    
      
    
- [[JavaScript Closures. Encapsulation, Currying & Output Puzzles|JavaScript Closures and Scope Chains]]
    
      
    
- [[JavaScript Variables, Scopes, and Execution Context|JavaScript Variable Declarations: var, let, and const]]
    
      
    
- [[The `this` Keyword & Execution Bindings|The this Keyword and Execution Bindings]]
    
      
    

## Tags

#fullstack #interview #javascript #scope #lexical-environment #shadowing

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups