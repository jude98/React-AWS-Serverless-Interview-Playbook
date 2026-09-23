# Prototypal Inheritance, Object Prototypes & Constructor Functions

## Key Concepts

> [!summary] Prototypal Inheritance Defined
>
> JavaScript uses **delegation-based prototypal inheritance**, not classical class-based inheritance. Objects directly inherit properties and methods from other objects via an internal link known as `[[Prototype]]`. When a property or method is accessed on an object, the engine searches the object itself first, then walks up the **Prototype Chain** until it finds the property or reaches the end (`null`).


> [!abstract] Function `.prototype` vs. Object `[[Prototype]]` (`__proto__`)
>
> - **`Function.prototype`**: A property that exists on function objects (specifically constructor functions). It acts as the blueprint object that will be assigned as the `[[Prototype]]` of any new instance instantiated via `new Constructor()`.
>
> - **`[[Prototype]]` (or `__proto__`)**: The internal pointer present on **every object instance** pointing to its parent prototype object.
>
> - Relationship: `new Person().__proto__ === Person.prototype` (or standard `Object.getPrototypeOf(new Person()) === Person.prototype`).


> [!info] The `new` Operator Lifecycle
>
> When invoking a constructor function with `new Constructor(args)`:
>
> 1. A brand new plain JavaScript object `{}` is allocated in memory.
>
> 2. The internal `[[Prototype]]` of this new object is set to `Constructor.prototype`.
>
> 3. The constructor function is executed with its `this` context bound to the newly created object.
>
> 4. If the constructor returns a non-primitive object explicitly, that object is returned; otherwise, the newly created object (`this`) is returned.


> [!tip] Static Methods in Prototype Architecture
>
> - **Instance Methods**: Attached to `Constructor.prototype` (`Person.prototype.walk`). Shared across all instances via prototype delegation to save memory.
>
> - **Static Methods**: Attached directly to the constructor function object itself (`Person.isPerson = function() {}`). They are called directly on the constructor (`Person.isPerson()`) and are **not** present on instances (`new Person().isPerson` is `undefined`).


## Common Interview Questions

- "What is the difference between `prototype` and `__proto__`?"

- "What four steps happen behind the scenes when the `new` keyword is executed?"

- "How do you implement inheritance using pure ES5 constructor functions and `Object.create()`?"

- "Why should methods be added to `Constructor.prototype` instead of inside `this.method = function()` inside the constructor?"

- "What are static methods in JavaScript, how were they created before ES6 `class`, and can instances access them?"

- "What does `Object.create(proto)` do under the hood, and how does it differ from `new`?"

- "What happens if a constructor function explicitly returns a primitive vs. an object?"

## Strong Answers / Talking Points

### 1. Memory Efficiency: Prototype Methods vs. In-Constructor Methods

- Defining methods inside a constructor (`this.greet = function() { ... }`):

    - Every single instance allocates a completely new closure/function object in heap memory. 10,000 instances create 10,000 separate function instances.

- Defining methods on `Constructor.prototype` (`Constructor.prototype.greet = function() { ... }`):

    - Only **one** copy of the function exists in memory on the prototype object. All 10,000 instances delegate lookups to that single shared reference via the prototype chain.

### 2. Prototypal Inheritance Implementation (ES5 Pattern)

- To make a child constructor inherit from a parent constructor:

    1. **Call Parent Constructor for Own Properties**: Use `Parent.call(this, args)` inside `Child` to initialize instance properties on the newly created `this`.

    2. **Link the Prototypes**: Use `Child.prototype = Object.create(Parent.prototype)` so child instances delegate method lookups to parent prototype.

    3. **Reset the Constructor Pointer**: `Object.create` overwrites `Child.prototype.constructor`. Always manually reset `Child.prototype.constructor = Child`, or instance checks like `instance.constructor` will erroneously point to `Parent`.

### 3. Static Methods: Placement and Inheritance

- Static methods are utility functions that belong to the namespace of the constructor/class (e.g., `Array.isArray()`, `Object.keys()`).

- In ES5: Defined by assigning directly to the constructor: `Child.myStaticMethod = function() {}`.

- In ES6 `class Child extends Parent`: Static methods are also inherited because the engine sets `Object.setPrototypeOf(Child, Parent)`. In manual ES5 prototypes, static methods on `Parent` must be manually copied or linked via `Object.setPrototypeOf(Child, Parent)`.

### 4. Constructor Return Values

- If a constructor returns a primitive (`return 42;`, `return "string";`, `return true;`), the engine **ignores** the return statement and returns the freshly constructed `this` instance.

