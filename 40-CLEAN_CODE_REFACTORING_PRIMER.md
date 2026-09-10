# Clean Code & Refactoring — A Primer №40

*The daily craft: writing code that the next person — usually you, in six months — can read, trust and change. Where №41 (Software Design & Patterns) and №42 (Architecture) work at the level of components and systems, this works at the level of **names, functions, classes and the small structural decisions you make hundreds of times a day.** Java-flavoured, Practiq-grounded.*

The premise that justifies all of it: **code is read far more often than it is written, and changed far more often than it is created.** Almost nothing you write is finished — it will be extended, debugged, ported, and read by someone reconstructing your intent under time pressure. So the goal isn't code that works (that's the floor); it's **code that is cheap to change**.

The single most useful measure, and the one to apply when style arguments get abstract: **how painful is the change I didn't anticipate?** A codebase where an unforeseen requirement means touching one well-named class is healthy. One where it means edits in nine files and a two-day archaeology session is not — regardless of how clever any individual file looks.

Contents:

- **Part 1** — the principles underneath
- **Part 2** — naming
- **Part 3** — functions
- **Part 4** — comments and documentation
- **Part 5** — classes and SOLID
- **Part 6** — the acronyms, with their nuance
- **Part 7** — error handling
- **Part 8** — code smells: the catalogue
- **Part 9** — refactoring: the moves and the discipline
- **Part 10** — code review
- **Part 11** — when to use what

## Smell index — symptom to fix

| You notice… | It's called | Fix | §|
|---|---|---|---|
| A method you must scroll to read | Long Method | Extract Method | §8.1, §9.2 |
| A class doing several unrelated jobs | Large Class / God Object | Extract Class | §8.1 |
| Five-plus parameters | Long Parameter List | Introduce Parameter Object | §8.1 |
| The same change hits many files | Shotgun Surgery | Move Method/Field together | §8.2 |
| One class changes for many reasons | Divergent Change | Extract Class (SRP) | §8.2 |
| A method obsessed with another class's data | Feature Envy | Move Method | §8.3 |
| `String status` where a type belongs | Primitive Obsession | Replace with Value Object | §8.1 |
| The same `switch` on type in several places | Switch Statements | Polymorphism / sealed + patterns | §8.3 |
| `a.getB().getC().doThing()` | Message Chains | Hide Delegate / tell-don't-ask | §8.4 |
| Copy-pasted logic | Duplicated Code | Extract Method/Class | §8.1 |
| A comment explaining *what* the code does | (a naming failure) | rename, extract | §4.2 |
| Flag parameter `doThing(true)` | Flag Argument | split into two methods | §3.4 |
| Deep `if` nesting | Arrow Code | Guard Clauses | §3.3 |
| A class that just passes calls through | Middle Man / Lazy Class | Inline Class | §8.4 |

---

# Part 1 — The principles underneath

Four ideas generate most of the specific rules. Internalise these and the rules become derivable rather than memorised.

**1. Reveal intent.** Code should say *what* it's doing and *why* it exists. The machine doesn't care about names; the reader does, and the reader is the constraint.

**2. Minimise what must be held in the head.** Human working memory is roughly seven items. A method with fifteen locals, four levels of nesting and three responsibilities exceeds it, so the reader cannot verify correctness — they can only hope. Every extraction and every good name is an act of offloading memory.

**3. Localise change.** Related things belong together; unrelated things belong apart. That's cohesion and coupling (№41) at the scale of a file. When one conceptual change requires an edit in one place, the design is right.

**4. Simplicity is the default; complexity must earn its place.** Every abstraction, layer and configuration option is a cost paid on every future read and change. Add them when a real need proves them, not in anticipation.

> **The tell — principles:** when two options look equally good, choose the one a competent stranger would understand faster. Cleverness is a cost you pay repeatedly; clarity is an asset that compounds.

---

# Part 2 — Naming

The highest-leverage skill in the document, and the cheapest to improve.

## 2.1 The rules

**Reveal intent.** `daysUntilExpiry` over `d`. `isEligibleForReview` over `flag`. If a name needs a comment to explain it, the name is wrong.

**Encode the unit or type when ambiguity is possible.** `timeoutMs`, `priceInPence`, `maxRetries`. The Mars Climate Orbiter was lost to a units mismatch; your bug will be smaller but the same shape.

**Booleans read as questions,** phrased positively: `isValid`, `hasAccess`, `canApprove`. Avoid negatives — `if (!isNotReady)` is a puzzle for no reason.

**Methods are verbs; classes are nouns.** `calculateDifficulty()`, `QuestionValidator`. Predicates start `is`/`has`/`can`; retrievals `get`/`find` (`find` conventionally implies "may be absent" and returns `Optional`).

**Consistency beats individual perfection.** Pick your `find`/`get`/`fetch` conventions and hold them across the codebase. Four synonyms with no distinction force the reader to check whether there is one.

**Length should scale with scope.** A loop index `i` is fine; a field used across a 200-line class is not. The wider the scope, the more the name must carry.

## 2.2 What to avoid

Single letters outside tiny scopes; abbreviations that aren't universal (`calc`, `mgr`, `proc` — though `id`, `url`, `http` are fine); noise words (`QuestionData`, `QuestionInfo`, `QuestionObject` — what distinguishes them?); type suffixes in names (`questionList` becomes a lie the day it's a `Set`); mental-mapping demands (`theList`, `temp2`); and misleading names — **a name that is actively wrong is worse than one that is merely vague**, because it produces confident incorrect assumptions.

```java
// Before
public List<Question> get(Long c, boolean f) { ... }

// After
public List<Question> findApprovedByConcept(Long conceptId, boolean includeArchived) { ... }
```

> **The tell — naming:** if you struggle to name something, you usually don't yet understand what it does — or it does more than one thing. The difficulty is diagnostic information about the design, not a vocabulary problem.

---

# Part 3 — Functions

## 3.1 Small, and one job

A function should do one thing, at one level of abstraction. The practical test: **can you extract a meaningful chunk and give it a name that isn't just a restatement of its code?** If yes, it was doing more than one thing.

"One level of abstraction" is the subtler half. This mixes levels:

```java
void publishQuestion(Question q) {
    validate(q);                                    // high level
    q.setStatus(APPROVED);                          // mid
    String sql = "UPDATE question SET ...";         // low — jarring
    jdbc.execute(sql);
    notificationService.notifyReviewers(q);         // high again
}
```

The reader has to keep zooming in and out. Each function should read as a coherent narrative at a single altitude.

## 3.2 Few parameters

Zero to two is comfortable, three is a stretch, four-plus is a smell. Long lists are hard to read, easy to misorder (especially with same-typed adjacent parameters), and usually signal that some arguments belong together:

```java
// Before
createQuestion(String stem, String optionA, String optionB, String optionC,
               int difficulty, String code, Long conceptId, boolean draft)

// After
createQuestion(QuestionDraft draft, Difficulty difficulty, Long conceptId)
```

## 3.3 Guard clauses over nesting

Deep nesting ("arrow code") forces the reader to track accumulated conditions. Invert and return early so the happy path is flat:

```java
// Before — the real work is buried three levels deep
public void approve(Question q) {
    if (q != null) {
        if (q.getStatus() == PENDING) {
            if (currentUser().canApprove()) {
                q.setStatus(APPROVED);
            }
        }
    }
}

// After — preconditions stated and dispatched, then the work
public void approve(Question q) {
    Objects.requireNonNull(q, "question");
    if (q.getStatus() != PENDING) throw new IllegalStateException("not pending: " + q.getStatus());
    if (!currentUser().canApprove()) throw new AccessDeniedException();

    q.setStatus(APPROVED);
}
```

