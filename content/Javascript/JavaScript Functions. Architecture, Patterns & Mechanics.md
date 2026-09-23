# JavaScript Functions: Architecture, Patterns & Mechanics

## Key Concepts

> [!summary] First-Class Citizens / First-Class Functions
>
> In JavaScript, functions are **first-class citizens** (first-class objects). They can be stored in variables, passed as arguments to other functions (callbacks), returned from functions (higher-order functions), and assigned dynamic properties.


> [!abstract] Function Declaration vs. Function Expression
>
> - **Function Declaration (Statement)**: Defined using `function name() {}`. Parsed during Phase 1 (Memory Creation Phase) and **hoisted with its full body**, making it callable before its definition.
>
> - **Function Expression**: A function assigned to a variable (`const fn = function() {}`). Only the variable declaration is hoisted; invoking it before the assignment statement throws a `TypeError` (`var`) or `ReferenceError` (`let`/`const`).
>
> - **Named Function Expression (NFE)**: An expression with an internal name (`const fn = function myName() {}`). The internal identifier is accessible **only inside its own function body**, which is useful for self-recursion and clean stack traces.


> [!info] Arrow Functions vs. Regular Functions
>
> Arrow functions (`() => {}`) provide concise syntax, but introduce four fundamental behavioral differences:
>
> 1. **No own `this`**: They resolve `this` lexically from the enclosing execution context.
>
> 2. **No `arguments` object**: Must use rest parameters (`...args`) instead.
>
> 3. **Cannot be used as constructors**: Lack internal `[[Construct]]` slot; invoking with `new` throws a `TypeError`.
>
> 4. **No `prototype` property**: Cannot participate in prototype-based inheritance as constructor blueprints.


> [!tip] Parameters vs. Arguments & Variadic Handling
>
> - **Parameters**: Identifiers declared in the function's definition signature.
>
> - **Arguments**: Actual values passed to the function when it is invoked.
>
> - **Default Parameters**: Assigned using `=`; evaluate only when the passed argument is strictly `undefined`.
>
> - **Rest Parameters (`...args`)**: Collects remaining arguments into a true `Array` instance.
>
> - **`arguments` Object**: Legacy, array-like object available only in non-arrow functions.


> [!danger] Constructor Functions vs. Factory Functions
>
> - **Constructor Function**: Invoked with `new`. Allocates a fresh object inheriting from the constructor's `.prototype`, binds `this` to that instance, and returns it implicitly.
>
> - **Factory Function**: Any function that constructs and returns a **brand-new object instance every time it is invoked** without requiring the `new` keyword.


## Common Interview Questions

- "What does it mean that functions are 'first-class citizens' in JavaScript?"

- "Compare Function Declarations, Anonymous Function Expressions, and Named Function Expressions in terms of hoisting and stack traces."

- "What are the four architectural differences between an arrow function and a regular function?"

- "Explain the difference between `arguments` and rest parameters (`...args`)."

- "What is an IIFE (Immediately Invoked Function Expression), and what problem did it solve before ES6?"

- "What is function composition, and how do you implement a `pipe` or `compose` utility from scratch?"

- "What is a Factory Function, and why would you use it instead of a Constructor Function or ES6 `class`?"

## Strong Answers / Talking Points

### 1. Function Statement vs. Expression vs. Named Function Expression (NFE)

- **Declaration / Statement**: `function calculateTotal() {}`. Hoisted in full; can live anywhere in the file.

- **Anonymous Expression**: `const handler = function() {}`. Anonymous expressions historically caused `(anonymous function)` entries in call stacks, making profiling and production error logs harder to debug.

- **Named Function Expression (NFE)**: `const factorial = function fact(n) { return n <= 1 ? 1 : n * fact(n - 1); };`.

    - The identifier `fact` is scoped exclusively to the function itself.

    - Improves debuggability in call stacks.

    - Prevents tight coupling to external variable names during recursion.

### 2. The Mechanics of IIFEs (Immediately Invoked Function Expressions)

- **Syntax**: `(function() { /* code */ })();` or `(() => { /* code */ })();`.

- **Primary Use Case**: Before ES6 block-scoping (`let`/`const`) and native ESM modules, any variable declared with `var` at the top level leaked onto `window`. IIFEs wrapped variables in an isolated function execution context, creating privacy and preventing global namespace pollution (Module Pattern).

### 3. Constructor Functions vs. Factory Functions

- **Constructor Functions**:

    - Capitalized convention: `function User(name) { this.name = name; }`.

    - Calling with `new`:

        1. Creates a blank object `{}`.

        2. Sets its `[[Prototype]]` link to `User.prototype`.

        3. Executes `User` with `this` bound to the new object.

        4. Returns `this` (unless the function explicitly returns another non-primitive object).

- **Factory Functions**:

    - `function createUser(name) { return { name, login() {} }; }`.

    - Avoids `this` context binding issues (no bugs if someone forgets `new`).

    - Simplifies composition over inheritance and enforces true data privacy via closures.

### 4. Function Composition (`compose` vs. `pipe`)

- Combines multiple pure, unary (single-argument) functions so the output of one function becomes the input of the next: $f(g(x))$.

- **`compose`**: Evaluates functions from **right to left** (mathematical standard: `compose(f, g)(x)` $\to$ `f(g(x))`).

- **`pipe`**: Evaluates functions from **left to right** (data-pipeline standard: `pipe(f, g)(x)` $\to$ `g(f(x))`).

## Code Snippets / Examples

### Declarations, Expressions & NFE

```javascript
// 1. Function Declaration (Hoisted completely)
hoistedFn(); // Output: "Callable before definition"
function hoistedFn() {
  console.log("Callable before definition");
}

// 2. Anonymous Function Expression
// unhoistedFn(); // TypeError: unhoistedFn is not a function (if declared with var)
const unhoistedFn = function() {
  console.log("Executed");
};

// 3. Named Function Expression (NFE)
const compute = function calculateFibonacci(n) {
  if (n <= 1) return n;
  // 'calculateFibonacci' is accessible internally for recursion
  return calculateFibonacci(n - 1) + calculateFibonacci(n - 2);
};
// calculateFibonacci(5); // ReferenceError: calculateFibonacci is not defined outside
```

### Arrow Function Differences

```javascript
const counter = {
  count: 0,
  // Regular method: 'this' binds to counter when invoked as counter.increment()
  incrementRegular() {
    setTimeout(function() {
      // In sloppy mode, this === window; this.count evaluates to undefined/NaN
      console.log("Regular this:", this.count);
    }, 10);
  },
  // Arrow function retains 'this' lexically from surrounding incrementArrow scope
  incrementArrow() {
    setTimeout(() => {
      console.log("Arrow this:", this.count); // Output: 0
    }, 10);
  }
};

// Constructor restriction:
const ArrowConstructor = () => {};
// new ArrowConstructor(); // TypeError: ArrowConstructor is not a constructor
```

### Rest Parameters vs. `arguments`

```javascript
// Legacy arguments object (array-like, missing array methods)
function legacySum() {
  // arguments lacks .reduce, must convert: Array.from(arguments)
  return Array.prototype.reduce.call(arguments, (acc, val) => acc + val, 0);
}

// Modern Rest Parameters (real Array instance)
function modernSum(multiplier, ...numbers) {
  // 'numbers' is a true Array
  return numbers.reduce((acc, val) => acc + val * multiplier, 0);
}

console.log(modernSum(2, 1, 2, 3)); // (1*2) + (2*2) + (3*2) = 12
```

### Factory Function vs. Constructor Function

```javascript
// 1. Constructor Function (Requires 'new')
function PersonConstructor(name) {
  this.name = name;
}
PersonConstructor.prototype.greet = function() {
  return `Hi, I am ${this.name}`;
};
const personA = new PersonConstructor("Alice");

// 2. Factory Function (Returns new object every time; no 'new' required)
function createPersonFactory(name) {
  // Encapsulated private state via closure
  const createdAt = Date.now();

  return {
    name,
    greet() {
      return `Hi, I am ${name}`;
    },
    getMetadata() {
      return { createdAt };
    }
  };
}
const personB = createPersonFactory("Bob");
```

### Implementing `pipe` and `compose`

```javascript
// Functional Composition Implementations using Array.prototype.reduce

// Pipe: Left-to-Right execution
const pipe = (...fns) => (initialValue) =>
  fns.reduce((acc, fn) => fn(acc), initialValue);

// Compose: Right-to-Left execution
const compose = (...fns) => (initialValue) =>
  fns.reduceRight((acc, fn) => fn(acc), initialValue);

// Usage
const trim = (str) => str.trim();
const toLowerCase = (str) => str.toLowerCase();
const wrapInTag = (tag) => (str) => `<${tag}>${str}</${tag}>`;

const formatPipeline = pipe(
  trim,
  toLowerCase,
  wrapInTag("span")
);

console.log(formatPipeline("   HELLO WORLD  ")); // "<span>hello world</span>"
```

## Comparison Matrix: Arrow vs. Regular Function

|**Feature**|**Regular Function**|**Arrow Function**|
|---|---|---|
|**`this` Binding**|Dynamic (determined by call site)|**Lexical** (inherited from enclosing scope)|
|**Usable as Constructor (`new`)**|Yes|**No** (throws `TypeError`)|
|**`arguments` Object**|Present (array-like)|**Absent** (must use `...rest`)|
|**Has `.prototype` Property**|Yes|**No**|
|**Duplicate Parameter Names**|Allowed in sloppy mode|**Forbidden** (throws `SyntaxError`)|
|**Methods in Objects**|Preferred for object methods|Can lead to broken `this` in object methods|

## Related Topics

- [[JavaScript Execution Context, Memory Creation & Hoisting Mechanics]]

- [[The `this` Keyword & Execution Bindings|The this Keyword and Execution Bindings]]

- [[JavaScript Closures. Encapsulation, Currying & Output Puzzles|JavaScript Closures and Scope Chains]]

- [[JavaScript Data Types, Objects & Prototypal Inheritance]]

## Tags

#fullstack #interview #javascript #functions #arrow-functions #iife #functional-programming #factory-pattern

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups
