# Clean Architecture, Directory Structure & DTOs

## Key Concepts

- **Clean Architecture (Robert C. Martin / Uncle Bob):** A layered architectural pattern designed to separate software into concentric layers around business rules, enforcing the **Dependency Rule**: _dependencies must point inward only_.

- **The Dependency Rule:** Inner layers know nothing about outer layers. Business logic does not depend on databases, UI frameworks, ORMs, or third-party SDKs.

- **Layers (Center to Outer):**

    1. **Enterprise Business Rules (Entities):** Plain domain objects and invariants.

    2. **Application Business Rules (Use Cases):** Application-specific workflows, orchestrations, and interfaces.

    3. **Interface Adapters (Controllers, Gateways, Presenters):** Translators converting data between use cases and external platforms.

    4. **Frameworks & Drivers (Web, DB, Devices, UI):** Low-level details (Express, NestJS, Prisma, PostgreSQL, Redis).

- **Data Transfer Object (DTO):** A dumb data container object with no business logic or behavior, used strictly to serialize, validate, and transfer data across architectural boundaries (e.g., API request body $\rightarrow$ controller $\rightarrow$ use case).

## Common Interview Questions

- What is Clean Architecture, and how does the "Dependency Rule" protect business logic from framework lock-in?

- How does Clean Architecture compare to Hexagonal Architecture (Ports & Adapters) and Onion Architecture?

- What is a DTO, and why shouldn't domain entities be returned directly to the client as API responses?

- How do you structure a production-grade full-stack / Node.js backend using Clean Architecture?

- Where do database schemas and ORM models (Prisma/TypeORM) belong in Clean Architecture, and why shouldn't they double as domain entities?

- How do you manage dependency injection in Clean Architecture without tight coupling to an IoC library?

## Strong Answers / Talking Points

### 1. The Core Layers & The Dependency Rule

```text
+------------------------------------------------------------+
|  Frameworks & Drivers (Express, Fastify, Prisma, PostgreSQL) |
|   +----------------------------------------------------+   |
|   |  Interface Adapters (Controllers, Repositories)    |   |
|   |   +--------------------------------------------+   |   |
|   |   |  Use Cases (Application Business Rules)    |   |   |
|   |   |   +------------------------------------+   |   |   |
|   |   |   |  Entities (Core Domain Logic)      |   |   |   |
|   |   |   +------------------------------------+   |   |   |
|   |   +--------------------------------------------+   |   |
|   +----------------------------------------------------+   |
+------------------------------------------------------------+
            Dependencies point INWARD ONLY  ----->
```

- **Domain / Entities (Innermost):**

    - Plain TypeScript classes or value objects.

    - Zero external imports (no ORM decorators, no framework libraries).

    - Encapsulates critical enterprise rules (e.g., calculating tax, ensuring user age $\ge$ 18).

- **Use Cases / Application:**

    - Coordinates domain entities to execute user goals (e.g., `RegisterUserUseCase`, `PlaceOrderUseCase`).

    - Defines **Ports / Interfaces** for data access (e.g., `IUserRepository`), while concrete implementations live outside.

- **Interface Adapters:**

    - Controllers take HTTP payloads, parse them into DTOs, and pass them to use cases.

    - Repository implementations fulfill use-case interfaces by querying SQL/NoSQL databases via an ORM or driver.

- **Frameworks & Drivers (Outermost):**

    - Express/Fastify routes, database connections, message queue workers, third-party API clients.

    - Can be swapped out (e.g., migrating from Express to Fastify, or PostgreSQL to DynamoDB) without modifying a single line of business logic in the inner layers.

### 2. What is a DTO (Data Transfer Object)?

- **Definition:** An object designed solely to ferry data across network or architectural boundaries. It contains data attributes, serialization logic, and validation tags—**never business behavior**.

- **Why DTOs are mandatory in Clean Architecture:**

    - **Security (Prevents Over-Posting / Mass Assignment):** Exposing domain models directly allows malicious users to inject fields like `isAdmin: true` or `role: "superuser"`. DTOs define strict whitelists of accepted inputs.

    - **Decoupling Domain from API Contracts:** Changing an internal database column or domain entity property should not automatically break external REST/GraphQL client contracts.

    - **Performance / Minimization:** DTOs filter out sensitive database columns (e.g., `password_hash`, internal audit timestamps) before payloads reach the wire.

### 3. Production Directory Structure (Domain-Driven / Clean Layout)

```text
src/
├── core/                           # Cross-cutting foundational logic
│   ├── errors/                     # AppError, NotFoundError, DomainError
│   └── result/                     # Result<T, E> functional monad
│
├── modules/
│   └── user/
│       ├── domain/                 # 1. CORE DOMAIN (No external dependencies)
│       │   ├── entities/           # User.ts (domain methods & invariants)
│       │   ├── value-objects/      # Email.ts, PasswordHash.ts
│       │   └── events/             # UserCreatedEvent.ts
│       │
│       ├── application/            # 2. USE CASES (Orchestration & Ports)
│       │   ├── use-cases/          # CreateUserUseCase.ts, GetUserProfileUseCase.ts
│       │   ├── dtos/               # CreateUserRequestDTO.ts, UserResponseDTO.ts
│       │   └── ports/              # IUserRepository.ts, INotificationService.ts
│       │
│       ├── infrastructure/         # 3. ADAPTERS & DRIVERS (Concrete details)
│       │   ├── persistence/        # PrismaUserRepository.ts, UserSchema.prisma
│       │   ├── services/           # SendGridNotificationService.ts
│       │   └── http/               # UserController.ts, user.routes.ts
│       │
│       └── index.ts                # Module wiring / DI container entrypoint
│
└── main.ts                         # App bootstrap, HTTP server listener
```

## Code Snippets / Examples

```typescript
// ============================================================================
// 1. DTO: Boundary Data Contract (with validation)
// ============================================================================
export interface CreateUserDTO {
  email: string;
  name: string;
  age: number;
}

export interface UserResponseDTO {
  id: string;
  email: string;
  name: string;
  createdAt: string; // Serialized ISO string, omitting sensitive internal state
}

// ============================================================================
// 2. Domain Entity: Invariants & Pure Business Rules (Zero external imports)
// ============================================================================
export class User {
  private constructor(
    public readonly id: string,
    public email: string,
    public name: string,
    public age: number,
    public readonly createdAt: Date
  ) {
    if (age < 18) {
      throw new Error("Domain invariant violated: User must be at least 18 years old.");
    }
  }

  // Factory method enforces validation upon creation
  public static create(id: string, email: string, name: string, age: number): User {
    return new User(id, email, name, age, new Date());
  }
}

// ============================================================================
// 3. Application Port: Inverted dependency interface
// ============================================================================
export interface IUserRepository {
  save(user: User): Promise<void>;
  findByEmail(email: string): Promise<User | null>;
}

// ============================================================================
// 4. Use Case: Orchestrates the flow using DTOs and Interfaces
// ============================================================================
export class CreateUserUseCase {
  constructor(private readonly userRepository: IUserRepository) {}

  async execute(dto: CreateUserDTO): Promise<UserResponseDTO> {
    const existing = await this.userRepository.findByEmail(dto.email);
    if (existing) {
      throw new Error("Conflict: Email is already registered.");
    }

    const newUser = User.create(
      crypto.randomUUID(),
      dto.email,
      dto.name,
      dto.age
    );

    await this.userRepository.save(newUser);

    // Map entity to outbound response DTO (prevent exposing internals)
    return {
      id: newUser.id,
      email: newUser.email,
      name: newUser.name,
      createdAt: newUser.createdAt.toISOString(),
    };
  }
}

// ============================================================================
// 5. Interface Adapter: Controller (Translates HTTP to Use Case)
// ============================================================================
export class UserController {
  constructor(private readonly createUserUseCase: CreateUserUseCase) {}

  async handleCreate(req: { body: unknown }, res: { status: Function; json: Function }) {
    try {
      const dto = req.body as CreateUserDTO; // Validated via Zod/Joi in production middleware
      const responseDto = await this.createUserUseCase.execute(dto);
      return res.status(201).json(responseDto);
    } catch (err: any) {
      return res.status(400).json({ error: err.message });
    }
  }
}
```

## Related Topics

- [[SOLID Principles|SOLID Principles in Fullstack Architecture]]

- [[Clean Architecture, Directory Structure & DTOs|Hexagonal Architecture (Ports and Adapters)]]

- [[Clean Architecture, Directory Structure & DTOs|Domain-Driven Design (DDD) Fundamentals]]

- [[Object-Oriented Programming (OOP) in JavaScript & TypeScript]]

- [[Clean Architecture, Directory Structure & DTOs|API Security: Mass Assignment and Input Validation]]

## Tags

#fullstack #interview #software-architecture #clean-architecture #design-patterns #dto

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups