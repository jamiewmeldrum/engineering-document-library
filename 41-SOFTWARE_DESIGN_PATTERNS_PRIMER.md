# Software Design & Patterns — A Primer №41

*The level between writing a good function (№40) and structuring a whole system (№42): **how objects and modules relate.** Paradigms, the two properties that matter more than any rule, the design patterns worth knowing, and enough Domain-Driven Design to model a domain deliberately. Java-flavoured, Practiq-grounded.*

The claim this document rests on: **almost every design principle you'll meet is a mechanism for achieving low coupling and high cohesion.** SOLID, patterns, layering, dependency injection, events, DDD — all of it. If you only take one thing, take Part 2, because it lets you evaluate any design idea (including ones invented after this was written) by asking what it does to coupling and cohesion.

The second idea, which stops patterns from becoming a disease: **a pattern is a solution to a problem in a context.** Applied without the problem, it's just ceremony. The value of knowing them is partly the solutions and mostly the **vocabulary** — being able to say "that's a strategy" and have a colleague immediately understand the shape.

Contents:

- **Part 1** — paradigms: OOP, FP, and why modern code is both
- **Part 2** — coupling and cohesion: the master properties
- **Part 3** — dependency injection and inversion of control
- **Part 4** — creational patterns
- **Part 5** — structural patterns
- **Part 6** — behavioural patterns
- **Part 7** — anti-patterns
- **Part 8** — domain-driven design, the useful subset
- **Part 9** — modelling in practice
- **Part 10** — when to use what

## Pattern index — problem to pattern

| The problem | The pattern | §|
|---|---|---|
| Swap an algorithm at runtime | **Strategy** | §6.1 |
| Create objects without naming the concrete class | **Factory** | §4.1 |
| Construct something complex step by step | **Builder** | §4.2 |
| One shared instance | **Singleton** (usually: let the container do it) | §4.3 |
| Make an incompatible interface fit | **Adapter** | §5.1 |
| Add behaviour by wrapping | **Decorator** | §5.2 |
| Simplify access to a complex subsystem | **Facade** | §5.3 |
| Stand in for an expensive or remote object | **Proxy** | §5.4 |
| Notify many listeners of a change | **Observer** | §6.2 |
| Behaviour depends on internal state | **State** | §6.3 |
| Fixed algorithm, varying steps | **Template Method** | §6.4 |
| Operate over a structure without changing it | **Visitor** (or pattern matching) | §6.5 |
| Encapsulate a request as an object | **Command** | §6.6 |
| Hide persistence behind a collection-like interface | **Repository** | §8.5 |
| Wrap a primitive with meaning and validation | **Value Object** | §8.3 |

---

# Part 1 — Paradigms

## 1.1 Object-oriented programming

The core move: **bundle data with the behaviour that operates on it.** An object owns its state and exposes operations; nobody else reaches inside.

The four ideas, with what each actually buys you:

- **Encapsulation** — internals are hidden behind an interface, so you can change them without breaking callers. This is what makes change local (№40 §1).
- **Abstraction** — expose the *what*, hide the *how*. Callers depend on a concept, not a mechanism.
- **Inheritance** — share structure and behaviour via an is-a hierarchy. The most over-used of the four; see §7 and №40 §6.
- **Polymorphism** — one interface, many implementations, selected at runtime. **This is the engine of extensibility** and the mechanism underneath half the patterns below.

## 1.2 Functional programming

The core move: **compute with pure functions over immutable data**, avoiding shared mutable state.

- **Pure functions** — same input, same output, no side effects. Trivially testable, trivially parallelisable, trivially cacheable. You can reason about them locally, which is the whole point.
- **Immutability** — data doesn't change; transformations produce new values. Removes an entire class of bugs (aliasing, unexpected mutation) and is a genuine *concurrency* technique (№12 §7.1).
- **Higher-order functions** — functions as values, passed and returned. `Comparator`, `Function`, stream operations.
- **Declarative composition** — describe the transformation, not the loop mechanics.

## 1.3 Modern Java is both, and that's correct

