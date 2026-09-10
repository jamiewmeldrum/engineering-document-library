# The Library Roadmap

*The plan for building out the engineering documentation set. What exists, what's missing, and the order to build it in. A living planning doc — reorder freely; each backlog entry is a ready-to-run task you can set me on when you have the tokens.*

## The shape of the thing

The model we've landed on: **the Engineer's Map is the index; everything else is a deep primer** (teaches one topic properly) **or the one catalogue** (the tool cards). The map orients and connects; the primers fill the gaps. Ultimately every topic in the map wants its own primer — this roadmap is the ordered path to that, not a suggestion that some topics don't deserve depth.

Two honest framing notes up front:

- **Interview timing.** If the early-August application window still holds, note that your interview *foundation is already substantial* — the six-plus docs below cover most of what a mid/senior Java screen actually probes. The remaining interview-critical gaps are narrow and named (Wave 1). You're not starting from zero on interviews; you're closing specific gaps.
- **Pace.** This is a backlog to work down over months, not a fortnight. You can't absorb twenty primers in two weeks even if I could write them that fast — so the sequence front-loads the highest-leverage ones and lets the rest wait. Build one, actually read it, then pull the next.

---

## Where we are — the library today

Eight documents, ~55,000 words. This is already a serious foundation.

| Doc | Type | Covers | Interview surface |
|---|---|---|---|
| **Engineer's Map** | index/map | the whole field, breadth-first; the front door | judgment, tradeoffs |
| **Java data-access primer** | primer (L) | JDBC→JPA→Hibernate→SQL, transactions, indexing, query plans | SQL, DB, ORM, transactions |
| **JPA & Hibernate reference** | reference (M) | every consumer annotation/class + a worked entity model | ORM, data modelling |
| **Java Collections reference** | reference (M) | the framework, HashMap internals, sorting, concurrency | data structures, Java |
| **Modern Java primer** | primer (L) | the language since 8, JVM, GC, packaging, release history | Java, JVM, GC |
| **Docker primer** | primer (L) | containers, images/layers, runtime stack, Dockerfiles | containers, ops |
| **Networking primer** | primer (L) | OSI/TCP-IP, TCP/UDP, DNS, HTTP, TLS, certs, cookies | networking, security |
| **Concurrency primer** | primer (M) | the CS foundation: memory model, races, coordination, patterns | concurrency, threading |

**Interview topics already well-covered:** Java language, collections/data-structures, concurrency, SQL/transactions/ORM, networking, containers. That's a lot of a Java-backend screen already in the bank.

---

## The backlog

Every entry: **what it covers · why it matters · size · dependencies.** Sizes match what I've produced — S ≈ 2–3k words, M ≈ 4–6k (concurrency/networking), L ≈ 7–10k (data-access/Java), XL = the catalogue.

### Tier 1 — Interview-critical foundations

The narrow set of remaining gaps that a mid/senior interview actually tests, and the hardest CS foundations. Highest priority.

**1. Algorithms, Data Structures & Problem-Solving Patterns** · **L** · no deps
The formal-CS core you never drilled, done practically. Big-O with real intuition, the data-structure→cost mapping, and the *interview patterns as patterns to recognise*: hashing for lookups, two-pointers, sliding window, BFS/DFS, binary search, recursion/backtracking, dynamic-programming-as-memoisation, greedy. Not a LeetCode grind — the mental toolkit that makes the grind tractable. **The single biggest interview lever you don't yet have.**

**2. System Design** · **L** · soft-deps on Distributed Systems, Architecture (but can stand alone)
The mid/senior interview round, and one that plays directly to your strength (breadth at the system level). The method: requirements → back-of-envelope estimation → API → data model → scale-out → the tradeoffs, worked through canonical problems (URL shortener, feed, rate limiter, chat). Synthesises data + distributed + architecture into an *answer structure*. **High value, time-sensitive.**

