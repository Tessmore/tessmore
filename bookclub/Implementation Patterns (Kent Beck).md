## Book: Implementation Patterns (`Kent Beck`)

The central thesis of the book is that **code is a form of communication.** Beck argues that you don't write code for the compiler; you write it for the next human being who has to read it. He bases his patterns on three foundational values:

* **Communication:** Code should be easy to "get" at a glance.
* **Simplicity:** Eliminate everything that doesn't add value.
* **Flexibility:** Leave room for change, but don't over-engineer for "potential" future needs.

The rest of this document groups Beck's patterns under those three values.

## 1. Communication — Code Should Be Easy to "Get" at a Glance

### Code Should Communicate Intent

The primary purpose of code is to *communicate*, not to instruct the machine.

Your code should make the *why* obvious, not just the *what*. Clear intent is the foundation of **S (Single Responsibility)** — a function with one responsibility is easier to name clearly.

**Bad**
```ts
function handle(u: User) {
  if (!u.a) throw new Error("x");
  return u.d.map(x => x.v);
}
```

**Better**
```ts
function getVerifiedDeviceIds(user: User): string[] {
  if (!user.isActive) {
    throw new Error("User must be active to retrieve devices");
  }

  return user.devices.map(device => device.id);
}
```

### Names Reveal Purpose

Beck emphasizes naming as the highest‑leverage implementation skill. Every name (variable, method, class) should explain *why* it exists and *what* it does, not how it works. Good names make **interfaces** meaningful, reduce the need for comments, make code self‑documenting, and shorten onboarding for new developers.

- Prefer **nouns** for classes/interfaces: `InvoiceGenerator`, `UserRepository`
- Prefer **verbs** for functions: `calculateTotal`, `fetchUser`, `validateInput`
- Prefer **domain terms** over abbreviations

```ts
// Weak
const x = calculate(u);

// Strong
const monthlyInvoice = calculateMonthlyInvoice(user);
```

The same rule applies to method names on classes — they should describe the domain action, not a technical mechanism.

**Bad**
```ts
user.updateStatus(1);
cart.process(item);
```

**Better**
```ts
user.suspend();
cart.add(item);
```

### Expressive Conditionals

Conditionals should read like prose. Readable conditionals reduce the need for comments.

**Bad**
```ts
if (u.s === 1 && !u.d) { ... }
```

**Better**
```ts
const isSuspended = user.status === "SUSPENDED";
const hasNoDevices = user.devices.length === 0;

if (isSuspended && hasNoDevices) {
  suspendAccount(user);
}
```

### Replace Comments with Code

Comments can get out of sync with the code. Method names can too, but it's much easier to catch.

**Bad**
```ts
// Check if user is adult
if (user.age >= 18) { ... }
```

**Better**
```ts
function isAdult(user: User) {
  return user.age >= 18;
}

if (isAdult(user)) { ... }
```

This targets comments that restate the *what* — a comment that just narrates the code should become a well‑named function. It is **not** a blanket case against comments: a comment that records the *why* (design rationale, a domain rule, a warning about a non‑obvious constraint) carries information no name can, and should be kept. See Ousterhout §7, "Comments Capture What Code Cannot."

### Intention-Revealing Result Objects

For *expected* failures at a function boundary — a declined payment, validation of user‑supplied input, a lookup that may miss — prefer a shared `Result` class over throwing. Exceptions can be messy to catch across the network, and a `Result` makes the two possible outcomes explicit in the type.

| Property | Purpose |
| :--- | :--- |
| **isSuccess** | Boolean indicating the outcome. |
| **value** | The returned data (if successful). |
| **error** | A structured error message or code (if failed). |

> **Beck's Insight:** This follows the "Intention-Revealing Interface" pattern. The caller of a function knows exactly what two states to expect without checking for hidden exceptions.

> **Scope — `Result` is not a ban on `throw`.** `Result` is for *expected domain outcomes*. A *broken invariant* (a negative `Money`, a `Temperature` below absolute zero, an `end` before `start`) is a programmer error that should never happen — those still `throw` from the constructor, as in the value‑object examples below and in Ousterhout's `Money`/`Booking` guards. Rule of thumb: **throw for broken invariants, return a `Result` for expected boundary failures.**

## 2. Simplicity — Eliminate Everything That Doesn't Add Value

### Small, Focused Functions

Small functions reduce cognitive load. Each function should have one reason to change (Single Responsibility Principle).

**Bad**
```ts
function fulfillOrder(order: Order) {
  // validate
  // calculate totals
  // apply discounts
  // persist
  // send email
}
```

**Better**
```ts
function validate(order: Order) { /* ... */ }
function calculateTotals(order: Order) { /* ... */ }
function applyDiscounts(order: Order) { /* ... */ }
function persist(order: Order) { /* ... */ }
function notify(order: Order) { /* ... */ }

function fulfillOrder(order: Order) {
  validate(order);
  const totals = calculateTotals(order);
  const discounted = applyDiscounts(totals);
  persist(discounted);
  notify(discounted);
}
```

