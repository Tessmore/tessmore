# Book: A Philosophy of Software Design (`John Ousterhout`)

## **1. Complexity Is the Enemy**

Complexity is the silent tax on every software system. It accumulates slowly, spreads unevenly, and eventually makes every change harder than it should be.
Ousterhout’s core message still holds: **your job as a software designer is to fight complexity relentlessly**.

### **1.1 What Creates Complexity?**
- **Obviousness decay** — code that once made sense becomes unclear as the system grows.
- **Shallow abstractions** — modules that expose too many details or require callers to understand internal behavior.
- **Temporal inconsistency** — quick fixes that solve today’s problem but create tomorrow’s debt.
- **Information leakage** — when modules reveal too much about how they work.

### **1.2 The Modern TypeScript View**
TypeScript gives us powerful tools to reduce complexity:
- **Types** that encode intent
- **Interfaces** that enforce boundaries
- **Union types** that eliminate invalid states
- **Generics** that reduce duplication
- **Visibility modifiers** that protect invariants

But these tools only help if we use them to create **deep**, **cohesive**, **stable** modules.

---

## **2. Deep Modules, Not Shallow Ones**

A deep module has:
- A **small surface area** (simple interface)
- A **large internal implementation** (complexity hidden inside)
- A **single, coherent purpose** (SRP)

A shallow module does the opposite:
- Large surface area
- Small internal logic
- Forces callers to understand its internals

### **2.1 Example: Shallow Module (Bad)**

```ts
// BAD: exposes too many details
export class OrderProcessor {
  validate(order: Order): boolean { ... }
  calculatePrice(order: Order): number { ... }
  log(order: Order): void { ... }
  save(order: Order): void { ... }
}
```

Callers must orchestrate the workflow themselves:

```ts
if (processor.validate(order)) {
  const price = processor.calculatePrice(order);
  processor.log(order);
  processor.save(order);
}
```

The module leaks its internal steps.
This is **shallow**.

---

### **2.2 Deep Module (Good)**

```ts
export class OrderProcessor {
  constructor(
    private readonly validator: OrderValidator,
    private readonly calculator: PriceCalculator,
    private readonly discounts: DiscountService,
    private readonly repo: OrderRepository,
    private readonly notifier: NotificationService,
  ) {}

  fulfill(order: Order): ProcessedOrder {
    this.validator.validate(order);
    const totals = this.calculator.calculate(order);
    const priced = this.discounts.apply(totals);
    const saved = this.repo.save(priced);
    this.notifier.sendConfirmation(saved);
    return saved;
  }
}
```

Callers now see only:

```ts
processor.fulfill(order);
```

This is **deep**:
- One public method — one narrow, intention‑revealing interface
- Internal complexity hidden behind it
- The orchestrator changes only when the *workflow* changes; each collaborator owns its own reason to change (SRP)

The collaborators (`OrderValidator`, `PriceCalculator`, `DiscountService`, …) are themselves small, single‑responsibility classes — exactly what Beck argues for in *Implementation Patterns* ("Classes Should Be Small and Focused"). A deep module and small focused classes are **not** in tension: keep the **interface** narrow (Ousterhout) while decomposing the **implementation** into small, injected collaborators (Beck).

---

### **2.3 Deep Modules and SOLID**
- **SRP** → one reason to change
- **OCP** → internals evolve without breaking callers
- **LSP** → substitutable implementations behind the same interface
- **ISP** → small, intention‑revealing interfaces
- **DIP** → callers depend on abstractions, not implementations

Deep modules are SOLID modules.

---

## **3. Strategic Programming Over Tactical Fixes**

Tactical programming is reactive:
- “Just patch it”
- “Quick fix”
- “We’ll clean it up later”

Strategic programming is proactive:
- Think before coding
- Design for clarity
- Refactor continuously
- Improve abstractions as understanding grows

### **3.1 Tactical Fix Example (Bad)**

```ts
// Quick fix: add a flag
function getPrice(order: Order, includeTax = false) {
  const base = order.items.reduce((s, i) => s + i.price, 0);
  return includeTax ? base * 1.21 : base;
}
```

Flags are complexity multipliers:
- They create branching behavior
- They leak internal decisions
- They grow over time

---

### **3.2 Strategic Rewrite (Good)**

```ts
interface PriceStrategy {
  calculate(order: Order): number;
}

class BasePrice implements PriceStrategy {
  calculate(order: Order) {
    return order.items.reduce((s, i) => s + i.price, 0);
  }
}

const VAT_RATE = 1.21; // 21% Dutch BTW — the "why" a name alone can't carry

class TaxedPrice implements PriceStrategy {
  constructor(private readonly base: PriceStrategy) {}
  calculate(order: Order) {
    return this.base.calculate(order) * VAT_RATE;
  }
}
```

Usage:

```ts
const price = new TaxedPrice(new BasePrice()).calculate(order);
```

This is:
- Extensible
- Testable
- Open for extension, closed for modification (OCP)
- Free of flags and branching

Strategic programming **reduces future complexity**.

---

## **4. Continuous Refinement**

