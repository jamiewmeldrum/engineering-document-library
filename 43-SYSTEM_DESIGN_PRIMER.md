# System Design — A Primer №43

*The interview round that plays to breadth rather than syntax, and the discipline behind every architecture decision you'll make on the job. Where №31 (Distributed Systems) gives you the **components and their physics**, this gives you the **method**: how to take a vague prompt like "design Twitter" and turn it into a structured, defensible design in 45 minutes. Prerequisite: №31. Companion: №42 (Architecture, planned).*

The thing nobody tells you: **there is no right answer, and the interviewer knows it.** A system-design round is not a test of whether you've memorised Twitter's architecture. It's a test of whether you can handle ambiguity, ask the right questions, reason about scale with numbers, make trade-offs explicitly, and communicate all of it clearly. The candidates who fail are rarely the ones who chose the "wrong" database — they're the ones who dive into details without scoping, design for a billion users when asked for a thousand, or go silent.

The one habit that carries the entire round: **narrate a structured process.** Requirements → numbers → API → data → architecture → deep dive → trade-offs. Even if your design is imperfect, walking that path visibly is most of the assessment.

Contents:

- **Part 1** — the method: a time-boxed framework
- **Part 2** — requirements and scoping
- **Part 3** — back-of-envelope estimation (and the numbers to know)
- **Part 4** — API and data model
- **Part 5** — the building blocks
- **Part 6** — the scaling toolkit, in order
- **Part 7** — worked example: a URL shortener, end to end
- **Part 8** — more worked designs: rate limiter, news feed, chat
- **Part 9** — designing your own system (Practiq)
- **Part 10** — the trade-off vocabulary
- **Part 11** — what's actually being assessed, and how it goes wrong

## Requirement → technique index

The lookup that turns a stated need into a design move.

| The requirement… | Reach for | §|
|---|---|---|
| "read-heavy, same data repeatedly" | cache (CDN → Redis → local) | §6.3 |
| "must survive a machine dying" | replication + multi-AZ + health checks | §6.5 |
| "more writes than one DB can take" | partition/shard by key | §6.7 |
| "slow work shouldn't block the response" | queue + async workers | §6.6 |
| "global users, low latency" | CDN + regional replicas | §6.4 |
| "must not double-charge / duplicate" | idempotency keys | №31 §8.1 |
| "handle traffic spikes" | queue buffering, autoscaling, load shedding | §6.6, №31 §10.4 |
| "search over text" | search index (Elasticsearch), not `LIKE` | §5 |
| "who's online / live updates" | WebSockets / SSE + presence store | §8.4 |
| "limit abuse / fair usage" | rate limiter (token bucket) | §8.2 |
| "strict ordering of events" | single partition per key, sequence numbers | №31 §7.4 |
| "huge files (images, PDFs)" | object storage (S3) + CDN, never the DB | §5 |
| "analytics without hurting prod" | replica, or a separate OLAP store/warehouse | §6.5 |

---

# Part 1 — The method

A 45-minute round, time-boxed. Say the structure out loud at the start ("I'll scope requirements, do some estimation, sketch the API and data model, then the architecture, then go deep where it matters") — it signals competence before you've designed anything.

| Phase | Time | What you produce |
|---|---|---|
| **1. Requirements & scope** | ~5 min | functional list, non-functional targets, explicit out-of-scope |
| **2. Estimation** | ~5 min | QPS, storage, bandwidth — the numbers that drive decisions |
| **3. API** | ~5 min | the handful of endpoints, request/response shapes |
| **4. Data model** | ~5 min | entities, access patterns, storage choice |
| **5. High-level design** | ~10 min | the box diagram: client → edge → services → data |
| **6. Deep dive** | ~10 min | one or two areas, usually the interesting bottleneck |
| **7. Bottlenecks & trade-offs** | ~5 min | what breaks first, what you'd change, what you traded |

Two rules that matter more than the phases:

- **Drive, but check in.** "I'm going to assume reads dominate writes 100:1 — does that match what you have in mind?" You're steering, not lecturing.
- **Breadth first, then depth.** Get a complete working design on the board before optimising any part. A finished simple design beats a beautifully-optimised fragment.

---

# Part 2 — Requirements and scoping

The prompt is deliberately vague. **"Design Twitter" is not a specification; it's an invitation to ask questions.** Failing to scope is the most common way to fail the round.

## 2.1 Functional requirements — what it does

Pin down the 3–5 core operations, and explicitly cut the rest. For "design Twitter": post a tweet, follow a user, view a home timeline. Explicitly out of scope: DMs, search, ads, trending, notifications — say so out loud. **Narrowing the scope is a senior move, not a dodge**; you cannot design ten features in 45 minutes and attempting it guarantees a shallow answer.

## 2.2 Non-functional requirements — what it must be like

These drive the architecture far more than the features do:

| Dimension | Ask | Drives |
|---|---|---|
| **Scale** | how many users? DAU? | everything |
| **Read/write ratio** | reads-heavy or write-heavy? | caching, replication vs sharding |
| **Latency** | p99 target? | caching, CDN, sync vs async |
| **Availability** | can it be down? 99.9% or 99.99%? | replication, multi-AZ, failover |
| **Consistency** | is stale data acceptable? | №31 §4 — strong vs eventual |
| **Durability** | can we ever lose data? | replication factor, backups |
| **Growth** | 10× in a year? | headroom, partitioning strategy |

## 2.3 The scoping questions worth asking

Pick a handful, not all: *How many daily active users? What's the read/write ratio? How big is a typical item? Do reads need to be immediately consistent, or is a second of staleness fine? Global or single-region? What's the retention — forever, or 30 days? Is this greenfield or do we have existing infrastructure?*

> **The tell — scoping:** the interviewer usually has target numbers in mind and will happily give them. Ask for **DAU, read/write ratio, and the consistency requirement** as a minimum — those three determine most of your design. Then state your assumptions explicitly and move; don't spend fifteen minutes gathering requirements.

---

# Part 3 — Back-of-envelope estimation

The phase that most distinguishes candidates. Not because arithmetic is hard, but because **numbers turn opinions into decisions**: "we need sharding" is a claim; "3TB/year and 12k writes/sec, which is past a single Postgres primary, so we partition" is engineering.

## 3.1 The latency numbers everyone should know

Orders of magnitude, not exact values — this is the ladder from №00 §1.2, extended across the network:

| Operation | Time | Relative |
|---|---|---|
| L1 cache reference | ~1 ns | 1 |
| Main memory reference | ~100 ns | 100× |
| SSD random read | ~100 µs (0.1 ms) | 100,000× |
| Network round trip, same datacentre | ~0.5 ms | 500,000× |
| Disk seek (spinning) | ~10 ms | 10,000,000× |
| Round trip, cross-continent | ~150 ms | 150,000,000× |

Read these as: **memory is fast, disk is slow, the network is slower, and crossing an ocean is catastrophic.** That single hierarchy justifies caching, CDNs, regional replicas, and minimising round trips — and it's why the N+1 problem (№20 §2.8) is a *system design* issue, not just an ORM one.

## 3.2 The arithmetic shortcuts

- **Seconds in a day ≈ 86,400 ≈ 10⁵.** So **1M events/day ≈ 12/sec**, and **100M/day ≈ 1,200/sec**. Memorise this one conversion and most QPS maths becomes trivial.
- **Peak ≈ 2–3× average.** Design for peak.
- **Powers of two/ten:** 1 KB ≈ 10³ bytes, 1 MB ≈ 10⁶, 1 GB ≈ 10⁹, 1 TB ≈ 10¹².
- **A rough server** handles ~1,000–10,000 QPS for simple work; a Postgres primary handles thousands of writes/sec with decent hardware. Use these as sanity anchors.
- **Round everything.** 86,400 → 10⁵. 365 → 400. Nobody wants precision; they want the order of magnitude.

## 3.3 The three estimates you usually need

**Traffic (QPS):** `DAU × actions per user per day ÷ 86,400`, then ×2–3 for peak.
**Storage:** `items per day × size per item × retention`, ×replication factor.
**Bandwidth:** `QPS × payload size`.

Worked, for a service with 10M DAU where each user reads 20 items and writes 1:

```
Writes: 10M × 1  = 10M/day  ≈ 116/sec   → ~350/sec at peak
Reads:  10M × 20 = 200M/day ≈ 2,300/sec → ~7,000/sec at peak
Read:write ratio ≈ 20:1                 → READ-HEAVY: cache + read replicas
Storage: 10M writes/day × 1 KB = 10 GB/day ≈ 3.6 TB/year
         ×3 replication ≈ 11 TB/year     → plan partitioning, or cheap object storage
Bandwidth (read): 7,000/sec × 1 KB ≈ 7 MB/sec ≈ 56 Mbps → trivial
```

Notice how the numbers *made the decisions*: the 20:1 ratio says cache and replicate before you shard; 3.6 TB/year says a single node is fine for a while but plan for growth; the bandwidth is a non-issue so don't waste time on it.

> **The tell — estimation:** do it out loud, round aggressively, and then **say what the number implies**. The number itself scores nothing; "which means a single database handles this comfortably, so I won't shard" is the point. And if a number comes out absurd, say so — noticing that your estimate is implausible is a good signal.

---

# Part 4 — API and data model

## 4.1 The API

A handful of endpoints, showing you've thought about the contract (№45, planned). Keep it terse:

```
POST /v1/tweets            { text }                 → 201 { id, createdAt }
GET  /v1/timeline?cursor=  &limit=20                → 200 { items[], nextCursor }
POST /v1/users/{id}/follow                          → 204
```

Three details that earn credit: **cursor-based pagination** rather than offset (offset degrades on deep pages — №20 §3.8), an **idempotency key** on creates (№31 §8.1), and sensible status codes including **429** for rate limiting.

## 4.2 The data model

Sketch the entities and — more importantly — the **access patterns**, because those decide the storage:

- *How is this data read?* By id? By range? By relationship? Full-text?
- *What's the write pattern?* Append-only? Frequent updates? Bulk?
- *What must be transactional together?*

Then pick storage per data type — polyglot persistence is normal (№31, №20 §7):

| Data | Store | Why |
|---|---|---|
| core relational entities | **Postgres** | transactions, constraints, joins, flexible queries |
| sessions, hot lookups, counters | **Redis** | in-memory speed, TTLs |
| images, PDFs, video | **S3 / object storage** | cheap, durable; **never blobs in the DB** |
| full-text search | **Elasticsearch** | inverted index; `LIKE` doesn't scale |
| append-only events, fan-out | **Kafka** | ordered log, replay, multiple consumers |
| very high-volume simple KV | **DynamoDB/Cassandra** | horizontal write scale |

> **The tell — storage:** default to **one relational database** and justify every addition. "Postgres plus a cache" covers a remarkable proportion of real systems (№20 §7). Adding a second store adds consistency problems (№31 §4), operational burden, and a sync mechanism — so name the specific thing Postgres does badly before you reach for it.

---

# Part 5 — The building blocks

The standard vocabulary. Almost every design is an arrangement of these:

```
Client
  ↓  DNS (Route 53)
CDN (static assets, edge cache)
  ↓
Load balancer (ALB) ── TLS termination, health checks
  ↓
API gateway (optional) ── auth, rate limiting, routing
  ↓
Application servers (STATELESS, autoscaled)  ←→  Cache (Redis)
  ↓                    ↓
Database              Message queue (SQS/Kafka)
 primary + replicas      ↓
  ↓                    Workers (async jobs)
Object storage (S3)      ↓
                       Search index / analytics store
```

The pieces and their one-line jobs: **DNS** maps name → address; **CDN** serves static content from the edge; **load balancer** spreads traffic and removes unhealthy instances; **API gateway** handles cross-cutting edge concerns; **app servers** are stateless so any request can go anywhere (№31 §5.1); **cache** absorbs repeated reads; **database** is the source of truth; **queue** decouples slow work; **workers** do it; **object storage** holds big blobs; **search/analytics stores** serve query shapes the primary DB handles badly.

*Depth on the edge components — load balancers, CDNs, proxies, VPCs — is №51 §10.*

---

# Part 6 — The scaling toolkit, in order

When something won't scale, apply these **in roughly this order** — cheapest and simplest first. Reaching for sharding before caching is a classic error.

## 6.1 Make it stateless first
Non-negotiable prerequisite (№31 §5.1). Without it, nothing else works — you can't add instances if state lives in one.

## 6.2 Vertical scaling
A bigger machine. Unfashionable but correct as a first move: no complexity, immediate, and modern hardware is enormous. Limits: a ceiling, and a single point of failure.

## 6.3 Caching
**The biggest win per unit of effort** for read-heavy systems. Layers from the outside in: CDN → API cache → application/Redis → database cache. Patterns, invalidation and the stampede problem are №31 §9. If your read:write ratio is 20:1, caching is the first thing to reach for and often the only thing you need.

## 6.4 CDN
Static assets and cacheable responses served from the edge, near the user. Removes both latency (§3.1 — cross-continent RTT) and origin load.

## 6.5 Horizontal scaling + replication
More app instances behind the load balancer (stateless, so trivial). For the database: **read replicas** to scale reads (accepting replication lag — №31 §5.2), multi-AZ for availability. Also the answer to "analytics without hurting prod": point it at a replica.

## 6.6 Async processing
Move slow, retryable, or bursty work off the request path onto a **queue** with workers (№31 §7). This does three things at once: fast responses, absorbed traffic spikes (the queue is a buffer), and retryable failures. Anything that doesn't need to complete before the user gets their answer belongs here.

## 6.7 Partitioning / sharding
Splitting **writes** across nodes — the last resort, because it's a permanent tax on every query, join, transaction and migration (№31 §5.3). Choose the partition key by access pattern, and beware hotspots.

## 6.8 Specialised stores
Once one database is doing three jobs badly, split by workload: a search index for text, an OLAP store for analytics, a KV store for hot lookups — each with its own sync and consistency implications.

> **The tell — scaling:** the ladder is **stateless → vertical → cache → CDN → replicas → async → shard**. Say the order out loud in an interview; jumping straight to "we'll shard it" without caching first reads as pattern-matching rather than reasoning. And always ask "what's the actual bottleneck?" before choosing — scaling the wrong layer is wasted work.

---

# Part 7 — Worked example: a URL shortener

The canonical starter problem, because it exercises estimation, key generation, read-heavy caching, and storage choice in a small space.

## 7.1 Requirements

**Functional:** shorten a long URL → short code; redirect a short code → original; codes are unique; optional custom alias and expiry.
**Out of scope:** analytics dashboards, user accounts, editing.
**Non-functional:** very read-heavy; redirect latency must be low (~<100ms); high availability (a dead shortener breaks every link ever issued); URLs are effectively immutable, so **eventual consistency is fine for reads**.

## 7.2 Estimation

```
Assume 100M new URLs/month → 100M ÷ (30 × 86,400) ≈ 40 writes/sec
Read:write assumed 100:1     → ~4,000 reads/sec (peak ~10,000)
Storage: 100M/month × 500 bytes ≈ 50 GB/month ≈ 600 GB/year
         (5 years ≈ 3 TB — one database can hold this)
```

Conclusions the numbers hand you: **read-heavy → cache aggressively**; writes are trivial (40/sec — no sharding needed for a long time); storage is modest.

## 7.3 API

```
POST /v1/urls    { longUrl, customAlias?, expiresAt? }  → 201 { shortUrl }
GET  /{code}                                            → 302 → Location: longUrl
```

Use **301 vs 302** deliberately: 301 (permanent) lets browsers cache the redirect — great for load, terrible if you ever need to change or measure it. **302** keeps control and enables click counting. Worth saying out loud; it's a small trade-off that shows attention.

## 7.4 Key generation — the interesting part

The core design question: how do you produce a short, unique code?

| Approach | How | Trade |
|---|---|---|
| **Hash the URL** (MD5/SHA, truncated) | take first 7 chars of a hash, base62 | deterministic, but **collisions** need detect-and-retry |
| **Random** 7 chars | generate, check uniqueness | simple; needs a uniqueness check per write |
| **Counter + base62** | auto-increment id, encode to base62 | **no collisions**, shortest codes, but sequential = guessable/enumerable |
| **Counter with pre-allocated ranges** | each app node grabs a block of ids | no coordination per write, still collision-free |

**Base62** (`a-zA-Z0-9`) is the encoding: 62⁷ ≈ **3.5 trillion** codes in 7 characters — comfortably enough. My pick: **counter with per-node pre-allocated ranges**, base62-encoded — collision-free without a coordination round trip per write, which is the same "pre-allocate a block" trick as a Postgres sequence's `allocationSize` (№21 §2). If enumerability matters (someone crawling every short URL), hash or shuffle the counter before encoding.

## 7.5 Design

```
Client → CDN/DNS → Load balancer → App servers (stateless)
                                      ↓ read path
                                   Redis (code → longUrl, high TTL)
                                      ↓ miss
                                   Postgres (code PK, longUrl, expiresAt)
```

**Read path (the hot one):** look up the code in Redis; on hit, 302 immediately. On miss, read Postgres, populate the cache, redirect. With a decent hit rate the database sees almost nothing.
**Write path:** allocate an id from the node's range, base62-encode, insert, return.

The data model is one table with the code as primary key — every read is a point lookup by PK (№20 §4.3), which is the fastest thing a database does.

## 7.6 Bottlenecks and follow-ups

- **Cache hit rate** is the whole system's performance. URLs are immutable, so TTLs can be long and invalidation is a non-issue — an unusually friendly caching problem.
- **Hot keys:** a viral link concentrates traffic on one entry — the CDN/edge absorbs it.
- **Scale-out later:** if writes ever grow, partition by code prefix; reads are already independent point lookups, so this shards cleanly.
- **Expiry:** a background job deleting expired rows, or lazy deletion on read.

---

# Part 8 — More worked designs

Sketched more briefly — the key insight and the main trade-off for each.

## 8.1 Design a key-value store / cache (the "explain the internals" one)
The point is to show you understand №31: partitioning via **consistent hashing**, replication factor N with **quorum reads/writes (R + W > N)**, gossip for membership, vector clocks or last-write-wins for conflicts, and an LSM-tree or hash index for storage. This one is a distributed-systems knowledge check wearing a design costume.

## 8.2 Design a rate limiter

The algorithms — know these by name, it's a common follow-up:

| Algorithm | How | Trade |
|---|---|---|
| **Fixed window** | count per minute-bucket | simple; allows a 2× burst at boundaries |
| **Sliding window log** | timestamps of every request | exact; memory-heavy |
| **Sliding window counter** | weighted blend of current+previous window | good approximation, cheap — common choice |
| **Token bucket** | tokens refill at a rate; each request takes one | **allows controlled bursts** — usually the best fit |
| **Leaky bucket** | fixed-rate outflow queue | smooths output completely |