- If a constructor returns a complex object (`return { custom: true };`), the engine **discards** `this` and returns that custom object instead (breaking the instance prototype link to `Constructor.prototype`).

## Code Snippets / Examples

### Hand-Rolling Prototypal Inheritance (ES5 Mechanics)

```javascript
// 1. Parent Constructor
function Animal(name) {
  this.name = name;
}

// Parent Prototype Method (Shared across all instances)
Animal.prototype.eat = function() {
  return `${this.name} is eating.`;
};

// Parent Static Method (Called on constructor directly)
Animal.compareSize = function(a, b) {
  return a.name.length - b.name.length;
};

// 2. Child Constructor
function Dog(name, breed) {
  // Step A: Inherit own instance properties via explicit binding
  Animal.call(this, name);
  this.breed = breed;
}

// Step B: Set up prototype delegation chain (Dog -> Animal -> Object -> null)
Dog.prototype = Object.create(Animal.prototype);

// Step C: Repair constructor pointer
Dog.prototype.constructor = Dog;

// Step D: Add child-specific prototype methods
Dog.prototype.bark = function() {
  return `${this.name} barks!`;
};

// Step E: Inherit static methods (linking constructor prototypes)
Object.setPrototypeOf(Dog, Animal);

// 3. Testing the Chain
const rover = new Dog("Rover", "Golden Retriever");

console.log(rover.bark());        // "Rover barks!" (from Dog.prototype)
console.log(rover.eat());         // "Rover is eating." (delegated to Animal.prototype)
console.log(rover instanceof Dog);    // true
console.log(rover instanceof Animal); // true
console.log(Dog.compareSize({name: "A"}, {name: "BB"})); // -1 (inherited static method)
```

### Polyfilling the `new` Keyword

```javascript
// Implementing custom 'new' operator to demonstrate underlying engine behavior
function customNew(Constructor, ...args) {
  // Step 1: Create a fresh object whose prototype links to Constructor.prototype
  const newInstance = Object.create(Constructor.prototype);

  // Step 2: Execute Constructor with 'this' bound to the new instance
  const result = Constructor.apply(newInstance, args);

  // Step 3: Return the result if it's an object, otherwise return newInstance
  return (typeof result === "object" && result !== null) || typeof result === "function"
    ? result
    : newInstance;
}

// Usage test
function Person(name) {
  this.name = name;
}
Person.prototype.greet = function() {
  return `Hi, I'm ${this.name}`;
};

const user = customNew(Person, "Sarah");
console.log(user.greet()); // "Hi, I'm Sarah"
console.log(user instanceof Person); // true
```

### Constructor Return Value Behavior

```javascript
function IgnoredPrimitive() {
  this.value = "Original Instance";
  return 100; // Primitive return is ignored by the engine
}

function OverriddenObject() {
  this.value = "Original Instance";
  return { value: "Hijacked Object" }; // Object return overrides 'this'
}

console.log(new IgnoredPrimitive().value); // "Original Instance"
console.log(new OverriddenObject().value);   // "Hijacked Object"
```

## Comparison Matrix: ES5 Prototypes vs. ES6 Classes

|**Feature**|**ES5 Constructor Functions**|**ES6 class Syntax**|
|---|---|---|
|**Under the Hood**|Prototypal delegation|Prototypal delegation (Syntactic sugar)|
|**Invocation without `new`**|Runs as regular function (pollutes global/`undefined`)|Throws `TypeError: Class constructor cannot be invoked without 'new'`|
|**Hoisting**|Hoisted with body (Function Declaration)|Hoisted into **Temporal Dead Zone (TDZ)**|
|**Execution Mode**|Sloppy by default (unless `"use strict"`)|**Strict mode** automatically enabled in entire body|
|**Method Enumerability**|Prototype methods are enumerable by default|Methods defined in class body are **non-enumerable**|
|**Parent Invocation**|`Parent.call(this, ...)`|`super(...)` (must be called before accessing `this`)|

## Related Topics

- [[JavaScript Data Types, Objects & Prototypal Inheritance]]

- [[JavaScript Functions. Architecture, Patterns & Mechanics|JavaScript Functions: Architecture, Patterns & Mechanics]]

- [[The `this` Keyword & Execution Bindings|The this Keyword and Execution Bindings]]

- [[Object-Oriented Programming (OOP) in JavaScript & TypeScript|ES6 Classes: Private Fields, Inheritance, and Method Overriding]]

## Tags

#fullstack #interview #javascript #prototype #prototypal-inheritance #constructor-functions #new-operator

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups
