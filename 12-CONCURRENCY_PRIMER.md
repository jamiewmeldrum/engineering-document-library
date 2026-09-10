# Concurrency — A Primer

*The CS foundation beneath the applied Java. Where the Modern Java primer §12 gives you the Java concurrency **toolkit** (threads → executors → `CompletableFuture` → virtual threads), this goes underneath it: **why** concurrency is hard at the level of hardware and the memory model, the full taxonomy of what goes wrong, and how to reason about correctness rather than test-and-pray. Java-flavoured, but the ideas are universal. Practiq lens: a Micronaut service whose beans are shared singletons.*

Concurrency is the topic where self-taught and formally-trained engineers diverge most, because the hard part is invisible: it's not syntax, it's a **mental model of a machine that doesn't do what your code appears to say.** The code `count++` looks atomic and sequential. It is neither. Every bug in this document comes from that gap between the code you read and the machine that runs it.

The one sentence to hold: **concurrency bugs are the price of shared mutable state, and almost every technique here is a way to avoid it, contain it, or coordinate access to it.** If nothing is shared, or nothing is mutable, most of this document doesn't apply to you — which is itself the most important lesson.

Contents:

- **Part 1** — concurrency vs parallelism, and why it's hard (hardware, OS, the three lies)
- **Part 2** — the bug taxonomy: races, atomicity, visibility, ordering, deadlock, livelock, starvation
- **Part 3** — the memory model and happens-before (the conceptual core)
- **Part 4** — coordination primitives: mutual exclusion, locks, semaphores, monitors
- **Part 5** — atomics, compare-and-swap, and lock-free
- **Part 6** — the higher-level toolkit (and where it points at Modern Java §12)
- **Part 7** — patterns for correct concurrency
- **Part 8** — deadlock, avoided
- **Part 9** — reasoning about, testing, and debugging concurrency
- **Part 10** — when to use what, and the questions worth being able to answer

## Symptom index

| Symptom | Cause | Go to |
|---|---|---|
| A counter/total is occasionally wrong under load | lost update (read-modify-write race) | §2.1 |
| A `HashMap` corrupted / infinite loop under concurrent writes | non-thread-safe structure mutated concurrently | §2.1, §6 |
| One thread's change is never seen by another | visibility (no happens-before) | §2.3, §3 |
| A flag set by one thread never stops another's loop | visibility — needs `volatile` | §2.3 |
| Two threads freeze, both waiting forever | deadlock | §2.4, §8 |
| Threads busy but no progress | livelock | §2.5 |
| One thread never gets a turn | starvation | §2.5 |
| "Passed 1,000 times, fails in prod" | a race — timing-dependent, non-deterministic | §9 |
| `ConcurrentModificationException` | modifying a collection while iterating (often single-threaded) | Collections §9 |
| A shared field on a Micronaut/Spring bean behaves erratically | singleton with mutable state | §7.2 |
| A check-then-act "worked" then didn't | compound operation wasn't atomic | §2.1, §5 |

---

# Part 1 — Concurrency vs parallelism, and why it's hard

## 1.1 The distinction

- **Concurrency** is *dealing with* many things at once — a **structuring** of your program so tasks can make progress independently. It's about *composition and coordination*.
- **Parallelism** is *doing* many things at once — **simultaneous execution**, which requires multiple CPU cores.

A single core can be **concurrent but not parallel**: the OS scheduler rapidly switches between tasks (context switches), interleaving them so fast they appear simultaneous. Add cores and concurrency *can become* parallelism. The classic framing (Rob Pike): concurrency is about *dealing with* lots of things at once; parallelism is about *doing* lots of things at once. You can have either without the other.

Why it matters: most concurrency bugs are **interleaving** bugs — they happen because a context switch landed in the worst possible place — and those occur even on a single core. You do not need parallel hardware to hit a race. Which is why they're so insidious: they can lie dormant on your fast laptop and surface on a differently-scheduled production box.

## 1.2 The three things that make it hard

Concurrency is hard for three concrete, mechanical reasons — each a place where the machine diverges from your code:

**1. The scheduler can interrupt you anywhere.** The OS can suspend a thread *between any two machine instructions* (Engineer's Map §1.3). Your one line of Java is several instructions; the switch can land in the middle. You do not control when.

**2. Threads share memory.** Threads within a process share the heap (Engineer's Map §1.3). Two threads touching the same object are two hands in the same drawer. This is the source of *all* the trouble — and the thing immutability removes.

**3. The hardware lies about memory for speed.** This is the one nobody self-taught knows, and it's the deepest:
   - Each core has its own **caches** (Engineer's Map §1.2). A write by core 1 may sit in core 1's cache, invisible to core 2, for a while. There is no automatic "everyone sees my write immediately."
   - The CPU and compiler **reorder instructions** for performance, as long as the result looks the same *to a single thread*. But another thread watching can see the reordered sequence, which may be nonsense from its point of view.

So the machine your concurrent code runs on is one where **operations aren't atomic, writes aren't immediately visible, and instructions don't run in the order you wrote them.** Sequential code is shielded from all three. Concurrent code is not, and Part 3 (the memory model) is the set of rules for clawing back guarantees.

> **The tell — Part 1:** if you remember one thing, it's that "it looks sequential and atomic" is an illusion the hardware maintains *only for a single thread*. The moment a second thread observes your memory, that illusion breaks, and you must explicitly re-establish the guarantees you took for granted.

---

# Part 2 — The bug taxonomy

The named failure modes. Knowing the taxonomy is half of diagnosing them.

## 2.1 Race conditions — the root failure

A **race condition** is any bug where the outcome depends on the *timing* of thread interleaving. The canonical example, and worth seeing at the instruction level because it demystifies everything:

```java
count++;   // looks atomic. It is THREE operations:
           //   1. read  count           (say, 5)
           //   2. add 1                  (6)
           //   3. write count            (6)
```

Two threads, both incrementing, can interleave:

```
Thread A: read count → 5
Thread B: read count → 5        ← B read before A wrote
Thread A: add 1 → 6, write → 6
Thread B: add 1 → 6, write → 6  ← should be 7
```

Two increments, one lost. This is a **lost update**, and it's the archetype. It requires no exotic hardware — just an unlucky context switch between the read and the write.

The general shape is **check-then-act** and **read-modify-write**: any sequence where you observe state and then act on that observation is a race unless the whole sequence is atomic, because the state can change between the observing and the acting:

```java
if (!map.containsKey(k)) {   // check
    map.put(k, v);           // act — but another thread may have put k between the two
}
```

**Data race** is the stricter, memory-model term: two threads access the same location, at least one writes, and there's no synchronisation ordering them (§3). A data race is *undefined behaviour* — the program is broken, not merely occasionally wrong. (All data races are race conditions; not all race conditions are data races — you can have a timing bug even through correctly-synchronised operations.)

## 2.2 Atomicity — the missing indivisibility

An operation is **atomic** if it happens all-at-once from every thread's view — no other thread can observe it half-done. `count++` isn't atomic (§2.1). Neither is `long`/`double` assignment on some platforms historically (a 64-bit write could tear into two 32-bit halves). The fix is either a lock making the sequence atomic *by exclusion* (§4), or an atomic primitive doing it *in hardware* (§5).

## 2.3 Visibility — the change nobody sees

Even with no interleaving problem, a write by one thread may simply **never become visible** to another, because it's sitting in a core's cache (§1.2). The classic:

```java
private boolean running = true;   // NOT volatile

// Thread A (worker loop)
while (running) { doWork(); }      // may loop FOREVER

// Thread B
running = false;                   // A may never see this
```

Thread B sets `running = false`; thread A's loop may never observe it and spins forever. Nothing is racing — it's pure **visibility**. The compiler may even hoist the read out of the loop entirely (it sees no local write to `running`), turning it into `while (true)`. The fix is `volatile` (§3), which forces the write to main memory and the read to re-fetch it.

## 2.4 Deadlock — the mutual wait

Two (or more) threads each holding a resource the other needs, waiting forever:

```java
// Thread A                    // Thread B
synchronized (lock1) {         synchronized (lock2) {
  synchronized (lock2) { ... }   synchronized (lock1) { ... }
}                              }
// A holds lock1, wants lock2.  B holds lock2, wants lock1.  Both wait forever.
```

Deadlock requires **all four Coffman conditions** simultaneously — and breaking *any one* prevents it, which is the key to §8:

1. **Mutual exclusion** — resources can't be shared.
2. **Hold and wait** — a thread holds one resource while waiting for another.
3. **No preemption** — resources can't be forcibly taken.
4. **Circular wait** — a cycle of threads each waiting on the next.

The dining philosophers problem is the textbook illustration: five philosophers, five forks, each needs two — grab the wrong way round and everyone starves holding one fork.

## 2.5 Livelock and starvation — the subtler cousins

- **Livelock:** threads are *active* but make no progress — each responds to the other and nothing advances. Two people stepping aside in a corridor, repeatedly, in the same direction. Busy, not blocked, but stuck.
- **Starvation:** a thread never gets the resource it needs because others keep taking it — an unfair lock, or a low-priority thread perpetually jumped by high-priority ones. It makes *some* progress possible in principle but never in practice for the victim.

> **The tell — Part 2:** the diagnostic vocabulary matters. "Wrong value under load" → race/atomicity. "Change never seen" → visibility. "Frozen, waiting" → deadlock. "Busy but stuck" → livelock. "One thread never served" → starvation. Naming it correctly points straight at the fix.

---

# Part 3 — The memory model and happens-before

This is the conceptual core, and the part most people never learn. It's what turns "sprinkle `synchronized` until it works" into actually understanding *why* it works.

## 3.1 The problem the memory model solves

Given §1.2 (caches) and §1.2 (reordering), the language needs **rules** defining when one thread is *guaranteed* to see another's writes, and in what order. Without such rules, no concurrent program could be reasoned about. The **Java Memory Model (JMM)** is that rulebook — and every managed language has an equivalent. Its central concept is **happens-before**.

## 3.2 Happens-before

**Happens-before** is a guarantee about visibility and ordering: if action X *happens-before* action Y, then X's effects (all its memory writes) are **visible to and ordered before** Y. It is *not* about wall-clock time — it's a formal ordering the JMM promises to respect.

Within a single thread, everything happens-before the next thing in program order (the sequential illusion). The interesting part is what establishes happens-before *across* threads:

| This… | …happens-before… | So |
|---|---|---|
| a write to a `volatile` field | a subsequent read of that same field | the reader sees the write (and everything before it) |
| **unlocking** a monitor (`synchronized` exit) | a subsequent **lock** of the same monitor | the next thread in sees all prior changes |
| `thread.start()` | the first action of the started thread | the new thread sees everything set up before it |
| a thread's final action | another thread's `thread.join()` returning | the joiner sees the finished thread's work |
| writing an `AtomicX` / releasing a `Lock` | reading it / acquiring it | the reader sees the writer's work |

The unifying idea: **synchronisation actions create happens-before edges**, and those edges are the *only* thing that guarantees one thread sees another's writes. No edge → no guarantee → the visibility bug in §2.3.

## 3.3 What this reframes

Two things click into place once you have happens-before:

- **`volatile` is about visibility and ordering, not mutual exclusion.** A `volatile boolean running` fixes §2.3 because the write happens-before the read. But `volatile int i; i++` is *still* a race — `volatile` guarantees you see the latest value, but the read-modify-write isn't atomic (§2.2). `volatile` publishes; it doesn't serialise.
- **`synchronized` does two jobs at once.** It provides mutual exclusion (only one thread in the block) *and* a happens-before edge (the next thread to enter sees the last thread's writes). People remember the first and forget the second, then are baffled when removing a "pointless" lock breaks visibility. The lock was carrying the memory guarantee.

> **The tell — Part 3:** the question to ask of any concurrent code is *"what establishes happens-before between the write and the read?"* If the answer is "nothing," you have a visibility bug even if you never see it in testing. `volatile` for a single published flag/reference; a lock or atomic for anything you read-modify-write.

---

# Part 4 — Coordination primitives

The tools for controlling access to shared mutable state. These are CS concepts first; the Java mapping is noted, with the applied API surface in Modern Java §12.

## 4.1 Mutual exclusion and the critical section

A **critical section** is a region of code that accesses shared state and must not run concurrently with itself. **Mutual exclusion** is the property that only one thread executes it at a time. Every lock is a mechanism for enforcing mutual exclusion over a critical section — keep critical sections *small* (hold the lock briefly), because they serialise, and serialisation is the enemy of the throughput you wanted from concurrency in the first place.

## 4.2 Locks / mutexes

A **mutex** ("mutual exclusion") is the basic lock: acquire it, do the critical work, release it; while you hold it, others block. In Java:

```java
// intrinsic lock — every object has one
synchronized (this) { balance += amount; }   // acquire on enter, release on exit (even on exception)

// explicit lock — more control
private final ReentrantLock lock = new ReentrantLock();
lock.lock();
try { balance += amount; }
finally { lock.unlock(); }                    // MUST be in finally, or a thrown exception deadlocks everyone
```

`ReentrantLock` buys what `synchronized` can't: **`tryLock()`** (attempt, don't block — the key to deadlock avoidance, §8), **timed** acquisition, **interruptible** waiting, and **fairness** (FIFO ordering, at a throughput cost). "Reentrant" means the holding thread can re-acquire it (so a synchronised method calling another doesn't self-deadlock) — `synchronized` is reentrant too.

**Read-write locks** (`ReadWriteLock`) split the lock: many readers concurrently, *or* one writer exclusively. A win when reads vastly outnumber writes, because readers don't block each other.

## 4.3 Semaphores

A **semaphore** generalises the mutex to *N* permits: it allows up to N threads through at once. `acquire()` takes a permit (blocking if none), `release()` returns one. A mutex is a semaphore with one permit. The natural use is **limiting concurrency** — "at most 10 threads may hit this downstream service":

```java
private final Semaphore permits = new Semaphore(10);
permits.acquire();
try { callRateLimitedService(); }
finally { permits.release(); }
```

## 4.4 Monitors and condition variables

A **monitor** is the pattern of "a lock plus the ability to wait for a condition" — an object whose methods are mutually exclusive and which threads can wait *inside*. The primitive is the **condition variable**: a thread holding the lock can `wait()` (release the lock and sleep until signalled) and another can `signal()`/`notify()` it. This is how you build producer-consumer coordination — "wait until the queue is non-empty."

```java
// the classic wait/notify — note the WHILE, not IF
synchronized (queue) {
    while (queue.isEmpty()) {          // MUST be while: guard against spurious wakeups + stale conditions
        queue.wait();                  // releases the lock, sleeps, re-acquires on wake
    }
    return queue.poll();
}
```

The `while`-not-`if` rule is a genuine trap: a woken thread must **re-check** the condition, because it might have been woken spuriously, or another thread might have consumed the item first. In modern code you rarely hand-roll this — `BlockingQueue` (§6) *is* this pattern, packaged and correct.

> **The tell — Part 4:** reach for the *highest-level* tool that fits. A `BlockingQueue` over hand-rolled `wait/notify`; an `AtomicInteger` over a lock around a counter; a `ReadWriteLock` only when reads genuinely dominate and you've measured the contention. Every explicit `lock()` needs its `unlock()` in a `finally`. Keep critical sections tiny.

---

# Part 5 — Atomics, compare-and-swap, and lock-free

Locks work by *exclusion* — everyone else waits. There's a faster family that works *without* blocking, and it's worth understanding because it's what the concurrent collections are built on.

## 5.1 Compare-and-swap (CAS) — the hardware primitive

Modern CPUs provide an atomic **compare-and-swap** instruction: "atomically, if this location still holds the value I expect, set it to the new value; otherwise tell me it changed." One indivisible hardware operation. It's the foundation of lock-free programming:

```java
// AtomicInteger.incrementAndGet(), conceptually:
int current, next;
do {
    current = get();          // read
    next = current + 1;       // compute
} while (!compareAndSet(current, next));   // "if still `current`, set `next`" — retry if someone changed it
```

Instead of *preventing* interference with a lock, CAS *detects* it (the compare fails) and **retries**. This is **optimistic concurrency** — assume no conflict, proceed, and roll back if you were wrong — and it's the exact same idea as JPA's `@Version` optimistic locking one layer up (JPA reference §2.7, data-access §2.7). Under low contention it's much faster than locking (no blocking, no context switches); under high contention the retries pile up and a lock can win.

## 5.2 Atomic classes

`AtomicInteger`, `AtomicLong`, `AtomicReference`, `LongAdder` wrap CAS into usable operations — `incrementAndGet`, `compareAndSet`, `updateAndGet`, `getAndSet`. They fix the §2.1 counter without a lock:

```java
private final AtomicInteger count = new AtomicInteger();
count.incrementAndGet();   // atomic, lock-free, correct
```

(`LongAdder` beats `AtomicLong` under *high* contention by spreading the count across cells and summing on read — a specialised tool worth knowing exists.)

## 5.3 Lock-free and the ABA problem

**Lock-free** structures guarantee *some* thread always makes progress (no thread can block all others), built from CAS loops. `ConcurrentLinkedQueue` is one. They're brilliant but subtle — the famous **ABA problem**: a CAS checking "is it still A?" succeeds even though the value went A→B→A in between, which may be wrong if that round-trip mattered (a reused node). Solutions use a version stamp (`AtomicStampedReference`). You'll rarely write lock-free structures — but knowing *why* they're hard is why you should use the library's, not roll your own.

> **The tell — Part 5:** for a shared counter or flag, an **atomic** beats a lock — lock-free, no blocking, correct. It's the same optimistic pattern as `@Version`: assume no conflict, detect and retry if wrong. Don't build lock-free data structures yourself; the concurrency toolkit already has correct ones (§6).

---

# Part 6 — The higher-level toolkit

You should almost never touch raw threads, locks, or `wait/notify` in application code. The `java.util.concurrent` library packages the primitives above into correct, high-level tools. This is covered applied in **Modern Java §12** — here's the map of *what each is for* and which primitive it's built on:

| Tool | Built on | For |
|---|---|---|
| **`ExecutorService`** / thread pools | threads + a work queue | run tasks without managing thread lifecycle |
| **`CompletableFuture`** | executors | compose async pipelines without blocking |
| **`ConcurrentHashMap`** | fine-grained locks + CAS | a shared map with atomic `compute`/`merge` (Collections §11) |
| **`CopyOnWriteArrayList`** | copy-on-write | a shared list, many reads, rare writes |
| **`BlockingQueue`** (`ArrayBlockingQueue`, `LinkedBlockingQueue`) | locks + condition variables | producer-consumer — §4.4 done for you |
| **`CountDownLatch`** | AQS | wait for N tasks to finish |
| **`CyclicBarrier`** | locks/conditions | N threads wait for each other at a rendezvous |
| **atomics** | CAS | lock-free counters/refs (§5) |
| **virtual threads** (Java 21) | JVM scheduling | millions of cheap threads for I/O-bound work |

Most of these are built on the **AbstractQueuedSynchronizer (AQS)** — the internal framework Java uses to implement locks, latches, and semaphores uniformly. You won't touch AQS, but it's why they behave consistently.

**The virtual-threads shift** (Modern Java §12.2) is the big recent change and worth restating here for its *concurrency* meaning: virtual threads are so cheap you can go back to the simple "one thread per task, write blocking code" model at massive scale, because a blocked virtual thread costs almost nothing (the JVM unmounts it). This largely dissolves the reason async/reactive frameworks existed *for I/O-bound work* — you get scalability without the callback complexity. It does **not** help CPU-bound work (you still have N cores).

> **The tell — Part 6:** the correct amount of hand-written locking in modern application code is close to zero. Producer-consumer → `BlockingQueue`. Shared map → `ConcurrentHashMap`. Counter → atomic. Task management → `ExecutorService`. I/O-bound scale → virtual threads. If you're writing `synchronized` and `wait()`, ask whether a `java.util.concurrent` class already does it correctly.

---

# Part 7 — Patterns for correct concurrency

The strategies, in the order you should prefer them. The first two are the point: **the best concurrency has no locks because it has no shared mutable state.**

## 7.1 Immutability — safety by construction

An immutable object can be shared freely across any number of threads with **zero synchronisation**, because there's nothing to race — no thread can change it, so every thread sees a consistent value. This is why records, `final` fields, and defensive copies (Modern Java §11.3) are a *concurrency* technique, not just a style. Make it immutable and the entire taxonomy of Part 2 simply doesn't apply.

```java
public record ScoredQuestion(Long id, int score) {}   // shareable across threads, no locks, ever
```

## 7.2 Confinement — don't share in the first place

If state is only ever touched by one thread, there's no concurrency problem. **Thread confinement** keeps mutable state local to a thread (`ThreadLocal`, or just local variables — a method's locals live on that thread's stack and are never shared). **The Practiq-relevant corollary:** a Micronaut (or Spring) bean is a **singleton by default**, shared across all request threads. A mutable *field* on that singleton is shared mutable state, touched concurrently by every request — the single most common concurrency bug in a web app. Keep beans **stateless**; put per-request state in local variables, not fields.

```java
@Singleton
class QuestionService {
    private int lastId;                       // BUG: shared across all request threads
    void handle(Request r) { lastId = r.id(); ... }   // races between requests

    void handleOk(Request r) { int id = r.id(); ... } // FINE: local, confined to this thread
}
```

## 7.3 The classic coordination patterns

- **Producer-consumer:** producers put work on a `BlockingQueue`, consumers take it. Decouples rates, smooths bursts, and the queue handles all the coordination.
- **Fork-join / divide-and-conquer:** split a big task into subtasks, run in parallel, combine (the `ForkJoinPool` behind parallel streams).
- **Read-copy-update / copy-on-write:** for read-heavy data, readers touch an immutable snapshot while writers build a new one (`CopyOnWriteArrayList`).
- **Guarded suspension:** wait until a condition holds before proceeding (§4.4 — but use a `BlockingQueue`).

> **The tell — Part 7:** the hierarchy of preference is **immutability → confinement → safe concurrent structures → explicit locks**, in that order, and you should rarely reach the last one. "How do I lock this correctly?" is often the wrong question; "why is this shared and mutable at all?" is the right one.

---

# Part 8 — Deadlock, avoided

Deadlock needs all four Coffman conditions (§2.4); break any one and it can't happen. The practical techniques:

- **Lock ordering (breaks circular wait).** The single most effective rule: **always acquire multiple locks in the same global order**, everywhere. If every thread grabs `lock1` before `lock2`, the cycle in §2.4 is impossible. Where locks have no natural order, order by identity hash.
- **Lock timeouts / `tryLock` (breaks hold-and-wait).** `ReentrantLock.tryLock(timeout)` lets a thread give up rather than wait forever — acquire what you can, and if you can't get everything, release and retry. This is why `ReentrantLock` beats `synchronized` for multi-lock code.
- **Coarser locking (breaks the need for multiple locks).** One lock instead of two removes the ordering problem entirely — at a throughput cost. Often the right trade for code that isn't hot.
- **Avoid nested locks / calling foreign code while holding a lock.** Calling a method you don't control while holding a lock is how surprise deadlocks happen — you don't know what it locks.

Detection when it does happen: a **thread dump** (`jstack`, or `kill -3`) shows each thread's held and awaited locks, and the JVM explicitly flags detected deadlocks ("Found one Java-level deadlock"). That's your first move when an app freezes.

> **The tell — Part 8:** if you must hold two locks, establish and document a **global lock order** and never deviate. Better still, restructure so you never hold two at once. When something hangs, take a thread dump *first* — it usually names the deadlock outright.

---

# Part 9 — Reasoning about, testing, and debugging concurrency

## 9.1 The hard truth about testing

**Concurrency bugs are non-deterministic**, so ordinary tests are nearly worthless for finding them. A race that needs a context switch in one exact spot might surface once in ten million runs — passing your test suite a thousand times proves *nothing*. You cannot test your way to concurrent correctness; you must **reason** your way there and use tests as a weak backstop.

The consequence: concurrency correctness is a **design-time** property, established by the techniques in Parts 3–8 (no shared mutable state; happens-before edges where sharing is unavoidable; safe structures), not a test-time one. "It passed" is not evidence.

## 9.2 What actually helps

- **Reason about happens-before (§3).** For every shared field, ask what synchronisation orders the writes and reads. No answer → bug.
- **Minimise the surface.** The less shared mutable state, the less to reason about. Immutability and confinement shrink the problem to near-zero (§7).
- **Stress and fuzz.** Run with many threads, tight loops, artificial delays (`Thread.yield()`/random sleeps) inserted to widen the race window — makes rare interleavings more likely. Tools like **jcstress** (the JVM concurrency stress harness) and race detectors exist for exactly this.
- **Thread dumps for hangs (§8); profilers for contention.** A profiler shows lock contention (threads blocked waiting) — the performance cost of over-synchronising.
- **Static analysis** (`@GuardedBy` annotations, SpotBugs) catches some classes of error at build time.

> **The tell — Part 9:** never trust "it passed the test" for concurrency. Design it correct (immutable/confined/happens-before), then stress it to try to *disprove* your reasoning. When it hangs in prod, thread dump first. And treat any "works 999/1000 times" as a definite race, not flakiness to retry away.

---

# Part 10 — When to use what

## 10.1 Decision framework

**A. Sharing strategy (the first and most important decision).** Tell → immutable: the data doesn't change after construction (records, value objects) → share freely, no locks. Tell → confined: mutable but single-threaded → local variables, `ThreadLocal`, stateless beans. Tell → shared + mutable: unavoidable → the tools below. Default: **make it immutable or confined and skip the rest of this table.**

**B. The synchronisation tool.** Tell → atomic: a single counter/flag/reference → `AtomicX`. Tell → concurrent collection: shared map/list/queue → `ConcurrentHashMap`/`BlockingQueue`. Tell → explicit lock: a genuine multi-step critical section → `ReentrantLock` (or `synchronized`). Tell → semaphore: limiting concurrency to N. Default: **highest-level tool that fits; a `java.util.concurrent` class before a hand-rolled lock.**

**C. Threading model.** Tell → virtual threads: I/O-bound, task-per-request (Java 21+). Tell → fixed pool: CPU-bound work sized to cores. Tell → `CompletableFuture`: composing async stages. Default: **virtual threads for I/O on 21+, bounded pool for CPU work.**

**D. `volatile` vs lock vs atomic.** Tell → `volatile`: publish a single flag/reference (visibility only, no compound update). Tell → atomic: lock-free read-modify-write of one value. Tell → lock: a multi-variable invariant that must update together. Default: **`volatile` for a flag, atomic for a counter, lock for an invariant across several fields.**

## 10.2 The questions worth being able to answer

Use these as a self-check, or as the things to probe when reviewing someone else's concurrent code. Every one reduces to *understanding the model* rather than reciting APIs:

- **"What's the difference between concurrency and parallelism?"** → §1.1 (dealing-with vs doing).
- **"Why isn't `count++` thread-safe?"** → §2.1, show the read-modify-write interleaving.
- **"What does `volatile` do?"** → §3.3: visibility and ordering (happens-before), *not* atomicity — and give the `count++` counterexample.
- **"`synchronized` vs `ReentrantLock`?"** → §4.2: reentrancy both, but Lock adds tryLock/timeout/interruptible/fairness.
- **"How do you prevent deadlock?"** → §8: break a Coffman condition; lock ordering is the practical one.
- **"How would you make a class thread-safe?"** → §7: *first* ask if it can be immutable or confined; only then reach for synchronisation.

The framing underneath all of them: **the best answer to most "how do I make this thread-safe?" questions is "remove the shared mutable state so I don't have to."** Reaching straight for `synchronized` treats the symptom; reaching for immutability or confinement first removes the problem.

---

# How to expand this

- *Go deeper on the applied Java API:* Modern Java primer §12 (executors, `CompletableFuture`, virtual threads, the primitives).
- *Adjacent foundations:* the hardware/OS layer under all this is Engineer's Map §1; optimistic locking as the same pattern one layer up is JPA reference §2.7; the distributed-systems cousin (where "shared state" becomes "state across machines" and locks become consensus) is Engineer's Map §7.
- *Candidates for their own deep treatment, if useful:* the Java Memory Model formally (happens-before, safe publication, final-field semantics, double-checked locking); the `java.util.concurrent` toolkit in full (AQS, all the synchronisers) with worked examples; virtual threads and structured concurrency end to end for a Micronaut service.

*This is a foundations primer written from stable CS/JVM knowledge; the model (memory, happens-before, CAS, Coffman) doesn't drift. Virtual-thread specifics track Java 21 with the pinning fixes noted in Modern Java §12.2 (Java 24).*
