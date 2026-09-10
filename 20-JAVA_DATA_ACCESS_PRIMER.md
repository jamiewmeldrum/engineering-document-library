# Java Data Access — A Primer

*A working engineer's mental model for talking to a relational database from Java, with a Practiq/Micronaut lens. Parts 1–6 are deliberately pinned to **your** stack — Jakarta Persistence 3.x (`jakarta.*`), Hibernate 6, Micronaut Data 4, PostgreSQL — because a concrete stack is easier to reason about than an abstract one. **Part 7 pulls the camera all the way back**: what else exists at each layer, what Postgres even *is*, and why these particular defaults are reasonable — so the pinning reads as a choice, not an assumption.*

This is a TL;DR of a genuinely large subject, so it's long — but it's dense, not padded. The aim: after reading it you should be able to point at any line of your data layer and say **which layer it belongs to, who implements it, what SQL it becomes, and why you'd choose it over the alternative.** That last clause — the "why you'd choose it" — is the thing you said you keep doing blind, so it runs through the whole document and is collected into a decision reference in Part 6.

A recurring theme worth stating up front: **almost everything the ORM does is a SQL idea wearing a Java costume.** A transaction is `BEGIN … COMMIT`. A managed entity is a row you've read inside an open transaction. Dirty checking is a deferred `UPDATE`. Optimistic locking is a `WHERE version = ?`. Lazy loading is a second `SELECT` you didn't write. When Hibernate surprises you, the move is always the same: *ask what SQL it just ran, and when.* Part 3.1 shows you how to make that SQL visible — do that early.

Contents:

- **Part 1** — end to end: from your code to bytes on a socket, and the fork (map rows vs. map objects)
- **Part 2** — JPA, Hibernate, and the frameworks: the persistence context, transactions, identity, locking, fetching — each tied to its SQL
- **Part 3** — the query APIs and the SQL each becomes, plus pagination/projection/bulk concerns
- **Part 4** — data modelling and physical optimisation: how tables connect, indexes, the clustered-index myth, reading the query plan
- **Part 5** — deeper JPA machinery: the Metamodel API, entity graphs, caching, subqueries, and the rest of the map
- **Part 6** — *When to use what*: consolidated decision frameworks
- **Part 7** — the wider landscape: what a database even is, what else there is besides Postgres and Hibernate, and where your choices sit
- **Part 8** — a worked trace: one real request followed through every layer, log line by log line
- **How to expand this doc**

## Symptom index — open here when something's wrong

| Symptom | Go to |
|---|---|
| App hangs / times out under load | 1.6 (pool exhaustion — a transaction held open too long) |
| `LazyInitializationException` | 2.4 (touched a lazy association after the transaction closed), fixes in 2.8 / 5.2 |
| An `UPDATE` fired that I never asked for | 2.5 (dirty checking — it's correct behaviour) |
| I set a field and *nothing* was saved | 2.3 / 2.5 (the entity was detached — no snapshot, no dirty check) |
| SQL statements run in a different order than my code | 2.5 (write-behind; Hibernate reorders at flush) |
| A constraint violation that "can't happen" from my code order | 2.5 (flush ordering) — check the actual statement order in the log |
| Endpoint slow; log shows dozens of near-identical `SELECT`s | 2.8 (N+1) — fixes: join fetch, entity graph (5.2), batch size, projection (3.8) |
| Bulk/seed inserts are slow | 2.6 (IDENTITY blocks batching → use SEQUENCE) + 5.5 (flush/clear in loops) |
| Two users edited the same row; one edit vanished | 2.7 (lost update → `@Version` optimistic locking) |
| Duplicate parent rows after adding a join | 5.4 (one-to-many join multiplies rows → `DISTINCT`, or `EXISTS`) |
| A `NOT IN` query silently returns nothing | 5.4 (the NULL trap → use `NOT EXISTS`) |
| Query fast in dev, slow in production | 4.5 (seed-size data hides index problems — `EXPLAIN ANALYZE` at realistic volume) |
| Filter query ignores my index | 4.3 (composite column order / leftmost-prefix; or stale stats, 4.5) |
| Entities are stale after a bulk JPQL update | 3.8 (bulk ops bypass the persistence context — clear it) |
| Tests pass on H2, fail on Postgres (or vice versa) | 7.3 (dialect drift — the reason for Testcontainers) |
| Repository method won't compile | 2.12 (that's Micronaut doing its job — fix the method name) |

---

# Part 1 — The end-to-end picture

## 1.1 The one-sentence version

Your code hands a query (in some form) to a **JPA provider** (Hibernate), which turns it into **SQL**, wraps it in a **JDBC** `PreparedStatement`, runs it inside a **transaction** on a **connection** borrowed from a pool, and lets the **JDBC driver** speak Postgres's wire protocol over a socket. Rows come back and are **hydrated** into your objects.

Everything else is detail about *which form* you hand the query in, *when* the SQL actually runs, and *how much* the layers do for you.

## 1.2 The layers

Top = closest to you, bottom = closest to the metal:

```
┌──────────────────────────────────────────────────────────────┐
│  YOUR CODE                                                     │
│  entities (@Entity), repository interfaces, specifications     │
├──────────────────────────────────────────────────────────────┤
│  DATA-ACCESS FRAMEWORK           ← Micronaut Data / Spring Data│
│  repositories, derived finders, pagination, specifications     │
│  (boilerplate reduction; delegates down — does NOT do ORM)     │
├──────────────────────────────────────────────────────────────┤
│  JPA — the SPECIFICATION         ← jakarta.persistence.*       │
│  a contract: annotations + EntityManager + JPQL + Criteria     │
│  (interfaces and rules only — ships NO working code)           │
├──────────────────────────────────────────────────────────────┤
│  JPA PROVIDER — the IMPLEMENTATION  ← Hibernate                │
│  entity lifecycle, dirty checking, SQL generation, dialects    │
├──────────────────────────────────────────────────────────────┤
│  CONNECTION POOL                 ← HikariCP                    │
│  reuses a small set of live connections (each = a SQL session) │
├──────────────────────────────────────────────────────────────┤
│  JDBC — the SPECIFICATION        ← java.sql.*                  │
│  Connection / PreparedStatement / ResultSet / transactions     │
├──────────────────────────────────────────────────────────────┤
│  JDBC DRIVER — the IMPLEMENTATION   ← pgjdbc                   │
│  translates JDBC calls into Postgres's binary wire protocol    │
├──────────────────────────────────────────────────────────────┤
│  POSTGRESQL                                                    │
└──────────────────────────────────────────────────────────────┘
```

Internalise two things:

- **"Specification" appears twice (JDBC, JPA) and "implementation" appears twice (pgjdbc, Hibernate).** That pairing is the whole trick of the Java data ecosystem: a stable interface you code against, a swappable engine behind it. You import `jakarta.persistence.*`; at runtime *Hibernate* is what actually runs. You import `java.sql.*`; at runtime *pgjdbc* is what actually runs.
- The **framework is not a fourth engine.** Micronaut Data sits *on top of* JPA and delegates *down* into it. `questionRepository.findAll(spec)` ends up driving Hibernate → JDBC → pgjdbc → Postgres.

## 1.3 JDBC — the foundation

JDBC (`java.sql`) is the lowest-level standard for talking to *any* relational database from Java, and it's part of the JDK. Write it by hand once so you know what everyone above is hiding:

```java
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(
         "select id, concept_id, status from question where concept_id = ?")) {
    ps.setLong(1, conceptId);
    try (ResultSet rs = ps.executeQuery()) {
        while (rs.next()) {
            long id = rs.getLong("id");
            String status = rs.getString("status");
            // ...you map each column onto a field, by hand
        }
    }
}
```

JDBC gives you: a connection, **parameterised** statements (`?` placeholders — the mechanism that prevents SQL injection, because values never become part of the SQL text), and a forward-only cursor over rows. It gives you **nothing** about mapping rows to objects, relationships, or change tracking. Every abstraction above JDBC exists to remove some of that manual `rs.getX()` drudgery. The differences between them are *how much* they remove and *what they charge for it.*

## 1.4 The driver

`java.sql` is only interfaces. **pgjdbc** (`org.postgresql:postgresql`) is the concrete implementation for Postgres — it opens the socket, speaks the wire protocol, and presents results back through the JDBC interfaces. Swap Postgres for MySQL and you swap the driver, not your JDBC code. Same spec/implementation split as JPA/Hibernate, one layer down.

## 1.5 A connection *is* a SQL session

This is the first place the "it's all SQL underneath" theme bites. A JDBC `Connection` is not a dumb pipe — it's a **stateful session** on the Postgres server. That session holds server-side state: the current transaction, the `search_path`, session settings, temporary tables, server-side prepared statements, and the autocommit flag.

The default state of a fresh JDBC connection is **autocommit = on**: every statement runs in its own implicit transaction and commits immediately. The instant you want two statements to be atomic — or you want a persistence context, which needs a stable transaction to live in — something must turn autocommit **off**, issue a `BEGIN`, and later `COMMIT`. In your stack, *Hibernate does this*, driven by `@Transactional`. Which is the whole point of 1.7.

Because a connection carries session state, **it is not safe to share one across threads**, and it must be returned to the pool in a clean state. This is why a leaked transaction (a connection never returned, still mid-`BEGIN`) is so poisonous: it removes a session from the pool *and* may hold locks.

## 1.6 The connection pool

Opening a Postgres connection is expensive — TCP handshake, auth, session setup, tens of milliseconds. Per-request connections would collapse under load. **HikariCP** keeps a small pool of already-open sessions and lends them out. When your code (or Hibernate) asks the `DataSource` for a connection, it borrows one from Hikari and returns it when the transaction ends.

Two facts with outsized consequences:

- **A transaction holds its connection for its entire duration.** Open a transaction, do slow work (an HTTP call, a big computation) mid-transaction, and you've pinned a pooled connection the whole time. Pool exhaustion — "the app hangs under load" — is very often *transactions held open too long*, not too few connections. Keep transactions short and free of I/O.
- **Pool size is one of the highest-leverage knobs in the stack.** Bigger is not better past a point — Postgres has its own connection ceiling, and too many concurrent connections thrash it. As a DevOps-shaped engineer, your instincts here already transfer.

## 1.7 Transactions are a SQL idea

You asked for exactly this, so it gets real estate. A transaction is **not** a Java or Hibernate concept that happens to touch the database — it *is* a database concept that Java merely drives.

At the SQL level a transaction is a bracket: `BEGIN … COMMIT` (or `ROLLBACK`). Everything inside is **atomic** (all-or-nothing), moves the DB between **consistent** states, is **isolated** from other transactions to some degree, and once committed is **durable** — ACID. Postgres implements this with **MVCC** (multi-version concurrency control): each transaction sees a *snapshot* of the data, writers create new row versions rather than overwriting in place, and so **readers don't block writers and writers don't block readers**. This is why Postgres stays lively under mixed load and why its default isolation is usually fine.

**Isolation level** is the knob for *how much* a transaction is protected from concurrent ones. It's a pure SQL concept (`SET TRANSACTION ISOLATION LEVEL …`), and the levels are defined by which anomalies they permit:

