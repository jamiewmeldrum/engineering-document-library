# Java Collections — A Reference

*A compact lookup reference: what each type is, its actual cost, when to reach for it, and the surrounding ideas (sorting, equality, immutability, streams) that make it work. Java 21. Companion to the Java data-access primer — same conventions: tables, "the tell", derive-don't-memorise.*

The organising idea: the Collections Framework is **interfaces** (the contract you should code against) and **implementations** (the data structure you actually get). Declare the interface, instantiate the implementation — `List<Question> qs = new ArrayList<>();` — so you can swap the structure without touching the calling code. Almost every "which collection?" decision is really "which *implementation*, given how I'll use it?"

---

## 1. The type hierarchy

```
Iterable
└── Collection
    ├── List      — ordered, indexed, duplicates allowed
    ├── Set       — no duplicates, (usually) no index
    └── Queue     — ordered for processing (FIFO etc.)
        └── Deque — double-ended: both queue and stack

Map              — key→value pairs (NOT a Collection; separate root)
```

`Map` sitting outside `Collection` is not an oversight: a `Collection` is a bag of single elements, a `Map` is a bag of *pairs*. It offers `keySet()`, `values()`, `entrySet()` as collection *views* onto itself.

---

## 2. List — ordered, indexed, duplicates allowed

| Implementation | Backed by | get(i) | add (end) | add/remove (middle) | Use when |
|---|---|---|---|---|---|
| **ArrayList** | resizable array | **O(1)** | O(1) amortised | O(n) (shifts) | **the default.** Random access, iteration, append |
| **LinkedList** | doubly-linked nodes | O(n) | O(1) | O(1) *at a held position* | almost never — use as a `Deque` if anything |
| **CopyOnWriteArrayList** | array, copied on write | O(1) | O(n) (copies!) | O(n) | many readers, very rare writes, concurrent |

```java
List<Question> questions = new ArrayList<>();          // declare the interface
questions.add(q1);                                      // append
questions.add(0, q2);                                   // insert at index — O(n), shifts
questions.get(0);                                       // O(1)
questions.remove(q2);                                   // by object (uses equals)
questions.remove(0);                                    // by index — watch the overload!
questions.contains(q1);                                 // O(n) — linear scan
questions.indexOf(q1);                                  // O(n)
questions.set(0, q3);                                   // replace
```

Note `remove(int)` vs `remove(Object)`: on a `List<Integer>`, `list.remove(1)` removes **index 1**, not the value 1. Use `list.remove(Integer.valueOf(1))` for the value. A genuine bug source.

**Finding duplicates** — the `Set`-return idiom is the neat one:

```java
// does it contain duplicates? (Set.add returns false if already present)
boolean hasDupes = questions.size() != new HashSet<>(questions).size();

// which ones are duplicated?
Set<Question> seen = new HashSet<>();
Set<Question> dupes = questions.stream()
    .filter(q -> !seen.add(q))       // add() returns false → already seen
    .collect(Collectors.toSet());

// dedupe, preserving order
List<Question> unique = new ArrayList<>(new LinkedHashSet<>(questions));
```

All three depend entirely on a correct `equals`/`hashCode` (§7) — with the default identity-based ones, nothing is ever a duplicate.

> **The tell:** use `ArrayList` unless you can name why not. `LinkedList` is the classic trap — its O(1) insert only applies if you're already *at* the node (via an iterator); reaching index *i* costs O(n) anyway, and its cache locality is dreadful. Even queue/stack use is better served by `ArrayDeque`.

---

## 3. Set — no duplicates

| Implementation | Backed by | Order | add/contains | Use when |
|---|---|---|---|---|
| **HashSet** | `HashMap` | **none** (arbitrary) | O(1) avg | **the default.** Membership tests, dedup |
| **LinkedHashSet** | hash + linked list | **insertion order** | O(1) avg | dedup but output order must be predictable |
| **TreeSet** | red-black tree | **sorted** | O(log n) | need sorted iteration, or range queries |
| **EnumSet** | bit vector | enum declaration order | O(1), tiny | **keys are enum constants** — extremely fast/compact |
| **CopyOnWriteArraySet** | copy-on-write array | insertion | O(n) | tiny set, mostly reads, concurrent |

