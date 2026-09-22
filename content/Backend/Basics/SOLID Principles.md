

## Key Concepts

- Coined by Robert C. Martin (Uncle Bob); five foundational object-oriented design principles to build maintainable, testable, and loosely coupled software.
    
      
    
- **S – Single Responsibility Principle (SRP):** A class/module should have only one reason to change. It should address a single actor or business domain.
    
      
    
- **O – Open/Closed Principle (OCP):** Software entities should be open for extension, but closed for modification. Add new features by writing new code, not rewriting existing, tested code.
    
      
    
- **L – Liskov Substitution Principle (LSP):** Subtypes must be substitutable for their base types without altering the correctness or intended behavior of the program.
    
      
    
- **I – Interface Segregation Principle (ISP):** Clients should not be forced to depend on methods they do not use. Prefer small, focused interfaces over bloated, general-purpose ones.
    
      
    
- **D – Dependency Inversion Principle (DIP):** High-level modules should not depend on low-level modules; both should depend on abstractions (interfaces). Abstractions should not depend on details; details should depend on abstractions.
    
      
    

## Common Interview Questions

- What are the SOLID principles, and why do we apply them in full-stack application development?
    
      
    
- How do you recognize a violation of the Single Responsibility Principle in a service layer?
    
      
    
- Can you explain the Liskov Substitution Principle using the classic Rectangle-Square problem?
    
      
    
- What is the difference between Dependency Inversion, Inversion of Control (IoC), and Dependency Injection (DI)?
    
      
    
- How does the Interface Segregation Principle prevent "fat interfaces" in TypeScript or microservice design?
    
      
    
- How does following the Open/Closed Principle reduce regression bugs during deployment?
    
      
    

## Strong Answers / Talking Points

### 1. The SOLID Breakdown

#### S: Single Responsibility Principle (SRP)

- **Core Premise:** "One reason to change." Often misunderstood as "doing only one function." It actually means serving a single stakeholder or operational domain.
    
      
    
- **Violation:** A `UserService` that validates user input, writes records to PostgreSQL, formats HTML confirmation emails, and dispatches Stripe charges.
    
      
    
- **Fix:** Decompose into distinct units: `UserRepository` (data persistence), `EmailService` (notification), and `PaymentProcessor` (billing).
    
      
    

#### O: Open/Closed Principle (OCP)

- **Core Premise:** Extend behavior via polymorphism, composition, or strategy patterns rather than mutating existing tested logic via deep nested `switch/case` or `if/else` ladders.
    
      
    
- **Violation:** Modifying an `OrderProcessor` method with a new `if (paymentType === 'crypto')` check every time a new payment integration is added.
    
      
    
- **Fix:** Define a `PaymentStrategy` interface; inject concrete implementations (`StripeStrategy`, `PayPalStrategy`, `CryptoStrategy`).
    
      
    

#### L: Liskov Substitution Principle (LSP)

- **Core Premise:** If $S$ is a subtype of $T$, objects of type $T$ may be replaced with objects of type $S$ without breaking runtime invariants, throwing unexpected exceptions, or weakening post-conditions.
    
      
    
- **Violation:**
    
      
    - Subclass overriding a method to throw `Error("Not Supported")` (e.g., `ReadOnlyFile` extending `File` and throwing on `write()`).
        
          
        
    - Classic Square/Rectangle: A `Square` extending `Rectangle` where setting `width` changes `height` implicitly violates the caller's assumption that dimensions are independently mutable.
        
          
        
- **Fix:** Use composition, or split the hierarchy so only genuine subtypes inherit behavioral contracts.
    
      
    

#### I: Interface Segregation Principle (ISP)

- **Core Premise:** Role interfaces over fat interfaces. No consumer should depend on a contract containing functions it never invokes.
    
      
    
- **Violation:** A single `Worker` interface defining `work()`, `eat()`, `sleep()`, and `fileTaxes()`. An automated `RobotWorker` implementing this interface is forced to implement dummy `eat()` and `sleep()` methods.
    
      
    
- **Fix:** Break the contract into `Workable`, `Feedable`, and `TaxPayer` interfaces.
    
      
    

#### D: Dependency Inversion Principle (DIP)

- **Core Premise:** Decouple high-level policy (domain logic) from low-level detail (databases, third-party SDKs, file systems).
    
      
    
- **Distinction to nail in interviews:**
    
      
    - **DIP:** The architectural principle (rely on abstractions).
        
          
        
    - **IoC (Inversion of Control):** The pattern inversion (framework controls flow, not your main script).
        
          
        
    - **DI (Dependency Injection):** The structural mechanism used to supply concrete dependencies to a class constructor or factory.
        
          
        

### 2. Trade-Offs & Real-World Pragmatism

- **Over-Engineering Risk:** Prematurely applying SOLID leads to unnecessary abstraction layers, hundreds of single-method interfaces, and mental overhead (indirection tax).
    
      
    
- **Rule of Thumb:** Follow YAGNI (You Aren't Gonna Need It) first. Refactor toward OCP/ISP when patterns repeat at least 2–3 times (Rule of Three), or when writing unit tests reveals tight coupling.
    
      
    

## Code Snippets / Examples

```TypeScript
// ============================================================================
// 1. OCP & DIP: Payment Processing Strategy
// ============================================================================

// High-level abstraction (DIP)
export interface PaymentGateway {
  charge(amountCents: number): Promise<{ success: boolean; txId: string }>;
}

// Low-level detail 1 (OCP: Closed for modification, open for extension)
export class StripeGateway implements PaymentGateway {
  async charge(amountCents: number) {
    // Concrete Stripe API calls
    return { success: true, txId: `stripe_${Date.now()}` };
  }
}

// Low-level detail 2
export class PayPalGateway implements PaymentGateway {
  async charge(amountCents: number) {
    // Concrete PayPal API calls
    return { success: true, txId: `paypal_${Date.now()}` };
  }
}

// High-level domain logic: Depends ONLY on the abstraction, not concrete SDKs
export class CheckoutService {
  // Dependency Injection via constructor
  constructor(private readonly paymentGateway: PaymentGateway) {}

  async processOrder(orderId: string, totalCents: number) {
    const result = await this.paymentGateway.charge(totalCents);
    if (!result.success) {
      throw new Error(`Payment failed for order ${orderId}`);
    }
    return { orderId, status: "PAID", txId: result.txId };
  }
}

// ============================================================================
// 2. ISP: Segregating Read and Write operations (CQRS-ready)
// ============================================================================

// Fat interface violation:
// interface EntityStore<T> { read(id: string): T; write(entity: T): void; delete(id: string): void; }

// Segregated contracts (ISP)
export interface Reader<T> {
  findById(id: string): Promise<T | null>;
}

export interface Writer<T> {
  save(entity: T): Promise<void>;
  delete(id: string): Promise<void>;
}

// Client requiring read-only access is not forced to depend on mutation capabilities
export class AnalyticsReportGenerator {
  constructor(private readonly userReader: Reader<{ id: string; name: string }>) {}

  async generateSummary(userId: string) {
    const user = await this.userReader.findById(userId);
    return `Report for ${user?.name ?? "Unknown"}`;
  }
}
```

## Related Topics

- [[Object-Oriented Programming (OOP) in JavaScript & TypeScript]]
    
      
    
- [[Design Patterns: Factory, Strategy, and Observer]]
    
      
    
- [[Clean Architecture and Hexagonal / Ports & Adapters Architecture]]
    
      
    
- [[Inversion of Control & Dependency Injection Frameworks]]
    
      
    
- [[Unit Testing, Mocks, and Test-Driven Development (TDD)]]
    
      
    

## Tags

#fullstack #interview #software-architecture #oop #solid-principles #clean-code

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups