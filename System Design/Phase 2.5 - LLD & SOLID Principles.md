---
tags: [system-design, lld, solid, design-patterns, oop, low-level-design]
status: not-started
phase: 2.5
---

# Phase 2.5 — Low-Level Design (LLD) & SOLID Principles

> [!IMPORTANT]
> LLD is a **separate interview round** at most product-based companies in India (Swiggy, Razorpay, Freshworks, Atlassian). It tests your ability to design classes, interfaces, and relationships for a specific feature — not the whole system. Don't skip this thinking HLD covers it. It doesn't.

---

## 🧭 Overview

Low-Level Design (LLD) asks: *"Given this feature, how would you structure the code?"* You'll draw class diagrams, define interfaces, apply design patterns, and write skeleton code. SOLID principles are the underlying foundation that makes your LLD answers defensible.

This phase sits between [[Phase 2 - Core Building Blocks]] and [[Phase 3 - Data & Storage Deep Dive]] because solid OOP thinking is a prerequisite for writing maintainable microservices.

---

## 📋 Topics Table

| Topic | Level | Why It Matters | Est. Time |
|-------|-------|----------------|-----------|
| SOLID Principles | Must-Know | Foundation of every LLD answer | 2 hrs |
| Creational Design Patterns | Must-Know | Singleton, Factory, Builder, Prototype | 2 hrs |
| Structural Design Patterns | Must-Know | Adapter, Decorator, Facade, Proxy, Composite | 2 hrs |
| Behavioral Design Patterns | Must-Know | Observer, Strategy, Command, Iterator, State | 2 hrs |
| UML Class Diagrams | Must-Know | Communication tool in LLD interviews | 1.5 hrs |
| LLD Case: Parking Lot | Must-Know | Classic beginner LLD problem | 1.5 hrs |
| LLD Case: Library Management System | Must-Know | CRUD + relationships | 1.5 hrs |
| LLD Case: Snake and Ladder / Chess | Good-to-Know | State + game logic | 1.5 hrs |
| LLD Case: Elevator System | Must-Know | State machine, scheduling | 1.5 hrs |
| LLD Case: Cab Booking (Ola/Uber micro) | Must-Know | Strategy + Observer | 1.5 hrs |
| LLD Case: Notification System (code) | Must-Know | Observer + Strategy — bridges to HLD | 1.5 hrs |
| API Design principles (REST best practices) | Must-Know | Versioning, naming, idempotency | 1 hr |

---

## 📖 Detailed Notes

---

### SOLID Principles

SOLID is an acronym for five object-oriented design principles that make code **maintainable, extensible, and testable**.

---

#### S — Single Responsibility Principle (SRP)

**A class should have only one reason to change.**

Every class should do one thing and do it well. If a class handles user authentication AND sends emails AND logs events, it has three reasons to change — a violation.

**Bad (violates SRP):**
```javascript
class UserService {
  createUser(data) { /* ... */ }
  sendWelcomeEmail(user) { /* ... */ }  // ← not UserService's job
  logUserCreation(user) { /* ... */ }   // ← not UserService's job
}
```

**Good (SRP applied):**
```javascript
class UserService {
  createUser(data) { /* ... */ }
}
class EmailService {
  sendWelcomeEmail(user) { /* ... */ }
}
class AuditLogger {
  logUserCreation(user) { /* ... */ }
}
```

> [!TIP] You likely know this — just revise
> As a MERN developer you've likely separated controllers, services, and models. That IS SRP applied to Node.js architecture. Articulate it that way in interviews.

---

#### O — Open/Closed Principle (OCP)

**Software entities should be open for extension, but closed for modification.**

You should be able to add new behavior without changing existing code. Add new classes, don't edit old ones.

**Bad (violates OCP):** Every time a new payment method is added, you modify `PaymentProcessor`:
```javascript
class PaymentProcessor {
  process(type, amount) {
    if (type === 'credit_card') { /* ... */ }
    else if (type === 'upi') { /* ... */ }      // added later — modified existing class
    else if (type === 'wallet') { /* ... */ }   // added later — modified existing class
  }
}
```

**Good (OCP applied):** New payment methods extend an interface:
```javascript
// Define the contract
class PaymentMethod {
  process(amount) { throw new Error('Must implement'); }
}

class CreditCardPayment extends PaymentMethod {
  process(amount) { /* credit card logic */ }
}

class UPIPayment extends PaymentMethod {
  process(amount) { /* UPI logic */ }
}

// Processor never changes — it works with any PaymentMethod
class PaymentProcessor {
  process(paymentMethod, amount) {
    paymentMethod.process(amount);
  }
}
```

---

#### L — Liskov Substitution Principle (LSP)

**Subtypes must be substitutable for their base types without breaking correctness.**

If class B extends class A, you must be able to use B anywhere A is expected without the program breaking.

**Violation example:**
```javascript
class Bird {
  fly() { /* ... */ }
}
class Penguin extends Bird {
  fly() { throw new Error("Penguins can't fly!"); }  // ← Breaks LSP
}
```

**Fix:** Model the hierarchy correctly:
```javascript
class Bird { }
class FlyingBird extends Bird { fly() { /* ... */ } }
class Penguin extends Bird { swim() { /* ... */ } }
```

> [!NOTE]
> LSP violations often appear when inheritance is used for code reuse rather than for true "is-a" relationships. Prefer composition over inheritance in most cases.

---

#### I — Interface Segregation Principle (ISP)

**Clients should not be forced to depend on interfaces they don't use.**

Don't create fat interfaces. Split them into smaller, role-specific ones.

**Bad (fat interface):**
```javascript
// Every implementor must implement ALL methods — even irrelevant ones
class Animal {
  walk() {}
  fly() {}
  swim() {}
}
```

**Good (segregated interfaces):**
```javascript
class Walkable { walk() {} }
class Flyable  { fly()  {} }
class Swimmable { swim() {} }

class Duck extends Walkable, Flyable, Swimmable { /* implements all 3 */ }
class Dog extends Walkable, Swimmable { /* implements 2 */ }
```

---

#### D — Dependency Inversion Principle (DIP)

**High-level modules should not depend on low-level modules. Both should depend on abstractions.**

Your business logic (high-level) should not directly instantiate infrastructure components (low-level). Depend on interfaces, not concrete implementations.

**Bad (high-level depends on low-level):**
```javascript
class OrderService {
  constructor() {
    this.db = new MySQLDatabase();  // ← hardcoded dependency
    this.mailer = new SendGridMailer();  // ← hardcoded dependency
  }
}
```

**Good (DIP + Dependency Injection):**
```javascript
class OrderService {
  constructor(database, mailer) {  // ← accepts abstractions
    this.db = database;
    this.mailer = mailer;
  }
}

// Caller injects the concrete implementations
const service = new OrderService(new MySQLDatabase(), new SendGridMailer());
// Or swap with mocks in tests:
const service = new OrderService(new MockDatabase(), new MockMailer());
```

> [!TIP] You likely know this — just revise
> This is exactly what dependency injection frameworks (NestJS uses DI heavily) do for you. Knowing the *principle* behind it makes your answer credible.

---

### Design Patterns

Design patterns are reusable solutions to commonly occurring problems in software design. There are 23 classic Gang of Four patterns — you need to know ~10 well for interviews.

---

### Creational Patterns

#### Singleton
Ensures a class has only one instance and provides a global access point.

```javascript
class DatabaseConnection {
  static instance = null;
  
  static getInstance() {
    if (!DatabaseConnection.instance) {
      DatabaseConnection.instance = new DatabaseConnection();
    }
    return DatabaseConnection.instance;
  }
  
  private constructor() { /* connect to DB */ }
}
```

**Use cases:** DB connection pool, logger, config manager, thread pool.

> [!WARNING]
> Singleton introduces global state and makes testing harder (hard to mock). Use it only when a single instance is a genuine requirement, not just convenient.

---

#### Factory Method
Define an interface for creating objects, but let subclasses decide which class to instantiate.

```javascript
class NotificationFactory {
  static create(type) {
    switch(type) {
      case 'email': return new EmailNotification();
      case 'sms':   return new SMSNotification();
      case 'push':  return new PushNotification();
      default: throw new Error(`Unknown type: ${type}`);
    }
  }
}

// Usage — caller doesn't know the concrete class
const notification = NotificationFactory.create('email');
notification.send(user, message);
```

**Use cases:** Payment method creation, notification type selection, parser creation (JSON/XML/CSV).