Design is not a one‑time event.
It is a **continuous conversation** between:
- Developers
- Domain experts
- The evolving business

A module should become **deeper** over time, not wider.

### **4.1 The Refinement Loop**
1. Build a simple abstraction
2. Use it
3. Discover friction
4. Refine the abstraction
5. Hide more complexity
6. Simplify the interface

This is how deep modules emerge.

---

### **4.2 Example: Evolving a Domain Abstraction**

**Initial version (shallow):**

```ts
class Payment {
  constructor(
    public amount: number,
    public currency: string,
    public method: "ideal" | "creditcard"
  ) {}
}
```

Problems:
- No invariants
- No behavior
- Callers must understand rules

---

**Refined version (deep):**

```ts
class Money {
  constructor(
    private readonly amount: number,
    private readonly currency: "EUR" | "USD"
  ) {
    if (amount < 0) throw new Error("Negative money not allowed");
  }

  add(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new Error("Currency mismatch");
    }
    return new Money(this.amount + other.amount, this.currency);
  }
}

interface PaymentMethod {
  charge(amount: Money): Promise<void>;
}

class Payment {
  constructor(
    private readonly method: PaymentMethod,
    private readonly total: Money
  ) {}

  pay() {
    return this.method.charge(this.total);
  }
}
```

Now:
- Invariants are protected
- Behavior is encapsulated
- Callers see a simple interface
- The module is **deep**

> **Throwing here is deliberate.** `Money` throws because a negative amount or a currency mismatch is a *broken invariant* — a programmer error that should never occur if callers are correct. Throwing is the right tool for invariant violations. For *expected* failures at a boundary (a declined payment, invalid user‑supplied input), prefer returning a `Result` object over throwing — see Beck's "Intention‑Revealing Result Objects." Rule of thumb: **throw for broken invariants, return a `Result` for expected boundary failures.**

---

## **5. Information Hiding and Abstraction**

A good abstraction:
- Hides details
- Protects invariants
- Prevents misuse
- Reduces cognitive load

### **5.1 Example: Hiding Implementation Details**

```ts
// BAD: exposes internal representation
type User = {
  id: string;
  firstName: string;
  lastName: string;
};
```

Callers must assemble names themselves.

---

```ts
// GOOD: hide representation, expose behavior
class User {
  constructor(
    private readonly first: string,
    private readonly last: string
  ) {}

  fullName() {
    return `${this.first} ${this.last}`;
  }
}
```

The representation is now irrelevant.

---

## **6. Designing for Obviousness**

The best code is the code that:
- Requires no comments
- Reads like prose
- Makes illegal states unrepresentable
- Communicates intent

### **6.1 Example: Illegal States Represented in Types**

```ts
// BAD: optional fields create invalid states
interface Booking {
  start?: Date;
  end?: Date;
}
```

---

```ts
// GOOD: enforce the invariant in the constructor
class Booking {
  constructor(
    private readonly start: Date,
    private readonly end: Date
  ) {
    if (end <= start) throw new Error("Invalid booking range");
  }
}
```

A `Booking` can never exist with an invalid range — the constructor guard makes the illegal state unrepresentable, enforced at the boundary where the object is created. (The guard is a *runtime* check, not a compile‑time type; throwing is correct here because an `end` before `start` is a broken invariant — see the throw‑vs‑`Result` rule above.) Used this way, the constructor becomes a design tool.

---

## **7. Comments Capture What Code Cannot**

A common myth is that good code is "self‑documenting" and that every comment is a sign of failure. Ousterhout rejects this. Code can express *what* it does, but it usually cannot express *why* it does it, what it deliberately does **not** do, or which non‑obvious constraints a caller must respect — and those are exactly the things the next reader needs most. Good comments are part of the design, not an admission of defeat.

The rule is therefore two‑part:

- **Replace "what" comments with names.** A comment that merely restates the code (`// check if user is adult`) should become an intention‑revealing function (`isAdult(user)`). This is Beck's "Replace Comments with Code."
- **Write "why" comments for what names can't carry.** Design rationale, domain rules, cross‑module contracts, and warnings about non‑obvious constraints belong in comments — they record information that exists nowhere else in the code.

Never use a comment to *compensate* for bad naming or a shallow abstraction. But never delete a comment that records the *why*: once it's gone, that knowledge is lost.

---

## **8. Designing for Change**

A deep module:
- Changes rarely
- Changes for one reason
- Changes without breaking callers

A shallow module:
- Changes often
- Changes for many reasons
- Forces callers to change

### **8.1 Example: DIP for Change Isolation**

```ts
interface NotificationSender {
  send(message: string): Promise<void>;
}

class OrderNotifier {
  constructor(private readonly sender: NotificationSender) {}

  notify(orderId: string) {
    return this.sender.send(`Order ${orderId} processed`);
  }
}
```

Now:
- Email, SMS, Slack, etc. can be swapped
- OrderNotifier never changes
- The module is deep and stable

## Conclusion

Design software so that each module provides a simple interface that hides a deep, evolving, well‑structured implementation.
