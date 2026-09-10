# JPA & Hibernate — Annotation and Class Reference

*A compact lookup reference for the surface a **consumer** of JPA/Hibernate actually uses — not the SPI a library author would implement. Jakarta Persistence 3.x (`jakarta.persistence.*`), Hibernate 6, Postgres. Companion to the Java data-access primer, which explains the **why**; this is the **what**.*

Two conventions throughout: **[JPA]** = standard, portable, `jakarta.persistence.*`. **[HIB]** = Hibernate-specific, `org.hibernate.annotations.*` — works, but ties you to Hibernate. Prefer JPA unless the Hibernate one earns it.

---

## 1. Entity & table mapping

| Annotation | Does | Notes |
|---|---|---|
| **`@Entity`** [JPA] | marks a class as a mapped entity | needs a no-arg constructor, non-final, an `@Id` |
| **`@Table(name, schema, indexes, uniqueConstraints)`** [JPA] | maps to a specific table | omit and it defaults to the class name |
| **`@Column(name, nullable, length, unique, insertable, updatable, precision, scale)`** [JPA] | maps a field to a column | omit and it defaults to the field name |
| **`@Transient`** [JPA] | **do not persist this field** | not `transient` the Java keyword (though that works too) |
| **`@Basic(fetch, optional)`** [JPA] | default mapping for simple types | rarely written explicitly |
| **`@Lob`** [JPA] | large object (`TEXT`/`BYTEA`) | for big strings/binaries |
| **`@Enumerated(EnumType.STRING)`** [JPA] | how an enum is stored | **always `STRING`**; `ORDINAL` stores an int and breaks the moment someone reorders the enum |
| **`@Temporal`** [JPA] | legacy `Date`/`Calendar` precision | **unnecessary** with `java.time` — use `LocalDate`/`Instant` and skip it |
| **`@Embeddable` / `@Embedded`** [JPA] | a value object living **in the same table** | no identity, no join — just grouped columns |
| **`@AttributeOverride(name, column)`** [JPA] | rename an embeddable's columns at the use site | needed when embedding the same type twice |
| **`@Convert(converter = X.class)`** [JPA] | apply an `AttributeConverter` | custom Java↔column mapping |
| **`@Formula("sql")`** [HIB] | a read-only field computed by **raw SQL** | handy, but it's SQL in your entity |
| **`@Where(clause)` / `@Filter`** [HIB] | always-applied SQL restriction (e.g. soft delete) | powerful; easy to surprise yourself with |
| **`@NaturalId`** [HIB] | marks the business key | enables `Session.byNaturalId(...)` lookups |
| **`@DynamicUpdate` / `@DynamicInsert`** [HIB] | only include **changed** columns in the SQL | helps with wide tables / `@Version` contention; costs SQL-statement caching |

**A worked entity — most of the above in one place:**

```java
@Entity
@Table(name = "question", indexes = @Index(columnList = "concept_id, status"))
public class Question {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "question_seq")
    @SequenceGenerator(name = "question_seq", sequenceName = "question_id_seq",
                       allocationSize = 50)
    private Long id;

    @Version                                    // optimistic locking
    private long version;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)   // override EAGER default
    @JoinColumn(name = "concept_id", nullable = false)
    private Concept concept;

    @Enumerated(EnumType.STRING)                // never ORDINAL
    @Column(nullable = false, length = 20)
    private Status status;

    @Column(columnDefinition = "text")
    private String body;

    @Embedded                                   // columns live in this table
    private Difficulty difficulty;

    @Transient                                  // never persisted
    private boolean dirtyFlag;

    @CreationTimestamp private Instant createdAt;
    @UpdateTimestamp   private Instant updatedAt;

    protected Question() {}                     // JPA needs a no-arg constructor
}
```

---

## 2. Identity & keys

| Annotation | Does |
|---|---|
| **`@Id`** [JPA] | the primary key field |
| **`@GeneratedValue(strategy, generator)`** [JPA] | how the key is generated — `SEQUENCE`, `IDENTITY`, `TABLE`, `AUTO` |
| **`@SequenceGenerator(name, sequenceName, allocationSize)`** [JPA] | configures a sequence; `allocationSize` (default 50) is the pre-allocation block |
| **`@EmbeddedId`** / **`@IdClass`** [JPA] | composite primary keys — two ways to do the same job |
| **`@MapsId`** [JPA] | share the parent's PK in a one-to-one/many-to-one |
| **`@Version`** [JPA] | **optimistic locking** — adds `where version=?` to updates; throws `OptimisticLockException` on conflict |

> **The tell:** on Postgres use **`SEQUENCE`** — `IDENTITY` silently disables JDBC insert batching (the id must come back from each insert). Surrogate `@Id` + a `@NaturalId`/unique constraint on the business key beats a natural PK. `@Version` on anything a human edits over an HTTP round-trip.

---

## 3. Relationships

| Annotation | Does | Default fetch |
|---|---|---|
| **`@ManyToOne`** [JPA] | the FK side — the workhorse | **EAGER** ← override to LAZY |
| **`@OneToMany(mappedBy = "...")`** [JPA] | the inverse collection side | LAZY |
| **`@OneToOne`** [JPA] | one-to-one | **EAGER** ← override to LAZY |
| **`@ManyToMany`** [JPA] | needs a junction table | LAZY |
| **`@JoinColumn(name, referencedColumnName, nullable)`** [JPA] | names the FK column | — |
| **`@JoinTable(name, joinColumns, inverseJoinColumns)`** [JPA] | names the junction table for `@ManyToMany` | — |
| **`@OrderBy("field ASC")`** [JPA] | sort a collection **in SQL** on load | — |
| **`@OrderColumn`** [JPA] | persist list *position* in its own column | usually more trouble than it's worth |
| **`@MapKey` / `@MapKeyColumn`** [JPA] | map a collection as a `Map` | — |
| **`@ElementCollection`** [JPA] | a collection of **basic/embeddable** values (not entities) in a side table | LAZY |
| **`@BatchSize(size = 25)`** [HIB] | load lazy associations in `in (?,?,…)` batches | turns 1+N into 1+(N/size) |

**Key attributes on the relationship annotations:**

| Attribute | Effect |
|---|---|
| `fetch = FetchType.LAZY / EAGER` | when the association loads |
| `cascade = {PERSIST, MERGE, REMOVE, REFRESH, DETACH, ALL}` | which operations propagate to children |
| `orphanRemoval = true` | removing a child from the collection **deletes** it |
| `mappedBy = "field"` | marks the **inverse** (non-owning) side |
| `optional = false` | the association is required (→ inner join, NOT NULL) |

**Bidirectional one-to-many — and the trap:**

```java
@Entity
public class Concept {
    @Id @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;

    // INVERSE side — mappedBy points at the field in Question that owns the FK
    @OneToMany(mappedBy = "concept", cascade = CascadeType.ALL, orphanRemoval = true)
    private Set<Question> questions = new HashSet<>();

    // ALWAYS sync both sides yourself — JPA does not do it for you
    public void addQuestion(Question q) {
        questions.add(q);
        q.setConcept(this);          // ← without this, the FK is never set
    }
    public void removeQuestion(Question q) {
        questions.remove(q);
        q.setConcept(null);
    }
}

@Entity
public class Question {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "concept_id")   // OWNING side — holds the FK column
    private Concept concept;
}
```

**The trap:** only the **owning** side (the one with `@JoinColumn`) is written to the database. Add a `Question` to `concept.questions` without setting `q.setConcept(this)` and the collection looks right in memory, but **no `UPDATE` sets the FK** — the change silently vanishes on reload. Hence the `addX`/`removeX` helper pattern above; write it once per relationship and always use it.

**Many-to-many via a junction table** (the Practiq Concept↔SpecSection shape):

```java
@ManyToMany
@JoinTable(name = "concept_spec_section",
    joinColumns        = @JoinColumn(name = "concept_id"),
    inverseJoinColumns = @JoinColumn(name = "spec_section_id"))
private Set<SpecSection> specSections = new HashSet<>();
```

> **The tells:** make **everything LAZY** (override the to-one EAGER default) and fetch explicitly per query. `mappedBy` names the **owning** side's field — the owner is whoever holds the FK; forget this and you get a phantom extra join table or an update that doesn't stick. Cascade only for **composition** (children the parent owns), never across shared references. `Set` vs `List` on a collection genuinely changes behaviour (two fetched `List`s → `MultipleBagFetchException`). **Use `Set` for `@ManyToMany`** — a `List` makes Hibernate delete and re-insert the whole junction row set on any change.

---

## 4. Inheritance

| Annotation | Does |
|---|---|
| **`@Inheritance(strategy = ...)`** [JPA] | `SINGLE_TABLE` (default — one table + discriminator; fast, nullable columns), `JOINED` (table per class, joined on PK; normalised), `TABLE_PER_CLASS` (rarely worth it) |
| **`@DiscriminatorColumn(name, discriminatorType)`** [JPA] | the subtype marker column (SINGLE_TABLE) |
| **`@DiscriminatorValue("MCQ")`** [JPA] | this subclass's marker value |
| **`@MappedSuperclass`** [JPA] | share mapped fields via a **non-entity** parent (no table of its own) — the common one for a shared `id`/audit base |

---

## 5. Lifecycle callbacks & auditing

| Annotation | Fires |
|---|---|
| **`@PrePersist` / `@PostPersist`** [JPA] | before/after INSERT |
| **`@PreUpdate` / `@PostUpdate`** [JPA] | before/after UPDATE |
| **`@PreRemove` / `@PostRemove`** [JPA] | before/after DELETE |
| **`@PostLoad`** [JPA] | after an entity is loaded |
| **`@EntityListeners(X.class)`** [JPA] | put those callbacks in a separate class |
| **`@CreationTimestamp` / `@UpdateTimestamp`** [HIB] | auto-populate created/updated |
| *(Micronaut Data: `@DateCreated` / `@DateUpdated` — the framework's equivalent, prefer these in a Micronaut app)* | |

---

## 6. Queries & fetch plans

| Annotation | Does |
|---|---|
| **`@NamedQuery(name, query)`** [JPA] | a named JPQL query, validated at **startup** |
| **`@NamedNativeQuery`** [JPA] | a named native SQL query |
| **`@NamedEntityGraph` / `@NamedAttributeNode`** [JPA] | a reusable **fetch plan** — declares which associations to fetch eagerly for a given query |
| **`@Query("JPQL")`** *(framework)* | Spring Data / Micronaut Data — not JPA itself; `nativeQuery = true` for raw SQL |
| **`@Join(value, type)`** *(Micronaut)* | fetch-join on a repository method; `Type.FETCH` is the default |

**Entity-graph hints** (passed as query hints): `jakarta.persistence.fetchgraph` = fetch *exactly* the listed attributes eagerly; `jakarta.persistence.loadgraph` = fetch those *in addition to* the mapping defaults.

---

## 7. Caching

| Annotation | Does |
|---|---|
| **`@Cacheable`** [JPA] | this entity is eligible for the **second-level cache** |
| **`@Cache(usage = ...)`** [HIB] | the concurrency strategy: `READ_ONLY`, `NONSTRICT_READ_WRITE`, `READ_WRITE`, `TRANSACTIONAL` |

> **The tell:** L2 is for **read-mostly reference data with a clear eviction story**, and only after you've measured a hotspot. Bulk JPQL updates bypass it and leave it stale.

---

## 8. The runtime classes you actually touch

### Core [JPA]

| Type | Is |
|---|---|
| **`EntityManager`** | the main runtime API — the persistence context handle |
| `EntityManagerFactory` | creates them; app-scoped (usually framework-managed) |
| `EntityTransaction` | `begin`/`commit`/`rollback` — usually replaced by `@Transactional` |
| `PersistenceContext` / `PersistenceUnit` | injection annotations for the above |

**`EntityManager` — the methods that matter:**

| Method | Does |
|---|---|
| `persist(e)` | transient → managed; queues an INSERT |
| `find(Class, id)` | load by PK (checks L1 cache first) |
| `getReference(Class, id)` | a **lazy proxy** — no SELECT until touched; ideal for setting an FK without loading the row |
| `merge(e)` | copy a **detached** entity's state onto a managed one; **returns** the managed instance (the argument stays detached — the classic trap) |
| `remove(e)` | queues a DELETE |
| `refresh(e)` | re-read from the DB, discarding in-memory changes |
| `detach(e)` / `clear()` | evict one / all from the persistence context |
| `flush()` | push pending SQL **now** (doesn't commit) |
| `contains(e)` | is this entity managed? |
| `lock(e, LockModeType)` | apply a lock |
| `createQuery` / `createNamedQuery` / `createNativeQuery` | build queries |
| `getCriteriaBuilder()` | entry point to the Criteria API |
| `getMetamodel()` | the **dynamic** metamodel (runtime reflection over mappings) |

### Queries [JPA]

`Query` (untyped) · **`TypedQuery<T>`** (prefer it) · `StoredProcedureQuery` · `Tuple` (multi-select results).
Key methods: `setParameter`, `getResultList`, **`getSingleResultOrNull()`** (Jakarta 3.2 — safer than `getSingleResult()`, which throws `NoResultException`), `setFirstResult`/`setMaxResults` (→ SQL `offset`/`limit`), `setHint`, `setLockMode`, `executeUpdate` (bulk).

### Criteria API [JPA]

`CriteriaBuilder` (the factory — `equal`, `and`, `or`, `like`, `gt`, `in`, `isNull`, `exists`, `count`) · `CriteriaQuery<T>` (`select`, `where`, `orderBy`, `groupBy`, `subquery`) · `Root<T>` (the FROM entity — `get`, `join`, `fetch`) · `Predicate` · `Join<X,Y>` · `Subquery<T>` · `Expression<T>`. Plus `CriteriaUpdate`/`CriteriaDelete` for bulk.

**Static metamodel:** generated `Question_` classes (via `hibernate-jpamodelgen` as an `annotationProcessor`) let you write `root.get(Question_.conceptId)` instead of `root.get("conceptId")` — turning a runtime failure into a compile error. Worth it if you hand-write Criteria/Specifications.

### Enums [JPA]

| Enum | Values |
|---|---|
| `FetchType` | `LAZY`, `EAGER` |
| `CascadeType` | `PERSIST`, `MERGE`, `REMOVE`, `REFRESH`, `DETACH`, `ALL` |
| `EnumType` | `STRING`, `ORDINAL` |
| `LockModeType` | `OPTIMISTIC`, `OPTIMISTIC_FORCE_INCREMENT`, `PESSIMISTIC_READ`, `PESSIMISTIC_WRITE`, `PESSIMISTIC_FORCE_INCREMENT`, `NONE` |
| `InheritanceType` | `SINGLE_TABLE`, `JOINED`, `TABLE_PER_CLASS` |
| `GenerationType` | `SEQUENCE`, `IDENTITY`, `TABLE`, `AUTO`, `UUID` |
| `FlushModeType` | `AUTO`, `COMMIT` |

### Exceptions [JPA]

`PersistenceException` (root) · `EntityNotFoundException` · `EntityExistsException` · `NoResultException` / `NonUniqueResultException` (from `getSingleResult`) · **`OptimisticLockException`** (`@Version` conflict) · `PessimisticLockException` · `RollbackException` · `TransactionRequiredException` · **`LazyInitializationException`** [HIB] (touched a lazy association after the session closed).

### Hibernate-specific [HIB]

`Session` (extends `EntityManager`; `unwrap(Session.class)` to reach it) · `SessionFactory` · `StatelessSession` (no persistence context — for bulk) · `Session.byNaturalId(...)` · `Hibernate.initialize(proxy)` / `Hibernate.isInitialized(proxy)`.

### Transactions

`jakarta.transaction.Transactional` — standard, but **no `isolation`/`readOnly` attributes**.
`io.micronaut.transaction.annotation.Transactional` — exposes `isolation`, `propagation`, `readOnly`; plus **`@ReadOnly`** as a shorthand stereotype. (Micronaut docs note `readOnly` is a *hint* — it won't necessarily make writes fail.)
`org.springframework.transaction.annotation.Transactional` — Spring's equivalent.

---

## 9. The runtime patterns, in code

**Dirty checking — no `save()` call:**

```java
@Transactional
public void approve(Long id) {
    Question q = em.find(Question.class, id);   // SELECT → managed
    q.setStatus(Status.APPROVED);               // just a setter
}   // at COMMIT: dirty check → UPDATE question set status=?, version=? where id=? and version=?
```

**`getReference` — set an FK without loading the row:**

```java
Question q = new Question();
q.setConcept(em.getReference(Concept.class, conceptId));  // proxy — NO select
em.persist(q);                                             // INSERT uses the id only
```

**`merge` — the return value is the managed one:**

```java
Question detached = /* came back from a web layer */;
Question managed = em.merge(detached);   // ← use the RETURN value
managed.setStatus(APPROVED);             // tracked
// detached.setStatus(...) would do nothing — still detached
```

**Criteria + a composable specification:**

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Question> cq = cb.createQuery(Question.class);
Root<Question> root = cq.from(Question.class);

List<Predicate> predicates = new ArrayList<>();
predicates.add(cb.equal(root.get(Question_.status), Status.APPROVED));   // metamodel
if (conceptId != null)
    predicates.add(cb.equal(root.get(Question_.concept).get(Concept_.id), conceptId));

cq.where(cb.and(predicates.toArray(new Predicate[0])));
List<Question> results = em.createQuery(cq).getResultList();
// → where q1_0.status=? [and q1_0.concept_id=?]  — grows with the runtime conditions
```

**A subquery — parents filtered by children (`EXISTS`, not a join):**

```java
Subquery<Long> sub = cq.subquery(Long.class);
Root<Question> q = sub.from(Question.class);
sub.select(cb.literal(1L))
   .where(cb.equal(q.get(Question_.concept), conceptRoot),
          cb.equal(q.get(Question_.status), Status.APPROVED));
cq.where(cb.exists(sub));    // concepts that HAVE an approved question — each returned once
```

**JPQL with a fetch join — the N+1 killer:**

```java
List<Question> qs = em.createQuery("""
        select q from Question q
        join fetch q.concept
        where q.status = :status
        """, Question.class)
    .setParameter("status", Status.APPROVED)
    .setMaxResults(50)                     // → limit ?
    .getResultList();
```

**Bulk update — fast, but bypasses the persistence context:**

```java
@Transactional
public int approveAll(Long conceptId) {
    int updated = em.createQuery("""
            update Question q set q.status = :approved
            where q.conceptId = :cid and q.status = :draft
            """)
        .setParameter("approved", APPROVED)
        .setParameter("cid", conceptId)
        .setParameter("draft", DRAFT)
        .executeUpdate();          // ONE SQL UPDATE over many rows
    em.clear();                     // loaded entities are now stale — evict them
    return updated;
}
```

**Optimistic locking, caught:**

```java
try {
    questionService.approve(id);
} catch (OptimisticLockException e) {
    // someone else changed the row first → 409 Conflict, tell them to reload
}
```

---

## 10. The traps, collected

- **`@Enumerated`** defaults to `ORDINAL` → always specify `STRING`.
- **`@ManyToOne`/`@OneToOne`** default to `EAGER` → override to `LAZY`.
- **`merge()`** returns the managed instance; the argument stays detached. Use the return value.
- **`getSingleResult()`** throws on no rows → prefer `getSingleResultOrNull()` (Jakarta 3.2).
- **`GenerationType.IDENTITY`** kills insert batching on Postgres → use `SEQUENCE`.
- **`mappedBy`** goes on the **inverse** side; the FK holder is the owner.
- **`cascade = REMOVE`** across a shared reference deletes things you didn't mean to.
- **Two fetched collections** in one query → `MultipleBagFetchException` / cartesian product.
- **Bulk `executeUpdate`** bypasses the persistence context and L2 cache → entities go stale; `clear()` after.
- **Entity `equals`/`hashCode`** on a generated `@Id` breaks in hash collections before persist.
- **`@Transient`** (JPA) ≠ `transient` (serialization) — though both exclude the field.

---

---

## 11. A worked entity model — Practiq, end to end

Everything above, in one coherent domain, with the design reasoning made explicit. This is a **plausible Practiq model** — treat it as a worked example to argue with, not a spec to adopt. Every annotation from §1–8 appears at least once.

## 11.1 The domain, and the shape it forces

Six entities. The requirements that drive the design:

1. **A `Concept` is board-agnostic** ("Newton's Second Law" is the same physics for AQA and OCR) and is **tagged** to board-specific `SpecSection`s. → Concept↔SpecSection is **many-to-many**, so a junction table. This is the whole point of the model: one canonical concept, reused across boards, never duplicated.
2. **A `Question` is tagged to one or more `Concept`s** → also many-to-many. (If a question only ever had one concept, this would be a plain FK — the decision is §11.6.)
3. **A `SpecSection` belongs to exactly one `ExamBoard`** → one-to-many, FK on `SpecSection`.
4. **A `Question` has body content that's large and not always needed** → a lazy one-to-one, split off.
5. **Questions are human-reviewed** → `@Version` optimistic locking, audit fields, a status enum.
6. **Attempts are recorded per question** → one-to-many, and high-volume.

## 11.2 The shared base — `@MappedSuperclass`

```java
@MappedSuperclass                                  // NOT an entity: no table of its own
public abstract class BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "base_seq")
    private Long id;

    @Version                                       // optimistic locking for everything
    private long version;

    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    @Column(nullable = false)
    private Instant updatedAt;

    @PrePersist
    void onCreate() { createdAt = updatedAt = Instant.now(); }

    @PreUpdate
    void onUpdate() { updatedAt = Instant.now(); }

    public Long getId() { return id; }

    // equals/hashCode by ID, null-safe for transient entities — see §11.7
    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof BaseEntity that)) return false;
        if (!getClass().equals(Hibernate.getClass(o))) return false;   // proxy-safe
        return id != null && id.equals(that.getId());
    }
    @Override public int hashCode() { return getClass().hashCode(); }  // constant — see §11.7
}
```

**Why `@MappedSuperclass` and not `@Inheritance`:** these entities aren't *subtypes* of a common thing — they just share columns. `@MappedSuperclass` gives shared mapped fields with **no table, no discriminator, no polymorphic queries**. `@Inheritance` would model "a Question IS-A BaseEntity" as a queryable hierarchy, which is nonsense here.

**Why `@PrePersist` and not `@CreationTimestamp`:** the lifecycle callbacks are **[JPA]** standard; `@CreationTimestamp` is **[HIB]**. Both work. In a Micronaut app, `@DateCreated`/`@DateUpdated` is the third option. Pick one and be consistent — I've used the portable one here.

## 11.3 The entities

**`ExamBoard`** — a small, read-mostly reference table. The obvious L2 cache candidate:

```java
@Entity
@Table(name = "exam_board")
@Cacheable                                      // [JPA] eligible for L2
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)   // [HIB] never changes after insert
public class ExamBoard extends BaseEntity {

    @NaturalId                                  // [HIB] the business key
    @Column(nullable = false, unique = true, length = 20)
    private String code;                        // "AQA", "OCR", "EDEXCEL"

    @Column(nullable = false)
    private String name;

    @OneToMany(mappedBy = "examBoard", cascade = CascadeType.ALL, orphanRemoval = true)
    private Set<SpecSection> specSections = new HashSet<>();
}
```

Surrogate `@Id` **plus** a `@NaturalId`/unique on `code`: identity stability *and* the business-uniqueness guarantee. `cascade = ALL` + `orphanRemoval` is right here — a `SpecSection` is **owned** by its board and is meaningless without it (composition).

**`SpecSection`** — board-specific syllabus reference:

```java
@Entity
@Table(name = "spec_section",
       uniqueConstraints = @UniqueConstraint(columnNames = {"exam_board_id", "reference"}),
       indexes = @Index(name = "idx_spec_board", columnList = "exam_board_id"))
public class SpecSection extends BaseEntity {

    @ManyToOne(fetch = FetchType.LAZY, optional = false)   // override the EAGER default
    @JoinColumn(name = "exam_board_id", nullable = false)
    private ExamBoard examBoard;                            // OWNING side — holds the FK

    @Column(nullable = false, length = 20)
    private String reference;                               // "4.2.1"