Note the second version also *reports* why it did nothing, where the first silently succeeds having done nothing — a much worse bug.

## 3.4 No surprises

**Command-query separation:** a method either *does* something (command, returns void) or *answers* something (query, returns a value, no side effects). `getUser()` that also lazily creates and persists a user is a landmine.

**No flag arguments.** `render(true)` is unreadable at the call site and means the method has two behaviours. Split it: `renderSummary()` and `renderFull()`.

**No hidden side effects.** A function that modifies its arguments, mutates global state, or writes to disk when its name suggests a pure calculation will eventually cost someone a day.

> **The tell — functions:** read the method aloud as a sentence. "Approve: require the question, check it's pending, check permission, set approved." If it reads as one coherent sentence at one altitude, it's the right size. If you run out of breath or change subject, extract.

---

# Part 4 — Comments and documentation

## 4.1 The good ones

Comments that explain **why**: a non-obvious business rule, a workaround for an external bug, a deliberate performance trade-off, a decision that looks wrong but isn't.

```java
// Postgres SEQUENCE rather than IDENTITY: IDENTITY disables JDBC insert
// batching, which the seed pipeline depends on. See PRACTIQ_MASTER decision log.
@GeneratedValue(strategy = GenerationType.SEQUENCE)
```

That comment carries information the code cannot: the reasoning, the rejected alternative, and where to read more. Also valuable: API documentation on public contracts, warnings of consequences, and `TODO`s with a ticket reference.

## 4.2 The bad ones

**Comments that explain *what*.** `// increment the counter` above `counter++`. Noise that adds reading time.

**Comments compensating for bad names.** `int d; // days until expiry` — rename the variable and delete the comment.

**Commented-out code.** Delete it. Git remembers (№53, planned). Commented-out blocks rot, confuse, and nobody ever dares remove them.

**Comments that have drifted.** The fundamental hazard: comments aren't compiled or tested, so they lie silently. **A wrong comment is worse than no comment**, and every comment is a maintenance liability. This is the real argument for expressing intent in code — code can't drift from itself.

> **The tell — comments:** try to delete the comment by improving the code (rename, extract a well-named method, introduce a value object). If you can't — because the information genuinely isn't expressible in code, like *why* — that's a comment worth keeping.

---

# Part 5 — Classes and SOLID

## 5.1 Class-level basics

Classes should be **small**, **cohesive** (their fields and methods genuinely belong together — if a subset of methods uses only a subset of fields, there are two classes here), and **encapsulated** (expose behaviour, not data). Prefer `final` fields and immutability where practical (№10 §11.3) — immutable objects are thread-safe, cacheable, and can't be corrupted by callers.

**Tell, don't ask** is the encapsulation heuristic worth internalising: instead of pulling data out to make a decision, tell the object to do the thing.

```java
// Ask — the caller reaches in and reasons about another object's state
if (question.getStatus() == PENDING && question.getReviewCount() >= 2) {
    question.setStatus(APPROVED);
}

// Tell — the logic lives with the data it concerns
question.approveIfReady();
```

## 5.2 SOLID, decoded

Plain meanings, with the signal that you've violated each:

**S — Single Responsibility.** *One class, one reason to change.* Not "does one thing" — one *axis of change*. A class that formats a report and also encodes tax rules changes when either formatting or tax law changes: different stakeholders, different cadences. **Signal:** the class is named `...Manager`/`...Helper`/`...Util`, or you need "and" to describe it.

**O — Open/Closed.** *Open for extension, closed for modification.* Add behaviour without editing working, tested code. **Signal:** every new feature edits the same `switch`. **Fix:** polymorphism or a strategy (№41). Note modern Java's sealed interfaces plus pattern matching (№10 §2.5) give a deliberate, compiler-checked inversion of this — sometimes an exhaustive switch over a *closed* set is better, because the compiler catches the missing case.

