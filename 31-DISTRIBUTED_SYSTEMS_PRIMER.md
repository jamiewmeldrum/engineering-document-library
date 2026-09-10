# Distributed Systems — A Primer №31

*The genuinely hard foundation. Where №12 (Concurrency) is about coordinating threads that share memory, this is about coordinating **machines that share nothing** — connected only by a network that can lose, delay, duplicate, and reorder messages, and that fails partially and silently. Practiq lens: a Micronaut API, a Postgres, an extractor service, and a load balancer, all on AWS — which is a distributed system whether or not you call it one.*

The one sentence that generates almost everything else: **in a distributed system you cannot tell the difference between a machine that is slow, a machine that is dead, and a network that dropped your message.** You send a request; nothing comes back. Did it never arrive? Did it execute and the *reply* was lost? Is it executing right now, slowly? You cannot know — not with more logging, not with a better client, not ever. Every technique in this document is a way of building something reliable on top of that irreducible uncertainty.

The second idea, which follows: **partial failure is the defining feature.** In a single process, things work or the process dies — failure is total and obvious. In a distributed system, 3 of 5 nodes are fine, one is slow, one is unreachable, and the system must keep serving. There is no moment when everything is healthy.

Contents:

- **Part 1** — what makes it distributed, and why it's hard
- **Part 2** — the eight fallacies
- **Part 3** — time, ordering and causality
- **Part 4** — CAP, PACELC and the consistency models
- **Part 5** — replication and partitioning
- **Part 6** — consensus and coordination
- **Part 7** — communication: sync, async, queues, streams, delivery semantics
- **Part 8** — idempotency and distributed transactions
- **Part 9** — caching
- **Part 10** — resilience patterns
- **Part 11** — observing a distributed system
- **Part 12** — when to use what, and the questions worth being able to answer

## Symptom index

| Symptom | Cause | Go to |
|---|---|---|
| A duplicate record/charge appeared | a retry after a lost *response* | §8.1 |
| User writes data, immediately reads it back, it's missing | replication lag (read-your-writes) | §5.2 |
| One slow dependency takes the whole service down | no timeout / no bulkhead → thread exhaustion | §10.1, §10.4 |
| A brief outage turns into a total meltdown on recovery | retry storm / thundering herd | §10.2, §9.3 |
| Session lost when a request hits a different instance | state held in memory on one node | §5.1 |
| Two nodes both think they're the leader | split brain (no quorum/fencing) | §6.3 |
| Cache expiry causes a load spike on the database | cache stampede | §9.3 |
| Events processed out of order | no ordering guarantee across partitions | §7.4 |
| "It worked, then timed out, and now the state is unclear" | the fundamental uncertainty | §1.1, §8.1 |
| One shard/partition is far hotter than the others | poor partition key | §5.4 |
| Clock-based logic behaves bizarrely across machines | clock skew — never trust wall clocks | §3.1 |

---

# Part 1 — What makes it distributed, and why it's hard

## 1.1 The definition that matters

A system is distributed when its components run on **separate machines communicating over a network**. That's it — and by that definition your Practiq deployment already is one: the API and Postgres are separate machines, so every query crosses a network (№51 §4). You don't need microservices to inherit these problems; you need one network hop.

What the network boundary changes, versus a method call in one process:

| | In-process call | Network call |
|---|---|---|
| Latency | nanoseconds | milliseconds — **10⁶× slower** (№00 §1.2) |
| Failure | the process dies (total) | **partial** — it might work, fail, or hang |
| Outcome on timeout | n/a | **unknown** — this is the killer |
| Ordering | guaranteed | messages may reorder |
| Delivery | guaranteed | may be lost or duplicated |

The row that matters is **"unknown."** A local call either returns or throws. A remote call has a third outcome — *no answer* — and no way to distinguish "didn't happen" from "happened, reply lost." Every duplicate order, double charge, and orphaned record traces to something in that third state.

## 1.2 The impossibility results, briefly

Two pieces of theory worth knowing by name, because they explain why you can't just engineer the problem away:

- **The Two Generals Problem:** two parties over an unreliable channel can never be *certain* they agree, because the last message's delivery is always unconfirmed. Consequence: **guaranteed exactly-once delivery across a network is impossible.** Hence §8.
- **FLP impossibility:** in an asynchronous network where even one node may fail, no algorithm can guarantee consensus in bounded time. Consequence: real consensus systems (§6) use timeouts and accept "usually fast, occasionally slower" rather than a hard guarantee.

You don't need the proofs. You need the conclusion: **certainty is not available. Design for uncertainty instead of trying to eliminate it.**

## 1.3 Why we do it anyway

Distribution is a cost you pay for something. Be clear which:

- **Scale** — beyond what one machine can serve (horizontal scaling).
- **Availability** — surviving a machine, rack, or datacentre failure (multi-AZ, №51 §10).
- **Latency** — putting data near users (CDN, regional replicas).
- **Organisational** — independent teams shipping independently (the *real* microservices driver, №00 §8.2).

> **The tell:** if you're not getting one of those four, distribution is pure cost. This is why "start with a monolith" (№00 §8.2) is the right default — Practiq's monolith-first-with-a-documented-split gets you clean boundaries without paying the distributed tax before you need it.

---

# Part 2 — The eight fallacies

The classic list (Deutsch/Gosling, Sun, 1990s) of false assumptions engineers make. Each one names a real production failure:

| Fallacy | Reality | Design response |
|---|---|---|
| The network is reliable | packets drop; links fail | retries + idempotency (§8) |
| Latency is zero | ms per hop, worse cross-region | batch, cache, minimise round trips |
| Bandwidth is infinite | payloads cost | paginate, compress, project (№20 §3.8) |
| The network is secure | it isn't | TLS, authn/authz everywhere (№51 §7) |
| Topology doesn't change | instances come and go constantly | service discovery, no hardcoded IPs |
| There is one administrator | many teams, many configs | explicit contracts, versioning |
| Transport cost is zero | serialisation + bandwidth + money | mind payload size, egress charges |
| The network is homogeneous | mixed everything | standard protocols, no assumptions |

The unifying lesson: **every remote call is a request that might not come back, over a link you don't control.** Code that treats a network call like a local one is code that works in dev and fails in production.

---

# Part 3 — Time, ordering and causality

A subtle area that produces baffling bugs, and the direct cousin of №12 §3 (happens-before) — one layer out.

## 3.1 You cannot trust clocks

Every machine has its own clock, and they disagree. NTP corrects drift but leaves **skew** (milliseconds to seconds) and can jump time *backwards*. Consequences:

- **Never order events across machines by wall-clock timestamp.** "Latest timestamp wins" silently loses writes when clocks disagree.
- **Never measure elapsed time with wall-clock time** — a clock adjustment mid-measurement gives you negative durations. Use a monotonic clock (`System.nanoTime()`, not `Instant.now()`) for durations; use `Instant` for *recording when something happened* (№10 §9).
- **Never use timestamps for lock expiry or distributed correctness.** That's what fencing tokens are for (§6.3).

## 3.2 Logical time

Since physical time is unreliable, distributed systems order events **causally** instead:

- **Lamport clocks:** a counter per node, incremented on each event and piggybacked on messages; the receiver takes `max(local, received) + 1`. Gives a total order consistent with causality — if A caused B, A's counter is lower. (It can't tell you whether two events were truly concurrent.)
- **Vector clocks:** a counter *per node*, so you can detect genuine **concurrency** — two events where neither caused the other. That's how systems detect conflicting concurrent writes.
- **Happens-before** is the same relation as in the JMM (№12 §3): if A happens-before B, A's effects are visible to B. Same idea, different scale — threads sharing memory versus nodes sharing messages.

> **The tell — time:** treat wall-clock timestamps as *human-readable metadata*, never as a correctness mechanism. If your logic depends on "which came first" across machines, you need a version number, a sequence, or a consensus system — not a clock.

---

# Part 4 — CAP, PACELC and consistency models

The most-cited and most-mangled topic in the field. Getting it precisely right is what separates a useful conversation about consistency from a circular one.

## 4.1 CAP, stated properly

**In the presence of a network Partition, a system must choose between Consistency and Availability.**

- **C** — every read sees the most recent write (linearizability).
- **A** — every request to a non-failed node gets a (non-error) response.
- **P** — the system keeps working despite messages being lost between nodes.

The common misstatement is "pick two of three." **You don't get to choose P.** Networks partition; that's physics, not architecture. So CAP is really a binary choice about *behaviour during a partition*:

- **CP** — refuse to serve rather than serve possibly-stale data (a Postgres primary that won't accept writes when isolated; ZooKeeper/etcd).
- **AP** — keep serving, accept that answers may be stale and reconcile later (DynamoDB in some modes, Cassandra, DNS).

And the crucial caveat: **CAP only describes behaviour during a partition.** Most of the time there is no partition, and the theorem says nothing at all about how the system behaves then — which is why the extension matters.

## 4.2 PACELC — the more useful version

**If Partition, choose Availability or Consistency; Else (normal operation), choose Latency or Consistency.**

That "else" is where you actually live. Even with a healthy network, strong consistency costs latency — a synchronous cross-node acknowledgement takes real milliseconds. Every read-replica setup is a PACELC decision: you traded consistency for latency, and you did it on a *good* day, not during a partition.

## 4.3 The consistency models

A spectrum, not a binary. Strongest to weakest:

| Model | Guarantees | Cost |
|---|---|---|
| **Linearizable (strong)** | reads always see the latest committed write; the system behaves as if there's one copy | highest latency, lowest availability |
| **Sequential** | all nodes see operations in the same order (not necessarily real-time) | high |
| **Causal** | causally-related operations are seen in order by everyone; concurrent ones may differ | moderate — often the sweet spot |
| **Read-your-writes** | *you* always see your own writes (others may lag) | cheap, and what users actually notice |
| **Monotonic reads** | you never see time go backwards | cheap |
| **Eventual** | if writes stop, replicas converge... eventually | cheapest, most available |

The practical insight: **"eventual consistency" is often fine, but "read-your-writes" is usually non-negotiable** — a user who saves a form and immediately sees stale data will file a bug, even though the system is behaving as designed. That specific guarantee is what §5.2 is about.

> **The tell — consistency:** ask *"what does a user actually notice if this is 500ms stale?"* Ordering someone's own edits: they'll notice — needs read-your-writes or a strong read. A dashboard count, a search index, a recommendation: they won't — eventual is fine and much cheaper. Don't buy linearizability for data nobody is watching that closely.

---

# Part 5 — Replication and partitioning

The two ways to use more than one machine. **Replication** = the same data on several nodes (for availability and read scale). **Partitioning/sharding** = different data on different nodes (for write and storage scale). Real systems do both.

## 5.1 Stateless services first

Before any of this: **make your application nodes stateless.** If a request can be served by any instance, you can add, remove, and replace instances freely, and a load balancer can route anywhere (№51 §10.2). State goes in a database, a cache, or a token — never in an instance's memory.

This is the single most practical distributed-systems rule for a service like Practiq, and it's why an in-memory session breaks the moment you run two Fargate tasks, and why a mutable field on a singleton bean is a bug (№12 §7.2). Same principle, two scales.

## 5.2 Replication topologies

**Single-leader (primary-replica)** — all writes go to one leader, which streams changes to followers that serve reads. The default (RDS, Postgres streaming replication). Simple, no write conflicts. Two consequences:

- **Replication lag:** a follower is milliseconds-to-seconds behind. Read your own write from a follower and it may not be there — the classic **read-your-writes** violation. Fixes: read from the leader after your own writes, pin a user to the leader for a short window, or track a version and wait for it.
- **Failover:** if the leader dies, a follower is promoted — which needs consensus to avoid two leaders (§6.3), and risks losing writes not yet replicated.

**Multi-leader** — several nodes accept writes (multi-region). Better write availability, but you *will* get **write conflicts** and need a resolution strategy (last-write-wins — which loses data and depends on clocks, §3.1; or CRDTs; or application-level merge).

**Leaderless (quorum)** — any node accepts writes; clients write to several and read from several. With **N** replicas, **W** write acknowledgements and **R** read responses, you get overlap (and therefore fresh reads) when **R + W > N**. Dynamo/Cassandra style. Tunable consistency per-operation.

## 5.3 Partitioning

Splitting data so each node holds a subset:

- **By key range** — ordered, so range scans are efficient, but prone to **hotspots** (all today's data on one node).
- **By hash of the key** — even distribution, but range queries must hit every partition.
- **Consistent hashing** — the technique that makes adding/removing nodes cheap: keys map to points on a ring, so a new node steals only its neighbours' keys instead of forcing a global reshuffle.

The hard part is choosing the **partition key**. A bad one creates a hot partition that becomes the bottleneck while other nodes idle — and repartitioning a live system is genuinely painful, which is why this is one of those decisions worth getting right early (№00 §8).

> **The tell — replication/partitioning:** replicate for availability and read scale; partition for write and storage scale. Reach for partitioning *last* — a single well-indexed Postgres with read replicas takes you far further than most teams assume (№20 §7), and sharding imposes a permanent tax on every query and migration.

---

# Part 6 — Consensus and coordination

## 6.1 The problem

Sometimes nodes must **agree**: who is the leader, is this transaction committed, which config is current. That's **consensus** — and per §1.2 it's provably hard in an asynchronous network.

## 6.2 Quorums and the algorithms

The core mechanism is a **quorum**: a majority (more than half). Because any two majorities of the same set must overlap, a majority decision can't contradict a previous one. This is why consensus systems use **odd numbers** of nodes (3, 5) — 5 nodes tolerate 2 failures; adding a 6th tolerates the same 2 while making the quorum bigger.

**Raft** and **Paxos** are the algorithms; Raft is designed to be understandable and is what you'll meet. The shape: elect a leader, the leader appends entries to a replicated log, an entry commits once a majority has it. You will essentially never implement one — but you *use* them constantly, inside etcd, ZooKeeper, Consul, Kafka's controller, and every managed database's failover mechanism.

## 6.3 Split brain and fencing

**Split brain** is the failure where a partition leaves two nodes each believing they're the leader, both accepting writes — the corruption scenario consensus exists to prevent. Quorum is the primary defence: a minority partition can't reach majority, so it can't act.

The subtler problem: a leader can be *paused* (GC pause, VM suspend), lose its lease, and wake up still thinking it's the leader. Defence: **fencing tokens** — each leadership grant carries a monotonically increasing number, and the storage layer rejects writes bearing a stale token. Note this is a *version number*, not a timestamp (§3.1). It's the same idea as optimistic locking's `@Version` (№21 §2.7) — reject the write whose view of the world is out of date.

> **The tell — consensus:** never build it yourself. Use etcd/ZooKeeper/Consul, or the coordination your managed service already provides. "Distributed lock" is a phrase that should make you check whether you actually need one — most cases are better solved by idempotency (§8) or a database constraint/optimistic lock (№20 §2.7), both far simpler and harder to get wrong.

---

# Part 7 — Communication

## 7.1 Synchronous vs asynchronous

| | **Synchronous** (HTTP/gRPC) | **Asynchronous** (queue/event) |
|---|---|---|
| Caller | waits for a response | fires and continues |
| Coupling | temporal — callee must be up *now* | decoupled — callee can be down |
| Failure | propagates to the caller immediately | absorbed by the broker; retried later |
| Simplicity | easy to reason about and debug | harder — flow is implicit |
| Latency | end-to-end, bounded by the slowest hop | request returns fast; work completes later |

The rule of thumb: **synchronous when the caller genuinely needs the answer to proceed; asynchronous when it doesn't.** Practiq's PDF extraction is the textbook async case — the upload should return immediately, and extraction (slow, retryable, occasionally failing) should happen on a queue. Making the user's HTTP request wait for pdfplumber is the wrong shape.

## 7.2 Queues vs streams

| | **Message queue** (SQS, RabbitMQ) | **Event stream** (Kafka, Kinesis) |
|---|---|---|
| Model | a work item, consumed **once** | an append-only **log**, replayable |
| After consumption | message deleted | retained; other consumers read independently |
| Consumers | competing workers | independent consumer groups, each with its own offset |
| Ordering | limited (FIFO queues) | ordered **within a partition** |
| Use for | task distribution, buffering work | event sourcing, fan-out, replay, analytics |

The distinction that matters: a queue **distributes work**; a stream **records history**. If several unrelated things need to react to "a question was approved," that's a stream (or pub/sub). If one worker needs to extract one PDF, that's a queue.

## 7.3 Backpressure

If producers outpace consumers, something must give: the queue grows unboundedly until it falls over, or you push back. **Backpressure** is signalling upstream to slow down — bounded queues, rate limiting, rejecting with 429 (№51 §5.2). A system without backpressure doesn't degrade gracefully; it collapses.

## 7.4 Delivery semantics — and why exactly-once is a myth

| Semantic | Means | Reality |
|---|---|---|
| **At-most-once** | fire and forget; may be lost | fine for metrics, telemetry |
| **At-least-once** | retried until acknowledged; **may duplicate** | **the practical default** |
| **Exactly-once** | delivered precisely once | **not achievable across a network** (§1.2) |

Systems advertising "exactly-once" deliver at-least-once *plus* deduplication or transactional offsets — which is exactly-once **processing**, achieved by making the effect idempotent. Which is the whole of the next Part, and the single most important practical technique in this document.

---

# Part 8 — Idempotency and distributed transactions

## 8.1 Idempotency — the central practical skill

An operation is **idempotent** if performing it twice has the same effect as performing it once. Because §1.1 means you *must* retry, and §7.4 means retries *will* duplicate, idempotency is what makes retrying safe. It is the load-bearing technique of distributed systems.

Naturally idempotent: `PUT` (set to this value), `DELETE`, "set status = APPROVED". Not idempotent: `POST` (create another), "increment by 1", "send an email".

The standard mechanism is an **idempotency key** — the client generates a unique id per logical operation and sends it with every retry; the server records processed keys and returns the original result instead of re-executing:

```java
@Transactional
public Result submit(String idempotencyKey, Payload p) {
    // unique constraint on idempotency_key does the real work
    Optional<Result> existing = repo.findByKey(idempotencyKey);
    if (existing.isPresent()) return existing.get();     // replay: return the first outcome
    Result r = doTheWork(p);
    repo.save(new Record(idempotencyKey, r));            // constraint violation ⇒ concurrent retry
    return r;
}
```

The database's unique constraint is doing the heavy lifting — a lesson that recurs: **push correctness into the datastore where you can** (№20 §4.1). Other routes to idempotency: natural keys with upsert (`INSERT ... ON CONFLICT`), version checks (№21 §2.7), and designing operations as *set-to-a-value* rather than *adjust-by-an-amount*.

## 8.2 Distributed transactions

A single database gives you ACID across multiple rows (№20 §1.7). Across services or databases, you don't get that.

**Two-phase commit (2PC)** — a coordinator asks all participants to prepare, then to commit. It works, but it's **blocking**: if the coordinator dies after "prepare," participants hold locks indefinitely. Slow, fragile, and largely avoided in modern designs.

**Sagas** — the practical alternative. Break the distributed transaction into a sequence of *local* transactions, each with a **compensating action** that semantically undoes it. If step 3 fails, run the compensations for 2 and 1. You give up atomicity and isolation; you get availability and no distributed locks. Note compensation is not rollback — you can't unsend an email, so you send an apology.

**The outbox pattern** — solves the "update the database *and* publish an event atomically" problem. You can't transactionally write to Postgres and to Kafka; instead, write the event to an `outbox` table **in the same local transaction** as the state change, and a separate process reads that table and publishes. Atomic, because both writes are in one database transaction.

> **The tell — transactions:** keep transactions **local** wherever possible. Design service boundaries so a business operation is one local transaction (which is a strong argument for a monolith, №00 §8.2). When you truly must span services: saga + outbox + idempotent consumers. Never 2PC.

---

# Part 9 — Caching

The highest-leverage performance technique, and the one with the sharpest edges. Caching is trading **freshness for speed** (a PACELC choice, §4.2).

## 9.1 The patterns

| Pattern | How | Trade |
|---|---|---|
| **Cache-aside** (lazy) | app checks cache; on miss, loads from DB and populates | simple, the default; first request always slow |
| **Read-through** | cache itself loads on miss | cleaner app code; needs support |
| **Write-through** | write to cache and DB together | cache always fresh; slower writes |
| **Write-behind** | write to cache, flush to DB async | fast writes; **risk of data loss** |

## 9.2 Invalidation

"There are only two hard things in computer science: cache invalidation and naming things." The options, in order of preference:

- **TTL (expiry)** — simplest and most robust: accept staleness for a bounded window. Usually the right answer.
- **Explicit invalidation on write** — fresher, but you must find *every* write path, and a missed one means permanent staleness.
- **Versioned keys** — include a version in the cache key so a bump orphans the old entry (no deletion needed).

## 9.3 The failure modes

- **Stampede / thundering herd:** a popular key expires and a thousand concurrent requests all miss and hit the database simultaneously — the cache was the only thing keeping the DB alive. Fixes: **jitter the TTLs** (so keys don't expire in lockstep), single-flight locking (one request recomputes, others wait), or refresh-ahead.
- **Cold cache after restart/deploy:** the database sees full load until the cache warms. Consider warming, or staggered restarts.
- **Cache penetration:** repeated requests for keys that don't exist bypass the cache every time. Cache the negative result too.

## 9.4 Where caches live

Client → CDN (№51 §10.3) → API gateway → application in-memory → distributed cache (Redis/ElastiCache) → database's own cache. **The fastest request is one that never reaches your service**, so cache as far out as correctness allows. Note the in-memory layer becomes inconsistent across instances (§5.1) — fine for immutable reference data, wrong for anything users mutate.

> **The tell — caching:** cache **read-heavy, change-rarely, tolerate-staleness** data. For Practiq that's `Concept`/`SpecSection`/board metadata — the same conclusion the L2-cache discussion reached from the ORM side (№20 §5.3, №21 §7). Start with a TTL and no explicit invalidation; add complexity only when the staleness window demonstrably hurts. And always jitter your TTLs.

---

# Part 10 — Resilience patterns

Failure is the normal case (§1.1), so these aren't advanced techniques — they're the baseline for any code that makes a network call.

## 10.1 Timeouts

**Every remote call needs a timeout.** A call without one waits forever, holding a thread and a connection; enough of those and the service is dead — killed not by an error but by *waiting*. This is the most common cause of cascading failure.

Set both a **connect** and a **read** timeout, budget them (if your endpoint must answer in 2s, an internal call can't be allowed 5s), and remember the caller's timeout should exceed the callee's or you'll retry work that's still running.

## 10.2 Retries — with backoff and jitter

Retries fix transient failures and *cause* outages when done naively. The rules:

- **Only retry idempotent operations** (§8.1), and only retryable errors — a 500 or timeout, never a 400.
- **Exponential backoff:** wait 100ms, 200ms, 400ms… so you don't hammer a struggling service.
- **Jitter:** randomise the delay. Without it, every client retries in perfect synchrony and you get a **retry storm** — coordinated waves that keep the recovering service down.
- **Cap the attempts.** Retrying forever converts a brief failure into an outage.

```java
long delay = 100;
for (int attempt = 1; attempt <= 4; attempt++) {
    try { return call(); }
    catch (RetryableException e) {
        if (attempt == 4) throw e;
        Thread.sleep(delay + ThreadLocalRandom.current().nextLong(delay));  // backoff + jitter
        delay *= 2;
    }
}
```

## 10.3 Circuit breaker

If a dependency is down, continuing to call it wastes resources and delays every request. A **circuit breaker** wraps the call and tracks failures:

- **Closed** — calls pass through (normal).
- **Open** — after a failure threshold, calls **fail immediately** without attempting the network. Fast failure instead of slow timeouts.
- **Half-open** — after a cooldown, let one trial call through; success closes the circuit, failure re-opens it.

The point is to **fail fast and let the dependency recover** instead of drowning it in traffic it can't serve.

## 10.4 Bulkheads and load shedding

- **Bulkhead:** isolate resources per dependency (separate connection/thread pools), so one slow dependency exhausts only its own pool rather than every thread in the service. Named after ship compartments — a breach floods one section, not the hull.
- **Load shedding:** under overload, reject excess requests quickly (429) rather than accepting everything and serving nothing acceptably. Rejecting 10% cleanly beats failing 100% slowly.
- **Graceful degradation:** serve a reduced experience rather than an error — stale cache, a default, hiding a non-essential panel. A slightly-wrong page beats a 500.

## 10.5 Cascading failure

The pattern behind most large outages: service C slows → B's threads all block waiting on C (no timeout, §10.1) → B stops responding → A's threads block on B → the whole system is down, all because one dependency got slow. Note **slow is worse than down**: a fast failure is contained; a slow one propagates by consuming resources everywhere upstream. Timeouts, circuit breakers, and bulkheads exist to break this chain.

> **The tell — resilience:** every remote call gets a **timeout**; every retry gets **backoff + jitter** and idempotency; every dependency gets a **circuit breaker** and its own pool. These aren't optional extras for large systems — a two-service app with an external API call needs them.

---

# Part 11 — Observing a distributed system

You cannot debug what you cannot see, and in a distributed system no single machine holds the story of a request.

- **Correlation/trace IDs:** generate an ID at the edge, propagate it through every call and log line. Without it, reconstructing a request across services is guesswork. This is the single highest-value thing to implement first.
- **Distributed tracing** (OpenTelemetry): spans stitched into one timeline showing where a request spent its time. It's how you find "the 2s is actually 40 sequential DB calls" — an N+1 (№20 §2.8) rendered visible.
- **Structured logs, aggregated centrally** — per-instance log files are useless when instances are ephemeral.
- **Metrics and the four golden signals** — latency, traffic, errors, saturation. Alert on **symptoms users feel**, not causes.
- **Health checks** — so the load balancer routes away from sick instances (№50 §7.1, №51 §10.2). Distinguish *liveness* (am I alive?) from *readiness* (can I serve traffic yet?).

*This gets its own treatment in №57 (Observability & Production Operations).*

---

# Part 12 — When to use what

## 12.1 Decision frameworks

**A. Distribute at all?** Tell → yes: you need scale beyond one machine, availability across failures, latency near users, or independent team deploys. Tell → no: none of those apply. Default: **a well-structured monolith; distribute when you can name which of the four you're buying.**

**B. Sync or async?** Tell → sync: the caller needs the answer to proceed. Tell → async: it doesn't (extraction, email, indexing, analytics). Default: **sync for reads and user-facing writes; async for slow, retryable, or fan-out work.**

**C. Queue or stream?** Tell → queue: distribute work to competing consumers, each item once (SQS). Tell → stream: multiple independent consumers, replay, or history matters (Kafka). Default: **queue unless you need replay or fan-out.**

**D. Consistency level?** Tell → strong: money, inventory, anything where stale means wrong. Tell → read-your-writes: users editing their own data (the usual minimum). Tell → eventual: dashboards, search indexes, recommendations. Default: **strong within one database; eventual across services, with read-your-writes where users would notice.**

**E. Scale reads?** Tell → cache first (cheapest, biggest win). Tell → read replicas next (accepting lag). Tell → partition last (permanent complexity). Default: **cache → replicate → partition, in that order.**

**F. Handling a failed call?** Tell → retry: idempotent and transient. Tell → circuit-break: the dependency is failing repeatedly. Tell → degrade: you can serve something useful without it. Tell → fail fast: none of the above. Default: **timeout always; retry with backoff+jitter if idempotent; circuit breaker around every external dependency.**

## 12.2 The questions worth being able to answer

These are the load-bearing questions in this field, and they are the ones to ask of any design that crosses a network (№43). Where each is answered:

- **"Explain CAP."** → §4.1. The precise version: **P isn't optional**, the choice is about behaviour *during* a partition, and PACELC is the more useful formulation.
- **"Strong vs eventual consistency?"** → §4.3, with the practical framing: what does the user actually notice?
- **"How do you handle a failed request?"** → §10 — timeout, backoff+jitter, idempotency, circuit breaker. Name the retry storm.
- **"How do you avoid duplicate processing?"** → §8.1 — idempotency keys, and "exactly-once delivery doesn't exist; exactly-once *processing* does, via idempotency."
- **"How do you scale reads/writes?"** → §5, in the order of E above.
- **"How do services communicate?"** → §7, with the sync/async trade.
- **"Distributed transactions?"** → §8.2 — sagas and the outbox; explain why 2PC is avoided.

The framing underneath all of them: **name the trade-off, then make a call.** "I'd use eventual consistency here because the user never sees this data directly, and the availability win is worth it — but for the approval workflow I'd want strong consistency in a single transaction, because two reviewers double-approving is a real bug." That's engineering; reciting CAP is trivia.

---

# How to expand this

- *The single-machine cousin:* №12 Concurrency — same coordination problems where the medium is shared memory rather than a network; happens-before, optimistic concurrency (CAS) and fencing tokens are the same idea at different scales.
- *The network underneath:* №51 Networking — TCP, timeouts, load balancers, what a partition physically is.
- *The transactional foundation:* №20 §1.7 (transactions, isolation, MVCC) and №21 §2.7 (optimistic locking).
- *Next in the roadmap:* **№43 System Design** applies all of this to canonical design problems — this primer is its prerequisite.
- *Candidates for deeper treatment:* event-driven architecture and event sourcing/CQRS end to end; Kafka specifically; a worked saga + outbox implementation for Practiq's extraction pipeline.

*Foundations primer written from stable knowledge — CAP, consensus, the fallacies and the resilience patterns don't drift. Specific tool behaviours (a given broker's guarantees, a managed service's consistency model) do; check those against current docs before relying on them.*
