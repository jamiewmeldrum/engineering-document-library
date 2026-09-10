# Frameworks & Dependency Injection — A Primer №14

*Demystifying the magic. What a framework actually does when it "wires everything up", how `@Transactional` works, why a self-invoked annotated method silently does nothing, and the genuine architectural difference between Micronaut's compile-time approach and Spring's runtime one. Practiq-grounded — you build on Micronaut, so the compile-time story is the one that pays off.*

The idea that turns frameworks from magic into mechanism: **a library is code you call; a framework is code that calls you.** That inversion — you write pieces, something else decides when they run and what they receive — is what "Inversion of Control" names, and it's the single distinction underneath dependency injection, lifecycle callbacks, interceptors and configuration binding.

The second idea, which is the practical payoff of this document: **almost all framework "magic" is one of three things** — an object graph assembled from metadata, a proxy wrapping your object, or code generated at build time. Once you can identify which of the three is happening, the surprising behaviours stop being surprising.

Contents:

- **Part 1** — what a framework is
- **Part 2** — dependency injection
- **Part 3** — the container and bean lifecycle
- **Part 4** — scopes, and the singleton trap
- **Part 5** — compile-time vs runtime: Micronaut and Spring
- **Part 6** — AOP, proxies and interceptors
- **Part 7** — configuration
- **Part 8** — testing with a framework
- **Part 9** — native image, and why the architecture matters
- **Part 10** — when to use what

## Confusion index

| The surprise | The cause | §|
|---|---|---|
| `@Transactional` does nothing on an internal call | proxy self-invocation | §6.4 |
| A field on a service behaves erratically under load | singleton with mutable state | §4.3 |
| "No bean of type X" at startup | missing/ambiguous registration | §3.4 |
| Circular dependency error | A needs B needs A | §3.5 |
| Works in a test, fails in production | different active configuration | §7.3 |
| Slow startup | runtime classpath scanning and reflection | §5.2 |
| Native image fails on reflection | runtime reflection can't be seen at build | §9 |
| An `@Value` is null | injected after construction, or wrong key | §7.2 |
| Two beans of the same type, wrong one injected | no qualifier | §3.4 |

---

# Part 1 — What a framework is

## 1.1 Library versus framework

```java
// Library — YOU control the flow
String json = objectMapper.writeValueAsString(question);

// Framework — IT controls the flow; you supply the pieces
@Controller("/api/v1/questions")
class QuestionController {
    @Get                                  // the framework decides when this runs
    List<QuestionSummary> list(@QueryValue @Nullable Long conceptId) { ... }
}
```

Nothing in your code calls `list()`. The framework listens on a socket, parses HTTP, matches a route, converts parameters, invokes your method, serialises the return value, and handles the exception if one escapes. You wrote the business logic; it wrote everything else.

## 1.2 What you get, and what you pay

**Get:** no boilerplate for the 90% that's the same in every application; consistent solutions to cross-cutting concerns (№42 §6); community conventions and integrations; and a lot of correctness you'd otherwise implement badly.

**Pay:** a learning curve; indirection that makes control flow harder to follow; magic that's baffling when it misbehaves; lock-in; and version upgrades that occasionally break things. The trade is nearly always worth it for application development — but the cost is *specifically* the "I don't understand what just happened" tax, which is what this document is written to reduce.

---

# Part 2 — Dependency injection

## 2.1 The problem

```java
class QuestionService {
    private final QuestionRepository repo = new HibernateQuestionRepository(entityManager);
    private final NotificationService notifier = new EmailNotificationService(smtpConfig);
}
```

This class constructs its own collaborators, so it: can't be tested without a database and an SMTP server; can't have either swapped; must know how to build things unrelated to its purpose; and couples high-level policy to low-level detail (a Dependency Inversion violation, №40 §5.2).

## 2.2 The fix

**Dependencies are supplied from outside.**

```java
@Singleton
class QuestionService {
    private final QuestionRepository repo;
    private final NotificationService notifier;

    QuestionService(QuestionRepository repo, NotificationService notifier) {
        this.repo = repo;
        this.notifier = notifier;
    }
}
```

The class now declares what it *needs*, not how to *get* it. Something else — the container — decides what to supply.

**Inversion of Control** is the general principle (the framework calls you); **dependency injection** is the specific application of it to object construction. DI is IoC applied to dependencies.

## 2.3 The three styles

| Style | Form | Verdict |
|---|---|---|
| **Constructor** | parameters on the constructor | **use this** |
| **Field** | `@Inject` on a field | avoid |
| **Setter** | `@Inject` on a setter | rare; for genuinely optional dependencies |

**Constructor injection** wins for concrete reasons: fields can be `final` (immutable, thread-safe); the object is **never in a partially-constructed state**; dependencies are **explicit and visible** in the signature; and you can instantiate it in a test with `new` and no container at all. A constructor with nine parameters is also a loud SRP warning (№40 §3.2) — field injection hides that signal, which is part of why it's popular and part of why it's harmful.

## 2.4 What DI actually buys

**Testability** — pass a double in the constructor, no container needed (№44 §4). **Swappability** — change the implementation without touching consumers. **Explicit dependencies** — the constructor is a manifest. **Lifecycle management** — the container handles creation, ordering and destruction.

Note DI is not the same as "using a DI framework." You can inject dependencies by hand in `main()`; the framework just automates the wiring at scale.

---

# Part 3 — The container and bean lifecycle

## 3.1 The container

The **IoC container** owns a registry of **beans** (framework-managed objects), knows how to construct them, resolves their dependencies, and supplies them on request. Constructing that registry is the "wiring up" that happens at startup.

Registration happens by **annotation** (`@Singleton`, `@Service`, `@Controller`, `@Repository`), by **factory method** (`@Factory` + `@Bean` / `@Configuration` + `@Bean`) for third-party classes you can't annotate, or by **conditional registration** (`@Requires` / `@ConditionalOnProperty`) so a bean exists only under certain configuration.

## 3.2 The dependency graph

The container builds a DAG of beans, topologically sorts it, and constructs in dependency order. Same idea as Terraform's resource graph (№55 §3.5): declared relationships determine the ordering.

## 3.3 Lifecycle

**Instantiate** (constructor, dependencies injected) → **populate** (field/setter injection, config binding) → **post-construct** (`@PostConstruct` — the place for initialisation that needs dependencies present) → **in service** → **pre-destroy** (`@PreDestroy` — release resources, close pools) → **destroyed**.

`@PostConstruct` exists because constructor-time is too early for some work: the object must be fully constructed and injected first. For anything that should happen once the *whole application* is ready, prefer an application-ready event.

## 3.4 Resolution and ambiguity

Resolution is normally **by type**. Two implementations of one interface is therefore ambiguous, and the container will refuse to guess:

```java
@Singleton @Named("pdf")      class PdfExtractor implements Extractor { }
@Singleton @Named("markdown") class MarkdownExtractor implements Extractor { }

// Disambiguate at the injection point
QuestionService(@Named("pdf") Extractor extractor) { ... }

// Or mark a default
@Singleton @Primary class PdfExtractor implements Extractor { }

// Or take them all
QuestionService(List<Extractor> extractors) { ... }     // every implementation
```

That last form is a genuinely useful pattern — a strategy registry (№41 §6.1) assembled automatically, where adding a new implementation requires no change to the consumer (Open/Closed, №40 §5.2).

**"No bean of type X available"** means: it isn't annotated, it isn't in a scanned package, a `@Requires` condition isn't met, or you're asking for a concrete class where only an interface is registered.

## 3.5 Circular dependencies

A needs B, B needs A. With constructor injection this is **impossible to satisfy** — neither can be built first — so the container fails at startup with a clear error. That failure is a **feature**: a cycle is a design problem (№41 §2.1), and the fix is to extract the shared concern into a third bean or rethink the responsibilities. (Field injection can paper over cycles, which is another argument against it.)

---

# Part 4 — Scopes

## 4.1 The common scopes

| Scope | One instance per | Use for |
|---|---|---|
| **Singleton** | application | **the default** — stateless services, repositories |
| **Prototype** | injection point | stateful helpers you want fresh each time |
| **Request** | HTTP request | per-request context |
| **Session** | user session | rarely — it breaks statelessness (№31 §5.1) |
| **Refreshable / Context** | framework-specific | configuration that can be reloaded |

## 4.2 Singleton by default

Almost every bean should be a singleton: one instance, shared, created once. That's efficient and correct **provided the bean is stateless**.