---

#### Builder
Construct a complex object step by step. Separate construction from representation.

```javascript
class QueryBuilder {
  constructor() { this.query = {}; }
  
  select(fields) { this.query.fields = fields; return this; }
  from(table)    { this.query.table = table;  return this; }
  where(cond)    { this.query.cond = cond;    return this; }
  limit(n)       { this.query.limit = n;      return this; }
  build()        { return this.query; }
}

const query = new QueryBuilder()
  .select(['id', 'name'])
  .from('users')
  .where({ active: true })
  .limit(100)
  .build();
```

**Use cases:** SQL query builders, HTTP request builders, complex config objects, test data factories.

---

### Structural Patterns

#### Adapter
Convert the interface of a class into another interface clients expect. Allows incompatible interfaces to work together.

```javascript
// Third-party payment library has a different interface
class StripeAPI {
  chargeCard(cardToken, amountInCents) { /* ... */ }
}

// Your app expects this interface
class PaymentAdapter {
  constructor() { this.stripe = new StripeAPI(); }
  
  // Adapts your interface to Stripe's
  processPayment(amount, currency, cardToken) {
    const amountInCents = amount * 100;
    return this.stripe.chargeCard(cardToken, amountInCents);
  }
}
```

**Use cases:** Integrating third-party libraries, legacy system integration.

---

#### Decorator
Attach additional behavior to an object dynamically without subclassing.

```javascript
// Base logger
class Logger {
  log(message) { console.log(message); }
}

// Decorator adds timestamp
class TimestampLogger {
  constructor(logger) { this.logger = logger; }
  log(message) { this.logger.log(`[${new Date().toISOString()}] ${message}`); }
}

// Decorator adds request ID
class RequestIDLogger {
  constructor(logger) { this.logger = logger; }
  log(message) { this.logger.log(`[req-${Math.random()}] ${message}`); }
}

// Chain decorators
const logger = new RequestIDLogger(new TimestampLogger(new Logger()));
logger.log("User created");
// Output: [req-0.123] [2026-03-20T...] User created
```

**Use cases:** Middleware chains (Express uses this!), adding auth/logging/caching to services.

---

#### Facade
Provide a simplified interface to a complex subsystem.

```javascript
// Complex subsystem
class InventoryService { reserve(itemId, qty) {} }
class PaymentService   { charge(userId, amount) {} }
class ShippingService  { schedule(orderId) {} }
class EmailService     { sendConfirmation(userId) {} }

// Facade — simple interface for checkout
class CheckoutFacade {
  checkout(userId, cart) {
    this.inventory.reserve(cart.items);
    this.payment.charge(userId, cart.total);
    const order = this.shipping.schedule(cart);
    this.email.sendConfirmation(userId);
    return order;
  }
}
```

**Use cases:** Checkout flows, SDK entry points, API controllers that orchestrate multiple services.

---

### Behavioral Patterns

#### Observer (Publish-Subscribe)
Define a one-to-many dependency: when one object changes state, all dependents are notified.

```javascript
class EventEmitter {
  constructor() { this.listeners = {}; }
  
  on(event, callback) {
    if (!this.listeners[event]) this.listeners[event] = [];
    this.listeners[event].push(callback);
  }
  
  emit(event, data) {
    (this.listeners[event] || []).forEach(cb => cb(data));
  }
}

const orderEvents = new EventEmitter();
orderEvents.on('order.placed', (order) => inventoryService.reserve(order));
orderEvents.on('order.placed', (order) => emailService.sendConfirmation(order));
orderEvents.on('order.placed', (order) => analyticsService.track(order));

// When order is placed, all observers are notified
orderEvents.emit('order.placed', newOrder);
```

> [!NOTE]
> Node.js's built-in `EventEmitter` is exactly this pattern. Kafka and message queues are the distributed version of the Observer pattern. Bridging this connection impresses interviewers.

---

#### Strategy
Define a family of algorithms, encapsulate each one, and make them interchangeable.

```javascript
// Sorting strategies
class BubbleSort  { sort(data) { /* O(n²) */ } }
class QuickSort   { sort(data) { /* O(n log n) */ } }
class MergeSort   { sort(data) { /* O(n log n) */ } }

class Sorter {
  constructor(strategy) { this.strategy = strategy; }
  setStrategy(strategy) { this.strategy = strategy; }
  sort(data) { return this.strategy.sort(data); }
}

// Swap strategy at runtime based on data size
const sorter = new Sorter(new QuickSort());
if (data.length > 1_000_000) sorter.setStrategy(new MergeSort());
sorter.sort(data);
```

