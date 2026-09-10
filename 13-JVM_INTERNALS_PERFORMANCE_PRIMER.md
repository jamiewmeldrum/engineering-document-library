# JVM Internals, Performance & Profiling — A Primer №13

*What actually happens when you run `java -jar app.jar`: class loading, bytecode, the JIT, memory layout, garbage collection in depth, and how to find out why something is slow. Where №10 §13 gives you the working model of JVM memory and GC, this goes underneath it — the mechanisms, the tooling, and the discipline of measuring rather than guessing.*

The idea that organises the whole document: **the JVM is not an interpreter, and it is not a compiler — it is an adaptive runtime that starts by interpreting and progressively compiles the code that turns out to matter, using information only available at runtime.** That's why Java is slow for the first few seconds and then very fast; why a microbenchmark that ignores warmup produces nonsense; and why the JVM can sometimes beat statically-compiled languages on long-running workloads, because it optimises against *observed* behaviour rather than predicted behaviour.

The second idea, and the one that saves the most wasted effort: **you cannot optimise what you have not measured, and your intuition about where time goes is usually wrong.** Every performance section below ends in the same place — profile first.

Contents:

- **Part 1** — the architecture
- **Part 2** — class loading
- **Part 3** — bytecode and the execution engine
- **Part 4** — the JIT compiler
- **Part 5** — memory layout
- **Part 6** — garbage collection in depth
- **Part 7** — reading GC logs and tuning
- **Part 8** — the JVM in a container
- **Part 9** — profiling
- **Part 10** — benchmarking with JMH
- **Part 11** — the troubleshooting playbook
- **Part 12** — when to use what

## Symptom index

| Symptom | Likely cause | Go to |
|---|---|---|
| `OutOfMemoryError: Java heap space` | leak or genuinely undersized heap | §11.1 |
| `OutOfMemoryError: Metaspace` | class-loading leak (redeploys, dynamic proxies) | §5.3, §11.1 |
| `OutOfMemoryError: unable to create native thread` | thread leak or OS limits | §11.1 |
| Container OOM-killed but heap looked fine | **non-heap memory** — off-heap, threads, metaspace | §8.2 |
| Long unpredictable pauses | GC — wrong collector or heap pressure | §7 |
| High CPU with low throughput | GC thrashing, or a hot loop | §11.2 |
| Slow for 30 seconds after every deploy | JIT warmup | §4.4 |
| `StackOverflowError` | unbounded recursion (or a cycle in serialisation) | §5.4 |
| Memory grows steadily, never released | leak — an unintended reference | §11.1 |
| Benchmark says the code is infinitely fast | dead-code elimination — use JMH | §10 |

---

# Part 1 — The architecture

## 1.1 The pieces

```
┌─────────────────────────────────────────────────────────────┐
│  Class Loader Subsystem                                      │
│    loading → linking (verify, prepare, resolve) → init       │
├─────────────────────────────────────────────────────────────┤
│  Runtime Data Areas                                          │
│    Heap │ Metaspace │ Stacks │ PC registers │ Code cache     │
├─────────────────────────────────────────────────────────────┤
│  Execution Engine                                            │
│    Interpreter │ JIT (C1, C2) │ Garbage Collector            │
├─────────────────────────────────────────────────────────────┤
│  JNI / Native libraries                                      │
└─────────────────────────────────────────────────────────────┘
```

## 1.2 The specification/implementation split

The **JVM Specification** defines behaviour; **HotSpot** (in OpenJDK) is the implementation almost everyone runs, and the one this document describes. Others exist — GraalVM (which can also compile ahead of time to a native binary, №10 §14.3), Eclipse OpenJ9 (lower memory, faster startup), Azul Zing.

Same spec/implementation pattern as JPA/Hibernate and JDBC/pgjdbc (№20 §1.2) — you write against the spec; a particular implementation runs.

---

# Part 2 — Class loading

## 2.1 The phases

**Loading** — find the `.class` bytes and create a `Class` object. **Linking** — *verify* (check the bytecode is safe and well-formed — a genuine security boundary), *prepare* (allocate static fields with default values), *resolve* (turn symbolic references into direct ones, usually lazily). **Initialisation** — run static initialisers and static field assignments, exactly once, thread-safely.

Class loading is **lazy**: a class is loaded when first actively used, which is why a `ClassNotFoundException` can appear minutes into a run rather than at startup.

## 2.2 The loader hierarchy and delegation

**Bootstrap** (core JDK classes) → **Platform** → **Application** (your classpath) → any custom loaders.

The **parent delegation model**: a loader asks its parent before attempting to load a class itself. This is a security and consistency mechanism — you cannot replace `java.lang.String` with your own, because the bootstrap loader always gets asked first.

**Class identity is (name + loader)**, not name alone. Two loaders loading the same class file produce two distinct, incompatible classes — the source of `ClassCastException: com.x.Foo cannot be cast to com.x.Foo`, which looks impossible until you know this rule. It's how application servers isolate deployments, and how OSGi and plugin systems work.

## 2.3 Where it bites

Framework startup (scanning and loading thousands of classes) is a large part of JVM start time — and **avoiding runtime class scanning is precisely what Micronaut's compile-time approach buys** (№14). Metaspace leaks come from repeatedly loading classes without releasing the loader — classic in hot-redeploy environments and with heavy dynamic proxy generation.

---

# Part 3 — Bytecode and the execution engine

## 3.1 Bytecode

`javac` compiles source to **bytecode** — a compact, stack-based instruction set stored in `.class` files. This is the portability layer: bytecode runs on any JVM.

```java
int sum(int a, int b) { return a + b; }
```
```
iload_1        // push a
iload_2        // push b
iadd           // pop two, push sum
ireturn        // return top of stack
```

Inspect it with `javap -c ClassName`. Worth doing once — seeing that a string concatenation in a loop becomes a `StringBuilder` allocation per iteration (№10 §3.2), or that a lambda becomes an `invokedynamic`, makes those abstractions concrete.

Note the JVM is a **stack machine** (operands pushed and popped) rather than a register machine, which keeps bytecode compact and portable at the cost of needing the JIT to map it onto real registers.

## 3.2 Interpretation, then compilation

Execution starts in the **interpreter**: read a bytecode, do it, repeat. Simple, immediate, slow. Meanwhile the JVM **counts** method invocations and loop back-edges, and when a method crosses a threshold it becomes a candidate for **JIT compilation** to native code.

That's the adaptive model: start immediately at low speed, and spend compilation effort only on code that's actually hot.

---

# Part 4 — The JIT compiler

## 4.1 Tiered compilation

HotSpot has two compilers and uses both:

| Tier | Compiler | Character |
|---|---|---|
| 0 | Interpreter | immediate, slow, gathers profile data |
| 1–3 | **C1** (client) | compiles fast, optimises lightly, adds profiling |
| 4 | **C2** (server) | compiles slowly, optimises aggressively |

**Tiered compilation** (the default) runs code through the interpreter, then C1 with profiling, then C2 once it's proven hot and well-profiled. You get fast startup *and* eventual peak performance.

## 4.2 What the JIT actually does

The optimisations worth knowing, because they explain surprising behaviour:

- **Inlining** — the most important. Replacing a call with the method body eliminates call overhead *and*, crucially, exposes the inlined code to every other optimisation. Small hot methods are essentially free.
- **Escape analysis** — if an object provably never escapes its method, the JVM can allocate it on the stack or eliminate it entirely (**scalar replacement**), and remove its locks. This is why "avoid allocation" advice is often obsolete: short-lived objects may never be allocated at all.
- **Devirtualisation** — a call through an interface with only one implementation observed becomes a direct call, then gets inlined. This is why "interfaces are slow" is untrue in practice.
- **Loop optimisations** — unrolling, hoisting invariants, eliminating bounds checks it can prove are unnecessary.
- **Dead code elimination**, constant folding, common subexpression elimination.

## 4.3 Speculation and deoptimisation

The JIT compiles against **observed behaviour**: "this call site has only ever seen `HibernateQuestionRepository`, so devirtualise and inline it." That's a *speculation*, guarded by a check. If a second implementation appears later, the guard fails and the JVM **deoptimises** — discards the compiled code, falls back to the interpreter, and recompiles with the new information.

This is the JVM's genuine advantage over ahead-of-time compilation: it optimises for what actually happens, not what might. It's also why performance can change shape mid-run, and why a code path that's fast in production can be slow in a test that exercises it differently.

## 4.4 Warmup

**Warmup is the practical consequence of all of the above.** A freshly started JVM is interpreting, has an empty code cache, cold caches, an unpopulated heap and unresolved classes. Peak throughput arrives after thousands of iterations.

What follows: **benchmarks must discard warmup** (§10); **the first requests after a deploy are slow**, which matters for canary analysis and health-check timeouts (№56 §8); and for short-lived processes (Lambda, CLI tools) the JVM may *never* reach peak — which is exactly the case GraalVM native image addresses (№10 §14.3), trading peak throughput for instant startup.