| Level | Dirty read | Non-repeatable read | Phantom read | Notes on Postgres |
|---|---|---|---|---|
| READ UNCOMMITTED | — | poss. | poss. | Postgres treats this as READ COMMITTED; it never allows dirty reads |
| **READ COMMITTED** *(PG default)* | no | poss. | poss. | Each statement sees the latest committed snapshot |
| REPEATABLE READ | no | no | no* | Snapshot fixed at first statement; may fail with a serialization error |
| SERIALIZABLE | no | no | no | Full serializability (SSI); can abort txns with `serialization_failure` |

- *dirty read* = seeing another transaction's uncommitted changes.
- *non-repeatable read* = reading a row twice in one transaction and getting different values because someone committed in between.
- *phantom read* = re-running a query and getting new rows that appeared.

**Where this lives in your stack.** By default Hibernate runs at the connection/DB default, i.e. **READ COMMITTED** on Postgres, and that is the correct default for a web app. You override it only for a specific reason (a multi-step read-modify-write that must see a stable snapshot → REPEATABLE READ; a genuinely serializable invariant → SERIALIZABLE, and then you must be ready to *retry* on serialization failures). You can pin isolation per datasource, or per transaction where the framework supports it (Micronaut's `@Transactional` exposes `isolation` and `propagation`; the plain JTA `jakarta.transaction.Transactional` does not — reach for the Micronaut one if you need to set them).

**The Java side is thin.** `@Transactional` on a method means: at method entry, turn autocommit off and `BEGIN`; at normal return, `COMMIT`; on a runtime exception, `ROLLBACK`. That's it. The Java annotation is a bracket-generator for SQL you'd otherwise type. Everything about the persistence context in Part 2 is glued to *this* bracket — the transaction is the container the whole ORM unit-of-work lives inside.

> **The tell — isolation:** stay on READ COMMITTED unless you can name the concurrent anomaly you're preventing. If you can't name it, you don't need a higher level; you need optimistic locking (2.7), which solves the common "two users edited the same row" case far more cheaply than cranking isolation.

## 1.8 The fork in the road: map rows, or map objects

Above JDBC, everything splits into two philosophies. This is *the* choice Practiq made (mostly implicitly), and it's the one you want to make on purpose.

**Path A — stay close to SQL ("row mappers").** You write SQL (or something SQL-shaped) and the tool just runs it and shuffles columns into objects. You think in tables, rows, result sets. No unit of work, no change tracking, no lazy loading. Examples:

- **Raw JDBC** — total control, total boilerplate.
- **`JdbcTemplate` / Micronaut Data JDBC** — thin mappers; you write SQL (or simple derived queries), they handle the `ResultSet` plumbing.
- **jOOQ** — a type-safe SQL DSL: SQL *in Java*, checked against your real schema at compile time. Superb if SQL is your first language and you want the compiler policing it.
- **MyBatis** — SQL in XML/annotations mapped explicitly to objects; common where DBAs own the SQL.

**Path B — think in objects ("ORM").** You declare the object→table mapping *once* (annotations), then work with object graphs; the tool generates the SQL and maintains a **unit of work** (the persistence context). You get change tracking, lazy loading, cascades, caching — real power — for the price of a thick, sometimes leaky abstraction (N+1, `LazyInitializationException`, surprise flushes). This is **JPA/Hibernate**.

| | **Path A — row mapper** | **Path B — ORM (JPA/Hibernate)** |
|---|---|---|
| You think in | tables, rows, SQL | objects, graphs |
| SQL is | written by you, explicit | generated, mostly hidden |
| Change tracking | none — you write every `UPDATE` | automatic (dirty checking) |
| Relationships | manual joins | mapped associations, lazy/eager |
| Read a big report | trivial, SQL does what you say | fights you (projections, N+1) |
| Complex domain writes | verbose, error-prone | cascades handle the graph |
| Surprises | few — what you write is what runs | flush timing, N+1, lazy-init |
| Best when | read-heavy, SQL-shaped, reporting | rich domain model, graph writes |

> **The tell — A vs B:** Practiq is Path B (Micronaut Data **JPA**), which is right for a domain that's genuinely a `Concept → Question → …` graph with a human-review workflow and cross-board reuse. But the choice is *per subsystem, not per app.* If some future read-heavy analytics or export path starts fighting the ORM, drop *that path* to Path A (Micronaut Data JDBC or native SQL) without guilt. Part 3 shows the escape hatches.

---

# Part 2 — JPA, Hibernate, and the frameworks

## 2.1 What JPA is (and is not)

**JPA — Jakarta Persistence — is a specification, not a library that does anything.** It's a document plus a set of interfaces and annotations under `jakarta.persistence.*`. It ships no working engine. Adding JPA to a project means adding *the API jar* (definitions) **plus** *a provider* (Hibernate) that implements it.

Think of JPA as **two bundled things**, because you use them very differently:

1. **A mapping standard** — the annotations: `@Entity`, `@Id`, `@Column`, `@ManyToOne`, `@Enumerated`, `@Table`. Declarative object-relational mapping: "this class is that table, this field is that column, this enum stores as a string." *You use this constantly* — every Practiq entity is nothing but this.
2. **A runtime API** — how you interact at runtime: `EntityManager`, JPQL, the Criteria API, the entity lifecycle. *You use this sometimes directly* (your `QuerySpecification` builds a Criteria `Predicate` — pure JPA runtime API) *and mostly through the framework's abstraction* (you call `repository.findAll(...)` and Micronaut Data drives the `EntityManager` for you).

The thing you almost **never** do is *implement* JPA's interfaces. You don't write an `EntityManager`; Hibernate does. That resolves the "I implement something somehow" fuzziness: **what you implement is entities (JPA annotations) and repository interfaces (a framework feature)** — never the spec itself.

## 2.2 The persistence context — the beating heart

Learn this one properly; every surprising ORM behaviour flows from it.

The **persistence context** is a *unit of work*: a map of the entities Hibernate is currently **managing**, scoped (almost always) to a transaction. It's also called the **first-level cache**. While an entity is "in" the context, Hibernate holds a **snapshot** of it and watches it.

Two guarantees make it more than a cache:

- **Identity guarantee:** within one persistence context, loading the same row twice gives you the *same object instance* (`==`). The context de-duplicates. This is why you don't get two different `Question` objects for id 5 inside one transaction.
- **Unit of work:** all your reads and pending writes accumulate here and are reconciled to the DB *together* at flush — in an order Hibernate controls (2.5), not the order you wrote them.

## 2.3 Entity lifecycle states, and the SQL each transition emits

Every entity is in one of four states. The value of this table is the **SQL column** — each transition is a specific statement (or the deliberate *absence* of one):

| State | Meaning | In context? | SQL on entering this state |
|---|---|---|---|
| **Transient / new** | just `new`ed, no identity | no | none |
| **Managed / persistent** | tracked, changes watched | yes | `INSERT` (on persist) — or a `SELECT` populated it |
| **Detached** | was managed, context closed/evicted | no | none — and this is the trap: changes are silently ignored |
| **Removed** | marked for deletion | yes | `DELETE` at flush |

Transitions: `persist()` moves new→managed (queues an `INSERT`); a query or `find()` loads a row as managed (a `SELECT`); `remove()` moves managed→removed (queues a `DELETE`); the transaction ending, or `detach()/clear()`, moves managed→detached (no SQL, but now dirty checking is off for it — mutate it all you like, nothing happens). `merge()` takes a detached entity and copies its state onto a managed one.

## 2.4 The persistence context is glued to the transaction

This is where 1.7 and 2.2 fuse, and it's the single most clarifying fact in the whole document.

`@Transactional` does two things at once: it brackets a **SQL transaction** (`BEGIN … COMMIT`) *and* it bounds the **persistence context**. They open together and close together. So:

- Entities loaded inside the method are **managed** only for the duration of that transaction.
- At the boundary (`COMMIT`), Hibernate **flushes** pending changes (the `INSERT`/`UPDATE`/`DELETE`s) and then the transaction commits.
- After the boundary, every entity is **detached**.

Now the infamous `LazyInitializationException` is obvious rather than mysterious: a lazy association is a *second `SELECT` that only fires when you touch it, and only works while the transaction/session is open.* Touch it in your controller or JSON serialiser — *after* the `@Transactional` service method returned and the context closed — and there's no open session to run that `SELECT`, so it throws. The fix isn't magic; it's *"fetch what you need while the transaction is still open"* (a `join fetch`, an entity graph, or a projection — 2.8, 3.6).

> **The tell — transaction boundaries:** put `@Transactional` on the *service* method that owns the unit of work, not sprinkled on repositories, and make sure everything you'll serialise is loaded before that method returns. Keep the boundary tight (no HTTP calls inside) so you don't pin a pooled connection (1.6).

## 2.5 Dirty checking and write-behind — the deferred `UPDATE`

**Dirty checking:** you do **not** call `save()` to update a managed entity. At flush, Hibernate compares each managed entity to its snapshot and auto-generates an `UPDATE` for whatever differs:

```java
@Transactional
void approve(Long id) {
    Question q = questionRepository.findById(id).orElseThrow(); // SELECT, now managed
    q.setStatus(Status.APPROVED);                                // just a field set
}   // at COMMIT, Hibernate notices status changed → UPDATE fires
```
```sql
update question set status=?, difficulty=?, type=?, ... where id=?
```
So "I set a field and an `UPDATE` appeared that I never asked for" is *correct*. And its mirror — "I set a field on a **detached** entity and nothing happened" — is also correct: no snapshot is being watched.

**Write-behind / flush timing:** the SQL is not sent when you call the setter. Hibernate **batches** pending work and flushes it (a) automatically just before a query whose result could be affected, and (b) at commit. Consequences:

- **Statement order ≠ code order.** At flush, Hibernate imposes its own order (roughly: inserts, then updates, then collection changes, then deletes), regardless of the order you wrote the Java. This occasionally trips a foreign-key or unique constraint in a way that looks impossible until you see the actual flush order in the SQL log.
- **Batching is a real perf lever.** With `hibernate.jdbc.batch_size` set (and `order_inserts`/`order_updates` on), Hibernate sends many `INSERT`s as one JDBC batch — far fewer round-trips. Big caveat that ties straight into identity generation, below.

## 2.6 Identity: `@Id`, `@GeneratedValue`, and why the strategy is a SQL/perf decision

How a new row gets its primary key is a SQL concern with a direct performance consequence, and it's a classic blind choice.

| Strategy | What Postgres does | Batch inserts? | Notes |
|---|---|---|---|
| **IDENTITY** | column is `GENERATED … AS IDENTITY` / serial; the `INSERT` returns the id | **No** | Hibernate needs each id back immediately, so it *cannot* batch inserts |
| **SEQUENCE** | a real Postgres `SEQUENCE`; Hibernate pre-allocates a block of ids | **Yes** | Fewer round-trips; the recommended strategy on Postgres |
| TABLE | a table emulating a sequence | yes-ish | slow, portable-only; avoid |
| AUTO | provider picks (often SEQUENCE on PG) | depends | be explicit instead |

The non-obvious bit: **IDENTITY silently disables insert batching.** Because the id is generated *by the `INSERT` itself*, Hibernate must run each insert and read back its key one at a time — it can't bundle a batch. **SEQUENCE** lets Hibernate call `nextval` and pre-allocate a block (default allocation size 50), so it knows the ids up front and can batch. For anything write-heavy — and a **seed pipeline is exactly that** — SEQUENCE is the right call on Postgres.

> **The tell — id generation:** on Postgres, default to **SEQUENCE**; only pick IDENTITY when you truly don't care about insert throughput and want the simplest possible mapping. If your seed/bulk inserts feel slow, this is the first thing to check.

## 2.7 Optimistic locking: `@Version` — concurrency control you can see in the `WHERE`

The common concurrency problem in a web app isn't exotic isolation anomalies — it's the **lost update**: two users load the same `Question`, both edit, both save; the second silently clobbers the first. Cranking isolation is the wrong, expensive fix. The right one is a `@Version` field:

```java
@Version
private long version;
```
Now every update Hibernate generates carries the version in its `WHERE` and bumps it:
```sql
update question set status=?, ..., version=? where id=? and version=?
```
If someone else updated the row first, the version no longer matches, **0 rows are affected**, and Hibernate throws `OptimisticLockException` instead of silently overwriting. No locks are held between the read and the write — hence *optimistic*. You catch the exception and tell the user "someone changed this; reload."

The alternative, **pessimistic** locking, is a `SELECT … FOR UPDATE` that holds a row lock until commit (`LockModeType.PESSIMISTIC_WRITE`) — use it only when you must serialise contending writers at the DB and can't tolerate a retry.

> **The tell — locking:** Practiq's human review workflow is the textbook optimistic-locking case. Two reviewers approving/editing the same `Question` is precisely "lost update." Add `@Version` to entities that a human edits over an HTTP round-trip; it's nearly free and turns a silent data-loss bug into a clean, catchable conflict. Reach for pessimistic only for true serialise-at-the-DB needs (rare here).

## 2.8 Fetching: lazy vs eager, N+1, and the four fixes

Associations don't come for free — each is a potential extra `SELECT`. `@ManyToOne`/`@OneToOne` default to **EAGER**, collections (`@OneToMany`/`@ManyToMany`) default to **LAZY**. Lazy means "load it in a separate `SELECT` the moment you touch it, if the session is open."

The **N+1 problem** is the canonical failure: load 50 `Question`s (**1** query), loop and read `q.getConcept()` on each → **50** more queries. 1 + N. It's invisible until you look at the SQL — which is why 3.1 matters. The four fixes, each with its SQL shape:

| Fix | How | SQL | When |
|---|---|---|---|
| **JOIN FETCH** | JPQL `join fetch q.concept` | one query with a `join` | you know up front you need the association for these rows |
| **Entity graph** | `@EntityGraph` on the method | same as join fetch, but reusable/declarative | want to keep the query and vary the fetch |
| **Batch fetching** | `@BatchSize` / `hibernate.default_batch_fetch_size` | N+1 becomes 1 + (N/size) via `where concept_id in (?,?,…)` | many small lazy loads you can't easily join |
| **Projection/DTO** | select only the columns you need | one lean query, no association objects at all | read-only list/report endpoints |

`JOIN FETCH` example:
```java
@Query("SELECT q FROM Question q JOIN FETCH q.concept WHERE q.conceptId = :cid")
```
```sql
select q1_0.id, ..., c1_0.id, c1_0.name
from question q1_0 join concept c1_0 on c1_0.id=q1_0.concept_id
where q1_0.concept_id=?
```
One query instead of 1+N.

> **The tell — fetching:** make associations **LAZY** by default (even the to-one ones — override the eager default), and fetch *explicitly and per-query* what a given endpoint needs. Eager-by-default is how you end up loading half the object graph on every call. For read-heavy list endpoints, prefer a **projection** over an entity (3.8) — you sidestep the whole N+1 question by never having the association objects in the first place.

## 2.9 Cascades and orphan removal — object-graph writes → multiple statements

Cascade tells Hibernate to propagate an operation across an association: `cascade = PERSIST` means persisting a parent also `INSERT`s its children; `orphanRemoval = true` means removing a child from the parent's collection `DELETE`s it. One `save` of a parent can become several statements. Cascade for **composition** (children the parent truly owns and that shouldn't outlive it); *don't* cascade across **shared references** (a `Concept` referenced by many `Question`s must not be deleted because one `Question` was).

## 2.10 Caching: L1 vs L2

The persistence context is the **first-level (L1) cache** — always on, transaction-scoped, gives you the identity guarantee. The **second-level (L2) cache** is optional, shared across transactions, and caches entities/collections/query results between requests. L2 is a genuine footgun early: it adds invalidation complexity and stale-read risk. Leave it off until a *measured* read-hotspot justifies it.

## 2.11 Hibernate — the provider — and the dialect

**Hibernate is the implementation of the JPA spec** (EclipseLink is the main alternative; rarely chosen for new work). It manages the persistence context, runs dirty checking, and — crucially — **generates the SQL**. The one Hibernate component to know by name is the **dialect**: `PostgreSQLDialect` teaches Hibernate Postgres's specific SQL — its pagination syntax (`limit/offset`), type mappings, functions, sequence support. Same JPQL, different dialect, different SQL. This is exactly what makes JPQL *portable* in a way native SQL isn't.

## 2.12 Where Spring Data and Micronaut Data fit — and the compile-vs-runtime split

Working with the raw `EntityManager` is verbose. The **data-access frameworks** kill that boilerplate: you declare a *repository interface* and the framework provides the implementation.

```java
@Repository
interface QuestionRepository extends CrudRepository<Question, Long>,
                                     JpaSpecificationExecutor<Question> {
    List<Question> findByConceptIdAndStatus(Long conceptId, Status status);
}
```
You never write the body — the framework reads the method name, works out the query, and generates the code. **This repository layer is not JPA**; it's a framework feature that *delegates into* JPA/Hibernate. That's the last piece of "spring/micronaut sits somewhere on top": it sits *above* JPA, generating repositories that call the `EntityManager` you'd otherwise call by hand.

The two mainstream frameworks differ in **when** they do their work — the most important thing to understand about why you're on Micronaut:

- **Spring Data JPA — runtime.** Method names are parsed and repository proxies built **at startup / first use** via reflection and dynamic proxies. Flexible, huge ecosystem; but slower startup, more runtime magic, and a typo'd `findByConcptId` explodes at **runtime**.
- **Micronaut Data — compile time.** An annotation processor parses method names and **generates the query and implementation during compilation** — no runtime reflection, no proxies. Faster startup, GraalVM-native-friendly, lower memory, and the big one: **a bad query method is a compile error.** Errors are refused, not deferred. This *is* Micronaut's philosophy, and it's why "engineer intentionally" and "Micronaut" pair naturally.

### The fork inside Micronaut Data you must not blur

Micronaut Data ships in **two variants**, and they're very different animals:

| | **Micronaut Data JPA** (Practiq) | **Micronaut Data JDBC** |
|---|---|---|
| Engine underneath | **Hibernate** at runtime | none — a lightweight mapper |
| Persistence context / dirty checking | **yes** | **no** |
| Lazy loading | **yes** (and N+1 risk) | **no** — you fetch explicitly |
| Criteria / `QuerySpecification` | yes (Hibernate-backed) | its own, limited |
| Mental model | Path B (objects/graphs) | Path A (rows, near SQL) |

You're on **Micronaut Data JPA**: repositories are generated at compile time, *but at runtime Hibernate is fully present* — persistence context, dirty checking, lazy loading, the lot. So you get Micronaut's compile-time repository generation **and** Hibernate's ORM machinery. When Practiq mentions `@Enumerated(EnumType.STRING)` and a `QuerySpecification` with `toPredicate(root, query, cb)`, those are *Hibernate/JPA* concepts surfacing through a *Micronaut* repository — both true at once, and now you can see exactly where each lives.

## 2.13 The full stack, restated in Practiq's actual components

```
Question entity + QuestionRepository + your QuerySpecification
        │
Micronaut Data (compile-time generated repo impl)        ← framework
        │
Jakarta Persistence API  (EntityManager, Criteria)       ← spec
        │
Hibernate 6  (persistence context, dirty checking,       ← provider
              SQL generation, PostgreSQLDialect)
        │
HikariCP  (connection pool; each connection = a session)
        │
JDBC  (PreparedStatement, ResultSet, tx control)         ← spec
        │
pgjdbc  (PostgreSQL driver)                              ← implementation
        │
PostgreSQL  (MVCC, transactions, sequences)
```

---

# Part 3 — The query APIs and the SQL they become

Six meaningfully-different ways to express "get me these rows." Three are JPA proper (JPQL, Criteria, native), two are framework features (derived finders, specifications), and named queries package the first. Below: shape, SQL, where errors surface, when to reach for it — using your real domain (`Question` filtered by `conceptId`, with `status = APPROVED`).

## 3.1 First: make the SQL visible

You cannot engineer this layer intentionally while the SQL is invisible. Turn it on in `application.yml` (use logging, not bare `show_sql`):

```yaml
jpa:
  default:
    properties:
      hibernate:
        format_sql: true          # pretty-print
logger:
  levels:
    org.hibernate.SQL: DEBUG                # every statement
    org.hibernate.orm.jdbc.bind: TRACE      # the values bound to each ?
```
`org.hibernate.SQL` at `DEBUG` prints each statement; the `bind` logger at `TRACE` shows what got bound to each `?`. For real timings and the fully-inlined statement (ideal when hunting an N+1 or a surprise flush), **p6spy** wraps the driver and logs at the JDBC layer. All SQL below is roughly Hibernate 6 output (its aliases look like `q1_0`, params render as `?`).

## 3.2 Derived query methods (finder methods)

**What:** a repository method whose *name* is the query. A framework feature, generated **at compile time** in Micronaut.
```java
List<Question> findByConceptIdAndStatus(Long conceptId, Status status);
```
```sql
select q1_0.id, q1_0.concept_id, q1_0.difficulty, q1_0.status, q1_0.type
from question q1_0 where q1_0.concept_id=? and q1_0.status=?
```
**Errors surface:** compile time. **Reach for it when:** the query is simple, static, ≤2–3 fixed conditions, and reads well as a name. It falls over the moment a condition is *optional* — you can't say "filter by status only if one was supplied" in a method name. That cliff is where specifications take over.

## 3.3 JPQL

**What:** JPA's own query language — SQL-shaped but over **entities and fields** (`Question`, `q.conceptId`), not tables and columns. The provider translates it to dialect SQL; this is the portability layer.
```java
@Query("SELECT q FROM Question q WHERE q.conceptId = :cid AND q.status = :status")
List<Question> findApproved(Long cid, Status status);
```
Produces the same SQL as the finder above — JPQL *becomes* SQL via the dialect. It's also the natural home for `JOIN FETCH` (2.8), aggregates (`COUNT`, `GROUP BY`), and anything too rich for a method name. **Errors surface:** application **startup** (Hibernate validates JPQL against the metamodel on boot) — a typo'd field fails fast, not on first request. **Reach for it when:** richer than a finder but still essentially *static*, and you want it readable.

## 3.4 The Criteria API

**What:** JPA's *programmatic, type-safe* query builder. You assemble the query from objects — `Root`, `CriteriaQuery`, `CriteriaBuilder`, `Predicate` — instead of a string. Verbose, but **composable at runtime**. This is the machinery your `QuerySpecification` stands on.
```java
CriteriaBuilder cb = ...;
CriteriaQuery<Question> cq = cb.createQuery(Question.class);
Root<Question> q = cq.from(Question.class);
cq.where(cb.equal(q.get("conceptId"), conceptId));
```
Same `where q1_0.concept_id=?` SQL — Criteria and JPQL are two front-ends to one query engine. **Errors surface:** compile time for structure, but string attribute names (`q.get("conceptId")`) fail at **runtime** unless you use the generated **static metamodel** (`Question_.conceptId`), which pushes even that to compile time — worth generating if you lean on Criteria. **Reach for it when:** the query must be **built dynamically** — conditions unknown until runtime. Which is the filtering case, and why the next item exists.

## 3.5 `QuerySpecification` / `Specification`

**What:** a thin, composable wrapper over Criteria. A `QuerySpecification<T>` is essentially one method:
```java
Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder cb);
```
Each spec is a reusable `where`-fragment; compose with `and`/`or` and hand to a `JpaSpecificationExecutor` method (`findAll(spec)`). The idiomatic answer to dynamic filtering in both frameworks.
```java
static QuerySpecification<Question> hasConcept(Long id) {
    return (root, query, cb) -> cb.equal(root.get("conceptId"), id);
}
static QuerySpecification<Question> isApproved() {
    return (root, query, cb) -> cb.equal(root.get("status"), Status.APPROVED);
}
// controller: conceptId optional, status always APPROVED
var spec = QuerySpecification.where(isApproved());
if (conceptId != null) spec = spec.and(hasConcept(conceptId));
questionRepository.findAll(spec);
```
The SQL **grows to match the runtime conditions**: `where q1_0.status=?` alone, or `where q1_0.status=? and q1_0.concept_id=?` when a `conceptId` is supplied — the entire point. **Errors surface:** compile time for structure, runtime for string attributes (metamodel removes this). **Reach for it when:** **optional/dynamic filters** — the `GET /api/v1/questions?conceptId=` case, where a filter may or may not be present and you'll add more dimensions later. This is the right tool for Practiq's filtering, and why you're building it in Sprint 0.2. It also *dictates* your test strategy: because each spec is a lambda over Criteria objects, instance equality is meaningless — hence `any()` + `verify()` at component level, and `.toPredicate(root, query, cb)` against mocked Criteria objects at unit level. The API's shape drives the tests; that's the sign you're working *with* the grain of the tool.

## 3.6 Native SQL

**What:** raw SQL straight to the driver, bypassing JPQL translation — the escape hatch to Path A.
```java
@Query(value = "SELECT * FROM question WHERE concept_id = :cid", nativeQuery = true)
List<Question> findRaw(Long cid);
```
Runs verbatim (Hibernate can still map the `ResultSet` back to `Question`). **Errors surface:** runtime (the DB validates). **Reach for it when:** JPQL/Criteria can't express or can't do it efficiently — Postgres-specific features (JSONB operators, `INSERT … ON CONFLICT`, CTEs, window functions, full-text search) or a hand-tuned query. You trade portability for power. Your **seed pipeline's single staging-table CTE** is a natural native/Flyway citizen precisely because it's Postgres-shaped and CTE-based — correctly kept *out* of the entity/JPQL layer.

## 3.7 Named queries

**What:** a JPQL (or native) query defined once with `@NamedQuery`, referenced by name, validated at **startup**. Purely a *packaging* choice — same JPQL. Rarely worth it alongside finders and specifications; here so you recognise it in the wild.

## 3.8 Cross-cutting concerns that change the SQL

These aren't separate APIs — they modify whatever query you ran, and each has a SQL shape and a decision attached.

**Pagination.** `Pageable`/`limit`/`offset` becomes SQL `limit`/`offset`:
```sql
select ... from question q1_0 where q1_0.status=? order by q1_0.id limit ? offset ?
```
Two costs to know: (1) returning a `Page` (with a total count) fires a **second** `select count(*)` query — skip it (`Slice`/`List`) if you don't need the total; (2) **offset pagination degrades on deep pages** — Postgres still scans and discards the offset rows. For large datasets, **keyset (seek) pagination** — `where id > ? order by id limit ?` — stays fast because it seeks instead of counting. *Tell:* fine to start with offset; switch to keyset when pages get deep or the table gets big.

**Sorting.** `Sort`/`order by` → SQL `ORDER BY`. Dynamic sort composes with specifications. Nothing exotic, but note sorting on an unindexed column is a sort in the DB — check the index.

**Projections — entity vs DTO vs interface.** Selecting a whole entity means `select` *all* mapped columns *and* putting a managed object in the context. A **projection** selects only the columns you need into a DTO (JPQL constructor expression `select new com.practiq.QuestionSummary(q.id, q.status)`) or an interface projection:
```sql
-- entity:      select q1_0.id, q1_0.concept_id, q1_0.difficulty, q1_0.status, q1_0.type, ...
-- projection:  select q1_0.id, q1_0.status                 -- only what you asked for
```
Projections are **not managed** (read-only), lighter, and sidestep N+1 entirely because there are no association objects to lazily load. *Tell:* for read-heavy **list/summary endpoints** (your `GET /questions` list is a candidate), return a projection, not the entity. Reach for the full entity when you're going to *modify* it in the same transaction.

**Bulk update/delete via JPQL.** `UPDATE Question q SET q.status = :s WHERE …` compiles to a single SQL `UPDATE` over many rows — efficient, but it **bypasses the persistence context**: no dirty checking, no `@Version` bump (unless you handle it), and any already-loaded entities in the L1 cache go **stale**. *Tell:* great for mass changes (e.g. bulk-approve), but run it in its own transaction and don't mix it with entities you've loaded and still hold; clear the context after.

## 3.9 The comparison, at a glance

| API | Level | Dynamic? | Type-safe? | Errors surface | SQL control | Reach for it when |
|---|---|---|---|---|---|---|
| **Derived finders** | Framework | No | Yes (name) | **Compile** (Micronaut) | Low | Simple, static, ≤3 fixed conditions |
| **JPQL** | JPA | No | No (string) | **Startup** | Low–med | Static but richer; `JOIN FETCH`, aggregates |
| **Criteria** | JPA | **Yes** | With metamodel | Compile/runtime | Med | Dynamic queries, raw |
| **QuerySpecification** | Framework→Criteria | **Yes** | With metamodel | Compile/runtime | Med | **Optional/composed filters (your case)** |
| **Native SQL** | JPA→raw | Yes | No | **Runtime** (DB) | **Total** | DB-specific features, tuning, CTEs |
| **Named queries** | JPA | No | No (string) | **Startup** | Low–med | Reused static JPQL (rarely) |

The trend is the lesson: moving *down* toward native SQL buys **control** and **DB-specific power** at the cost of **portability** and **early error detection**; moving *up* toward finders buys **safety** and **brevity** at the cost of **dynamism**. There's no "best" — there's a right tool per query, and now you can name the trade.

---

# Part 4 — Data modelling and physical optimisation

Everything so far assumed the tables already exist and are shaped sensibly. This part is about the shaping: how tables should connect, how to make the database *find* rows fast (indexes), the one place your instincts from other databases will actively mislead you (clustered indexes), and how to see and fix a slow query. It sits **below** JPA — this is Postgres itself — and in Practiq it lives in **Flyway migrations**, not entity annotations. Hibernate *can* emit `@Index`/DDL, but you've put schema in Flyway (Flyway owns schema, never content), so every index below is a migration statement, versioned next to the schema it serves.

## 4.1 How tables connect: keys and relationships

**Primary key (PK).** Uniquely identifies a row: not-null, unique, the anchor other tables point at. Two flavours:

- **Surrogate key** — a synthetic id with no business meaning (a `BIGINT` from a sequence). Stable, uniform, never needs to change. Practiq's default, and the right one.
- **Natural key** — a real-world unique attribute (an ISBN, an email). Meaningful but risky: business values change, and changing a PK ripples through every FK that references it. Prefer a surrogate PK **and** a `UNIQUE` constraint on the natural key — you get identity stability *and* the business-uniqueness guarantee.

**Foreign key (FK).** A column holding another table's PK value, with a constraint that enforces **referential integrity**: you can't insert a `question.concept_id` that has no matching `concept`, and (by default) can't delete a `concept` still referenced by a `question`. The FK is where a relationship physically lives. `ON DELETE` decides what happens when the parent goes: `RESTRICT`/`NO ACTION` (block it — the safe default), `CASCADE` (delete the children too), `SET NULL` (orphan them). Pick deliberately; DB-level `CASCADE` is powerful and easy to regret.

**The cardinalities, and how each is physically built:**

- **One-to-many / many-to-one** — the FK lives on the **many** side. One `Concept`, many `Question`s → `question.concept_id → concept.id`. The workhorse relationship. (JPA: `@ManyToOne @JoinColumn` on `Question`; optional `@OneToMany(mappedBy)` on `Concept`.)
- **Many-to-many** — can't live on either side, so you add a **junction (join) table** with two FKs and a composite PK. A `Concept` tagged to many `SpecSection`s *and* a `SpecSection` holding many `Concept`s → `concept_spec_section(concept_id, spec_section_id)`, PK = both columns. **This junction table is exactly what makes your board-agnostic tagging work** — one canonical `Concept` reused across boards, no duplication. (JPA: `@ManyToMany @JoinTable`.) If a question can be tagged to *several* concepts, "question↔concept" is also many-to-many (a `question_concept` table); if it belongs to exactly one, it's a plain FK. That's a modelling decision for you — the `?conceptId=` filter reads the same either way, but the write model and the SQL differ.
- **One-to-one** — a FK with a `UNIQUE` on it, or a shared PK. Rare; usually a nudge to either merge the tables or split off genuinely optional/heavy data (e.g. a large `question_body` you don't always load).

## 4.2 Normalisation, briefly and without dogma

One rule carries most of the weight: **each fact lives in exactly one place.** Don't copy a concept's name onto every question that references it — store it once in `concept` and point at it. Payoff: no *update anomalies* (rename once, not across 500 rows) and no inconsistency (500 rows can't disagree). Your board-agnostic Concept design *is* this principle in action: one canonical Concept, tagged, not a copy per board.

Third normal form (roughly: every non-key column depends on the key, the whole key, and nothing but the key) is the sane default for a transactional schema. **Denormalise deliberately and rarely** — a cached count, a duplicated column to skip a hot join — and only once you've *measured* the read cost and accepted the write-time duplication it creates. Premature denormalisation is how schemas rot.

## 4.3 Indexes: what they are and the trade they make

An index is a **separate data structure** — a **B-tree** by default in Postgres — that lets the DB locate rows without reading the whole table: the index at the back of a book versus flipping every page. The trade is unavoidable and worth stating flat:

**Indexes make reads faster and writes slower.** Every `INSERT`/`UPDATE`/`DELETE` must maintain every index on the table, and each index costs disk. So you do **not** index everything — you index what your queries actually *filter, join, and sort* on.

**The high-value candidates:**

- **Foreign-key columns you join or filter on.** Critical gotcha: **Postgres does *not* auto-index FK columns.** The constraint is enforced, but no index exists unless you add one. `question.concept_id` drives your list endpoint → it wants an index.
- Columns in `WHERE`, `JOIN … ON`, and `ORDER BY`.

**B-tree** (the default) handles equality and ranges — `=`, `<`, `>`, `BETWEEN`, `ORDER BY`, prefix `LIKE 'foo%'` — and is what you want the large majority of the time.

### Composite (multi-column) indexes — column order matters

An index on `(concept_id, status)` is sorted by `concept_id` first, then `status` within each. The **leftmost-prefix rule**: it serves queries filtering on `concept_id`, or on `concept_id AND status` — but **not** `status` alone. So for Practiq's core query (filter by `concept_id`, always `status = 'APPROVED'`), a composite index on `(concept_id, status)` is close to ideal, with the most-selective / always-present column **first**. Get the order wrong and Postgres simply won't use it.

### Unique indexes and unique constraints

A **unique constraint** says "no two rows share this value"; Postgres enforces it *with* a **unique index** underneath — so a unique constraint hands you an index for free. Use it on natural keys (a `concept.slug`, an external question code). Two Postgres powers to know:

- **Partial unique index** — uniqueness *only where a condition holds*: `CREATE UNIQUE INDEX … ON question (external_ref) WHERE status = 'APPROVED'` means "at most one APPROVED row per external_ref, but drafts may duplicate." Perfect for workflow states.
- **Multi-column unique** — the junction PK `(concept_id, spec_section_id)` is a unique index that also stops you tagging the same pair twice.

### Partial indexes — index only the rows you actually query

If your public endpoint *only ever* filters `status = 'APPROVED'`, an index over *all* statuses wastes space and write effort on rows you never read that way. A **partial index** covers just the hot subset:

```sql
CREATE INDEX idx_question_concept_approved
  ON question (concept_id) WHERE status = 'APPROVED';
```

Smaller, cheaper to maintain, and a strong fit for Practiq specifically — approval is already baked into your public read path at the controller signature, so the index can mirror the exact same constraint.

### Covering indexes / index-only scans

If an index contains *every column a query needs*, Postgres can answer from the index alone and never touch the table — an **index-only scan**, the fast path. The `INCLUDE` clause adds non-key payload columns for exactly this:

```sql
CREATE INDEX idx_q_concept ON question (concept_id) INCLUDE (status, difficulty);
```

This is the natural partner to **projections** (3.8): a lean projection served by a covering index is about as fast as Postgres reads get.

### Other index types (know they exist, reach rarely)

- **GIN** — JSONB, arrays, full-text search. If you ever store question content/metadata as `JSONB`, GIN is how you index *inside* it.
- **GiST** — ranges, geometric, similarity search.
- **BRIN** — a tiny index for very large, naturally-ordered (append-only / time-ordered) tables; the closest Postgres gets to a cheap "physical-order" index.
- **Hash** — equality only; B-tree almost always wins, so skip it.

## 4.4 "Clustered indexes" — the mental model to unlearn on Postgres

You named this one specifically, and it's the single place instincts from SQL Server or MySQL/InnoDB will actively mislead you.

In **SQL Server / InnoDB**, a table *is* its clustered index: rows are physically stored **in primary-key order**, and the clustered index and the table are the same object. Every other index is "secondary" and points back into that clustered order. This is why "which column is the clustered index?" is a real design question there.

**Postgres has no clustered indexes.** Postgres tables are **heap-organised**: rows sit in a heap in roughly insertion order, and *every* index is secondary — each index entry points at a physical location (a `ctid`) in the heap. There is no "the table is sorted by its PK." Consequences:

- The PK index is just another B-tree; it does **not** dictate physical row order.
- There *is* a one-shot `CLUSTER table USING some_index` command that physically reorders the heap to match an index — **but it's one-time, takes an exclusive lock, and is not maintained**: subsequent inserts/updates land wherever there's free space, and the ordering decays. It's rarely used.
- The locality benefit people *want* from clustering — fewer page reads on range scans — you get on Postgres through good index design and **index-only scans** (covering indexes above), and on giant ordered tables through **BRIN**. You almost never think about physical clustering at all.

Net: when a SQL Server tutorial says "make this the clustered index," the Postgres translation is usually just "create a normal index; the heap + secondary-index model handles it." Don't go hunting for a clustered index to configure — there isn't one.

## 4.5 Optimisation: see the plan, then fix it

The spine of this whole document — *ask what SQL runs* — goes one level deeper here: **ask what plan the SQL runs under.** Postgres's planner is **cost-based**: from table **statistics** it estimates whether a full scan or an index is cheaper, and picks. You inspect its choice with `EXPLAIN` (the plan) and `EXPLAIN ANALYZE` (plan *plus* real execution timing):

```sql
EXPLAIN ANALYZE
SELECT id, status FROM question WHERE concept_id = 42 AND status = 'APPROVED';
```

What you're reading:

- **Seq Scan** — reading the whole table. Fine on a tiny table or when you're returning most of it; a red flag on a large table with a selective filter (a missing or unused index).
- **Index Scan** — using an index to find rows, then fetching each from the heap.
- **Index Only Scan** — answered entirely from a covering index, heap untouched. The fast path.
- **Estimated vs actual rows** — if these diverge wildly, statistics are stale; run `ANALYZE` (or check autovacuum) so the planner has good numbers to plan with.

**The practical wins, in the order you'd usually try them:**

1. **Index the filter/join/sort columns** — especially FK columns Postgres didn't index for you.
2. **Match a composite index to your real query** (right columns, right order) — `(concept_id, status)`, or the partial `WHERE status='APPROVED'` index, for Practiq's read path.
3. **Select fewer columns** (projections, 3.8) — enables index-only scans and moves less data.
4. **Kill N+1** (2.8) — frequently the actual cause of "the endpoint is slow," and invisible without SQL logging.
5. **Don't over-index** — every index taxes every write; a table with fifteen indexes is usually a schema no one has measured.

**One trap that will bite you specifically:** on tiny seed data, *everything is a Seq Scan and everything looks fine*, because scanning 40 rows is cheaper than any index lookup. Index problems only surface at volume. So validate performance against realistically-sized data, not the seed set — otherwise you ship a schema that's fast in dev and folds in production. And keep indexes **in Flyway migrations**, versioned with the schema they optimise, not in entity annotations — same clean boundary you already drew.

## 4.6 A note on column types

Types are part of modelling. Quick Postgres-specific guidance: use `BIGINT` for surrogate keys (you'll blow past `INT`'s ~2.1bn eventually, and migrating a live PK is misery); prefer `TEXT` over `VARCHAR(n)` unless you truly need a length cap (identical performance in Postgres, no arbitrary limits); use `TIMESTAMPTZ` not `TIMESTAMP` for instants; use `NUMERIC` for exact decimals, never `FLOAT` for anything you'll sum or compare precisely. For your **uppercase string enums** (`@Enumerated(EnumType.STRING)`): a sound, flexible choice — readable in the DB and trivial to extend with a new value. The alternatives — a native Postgres `ENUM` type (compact but rigid; altering it is a migration chore) or a `SMALLINT` code (tiny but opaque, and you lose the self-documenting value) — trade readability for a handful of bytes. At your stage, string enums are right; add a `CHECK` constraint or an FK to a small reference table if you want the DB itself to police the allowed set.

---

# Part 5 — Deeper JPA machinery

Parts 1–4 are the load-bearing model. This part is the next ring out: the tools you'll reach for once the basics are habit — the ones you named (Metamodel API, entity graphs, caching, subqueries) plus the handful of others I'd actively flag for where Practiq is. Same rule throughout: each is tied to the SQL it changes and comes with a "when."

## 5.1 The Metamodel API — static vs dynamic

There are two different things both called "metamodel," and people conflate them.

**The static metamodel** is the one that will improve your code *this sprint*. It's a set of generated classes — `Question_`, `Concept_` — emitted at compile time by an annotation processor (the standard JPA static metamodel generator, `hibernate-jpamodelgen`, added as an `annotationProcessor` dependency). For each entity `Question` it produces a `Question_` with a typed static field per attribute: `Question_.conceptId`, `Question_.status`, `Question_.difficulty`. You then swap the stringly-typed Criteria access for the typed one:

```java
// before — string, fails at RUNTIME if the field is renamed or mistyped
cb.equal(root.get("conceptId"), id)
// after — typed, fails at COMPILE TIME, and rename-refactors it for you
cb.equal(root.get(Question_.conceptId), id)
```

That is the exact runtime-vs-compile gap from 3.4/3.5 closed. It matters *specifically* for your `QuerySpecification` factories, because their one structural weakness is those `root.get("...")` strings — the static metamodel turns a typo or a field rename from a runtime surprise into a red squiggle, and complements your unit tests (the `.toPredicate(root, query, cb)` calls) by making the attribute references themselves compiler-checked.

**The dynamic metamodel** is a different beast: `entityManager.getMetamodel()` returns a `Metamodel` you can walk at runtime — `EntityType<Question>`, its `Attribute`s, their types. It's reflection over your mapping, used by *generic* libraries and frameworks (validators, mappers, admin-UI generators) that must work over any entity. You'll rarely touch it in application code; recognise it so you don't confuse it with the static one.

> **The tell:** if you write Criteria/Specifications by hand (you do), generate the static metamodel — it's near-free and removes the only fragile part of that code. The dynamic metamodel is a library-author's tool; leave it be unless you're building something generic over arbitrary entities.

## 5.2 Entity graphs — declarative fetch plans

Entity graphs solve the fetching problem (2.8) without either of the two blunt instruments: hardcoding `fetch = EAGER` on the mapping (which then over-fetches on *every* query) or hand-writing a bespoke `JOIN FETCH` for every variation. An entity graph says, *for this query only,* "also fetch these associations," and the provider folds them into one query.

The JPA-standard form is `@NamedEntityGraph` on the entity plus an `EntityGraph` hint on the query; there are two hint flavours — a **fetch graph** (`jakarta.persistence.fetchgraph`: fetch *exactly* the listed attributes eagerly, everything else lazy) and a **load graph** (`jakarta.persistence.loadgraph`: fetch the listed attributes eagerly *on top of* the mapping defaults). Micronaut Data's ergonomic equivalent, and the one you'll likely use, is the `@Join` annotation on a repository method *(verified against the Micronaut Data docs: for JPA it generates a `JOIN FETCH`, `Type.FETCH` is the default, and the annotation is repeatable for multiple associations)*:

```java
@Join(value = "concept", type = Join.Type.FETCH)
List<Question> findByStatus(Status status);
```

**SQL produced:** the same single join as a `JOIN FETCH` — N+1 gone:
```sql
select q1_0.id, ..., c1_0.id, c1_0.name
from question q1_0 join concept c1_0 on c1_0.id=q1_0.concept_id
where q1_0.status=?
```

The reason this earns its own tool rather than "just write JOIN FETCH": you **cannot** bolt a `JOIN FETCH` onto a *derived finder* (there's no query string to edit), but you *can* annotate that finder with `@Join`/an entity graph — so you keep the clean method-name query and still control the fetch. One hard caveat, straight from the SQL: fetching **two collections** in one graph produces a cartesian product (Hibernate throws `MultipleBagFetchException` for two `List`s, and even where it doesn't, the row count multiplies). **Fetch at most one collection per query**; fetch the rest separately or via batch fetching.

> **The tell:** reach for an entity graph / `@Join(FETCH)` when a *specific* endpoint needs an association and you want to keep the query otherwise declarative — especially over a derived finder. Keep collections to one per graph.

## 5.3 Caching, in depth

Recapping 2.10 and then going deeper, because "should I cache this?" is a decision you'll face and the footguns are real.

**L1 — the persistence context.** Always on, transaction-scoped, gives the identity guarantee. A `find()` for an id already loaded in this transaction returns the same instance with **no SQL**. You don't configure it; you just benefit.

**L2 — the shared, opt-in cache.** Lives *across* transactions and sessions, backed by a provider (Caffeine, Ehcache, Redis), enabled per entity with `@Cacheable`. On a cache hit, Hibernate reconstructs the entity from the cache and issues **no `SELECT`**. It has **concurrency strategies**, and picking the right one is the whole game:

- `READ_ONLY` — for data that never changes after insert. Fastest, safest. 
- `NONSTRICT_READ_WRITE` — rare updates, brief staleness tolerable.
- `READ_WRITE` — updates happen, needs consistency; uses soft locks.
- `TRANSACTIONAL` — full transactional cache, needs a JTA setup; heavy.

**The query cache** is a separate thing: it caches the *result — the row ids —* of a specific query+parameter combination, and it only works alongside the L2 entity cache. Its weakness is invalidation: **any** write to a table invalidates all cached queries touching it, so on anything write-heavy it thrashes and costs more than it saves.

The footguns, all of which are staleness in disguise: cache invalidation across a *cluster* is genuinely hard; **bulk JPQL updates/deletes bypass the cache** (3.8) and leave it stale until eviction; and the query cache's blunt invalidation makes it a poor fit for busy tables.

> **The tell:** L2 is for **read-mostly reference data with a clear eviction story** — and Practiq has an obvious candidate in `Concept`/`SpecSection`/board metadata, which is read constantly and changes rarely (`READ_ONLY` or `NONSTRICT_READ_WRITE`). But *not now*: it adds invalidation complexity, and until you've measured a read hotspot you're solving a problem you don't have. Leave L2 off, note the candidates, revisit when a profiler points at them. Query cache: probably never for this app.

## 5.4 Subqueries — filtering parents by their children

A subquery is a nested `SELECT` inside another query, and it's the right tool for a whole class of questions that read awkwardly as joins: "concepts that *have* an approved question," "concepts with *no* approved questions yet," "questions harder than the average for their concept." Two shapes:

- **Uncorrelated** — the inner query is self-contained and runs once (e.g. "questions whose difficulty is above the overall average").
- **Correlated** — the inner query references the outer row, conceptually running per outer row (e.g. "concepts that have at least one approved question" — the inner query points back at the outer concept). The planner usually rewrites these into semi-/anti-joins, so "runs per row" is the mental model, not the physical reality.

**In JPQL**, via `EXISTS`, `IN`, `ANY`/`ALL`, or a scalar subquery. The canonical Practiq one — concepts that actually have approved questions, for a syllabus view:
```java
@Query("""
    SELECT c FROM Concept c
    WHERE EXISTS (SELECT 1 FROM Question q
                  WHERE q.concept = c AND q.status = 'APPROVED')
    """)
```
```sql
select c1_0.id, c1_0.name from concept c1_0
where exists (select 1 from question q1_0
              where q1_0.concept_id=c1_0.id and q1_0.status=?)
```

**In Criteria** (and therefore composable into a `QuerySpecification`): `query.subquery(...)`, then `cb.exists(sub)`. This is how you add a "has-an-approved-question" filter as a *reusable spec fragment* alongside your other predicates — a genuinely powerful extension of the filtering pattern you're already building.

The part that prevents bugs — **`EXISTS` vs `IN` vs a plain `JOIN`:**

- `EXISTS` is a **semi-join**: it checks *existence* and stops, returning each parent **once**. No row multiplication.
- A plain `JOIN` to a one-to-many **multiplies** the parent row per matching child, so you need `DISTINCT` to dedupe — a classic source of accidental duplicates when someone reaches for a join where they wanted existence.
- `NOT EXISTS` is an **anti-join**: parents with *no* match ("concepts with no approved questions" — exactly the gap a content workflow wants to surface). Prefer `NOT EXISTS` over `NOT IN`, because **`NOT IN` has a NULL trap**: if the subquery yields a single `NULL`, `NOT IN` returns *no rows at all*, silently. This one bites people hard.

> **The tell:** use `EXISTS`/`NOT EXISTS` when you're filtering parents by whether children exist and you **don't need child columns** in the result; use a `JOIN` (with `DISTINCT` if one-to-many) when you need the child data in the output; pull it into a separate query when the subquery is expensive and reused. On Postgres the planner treats well-written `EXISTS`/`IN` semi-joins about equally — write for *correctness and clarity* first, and check the plan (4.5) if it's hot.

## 5.5 The rest of the map — what else is worth a name

You asked what else I'd raise. Here's the curated remainder: the ones I'd actually flag for Practiq's stage, each with enough to know whether you need it. Below them, a short list of things I'm *deliberately* leaving off the map so you know the edge of it.

**Read-only transactions.** On a query-only service method, marking the transaction read-only tells Hibernate to skip dirty-checking snapshots (no entity is going to change) and can route reads to a replica. Micronaut's idiom *(verified against the Micronaut Data docs)*: use Micronaut's own `io.micronaut.transaction.annotation.Transactional(readOnly = true)` — or the dedicated `@ReadOnly` stereotype annotation, which is literally the same advice with read-only pre-set. Note the docs are explicit that the flag is a *hint* to the transaction subsystem — it won't necessarily cause write attempts to fail — so it's an optimisation and a statement of intent, not a guard rail. Nearly free and correct-by-intent: put it on every pure-read endpoint — your `GET /questions` path included.

**Batch processing and the persistence context in loops — the seed-pipeline footgun.** Dirty checking cost scales with the number of *managed* entities, so a loop that `persist()`s thousands of rows without clearing balloons the L1 cache and slows to a crawl (each flush re-checks everything). The pattern when you do bulk work through the ORM: `entityManager.flush()` then `clear()` every N rows (≈50), set `hibernate.jdbc.batch_size`, and use **SEQUENCE** ids so inserts can actually batch (2.6). But note the cleanest option is the one you already chose for seeding: **bypass the ORM entirely** with a native/CTE load — no persistence context to manage, no batching subtleties. Worth knowing the ORM path exists and why you avoided it.

**`AttributeConverter` (`@Converter`).** A bidirectional Java↔column mapper for when `@Enumerated` and built-ins don't fit — a value object stored in one column, a custom encoding. Relevant if your `difficulty` `{value, code}` ever collapses into a single stored column, or for any small value type you want persisted your way.

**Embeddables (`@Embeddable` / `@Embedded`).** Group a cluster of columns into a value object that lives *in the same table* — no join, no separate identity. Good for cohesive value clusters (an `Audit` block, a `Money` amount+currency). The distinction to hold: an `@Entity` has its own table and identity; an `@Embeddable` is just columns wearing a class.

**Inheritance mapping (`@Inheritance`).** If `Question` subtypes ever diverge *structurally* — MCQ with options, free-response with a marking rubric — JPA offers `SINGLE_TABLE` (one table, a discriminator column, subtype-specific columns nullable — fast, denormalised, the usual default), `JOINED` (a table per class joined on PK — normalised, more joins), and `TABLE_PER_CLASS` (rarely worth it). Flagging it as a fork to *recognise*, not necessarily adopt: your nullable-`difficulty`/`type` approach already handles incomplete extraction without inheritance, so you only reach here if the *shape* of questions genuinely splits.

**Auditing.** Micronaut Data's `@DateCreated` / `@DateUpdated` (or JPA `@PrePersist`/`@PreUpdate` callbacks) auto-populate created/updated timestamps. Cheap, and you'll want it on most entities sooner than you think.

**Lifecycle callbacks & query hints — brief.** `@PrePersist`, `@PostLoad` etc. let you hook entity events; query hints (`jakarta.persistence.query.timeout`, fetch size, lock timeouts) are the knobs for pathological queries. Reach for them by name when a specific need appears.

**Deliberately off the map for now** (so you know it's a choice, not an oversight): transaction **propagation** modes (`REQUIRES_NEW`, `NESTED` — matters once you nest transactional services); **reactive / R2DBC** repositories (Micronaut supports them, but you're not building a reactive app); **stored procedures** (`@NamedStoredProcedureQuery` — your logic lives in Java and Flyway, not the DB); **multi-tenancy** and **sharding** (scale concerns you're nowhere near); **custom Hibernate types/dialects** (rarely needed on stock Postgres). Any of these can graduate onto the map when a real need turns up — none should occupy you at Sprint 0.2.

---

# Part 6 — When to use what: consolidated decision frameworks

Each axis below is a decision you've probably been making implicitly. Format: **the axis → the options → the "tell" that pushes you each way → the default.** This is the section to skim mid-decision.

**A. ORM vs row mapper (per subsystem, not per app).**
Options: Micronaut Data JPA (ORM) / Micronaut Data JDBC / native SQL / jOOQ.
Tell → ORM: rich domain model, graph writes, human-edited entities, you want dirty checking and cascades. Tell → row mapper: read-heavy, reporting/analytics, SQL-shaped work, or the ORM is fighting you (endless projections and fetch tuning).
Default: **ORM for the transactional core (Practiq's current path); drop specific read-heavy paths to a row mapper when they fight you.**

**B. Which query API (per query).**
Tell → derived finder: static, ≤3 fixed conditions. Tell → JPQL: static but needs join-fetch/aggregate/too-gnarly-for-a-name. Tell → QuerySpecification: **any optional/dynamic filter** (`?conceptId=`). Tell → native: Postgres-specific feature, tuning, or bulk/seed (or push to Flyway if it's schema/seed).
Default: **finder → outgrow it → specification for dynamic, JPQL for rich-static, native as escape hatch.**

**C. Fetch strategy.**
Tell → make it LAZY and join-fetch per query: you sometimes need the association, sometimes don't. Tell → batch fetch: many small unavoidable lazy loads. Tell → projection: read-only list endpoint. Tell → (rarely) eager: an association you *always* need on *every* load of that entity.
Default: **LAZY everywhere (override the to-one eager default) + fetch explicitly per query; projections for list endpoints.**

**D. Identity generation.**
Tell → SEQUENCE: Postgres, and you care about insert throughput / batch inserts (seed pipeline). Tell → IDENTITY: you want the simplest mapping and don't care about batching.
Default (Postgres): **SEQUENCE.**

**E. Transaction boundary & isolation.**
Boundary: `@Transactional` on the **service** method owning the unit of work; keep it short; no network I/O inside; load everything you'll serialise before it returns. Isolation: **READ COMMITTED** unless you can name the concurrent anomaly you're preventing (then REPEATABLE READ / SERIALIZABLE *with retry*).
Default: **service-level boundary, READ COMMITTED.**

**F. Concurrency / lost updates.**
Tell → optimistic (`@Version`): a human reads-then-writes a row over an HTTP round-trip (Practiq review workflow). Tell → pessimistic (`SELECT … FOR UPDATE`): you must serialise contending writers at the DB and can't retry.
Default: **`@Version` optimistic locking on human-edited entities.**

**G. Entity vs projection (per read).**
Tell → entity: you'll modify it in this transaction, or you need the managed graph. Tell → projection/DTO: read-only, list/summary, or you want to dodge N+1.
Default: **entity for writes, projection for read-only lists.**

**H. Where does data logic live?**
Tell → Java + JPQL/Criteria: request-time domain queries. Tell → native SQL: Postgres-specific request-time queries. Tell → **Flyway**: schema and seed/reference data (Flyway owns schema, never content, per your D-decisions). Tell → application code + `@Scheduled`: housekeeping like anonymous-attempt cleanup now (EventBridge/Lambda later).
Default: **domain queries in Java/JPQL; DB-specific in native; schema/seed in Flyway — keep the boundary clean.**

**I. Relationship cardinality (per association).**
Tell → one-to-many (FK on the many side): the child belongs to exactly one parent. Tell → many-to-many (junction table): the child can belong to several parents — and "tagging" language is the giveaway (a Concept tagged to many SpecSections; a Question possibly tagged to several Concepts). Tell → one-to-one (unique FK / shared PK): genuinely optional or heavy data split off, otherwise merge the tables.
Default: **FK on the many side for ownership; a junction table for any tagging/reuse relationship.**

**J. Indexing (per query pattern).**
Tell → add an index: an FK you join/filter on (Postgres won't add it for you), or a `WHERE`/`ORDER BY` column on a table big enough to matter. Tell → composite index: you filter on the same *set* of columns together (order it most-selective-first, matching the query). Tell → partial index: you almost always query one subset (`status = 'APPROVED'`). Tell → covering/`INCLUDE`: a hot read that a projection could satisfy from the index alone. Tell → *don't* index: a column you never filter/sort on, or a write-hot table where the index earns nothing.
Default: **index your actual query predicates (FKs first), match composites to real queries, partial-index the hot subset — all in Flyway, validated on realistic data, not the seed set.**

**K. Caching (per entity/query).**
Tell → L1: it's always on, nothing to decide. Tell → L2 (`@Cacheable`): read-mostly reference data with a clear eviction story (Concept/SpecSection/board metadata) — and only once a profiler says a read is hot. Tell → *don't* cache: write-heavy tables, or anything where staleness is unacceptable; and the query cache is almost never worth it here.
Default: **rely on L1; leave L2 off, note the reference-data candidates, revisit only when you've measured a hotspot.**

**L. Fetch plan (per read).**
Tell → LAZY + `@Join(FETCH)`/entity graph: this endpoint needs the association; keep the query otherwise clean (works even on a derived finder). Tell → batch fetch: many small unavoidable lazy loads. Tell → projection: read-only list — dodge the associations entirely. Tell → *don't*: fetch two collections in one query (cartesian blow-up) — one collection per query.
Default: **LAZY everywhere + per-query fetch via `@Join(FETCH)`/entity graph; projections for lists; one collection per fetch.**

**M. Query shape for "parents with/without children" (per query).**
Tell → `EXISTS`/`NOT EXISTS`: you're filtering by whether children exist and don't need child columns (concepts that have / lack approved questions). Tell → `JOIN` (+ `DISTINCT` if one-to-many): you need child columns in the result. Tell → avoid `NOT IN` on a nullable subquery (silent empty-result NULL trap) — use `NOT EXISTS`.
Default: **`EXISTS`/`NOT EXISTS` for existence filters, `JOIN` when you need the child data, never `NOT IN` over nullable columns.**

---

# Part 7 — The wider landscape

Everything so far drilled *into* your stack. This part pulls back to show the territory it sits in, because "why Postgres and Hibernate?" only means something once you can see what you'd be choosing instead. It answers your three questions directly: what a database even is, what else there is besides Hibernate, and what the other database providers are.

## 7.1 What even is a database? The client–server picture

Step right back. **"Postgres" is a separate program** — a long-running *server process* that owns your data on disk and mediates every access to it. Your Java app never touches the data files; it's a **client** that opens a connection to that server (over TCP, via pgjdbc) and asks it to do things. This single fact is why all the Part 1 machinery exists: connections, sessions, authentication, a wire protocol, connection pools — all of it is the overhead of *two separate programs talking over a socket*. The database is not a library inside your app; it's a service your app calls.

The vocabulary, untangled:

- A **DBMS** (database management system) is the software that manages storage, concurrency, durability, and querying — Postgres, MySQL, Oracle are DBMSs. Colloquially "a database" means both this software *and* the data it holds.
- An **RDBMS** (relational DBMS) is the family that models data as **tables (relations)** of rows and columns, links them with keys, queries them with **SQL**, and guarantees **ACID** transactions. Postgres is one of these. Everything in Parts 1–6 assumed this model.
- The alternative to client–server is **embedded**: a database that runs *inside* your process as a library, with the data in a local file and no server, no network, no auth. **SQLite** and Java's **H2/HSQLDB** are embedded. No pool, no wire protocol — because there's no server to connect to. (This distinction matters to you concretely: it's *why* testing against embedded H2 diverges from production Postgres, and why you correctly run **real Postgres in Testcontainers** instead — 7.3.)

## 7.2 What Postgres actually is

**PostgreSQL is an open-source, object-relational DBMS** that grew out of the POSTGRES research project at Berkeley in the mid-1980s, gained SQL in the mid-90s, and has been developed by a global community ever since under a permissive BSD-style licence (no company owns it, no per-core bill). Its reputation, earned over ~three decades, is **correctness and feature-richness first** — it's the sensible default for new applications today, which is why your stack landed on it without much argument.

What "object-relational" and "extensible" actually buy you, beyond a plain SQL engine:

- **Rich built-in types** — `JSONB`, arrays, ranges, `UUID`, network types — so you can model things a stricter engine would force into awkward shapes.
- **A genuinely extensible core** — you can add types, functions, operators, and even *index access methods*. This is why Postgres has an extension ecosystem that keeps it relevant far outside plain OLTP: **PostGIS** (geospatial), **pg_trgm** (fuzzy text), **TimescaleDB** (time-series), and **pgvector** (embedding/similarity search — flagged again in 7.6 because it's directly relevant to an adaptive-learning product).
- **MVCC** (the snapshot model from 1.7) and strong SQL-standard compliance.

Physically, a running Postgres is a **cluster of processes** (a supervisor plus one backend process per connection — which is *exactly why* connection count is precious and pooling matters, 1.6), a **data directory** on disk, and a **write-ahead log (WAL)** that makes commits durable and powers replication. It listens on port **5432**. When Testcontainers "spins up Postgres," it starts precisely this in a container and hands your tests a real one.

And "Postgres" comes in **managed flavours** — Amazon RDS/Aurora, Google Cloud SQL, Supabase, Neon — which are the *same Postgres* with someone else running the server, backups, and failover. Relevant to your AWS/Terraform trajectory: you'll almost certainly run RDS/Aurora Postgres in production and keep Testcontainers Postgres in CI, and your Hibernate/pgjdbc code won't know the difference.

## 7.3 The other relational databases

These are Postgres's siblings — same relational model and SQL, different trade-offs. Because JPA/Hibernate abstracts over the dialect, switching between them is *more* tractable from Java than from raw SQL, but it's never free.

- **MySQL / MariaDB** — the other dominant open-source RDBMS, and the one behind a huge slice of the web (WordPress, classic LAMP). Historically simpler and very fast for straightforward read-heavy workloads; its **InnoDB** engine stores rows physically in primary-key order — i.e. **the clustered-index model from 4.4 that Postgres deliberately doesn't have.** *This is where that mental model you were carrying comes from.* MariaDB is the community fork created after Oracle acquired MySQL. Historically weaker on advanced SQL and strictness than Postgres, though the gap has narrowed.
- **Microsoft SQL Server** — the enterprise Windows-native option (now also on Linux), with **T-SQL**, strong tooling, and clustered indexes front-and-centre (the other place your clustered-index vocabulary comes from). Commercial/licensed.
- **Oracle Database** — the enterprise incumbent: extremely capable, extremely expensive, **PL/SQL**. Worth naming because insulating an app from *this specific* vendor lock-in is a large part of *why JPA exists at all* — write to the spec, swap the dialect.
- **SQLite** — embedded, serverless, a single file, running in-process. Among the most widely deployed databases on earth (it's in your phone and browser). Perfect for local apps, on-device storage, and quick tests; unsuitable as a concurrent multi-writer server because writes take a database-level lock. It is *not* a smaller Postgres — it's a different tool.
- **H2 / HSQLDB / Derby** — in-memory/embedded Java databases, historically popular for tests because they start instantly. The catch is **dialect drift**: H2's "Postgres compatibility mode" is approximate, so tests can pass against H2 and fail against real Postgres (or vice versa). That gap is the whole justification for your **Testcontainers-with-real-Postgres** integration tier — you test against the engine you actually ship on.
- **Distributed / "NewSQL"** — CockroachDB, YugabyteDB, Google Spanner, AWS Aurora: relational databases built to scale horizontally across nodes. The pragmatic hook for you: several (Cockroach, Yugabyte) are **Postgres-wire-compatible**, so your pgjdbc + Hibernate stack largely *just works* against them — a door left open if Practiq ever needed that scale (it won't for a long time).

## 7.4 Beyond relational: the NoSQL families

The relational model is a choice, not the only option. "NoSQL" is a grab-bag of non-relational stores, each shaped for a data pattern that tables handle awkwardly. The useful lens is *when you'd actually reach for one* — and the honest answer for most apps, including Practiq, is "as a **complement** to Postgres, not a replacement":

- **Document** (MongoDB, DynamoDB) — schema-flexible JSON-like documents. Good for heterogeneous or deeply nested data with no fixed shape; you trade away easy cross-document joins and some consistency guarantees. (Note Postgres's `JSONB` covers a lot of "I want document flexibility" *without* leaving the relational world.)
- **Key-value** (Redis, Memcached) — blindingly fast in-memory lookups. Caching, sessions, rate-limits, queues. Redis usually sits *beside* Postgres — and is a common **L2 cache provider** (5.3), which is the most likely way it'd ever enter Practiq.
- **Wide-column** (Cassandra, ScyllaDB) — enormous write throughput with tunable consistency, modelled query-first. For big-scale event/time-series data; overkill below that scale.
- **Graph** (Neo4j) — relationships as first-class citizens, for traversals like social graphs and recommendations. Worth a specific note: **graph-*shaped* data doesn't require a graph database.** Your `Concept ↔ SpecSection ↔ Question` web is graph-ish, but at your scale a relational model with junction tables (4.1) handles it cleanly with full SQL and transactions. Reach for Neo4j only when deep, variable-length traversals become the *primary* workload.
- **Search** (Elasticsearch / OpenSearch) — full-text search and analytics, typically alongside a primary DB that remains the source of truth. If Practiq's question/notes search outgrows Postgres full-text, this is the escalation.
- **Vector** (Pinecone, Weaviate — or **pgvector inside Postgres**) — similarity search over embeddings, the storage layer behind AI/semantic features. Flagged because it's plausibly *relevant* to an adaptive-learning platform (7.6), and because you'd get it as a Postgres extension rather than a new database.

Two framings to carry away. **Polyglot persistence:** real systems often run Postgres as the source of truth *plus* Redis for cache *plus* maybe a search index — different tools for different jobs, not one database to rule them all. And the underlying trade, often labelled with the **CAP theorem** (and frequently oversimplified): relational databases lean toward strong **consistency**, while many NoSQL stores trade consistency for **availability and horizontal scale** under network partitions. The practical caution: don't reach for NoSQL reflexively "for scale." Postgres scales much further than most teams ever need, and leaving it costs you transactions, joins, and constraints — the very things Parts 1–6 are built on.

## 7.5 The Java persistence landscape beyond Hibernate

Your literal question — "what else is there other than Hibernate?" — splits into two very different answers.

**At the JPA-provider level, the list is short.** JPA is a spec (2.1); Hibernate is one implementation of it. The only other live one you'd meet is **EclipseLink** (the spec's reference implementation, still maintained, rarely chosen for new work); **OpenJPA** and **DataNucleus** exist but are niche/legacy. So "another JPA provider instead of Hibernate" is a real but small choice — for practical purposes, *JPA provider ≈ Hibernate* today.

**The real alternatives live one level up — at the paradigm level** (this is Part 1.8's Path A/B, now with names). Instead of "which JPA provider," the live question is "do I want the full ORM at all, here?":

- **Lightweight mappers** — Micronaut Data JDBC, Spring Data JDBC: repositories and mapping, but *no* persistence context, dirty checking, or lazy loading. Path A with conveniences.
- **jOOQ** — a type-safe SQL DSL: you write SQL in Java, checked against your real schema at compile time. The pick when SQL *is* your model and you want the compiler policing it.
- **MyBatis** — SQL in annotations/XML mapped to objects; favoured where SQL is owned deliberately.
- **JDBI**, **Spring `JdbcTemplate`/`JdbcClient`** — thin, explicit wrappers over JDBC.
- **Kotlin-native** — Exposed, Ktorm (not your language, but you'll see them).
- **Reactive** — a whole separate axis: **R2DBC**, the Vert.x SQL client, and **Hibernate Reactive** replace blocking JDBC with non-blocking drivers for reactive apps. Micronaut supports this; you're not building a reactive app, so it stays off your map — but it's the answer to "what if my whole app is non-blocking?"

The shape to remember: from **raw JDBC** at one end to **full ORM (Hibernate)** at the other is a *spectrum*, and — as in 1.8 — you can sit at different points **per subsystem**. Practiq's transactional core is rightly at the ORM end; a future reporting or export path could sit at the jOOQ/native end without contradiction.

## 7.6 Where Practiq's choices land — and what would change your mind

Pulling it together honestly: **Postgres + Hibernate/JPA via Micronaut Data JPA is a mainstream, well-judged default** for what Practiq is — a domain-model-heavy app with a rich `Concept`/`Question` graph, a human-review workflow, and cross-board reuse. Nothing here is a choice you'll have to apologise for. What would *legitimately* move you off each, so you'd recognise the moment:

- **Off Postgres** — only for a genuinely different shape: embedded/on-device (SQLite), horizontal scale you're nowhere near (distributed SQL, ideally Postgres-compatible so the stack survives), or data that truly isn't relational. None of these is on your horizon.
- **Off the ORM, per subsystem** — when a read-heavy reporting/analytics/export path starts fighting the persistence context and N+1 tuning, drop *that path* to a row mapper, jOOQ, or native SQL (Part 6-A). This is a *local* move, not a re-platform.
- **Add something alongside** — Redis as an L2/cache/session layer if a read hotspot is measured (5.3); a search index if full-text search outgrows Postgres; and the one genuinely forward-looking note for you: **pgvector**. An adaptive-learning platform that ranks or recommends questions by similarity is a natural fit for embedding search — and you'd get it as a **Postgres extension**, inside the database you already run, rather than bolting on a separate vector store. Worth filing away as "Postgres probably already does that" for whenever the adaptive engine gets serious.

The meta-point, which is the whole reason this part exists: your stack is a *position* in a landscape, chosen against real alternatives — not the only way, and not an accident. That's exactly the footing you said you wanted.

---

# Part 8 — A worked trace: one request, every layer

Everything above is the theory; this is the theory *happening*. One request — `GET /api/v1/questions?conceptId=42` — followed from the socket to the rows and back, with the actual log lines you'd see and, at each step, which section of this document is executing. Read it once now, then again with your own logs open (3.1 enables them) — the shape will match.

The cast, from your actual Sprint 0.2 design: a `QuestionController` whose public endpoint signature hard-codes `APPROVED` (no status param exists to abuse), a `QuestionService`, spec factories, and a `QuestionRepository extends JpaSpecificationExecutor<Question>`.

**Step 0 — before the request: startup.** At *compile time*, Micronaut's annotation processor already generated the implementation of `QuestionRepository` (2.12) — that work is done, baked into the jar. At *boot*, Hibernate built its metamodel from your `@Entity` classes and validated any JPQL `@Query` strings against it (3.3), and Hikari opened its pool of Postgres connections (1.6) — each one a live authenticated session (1.5). Nothing about *this* request has happened yet, but every layer is already standing.

**Step 1 — HTTP in.** Micronaut routes the request, binds `conceptId=42` to a `Long`, and calls the controller. Because the endpoint signature is `list(@QueryValue @Nullable Long conceptId)` with no status parameter, `APPROVED` isn't a decision being made per-request — it's structurally unexpressible to ask for anything else. (This is a design-level version of the same idea as parameterised SQL in 1.3: make the wrong thing impossible to *say*.)

**Step 2 — the service opens the unit of work.** The controller calls `questionService.findQuestions(42)`, annotated `@ReadOnly` (5.5 — it's a pure read). Micronaut's transaction interceptor borrows a connection from Hikari, switches off autocommit, and begins a transaction (1.7). Hibernate opens a session — the **persistence context** now exists, glued to this transaction (2.4). On the wire, roughly:

```
BEGIN
```

**Step 3 — the spec composes.** Inside the service (3.5):

```java
var spec = QuerySpecification.where(isApproved());
if (conceptId != null) spec = spec.and(hasConcept(conceptId));   // 42 → yes
return questionRepository.findAll(spec);
```

No SQL yet. A `QuerySpecification` is a *description* — a lambda that knows how to produce a `Predicate` when handed Criteria objects. Composition is just building a bigger description.

**Step 4 — the repository call descends the stack.** `findAll(spec)` enters the Micronaut-generated implementation, which drives the JPA runtime API (2.1): it creates a `CriteriaQuery<Question>`, a `Root<Question>`, calls **your** `toPredicate(root, query, cb)` — the exact method your unit tests exercise — and hands the assembled Criteria query to Hibernate. Hibernate translates it through `PostgreSQLDialect` (2.11) into SQL, wraps it in a JDBC `PreparedStatement` (1.3), and pgjdbc sends it down the socket (1.4). Your log (3.1) shows:

```
DEBUG org.hibernate.SQL:
    select q1_0.id, q1_0.concept_id, q1_0.difficulty, q1_0.status, q1_0.type
    from question q1_0
    where q1_0.status=? and q1_0.concept_id=?
TRACE org.hibernate.orm.jdbc.bind: binding parameter (1:VARCHAR) <- [APPROVED]
TRACE org.hibernate.orm.jdbc.bind: binding parameter (2:BIGINT)  <- [42]
```

Two things to notice, both of which you designed: the `where` clause has exactly two predicates *because* the spec composed two (drop the query param and re-run — you'll see the `concept_id` predicate vanish, which is 3.5's whole point made visible); and the enum bound as the string `APPROVED` is your `@Enumerated(EnumType.STRING)` uppercase decision on the wire.

**Step 5 — Postgres plans and executes.** The planner (4.5) takes over: with the partial index from 4.3 in place (`ON question (concept_id) WHERE status = 'APPROVED'`), `EXPLAIN ANALYZE` on this statement shows an **Index Scan** on realistic data volume — and, on your 40-row seed set, a **Seq Scan instead, which is fine and means nothing** (the 4.5 trap: never judge indexes at seed scale). MVCC (1.7) means this read takes no locks and blocks no writers.

**Step 6 — hydration.** Rows come back through the `ResultSet`; Hibernate hydrates each into a `Question` entity and registers it in the persistence context (2.2) — each is now **managed** (2.3), with a snapshot taken. If your serialisation path touched `question.getConcept()` here, *this* is the moment N+1 would detonate — one extra `SELECT` per row (2.8), visible in the log as a stutter of identical statements. The fixes are `@Join(FETCH)` (5.2) or, better for this list endpoint, a projection (3.8) that never materialises the association at all.

**Step 7 — the unit of work closes.** The service method returns. Nothing was modified, so dirty checking (2.5) compares snapshots, finds nothing, and flushes no SQL — the read-only hint (5.5) let Hibernate skip even keeping detailed snapshots. On the wire:

```
COMMIT
```

The persistence context closes; every `Question` is now **detached** (2.3). The connection goes back to Hikari's pool, clean (1.5–1.6). Total connection hold time: milliseconds — which is why this endpoint can serve many concurrent users from a pool of ten connections.

**Step 8 — HTTP out.** The controller maps entities to the response shape (difficulty serialising as `{"value": 3, "code": "MEDIUM"}` per your design) and Micronaut writes the JSON. Detached entities are fine to read from — the fields are just Java state now — as long as nothing lazy is touched (2.4).

**The same trace, as a debugging template.** When any endpoint misbehaves, walk these steps in order and ask where reality diverged: Is a connection even acquired (pool exhausted? — step 2)? Does the logged SQL have the predicates you expect (spec composition — steps 3–4)? Are the bound parameters right (step 4's TRACE lines)? Is the plan sane at realistic volume (step 5)? Is there a stutter of extra SELECTs (step 6)? Did an unexpected flush emit writes (step 7)? Eight questions, each with a section behind it — that's this whole document folded into one diagnostic sweep.

---

# How to use and expand this document

- Suggested home: `docs/` in `practiq-api`, or an appendix off `PRACTIQ_MASTER.md`. It's deliberately Practiq-flavoured so it doubles as "why is the data layer shaped like this" onboarding for a reviewer or future-you.
- Good next deep-dives, any of which I can expand into a standalone worked section with full SQL traces: **the persistence context + flush modes end to end** (with a step-by-step SQL log); **a full N+1 walkthrough** comparing all four fixes' SQL side by side; **wiring the static metamodel generator** into your Gradle build (the exact `annotationProcessor` setup, then converting the `QuerySpecification` factories to `Question_`); **`@Transactional` propagation and boundaries** in Micronaut specifically; **keyset pagination** as a concrete recipe; **a native-SQL/Flyway pattern** for the seed CTE; **optimistic-locking end to end** wired into the review workflow, including the HTTP-level conflict response; **the Practiq schema drawn out** as a concrete ERD + Flyway migrations (tables, FKs, junction tables, and the exact indexes for your query patterns) with `EXPLAIN ANALYZE` before/after on realistic data.
- Ask for any of the above and I'll write it against your actual entities.

*Caveat: written from knowledge of stable Jakarta Persistence 3.x / Hibernate 6 / Micronaut Data behaviour, not transcribed from the 600-page spec. The concepts are stable across 3.x; if you want a specific 3.2-version claim or an exact Micronaut Data API signature verified against the current docs, point me at it and I'll check it properly.*