The paradigm war is over and the answer was "both." Idiomatic modern Java is **object-oriented in the large, functional in the small**: objects and interfaces define the structure and boundaries; within methods, streams, lambdas, records and immutability do the work (№10 §2, §6, §7).

The practical synthesis:

| Use OOP for | Use FP for |
|---|---|
| modelling the domain (entities with behaviour) | transformations and calculations |
| defining boundaries and seams (interfaces) | data pipelines (streams) |
| managing lifecycle and identity | value objects (records) |
| polymorphic dispatch on type | avoiding shared mutable state |

**Push side effects to the edges.** The valuable idea FP contributes to ordinary code: keep the core logic pure and testable, and confine I/O, mutation and randomness to a thin outer layer. That single habit makes most code dramatically easier to test (№44 §7).

---

# Part 2 — Coupling and cohesion

The master properties. Everything else is instrumentation.

## 2.1 Coupling — how much things depend on each other

**Low coupling is good.** Tightly coupled code means a change here breaks something over there, and you can't understand, test or reuse one piece without dragging in the others.

Degrees, worst to best:

| Coupling | Means | Example |
|---|---|---|
| **Content** (worst) | one module reaches into another's internals | reflection into private fields |
| **Common** | shared global mutable state | a static mutable registry |
| **Control** | one passes a flag telling the other *how* to behave | `render(true)` (№40 §3.4) |
| **Stamp** | passing a whole object when a field would do | passing `Question` to get its id |
| **Data** (best) | passing exactly the data needed | `approve(questionId)` |

Also worth naming: **temporal coupling** (A must be called before B, unenforced by the types — a classic source of bugs) and **afferent/efferent coupling** (how many modules depend *on* you, versus how many you depend *on*). High afferent coupling means you must be stable; high efferent coupling means you're fragile.

## 2.2 Cohesion — how focused a module is

**High cohesion is good.** A cohesive class's parts all serve one purpose, so the class has one reason to change (SRP, №40 §5.2) and its name is easy to write.

