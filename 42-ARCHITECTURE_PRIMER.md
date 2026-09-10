# Architecture — A Primer №42

*The level above design: how you slice a **whole system** into parts, where the boundaries go, and which decisions you'll regret. Where №41 organises objects and modules, this organises components, deployables and the dependencies between them. Practiq-grounded — your monolith-first-with-a-documented-split is the running case study.*

The working definition, and the one that makes the topic tractable: **architecture is the set of decisions that are expensive to reverse.** Renaming a method is cheap. Changing your database, splitting a monolith, or adopting an event-driven core is not. That single criterion tells you what deserves architectural deliberation and what should just be decided and moved past — and it means the goal of architecture isn't to get everything right up front, but to **keep the expensive decisions few, deliberate, and deferred as long as reasonably possible.**

The second idea, which is the corrective to most architecture enthusiasm: **the best architecture is the least architecture that solves today's problem without painting you into a corner.** Every layer, boundary and indirection is a permanent tax on every future change. Structure should be earned, not anticipated.

Contents:

- **Part 1** — what architecture is and isn't
- **Part 2** — layered architecture
- **Part 3** — hexagonal, onion, clean: the dependency rule
- **Part 4** — monolith, modular monolith, microservices
- **Part 5** — communication styles
- **Part 6** — cross-cutting concerns
- **Part 7** — quality attributes and trade-offs
- **Part 8** — making and recording decisions
- **Part 9** — evolutionary architecture
- **Part 10** — when to use what

## Decision index

| The question | The answer lives in | §|
|---|---|---|
| Where do I put this class? | layering / hexagonal | §2, §3 |
| Should this be a separate service? | monolith vs microservices | §4 |
| Sync call or event? | communication styles | §5 |
| Where does auth/logging/validation go? | cross-cutting concerns | §6 |
| How do I justify this choice? | quality attributes, ADRs | §7, §8 |
| We might need to scale — should I split now? | §4.5 (almost certainly not) | §4 |
| The domain depends on Hibernate — is that bad? | the dependency rule | §3.2 |
| How do I stop it decaying? | fitness functions | §9 |

---

# Part 1 — What architecture is and isn't

## 1.1 The definition

Architecture is the **structure of a system: its components, their responsibilities, their relationships, and the principles governing its evolution.** In practice you're deciding four things:

1. **Boundaries** — what the parts are and what belongs in each.
2. **Dependencies** — which parts may know about which others, and in which direction.
3. **Communication** — how the parts talk (in-process calls, HTTP, events).
4. **Deployment** — what ships together and what ships separately.

Everything else — patterns, frameworks, libraries — is implementation detail underneath those four.

## 1.2 What it isn't

**Not a diagram.** A diagram is a communication artifact; the architecture is the actual dependency structure in the code, which frequently diverges from the picture on the wiki (§9.2).

**Not a technology list.** "We use Micronaut, Postgres and AWS" describes materials, not structure.

**Not a phase.** Architecture is not something you complete before coding; it's a continuous stream of decisions, most made while implementing.

**Not the architect's job alone.** The people writing the code make architectural decisions daily, whether or not they call them that. Which is why understanding this material matters even without the title.

## 1.3 Why it matters, stated honestly

Good architecture doesn't make features work — features work regardless. It changes the **cost curve**: how long the tenth feature takes compared with the first, whether a new person can be productive in a week or a quarter, whether one team's change breaks another's, and whether you can replace a component without a rewrite. **A bad architecture doesn't fail on day one; it fails on month eighteen**, when velocity has quietly halved and nobody can point at the moment it happened.

---

# Part 2 — Layered architecture

The most common structure, and the right starting point for most applications.

## 2.1 The layers

```
┌──────────────────────────────────────┐
│ Presentation   controllers, DTOs     │  HTTP in, JSON out
├──────────────────────────────────────┤
│ Application    services, use cases   │  orchestration, transactions
├──────────────────────────────────────┤
│ Domain         entities, rules       │  the actual business logic
├──────────────────────────────────────┤
│ Infrastructure repositories, clients │  database, external APIs
└──────────────────────────────────────┘
```

Each layer's job:

- **Presentation** — translate the outside world into calls on the application. HTTP concerns (status codes, serialisation, validation of request shape) live here **and nowhere else**.
- **Application** — orchestrate use cases. Thin: fetch, delegate to the domain, persist, publish. Transaction boundaries live here (№20 §2.4).
- **Domain** — the business rules, invariants and domain concepts. **The valuable part**, and the part that should outlive your framework choices.
- **Infrastructure** — technical implementations: persistence, HTTP clients, message publishing.

## 2.2 The rules that make it work

**Dependencies point one way** — downward (or inward, §3). Presentation may call Application; Application may not call Presentation. Break this and layering buys you nothing.

**Don't skip layers** — a controller reaching directly into a repository bypasses the transaction and use-case logic that was supposed to live in between. It works, until it doesn't.

**Don't leak concepts** — no HTTP status codes in the domain, no JPA annotations in the presentation DTOs, no SQL in the service layer. When a layer's vocabulary appears in another layer, the boundary has failed.

## 2.3 The trade-offs

**For:** familiar to everyone, easy to navigate, clean separation of technical concerns, straightforward testing per layer.

**Against:** layers can become anemic pass-throughs (a controller calling a service that calls a repository, each adding nothing — №41 §7); **a change to one feature touches every layer**, which is the opposite of the "things that change together belong together" heuristic (№41 §2.3); and it organises by *technical role* rather than by *feature*.

## 2.4 Package by feature, not by layer

The fix for that last objection, and a genuinely valuable change:

```
com.practiq.question.QuestionController      ← by feature: everything for one
com.practiq.question.QuestionService            concept sits together
com.practiq.question.QuestionRepository
com.practiq.concept.ConceptController
com.practiq.concept.ConceptService
```

rather than `com.practiq.controllers.*`, `com.practiq.services.*`, `com.practiq.repositories.*`. The layering still exists — the rules still apply — but the *packages* group things that change together, which means a feature change is local, and you can see the boundaries of a potential future service at a glance (§4.4). Package-private visibility then genuinely enforces feature boundaries.

> **The tell — layering:** keep the layers, but package by feature. If adding a field means editing five files across five packages, your packaging is fighting your change pattern.

---

# Part 3 — Hexagonal, onion, clean

Three names for essentially one idea, and the most valuable structural concept in this document.

## 3.1 The problem being solved

In naive layering, the domain sits *above* infrastructure but often *depends* on it — your entities carry JPA annotations, your services import the persistence framework, your business logic knows about HTTP. The consequence: **the most valuable, longest-lived part of your system is coupled to the parts most likely to change.** Frameworks and databases come and go; the rules of your domain don't.

## 3.2 The dependency rule

**All dependencies point inward, toward the domain. The domain depends on nothing.**

```
        ┌─────────────────────────────────────┐
        │   Infrastructure  (adapters)        │   ← Postgres, HTTP, SQS
        │  ┌───────────────────────────────┐  │
        │  │  Application  (use cases)     │  │
        │  │   ┌───────────────────────┐   │  │
        │  │   │      Domain           │   │  │   ← entities, rules, ports
        │  │   │  (no dependencies)    │   │  │
        │  │   └───────────────────────┘   │  │
        │  └───────────────────────────────┘  │
        └─────────────────────────────────────┘
                  dependencies point ⟶ inward
```

The mechanism that makes an inward-only rule possible is **Dependency Inversion** (№40 §5.2): the domain defines an *interface* it needs (a **port**), and infrastructure provides an *implementation* (an **adapter**). The arrow of dependency is inverted relative to the arrow of control flow.

```java
// domain — a PORT. No framework, no persistence knowledge.
package com.practiq.domain;
public interface QuestionRepository {
    Optional<Question> findById(Long id);
    void save(Question question);
}

// infrastructure — an ADAPTER. Depends inward on the domain interface.
package com.practiq.infrastructure.persistence;
@Singleton
class JpaQuestionRepository implements QuestionRepository { ... }
```

At runtime the service calls the repository (control flows outward); at compile time the infrastructure depends on the domain (dependency points inward). That inversion is the whole trick.

## 3.3 Ports and adapters

**Ports** are the interfaces at the boundary — *driving* ports (how the outside invokes the application: a use-case interface) and *driven* ports (what the application needs from the outside: repository, notifier, clock). **Adapters** implement them: a REST controller is a driving adapter; a JPA repository, an SQS publisher and a system clock are driven adapters.

The payoff is concrete: **you can swap any adapter without touching the core**, and you can test the entire domain and application layer with in-memory adapters — no database, no HTTP, no containers, in milliseconds (№44 §4.3).

## 3.4 The honest trade-off

This costs something real: more interfaces, more mapping between domain objects and persistence entities, and more indirection to read through. For a small CRUD service it's ceremony.

**Where Practiq sits, honestly:** you let JPA entities *be* the domain model, which couples the domain to Hibernate. That is a legitimate, extremely common pragmatic trade at your scale — the cost of full separation (a parallel set of domain objects plus mappers) would exceed the benefit for a solo project. What matters is **knowing you've made the trade**, and taking the part of the discipline that's free: keep the *logic* in the entities rather than letting them go anemic (№41 §7), keep HTTP concerns out of the service layer, and keep infrastructure imports out of your business rules. If Practiq ever grew a second delivery mechanism or a second persistence store, you'd introduce ports then — and the feature-packaged structure (§2.4) makes that a contained change.

> **The tell — hexagonal:** apply the dependency rule *directionally* even when you don't build the full structure. The question "does my business logic import anything from infrastructure?" is worth asking regardless of how many interfaces you're willing to write.

---

# Part 4 — Monolith, modular monolith, microservices

The decision that generates the most heat and the most regret.

## 4.1 The three shapes

| | **Monolith** | **Modular monolith** | **Microservices** |
|---|---|---|---|
| Deployables | one | one | many |
| Internal boundaries | often weak | **strong, enforced** | physical (network) |
| Data | one database | one database, module-owned schemas | database per service |
| Transactions | ACID across everything | ACID across everything | **distributed — sagas** (№31 §8.2) |
| Communication | method calls | method calls | network (§5) |
| Deploy independently | no | no | yes |
| Scale independently | no | no | yes |
| Operational cost | low | low | **high** |
| Debugging | a stack trace | a stack trace | distributed tracing (№31 §11) |
| Team fit | one team | one to a few teams | many independent teams |

## 4.2 The honest case for microservices

They solve **an organisational problem before a technical one**: independent teams shipping without coordinating releases. The genuine benefits — independent deployment, independent scaling, technology heterogeneity, fault isolation — are real but conditional. They arrive only when you have enough teams that coordination is the bottleneck.

## 4.3 The costs, stated plainly

You have converted **method calls into network calls** and inherited the entire contents of №31: partial failure, latency, retries, idempotency, eventual consistency, distributed tracing, service discovery, versioned contracts. You have converted **ACID transactions into sagas**. You have multiplied your operational surface by the number of services. And you have made the *worst* failure mode possible: a **distributed monolith**, where services are separately deployed but so coupled that they must be released together — all of the cost, none of the benefit.

The empirical pattern is consistent: **most teams that adopt microservices early get the costs without the benefits.** There's a well-worn cycle of prominent teams splitting into services and later consolidating back, having found the coordination overhead exceeded the gains.

## 4.4 The modular monolith — the sensible default

One deployable, but with **strict internal module boundaries**: each module owns its data, exposes a narrow public interface, and may not reach into another's internals. Enforced by package structure, package-private visibility, module systems, or build tooling (ArchUnit tests — §9.3).

You get: local reasoning and clean boundaries, one deploy, ACID transactions, one stack trace when it breaks, and **the option to extract a module into a service later** — because the seam already exists. A well-modularised monolith is a strictly better starting point than either a mud-ball monolith or premature microservices.

## 4.5 Where Practiq sits, and why it's right

Your documented plan — **monolith-first with a documented target service split** (`practiq-api`, `practiq-processor`, `practiq-extractor`, `practiq-frontend`, `practiq-infrastructure`) — is the correct architecture for a solo project at your scale, and defensible on its merits (№43 §9).

The reasoning worth being able to state: you are one developer, so there is no coordination problem to solve; your traffic is a few requests per second, so there is no scaling problem to solve; and your review workflow benefits from real transactions. Meanwhile the *one* component with a genuinely different profile — the **extractor** (different runtime, different language, bursty, slow, retryable, failure-tolerant) — is already separated, which is exactly right: **split where the seam is real, not on principle.**

The extraction trigger to watch for later: a module that needs to scale independently, fail independently, be written in a different stack, or be owned by a different team. Absent one of those, splitting adds cost and subtracts nothing.

> **The tell — splitting:** "we might need to scale" is not a reason to distribute; it's a reason to keep boundaries clean so you *can*. Extract a service when you can name the specific benefit and accept the specific cost. You can always split later; un-splitting is far harder.

---

# Part 5 — Communication styles

## 5.1 Synchronous request/response

HTTP, gRPC, or an in-process method call. Simple, immediately consistent, easy to debug — and it **couples availability**: the caller can only succeed if the callee is up and fast. Chains of synchronous calls multiply latency and failure probability, and produce cascading failure without timeouts and circuit breakers (№31 §10).

## 5.2 Asynchronous messaging

Queues and events. The caller hands off and continues; the work happens later. **Temporal decoupling** is the benefit — the consumer can be down, slow, or restarted, and the work still completes. The costs are eventual consistency, harder debugging, and the need for idempotent consumers (№31 §8).

## 5.3 Event-driven architecture

Components emit **events** ("QuestionApproved") and others react, with no direct references between them. Excellent decoupling: adding a new reactor requires no change to the emitter.

The trade is significant and worth naming: **the control flow becomes implicit.** No call graph shows you what happens when a question is approved; you have to know which subscribers exist. Debugging becomes archaeology across services, and reasoning about the system requires holding the event topology in your head. Event-driven is powerful and it is *not* the simpler option.

**Event notification** (a thin "this happened, go look") versus **event-carried state transfer** (the event contains the data, so consumers needn't call back) versus **event sourcing** (the event log *is* the source of truth, state is derived) are increasing levels of commitment. Event sourcing in particular is a serious undertaking — powerful for audit and temporal queries, expensive in complexity.

## 5.4 Choosing

| Use | When |
|---|---|
| **Sync** | the caller needs the result to proceed; user-facing reads and writes |
| **Async queue** | slow, retryable work the caller doesn't need to wait for (Practiq's PDF extraction) |
| **Events** | several independent components must react, and you want them decoupled |

> **The tell — communication:** default to synchronous for simplicity, and reach for async when you can name the benefit — decoupling availability, absorbing bursts, or fanning out to multiple consumers. Don't adopt events for elegance; adopt them for a specific decoupling you need.

---

# Part 6 — Cross-cutting concerns

Things every part of the system needs and no single part owns: **authentication and authorisation, logging, error handling, validation, transactions, caching, metrics and tracing, configuration.**

The failure mode is scattering them — auth checks copied into forty controllers, logging formatted differently in each service, error responses inconsistent across endpoints. Scattered concerns are the definition of shotgun surgery (№40 §8.2).

The mechanisms, cheapest first: **framework filters and interceptors** (one place for auth, correlation IDs, request logging); **AOP/annotations** (`@Transactional`, `@Cacheable` — implemented with proxies, №41 §5.4, which is why self-invocation silently bypasses them); **a global exception handler** mapping domain exceptions to HTTP responses in one place; **the edge** (an API gateway or load balancer handling TLS, rate limiting, and coarse auth before the request reaches you, №51 §10).

The principle: **each cross-cutting concern should have exactly one place it's implemented, and application code should not repeat it.** If you can grep for a concern and find it in fifty files, that's the bug.

---

# Part 7 — Quality attributes and trade-offs

Architecture is chosen against **quality attributes** (the "-ilities"), and they conflict — which is why there's no universally best architecture.

| Attribute | Means | Bought with | At the cost of |
|---|---|---|---|
| **Performance** | latency and throughput | caching, denormalisation, colocating | consistency, simplicity |
| **Scalability** | handles growth | statelessness, partitioning, async | complexity, consistency |
| **Availability** | stays up | redundancy, multi-AZ, degradation | cost, consistency (№31 §4) |
| **Consistency** | data is correct everywhere | transactions, sync replication | latency, availability |
| **Security** | resists attack | layers, least privilege, encryption | convenience, some latency |
| **Maintainability** | cheap to change | modularity, tests, clarity | up-front effort |
| **Testability** | verifiable | seams, DI, pure cores | indirection |
| **Operability** | runnable in production | observability, health checks, IaC | build effort |
| **Cost** | money | right-sizing, serverless, managed | control, sometimes performance |

**The skill is naming which attributes matter *for this system* and accepting the cost on the others.** A banking ledger prioritises consistency and security over latency. A social feed prioritises availability and latency over consistency. Practiq, honestly, prioritises **maintainability and operability** — it's a solo-developer learning project and portfolio piece where velocity and clarity matter more than scale, and that's a legitimate, statable position.

---

# Part 8 — Making and recording decisions

## 8.1 The reversibility test

Sort decisions by cost of reversal:

- **Cheap (two-way doors)** — a library choice, a package layout, an internal API shape. **Decide fast, move on, change it if wrong.** Deliberating over these is pure waste.
- **Expensive (one-way doors)** — your database, your language, your public API contract, your service boundaries, your data model. **These deserve real deliberation**, prototypes, and written reasoning.

Most decisions are two-way doors treated as one-way. The discipline is telling them apart and spending your deliberation budget accordingly.

## 8.2 Architecture Decision Records

A short document per significant decision, versioned alongside the code:

```markdown
# ADR-012: Use SEQUENCE rather than IDENTITY for entity IDs

## Status
Accepted — 2026-03-14

## Context
Postgres. The seed pipeline inserts thousands of questions per run.
Hibernate cannot batch inserts when IDs are generated by the INSERT itself.

## Decision
Use GenerationType.SEQUENCE with allocationSize 50 on all entities.

## Consequences
+ Insert batching works; the seed pipeline is materially faster.
+ IDs are pre-allocatable without a round trip per row.
− IDs may have gaps (blocks are pre-allocated and not always consumed).
− Slightly more mapping configuration per entity.

## Alternatives considered
IDENTITY — simplest mapping, but silently disables batching.
TABLE — portable, slow; no reason to accept the cost on Postgres.
```

Why they're worth the ten minutes: in a year, nobody remembers *why*, and without the reasoning people either cargo-cult the decision or reverse it without knowing what it cost. An ADR captures the **context and the rejected alternatives**, which is the part that's genuinely irrecoverable later. Your `PRACTIQ_MASTER.md` decision log is this practice already — the format above just makes the *alternatives* and *consequences* explicit.

## 8.3 The discipline that matters most

**Record the decision when it's made, not when it's questioned.** And keep the distinction you already hold firmly: **discussion is not adoption.** A decision isn't a decision until it's confirmed and written down — which prevents the failure mode where an exploratory conversation silently becomes an architectural commitment nobody agreed to.

---

# Part 9 — Evolutionary architecture

## 9.1 Architecture is not finished

The system you design for today's requirements will meet requirements you can't currently see. The response is not to guess harder — it's to keep the system **easy to change**: strong module boundaries, comprehensive tests, small deployable increments, and decisions deferred until you have information.

**Defer irreversible decisions until the "last responsible moment"** — the point past which delaying costs more than deciding. Choosing a message broker before you know your throughput profile is guessing; choosing it when you have real numbers is engineering.

## 9.2 Architectural drift

The code diverges from the intended structure, one small violation at a time: a controller reaching into a repository "just this once," a domain class importing an infrastructure type, a module reading another's tables. Nobody makes the decision to abandon the architecture — it erodes, and eventually the diagram on the wiki describes a system that no longer exists.

## 9.3 Fitness functions

The countermeasure: **make architectural rules executable.** ArchUnit lets you assert structure as a test:

```java
@Test
void domainMustNotDependOnInfrastructure() {
    noClasses().that().resideInAPackage("..domain..")
        .should().dependOnClassesThat().resideInAPackage("..infrastructure..")
        .check(new ClassFileImporter().importPackages("com.practiq"));
}
```

Now a violation fails CI rather than accumulating silently. Other fitness functions: performance budgets, dependency-cycle checks, package-boundary rules, security scans. **This is the single most practical idea in this Part** — an architecture rule nobody enforces is a preference, and preferences lose to deadlines.

## 9.4 Strangler fig

The pattern for replacing a system without a big-bang rewrite: put a facade in front of the old system, implement new functionality in the new one, and progressively route traffic across until the old system is dead and can be removed. Incremental, reversible at every step, and the standard answer to "how do we migrate off this?"

---

# Part 10 — When to use what

**A. How much architecture?** Tell → more: the system is large, long-lived, multi-team, or the domain is genuinely complex. Tell → less: small, exploratory, solo, or the requirements are still moving. Default: **the least that solves today's problem while keeping boundaries clean.**

**B. Layered or hexagonal?** Tell → layered: a straightforward CRUD-shaped application. Tell → hexagonal: complex domain logic, multiple delivery mechanisms or data sources, or long expected life. Default: **layered, packaged by feature, with the dependency rule applied directionally** (§3.4).

**C. Monolith, modular monolith, or services?** Tell → monolith: small team, early product. Tell → modular monolith: you want boundaries and future optionality (**the default**). Tell → microservices: multiple teams needing independent deploys, or genuinely divergent scaling/technology needs. Default: **modular monolith; extract a service when you can name the specific benefit.**

**D. Sync or async?** Tell → sync: the caller needs the answer. Tell → async: slow, retryable, or fan-out work. Default: **sync unless you can name the decoupling you're buying.**

**E. Where does this logic go?** Tell → domain: it's a business rule. Tell → application service: it orchestrates a use case across objects. Tell → infrastructure: it's about a technical mechanism. Tell → presentation: it's about HTTP. Default: **domain first — services are where anemic models come from** (№41 §7).

**F. Decide now or defer?** Tell → now: it's a one-way door and blocking progress. Tell → defer: reversible, or more information is coming soon. Default: **decide two-way doors immediately; defer one-way doors to the last responsible moment.**

**G. Write an ADR?** Tell → yes: expensive to reverse, non-obvious, or you rejected a plausible alternative. Tell → no: routine and self-evident. Default: **if you'd have to explain the reasoning to someone in six months, write it down.**

---

# How to expand this

- *Related:* №41 Software Design (the same thinking at object level — coupling, cohesion, DDD aggregates as service-boundary candidates); №40 Clean Code (at function level); №31 Distributed Systems (everything microservices inherit); №43 System Design (the same decisions taken from requirements to a design); №44 Testing (fitness functions, testability as an architectural property); №20 §2.4 (transaction boundaries as an application-layer concern).
- *Candidates for deeper treatment:* **a modular-monolith blueprint for Practiq** with enforced boundaries and ArchUnit rules; **event-driven architecture and event sourcing** properly; **the C4 model** for documenting architecture; **migration patterns** (strangler fig, branch by abstraction) worked through.

*Stable material, written from knowledge — layering, the dependency rule, the monolith/services trade-off and decision records don't drift. The canonical sources: Evans and Vernon for DDD-informed boundaries, Fowler's writing on monoliths and microservices, Ford et al.* Building Evolutionary Architectures *for fitness functions.*
