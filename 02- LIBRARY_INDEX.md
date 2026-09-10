# The Library Index — №02

*The filing system for the engineering documentation set. A banded decimal scheme where the number encodes the topic area, with gaps left for growth. This is the authoritative catalogue — every document, existing and planned, has a reserved number here. Referenced by number going forward (e.g. "see №20 §2.4").*

## The scheme

Documents are grouped into **bands of ten by domain**. The tens digit tells you the area at a glance; the units number individual docs within it, in rough reading order. Gaps are deliberate — new docs slot into their band without renumbering anything.

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
| **90–99** | Catalogues & reference |

Two docs that genuinely span bands are filed by their **tightest pairing, not a purist taxonomy**: Concurrency (№12) sits with Java because that's its delivery vehicle, though it's also a CS foundation; Networking (№51) sits with Docker because they're heavily cross-linked, though it's also a systems foundation. Cross-references handle the overlaps.

## The catalogue

**Status:** ✅ built · 🔲 planned. Roadmap wave in the last column (see №01).

### 00–09 · Meta & orientation

| № | Title | Status | Wave |
|---|---|---|---|
| **00** | The Engineer's Map | ✅ | — |
| **01** | The Library Roadmap | ✅ | — |
| **02** | The Library Index *(this doc)* | ✅ | — |

### 10–19 · Java language & platform

| № | Title | Status | Wave |
|---|---|---|---|
| **10** | Modern Java Primer | ✅ | — |
| **11** | Java Collections Reference | ✅ | — |
| **12** | Concurrency Primer | ✅ | — |
| **13** | JVM Internals, Performance & Profiling | 🔲 | 4 |
| **14** | Frameworks & Dependency Injection | 🔲 | 4 |

### 20–29 · Data & persistence

| № | Title | Status | Wave |
|---|---|---|---|
| **20** | Java Data-Access Primer | ✅ | — |
| **21** | JPA & Hibernate Reference | ✅ | — |
| **22** | SQL Mastery & Database Internals | 🔲 | 4 |

### 30–39 · CS foundations

| № | Title | Status | Wave |
|---|---|---|---|
| **30** | Algorithms, Data Structures & Patterns | ✅ | 1 |
| **31** | Distributed Systems | 🔲 | 1 |
| **32** | Operating Systems & How Computers Work | 🔲 | (opt) |

### 40–49 · Software craft & design

| № | Title | Status | Wave |
|---|---|---|---|
| **40** | Clean Code & Refactoring | 🔲 | 2 |
| **41** | Software Design & Patterns | 🔲 | 2 |
| **42** | Architecture | 🔲 | 2 |
| **43** | System Design | 🔲 | 1 |
| **44** | Testing & Correctness | 🔲 | 1 |
| **45** | APIs & Interface Design | 🔲 | 4 |

### 50–59 · Infrastructure, tooling & ops

| № | Title | Status | Wave |
|---|---|---|---|
| **50** | Docker Primer | ✅ | — |
| **51** | Networking Primer | ✅ | — |
| **52** | Linux & the Command Line | 🔲 | 3 |
| **53** | Git | 🔲 | 2/3 |
| **54** | Cloud & AWS | 🔲 | 3 |
| **55** | Infrastructure as Code (Terraform/OpenTofu) | 🔲 | 3 |
| **56** | CI/CD & DevOps | 🔲 | 3 |
| **57** | Observability & Production Operations | 🔲 | 3 |

### 60–69 · Security

| № | Title | Status | Wave |
|---|---|---|---|
| **60** | Application Security | 🔲 | 3 |

### 70–79 · Frontend

| № | Title | Status | Wave |
|---|---|---|---|
| **70** | Frontend for Backend Engineers | 🔲 | 4 |

### 90–99 · Catalogues & reference

| № | Title | Status | Wave |
|---|---|---|---|
| **90** | The Tool Cards | 🔲 | parallel |

## Conventions going forward

- **Filenames** carry the number as a prefix: `NN-TITLE.md` — so the directory sorts into band order. E.g. the next planned doc would be `31-DISTRIBUTED_SYSTEMS_PRIMER.md`.
- **References** use the number: "covered in №20 §2.4" rather than the full title. The map (№00) and roadmap (№01) will resolve their "go deeper" pointers to numbers as docs land.
- **New topics** slot into the right band at the next free number; the gaps mean nothing ever has to be renumbered.
- **Status** here is the single source of truth for what's built — updated as each doc lands.

## The existing nine — filename alignment

The nine built docs were delivered before this scheme existed, so their current filenames don't carry numbers. Their **numbers are assigned above regardless** — the catalogue is authoritative. If you'd like the actual files renamed to `NN-...` so the filesystem matches, say so and I'll re-issue them with numbered filenames in one pass. Otherwise, new docs get numbered filenames from here on and the index maps the old ones.

## Current progress

**9 of ~29 built** — the foundations are dense. Built: №00, 01, 02, 10, 11, 12, 20, 21, 30, 50, 51 *(that's 11 counting the three meta docs; 8 subject primers + 3 meta)*. Next by roadmap: **№31 Distributed Systems** and **№43 System Design** (Wave 1), then the Wave 2 craft docs.