## 4.3 The singleton trap — the most important thing in this document

```java
@Singleton
class QuestionService {
    private Question current;                     // ← SHARED ACROSS EVERY REQUEST THREAD

    public void process(Long id) {
        current = repo.findById(id).orElseThrow(); // thread A writes
        validate();                                // thread B has overwritten it
        save(current);                             // saves the wrong question
    }
}
```

One instance is shared by **every concurrent request**, so a mutable field is shared mutable state across threads (№12 §7.2) — with no lock, no visibility guarantee, and a race that appears only under load and is nearly impossible to reproduce locally.

**Keep injected beans stateless.** Per-request state belongs in **local variables** (confined to the calling thread's stack) or is passed as parameters:

```java
@Singleton
class QuestionService {
    public void process(Long id) {
        Question current = repo.findById(id).orElseThrow();   // local — confined, safe
        validate(current);
        save(current);
    }
}
```

Injected dependencies as `final` fields are fine — they're immutable references to other stateless singletons. It's *mutable* state that's the problem.

**Scope mismatch** is the related trap: injecting a request-scoped bean into a singleton would capture the first request's instance forever. Frameworks handle this by injecting a proxy that resolves per call — but it's worth knowing that's what's happening.

---

# Part 5 — Compile-time vs runtime: Micronaut and Spring

The genuine architectural difference, and the reason your stack behaves as it does.

## 5.1 Spring: runtime

At startup Spring **scans the classpath** for annotated classes, builds bean definitions **reflectively**, resolves dependencies at runtime, and creates **dynamic proxies** (JDK or CGLIB) for AOP.

The consequences: enormous flexibility (conditional beans, runtime configuration, an unmatched ecosystem), but **slower startup** (scanning thousands of classes), **higher memory** (reflection metadata and proxy classes), and **errors surfacing at runtime** — a missing bean is discovered when the context loads, not when you compile.

## 5.2 Micronaut: compile time

Micronaut's **annotation processor** runs during `javac`. It analyses your annotations and **generates the wiring code as real Java classes** in `build/classes`. At runtime there is no scanning and essentially no reflection — the framework executes generated code that directly calls your constructors.

The consequences:

- **Fast startup** — nothing to discover; the graph is already known.
- **Low memory** — no reflection metadata, no runtime proxy generation.
- **Compile-time errors** — a missing or ambiguous bean fails the *build*. Errors are refused, not deferred (the same philosophy as its repository generation, №20 §2.12).
- **Native-image friendly** — no runtime reflection is exactly what AOT compilation needs (§9).

The cost is less runtime dynamism, and a build step that does real work.

## 5.3 Side by side

| | **Spring** | **Micronaut** |
|---|---|---|
| Wiring | runtime, reflective | **compile time, generated** |
| Startup | slower | fast |
| Memory | higher | lower |
| Missing bean | runtime failure | **compile error** |
| Reflection | pervasive | minimal |
| Native image | needs extensive config/AOT tooling | designed for it |
| Ecosystem | **vast** | smaller but coherent |

## 5.4 Go and look

The single most demystifying exercise available to you: **build Practiq and read the generated sources.** You'll find `$QuestionService$Definition` classes containing plain, readable Java that constructs your beans and injects dependencies. The "magic" turns out to be code — code you can open in an editor. Very few things dispel framework anxiety as quickly.

---

# Part 6 — AOP, proxies and interceptors

## 6.1 The problem

Cross-cutting concerns — transactions, caching, security, retries, metrics, logging — are needed everywhere and belong to nothing (№42 §6). Writing them into every method scatters them and guarantees inconsistency.

## 6.2 Aspect-Oriented Programming

Declare the concern **once** and apply it declaratively:

```java
@Transactional                  // begin/commit around this method
@Cacheable("concepts")          // check the cache first
@Retryable(attempts = "3")      // retry on failure
public Concept findConcept(Long id) { ... }
```

The vocabulary: an **aspect** is the concern; **advice** is the code that runs (before/after/around); a **join point** is a place it *could* apply; a **pointcut** selects which join points it *does* apply to.

## 6.3 The mechanism — proxies

The container doesn't inject your object. It injects a **proxy** implementing the same interface, which wraps your instance:

```
caller → [ proxy: begin transaction ] → your method → [ proxy: commit ] → caller
```

Spring builds these at runtime (JDK dynamic proxies for interfaces, CGLIB subclasses for classes); **Micronaut generates them at compile time**, consistent with §5.2. Either way, the interception happens **at the boundary of the object** — and that single fact explains the most notorious framework bug there is.

## 6.4 Self-invocation — why `@Transactional` silently does nothing

```java
@Singleton
class QuestionService {

    public void importAll(List<Question> questions) {
        questions.forEach(this::saveOne);        // ← internal call: `this`, not the proxy
    }

    @Transactional
    public void saveOne(Question q) { ... }      // NO TRANSACTION HAPPENS
}
```

`this.saveOne(...)` calls the real object directly. The call **never leaves the object**, so it never passes through the proxy, so the transactional advice never runs. There is no error — it just silently doesn't work, and you discover it when a partial failure leaves half the data written.

The same applies to `@Cacheable`, `@Retryable`, `@Async`, `@Secured` — every proxy-based annotation.

**The fixes:** put the annotation on the **entry point** that's called from outside; or move the annotated method to a **separate bean** and inject it; or (Spring) inject a self-reference to the proxy. The cleanest is usually the first — annotate the service method that represents the unit of work (№20 §2.4), which is where the transaction boundary belongs anyway.

**Also required:** proxied methods must be **public** and **non-final** (a final method can't be overridden by a CGLIB subclass), and the object must be **obtained from the container** — `new QuestionService(...)` in a test gives you an unproxied object with no transactional behaviour at all.

> **The tell — proxies:** whenever a framework annotation "doesn't work", ask **"did this call cross the object boundary?"** If it came from inside the same object, it bypassed the proxy. That one question resolves most of these incidents.

---

# Part 7 — Configuration

## 7.1 Externalised configuration

Config comes from outside the artifact (№56 §7.2), typically layered by precedence: **command-line arguments → environment variables → external files → packaged `application.yml` → defaults**. The same build runs everywhere, configured differently.

## 7.2 Binding

```yaml
practiq:
  extraction:
    max-pages: 200
    timeout: 30s
    enabled: true
```

```java
@ConfigurationProperties("practiq.extraction")
public class ExtractionConfig {
    private int maxPages = 100;          // default if absent
    private Duration timeout = Duration.ofSeconds(10);
    private boolean enabled = true;
    // getters/setters — bound and type-converted at startup
}
```

**Prefer typed configuration objects to scattered `@Value` injections.** You get type conversion, defaults, validation, one place to see all related settings, and — with `@Validated` — startup failure on a bad value rather than a `NumberFormatException` at 3am. The same "make invalid states unrepresentable" argument as value objects (№41 §8.3).

If an `@Value` is null, it's usually injected after construction (so it isn't available in the constructor) or the key doesn't match — typed config objects avoid both problems.

## 7.3 Environments and conditional beans

```java
@Singleton
@Requires(env = "test")                        // only in the test environment
class InMemoryExtractor implements Extractor { }

@Singleton
@Requires(property = "practiq.extraction.enabled", value = "true")
class PdfExtractor implements Extractor { }
```

Environments (Micronaut) / profiles (Spring) select configuration files and conditional beans. Useful, and worth using sparingly: **conditional beans mean the object graph differs between environments**, which is exactly the "works in test, fails in production" failure mode. Keep the differences few and deliberate — ideally only genuine infrastructure stand-ins.

---

# Part 8 — Testing with a framework

## 8.1 The two levels

**Plain unit tests need no framework at all** — constructor injection means you just call `new`:

```java
var service = new QuestionService(mockRepo, mockNotifier);   // fast, no container
```

This is the payoff of §2.3 and it should be the bulk of your tests (№44 §2).

**Framework tests** start a context when you're testing the wiring, the HTTP layer, or transactional behaviour:

```java
@MicronautTest
class QuestionControllerTest {
    @Inject @Client("/") HttpClient client;
    @Inject QuestionRepository repository;

    @MockBean(QuestionRepository.class)
    QuestionRepository mockRepository() { return mock(QuestionRepository.class); }
}
```

## 8.2 The gotcha

**A `new`-ed object is not proxied** (§6.4), so `@Transactional`, `@Cacheable` and `@Retryable` do nothing in a plain unit test. If you're testing that behaviour, you need a context-based test — and if you're testing your own logic, you don't need those annotations to fire at all, which is exactly why the two levels exist.

Note the corollary for Practiq's tiers (№44 §2.2): unit tests exercise logic without the container; component tests exercise wiring with mocks; integration tests exercise the real thing with real Postgres — and transactional semantics genuinely only show up in that last tier.

---

# Part 9 — Native image, and why the architecture matters

**GraalVM native image** compiles ahead of time to a standalone binary: millisecond startup, a fraction of the memory, no JVM required (№10 §14.3, №13 §4.4).

The constraint that makes it hard is a **closed-world assumption** — everything reachable must be known at build time. Runtime reflection, dynamic proxies, dynamic class loading and runtime bytecode generation are all invisible to the compiler, so anything relying on them needs explicit configuration or fails at runtime.

Which is precisely why the compile-time architecture matters: **Micronaut's generated wiring and compile-time proxies are already static**, so native image works with far less friction than a framework built on runtime reflection. (Spring has closed much of this gap with Spring AOT, but it's added machinery rather than the original design.)

The trade to weigh: you gain startup and memory; you lose JIT peak throughput (§13 §4), pay a slow build, and accept reflection constraints. **For a long-running API like `practiq-api`, the JVM is likely still the right choice; for Lambda or scale-to-zero workloads, native image is compelling.** It remains a worthwhile spike precisely because Micronaut makes it cheap to try.

---

# Part 10 — When to use what

**A. Framework or plain Java?** Tell → framework: a web application, anything with real cross-cutting concerns, anything a team maintains. Tell → plain: a small library, a CLI, a focused tool. Default: **framework for applications, none for libraries** (a library shouldn't impose one on its consumers).

**B. Which injection style?** Tell → constructor: everything. Tell → setter: genuinely optional dependencies. Tell → field: never in production code. Default: **constructor.**

**C. Which scope?** Tell → singleton: stateless services (almost everything). Tell → prototype: the bean holds per-use state. Tell → request: per-request context you can't pass as a parameter. Default: **singleton, and keep it stateless** (§4.3).

**D. Annotation or explicit factory?** Tell → annotation: your own classes. Tell → `@Factory`/`@Bean`: third-party classes, or construction that needs logic. Default: **annotate yours, factory for theirs.**

**E. AOP or explicit code?** Tell → AOP: a genuine cross-cutting concern applied consistently (transactions, caching, metrics). Tell → explicit: business logic — hiding domain behaviour behind an annotation makes it invisible. Default: **AOP for infrastructure concerns only.**

**F. `@Value` or `@ConfigurationProperties`?** Tell → typed config object: more than one related property, or you want validation. Tell → `@Value`: a single standalone value. Default: **typed configuration objects.**

**G. Framework test or unit test?** Tell → unit: business logic. Tell → framework: wiring, HTTP, transactional or proxied behaviour. Default: **unit tests by default; framework tests only where the framework is what you're testing** (№44).

**H. Micronaut or Spring?** Tell → Micronaut: startup time, memory, native image, compile-time safety. Tell → Spring: ecosystem breadth, team familiarity, an integration that only Spring has. Default (Practiq): **Micronaut is well-chosen** — the compile-time model matches your preference for errors caught at build rather than deferred to runtime.

---

# How to expand this

- *Related:* №40 §5.2 (Dependency Inversion — the principle DI implements), №41 §3 (DI in design terms), №42 §3 (the same inversion at architectural scale — ports and adapters), №12 §7.2 (the singleton/shared-state trap), №20 §2.12 (Micronaut's compile-time repositories — the same idea applied to data access), №44 §4 (test doubles, which DI makes possible), №13 §9 (native image trade-offs).
- *Candidates for deeper treatment:* **reading Micronaut's generated code** for a Practiq service, annotation by annotation; **writing a custom interceptor** (an `@Around` advice for retries or auditing); **Spring vs Micronaut migration** considerations; **native image on Practiq** with the reflection-configuration story worked through.

*Stable concepts — IoC, DI, scopes, proxy-based AOP and the compile-time/runtime distinction don't drift. Framework APIs and annotation names do; check the current Micronaut documentation for exact syntax.*
