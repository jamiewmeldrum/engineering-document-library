# Modern Java — A Primer for the Returning Engineer

*For someone who can write code and solve problems, but never learned Java formally and has been away a few years. Aligned to **Java 21** (what Practiq runs) with an eye on **Java 25** (the current LTS, Sept 2025). Release facts verified July 2026. Companion to the data-access, collections, and JPA references — same conventions: tables, snippets, "the tell", derive-don't-memorise.*

The premise: you don't need Java explained from `public static void main`. You need the **delta** — what changed, what's now idiomatic, what you'd get pulled up on in a code review, and what the JVM is doing while your code runs. This document is that, organised so you can read it front-to-back once and then use it as a lookup.

The single most useful orientation: **Java's rate of change went up sharply in 2018.** Before that, a release every 2–3 years. Since then, **one every six months**, with a **Long-Term Support (LTS)** release every two years. So "Java has barely changed" is a 2015 opinion. If you last wrote Java 8 or 11 seriously, the language you're coming back to has records, sealed types, pattern matching, switch expressions, text blocks, and virtual threads — and the idioms have moved with them.

**Where the versions stand (July 2026):**

| Release | Date | Status |
|---|---|---|
| Java 8 | 2014 | the great holdout; still in the wild, EOL creeping |
| Java 11 | 2018 | LTS — the first modern LTS |
| Java 17 | 2021 | LTS — huge adoption |
| **Java 21** | Sept 2023 | LTS — **what Practiq runs**; virtual threads land here |
| Java 25 | Sept 2025 | **LTS — the current one** |
| Java 26 | Mar 2026 | non-LTS, supported 6 months (until Sept 2026) |
| Java 27 | Sept 2026 | next non-LTS |
| Java 29 | Sept 2027 | the next LTS |

> **The tell — which version:** run an **LTS** in production; use non-LTS only if you want a specific feature and can upgrade every six months. Java 21 is a perfectly defensible choice for Practiq right now (broad tooling/library support, virtual threads, all the pattern matching). **Java 25 is the current LTS** and is the natural upgrade target — worth doing at a quiet moment, not mid-sprint. Skip 26/27 for a product.

Contents:

- **Part 1** — what changed while you were away (the orientation table)
- **Part 2** — modern data modelling: records, sealed types, pattern matching, switch
- **Part 3** — strings and text
- **Part 4** — generics
- **Part 5** — collections (brief — the deep dive is its own doc)
- **Part 6** — lambdas and functional interfaces
- **Part 7** — streams
- **Part 8** — Optional
- **Part 9** — date and time
- **Part 10** — exceptions
- **Part 11** — equality, immutability, and object basics
- **Part 12** — concurrency, from threads to virtual threads
- **Part 13** — the JVM: memory and garbage collection
- **Part 14** — packaging: classpath, modules, jars, containers
- **Part 15** — files and I/O
- **Part 16** — security fundamentals
- **Part 17** — what changed in every release since Java 6
- **Part 18** — when to use what: decision frameworks

---

# Part 1 — What changed while you were away

If you read one table, this one. Old idiom → modern idiom, and where it's covered.

| You probably write | Modern Java | Since | §|
|---|---|---|---|
| `new Date()`, `Calendar`, `SimpleDateFormat` | `java.time`: `Instant`, `LocalDate`, `DateTimeFormatter` | 8 | §9 |
| anonymous inner classes | lambdas and method references | 8 | §6 |
| loops that filter/transform | streams | 8 | §7 |
| returning `null` | `Optional` | 8 | §8 |
| a POJO with 60 lines of getters/`equals` | **`record`** | 16 | §2 |
| `String` concat in a loop | `StringBuilder`, or `String.join`/`Collectors.joining` | always | §3 |
| escaped multi-line SQL/JSON strings | **text blocks** (`"""`) | 15 | §3 |
| `if/else if` chains on type | **pattern matching** for `instanceof`/`switch` | 16 / 21 | §2 |
| `switch` with `break` everywhere | **switch expressions** (`->`, `yield`) | 14 | §2 |
| `Map<String, List<Question>> m = new HashMap<>();` | `var m = new HashMap<String, List<Question>>();` | 10 | §2 |
| an abstract class + "don't subclass this" comment | **`sealed`** | 17 | §2 |
| a thread pool sized to your CPU count for blocking I/O | **virtual threads** | 21 | §12 |
| `new ArrayList<>(Arrays.asList(...))` for a constant | `List.of(...)` | 9 | §5 |
| `Collections.unmodifiableList(x)` | `List.copyOf(x)` | 10 | §5 |
| CMS or Parallel GC | **G1** (default), or ZGC for low latency | 9 / 15+ | §13 |
| a fat JAR + a full JRE | jlink/jpackage, or a slim container image | 9+ | §14 |
| `System.getSecurityManager()` | it's gone — security is now build/runtime discipline | 17 dep / 24 | §16 |

**The through-line:** Java has been getting *less ceremonial*. Records, `var`, text blocks, pattern matching, and switch expressions all exist to delete boilerplate that never carried meaning. If you find yourself writing 40 lines to express one idea, there's now probably a shorter way — that's the instinct to rebuild.

---

# Part 2 — Modern data modelling

The biggest language shift since Java 8, and the one that most changes how code *looks*.

## 2.1 `var` — local type inference (Java 10)

```java
var questions = new ArrayList<Question>();          // inferred: ArrayList<Question>
var byConcept = new HashMap<Long, List<Question>>();
for (var q : questions) { ... }
var total = questions.size();                        // int
```

`var` is **not** dynamic typing. The type is inferred at compile time and fixed forever; `var x = 5; x = "hi";` won't compile. It's local variables only — not fields, not parameters, not return types.

> **The tell:** use `var` when the type is obvious from the right-hand side (`var q = new Question();`) or when the type name is noise (`var e : map.entrySet()`). *Don't* use it when it hides something you need to know (`var result = service.process();` — process what into what?). Readability is the only criterion.

## 2.2 Records (Java 16) — the boilerplate killer

A `record` is an immutable data carrier. This:

```java
public record QuestionSummary(Long id, String text, Status status) {}
```

...generates a canonical constructor, private final fields, accessors (`id()`, not `getId()`), and correct `equals`, `hashCode`, and `toString`. It replaces ~60 lines of POJO with one.

```java
var s = new QuestionSummary(1L, "What is force?", Status.APPROVED);
s.id();                      // 1  — note: no "get" prefix
s.equals(new QuestionSummary(1L, "What is force?", Status.APPROVED));  // true — value equality
```

**Validation** via a compact constructor:

```java
public record QuestionSummary(Long id, String text, Status status) {
    public QuestionSummary {                       // no parameter list, no assignment
        Objects.requireNonNull(text);
        if (text.isBlank()) throw new IllegalArgumentException("text required");
    }
    public boolean isApproved() { return status == Status.APPROVED; }   // methods are fine
}
```

Records are **final**, can't extend anything, and their fields are final. That's the point — they're *values*, not entities.

> **The tell:** record for anything that's **data with no identity**: DTOs, API responses, projections, value objects, map keys, method return tuples. **Not** for JPA entities — those are mutable, need a no-arg constructor, and have identity semantics the ORM manages (see the JPA reference). Practiq's **projections** (§7, and the data-access primer's 3.8) are ideal records; `Question` itself is not.

## 2.3 Sealed types (Java 17) — closed hierarchies

`sealed` says "these and only these may extend me," enforced by the compiler:

```java
public sealed interface Answer permits McqAnswer, FreeTextAnswer, NumericAnswer {}

public record McqAnswer(int selectedOption)      implements Answer {}
public record FreeTextAnswer(String text)        implements Answer {}
public record NumericAnswer(double value, String unit) implements Answer {}
```

The payoff arrives in §2.5: the compiler knows the list is complete, so a `switch` over it needs no `default` and **fails to compile if you add a fourth type and forget to handle it**. That's an entire class of bug converted into a build error.

> **The tell:** sealed + records = an "algebraic data type" — a fixed set of shapes, each carrying its own data. Perfect for results, states, and command/event types. Reach for it whenever you'd otherwise write a comment saying "only these three implementations exist."

## 2.4 Switch expressions (Java 14)

Old switch was a *statement* riddled with fall-through bugs. New switch is an **expression** that returns a value:

```java
// old — verbose, and one missing `break` is a live bug
int points;
switch (difficulty) {
    case EASY: points = 1; break;
    case MEDIUM: points = 3; break;
    case HARD: points = 5; break;
    default: throw new IllegalStateException();
}

// new — an expression, no fall-through, exhaustive
int points = switch (difficulty) {
    case EASY   -> 1;
    case MEDIUM -> 3;
    case HARD   -> 5;
};                                     // no default needed: enum is exhaustive

// multi-statement arms use yield
String label = switch (status) {
    case APPROVED -> "Live";
    case DRAFT, PENDING -> {           // multiple labels
        log.debug("not live yet");
        yield "In review";
    }
};
```

The `->` form **never falls through**. Over an enum, the compiler checks exhaustiveness — add a `Status` value and every switch missing it fails to compile.

## 2.5 Pattern matching (Java 16 → 21)

**`instanceof` patterns** (16) kill the cast dance:

```java
// old
if (obj instanceof Question) {
    Question q = (Question) obj;
    return q.getStatus();
}
// new — bind and test in one
if (obj instanceof Question q) {
    return q.getStatus();
}
// and it composes
if (obj instanceof Question q && q.getStatus() == APPROVED) { ... }
```

**Switch patterns + record deconstruction** (21) is where it all comes together:

```java
String describe(Answer a) {
    return switch (a) {
        case McqAnswer(int option)            -> "Chose option " + option;
        case FreeTextAnswer(String text)      -> "Wrote " + text.length() + " chars";
        case NumericAnswer(double v, String u) when v < 0 -> "Invalid negative " + u;
        case NumericAnswer(double v, String u) -> v + " " + u;
    };   // exhaustive: Answer is sealed → no default → add a 4th type and this won't compile
}
```

Note three things at once: the **record deconstruction** (`McqAnswer(int option)` pulls the field straight out), the **`when` guard**, and the **exhaustiveness** that `sealed` bought you. Also `case null ->` is now allowed, so switch no longer NPEs on null by surprise.

> **The tell:** sealed interface + records + switch patterns is the modern way to model "one of N shapes." It's what other languages call sum types. If Practiq's `Question` subtypes ever genuinely diverge (MCQ vs free-response), this — not inheritance — is the first thing to reach for at the domain level.

## 2.6 Text blocks (Java 15)

```java
// old
String sql = "select q.id, q.status\n" +
             "from question q\n" +
             "where q.concept_id = ?\n";

// new
String sql = """
        select q.id, q.status
        from question q
        where q.concept_id = ?
        """;
```

Incidental indentation is stripped automatically (based on the closing delimiter's position). Perfect for JPQL/SQL, JSON fixtures, and test data — you already saw this in the JPA reference's queries.

---

# Part 3 — Strings and text

## 3.1 Immutability and the pool

`String` is **immutable** — every "modification" returns a new object. This is why it's thread-safe, cacheable, and safe as a map key. String *literals* are interned in a pool, so `"abc" == "abc"` is true, but `new String("abc") == "abc"` is false — which is the entire reason **you compare strings with `.equals()`, never `==`**.

```java
String a = "abc", b = "abc";
a == b;                      // true — same pooled literal
new String("abc") == a;      // false — different object
new String("abc").equals(a); // true  — always use this
```

## 3.2 StringBuilder — and when it actually matters

```java
// BAD in a loop — each += creates a new String; O(n²) copying
String out = "";
for (Question q : questions) out += q.getText() + ", ";

// GOOD
StringBuilder sb = new StringBuilder();
for (Question q : questions) sb.append(q.getText()).append(", ");
String out = sb.toString();

// BETTER — say what you mean
String out = questions.stream().map(Question::getText).collect(Collectors.joining(", "));
String csv = String.join(", ", List.of("a", "b", "c"));
```

Nuance worth having: the compiler **already** rewrites simple concatenation (`"a" + b + "c"`) efficiently — in one expression, `+` is fine and clearer. The problem is **concatenation in a loop**, where each iteration allocates a fresh String and copies everything so far. `StringBuffer` is the synchronised, obsolete twin of `StringBuilder`; you will never need it.

## 3.3 The methods worth knowing

```java
"  hi  ".strip();                    // 11+ — Unicode-aware; trim() is the old ASCII-only one
"".isBlank();                        // 11 — true for whitespace-only; isEmpty() is length==0
"ab".repeat(3);                      // 11 → "ababab"
"a\nb".lines();                      // 11 → Stream<String>
"x".formatted(1);                    // 15 — instance form of String.format
String.join(", ", parts);            // 8
"a,b,,c".split(",");                 // note: trailing empties dropped by default
"text".chars();                      // IntStream of code points
```

Formatting: `String.format("%s scored %d (%.1f%%)", name, score, pct)` — or `"%s".formatted(x)`.

## 3.4 Regex, briefly

```java
private static final Pattern SPEC_REF = Pattern.compile("^(\\d+)\\.(\\d+)(?:\\.(\\d+))?$");

Matcher m = SPEC_REF.matcher("4.2.1");
if (m.matches()) {
    String major = m.group(1), minor = m.group(2), point = m.group(3);  // may be null
}
```

**Compile the `Pattern` once** into a static final field — `String.matches()`/`replaceAll()` recompile the regex on every call, which is a real cost in a loop.

> **The tell:** `+` for simple concatenation, `StringBuilder` for loops, `Collectors.joining`/`String.join` when you're building from a collection (clearest of all), text blocks for anything multi-line.

---

# Part 4 — Generics

The thing most self-taught Java devs half-know. Two ideas carry it.

## 4.1 What they are, and erasure

Generics are **compile-time** type safety. `List<String>` guarantees the compiler rejects `list.add(42)`. But at runtime, the type parameter is **erased** — a `List<String>` and a `List<Integer>` are both just `List`. Consequences you'll actually hit:

```java
list instanceof List<String>     // won't compile — the type isn't there at runtime
new T[10]                        // can't — no runtime type
List<String>.class               // doesn't exist
void f(List<String> a) {}
void f(List<Integer> a) {}       // won't compile — same erasure, same signature
```

This is why you occasionally see `Class<T>` passed as a parameter (`em.find(Question.class, id)`) — it's how an API recovers at runtime what erasure threw away.

## 4.2 Wildcards — and PECS

```java
List<Question> qs = ...;
List<? extends Question> readable = qs;    // producer — you can READ Questions out
List<? super Question> writable = ...;     // consumer — you can WRITE Questions in
List<?> unknown = qs;                      // unknown type — read Objects only
```

The mnemonic is **PECS — Producer `extends`, Consumer `super`**:

- If the parameter **produces** values you'll read → `? extends T`. You can read `T`s; you can't add (the compiler doesn't know which subtype it really is).
- If the parameter **consumes** values you'll write → `? super T`. You can add `T`s; reads give you `Object`.

```java
// produces Questions for us to read → extends
double averageDifficulty(Collection<? extends Question> qs) { ... }

// consumes the Questions we write → super
void drainInto(Collection<? super Question> sink) { sink.addAll(myQuestions); }
```

You've seen PECS in the wild without noticing: `Comparator.comparing(Function<? super T, ? extends U>)` — it *consumes* a `T` and *produces* a `U`.

## 4.3 Generic methods and bounds

```java
// generic method — <T> declares the parameter
static <T> List<T> firstN(List<T> src, int n) { return src.subList(0, min(n, src.size())); }

// bounded — T must be Comparable with itself
static <T extends Comparable<T>> T max(List<T> list) { ... }

// multiple bounds
static <T extends Comparable<T> & Serializable> void f(T t) { ... }

// generic class
public class Cache<K, V> {
    private final Map<K, V> map = new HashMap<>();
    public V get(K key) { return map.get(key); }
}
```

> **The tell:** don't reach for generics because they look sophisticated — reach when you're genuinely writing something type-agnostic (a container, a repository interface, a utility). Consuming generics correctly (declaring `List<Question>`, understanding a library signature with `? extends`) is 95% of what you need; *writing* deeply generic APIs is the other 5% and mostly a library author's job. Never use a **raw type** (`List` instead of `List<Question>`) — it disables all checking and is a legacy artifact.

---

# Part 5 — Collections

Covered properly in the **Java Collections reference** — the type hierarchy, every implementation with real costs, `HashMap` internals, sorting, `equals`/`hashCode`, immutability, streams, concurrency, and the decision table. The two-line summary:

- **Defaults:** `ArrayList`, `HashSet`, `HashMap`, `ArrayDeque`. Deviate only for a nameable reason (sorted → `TreeX`, insertion order → `LinkedX`, enum keys → `EnumX`, concurrent → `ConcurrentHashMap`).
- **Modern factories:** `List.of(...)`, `Set.of(...)`, `Map.of(...)` for constants (Java 9); `List.copyOf(x)` for defensive copies (Java 10); `stream().toList()` (Java 16).

---

# Part 6 — Lambdas and functional interfaces

## 6.1 The mechanics

A **functional interface** has exactly one abstract method. A lambda is an inline implementation of it.

```java
// this interface...
@FunctionalInterface interface Validator { boolean test(Question q); }
// ...is satisfied by
Validator v = q -> q.getText() != null;

// old: anonymous inner class          // new: lambda
Runnable r = new Runnable() {          Runnable r = () -> doWork();
    public void run() { doWork(); }
};
```

## 6.2 The built-in interfaces you must recognise

You'll see these constantly in library signatures — knowing them is how you *read* modern Java:

| Interface | Shape | Reads as |
|---|---|---|
| `Function<T,R>` | `R apply(T)` | transform |
| `Predicate<T>` | `boolean test(T)` | test |
| `Consumer<T>` | `void accept(T)` | side effect |
| `Supplier<T>` | `T get()` | produce/defer |
| `BiFunction<T,U,R>` | `R apply(T,U)` | two-arg transform |
| `UnaryOperator<T>` | `T apply(T)` | same-type transform |
| `BinaryOperator<T>` | `T apply(T,T)` | reduce/combine |
| `Runnable` | `void run()` | do it |
| `Comparator<T>` | `int compare(T,T)` | order |

Primitive variants exist to avoid boxing: `IntPredicate`, `ToIntFunction`, `IntSupplier`…

## 6.3 Method references — four kinds

```java
Question::getText            // instance method of an arbitrary object → q -> q.getText()
System.out::println          // instance method of a specific object   → x -> System.out.println(x)
Question::new                // constructor                            → () -> new Question()
Integer::parseInt            // static method                          → s -> Integer.parseInt(s)
```

> **The tell:** use a method reference when the lambda would just call one method (`q -> q.getText()` → `Question::getText`). Use a lambda when there's any logic. Keep lambdas to one expression — if it needs braces and three statements, extract a named method and reference it.

---

# Part 7 — Streams

## 7.1 The model

`source → intermediate operations (lazy) → terminal operation (triggers everything)`. Nothing runs until the terminal op — the pipeline is a *description* until then.

```java
List<String> texts = questions.stream()               // source
    .filter(q -> q.getStatus() == APPROVED)           // intermediate — lazy
    .sorted(comparing(Question::getDifficulty))       // intermediate — lazy
    .map(Question::getText)                           // intermediate — lazy
    .limit(10)                                        // intermediate — short-circuits
    .toList();                                        // terminal — NOW it runs
```

Laziness matters: `filter().findFirst()` stops at the first match rather than filtering everything.

## 7.2 The operations

**Intermediate:** `filter`, `map`, `flatMap` (flatten nested — `stream.flatMap(c -> c.getQuestions().stream())`), `distinct`, `sorted`, `limit`, `skip`, `peek` (debugging only), `takeWhile`/`dropWhile` (9).

**Terminal:** `forEach`, `toList` (16), `collect`, `reduce`, `count`, `anyMatch`/`allMatch`/`noneMatch`, `findFirst`/`findAny`, `min`/`max`, `sum`/`average` (on primitive streams).

## 7.3 Collectors — the workhorses

```java
// group
Map<Long, List<Question>> byConcept = questions.stream()
    .collect(groupingBy(Question::getConceptId));

// group + downstream: count per concept
Map<Long, Long> counts = questions.stream()
    .collect(groupingBy(Question::getConceptId, counting()));

// group + map values
Map<Long, List<String>> textsByConcept = questions.stream()
    .collect(groupingBy(Question::getConceptId, mapping(Question::getText, toList())));

// partition on a boolean
Map<Boolean, List<Question>> split = questions.stream()
    .collect(partitioningBy(q -> q.getStatus() == APPROVED));

// to a map — watch for duplicate keys
Map<Long, Question> byId = questions.stream()
    .collect(toMap(Question::getId, identity()));                 // throws on dup key
Map<Long, Question> safe = questions.stream()
    .collect(toMap(Question::getId, identity(), (a, b) -> a));    // merge fn: keep first

// join
String csv = questions.stream().map(Question::getText).collect(joining(", ", "[", "]"));

// stats in one pass
IntSummaryStatistics stats = questions.stream().mapToInt(Question::getDifficulty).summaryStatistics();
stats.getAverage(); stats.getMax(); stats.getCount();
```

## 7.4 Primitive streams and `flatMap`

```java
IntStream.rangeClosed(1, 10).sum();
questions.stream().mapToInt(Question::getDifficulty).average().orElse(0);

// flatMap — one-to-many, flattened
List<Question> all = concepts.stream()
    .flatMap(c -> c.getQuestions().stream())
    .toList();
```

## 7.5 Parallel streams — the honest note

`.parallelStream()` splits work across the common ForkJoin pool. It is **rarely** a win: it needs a large N, a cheap-to-split source (arrays/`ArrayList`, not `LinkedList`), and independent, CPU-bound, side-effect-free work. It can be actively *harmful* — it monopolises a shared pool, and inside a web server (where every request is already a thread) it usually makes throughput worse, not better.

> **The tell:** streams for **transformation pipelines** — filter/map/group/collect. A plain `for` loop for simple iteration with side effects; it's clearer and often faster. Never mutate external state from a stream (`forEach(x -> list.add(x))` is a `collect` written badly). Don't use `parallelStream()` without measuring. And **don't stream a database** — an N+1 hidden in a `.map(q -> q.getConcept())` is still an N+1 (see the data-access primer).

---

# Part 8 — Optional

A container that is either present or empty. Its **purpose is return types** — to make "might be absent" visible in the signature instead of a lurking `null`.

```java
Optional<Question> found = repository.findById(id);

// good — declarative
String text = found.map(Question::getText).orElse("(none)");
found.ifPresent(q -> log.info("found {}", q.getId()));
Question q = found.orElseThrow(() -> new NotFoundException(id));
Optional<Question> approved = found.filter(x -> x.getStatus() == APPROVED);

// bad — you've just reinvented a null check with extra steps
if (found.isPresent()) { Question x = found.get(); ... }
```

`orElse(x)` always evaluates `x`; `orElseGet(() -> expensive())` only evaluates on absence — use `orElseGet` when the fallback costs anything.

> **The tell:** `Optional` as a **return type** for "might not exist." **Not** as a field (it isn't serialisable and bloats objects), **not** as a parameter (overload or accept null instead), **never** call `.get()` without checking. And note `Optional.of(null)` throws — use `ofNullable`.

---

# Part 9 — Date and time (`java.time`, Java 8)

The old `Date`/`Calendar`/`SimpleDateFormat` API was mutable, not thread-safe (a shared `SimpleDateFormat` is a classic production bug), and had months numbered from zero. `java.time` replaced all of it: **immutable, thread-safe, and explicit about what it represents.**

## 9.1 Picking the right type

| Type | Is | Use for |
|---|---|---|
| **`Instant`** | a point on the UTC timeline | **timestamps** — created/updated, logs, events |
| `LocalDate` | a date, no time, no zone | birthdays, exam dates |
| `LocalTime` | a time, no date, no zone | opening hours |
| `LocalDateTime` | date+time, **no zone** — *not* a real instant | wall-clock in an unspecified place |
| `ZonedDateTime` | date+time+zone | a time as a human in a place experiences it |
| `OffsetDateTime` | date+time+UTC offset | wire formats, DB `TIMESTAMPTZ` |
| `Duration` | time-based amount (seconds/nanos) | timeouts, elapsed |
| `Period` | date-based amount (y/m/d) | "3 months from now" |

The distinction that catches everyone: **`LocalDateTime` is not a moment in time.** "2026-07-17 14:00" is a different instant in Leeds than in Tokyo. If you're recording *when something happened*, use **`Instant`**. Use `LocalDateTime` only for genuinely zoneless wall-clock values.

```java
Instant now = Instant.now();                         // UTC timestamp — store this
LocalDate today = LocalDate.now();
LocalDate exam = LocalDate.of(2026, 6, 15);          // months are 1-based now, thank god

// arithmetic — immutable, returns new objects
LocalDate deadline = exam.minusWeeks(2);
Instant expiry = now.plus(Duration.ofHours(24));

// comparison
exam.isBefore(today); exam.isAfter(today);

// zones — conversion is explicit
ZonedDateTime leeds = now.atZone(ZoneId.of("Europe/London"));
Instant back = leeds.toInstant();

// durations
Duration elapsed = Duration.between(start, Instant.now());
elapsed.toMillis();
Period age = Period.between(birthDate, today);       // years/months/days

// formatting — DateTimeFormatter IS thread-safe (unlike SimpleDateFormat)
private static final DateTimeFormatter FMT = DateTimeFormatter.ofPattern("dd/MM/yyyy");
String s = exam.format(FMT);
LocalDate parsed = LocalDate.parse("15/06/2026", FMT);
LocalDate iso = LocalDate.parse("2026-06-15");       // ISO by default
```

> **The tell:** **store and transmit `Instant`** (UTC), convert to a zone only at the edges for display. In Postgres, `Instant`/`OffsetDateTime` ↔ `TIMESTAMPTZ`, `LocalDate` ↔ `DATE`. Never use `Date`/`Calendar` in new code — and if you meet a shared `SimpleDateFormat` field in old code, that's a thread-safety bug, not a style issue.

---

# Part 10 — Exceptions

## 10.1 The hierarchy

```
Throwable
├── Error                    — JVM-level, don't catch (OutOfMemoryError, StackOverflowError)
└── Exception
    ├── RuntimeException     — UNCHECKED — programming errors (NPE, IllegalArgument, IllegalState)
    └── everything else      — CHECKED — compiler forces catch-or-declare (IOException, SQLException)
```

## 10.2 Checked vs unchecked — the argument you should know

Checked exceptions are a Java peculiarity: the compiler *forces* you to handle or declare them. The modern consensus (and what Spring/Micronaut/Hibernate all do) is that they were **mostly a mistake** — they don't compose with lambdas/streams, they leak implementation details up the call stack, and in practice they produce the worst possible outcome: `catch (Exception e) {}`. Hibernate wraps `SQLException` (checked) into unchecked `PersistenceException` for exactly this reason.

> **The tell:** for new code, prefer **unchecked** exceptions. Use checked only when the caller can *genuinely recover* and you want to force the decision — which is rare. Never swallow (`catch (Exception e) {}`), never catch `Throwable`/`Error`, and always preserve the cause: `throw new AppException("loading question " + id, e)`.

## 10.3 The modern syntax

```java
// try-with-resources (7) — auto-closes anything AutoCloseable, in reverse order
try (var conn = dataSource.getConnection();
     var ps = conn.prepareStatement(sql)) {
    ...
}   // closed automatically, even on exception

// multi-catch (7)
try { ... }
catch (IOException | SQLException e) { log.error("failed", e); throw new AppException(e); }

// custom, unchecked
public class QuestionNotFoundException extends RuntimeException {
    public QuestionNotFoundException(Long id) { super("No question with id " + id); }
}
```

`finally` still runs for cleanup that isn't a resource. Note a `return` in `finally` swallows exceptions — never do it.

---

# Part 11 — Equality, immutability, object basics

## 11.1 `==` vs `equals`

`==` compares **references** (identity) for objects, **values** for primitives. `.equals()` compares **content** — if the class overrides it. That's the whole `String` story from §3.1.

Boxing trap worth knowing:
```java
Integer a = 127, b = 127;    a == b;   // true  — Integer cache (-128..127)
Integer c = 128, d = 128;    c == d;   // false — outside the cache
```
Never `==` on boxed types. Use `.equals()` or unbox to `int`.

## 11.2 The `equals`/`hashCode` contract

Covered in the collections reference — the rules, the mutation trap, and why `HashMap` depends on it. The practical: override both together, use `Objects.equals`/`Objects.hash`, or **use a record and get it free**.

## 11.3 Immutability

```java
public final class Money {                       // final: no subclass can break it
    private final BigDecimal amount;             // final fields
    private final List<String> tags;

    public Money(BigDecimal amount, List<String> tags) {
        this.amount = amount;
        this.tags = List.copyOf(tags);           // defensive copy IN
    }
    public List<String> tags() { return tags; }  // already immutable — safe to return
}
```
Or just: `public record Money(BigDecimal amount, List<String> tags) {}` with a compact constructor doing the `copyOf`. Immutable objects are automatically thread-safe, safe as map keys, and can't be corrupted by a caller — reach for them by default.

## 11.4 Numbers

`double`/`float` are **binary** floating point: `0.1 + 0.2 != 0.3`. Never use them for money or anything requiring exactness — use **`BigDecimal`** (and its `compareTo`, not `equals`, since `2.0` and `2.00` differ in scale). Integer overflow is silent (`Integer.MAX_VALUE + 1` wraps negative); `Math.addExact` throws instead.

---

# Part 12 — Concurrency

## 12.1 The layers

You almost never touch raw `Thread` anymore. The progression:

```java
// 1. raw threads — you manage lifecycle. Don't.
new Thread(() -> work()).start();

// 2. executors (Java 5) — a pool you submit tasks to
ExecutorService pool = Executors.newFixedThreadPool(8);
Future<Integer> f = pool.submit(() -> compute());
int result = f.get();                          // BLOCKS
pool.shutdown();

// 3. CompletableFuture (8) — composable async, no blocking
CompletableFuture.supplyAsync(() -> fetchQuestions())
    .thenApply(this::score)
    .thenAccept(this::save)
    .exceptionally(e -> { log.error("failed", e); return null; });

// combining
var a = CompletableFuture.supplyAsync(() -> loadConcept());
var b = CompletableFuture.supplyAsync(() -> loadQuestions());
a.thenCombine(b, (concept, qs) -> merge(concept, qs));
```

## 12.2 Virtual threads (Java 21) — the big one

Platform threads are OS threads: ~1 MB of stack each, expensive to create, so you pool them. That's why every server was built around "a small pool of threads, don't block them."

**Virtual threads** are lightweight threads managed by the JVM, not the OS. They're cheap enough to create *millions*, and when one blocks on I/O the JVM **unmounts** it from its carrier thread and runs something else. The blocking call still *looks* blocking in your code — the JVM handles the yield.

```java
// one virtual thread per task — this is now fine with 100,000 tasks
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 100_000).forEach(i ->
        executor.submit(() -> { callSlowService(i); return null; }));
}   // close() waits for all to finish

Thread.startVirtualThread(() -> work());     // one-off
```

**Why it matters:** it makes simple blocking code scale like reactive code. The whole reason reactive/async frameworks existed was "don't block the thread" — virtual threads largely dissolve that constraint for I/O-bound work. For a Micronaut service doing DB and HTTP calls, this is the headline Java 21 feature.

Two caveats: virtual threads don't help **CPU-bound** work (you still only have N cores), and `synchronized` blocks could *pin* a virtual thread to its carrier in Java 21 (largely fixed by Java 24 — prefer `ReentrantLock` if you're on 21 and holding a lock across I/O).

## 12.3 The primitives you should still know

```java
private final AtomicInteger counter = new AtomicInteger();   // lock-free counter
counter.incrementAndGet();

private final ReentrantLock lock = new ReentrantLock();      // explicit lock
lock.lock(); try { ... } finally { lock.unlock(); }

private volatile boolean running = true;   // visibility across threads (NOT atomicity)

synchronized void f() { ... }              // the old blunt instrument
```

`volatile` guarantees *visibility*, not atomicity — `volatile int i; i++` is still a race (read-modify-write). Use `AtomicInteger` for that.

> **The tell:** the best concurrency is **no shared mutable state** — immutable objects and message passing. When you must share: `ConcurrentHashMap`/`Atomic*` for simple state, `ReentrantLock` when you need real locking, virtual threads for I/O-bound task-per-request. And in a Micronaut app, most concurrency is the framework's problem, not yours — your job is to not introduce shared mutable state into it.

---

# Part 13 — The JVM: memory and garbage collection

## 13.1 The memory model

| Region | Holds | Notes |
|---|---|---|
| **Heap** | all objects | GC'd; `-Xms`/`-Xmx` set it |
| ├─ Young gen (Eden + survivors) | new objects | most die here — **minor GC**, fast |
| └─ Old gen (tenured) | long-lived survivors | **major/full GC**, slower |
| **Metaspace** | class metadata | native memory, grows (replaced PermGen in Java 8) |
| **Stack** (per thread) | frames, locals, refs | `StackOverflowError` when too deep |
| Code cache | JIT-compiled native code | |

## 13.2 How GC works

The **generational hypothesis**: most objects die young. So the heap is split, and the young gen is collected often and cheaply (copying the few survivors), while the old gen is collected rarely and expensively. Collection is **mark-and-sweep** in principle: from GC roots (stack refs, statics), mark everything reachable; whatever isn't reachable is garbage. Objects get promoted to old gen after surviving a few young collections.

You do **not** free memory. `System.gc()` is a suggestion — never call it. A "memory leak" in Java is not unfreed memory; it's **an unintentional reference** keeping an object reachable — a static collection that grows forever, a cache with no eviction, an unclosed resource, a listener never deregistered.

## 13.3 The collectors

| Collector | Shape | Use when |
|---|---|---|
| **Serial** | single-threaded | tiny heaps, containers with 1 CPU |
| **Parallel** | multi-threaded, stop-the-world | batch jobs — max throughput, pauses fine |
| **G1** *(default since 9)* | region-based, mostly concurrent, pause-target | **the default; leave it alone** |
| **ZGC** | concurrent, sub-millisecond pauses, huge heaps | latency-critical, big heaps |
| **Shenandoah** | concurrent, low pause | similar niche to ZGC |
| **Epsilon** | does nothing | testing only |

```bash
-Xms512m -Xmx512m               # set both equal in a container — avoids resize churn
-XX:+UseG1GC                    # default anyway
-XX:MaxGCPauseMillis=200        # G1's pause target (a goal, not a guarantee)
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp
-Xlog:gc*:file=gc.log           # GC logging (9+; replaced -XX:+PrintGCDetails)
```

## 13.4 The JVM in a container — the Practiq-relevant bit

Historically the JVM read the *host's* RAM and sized its heap to a fraction of that — catastrophic in a container limited to 512 MB, because the JVM would happily plan for 8 GB and get OOM-killed by the kernel. Modern JVMs (10+) are **container-aware**: they read cgroup limits (see the Docker primer §2.3).

```bash
-XX:MaxRAMPercentage=75.0       # use 75% of the CONTAINER limit — the modern way
```

Prefer `MaxRAMPercentage` over a hard `-Xmx` in containers, and remember the JVM uses **more than the heap** (metaspace, thread stacks, code cache, direct buffers) — so 75% leaves headroom for the rest. "Container OOM-killed but heap looked fine" is almost always this.

> **The tell:** don't tune GC. Set a sensible heap (`MaxRAMPercentage` in containers), leave **G1** alone, enable GC logging and heap-dump-on-OOM, and only reach for ZGC if you have *measured* pause-time problems. Most "GC problems" are allocation problems — an N+1 loading 10,000 entities, or an unbounded cache.

---

# Part 14 — Packaging: classpath, modules, jars

## 14.1 The classpath

The classpath is the list of jars/directories the JVM searches for classes. It's flat, order-dependent, and has no notion of versions — hence **"jar hell"**: two libraries wanting different versions of a third, and the winner is whoever appears first. Build tools (Gradle/Maven) manage this via dependency resolution; that's most of what they do.

## 14.2 The module system (Java 9, JPMS)

Modules add real encapsulation: a `module-info.java` declares what a module *requires* and *exports*; anything not exported is genuinely inaccessible, even by reflection.

```java
module com.practiq.api {
    requires java.sql;
    requires io.micronaut.core;
    exports com.practiq.api.domain;      // only this package is public
}
```

**The honest take:** JPMS modularised the JDK itself (which is why `jlink` can build a runtime with only the modules you need), but **most applications never modularise**. Spring/Micronaut apps typically run on the classpath. You will meet JPMS mostly as: (a) the reason `jlink`/`jpackage` work, and (b) the source of `InaccessibleObjectException` / "illegal reflective access" errors when a library reflects into JDK internals — fixed with `--add-opens`.

> **The tell:** don't modularise a Micronaut app for its own sake. Do know `--add-opens` exists for when a library needs deep reflection into the JDK.

## 14.3 Packaging options

| Tool | Produces | Use for |
|---|---|---|
| `jar` | a plain jar | libraries |
| fat/shadow jar (Gradle Shadow, Micronaut's `shadowJar`) | one jar with all deps | **the usual for a service** |
| `jlink` | a custom JRE with only needed modules | small runtime images |
| `jpackage` | a native installer (.deb/.msi/.dmg) | desktop apps |
| **GraalVM native-image** | a native binary, no JVM | fast startup, low memory — Micronaut's speciality |
| a container image | see the Docker primer | **how Practiq actually ships** |

GraalVM native-image is worth naming: it AOT-compiles to a native binary with millisecond startup and a fraction of the memory — the trade being a slow build, no runtime reflection unless configured, and no JIT peak-performance ramp. Micronaut's compile-time DI (no runtime reflection) is *designed* for this, which is why native-image is far smoother there than in a classic Spring app.

---

# Part 15 — Files and I/O

Use **NIO.2** (`java.nio.file`, Java 7) — `Path`/`Files`, not the legacy `File`.

```java
Path p = Path.of("/data/questions.json");                  // 11+ (Paths.get is the old form)

String content = Files.readString(p);                       // 11
List<String> lines = Files.readAllLines(p);
Files.writeString(p, json);                                 // 11
byte[] bytes = Files.readAllBytes(p);

Files.exists(p); Files.size(p); Files.createDirectories(p.getParent());
Files.copy(src, dest, StandardCopyOption.REPLACE_EXISTING);
Files.delete(p); Files.deleteIfExists(p);

// stream a big file — don't load it all
try (Stream<String> lines = Files.lines(p)) {              // MUST close — it holds a handle
    lines.filter(l -> l.startsWith("Q")).forEach(this::process);
}

// walk a tree
try (Stream<Path> walk = Files.walk(base)) {
    walk.filter(Files::isRegularFile).forEach(...);
}
```

Streams and readers are `AutoCloseable` → **always try-with-resources** (§10.3). And note `Files.lines`/`Files.walk` return streams backed by open handles — closing them is not optional.

**HTTP client** (Java 11) — no more third-party client for simple calls:

```java
HttpClient client = HttpClient.newHttpClient();
HttpRequest req = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/x"))
    .header("Accept", "application/json")
    .timeout(Duration.ofSeconds(10))
    .GET().build();
HttpResponse<String> resp = client.send(req, HttpResponse.BodyHandlers.ofString());
resp.statusCode(); resp.body();
```
It's HTTP/2 by default and has an async form (`sendAsync` → `CompletableFuture`). Java 26 added HTTP/3 support.

---

# Part 16 — Security fundamentals

Not a security course — the Java-specific surface you should recognise.

## 16.1 The `SecurityManager` is gone

If you remember `SecurityManager` and policy files: that model is **dead** (deprecated in 17, disabled/removed by 24). It never worked well and almost nobody used it correctly. Modern Java security is **dependency hygiene, input validation, and runtime configuration** — not a sandbox inside the JVM.

## 16.2 What actually matters

**Dependencies are your biggest attack surface.** Log4Shell was a logging library. Use `gradle dependencyCheck`/OWASP Dependency-Check or GitHub Dependabot, keep things patched, and pin versions.

**Deserialization is genuinely dangerous.** Java's native serialization (`ObjectInputStream`) can execute code during deserialization — it's the root of a large family of RCE exploits. **Never deserialize untrusted data with native Java serialization.** Use JSON (Jackson) with explicit types. There's a serialization filter (`ObjectInputFilter`, Java 9) if you're stuck with it.

**Injection** — always parameterise. `PreparedStatement` with `?` (data-access primer §1.3); JPQL/Criteria with bound parameters. String-concatenated SQL is the bug.

**Secrets** — never in code, never in an image layer (Docker primer §7.6), never in logs. Env vars or a secret manager.

## 16.3 The crypto APIs you'd actually touch

```java
// password hashing — NEVER a plain hash. Use bcrypt/argon2/PBKDF2 via a library.
// Random for security — SecureRandom, not Random
SecureRandom rnd = new SecureRandom();
byte[] salt = new byte[16]; rnd.nextBytes(salt);

// hashing (for integrity/checksums, NOT passwords)
MessageDigest md = MessageDigest.getInstance("SHA-256");
byte[] digest = md.digest(data);

// Base64 (encoding, NOT encryption — it hides nothing)
String enc = Base64.getEncoder().encodeToString(bytes);
```

`java.security` (`MessageDigest`, `SecureRandom`, `KeyStore`, `Signature`), `javax.crypto` (`Cipher`, `KeyGenerator`), TLS via `SSLContext`/JSSE, and `keytool` for keystores. Java 24 added quantum-resistant algorithms (ML-KEM/ML-DSA); Java 27 is slated to add post-quantum hybrid key exchange for TLS 1.3.

> **The tell:** never roll your own crypto, never use `Random` where security matters (`SecureRandom`), never store a password as anything but a slow salted hash (bcrypt/argon2), never native-deserialize untrusted input. 90% of real Java security is patching dependencies and parameterising queries.

---

# Part 17 — What changed in every release since Java 6

The full arc. **Bold = the ones that changed how code is written.**

## The old cadence (2006–2017)

| Version | Year | What landed |
|---|---|---|
| **6** | 2006 | scripting, JDBC 4, compiler API. Ancient. |
| **7** | 2011 | **try-with-resources**, multi-catch, diamond `<>`, strings in switch, **NIO.2** (`Path`/`Files`), numeric underscores |
| **8** | 2014 | **lambdas**, **streams**, **`java.time`**, `Optional`, default methods, method refs, PermGen→Metaspace. *The most important release ever; the reason so many stayed.* |
| **9** | 2017 | **JPMS modules**, `List.of`/`Set.of`/`Map.of`, JShell, **G1 becomes default**, `jlink`, private interface methods |

## The six-month cadence (2018 →)

| Version | Year | What landed |
|---|---|---|
| **10** | 2018 | **`var`**, `List.copyOf`, container-awareness, parallel full GC for G1 |
| **11** *(LTS)* | 2018 | **`HttpClient`**, `String.strip/isBlank/repeat/lines`, `Files.readString/writeString`, run a single `.java` file, ZGC (experimental), applets gone |
| 12 | 2019 | switch expressions (preview), Shenandoah, `Collectors.teeing` |
| 13 | 2019 | text blocks (preview), dynamic CDS |
| **14** | 2020 | **switch expressions (final)**, records (preview), **helpful NullPointerExceptions**, `instanceof` patterns (preview) |
| **15** | 2020 | **text blocks (final)**, sealed (preview), **ZGC production-ready**, Shenandoah production-ready |
| **16** | 2021 | **records (final)**, **`instanceof` patterns (final)**, `Stream.toList()`, Unix domain sockets, strong JDK internal encapsulation |
| **17** *(LTS)* | 2021 | **sealed classes (final)**, pattern matching for switch (preview), `SecurityManager` deprecated, new `RandomGenerator` API, applet API deprecated |
| 18 | 2022 | **UTF-8 the default charset**, simple web server, code snippets in Javadoc |
| 19 | 2022 | virtual threads (preview), structured concurrency (incubator), FFM (preview) |
| 20 | 2023 | more previews (scoped values, record patterns) |
| **21** *(LTS)* | 2023 | **virtual threads (final)**, **pattern matching for switch (final)**, **record patterns (final)**, sequenced collections, generational ZGC, string templates (preview) |
| 22 | 2024 | **FFM API (final)**, unnamed variables `_`, statements before `super()` (preview) |
| 23 | 2024 | Markdown Javadoc, ZGC generational by default, primitive patterns (preview) |
| 24 | 2025 | **quantum-resistant crypto** (ML-KEM/ML-DSA), `SecurityManager` permanently disabled, compact object headers, **virtual-thread pinning largely fixed**, AOT class loading (Leyden) |
| **25** *(LTS)* | 2025 | **compact source files & instance `main`** (simpler entry point), **scoped values (final)**, flexible constructor bodies, module import declarations, AOT caching, PEM API (preview), structured concurrency still previewing |
| **26** | Mar 2026 | **HTTP/3 in `HttpClient`**, lazy constants (formerly stable values), AOT object caching with any GC (incl. ZGC), G1 throughput +~15%, **"prepare to make final mean final"** (warnings for reflective final-field mutation), **Applet API removed**, structured concurrency (6th preview), primitive patterns (4th preview) |
| 27 | Sept 2026 | in flight — post-quantum hybrid key exchange for TLS 1.3 targeted |

**The arcs worth seeing rather than memorising:**

- **8 → the functional turn.** Lambdas/streams/`java.time`/`Optional`. Everything since builds on it.
- **9 → the cadence change.** Modules were the headline; the six-month release train was the actual revolution.
- **14–21 → the language-ergonomics arc.** Records, sealed, switch expressions, pattern matching, text blocks — each shipped incrementally through preview, and together they land as "modern Java" in **21**.
- **19–24 → Project Loom.** Virtual threads, structured concurrency, scoped values: making concurrency simple again.
- **Ongoing projects to recognise:** **Loom** (concurrency), **Panama** (native interop — FFM), **Valhalla** (value types — still not shipped, "ready when it's ready"), **Leyden** (startup/AOT), **Amber** (language ergonomics — records, patterns, etc.).

> **The tell — what you actually need:** if you know **Java 8 well**, the gap that matters is **records, sealed, switch expressions, pattern matching, text blocks, `var`, and virtual threads** (§2, §12). That's it. Everything else is API polish you'll pick up by IDE autocomplete.

---

# Part 18 — When to use what

**A. Java version.** Tell → LTS: production. Tell → non-LTS: you need one feature and can upgrade in 6 months. Default: **21 now, plan the move to 25.**

**B. Record vs class.** Tell → record: immutable data, no identity (DTOs, projections, value objects, results). Tell → class: mutable state, identity, framework requirements (JPA entities). Default: **record unless something forces a class.**

**C. Sealed + pattern matching vs inheritance.** Tell → sealed: a *fixed, known* set of shapes you'll switch over. Tell → open inheritance: genuine extensibility by others. Default: **sealed for closed domains — get compile-time exhaustiveness.**

**D. Stream vs loop.** Tell → stream: filter/map/group/collect pipelines. Tell → loop: simple iteration with side effects, early exit, or you need the index. Default: **loop for simple, stream for transformation. Never parallel without measuring.**

**E. `Optional` vs null.** Tell → `Optional`: a return type that might legitimately be absent. Tell → null: internal fields, parameters. Default: **`Optional` at API boundaries, never as a field.**

**F. Checked vs unchecked exception.** Tell → checked: caller can genuinely recover and you want to force it (rare). Tell → unchecked: everything else. Default: **unchecked.**

**G. Concurrency model.** Tell → virtual threads: I/O-bound, task-per-request. Tell → fixed pool: CPU-bound work. Tell → `CompletableFuture`: composing async stages. Default: **virtual threads for I/O on 21+; and prefer no shared mutable state to any of it.**

**H. `Instant` vs `LocalDateTime`.** Tell → `Instant`: a moment that happened (timestamps). Tell → `LocalDate`/`LocalTime`: a zoneless calendar/clock value. Tell → `ZonedDateTime`: display to a human in a place. Default: **`Instant` for storage, convert at the edges.**

**I. GC.** Tell → G1: everything. Tell → ZGC: measured pause problems on big heaps. Tell → Parallel: throughput batch jobs. Default: **G1, `MaxRAMPercentage` in containers, don't tune.**

**J. Packaging.** Tell → shadow jar in a container: the normal path (Practiq). Tell → native-image: startup/memory matter a lot and you're on Micronaut. Tell → jlink/jpackage: distributing a runtime/desktop app. Default: **shadow jar + container; native-image is a spike worth doing on Micronaut.**

---

# How to use and expand this document

- Suggested home: `docs/` in `practiq-api`, next to the data-access primer.
- Related docs: **Java Collections reference** (the deep dive for §5), **JPA & Hibernate reference** (annotations/classes), **Java data-access primer** (the whole DB stack), **Docker primer** (where §13.4 and §14.3 land in production).
- Good next deep-dives, any of which I can expand: **virtual threads properly** (pinning, structured concurrency, what it means for Micronaut and your JDBC pool — the pool becomes the bottleneck once threads aren't); **records + pattern matching applied to Practiq's domain** (modelling `Answer`/`Result` as sealed hierarchies); **the Java 21 → 25 upgrade** as a concrete checklist for `practiq-api`; **GraalVM native-image on Micronaut** with real numbers; **streams in depth** (custom collectors, laziness, spliterators); **GC tuning and reading a GC log** with a worked example; **JMH benchmarking** so performance claims are measured, not asserted.

*Caveat: release facts and version status verified July 2026 against current sources. The language/API behaviour is written from knowledge and is stable; if you want a specific JEP's detail or an exact Java 25/26 API signature verified, point me at it.*