**L — Liskov Substitution.** *A subtype must be usable wherever its parent is, without surprising the caller.* **Signal:** an override throws `UnsupportedOperationException`, strengthens a precondition, or weakens a guarantee. The classic: `Square extends Rectangle` breaks because setting a square's width changes its height, which no `Rectangle` caller expects.

**I — Interface Segregation.** *Many small interfaces beat one fat one.* **Signal:** implementers stub methods with empty bodies because they only need part of the interface.

**D — Dependency Inversion.** *Depend on abstractions, not concretions.* **Signal:** `new ConcreteThing()` inside business logic. **This is the one that shapes your day** — it's why your service receives a `QuestionRepository` interface rather than constructing a Hibernate-backed class. That seam is what makes the code testable (swap a double, №44 §4) and swappable. Micronaut's compile-time DI is D delivered as a framework feature.

## 5.3 Using SOLID sanely

These are *heuristics for where change hurts least*, not laws. Applied dogmatically they produce the opposite failure: an interface per class, a factory per interface, six files to follow one behaviour. **Apply them where change is likely.** A stable, simple class doesn't need five abstractions defending it. Add the seam when you feel the pain, or when you can name the change that's coming.

---

# Part 6 — The acronyms, with their nuance

**DRY — Don't Repeat Yourself.** The target is duplicated **knowledge**, not duplicated **characters**. The crucial nuance: **incidental duplication** — code that looks identical today but exists for different reasons and will change independently — should *stay* duplicated. Merging it couples two unrelated things, and the next change forces a parameter, then a flag, then a branch, and now one function serves two masters badly. **Ask: if requirement A changes, must B change too?** Yes → extract. No → coincidence, leave it.

**YAGNI — You Aren't Gonna Need It.** Don't build for imagined futures. The abstraction added "for flexibility" usually guesses the wrong axis, and you pay for it daily while never using it.

**KISS.** The simplest thing that works is usually right. Complexity should be a response to demonstrated need.

**Composition over inheritance.** Inheritance is rigid (one parent, fixed at compile time), couples you to a superclass's internals, and breaks in the ways Liskov describes. Composition — holding and delegating to collaborators — is more flexible and more testable. Use inheritance for genuine, stable is-a relationships; default to composition.

**Law of Demeter.** Talk to your immediate collaborators, not their internals. `order.getCustomer().getAddress().getPostcode()` couples you to three classes' structures, any of which can break you.

**Boy Scout Rule.** Leave code slightly better than you found it. Continuous small improvement beats "cleanup sprints" that never get scheduled.

> **The tell — acronyms:** these heuristics collide (DRY vs YAGNI, abstraction vs simplicity). When they conflict, ask which change you're optimising for — and if you can't name the change, favour the simpler code.

---

# Part 7 — Error handling

## 7.1 Fail fast, fail loudly

Validate at the boundary and throw immediately with a message carrying context. The worst outcome is **silent failure** — a swallowed exception, a default returned where an error occurred. The bug then surfaces somewhere else entirely, hours later, with no trace to the cause.

```java
// Catastrophic — the error is destroyed
try { process(q); } catch (Exception e) { }

// Bad — logged and swallowed; the caller thinks it worked
try { process(q); } catch (Exception e) { log.error("failed", e); }

// Good — wrapped with context, cause preserved, propagated
try { process(q); }
catch (ExtractionException e) {
    throw new QuestionProcessingException("extracting question " + q.getId(), e);
}
```

## 7.2 The rules

**Preserve the cause** — always pass the original as the cause; a trace with no `Caused by` has destroyed the evidence. **Include context** — "invalid difficulty 7 for question 4821 (expected 1–5)", not "invalid input". **Don't use exceptions for control flow** — expensive and obscures the normal path. **Return `Optional`, not null**, for legitimate absence (№10 §8). **Prefer unchecked exceptions** in new code (№10 §10.2). **Don't catch what you can't handle** — catching to log-and-rethrow at every layer prints the same trace five times.

---

# Part 8 — Code smells: the catalogue

A **smell** is a surface symptom suggesting a deeper problem — a prompt to look, not a verdict.

## 8.1 Bloaters — things that grew too big

**Long Method** — the most common; Extract Method until each does one thing. **Large Class / God Object** — knows and does everything; Extract Class along responsibility lines. **Long Parameter List** — Introduce Parameter Object. **Primitive Obsession** — `String status`, `int difficulty`, `String email` where types belong; value objects move validation into the type and make invalid states unrepresentable. **Data Clumps** — the same three parameters travelling together; an object trying to be born. **Duplicated Code** — extract, *if* the knowledge is genuinely shared (§6).

## 8.2 Change preventers

**Divergent Change** — one class modified for many different reasons (an SRP violation); split it. **Shotgun Surgery** — the inverse: one conceptual change forces edits across many classes, so related behaviour is scattered; move it together. These are opposites, and over-correcting one causes the other — the balance point is cohesion.

## 8.3 Object-orientation abusers

**Feature Envy** — a method more interested in another class's data than its own; Move Method. **Switch Statements** — repeated switching on a type code; replace with polymorphism (or deliberate sealed types + exhaustive patterns, №10 §2.5). **Refused Bequest** — a subclass that doesn't want what it inherits; a Liskov signal. **Temporary Field** — a field only sometimes set, confusing every reader about valid states.

## 8.4 Couplers and dispensables

**Message Chains** — `a.getB().getC().getD()`; Hide Delegate. **Middle Man** — a class that only delegates; inline it. **Inappropriate Intimacy** — two classes in each other's internals. **Lazy Class** — no longer earns its keep. **Speculative Generality** — abstraction for a future that never came; delete it. **Dead code** and **commented-out code** — delete; git remembers.

## 8.5 Magic values

```java
if (question.getDifficulty() > 3) { ... }                // 3 means what?
if (question.getDifficulty() > HARD_THRESHOLD) { ... }   // named, searchable, changeable in one place
```

---

# Part 9 — Refactoring

## 9.1 What it is (precisely)

**Refactoring is changing the structure of code without changing its observable behaviour.** The precision matters: if behaviour changes it isn't refactoring, it's rewriting, and it carries entirely different risk. The discipline:

1. **Ensure test coverage first.** Tests are what make refactoring safe rather than reckless (№44). No tests → characterisation tests first (§9.4).
2. **Small steps** — one named transformation at a time.
3. **Tests green after each step.** Red → undo, take a smaller step.
4. **Separate refactoring commits from behaviour commits.** A diff that both moves code and changes logic is nearly unreviewable.

## 9.2 The core moves

| Move | Does | Use when |
|---|---|---|
| **Extract Method** | pull a block into a named method | long method; a block needs a comment |
| **Inline Method** | the reverse | the method adds nothing |
| **Extract Variable** | name an expression | complex conditional |
| **Extract Class** | split responsibilities | large class, divergent change |
| **Move Method/Field** | relocate to where it belongs | feature envy |
| **Rename** | improve a name | any time — best value/effort ratio |
| **Introduce Parameter Object** | group parameters | long parameter list, data clumps |
| **Replace Conditional with Polymorphism** | subtype/strategy | repeated type switching |
| **Replace Magic Number with Constant** | name it | unexplained literal |
| **Decompose Conditional** | extract condition and branches | complex `if` |
| **Replace Primitive with Object** | introduce a type | primitive obsession |
| **Encapsulate Field/Collection** | expose behaviour not data | leaking internals |

Modern IDEs perform most of these mechanically and safely — **learn the shortcuts**, because a refactor you can do in two seconds is one you'll actually do.

## 9.3 When to refactor

**Continuously, in small doses.** Specifically: **before** adding a feature to code that makes it awkward (reshape first, then add cleanly — usually faster than fighting the structure); **after** getting something working, while it's fresh; **when you spot a smell** while reading; **during review** follow-ups.

**Not** as a separate "refactoring sprint" — those get cut, and their size makes them risky. And **not** while also changing behaviour.

## 9.4 Legacy code without tests

The bind: you need tests to refactor safely, but the code is untestable. The way through (Feathers): find a **seam** — a place where behaviour can be altered without editing the code (an injectable dependency, an overridable method). Write **characterisation tests** pinning down what the code *currently does* — not what it should do; you're capturing behaviour, bugs included. Then refactor behind that net, and only afterwards change behaviour deliberately.

> **The tell — refactoring:** if you can't confidently say the behaviour is unchanged, you're not refactoring — stop and get tests first. If a refactor runs hours without going green, revert. Small steps are the whole method.

---

# Part 10 — Code review

The main mechanism by which standards actually propagate. About the code, never the person.

**As reviewer:** be specific and actionable ("this NPEs if `concept` is null" beats "needs work"); explain the *why*, so it teaches; **distinguish blocking issues from preferences** explicitly ("nit:" vs "blocking:"); ask questions when unsure rather than issuing verdicts; approve when it's *better than what's there*, not when it's perfect. Look for correctness, edge cases, tests, security and readability — and **let the formatter handle formatting** (automate it and stop discussing it).

**As author:** small PRs get real review, big ones get rubber-stamped — keep them small and single-purpose; explain context in the description; separate refactoring from behaviour change (§9.1); respond to everything, even to disagree; don't take it personally — the code is being reviewed, not you.

---

# Part 11 — When to use what

**A. Extract or leave inline?** Tell → extract: the block needs a comment, is reused, or sits at a different abstraction level. Tell → leave: used once, obvious, extraction adds only indirection. Default: **extract when you can give it a name that adds information.**

**B. DRY or duplicate?** Tell → extract: both places encode the same *knowledge* and will always change together. Tell → duplicate: they merely look alike today. Default: **duplicate twice, extract on the third — by then you can see the real shape.**

**C. Abstract now or later?** Tell → now: you have two or three concrete cases and can see the axis of variation. Tell → later: one case and a hypothesis. Default: **later — YAGNI beats speculative generality.**

**D. Comment or refactor?** Tell → refactor: the comment explains *what*. Tell → comment: it explains *why*, or documents a public contract. Default: **try to delete the comment by improving the code.**

**E. Inheritance or composition?** Tell → inheritance: a genuine, stable is-a relationship where every parent operation makes sense on the child. Tell → composition: everything else. Default: **composition.**

**F. Refactor now or ticket it?** Tell → now: it's in code you're already touching and it's small. Tell → ticket: large, risky, or unrelated to your change. Default: **Boy Scout the small stuff; ticket the structural work.**

**G. Fix the smell or leave it?** Tell → fix: it's in code you're changing, or actively causing bugs. Tell → leave: stable, isolated, untouched. Default: **fix smells in code you're editing; note the rest.**

---

# How to expand this

- *Related:* №41 Software Design & Patterns (planned — the same thinking one level up: coupling, cohesion, patterns, DDD); №42 Architecture (planned); №44 Testing (testability *is* design quality — §5.2 there and §5 here are the same insight from two directions); №10 Modern Java (records, sealed types and pattern matching are language-level answers to several smells here).
- *Candidates for deeper treatment:* **a refactoring catalogue** with worked before/after for each move; **working with legacy code** (seams, characterisation tests, dependency-breaking) as a standalone; **a Practiq code-review checklist** tuned to your stack.

*Stable craft, written from knowledge — naming, functions, SOLID, smells and refactoring moves don't drift. The canonical sources if you want them: Fowler's* Refactoring *for the moves, Feathers'* Working Effectively with Legacy Code *for the untested case.*