`TreeSet` (a `NavigableSet`) also gives `first()`, `last()`, `headSet()`, `tailSet()`, `subSet()`, `floor()`, `ceiling()` — range and neighbour queries a hash set can't answer.

```java
Set<Status> reviewable = EnumSet.of(Status.DRAFT, Status.PENDING);   // enum → EnumSet
boolean added = seen.add(q);        // false if already present — useful as a test
Set<Long> a = new HashSet<>(List.of(1L, 2L, 3L));
Set<Long> b = new HashSet<>(List.of(2L, 3L, 4L));
Set<Long> union = new HashSet<>(a); union.addAll(b);         // {1,2,3,4}
Set<Long> intersect = new HashSet<>(a); intersect.retainAll(b); // {2,3}
Set<Long> diff = new HashSet<>(a); diff.removeAll(b);          // {1}

TreeSet<Question> byDiff = new TreeSet<>(Comparator.comparing(Question::getDifficulty));
byDiff.first(); byDiff.headSet(q); byDiff.ceiling(q);         // range/neighbour queries
```

> **The tell:** `HashSet` by default; `LinkedHashSet` when tests or output need deterministic order; `TreeSet` when you need *sorted* or *range*; `EnumSet` whenever the elements are enum constants (e.g. a set of `Status` — it's a bitmask under the hood and beats `HashSet` comfortably).

---

## 4. Map — key→value

| Implementation | Backed by | Order | get/put | Use when |
|---|---|---|---|---|
| **HashMap** | hash table | **none** | O(1) avg | **the default** |
| **LinkedHashMap** | hash + linked list | insertion *(or access)* | O(1) avg | predictable order; **LRU cache** via access-order + `removeEldestEntry` |
| **TreeMap** | red-black tree | **sorted by key** | O(log n) | sorted iteration, range queries (`NavigableMap`) |
| **EnumMap** | array indexed by ordinal | enum order | O(1), tiny | **enum keys** |
| **ConcurrentHashMap** | segmented hash table | none | O(1) avg | **concurrent access** — the default for shared mutable maps |
| **Hashtable** | hash table | none | O(1), fully synchronised | **never** — legacy, use `ConcurrentHashMap` |

**Modern `Map` methods worth knowing** (they replace a lot of clumsy null-checking):

| Method | Does |
|---|---|
| `getOrDefault(k, def)` | value or a default, no null check |
| `putIfAbsent(k, v)` | only inserts if absent |
| `computeIfAbsent(k, k -> new ArrayList<>())` | **build multi-maps in one line** |
| `compute` / `merge(k, 1, Integer::sum)` | **counting/accumulating** idiom |
| `forEach((k,v) -> ...)` | iterate pairs directly |

```java
// group questions by concept — the computeIfAbsent idiom
Map<Long, List<Question>> byConcept = new HashMap<>();
for (Question q : questions)
    byConcept.computeIfAbsent(q.getConceptId(), k -> new ArrayList<>()).add(q);

// count questions per status — the merge idiom
Map<Status, Integer> counts = new EnumMap<>(Status.class);
for (Question q : questions)
    counts.merge(q.getStatus(), 1, Integer::sum);
```

**Traversing a Map — the four ways** (worth having at your fingertips; the last is usually right):

```java
// 1. entrySet — best: one pass, key and value together
for (Map.Entry<Long, Question> e : map.entrySet())
    System.out.println(e.getKey() + " → " + e.getValue());

// 2. keySet, then get(k) — WORST: a second hash lookup per key
for (Long k : map.keySet()) System.out.println(k + " → " + map.get(k));

// 3. values() — when you don't need keys
for (Question q : map.values()) process(q);

// 4. forEach — cleanest for simple side effects
map.forEach((k, v) -> System.out.println(k + " → " + v));
```

Note an `Iterator` over `entrySet()` is also the only way to **remove** entries mid-traversal safely.

### 4.1 How HashMap actually works

Worth understanding properly — it's the most-asked collections question in interviews, and it explains the `equals`/`hashCode` contract (§7) rather than leaving it as a rule to memorise.

**The structure.** A `HashMap` is an **array of buckets** (`Node[] table`). Each `Node` holds the key, the value, the cached hash, and a `next` pointer.

**`put(k, v)`:**
1. Compute `k.hashCode()`, then apply an internal **spread** function (XOR the high bits down into the low bits). Why: the bucket index uses only the *low* bits, so keys differing only in their high bits would otherwise all collide.
2. Index the bucket: `(n - 1) & hash`, where `n` is the table length. This works as a cheap modulo *because n is always a power of two* — which is why capacity is always rounded up to one.
3. If the bucket is empty → place the node.
4. If occupied (a **collision**) → walk the chain, comparing **first `hash`, then `equals`**. Match → overwrite the value. No match → append.

**`get(k)`:** same hash → same bucket → walk the chain comparing `hash` then `equals`. This is exactly why breaking the contract breaks the map: an object whose `hashCode` changed hashes to a *different bucket* than the one it's sitting in, so `get`/`contains` look in the wrong place and find nothing.

**Treeification (Java 8+).** If a single bucket's chain exceeds **8** nodes (and the table is ≥ 64), that chain converts to a **red-black tree**, turning worst-case lookup from O(n) to O(log n). This was a DoS hardening measure — before it, an attacker who could force mass collisions could degrade a `HashMap` to a linked list. It untreeifies below 6.

**Load factor and resizing.** Default capacity **16**, load factor **0.75**. When `size > capacity × loadFactor` (12 for a fresh map), the table **doubles** and every entry is **rehashed** into the new, larger table — an O(n) operation. So a map you know will hold ~1000 entries is worth pre-sizing (`new HashMap<>(1400)`) to avoid several resize cycles.

**Complexity:** O(1) average for `get`/`put`; O(log n) worst case in a treeified bucket. "O(1)" assumes a decent `hashCode` spread — a `hashCode()` returning a constant is legal, correct, and turns your map into a list.

> **The tell:** `HashMap` performance *is* the quality of your `hashCode`. Use immutable keys, let records/IDE generate `hashCode`, and pre-size when you know the volume.

**`HashSet` footnote:** it's a `HashMap` with a dummy value object in every entry. Literally — that's the whole implementation. Which is why its characteristics are identical.

---

## 5. Queue & Deque — processing order

| Implementation | Shape | Use when |
|---|---|---|
| **ArrayDeque** | resizable circular array | **the default for both queue and stack.** Fast, no nulls allowed |
| **LinkedList** | linked nodes | implements `Deque` too, but `ArrayDeque` is better |
| **PriorityQueue** | binary heap | poll the **smallest/highest-priority** element next (not FIFO) |
| **ArrayBlockingQueue / LinkedBlockingQueue** | blocking | **producer-consumer** across threads |
| **ConcurrentLinkedQueue** | lock-free | high-throughput concurrent, non-blocking |

Two API pairs that matter — one throws, one signals:

| Operation | Throws on failure | Returns special value |
|---|---|---|
| insert | `add(e)` | `offer(e)` → false |
| remove | `remove()` | `poll()` → null |
| examine | `element()` | `peek()` → null |

```java
// as a QUEUE (FIFO)
Deque<Question> queue = new ArrayDeque<>();
queue.offer(q1);              // add to tail
Question next = queue.poll(); // take from head — null if empty

// as a STACK (LIFO) — this replaces the legacy Stack class
Deque<Question> stack = new ArrayDeque<>();
stack.push(q1);               // add to head
Question top = stack.pop();   // take from head — throws if empty

// priority: smallest/most-urgent first, NOT insertion order
Queue<Question> urgent = new PriorityQueue<>(Comparator.comparing(Question::getDifficulty));
urgent.poll();                // lowest difficulty first
```

`PriorityQueue` catch: only `peek()`/`poll()` respect priority — **iterating it does not give sorted order** (it's a heap, not a sorted list). To drain in order, `poll()` in a loop.

> **The tell:** need a **stack**? `ArrayDeque` (`push`/`pop`), **not** the legacy `Stack` class (it's synchronised and extends `Vector`). Need a **queue**? `ArrayDeque`. Need **"most urgent first"**? `PriorityQueue`. Need to **hand work between threads**? a `BlockingQueue`.

---

## 6. Sorting and ordering

Two distinct mechanisms — this distinction is the point:

- **`Comparable<T>`** — *natural order*, implemented **by the class itself** (`int compareTo(T other)`). One per class. `String`, `Integer`, `LocalDate` all have it.
- **`Comparator<T>`** — an *external* ordering, one of many, passed in.

```java
questions.sort(Comparator.comparing(Question::getDifficulty));                 // by one key
questions.sort(Comparator.comparing(Question::getDifficulty).reversed());      // descending
questions.sort(Comparator.comparing(Question::getConceptId)
                         .thenComparing(Question::getDifficulty));             // tie-break
questions.sort(Comparator.comparing(Question::getReviewedAt,
                         Comparator.nullsLast(Comparator.naturalOrder())));    // null-safe
```

Also: `Comparator.comparingInt/Long/Double` (avoids boxing), `Collections.sort(list)`, `list.sort(cmp)`, `Arrays.sort(arr)`, `TreeMap/TreeSet` constructors take a comparator.

**Contract, and why breaking it bites:** `compareTo`/`compare` must be *consistent, transitive, and antisymmetric* — `sgn(a.compareTo(b)) == -sgn(b.compareTo(a))`. An inconsistent comparator throws `IllegalArgumentException: Comparison method violates its general contract!` from `sort` (the classic being a `compare` written as `a.x - b.x` that overflows on large ints — use `Integer.compare(a.x, b.x)`). And it *should* be consistent with `equals`, or `TreeSet`/`TreeMap` will consider unequal objects duplicates (they use `compareTo`, **not** `equals`, to decide identity — a genuine gotcha).

Java's sort is **stable** for objects (`TimSort` — equal elements keep their relative order), which is what makes `thenComparing`-style multi-pass sorting behave.

> **The tell:** one obvious intrinsic order → `Comparable`. Multiple/contextual orders, or you don't own the class → `Comparator`. In practice, `Comparator.comparing(...)` covers ~95% of real sorting.

---

## 7. equals / hashCode — the contract hash collections depend on

`HashMap`/`HashSet` *are* `hashCode()` and `equals()`. The contract:

1. Equal objects **must** have equal hash codes.
2. Unequal objects *may* share a hash code (a collision — fine, just slower).
3. Both must be consistent while the object is in the collection.

**The mutation trap:** put an object in a `HashSet`, then mutate a field used by `hashCode()` — it now hashes to a different bucket and is effectively lost (`contains` returns false while it's still in there). Hash keys should be **immutable**, or at least never mutated in their key fields.

**Practical:** override both together, always; use `Objects.equals`/`Objects.hash`; let the IDE generate them; or use a **`record`**, which generates correct `equals`/`hashCode`/`toString` for you — the right default for value types and map keys.

> **JPA footnote:** entity `equals`/`hashCode` is a known trap — a generated `@Id` is null before persist, so id-based equality breaks in sets. Prefer a business/natural key, or don't put unsaved entities in hash collections. (See the data-access primer.)

---

## 8. Immutability and defensive copies

| Factory | Gives | Nulls? | Notes |
|---|---|---|---|
| `List.of(...)`, `Set.of(...)`, `Map.of(k,v,...)` | **truly immutable** | **rejected** | Java 9+; the modern default |
| `List.copyOf(coll)` | immutable snapshot | rejected | defensive copy |
| `Collectors.toUnmodifiableList()` | immutable | rejected | stream terminal |
| `Collections.unmodifiableList(l)` | **unmodifiable *view*** | allowed | **the wrapped list can still change under you** |
| `Arrays.asList(a, b)` | **fixed-size view of an array** | allowed | `set` works, `add`/`remove` throw |

Three traps worth internalising: `Collections.unmodifiableList` is a *view*, not a copy — mutate the original and the "unmodifiable" one changes too. `Arrays.asList` is backed by the array and is fixed-size. `List.of(...)` rejects nulls outright, which surprises code that relied on a null slot.

> **The tell:** returning a collection from a class? Return `List.copyOf(...)` or an immutable view so callers can't mutate your internals. Building a constant? `List.of(...)`.

---

## 9. Iteration and safe removal

```java
for (Question q : questions) { ... }              // enhanced for — the default
questions.forEach(q -> ...);                       // Iterable.forEach
```

**`ConcurrentModificationException`** is the classic: structurally modifying a collection while iterating it. Three correct fixes:

```java
questions.removeIf(q -> q.getStatus() == DRAFT);   // best — expressive and safe
// or an explicit Iterator:
var it = questions.iterator();
while (it.hasNext()) if (it.next().isStale()) it.remove();
// or iterate a copy, mutate the original
```

Note the exception is **best-effort, not a guarantee** — it's a bug detector, not a lock. For genuinely concurrent access use `ConcurrentHashMap`/`CopyOnWriteArrayList`, whose iterators never throw it.

**Fail-fast vs fail-safe** — the standard vocabulary for the two iterator styles:

| | **Fail-fast** | **Fail-safe** (better: *weakly consistent*) |
|---|---|---|
| Who | `ArrayList`, `HashMap`, `HashSet` — the standard collections | `ConcurrentHashMap`, `CopyOnWriteArrayList` |
| On concurrent modification | throws **`ConcurrentModificationException`** | doesn't throw |
| Mechanism | an internal `modCount` is bumped on every structural change; the iterator snapshots it and compares on each `next()` | iterates a snapshot (copy-on-write) or tolerates concurrent updates (CHM) |
| You see | changes made through the iterator only | possibly stale data — elements added after iteration began may or may not appear |

"Fail-safe" is a slightly misleading name: nothing is safe, you've just traded an exception for **potentially stale reads**. `CopyOnWriteArrayList`'s iterator walks the array as it was at creation, so later writes are invisible to it. `ConcurrentHashMap`'s is *weakly consistent* — it reflects some updates, guarantees no duplicates or CME, and never throws.

> **The tell:** a CME is a **bug being reported to you**, not a concurrency problem to paper over by swapping in a concurrent collection. Single-threaded? Use `removeIf`/`Iterator.remove`. Genuinely multi-threaded? Then pick the concurrent collection deliberately, understanding you're accepting stale reads.

---

## 10. Streams — the processing layer over collections

Streams don't replace collections; they're a pipeline **over** them: `source → intermediate (lazy) → terminal (triggers work)`.

```java
Map<Long, List<String>> approvedTextByConcept = questions.stream()
    .filter(q -> q.getStatus() == APPROVED)                       // intermediate
    .sorted(Comparator.comparing(Question::getDifficulty))        // intermediate
    .collect(Collectors.groupingBy(Question::getConceptId,        // terminal
             Collectors.mapping(Question::getText, toList())));
```

| Collector | Gives |
|---|---|
| `toList()` / `toSet()` | a `List`/`Set` |
| `toMap(k, v)` | a `Map` — **throws on duplicate keys** unless you pass a merge fn |
| `groupingBy(fn)` | `Map<K, List<T>>` — the workhorse |
| `partitioningBy(pred)` | `Map<Boolean, List<T>>` |
| `joining(", ")` | a `String` |
| `counting()`, `summingInt()`, `averagingDouble()` | aggregates |

`stream().toList()` (Java 16+) is the concise immutable-list terminal.

> **The tell:** streams for *transformation pipelines* (filter→map→group). A plain loop for simple iteration with side effects — a `for` loop is clearer and faster than a one-step stream. Don't reach for `parallelStream()` without measuring; it's rarely a win below very large N and can be actively harmful.

---

## 11. Concurrency

| Need | Use |
|---|---|
| shared mutable map | **`ConcurrentHashMap`** |
| shared list, rare writes | `CopyOnWriteArrayList` |
| hand work between threads | `BlockingQueue` (`ArrayBlockingQueue`, `LinkedBlockingQueue`) |
| sorted concurrent map | `ConcurrentSkipListMap` |
| anything | **never** `Hashtable`/`Vector`/`synchronizedMap` for new code |

`Collections.synchronizedMap(m)` wraps every method in a lock but **compound operations still aren't atomic** (`if (!map.containsKey(k)) map.put(k, v)` is a race) — and you must manually synchronise while iterating. `ConcurrentHashMap` solves both with atomic `putIfAbsent`/`compute`/`merge`.

---

## 12. The legacy corner — and why it's retired

These are pre-Java-2 classes you'll meet in old code and interview questions. You won't write them, but knowing *why* they lost is the actual lesson.

| Legacy | Modern | Why the legacy one lost |
|---|---|---|
| **Vector** | `ArrayList` | every method is `synchronized` — you pay lock overhead on every call even single-threaded, and it *still* doesn't make compound operations safe |
| **Hashtable** | `HashMap` / `ConcurrentHashMap` | same: method-level synchronisation, one lock for the whole table → no concurrency. Also rejects null keys/values |
| **Stack** (extends Vector) | `ArrayDeque` | synchronised, and inherits `Vector`'s API — you can index into a "stack", which is nonsense |
| **Enumeration** | `Iterator` | no `remove()`, clumsier method names (`hasMoreElements`/`nextElement`) |

**`HashMap` vs `Hashtable`** — the classic question, and the real answer: `Hashtable` is synchronised, `HashMap` isn't; `Hashtable` rejects nulls, `HashMap` allows one null key and many null values; `Hashtable` predates the Collections Framework and was retrofitted into it. **But the answer that actually matters:** the choice isn't between them. If you need thread-safety you want `ConcurrentHashMap`, not `Hashtable` — because `Hashtable` locks the *entire table* on every operation, while `ConcurrentHashMap` locks only the bucket being written (and reads are essentially lock-free). Same safety, dramatically better throughput under contention. `Hashtable` is safe *and* slow — the worst pairing.

**`ArrayList` vs `Vector`:** same data structure, but `Vector` synchronises every method and doubles its capacity on growth (`ArrayList` grows by ~50%). The synchronisation is the point: it's the wrong granularity. Locking per-method doesn't make `if (!v.contains(x)) v.add(x)` atomic anyway — you need an external lock for that, at which point `Vector`'s internal locking is pure overhead.

**`Iterator` vs `Enumeration`:** `Iterator` added `remove()` (safe removal mid-traversal) and fail-fast behaviour. `Enumeration` survives only in legacy APIs.

> **The lesson worth carrying:** these classes are a case study in **thread-safety at the wrong granularity**. Synchronising individual methods gives you safety that's simultaneously too slow (locks you often don't need) and too weak (compound operations still race). That's why the modern answer is *either* an unsynchronised collection you use single-threaded, *or* a purpose-built concurrent collection with atomic compound operations (`putIfAbsent`, `compute`, `merge`) — never a blanket-synchronised one.

---

## 13. Utility classes

- **`Collections`** — `sort`, `reverse`, `shuffle`, `max`, `min`, `frequency`, `emptyList()`, `singletonList(x)`, `nCopies`, `unmodifiable*`, `synchronized*`.
- **`Arrays`** — `asList`, `sort`, `binarySearch`, `fill`, `copyOf`, `stream`, `equals`/`deepEquals`, `toString`/`deepToString`.
- **`Objects`** — `equals`, `hash`, `requireNonNull`, `toString`, `requireNonNullElse`.

---

## 14. Choosing — the decision table

| I need… | Use |
|---|---|
| ordered, indexed, general purpose | **`ArrayList`** |
| no duplicates, order irrelevant | **`HashSet`** |
| no duplicates, insertion order | `LinkedHashSet` |
| no duplicates, sorted / range queries | `TreeSet` |
| key→value, general purpose | **`HashMap`** |
| key→value, sorted / range queries | `TreeMap` |
| key→value, insertion order or LRU | `LinkedHashMap` |
| keys/elements are enum constants | **`EnumMap` / `EnumSet`** |
| FIFO queue or stack | **`ArrayDeque`** |
| "most urgent first" | `PriorityQueue` |
| shared across threads | `ConcurrentHashMap` / `BlockingQueue` |
| a constant | `List.of(...)` / `Set.of(...)` / `Map.of(...)` |

**The two-question shortcut:** *(1) Do I need key→value, uniqueness, or a sequence?* → picks the interface. *(2) Do I need ordering (none / insertion / sorted) and is it concurrent?* → picks the implementation. Everything else is a special case: enum keys → `Enum*`, priority → heap, tiny+read-mostly+concurrent → copy-on-write.

---

## 15. The classic interview questions → where they're answered

Collections is a guaranteed interview topic. The classics, mapped to sections — and a note on which are actually still asked.

| Question | Section | Still asked? |
|---|---|---|
| Difference between `List`, `Set`, `Map`? | §1–4 | **yes** — fundamentals |
| When would you use X over Y? | §14 | **yes** — this is the modern focus |
| How does `HashMap` work internally? | **§4.1** | **yes** — the single most-asked |
| `equals`/`hashCode` contract; what breaks if violated? | §7, §4.1 | **yes** |
| `HashMap` vs `Hashtable`? | §12 | asked, but the *real* answer is `ConcurrentHashMap` |
| `Hashtable` vs `ConcurrentHashMap`? | §12, §11 | **yes** — lock granularity is the point |
| `ArrayList` vs `LinkedList`? | §2 | **yes** |
| `ArrayList` vs `Vector`? | §12 | fading — but the *why* still matters |
| Fail-fast vs fail-safe iterators? | §9 | **yes** |
| `Iterator` vs `Enumeration`? | §12 | mostly retired |
| Four ways to traverse a `Map`? | §4 | **yes** — often as a coding task |
| How do you find duplicates in a `List`? | §2 | **yes** — coding task |
| `Comparable` vs `Comparator`? | §6 | **yes** |
| How do you make a collection thread-safe? | §11, §12 | **yes** |
| How do you make a collection immutable? | §8 | **yes** — the view-vs-copy trap |

**The framing that separates a good answer from a recited one:** most of these are "difference between X and Y" questions, and the strong answer never stops at the difference — it names the **trade-off** and then says **which you'd actually pick and why**. "`Hashtable` is synchronised and `HashMap` isn't" is a recited fact. "Both are the wrong choice — if I need thread-safety I want `ConcurrentHashMap`, because `Hashtable` locks the whole table per operation while CHM locks per-bucket and gives me atomic `compute`/`merge`; `Hashtable` is safe *and* slow" is an engineer answering. Same for `ArrayList`/`LinkedList`: don't recite "array vs nodes," say "`ArrayList` always, unless I'm at a held iterator position — `LinkedList`'s O(1) insert is a lie once you count the O(n) traversal to reach the node, and its cache locality is terrible."

That's the whole reason this document leads with *the tell* on every section rather than just characteristics: the characteristics are the setup, the decision is the answer.

---

## Expanding this

Ask and I'll expand any of: **`ConcurrentHashMap` internals** (bucket-level locking, the Java 8 rewrite away from segments, why reads are lock-free); **`Comparator` in depth** (contract violations, null handling, locale-aware `Collator`); **streams properly** (laziness, short-circuiting, custom collectors, when parallel actually pays); **`ArrayList` growth mechanics** (capacity, `ensureCapacity`, why `System.arraycopy` beats linked nodes in practice); **collections in a JPA context** (why `Set` vs `List` on `@OneToMany` changes the SQL, `MultipleBagFetchException`, `@OrderColumn`) — that last one is the highest-value follow-up given Practiq, and it's where this doc meets the JPA reference.
