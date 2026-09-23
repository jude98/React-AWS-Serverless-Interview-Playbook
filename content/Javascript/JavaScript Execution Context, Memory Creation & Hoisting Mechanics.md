# JavaScript Execution Context, Memory Creation & Hoisting Mechanics

## Key Concepts

> [!summary] Execution Context & The Call Stack
>
> An Execution Context (EC) is the abstract environment where JavaScript code is evaluated and executed. The engine tracks these contexts using the Call Stack (LIFO: Last In, First Out). The engine begins with the Global Execution Context (GEC), and a new Function Execution Context (FEC) is created whenever a function is invoked.


> [!abstract] Two-Phase Execution Lifecycle
>
> Every Execution Context runs in two discrete phases:
>
> 1. **Phase 1: Creation Phase (Memory Allocation)**: The engine scans declarations, registers memory space for identifiers, initializes variables, and resolves lexical references.
>
> 2. **Phase 2: Code Execution Phase**: The engine executes code line-by-line, runs assignments, resolves operations, and handles function calls.


> [!info] Hoisting Mechanics
>
> Hoisting is the observable side effect of Phase 1:
>
> - **Function Declarations**: Stored entirely in memory with the function body attached, making them fully callable before declaration.
>
> - **`var` Declarations**: Registered and initialized immediately with `undefined`.
>
> - **`let` and `const` Declarations**: Registered in lexical memory, but left **uninitialized**.


> [!danger] Temporal Dead Zone (TDZ)
>
> The period between entering a block scope (Phase 1) and the point where the `let` or `const` variable's declaration statement is evaluated in Phase 2. Accessing the variable during this window throws a `ReferenceError`.


## Common Interview Questions

- "Walk me through what happens inside the JavaScript engine when a script is executed."

- "What are the two phases of an Execution Context, and what happens in each?"

- "What is hoisting under the hood? Does JavaScript physically move code to the top of the file?"

- "Why does accessing a `var` before declaration give `undefined`, but doing the same for `let` throws a `ReferenceError`?"

- "What is the Temporal Dead Zone (TDZ), and how does it prevent runtime bugs?"

- "How does the Call Stack interact with the Global Execution Context and Function Execution Contexts during recursion?"

## Strong Answers / Talking Points

### 1. The Anatomy of an Execution Context

An Execution Context contains three primary components:

1. **Lexical Environment**: Holds identifier-to-variable mappings for `let`, `const`, and block-level bindings, plus an outer lexical reference (enabling scope chain traversal).

2. **Variable Environment**: Holds declarations for `var` variables and traditional function declarations.

3. **`this` Binding**: Evaluated when the context is established based on the call-site and function type.

### 2. Deep Dive: Phase 1 (Creation Phase / Memory Allocation)

- When the engine enters a scope (GEC or FEC), before running line 1:

    - Allocates space in memory for all declared variables and functions.

    - Traditional functions (`function foo() {}`) are parsed and their pointers to the function body are saved immediately.

    - `var` variables are initialized to `undefined`.

    - `let` and `const` bindings are declared in the Lexical Environment Record without an initialization flag.

### 3. Deep Dive: Phase 2 (Code Execution Phase)

- Executes code line-by-line from top to bottom.

- When it reaches an assignment (`x = 5`), the allocated memory slot is updated with the actual value.

- When it reaches a `let`/`const` declaration line, that identifier is marked as initialized (exiting the TDZ).

- When a function is called, the engine pauses current execution, pushes a brand-new Function Execution Context onto the Call Stack, and runs Phase 1 & Phase 2 for that function.

- Once a function returns, its execution context is popped off the Call Stack.

### 4. Trade-Offs & Why TDZ Exists

- `var`'s early initialization to `undefined` masked runtime bugs: variables could be accessed silently before assignment, causing hard-to-trace bugs.

- TDZ enforces temporal correctness: using a variable before defining it becomes an explicit runtime error rather than silent bug propagation.

## Code Snippets / Examples

### Execution Phases & Hoisting in Practice

```javascript
console.log(varGreeting); // Output: undefined (hoisted + initialized to undefined)
// console.log(letGreeting); // Uncaught ReferenceError: Cannot access 'letGreeting' before initialization (TDZ)

var varGreeting = "Hello from var";
let letGreeting = "Hello from let"; // TDZ for letGreeting ends here

sayHello(); // Output: "Hello World" (Functions hoisted with complete body)

function sayHello() {
  console.log("Hello World");
}

// sayGoodbye(); // Uncaught TypeError: sayGoodbye is not a function
var sayGoodbye = function() {
  console.log("Goodbye");
};
```

### Call Stack Execution Flow

```javascript
function second() {
  console.log("Inside second");
}

function first() {
  second();
  console.log("Inside first");
}

first();

// Call Stack Progression:
// 1. [Global EC]
// 2. [Global EC, first() EC]
// 3. [Global EC, first() EC, second() EC]
// 4. second() finishes -> [Global EC, first() EC]
// 5. first() finishes  -> [Global EC]
```

## Related Topics

- [[JavaScript Closures. Encapsulation, Currying & Output Puzzles|JavaScript Closures and Scope Chains]]

- [[Strict Mode, the Global Object, and Runtime Environments|JavaScript Variables: var, let, const, and the Global Object]]

- [[Asynchronous JavaScript, Event Loop & Concurrency Model|JavaScript Event Loop and Concurrency Model]]

- [[JavaScript Garbage Collection. Reachability, Mark-and-Sweep & Generational Memory|Call Stack, Heap Memory, and Garbage Collection]]

## Tags

#fullstack #interview #javascript #execution-context #hoisting #tdz

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups
