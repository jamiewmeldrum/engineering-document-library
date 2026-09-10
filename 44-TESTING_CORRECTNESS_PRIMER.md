# Testing & Correctness — A Primer №44

*How you know your code works, and how you find out why it doesn't. Two halves that belong together: **testing** (establishing correctness before it breaks) and **debugging** (a method, not a panic, for when it does). Java/JUnit-flavoured, Practiq-grounded — your three-tier structure and Testcontainers setup are the worked example throughout.*

The framing that changes how you write tests: **a test suite is not a correctness proof, it's a change-enablement device.** Tests don't primarily prove the code works today — you could verify that by hand once. They exist so that six months from now you can change something and find out in ninety seconds whether you broke anything. Every decision below follows from that: tests must be **fast** (or you won't run them), **reliable** (or you'll ignore them), and **coupled to behaviour rather than implementation** (or refactoring breaks them and you'll stop refactoring).

The corollary, which is the most useful heuristic in this document: **when a test fails, it should tell you what broke, not merely that something did.** A test that fails and requires twenty minutes of investigation to interpret is barely better than no test.

Contents:

- **Part 1** — why test, and what tests are actually for
- **Part 2** — the pyramid: levels of test and their trade-offs
- **Part 3** — what makes a good test
- **Part 4** — test doubles, and the over-mocking trap
- **Part 5** — TDD as a design tool
- **Part 6** — beyond example-based testing
- **Part 7** — testing hard things (time, randomness, concurrency, external systems)
- **Part 8** — coverage, quality metrics, and their misuse
- **Part 9** — debugging as a discipline
- **Part 10** — when to use what

## Symptom index

| Symptom | Cause | Go to |
|---|---|---|
| Tests break every time I refactor | testing implementation, not behaviour | §3.2 |
| Tests pass but production breaks | mocked away the thing that was wrong | §4.4 |
| A test fails intermittently | flakiness: time, order, shared state, async | §3.5 |
| The suite takes 20 minutes so nobody runs it | too many high-level tests (ice-cream cone) | §2.3 |
| A failure doesn't tell me what's wrong | too many assertions, or too high a level | §3.3 |
| Writing tests feels impossible for this class | the design is the problem, not the test | §5.2 |
| 90% coverage and still full of bugs | coverage measures execution, not verification | §8.1 |
| Test passes alone, fails in the suite | shared mutable state or order dependence | §3.5 |
| I can't reproduce the production bug | need the exact conditions — inputs, data, timing | §9.2 |

---

# Part 1 — Why test

## 1.1 The four things tests buy you

1. **Regression safety.** The dominant benefit. You change code and learn immediately whether existing behaviour still holds. Without this, every change carries unbounded risk and the codebase calcifies — people stop touching things they don't understand, and the rot compounds.
2. **Design pressure.** Hard-to-test code is almost always badly-designed code (§5.2). Tests give you early, cheap feedback on coupling.
3. **Executable specification.** A good test name plus its body documents intended behaviour more reliably than a comment, because it can't drift — if it drifts, it fails.
4. **Confidence to deploy.** The link between testing and shipping frequency is direct: teams that deploy often do so because they trust their suite (№56, planned).

## 1.2 What tests cost

Being honest about this shapes better decisions. Tests are code you write, maintain, debug and refactor. A bad test is worse than no test — it fails spuriously, blocks legitimate refactors, and trains the team to ignore failures. So the goal is not "maximum tests"; it's **the smallest suite that gives you confidence to change the system**.

> **The tell — why:** if you can't articulate what a test would catch, don't write it. Tests that assert what the code obviously does (a getter returns the field) cost maintenance and buy nothing.

---

# Part 2 — The pyramid

## 2.1 The levels

| Level | Scope | Speed | Confidence per test | Count |
|---|---|---|---|---|
| **Unit** | one class/function, dependencies stubbed | ms | narrow | many |
| **Integration** | several components, real infrastructure | 100ms–seconds | broad | some |
| **End-to-end** | the whole system via its real interface | seconds–minutes | broadest | few |

The trade is monotonic: as scope grows, so does confidence *and* runtime *and* flakiness *and* the difficulty of localising a failure. A unit test failure points at a method; an E2E failure points at "somewhere in the system."

## 2.2 Practiq's three tiers, and what each is for

Your structure maps cleanly, and each tier has a distinct job — worth being explicit because the value comes from *not* duplicating coverage across them:

- **Unit (`*Test`)** — pure logic in isolation. Your `QuerySpecification` factories tested by calling `.toPredicate(root, query, cb)` against mocked Criteria objects. Fast, precise, run constantly.
- **Component (`*CT`)** — wiring and collaboration with dependencies mocked: the controller/service layer, verified with `any()` + `verify()` because spec lambdas have no meaningful identity. Catches integration-of-*my*-code problems without infrastructure.
- **Integration (`*IT`)** — the real thing against **real Postgres via Testcontainers**. This tier is non-negotiable and catches the class of bug no amount of mocking can: SQL that doesn't compile, dialect differences, constraint violations, transaction and flush semantics, N+1 queries, migration errors. H2 would be faster and would lie to you (№51 §7.3, №20 §7.3).

The rule that keeps this coherent: **each tier tests what the tiers below it can't.** Don't re-test business logic through the integration tier; don't try to test SQL behaviour with mocks.

## 2.3 The anti-patterns

**The ice-cream cone** — mostly E2E, few units. Slow, flaky, and when something fails you learn only that the system is broken. This happens when teams test through the UI because it feels "more real."

**The hourglass** — many units and many E2E, nothing in between. The middle is where most integration bugs live; skipping it means unit tests pass, E2E fails, and there's nothing to localise the problem.

**The testing trophy** (Kent C. Dodds' counter-proposal) argues the *integration* layer deserves the most weight, since it catches real bugs at acceptable speed. There's genuine merit to it — the honest position is that the pyramid's *shape* matters less than its *principle*: **push each test to the lowest level that can meaningfully catch the bug.**

> **The tell — level:** ask "what's the cheapest test that would have caught this bug?" A null check → unit. A wrong SQL query → integration. A broken deployment config → E2E/smoke. Writing an E2E test for logic a unit test could cover is paying ten times the runtime for less diagnostic value.

---

# Part 3 — What makes a good test

## 3.1 Structure: Arrange–Act–Assert

```java
@Test
void approve_pendingQuestion_setsStatusToApproved() {
    // Arrange — set up the world
    Question question = aQuestion().withStatus(PENDING).build();
    when(repository.findById(1L)).thenReturn(Optional.of(question));

    // Act — one action, the thing under test
    service.approve(1L);

    // Assert — verify the outcome
    assertThat(question.getStatus()).isEqualTo(APPROVED);
}
```

Three visible phases. If your Act section has several steps, you're probably testing more than one thing. If Arrange is enormous, the class under test likely has too many dependencies (§5.2).

## 3.2 Test behaviour, not implementation

The single most important property, and the one most often violated.

```java
// BAD — coupled to how it's done
verify(repository).findById(1L);
verify(question).setStatus(APPROVED);
verify(repository).save(question);      // and now dirty checking makes save() unnecessary → test breaks, code is fine

// GOOD — coupled to what happens
service.approve(1L);
assertThat(question.getStatus()).isEqualTo(APPROVED);
```

A test that asserts on *interactions* breaks when you refactor internals; a test that asserts on *outcomes* survives. The heuristic: **if you can rewrite the method body entirely, keeping the same observable behaviour, and the test still passes — it's a good test.**

There's a real exception: sometimes the interaction *is* the behaviour. "Sends an email on approval" can only be verified by checking the email service was called. Use interaction verification when the side effect is the point, outcome assertions everywhere else.

## 3.3 One reason to fail

Each test should verify one logical thing, so a failure is diagnostic. That's *one logical assertion*, not literally one `assertThat` — asserting three fields of one returned object is fine; asserting the return value *and* a database write *and* a published event is three tests wearing one name.

## 3.4 Naming

The name should state scenario and expectation, so a failure report reads as a specification:

```
approve_pendingQuestion_setsStatusToApproved
approve_alreadyApprovedQuestion_throwsIllegalState
findAll_noConceptIdSupplied_returnsAllApproved
```

Pattern: `method_condition_expectedOutcome`. When the CI output lists failures, you should be able to diagnose from names alone.

## 3.5 FIRST, and the flakiness problem

**F**ast, **I**solated, **R**epeatable, **S**elf-validating, **T**imely. The one that causes the most real pain is **isolated**:

A **flaky** test — one that passes and fails without code changes — is worse than a deleted test, because it trains everyone to re-run CI and ignore red. Causes, in rough order of frequency: **shared mutable state** between tests (a static field, a shared database row); **order dependence** (test B relies on test A having run); **time** (a test that fails at midnight or in a different timezone); **async/timing** (sleeping and hoping instead of waiting for a condition); **randomness** (unseeded); **external systems** (a network call in a unit test).

Fix flakes immediately or delete them. A suite you don't trust is a suite you don't have.

## 3.6 Test data builders

Fixture setup is where test readability goes to die. A builder with sensible defaults keeps each test focused on the one field that matters:

```java
Question question = aQuestion()
        .withStatus(PENDING)     // only what THIS test cares about
        .build();                // everything else is a sane default
```

The alternative — a giant shared `setUp()` fixture used by forty tests — couples them all together and makes each test's actual preconditions invisible.

---

# Part 4 — Test doubles

## 4.1 The five kinds

Precise vocabulary (Meszaros's taxonomy), because people say "mock" for all of them:

| Double | Is | Use when |
|---|---|---|
| **Dummy** | a placeholder, never used | filling a required parameter |
| **Stub** | returns canned answers | you need the collaborator to *provide* something |
| **Spy** | a real object recording how it was called | you need real behaviour plus verification |
| **Mock** | pre-programmed with expectations, **verifies interactions** | the interaction *is* the behaviour |
| **Fake** | a working lightweight implementation | in-memory repository, embedded broker |

The distinction that matters practically: a **stub** helps you *arrange*; a **mock** helps you *assert*. Using mocks for arrangement is how tests become brittle.

## 4.2 Mockito in practice

```java
// stub — provides a value
when(repository.findById(1L)).thenReturn(Optional.of(question));

// mock verification — the interaction is the point
verify(emailService).sendApprovalNotice(question);
verify(repository, never()).delete(any());

// argument matchers — the pattern your spec tests need,
// because lambdas have no meaningful equality
verify(repository).findAll(any(QuerySpecification.class));

// capture — assert on what was actually passed
ArgumentCaptor<Question> captor = ArgumentCaptor.forClass(Question.class);
verify(repository).save(captor.capture());
assertThat(captor.getValue().getStatus()).isEqualTo(APPROVED);
```

`ArgumentCaptor` is the underused one — it lets you assert on *content* rather than merely that a call happened, which is usually what you actually care about.

## 4.3 Where to draw the line

Mock at **architectural boundaries** — things that are slow, non-deterministic, external, or expensive: HTTP clients, message brokers, the clock, third-party APIs, payment gateways. Use the real thing **within** your boundary: your own value objects, domain logic, pure functions. Never mock what you don't own *directly* — wrap a third-party client in your own interface and mock that, so a library upgrade doesn't shatter your tests.

## 4.4 The over-mocking trap

```java
// This test verifies that the mocks were configured. Nothing else.
when(a.doThing()).thenReturn(x);
when(b.process(x)).thenReturn(y);
when(c.save(y)).thenReturn(z);
assertThat(service.run()).isEqualTo(z);
```

If every collaborator is mocked, the only thing under test is the wiring — and you've encoded your current implementation into the test, so any refactor breaks it. Worse, mocks *lie*: your mocked repository returns exactly what you told it to, including things the real database never would.

**This is precisely why Practiq's integration tier exists.** A mocked repository will happily return a `Question` with a null `concept` that violates a NOT NULL constraint, or accept a `QuerySpecification` that generates invalid SQL. Only real Postgres tells you the truth.

> **The tell — doubles:** prefer a **fake** or the real thing over a mock where practical; mock the boundary, not the interior. If a test needs more than about three doubles, that's a design signal (§5.2), not a testing problem.

---

# Part 5 — TDD as a design tool

## 5.1 The cycle

**Red** — write a failing test for the next small behaviour. **Green** — make it pass as simply as possible, even crudely. **Refactor** — clean up with the test as your safety net. Repeat in minutes-long loops.

The discipline that makes it work is watching the test **fail first**. A test that passes before you write the code is testing nothing — a surprisingly common and entirely silent failure mode.

## 5.2 What it's actually for

The tests are a by-product. The real benefit is that **writing the test first forces you to use your API before you build it**, from the caller's perspective. You immediately feel awkward signatures, hidden dependencies and unclear responsibilities — at the point where they're cheap to fix.

Hence the most valuable diagnostic in this whole document: **"this is hard to test" is almost never a testing problem; it's a design problem.**

| Testing pain | Design problem underneath |
|---|---|
| Enormous Arrange section | too many dependencies — the class does too much |
| Must mock a static/singleton | hidden dependency; inject it instead |
| Can't isolate the logic | business logic tangled with I/O |
| Must test through three layers | no seam at the right level |
| Test needs the real clock/filesystem | side effects not pushed to the edges |

## 5.3 Pragmatic TDD

Dogma isn't required. TDD is excellent for algorithmic logic, bug fixes (write the failing test that reproduces it first — then you know it's fixed *and* it can't regress), and API design. It's less useful for exploratory spikes and UI layout. Writing tests immediately after, while the design is fluid, captures most of the benefit.

The one place I'd argue it's near-mandatory: **fixing a bug.** Reproduce it as a failing test before touching the code, or you don't actually know you fixed it.

---

# Part 6 — Beyond example-based testing

## 6.1 The limitation

Every test above checks *one example*. Your three hand-picked cases pass; the empty list, the Unicode string, the integer at `MAX_VALUE` were never considered. Example-based testing only finds bugs in cases you already thought of — which are, by definition, the cases you probably handled.

## 6.2 Property-based testing

Instead of specific examples, state a **property** that must hold for *all* inputs, and let the framework generate hundreds of cases including nasty edges. When it finds a failure, it **shrinks** it to the minimal reproducing case.

```java
@Property
void serialisingThenDeserialising_returnsAnEquivalentObject(@ForAll Question q) {
    assertThat(deserialise(serialise(q))).isEqualTo(q);   // round-trip property
}
```

Properties that generalise well: **round-trip** (encode/decode, save/load), **invariants** (sorted output is always ordered; a total never goes negative), **idempotence** (applying twice equals applying once — directly relevant to №31 §8), **commutativity/equivalence** (an optimised implementation always agrees with the naive one). jqwik is the Java tool.

## 6.3 The other useful kinds

- **Contract tests** — verify a consumer and provider agree on an interface, without spinning up both. Valuable across service boundaries (Pact).
- **Snapshot/approval tests** — capture output and diff against a stored copy. Cheap for complex output (generated HTML, reports); dangerous if approved carelessly.
- **Mutation testing** — deliberately mutate your code (flip a `<` to `<=`, remove a line) and check whether any test fails. If none does, your tests don't really cover that logic. **This is a far better quality signal than line coverage** (§8.1). PIT is the Java tool.
- **Performance/load tests** — verify behaviour under expected and excessive load (JMeter, k6, Gatling).
- **Smoke tests** — a handful of critical-path checks run post-deploy against production.

---

# Part 7 — Testing hard things

## 7.1 Time

Never call `Instant.now()` directly in testable code. Inject a `Clock`:

```java
// production
@Singleton Clock clock() { return Clock.systemUTC(); }

// in the class
private final Clock clock;
Instant now = Instant.now(clock);

// in the test — time is now deterministic
Clock fixed = Clock.fixed(Instant.parse("2026-07-23T10:00:00Z"), ZoneOffset.UTC);
```

This kills a whole family of flakes (timezone-dependent failures, tests that break at month boundaries, "expires in 30 days" logic) and makes time-based behaviour directly assertable.

## 7.2 Randomness and IDs

Same principle: inject the source. A seeded `Random` or an injected id generator makes the output deterministic. Anything that generates its own randomness internally is untestable by construction.

## 7.3 External systems

The ladder, cheapest first: **stub the client interface** (unit level) → **a fake in-process server** (WireMock for HTTP) → **Testcontainers** running the real dependency → **the actual sandbox environment** (rare, slow, for contract confidence).

**Testcontainers deserves emphasis** because it's your setup and it's the modern answer: it starts the real Postgres in Docker for the test run, so you test against the engine you deploy on. It removes the entire class of "passed on H2, failed on Postgres" bugs (dialect differences, constraint behaviour, transaction semantics, sequence handling). The cost is seconds of startup, mitigated by container reuse across the suite.

## 7.4 Concurrency

Covered in №12 §9, and the conclusion bears repeating because it's counter-intuitive: **you cannot test your way to concurrent correctness.** A race that needs one specific interleaving may pass ten million runs. Concurrency correctness is established at *design* time (immutability, confinement, happens-before) and tests are only a weak backstop. Stress tools (jcstress) widen the odds of catching one; they don't prove absence.

## 7.5 Databases

Each test must start from a known state. Options: **transactional rollback** (wrap the test in a transaction that's never committed — fast, but doesn't test real commit behaviour and breaks if the code manages its own transactions), **truncate between tests** (slower, more realistic), or **a fresh container per class** (slowest, most isolated). Practiq's Flyway migrations should run against the test container, so migrations are themselves tested — a genuinely valuable side benefit.

---

# Part 8 — Coverage and its misuse

## 8.1 What coverage measures

Line/branch coverage measures **which code was executed**, not **whether anything was verified**. This test achieves 100% coverage of a method and asserts nothing:

```java
@Test void itRuns() { service.approve(1L); }    // no assertion. 100% covered. Zero value.
```

Coverage is therefore a **negative indicator only**: low coverage reliably tells you something is untested; high coverage tells you nothing about quality. A team targeting 90% will hit 90% — often by writing assertion-free tests over getters, while the gnarly conditional logic stays untested.

**Mutation testing** (§6.3) is the honest alternative: it measures whether your tests would *notice* a change in behaviour.

## 8.2 A sane policy

Cover the code where bugs are expensive: business logic, edge cases, anything with branching, anything that's broken before. Don't chase coverage on generated code, DTOs, or trivial delegation. If you must set a number, apply it to *new* code and treat it as a smoke alarm, not a goal.

> **The tell — coverage:** never ask "what's our coverage?"; ask "if I broke this behaviour, would a test fail?" That question is answerable by trying it — go and break something deliberately and watch what happens.

---

# Part 9 — Debugging as a discipline

Testing prevents; debugging responds. Debugging is a **method**, and the difference between people who are fast at it and people who aren't is almost entirely whether they follow one.

## 9.1 The method

1. **Reproduce it reliably.** Everything depends on this. An intermittent bug you can't trigger cannot be confirmed fixed. Invest here first — find the exact input, data state, timing or configuration.
2. **Read the error properly.** Read the *whole* stack trace, top to bottom, and find the deepest frame in *your* code. Read the "Caused by" chain to its root. An astonishing number of bugs are solved by actually reading the message rather than skimming it.
3. **Form one hypothesis.** "The status is null because the mapper skips it when the source is empty." Specific and falsifiable — not "something's wrong with the mapping."
4. **Test it by changing one thing.** Change several and you learn nothing from the result.
5. **Narrow the search space.** Binary search: does the bug exist halfway through the pipeline? Bisect the code path, the data, or the history (`git bisect` finds the introducing commit mechanically — №53, planned).
6. **Fix the cause, not the symptom.** A null check that hides why the value was null is a deferred bug.
7. **Write the test that would have caught it** (§5.3), then verify it fails before your fix and passes after.

## 9.2 The techniques

**The debugger** beats print statements for inspecting state — breakpoints, conditional breakpoints (`questionId == 42`, invaluable in loops), watch expressions, stepping, and *evaluate expression* to try things live. Learn your IDE's debugger properly; it's a leveraged skill.

**Logging** wins where the debugger can't reach: production, concurrency (breakpoints change timing and hide races), and anything spanning services. Structured logs plus a correlation ID (№31 §11) are what make a distributed bug tractable.

**Bisection** applies to more than git: comment out half the pipeline, halve the input data, disable half the config. Halving beats guessing.

**Rubber ducking.** Explaining the problem aloud, in full, to something that can't help you solves a genuinely large share of bugs — because articulating it forces you to state assumptions you'd been skipping over.

**Question your assumptions.** Bugs live in the gap between what you believe the code does and what it does. When thoroughly stuck, the culprit is nearly always something you're *certain* is fine. Verify it anyway: is the code you're editing actually the code that's running? Is that config value what you think? Did the build actually rebuild?

## 9.3 Production debugging

You often can't attach a debugger. The toolkit: structured logs and log aggregation; metrics to find *when* it changed; distributed tracing to find *where* time went (№31 §11); thread dumps for hangs and deadlocks (№12 §8); heap dumps for memory issues (`-XX:+HeapDumpOnOutOfMemoryError`, №10 §13); and — first, always — **what changed?** Deployments, config changes and data changes cause most incidents, and "roll back and investigate calmly" beats "debug live under pressure."

> **The tell — debugging:** resist the urge to change things at random until it works. That path gives you a system that works for unknown reasons, which is not a fixed system. One hypothesis, one change, observe, repeat.

---

# Part 10 — When to use what

**A. Which test level?** Tell → unit: pure logic, many edge cases, fast feedback. Tell → integration: anything crossing a boundary you don't control (SQL, HTTP, serialisation). Tell → E2E: a handful of critical user journeys only. Default: **the lowest level that could catch the bug.**

**B. Real, fake, or mock?** Tell → real: it's fast, deterministic and yours. Tell → fake: you need working behaviour without the cost (in-memory implementation). Tell → mock: the interaction is the behaviour, or the dependency is external/slow/non-deterministic. Default: **real inside your boundary, doubles at the edges.**

**C. TDD or test-after?** Tell → TDD: fixing a bug (always), designing a new API, algorithmic logic. Tell → after: exploratory spikes, UI. Default: **test-first for bugs and APIs; test-soon for everything else.**

**D. Example-based or property-based?** Tell → examples: specific known behaviours and edge cases. Tell → properties: invariants, round-trips, idempotence, or when the input space is large. Default: **examples, with properties for the handful of places a true invariant exists.**

**E. Fix a flaky test or delete it?** Tell → fix: it covers something valuable and the cause is knowable. Tell → delete: nobody trusts it and nobody will fix it. Default: **fix within a day or delete — never leave it red-amber.**

**F. Debugger or logging?** Tell → debugger: local, reproducible, you need to inspect state. Tell → logging: production, concurrent, distributed, or timing-sensitive. Default: **debugger locally, structured logs everywhere else.**

---

# How to expand this

- *Related:* №12 §9 (why concurrency can't be tested into correctness), №31 §11 (observability — production's version of debugging), №20 §7.3 and №50 (why Testcontainers over H2), №40 Clean Code (testability *is* design quality), №56 CI/CD (planned — where the suite becomes a gate).
- *Candidates for deeper treatment:* **JUnit 5 + AssertJ + Mockito in depth** with Practiq examples; **Testcontainers patterns** (reuse, shared containers, Flyway integration, performance); **property-based testing with jqwik** worked properly; **a debugging playbook** for production incidents specifically.

*Stable practice, written from knowledge — the pyramid, the doubles taxonomy, TDD and the debugging method don't drift. Specific library APIs (JUnit, Mockito, Testcontainers versions) do; check current docs for exact syntax.*
