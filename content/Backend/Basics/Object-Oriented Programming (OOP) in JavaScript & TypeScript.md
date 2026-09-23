# Object-Oriented Programming (OOP) in JavaScript & TypeScript

## Key Concepts

- **Class vs. Object:** A **class** is a blueprint/template defining state (fields) and behavior (methods). An **object** is a concrete instance of a class allocated in memory with its own identity and state.

- **Prototypal Foundations:** Under the hood, JavaScript does **not** have classical class-based inheritance. ES6 `class` syntax is syntactic sugar over prototype chains (`[[Prototype]]` / `__proto__`).

- **The Four Pillars:**

    - **Encapsulation:** Bundling data and behavior into a single unit while restricting direct external access to internal state.

    - **Inheritance:** Enabling a child class to inherit fields and methods from a parent class to foster code reuse.

    - **Polymorphism:** The ability of different classes to respond to the same method call with distinct, specialized implementations (method overriding).

    - **Abstraction:** Hiding complex low-level implementation details and exposing only an intuitive, high-level interface.

- **`super` keyword:** In child constructors, `super(...args)` calls the parent constructor and initializes `this`. In methods, `super.methodName()` accesses overridden parent implementations.

- **Static vs. Instance:** `static` members belong directly to the class constructor function itself rather than to individual instantiated objects.

- **Abstract Class vs. Interface (TypeScript):** An abstract class provides partial implementation and constructor logic with enforced sub-classing; an interface is a pure type contract stripped entirely during compilation (zero runtime footprint).

## Common Interview Questions

- How does classical OOP in languages like Java or C++ differ from JavaScript's prototype-based OOP?

- What does the `super()` call actually do under the hood in a derived JavaScript class constructor, and why must it be called before accessing `this`?

- How is true encapsulation achieved in modern JavaScript versus TypeScript compile-time encapsulation?

- How do you implement polymorphism and abstraction in JavaScript without native `abstract` or `interface` keywords at runtime?

- What are the operational and structural differences between an `interface` and an `abstract class` in TypeScript?

- What is the purpose of the `static` keyword, and where do static properties live on the prototype chain?

## Strong Answers / Talking Points

### 1. The Four Pillars Implemented in JS / TS

#### A. Encapsulation

- **Old JS Workaround:** Closures inside factory functions or prefixing with underscores (`_privateField` — purely by convention, no real enforcement).

- **TypeScript (Compile-Time Only):** `private` and `protected` access modifiers. Prevents unauthorized compilation access, but the property remains completely public and accessible at JavaScript runtime.

- **Modern JavaScript (Runtime Enforced):** Hard private fields using the `#` hash prefix (ECMAScript 2022). Enforced at the V8 engine level via private names / internal slots (`[[PrivateElements]]`). Cannot be inspected via `Object.keys()` or accessed outside the class lexical scope.

#### B. Inheritance & The `super` Mechanism

- Derived classes extend parent classes via the `extends` keyword.

- **Why `super()` is required before `this`:**

    - In a base class, the object memory for `this` is initialized immediately by the engine when `new` is called.

    - In a derived class, the derived constructor does not create its own `this`. It relies on the parent constructor (`super()`) to bind the newly allocated prototype instance to `this`.

    - Accessing `this` prior to calling `super()` results in a fatal `ReferenceError: Must call super constructor before accessing 'this'`.

#### C. Polymorphism

- Achieved through **Method Overriding**. A subclass redefines a method present in its superclass.

- When invoked on an instance, the JS runtime traverses the prototype chain (`instance.__proto__.__proto__...`) and executes the first matching method definition it finds.

#### D. Abstraction

- **JavaScript (Runtime):** Can be emulated by throwing an error inside a base class method to simulate an abstract method, forcing the caller to use a concrete subclass.

- **TypeScript (Compile-Time):** Native `abstract class` and `abstract` method signatures, or pure contracts via `interface`.

### 2. `static` Members

- `static` properties and methods belong to the class constructor object itself, not to the instances created from it.

- In JS prototype mechanics: If `class Dog extends Animal`, static methods inherit through constructor prototypes: `Object.getPrototypeOf(Dog) === Animal`.

- Typical use cases: Utility functions (`Math.max()`), factory methods (`User.createAnonymous()`), and singleton instances.

### 3. Interface vs. Abstract Class (TypeScript)

|**Dimension**|**interface**|**abstract class**|
|---|---|---|
|**Runtime Presence**|**Zero.** Completely erased during TS $\rightarrow$ JS compilation.|**Persists.** Generates an actual JavaScript constructor function and prototype.|
|**Implementation**|Pure contract. Cannot contain executable logic, method bodies, or default values.|Can provide concrete method implementations, default state, and helper utilities alongside abstract signatures.|
|**Constructor**|Cannot have a constructor; cannot be instantiated.|Has a constructor (accessible via `super()`), but cannot be instantiated directly via `new`.|
|**Multiple Inheritance**|A class can implement **multiple** interfaces (`implements A, B, C`).|A class can extend only **one** abstract class (single inheritance constraint).|
|**Primary Use Case**|Defining data shapes, DTOs, API payloads, or behavioral contracts across disparate modules.|Base domain models sharing core business logic, template method patterns, and shared state initialization.|

## Code Snippets / Examples

```typescript
// ============================================================================
// 1. Interface: Pure compile-time contract (Erased at runtime)
// ============================================================================
interface PaymentProcessor {
  processPayment(amount: number): Promise<boolean>;
}

// ============================================================================
// 2. Abstract Class: Base template with shared state and implementation
// ============================================================================
abstract class BaseGateway implements PaymentProcessor {
  // Encapsulation: # is true runtime private (ES2022); cannot be accessed outside
  #apiKey: string;

  // TypeScript protected: accessible in child classes, blocked from outside callers
  protected environment: "sandbox" | "production";

  // Static member: belongs to the class itself, not instances
  static DEFAULT_TIMEOUT_MS = 5000;

  constructor(apiKey: string, environment: "sandbox" | "production") {
    this.#apiKey = apiKey;
    this.environment = environment;
  }

  // Helper method accessing hard private state internally
  protected getMaskedKey(): string {
    return `***${this.#apiKey.slice(-4)}`;
  }

  // Abstraction: Derived classes MUST provide a concrete implementation
  abstract processPayment(amount: number): Promise<boolean>;

  // Static Factory Method
  static logVersion(): void {
    console.log("Gateway SDK v2.4.0");
  }
}

// ============================================================================
// 3. Inheritance & Polymorphism
// ============================================================================
class StripeGateway extends BaseGateway {
  private readonly webhookSecret: string;

  constructor(apiKey: string, webhookSecret: string) {
    // Calling super() binds the parent constructor to 'this'
    super(apiKey, "production");
    this.webhookSecret = webhookSecret;
  }

  // Polymorphism: Specific implementation overriding abstract signature
  async processPayment(amount: number): Promise<boolean> {
    console.log(`Charging $${amount} via Stripe [Env: ${this.environment}]`);
    console.log(`Using Key: ${this.getMaskedKey()}`);
    return true;
  }
}

class PayPalGateway extends BaseGateway {
  constructor(apiKey: string) {
    super(apiKey, "sandbox");
  }

  // Polymorphism: Alternative implementation of the exact same method signature
  async processPayment(amount: number): Promise<boolean> {
    console.log(`Redirecting to PayPal approval flow for $${amount}`);
    return true;
  }
}

// ============================================================================
// Execution
// ============================================================================
const processors: PaymentProcessor[] = [
  new StripeGateway("sk_live_987654321", "whsec_abc123"),
  new PayPalGateway("client_id_paypal_5544"),
];

// Polymorphic invocation
processors.forEach(async (p) => {
  await p.processPayment(150);
});

// Static call directly on the class
BaseGateway.logVersion();
```

## Related Topics

- [[Prototypal Inheritance, Object Prototypes & Constructor Functions|Prototypal Inheritance & The Prototype Chain]]

- [[Object-Oriented Programming (OOP) in JavaScript & TypeScript|Composition over Inheritance]]

- [[SOLID Principles|SOLID Principles in Fullstack Architecture]]

- [[Clean Architecture, Directory Structure & DTOs|Factory and Singleton Design Patterns]]

- [[Object-Oriented Programming (OOP) in JavaScript & TypeScript|TypeScript Type System: Type Aliases vs Interfaces]]

## Tags

#fullstack #interview #javascript #typescript #oop #design-patterns

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups