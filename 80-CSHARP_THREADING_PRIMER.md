# C# Threading and Asynchrony — A Primer №80

*Covers the .NET concurrency model from the OS thread upwards: processes and threads, the thread pool, `Task`, `async`/`await` and the state machine behind it, cancellation, the memory model and atomic operations, locks and coordination primitives, channels, and data parallelism. Deliberately excludes distributed concurrency, which is №31's subject; actor frameworks such as Orleans; Rx/`IObservable`; and desktop UI dispatching beyond the minimum needed to explain why `SynchronizationContext` exists. The lens is server-side .NET on ASP.NET Core, running on .NET 10.*

*Its relationship to **№12 Concurrency** is worth stating plainly, because they overlap. №12 teaches the ideas: races, happens-before, the bug taxonomy, why you cannot test concurrency into correctness. Those ideas are true in every language and are not repeated here. This document teaches **what .NET actually does**: how its thread pool schedules, what its memory model guarantees (which is stronger than the JVM's in one specific and useful way, §5.8), and how the `async`/`await` transform works. Read №12 first if the concepts are shaky; read this one if the concepts are fine and the platform is new.*

---

## The organising idea

Nearly everyone learns `await` as a way of waiting, and nearly every serious .NET concurrency bug follows from that misreading. It isn't a wait.

**`await` is a `return`. When your method hits an `await` on something that hasn't finished, it returns to its caller immediately, leaving behind a callback describing what to do with the rest of the method when the result arrives. There is no thread sitting anywhere. The thread that was running your method has gone off to do something else entirely, and the "rest of the method" will be run later, possibly by a different thread.**

Once you hold that, the rest of the model falls out of it. The compiler has to chop your method into resumable pieces, so it builds a state machine (§8.1). The method has to return *something* to its caller before it has a result, so it returns a `Task`, which is a handle on a result that doesn't exist yet (§2.1). If the method returns `void` there is no handle, so nobody can observe when it finishes or that it threw, which is why `async void` takes the process down (§3.3). If the caller blocks on that `Task` instead of awaiting it, the thread that was supposed to run the continuation is the very thread that's blocked, and you have a deadlock (§3.4). Local variables have to survive across the gap, so they get hoisted onto the heap (§8.1). And because the continuation runs later, "later" needs a policy about *where*, which is `SynchronizationContext` and `ConfigureAwait` (§3.5).

## The second idea

**There are three different problems here wearing similar clothing, and .NET gives you a different toolkit for each. Concurrency is about not wasting threads while nothing is happening: that's `async`/`await`, and it uses fewer threads, not more. Parallelism is about finishing CPU work sooner by using more cores: that's the thread pool, `Parallel`, and PLINQ, and it uses more threads. Shared mutable state is about two threads touching the same memory at once: that's locks, `Interlocked`, `volatile` and immutability, and neither of the other two toolkits helps with it at all.**

The corollary that removes fear: most application code needs only the first, and most of what you'll read about the third can be replaced by "don't share mutable state". A web request handler that awaits a database call and an HTTP call is doing nothing exotic. You do not need `Thread`, you almost never need `Parallel`, and if you find yourself reaching for `volatile` you are usually one design change away from not needing it. The hard parts of this document exist so you can recognise the rare cases where you do, and so you can read other people's code that reached for them wrongly.

---

## Contents

| Part | Covers |
|---|---|
| **1. The substrate** | OS threads, processes, the thread pool, work stealing, thread injection, starvation |
| **2. `Task`** | The promise model, creation, composition, exceptions, the `StartNew` trap |
| **3. `async`/`await`** | What the keywords mean, `async void`, sync-over-async deadlocks, contexts, `ValueTask`, async streams |
| **4. Cancellation** | The cooperative model, tokens, linked sources, timeouts, ASP.NET Core integration |
| **5. Memory and atomicity** | Reordering, tearing, `volatile`, `Interlocked`, compare-and-swap, barriers, ARM64 |
| **6. Locks and coordination** | `lock` and `System.Threading.Lock`, async mutual exclusion, the primitive zoo, concurrent collections, channels |
| **7. Parallelism** | `Parallel`, PLINQ, partitioning, false sharing, and why this is usually wrong on a server |
| **8. Internals** | The generated state machine, `TaskCompletionSource`, custom awaitables, schedulers, runtime async |
| **9. Practice on a server** | DI lifetimes, background services, fire-and-forget, EF Core, diagnosing it in production |
| **10. When to use what** | Decision axes |

---

## The symptom index

Use this when something is already wrong.

| When you notice… | Go to |
|---|---|
| Latency collapses under load but CPU sits near idle | §1.5 thread pool starvation |
| It works in a console app or a unit test, and hangs in the real app | §3.4 sync-over-async |
| The process died with an unhandled exception and no stack trace you recognise | §3.3 `async void` |
| A bug that only reproduces on ARM64 (Graviton, Ampere, Apple silicon) | §5.9 |
| A counter is occasionally short by a few | §5.3 read-modify-write |
| `ConcurrentDictionary.GetOrAdd` ran your factory twice | §6.7 |
| `InvalidOperationException` from a plain `Dictionary` or `List` under load | §6.7 |
| "A second operation was started on this context" from EF Core | §9.4 |
| A `Task` never completes and nothing is obviously blocked | §3.4, §8.2 |
| A `SemaphoreSlim` throws `SemaphoreFullException` | §6.3 |
| Cancellation is requested and the work carries on regardless | §4.1 |
| Stack traces are a wall of `MoveNext` frames | §8.1, §8.6 |
| Memory climbs steadily and the heap is full of queued work items | §6.8 unbounded channels |
| A `Timer` callback re-entered itself | §4.6 |
| `Task.Run` in a request handler made throughput worse, not better | §7.4 |
| A field written by one thread is never seen by another, forever | §5.4 `volatile` |
| Startup hangs after adding a `BackgroundService` | §9.3 |
| Two threads deadlock on two locks | §6.6 |
| `HttpClient` throws `SocketException`/port exhaustion under load | §9.5 |

---

# Part 1 — The substrate: processes, threads, and the pool

## 1.1 What a `Thread` actually is

A .NET `Thread` is a real operating system thread. There is no green-thread or virtual-thread layer in .NET: `new Thread(...)` results in a `pthread_create` on Linux or a `CreateThread` on Windows, and the OS scheduler, not the CLR, decides when it runs. That has three consequences worth internalising.

It is expensive. Each thread reserves stack space (1 MB by default on Windows, typically 8 MB of *reserved* address space on Linux with pages committed lazily), plus kernel bookkeeping. Creating one costs on the order of a hundred microseconds; that sounds small until you do it per request.

It is preemptively scheduled. Your thread can be suspended between any two machine instructions, including in the middle of a `long++`. Everything in §5 exists because of this.

Context switching between threads costs a few microseconds of direct cost plus a much larger indirect cost in cache pollution. Two hundred runnable threads on eight cores do not do more work than eight; they do less, and with worse latency.

```csharp
var t = new Thread(Work) { IsBackground = true, Name = "importer" };
t.Start();
```

`IsBackground` is the flag that catches people out. A **foreground** thread keeps the process alive: `Main` can return and the process will sit there until every foreground thread finishes. A **background** thread is killed abruptly at process exit with no `finally` blocks run. `new Thread(...)` defaults to foreground. Every thread pool thread is background. If a console app refuses to exit, a stray foreground thread is the first suspect.

Two APIs from the .NET Framework era are gone and will not come back. `Thread.Abort` throws `PlatformNotSupportedException` on .NET Core and later, because injecting an exception at an arbitrary instruction cannot leave shared state consistent. `Thread.Suspend`/`Resume` are likewise unsupported. There is no way to stop a thread from outside; cooperative cancellation (§4) is the only mechanism, and it is a mechanism precisely because the alternative was unsound.

> **The tell — creating a thread:** create a dedicated `Thread` only when the work is long-lived (minutes to the process lifetime), needs a non-default stack size, needs a specific priority or apartment state, or must not be subject to pool scheduling. For everything else, use the pool. In practice, in modern server code, the honest count of legitimate `new Thread` calls is close to zero.

## 1.2 Processes

`System.Diagnostics.Process` starts and observes OS processes. .NET has no in-process isolation boundary any more: `AppDomain` was a .NET Framework feature and does not exist on .NET Core and later, so the only way to isolate code that might corrupt memory, leak native resources, or need killing is a separate process. `AssemblyLoadContext` gives you assembly isolation and unloadability, but it is not a security or fault boundary; a stack overflow or an `AccessViolationException` still takes the whole process down.

The practical reasons to reach for a process rather than a thread: running untrusted or plugin code, wrapping a native tool, needing a hard memory ceiling per unit of work, or needing to kill a runaway computation. The cost is that you have lost shared memory and must now serialise across a pipe, a socket, or a file, which is a different and usually larger engineering problem.

```csharp
using var p = Process.Start(new ProcessStartInfo("ffmpeg", "-i in.mp4 out.webm")
{
    RedirectStandardError = true,
    UseShellExecute = false
});
// BROKEN: WaitForExit before draining a redirected stream can deadlock
// when the child fills the 4 KB pipe buffer and blocks on write.
// p.WaitForExit();

// FIXED: consume the stream, then wait.
string stderr = await p.StandardError.ReadToEndAsync();
await p.WaitForExitAsync();
```

## 1.3 Delegates: what "some work" is made of

Before the pool, a note on the types the pool traffics in, since C#'s delegate family confuses people arriving from languages with a single function type.

`Action` is a delegate returning `void`; `Action<T>`, `Action<T1,T2>` and so on take up to sixteen parameters. `Func<TResult>` returns a value; `Func<T,TResult>` takes an argument and returns a value, with the return type always last in the type argument list. `Predicate<T>` is a legacy alias for `Func<T,bool>`. These are ordinary reference types: a delegate instance is an object holding a method pointer plus a target instance (for instance methods) or null (for statics).

Two things matter for concurrency. First, a lambda that captures a variable does not capture its value, it captures the variable, hoisted into a compiler-generated closure class on the heap. Capture a loop variable in a `for` and every queued delegate sees the same mutated field. (`foreach` variables are per-iteration since C# 5, so they are safe; `for` variables are not.)

```csharp
// BROKEN: all five work items may print 5.
for (int i = 0; i < 5; i++)
    ThreadPool.QueueUserWorkItem(_ => Console.WriteLine(i));

// FIXED: give each iteration its own variable to capture.
for (int i = 0; i < 5; i++)
{
    int local = i;
    ThreadPool.QueueUserWorkItem(_ => Console.WriteLine(local));
}
```

Second, that closure object is shared mutable state by construction. If two queued delegates capture the same local and both write to it, you have the problem in §5 without having written a single `static` field.

## 1.4 The thread pool

The .NET thread pool is a process-wide singleton, reachable statically (`ThreadPool.QueueUserWorkItem`) and used implicitly by almost everything: `Task.Run`, timers, async continuations, socket completions, ASP.NET Core request dispatch. Since .NET 6 the default implementation is the "portable thread pool", written in managed code rather than the native CLR pool, on all platforms.

Its structure is worth knowing because its failure mode is a production incident.

**Queues.** There is one global queue plus a local queue per pool thread. Work queued from *outside* the pool goes to the global queue. Work queued from *inside* a pool thread goes to that thread's local queue, which is LIFO: the most recently queued item is taken first, because it is the most likely to be cache-hot and the most likely to be a continuation of what that thread was just doing. When a thread's local queue is empty it takes from the global queue, and when that is empty it **steals** from the tail of another thread's local queue. This is why fine-grained task graphs work well without central contention.

**Thread injection.** The pool guarantees it will run `MinThreads` threads without delay; the default is the processor count. Beyond that, if all threads are busy and work is still queued, the pool adds threads slowly, at a rate Microsoft documents as **1 to 2 threads per second**. It also runs a hill-climbing controller that periodically nudges the thread count up or down and measures whether completed-work throughput improved, keeping changes that help. The slow injection is deliberate: if the queue is backed up because of a transient burst, adding threads is waste, and if it is backed up because threads are blocked, adding threads is a treatment for the symptom.

**Since .NET 6** the pool also detects blocking in certain `Task` APIs (`Task.Wait`, `.Result`, `Task.WaitAll`) and injects threads faster in response. This makes bad code degrade more gracefully; it does not make it good.

## 1.5 Thread pool starvation

This is the single most common serious .NET performance failure, and its signature is unmistakable once you have seen it: **latency goes vertical while CPU utilisation stays low**. That combination means threads are not computing, they are blocked, and requests are queueing behind a pool that will only grow at one or two threads per second.

The cause is almost always **sync-over-async**: calling an asynchronous method and blocking on the result.

```csharp
// BROKEN: occupies a pool thread for the entire duration of the I/O.
public ActionResult<Customer> Get(int id)
{
    return _repo.GetAsync(id).Result;
}

// FIXED: the pool thread is returned to the pool for the duration of the I/O.
public async Task<ActionResult<Customer>> Get(int id)
{
    return await _repo.GetAsync(id);
}
```

The arithmetic is brutal. Suppose each request does one 100 ms database call, and the pool has 8 threads. Awaiting: each thread handles roughly 10 requests per second of *waiting* plus whatever CPU work remains, so hundreds of requests per second are fine. Blocking: 8 threads × 10 requests/second = 80 requests/second, and request 81 waits for a thread that the pool will grant at a rate of one per second. At 200 requests/second you need about 20 threads; the pool takes ten seconds to get there, and by then the queue holds two thousand requests and your load balancer has started health-checking you out of the pool.

Raising `ThreadPool.SetMinThreads` is the standard emergency mitigation and it does work: it removes the injection delay for the first N threads. It is a tourniquet, not a fix, it hides the real problem, and it costs memory and context switches. Fix the blocking call.

> **The tell — starvation vs saturation:** if latency is bad and CPU is high, you have a CPU problem: profile and optimise. If latency is bad and CPU is low, you have a blocking problem: find the `.Result`, `.Wait()`, `.GetAwaiter().GetResult()`, or synchronous I/O on a pool thread. Confirm it with `dotnet-counters monitor --counters System.Runtime` and watch `ThreadPool Queue Length` and `ThreadPool Thread Count`: a queue length that climbs while thread count crawls upward is starvation, conclusively.

---

# Part 2 — `Task`: the unit of pending work

## 2.1 What a `Task` is

A `Task` is a **promise**: an object representing an operation that will eventually complete, fail, or be cancelled, with a list of continuations to run when it does. It is not a thread, it does not imply a thread, and in most server code no thread is associated with it at all.

Its state machine has six values (`TaskStatus`), but only the three terminal ones matter day to day: `RanToCompletion`, `Faulted`, `Canceled`. A task in a terminal state is immutable forever; awaiting it again is free and returns the same result.

The two ways a `Task` comes into existence are worth separating in your head, because they behave completely differently:

- **Compute-bound**: `Task.Run(() => Compute())` schedules a delegate on the thread pool. There is a thread, and it is busy.
- **I/O-bound / promise-style**: an async method or a `TaskCompletionSource` (§8.2) produces a `Task` that will be completed later by something else, usually an I/O completion callback from the OS. There is no thread.

The single most useful sentence about `Task` is that **the type tells you nothing about which of those it is**. `File.ReadAllTextAsync` and `Task.Run(() => File.ReadAllText(path))` have the same type and wildly different costs. This is why the guidance "don't wrap synchronous work in `Task.Run` inside a library" exists: you are lying to your caller about the cost model.

## 2.2 `Task.Run` versus `Task.Factory.StartNew`

`Task.Factory.StartNew` is the older API and it is a trap in three separate ways. Use `Task.Run` unless you can articulate why not.

| | `Task.Run` | `Task.Factory.StartNew` |
|---|---|---|
| Scheduler | Always `TaskScheduler.Default` (the pool) | `TaskScheduler.Current`, which may be something else entirely |
| Attach to parent | Never | Possible, and surprising |
| `Func<Task>` overload | Unwraps: returns the inner task | Returns `Task<Task>`: completes when the delegate *starts* the inner task |
| Long-running hint | Not available | `TaskCreationOptions.LongRunning` |

The unwrapping difference is the killer:

```csharp
// BROKEN: outer task completes immediately; exceptions and results are lost.
Task outer = Task.Factory.StartNew(async () => await DoWorkAsync());
await outer;               // returns as soon as DoWorkAsync has *started*

// FIXED, either:
await Task.Run(async () => await DoWorkAsync());
// or, if you must use StartNew:
await Task.Factory.StartNew(async () => await DoWorkAsync()).Unwrap();
```

The one thing `StartNew` still offers is `TaskCreationOptions.LongRunning`, which asks the scheduler for a dedicated thread rather than a pool thread. That is the right call for a loop that will run for the process lifetime, because parking it on a pool thread permanently removes that thread from the pool.

> **The tell — `Task.Run` or not:** wrap in `Task.Run` when you are *in application code* and about to do genuine CPU work that would otherwise block a request thread or a UI thread. Do not wrap in library code to make a synchronous API look asynchronous: it consumes a thread and hides that fact from the caller, who could have made a better decision. Do not wrap an already-async call: `await Task.Run(() => client.GetAsync(url))` adds a thread hop and buys nothing.

## 2.3 Completed tasks without work

Three allocations you should know because they are on hot paths everywhere:

```csharp
Task.CompletedTask                 // cached singleton, zero allocation
Task.FromResult(42)                // allocates: 42 is outside the internal cache
Task.FromException(new IOException())
Task.FromCanceled(token)
ValueTask.CompletedTask            // struct, no allocation at all
```

`Task.FromResult` does cache a handful of results (verified on .NET 10: small non-negative `int`s, `true`/`false`, and `null` references return the same instance every time; anything else allocates), so `Task.FromResult(true)` is free and `Task.FromResult(42)` is not. Do not rely on that; rely on `Task.CompletedTask` and `ValueTask.CompletedTask`, which are unconditionally allocation-free.

An `async` method that returns without ever awaiting an incomplete task still allocates a state machine box unless the JIT can avoid it, which is the entire motivation for `ValueTask` (§3.7).

## 2.4 Composition

```csharp
// Fan out, wait for all. Runs concurrently; the tasks are already started.
Task<Order[]> all = Task.WhenAll(ids.Select(id => _repo.GetAsync(id)));

// First to finish. Note: the losers keep running.
Task winner = await Task.WhenAny(primary, fallback);

// Process results as they arrive, in completion order (.NET 9+).
await foreach (Task<Order> t in Task.WhenEach(tasks))
    Handle(await t);

// Timeout without a linked CTS (.NET 6+).
Order o = await _repo.GetAsync(id).WaitAsync(TimeSpan.FromSeconds(2));
```

Three things about `WhenAll` that bite. The tasks must already be running: `WhenAll` starts nothing, it observes. If more than one task faults, the returned task's `Exception` is an `AggregateException` containing all of them, but `await` rethrows only the *first*: to see them all you must inspect `task.Exception` after catching. And `WhenAll` over thousands of tasks that each open a connection will happily open thousands of connections; bound it with `SemaphoreSlim` (§6.3) or `Parallel.ForEachAsync` (§7.1).

`WhenAny` leaks by default. The losing tasks continue to completion, and if one of them faults after nobody is watching you have an unobserved exception (§2.5). Combine it with a `CancellationTokenSource` so the losers are actually stopped, or accept the cost knowingly.

`Task.WhenEach`, added in .NET 9, replaces the old "remove the completed one from a list and call `WhenAny` again" pattern, which was O(n²) in the number of tasks.

## 2.5 Exceptions

An exception thrown inside a task does not propagate at the throw site. It is captured, stored on the task, and rethrown when someone observes the task, either by awaiting it or by touching `.Result`, `.Wait()` or `.Exception`.

The difference between the two observation styles matters. `await` rethrows the original exception with its stack trace preserved (via `ExceptionDispatchInfo`). `.Wait()` and `.Result` throw an `AggregateException` wrapping it. This is why catch blocks that work in one style silently stop matching in the other.

```csharp
try { await task; }               catch (IOException) { }  // matches
try { task.Wait(); }              catch (IOException) { }  // does NOT match
try { task.Wait(); }              catch (AggregateException) { }  // matches
```

If a faulted task is never observed at all, nothing happens by default on modern .NET: the exception is swallowed when the task is garbage collected, and only surfaces through the `TaskScheduler.UnobservedTaskException` event. That silence is a common source of "the work just stopped happening" mysteries in fire-and-forget code (§9.6).

> **The tell — where to put the `try`:** put it around the `await`, not around the call that produces the task. `try { var t = Foo(); } catch` catches only exceptions thrown *synchronously before the first await* inside `Foo`, which for a well-behaved async method is almost nothing (argument validation, if that). The interesting failures arrive at the `await`.

---

# Part 3 — `async` and `await`

## 3.1 What the keywords actually mean

`async` is not part of the method's signature in any meaningful sense. It is a marker to the compiler saying "rewrite this method body as a state machine, and treat `await` as a keyword inside it". Callers cannot see it, you cannot declare it in an interface, and whether an implementation is `async` is an implementation detail. An interface method returning `Task<T>` may be implemented with or without `async`.

`await` does five things in sequence:

1. Calls `GetAwaiter()` on the operand.
2. Checks `awaiter.IsCompleted`. **If true, execution continues straight through, on the same thread, with no yielding and no allocation.** This fast path is why awaiting a cached `Task.CompletedTask` in a loop is nearly free, and why an async method whose data is in a cache behaves synchronously.
3. If false, calls `awaiter.OnCompleted(continuation)`, handing over a delegate that resumes the state machine.
4. **Returns to the caller.** The caller receives the `Task` produced by the method builder. That task is not complete.
5. Later, when the operation finishes, something (an I/O completion, a timer, another task's continuation) invokes that delegate, which re-enters the state machine at the saved resume point.

Steps 4 and 5 are the whole model. The thread that ran steps 1 through 4 is gone; it returned up the stack and was handed back to whatever owns it. Nothing is waiting anywhere.

## 3.2 Return types

| Signature | Meaning |
|---|---|
| `async Task` | Asynchronous, no result. The normal case. |
| `async Task<T>` | Asynchronous with a result. The normal case. |
| `async ValueTask` / `ValueTask<T>` | Same, optimised for the frequently-synchronous case (§3.7). |
| `async void` | Fire and forget with no handle. See below. |
| `async IAsyncEnumerable<T>` | Async stream, consumed with `await foreach` (§3.8). |
| `async` returning a custom type | Possible via `[AsyncMethodBuilder]`; you will not write one (§8.3). |

## 3.3 `async void`

An `async void` method returns nothing, so the caller receives no `Task`, so there is no way to know when it finished and no way to observe its exception. When an exception escapes an `async void` method, the builder rethrows it **on the captured synchronization context, or on the thread pool if there is none**, where it becomes an unhandled exception and terminates the process.

```csharp
// BROKEN: an exception here kills the process, and the caller can't await it.
public async void OnMessage(Message m) => await _handler.HandleAsync(m);

// FIXED
public async Task OnMessageAsync(Message m) => await _handler.HandleAsync(m);
```

The single legitimate use is an event handler whose signature is fixed by a framework (`void Button_Click(object, EventArgs)`), and even then the body should be entirely inside a try/catch. If you are writing server code, the honest rule is: **no `async void`, ever**. Enable the analyser and treat it as an error.

The related trap is passing an async lambda where an `Action` is expected. `Action` returns `void`, so `async () => await Foo()` compiles as `async void`, silently, with all of the above consequences:

```csharp
// BROKEN: this lambda binds to Action, so it is async void.
list.ForEach(async item => await ProcessAsync(item));

// FIXED
foreach (var item in list) await ProcessAsync(item);
```

Overload resolution prefers `Func<Task>` over `Action` when both exist, so APIs designed for async (like `Task.Run`) are safe. APIs that predate it (`List<T>.ForEach`, most `Timer` callbacks, `Parallel.ForEach`) are not.

## 3.4 Async all the way, and the deadlock

The classic .NET deadlock needs three ingredients: a `SynchronizationContext` that runs continuations on a specific thread, an async method that captures it, and a caller that blocks that same thread waiting for the result.

```csharp
// BROKEN, in any app with a SynchronizationContext (WPF, WinForms, classic ASP.NET):
public string Get()
{
    return GetAsync().Result;      // blocks THIS thread
}
private async Task<string> GetAsync()
{
    await _http.GetStringAsync(url);   // captures the context
    return "done";                     // needs THIS thread to resume. It never will.
}
```

The continuation is queued to the context, the context's only thread is blocked inside `.Result`, and neither can proceed.

**ASP.NET Core has no `SynchronizationContext`.** It was removed deliberately, so this exact deadlock does not occur there. That fact does more harm than good, because it means the same code that deadlocks in a WPF app merely *starves the thread pool* in ASP.NET Core (§1.5): slower to diagnose, identical in cause. Do not conclude you are safe. The rule is unchanged: **once a call chain is async, keep it async to the top**.

There is no reliable way to block on async code. `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` and the "nested `Task.Run`" trick all trade one failure mode for another. `.GetAwaiter().GetResult()` is marginally better than `.Result` in that it throws the original exception rather than an `AggregateException`, which is the only reason to prefer it in the places where you genuinely have no choice: `Main` before C# 7.1, a static constructor, a `Dispose` that must call `DisposeAsync`.

> **The tell — sync boundary:** you may block on async code exactly once, at the outermost frame you control, where there is no context to deadlock against and no request thread to starve. Everywhere else, propagate `Task`. If a synchronous interface is forcing your hand, the fix is a different interface, not a cleverer block.

## 3.5 `SynchronizationContext`, `ExecutionContext`, and `ConfigureAwait`

Two distinct ambient contexts flow through async code and people conflate them constantly.

**`ExecutionContext`** carries ambient state across async boundaries: `AsyncLocal<T>` values, the security context, and (in ASP.NET Core) things like the current activity for distributed tracing. It flows automatically, always, and you cannot switch it off with `ConfigureAwait`. This is what makes `HttpContext` accessible via `IHttpContextAccessor` after an await.

**`SynchronizationContext`** answers "where should continuations run?". WPF and WinForms install one that marshals back to the UI thread. Classic ASP.NET installed one tied to the request. **ASP.NET Core installs none**, so `SynchronizationContext.Current` is null and continuations run on whatever thread completed the I/O, taken from the pool.

`ConfigureAwait(false)` says "I don't need to resume on the captured context; resume anywhere". Its effects:

- In a UI app: prevents the marshal back to the UI thread. This is what breaks the deadlock in §3.4.
- In ASP.NET Core: **no effect at all**, since there is nothing to capture. Slight overhead reduction at best.
- It does *not* stop `ExecutionContext` flowing, so `AsyncLocal` still works.
- It applies to that one `await`, not the whole method.

So: in **library** code, use `ConfigureAwait(false)` on every await, because you do not know whether your caller is a UI app. In **application** code targeting ASP.NET Core, it is noise; most teams omit it. Do not add it to application code and imagine you have fixed a performance problem.

.NET 8 added a richer form, `ConfigureAwait(ConfigureAwaitOptions)`:

```csharp
await task.ConfigureAwait(ConfigureAwaitOptions.None);                    // == ConfigureAwait(false)
await task.ConfigureAwait(ConfigureAwaitOptions.ContinueOnCapturedContext); // == ConfigureAwait(true)
await task.ConfigureAwait(ConfigureAwaitOptions.SuppressThrowing);        // await without rethrowing
await task.ConfigureAwait(ConfigureAwaitOptions.ForceYielding);           // always yield, even if complete
```

`SuppressThrowing` is the useful addition: awaiting a task purely to know it has finished, without caring whether it faulted, previously required a try/catch. It is only valid on non-generic `Task` (there is no sensible result to hand back for `Task<T>`), and the built-in analyser rule CA2261 enforces that. `ForceYielding` is `Task.Yield()` semantics attached to an existing await, useful for guaranteeing you get off the current stack.

## 3.6 `AsyncLocal<T>` versus `ThreadLocal<T>`

`ThreadLocal<T>` is per-thread storage and is nearly useless in async code, because your method resumes on a different thread than it started on. `AsyncLocal<T>` is per-*logical-call-context* storage: it flows down through awaits into the continuation.

The semantics that surprises people: `AsyncLocal` flows *downward* only. A value set inside an async method is visible to everything that method calls, and to its continuations, but writing to it does **not** propagate back up to the caller, because the caller has its own copy of the `ExecutionContext`. It is a copy-on-write tree, not a shared variable.

```csharp
static readonly AsyncLocal<string> Tenant = new();

async Task OuterAsync()
{
    Tenant.Value = "acme";
    await InnerAsync();               // sees "acme"
    Console.WriteLine(Tenant.Value);  // still "acme", NOT whatever Inner set
}
async Task InnerAsync()
{
    Tenant.Value = "globex";          // visible only below here
    await Task.Yield();
}
```

Use it for genuinely ambient cross-cutting concerns (tenant, correlation id, trace context). Do not use it as a way to smuggle parameters, which is what it degenerates into.

## 3.7 `ValueTask`

`Task` is a class, so every async operation allocates one, plus a boxed state machine if it suspends. For the vast majority of code this is irrelevant: one small allocation per database call is not your problem. For genuinely hot paths where the operation **usually completes synchronously**, `ValueTask<T>` avoids the allocation by being a struct that holds either a result directly or a `Task`.

The canonical case is a cache:

```csharp
public ValueTask<Config> GetAsync(string key)
{
    if (_cache.TryGetValue(key, out var c)) return new ValueTask<Config>(c);  // no allocation
    return new ValueTask<Config>(LoadSlowAsync(key));                         // falls back to a Task
}
```

The cost is a much stricter contract. A `ValueTask` may be consumed **once**, and only by awaiting it. You must not: await it twice, call `.Result` before it completes, await it concurrently from two places, or store it in a field and observe it later. Violating these is undefined behaviour, not an exception, because `IValueTaskSource` implementations recycle their backing objects. If you need to do any of those things, call `.AsTask()` once and use the result.

> **The tell — `Task` or `ValueTask`:** default to `Task`. Return `ValueTask<T>` only when you have measured, the method completes synchronously most of the time, and it sits on a path hot enough for one allocation to matter. Consume a `ValueTask` with exactly one `await` at the call site, or convert it with `AsTask()` immediately. Never expose `ValueTask` on a public API that callers might reasonably want to cache.

## 3.8 Async streams

`IAsyncEnumerable<T>` is the async counterpart of `IEnumerable<T>`: a sequence whose *elements* arrive asynchronously, as opposed to `Task<IEnumerable<T>>`, which is a single asynchronous arrival of a whole collection. Use it when items are produced over time or when the collection is too large to materialise: paged API results, a streaming query, a message feed.

```csharp
public async IAsyncEnumerable<Order> StreamAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    string? cursor = null;
    do
    {
        Page page = await _api.GetPageAsync(cursor, ct);
        foreach (var o in page.Items) yield return o;
        cursor = page.Next;
    } while (cursor is not null);
}

await foreach (Order o in StreamAsync().WithCancellation(ct).ConfigureAwait(false))
    Process(o);
```

The `[EnumeratorCancellation]` attribute is the non-obvious part. Without it, the token passed to `WithCancellation` at the call site never reaches your method body, and cancellation silently does nothing. The attribute tells the compiler to wire the consumer's token into that parameter.

## 3.9 `IAsyncDisposable`

`await using` calls `DisposeAsync`. Implement `IAsyncDisposable` when cleanup genuinely does I/O: flushing a buffered stream, sending a close frame on a socket, committing a transaction. Implementing both `IDisposable` and `IAsyncDisposable` is normal, and `Dispose()` should not simply call `DisposeAsync().GetAwaiter().GetResult()`, which reintroduces §3.4. Where you must support both, write the synchronous cleanup path properly.

---

# Part 4 — Cancellation

## 4.1 Cancellation is cooperative, always

There is no mechanism in .NET to stop running code from outside. `Thread.Abort` is gone (§1.1). What exists instead is a signalling protocol: a `CancellationTokenSource` owns the ability to signal, a `CancellationToken` is a read-only view of that signal handed to the work, and **the work is responsible for checking it**.

This means cancellation only happens where someone wrote code to make it happen. A token passed to a method that ignores it does nothing. The most common cancellation bug is not a race, it is simply forgetting to pass the token down one level, at which point everything below that point is uncancellable.

```csharp
using var cts = new CancellationTokenSource();
Task work = DoWorkAsync(cts.Token);
cts.Cancel();                       // requests; does not stop anything by itself

try { await work; }
catch (OperationCanceledException) { /* the work cooperated */ }
```

`CancellationTokenSource` is `IDisposable` and you should dispose it, particularly when using `CancelAfter`, which allocates a timer. Leaked CTS instances with timers are a real, if unglamorous, memory leak.

## 4.2 Observing the token

Two idioms, with different meanings:

```csharp
ct.ThrowIfCancellationRequested();   // abandon: throw OperationCanceledException
if (ct.IsCancellationRequested) { }  // handle: flush, checkpoint, return partial results
```

Throwing is the default and it is correct most of the time, because it puts the task into the `Canceled` state, which callers can distinguish from a fault. Checking the boolean is for the cases where abrupt abandonment loses work you care about.

Every async framework API that accepts a token throws `OperationCanceledException` (or its subclass `TaskCanceledException`) when cancelled. `catch (OperationCanceledException)` catches both; catching only `TaskCanceledException` will miss cases.

The subtlety worth knowing: whether an exception is treated as *cancellation* rather than a *fault* depends on whether the `OperationCanceledException.CancellationToken` matches the token the task was created with. Throwing a bare `new OperationCanceledException()` from inside a task produces a faulted task, not a cancelled one. Use `ThrowIfCancellationRequested` and this never comes up.

## 4.3 Composing tokens

```csharp
using var timeout = new CancellationTokenSource(TimeSpan.FromSeconds(5));
using var linked = CancellationTokenSource.CreateLinkedTokenSource(
    requestAborted, timeout.Token, _shutdown.Token);

await _repo.QueryAsync(linked.Token);   // cancelled by whichever fires first
```

A linked source fires when any of its sources fires. This is the standard shape for "cancel if the client disconnects, or the operation exceeds five seconds, or the host is shutting down". **Dispose the linked source**: it registers callbacks on all its parents, and failing to dispose it keeps it alive as long as the longest-lived parent, which for a host shutdown token is the process lifetime. This is one of the more common real leaks in ASP.NET Core code.

If you only need a timeout on a single awaited task, `WaitAsync` (.NET 6+) is cheaper and clearer:

```csharp
await _repo.QueryAsync(ct).WaitAsync(TimeSpan.FromSeconds(5), ct);
```

Note what `WaitAsync` does and does not do: it stops *you* waiting. The underlying operation carries on unless it was given a token that fires too. A timeout that does not cancel the work is a resource leak under load. №31 §10.1 and §10.2 cover choosing the timeout value and what to do when it fires; everything said there about retries, backoff and jitter applies unchanged to a `CancellationToken` in a `Polly` pipeline.

## 4.4 Registration callbacks

`ct.Register(callback)` runs a delegate when cancellation is requested. Two facts about it are load-bearing:

The callback runs **synchronously on the thread that calls `Cancel()`**, unless it was already cancelled at registration time, in which case it runs immediately on the registering thread. So a slow or blocking callback blocks the canceller, and if the callback takes a lock that the canceller already holds, you deadlock.

`Register` returns a `CancellationTokenRegistration` that must be disposed to unregister. If you register against a long-lived token from inside a short-lived operation and never dispose, you have built a leak with the same shape as the linked-source one.

```csharp
using var reg = ct.Register(static s => ((Socket)s!).Close(), socket);
await socket.ReceiveAsync(buffer);   // Close() unblocks the receive
```

That pattern, cancelling something that has no token-aware API by forcibly closing the resource, is the standard escape hatch for uncancellable operations.

## 4.5 Cancellation in ASP.NET Core

Three tokens matter on a server, and mixing them up produces confusing behaviour.

| Token | Fires when | Use for |
|---|---|---|
| `HttpContext.RequestAborted` | The client disconnects | Abandoning work nobody will receive |
| `IHostApplicationLifetime.ApplicationStopping` | Shutdown begins | Draining background work |
| `BackgroundService`'s `stoppingToken` | Shutdown begins | The service's own loop |

A controller action or minimal API handler that takes a `CancellationToken` parameter is bound to `RequestAborted` automatically. Pass it into your data access and you get free abandonment of orphaned work, which under load is a meaningful capacity saving.

The trap: do **not** pass `RequestAborted` into an operation that must complete once started. If a client disconnects halfway through a payment write, you do not want the write cancelled at an arbitrary point. Non-idempotent, must-complete work should either run under a different token or be handed to a durable queue.

## 4.6 Timers, and the re-entrancy trap

`System.Threading.Timer` invokes its callback on a pool thread and **does not wait for the previous callback to finish**. If the work takes longer than the interval, callbacks overlap, and you have concurrency you never asked for. The callback is also an `Action`-shaped delegate, so an async lambda there is `async void` (§3.3).

`PeriodicTimer` (.NET 6+) fixes both problems by inverting the control flow:

```csharp
using var timer = new PeriodicTimer(TimeSpan.FromSeconds(30));
while (await timer.WaitForNextTickAsync(stoppingToken))
{
    await DoWorkAsync(stoppingToken);   // next tick cannot start until this returns
}
```

Ticks that occur while you are working are dropped rather than queued, which for periodic maintenance work is exactly what you want. `WaitForNextTickAsync` returns false when the timer is disposed, giving a clean loop exit.

> **The tell — which cancellation shape:** if you need a deadline on one await, use `WaitAsync`. If you need a deadline that also stops the underlying work, use a `CancellationTokenSource(timespan)` and pass its token in. If you need "any of several reasons to stop", use a linked source, and dispose it. If the operation has no token support at all, register a callback that closes the resource.

---

# Part 5 — The memory model and atomicity

This is the part of concurrency that does not announce itself. Everything so far fails loudly: a deadlock hangs, a starved pool times out. Memory model bugs produce a program that is correct 99.999% of the time and wrong at 3am under load, on one instance, once a week.

## 5.1 Three layers reorder your code

When you write `_data = x; _ready = true;` there are three independent parties entitled to make those happen in the other order.

The **C# compiler** may reorder and eliminate operations, though in practice Roslyn is conservative. The **JIT** is far more aggressive: it hoists loads out of loops, keeps fields in registers, folds redundant reads, and reorders freely. (№13 covers the equivalent machinery on the JVM side; the optimisations are the same family, and the reason both runtimes are entitled to them is the same.) The **CPU** reorders at the hardware level: store buffers delay writes becoming visible, and weakly-ordered architectures let loads and stores pass each other.

The rule they all obey is **single-thread consistency**: a single thread always observes its own operations in program order. No such guarantee is offered to any other thread. The .NET memory model states it plainly: *"the effects of ordinary reads and writes can be reordered as long as that preserves single-thread consistency"*.

The classic demonstration:

```csharp
// BROKEN: this loop may never terminate, and it is not a bug in the CPU.
private bool _stop;                       // ordinary field
private int _result;

void Worker()
{
    while (!_stop) { }                    // JIT may hoist the read out of the loop entirely,
    Console.WriteLine(_result);           // compiling it to `if (!_stop) while(true);`
}
void Stopper() { _result = 42; _stop = true; }
```

Two failures are latent here. The reader may never see `_stop` change, because the JIT is entitled to read it once and cache it in a register. And even if it does see `_stop`, it may see `_result` as 0, because there is nothing ordering the two writes in `Stopper` or the two reads in `Worker`.

## 5.2 What is atomic, and what tears

The guarantee is precise and narrow:

> Memory accesses to **properly aligned** data of primitive and enum types with sizes **up to the platform pointer size** are always atomic.

On 64-bit, that means `bool`, `byte`, `short`, `char`, `int`, `long`, `float`, `double`, all reference types, `IntPtr`, and enums are read and written atomically. You will never observe half of an `int`.

What is not atomic, and can **tear** (you observe a value that was never written):

| Type | Tears when |
|---|---|
| `long`, `double`, `ulong` | On 32-bit platforms (pointer size is 4) |
| `decimal` | Always: 16 bytes |
| `Guid` | Always: 16 bytes |
| `DateTime`, `TimeSpan` | Implementation-defined; treat as unsafe |
| Any multi-field `struct` | Always, unless it fits in a pointer and is aligned |
| Misaligned fields (explicit layout, unsafe) | Always |

A torn `Guid` is a value made of the first half of one and the second half of another: a correlation id that exists in no log, ever. A torn `decimal` is a monetary amount that was never a legitimate price. These are the worst bugs in this document because the evidence is destroyed by the bug itself.

For 64-bit values on 32-bit platforms (which you may still meet in embedded or 32-bit IIS worker processes), `Interlocked.Read(ref long)` gives an atomic read.

## 5.3 Atomic does not mean thread-safe

`_count++` is three operations: read, add, write. Each is atomic. The sequence is not. Two threads can read 7, both compute 8, both write 8, and one increment is gone. Nothing about atomicity of the individual reads and writes helps.

```csharp
private int _count;

void Broken()   => _count++;                     // lost updates
void Fixed1()   => Interlocked.Increment(ref _count);
void Fixed2()   { lock (_gate) _count++; }
```

The same applies to any composite operation: `if (_x == null) _x = new()`, `_list.Add` after `_list.Contains`, `dict[k] = dict[k] + 1`. **Any read-modify-write on shared state needs either an atomic primitive or a lock.** №12 §2.2 states the general form of this. This is also why `ConcurrentDictionary` being thread-safe does not make `if (!dict.ContainsKey(k)) dict[k] = v` thread-safe (§6.7).

## 5.4 `volatile` and the `Volatile` class

`volatile` is about **ordering and visibility**, never about atomicity. Marking a field volatile does not make `++` safe.

Its precise meaning in .NET:

- A **volatile read has acquire semantics**: no read or write later in program order may be moved ahead of it.
- A **volatile write has release semantics**: the effects of all earlier reads and writes become observable before the volatile write does.

That asymmetry is what makes the publication pattern work: write the data, then volatile-write the flag; volatile-read the flag, then read the data. If you see the flag, you are guaranteed to see the data.

```csharp
private int _result;
private volatile bool _ready;      // volatile write below, volatile read above

void Producer() { _result = 42; _ready = true; }         // release
void Consumer() { while (!_ready) { } Print(_result); }   // acquire: _result is guaranteed 42
```

The keyword has restrictions that surprise people. It **cannot be applied to `long`, `ulong`, `double`, `decimal` or struct fields** (`volatile long _x;` is compile error CS0677), nor to locals, nor to array elements. And passing a volatile field to an ordinary `ref` parameter silently loses the volatility, which the compiler flags as warning CS0420. The `Volatile` class covers every one of those gaps:

```csharp
Volatile.Write(ref _flag, true);       // works on long, double, T, array elements
bool f = Volatile.Read(ref _flag);
```

Prefer `Volatile.Read`/`Volatile.Write` in new code. Making a field `volatile` applies the semantics to *every* access to it, including the 99% that do not need it, and hides at the declaration a fact the reader needs at the use site. The explicit calls document the intent where it matters.

What `volatile` does not give you: it is not a full fence. A volatile write followed by a volatile read of a *different* field can still be reordered with respect to each other (this is the store-load reordering that makes Dekker's algorithm fail). If you need that, you need `Interlocked` or an explicit full barrier.

## 5.5 `Interlocked`

`Interlocked` operations are atomic read-modify-write, implemented with the CPU's lock-prefixed or load-linked/store-conditional instructions, and they carry **full-fence semantics**: their effects become observable neither earlier nor later than their program-order position relative to other memory operations.

```csharp
Interlocked.Increment(ref _count);          // returns the NEW value
Interlocked.Decrement(ref _count);
Interlocked.Add(ref _total, 5);             // returns the new value
Interlocked.Exchange(ref _current, next);   // returns the OLD value
Interlocked.CompareExchange(ref _x, newValue, comparand);  // returns the ORIGINAL value
Interlocked.Or(ref _flags, 0b100);          // .NET 5+
Interlocked.And(ref _flags, ~0b010);
```

The return-value conventions are inconsistent and worth memorising: `Increment`/`Add` give you the result, `Exchange`/`CompareExchange` give you what was there before.

The costs are worth carrying in your head as ratios. Measured on .NET 10 on a two-core VM: a `Volatile.Write` is about 0.3 ns, an uncontended `Interlocked.Increment` about 7.6 ns, and an uncontended `lock` enter-and-exit about 19 ns. So **an atomic operation costs roughly twenty plain writes, and an uncontended lock costs roughly two-and-a-half atomics**. A *contended* lock that reaches the kernel is microseconds, three orders of magnitude worse, which is why the uncontended numbers are the wrong thing to optimise.

Under heavy contention on a single cache line, `Interlocked` degrades badly too, because every core is fighting for exclusive ownership of that line. A counter incremented by 64 threads is slower than 64 per-thread counters summed at the end, by a large factor (§7.3).

## 5.6 Compare-and-swap: the primitive everything is built on

`CompareExchange(ref location, value, comparand)` atomically does: *if `location` equals `comparand`, set it to `value`; return whatever `location` held originally*. You check success by comparing the return value against your comparand.

This is the foundation of every lock-free algorithm. The pattern is always the same loop: read, compute, attempt to swap, retry if someone beat you.

```csharp
// Atomically apply an arbitrary function to a shared value.
static int Update(ref int location, Func<int, int> f)
{
    int original, updated;
    do
    {
        original = Volatile.Read(ref location);
        updated  = f(original);
    }
    while (Interlocked.CompareExchange(ref location, updated, original) != original);
    return updated;
}
```

It also gives you idempotent lazy publication without a lock:

```csharp
private Config? _config;
public Config Get() =>
    _config ?? Interlocked.CompareExchange(ref _config, Build(), null) ?? _config;
```

That version may call `Build()` more than once under a race but will only ever *publish* one instance, and every caller receives the same one. That trade (redundant work, single result) is often the right one for cheap idempotent construction, and the wrong one for anything expensive or side-effecting.

**Cross-reference worth holding onto, and the most valuable one in this document:** this is the same pattern as optimistic concurrency control in a database (`UPDATE ... WHERE version = @expected`, and see №22 on MVCC), as an HTTP `If-Match` conditional request, and as a fencing token guarding against a split-brain writer (№31 §6.3). Read the current version, compute a new one, publish it conditional on nothing having changed underneath. **One idea at four scales: a CPU instruction, a row lock, an HTTP header, and a distributed-systems safety property.** №12 §5.1 introduces the instruction; №31 §8.1 shows what it becomes when the two parties are on different machines and the network can lose your answer. Recognising the shape in a new context saves you learning the context.

The classic hazard, ABA, is where a value changes from A to B and back to A between your read and your CAS, so the CAS succeeds although the world moved. .NET's GC makes the pointer form of this mostly a non-issue (a recycled object cannot be the same reference while you hold it), which is why lock-free data structures are notably easier to write here than in C++.

## 5.7 Barriers

`Thread.MemoryBarrier()` (equivalently `Interlocked.MemoryBarrier()`) is a full fence: no memory operation may cross it in either direction. `Interlocked.MemoryBarrierProcessWide()` is a much heavier, cross-process variant used by the runtime itself.

You will almost never write one. If you find yourself reaching for an explicit barrier, you are writing a lock-free algorithm, and the correct move is to check whether `System.Collections.Concurrent`, `Channel<T>`, or `Interlocked` already solves it. Hand-rolled lock-free code is genuinely difficult to get right and impossible to test into confidence, because the failure is a hardware reordering that your test machine may not perform.

## 5.8 Double-checked locking, and why not to write it

The canonical low-cost lazy initialisation:

```csharp
private volatile Config? _config;          // volatile is REQUIRED here
private readonly Lock _gate = new();

public Config Get()
{
    if (_config is null)                   // fast path, no lock
    {
        lock (_gate)
        {
            if (_config is null)           // re-check under the lock
                _config = new Config();    // volatile write publishes it safely
        }
    }
    return _config;
}
```

On most platforms this pattern is the textbook broken one: another thread sees a non-null reference before the constructor's writes to the object's fields are visible, and observes a half-initialised object. In Java it stays broken to this day unless the field is `volatile` or every field of the constructed object is `final`; in C++ it needs an explicit release.

**.NET does not have that hazard, and this is one of the places where the runtime's memory model is deliberately stronger than the specification it inherited.** The documented model states that *"object assignment to a location potentially accessible by other threads is a release with respect to accesses to the instance's fields/elements and metadata"*: storing an object reference into shared memory is a committing point for everything reachable through it. A thread that sees the reference sees a fully constructed object.

Keep the `volatile` regardless. It costs nothing on x64, it documents that the field is read outside the lock, and the guarantee above covers *object assignment* specifically, not the general flag-plus-data publication in §5.4. Matching a volatile write with a volatile read is the habit that keeps you out of trouble in the cases that are not covered.

Better still, do not write this at all. `Lazy<T>` does it correctly and reads better:

```csharp
private readonly Lazy<Config> _config = new(() => new Config());
public Config Get() => _config.Value;
```

`Lazy<T>` defaults to `LazyThreadSafetyMode.ExecutionAndPublication`, meaning the factory runs exactly once even under contention. `PublicationOnly` allows the factory to race (like the CAS version in §5.6) but publish one winner. `None` is single-threaded only and will corrupt if you are wrong about that.

> **The tell — which synchronisation primitive:** if you are counting, accumulating, or setting a flag, use `Interlocked`. If you are publishing a reference once, use `Lazy<T>`. If you need to see a flag another thread set, use `Volatile.Read`/`Write`. If you are performing more than one operation that must be seen as a unit, use a lock. If you reach for `Thread.MemoryBarrier`, stop and find the library that already did this.

## 5.9 Why it only breaks on ARM64

x86 and x64 implement **total store order**: loads are not reordered with loads, stores are not reordered with stores, and loads are not reordered with earlier stores. The only reordering the hardware performs is store-then-load. This means ordinary reads and writes on x64 already have acquire-release semantics *for free*, and a large amount of code that is formally broken under the .NET memory model runs correctly on x64 forever.

ARM64 is weakly ordered. Loads and stores may be reordered in both directions unless you use acquire/release instructions (`ldar`/`stlr`) or explicit barriers. .NET emits those for volatile accesses on ARM64, so correctly-written code is correct; incorrectly-written code that survived a decade on Intel starts failing.

The practical significance is current, not academic: AWS Graviton, Azure Cobalt and Ampere instances, Apple silicon developer machines, and ARM-based CI runners are all ordinary deployment targets now. **"We moved to Graviton and started seeing impossible bugs" is a memory model story until proven otherwise.** Test on ARM64 if you deploy on ARM64.

## 5.10 The actual answer: don't share mutable state

Everything above is a set of tools for making shared mutable state safe. The better engineering move is usually to not have any.

Immutable objects need no synchronisation at all, and on .NET that is a guarantee rather than a hope: by §5.8's publication rule, any thread that obtains the reference sees the fully constructed object. `readonly` is not what provides that guarantee; it is what stops the object being mutated afterwards, which is what makes the guarantee useful. Records give you this cheaply, with `with` expressions for derived values. `System.Collections.Immutable` gives you persistent collections where "mutation" produces a new collection sharing most of its structure.

Confinement is the other half: give each thread its own state and combine at the end. That is what `Parallel.For`'s thread-local overloads do (§7.3), what per-thread counters do, and what a `Channel` does by making the queue the only shared thing.

**Locks scale poorly, immutability scales perfectly, and confinement scales perfectly.** The order in which you should reach for them follows directly. №12 §7.1 and §7.2 make the same argument without a language attached; what .NET adds is that the publication guarantee above makes immutability airtight rather than merely conventional.

---

# Part 6 — Locks, coordination, and safe data structures

## 6.1 `lock`, `Monitor`, and `System.Threading.Lock`

`lock (x) { ... }` has always been syntax over `Monitor.Enter`/`Monitor.Exit` in a try/finally. Monitor locks are:

- **Reentrant.** The same thread may acquire the same lock repeatedly; it must exit as many times as it entered.
- **Thread-affine.** Only the acquiring thread may release. This is why you cannot hold one across an `await`.
- **Cheap when uncontended.** Tens of nanoseconds (§5.5), with a spin phase before falling back to a kernel wait. Expensive when contended, by orders of magnitude.

Historically the lock object was any reference type, with all the hazards that implies: `lock(this)`, `lock(typeof(Foo))` and `lock("some string")` all lock on objects that other, unrelated code can also lock on, producing deadlocks between components that have never heard of each other. The convention was a private `readonly object _gate = new()`.

.NET 9 and C# 13 introduced `System.Threading.Lock`, a purpose-built type, and the compiler special-cases it:

```csharp
private readonly Lock _gate = new();       // System.Threading.Lock

public void Add(int x)
{
    lock (_gate) { _total += x; }          // compiles to using (_gate.EnterScope())
}
```

When the operand of `lock` is statically typed as `System.Threading.Lock`, the compiler emits `using (x.EnterScope())` rather than `Monitor.Enter`. It is modestly faster than `Monitor` (19 ns versus 21 ns uncontended in the measurement above, so this is not the reason to switch) and it removes the "anyone can lock on my object" problem by construction, which is. If you upcast it to `object`, the compiler warns (CS9216), because `lock` on the upcast would silently fall back to monitor semantics on the `Lock` instance itself.

**Use `System.Threading.Lock` in all new code on .NET 9 or later.** The migration is a one-word type change from `object` to `Lock`, with no call-site changes.

## 6.2 You cannot `await` inside a `lock`

The compiler rejects it outright (CS1996), and the reason is worth understanding rather than working around. A monitor lock is owned by a *thread*. An `await` may resume on a different thread. If the compiler allowed it, `Monitor.Exit` would run on a thread that never entered, throwing `SynchronizationLockException`, and in the meantime the lock would be held across an unbounded I/O operation, serialising your entire application behind one network call.

The fix is a lock that is not thread-affine.

## 6.3 `SemaphoreSlim`: the async mutual exclusion primitive

A semaphore with an initial and maximum count of 1 is a mutex, and `SemaphoreSlim` has an async wait.

```csharp
private readonly SemaphoreSlim _gate = new(1, 1);

public async Task<Config> GetAsync(CancellationToken ct)
{
    await _gate.WaitAsync(ct);
    try
    {
        return _cached ??= await LoadAsync(ct);   // safe to await while "holding" it
    }
    finally { _gate.Release(); }
}
```

Three rules. The `Release` goes in a `finally`, always: an exception between wait and release permanently reduces the semaphore's capacity, and after N failures your application stops. `SemaphoreSlim` is **not reentrant**: a method that takes it and calls another method that takes it deadlocks against itself, which monitor locks would have tolerated. And releasing more times than you waited throws `SemaphoreFullException`, which in practice means someone put `Release()` on a path that did not acquire.

The other use of `SemaphoreSlim` is throttling concurrency rather than excluding it, which is how you bound a fan-out:

```csharp
var gate = new SemaphoreSlim(10);          // at most 10 concurrent
await Task.WhenAll(urls.Select(async url =>
{
    await gate.WaitAsync(ct);
    try { await FetchAsync(url, ct); }
    finally { gate.Release(); }
}));
```

## 6.4 `ReaderWriterLockSlim`

Allows many concurrent readers or one writer. The intuition (reads are cheap, so let them run together) is right, and the practice usually disappoints: `ReaderWriterLockSlim` is several times more expensive to acquire than a monitor lock even in the read path, so unless reads are both frequent and *long*, a plain lock wins. Its upgradeable-read mode adds real complexity, and it defaults to non-reentrant, so recursive acquisition throws rather than working.

Reach for it when you have a large in-memory structure, read overwhelmingly often, held for a meaningful duration. For anything smaller, use a plain lock, `ConcurrentDictionary`, or an immutable snapshot swapped with `Interlocked.Exchange`.

## 6.5 The signalling primitives

| Primitive | Shape | Use |
|---|---|---|
| `ManualResetEventSlim` | Gate: opens once, stays open | "Initialisation is complete, everyone proceed" |
| `AutoResetEvent` | Turnstile: releases one waiter, closes | Hand-off of a single item |
| `SemaphoreSlim` | Counted permits | Throttling, async mutual exclusion |
| `CountdownEvent` | Waits for N signals | "Wait for all N workers to report in" |
| `Barrier` | Rendezvous, reusable per phase | Phased parallel algorithms (simulation steps) |
| `Mutex` | Named, cross-process, thread-affine | Single-instance enforcement across processes |
| `Monitor.Wait`/`Pulse` | Condition variable inside a lock | Hand-rolled blocking queues |

The `Slim` variants spin briefly before blocking, so they are far cheaper for short waits, and they do not allocate a kernel handle unless someone actually blocks. Prefer them unless you need a named, cross-process object or a `WaitHandle`.

`Monitor.Wait`/`Pulse` is a condition variable: `Wait` releases the lock and blocks; `Pulse`/`PulseAll` wakes waiters, which then re-acquire. It is the classic way to build a blocking queue, and you should not build one: `Channel<T>` (§6.8) is better in every respect.

`Mutex` deserves one warning: it is thread-affine like `Monitor`, so it cannot be held across an await, and a named `Mutex` abandoned by a crashing process raises `AbandonedMutexException` in the next acquirer. That exception is information, not noise: the shared state it protected may be inconsistent.

> **The tell — which coordination primitive:** the question to ask is not "what am I protecting?" but "what am I waiting for?". Waiting for *exclusive access* to state is a lock (`System.Threading.Lock`, or `SemaphoreSlim(1,1)` if you must await inside). Waiting for an *event to have happened* is `ManualResetEventSlim`, or better, the `Task` that already represents it. Waiting for *capacity* is `SemaphoreSlim(n)`. Waiting for *work to arrive* is a `Channel<T>`, never a condition variable you wrote yourself. If the answer is "several of these at once", you are building a pipeline and should model it as one.

## 6.6 Deadlock

Two shapes, one of which is not really a deadlock.

**Lock-order inversion** is the textbook one (№12 §2.4 and Part 8 treat it in full, including the four Coffman conditions): thread A holds lock 1 and wants lock 2, thread B holds lock 2 and wants lock 1. The cure is discipline, not cleverness: **define a global order over your locks and always acquire in that order**. If you cannot state that order for your codebase, you have too many locks.

**The async deadlock** (§3.4) is different: nothing is holding a lock, a thread is simply blocked waiting for work that requires that thread. Fixing it is not about lock ordering; it is about not blocking.

Practical mitigations for the first: hold locks for the shortest possible region, never call out to unknown code (a virtual method, a delegate, an event handler) while holding a lock, and prefer a single coarse lock over several fine ones until you have measured contention. `Monitor.TryEnter(obj, timeout)` lets you detect rather than hang, but treating the timeout as a retry loop just converts a deadlock into a livelock.

## 6.7 Concurrent collections

`System.Collections.Concurrent` gives you internally-synchronised collections. What it does not give you is atomicity across two calls.

| Type | Notes |
|---|---|
| `ConcurrentDictionary<K,V>` | Fine-grained striped locking for writes, lock-free reads |
| `ConcurrentQueue<T>` | Lock-free FIFO, segmented |
| `ConcurrentStack<T>` | Lock-free LIFO, CAS on head |
| `ConcurrentBag<T>` | Thread-local storage with stealing; fast only when the same thread adds and takes |
| `BlockingCollection<T>` | Blocking producer/consumer wrapper. Superseded by `Channel<T>` |

Two `ConcurrentDictionary` behaviours cause real bugs.

**The `GetOrAdd` factory is not run under the lock.** Two threads racing on a missing key may both invoke your factory. Exactly one result is stored and both callers receive that one, but the other object was still constructed, and if constructing it had side effects (opening a connection, registering a callback, incrementing a counter) those happened.

```csharp
// BROKEN if creating a connection is expensive or side-effecting.
var conn = _pool.GetOrAdd(key, k => new Connection(k));

// FIXED: the value is cheap to create; the expensive part happens once.
var conn = _pool.GetOrAdd(key, k => new Lazy<Connection>(() => new Connection(k))).Value;
```

**Check-then-act is still a race.** `if (!dict.ContainsKey(k)) dict[k] = v` is exactly as broken as it would be on a plain `Dictionary`. Use `TryAdd`, `GetOrAdd`, or `AddOrUpdate`, which are the atomic composite operations the type exists to provide. `Count` and enumeration are snapshots that may be stale before you use them.

The alternative worth knowing: for read-mostly data, an **immutable dictionary swapped atomically** often beats `ConcurrentDictionary`, because readers pay nothing at all.

```csharp
private ImmutableDictionary<string, Rule> _rules = ImmutableDictionary<string, Rule>.Empty;

public Rule? Get(string k) => _rules.GetValueOrDefault(k);   // no synchronisation

public void Reload(IEnumerable<Rule> rules) =>
    Volatile.Write(ref _rules, rules.ToImmutableDictionary(r => r.Key));
```

Finally, the unsynchronised collections. `Dictionary<K,V>` and `List<T>` are not merely "last write wins" under concurrent mutation: they can corrupt internally, producing an infinite loop inside `Dictionary` lookup or an `IndexOutOfRangeException` from `List.Add`. A hung thread spinning inside `Dictionary.FindEntry` is a well-known signature of unsynchronised concurrent access.

## 6.8 Channels

`System.Threading.Channels` is the modern producer/consumer primitive: an async, allocation-conscious, optionally-bounded queue. It replaces `BlockingCollection<T>` and every hand-rolled `Monitor.Wait`/`Pulse` queue.

```csharp
var channel = Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(1000)
{
    FullMode = BoundedChannelFullMode.Wait,   // apply back pressure to producers
    SingleReader = true,                      // enables internal optimisations
    SingleWriter = false
});

// Producer
await channel.Writer.WriteAsync(item, ct);    // asynchronously blocks when full

// Consumer
await foreach (WorkItem item in channel.Reader.ReadAllAsync(ct))
    await ProcessAsync(item, ct);

channel.Writer.Complete();                    // consumers' loop then ends
```

**Bounded versus unbounded is the decision that matters, and the default should be bounded.** An unbounded channel converts a slow consumer into unbounded memory growth, and the failure mode is an OOM kill hours later with no obvious cause. A bounded channel converts it into back pressure at the producer, which is visible, measurable, and usually what you actually want. This is the in-process case of №31 §7.3: the same choice you face between a queue and a stream between services, at nanosecond rather than millisecond scale. `BoundedChannelFullMode` also offers `DropOldest`, `DropNewest` and `DropWrite` for telemetry-style workloads where losing data beats losing the process.

.NET 9 added `Channel.CreateUnboundedPrioritized<T>`, which dequeues by priority rather than FIFO.

> **The tell — which concurrent structure:** if it is a queue of work between stages, use `Channel<T>`, bounded. If it is a shared lookup written occasionally and read constantly, use an immutable dictionary swapped with `Volatile.Write`. If it is a shared lookup written constantly, use `ConcurrentDictionary` and only its atomic composite methods. If you need two operations to be atomic together, none of these help and you need a lock.

---

# Part 7 — Data parallelism

## 7.1 `Parallel` and `Parallel.ForEachAsync`

`Parallel.For` and `Parallel.ForEach` partition a range or collection across pool threads and block until every partition is done. They are for **CPU-bound** work over independent items.

```csharp
Parallel.ForEach(images, opts, img => Resize(img));    // synchronous, blocks the caller
```

`Parallel.ForEachAsync` (.NET 6+) is the async-aware version and the one you usually want, because it does not block, and because `MaxDegreeOfParallelism` gives you the throttling that a naive `Task.WhenAll` lacks:

```csharp
await Parallel.ForEachAsync(
    urls,
    new ParallelOptions { MaxDegreeOfParallelism = 8, CancellationToken = ct },
    async (url, token) => await FetchAndStoreAsync(url, token));
```

Note what changed: the delegate is `Func<T, CancellationToken, ValueTask>`, so async lambdas bind correctly rather than becoming `async void`. Passing an async lambda to plain `Parallel.ForEach` is the §3.3 bug, and it is common: the loop reports completion as soon as every lambda has *started*.

Exceptions from `Parallel` are collected into an `AggregateException` after the remaining iterations finish; the loop does not stop at the first failure unless you cancel it. `ParallelLoopState.Break()` and `.Stop()` let you exit early, with `Break` completing all lower-indexed iterations and `Stop` abandoning immediately.

## 7.2 PLINQ

`.AsParallel()` turns a LINQ query into a parallel one. It is elegant and it is very often slower than the sequential version, because partitioning, merging, and delegate invocation cost more than the work per element.

```csharp
var hits = documents.AsParallel()
                    .WithDegreeOfParallelism(4)
                    .Where(d => Matches(d, pattern))     // must be expensive to pay off
                    .ToList();
```

PLINQ pays when there are many elements, the per-element work is substantial (microseconds, not nanoseconds), the operations are pure, and you do not need ordering. `AsOrdered()` restores order and costs a large fraction of the benefit. Measure it; do not assume it.

## 7.3 Partitioning, accumulation, and false sharing

Two mechanical facts determine whether parallel loops actually scale.

**Aggregate thread-locally.** A shared accumulator turns your parallel loop into a serialised queue on one cache line.

```csharp
// BROKEN: every iteration contends on one variable.
long total = 0;
Parallel.ForEach(items, x => Interlocked.Add(ref total, Weigh(x)));

// FIXED: per-thread accumulator, combined once per thread at the end.
long total = 0;
Parallel.ForEach(items,
    () => 0L,                                   // thread-local init
    (x, state, local) => local + Weigh(x),      // body
    local => Interlocked.Add(ref total, local)  // thread-local finaliser
);
```

**False sharing** is the same problem at a lower level. CPU caches work in 64-byte lines. Two threads writing to *different* variables that happen to share a cache line will invalidate each other's cache on every write, producing contention with no logical sharing at all. An array of per-thread counters, `long[] counts` indexed by thread, is the classic instance: eight counters, one cache line, no scaling whatsoever. Pad to a cache line, or use the thread-local pattern above and avoid the array.

## 7.4 Why this is usually wrong on a web server

A web server is **already parallel**: it is handling many requests concurrently, and under load every core is already busy. Using `Parallel.ForEach` inside a request handler does not add throughput, because there is no idle capacity to claim. It takes threads from the same pool that serves other requests, so it improves the latency of *this* request by degrading everyone else's, and it does so unpredictably. Under real load it usually makes p99 latency worse across the board while improving nothing. №43 has the capacity arithmetic; the short version is that a server's throughput is set by its bottleneck resource, and taking more of that resource per request lowers it.

The exceptions are narrow and real: a batch endpoint or background job with known low concurrency, work that is genuinely long and genuinely CPU-bound, or a worker service whose entire job is one computation at a time.

> **The tell — parallelise or not:** parallelise only if the work is CPU-bound, the items are independent, there is idle CPU capacity to use, and you have measured the sequential version and found it too slow. On a request path under concurrent load, three of those four are usually false. Concurrency (`async`) increases how many requests you can serve; parallelism decreases how long one takes, at the cost of the others.

---

# Part 8 — Internals

## 8.1 The generated state machine

Take this method:

```csharp
async Task<int> GetTotalAsync(int id)
{
    Order o = await _repo.GetAsync(id);
    int shipping = await _rates.GetAsync(o.Region);
    return o.Total + shipping;
}
```

The compiler rewrites it into roughly the following (simplified: the real output uses `AsyncTaskMethodBuilder`, hides the awaiters in fields, and is far less readable):

```csharp
struct StateMachine : IAsyncStateMachine
{
    public int _state;                       // -1 = running, 0/1 = resume points
    public AsyncTaskMethodBuilder<int> _builder;
    public MyClass _this;
    public int _id;
    public Order _o;                         // a LOCAL, hoisted to a field
    public int _shipping;                    // ditto
    private TaskAwaiter<Order> _awaiter1;

    public void MoveNext()
    {
        try
        {
            switch (_state)
            {
                case -1:
                    _awaiter1 = _this._repo.GetAsync(_id).GetAwaiter();
                    if (!_awaiter1.IsCompleted)          // the fast path check
                    {
                        _state = 0;
                        _builder.AwaitUnsafeOnCompleted(ref _awaiter1, ref this);
                        return;                          // <-- "await is a return"
                    }
                    goto case 0;
                case 0:
                    _o = _awaiter1.GetResult();          // rethrows if faulted
                    /* ... second await, same shape ... */
                    break;
            }
            _builder.SetResult(_o.Total + _shipping);
        }
        catch (Exception e) { _builder.SetException(e); }
    }
}
```

Four things fall out of this that explain behaviour you will otherwise find mysterious.

**Locals become fields.** Anything alive across an `await` is hoisted into the state machine, and the state machine is boxed onto the heap the first time the method suspends. A large `struct` local held across an await is copied to the heap; a `Span<T>` cannot be held across an await at all, and the compiler rejects it (CS4007), because it cannot live on the heap.

**The struct is only boxed if you suspend.** A method that always completes synchronously never allocates the box. This is why `IsCompleted` fast-pathing matters and why `ValueTask` exists.

**Stack traces are the state machine's.** Frames read as `MyClass+<GetTotalAsync>d__7.MoveNext()`, and the *logical* caller is not on the physical stack at all, because it returned long ago. `AsyncTaskMethodBuilder` stitches enough back together to make traces usable, but this is why async stack traces have historically been poor.

**Exceptions become task state.** The whole body is inside one try/catch that routes everything to `SetException`. Nothing escapes an async method synchronously except exceptions thrown before the builder starts, which is why §2.5's advice about where to put the `try` holds.

## 8.2 `TaskCompletionSource`: the bridge from callbacks

`TaskCompletionSource<T>` gives you a `Task` you complete manually. It is how you adapt a callback-based, event-based, or message-driven API into the async model.

```csharp
public Task<Response> SendAsync(Request r, CancellationToken ct)
{
    var tcs = new TaskCompletionSource<Response>(
        TaskCreationOptions.RunContinuationsAsynchronously);   // see below

    _pending[r.CorrelationId] = tcs;
    _transport.Send(r);

    ct.Register(static s => ((TaskCompletionSource<Response>)s!).TrySetCanceled(), tcs);
    return tcs.Task;
}
// elsewhere, on the receive path:
void OnResponse(Response resp) => _pending[resp.CorrelationId].TrySetResult(resp);
```

**`RunContinuationsAsynchronously` is not optional in production code.** Without it, `TrySetResult` runs every awaiting continuation *inline, synchronously, on the thread that called it*. If that thread is your socket read loop, you have just run arbitrary user code on your I/O thread, and if that code blocks, your transport stops. Worse, an inline continuation that takes a lock the completer already holds is an instant deadlock. The flag makes completion queue the continuations to the pool instead.

Prefer the `Try*` methods over `Set*`: `SetResult` on an already-completed source throws, and in a race between a response arriving and a timeout firing, both will try.

## 8.3 The awaitable pattern

`await` is structural, not nominal. Anything is awaitable if it has an accessible `GetAwaiter()` returning a type that implements `INotifyCompletion` and exposes `bool IsCompleted` and `T GetResult()`. That is the entire contract, and it is why you can `await` a `Task`, a `ValueTask`, a `YieldAwaitable` from `Task.Yield()`, and (via extension methods that ship in the BCL) things that are not tasks at all.

You will rarely implement one. It is worth knowing it exists because it explains how libraries make their own types awaitable without inheriting from anything, and because `[AsyncMethodBuilder(typeof(...))]` lets a library define what `async MyType Foo()` compiles into.

## 8.4 `TaskScheduler` and where continuations run

`TaskScheduler` decides on what thread a task's delegate runs. `TaskScheduler.Default` is the thread pool. `TaskScheduler.FromCurrentSynchronizationContext()` marshals to a UI thread. `ConcurrentExclusiveSchedulerPair` gives you a reader/writer pair of schedulers, which is an occasionally elegant way to serialise writes without a lock.

Two facts to keep straight, because people mix them up: `TaskScheduler` affects tasks scheduled through the TPL (`Task.Run`, `StartNew`, continuations). `SynchronizationContext` affects `await` resumption. `ConfigureAwait(false)` opts out of the second, not the first.

## 8.5 `IValueTaskSource` and pooling

For allocation-free async in the hottest possible paths (sockets, pipes, the ASP.NET Core server itself), `ValueTask` can wrap an `IValueTaskSource<T>` implementation rather than a `Task`. The source object is pooled and **reused after the value task is consumed**, which is the real reason the "consume exactly once" rule in §3.7 is not merely advisory: consuming twice reads a recycled object now serving a different operation.

`[AsyncMethodBuilder(typeof(PoolingAsyncValueTaskMethodBuilder<>))]` lets you opt an `async ValueTask` method into a pooled builder, trading a small pooling overhead for zero steady-state allocation. This is a library-author tool. Measure before, measure after, and do not litter application code with it.

## 8.6 Runtime async (.NET 11, preview)

Everything above describes the compiler-generated state machine model, unchanged in shape since C# 5. .NET 11 is introducing **runtime async**, which moves suspension and resumption into the runtime itself: the compiler emits simpler IL without a full `IAsyncStateMachine`, and the JIT and runtime handle the suspend/resume.

The expected payoffs are stack traces without synthetic `MoveNext` frames, no boxed state machine, potentially no `Task` allocation for chains of async calls that complete without suspending, and cross-method optimisations that a compiler-only transform cannot perform.

Status as of .NET 11 Preview 1: CoreCLR support is on by default in the runtime, but generating runtime-async code requires opting in per project (`<EnablePreviewFeatures>true</EnablePreviewFeatures>` and `<Features>$(Features);runtime-async=on</Features>`), and **none of the core runtime libraries are compiled with it yet**, so most of the benefit is still ahead. Your source code does not change. Treat this as something to watch rather than adopt: it will make the model in §8.1 a description of history rather than of the present, but not this year.

---

# Part 9 — Practice on a server

## 9.1 DI lifetimes are a thread-safety contract

№14 covers the container model, scopes and the lifetime traps in the abstract. The point to add here is that **the three lifetimes in `Microsoft.Extensions.DependencyInjection` are, read correctly, a statement about concurrency**:

| Lifetime | Concurrency implication |
|---|---|
| `Transient` | New instance per resolution. No sharing, no thread-safety requirement. |
| `Scoped` | One per request (per `IServiceScope`). Confined to one logical flow, so single-threaded *unless you fan out inside the request*. |
| `Singleton` | Shared by every concurrent request. **Must be thread-safe.** |

Any mutable field on a singleton is shared mutable state across every request in the process, and §5 applies in full. This is the most common way concurrency bugs enter application code that never mentions a thread: someone caches "the current user" or "the last result" in a field on a singleton service.

The related trap is the **captive dependency**: injecting a scoped service into a singleton. The container resolves it once, at singleton construction, and that one instance is then shared by every request forever. For a `DbContext` (scoped by default) this produces exactly the failure in §9.4. The default container detects and throws on this at validation time when `ValidateScopes` is on, which it is in the Development environment; make sure it is on in your integration tests too.

Fanning out inside a request breaks the scoped guarantee as well. `Task.WhenAll` over three operations that each use the same injected scoped `DbContext` is concurrent access to a non-thread-safe object.

## 9.2 The request path

ASP.NET Core dispatches each request onto a thread pool thread with **no `SynchronizationContext`**. Every `await` in your handler may resume on a different pool thread than the one it started on. This is invisible and fine, with two consequences worth naming: thread-affine state (`ThreadLocal`, `[ThreadStatic]`, thread identity, thread culture set imperatively) is unreliable across awaits, and `HttpContext` must not be touched after the response completes, because it is pooled and recycled. Fire-and-forget work that captures `HttpContext` and uses it later is reading a recycled object serving somebody else's request.

## 9.3 `BackgroundService` and the host

`BackgroundService.ExecuteAsync` is started, not awaited, by the host. But "started" means called: the host calls it and waits for it to **return or reach its first suspension point**. Any synchronous work before your first `await` runs during host startup, and blocks it.

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    // BROKEN: this runs before the host finishes starting.
    LoadTwoHundredMegabytesOfReferenceData();

    // FIXED: yield first, so the host completes startup and this continues on the pool.
    await Task.Yield();
    LoadTwoHundredMegabytesOfReferenceData();

    using var timer = new PeriodicTimer(TimeSpan.FromMinutes(1));
    while (await timer.WaitForNextTickAsync(stoppingToken))
    {
        try { await DoWorkAsync(stoppingToken); }
        catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested) { throw; }
        catch (Exception ex) { _logger.LogError(ex, "iteration failed"); }  // survive one bad tick
    }
}
```

The try/catch inside the loop is not defensive clutter. **Since .NET 6, an unhandled exception escaping `ExecuteAsync` is logged and stops the entire host by default** (`BackgroundServiceExceptionBehavior.StopHost`). That is a deliberate improvement on the previous behaviour, where the exception was silently swallowed and the service simply stopped doing anything. But it means one transient failure on one iteration takes your process down unless you decide otherwise, per iteration, explicitly.

Note the `when (stoppingToken.IsCancellationRequested)` filter: it distinguishes "we are shutting down, let this propagate" from "the operation cancelled for some other reason", which should be logged like any other failure.

## 9.4 EF Core and `DbContext`

`DbContext` is **not thread-safe**, by design and by documentation, for the same reason and with the same consequences as a Hibernate `Session` (№20): it is a unit-of-work object holding a change-tracking graph, and concurrent mutation of that graph is exactly the problem in §5.3. Concurrent use produces `InvalidOperationException: A second operation was started on this context instance before a previous operation completed`, and if you are unlucky, silent state corruption instead.

```csharp
// BROKEN: three concurrent operations on one DbContext.
await Task.WhenAll(
    _db.Orders.CountAsync(),
    _db.Customers.CountAsync(),
    _db.Products.CountAsync());

// FIXED: sequential on one context...
int a = await _db.Orders.CountAsync();
int b = await _db.Customers.CountAsync();

// ...or a context per concurrent operation, from the factory.
await Task.WhenAll(ids.Select(async id =>
{
    await using var db = await _factory.CreateDbContextAsync();
    return await db.Orders.CountAsync(o => o.CustomerId == id);
}));
```

`AddDbContextFactory` exists precisely for the second shape. Use it whenever a single logical operation needs concurrent database work, and in any singleton or background service, where the scoped-context assumption does not hold.

## 9.5 `HttpClient`

Two opposite mistakes, both common.

Creating an `HttpClient` per request exhausts ephemeral ports: each disposed client leaves sockets in `TIME_WAIT` for a couple of minutes, and under load you run out. (№51 explains why `TIME_WAIT` exists and why it lasts as long as it does; it is not arbitrary.) Holding one static `HttpClient` forever fixes that and introduces a subtler problem: the connection pool never re-resolves DNS, so a failover or a blue/green swap behind a DNS change keeps hitting the old endpoint indefinitely.

`IHttpClientFactory` resolves both by pooling the underlying `HttpMessageHandler` and rotating it on a schedule. If you cannot use it, set the lifetime explicitly:

```csharp
var handler = new SocketsHttpHandler
{
    PooledConnectionLifetime = TimeSpan.FromMinutes(2),   // forces periodic DNS re-resolution
    MaxConnectionsPerServer = 50
};
static readonly HttpClient Client = new(handler);
```

`HttpClient` itself is thread-safe for concurrent requests, so one instance serving many callers is correct.

## 9.6 Fire-and-forget, done properly

Sometimes you genuinely want to start work and not wait: a metric, an audit write, a cache warm. The naive version loses exceptions (§2.5) and, if the host shuts down mid-flight, loses the work.

```csharp
// BROKEN: exceptions vanish; nothing knows this is in flight.
_ = SendAuditAsync(evt);

// ADEQUATE: at least the failure is visible.
_ = SendAuditAsync(evt).ContinueWith(
        t => _logger.LogError(t.Exception, "audit failed"),
        TaskContinuationOptions.OnlyOnFaulted);

// BETTER: hand it to a bounded channel drained by a BackgroundService.
await _auditChannel.Writer.WriteAsync(evt, ct);
```

The third form is the one to reach for by default. It gives you back pressure, a single place to handle failures, a natural drain on shutdown, and a queue depth you can put on a dashboard. Fire-and-forget without a queue is an unmeasurable, unbounded, unloggable background workload, which is a description of an incident waiting to happen.

## 9.7 Diagnosing it in production

№57 covers what to instrument and how to alert on it, and №52 covers reading a hung process from the shell. This section is the .NET-specific layer on top of both: the tools, in the order you should reach for them.

**`dotnet-counters monitor -p <pid> --counters System.Runtime`** is the first thing to run. Watch `ThreadPool Thread Count`, `ThreadPool Queue Length`, and `ThreadPool Completed Work Item Count`. Queue length climbing while thread count creeps up by ones is starvation (§1.5), diagnosed in thirty seconds.

**`dotnet-stack report -p <pid>`** prints managed stacks for every thread. If dozens of them are sitting in `Task.Wait`, `ManualResetEventSlim.Wait` or `Monitor.Enter`, you have found your blocking call and its caller.

**`dotnet-dump collect`** then `dotnet-dump analyze`, with the SOS commands `!threadpool` (queue and thread counts), `!syncblk` (which thread owns which monitor lock: this is how you resolve a deadlock), and `!dumpasync` (walks the async state machines on the heap and reconstructs pending async chains, which no stack trace can show you).

**`dotnet-trace`** with the `TasksAndTaskScheduling` provider when you need to see the scheduling itself.

> **The tell — first move on a concurrency incident:** get `ThreadPool Queue Length` and CPU utilisation before touching anything else. High queue plus low CPU means blocking; find it with `dotnet-stack`. High CPU means computation; profile it. Neither means the problem is not in the pool and you should look at the dependency you are calling.

---

# Part 10 — When to use what

**A. Thread, thread pool, or async?**

Tell → dedicated `Thread`: the work runs for the process lifetime, or needs a non-default stack, priority or apartment state. Tell → `Task.Run` on the pool: short CPU-bound work in application code that would otherwise block a request or UI thread. Tell → plain `async`/`await`: the work is I/O, or is composed of things that are. Default: **`async`/`await`, with no `Thread` and no `Task.Run` anywhere in the call chain.**

**B. Wrap in `Task.Run`?**

Tell → yes: you are in application code, the work is genuinely CPU-bound, and it sits on a latency-sensitive path. Tell → no: you are in library code, or the work is already async, or you are doing it to "make it async". Default: **no.**

**C. `Task` or `ValueTask`?**

Tell → `ValueTask<T>`: measured hot path, completes synchronously most of the time, single `await` at every call site. Tell → `Task`: everything else, and anything public whose callers might store or re-await it. Default: **`Task`.**

**D. Which memory-safety tool?**

Tell → immutability: the value can be replaced wholesale rather than mutated. Tell → `Interlocked`: a single counter, flag, or reference publication. Tell → `Volatile.Read`/`Write`: a flag whose visibility (not atomicity) is the issue. Tell → a lock: two or more operations must be seen as one unit. Tell → an explicit barrier: essentially never. Default: **immutability, then `Interlocked`, then a lock.**

**E. `lock` or `SemaphoreSlim`?**

Tell → `lock` (on a `System.Threading.Lock`): the critical section is short, purely CPU, and contains no `await`. Tell → `SemaphoreSlim`: the critical section contains an `await`, or you want to throttle rather than exclude. Default: **`lock` on a `System.Threading.Lock`; restructure so the await falls outside the critical section if you can.**

**F. Which shared collection?**

Tell → immutable snapshot swapped with `Volatile.Write`: read constantly, written rarely. Tell → `ConcurrentDictionary`: written often, and every composite operation you need exists as an atomic method on it. Tell → a plain collection under a lock: you need multi-step atomicity, or the collection is small and contended lightly. Default: **`ConcurrentDictionary` for shared mutable lookups, immutable snapshot for configuration-shaped data.**

**G. Bounded or unbounded channel?**

Tell → unbounded: the producer is provably rate-limited by something else, and you have measured the peak. Tell → bounded, `FullMode.Wait`: you want back pressure (almost always). Tell → bounded, `FullMode.DropOldest`: the data is telemetry and freshness beats completeness. Default: **bounded, `Wait`, with the capacity chosen deliberately and the queue depth on a dashboard.**

**H. Parallel, concurrent, or sequential?**

Tell → `Parallel.ForEachAsync` with a degree limit: many independent items, real per-item cost, and spare capacity. Tell → `Task.WhenAll` with a `SemaphoreSlim`: the items are I/O and you need to bound concurrency. Tell → sequential: on a request path under load, or when the items are cheap. Default: **sequential until measured, then bounded concurrency, and only then parallelism.**

**I. `ConfigureAwait(false)`?**

Tell → yes, on every await: you are writing a library that others may call from WPF, WinForms, or MAUI. Tell → no: you are writing an ASP.NET Core application, where there is no context to capture. Default: **yes in libraries, omit in ASP.NET Core application code, and enforce whichever you pick with an analyser rather than review.**

**J. How to express a deadline?**

Tell → `task.WaitAsync(timeout)`: you only need to stop waiting. Tell → `new CancellationTokenSource(timeout)` and pass the token down: you need the underlying work to stop too. Tell → `CreateLinkedTokenSource`: several independent reasons to stop, and remember to dispose it. Default: **a `CancellationTokenSource` with a timeout, threaded all the way down.**

---

# How to expand this

## Already in the library

Read these rather than duplicating them here.

| Document | What it gives this one |
|---|---|
| **№12 Concurrency** | The concepts underneath every section of Part 5 and Part 6: happens-before, the bug taxonomy, why testing cannot establish correctness. Read it first if any of §5 felt like assertion rather than explanation. |
| **№31 Distributed Systems** | Where §5.6 goes when the two parties are on different machines: fencing tokens (§6.3), idempotency (§8.1), back pressure (§7.3), timeouts and retries (§10). |
| **№13 JVM Internals** | The runtime-level machinery §5.1 and §8.1 assume: JIT behaviour, allocation cost, why "it allocates" matters. The JVM is the subject, but the mechanisms transfer. |
| **№14 Frameworks & DI** | Container lifetimes and scopes, which §9.1 reads as a concurrency contract. |
| **№20 Java Data-Access** | The unit-of-work model behind §9.4's `DbContext` rule. |
| **№51 Networking** | `TIME_WAIT`, connection pooling and DNS, behind §9.5. |
| **№57 Observability** | What to measure and alert on, around §9.7's .NET-specific tooling. |

## The .NET band, 80–89

This document opens the band. The library covers the JVM stack densely and .NET not at all, which is now the wrong shape. The gaps, in the order they would pay off:

- **№81 C# and .NET — A Primer.** The counterpart to №10: the language's model and idioms, records and structs, `IDisposable` and `using`, LINQ against Streams, nullable reference types, the type system's actual differences. The single highest-value missing document.
- **№82 ASP.NET Core.** Kestrel, the middleware pipeline, `System.IO.Pipelines`, minimal APIs, configuration and hosting. Pairs directly with §9, which currently assumes it.
- **№83 EF Core.** The counterpart to №20 and №21: change tracking, the query pipeline, migrations, and where it diverges from Hibernate in ways that bite.
- **№84 Azure.** The counterpart to №54 and №91, and the most directly useful of these for consultancy work.
- **№85 The .NET runtime.** GC generations and modes, tiered compilation and OSR, Native AOT, `Span<T>` and the allocation-free style. §3.7, §8.1 and §8.5 all lean on a price list this document does not supply.

## Candidates for deeper treatment

**`System.Threading.Channels` and back-pressure design**: §6.8 covers a fraction of it, and the subject is really queue theory plus API design. **Lock-free data structures**: §5.6 is an introduction, not a treatment, and the honest version needs a machine model. **CPU architecture for application developers**: cache lines, coherence protocols and store buffers are the actual mechanism behind §5.9 and §7.3, and no document in the library owns them.

---

*Mechanisms described here (monitors, compare-and-swap, the state machine transform, pool work-stealing) are stable and have been for a decade; they will not drift. The memory model statements in §5, including the atomicity guarantee for aligned primitives up to pointer size and the object-assignment release rule in §5.8, were checked against the dotnet/runtime memory model specification rather than written from memory, because that specification is stronger than ECMA-335 and most published advice predates it. Version-specific claims were verified on 10 September 2026 against Microsoft documentation and the dotnet/core release notes: .NET 10 is the current release and current LTS, shipped 11 November 2025 and supported to 14 November 2028; .NET 8 (LTS) and .NET 9 (STS) both leave support on 10 November 2026; `System.Threading.Lock` arrived in .NET 9 with C# 13; `Task.WhenEach` and `Channel.CreateUnboundedPrioritized` in .NET 9; `ConfigureAwaitOptions` in .NET 8; `Parallel.ForEachAsync`, `PeriodicTimer` and `Task.WaitAsync` in .NET 6; the portable thread pool became the default in .NET 6; `BackgroundServiceExceptionBehavior.StopHost` became the default in .NET 6; thread pool injection beyond the minimum is documented at 1 to 2 threads per second. Runtime async (§8.6) is a .NET 11 preview feature and its details are the most likely thing here to change: re-check it against the .NET 11 release notes before relying on any of it. Every code sample here was compiled against the .NET 10.0.112 SDK, and the behavioural claims (`Interlocked` return-value conventions, `AsyncLocal` not flowing upward, `await` versus `Wait` exception shapes, `SuppressThrowing`, `Thread.Abort` throwing `PlatformNotSupportedException`, default minimum worker threads equalling `Environment.ProcessorCount`) were executed rather than asserted. The compiler diagnostics cited (CS0677, CS0420, CS1996, CS4007, CS9216, CA2261) were reproduced. The nanosecond figures in §5.5 and §6.1 were measured on a shared two-core virtual machine: treat the ratios between them as the signal and the absolute values as indicative only.*
