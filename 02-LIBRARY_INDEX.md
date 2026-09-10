# The Library Index — №02

*The catalogue and filing system for the engineering documentation library. **31 documents.** The JVM-stack coverage is complete; the .NET band (80–89) opened in September 2026 and is the live edge. Regenerated when documents land.*

## The scheme

Banded decimal: the tens digit encodes the domain, gaps left so new documents slot in without renumbering. Filenames carry the number as a prefix (`NN-TITLE.md`) so the directory sorts into band order.

| Band | Domain |
|---|---|
| **00–09** | Meta & orientation |
| **10–19** | Java language & platform |
| **20–29** | Data & persistence |
| **30–39** | CS foundations |
| **40–49** | Software craft & design |
| **50–59** | Infrastructure, tooling & ops |
| **60–69** | Security |
| **70–79** | Frontend |
| **80–89** | .NET & C# |
| **90–99** | Catalogues & reference |

Three documents are filed by **tightest pairing rather than purist taxonomy**: **№12 Concurrency** sits with Java because that's its delivery vehicle (though it's also a CS foundation), **№51 Networking** sits with infrastructure because it's heavily cross-linked with Docker (though it's also a systems foundation), and **№80 C# Threading** sits in the .NET band rather than beside №12, because what it teaches is the platform rather than the concept. Cross-references handle the overlap.

---

## The catalogue

### 00–09 · Meta & orientation

| № | Title | Covers |
|---|---|---|
| **00** | The Engineer's Map | breadth-first overview of the whole field; the front door, points into everything else |
| **01** | The Library Roadmap | the build plan and sequencing (historical — now superseded by this index) |
| **02** | The Library Index | *this document* — the scheme and catalogue |
| **03** | The Study Syllabus | *reserved, not written* — every topic and subtopic as a flat outline, for study tracking |

### 10–19 · Java language & platform

| № | Title | Covers |
|---|---|---|
| **10** | Modern Java Primer | the delta since Java 8: records, sealed types, pattern matching, streams, `java.time`, generics, JVM basics, release history to 27 |
| **11** | Java Collections Reference | the framework, `HashMap` internals, sorting, `equals`/`hashCode`, immutability, concurrency, the legacy classes |
| **12** | Concurrency Primer | the memory model and happens-before, the bug taxonomy, locks and atomics, deadlock, why you can't test concurrency into correctness |
| **13** | JVM Internals, Performance & Profiling | class loading, bytecode, the JIT, memory layout, GC in depth, JFR and flame graphs, JMH, container sizing |
| **14** | Frameworks & Dependency Injection | IoC and DI, bean lifecycle and scopes, Micronaut compile-time vs Spring runtime, AOP and proxies, the self-invocation trap |

### 20–29 · Data & persistence

| № | Title | Covers |
|---|---|---|
| **20** | Java Data-Access Primer | the full stack JDBC→JPA→Hibernate→SQL, transactions, the persistence context, N+1, the query APIs, indexing, query plans |
| **21** | JPA & Hibernate Reference | every consumer-facing annotation and runtime class, a worked entity model with justified design, collections in JPA |
| **22** | SQL Mastery & Database Internals | joins, windows, CTEs, upserts; storage, WAL, MVCC, vacuum, index internals, the planner |

### 30–39 · CS foundations

| № | Title | Covers |
|---|---|---|
| **30** | Algorithms, Data Structures & Patterns | Big-O, the structure→cost mapping, core algorithms, the thirteen problem-solving patterns, interview method |
| **31** | Distributed Systems | the fallacies, CAP/PACELC, consistency models, replication and partitioning, consensus, idempotency, sagas, caching, resilience patterns |

### 40–49 · Software craft & design

| № | Title | Covers |
|---|---|---|
| **40** | Clean Code & Refactoring | naming, functions, SOLID decoded, DRY/YAGNI with nuance, code smells, the refactoring moves, code review |
| **41** | Software Design & Patterns | OOP vs FP, coupling and cohesion, DI, the GoF patterns worth knowing, anti-patterns, DDD's useful subset |
| **42** | Architecture | layering, hexagonal and the dependency rule, monolith vs microservices, communication styles, ADRs, fitness functions |
| **43** | System Design | the interview method, estimation, the building blocks, the scaling ladder, worked designs, trade-off vocabulary |
| **44** | Testing & Correctness | the pyramid, what makes a good test, test doubles, TDD as design, property-based testing, debugging as a discipline |

### 50–59 · Infrastructure, tooling & ops

| № | Title | Covers |
|---|---|---|
| **50** | Docker Primer | containers vs VMs, namespaces/cgroups/union FS, images and the writable layer, volumes, Dockerfiles, the alternatives |
| **51** | Networking Primer | OSI/TCP-IP, IP and subnets, DNS, TCP/UDP, HTTP 1.1–3, crypto foundations, TLS, certificates, cookies and CORS, load balancers |
| **52** | Linux & the Command Line | the process model, permissions, the shell, grep/sed/awk, system inspection, systemd, scripting, troubleshooting |
| **53** | Git | the object model, branches as labels, merge vs rebase, undoing things, conflicts, bisect, branching strategies |
| **54** | AWS Primer (SAA-C03 & DVA-C02) | exam-oriented: domains, the services by category, decision frameworks, question patterns, a study plan |
| **55** | Infrastructure as Code | why IaC, the Terraform model, state in depth, modules, environments, testing, the licensing landscape |
| **56** | CI/CD & DevOps | CI discipline, delivery vs deployment, pipeline anatomy, GitHub Actions, deployment strategies, feature flags, expand/contract migrations, DORA |
| **57** | Observability & Production Operations | monitoring vs observability, logs/metrics/traces, OpenTelemetry, golden signals, SLOs and error budgets, alerting, incident response |

### 60–69 · Security

| № | Title | Covers |
|---|---|---|
| **60** | Application Security | the mindset, injection, authentication, authorisation and IDOR, OAuth2/OIDC/JWT, validation and encoding, secrets, supply chain, threat modelling |

### 70–79 · Frontend

| № | Title | Covers |
|---|---|---|
| **70** | Frontend for Backend Engineers | the browser model, rendering strategies, React, state management, the build toolchain, the API and auth boundary |

### 80–89 · .NET & C#

| № | Title | Covers |
|---|---|---|
| **80** | C# Threading and Asynchrony | the thread pool and starvation, `Task`, the `async`/`await` state machine, cancellation, the .NET memory model and atomics, locks and channels, parallelism, ASP.NET Core practice |

### 90–99 · Catalogues & reference

| № | Title | Covers |
|---|---|---|
| **90** | The Tool Cards | ~110 tools across 17 problem categories: what each is, when to reach for it, when not, AWS equivalent |
| **91** | AWS Service Reference | ~60 AWS services at even depth, mechanism-first: what it is, how it actually works, key concepts, gotchas |

---

## Reading paths

**Start here:** №00 (the map) orients you and points into everything else.

**For interview preparation:** №30 (algorithms and patterns) → №43 (system design) → №31 (distributed systems) → then the language and data documents (№10, №11, №12, №20, №21) for the technical rounds.

**For AWS certification:** №54 (exam-oriented, with a study plan) paired with №91 (understanding the services), supported by №51 (networking) and №50 (containers).

**For daily craft:** №40 → №41 → №42, in that order — function level, then object level, then system level.

**For the production path:** №50 → №56 → №55 → №57, following an artifact from image to pipeline to infrastructure to operation.

**For the .NET stack:** №12 (the concepts) → №80 (what .NET actually does with them) → №14 (container lifetimes, which §9.1 of №80 reads as a concurrency contract). The band is thin by design so far; №80 closes with the build order for the rest.

**When something's broken:** most documents open with a symptom index. №52 §11 (Linux), №51 §11 (network), №57 §10 (incident response), №44 §9 (debugging method) and №80's symptom index (.NET concurrency) are the diagnostic entry points.

---

## Conventions

- **References by number:** "covered in №20 §2.4" rather than by title.
- **Every document** carries a symptom or decision index near the top, "the tell" callouts at each decision point, a *when to use what* section at the end, and cross-references into the rest of the library.
- **Volatile facts were verified** at the time of writing. Each document's closing note states what is stable and what should be re-checked.
- **New topics** slot into the appropriate band at the next free number; the gaps mean nothing needs renumbering.

**Filename drift.** The convention is `NN-TITLE.md` with no spaces around the hyphen, but ten of the earlier files use `NN - TITLE.md`: №00, 01, 10, 11, 12, 20, 21, 30, 50, 51. They sort correctly but do not match on a strict prefix, which will bite any script that parses the number. Worth a single renaming pass.

---

## Status

**31 documents.** Four meta (one of them, №03, reserved but unwritten), twenty-five subject primers, two catalogues.

Two originally-planned documents were **folded** rather than built: **№32 Operating Systems** (covered by №52 and №00 §1) and **№45 API Design** (covered within №41 §4.4, №51 §5 and №70 §7).

The JVM-stack coverage is complete and further work there is expansion rather than gap-filling. **The live gap is the .NET band**, opened by №80 in September 2026: №81 (C# and .NET), №82 (ASP.NET Core), №83 (EF Core), №84 (Azure) and №85 (the .NET runtime) are named and unwritten, in that order of payoff. №80 §"The .NET band" states the case for each.