    @Column(nullable = false)
    private String title;

    @ManyToMany(mappedBy = "specSections")                  // INVERSE — Concept owns the join
    private Set<Concept> concepts = new HashSet<>();
}
```

The composite unique on `(exam_board_id, reference)` says "4.2.1 is unique *within* AQA, but OCR can have its own 4.2.1" — a business rule the database now enforces rather than the application hoping.

**`Concept`** — the board-agnostic heart of the model:

```java
@Entity
@Table(name = "concept")
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.NONSTRICT_READ_WRITE)   // read-mostly, rare edits
public class Concept extends BaseEntity {

    @NaturalId
    @Column(nullable = false, unique = true, length = 100)
    private String slug;                        // "newtons-second-law"

    @Column(nullable = false)
    private String name;

    @Lob                                        // large text
    @Column(columnDefinition = "text")
    private String description;

    // MANY-TO-MANY → junction table. Concept is the OWNING side (it declares @JoinTable).
    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(name = "concept_spec_section",
        joinColumns        = @JoinColumn(name = "concept_id"),
        inverseJoinColumns = @JoinColumn(name = "spec_section_id"))
    @BatchSize(size = 25)                       // [HIB] 1+N → 1+(N/25) when loading many
    private Set<SpecSection> specSections = new HashSet<>();

    @ManyToMany(mappedBy = "concepts")          // INVERSE — Question owns this join
    private Set<Question> questions = new HashSet<>();

    // ── sync helpers: JPA will NOT do this for you ──
    public void tagTo(SpecSection s) { specSections.add(s); s.getConcepts().add(this); }
    public void untagFrom(SpecSection s) { specSections.remove(s); s.getConcepts().remove(this); }
}
```

**`Question`** — the entity with the review workflow:

```java
@Entity
@Table(name = "question",
       indexes = {
           @Index(name = "idx_q_status", columnList = "status"),
           @Index(name = "idx_q_source", columnList = "source_ref")
       })
@DynamicUpdate                                  // [HIB] only changed columns in the UPDATE
public class Question extends BaseEntity {

    @Enumerated(EnumType.STRING)                // NEVER ORDINAL
    @Column(nullable = false, length = 20)
    private Status status = Status.DRAFT;

    @Enumerated(EnumType.STRING)
    @Column(length = 20)
    private QuestionType type;                  // nullable — extraction may not know yet

    @Embedded                                   // value object → columns in THIS table
    private Difficulty difficulty;

    @Column(name = "source_ref", length = 100)
    private String sourceRef;

    @Convert(converter = TagListConverter.class)   // custom Java↔column mapping
    @Column(name = "tags")
    private List<String> tags = new ArrayList<>();

    @Formula("(select count(*) from attempt a where a.question_id = id)")  // [HIB] read-only SQL
    private int attemptCount;

    @Transient                                  // never persisted
    private boolean flaggedInSession;

    // one-to-one, lazy: the body is large and not always needed
    @OneToOne(mappedBy = "question", cascade = CascadeType.ALL,
              orphanRemoval = true, fetch = FetchType.LAZY, optional = false)
    private QuestionBody body;

    // MANY-TO-MANY, owning side
    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(name = "question_concept",
        joinColumns        = @JoinColumn(name = "question_id"),
        inverseJoinColumns = @JoinColumn(name = "concept_id"))
    private Set<Concept> concepts = new HashSet<>();

    // one-to-many. NOT cascaded from here — attempts are high-volume, managed separately
    @OneToMany(mappedBy = "question", fetch = FetchType.LAZY)
    @OrderBy("attemptedAt DESC")                // sorted in SQL on load
    private List<Attempt> attempts = new ArrayList<>();

    public void tagTo(Concept c) { concepts.add(c); c.getQuestions().add(this); }
    public void setBody(QuestionBody b) { this.body = b; b.setQuestion(this); }
}
```

**`Difficulty`** — an `@Embeddable` value object, not an entity:

```java
@Embeddable
public class Difficulty {
    @Column(name = "difficulty_value")
    private Integer value;                      // 1–5

    @Enumerated(EnumType.STRING)
    @Column(name = "difficulty_code", length = 10)
    private DifficultyCode code;                // EASY / MEDIUM / HARD

    protected Difficulty() {}                   // JPA needs it
}
```

It has **no identity and no table** — it's two columns on `question` wearing a class. That's the entity-vs-embeddable line: `Concept` has identity and is shared; `Difficulty` is a value that belongs to exactly one question.

**`QuestionBody`** — the split-off heavy data (shared-PK one-to-one):

```java
@Entity
@Table(name = "question_body")
public class QuestionBody {

    @Id
    private Long id;                            // NO @GeneratedValue — it shares Question's PK

    @MapsId                                     // this FK IS the PK
    @OneToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "id")
    private Question question;

    @Lob @Column(columnDefinition = "text", nullable = false)
    private String stem;

    @ElementCollection(fetch = FetchType.LAZY)  // collection of BASIC values → side table
    @CollectionTable(name = "question_option", joinColumns = @JoinColumn(name = "question_id"))
    @OrderColumn(name = "position")             // list order persisted in its own column
    @Column(name = "option_text")
    private List<String> options = new ArrayList<>();
}
```

`@MapsId` is the clean shared-PK one-to-one: `question_body.id` is simultaneously its PK and its FK to `question`. `@ElementCollection` is the right call for `options` — they're **basic values owned by the body**, not entities with identity. Nobody queries "find option 3"; options don't exist without their question.

**`Attempt`** — high-volume, mostly append:

```java
@Entity
@Table(name = "attempt", indexes = @Index(columnList = "question_id, attempted_at"))
public class Attempt extends BaseEntity {

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "question_id", nullable = false)
    private Question question;

    @Column(name = "attempted_at", nullable = false)
    private Instant attemptedAt;

    @Column(nullable = false)
    private boolean correct;

    @Column(name = "anonymous_session_id", length = 64)
    private String anonymousSessionId;          // nullable — anonymous attempts
}
```

## 11.4 Inheritance — *if* question types ever diverge

Not in the model above (nullable `type` handles it). But if MCQ and free-response genuinely diverge **structurally**:

```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)      // one table + discriminator
@DiscriminatorColumn(name = "question_kind", discriminatorType = DiscriminatorType.STRING)
public abstract class Question extends BaseEntity { ... }

@Entity @DiscriminatorValue("MCQ")
public class McqQuestion extends Question {
    private Integer correctOption;              // must be nullable — SINGLE_TABLE shares columns
}

@Entity @DiscriminatorValue("FREE_TEXT")
public class FreeTextQuestion extends Question {
    @Lob private String markScheme;
}
```

`SINGLE_TABLE` is fast (no joins) but forces subtype columns to be nullable — you lose `NOT NULL` as a guarantee. `JOINED` keeps them normalised at the cost of a join per query. **The tell:** don't reach for inheritance until the *shape* actually differs; a nullable column is cheaper than a hierarchy.

## 11.5 Collections in a JPA context

This is where the collections reference and this one meet, and where the choice of `Set` vs `List` **changes your SQL**. It's not a style preference.

| Mapping | Hibernate calls it | Behaviour |
|---|---|---|
| `List` with no `@OrderColumn` | a **bag** | unordered, allows duplicates — **the problem child** |
| `List` + `@OrderColumn` | an indexed list | order persisted in a column |
| `List` + `@OrderBy` | still a bag | ordered by SQL `ORDER BY` **on load only** |
| `Set` | a set | no duplicates, no order — **the safe default** |
| `Map` + `@MapKey*` | a map | keyed collection |

**The four things that actually bite:**

**1. `MultipleBagFetchException`.** Fetch-join two `List` collections in one query and Hibernate throws — because two bags produce a cartesian product it can't de-duplicate:

```java
// BOOM: MultipleBagFetchException
@Query("select q from Question q join fetch q.concepts join fetch q.attempts")
```
Fixes: make them `Set`s (Hibernate can dedupe), or fetch **one collection per query** and let the second batch-load. This is why `Set` is the default above.

**2. `@ManyToMany` with `List` is quietly terrible.** Remove one row from a `List`-mapped junction and Hibernate **deletes every junction row and re-inserts the survivors** — it can't identify rows without an index. With a `Set`, it issues one targeted `DELETE`. This alone justifies `Set` for every `@ManyToMany`.

**3. Cartesian products.** Even legally, fetching a to-many multiplies parent rows — 1 question × 3 concepts = 3 rows, one question object. `Set` hides it; a bag would show you duplicates. Hibernate 6 de-duplicates entity results automatically, but the *rows crossing the wire* still multiply — fetch two collections and you get 3×5=15 rows for one question. Fetch **one collection at a time**.

**4. `equals`/`hashCode` on entities in a `Set`.** A `HashSet` calls `hashCode` on add. A transient entity's `@Id` is `null`; after persist it isn't — so a **hash based on `id` changes after save**, and the entity is lost in its own set (collections reference §7's mutation trap, in the wild). Hence §11.7.

> **The tell:** **`Set` for every association** unless you have a real reason. `List` + `@OrderColumn` only when persisted order is a genuine requirement (question options — hence `@ElementCollection` + `@OrderColumn` above). `@OrderBy` when you want sorted-on-load without storing order (attempts). Never two bags in one fetch.

## 11.6 The design decisions, justified

The choices worth being able to defend — each with the alternative and the tell:

| Decision | Chosen | Alternative | Why |
|---|---|---|---|
| Concept↔SpecSection | **many-to-many junction** | duplicate a Concept per board | one canonical concept, no update anomalies — *the entire premise of board-agnostic tagging* |
| Question↔Concept | **many-to-many** | FK (one concept per question) | a question genuinely spans concepts; a junction costs one table and buys real flexibility. **If it's always exactly one, use the FK** — don't pay for generality you don't need |
| PKs | **surrogate `@Id` + `@NaturalId`** | natural key as PK | business values change; a PK change ripples through every FK |
| ID generation | **SEQUENCE** | IDENTITY | IDENTITY silently kills insert batching on Postgres — the seed pipeline cares |
| Concurrency | **`@Version`** | pessimistic / higher isolation | the review workflow is textbook lost-update: two reviewers, HTTP round-trip, no locks held |
| Body split | **lazy one-to-one, `@MapsId`** | columns on `question` | large text you rarely need on list endpoints; the split keeps the hot table narrow |
| `options` | **`@ElementCollection`** | an `Option` entity | options have no identity and are never queried independently — an entity would be ceremony |
| `Difficulty` | **`@Embeddable`** | two loose columns / an entity | cohesive value with behaviour, no identity — columns wearing a class |
| Attempts cascade | **none** | `cascade = ALL` | high-volume; you don't want deleting a question to cascade through 10k attempts in the persistence context. Handle with a bulk `DELETE` or DB-level FK rule |
| Enum storage | **`EnumType.STRING`** | ORDINAL / native PG enum | readable in the DB, survives reordering; ORDINAL breaks the day someone inserts a value |
| Fetching | **LAZY everywhere** | the EAGER to-one default | fetch per query with `@Join`/entity graph; eager-by-default loads half the graph on every call |
| L2 cache | **on for reference data only** | on everywhere / nowhere | ExamBoard/Concept are read constantly, change rarely. Question/Attempt are write-active — caching them buys staleness |
| Shared fields | **`@MappedSuperclass`** | `@Inheritance` | they share columns, not a type hierarchy |

## 11.7 Entity `equals`/`hashCode` — the trap, resolved

The `BaseEntity` implementation above looks odd. It's deliberate, and it's the standard answer to a genuine problem:

- **`hashCode()` returns a constant** (`getClass().hashCode()`). Ugly — every entity of a type lands in one hash bucket — but it's **stable across the transient→persistent transition**, which is the only property that matters. Collections of entities are small; the bucket cost is irrelevant. A hash that *changes* when Hibernate assigns the id would lose the object inside its own `HashSet`.
- **`equals()` requires a non-null id** and returns false for two transient instances. Two unsaved questions are not equal — correct: they have no identity yet.
- **`Hibernate.getClass(o)`** instead of `o.getClass()`: a lazy proxy's class is a generated subclass, so a naive `getClass()` comparison says a proxy ≠ its entity. This unwraps it.

The alternative — and the better one when you have a real business key — is to base `equals`/`hashCode` on the `@NaturalId` (`concept.slug`, `examBoard.code`), which is immutable and known before persist. Use that when it exists; use the id-based version above when it doesn't.

## 11.8 The use cases, end to end

**(a) The public list endpoint — `GET /questions?conceptId=`.** Read-only, needs no association objects → a **projection**, not entities:

```java
public record QuestionSummary(Long id, String stem, Status status, Integer difficulty) {}

@ReadOnly                                                   // Micronaut: read-only tx
public List<QuestionSummary> list(@Nullable Long conceptId) {
    var spec = QuerySpecification.<Question>where((root, q, cb) ->
            cb.equal(root.get(Question_.status), Status.APPROVED));
    if (conceptId != null) {
        spec = spec.and((root, q, cb) -> {
            Join<Question, Concept> c = root.join(Question_.concepts);   // join the junction
            return cb.equal(c.get(Concept_.id), conceptId);
        });
    }
    return repository.findAll(spec).stream().map(this::toSummary).toList();
}
```
Design notes: `APPROVED` is structural (no status parameter exists to abuse); the spec composes so the `where` grows with the runtime filters; the projection sidesteps N+1 entirely because no association is ever materialised. The index that serves this is `(concept_id, status)` — **in Flyway**, not `@Index`.

**(b) The review workflow — approve a question.** Optimistic locking earning its keep:

```java
@Transactional
public void approve(Long id, String reviewer) {
    Question q = repository.findById(id).orElseThrow(() -> new NotFoundException(id));
    if (q.getStatus() != Status.PENDING) throw new IllegalStateException("not pending");
    q.setStatus(Status.APPROVED);          // dirty checking → UPDATE at commit
}   // UPDATE question set status=?, version=? where id=? and version=?
    // → 0 rows if another reviewer got there first → OptimisticLockException → HTTP 409
```
No `save()` call — the entity is managed, so dirty checking emits the `UPDATE`. `@DynamicUpdate` means only `status`, `version`, `updated_at` are in it, which narrows the window for version contention.

**(c) The syllabus view — concepts with approved questions.** An `EXISTS` semi-join, because we need *existence*, not child columns:

```java
@Query("""
    select distinct c from Concept c
    join fetch c.specSections s
    where s.examBoard.code = :board
      and exists (select 1 from Question q join q.concepts qc
                  where qc = c and q.status = 'APPROVED')
    """)
List<Concept> syllabusFor(String board);
```
One fetch-join (one collection only — §11.5), and `exists` rather than a join to `questions` so concepts aren't multiplied by their question count.

**(d) The seed pipeline — and why it leaves the ORM.** Bulk-loading thousands of questions through the persistence context means dirty-checking every one on every flush. The ORM options are `flush()`/`clear()` every ~50 rows plus `hibernate.jdbc.batch_size`, or `StatelessSession` (no persistence context at all). But the right answer here is the one already chosen: a **native CTE against a staging table, in Flyway** — Postgres-shaped, set-based, no persistence context to manage. **The tell:** when the ORM's unit-of-work is pure overhead, leave it. Per subsystem, not per app.

**(e) Housekeeping — purge old anonymous attempts.** A bulk JPQL update that bypasses everything:

```java
@Transactional
public int purgeAnonymous(Instant before) {
    int n = em.createQuery("""
            delete from Attempt a
            where a.anonymousSessionId is not null and a.attemptedAt < :before
            """)
        .setParameter("before", before)
        .executeUpdate();     // ONE SQL DELETE — no entities loaded, no cascade
    em.clear();               // anything already loaded is now stale
    return n;
}
```
This is exactly why `Attempt` has **no cascade from `Question`**: a set-based delete is the right tool, and cascade would have dragged the whole graph into memory.

---

## Expanding this

Ask and I'll expand any of: **composite keys** (`@EmbeddedId` vs `@IdClass`, worked); **`AttributeConverter`** written end to end (the `TagListConverter` above); **the static metamodel** wired into your Gradle build (`Question_`, `Concept_`); **`@Version` optimistic locking** through to the HTTP 409 handler; **entity graphs vs `@Join` vs `JOIN FETCH`** with the SQL side by side; **`StatelessSession`** for the seed pipeline; **the Flyway migrations** for this whole model, with the indexes and `EXPLAIN ANALYZE` before/after.
