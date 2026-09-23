# JavaScript Data Types, Objects & Prototypal Inheritance

## Key Concepts

> [!summary] Primitives vs. Structural Types (Objects)
> 
> JavaScript has **8 data types**:
> 
>   
> 
> - **7 Primitives**: `string`, `number`, `bigint`, `boolean`, `undefined`, `symbol`, and `null`. They are immutable and passed by value.
>     
>       
>     
> - **1 Structural / Reference Type**: `object` (which includes plain objects, arrays, functions, dates, regex, maps, and sets). Objects are mutable and passed by reference.
>     
>       
>     

> [!abstract] "Everything in JavaScript is an Object" — Myth vs. Reality
> 
> Primitives are **not** objects. However, when you access a method or property on a primitive (e.g., `'hello'.toUpperCase()`), the engine performs **autoboxing** (or primitive wrapping). It temporarily wraps the primitive in its corresponding wrapper object (`String`, `Number`, `Boolean`), executes the method, and immediately discards the wrapper object for garbage collection.
> 
>   

> [!info] The `typeof` Operator & Its Quirks
> 
>   
> 
> - `typeof null === 'object'` is a historical bug in JavaScript's original C implementation (type tag for references was `000`, and `null` was represented as the NULL pointer `0x00`).
>     
>       
>     
> - `typeof function() {} === 'function'` is an intentional exception for callable objects.
>     
>       
>     
> - `typeof NaN === 'number'`, even though `NaN` stands for "Not-a-Number".
>     
>       
>     

> [!tip] Prototype & Prototypal Inheritance
> 
> Every JavaScript object contains an internal slot called `[[Prototype]]` (accessible via `Object.getPrototypeOf(obj)` or legacy `__proto__`). If a property is not found on the object itself, the engine traverses up the **Prototype Chain** until it reaches `Object.prototype.`[[Prototype]]`, which terminates at `null`.
> 
>   

## Common Interview Questions

- "What are the 8 data types in JavaScript, and how do primitives differ from reference types?"

- "Why does `typeof null` return `'object'`, and how do you reliably check for `null`?"

- "Is JavaScript truly an object-oriented language if everything is supposedly an object?"

- "Explain autoboxing / primitive wrapper objects. What happens under the hood when calling `'abc'.slice(1)`?"

- "What is the difference between `__proto__` and `prototype`?"

- "How does the Prototype Chain work, and what sits at the very top of it?"

- "What is the difference between `Object.create(proto)` and classical constructor functions / ES6 classes?"

## Strong Answers / Talking Points

### 1. Primitives vs. Reference Types

- **Memory Allocation**:

    - Primitives are stored directly in memory (often on the execution stack or directly within the context's frame).

    - Objects are stored in the memory **Heap**. The variable holds a memory address (pointer) referencing the object.

- **Copy Semantics**:

    - Copying a primitive duplicates the actual value.

    - Copying an object duplicates the memory pointer—mutating properties on one affects all other references.

### 2. Autoboxing (Primitive Wrappers)

- When evaluating `'text'.length`:

    1. The engine detects property access on a primitive `string`.

    2. It wraps it in `new String('text')`.

    3. It retrieves `.length` (defined on `String.prototype`).

    4. It destroys/discards the temporary instance.

- _Trade-off_: Avoid manual instantiation (`new String('text')` or `new Boolean(false)`). It produces truthy objects instead of raw primitives, introducing subtle bugs (e.g., `if (new Boolean(false))` evaluates to `true`).

### 3. `__proto__` vs. `prototype`

- `prototype`: A property that exists **only on functions** (constructors). It specifies what will become the `[[Prototype]]` of any instance created with `new Foo()`.

- `[[Prototype]]` (or `__proto__`): The actual link present on **every object instance** pointing to the prototype object it inherits methods and properties from.

### 4. Prototypal Inheritance vs. Classical Inheritance

- Classical inheritance (Java/C++) copies class blueprints at instantiation.

- JavaScript inheritance is purely **delegation-based**: instances do not duplicate parent methods; they simply hold a live link up the chain.

- ES6 `class` syntax is syntactic sugar over prototypal inheritance and does not introduce a classical OOP object model under the hood.

## Code Snippets / Examples

### `typeof` Quirks & Robust Type Checking

```javascript
// Quirks
console.log(typeof null);        // "object" (historical bug)
console.log(typeof NaN);         // "number"
console.log(typeof function(){});// "function"
console.log(typeof []);          // "object"

// Robust Type Checking Alternatives
const isNull = (val) => val === null;
const isArray = (val) => Array.isArray(val);

// The canonical universal type-checking trick
const getType = (val) => Object.prototype.toString.call(val).slice(8, -1);
console.log(getType(null));      // "Null"
console.log(getType([]));        // "Array"
console.log(getType(/regex/));   // "RegExp"
```

### Autoboxing Demonstration

```javascript
const str = "hello";
str.customProp = 42;

// Behind the scenes:
// 1. Temporary object created: new String("hello").customProp = 42;
// 2. Temporary object immediately destroyed.

console.log(str.customProp); // undefined (accesses a new wrapper instance with no such property)
```

### Prototype Chain & Prototypal Delegation

```javascript
const animal = {
  eats: true,
  walk() {
    return "Moving...";
  }
};

// Create dog inheriting from animal via prototype delegation
const dog = Object.create(animal);
dog.barks = true;

console.log(dog.barks); // true (own property)
console.log(dog.eats);  // true (delegated up the chain to animal)
console.log(dog.walk());// "Moving..." (delegated)

// Prototype chain traversal:
// dog -> animal -> Object.prototype -> null
console.log(Object.getPrototypeOf(dog) === animal);              // true
console.log(Object.getPrototypeOf(animal) === Object.prototype); // true
console.log(Object.getPrototypeOf(Object.prototype));            // null
```

## Related Topics

- [[JavaScript Execution Context, Memory Creation & Hoisting Mechanics]]

- [[JavaScript Variables, Scopes, and Execution Context|JavaScript Variable Declarations: var, let, and const]]

- [[The `this` Keyword & Execution Bindings|The this Keyword and Execution Bindings]]

- [[JavaScript Data Types, Objects & Prototypal Inheritance|Object Immutability, Deep Clones, and Memory References]]

## Tags

#fullstack #interview #javascript #data-types #prototypes #inheritance #oop

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups