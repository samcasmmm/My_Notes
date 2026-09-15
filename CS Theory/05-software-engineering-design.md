[🏠 Back to CS Theory Index](./README.md) • [⬅️ Prev: Distributed Systems](./04-distributed-systems.md) • [Next: System Design Blueprints ➡️](./06-system-design-blueprints.md)

<div align="center">
  <h1>05. Software Engineering & Object-Oriented Design</h1>
  <p><b>SOLID Principles, Gang of Four (GoF) Design Patterns, Clean Architecture, & Testing Pyramid</b></p>
</div>

---

## 📑 Module Index
- [1. SOLID Principles](#1-solid-principles)
  - [SRP, OCP, LSP, ISP, & DIP with Real-World Code Examples](#solid-in-practice)
- [2. Gang of Four (GoF) Design Patterns](#2-gang-of-four-gof-design-patterns)
  - [Creational Patterns: Singleton, Factory, Builder](#creational-patterns)
  - [Structural Patterns: Adapter, Decorator, Facade, Proxy](#structural-patterns)
  - [Behavioral Patterns: Strategy, Observer, State, Command](#behavioral-patterns)
- [3. Architectural Design & Domain Boundaries](#3-architectural-design)
  - [Clean Architecture & Hexagonal (Ports & Adapters)](#clean-architecture)
  - [MVC vs MVVM vs Component-Driven Architectures](#mvc-vs-mvvm)
- [4. Software Testing & Quality Assurance](#4-software-testing--quality-assurance)
  - [The Testing Pyramid (Unit vs Integration vs E2E)](#the-testing-pyramid)
  - [Test-Driven Development (TDD) & Mocking Best Practices](#tdd-and-mocking)
- [5. CI/CD & Modern Release Engineering](#5-cicd--modern-release-engineering)
  - [Trunk-Based Development vs GitFlow](#git-branching-strategies)
  - [Blue-Green Deployments, Canary Releases & Feature Flags](#zero-downtime-deployments)

---

## 1. SOLID Principles

### 1. S — Single Responsibility Principle (SRP)
> *"A class/module should have one, and only one, reason to change."*

```typescript
// ❌ VIOLATION: User class handles data, validation, AND database persistence
class User {
  constructor(public name: string, public email: string) {}
  saveToDatabase() { /* db insert query */ }
  sendWelcomeEmail() { /* SMTP send */ }
}

// ✅ REFACTORED: Decoupled into single responsibilities
class User {
  constructor(public name: string, public email: string) {}
}
class UserRepository {
  save(user: User) { /* db insert query */ }
}
class EmailService {
  sendWelcome(user: User) { /* SMTP send */ }
}
```

---

### 2. O — Open/Closed Principle (OCP)
> *"Software entities should be open for extension, but closed for modification."*

```typescript
// ✅ REFACTORED: Add new payment methods without touching existing code
interface PaymentProcessor {
  process(amount: number): Promise<void>;
}

class StripeProcessor implements PaymentProcessor {
  async process(amount: number) { /* Stripe API call */ }
}

class RazorpayProcessor implements PaymentProcessor {
  async process(amount: number) { /* Razorpay API call */ }
}

class CheckoutService {
  constructor(private processor: PaymentProcessor) {}
  async checkout(amount: number) {
    await this.processor.process(amount);
  }
}
```

---

### 3. L — Liskov Substitution Principle (LSP)
> *"Subtypes must be substitutable for their base types without altering program correctness."*

- **Classic Violation:** `Square` extending `Rectangle` where mutating width unexpectedly alters height.
- **Rule:** Derived classes must not throw unexpected exceptions or violate base contracts.

---

### 4. I — Interface Segregation Principle (ISP)
> *"Clients should not be forced to depend upon interfaces they do not use."*

```typescript
// ❌ VIOLATION: Bloated monolithic interface
interface Worker {
  work(): void;
  eat(): void;
  sleep(): void;
}

// ✅ REFACTORED: Focused segregated interfaces
interface Workable { work(): void; }
interface Feedable { eat(): void; }

class HumanWorker implements Workable, Feedable {
  work() { /* coding */ }
  eat() { /* lunch */ }
}
class RobotWorker implements Workable {
  work() { /* assembling */ }
}
```

---

### 5. D — Dependency Inversion Principle (DIP)
> *"High-level modules should not depend on low-level modules. Both should depend on abstractions."*

- Depend on interfaces (`DatabaseClient`), not concrete implementations (`PostgreSQLDriver`). Inject dependencies via constructor injection.

---

## 2. Gang of Four (GoF) Design Patterns

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ 1. CREATIONAL PATTERNS (Object Creation Mechanisms)                                   │
│    • Singleton: Guarantees single instance across app runtime (DB pool, Logger).      │
│    • Factory Method: Delegating object instantiation to subclasses.                    │
│    • Builder: Step-by-step construction of complex multi-attribute objects.            │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 2. STRUCTURAL PATTERNS (Class & Object Composition)                                    │
│    • Adapter: Translates incompatible interfaces (Legacy XML API -> Modern JSON).     │
│    • Decorator: Dynamically attaches extra behaviors to objects without subclassing.  │
│    • Facade: Provides simple interface over complex multi-subsystem engine.           │
│    • Proxy: Placeholder/surrogate controlling access to expensive object (Auth, Cache)│
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 3. BEHAVIORAL PATTERNS (Communication & Responsibility Assignment)                     │
│    • Strategy: Swappable algorithms selected at runtime (Sorting, Payment routing).    │
│    • Observer: Pub/Sub event broadcaster notifying multiple subscribers on change.     │
│    • State: Allows an object to alter behavior when its internal state changes.        │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

#### Strategy Pattern Example:
```typescript
interface CompressionStrategy {
  compress(files: string[]): Buffer;
}

class ZipCompression implements CompressionStrategy {
  compress(files: string[]): Buffer { /* ZIP logic */ return Buffer.from(""); }
}

class TarGzCompression implements CompressionStrategy {
  compress(files: string[]): Buffer { /* TarGz logic */ return Buffer.from(""); }
}

class FileArchiver {
  constructor(private strategy: CompressionStrategy) {}
  setStrategy(strategy: CompressionStrategy) { this.strategy = strategy; }
  archive(files: string[]) { return this.strategy.compress(files); }
}
```

---

## 3. Architectural Design

### Clean Architecture (Hexagonal / Onion)

```
        ┌────────────────────────────────────────────────────────┐
        │  FRAMEWORKS & DRIVERS (Express, React, Postgres, AWS)   │
        │  ┌──────────────────────────────────────────────────┐  │
        │  │  INTERFACE ADAPTERS (Controllers, Repositories)  │  │
        │  │  ┌────────────────────────────────────────────┐  │  │
        │  │  │  APPLICATION USE CASES (CreateOrderService)│  │  │
        │  │  │  ┌──────────────────────────────────────┐  │  │  │
        │  │  │  │  DOMAIN ENTITIES (Order, User, Money)│  │  │  │
        │  │  │  └──────────────────────────────────────┘  │  │  │
        │  │  └────────────────────────────────────────────┘  │  │
        │  └──────────────────────────────────────────────────┘  │
        └────────────────────────────────────────────────────────┘
```

- **The Dependency Rule:** Source code dependencies point **inwards only**. Domain entities have **zero dependencies** on external frameworks, databases, or UI libraries.

---

## 4. Software Testing & Quality Assurance

### The Testing Pyramid

```
                / \
               /   \
              / E2E \       (5-10%)  • Cypress, Playwright, Maestro
             /───────\
            /  INTEGR- \     (20-30%) • Supertest, Testcontainers (Real DBs)
           /   ATION    \
          /───────────────\
         /      UNIT       \ (60-70%) • Jest, Vitest (Pure functions, fast)
        /───────────────────\
```

- **Unit Tests:** Fast, isolated tests mocking all external network/DB dependencies.
- **Integration Tests:** Verifies communication between boundaries (Service layer + real PostgreSQL instance via Docker Testcontainers).
- **End-to-End (E2E) Tests:** Simulates real user interaction through frontend UI and full production backend.

---

## 5. CI/CD & Modern Release Engineering

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ Zero-Downtime Deployment Strategies                                                    │
├─────────────────────────┬──────────────────────────────────────────────────────────────┤
│ **Blue-Green**          │ Two identical production environments (Blue=Active, Green=   │
│                         │ New). Router instantly flips 100% traffic from Blue to Green.│
├─────────────────────────┼──────────────────────────────────────────────────────────────┤
│ **Canary Deployment**   │ Routes 5% of traffic to new version; monitors error rates.   │
│                         │ Gradually increases to 25% -> 50% -> 100% over hours.        │
├─────────────────────────┼──────────────────────────────────────────────────────────────┤
│ **Feature Flags**       │ Decouples code deployment from feature release. Allows       │
│                         │ instant runtime kill-switches without redeploying code.      │
└─────────────────────────┴──────────────────────────────────────────────────────────────┘
```

---

[➡️ Continue to Module 06: System Design Blueprints](./06-system-design-blueprints.md)
