# Book: Domain-Driven Design (Eric Evans)

> **Scope:** notes on Part I, *Putting the Domain Model to Work* (Chapters 1–4). These chapters are about the *philosophy* of modeling — why a model matters, how a team builds one together, and how it binds to code. The well-known tactical patterns (Entities, Value Objects, Aggregates, Repositories, Factories) and strategic patterns (Bounded Context, Context Map) come in later chapters and aren't covered here yet.

The central thesis of these chapters: **the heart of software for a complex domain is a *model* — a rigorously selected, organized abstraction of domain knowledge — and the model's value comes entirely from how tightly it is bound to a shared language and to the running code.** A model that lives only in a diagram, or only in an analyst's head, is worthless. Three moves make modeling effective:

1. **Bind the model to the implementation** — the same concepts drive the design and the code.
2. **Cultivate a Ubiquitous Language** rooted in the model and spoken by developers *and* domain experts.
3. **Crunch knowledge continuously** — modeling is an ongoing conversation, not an up-front phase.

These line up closely with the other two books. Where Beck argues *code is communication* and Ousterhout argues *fight complexity by hiding it behind deep modules*, Evans argues *the code should communicate a domain model, and that model is the thing worth getting right.* (Full correlation table at the end.)

---

## 1. Ingredients of Effective Modeling

> **Anecdote — the Monty Python editor.** The editor judged the clips on whether they were technically *clean* — nobody walking into shot, good lighting — not on whether the comedy in the shot actually landed.

The lesson for modeling: a model (or a codebase) can be technically tidy and still miss the point. Optimizing for surface correctness while losing the thing that makes the domain *work* is the same trap Ousterhout warns about when a "too complete" UML diagram buries meaning in detail — you can't see the forest for the trees (see §2 below).

### What made modeling succeed

1. **Binding the model and the implementation.**
   - Build a prototype early.
   - Expand it — or rebuild it — as you gain new knowledge.
2. **Cultivating a language based on the model.** Keep track of the names used in the software and in the real world, and how they map to each other.
3. **Developing a knowledge-rich model.** The objects had behavior and enforced rules. The model wasn't just a data schema; it was integral to solving a complex problem — it captured knowledge of various kinds.
   - → Exactly Beck's *Hide Data, Expose Behavior* and *Classes Protect Invariants*, and Ousterhout's *Information Hiding* (§5): a class models a domain concept and owns its rules, instead of being a bag of public fields.
4. **Distilling the model.** Concepts were added as the model matured, but — equally important — dropped when they stopped being useful or central. When an unneeded concept was entangled with a needed one, a new model was found that separated the essential concept so the rest could be discarded.
   - → Ousterhout §4: *a module should become deeper over time, not wider.* Distillation **is** deepening the model.
5. **Brainstorming and experimenting.**

### Why waterfall fails

> In the old waterfall method, the business experts talk to the analysts, and analysts digest and abstract and pass the result along to the programmers, who code the software. This approach fails because it completely lacks feedback.

> If programmers are not interested in the domain, they learn only what the application should do, not the principles behind it.

**Knowledge crunching** is the alternative — a continuous loop instead of a one-way handoff:

> The interaction between team members changes as all members crunch the model together. The constant refinement of the domain model forces the developers to learn the important principles of the business they are assisting, rather than to produce functions mechanically.

> The early work was essential. Key model elements were retained, but more important, that work set in motion the process of knowledge crunching that made all subsequent work effective.

→ This is the same loop Ousterhout calls **Continuous Refinement** (§4): *"design is a continuous conversation between developers, domain experts, and the evolving business."* Evans names the *input* to that loop (domain knowledge); Ousterhout names the *output* (progressively deeper modules). Same activity from two angles.

### Communicating intent — make the rule a named concept

The book's shipping example: overbooking is allowed up to 110% of a voyage's capacity. Buried inline, that rule is invisible:

```ts
// The rule has no name, can't be reused, and can't be vetted by a domain expert
if (voyage.bookedCargoSize() + cargo.size() > voyage.capacity() * 1.1) {
  // reject the booking
}
```

Promote it to a domain object:

```ts
if (!overbookingPolicy.isAllowed(cargo, voyage)) {
  // reject the booking
}
```

- The rule becomes a *thing with a name* (`OverbookingPolicy`) that a domain expert can recognize, discuss, and validate.
- The `*Policy` suffix signals "this is a business rule," and the rule now lives in one place — found, reused, documented, and tested once.

→ Three-way correlation:
- **Beck, *Expressive Conditionals* / *Names Reveal Purpose*:** the inline boolean becomes a named, intention-revealing call.
- **Ousterhout §3, *Strategic Programming*:** an inline conditional with a magic `1.1` is the *tactical* fix; a `Policy` object you can swap and extend is the *strategic* one (cf. his `PriceStrategy`). The 110% / `1.1` is also a textbook case for a named constant or a *why* comment — see Ousterhout's `VAT_RATE` (§3.2, §7).

---

## 2. Communication and the Use of Language — the Ubiquitous Language

> With a conscious effort by the team, the domain model can provide the backbone for that common language.

The model is not just a design artifact; it's the **vocabulary the whole team speaks** — in conversation, in documents, and in code.

> Try speaking out loud. Listen to [the] jargon/technical words used. See if we are talking about the same things.

> When people are talking, they naturally discover differences in interpretation and the meaning of their words, and they naturally resolve those differences. They find rough spots in the language and smooth them out.

> Play with the model as you talk about the system. Describe scenarios out loud using the elements and interactions of the model, combining concepts in ways allowed by the model. Find easier ways to say what you need to say, and then take those new ideas back down to the diagrams and code.

> When domain experts use this LANGUAGE in discussions with developers or among themselves, they quickly discover areas where the model is inadequate for their needs or seems wrong to them. The domain experts (with the help of the developers) will also find areas where the precision of the model-based language exposes contradictions or vagueness in their thinking.

One constraint: extensions to the language are fine, but —

> These dialects should not contain alternative vocabularies for the same domain that reflect distinct models.

→ This is the domain-level version of Beck's entire **Communication** value. Beck: *names are the highest-leverage skill; `user.suspend()` beats `user.updateStatus(1)`.* Evans adds the missing constraint: the name must come from — and feed back into — the **shared domain language**, not just be locally clear. A name that reads well to a developer but isn't a term the domain expert uses is a missed chance to align the model. It's also Ousterhout's **Designing for Obviousness** (§6) raised to the team level: code should read like prose *to everyone*, the business included.

### On diagrams

UML may not be the best tool, but a page / whiteboard with the key entities and relationships keeps people focused.

> Everyone will share a view of the relationships between the objects and, significantly, the objects' names. The spoken discussion can be more effective with this aid.

The failure mode is over-completeness:

> Too complete because people feel they have to put all the objects that they are going to code into a modeling tool. With all that detail, no one can see the forest for the trees.

> A UML diagram cannot convey two of the most important aspects of a model: the meaning of the concepts it represents, and what the objects are meant to do.

> Diagrams are a means of communication and explanation, and they facilitate brainstorming. They serve these ends best if they are minimal. Comprehensive diagrams of the entire object model fail to communicate or explain; they overwhelm the reader with detail and they lack meaning.

> The vital detail about the design is captured in the code. A well-written implementation should be transparent, revealing the model underlying it.

→ "Minimal diagram, meaning lives in the code" is Ousterhout's **deep module** idea applied to documentation: the diagram is a *narrow interface* (the few concepts and relationships that matter); the *deep implementation* is the code. And "a well-written implementation should be transparent, revealing the model" is Beck's thesis verbatim — **code is communication** — and Ousterhout §6, code that *communicates intent*. (Where a diagram or constant genuinely can't carry the *why*, that's the job of a comment — Ousterhout §7.)

---

## 3. Binding Model and Implementation — Model-Driven Design

> The pure analysis model even falls short of its primary goal of understanding the domain, because crucial discoveries always emerge during the design/implementation effort.

A model you can't — or don't — implement faithfully is a fiction. Insisting that *the* model is the one expressed in code is what keeps it honest.

### Object design that expresses the model

> The real breakthrough of object design comes when the code expresses the concepts of a model.

> Of course, if there were only one operation (as in the example), the script-based approach might be just as practical. But in reality, there were 20 or more. The MODEL-DRIVEN DESIGN scales easily and can include constraints on combining rules and other enhancements.

→ "One operation → a script is fine; 20+ → you need a model" is precisely Ousterhout's **tactical vs strategic** (§3) and Beck's *don't build for hypothetical futures*: the model earns its complexity once the domain is genuinely complex, not before.

### Don't fake a model in the UI

> Suppose a user tries to store a Favorite and types the following name for it: "Laziness: The Secret to Happiness". An error message will say: "A filename cannot contain any of the following characters: \ / : * ? " < > | ". What filename? On the other hand, if the Web page title already contains an illegal character, Internet Explorer will just quietly strip it out. The loss of data may be benign in this case, but not what the user would have expected. Quietly changing data is completely unacceptable in most applications.

> But trying to create in the UI an illusion of a model other than the domain model will cause confusion unless the illusion is perfect.