This is Beck's **Composed Method** pattern: break complex methods into smaller, well-named methods that all live at the same level of abstraction.

### Classes Should Be Small and Focused

> A class should be small enough to understand in one sitting.

The same SRP that applies to functions applies to classes.

**Bad**
```ts
class OrderProcessor {
  validate() {}
  calculateTotals() {}
  applyDiscounts() {}
  persist() {}
  notify() {}
  sendEmail() {}
  log() {}
  audit() {}
}
```

**Better**

Split responsibilities into separate classes. Each class has one job. Make `OrderProcessor` depend on abstractions, not implementations.

```ts
class OrderValidator { /* ... */ }
class PriceCalculator { /* ... */ }
class DiscountService { /* ... */ }
class OrderRepository { /* ... */ }
class NotificationService { /* ... */ }

class OrderProcessor {
  constructor(
    private readonly validator: OrderValidator,
    private readonly calculator: PriceCalculator,
    private readonly discounts: DiscountService,
    private readonly repo: OrderRepository,
    private readonly notifier: NotificationService,
  ) {}

  async fulfill(order: Order) {
    this.validator.validate(order);
    const totals = this.calculator.calculate(order);
    const discounted = this.discounts.apply(totals);
    await this.repo.save(discounted);
    await this.notifier.sendConfirmation(discounted);
  }
}
```

Notice the public interface is still a *single* method — `processor.fulfill(order)`. Callers never touch the five collaborators. That makes `OrderProcessor` a **deep module** in Ousterhout's sense (a narrow interface hiding a substantial implementation) *and* a set of small, single‑responsibility classes. The two ideas compose rather than compete: keep the **interface** narrow (Ousterhout) while decomposing the **implementation** into small injected classes (Beck). See Ousterhout §2, "Deep Modules" — it uses this exact example.

### Classes Represent Concepts, Not Containers

> A class should represent a concept in the domain, not a programming construct.

A class should model something that *exists* in your domain:

- `Cart`, not `CartManager`
- `Invoice`, not `InvoiceUtils`
- `UserService`, not `UserHelper`

This supports **DIP (Dependency Inversion)** — high-level modules depend on abstractions, not concrete manager classes.

**Bad — vague, technical, procedural**
```ts
class UserManager {
  data: any;

  save(user: any) { /* ... */ }
  load(id: string) { /* ... */ }
}
```

**Better — domain concepts**

Each class should have one reason to change (SRP). Introduce `UserRepository` as an abstraction; infrastructure depends on it, not vice versa (DIP).

```ts
class User {
  constructor(
    public readonly id: string,
    private active: boolean,
  ) {}

  deactivate() {
    this.active = false;
  }

  isActive() {
    return this.active;
  }
}

interface UserRepository {
  save(user: User): Promise<void>;
  findById(id: string): Promise<User | null>;
}
```

### Hide Data, Expose Behavior

> Data is private; behavior is public.

Objects should protect their own consistency. Encapsulation supports **LSP (Liskov Substitution)** — subclasses can't break invariants if invariants are protected behind behavior. The benefits:

- You prevent external code from mutating internal state.
- You make the class responsible for its own rules.
- You keep the API small and intention‑revealing.

**Bad**
```ts
class Cart {
  items: CartItem[] = [];
}
```

**Better**
```ts
class Cart {
  private items: CartItem[] = [];

  add(item: CartItem) {
    this.items.push(item);
  }

  get total() {
    return this.items.reduce((sum, item) => sum + item.price, 0);
  }
}
```

### Classes Protect Invariants

> Objects should protect their own consistency.

Why this matters:
- You prevent invalid states.
- You reduce the need for defensive programming everywhere else.
- You make illegal states unrepresentable.
- **LSP**: Subclasses can't break invariants if invariants are enforced internally.

```ts
class Temperature {
  constructor(private readonly celsius: number) {
    if (celsius < -273.15) {
      throw new Error("Temperature cannot be below absolute zero");
    }
  }

  toFahrenheit() {
    return this.celsius * 1.8 + 32;
  }
}
```

`Temperature` **throws** rather than returning a `Result` — and that's correct: a value below absolute zero is a broken invariant, not an expected outcome. This is the complement of the Result pattern above (throw for invariants, `Result` for expected boundary failures), and it matches Ousterhout's `Money` and `Booking` constructor guards.

### Value Objects for Domain Concepts

Beck emphasizes "State Patterns" that communicate intent. Instead of passing around raw `strings` or `numbers`, create small, immutable classes for domain concepts.

* **Examples:** `EmailAddress`, `Money`, `DateRange`, `PostalCode`.
* **Direct Access vs. Indirect Access:** When to use getters/setters versus accessing fields directly.
* **Lazy Initialization:** Handling resources only when they are needed.
* **Benefit:** Your React form and NestJS service both use the same validation logic. If the `EmailAddress` class exists, you *know* it's valid.