Flags worth knowing: `-XX:+PrintCompilation` (what's being compiled), `-XX:-TieredCompilation` (C2 only — occasionally useful for benchmarking), `-XX:ReservedCodeCacheSize` (a full code cache silently disables further compilation, which looks like a mysterious performance cliff).

---

# Part 5 — Memory layout

## 5.1 The regions

| Region | Holds | Shared? | Failure |
|---|---|---|---|
| **Heap** | all objects and arrays | yes | `OutOfMemoryError: Java heap space` |
| **Metaspace** | class metadata | yes | `OutOfMemoryError: Metaspace` |
| **Stack** (per thread) | frames, locals, references | no | `StackOverflowError` |
| **Code cache** | JIT-compiled native code | yes | compilation stops silently |
| **Direct/off-heap** | NIO buffers, Netty | yes | `OutOfMemoryError: Direct buffer memory` |

## 5.2 The heap

Divided generationally (§6.1): **Young** (Eden + two Survivor spaces) and **Old**. Objects are born in Eden, survive into Survivor spaces, and are **promoted** to Old after surviving a few collections.

Object layout: a **header** (mark word — hash, GC age, lock state; plus a class pointer), then fields, aligned to 8 bytes. An empty object costs ~16 bytes. **Compressed oops** (references stored as 32 bits) apply below a ~32 GB heap — which produces the counter-intuitive result that a 31 GB heap can hold *more* than a 33 GB one, because crossing the boundary makes every reference twice the size.

## 5.3 Metaspace

Class metadata, held in **native memory** (it replaced PermGen in Java 8, which was on-heap and fixed-size). It grows dynamically by default, which means a class-loading leak consumes native memory until the machine complains. Cap it in production: `-XX:MaxMetaspaceSize=256m` turns an unbounded leak into a clear, early error.

## 5.4 Stacks

One per thread, holding a frame per call. Default ~1 MB (`-Xss`). Deep or unbounded recursion produces `StackOverflowError` (№30 §3.3). Note the arithmetic that matters in containers: **thread count × stack size is native memory outside your heap** — 500 threads at 1 MB is 500 MB the JVM never counted as heap (§8.2).

---

# Part 6 — Garbage collection in depth

## 6.1 The generational hypothesis

**Most objects die young.** Empirically, the overwhelming majority of allocations become garbage almost immediately (loop temporaries, intermediate strings, short-lived DTOs), while a small proportion live for the process's lifetime (caches, connection pools, singletons).

The consequence is the design of every mainstream collector: collect the young generation **often and cheaply** (copy the few survivors, discard the rest — cost is proportional to *survivors*, not to garbage), and the old generation **rarely and expensively**.

## 6.2 How collection actually works

**Reachability, not reference counting.** From a set of **GC roots** (thread stacks, static fields, JNI references), the collector marks everything reachable; whatever is unmarked is garbage. This is why cycles are collected automatically and why a "memory leak" in Java is always **an unintended reference keeping something reachable** — a growing static map, a cache with no eviction, an unclosed resource, a listener never deregistered.

**Copying collection** for the young generation: live objects are copied to a survivor space and the whole Eden region is then free. It's fast *and* it compacts, which is why allocation is a pointer bump.

**Stop-the-world pauses**: some phases require application threads to stop at a **safepoint**. Modern collectors do most work concurrently and minimise these, but they cannot be eliminated entirely.

**Write barriers** are the hidden cost that makes concurrent collection possible: small snippets of code inserted at every reference write so the collector can track cross-generational and concurrent mutations. They're why GC has a throughput cost even when not collecting.

## 6.3 The collectors

| Collector | Model | Pause | Throughput | Use for |
|---|---|---|---|---|
| **Serial** | single-threaded, stop-the-world | high | low | tiny heaps, single-CPU containers |
| **Parallel** | multi-threaded, stop-the-world | high | **highest** | batch jobs where pauses don't matter |
| **G1** *(default)* | region-based, concurrent marking, incremental compaction | moderate, targetable | good | **the default — leave it alone** |
| **ZGC** | concurrent, coloured pointers, load barriers | **<1ms**, heap-size-independent | slight cost | latency-critical, large heaps |
| **Shenandoah** | concurrent, Brooks pointers | very low | slight cost | similar niche |
| **Epsilon** | doesn't collect | n/a | n/a | testing allocation behaviour only |

**G1** divides the heap into equal regions (typically 1–32 MB), each dynamically young or old. It marks concurrently and then collects the regions with the most garbage first — hence "Garbage First". It targets a pause goal (`-XX:MaxGCPauseMillis`, default 200ms) and adapts its work to try to meet it. That target is a *goal*, not a guarantee.

**ZGC** achieves sub-millisecond pauses **independent of heap size** using coloured pointers (metadata stored in unused pointer bits) and load barriers, doing essentially all work concurrently. It's now generational by default. Choose it when you have *measured* pause problems — it costs a little throughput and memory.

## 6.4 The allocation path

Fast: a thread allocates from its own **TLAB** (thread-local allocation buffer) by bumping a pointer — no synchronisation. When the TLAB is exhausted it gets a new one; when Eden is full, a **minor GC** runs. Large objects may go straight to Old (or to G1's humongous regions).

This is why allocation in Java is cheap — cheaper than `malloc` — and why "object pooling to avoid allocation" is usually counterproductive on a modern JVM: pooled objects survive, get promoted, and make the *expensive* generation bigger.

---

# Part 7 — Reading GC logs and tuning

## 7.1 Turn logging on — always

```bash
-Xlog:gc*:file=/var/log/gc.log:time,uptime,level,tags:filecount=5,filesize=20M
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/log/
```

GC logging is nearly free and it's the difference between diagnosing a memory problem in minutes and guessing for days. **Enable it in production before you need it** — the same argument as SQL logging (№20 §3.1) and observability generally (№57).

## 7.2 What to look for

Three numbers tell you most of the story:

- **Pause duration and frequency.** Occasional short young collections are healthy.
- **Heap occupancy *after* full collections.** If it climbs steadily across collections and never comes down, you have a **leak** — the collector is doing its job and the memory genuinely isn't garbage.
- **Time spent in GC.** Above ~5–10% of wall-clock is a problem. Approaching 90–98% with almost nothing reclaimed is the **death spiral** that precedes `OutOfMemoryError` — the JVM is repeatedly collecting and freeing nothing.

Tools: **GCViewer**, **GCEasy** (paste a log, get analysis), or JFR (§9.2).

## 7.3 Tuning, in order

The honest ordering, because most "GC tuning" is misdirected effort:

1. **Fix the allocation problem first.** Most GC pain is an application problem — an N+1 loading 10,000 entities (№20 §2.8), an unbounded cache, string concatenation in a loop. **Profile before tuning** (§9).
2. **Set the heap sensibly.** `-Xms` equal to `-Xmx` in a container avoids resize churn; use `MaxRAMPercentage` (§8).
3. **Leave G1 alone** unless you have measured evidence.
4. **Change the collector** only if the workload genuinely demands it (ZGC for measured pause problems, Parallel for throughput batch).
5. **Individual flags last**, one at a time, measured.

> **The tell — GC:** do not tune GC. Measure what's allocating, and fix that. The number of production problems solved by an obscure GC flag is far smaller than the number solved by removing an unnecessary allocation.

---

# Part 8 — The JVM in a container

## 8.1 Container awareness

Historically the JVM read the *host's* CPU and memory and sized itself accordingly — catastrophic in a container limited to 512 MB, because it would plan for the host's 32 GB and get OOM-killed. Modern JVMs (10+) read **cgroup limits** (№50 §2.3) and size the heap and available-processor count from the container's constraints.

## 8.2 The sizing trap that catches everyone

```bash
-XX:MaxRAMPercentage=75.0            # of the CONTAINER limit
```

**The JVM uses substantially more memory than its heap.** Total footprint is heap **plus** metaspace, thread stacks (count × ~1 MB), the code cache, direct/NIO buffers, GC structures and JVM overhead. Set `-Xmx` to the full container limit and you *will* be OOM-killed with a heap that looks healthy — the kernel kills the process, and nothing in your application log explains it (`dmesg`, №52 §8.6).

Hence 75% as a starting point, leaving headroom for the rest. **Native Memory Tracking** (`-XX:NativeMemoryTracking=summary`, then `jcmd <pid> VM.native_memory summary`) breaks down where non-heap memory has actually gone when you need to know.

Also worth setting in a container: `-XX:MaxMetaspaceSize` (bound the leak, §5.3), and `-XX:ActiveProcessorCount` if the JVM's CPU detection disagrees with your cgroup quota — it drives GC thread counts and `ForkJoinPool` sizing.

---

# Part 9 — Profiling

## 9.1 The discipline

**Measure, don't guess.** Every experienced engineer has a story about optimising the wrong thing for a week. And measure **in an environment that resembles production** — heap size, data volume, concurrency and warmup state all change the answer.

The order of questions: *is it actually slow?* (metrics, №57) → *where does the time go?* (traces, then a profiler) → *why?* (profile detail) → *fix* → **measure again**.

And keep perspective on where the time usually is: in a service like Practiq, the answer is almost always **I/O — database round trips, network calls** (№00 §1.2, №31) — not CPU in your Java code. Profile the whole request before micro-optimising a loop.

## 9.2 JDK Flight Recorder

**JFR is the first tool to reach for.** It's built into the JDK, designed for continuous production use with very low overhead (~1%), and records allocation, GC, locks, I/O, exceptions, threads and custom events.

```bash
# start with the app
-XX:StartFlightRecording=duration=60s,filename=app.jfr,settings=profile

# or attach to a running process
jcmd <pid> JFR.start duration=60s filename=/tmp/app.jfr settings=profile
jcmd <pid> JFR.dump filename=/tmp/app.jfr
```

Open the recording in **JDK Mission Control** for hot methods, allocation sites, GC behaviour, lock contention and I/O.

## 9.3 async-profiler and flame graphs

**async-profiler** avoids the safepoint bias that afflicts traditional sampling profilers (which can only sample at safepoints, systematically missing some code). It samples CPU, allocation, locks and native frames, and produces **flame graphs**:

```bash
./profiler.sh -d 60 -f flame.html <pid>
```

Reading a flame graph: **width = time spent** (self plus children), stacked bars = call depth. You scan for the *widest* frames, not the tallest — the tall towers are just deep call stacks, the wide plateaus are where time actually goes. It's the fastest way to answer "what is this process doing?"

## 9.4 The rest of the toolkit

```bash
jcmd <pid> help                      # everything available on a running JVM
jcmd <pid> Thread.print              # thread dump (= kill -3, №12 §8)
jcmd <pid> GC.heap_info
jcmd <pid> VM.native_memory summary  # if NMT enabled
jmap -histo:live <pid> | head -30    # objects by count/size — quick leak triage
jmap -dump:live,format=b,file=heap.hprof <pid>
jstat -gcutil <pid> 1000             # GC stats every second
```

**Heap dump analysis** with Eclipse MAT is the definitive leak tool: its **dominator tree** shows what's retaining memory, and "Path to GC Roots" tells you *which reference* is keeping an object alive — which is the actual answer to a leak (§6.2).

---

# Part 10 — Benchmarking with JMH

## 10.1 Why naive benchmarks lie

```java
long start = System.nanoTime();
for (int i = 0; i < 1_000_000; i++) { result = compute(i); }
System.out.println(System.nanoTime() - start);     // meaningless
```

Everything about this is wrong: it measures the **interpreter and C1** as much as the final compiled code (§4.4); the JIT may **eliminate the whole loop** as dead code if the result is unused; **constant folding** may compute it once; GC may run mid-measurement; and `System.nanoTime()` has its own overhead. Naive benchmarks routinely report code as thousands of times faster than reality, or infinitely fast.

## 10.2 JMH

The JDK's harness, which handles warmup, forking, dead-code elimination and statistics:

```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
@Fork(2)
public class DifficultyBenchmark {

    private List<Question> questions;

    @Setup public void setup() { questions = generate(1000); }

    @Benchmark
    public double averageDifficulty(Blackhole bh) {
        double avg = questions.stream().mapToInt(Question::getDifficulty).average().orElse(0);
        bh.consume(avg);          // ← stops the JIT eliminating the work
        return avg;
    }
}
```

The essentials: **`Blackhole`** consumes results so they can't be optimised away; **`@Fork`** runs in fresh JVMs so profiles don't leak between benchmarks; **warmup iterations** are discarded; and results come with error bars — **a difference inside the error margin is not a difference.**

## 10.3 The caveats

A microbenchmark measures a method in isolation, which is not how it behaves inside a real application (different inlining decisions, cache pressure, contention). It answers "is A faster than B in isolation?" — useful for a hot inner loop, misleading as a proxy for system performance. **For system performance, load-test the system** (k6, Gatling) and profile it under that load.

---

# Part 11 — The troubleshooting playbook

## 11.1 OutOfMemoryError

Read the message — it names the region:

- **`Java heap space`** → a leak, or genuinely too small. Get a heap dump (auto-generated if you set the flag, §7.1), open it in MAT, examine the dominator tree, find the path to GC roots. Usual suspects: an unbounded cache or static collection, a listener never removed, a `ThreadLocal` never cleared on a pooled thread, an unclosed resource.
- **`Metaspace`** → a class-loading leak: repeated redeploys, dynamic proxy generation, or a class loader retained by a stray reference.
- **`unable to create native thread`** → thread leak, or OS/cgroup limits. Take a thread dump and count.
- **`Direct buffer memory`** → off-heap NIO buffers not being released.
- **`GC overhead limit exceeded`** → the death spiral (§7.2): collecting constantly, reclaiming almost nothing.

## 11.2 High CPU

Find the offending thread, not just the process:

```bash
top -H -p <pid>                       # threads by CPU; note the highest TID
printf '%x\n' <tid>                   # convert to hex — thread dumps use hex nids
jcmd <pid> Thread.print | grep -A 30 <hex-tid>
```

Or skip the arithmetic and take a flame graph (§9.3), which usually names the culprit immediately. Common causes: a hot loop, GC thrashing (check GC logs first — high CPU with low throughput is classically GC), regex backtracking, or excessive logging.

## 11.3 Hangs and slowness

**Thread dump first** (`jcmd <pid> Thread.print` or `kill -3`), and take **three, seconds apart** — comparing them shows whether threads are stuck or merely busy. The JVM explicitly reports detected deadlocks (№12 §8). Look for many threads blocked on the same lock (contention), or all threads waiting on a connection pool (pool exhaustion — №20 §1.6, and a far more common cause of "the app is hanging" than anything JVM-level).

## 11.4 Memory that grows and never shrinks

Distinguish a leak from a large-but-stable working set: watch heap occupancy **after full GCs** over time (§7.2). Rising baseline = leak. Stable baseline = you just need a bigger heap. Compare two heap dumps taken an hour apart to see which object counts grew.

---

# Part 12 — When to use what

**A. Which collector?** Tell → G1: everything, unless proven otherwise. Tell → ZGC: measured pause problems, large heaps, latency-critical. Tell → Parallel: batch throughput where pauses are irrelevant. Tell → Serial: tiny single-CPU containers. Default: **G1, untouched.**

**B. Which profiler?** Tell → JFR: production, continuous, low overhead, broad picture. Tell → async-profiler: CPU/allocation detail and flame graphs. Tell → MAT + heap dump: memory leaks. Tell → JMH: comparing two implementations of a hot method. Default: **JFR first, flame graph next, MAT for leaks.**

**C. Tune or fix the code?** Tell → tune: you've profiled and the allocation is genuinely necessary. Tell → fix: almost always — the cause is an N+1, an unbounded cache, or a hot loop. Default: **fix the code.**

**D. Heap sizing in a container?** Tell → `MaxRAMPercentage=75`: the sane default. Tell → explicit `-Xmx`: you've measured the full native footprint. Default: **percentage-based, leaving headroom for non-heap** (§8.2).

**E. JVM or native image?** Tell → JVM: long-running services where peak throughput and JIT adaptivity win. Tell → native image: startup time and memory dominate (Lambda, CLI, scale-to-zero) and you can accept the build cost and reflection constraints. Default (Practiq): **JVM for the API; native image is a worthwhile spike given Micronaut** (№10 §14.3, №14).

**F. Micro-benchmark or load test?** Tell → JMH: comparing two implementations of one method. Tell → load test: does the *system* meet its requirements. Default: **load test for system questions; JMH only for genuinely hot code.**

**G. When to care about any of this?** Tell → now: you have a measured performance or memory problem. Tell → later: you don't. Default: **enable GC logging and heap-dump-on-OOM in production, then ignore all of this until something tells you otherwise.**

---

# How to expand this

- *Related:* №10 §13 (the working model this deepens), №12 (concurrency — thread dumps, contention, the memory model), №50 §2.3 (cgroups, the constraint the JVM reads), №52 §8 (Linux-level inspection and OOM kills), №57 (production observability), №20 §2.8 (the N+1 that causes most "GC problems").
- *Candidates for deeper treatment:* **a leak-hunting walkthrough** with a real heap dump in MAT; **reading a G1 log line by line**; **GraalVM native image on Micronaut** with real numbers and the reflection-configuration story; **JMH properly** (state scopes, parameterised benchmarks, interpreting variance).

*Mechanisms are stable — class loading, tiered JIT, generational GC and the memory regions don't drift. Defaults and collector capabilities do change between releases (ZGC became generational; G1 gains improvements most versions), so check the release notes for the JDK you're actually running.*