The point: leaking an implementation concept (a *filename*) into a place the user thinks of as a *title* breaks the model — and silently "fixing" the mismatch is worse than failing loudly.

→ Ousterhout's **information leakage** (§1.1) and **make illegal states unrepresentable** (§6): the model should make the bad state impossible, not paper over it after the fact. Beck's *Classes Protect Invariants* says the same at the object level — reject the invalid input at the boundary instead of mangling it. (And note the throw-vs-`Result` rule both books share: an illegal character in user-supplied input is an *expected boundary failure* → return a `Result`/validation error, don't silently mutate and don't crash.)

---

## 4. Isolating the Domain — Layered Architecture

Chapter 4 is about keeping the model from dissolving into the plumbing. Domain logic tends to get tangled with UI, persistence, and messaging — and once that happens you can no longer reason about the model on its own.

The classic layering:

| Layer | Responsibility |
| :--- | :--- |
| **User Interface / Presentation** | Show information; interpret the user's commands. |
| **Application** | Thin coordination of tasks; **no business rules**. Knows *what* to do, delegates *how*. |
| **Domain** | The model — concepts, rules, state. **The heart of the software.** |
| **Infrastructure** | Persistence, messaging, external services — the technical means. |

The rule that makes it work: **dependencies point inward, toward the domain.** The domain layer knows nothing about the UI, the database, or the framework.

→ This is the **most direct correlation across all three books:**
- **Beck, *Isolate Technical Details*:** "Keep domain code free from infrastructure concerns... swap any of them out without rewriting business logic." His `UserService` vs the raw HTTP-handler example *is* Evans's Application/Domain split.
- **Ousterhout §8, *Designing for Change* (DIP):** the domain depends on an *interface* (`NotificationSender`, `PaymentMethod`, `UserRepository`); infrastructure implements it. The arrow points at the abstraction, so email/SMS/Slack/Postgres are swappable and the domain never changes.

Worth noticing: Beck's `Money`, `Booking`, and `EmailAddress` value objects, his `Entity<T>` base class, and his `OrderPlacedEvent` are themselves **DDD tactical patterns** (Value Object, Entity, Domain Event). Part I sets up the philosophy that those later-chapter patterns implement — the two books are reading the same map from opposite ends.

### The Smart UI "Anti-Pattern"

Evans is careful here: putting all the logic directly in the UI (data-bound forms, business rules in event handlers) is a *legitimate* choice for a simple application with a short life and a junior team — it's fast and demands little skill. It becomes an **anti-pattern** the moment the domain is complex: rules get copy-pasted across screens, there's no model to reuse or reason about, and behavior can't be tested away from the UI.

→ Ousterhout's **tactical programming** (§3) and a **shallow module** (§2) by another name: quick now, but every new rule leaks across the UI and complexity compounds. Beck's *Avoid Static Methods* / "procedural code in disguise" warning points at the same smell — logic with no domain object to live on.

---

## Correlations at a Glance

| Evans — *Domain-Driven Design* (Ch 1–4) | Beck — *Implementation Patterns* | Ousterhout — *A Philosophy of Software Design* |
| :--- | :--- | :--- |
| Ubiquitous Language | Names Reveal Purpose; Code Communicates Intent | §6 Designing for Obviousness |
| Knowledge-rich model (behavior + rules) | Hide Data, Expose Behavior; Classes Protect Invariants; Value Objects | §5 Information Hiding |
| Communicating intent via Policy objects | Expressive Conditionals | §3 Strategic Programming (Strategy pattern) |
| Knowledge crunching / distillation | The three values reinforce each other | §4 Continuous Refinement ("continuous conversation") |
| Model-Driven Design (code expresses the model) | Code is communication | §6 Code communicates intent |
| Don't fake a model in the UI | Classes Protect Invariants; throw-vs-`Result` | §1.1 Information leakage; §6 illegal states unrepresentable |
| Isolating the Domain (layers) | Isolate Technical Details | §8 Designing for Change (DIP) |
| Smart UI anti-pattern | Avoid procedural code in disguise | §2 Shallow modules; §3 tactical programming |

### The One-Line Synthesis

All three books argue the same thing from different entry points:

- **Beck:** write code that *communicates* — to humans, in their language.
- **Ousterhout:** hide complexity behind *deep modules* so the system stays comprehensible.
- **Evans:** the thing the code should communicate, and the thing worth hiding complexity *for*, is a **shared model of the domain.**

Beck and Ousterhout tell you *how* to build a clear, deep module. Evans tells you *what it should be a module of* — and insists the answer is discovered in continuous conversation with the people who understand the problem.