**3. Distributed Systems** · **M/L** · no hard deps
The genuinely hard foundation the map only sketched. CAP done properly, consistency models (strong/eventual/causal), idempotency, message queues vs streaming, caching strategies and their two hard problems, replication/sharding/partitioning, consensus (why it's hard, not how to implement), and the failure patterns (timeouts, retries+backoff, circuit breakers, bulkheads). Feeds System Design directly.

**4. Testing & Correctness** · **M** · no deps
Directly your daily Practiq work *and* interview-relevant. The pyramid and why the ice-cream cone hurts, what makes a good test (behaviour not implementation, FIRST), test doubles and over-mocking, TDD as a design tool, property-based testing, and debugging as a *method* not a panic. Grounds against your Testcontainers/three-tier setup.

### Tier 2 — Everyday craft

What makes you better every day and shows in code review. Less interview-flashy, more career-durable.

**5. Clean Code & Refactoring** · **M** · no deps
Naming, functions, guard clauses, the SOLID principles decoded (not recited), DRY/YAGNI/KISS with the *nuance* (when DRY is wrong), code smells → refactoring moves, and the red-green-refactor discipline. The craft of code that's cheap to change.

**6. Software Design & Patterns** · **M/L** · soft-dep on Clean Code
OOP vs FP (and why modern code is both), coupling/cohesion as the master idea, the patterns that earn their keep (Strategy, Factory, Builder, Adapter, Observer, Decorator, DI) with the anti-patterns, and a light Domain-Driven Design (entities/value objects/aggregates/ubiquitous language) — the thinking behind your JPA entity model.

**7. Architecture** · **M/L** · soft-deps on Design, Distributed
Layering and hexagonal/ports-and-adapters, the monolith-vs-microservices trade done honestly (your monolith-first-with-documented-split is the case study), sync vs async vs event-driven communication, cross-cutting concerns, and how to make decisions that are expensive to reverse. Feeds System Design.

**8. Git** · **M** · no deps
You flagged this. The *model* first (commits as snapshots, branches as pointers, HEAD, the DAG) — get that and the commands stop being incantations. Then branching strategies, merge vs rebase (and when each), interactive rebase, resolving conflicts, `bisect`/`reflog`/`cherry-pick`, and undoing the mess. Daily tool, occasional interview question.

**9. Linux & the Command Line** · **M/L** · no deps
You flagged this. Processes and signals, the permission model, the filesystem, the shell (pipes, redirection, globbing), the essential tools (`grep`/`sed`/`awk`/`find`/`ss`/`ps`/`top`), systemd, and enough to be dangerous in a container or on a box. Underpins Docker, ops, and debugging; a genuine interview topic.

### Tier 3 — Systems, ops & infra

The production path — Practiq-relevant and the difference between "writes code" and "ships and runs software."

**10. Cloud & AWS** · **L** · no hard deps
The service-category model deep, IAM properly (the backbone), VPC/networking, the compute options (EC2/ECS/Fargate/Lambda/EKS) and when each, storage/database options, the shared-responsibility model, cost awareness, and well-architected principles. Maps Practiq's actual target stack.

**11. Infrastructure as Code (Terraform/OpenTofu)** · **M/L** · soft-dep on Cloud
The declarative model, state (and why it's the hard part), providers/resources/modules, plan/apply discipline, remote state + locking, the testing/CI story for infra, and the current licensing/fork landscape. Practiq uses Terraform — this is directly applicable.

**12. CI/CD & DevOps** · **M** · soft-deps on Docker, Cloud
Pipelines as quality gates, GitHub Actions concretely (your CI), build→test→scan→package→deploy, deployment strategies (blue-green/canary/rolling), artifact/registry flow (→ECR), environment promotion, and the twelve-factor config discipline.

**13. Observability & Production Operations** · **M** · soft-dep on Cloud
The three pillars (logs/metrics/traces) properly, structured logging, the four golden signals, dashboards and alerting-on-symptoms, SLOs/SLIs, incident response and blameless post-mortems, and OpenTelemetry. "You don't understand a system until you've watched it run."

**14. Application Security** · **M** · dep on Networking (has the crypto/TLS)
Where Networking covered the wire (crypto, TLS, certs, cookies), this covers the *app*: OWASP Top 10 with defences, authentication vs authorisation, OAuth2/OIDC/JWT flows drawn out, session management, secrets handling, dependency/supply-chain security, and the security mindset (never trust input, least privilege, defence in depth).

### Tier 4 — Deeper platform & specialisation

Valuable, Practiq-adjacent, and satisfying — but lower urgency. Build when the earlier tiers are done or when a specific need pulls one forward.

**15. JVM Internals, Performance & Profiling** · **M/L** · dep on Modern Java
Beneath Modern Java §13: class loading, the JIT (C1/C2, tiered compilation, warmup), the GC algorithms in depth and how to read a GC log, memory layout, escape analysis, profiling (async-profiler, JFR), benchmarking properly (JMH — so performance claims are measured), and container tuning. For when you want to *reason about* performance, not guess.

**16. Frameworks & Dependency Injection** · **M** · soft-deps on Design, Modern Java
How the magic works: the IoC container, DI properly, Micronaut's compile-time vs Spring's runtime approach (and why that matters for startup/native-image), AOP/interceptors, bean scopes and the singleton-mutable-state trap, and configuration. Demystifies the framework you build Practiq on.

**17. SQL Mastery & Database Internals** · **M/L** · complements data-access primer
Two halves. SQL *fluency* beyond the ORM: joins in depth, window functions, CTEs, aggregation, set operations, the tricks. And database *internals*: B-tree/heap storage, the WAL, MVCC mechanics, how the query planner actually decides, isolation implementation. Deepens what data-access started.

**18. APIs & Interface Design** · **M** · soft-dep on Networking
REST properly (resources, the maturity model, idempotency, pagination, versioning, error design), gRPC and GraphQL and when each, contract-first design, and the "make it hard to misuse" philosophy. The design layer above the HTTP protocol Networking covered.

**19. Frontend for Backend Engineers** · **M** · soft-dep on Networking
Deeper than the map's sketch: the browser rendering pipeline, the React model (components/state/props/virtual DOM/hooks), rendering strategies (SSR/CSR/SSG/hybrid), the build toolchain, state management, and the API/auth boundary. *Honest note: this is the one most defensible to leave at map-depth* — you're backend, and the map may be enough unless Practiq's frontend pulls you in.

### The Catalogue track (runs in parallel)

**20. The Tool Cards ("dating cards")** · **XL** · no deps
The reference catalogue — one tight card per tool: *what it is · the problem it solves · when to reach for it / when not · main alternatives · AWS managed equivalent · pairs-with.* Grouped by problem-category (relational DBs, the NoSQL families, caching, queues/streaming, containers/orchestration, IaC, CI/CD, web servers/proxies, observability/dashboarding, profiling/load-testing, build tools, VCS/hosting, secrets, auth/identity, cloud platforms, data/ETL). ~80–120 tools that genuinely earn a card, with the volatile facts (current status, AWS equivalents) web-verified.

This one is structurally different — a catalogue, not a primer — so it makes a good **change of mode**: something to build in a lighter register between the heavier primers, or in chunks (one category at a time). It doesn't block anything and can slot in whenever.

---

## Recommended sequence

Ordered by leverage: interview-critical + hardest-foundations first, then daily craft, then the production stack, then specialisation. The catalogue floats.

| Wave | Docs | Why this order |
|---|---|---|
| **Wave 1 — interview & core foundations** | Algorithms → System Design → Distributed Systems → Testing | closes the exact remaining interview gaps and the hardest CS foundations; Distributed feeds System Design, so ideally do it around the same time |
| **Wave 2 — the craft** | Clean Code → Design & Patterns → Architecture → Git | how you write and structure code day-to-day; Architecture retro-feeds System Design; Git is a quick, high-utility win to slot in anytime |
| **Wave 3 — daily tools & the production stack** | Linux → CI/CD → Observability → Cloud/AWS → IaC → Application Security | the ship-and-run layer; heavily Practiq-relevant; Linux underpins the rest so it leads |
| **Wave 4 — deeper platform** | JVM Internals → Frameworks/DI → SQL & DB Internals → API Design → (Frontend, if wanted) | specialisation and depth; pull any one forward the moment a real need appears |
| **Parallel** | Tool Cards | slot in as a lighter-register break, whole or by category |

Two sensible deviations from strict order:
- **Pull Git forward** to anytime — it's small, daily, and independent.
- **Pull a Wave-3/4 doc forward** if Practiq or an interview creates a concrete need (e.g. you're about to write Terraform → do IaC now; a system-design round is scheduled → do System Design and Distributed first, ahead of everything).

---

## What might legitimately stay lighter

Honest de-prioritisation, so we don't over-build lookup-shaped content:

- **Frontend (#19)** — map-depth may genuinely suffice for a backend role; build the full version only if Practiq's frontend work demands it.
- **Cloud (#10) and IaC (#11)** — partly lookup-shaped; the Tool Cards will cover the *what-tool-for-what*, so these primers can focus on the *concepts and decisions* rather than exhaustive service catalogues.
- **API Design (#18)** could fold into Software Design (#6) if you'd rather one doc than two.

Everything else genuinely wants the full treatment, as you said.

---

## How to use this

- **It's a menu, not a mandate.** Each entry is a self-contained task. Say "build the algorithms one" (or whichever) and I'll produce it at the concurrency-primer depth, in the house style, with a symptom index, "the tell" callouts, Practiq examples, interview callouts, and pointers into the rest of the library.
- **Reorder whenever.** Your priorities (an interview scheduled, a Practiq sprint needing Terraform) should override the default sequence.
- **One at a time.** Build it, read it, then pull the next — the reading is the point, and the backlog isn't going anywhere.
- **The map stays the index.** As each primer lands, it slots under the relevant map part, and the map's "go deeper" pointers start resolving to real documents instead of promises.

**Target end state:** ~28 documents — one map, one catalogue, ~26 deep primers — covering the field from the memory hierarchy to system design, each at the depth of your best current docs. A genuine self-taught-to-solid curriculum, built at a sustainable pace, mostly reusable long after any one interview.

When you've got the tokens, just name the next one. My vote for first: **Algorithms, Data Structures & Problem-Solving Patterns** — it's the biggest interview lever you don't already have.