Degrees, worst to best: **coincidental** (grouped for no reason — `Utils`), **logical** (similar category, different jobs), **temporal** (things done at the same time — `startup()`), **procedural**/**communicational** (steps in a sequence / operating on the same data), **functional** (best — everything contributes to one well-defined task).

The practical test: **if a subset of a class's methods uses only a subset of its fields, you have two classes.**

## 2.3 The tension, and the balance

They pull against each other. Maximally reduce coupling by putting everything in one class (perfectly decoupled from everything, hopelessly incohesive). Maximally increase cohesion by splitting into hundreds of tiny classes (each perfectly focused, all densely coupled). The balance point is: **things that change together belong together; things that change independently belong apart.**

That sentence is a complete design heuristic. It explains package structure, service boundaries (№42 §4), microservice splits, and why Practiq's `Concept`/`Question`/`SpecSection` group as they do.

> **The tell — the master question:** for any design decision, ask *"does this reduce coupling or increase cohesion?"* If neither, it's ceremony. If it does one at the cost of the other, decide which axis your likely changes fall along.

---

# Part 3 — Dependency injection and IoC

## 3.1 The problem

```java
class QuestionService {
    private final QuestionRepository repo = new HibernateQuestionRepository();  // ← the problem
}
```

This class *constructs* its dependency, which means: you can't test it without a database, you can't swap the implementation, and the high-level policy (approval logic) now depends on a low-level detail (Hibernate). It's a Dependency Inversion violation (№40 §5.2) and it's the single most common structural mistake in application code.

## 3.2 The fix

**Dependency Injection:** the dependency is supplied from outside.

```java
@Singleton
class QuestionService {
    private final QuestionRepository repo;                  // an interface

    QuestionService(QuestionRepository repo) { this.repo = repo; }   // constructor injection
}
```

**Inversion of Control** is the broader principle: the framework calls you rather than you calling it; something else decides what gets wired to what. DI is the most common form of IoC.

Prefer **constructor injection**: dependencies are explicit, the object is immutable and always valid, and an overlong constructor is a visible SRP warning. Field injection hides dependencies and makes the class untestable without a container; setter injection permits half-constructed objects.

## 3.3 What it actually buys

Testability (swap a double, №44), swappability, explicit dependencies (the constructor is a manifest of what this class needs), and lifecycle management. **Micronaut does this at compile time** — no runtime reflection, errors caught at build (№20 §2.12), which is the framework-level payoff of designing this way.

**One caution that bites:** container-managed beans are **singletons by default**, so a mutable field on a service is shared across every request thread. That's the single most common concurrency bug in a web app (№12 §7.2). Keep injected services **stateless**.

---

# Part 4 — Creational patterns

*How objects get made.*

## 4.1 Factory

**Problem:** the caller shouldn't know which concrete class it's getting, or construction requires logic.

```java
interface QuestionParser { Question parse(String raw); }

class QuestionParserFactory {
    static QuestionParser forFormat(SourceFormat format) {
        return switch (format) {
            case PDF      -> new PdfQuestionParser();
            case MARKDOWN -> new MarkdownQuestionParser();
            case JSON     -> new JsonQuestionParser();
        };
    }
}
```

The caller depends only on the interface. Variants: **static factory method** (`List.of()`, `Optional.of()` — often better than a constructor because it can be named, cached and return a subtype), **factory method** (subclasses decide the class), **abstract factory** (families of related objects — rarer, heavier).

## 4.2 Builder

**Problem:** many constructor parameters, several optional, and telescoping constructors are unreadable.

```java
Question q = Question.builder()
        .stem("What is Newton's second law?")
        .difficulty(3)
        .conceptId(42L)
        .build();                       // validation happens here
```

Readable at the call site, immutable result, validation in one place. You see it constantly: `HttpRequest.newBuilder()`, `Stream.Builder`. **Records with compact constructors** (№10 §2.2) cover many cases more cheaply — reach for a builder when there are genuinely many optional fields.

## 4.3 Singleton

**Problem:** exactly one instance should exist.

Hand-rolled singletons (`private static final INSTANCE`, or worse, double-checked locking) are a known anti-pattern: they're global mutable state, they hide dependencies, and they make testing miserable because you can't substitute them. **The right answer in a framework application is to let the DI container manage the scope** (`@Singleton`) — you get one instance *and* it's still injected, so it's still swappable and testable. If you must hand-roll, an `enum` singleton is the safe Java idiom.

## 4.4 Prototype and Object Pool

**Prototype** — copy an existing object rather than constructing one (useful when construction is expensive or the config is complex). **Object pool** — reuse expensive objects rather than recreating them; you meet it as connection pooling (№20 §1.6), rarely writing it yourself.

---

# Part 5 — Structural patterns

*How objects compose.*

## 5.1 Adapter

**Problem:** you have an interface you need and a class that does the job with the wrong shape.

```java
// Their API                            // What our domain wants
class PdfPlumberClient {                interface QuestionExtractor {
    JsonNode extract(byte[] pdf);           List<Question> extract(Document doc);
}                                       }

class PdfPlumberAdapter implements QuestionExtractor {
    private final PdfPlumberClient client;
    public List<Question> extract(Document doc) {
        return map(client.extract(doc.bytes()));      // translate both ways
    }
}
```

**This is the most practically valuable pattern in the list** for a working engineer: it's how you keep third-party libraries at arm's length. Your domain depends on *your* interface; if the vendor changes or you swap them out, one adapter changes and nothing else does. It's also the "ports and adapters" of hexagonal architecture (№42 §3).

## 5.2 Decorator

**Problem:** add behaviour to an object without changing its class or subclassing every combination.

```java
QuestionExtractor extractor =
    new CachingExtractor(
        new RetryingExtractor(
            new LoggingExtractor(new PdfPlumberAdapter(client))));
```

Each wrapper implements the same interface, adds one concern, and delegates. You compose behaviours at runtime instead of exploding a class hierarchy (`CachingRetryingLoggingExtractor`...). Java's I/O streams are the canonical example: `new BufferedReader(new InputStreamReader(in))`.

## 5.3 Facade

**Problem:** a subsystem is complicated and callers only need a simple slice.

A facade offers one coherent entry point over several collaborating classes. Your application **service layer** is often a facade over repositories, validators and mappers. It reduces coupling — callers depend on one simple interface rather than six internal classes.

## 5.4 Proxy

**Problem:** you need to control access to an object — because it's expensive, remote, or needs guarding.

Same interface, stands in for the real thing: **lazy loading** (a Hibernate proxy that doesn't fetch until touched — №20 §2.8), **remote** (a stub for a service across a network), **protection** (an access check before delegating), **caching**. Framework AOP (transactions, security, metrics) is implemented with proxies — which is why `@Transactional` on a self-invoked method silently does nothing: the call never leaves the object, so it never passes through the proxy. That's a genuinely common real-world bug.

## 5.5 Composite and Bridge

**Composite** — treat individual objects and compositions of them uniformly through one interface (a tree: files and folders, or a UI component hierarchy). **Bridge** — separate an abstraction from its implementation so both can vary independently (JDBC's `Connection` interface versus the driver, №20 §1.4).

---

# Part 6 — Behavioural patterns

*How objects collaborate and distribute responsibility.*

## 6.1 Strategy

**Problem:** several interchangeable algorithms, chosen at runtime.

```java
interface DifficultyScorer { int score(Question q); }

class QuestionService {
    private final DifficultyScorer scorer;                 // injected — swappable
    QuestionService(DifficultyScorer scorer) { this.scorer = scorer; }
}
```

In modern Java a strategy is often just a **lambda** — a `Comparator` is a strategy; a `Function` passed to a method is a strategy. **Your `QuerySpecification` is a strategy**: each spec is an interchangeable implementation of "produce a predicate," composed at runtime (№20 §3.5). Strategy is the standard cure for a growing `switch` (Open/Closed, №40 §5.2).

## 6.2 Observer

**Problem:** many objects need to react when something happens, and the subject shouldn't know who they are.

The publisher holds a list of subscribers and notifies them. Modern implementations rarely hand-roll it: framework **application events**, message queues (№31 §7), and reactive streams are all observer. It gives excellent decoupling — the publisher knows nothing about the consumers — at the cost of an **implicit control flow** that's harder to trace when debugging. That trade recurs at system scale as event-driven architecture (№42 §5).

## 6.3 State

**Problem:** an object's behaviour depends on its state, and you have `switch (status)` scattered everywhere.

Encapsulate each state as an object handling the operations legal in that state, and transitions become explicit rather than implied by a tangle of conditionals. Practiq's `DRAFT → PENDING → APPROVED` review workflow is a state machine; when the rules about what's legal in each state grow, this pattern (or an explicit state machine) beats accumulating conditionals. Modern Java alternative: **sealed types + exhaustive pattern matching** (№10 §2.5), where the compiler enforces that you handled every state.

## 6.4 Template Method

**Problem:** several processes share a skeleton but differ in steps.

A base class defines the algorithm and calls abstract hook methods that subclasses implement. Effective but inheritance-based, so it's rigid; the composition-based alternative is to pass the varying steps in as strategies/lambdas, which is usually better (№40 §6).

## 6.5 Visitor

**Problem:** you need to perform many different operations over a fixed object structure without putting all those operations in the classes themselves.

Genuinely useful for AST traversal and similar, but verbose and awkward — and largely obsoleted in modern Java by **pattern matching over sealed hierarchies** (№10 §2.5), which achieves the same "operate over a closed set of types" goal with a fraction of the ceremony and compiler-checked exhaustiveness.

## 6.6 Command, Chain of Responsibility, Iterator, Mediator, Memento

**Command** — encapsulate a request as an object, enabling queuing, logging, undo and retry (every job on a message queue is a command). **Chain of Responsibility** — pass a request along a chain until one handles it (servlet filters, middleware, interceptors). **Iterator** — traverse a collection without exposing its structure (built into the language). **Mediator** — centralise communication between components so they don't reference each other directly. **Memento** — capture and restore state (undo).

---

# Part 7 — Anti-patterns

Recognising these is as valuable as knowing the patterns:

| Anti-pattern | Is | Why it hurts |
|---|---|---|
| **God Object** | one class that knows and does everything | zero cohesion, everything couples to it |
| **Anemic Domain Model** | entities are bare data; all logic sits in services | you've written procedural code in OO clothing; behaviour scatters and duplicates |
| **Spaghetti** | no discernible structure | can't reason locally |
| **Lava Flow** | dead code nobody dares delete | grows forever, obscures the live paths |
| **Golden Hammer** | one favourite tool applied to everything | wrong tool, ignored trade-offs |
| **Premature Optimisation** | complexity for unmeasured performance | pays cost, buys nothing |
| **Poltergeist** | classes that just pass messages along | indirection with no value |
| **Big Ball of Mud** | the architecture is "whatever happened" | the default end state without deliberate design |
| **Pattern Fever** | patterns applied without the problem | ceremony, indirection, six files per behaviour |

**Anemic domain model** deserves the extra emphasis because it's so common and so easy to fall into with an ORM. If `Question` is nothing but fields and accessors while `QuestionService` holds every rule, you get logic duplicated across services, invariants nobody enforces, and objects that can be put into invalid states. The fix is "tell, don't ask" (№40 §5.1): put the behaviour with the data — `question.approve()` rather than `service.setStatusToApproved(question)`.

---

# Part 8 — Domain-Driven Design, the useful subset

DDD is a large body of work; this is the part that pays for itself immediately.

## 8.1 Ubiquitous language

**Use the domain's words in the code, exactly and consistently.** If educators say "spec section," the class is `SpecSection` — not `SyllabusItem` in one place and `CurriculumNode` in another. Every translation between the language of the domain and the language of the code is a place misunderstandings hide. This sounds trivial and is genuinely one of the highest-value practices in software.

## 8.2 Entities

Objects with **identity that persists through change**. A `Question` is the same question after you edit its text — its identity is its id, not its attributes. Equality is by identity, not value (which is exactly why entity `equals`/`hashCode` is tricky under an ORM, №21 §11.7).

## 8.3 Value objects

Objects defined **entirely by their attributes**, with no identity: `Difficulty`, `Money`, `EmailAddress`, `DateRange`. They should be **immutable**, compared by value, and — the key benefit — **self-validating**:

```java
public record Difficulty(int value) {
    public Difficulty {
        if (value < 1 || value > 5)
            throw new IllegalArgumentException("difficulty must be 1-5, got " + value);
    }
    public boolean isHard() { return value >= 4; }
}
```

Now an invalid difficulty **cannot exist anywhere in the system**, and the type carries meaning that `int` doesn't. This is the cure for Primitive Obsession (№40 §8.1), and it's the cheapest large improvement available to most codebases. In JPA these map naturally to `@Embeddable` (№21 §11.3).

## 8.4 Aggregates

A cluster of entities and value objects treated as **one consistency boundary**, with a single **aggregate root** as the only entry point. Rules: outside code holds a reference only to the root; invariants within the aggregate are always true; and **one transaction modifies one aggregate**.

That last rule is the practically important one — it tells you where transaction boundaries belong and, at system scale, where services could split (№42 §4, №31 §8.2). In Practiq, `Question` with its `QuestionBody` and options is a plausible aggregate: you never manipulate a body independently of its question.

## 8.5 Repositories, services, factories

**Repository** — a collection-like interface for retrieving and persisting aggregate roots, hiding the persistence mechanism. Your `QuestionRepository` is exactly this, and its value is that the domain depends on an interface, not on Hibernate.

**Domain service** — behaviour that genuinely doesn't belong to a single entity (an operation spanning several aggregates). Use sparingly; the temptation to put *everything* in services is what produces the anemic model (§7).

**Domain event** — something meaningful that happened (`QuestionApproved`), which other parts react to. Decouples the doer from the reactors (§6.2, and the outbox pattern, №31 §8.2).

## 8.6 Bounded contexts

A model is only valid within a boundary. "Question" means something different to the authoring team than to the analytics team — and forcing one universal model to serve both produces a bloated compromise that serves neither. **Bounded contexts** let each have its own model with an explicit translation between them. This is the DDD concept that most directly informs service boundaries (№42 §4): a good service boundary is usually a bounded context boundary.

---

# Part 9 — Modelling in practice

A short method for turning a domain into a design:

1. **Learn the language.** Talk to whoever knows the domain; write down their nouns and verbs. Those are your candidate types and operations.
2. **Find the invariants.** "A question can't be approved without a concept." "Difficulty is 1–5." Invariants tell you where value objects and aggregate boundaries go — each invariant must live inside something that can enforce it.
3. **Distinguish identity from value.** Does this thing stay itself when its data changes? Entity. Is it defined purely by its data? Value object.
4. **Push behaviour to the data.** For each rule, ask which object owns the information to enforce it — the rule belongs there (§7, №40 §5.1).
5. **Draw the consistency boundaries.** What must be true together, atomically? That's an aggregate, and a transaction.
6. **Keep the domain free of infrastructure.** No JPA annotations in your domain logic if you can help it, no HTTP concepts, no framework types in the core. The domain shouldn't know it's persisted or served over the web — that's the dependency rule (№42 §3).

Point 6 is a genuine trade-off in practice: full purity means mapping between domain objects and persistence entities, which is real work. Many pragmatic Java codebases (including, sensibly, Practiq) let JPA entities *be* the domain model and accept the coupling. That's defensible at your scale — but know you're making the trade, and keep the *logic* in the entities rather than letting them go anemic.

---

# Part 10 — When to use what

**A. Inheritance or composition?** Tell → inheritance: a genuine, stable is-a where every parent operation makes sense on the child. Tell → composition: everything else. Default: **composition** (№40 §6).

**B. Interface or concrete class?** Tell → interface: multiple implementations exist or are likely, you need a test seam, or it's a boundary to something external. Tell → concrete: one implementation, internal, stable. Default: **interfaces at boundaries, concrete classes inside.** Don't create an interface per class reflexively.

**C. Pattern or plain code?** Tell → pattern: you have the problem the pattern solves, and the name will help readers. Tell → plain: you'd be adding indirection for a hypothetical need. Default: **plain code until the problem shows up** (§7, Pattern Fever).

**D. Strategy, or a switch?** Tell → strategy: the set of behaviours grows, or they're chosen/injected at runtime. Tell → switch: a small, closed, stable set — and with sealed types you get compiler-checked exhaustiveness (№10 §2.5). Default: **sealed + switch for closed sets; strategy for open ones.**

**E. Entity or value object?** Tell → entity: it has identity that survives change. Tell → value object: it's defined by its attributes. Default: **value object where possible** — immutable, self-validating, no identity to manage.

**F. Domain service or entity method?** Tell → entity: the behaviour concerns one entity's own data. Tell → domain service: it genuinely spans several aggregates. Default: **entity method** — services are where anemic models come from.

**G. Rich or anemic domain model?** Tell → rich: real business rules and invariants exist. Tell → anemic (a DTO): it's genuinely just data in transit — an API response, a projection. Default: **rich for the domain, records for the wire.**

---

# How to expand this

- *Related:* №40 Clean Code (the same thinking at function and class level — SOLID lives there); №42 Architecture (the same thinking at system level — layering, hexagonal, service boundaries); №44 Testing (testability is the fastest feedback on whether your coupling is right); №21 §11 (Practiq's entity model with justified decisions); №10 §2 (records, sealed types and pattern matching as language-level answers to several patterns here).
- *Candidates for deeper treatment:* **the full GoF catalogue** with Java examples for the patterns skipped here; **DDD properly** (strategic design, context mapping, event storming); **a Practiq domain-model deep-dive** applying Part 9's method to your actual domain from scratch.

*Stable material, written from knowledge — paradigms, coupling/cohesion, the GoF patterns and DDD's core don't drift. The canonical sources: Gamma et al.* Design Patterns *for the catalogue, Evans'* Domain-Driven Design *and Vernon's* Implementing DDD *for the modelling.*
