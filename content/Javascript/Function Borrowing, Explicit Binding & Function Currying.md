
## Key Concepts

> [!summary] Explicit Binding & Function Borrowing
> 
> **Explicit Binding** forces a function to execute with a specific context object using `.call()`, `.apply()`, or `.bind()`. **Function Borrowing** is the pattern where an object utilizes an existing method from another object or built-in prototype (such as `Array.prototype` or `Object.prototype`) without copying or inheriting that method.
> 
>   

> [!abstract] `.call()` vs. `.apply()` vs. `.bind()`
> 
>   
> 
> - **`.call(thisArg, arg1, arg2, ...)`**: Invokes the function immediately with arguments passed individually as a comma-separated list.
>     
>       
>     
> - **`.apply(thisArg, [argsArray])`**: Invokes the function immediately with arguments passed as a single array or array-like object.
>     
>       
>     
> - **`.bind(thisArg, arg1, ...)`**: Does **not** invoke the function immediately. Instead, it returns a new bound function with `this` permanently set and optional preset leading arguments (**Partial Application**).
>     
>       
>     

> [!info] Function Currying Defined
> 
> Currying is a functional programming technique that transforms a function callable as $f(a, b, c)$ into a sequence of nested unary (single-argument) functions callable as $f(a)(b)(c)$. It enables high code reusability, function specialization, and declarative pipeline design.
> 
>   

> [!tip] Currying Mechanisms in JavaScript
> 
>   
> 
> 1. **Currying via `.bind()`**: Pre-sets leading arguments by passing them during function binding (partial application), ignoring the `thisArg` (passing `null` or `{}`).
>     
>       
>     
> 2. **Currying via Closures**: Nesting return functions that close over previous arguments in memory until all required parameters are collected.
>     
>       
>     

## Common Interview Questions

- "What is the difference between `.call()`, `.apply()`, and `.bind()`?"
    
      
    
- "What is function borrowing, and why would you borrow methods from `Array.prototype` or `Object.prototype`?"
    
      
    
- "How does function currying work, and how can you implement currying using `.bind()` vs. closures?"
    
      
    
- "Write an infinite currying function: `add(1)(2)(3)...()` or `add(1)(2)(3) == 6`."
    
      
    
- "What is the difference between Partial Application and Currying?"
    
      
    
- "How would you write a custom polyfill for `Function.prototype.bind` from scratch?"
    
      
    

## Strong Answers / Talking Points

### 1. Function Borrowing in Practice

- **Core Motivation**: Avoid code duplication. If another object or prototype already has the exact method logic needed, borrow it via `.call()` or `.apply()`.
    
      
    
- **Classic Use Case (Array-like Objects)**: Prior to ES6 `Array.from()`, `arguments` objects and DOM `NodeList` collections lacked array methods (`map`, `slice`, `filter`). Developers borrowed array methods: `Array.prototype.slice.call(arguments)`.
    
      
    
- **Defensive Borrowing**: Calling `Object.prototype.hasOwnProperty.call(obj, 'prop')` instead of `obj.hasOwnProperty('prop')` defends against edge cases where `obj` was created via `Object.create(null)` (no prototype) or has its own property named `hasOwnProperty`.
    
      
    

### 2. Partial Application vs. Currying

- **Currying**: Always translates a function of arity $N$ into $N$ sequential functions of arity 1: $f(a, b, c) \to f(a)(b)(c)$.
    
      
    
- **Partial Application**: Fixes a subset of a function's arguments upfront, returning a function of lower arity (e.g., transforming a 3-argument function into a 2-argument or 1-argument function).
    
      
    
- `Function.prototype.bind` natively implements **partial application**, but is frequently used to emulate curried function behavior.
    
      
    

### 3. Currying via `.bind()` vs. Closures

- **Via `.bind()`**:
    
      
    - `const multiplyByTwo = multiply.bind(null, 2);`
        
          
        
    - Concise for standard binary functions; leverages the engine's built-in bound-function optimization.
        
          
        
- **Via Closures**:
    
      
    - `const multiply = a => b => a * b;`
        
          
        
    - Idiomatic modern ES6 arrow syntax; avoids allocating unnecessary bound function wrapper objects.
        
          
        

## Code Snippets / Examples

### 1. Explicit Binding & Method Borrowing

JavaScript

```
const person1 = {
  firstName: "Jane",
  lastName: "Doe",
  getFullName(greeting = "Hello") {
    return `${greeting}, ${this.firstName} ${this.lastName}`;
  }
};

const person2 = {
  firstName: "John",
  lastName: "Smith"
};

// 1. .call() - Comma-separated arguments, executes immediately
console.log(person1.getFullName.call(person2, "Welcome")); 
// "Welcome, John Smith"

// 2. .apply() - Array of arguments, executes immediately
console.log(person1.getFullName.apply(person2, ["Greetings"])); 
// "Greetings, John Smith"

// 3. .bind() - Returns a new function with preset this
const getJohnFullName = person1.getFullName.bind(person2);
console.log(getJohnFullName("Hi")); 
// "Hi, John Smith"

// Defensive Method Borrowing from Object.prototype
const bareObject = Object.create(null); // No prototype!
bareObject.id = 101;
// bareObject.hasOwnProperty("id"); // TypeError: bareObject.hasOwnProperty is not a function
console.log(Object.prototype.hasOwnProperty.call(bareObject, "id")); // true
```

### 2. Currying via `.bind()` (Partial Application)

JavaScript

```
function calculateDiscount(discountPercent, price) {
  return price - (price * discountPercent);
}

// Partially applying the first argument using .bind()
const tenPercentDiscount = calculateDiscount.bind(null, 0.10);
const twentyPercentDiscount = calculateDiscount.bind(null, 0.20);

console.log(tenPercentDiscount(100)); // 90
console.log(twentyPercentDiscount(100)); // 80
```

### 3. Modern Currying via Closures & Infinite Currying

JavaScript

```
// 1. Standard 3-argument curried pipeline (ES6 arrow syntax)
const buildUrl = (protocol) => (domain) => (path) =>
  `${protocol}://${domain}/${path}`;

const httpsUrl = buildUrl("https");
const mySiteUrl = httpsUrl("api.example.com");

console.log(mySiteUrl("v1/users")); // "https://api.example.com/v1/users"
console.log(mySiteUrl("v1/orders")); // "https://api.example.com/v1/orders"

// 2. Classic Interview Puzzle: Infinite Currying with an Empty Termination Call
function add(a) {
  return function(b) {
    if (b !== undefined) {
      return add(a + b);
    }
    return a;
  };
}

console.log(add(1)(2)(3)(4)()); // 10

// 3. Classic Interview Puzzle: Currying with Implicit Coercion (valueOf / toString)
function curriedSum(a) {
  let currentSum = a;

  function inner(b) {
    currentSum += b;
    return inner;
  }

  inner.valueOf = () => currentSum;
  inner.toString = () => currentSum;

  return inner;
}

console.log(+curriedSum(1)(2)(3)); // 6 (coerced via valueOf)
```

### 4. Polyfill: `Function.prototype.bind`

JavaScript

```
// Hand-rolling a basic Function.prototype.bind polyfill
Function.prototype.myBind = function(context, ...boundArgs) {
  const originalFunction = this;

  if (typeof originalFunction !== "function") {
    throw new TypeError("Function.prototype.myBind - what is trying to be bound is not callable");
  }

  return function(...invokedArgs) {
    // Merge bound arguments with arguments provided at invocation time
    return originalFunction.apply(context, [...boundArgs, ...invokedArgs]);
  };
};

function introduce(city, country) {
  return `${this.name} lives in ${city}, ${country}`;
}

const boundIntroduce = introduce.myBind({ name: "Alex" }, "Berlin");
console.log(boundIntroduce("Germany")); // "Alex lives in Berlin, Germany"
```

## Comparison Matrix: `.call()` vs. `.apply()` vs. `.bind()`

|**Method**|**Invocation Timing**|**Arguments Format**|**Returns**|**Mutates Original Function?**|
|---|---|---|---|---|
|**`Function.prototype.call`**|**Immediate**|Comma-separated list (`arg1, arg2`)|Result of function execution|No|
|**`Function.prototype.apply`**|**Immediate**|Array or Array-like (`[arg1, arg2]`)|Result of function execution|No|
|**`Function.prototype.bind`**|**Deferred** (Manual later call)|Comma-separated list (`arg1, arg2`)|**New bound function instance**|No|

## Related Topics

- [[The this Keyword and Execution Bindings]]
    
      
    
- [[JavaScript Closures and Scope Chains]]
    
      
    
- [[JavaScript Functions: Architecture, Patterns & Mechanics]]
    
      
    
- [[Functional Programming Patterns: Pure Functions, Immutability & Memoization]]
    
      
    

## Tags

#fullstack #interview #javascript #call-apply-bind #function-borrowing #currying #partial-application

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups