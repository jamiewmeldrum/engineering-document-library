# The Engineer's Map — A Foundations Primer

*The one-stop mental model of what software engineering actually is: the CS foundations nobody taught you formally, and the craft that separates competent from good. Built to fill years of rust. Practiq lens throughout. Written July 2026; the one genuinely volatile fact (Terraform's licensing, §13) is verified.*

This is a **map, not a manual**. It covers a lot of ground at "I understand what this is, why it matters, and when it bites" depth — enough to reason, to make decisions, and to know when you are out of your depth — and points you at deeper material where you have it. It does **not** re-run the six documents you already have; where a topic is covered there in depth, this map gives you the connective idea and sends you on.

**Your existing library (this doc points at these for depth):**

| Doc | Covers |
|---|---|
| **Java data-access primer** | the whole DB stack: JDBC → JPA → Hibernate → SQL, transactions, indexing, query plans |
| **JPA & Hibernate reference** | every annotation/class a consumer uses, plus a worked entity model |
| **Java Collections reference** | the framework, `HashMap` internals, sorting, concurrency |
| **Modern Java primer** | the language since Java 8, the JVM, GC, packaging |
| **Docker primer** | containers, images/layers, the runtime stack, Dockerfiles |
| **Networking primer** | OSI/TCP/IP, TCP/UDP, DNS, HTTP, TLS, certificates, cookies |

**How to read this.** Front-to-back once to build the frame, then as a reference. Each part is a primer-in-miniature: the core model, the concepts that matter, "the tell" for decisions, and a *go deeper* pointer. The arc is deliberate — **machine → foundations → code → design → data → concurrency → distributed → architecture → correctness → security → operations → cloud → IaC → frontend → judgment** — bottom of the stack to top, then the meta-skills that tie it together.

Contents:

1. How computers actually work
2. CS foundations: data structures, algorithms, complexity
3. Writing good code
4. Designing software: OOP, FP, patterns, principles
5. Data and databases
6. Concurrency and parallelism
7. Distributed systems
8. Architecture
9. Correctness: testing and debugging
10. Security
11. Operations: CI/CD, observability, running things in production
12. Cloud and AWS
13. Infrastructure as Code
14. Front-end, for a backend engineer
15. Craft and judgment

---

# Part 1 — How computers actually work

Everything above this layer is an abstraction over a machine that does a few simple things very fast. You don't need to write assembly, but the model explains *why* code is fast or slow, why concurrency is hard, and why memory bugs happen.

## 1.1 The core loop

A CPU does one thing in a loop: **fetch an instruction, decode it, execute it, repeat** — billions of times a second. Instructions operate on **registers** (a handful of tiny, instant storage slots) and **memory** (vast, but far slower). That speed gap between the CPU and main memory (RAM) is the single most important performance fact in computing, and it's why caches exist.

## 1.2 The memory hierarchy — the fact that explains performance

| Level | Access time (rough) | Size |
|---|---|---|
| Register | instant | bytes |
| L1 cache | ~1 ns | ~64 KB |
| L2/L3 cache | ~10 ns | MBs |
| **Main memory (RAM)** | ~100 ns | GBs |
| SSD | ~100 µs (1000× RAM) | TBs |
| Network / spinning disk | ms+ (millions× RAM) | — |

Each level down is roughly an order of magnitude slower. The practical consequences you feel constantly:

- **Cache locality wins.** An `ArrayList` iterates faster than a `LinkedList` not because of Big-O but because the array's elements sit *contiguously* in memory — the CPU prefetches the next ones into cache. The linked list scatters nodes across RAM, and every hop is a cache miss (Collections reference §2). This is *why* the theoretical complexity often loses to the hardware reality.
- **Disk and network are a different universe.** An in-memory operation is nanoseconds; a database round trip is milliseconds — a million times slower. This is why the N+1 problem matters (data-access primer §2.8): it's not the work, it's the *number of trips*.
- **RAM is volatile; disk persists.** Power off, RAM is gone. Everything about persistence, durability, and databases exists because of this line.

> **The tell:** when something's slow, the first question is almost never "is my algorithm O(n²)?" — it's "how many times am I crossing a slow boundary?" (disk, network, cache). Count the round trips before you optimise the loop.

## 1.3 Processes, threads, and the OS

The **operating system** is the referee: it shares the CPU, memory, and devices among programs that each think they own the machine.

- A **process** is a running program with its *own* isolated memory space. Two processes can't touch each other's memory (that's the protection boundary — and the thing containers use, Docker primer §2.2).
- A **thread** is a unit of execution *within* a process, sharing the process's memory with its sibling threads. Cheap to switch between, but the shared memory is exactly what makes concurrency dangerous (§6).
- The OS **scheduler** rapidly switches the CPU between threads (a **context switch**), creating the illusion of simultaneity even on one core. Real parallelism needs multiple cores.
- **Virtual memory** gives each process a private, contiguous-looking address space that the OS maps to physical RAM (and to disk, via *paging*, when RAM is full — which is why a machine that starts swapping to disk falls off a cliff, per §1.2).

## 1.4 Numbers, bytes, text

- **Everything is bits.** A byte is 8 bits. Integers are fixed-width (32/64-bit) and **overflow silently by wrapping** (Modern Java primer §11.4). Floating point is *approximate* binary — `0.1 + 0.2 != 0.3` — so never use it for money.
- **Text is bytes + an encoding.** **UTF-8** is the answer (variable-width, ASCII-compatible, the web default). "Mojibake" (garbled characters) is always an encoding mismatch: bytes written as one encoding, read as another.
- **Endianness, signed/unsigned, two's complement** exist and occasionally bite at boundaries (binary protocols, bit manipulation), but rarely in application code.

*Go deeper:* the JVM's take on all of this — heap/stack, GC, container-awareness — is Modern Java primer §13.

---

# Part 2 — CS foundations: data structures, algorithms, complexity

The formal-training gap, done practically. You don't need to implement a red-black tree; you need to *choose* the right structure and *reason* about cost.

## 2.1 Big-O — the one piece of theory you must own

Big-O describes how an algorithm's cost **grows as the input grows**, ignoring constants. It's about *scaling*, not absolute speed.

| Notation | Name | Example | 1k items | 1M items |
|---|---|---|---|---|
| **O(1)** | constant | HashMap get, array index | 1 | 1 |
| **O(log n)** | logarithmic | binary search, balanced tree | ~10 | ~20 |
| **O(n)** | linear | scan a list | 1k | 1M |
| **O(n log n)** | linearithmic | good sorts | ~10k | ~20M |
| **O(n²)** | quadratic | nested loop over the same data | 1M | **10¹²** ← dead |
| **O(2ⁿ)** | exponential | brute-force combinations | — | heat death |

The lesson lives in that last column: O(n²) is fine for 100 items and catastrophic for a million. The **jump from O(n²) to O(n log n)** is the difference between "instant" and "never finishes" at scale — and it's usually the difference between a nested loop and using a `HashSet`/sort.

Also track **space complexity** (memory growth) and know that Big-O hides constants and cache effects (§1.2) — which is why a "slower" O(n) array scan often beats a "faster" O(1) linked-list operation in practice.

> **The tell — algorithmic cost in practice:** the practical skill is spotting the accidental O(n²) — a loop that calls `list.contains()` inside it (that's O(n) inside O(n)). The fix is almost always "put it in a `HashSet`/`HashMap` first." If you find yourself scanning a collection inside a loop over another, stop and hash.

## 2.2 The data structures, and their costs

You already have these in depth (Collections reference); here's the CS-level summary of *why* each has the cost it does:

| Structure | Fast at | Slow at | Because |
|---|---|---|---|
| **Array / ArrayList** | index access O(1), iterate | insert/delete in middle O(n) | contiguous memory; index is arithmetic, but inserting shifts everything |
| **Linked list** | insert/delete at a known node O(1) | random access O(n) | nodes chase pointers; no index arithmetic |
| **Hash table** | get/put O(1) avg | ordering, worst-case O(n) | key → bucket via hash; see HashMap internals, Collections §4.1 |
| **Balanced tree** (red-black) | ordered ops, range O(log n) | vs hash for point lookups | sorted structure, height ~log n |
| **Heap** | min/max O(1), insert O(log n) | arbitrary lookup | partially-ordered tree; the `PriorityQueue` |
| **Stack / Queue** | push/pop/enqueue O(1) | — | restricted-access lists (LIFO / FIFO) |
| **Graph** | model relationships | depends on algorithm | nodes + edges; adjacency list or matrix |
| **Trie** | prefix search | memory | tree keyed by string prefixes (autocomplete) |

The meta-skill is the *mapping from problem to structure*: "need uniqueness" → set; "need key→value" → map; "need order" → tree/sorted; "need most-urgent-first" → heap; "need relationships/paths" → graph.

## 2.3 Algorithms worth actually knowing

Not to implement from scratch, but to recognise and reason about:

- **Searching:** linear O(n); **binary search** O(log n) — but *only on sorted data*. The classic "halve the search space each step."
- **Sorting:** you'll never write one (the library's is better), but know **O(n log n)** is the floor for comparison sorts, that Java's is **stable** (Collections §6), and roughly how merge sort (divide/conquer) and quicksort (partition) work.
- **Graph traversal:** **BFS** (breadth-first, uses a queue, finds shortest unweighted path) and **DFS** (depth-first, uses a stack/recursion). Shortest weighted path is **Dijkstra**. You'll meet these as "find connected things" / "shortest route" / "dependency order" (topological sort).
- **Recursion & divide-and-conquer:** a function calling itself on a smaller input, with a base case. Elegant for trees and naturally-recursive problems; watch the stack (deep recursion → `StackOverflowError`).
- **Dynamic programming:** the one that intimidates — it's just "cache the answers to subproblems so you don't recompute them" (memoisation). Recognise it as "brute force with overlapping subproblems, made fast by remembering."
- **Two pointers / sliding window / greedy:** each is a trick for turning an O(n²) scan into O(n); §30 covers the full set.

> **The tell — CS depth:** "choose the right structure, don't write accidental O(n²), know when a round trip dominates" is 90% of what this ever costs you in practice. The named patterns (hash for lookups, two pointers, BFS/DFS, sliding window, basic DP) are worth knowing *as shapes to recognise* rather than algorithms to reproduce; №30 covers them properly. Nobody invents quicksort under pressure, and nobody needs to.

---

# Part 3 — Writing good code

The daily craft. Code is read far more than written, so the audience is the next person (often you, in six months). Good code optimises for *change* and *comprehension*, not cleverness.

## 3.1 Naming — the highest-leverage skill

Names are the primary interface to your intent. `daysUntilExpiry` beats `d`; `isEligibleForReview` beats `flag`. The rules: reveal intent, avoid abbreviations, make the type/unit obvious (`timeoutMs`, `priceInPence`), name booleans as questions (`isValid`, `hasAccess`), and let a good name replace a comment. A method that needs a comment to explain *what* it does usually needs a better name; comments should explain *why*, not *what*.

## 3.2 Functions

- **Small, one job.** A function should do one thing at one level of abstraction. If you can meaningfully extract a chunk and name it, it was doing two things.
- **Few parameters.** More than ~3 is a smell — often those parameters want to be an object.
- **No surprises.** A function named `getUser` that also writes to the database is a landmine. Command (does something) vs query (returns something) — don't mix.
- **Fail fast.** Validate inputs at the top and return/throw early (guard clauses) rather than nesting the happy path inside deep `if`s.

## 3.3 The SOLID principles — object design, decoded

The famous five. They're guidelines for *where change hurts least*, not laws:

| Principle | Plain meaning | The tell it's violated |
|---|---|---|
| **S** — Single Responsibility | one class, one reason to change | the class is named `...Manager`/`...Util` and does five things |
| **O** — Open/Closed | extend behaviour without editing existing code | adding a feature means a big `switch` you keep editing |
| **L** — Liskov Substitution | a subtype must be usable anywhere its parent is | a subclass throws `UnsupportedOperationException` on an inherited method |
| **I** — Interface Segregation | many small interfaces beat one fat one | implementers stub out methods they don't need |
| **D** — Dependency Inversion | depend on abstractions, not concretions | `new ConcreteThing()` buried in business logic instead of injected |

**D is the one that shapes your day:** it's why frameworks inject a `QuestionRepository` *interface* into your service rather than you `new`-ing a concrete class. That indirection is what makes the thing testable (swap a mock) and swappable (change the implementation without touching callers). Micronaut's compile-time DI is D as a framework feature.

## 3.4 The other acronyms worth knowing

- **DRY** (Don't Repeat Yourself) — but beware over-applying it; *incidental* duplication (two things that look alike today but change for different reasons) should stay duplicated. The real target is duplicated *knowledge*, not duplicated *lines*.
- **YAGNI** (You Aren't Gonna Need It) — don't build for imagined future needs. The abstraction you add "for flexibility" usually guesses wrong and just adds cost.
- **KISS** — the simplest thing that works is usually right.
- **Composition over inheritance** — prefer building behaviour by combining objects over deep class hierarchies (inheritance is rigid and leaks; §4).
- **Law of Demeter** — talk to your immediate collaborators, not their internals (`a.getB().getC().doThing()` is a coupling smell).

## 3.5 Code smells and refactoring

A **smell** is a surface symptom of a deeper design problem: long methods, large classes, long parameter lists, feature envy (a method more interested in another class's data), shotgun surgery (one change forces edits in many places), primitive obsession (a `String` where a value object belongs). **Refactoring** is changing structure *without* changing behaviour — and the safety net that makes it possible is tests (§9). The discipline: red-green-refactor, small steps, tests green throughout.

> **The tell — good code:** the real measure isn't elegance, it's "how painful is the next change?" Good code is code where the change you didn't anticipate is still cheap. Optimise for the reader and for change; treat cleverness as a cost.

---

# Part 4 — Designing software

Above individual functions sits the question of how objects and modules fit together.

## 4.1 Paradigms

- **Object-oriented (OOP):** bundle data with the behaviour that operates on it (objects). Its four ideas: **encapsulation** (hide internals behind an interface), **abstraction** (expose the what, hide the how), **inheritance** (share via an is-a hierarchy — use sparingly), **polymorphism** (one interface, many implementations — the engine of extensibility). Java is OO by default.
- **Functional (FP):** compute with pure functions and immutable data, avoiding shared mutable state. Its ideas — **pure functions** (same input → same output, no side effects), **immutability**, **higher-order functions** (functions as values) — are now baked into modern Java (streams, lambdas, records; Modern Java §2, §6, §7). You don't pick one; good modern code is OO in the large, functional in the small.

## 4.2 Coupling and cohesion — the two words that matter most

- **Coupling** = how much modules depend on each other. **Low is good.** Tightly coupled code means a change here breaks something over there.
- **Cohesion** = how focused a module is on one job. **High is good.** A cohesive class's parts all serve one purpose.

Almost every design principle is in service of **low coupling, high cohesion.** That's the phrase to keep in your head; SOLID, layering, interfaces, and events are all mechanisms for achieving it.

## 4.3 Design patterns — the ones that earn their keep

Patterns are named solutions to recurring problems. The trap is over-using them; the value is a shared vocabulary. The ones worth truly knowing:

| Pattern | Solves | You've seen it in |
|---|---|---|
| **Strategy** | swap an algorithm at runtime | a `Comparator`; your `QuerySpecification` |
| **Factory** | create objects without naming the concrete class | `List.of()`, framework bean creation |
| **Builder** | construct complex objects step by step | `HttpRequest.newBuilder()` |
| **Adapter** | make an incompatible interface fit | wrapping a third-party API |
| **Observer** | notify many listeners of an event | event listeners, pub/sub |
| **Decorator** | add behaviour by wrapping | `BufferedReader(new FileReader(...))` |
| **Singleton** | one shared instance | a Spring/Micronaut bean (but *don't* hand-roll it — it's a testing headache) |
| **Dependency Injection** | supply collaborators from outside | the whole framework you use |

*Anti-patterns* to recognise and avoid: God object (one class that does everything), spaghetti (no structure), magic numbers/strings (unexplained literals), premature optimisation, and copy-paste programming.

## 4.4 API and domain design

- **API design:** an API is a contract and a promise of stability. Make it hard to misuse (the type system as a guardrail — Practiq's `APPROVED`-only endpoint signature is this idea: make the wrong call unexpressible). Be consistent, name by intent, version deliberately, and design for the caller's use case, not your data model.
- **Domain modelling:** shape the code around the real-world concepts and rules (Concept, Question, SpecSection), not around database tables. The vocabulary of the code should match the vocabulary of the domain. This is the heart of Domain-Driven Design; even a light version (ubiquitous language, entities vs value objects, aggregates) pays off — and it's exactly the thinking behind your JPA entity model (JPA reference §11).

*Go deeper:* your entity model with justified design decisions is JPA reference §11; the value-object/entity distinction is JPA reference §11.5.

---

# Part 5 — Data and databases

The layer where most real systems live or die. You have deep coverage already; this is the map and the parts your docs don't stress.

## 5.1 The relational model and SQL

Data as **tables** (relations) of rows and columns, linked by keys, queried with **SQL**, with **ACID** transactions. This is the default for a reason: decades of hardening, real constraints, real transactions, and a query planner that optimises for you. **SQL fluency is non-negotiable** for a backend engineer — `SELECT/JOIN/GROUP BY/HAVING/window functions`, and reading a query plan.

*Go deeper (you have this cold):* the whole stack, normalisation, keys, indexes, the clustered-index truth, and `EXPLAIN ANALYZE` are data-access primer Parts 1–4; how tables connect and index strategy is data-access §4.

## 5.2 ACID and transactions

**A**tomicity (all-or-nothing), **C**onsistency (valid state to valid state), **I**solation (concurrent transactions don't corrupt each other), **D**urability (committed = survives a crash). Isolation levels trade correctness for concurrency (read committed → serializable). A transaction is a SQL concept, not a framework one — the single idea that most repays understanding.

*Go deeper:* transactions as a SQL idea, isolation levels, MVCC, and how `@Transactional` maps to `BEGIN/COMMIT` are data-access primer §1.7 and §2.4.

## 5.3 Beyond relational — the NoSQL families

Not "newer and better" — *different shapes for different problems*, usually a **complement** to a relational source of truth:

| Family | Shape | Reach for it when | Example |
|---|---|---|---|
| **Document** | JSON-ish documents | flexible/nested schema | MongoDB, DynamoDB |
| **Key-value** | K→V, often in-memory | caching, sessions | Redis |
| **Wide-column** | huge write throughput | big-scale time-series/events | Cassandra |
| **Graph** | relationships first-class | traversals, recommendations | Neo4j |
| **Search** | full-text/analytics | search beyond `LIKE` | Elasticsearch |
| **Vector** | similarity over embeddings | AI/semantic search | pgvector, Pinecone |
| **Time-series** | timestamped metrics | monitoring | InfluxDB |

Two framings to carry: **polyglot persistence** (Postgres as truth + Redis for cache + a search index is normal), and the honest caution — **don't reach for NoSQL reflexively "for scale."** Postgres scales far, and leaving it costs you transactions, joins, and constraints. Note Postgres's `JSONB` and `pgvector` cover a lot of "I need NoSQL" *without* leaving the relational world.

*Go deeper:* the full landscape and where Practiq's choices sit is data-access primer §7.

## 5.4 The ideas that recur

- **Indexing** is the biggest performance lever (data-access §4.3). No index = full scan.
- **Normalisation** (each fact in one place) vs deliberate **denormalisation** (duplicate for read speed, measured).
- **Migrations** (Flyway) version your schema as code — schema is code, and it lives in version control.
- **Connection pooling** — a DB connection is an expensive TCP session; pool and reuse (data-access §1.6, networking §4).
- **N+1** — the classic ORM performance bug: 1 query + N follow-ups (data-access §2.8).

> **The tell — data:** the default is a relational database (Postgres). Add other stores only for a *named* problem the relational one handles badly, and keep one source of truth. When a query is slow, look at the plan and the indexes before anything else.

---

# Part 6 — Concurrency and parallelism

Genuinely hard, and where senior/junior diverges. **Concurrency** = dealing with many things at once (structure); **parallelism** = doing many things at once (execution, needs multiple cores). A single core can be concurrent (interleaving) but not parallel.

## 6.1 Why it's hard: shared mutable state

The entire difficulty reduces to one thing: **multiple threads reading and writing the same memory** (§1.3). Threads share their process's memory, so two threads touching one variable can interleave in ways that corrupt it.

- **Race condition:** the outcome depends on timing. `count++` is read-modify-write — two threads can both read 5, both write 6, and you've lost an increment.
- **Deadlock:** thread A holds lock 1 and wants lock 2; thread B holds lock 2 and wants lock 1. Both wait forever.
- **Visibility:** without synchronisation, one thread's write may never become visible to another (caches, reordering). `volatile` guarantees visibility (not atomicity).

## 6.2 The tools

| Tool | Gives | Note |
|---|---|---|
| **Immutability** | safety by construction | *the best answer* — no shared mutable state, no problem |
| `synchronized` / `ReentrantLock` | mutual exclusion | only one thread in the critical section |
| **Atomics** (`AtomicInteger`) | lock-free atomic ops | for simple counters/flags |
| `volatile` | visibility | *not* atomicity — `volatile i; i++` still races |
| `ConcurrentHashMap` | safe shared map | atomic `compute`/`merge` |
| **Executors / thread pools** | manage threads for you | don't hand-roll threads |
| `CompletableFuture` | compose async work | non-blocking pipelines |
| **Virtual threads** (Java 21) | cheap threads for I/O | makes blocking code scale (Modern Java §12.2) |

## 6.3 The mental model

Prefer, in order: **(1) no shared state** (immutability, message-passing) → **(2) share via safe abstractions** (`ConcurrentHashMap`, atomics) → **(3) explicit locks**, and only if you must. Most application concurrency should be the framework's problem, not yours — your job is to *not introduce shared mutable state* into it. The classic bug is a shared mutable field on a singleton service (Micronaut beans are singletons by default).

> **The tell — concurrency:** if you're reaching for `synchronized`, first ask whether the state can be immutable or thread-local instead. And "it worked on my machine / passed 1000 times" proves nothing about a race — concurrency bugs are timing-dependent and non-deterministic. Design them out; don't test them out.

*Go deeper:* threads → executors → `CompletableFuture` → virtual threads, with the primitives, is Modern Java primer §12.

---

# Part 7 — Distributed systems

The moment your system is more than one process on one machine, a new class of problem appears — and modern systems (Practiq on AWS) are all distributed.

## 7.1 The fallacies of distributed computing

The eight classic false assumptions that cause outages: *the network is reliable; latency is zero; bandwidth is infinite; the network is secure; topology doesn't change; there is one administrator; transport cost is zero; the network is homogeneous.* Every one is false. Distributed systems are the discipline of building reliable things on unreliable networks (networking §2.1: IP is best-effort).

## 7.2 CAP and consistency

**CAP theorem:** in the presence of a network **P**artition, you must choose between **C**onsistency (every read sees the latest write) and **A**vailability (every request gets a response). You can't have both during a partition. Relational databases lean CP; many NoSQL stores lean AP.

- **Strong consistency:** reads always see the latest write (a single Postgres). Simple to reason about, harder to scale.
- **Eventual consistency:** replicas converge *eventually*; a read might be stale briefly (DNS, DynamoDB, CDNs). Scales, but your code must tolerate staleness.

The honest note: CAP is often oversimplified. The practical version is "consistency vs availability/scale is a real trade, and you decide per-system how much staleness you can tolerate."

## 7.3 The building blocks

- **Idempotency:** an operation you can safely retry (networking §5.2). Critical, because in a distributed system *you will retry* — timeouts don't tell you whether the first attempt succeeded. Design writes to be idempotent (idempotency keys, upserts).
- **Message queues** (SQS, RabbitMQ): decouple producer from consumer, absorb load spikes, enable async work and retries. "Do this later / do this reliably / don't lose it if the consumer is down."
- **Event streaming** (Kafka): an append-only log many consumers read — the backbone of event-driven architectures.
- **Caching:** store expensive results closer/faster (Redis, CDN). The two hard problems: **invalidation** (when is it stale?) and **cache stampede** (everything misses at once). Patterns: cache-aside, write-through, TTLs.
- **Load balancing** (networking §10.2), **replication** (copies for reads/failover), **sharding** (split data across nodes for scale).
- **Consensus** (Raft/Paxos): how distributed nodes agree on a value despite failures — you'll rarely implement it, but it's what's under etcd, databases' leader election, etc.

## 7.4 Failure is the normal case

In one process, failure is exceptional. In a distributed system it's constant — a node *is* down, a packet *is* lost, right now. So you design for it: **timeouts** (never wait forever), **retries with backoff** (but only for idempotent ops, and with jitter to avoid stampedes), **circuit breakers** (stop hammering a failing dependency), **graceful degradation** (serve stale/partial rather than nothing), and **health checks** (so the LB routes away from the sick).

> **The tell — distributed:** assume every network call can be slow, fail, or succeed-but-you-never-hear. Every remote call needs a timeout. Every retry needs idempotency. The question is never "what if this fails" but "*when* this fails, what happens?"

---

# Part 8 — Architecture

How you slice a whole system into parts. Architecture is the set of decisions that are expensive to change later.

## 8.1 Layering

The classic split, each layer depending only on the one below: **presentation** (controllers/API) → **application/service** (use cases, transactions) → **domain** (business logic/entities) → **infrastructure** (DB, external services). The rule that makes it work: dependencies point *inward/downward*, and the domain doesn't know about the web or the database. **Hexagonal / ports-and-adapters** is the refined version: the domain defines interfaces (ports), and adapters (a Postgres repository, a REST controller) plug in — so you can swap the database or the delivery mechanism without touching business logic. This is Dependency Inversion (§3.3) at architectural scale.

## 8.2 Monolith vs microservices — the real trade

| | **Monolith** | **Microservices** |
|---|---|---|
| Structure | one deployable | many small services |
| Simplicity | high — one codebase, one deploy, easy local dev | low — network between everything, distributed-systems tax (§7) |
| Scaling | scale the whole thing | scale services independently |
| Team fit | small teams | many teams working independently |
| Failure | one process | partial failure, needs resilience patterns |
| Data | one database, real transactions | a DB per service, no cross-service transactions |

The honest, current consensus: **start with a monolith.** Microservices solve an *organisational* scaling problem (many teams shipping independently) at a large *technical* cost (you've turned method calls into network calls, and single-DB transactions into distributed sagas). Most teams that adopt microservices early get the costs without the benefits. Practiq's documented plan — **monolith-first with a documented target service-split** — is exactly right: modular internally, so you *can* split later, but not paying the distributed tax before you need to.

## 8.3 Communication styles

- **Synchronous** (REST/HTTP, gRPC): request-response, simple, but couples caller to callee's availability and latency.
- **Asynchronous** (queues, events): decoupled, resilient, scalable — but harder to reason about (eventual consistency, ordering, debugging across hops).
- **Event-driven:** components emit and react to events rather than calling each other directly. Loose coupling, but the flow is implicit and harder to trace.

## 8.4 Cross-cutting concerns

Things every part needs and no part owns: **auth**, **logging**, **error handling**, **config**, **validation**, **observability**. Handle them consistently (framework filters/interceptors, middleware) rather than scattering them.

> **The tell — architecture:** the best architecture is the *least* architecture that solves today's problem while not painting you into a corner. Favour a well-structured monolith with clean internal boundaries. You can always extract a service later; you can rarely un-distribute one cheaply. "We might need to scale" is not a reason to distribute now — it's a reason to keep boundaries clean.

---

# Part 9 — Correctness: testing and debugging

Code that isn't tested isn't trustworthy, and debugging is a skill, not a talent.

## 9.1 The testing pyramid

| Level | Tests | Speed | How many |
|---|---|---|---|
| **Unit** | one class/function in isolation | ms | lots (the base) |
| **Integration** | components together (e.g. code + real DB) | slower | fewer |
| **End-to-end** | the whole system through the UI/API | slow, flaky | few (the tip) |

More at the bottom (fast, precise, cheap), fewer at the top (slow, broad, brittle). The **anti-pattern** is the ice-cream cone: mostly slow E2E tests and few units — slow, flaky, and bad at localising failures. Practiq's **Testcontainers** integration tier (real Postgres, not H2) is the right call precisely because it tests against the engine you ship on (Docker primer §5; networking on dialect drift).

## 9.2 What makes a good test

- **Tests behaviour, not implementation** — it shouldn't break when you refactor internals without changing behaviour.
- **Arrange-Act-Assert** structure; one logical assertion per test.
- **Fast, isolated, repeatable, self-checking** (FIRST). A test that needs manual inspection or passes/fails randomly is worse than none.
- **The name states the scenario and expectation** (`approve_pendingQuestion_setsStatusApproved`).
- **Test the edges:** empty, null, boundary, error paths — not just the happy path.

## 9.3 TDD, briefly

Red (write a failing test) → green (make it pass, simply) → refactor (clean up, tests still green). Its real benefit isn't the tests — it's that it forces you to design the interface from the *caller's* perspective and keeps units small and testable. You don't have to be dogmatic; the discipline of "write the test first sometimes" sharpens design.

## 9.4 Test doubles

**Mock** (verify interactions), **stub** (canned responses), **fake** (a working lightweight implementation, e.g. in-memory), **spy** (a real object with some methods watched). Over-mocking is a smell — a test that mocks everything tests your mocks, not your code. Mock at *architectural boundaries* (the external API, the clock), use the real thing within them.

## 9.5 Debugging as a discipline

The method, not the panic: **reproduce** it reliably → **read the actual error** (the stack trace names the file and line; read it top to bottom) → **form a hypothesis** → **test it by changing one thing** → **narrow the search space** (binary-search the code/history; `git bisect` finds the breaking commit). Tools: the debugger (breakpoints beat print statements for state inspection), logs (§11.2), and the layered climb for infra (networking §11). Rubber-ducking — explaining it aloud — solves a shocking number of bugs.

> **The tell — correctness:** the question isn't "does it work?" but "how do I *know* it works, and how will I know when it stops?" Write the test that would have caught the bug. And when debugging, resist changing things at random — one hypothesis, one change, observe.

---

# Part 10 — Security

You don't need to be a security specialist, but a good engineer has a security *baseline* — the instinct to not build the obvious hole.

## 10.1 The mindset

**Never trust input.** Every byte from outside — user, API, file, another service — is hostile until validated. **Defence in depth** (layers, so one failure isn't fatal). **Least privilege** (every component gets the minimum access it needs — the DB user, the IAM role, the container). **Fail secure** (an error should deny, not grant).

## 10.2 The common vulnerabilities (OWASP-shaped)

| Vulnerability | Is | Defence |
|---|---|---|
| **Injection** (SQL, command) | untrusted input executed as code | **parameterised queries** (never string-concat SQL), input validation |
| **Broken auth** | weak login/session handling | strong session management, MFA, rate-limiting |
| **XSS** | attacker's JS runs in a victim's browser | escape output, CSP, `HttpOnly` cookies (networking §9.6) |
| **Broken access control** | you can access what you shouldn't | check authz on *every* request, server-side, per-object |
| **Sensitive data exposure** | secrets/PII leaked | encrypt in transit (TLS) and at rest; don't log secrets |
| **CSRF** | attacker's site acts as you | `SameSite` cookies, CSRF tokens (networking §9.6) |
| **Vulnerable dependencies** | a library has a known CVE | scan (Dependabot/OWASP), patch, pin |
| **Security misconfiguration** | defaults, open ports, verbose errors | harden, least privilege, don't leak stack traces |

## 10.3 The engineer's baseline habits

Parameterise every query. Hash passwords with **bcrypt/argon2** (never plain SHA, never plaintext; networking §6.3). Secrets in a **secret manager** or env vars, never in code or an image layer (Docker §7.6). **Validate and sanitise** all input. Keep dependencies patched. Use TLS everywhere. Give every component least privilege (a tightly-scoped IAM role, a DB user that can't `DROP`). Don't roll your own crypto or auth — use vetted libraries.

*Go deeper:* the crypto primitives, TLS, certificates, and the browser attack model (XSS/CSRF/CORS/CSP/HSTS) are networking primer Parts 6–9.

> **The tell — security:** most breaches aren't exotic — they're an unparameterised query, an unpatched dependency, a leaked secret, or a missing authz check. Get the baseline reflexes right and you've closed the doors attackers actually use. Assume you'll be attacked; make the easy attacks impossible.

---

# Part 11 — Operations: running things in production

Writing the code is half the job; the other half is shipping it and keeping it alive. "It works on my machine" is the beginning, not the end.

## 11.1 CI/CD

- **Continuous Integration:** every push runs the build and the tests automatically (GitHub Actions). Catch breakage in minutes, on every commit, not at release. The pipeline is the quality gate: build → test → lint → security-scan → package (the container image).
- **Continuous Delivery/Deployment:** automate the path to production so releases are boring and frequent. Small, frequent deploys are *safer* than big rare ones — less changes per deploy, so less to go wrong and easier to pinpoint. Deployment strategies: **blue-green** (two environments, switch traffic), **canary** (roll out to a small % first), **rolling** (replace instances gradually). All exist to make deploys reversible and low-risk.

## 11.2 Observability — the three pillars

You cannot fix what you can't see. Production is a black box without instrumentation:

| Pillar | Answers | Tool shape |
|---|---|---|
| **Logs** | "what happened at this moment?" | structured (JSON) logs, aggregated (CloudWatch, ELK) |
| **Metrics** | "how is the system behaving over time?" | numbers over time (Prometheus, CloudWatch), dashboards (Grafana) |
| **Traces** | "where did this request spend its time across services?" | distributed tracing (OpenTelemetry, X-Ray) |

**Structured logging** (key-value, not free text) is what makes logs searchable. Log at the right level (ERROR/WARN/INFO/DEBUG), include a **correlation/trace ID** so you can follow one request across components, and **never log secrets or PII**. Metrics to watch: the **four golden signals** — latency, traffic, errors, saturation.

## 11.3 When production breaks

- **Alerting** on symptoms users feel (error rate, latency), not on causes (CPU) — alert fatigue kills. 
- **On-call / incident response:** detect → mitigate (stop the bleeding — roll back, scale up) → diagnose → fix → **blameless post-mortem** (the failure is the *system's*, not a person's; the output is a change that prevents recurrence).
- **SLOs/SLIs:** define what "healthy" means numerically (99.9% of requests under 200ms) so you know when you're breaching it.
- **Config and secrets** live outside the artifact (env vars, secret managers) — the same image runs in every environment, configured differently (twelve-factor).

> **The tell — operations:** you don't understand a system until you've watched it run in production. Instrument first (you can't debug what you can't see), deploy small and often (reversible beats perfect), and treat every incident as a lesson the system should learn, not a person to blame.

---

# Part 12 — Cloud and AWS

The cloud is **someone else's computers, rented by the second, via an API.** The shift that matters: infrastructure becomes *programmable and elastic* — you request a server (or a database, or a queue) with an API call and pay for what you use.

## 12.1 The service-category model

AWS has ~200 services, but they're a handful of categories. Learn the categories; the names are lookups:

| Category | Does | Core AWS |
|---|---|---|
| **Compute** | run code | EC2 (VMs), **ECS/Fargate** (containers), Lambda (functions), EKS (Kubernetes) |
| **Storage** | store bytes | **S3** (object store — the workhorse), EBS (disks), EFS (file) |
| **Database** | managed data | **RDS/Aurora** (relational), DynamoDB (NoSQL), ElastiCache (Redis) |
| **Networking** | connect and route | **VPC**, **ALB/NLB**, **Route 53** (DNS), **CloudFront** (CDN), API Gateway |
| **Messaging** | decouple | **SQS** (queue), SNS (pub/sub), EventBridge (events), Kinesis (streams) |
| **Identity** | who can do what | **IAM** (the security backbone) |
| **Observability** | see what's happening | **CloudWatch** (logs/metrics), X-Ray (tracing) |
| **Registry/CI** | ship containers | **ECR** (image registry), CodeBuild/CodePipeline |
| **Secrets** | manage credentials | Secrets Manager, Parameter Store |

## 12.2 The concepts that underpin all of it

- **Regions and Availability Zones:** a **region** is a geographic location (eu-west-2 = London); an **AZ** is an isolated datacentre within it. You deploy across *multiple AZs* for resilience — an AZ can fail; a well-built system survives it.
- **The shared responsibility model:** AWS secures the cloud *itself* (hardware, the datacentre, the managed service internals); **you** secure what you put *in* it (your data, your access rules, your app, your IAM policies). Most breaches are the customer's side — a public S3 bucket, an over-permissive IAM role.
- **IAM is the backbone:** every action is governed by policies (who can do what to which resource). **Roles** (assumed by services, no stored credentials) beat **access keys** (long-lived, leakable). Least privilege here is the single highest-leverage security practice on AWS.
- **Managed vs self-hosted:** the core trade of cloud. RDS (managed Postgres) costs more per hour than running Postgres on an EC2 you manage — but AWS handles backups, patching, failover, and replication. You're buying *undifferentiated heavy lifting* back as time. Default to managed unless you have a strong reason.
- **Elasticity and pay-per-use:** scale up under load, down when idle, pay for what you use. This is the actual economic point of cloud — and the source of surprise bills (NAT gateways, egress bandwidth, idle resources).

## 12.3 Practiq on AWS

The documented path maps cleanly: `practiq-frontend` static assets on **S3 + CloudFront**; `practiq-api` container on **ECS Fargate** behind an **ALB** (TLS via **ACM**); **RDS Postgres** in a private subnet; images in **ECR**; DNS in **Route 53**; logs/metrics in **CloudWatch**; secrets in **Secrets Manager**; the whole network a **VPC** with public/private subnets (networking §10.3 draws this end to end). Serverless **Lambda** is the natural home for isolated jobs (the anonymous-attempt cleanup).

> **The tell — cloud:** default to **managed services** (buy back the ops time), deploy across **multiple AZs** (assume one fails), lock down **IAM to least privilege** (it's where breaches happen), and **watch the bill** (elasticity cuts both ways). Don't run your own Postgres/Redis/queue on a raw VM unless you can name why RDS/ElastiCache/SQS won't do.

---

# Part 13 — Infrastructure as Code

**Clicking around a cloud console doesn't scale and isn't reproducible.** IaC means defining your infrastructure — servers, networks, databases, permissions — in **version-controlled code** that you apply to create/update it. The same discipline you bring to application code (review, versioning, repeatability), applied to infrastructure.

## 13.1 Why it matters

- **Reproducible:** stand up an identical environment (dev/staging/prod) from the same code. No "works in staging, mystery in prod."
- **Version-controlled:** infra changes are diffs, reviewed in PRs, with history and blame. A bad change is a `git revert`.
- **Documented by definition:** the code *is* the source of truth for what exists — no drift between a wiki and reality.
- **Idempotent:** apply it ten times, get the same result. It describes the *desired state*; the tool works out the changes.

## 13.2 Terraform's model

Terraform is the dominant IaC tool, and its model is worth understanding regardless of which fork you use:

- **Declarative HCL:** you describe the *desired* end state (a VPC, an RDS instance, a security group), not the steps. Terraform computes the diff.
- **Providers:** plugins that talk to a platform's API (the AWS provider, the Postgres provider). Thousands exist.
- **Resources:** the things you declare (`resource "aws_db_instance" "practiq" { ... }`).
- **State:** Terraform keeps a **state file** mapping your code to real-world resources. This is the crucial, tricky part — state is the source of truth for "what exists," must be stored remotely and locked (an S3 backend + locking) for teams, and must never be hand-edited or lost.
- **The workflow:** `init` (set up) → **`plan`** (show me what will change — *always read this*) → **`apply`** (make it so) → `destroy` (tear down). The `plan`/`apply` split is the safety feature: you see the diff before it happens.
- **Modules:** reusable, parameterised bundles of resources — the "functions" of Terraform, for DRY infrastructure.

## 13.3 The landscape shift you must know (verified July 2026)

This is the one genuinely current thing: **Terraform is no longer straightforwardly open-source.** In **August 2023**, HashiCorp relicensed Terraform from the permissive MPL to the **Business Source License (BSL)** — source-available, but restricting building competing products on it. The community forked the last MPL version into **OpenTofu**, now a **Linux Foundation / CNCF** project with open governance. In **early 2025, IBM acquired HashiCorp** ($6.4B), so Terraform is now an IBM product. By 2026 the two have **meaningfully diverged** (OpenTofu added state encryption, provider-defined functions; Terraform added Stacks and deeper HCP integration), though they remain largely command- and config-compatible. **For internal use, Terraform's licence changes nothing** day-to-day; the fork matters mainly for vendors and for teams wanting neutral governance. For a solo project like Practiq, either works — the *model* above is identical.

## 13.4 Alternatives

| Tool | Approach | Reach for it when |
|---|---|---|
| **Terraform / OpenTofu** | declarative HCL, multi-cloud | the default; cloud-agnostic IaC |
| **CloudFormation** | declarative YAML/JSON, AWS-only | all-in on AWS, want native integration |
| **AWS CDK / Pulumi** | infra in a *real* language (TS/Python/Java) | developer-heavy teams who want loops/abstractions/types |
| **Ansible** | procedural, config management | configuring existing servers (not provisioning) |

> **The tell — IaC:** if you created it by clicking in a console, it doesn't really exist — it can't be reviewed, reproduced, or recovered. Define infra as code, store state remotely and locked, **always read the `plan`** before `apply`, and treat infra changes with the same review rigour as app code. For Practiq, Terraform/OpenTofu + a remote S3 state backend is the standard, correct choice.

---

# Part 14 — Front-end, for a backend engineer

You don't need to be a front-end specialist, but you should understand the thing your API talks to — and Practiq has a React frontend. This is the mental model, not a React tutorial.

## 14.1 How the browser works

The browser: **fetches** HTML/CSS/JS over HTTP → **parses** HTML into the **DOM** (a tree of elements) → applies **CSS** for layout and style → **runs JavaScript**, which can manipulate the DOM dynamically. The three languages: **HTML** (structure), **CSS** (presentation), **JavaScript** (behaviour). Everything else is built on these three.

## 14.2 The rendering-strategy spectrum — the key distinction

Where does the HTML get built? This is the decision that shapes a frontend:

| Strategy | HTML built | Trade |
|---|---|---|
| **SSR** (server-side rendering) | on the server, per request | fast first paint, SEO-friendly; server does work |
| **CSR** (client-side / SPA) | in the browser, by JavaScript | rich interactivity; slow first paint, JS-heavy, worse SEO |
| **SSG** (static site generation) | at build time | fastest, cheapest (just files on a CDN); only for content that isn't per-user |
| **Hybrid** (meta-frameworks) | mix per route | the modern default — SSG the marketing page, CSR the app |

## 14.3 The SPA / React model

A **Single-Page Application** loads once and then rewrites the page in JavaScript as you navigate, calling your API for data (JSON over HTTPS) rather than fetching new HTML pages. **React** is the dominant library for building these. Its core ideas:

- **Components:** the UI is a tree of reusable, composable components (functions that return markup). Composition, like good backend code (§4).
- **Declarative:** you describe what the UI *should look like* for a given state; React works out the DOM changes. (Same declarative spirit as SQL or Terraform — say the *what*, not the *how*.)
- **State and props:** **state** is a component's internal data; **props** are data passed in from a parent. UI = f(state) — when state changes, the view re-renders.
- **The virtual DOM:** React diffs a lightweight in-memory representation and applies only the minimal real-DOM changes (the DOM is slow to touch).
- **Build step:** browsers don't run React/JSX/TypeScript directly. A **build tool** (Vite is the current standard; Webpack the older one) transpiles and bundles it into plain JS/CSS/HTML. This is the frontend equivalent of compiling.

## 14.4 Where it meets your world

- **The API contract:** the frontend consumes your REST/JSON API. The contract (shapes, status codes, errors) is the boundary — design it for the caller (§4.4).
- **CORS:** the moment the frontend (`practiq.io`) and API (`api.practiq.io`) are different origins, the browser enforces CORS, and it's the *server's* headers that fix it (networking §9.5).
- **Auth:** session cookie vs token, and where the token lives (networking §9.3) — a security decision that spans both sides.
- **Meta-frameworks** (Next.js etc.) wrap React with routing, SSR/SSG, and build config — the "batteries-included" layer. Worth knowing they exist; not something you need for a backend role.

> **The tell — frontend:** as a backend engineer, know enough to design a good API for it, reason about rendering strategy and auth, and not be mystified by the build step. The frontend is a client that turns your JSON into pixels; understand the contract and the browser security model (networking §9), and leave the CSS to someone who enjoys it.

---

# Part 15 — Craft and judgment

The layer that separates competent from good, and none of it is about knowing more syntax. This is what experience actually is.

## 15.1 Everything is a trade-off

There are no right answers, only trade-offs with context. Speed vs correctness, simplicity vs flexibility, consistency vs availability, build vs buy, now vs later. The senior skill isn't knowing *the* answer — it's **naming the trade-off, weighing it against the actual context, and being able to explain the choice.** "It depends" is the correct start to most answers; the value is in what it depends *on*. Every "the tell" in these documents is a trade-off made explicit.

## 15.2 Simplicity is the goal, not the constraint

The instinct to fight: adding complexity to feel thorough. **The best engineers remove things.** YAGNI, KISS, the least architecture that works (§8). Complexity is a cost paid on every future change — abstractions, layers, and flexibility all have to earn their place. When in doubt, do the simpler thing; you can add complexity when a real need proves it, but you can rarely remove it cheaply.

## 15.3 Technical debt

Debt is a deliberate trade: ship faster now, pay interest later in slower changes. It's **not** the same as bad code — taking on debt knowingly, with a plan, is a legitimate tool. The danger is *unacknowledged* debt that compounds silently. Name it, track it, and pay it down when it's blocking you — not compulsively, and not never.

## 15.4 Pragmatism vs rigour — knowing which mode

A good engineer switches deliberately: **rigour** for the things that are hard to change or dangerous to get wrong (the data model, the security boundary, the public API); **pragmatism** for the things that are easy to change or low-stakes (an internal helper, a first draft, a spike). Spending equal care on both is a mistake in *both* directions. The judgment is "how expensive is this to change later?" — invest care in proportion to that.

## 15.5 Reading code, and working with others

- **You read far more code than you write.** The skill of quickly understanding an unfamiliar codebase — following the entry points, trusting the types, running it, not needing to understand everything at once — is undervalued and trainable.
- **Code review** is about the code, not the person; be specific and kind; explain the *why*; distinguish "this is wrong" from "I'd prefer." Receiving it: it's about the code, not you.
- **Communication is the actual bottleneck** at senior level. The ability to explain a trade-off, write a clear design doc, or say "I don't know yet" is worth more than raw output. Your documented decision-log discipline (discussion ≠ adoption; confirm before writing it down) is exactly this skill.

## 15.6 Working sustainably

The failure mode worth naming directly: **all-or-nothing effort leads to burnout cycles**, and burnt-out engineers produce worse work, not more. Sustainable practice is an engineering skill, not a soft one — the systems that survive are the ones a tired person can still operate safely (small deploys, good tests, clear docs, low-ceremony processes). Build for the tired future version of yourself: the well-named function, the test that localises the failure, the doc that answers the question at 5pm on a Friday. **Consistent, moderate pace beats heroic sprints** — for the code, and for you.

## 15.7 Learning, forever

The field moves; the fundamentals don't. This whole document is fundamentals — machines, data structures, coupling, transactions, trade-offs — and they'll still be true in a decade when today's frameworks are gone. Invest most in the durable layer (the *why*), learn tools as you need them (the *what*), and stay comfortable saying "I don't know, let me find out." The engineers who last are the ones who kept the foundations sharp and treated every framework as temporary.

> **The tell — judgment:** when you don't know what to do, ask three questions: *What's the actual trade-off here? What's the simplest thing that could work? How expensive is this to change later?* Those three resolve most decisions. And when you catch yourself adding complexity to feel thorough, or sprinting toward burnout to feel productive — stop. Good engineering is calm, simple, and sustainable.

---

# Where to go from here

- **Depth on the topics you have:** the six documents indexed at the top. This map points into them throughout.
- **Where the depth now lives:** Linux and the command line is №52, Git is №53, and the tool catalogue is №90. This map predates all three and points at them rather than repeating them.
- **The framing to carry out of this document:** §15.1. Almost every useful answer to "how would you design or choose X?" has the same shape — *name the options, name the trade-off, make a justified call.* A map is only worth having if it changes what you do at the junctions.

*Caveat: this is a breadth-first map written from knowledge; the fundamentals are stable. The one high-drift fact — Terraform's licensing and the OpenTofu/IBM situation (§13.3) — was verified July 2026. For any specific current tool version or AWS behaviour, check before relying on it, and the tool-catalogue doc will carry that verification.*