**Use cases:** Payment methods, routing algorithms, compression algorithms, pricing strategies.

---

#### Command
Encapsulate a request as an object. Allows undo/redo, queuing, and logging of operations.

```javascript
class Command {
  execute() {}
  undo() {}
}

class TransferMoneyCommand extends Command {
  constructor(fromAccount, toAccount, amount) { /* ... */ }
  execute() { /* debit from, credit to */ }
  undo()    { /* credit from, debit to */ }
}

class CommandHistory {
  constructor() { this.history = []; }
  execute(command) {
    command.execute();
    this.history.push(command);
  }
  undo() {
    const command = this.history.pop();
    command.undo();
  }
}
```

**Use cases:** Undo/redo (text editors), transaction rollback, job queues, macro recording.

---

#### State
Allow an object to alter its behavior when its internal state changes.

```javascript
// States
class PendingState  { handle(order) { /* validate → move to Processing */ } }
class ProcessingState { handle(order) { /* charge → move to Shipped */ } }
class ShippedState  { handle(order) { /* deliver → move to Delivered */ } }
class DeliveredState { handle(order) { /* complete */ } }

class Order {
  constructor() { this.state = new PendingState(); }
  setState(state) { this.state = state; }
  process() { this.state.handle(this); }
}
```

**Use cases:** Order lifecycle, payment states, connection states (closed/connecting/open), elevator control.

---

### UML Class Diagrams

Know these relationships for whiteboard LLD:

| Relationship | UML Symbol | Meaning | Example |
|-------------|------------|---------|---------|
| Association | `A ——→ B` | A has a reference to B | Order has User |
| Aggregation | `A ◇——→ B` | B can exist without A | Department has Employees |
| Composition | `A ◆——→ B` | B cannot exist without A | House has Rooms |
| Inheritance | `A ——▷ B` | A is a type of B | Dog is an Animal |
| Interface | `A ——▷ «I»` | A implements interface I | EmailNotifier implements Notifier |
| Dependency | `A - - → B` | A uses B (transient) | Service uses Logger |

> [!TIP]
> In virtual interviews, you don't need perfect UML. Draw boxes with class names, list key methods/attributes inside, and use labeled arrows. Clarity over formality.

---

### LLD Case: Parking Lot

**Requirements:** Multiple floors, different vehicle types (bike, car, truck), different spot sizes, entry/exit, fee calculation.

**Key Classes:**
```
ParkingLot
  └── floors: Floor[]
  └── getAvailableSpot(vehicleType): Spot
  └── park(vehicle): Ticket
  └── unpark(ticket): Receipt

Floor
  └── spots: Spot[]
  └── getAvailableSpot(vehicleType): Spot

Spot (abstract)
  ├── BikeSpot
  ├── CarSpot
  └── TruckSpot
  └── isOccupied: boolean
  └── vehicle: Vehicle

Vehicle (abstract)
  ├── Bike
  ├── Car
  └── Truck

Ticket
  └── spot: Spot
  └── vehicle: Vehicle
  └── entryTime: DateTime

FeeCalculator (Strategy pattern)
  ├── HourlyFeeCalculator
  └── FlatFeeCalculator
```

**Patterns used:** Factory (create spot type), Strategy (fee calculation), Singleton (ParkingLot instance).

---

### LLD Case: Elevator System

**Requirements:** N elevators, M floors, minimize wait time, handle up/down requests.

**Key Classes:**
```
ElevatorSystem (Singleton)
  └── elevators: Elevator[]
  └── dispatch(floorRequest): Elevator  // Strategy pattern for scheduling

Elevator
  └── currentFloor: int
  └── state: ElevatorState (IDLE, MOVING_UP, MOVING_DOWN)
  └── destinations: PriorityQueue<int>
  └── addDestination(floor): void
  └── step(): void  // move one floor

ElevatorState (State pattern)
  ├── IdleState
  ├── MovingUpState
  └── MovingDownState

DispatchStrategy (Strategy pattern)
  ├── NearestElevatorStrategy
  └── LookAlgorithmStrategy  // SCAN disk scheduling variant
```

---

### LLD Case: Notification System (Code-level)

**Requirements:** Send notifications via email, SMS, push. User can set preferences. Easy to add new channels.

**Design (applying OCP + Strategy + Factory + Observer):**
```
NotificationService
  └── send(userId, event, data): void

NotificationChannel (interface)
  ├── EmailChannel
  ├── SMSChannel
  └── PushChannel
  └── send(user, message): void

UserPreferenceService
  └── getChannels(userId, event): NotificationChannel[]

NotificationFactory
  └── create(channelType): NotificationChannel

// Flow:
// Event fired → NotificationService
// → fetch user prefs → get channels list
// → for each channel: channel.send(user, message)
```

**Adding WhatsApp channel:** Create `WhatsAppChannel implements NotificationChannel`. Zero changes to `NotificationService`. ← This is OCP in action.

---

### API Design Principles

> [!TIP] You likely know this — just revise
> You've built REST APIs in Express. Know the principles by name.

**REST Best Practices:**
- Use nouns, not verbs: `GET /users/123` not `GET /getUser?id=123`
- Use HTTP methods semantically: GET (read), POST (create), PUT (replace), PATCH (update), DELETE (remove)
- Use plural nouns: `/users`, `/orders`, `/products`
- Nested resources for relationships: `GET /users/123/orders`
- HTTP status codes correctly: 200 OK, 201 Created, 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable Entity, 500 Internal Server Error
- Pagination: use cursor-based (`?cursor=abc`) over offset (`?page=2&limit=20`) at scale
- Versioning: `/v1/users` or header-based (`Accept: application/vnd.myapp.v1+json`)
- Idempotency headers for unsafe operations: `Idempotency-Key: uuid`

**GraphQL trade-offs:**
- Avoid over-fetching/under-fetching — clients specify exact fields
- N+1 query problem → use DataLoader (batching)
- Schema introspection is powerful but exposes your data model

**gRPC best practices:**
- Define contracts in `.proto` files (version them!)
- Use for internal service-to-service calls where performance matters
- Supports streaming (client, server, bidirectional)

---

## 📚 Suggested Resources

| Resource | Why |
|----------|-----|
| Gaurav Sen — "Low Level Design" playlist (YouTube) | Best LLD case walkthroughs in India context |
| Udit Agarwal — "LLD" playlist (YouTube) | Parking lot, elevator, snake ladder in Java/JS |
| refactoring.guru | Best visual explanation of all 23 design patterns |
| *Head First Design Patterns* (book) | Accessible, example-heavy intro |
| *Clean Code* — Robert C. Martin | SOLID + naming + structure in practice |
| NeetCode — "OOP Design" section | Interview-focused pattern application |

---

## ✅ Progress Tracker

### SOLID
- [ ] SRP — can identify violations in code and fix them
- [ ] OCP — can apply to a payment/notification system
- [ ] LSP — can spot incorrect inheritance hierarchies
- [ ] ISP — can split a fat interface into role-based ones
- [ ] DIP — can apply dependency injection to Node.js service

### Creational Patterns
- [ ] Singleton — know when to use and when NOT to
- [ ] Factory Method — applied to notification/payment type creation
- [ ] Builder — applied to query builder or config object

### Structural Patterns
- [ ] Adapter — third-party library integration
- [ ] Decorator — middleware chain, adding behavior without subclassing
- [ ] Facade — simplifying a complex subsystem

### Behavioral Patterns
- [ ] Observer — event-driven, EventEmitter, link to Kafka
- [ ] Strategy — swappable algorithms at runtime
- [ ] Command — undo/redo, job queues
- [ ] State — order lifecycle, elevator states

### LLD Cases
- [ ] Parking Lot — designed with class diagram
- [ ] Elevator System — State + Strategy applied
- [ ] Notification System — OCP + Factory + Observer applied
- [ ] One additional case (Library / Chess / Cab Booking)

### API Design
- [ ] REST best practices — naming, status codes, pagination
- [ ] Idempotency key pattern in API design
- [ ] Cursor vs offset pagination trade-off

---

*Back to index → [[System Design Roadmap]]*
*Next → [[Phase 3 - Data & Storage Deep Dive]]*