While Value Objects are defined by their attributes, **Entities** are defined by their **Identity**.

* **The Pattern:** Create a base `Entity<T>` class in your shared library.
* **Shared Logic:** Put methods on the entity that represent "Business Rules." For example, a `User` entity might have a method `.isActive()` or `.canPostContent()`.

### Record Patterns (DTO / Zod)

We use DTO/Zod schemas to eliminate the "I changed the backend field name but forgot to change the frontend" bug. One source of truth, validated at the boundary, removes a whole class of accidental complexity.

### Avoid Static Methods (Most of the Time)

Static methods are procedural code in disguise.

- Static methods cannot be mocked.
- They cannot hold state.
- They encourage procedural thinking.

**Bad**
```ts
class PriceUtils {
  static calculateTotal(items: Item[]) { /* ... */ }
}
```

**Better**
```ts
class PriceCalculator {
  calculate(items: Item[]) { /* ... */ }
}
```

## 3. Flexibility — Leave Room for Change Without Over-Engineering

### Prefer Composition Over Inheritance

Composition supports **OCP (Open/Closed)** — extend behavior without modifying existing classes. Why this matters:

- No fragile base classes.
- No weird prototype chains.
- No accidental override of methods.
- Easier to test and mock.

**Bad inheritance**
```ts
class AdminUser extends User {
  isAdmin = true;
}
```

**Better composition**
```ts
class User {
  constructor(
    public readonly id: string,
    public readonly role: Role,
  ) {}

  can(action: Action): boolean {
    return this.role.allows(action);
  }
}

const admin = new User("u_1", Role.Admin);
const editor = new User("u_2", Role.Editor);

if (admin.can(Action.DeletePost))  { /* allowed */ }
if (editor.can(Action.DeletePost)) { /* not allowed */ }
```

Here `User` *has-a* `Role` instead of `AdminUser is-a User`. Permission rules live on `Role`, and `User` delegates the question (`role.allows(...)`) rather than answering it itself. A few things that fall out for free:

- **Adding a new role** (`Role.Moderator`, `Role.Auditor`) needs a new role configuration, not a new subclass. The `User` class never changes — that's OCP in action.
- **A user's role can change at runtime** (a promotion, a demotion). Their *type* never has to, so no awkward `Object.setPrototypeOf` or "convert to AdminUser" hacks.
- **Behavior is configuration, not code**. Two roles can share the `User` class but answer `can(...)` differently.
- **Tests stay simple**. Pass a stub `Role` that returns whatever the test needs — no inheritance gymnastics, no mocking framework.

### Isolate Technical Details

Keep domain code free from infrastructure concerns. When the domain doesn't know about HTTP, databases, or queues, you can swap any of them out without rewriting business logic.

```ts
// Bad: domain logic mixed with HTTP
async function createUser(req: Request, res: Response) {
  const user = await db.insert(req.body);
  res.json(user);
}

// Good: separate layers (DIP and SRP work together)
class UserService {
  create(data: CreateUserDto) { /* ... */ }
}

async function createUserHandler(req: Request, res: Response) {
  const user = await userService.create(req.body);
  res.json(user);
}
```

### Domain Events

If you have complex logic (e.g., "When an Order is placed, send an email AND update inventory"), use shared **Event classes** to decouple the trigger from the reactions.

* **Pattern:** `OrderPlacedEvent`.
* **Usage:** The class contains the minimum data needed to describe what happened. New reactions can be added without touching the code that emits the event, which keeps the system flexible. It also makes Composed Methods cleaner because the "trigger" is a simple object.

## Conclusion — The Most Important Learnings

If you forget every specific pattern in this book, keep these:

1. **You write code for humans, not the compiler.** The reader is the customer. Optimize for the moment six months from now when somebody — probably you — has to understand this code under pressure.

2. **The three values reinforce each other.** Communication, Simplicity, and Flexibility aren't a menu — they pull in the same direction more often than they conflict. A name that reveals intent (Communication) usually points at a smaller responsibility (Simplicity), which is easier to swap out later (Flexibility).

3. **Names carry the most weight.** Naming is the highest-leverage skill in the book. A good name removes the need for a comment, a wrapper, or a second read. `user.suspend()` beats `user.updateStatus(1)` every time.

4. **Hide data, expose behavior.** Objects that own their invariants stop bugs at the source instead of asking every caller to be careful. Make illegal states unrepresentable.

5. **Compose, don't inherit.** Inheritance locks behavior into a type hierarchy at design time. Composition lets you change behavior at runtime, in tests, and as the domain evolves — without touching the consumer.

6. **Small units, one reason to change.** Whether it's a function, a class, or a module: if you can't summarize it in a sentence, it's doing too much.

7. **Don't build for hypothetical futures.** Flexibility means *room* to change, not *infrastructure* to change. Add the abstraction when the second use case shows up, not when you imagine it might.

The shortest summary of the whole book: **make the code obvious.** Everything else is in service of that.