**Token bucket** is the usual answer because real traffic is bursty and it permits bursts without exceeding the average. The distributed challenge: with many app instances, the counter must be shared — **Redis with atomic INCR/Lua** is the standard, accepting a little inaccuracy at the edges, or approximate local counters synced periodically if you need less coordination. Return **429** with a `Retry-After` header (№51 §5.2).

## 8.3 Design a news feed

The classic **fan-out** trade-off, and the best single illustration of "it depends":

- **Fan-out on write (push):** when you post, write the item into every follower's precomputed feed. **Reads are instant** (just fetch your list). Writes are expensive — a celebrity with 50M followers triggers 50M writes.
- **Fan-out on read (pull):** compute the feed at read time by querying the people you follow. **Writes are cheap**, reads are expensive and slow.
- **The real answer: hybrid.** Push for normal users; pull for celebrities, merged at read time. This is the "celebrity problem," and naming it is the point of the question.

Then: cache feeds in Redis, paginate with cursors, and accept eventual consistency (nobody notices a post appearing 2 seconds late — №31 §4.3).

## 8.4 Design a chat system

Key points: **WebSockets** for bidirectional push (№51 §5.5) — HTTP polling doesn't scale; a **connection registry** mapping user → which server holds their socket (Redis), because with N servers the recipient is probably connected elsewhere; **message ordering** via per-conversation sequence numbers rather than timestamps (№31 §3.1); storage partitioned by conversation id; **delivery/read receipts** as separate lightweight events; offline users get messages queued and a push notification. The interesting bottleneck is that connections are *stateful* — which conflicts with §6.1, so you need the registry plus sticky routing.

---

# Part 9 — Designing your own system

Interviewers frequently pivot to *your* project — which is an easier round if you've thought about it in these terms. Practiq, framed as a system design:

**Requirements.** Functional: browse/filter questions by concept, attempt a question, human review workflow, ingest questions from PDFs. Non-functional: read-heavy (students browsing ≫ editors writing); modest scale (thousands of DAU, not millions); **strong consistency for the review workflow** (two reviewers double-approving is a genuine bug — №31 §4.3); eventual is fine for browse.

**Estimation.** At even 10k DAU × 50 question views = 500k reads/day ≈ 6/sec, peak ~20/sec. That is *nothing* — a single Postgres serves it comfortably. **Saying this is the strong answer**: the numbers justify the monolith rather than apologising for it.

**Architecture.** CloudFront (static React) → ALB → Fargate tasks running the Micronaut monolith (stateless) → RDS Postgres, private subnet. PDF extraction is the one genuinely async piece: upload → S3 → **SQS** → the Python extractor worker → results written back for review (№31 §7.1). Reference data (`Concept`, `SpecSection`) is the caching candidate; questions and attempts are not.

**Trade-offs you can defend.** Monolith-first with a documented service split — you get clean boundaries without the distributed tax at this scale (№31 §1.3). Postgres only, no second datastore, until a named need appears. Optimistic locking (`@Version`) rather than higher isolation for the review workflow, because it's precisely a lost-update problem (№21 §2.7). Async only where the work is genuinely slow and retryable.

**What you'd change at 100×.** Read replicas, then a Redis cache for concepts, then split the extractor (already separable — it's a different runtime and scaling profile), then partition attempts by date if volume demanded it. Being able to say *what would break first and in what order* is the mark of someone who's actually thought about their system.

> **The tell — your own system:** interviewers respect "we're at a scale where a monolith and one Postgres is correct, and here's the number that proves it" far more than an over-engineered design. Know your scale, know your first bottleneck, know your reversible vs irreversible decisions (№00 §8).

---

# Part 10 — The trade-off vocabulary

The recurring tensions. Being able to *name* them is what a design interview is really testing:

| Trade-off | The tension | How to decide |
|---|---|---|
| **Consistency vs availability** | during a partition, refuse or serve stale? | what does the user notice? money → C; feeds → A (№31 §4) |
| **Latency vs consistency** | sync replication costs ms | PACELC — the "else" case you live in |
| **SQL vs NoSQL** | flexible queries + transactions vs write scale | default SQL; justify the switch |
| **Normalised vs denormalised** | write simplicity vs read speed | denormalise only with measured read pressure |
| **Sync vs async** | simplicity vs resilience/throughput | does the caller need the answer now? |
| **Push vs pull (fan-out)** | write cost vs read cost | follower distribution — hybrid for celebrities |
| **Cache freshness vs load** | staleness window | how stale is tolerable? start with TTL |
| **Monolith vs services** | simplicity vs independent scaling/deploys | team count, not user count (№00 §8.2) |
| **Build vs buy** | control vs time | is this your differentiator? |
| **Cost vs performance** | more nodes, more money | what's the actual budget? |

The sentence pattern that scores: *"I'd choose X here because [context-specific reason]; the cost is Y, which is acceptable because [reason]. If [condition changed], I'd switch to Z."* That single structure — choice, cost, condition-for-changing — is the whole game.

---

# Part 11 — What's actually being assessed

## 11.1 The real rubric

Not "did you get the right architecture." Interviewers are looking for:

1. **Handling ambiguity** — do you scope, or start building blind?
2. **Structured thinking** — is there a visible process?
3. **Quantitative reasoning** — do numbers inform decisions?
4. **Breadth** — do you know the building blocks and what they're for?
5. **Depth on demand** — can you go deep when pushed?
6. **Trade-off reasoning** — do you justify choices and name their costs?
7. **Communication** — can they follow you? Do you invite input?

Note that (7) is weighted heavily, and it's the one strong engineers most often lose points on by going quiet.

## 11.2 How it goes wrong

| Failure | Fix |
|---|---|
| Diving into details before scoping | always do Part 2 first |
| Designing for a billion users when asked for a thousand | let the estimate decide (Part 3) |
| Going silent while thinking | narrate: "I'm weighing cache vs replica here…" |
| Presenting one answer with no alternatives | name the option you rejected and why |
| Buzzword soup (Kafka! Kubernetes! microservices!) | justify every component; if you can't, remove it |
| Never mentioning failure | say what happens when a node/dependency dies (№31 §10) |
| Refusing to commit | state assumptions and move; a decided design beats a hedged one |
| Ignoring the interviewer's hints | a nudge ("what if this instance dies?") is a *gift* — follow it |

## 11.3 The last five minutes

Reserve time to close well: name the **bottleneck that breaks first**, the **failure modes** and how the design survives them, **what you'd monitor** (№31 §11), and **what you'd do differently with more time or scale**. Ending with "here's what I'd revisit first" is a strong finish — it shows you know a design is never done.

> **The tell — the round:** you're being assessed as a colleague in a design discussion, not as an oracle. Scope, quantify, decide, justify, and keep talking. Nobody expects perfection in 45 minutes; they expect structured reasoning and honest trade-offs.

---

# How to expand this

- *Prerequisite:* №31 Distributed Systems — every component here (replication, queues, caching, consistency, resilience) is explained properly there.
- *Adjacent:* №42 Architecture (planned) for the in-application structure; №51 §10 for the edge components; №20 for the database layer; №54 Cloud & AWS (planned) for the managed-service equivalents of every box in Part 5.
- *Practice, not reading:* this is the one topic where a document is genuinely insufficient. Design 8–10 systems out loud, on paper, timed. The canonical set: URL shortener, rate limiter, news feed, chat, web crawler, notification service, file storage (Dropbox), video streaming, ticket booking (contention!), and a payment system (idempotency!). The variety matters — each stresses a different constraint.
- *Candidates for deeper treatment:* a full worked "design Twitter/Instagram" with diagrams; a back-of-envelope estimation drill sheet; Practiq designed end to end as a formal design doc.

*Written from stable knowledge — the method, the estimation techniques and the classic problems don't drift. Specific managed-service limits and capabilities do; check those before quoting numbers in a real design.*
