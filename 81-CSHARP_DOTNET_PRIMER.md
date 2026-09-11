# C# and .NET — A Primer №81

*Covers the C# language and the .NET platform underneath it: the runtime and its type system, the SDK and project model, how values and references actually differ in memory, records and pattern matching, nullable reference types, collections and LINQ, disposal and lifetime, exceptions, text and time and serialisation, and the toolchain you build with. The lens is server-side application code on .NET 10. Deliberately excludes everything concurrent: threads, `Task`, `async`/`await`, cancellation, the memory model, locks and channels are **№80**'s subject and are referenced rather than repeated. Also excludes ASP.NET Core (№82), EF Core (№83), Azure (№84), and the runtime's performance machinery, meaning garbage collection modes, tiered compilation and Native AOT internals (№85).*

---

## The organising idea

C# looks like an ordinary object-oriented language with a large standard library, and if you read it that way most of it works and a specific list of things surprise you. Every item on that list is the same item.

**C#'s type system is split in two at the level of the runtime, not the compiler. A value type *is* its data; a reference type is a pointer to data that lives somewhere else. This is not an optimisation the runtime is free to undo, and it is not a hint: it changes what assignment means, what equality means, what `null` means, what passing to a method means, and what a field of that type costs.** Assigning a `struct` copies every byte of it. Assigning a `class` copies eight bytes and now two names refer to one object. That single fork explains boxing, `default`, why `int?` and `string?` are unrelated mechanisms wearing the same syntax, why mutable structs eat writes, why `==` means one thing on `string` and a different thing on `object`, why `Span<T>` needed a third category of type invented for it, and why C# has both `class` and `record` and both `struct` and `record struct`.

Hold that fork in mind and the language stops having a list of exceptions. It has one seam, and the surprises are the places you can see it.

## The second idea

**Nothing in C# is erased.** The compiler emits IL that carries the full type system with it, and the runtime enforces it. `List<int>` is a distinct type at runtime whose backing array holds `int`s laid out inline, not boxed objects. `typeof(T)` inside a generic method returns the real `T`. Attributes survive compilation and are readable by reflection or by a source generator. The runtime knows the exact type of every object on the heap.

That is the fact underneath a surprising amount of the ecosystem. It is why there was never a separate "primitive collections" story to learn: the general collections were always efficient over value types. It is why dependency-injection containers, serialisers, ORMs and test frameworks can all work by looking at types at startup, and why the same jobs can now be done at compile time instead by source generators reading the same information (§9.5). And it is why the allocation-free style, `Span<T>` and `ref struct` and `stackalloc` (§2.12), is expressible *in the language* rather than bolted on beside it: the runtime can be told about a type that must never reach the heap, because the runtime is the thing that decides.

---

## Contents

| Part | Covers |
|---|---|
| **1. The platform** | CLR, BCL and SDK; IL and JIT; assemblies and target frameworks; the project file; NuGet; deployment models; the historical wreckage; release cadence and support |
| **2. The type system** | Values and references, layout, `struct`/`class`/`record`, boxing, `default`, equality in five senses, nullable value types, generics and why they are reified, `Span<T>` and `ref struct` |
| **3. Declaring things** | `var`, target-typed `new`, primary constructors, `required`/`init`, collection expressions, records in full, pattern matching, `field`, extension members, the syntax you will actually meet |
| **4. Nullability** | What nullable reference types are and are not, the attributes, `!`, the interop holes, how to turn it on in an existing codebase |
| **5. Collections and LINQ** | The interface hierarchy, cost table, comparer contracts, immutable and frozen collections; then LINQ: iterators, deferred execution, the operators, the traps, `IQueryable` and expression trees |
| **6. Resources and lifetime** | `IDisposable`, `using`, the dispose pattern, finalizers, `SafeHandle`, `Lazy<T>`, static initialisation, disposal as a DI contract |
| **7. Errors** | No checked exceptions and what that does to design, filters, `throw` versus `throw ex`, `TryXxx`, guard clauses, cost, and where union types land |
| **8. Text, time, numbers, serialisation** | UTF-16 and `string`, the comparison traps, `StringBuilder` and spans, formatting and culture, `DateTimeOffset`/`DateOnly`/`TimeProvider`, `decimal` versus `double`, `System.Text.Json` |
| **9. The toolchain in practice** | Solutions and shared build files, central package management, analysers, warnings as errors, source generators, reflection, interop, testing |
| **10. When to use what** | Decision axes |

---

## The surprise index

Use this when the language has just done something you did not ask for.

| When you notice… | Go to |
|---|---|
| A mutation to a struct silently disappeared | §2.2 copying, §2.10 mutable structs |
| `==` compares two objects that are obviously equal and says `false` | §2.6 the five equalities |
| Two records that hold identical values are not equal | §3.5 `EqualityContract` |
| An object came out of a `Dictionary` lookup that should have hit | §5.3 the comparer contract |
| A LINQ query ran twice, or hit the database twice | §5.6 deferred execution, §5.8 multiple enumeration |
| A LINQ query threw `NotSupportedException` or silently loaded the table | §5.9 `IQueryable` and client evaluation |
| A `null` appeared in a variable the compiler swore was not null | §4.4 the interop holes |
| The nullable warnings are all gone after you added `!` somewhere | §4.3 the null-forgiving operator |
| `StartsWith` behaves differently on the build server than on your machine | §8.2 ordinal versus culture |
| `DateTime` values shifted by an hour after a round trip | §8.5 `Kind` |
| Money is off by fractions of a penny | §8.7 `decimal` versus `double` |
| Serialisation produced `{}` or dropped a property | §8.8 fields, casing, and access |
| A file handle stayed open long after the object went out of scope | §6.1 disposal, §6.3 finalizers |
| `ObjectDisposedException` from something you never disposed | §6.5 disposal in DI |
| An arithmetic result wrapped around instead of throwing | §2.13 `checked` |
| A `for` variable captured in a lambda gave the wrong value | §3.8 closures |
| A NuGet package restores at one version in one project and another elsewhere | §9.3 central package management |
| A library builds locally and fails on a machine with a different SDK | §1.6 TFMs and roll-forward |
| An `enum` holds a value that is not one of its members | §2.8 |
| An `interface` change broke every implementer, or none of them | §2.14 default interface members |
| The stack trace lost everything above the point of failure | §7.4 `throw ex` |

---

# Part 1 — The platform

## 1.1 What ".NET" actually names

Three things, and the word is used for all of them interchangeably, which is most of the confusion.

The **runtime** (the CLR, CoreCLR in the current implementation) is a native executable that loads your compiled code, lays out objects, compiles intermediate language to machine code, enforces the type system, and runs the garbage collector. The **base class library** (the BCL) is the set of assemblies shipped alongside it: `System.Private.CoreLib`, `System.Collections`, `System.Text.Json`, `System.Net.Http` and several hundred others, containing everything from `int` to HTTP clients. The **SDK** is the developer-side toolchain: the `dotnet` command, the C# compiler (Roslyn), MSBuild, NuGet, the templates, the test host. You install the SDK to build; you need only the runtime to run.

The runtime plus the BCL is what a machine needs. That combination is versioned as a whole and shipped as a whole: **.NET 10 means one runtime, one BCL, one set of guarantees**, and the version of C# you can use is tied to it. There is no separate "language version" you shop for independently, though you can pin one (§9.2).

## 1.2 IL, the JIT, and what "managed" means

The C# compiler does not emit machine code. It emits **intermediate language**, a stack-based instruction set, into an assembly, along with **metadata**: a complete, structured description of every type, member, signature and attribute in that assembly. At run time the JIT compiler translates IL to machine code one method at a time, the first time each method is called.

Two consequences matter at the language level, and both come back repeatedly.

The first is that metadata is complete and available. Nothing about your types is discarded on the way to IL. That is the second organising idea and it is what makes reflection, dependency injection, serialisation, ORMs and source generators all possible from the same information (§9.5, §9.6).

The second is that compilation happens with knowledge the C# compiler did not have: the actual CPU, the actual concrete types flowing through a call site. The JIT specialises accordingly, and it recompiles hot methods at higher optimisation later in the process's life. **This is why C# microbenchmarks written as a loop with a stopwatch are worthless**: you are usually measuring the unoptimised first-tier compilation. Use BenchmarkDotNet, which handles warm-up and tiering, or measure nothing. The machinery itself, tiering, on-stack replacement, inlining heuristics, is №85's subject.

"Managed" means the runtime owns memory: you allocate with `new` and never free, the garbage collector reclaims. The costs and tuning of that are №85's; what you need here is that **allocation is cheap and collection is not**, that collection pauses all your threads briefly, and that the way to make a .NET application fast is usually to allocate less rather than to tune the collector.

## 1.3 Assemblies

An assembly is a `.dll` (or `.exe`) containing IL plus metadata. It is the unit of deployment, of versioning, and of the `internal` access modifier. A project compiles to exactly one assembly.

`internal` members are visible within their own assembly and nowhere else, which makes the assembly the real encapsulation boundary in C#, more so than the namespace. Namespaces are naming only; they confer no access. The standard escape hatch is `InternalsVisibleTo`, usually granted to the test project:

```xml
<ItemGroup>
  <InternalsVisibleTo Include="MyLib.Tests" />
</ItemGroup>
```

Assemblies are loaded lazily, on first use of a type in them, which is why a missing dependency often surfaces as a `FileNotFoundException` deep into a request rather than at startup.

## 1.4 The project file

An SDK-style `.csproj` is small, and unusually for XML it is worth reading in full. This is a realistic one:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Serilog.AspNetCore" />
    <ProjectReference Include="../MyLib/MyLib.csproj" />
  </ItemGroup>
</Project>
```

Every `.cs` file under the project directory is compiled with no listing needed; that is what `Sdk="Microsoft.NET.Sdk"` brings in, along with default build targets and the reference to the framework itself. `ImplicitUsings` adds a set of `global using` directives appropriate to the SDK, which is why a new file can call `Console.WriteLine` and use `List<T>` with no `using` lines at all. `Nullable` switches on the nullable reference type analysis (Part 4) and is on by default in every template shipped since .NET 6.

`Microsoft.NET.Sdk` is the plain one. `Microsoft.NET.Sdk.Web` adds the ASP.NET Core framework reference and web-specific build behaviour; `Microsoft.NET.Sdk.Worker` does the same for hosted background services. **The SDK attribute is a real behavioural switch, not a label**: a web project with the wrong SDK will fail to resolve half the framework.

## 1.5 NuGet and restore

`PackageReference` names a dependency; restore resolves the whole graph and writes `obj/project.assets.json`, which is what the compiler and runtime actually read. Two rules of the resolver are worth knowing because they explain most dependency surprises.

**Version numbers are minimums, not exact matches.** `Version="8.0.1"` means "8.0.1 or the nearest higher version that satisfies everything else", and NuGet performs *nearest-wins* resolution: if two packages in your graph want different versions of a third, the version closest to your project in the graph wins, and a warning (NU1605) appears only if that means downgrading. Pin exactly with `[8.0.1]` if you must, which you rarely should.

**Transitive dependencies are real references.** Anything your dependency depends on is compiled against and shipped. `dotnet list package --include-transitive` shows the full set, and `--vulnerable` cross-references it against the GitHub advisory database; that command belongs in CI (№60 covers the supply-chain argument).

## 1.6 Target frameworks

The **target framework moniker** in `<TargetFramework>` says which platform surface you compile against. The ones you will see:

| TFM | Means |
|---|---|
| `net10.0` | .NET 10, any operating system |
| `net10.0-windows` | .NET 10 plus the Windows-only APIs (registry, WinForms, WPF) |
| `netstandard2.0` | The old portable surface; still the target for Roslyn analysers and source generators, and for any library that must be consumable from .NET Framework |
| `net48` | .NET Framework 4.8, a different runtime entirely (§1.8) |

A library can multi-target: `<TargetFrameworks>net10.0;netstandard2.0</TargetFrameworks>` builds twice and ships both, with `#if NETSTANDARD2_0` guarding the differences. Applications should not multi-target; they have exactly one runtime.

**Roll-forward** is the runtime-side counterpart. A framework-dependent application built for `net10.0` will run on the highest installed 10.x patch by default, and will not silently move to .NET 11. `<RollForward>` in the project file or `DOTNET_ROLL_FORWARD` at run time change that policy; `LatestMajor` is the setting that lets a `net8.0` application run on a machine that only has .NET 10 installed, and is the usual answer to "it builds locally and the container says the framework is missing".

> **The tell — which TFM to target:** applications target the current LTS, single-target, and upgrade on the LTS cadence. Libraries you publish internally target the same. Libraries you publish publicly, or that must be consumed by a .NET Framework codebase, multi-target `netstandard2.0` and pay for it with `#if`. Roslyn analysers and source generators have no choice: `netstandard2.0`, because that is what the compiler host loads.

## 1.7 Deployment models

| Model | What ships | When |
|---|---|---|
| **Framework-dependent** (default) | Your IL, plus dependencies. Needs a matching runtime installed | Container images built `FROM` a runtime image; any machine you control |
| **Self-contained** | The above plus the entire runtime and BCL, per platform | Machines where you cannot install a runtime; ~70 MB before trimming |
| **Single-file** | Self-contained, packed into one executable that unpacks or runs in place | Distribution convenience, not a performance feature |
| **ReadyToRun** | Ahead-of-time compiled IL alongside the IL, so the JIT has less to do at startup | Startup-latency-sensitive services; larger artifacts |
| **Native AOT** | A native binary with no JIT and no IL at all | Fast startup and low memory: CLI tools, serverless, small services. Costs you reflection, dynamic loading and `System.Reflection.Emit` |

Native AOT is the one with real design consequences, because the reflection-based patterns much of the ecosystem is built on stop working: anything that discovers types at run time must be replaced by a source generator that discovered them at compile time. If Native AOT is a goal, it constrains library choice from day one. The mechanics belong to №85.

## 1.8 The historical wreckage, and what of it still matters

There is a decade of naming to filter out when reading anything older than about 2021.

**.NET Framework** (up to 4.8.1) is the original Windows-only runtime, shipped as a Windows component. It is still supported for as long as the Windows versions carrying it are, and it receives security fixes and nothing else. It is a **different runtime**, not an older version of the current one: code does not move between them by changing a version number, and a large legacy .NET Framework codebase is a migration project, not an upgrade. In consultancy work you will meet it.

**.NET Core** 1.0 to 3.1 was the cross-platform rewrite. **.NET 5** dropped "Core" from the name and skipped 4 to avoid colliding with Framework 4.x. Everything from 5 onwards is one line: what people now just call .NET.

**.NET Standard** was a specification of an API surface that multiple runtimes agreed to implement, so that one library could be consumed by Framework, Core, Xamarin and Unity. It stopped at 2.1 and is effectively frozen. Its only live role is `netstandard2.0` as described above.

**Mono** is a second runtime implementation, now maintained in the same repository, used for WebAssembly and mobile. **Xamarin** was folded into MAUI.

The practical filter: **if a Stack Overflow answer mentions `app.config`, the GAC, `HttpWebRequest`, `AppDomain` or `ConfigurationManager`, it is Framework-era and may not apply.** The current equivalents are `appsettings.json` and `IConfiguration`, NuGet, `HttpClient`, `AssemblyLoadContext`, and the options pattern.

## 1.9 Cadence and support

A release ships every November. **Even-numbered releases are LTS, supported for three years; odd-numbered releases are STS, supported for eighteen months.** Both get patches monthly; both are production-quality, and the difference is only the support window.

As of this writing: **.NET 10 is the current release and the current LTS**, shipped 11 November 2025, supported until 14 November 2028. **.NET 8 (LTS) and .NET 9 (STS) both leave support on 10 November 2026**, which is a genuinely awkward coincidence: a codebase on .NET 8 for stability reasons has the same deadline as one on .NET 9 for feature reasons. **.NET 11 is in release candidate** and reaches general availability in November 2026, at which point it becomes the current STS and .NET 10 remains the LTS.

Upgrades between adjacent versions are usually a one-line TFM change plus a package bump. The breaking-change lists are published per release and are genuinely short; the pain, when there is any, comes from packages that have not yet shipped a build for the new TFM.

## 1.10 The CLI

The `dotnet` command is the whole toolchain. The verbs worth memorising:

```bash
dotnet new webapi -o Orders.Api      # scaffold from a template
dotnet build                          # restore implied
dotnet run                            # build and run
dotnet test                           # discover and run tests
dotnet publish -c Release             # produce a deployable artifact
dotnet format                         # apply .editorconfig style rules
dotnet list package --vulnerable --include-transitive
dotnet new gitignore                  # yes, this is a template
```

`dotnet build` produces something runnable on your machine; **`dotnet publish` produces the thing you deploy**, and the difference bites people who copy `bin/Debug` into a container and find dependencies missing.

Since .NET 10 a single `.cs` file is a runnable program with no project at all. `dotnet run app.cs` compiles and runs it, and directives at the top of the file supply what the project file would have:

```csharp
#!/usr/bin/env dotnet
#:package Humanizer@2.14.1
#:sdk Microsoft.NET.Sdk

Console.WriteLine(DateTime.UtcNow.ToString("O"));
```

That covers the scripting case that previously pushed people to PowerShell or a throwaway console project, and `dotnet project convert app.cs` promotes it to a real project when it outgrows one file.

---

# Part 2 — The type system

## 2.1 The fork

Every type in C# is a **value type** or a **reference type**, and the runtime treats them as different kinds of thing.

A value type variable *contains* the data. `struct`, `enum`, all the built-in numeric types, `bool`, `char`, `DateTime`, `Guid`, `TimeSpan` and every tuple are value types. A reference type variable contains a **reference**, a pointer-sized handle, to an object on the garbage-collected heap. `class`, `interface`, `delegate`, `string`, `object` and every array are reference types.

`record` is not a third category; it is a modifier. `record` and `record class` are reference types, `record struct` is a value type, and both sit on whichever side of the fork their underlying keyword puts them (§3.5).

Everything in this part follows from that split. The rest of the language is largely unsurprising.

## 2.2 What assignment means

Assignment copies the contents of the variable. For a value type, that is every byte of it. For a reference type, that is the reference, eight bytes on a 64-bit runtime, and the object itself is not touched.

```csharp
struct PointS { public int X, Y; }
class  PointC { public int X, Y; }

var a = new PointS { X = 1 };
var b = a;            // full copy
b.X = 99;             // a.X is still 1

var c = new PointC { X = 1 };
var d = c;            // copy of the reference
d.X = 99;             // c.X is now 99: one object, two names
```

The same rule applies to passing arguments, returning values, storing into fields and into collections. **A `List<PointS>` holds copies; a `List<PointC>` holds references.** `list[0].X = 5` on a `List<PointS>` does not even compile, because the indexer returns a copy and mutating a copy is provably pointless; the compiler refuses rather than letting you write a no-op (§2.10).

## 2.3 Where instances actually live

The common statement is "value types live on the stack and reference types on the heap". The first half is wrong often enough to cause real bugs.

**A value type instance lives wherever its storage lives.** A local of type `int` lives in a stack slot or a register. An `int` field of a class lives inside that class's object, on the heap. An `int` element of an array lives inline in the array's block, on the heap. A `struct` captured by a lambda lives in the compiler-generated closure object, on the heap. There is no separate "value type storage"; the type says how the data is arranged, not where.

What is always true: **an object of a reference type is always on the heap and always carries a header.** On 64-bit that header is sixteen bytes, an object header word and a method table pointer, and the smallest possible object is twenty-four bytes. That header is what makes reflection, locking, and virtual dispatch work, and it is also why a million small objects cost more than a million small structs by a factor that is mostly header.

Struct fields are laid out sequentially by default; class fields are laid out however the runtime prefers, usually reordered to pack efficiently. Neither guarantee is one you should build on except through explicit `[StructLayout]` for interop (§9.7).

## 2.4 Boxing

When a value type has to be treated as `object`, or as an interface it implements, the runtime allocates a heap object, copies the value into it, and gives you a reference to that. This is **boxing**. Unboxing copies back out and requires the runtime type to match exactly, otherwise `InvalidCastException`.

```csharp
int i = 42;
object o = i;          // box: an allocation
int j = (int)o;        // unbox: a copy back
long k = (long)o;      // InvalidCastException, not a widening conversion
```

Boxing is silent, which is what makes it a performance problem rather than a correctness one. The places it happens without any syntactic signal:

- assigning to `object` or to a non-generic collection;
- calling an interface method on a struct through an interface-typed variable;
- `string.Format` and older logging APIs taking `params object[]`, which boxes every value argument;
- calling the default `Equals(object)` or `GetHashCode()` on a struct that does not override them (§2.6);
- LINQ over a value type sequence where the query is typed as `IEnumerable` rather than `IEnumerable<T>`.

The important non-case: **generics do not box.** `List<int>` stores `int`s inline. `Dictionary<int, string>` hashes without boxing, provided the comparer path is generic. That is the second organising idea earning its place: because generics are reified, the collection over a value type is a genuinely different runtime type with the value laid out inline, not a collection of `object` with casts.

## 2.5 `default`, and the absence of "no value"

Every type has a default value, and every value type's default is all-bits-zero: `0`, `false`, `'\0'`, `DateTime.MinValue`, an all-zero `Guid`, a struct with every field defaulted. Every reference type's default is `null`.

```csharp
int x = default;                    // 0
Guid g = default;                   // 00000000-0000-0000-0000-000000000000
var p = default(PointS);            // X = 0, Y = 0
```

This has a design consequence worth stating plainly: **a struct always has a valid-looking instance you never wrote.** `new PointS()` and `default(PointS)` are the same thing, arrays of structs arrive pre-populated with it, and no constructor you write can prevent it, because the parameterless one cannot be suppressed. A struct that is only meaningful when constructed properly will still be handed to you zeroed at some point, which is the main argument for making invalid states impossible in the zero value or using a class instead.

## 2.6 Equality has five meanings

This is the largest single source of "why is this false".

| Mechanism | Default behaviour |
|---|---|
| `ReferenceEquals(a, b)` | Identity. Always reference comparison, never overridable |
| `a == b` | A **static, compile-time-bound operator**. On reference types with no overload: identity. On value types with no overload: does not compile. Overloaded on `string`, records, and most BCL structs |
| `a.Equals(b)` | A **virtual, run-time-dispatched method**. Reference types: identity unless overridden. Value types: field-by-field, reflectively, slowly, unless overridden |
| `IEquatable<T>.Equals(T)` | The strongly-typed, allocation-free path. What generic collections look for first |
| `IEqualityComparer<T>` | Equality supplied from outside, by the caller, for one collection or one operation |

The trap that catches everybody, once:

```csharp
string s1 = "hello";
string s2 = new string(['h','e','l','l','o']);

s1 == s2;                       // true:  string's == operator compares values
((object)s1) == ((object)s2);   // false: object's == is identity, chosen at compile time
s1.Equals(s2);                  // true:  virtual, dispatches to string.Equals
```

**`==` is resolved by the compiler from the static types of its operands; `Equals` is resolved by the runtime from the actual object.** Assign a string to an `object` variable and `==` changes meaning without any warning. This is precisely why generic code and collections use `Equals` and `IEqualityComparer<T>` rather than `==`: inside `List<T>.Contains`, the static type is `T`, and `==` would mean identity for every reference type.

For structs, the default `Equals(object)` is worse than slow: it boxes both operands and, unless the struct is bit-comparable, compares fields using reflection. Any struct used as a dictionary key or compared in a loop should implement `IEquatable<T>` and overload `==`/`!=`, or be declared a `record struct`, which generates all of it (§3.5).

The `GetHashCode` contract is the usual one and the consequences are the usual ones: **equal objects must return equal hash codes; a hash code must not change while the object is a key**. Mutating a field that participates in `GetHashCode` after inserting into a `Dictionary` or `HashSet` loses the entry, permanently and silently (§5.3). `HashCode.Combine(a, b, c)` is the correct way to write one; hand-rolled `17 * 31 + ...` is not wrong but is now pointless.

> **The tell — how to give a type equality:** if it is a data-shaped type, make it a `record` or `record struct` and take the generated equality. If it must be a plain `class` or `struct`, implement `IEquatable<T>`, override `Equals(object)` to call it, override `GetHashCode` with `HashCode.Combine`, and overload `==`/`!=` to call it too. Never override one of those without the others; the analysers (CA1067, CA2218) will tell you so. If equality means different things in different contexts, do not put any of it on the type: pass an `IEqualityComparer<T>` at the point of use.

## 2.7 Nullable value types

`int?` is `Nullable<int>`: a struct containing an `int` and a `bool`. It is a real, distinct type with real runtime behaviour, and it exists because a value type has no spare bit pattern to mean "absent" (§2.5).

```csharp
int? a = null;
a.HasValue;          // false
a.Value;             // InvalidOperationException
a.GetValueOrDefault(); // 0
int b = a ?? -1;     // -1
```

Two behaviours are worth carrying. **Lifted operators propagate null**: `a + b` where either is null gives null, and comparisons where either is null give `false`, including `a != b` where both are null, which is `false` in the numeric comparison sense but `true` under `Equals`. And **boxing a nullable is special-cased by the runtime**: boxing a `Nullable<int>` with no value produces a plain `null` reference, not a boxed struct, so `((object)a) == null` is `true` and `a.GetType()` throws. That is a deliberate rule, not an accident, and it is why `object` round-trips of nullables behave sensibly.

None of this is related to `string?`, which is Part 4's subject and is a compile-time fiction with no runtime presence at all. **The `?` suffix means two completely different things depending on which side of the fork the type is on**, and it is the single most misleading piece of syntax in the language.

## 2.8 Enums

An `enum` is a struct wrapping an integral type, `int` unless declared otherwise. That is the whole of it, and it means an enum value is not constrained to the members you declared:

```csharp
enum Status { Pending = 0, Active = 1, Closed = 2 }

var s = (Status)99;          // legal, no exception
s.ToString();                // "99"
Enum.IsDefined(s);           // false
```

Anything cast in from an integer, deserialised, or read out of a database can be a value you never declared. A `switch` over an enum therefore always needs a default arm that treats the unexpected as an error, and **exhaustiveness over an enum is not a thing the compiler can give you** (§3.6). If you need closed alternatives that the compiler enforces, use a sealed hierarchy today (§3.7) or union types when they land (§7.8).

`[Flags]` marks a bit-field enum and changes only `ToString` and parsing; the bitwise operators work either way. Declare the members as explicit powers of two, and use `HasFlag` freely, it stopped being slow years ago.

## 2.9 Choosing between `class` and `struct`

The default is `class`. A struct is an optimisation with real costs, and the case for one has to be made.

Choose a `struct` when **all** of the following hold: the type logically represents a single value; it is small, sixteen bytes or so, three fields of pointer size; it is immutable; it will exist in large numbers or in hot paths where the allocation and the header would matter. `Point`, `Money`, `Rgb`, an identifier wrapping a `Guid`. Notice that the BCL's own value types fit this exactly.

Choose a `class` otherwise, and in particular whenever the type has identity (two customers with the same fields are not the same customer), is polymorphic, is large, or is mutable.

> **The tell — class or struct:** default to `class`, and to `record` when the type is data. Reach for `struct` only when the type is a small immutable value that will exist in quantity, and then make it a `readonly record struct` so you get immutability and correct equality for free. If you find yourself wanting inheritance, mutation, or `null`, you wanted a class.

## 2.10 Mutable structs, and the defensive copy

A mutable struct is the language's sharpest edge, because mutation applied to a copy is silently lost.

```csharp
struct Counter { public int N; public void Bump() => N++; }

var list = new List<Counter> { new() };
// list[0].Bump();   // does not compile: indexer returns a copy

var array = new Counter[1];
array[0].Bump();     // DOES compile and DOES work: array indexing gives a reference

readonly Counter _field = new();
// _field.Bump();    // compiles, but operates on a hidden defensive copy: no effect
```

Three different behaviours from the same call, decided by how the storage is reached. Arrays hand out a real reference to the element; `List<T>`'s indexer is a property returning a copy; a `readonly` field is defensively copied before any non-readonly member is invoked on it, so the mutation lands on the copy and vanishes.

The defensive copy also happens on `in` parameters and on `readonly` fields of any struct type that is not declared `readonly struct`. That is a performance issue as well as a correctness one: passing a large struct by `in` to avoid copying it can produce more copies than passing it by value, because the compiler defensively copies at every member access. **Declaring `readonly struct` removes the defensive copies entirely**, because the compiler then knows no member can mutate.

> **The tell — mutable structs:** don't. Declare every struct `readonly struct` or `readonly record struct` and replace mutation with `with` or a new instance. If a struct genuinely must be mutable for performance (a hot accumulator, an interop buffer), keep it in a local or an array element, never in a field, a `List<T>`, or a captured variable, and write a comment saying why.

## 2.11 `ref`, `in`, `out`, and returning references

C# can pass and return references to storage, not just to objects.

| Modifier | Means |
|---|---|
| `ref` | An alias to the caller's storage. Readable and writable. Must be definitely assigned before the call |
| `out` | An alias the callee **must** assign. Need not be initialised beforehand |
| `in` | A read-only alias. Avoids a copy for large structs; causes defensive copies unless the type is a `readonly struct` |
| `ref readonly` | The same as `in`, but the caller must write `in` or `ref` at the call site, making the aliasing visible (C# 12) |

`out var` inlines the declaration, which is what makes the `TryXxx` pattern readable (§7.6):

```csharp
if (dict.TryGetValue(key, out var value)) { Use(value); }
```

Methods can also *return* references, with `ref` returns and `ref` locals, which is how `List<T>`'s `CollectionsMarshal.AsSpan` and `Dictionary`'s `CollectionsMarshal.GetValueRefOrNullRef` let you mutate storage in place. These are performance tools with sharp edges; you will read them more often than write them.

## 2.12 `Span<T>` and the third kind of type

A `Span<T>` is a pointer and a length: a window onto a contiguous run of memory, wherever that memory is. It can view a stack allocation, an array, a slice of a string, or unmanaged memory, uniformly, without copying.

```csharp
Span<byte> stack = stackalloc byte[128];          // no heap allocation at all
Span<int>  fromArray = new int[100].AsSpan(10, 5); // a 5-element window, no copy
ReadOnlySpan<char> word = "hello world".AsSpan(6); // "world", no substring allocation
```

For this to be safe, the runtime has to guarantee a `Span<T>` never outlives the memory it points at. That is what **`ref struct`** means: a type that may only ever live on the stack. The compiler enforces a list of restrictions to make that true. A `ref struct` cannot be boxed, cannot be a field of a class or of a non-`ref` struct, cannot be captured by a lambda or a local function, cannot be used as a type argument unless the generic declares `allows ref struct` (C# 13), and cannot survive across an `await` or a `yield return` (§80 §3.1 explains why: the state machine hoists locals to the heap). `Memory<T>` is the heap-storable counterpart you use when you need to cross one of those boundaries, and `.Span` gets you back.

C# 14 added implicit conversions between spans and arrays, which removes most of the `AsSpan()` noise from the calling code.

The payoff is that parsing, formatting, slicing and buffer work can be written with no allocation at all: `int.Parse(ReadOnlySpan<char>)`, `Utf8Formatter`, `string.Split` alternatives, `ReadOnlySpan<byte>` over a network buffer. **This is the idiom the modern BCL is written in**, so you will meet spans in signatures long before you need to write them yourself. The runtime-level payoff, and when it is worth the trouble, is №85's.

## 2.13 Numbers and overflow

Integer arithmetic is **unchecked by default**: `int.MaxValue + 1` wraps silently to `int.MinValue`. Constant expressions are checked at compile time, so the literal version is an error and the variable version is not, which is a confusing pair of behaviours to meet in the same afternoon.

```csharp
int max = int.MaxValue;
int wrapped = max + 1;              // -2147483648, no exception
int alsoWrapped = checked(max + 1); // OverflowException
```

Turn it on globally with `<CheckForOverflowUnderflow>true</CheckForOverflowUnderflow>`, which costs a few percent on arithmetic-heavy code and buys you a loud failure instead of a wrong answer. For anything financial or safety-relevant, take that trade.

Exceptions to the rule: `decimal` arithmetic **always** throws on overflow regardless of context, and floating-point arithmetic never does, producing `Infinity` or `NaN` instead. Integer division by zero throws `DivideByZeroException`; floating-point division by zero gives `Infinity`.

Since C# 11, arithmetic is also generic: `INumber<T>` and the interfaces beneath it expose operators as **static abstract interface members**, so you can write `static T Sum<T>(IEnumerable<T> xs) where T : INumber<T>`. This is the only mainstream use of static abstracts most code will have.

## 2.14 Interfaces

Interfaces are the polymorphism you should reach for. Two features change what they are, and both have traps.

**Default interface members** (C# 8) let an interface supply an implementation. The trap is dispatch: the member is visible only through an interface-typed reference, not through the implementing class.

```csharp
interface ILogger { void Log(string m); void LogError(string m) => Log("ERROR: " + m); }
class Console_ : ILogger { public void Log(string m) => Console.WriteLine(m); }

ILogger a = new Console_();
a.LogError("boom");        // fine
new Console_().LogError("boom");  // does not compile
```

Their intended use is adding a member to a published interface without breaking implementers. Using them as a general mixin mechanism produces exactly the confusion above.

**Static abstract members** (C# 11) let an interface require statics, including operators and factory methods, which is what makes `INumber<T>` and `IParsable<T>` work. `T.Parse(s)` inside a generic method is real code.

## 2.15 Generics are reified

At run time, `List<int>` and `List<string>` are different types with different method tables. `typeof(T)` inside a generic method returns the actual argument. `new T()` works given a `new()` constraint. A generic type can be reflected over and its arguments recovered.

The JIT does this by compiling **one shared native implementation for all reference type arguments**, since every reference is the same size and shape, and **a separate specialised implementation for each value type argument**, since the layout differs. That is the mechanical reason `List<int>` stores `int`s inline: there is code compiled specifically for `int`.

Constraints narrow what a type parameter can be and, more usefully, what you can do with it:

| Constraint | Allows |
|---|---|
| `where T : struct` | `T` is a non-nullable value type; enables `T?` meaning `Nullable<T>` |
| `where T : class` | `T` is a reference type |
| `where T : notnull` | Any non-nullable type; the usual choice for dictionary keys |
| `where T : new()` | `new T()` |
| `where T : SomeBase` | Members of `SomeBase` |
| `where T : ISomething` | Members of the interface, **without boxing** if `T` is a struct |
| `where T : unmanaged` | No references anywhere in the layout; enables pointers and `sizeof` |
| `where T : allows ref struct` | `T` may be a `ref struct` (C# 13); the caller loses boxing and heap storage |

The interface constraint deserves its line. `void Do<T>(T x) where T : IThing` calling `x.DoIt()` on a struct does **not** box, because the constraint lets the JIT emit a direct call. `void Do(IThing x)` with a struct argument does box. Same method body, one allocation of difference.

## 2.16 Variance

Interfaces and delegates can declare a type parameter covariant (`out`) or contravariant (`in`). `IEnumerable<out T>` means `IEnumerable<string>` is usable where `IEnumerable<object>` is expected; `Action<in T>` means an `Action<object>` is usable where an `Action<string>` is expected.

Two limits. **Variance applies only to reference conversions**, so `IEnumerable<int>` is not an `IEnumerable<object>` even though `int` boxes to `object`. And **classes are never variant**: `List<string>` is not a `List<object>`, and could not safely be, because you could add an `int` to it.

Arrays break that last rule for historical reasons and pay for it at run time:

```csharp
object[] xs = new string[3];   // legal: arrays are covariant
xs[0] = 42;                    // ArrayTypeMismatchException, at run time
```

Every array store carries a type check because of this. It is a known wart; the reason to know it is that the exception is otherwise inexplicable.

---

# Part 3 — Declaring things

## 3.1 Type inference, and how much of it to use

`var` infers the static type of a local from its initialiser. It changes nothing at run time; the variable is as strongly typed as if you had written the type out.

```csharp
var count = 0;                       // int
var items = new List<Order>();       // List<Order>
List<Order> items2 = new();          // target-typed new: the same thing, other way round
```

Target-typed `new` (C# 9) is the mirror image: the type comes from the left, the arguments from the right. It reads best on fields and on long generic types where the declaration already says everything:

```csharp
private readonly Dictionary<string, List<Order>> _byCustomer = new();
```

The house rule worth adopting: **use `var` when the initialiser names the type, and the explicit type when it does not.** `var orders = new List<Order>()` is fine; `var result = Process(input)` hides something the reader needs. Target-typed `new` for fields and for arguments, `var` for locals with obvious initialisers. Enforce whichever you pick with `.editorconfig` (§9.4) rather than in review.

## 3.2 Properties

Properties are methods that look like fields. The auto-property is the default form and generates a hidden backing field:

```csharp
public string Name { get; set; }
public string Id   { get; }                  // set only in the constructor
public string Slug { get; init; }            // set in the constructor OR an initialiser, then frozen
public required string Email { get; init; }  // the compiler forces the caller to set it
```

`init` (C# 9) is the accessor that makes object-initialiser syntax compatible with immutability: assignable during construction, including from a `with` expression, and read-only afterwards. `required` (C# 11) is the missing half: without it, an `init` property can simply be left unset, and the compiler has no way to insist. Together they give you immutable types constructed by name rather than by positional argument, which is what large data types actually want.

The `field` keyword (C# 14) removes the last reason to write a backing field by hand. Inside an accessor, `field` refers to the compiler-synthesised storage:

```csharp
public string Name
{
    get;
    set => field = value?.Trim() ?? throw new ArgumentNullException(nameof(value));
}
```

Before this, adding any logic to one accessor meant declaring a private field and writing both accessors out. Note the one breaking change it carries: inside a property accessor, `field` now means the backing field, so an existing variable of that name must be written `@field` or `this.field`.

## 3.3 Constructors and initialisers

Object initialiser syntax sets properties after the constructor runs, which is why it needs `set` or `init`:

```csharp
var o = new Order { Id = 7, Customer = "acme", Lines = { line1, line2 } };
```

The nested `Lines = { ... }` form calls `Add` on the existing collection rather than assigning a new one, which is why it works on a get-only collection property.

**Primary constructors** (C# 12 for classes and structs) put constructor parameters in the type header, in scope throughout the body:

```csharp
public class OrderService(IOrderRepository repo, ILogger<OrderService> logger)
{
    public async Task<Order> GetAsync(int id) =>
        await repo.FindAsync(id) ?? throw new NotFoundException(id);
}
```

The compiler captures each parameter into a hidden field only if a member body uses it. This is the shape almost all dependency-injected services should now take, and it removes the field-plus-assignment ceremony entirely.

Two things about it catch people. **The parameters are not properties** on a plain class (they are on a record, §3.5), so nothing is exposed publicly and nothing is generated. And **they are mutable**: a parameter can be assigned to inside a method, and the captured field is not `readonly`, so the compiler cannot stop you from writing `repo = somethingElse`. An analyser can. If a value must be immutable and public, declare a property and assign it from the primary constructor parameter.

## 3.4 Collection expressions

`[...]` (C# 12) constructs any collection the target type supports, and `..` spreads another sequence into it:

```csharp
int[]        a = [1, 2, 3];
List<string> b = ["x", "y"];
Span<byte>   c = [0, 1, 2];
int[]        d = [..a, 99, ..other];
ReadOnlySpan<char> e = [];     // no allocation at all
```

It works for arrays, spans, every BCL collection interface, and any type that either has a `[CollectionBuilder]` attribute or has an `Add` method and implements `IEnumerable`. The compiler picks the cheapest construction it can: for a span or array target with a known length there is no intermediate list, and for `IEnumerable<T>` targets it may produce a compiler-generated read-only type rather than a `List<T>`.

The practical effect is that the old menagerie, `new[] { }`, `new List<T> { }`, `Array.Empty<T>()`, `Enumerable.Empty<T>()`, `ImmutableArray.Create(...)`, collapses to one syntax that is at least as efficient as any of them. Prefer it everywhere.

## 3.5 Records

A record is a class or struct with value semantics generated for it. The positional form is the compact one:

```csharp
public record Person(string First, string Last);
```

For that one line the compiler generates: public `init`-only properties `First` and `Last`; `Equals`, `IEquatable<Person>` and `GetHashCode` comparing every field; `==` and `!=` calling `Equals`; a `ToString` that prints `Person { First = Ada, Last = Lovelace }`; a `Deconstruct` method; a protected copy constructor; and a clone method that powers `with`.

```csharp
var p = new Person("Ada", "Lovelace");
var q = p with { Last = "Byron" };    // copy, then apply the init-only assignments
p == new Person("Ada", "Lovelace");   // true: value equality
```

`with` is a copy-and-modify expression, not a mutation. It calls the copy constructor and then applies the listed `init` assignments, which is exactly why the properties have to be `init` rather than `get`-only.

You can write the body out instead of using positional parameters, and should whenever the type has more than about four members or needs validation:

```csharp
public record Person
{
    public required string First { get; init; }
    public required string Last  { get; init; }
    public string Full => $"{First} {Last}";   // computed members are excluded from equality
}
```

Only fields participate in the generated equality, so computed properties are free.

Two behaviours to know. **Inheritance is included in equality** through a hidden virtual `EqualityContract` property, so a `Manager` is never equal to a `Person` even with identical fields, in either direction. That is almost always what you want and is occasionally surprising. And **`record struct` positional members are mutable by default**: `public record struct Point(int X, int Y)` generates `get; set;`, not `get; init;`. Write `readonly record struct` to get the immutable version, which is what you almost always meant (§2.10).

> **The tell — record or class:** use a `record` when the type is defined by its data and two instances with the same data are interchangeable: DTOs, domain values, events, messages, configuration. Use a `class` when the type has identity, behaviour, or a lifecycle: entities, services, anything registered in a container. If you are about to write `Equals` by hand, you wanted a record. If the value equality would be wrong, you did not.

## 3.6 Pattern matching

Patterns are the part of C# that has changed most, and modern codebases lean on them heavily. They appear after `is`, in `switch` statements, and in `switch` expressions.

```csharp
// declaration and type patterns
if (shape is Circle c) { Use(c.Radius); }

// property patterns, including nested and extended paths
if (order is { Status: OrderStatus.Open, Customer.Country: "GB" }) { ... }

// relational and logical patterns
var band = temp switch
{
    < 0            => "freezing",
    >= 0 and < 15  => "cold",
    >= 15 and < 25 => "mild",
    _              => "warm"
};

// list patterns
var description = args switch
{
    []                    => "no arguments",
    [var only]            => $"one: {only}",
    ["--help", ..]        => "help requested",
    [_, .. var middle, _] => $"{middle.Length} in the middle"
};

// positional patterns, via Deconstruct
if (point is (0, 0)) { ... }
```

The `switch` **expression** returns a value and requires every arm to produce one. It is the form to reach for; the `switch` statement remains for side effects.

Exhaustiveness is partial. The compiler warns (CS8509) when it cannot prove every input is handled, and if an unhandled value arrives at run time the expression throws `SwitchExpressionException`. But it can rarely prove exhaustiveness for anything interesting: not for enums (§2.8, any integer casts in), and not for class hierarchies (a new subclass can appear in another assembly). **In practice you write the `_` arm and make it throw.**

```csharp
_ => throw new UnreachableException($"unhandled status {status}")
```

`UnreachableException` (.NET 7) exists precisely for this and is more informative than `InvalidOperationException`.

## 3.7 Closed hierarchies

Since exhaustiveness cannot be enforced by the compiler, the way to close a hierarchy today is to make it impossible to extend from outside: an abstract base with a private constructor and nested sealed implementations, or an abstract record with sealed records and no public constructor.

```csharp
public abstract record Result
{
    private Result() { }                                // no external subclass can call this
    public sealed record Ok(int Value)      : Result;
    public sealed record Failed(string Why) : Result;
}
```

The compiler still will not give you exhaustiveness warnings, but the set really is closed and the `_` arm really is unreachable.

**This is the gap C# 15 closes.** Union types and closed hierarchies are headline features of the .NET 11 release candidate, and when they ship the pattern above becomes a declaration rather than a construction. Do not build on them yet: release candidates move, and the syntax has changed more than once during preview.

## 3.8 Delegates, lambdas and closures

A delegate type is a named signature; a delegate value is a callable reference to a method, with or without a receiver. The generic ones cover almost everything: `Action`, `Action<T>` for no return value, `Func<T>`, `Func<TArg, TResult>` for a return value, `Predicate<T>` for the legacy `bool` case.

```csharp
Func<int, int> square = x => x * x;
var byLength = list.OrderBy(s => s.Length);        // lambda as an argument
Action log = Console.WriteLine;                    // method group conversion
```

Lambdas have a natural type since C# 10, so `var f = (int x) => x * 2;` compiles to a `Func<int, int>`, and since C# 12 they may declare default parameter values.

A lambda **captures variables, not values**. The compiler creates a class holding the captured variables and the lambda becomes a method on it, which means the lambda sees later changes to the variable, and the variable outlives the scope it was declared in:

```csharp
var fs = new List<Func<int>>();
for (int i = 0; i < 3; i++) fs.Add(() => i);      // all three return 3: one shared i
foreach (var x in xs)       fs.Add(() => x);      // correct: foreach declares x per iteration
```

The `for` version is still a live trap; the `foreach` version stopped being one in C# 5, when the iteration variable became per-iteration.

Capture also has a cost: a closure is a heap allocation, and a lambda that captures nothing is cached as a singleton while one that captures anything is not. `static` lambdas (C# 9) make the compiler enforce non-capture:

```csharp
items.Where(static x => x.IsActive);              // provably allocation-free
```

№80 §1.3 covers what happens when a delegate is handed to the thread pool as a unit of work.

## 3.9 Events

An `event` is a delegate field with restricted access: outside the declaring type, only `+=` and `-=` are permitted, so no subscriber can overwrite the invocation list or invoke it.

```csharp
public event EventHandler<OrderPlaced>? Placed;

protected void OnPlaced(Order o) => Placed?.Invoke(this, new OrderPlaced(o));
```

The `?.Invoke` is not optional style: an event with no subscribers is `null`, and the null-conditional call is the standard way to raise one safely.

The failure mode is a lifetime one. **A subscription is a strong reference from the publisher to the subscriber.** A long-lived publisher holding handlers for short-lived subscribers keeps every one of them alive, which is the classic managed memory leak. Unsubscribe in `Dispose` (§6.1), or prefer a mechanism with clearer ownership: an injected interface, a `Channel<T>` (№80 §6.8), or a mediator.

## 3.10 Extension methods and extension members

An extension method is a static method whose first parameter is marked `this`, callable as if it were an instance method on that type. Every LINQ operator is one.

```csharp
public static class StringExtensions
{
    public static bool IsBlank(this string? s) => string.IsNullOrWhiteSpace(s);
}
"  ".IsBlank();   // true
```

They bind statically and do not participate in virtual dispatch, so an extension method is never overridden and is always beaten by a real instance method of the same name. That is a feature: adding an instance method to a type silently and safely takes precedence over an extension.

C# 14 generalises this beyond methods. An `extension` block declares extension **properties**, **static members** and **operators**:

```csharp
public static class Enumerable_
{
    extension<T>(IEnumerable<T> source)
    {
        public bool IsEmpty => !source.Any();               // extension property
        public static IEnumerable<T> Empty => [];           // extension static member
    }
}
```

This closes a long-standing gap where LINQ-style fluency stopped at methods. The old `this`-parameter syntax remains valid and remains what you will see in existing code.

## 3.11 Operators, conversions and indexers

Operators are static methods with special names, and both operands' types are considered for overload resolution:

```csharp
public readonly record struct Money(decimal Amount, string Currency)
{
    public static Money operator +(Money a, Money b) =>
        a.Currency == b.Currency
            ? a with { Amount = a.Amount + b.Amount }
            : throw new InvalidOperationException("currency mismatch");

    public static implicit operator decimal(Money m) => m.Amount;   // usually a mistake
    public static explicit operator Money(decimal d) => new(d, "GBP");
}
```

**Implicit conversions are worth resisting.** They fire without any syntax at the call site, which means they fire in overload resolution too, and a type with implicit conversions in both directions produces ambiguity errors that are very hard to read. Make conversions explicit, or name them: `ToDecimal()`.

C# 14 adds user-defined compound assignment operators, so `operator +=` can be implemented directly rather than being synthesised from `operator +`. The reason is mutation in place for large value types: `+=` can now modify the target rather than construct and copy a new one.

Indexers are properties taking arguments, and `System.Index`/`System.Range` give them the from-the-end and slice syntax:

```csharp
var last  = items[^1];      // Index: one from the end
var slice = items[1..^1];   // Range: all but the first and last
```

A type gets range support automatically if it has a `Length` or `Count` property plus a `Slice(int, int)` method, which is why spans, arrays and strings all support it without anyone writing an overload.

## 3.12 The rest of the syntax you will meet

**Top-level statements** (C# 9): one file per project may omit the class and `Main`; `args` is in scope, `await` works, and the generated `Program` class is `internal`, which is why integration tests using `WebApplicationFactory<Program>` need `InternalsVisibleTo` or a `public partial class Program { }` line at the bottom of the file.

**File-scoped namespaces** (C# 10): `namespace Orders.Api;` at the top, no braces, no indentation for the whole file. Use them.

**Global usings** (C# 10): `global using System.Text.Json;` in any file applies to the whole project. `<ImplicitUsings>enable</ImplicitUsings>` supplies a sensible default set (§1.4). Put your own in a single `GlobalUsings.cs`.

**Local functions**: a named method declared inside a method body. `static` local functions cannot capture, which makes them free, and they are the right home for a helper used once. Prefer them to private methods that exist only to be called from one place, and to lambdas assigned to variables.

**Expression-bodied members**: `=>` for anything with a single expression, including constructors, properties, and operators.

**Deconstruction**: any type with a `Deconstruct` method, including every record and tuple, supports `var (a, b) = thing;`.

**`nameof`**: compile-time name of a symbol, used constantly in argument exceptions and logging. C# 14 allows unbound generics: `nameof(List<>)` is `"List"`.

**Raw string literals** (C# 11): three or more quotes, no escaping, leading whitespace stripped to the closing delimiter's indentation. The right way to embed JSON, SQL or XML in a test:

```csharp
var json = """
    { "id": 7, "name": "Ada \ not an escape" }
    """;
```

Add `$` for interpolation, and use more than one `$` to change the interpolation delimiter to that many braces, which is how you embed JSON containing braces.

**UTF-8 literals** (C# 11): `"OK"u8` is a `ReadOnlySpan<byte>` with no conversion at run time, for writing constants straight to a network or file stream.

**`partial`**: splits a declaration across files. Its real use is source generators (§9.5): you declare `partial` and the generator supplies the other half. C# 13 added partial properties and C# 14 partial constructors and events, both driven by generator scenarios.

---

# Part 4 — Nullability

## 4.1 What nullable reference types actually are

Reference types have always been nullable. What C# 8 added was not a runtime change but a **static analysis**: with the feature on, the compiler tracks, for every expression, whether it believes the value can be null, and warns when a possibly-null value is used where a non-null one is required.

```csharp
string  name;    // the compiler will warn if this is ever null
string? maybe;   // the compiler will warn if you dereference this without checking
```

Three things follow, and misunderstanding any of them causes the usual disappointment.

**It is erased.** The `?` produces no runtime check, no attribute you can test, no exception. It is recorded in metadata as `[Nullable]` and `[NullableContext]` so that other assemblies compiled against yours see your intent, and that is the whole of its runtime presence. A `string` parameter can receive `null` at run time and nothing will stop it.

**It produces warnings, not errors.** By design, so that the feature can be adopted incrementally. A codebase that does not treat them as errors gets no benefit from them at all after the first few weeks.

**It is unrelated to `int?`.** `Nullable<int>` is a struct with runtime behaviour (§2.7); `string?` is an annotation. The shared syntax is the single most misleading thing in the language.

## 4.2 Flow analysis

The compiler's model is a per-expression state, "not null" or "maybe null", updated by everything it can reason about:

```csharp
void Handle(Order? order)
{
    if (order is null) return;
    Use(order.Id);                    // fine: the early return narrowed the state

    var note = order.Note;            // string?
    if (!string.IsNullOrEmpty(note))
        Console.WriteLine(note.Length); // fine: IsNullOrEmpty is annotated (§4.5)

    order.Note ??= "none";            // assignment updates the state to not-null
}
```

The analysis is intraprocedural and resets at every method call it cannot see through, which is what the attributes in §4.5 exist to fix.

## 4.3 The null-forgiving operator

`x!` tells the compiler to treat `x` as not-null. **It checks nothing and generates nothing.** It is an assertion by you, and if you are wrong the `NullReferenceException` arrives exactly where it would have anyway, minus the warning that would have told you.

There are legitimate uses: an invariant the compiler cannot see, a test asserting a value is present, a framework that guarantees initialisation. There is one illegitimate use, and it is common: silencing a warning to get a build green. **A `!` that you cannot justify in a sentence is a bug you have annotated rather than fixed.** Treat a rising count of `!` in a codebase as the signal it is.

The one that looks innocent and is not:

```csharp
public string Name { get; set; } = null!;   // "the framework will set this"
```

Sometimes true, for an EF Core entity or a deserialised DTO. Usually it means `required` (§3.2) was the right answer and nobody reached for it.

## 4.4 Where nulls get in anyway

The analysis is sound only over code the compiler sees. The holes, in rough order of how often they bite:

**Deserialisation and materialisation.** `JsonSerializer.Deserialize<Order>(json)` will happily produce an `Order` whose non-nullable `string Name` is null, because the JSON omitted it. EF Core will do the same materialising a row. Neither goes through your constructor's validation unless you made it. This is the hole, and the fix is `required` plus a constructor, or validation at the boundary.

**Arrays.** `new string[10]` is ten nulls and produces no warning, because the compiler has no way to require initialisation of array elements. Same for `Array.Resize`, and for any `T[]` in a generic.

**Default structs.** A `struct` containing a non-nullable reference field, created via `default` or as an array element, has a null in that field with no constructor ever running (§2.5).

**Unannotated dependencies.** A package compiled without the feature is *oblivious*: the compiler assumes nothing, warns about nothing, and every reference type coming out of it is treated as neither null nor non-null. Older packages, and anything targeting `netstandard2.0` from before 2019, are in this category.

**Reflection and `Activator.CreateInstance`.** No constructor guarantees survive.

> **The tell — where to put the check:** put runtime null checks at the boundary of your code, where values arrive from outside the compiler's view: deserialised input, database rows, configuration, reflection, and public API parameters. Inside that boundary, trust the analysis and stop writing defensive checks. Doubling up, annotations plus a null check on every parameter of every private method, is the pattern that makes people conclude the feature is worthless.

`ArgumentNullException.ThrowIfNull(x)` (.NET 6) is the boundary check, one line, with the parameter name captured automatically by `[CallerArgumentExpression]`. `ArgumentException.ThrowIfNullOrWhiteSpace`, `ArgumentOutOfRangeException.ThrowIfNegative` and friends (.NET 8) cover the rest.

## 4.5 The attributes

When your method's nullability depends on something the caller cannot see from the signature, annotate it. These are in `System.Diagnostics.CodeAnalysis` and they are how the BCL makes `TryGetValue` and `IsNullOrEmpty` work properly.

| Attribute | Says |
|---|---|
| `[NotNullWhen(true)]` | This `out` or `ref` parameter is not null when the method returns `true` |
| `[MaybeNullWhen(false)]` | It may be null when the method returns `false` |
| `[NotNullIfNotNull(nameof(input))]` | The return is not null if `input` was not null |
| `[MemberNotNull(nameof(_cache))]` | After this method returns, that field is not null |
| `[MemberNotNullWhen(true, nameof(_cache))]` | The same, conditional on the return value |
| `[AllowNull]` / `[DisallowNull]` | Input may be null though the type says otherwise, or the reverse |
| `[DoesNotReturn]` | This method always throws; code after it is unreachable |

```csharp
public bool TryGet(string key, [NotNullWhen(true)] out Order? order) { ... }

if (TryGet(k, out var o)) { Use(o.Id); }   // no warning: the attribute did the work
```

`[MemberNotNull]` is the one that solves the lazy-initialisation complaint, where a field is assigned in an `EnsureLoaded()` method rather than the constructor and the compiler warns at every use.

## 4.6 Turning it on in an existing codebase

Do not switch `<Nullable>enable</Nullable>` on across a large solution and start fixing warnings; you will get thousands, most of them mechanical, and the meaningful ones will be lost in them.

The order that works: set `<Nullable>enable</Nullable>` at the solution level via `Directory.Build.props` (§9.2), then immediately add `#nullable disable` to the top of every existing file with a script. New files get the analysis; old files are untouched. Then remove the pragma one file at a time, starting at the leaves of the dependency graph, because annotating a type improves the warnings in everything that consumes it and doing it in the other order means annotating twice.

Once a project is clean, promote the warnings so they cannot come back:

```xml
<WarningsAsErrors>$(WarningsAsErrors);Nullable</WarningsAsErrors>
```

**Nullable warnings that are not errors are noise that people learn to scroll past.** The feature's whole value is in the enforcement.

---

# Part 5 — Collections and LINQ

## 5.1 The interface hierarchy

Six interfaces carry almost all of it.

```
IEnumerable<T>            foreach; nothing else
  ICollection<T>          + Count, Add, Remove, Contains, IsReadOnly
    IList<T>              + indexer, Insert, RemoveAt
    ISet<T>               + set operations
IReadOnlyCollection<T>    Count, and foreach
  IReadOnlyList<T>        + indexer
IDictionary<K,V> / IReadOnlyDictionary<K,V>
```

The read-only interfaces are **not** a promise of immutability. `List<T>` implements `IReadOnlyList<T>`; handing one out prevents the *recipient* from mutating and prevents nothing else. If a caller must not observe changes, give them a copy or an immutable collection (§5.5).

There is one long-standing wart worth recognising: **arrays implement `IList<T>`** and throw `NotSupportedException` from `Add` and `Insert`. `ICollection<T>.IsReadOnly` exists to let you ask, and almost nobody does.

> **The tell — what to expose:** parameters take the least specific interface that works, usually `IEnumerable<T>`, unless the method needs to enumerate more than once or needs the count, in which case take `IReadOnlyCollection<T>` and make the caller's cost visible. Return types are the opposite: return the most useful concrete or read-only type you can, usually `IReadOnlyList<T>` or the concrete `List<T>`, so callers are not forced to re-materialise. Never return `IEnumerable<T>` from something that has already materialised; you are hiding a fact the caller needs (§5.8).

## 5.2 The concrete collections and what they cost

| Type | Lookup | Insert | Notes |
|---|---|---|---|
| `T[]` | O(1) | fixed | Contiguous, no overhead beyond the header. Covariant, and pays a store check for it (§2.16) |
| `List<T>` | O(1) by index, O(n) by value | amortised O(1) at the end, O(n) elsewhere | Array-backed, capacity doubles from 4 |
| `Dictionary<K,V>` | O(1) average | O(1) average | Buckets plus an entries array; iteration order is unspecified |
| `HashSet<T>` | O(1) average | O(1) average | Same machinery, no values |
| `SortedDictionary<K,V>` | O(log n) | O(log n) | Red-black tree. Ordered iteration |
| `SortedList<K,V>` | O(log n) | O(n) | Two arrays. Cheaper memory and faster iteration than `SortedDictionary`, much worse insertion |
| `Queue<T>` / `Stack<T>` | n/a | O(1) | Circular buffer / array |
| `PriorityQueue<E,P>` | O(1) peek | O(log n) | .NET 6. A heap, so **not stable**: equal priorities come out in unspecified order |
| `LinkedList<T>` | O(n) | O(1) given a node | Almost never the right answer; the pointer chasing loses to `List<T>` at every realistic size |

Two practical notes. `List<T>` doubles its backing array when full, so building a large list without setting `Capacity` performs a series of copies totalling roughly twice the final size in allocations. And `Dictionary<K,V>` iteration order is **unspecified and unstable across versions**, so any code depending on it is a latent bug; sort explicitly if order matters.

## 5.3 The hashing contract

A hash-based collection is correct only if the key's hash is stable and consistent with its equality (§2.6). The failure mode is silent:

```csharp
var set = new HashSet<Order>();
set.Add(order);
order.Id = 99;          // Id participates in GetHashCode
set.Contains(order);    // false. The entry is in the wrong bucket, permanently
```

**Use immutable types as keys**, which is what `readonly record struct` and `record` are for. If a key must be mutable, take a copy of the key fields when inserting.

Supply the comparison from outside rather than mangling keys. A case-insensitive dictionary is a comparer, not a `ToLower()`:

```csharp
new Dictionary<string, Order>(StringComparer.OrdinalIgnoreCase)   // right
dict[key.ToLower()] = order;                                       // wrong: allocates, and
                                                                   // ToLower is culture-sensitive (§8.2)
```

An implementation detail that occasionally surfaces in profiling: `Dictionary<string, V>` starts with a fast non-randomised ordinal hash and **switches to a randomised hash for that instance** if one bucket accumulates too many collisions. That is a deliberate defence against hash-flooding denial of service on dictionaries keyed by user input, and it is why an adversarially-built dictionary gets slower and then abruptly recovers.

## 5.4 Immutable, read-only and frozen

Four different things, easily confused:

| | What it is | Use when |
|---|---|---|
| `IReadOnlyList<T>` | An interface. A view, no guarantee | Expressing intent in a signature |
| `.AsReadOnly()` / `ReadOnlyCollection<T>` | A wrapper over a live collection. Changes to the underlying show through | Handing out a defensive view you still own |
| `ImmutableArray<T>`, `ImmutableList<T>` | Genuinely immutable. "Mutation" returns a new instance | Shared state, especially across threads (№80 §6.7) |
| `FrozenDictionary<K,V>`, `FrozenSet<T>` | Immutable, expensive to build, optimised hard for reads (.NET 8) | Built once at startup, read for the process lifetime |

`ImmutableArray<T>` is a struct wrapping an array: reads are as fast as an array, and any change copies the whole thing, so it suits collections that are read constantly and replaced rarely. `ImmutableList<T>` is a balanced tree: O(log n) for everything, sharing structure between versions, so it suits collections that are genuinely edited. Choosing `ImmutableList<T>` because the name sounds like `List<T>` is the common mistake and costs a large constant factor on every read.

## 5.5 Iterators

`yield return` turns a method into a lazily-evaluated sequence. The compiler rewrites the method into a state machine implementing `IEnumerator<T>`, resuming from where it left off at each `MoveNext`.

```csharp
public static IEnumerable<int> Naturals()
{
    int i = 0;
    while (true) yield return i++;   // infinite, and fine, because nothing runs until asked
}
```

The trap is that **the body does not run until the first `MoveNext`**, which means argument validation in an iterator method fires at enumeration rather than at the call:

```csharp
// BROKEN: the exception surfaces at the foreach, not at the call
public static IEnumerable<T> Take<T>(IEnumerable<T> src, int n)
{
    if (n < 0) throw new ArgumentOutOfRangeException(nameof(n));
    foreach (var x in src) { if (n-- == 0) yield break; yield return x; }
}

// FIXED: validate eagerly in a normal method, delegate to a private iterator
public static IEnumerable<T> Take<T>(IEnumerable<T> src, int n)
{
    ArgumentOutOfRangeException.ThrowIfNegative(n);
    return Iterate(src, n);

    static IEnumerable<T> Iterate(IEnumerable<T> src, int n) { ... }
}
```

Every LINQ operator in the BCL is written this way, which is why `Where` with a null predicate throws immediately and not on enumeration.

`IAsyncEnumerable<T>` and `await foreach` are the asynchronous counterpart and belong to №80 §3.8.

## 5.6 LINQ: the model

LINQ is a set of extension methods on `IEnumerable<T>` (in `System.Linq.Enumerable`) that take and return sequences. Query syntax is sugar over exactly those calls:

```csharp
var q = from o in orders where o.Total > 100 orderby o.Date select o.Id;
var m = orders.Where(o => o.Total > 100).OrderBy(o => o.Date).Select(o => o.Id);  // identical
```

Query syntax has an advantage only for `join` and for multiple `from` clauses, where the method form gets ugly. Everything else reads better as methods; pick one per codebase and enforce it.

**Almost every operator is deferred.** `Where`, `Select`, `OrderBy`, `GroupBy`, `Take` and the rest return an object that has done nothing; the work happens when something enumerates it. The operators that force enumeration are the ones that must: `ToList`, `ToArray`, `ToDictionary`, `Count`, `Sum`, `Any`, `First`, `Single`, `Aggregate`, and `foreach` itself.

That laziness is the source of both the elegance and every trap in §5.8.

## 5.7 The operators worth knowing

| Category | Operators |
|---|---|
| Filter | `Where`, `OfType<T>`, `Distinct`, `DistinctBy`, `Take`, `Skip`, `TakeWhile`, `SkipWhile`, `TakeLast`, `SkipLast` |
| Project | `Select`, `SelectMany`, `Cast<T>`, `Chunk`, `Index` |
| Order | `OrderBy`, `ThenBy`, `OrderByDescending`, `Order`, `OrderDescending`, `Reverse` |
| Group and join | `GroupBy`, `Join`, `GroupJoin`, `ToLookup` |
| Aggregate | `Count`, `Sum`, `Min`, `Max`, `Average`, `Aggregate`, `MinBy`, `MaxBy`, `CountBy`, `AggregateBy` |
| Quantify | `Any`, `All`, `Contains`, `SequenceEqual` |
| Element | `First`, `FirstOrDefault`, `Single`, `SingleOrDefault`, `Last`, `ElementAt` |
| Set | `Union`, `Intersect`, `Except`, and the `...By` variants |
| Materialise | `ToList`, `ToArray`, `ToDictionary`, `ToHashSet`, `ToLookup` |
| Generate | `Enumerable.Range`, `Repeat`, `Empty`, `DefaultIfEmpty`, `Zip`, `Concat`, `Append`, `Prepend` |

Four that pay for themselves and are under-used. **`ToLookup`** builds a one-to-many dictionary in one pass and is the right answer to "group these and then look them up repeatedly". **`SelectMany`** flattens, and is what `from x in a from y in b` compiles to. **`Aggregate`** is the general fold when nothing more specific fits. **`Chunk(n)`** batches a sequence into arrays of `n`, which is what everybody used to hand-roll for bulk inserts and paged API calls.

The `...By` family (`MinBy`, `DistinctBy`, `MaxBy`, `ExceptBy`) takes a key selector and returns *elements*, not keys, which is what `OrderBy(...).First()` and `GroupBy(...).Select(g => g.First())` were being used for.

## 5.8 The traps

**Multiple enumeration.** An `IEnumerable<T>` that is a query, not a collection, re-runs the entire pipeline each time it is enumerated. Twice for the count and the loop, twice for two `foreach`es, and against a database that means two round trips.

```csharp
// BROKEN: two full enumerations, and Any() may re-query
IEnumerable<Order> q = repo.Query().Where(o => o.Open);
if (q.Any()) foreach (var o in q) { ... }

// FIXED
List<Order> open = repo.Query().Where(o => o.Open).ToList();
if (open.Count > 0) foreach (var o in open) { ... }
```

This is why returning `IEnumerable<T>` from a repository method is a poor API: the caller cannot tell whether it is free to enumerate twice.

**`Count()` versus `Any()`.** `Enumerable.Count()` has a fast path when the source implements `ICollection<T>`, so it is O(1) on a `List<T>`, but O(n) with full enumeration on a query. `Any()` stops at the first element. **Use `Any()` for existence, always**, and `.Count` (the property) rather than `.Count()` when you have a real collection.

**`Single` costs more than `First`.** `Single` must enumerate a second element to prove there is not one. On a database query that is a different plan. Use `Single` when uniqueness is an invariant you want checked, `First` when you just want one.

**Side effects in `Select`.** Deferred execution means the side effect happens at an unpredictable time, or twice, or never. Use `foreach` for effects and LINQ for values.

**`OrderBy` is stable; `List.Sort` is not.** `Enumerable.OrderBy` preserves the relative order of equal elements. `List<T>.Sort` and `Array.Sort` use introsort and do not. Code that relies on ties keeping their input order breaks when someone "optimises" one into the other.

**`GroupBy` buffers.** It cannot yield the first group until it has seen the last element, so it materialises the whole sequence. That is unavoidable in memory, and is why `GroupBy` against a large table should be pushed into the database.

## 5.9 `IQueryable` and expression trees

`IQueryable<T>` looks like `IEnumerable<T>` with the same operator names, and does something fundamentally different.

The operators on `IEnumerable<T>` take **delegates**: `Where(Func<T, bool>)`, a compiled method the runtime can call. The operators on `IQueryable<T>` take **expression trees**: `Where(Expression<Func<T, bool>>)`. The same lambda source text compiles to one or the other depending on the target type, and an `Expression<Func<T, bool>>` is not code, it is a **data structure describing the code**: a tree of nodes saying "binary comparison, greater-than, left is a member access on parameter `o` named `Total`, right is a constant 100".

```csharp
Func<Order, bool>             f = o => o.Total > 100;   // compiled to IL, callable
Expression<Func<Order, bool>> e = o => o.Total > 100;   // a tree, inspectable
e.Body;      // a BinaryExpression
f(order);    // calls it
e.Compile()(order);  // builds a delegate at run time, then calls it
```

A LINQ provider, EF Core being the one you will meet, walks that tree and emits SQL. That is the entire mechanism, and it explains the two things about it that confuse people.

**Why some perfectly valid C# fails.** `Where(o => MyHelper(o))` compiles fine and then throws at run time, because the provider reaches a node it cannot translate to SQL. EF Core 3.0 onwards throws rather than silently fetching the table and filtering in memory, which was the older and far worse behaviour.

**Where the boundary is.** `AsEnumerable()`, `ToList()`, `ToArray()` and `AsAsyncEnumerable()` switch from the provider to in-memory LINQ. Everything before the switch becomes SQL; everything after runs on the client over whatever came back. Putting the switch one line too early turns a `WHERE` clause into a full table scan, and the code looks identical.

> **The tell — `IQueryable` or `IEnumerable`:** keep `IQueryable<T>` inside the data-access layer and never let it escape. A repository or query method returns a materialised `List<T>` or `IReadOnlyList<T>`, having decided for itself what runs in the database. The moment `IQueryable<T>` crosses into application code, callers can and will attach clauses that decide your query plan from a place that knows nothing about your schema, and the difference between an index seek and a table scan becomes a line in a controller.

Expression trees have a second life outside LINQ: `expr.Compile()` builds a delegate at run time, which is how many mapping, validation and mocking libraries work. It uses `Reflection.Emit`, so it does not survive Native AOT (§1.7), which is why those libraries are steadily moving to source generators (§9.5).

## 5.10 What arrived when

Useful when reading a codebase that pins an older TFM, and when checking whether a method you remember is actually available:

| Version | Added |
|---|---|
| .NET 6 | `Chunk`, `MinBy`/`MaxBy`, `DistinctBy`/`ExceptBy`/`IntersectBy`/`UnionBy`, `TakeLast`/`SkipLast`, `TryGetNonEnumeratedCount`, `ElementAt(Index)` |
| .NET 7 | `Order`, `OrderDescending` (sort by the elements themselves, no key selector) |
| .NET 9 | `CountBy`, `AggregateBy`, `Index` |
| .NET 10 | `LeftJoin` and `RightJoin` on `IQueryable` **in EF Core 10**, not in `System.Linq` |
| .NET 11 (RC) | `Enumerable.LeftJoin` and `Enumerable.RightJoin` for in-memory sequences |

That last row is a distinction worth getting right, because the blog posts blur it: **the in-memory `LeftJoin` is a .NET 11 addition; what shipped in the .NET 10 timeframe was EF Core's queryable version.** On .NET 10 in memory you still write the `GroupJoin` plus `SelectMany` plus `DefaultIfEmpty` dance.

---

# Part 6 — Resources and lifetime

## 6.1 `IDisposable` and `using`

The garbage collector reclaims memory. It knows nothing about file handles, sockets, database connections, locks or native buffers, and it reclaims memory at a time nobody chose. `IDisposable` is the language's answer: a convention that a type holds something that must be released promptly, and one method to release it.

```csharp
using (var conn = new SqlConnection(cs)) { ... }   // statement form
using var conn2 = new SqlConnection(cs);            // declaration form (C# 8), disposes at end of scope
```

Both compile to `try { ... } finally { x?.Dispose(); }`. The declaration form removes a level of nesting and is the default in new code, at the cost of making the disposal point implicit; when the lifetime genuinely matters, use the block form so the reader can see it.

**Disposing does not free memory.** It releases whatever the object holds and marks it unusable; the object itself is collected later, like everything else. That distinction confuses people arriving from languages where the two are the same act.

`IAsyncDisposable` and `await using` exist for resources whose release is itself I/O, and are covered in №80 §3.9.

## 6.2 The dispose pattern, and when you need it

The full ceremonial pattern is:

```csharp
public class Handle : IDisposable
{
    private bool _disposed;

    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;
        if (disposing) { /* release other IDisposables you own */ }
        /* release unmanaged resources */
        _disposed = true;
    }
}
```

You need all of that only when the class **directly holds an unmanaged resource** (a raw handle or pointer) or is **unsealed and expects subclasses to add resources**. Neither is common in application code.

What application code almost always needs is the short version:

```csharp
public sealed class Repo(SqlConnection conn) : IDisposable
{
    public void Dispose() => conn.Dispose();
}
```

Sealed, no finalizer, no `disposing` flag, no `GC.SuppressFinalize`. **Sealing the class is what removes the ceremony**, which is one more reason to seal by default.

Two rules the convention imposes on you: **`Dispose` must be idempotent**, safe to call any number of times, and **using a disposed object should throw `ObjectDisposedException`**, not behave unpredictably. `ObjectDisposedException.ThrowIf(_disposed, this)` (.NET 7) is the one-liner for the second.

## 6.3 Finalizers, and why not to write one

A finalizer (`~Handle()`) is a method the garbage collector calls before reclaiming an object, as a last resort for objects nobody disposed. It is worth understanding exactly so you can avoid it.

The costs are severe. A finalizable object cannot be collected on the GC that finds it unreachable; it is queued instead, promoted to the next generation, and collected only on a later GC after the finalizer thread has run it. So finalization **at least doubles the object's lifetime and costs two collections**. Finalizers run on a single dedicated thread with no ordering guarantees, may run at process exit or not at all, and an unhandled exception in one takes down the process.

The right answer for a native handle is `SafeHandle`: an abstract BCL type that wraps a handle, reference-counts it so it cannot be released while a P/Invoke is using it, and releases it in a critical finalizer that the runtime guarantees will run. Every native resource you would write a finalizer for should be a `SafeHandle` subclass instead, at which point your own class needs no finalizer at all.

> **The tell — do I need a finalizer:** if you are not writing a `SafeHandle` subclass, no. If you think you are the exception, you are wrapping a native handle directly, and the fix is still a `SafeHandle`. `GC.SuppressFinalize` in a class with no finalizer is cargo cult; it does nothing and signals that the pattern was pasted rather than chosen.

## 6.4 `Lazy<T>` and static initialisation

`Lazy<T>` defers construction to first access and is **thread-safe by default** (`ExecutionAndPublication`: exactly one thread runs the factory, the others wait). It also caches exceptions by default, so a factory that throws will throw the same exception forever after; `LazyThreadSafetyMode.PublicationOnly` gets you retry behaviour at the cost of possibly running the factory more than once.

Static field initialisers and static constructors give you the same thing at the type level. The runtime guarantees a static constructor runs **exactly once, before any static member of the type is accessed, and is safe against concurrent access**. That guarantee is genuinely strong and is why a `private static readonly` field is the simplest correct singleton in C#:

```csharp
private static readonly HttpClient Shared = new();
```

One caveat that catches people: a type **without** an explicit static constructor is marked `beforefieldinit`, which permits the runtime to run its field initialisers earlier than first access, at any point before it. With an explicit static constructor, initialisation is precisely timed and slightly slower. Do not write an empty static constructor to force the strict timing unless you know why you need it.

The genuine hazard is blocking inside a static constructor. The runtime holds a lock for the duration, so a static constructor that waits on another thread, or blocks on async work, can deadlock the type initialisation for the whole process. №80 §3.4 covers the shape of that failure.

## 6.5 Disposal as a dependency-injection contract

In a container, **who disposes is decided by who constructed**. The Microsoft container disposes any `IDisposable` it created, at the end of the scope that created it: transients and scoped services at the end of the request scope, singletons at application shutdown. It does **not** dispose instances you constructed yourself and registered with `AddSingleton(instance)`, because it did not create them and does not know whether you still want it.

Two consequences.

**Do not `using` something you got from the container.** You will dispose an object the container will hand to somebody else, and the failure arrives later as an `ObjectDisposedException` in unrelated code.

**A transient `IDisposable` resolved from the root provider is never released until shutdown.** The root scope lives for the process lifetime, so transient disposables accumulate in it. This is the most common "memory leak" in an ASP.NET Core application, and the reason `IServiceScopeFactory` exists: create a scope, resolve inside it, dispose the scope.

The related bug is the **captive dependency**: a scoped service injected into a singleton is captured for the singleton's lifetime, so one request's `DbContext` becomes every request's `DbContext`. The container detects this at startup when validation is on, which it is by default in Development and should be everywhere:

```csharp
builder.Host.UseDefaultServiceProvider(o => { o.ValidateScopes = true; o.ValidateOnBuild = true; });
```

№14 covers container lifetimes as a design topic; №80 §9.1 covers the same registrations read as a thread-safety contract.

---

# Part 7 — Errors

## 7.1 The hierarchy, and the ones you actually use

Everything derives from `System.Exception`. The `SystemException`/`ApplicationException` split below it was an early design mistake and carries no meaning; do not derive from `ApplicationException`, and do not catch `SystemException`.

The ones worth recognising on sight:

| Exception | Means |
|---|---|
| `ArgumentException`, `ArgumentNullException`, `ArgumentOutOfRangeException` | The caller passed something invalid. Yours to throw, at your public boundary |
| `InvalidOperationException` | The object is in the wrong state for this call. The default choice when nothing more specific fits |
| `NotSupportedException` | This operation is never valid on this type. Thrown by `Array`'s `IList<T>.Add` (§5.1) |
| `KeyNotFoundException` | Dictionary indexer miss |
| `ObjectDisposedException` | Use after `Dispose` (§6.2) |
| `FormatException`, `OverflowException` | Parsing and arithmetic |
| `NullReferenceException`, `IndexOutOfRangeException` | Bugs. Never throw these deliberately, and never catch them |
| `OperationCanceledException` | Cooperative cancellation. See №80 §4 |
| `TimeoutException`, `IOException`, `HttpRequestException` | The outside world |

## 7.2 No checked exceptions

C# has no equivalent of a `throws` clause. A method signature tells you nothing about what it can fail with, and the compiler will never make you handle anything.

This was a deliberate choice, made after watching the alternative, and it has two consequences you have to design around.

**Documentation is the only contract.** `/// <exception cref="...">` in XML docs is the closest thing to a signature, and it is not enforced. Assume callers of your library will discover your exceptions in production.

**Expected failure should not be an exception at all.** Since nothing forces a caller to handle anything, an exception used for a routine outcome, a validation failure, a cache miss, a not-found, will be missed by somebody, and will be expensive when it fires (§7.7). Model expected outcomes in the return type and reserve exceptions for the genuinely exceptional:

```csharp
bool TryParse(string s, out int value);              // expected failure, in the signature
Order? FindOrder(int id);                            // absence is normal, so it is nullable
Order  GetOrder(int id);                             // absence is a bug, so this throws
```

That trio, `TryXxx` for parsing, nullable for lookup, throwing for invariants, is the BCL's own convention and is worth copying exactly, because it makes the intent readable from the name alone.

## 7.3 Filters

`catch (X e) when (condition)` (C# 6) is not sugar for `catch { if (!c) throw; }`, and the difference is mechanical. .NET exception handling is **two-pass**: the first pass walks up the stack asking each handler whether it will take the exception, and only when one says yes does the second pass unwind. **A filter runs during the first pass, while the stack is still intact.**

That gives you two things. A filter that returns `false` leaves the exception propagating with its original stack completely undisturbed, unlike catch-and-rethrow. And a filter is the correct place to log an exception you do not intend to handle, because it runs before any unwinding:

```csharp
catch (SqlException e) when (e.Number is 1205 or 1222) { await RetryAsync(); }  // deadlock, lock timeout

catch (Exception e) when (Log(e)) { }   // Log returns false: logs with the full stack, never catches
```

The second idiom is a genuine technique, not a trick, and is how several logging libraries capture first-chance detail.

## 7.4 `throw` versus `throw ex`

```csharp
catch (Exception e) { Cleanup(); throw; }     // FIXED:  original stack trace preserved
catch (Exception e) { Cleanup(); throw e; }   // BROKEN: stack trace reset to this line
```

`throw e` overwrites `StackTrace` with the current location, so everything above the catch is lost, and the resulting production trace points at the middle of your error handling rather than at the failure. It is the single most common way to destroy diagnosability.

When you must rethrow from somewhere else, a different thread, a continuation, after storing it, `ExceptionDispatchInfo` preserves the original:

```csharp
var captured = ExceptionDispatchInfo.Capture(e);
// ...later, elsewhere...
captured.Throw();   // original stack trace intact, with the new frame appended
```

That is what `await` uses internally to give you a sensible stack when a `Task` faults (№80 §2.5).

## 7.5 Custom exceptions

Derive from `Exception`, provide the conventional constructors, add whatever structured data callers need, and name it `...Exception`:

```csharp
public sealed class OrderNotFoundException(int orderId)
    : Exception($"Order {orderId} was not found")
{
    public int OrderId { get; } = orderId;
}
```

Two notes on things you will see in older code. `[Serializable]` plus the `(SerializationInfo, StreamingContext)` constructor was required when exceptions crossed AppDomain boundaries by binary serialisation; **binary serialisation is obsolete and disabled by default since .NET 8**, and that constructor should be removed rather than copied forward. And a custom exception per failure mode is over-engineering unless callers will actually branch on the type; if nobody catches it specifically, `InvalidOperationException` with a good message is better.

## 7.6 The `TryXxx` pattern

```csharp
public static bool TryParseOrderId(string? s, [NotNullWhen(true)] out OrderId? id)
```

Return `bool`, produce the value through `out`, never throw for the failure case, and annotate with `[NotNullWhen(true)]` so the nullable analysis flows through the `if` (§4.5). Pair it with a throwing `Parse` for callers who consider failure impossible. Every parsing type in the BCL follows this and so should yours.

## 7.7 What exceptions cost

Throwing is expensive: the runtime captures a stack trace, then performs the two-pass walk, and the cost is on the order of microseconds, thousands of times a plain return. Deep stacks cost more. The cost is paid at the *throw*, not at the `try`, so `try` blocks in code that does not throw are effectively free and there is no reason to avoid them.

The practical rule: **an exception per request is invisible; an exception per item in a loop over ten thousand items is a performance incident.** Validation loops, parsing bulk input, and cache lookups are where this bites, and they are exactly the places the `TryXxx` shape belongs.

`finally` blocks run on both the normal and the exceptional path, and are skipped only when the process is not really running any more: `StackOverflowException` (which cannot be caught at all, by design, because the runtime cannot guarantee stack to run a handler), `Environment.FailFast`, and a process kill.

## 7.8 Result types, and what is coming

C# has no built-in result or either type, so codebases that want failure-as-a-value reach for a library (`OneOf`, `ErrorOr`, `FluentResults`, `LanguageExt`) or hand-roll a closed hierarchy (§3.7). All of them work; none is a standard, and mixing two in one codebase is worse than picking either.

**Union types and closed hierarchies are headline C# 15 features in the .NET 11 release candidate**, which is what the ecosystem has been approximating. When they ship this section changes materially: the closed-hierarchy construction becomes a declaration and switch expressions over it become genuinely exhaustive. Until general availability in November 2026, treat that as a direction rather than a plan, and check the release notes rather than this document.

> **The tell — exception or return value:** ask whether the caller can reasonably be expected to continue. If the failure is part of the normal operation of the caller, invalid user input, a missing record, a parse failure on untrusted data, it belongs in the return type and the caller must see it in the signature. If the failure means an assumption of your code is false, a required configuration key missing, an invariant broken, a database unreachable, throw, and let it reach the boundary handler. The question is not how bad it is, it is whether the immediate caller has anything sensible to do about it.

---

# Part 8 — Text, time, numbers, serialisation

## 8.1 `string`

A `string` is an immutable sequence of **UTF-16 code units**, and `Length` counts code units, not characters. A character outside the Basic Multilingual Plane, most emoji, some CJK extensions, occupies two of them:

```csharp
"🙂".Length;                  // 2
"🙂".EnumerateRunes().Count(); // 1
```

`Rune` (.NET Core 3.0) is a Unicode scalar value and is the correct unit when you are actually processing text; `StringInfo` goes one level further to grapheme clusters, which is what a user calls a character. For the overwhelming majority of server code, none of this matters and you should not add it defensively. It matters when you truncate: **truncating by `Length` can split a surrogate pair and produce invalid text**, which is the one place to reach for `Rune`.

Immutability means every operation allocates. Literals are interned, so identical literals in one process are the same object, but strings built at run time are not. `string` is a class, so `==` on it is an overloaded operator doing ordinal value comparison, with all the caveats in §2.6.

## 8.2 Comparison, and the biggest trap in the BCL

String comparison in .NET is either **ordinal** (compare the code units, byte for byte) or **culture-sensitive** (compare according to a language's collation rules, so that "æ" might sort with "ae" and "co-op" might compare equal to "coop"). Both are correct for different jobs and catastrophic for the other one.

The defaults are inconsistent, and this is the thing to memorise:

| Call | Default |
|---|---|
| `a == b`, `string.Equals(a, b)` | **Ordinal** |
| `a.Contains(string)`, `a.IndexOf(char)`, `a.Replace` | **Ordinal** |
| `a.StartsWith(string)`, `a.EndsWith(string)`, `a.IndexOf(string)` | **Current culture** |
| `a.CompareTo(b)`, `string.Compare(a, b)`, `Array.Sort`, `OrderBy` on strings | **Current culture** |
| `a.ToUpper()`, `a.ToLower()` | **Current culture** |

So `s == "GET"` is ordinal and `s.StartsWith("GET")` is not, in the same method, with no visible difference. On a developer machine set to `en-GB` the culture-sensitive versions behave close enough to ordinal that nothing is noticed; on a server with a different culture, or with ICU behaving differently across versions, the behaviour changes.

The Turkish case is the canonical demonstration: in `tr-TR`, `"i".ToUpper()` is `"İ"`, not `"I"`, so a case-insensitive comparison of `"file".ToUpper() == "FILE"` fails on a Turkish server. This has broken real production systems.

```csharp
// BROKEN: culture-sensitive, and allocates twice
if (header.ToLower() == "content-type") { ... }

// FIXED: ordinal, allocation-free, and says what it means
if (header.Equals("content-type", StringComparison.OrdinalIgnoreCase)) { ... }
```

> **The tell — which comparison:** if a human will read the ordering, use culture-sensitive comparison and say so explicitly (`StringComparison.CurrentCulture`). For everything else, and that means identifiers, keys, file paths, HTTP headers, protocol tokens, enum names, database values, use `StringComparison.Ordinal` or `OrdinalIgnoreCase`. Turn on analysers CA1307 and CA1310, which flag every call that took a default, and make them errors. Never lower-case a string in order to compare it.

Since .NET 5, culture data comes from **ICU on every platform**, including Windows, which changed some collation results relative to the older Windows NLS behaviour. `<InvariantGlobalization>true</InvariantGlobalization>` removes ICU entirely, makes every culture the invariant one, and shrinks container images considerably; it is a reasonable default for a service that only ever does ordinal work, and it will throw at startup if anything asks for a real culture.

## 8.3 Building strings

`+` in a loop is O(n²) in allocations. The alternatives, in order of preference:

```csharp
string.Join(", ", items)                       // best when you have a sequence
$"{first} {last}"                              // best for a fixed shape
new StringBuilder().Append(x).Append(y).ToString()   // best for loops and conditionals
```

Interpolation is not `string.Format` any more. Since C# 10 the compiler lowers `$"..."` to a `DefaultInterpolatedStringHandler`, which writes into a pooled buffer and calls a generic `AppendFormatted<T>` for each hole, so **value types are no longer boxed** and there is no intermediate `object[]`.

The same mechanism is why `StringBuilder.Append($"{x}")` is efficient and why the logging APIs can be lazy. One thing it does **not** change:

```csharp
// BROKEN: the string is built even when the level is disabled, and there is no structured data
logger.LogDebug($"Processing order {order.Id} for {order.Customer}");

// FIXED: a message template. Arguments are captured as named fields, and formatting is skipped
logger.LogDebug("Processing order {OrderId} for {Customer}", order.Id, order.Customer);
```

Message templates are the whole basis of structured logging (№57), and interpolation destroys it by collapsing the template into a unique string per call. The `[LoggerMessage]` source generator (§9.5) generates the fastest form of the second and is worth using on hot paths.

## 8.4 Formatting, parsing and culture

Every format and parse operation takes an optional `IFormatProvider`, and the default is the current culture. That means `123.45.ToString()` produces `"123,45"` on a machine set to `de-DE`, and `decimal.Parse("1,234")` means different numbers in different places.

**Anything machine-readable must specify `CultureInfo.InvariantCulture`.** Anything human-readable should specify the user's culture explicitly rather than relying on the thread's. The rule is the same shape as §8.2's, and CA1305 enforces it.

```csharp
value.ToString("F2", CultureInfo.InvariantCulture);
DateTimeOffset.UtcNow.ToString("O");                    // round-trip: always invariant
decimal.TryParse(s, NumberStyles.Number, CultureInfo.InvariantCulture, out var d);
```

Parsing is generic since C# 11: `IParsable<T>` and `ISpanParsable<T>` expose `Parse` and `TryParse` as static abstract members, so `T.Parse(s, provider)` works inside a generic method (§2.14).

## 8.5 Dates and times

Four types, and choosing the wrong one is where the bugs come from.

| Type | Holds | Use for |
|---|---|---|
| `DateTimeOffset` | An instant, plus the UTC offset that was in force | **Anything that happened**: timestamps, audit records, event times |
| `DateTime` | A date and time, plus a three-valued `Kind` (Utc, Local, Unspecified) | Legacy interop, and only that |
| `DateOnly` / `TimeOnly` | A calendar date; a wall-clock time (.NET 6) | Birthdays, opening hours, anything with no instant attached |
| `TimeSpan` | A duration | Elapsed time, timeouts, intervals |

`DateTime`'s problem is `Kind`. It is metadata attached to the value, it is not part of equality or comparison, and it is routinely lost: serialise a `DateTime` and deserialise it and `Kind` is often `Unspecified`, at which point conversions silently apply the server's local offset. **Two `DateTime` values that are equal may be different instants, and two that are different instants may compare equal.** `DateTimeOffset` has no such ambiguity because the offset is part of the value.

The offset is not the time zone, though. `+01:00` does not tell you whether that is British Summer Time or Central European Time, and you cannot do future-dated arithmetic with it because you do not know when the rules change. **Store the zone ID separately when you need future local times**: a recurring 09:00 meeting in London is `DateOnly` plus `TimeOnly` plus `"Europe/London"`, resolved to an instant at the point of use, not a stored `DateTimeOffset`.

`TimeZoneInfo.FindSystemTimeZoneById` accepts both IANA (`"Europe/London"`) and Windows (`"GMT Standard Time"`) identifiers on all platforms since .NET 6, so the old cross-platform mapping problem is gone.

## 8.6 `TimeProvider`

`DateTime.UtcNow` in the middle of business logic makes that logic untestable. Every codebase used to grow its own `IClock` interface; since .NET 8 there is a standard one:

```csharp
public class ExpiryService(TimeProvider time)
{
    public bool IsExpired(Order o) => o.ExpiresAt <= time.GetUtcNow();
}

// production
services.AddSingleton(TimeProvider.System);

// tests
var fake = new FakeTimeProvider(new DateTimeOffset(2026, 1, 1, 0, 0, 0, TimeSpan.Zero));
fake.Advance(TimeSpan.FromHours(2));
```

`TimeProvider` also supplies timers and a stopwatch, which makes delay-based code testable too (`FakeTimeProvider` is in the `Microsoft.Extensions.TimeProvider.Testing` package). **Inject it everywhere rather than calling `DateTimeOffset.UtcNow` directly**; it costs nothing and removes an entire class of flaky test.

## 8.7 Numbers

`double` is IEEE-754 binary floating point: fast, and unable to represent `0.1` exactly. `decimal` is a 128-bit base-10 type with 28 to 29 significant digits: exact for decimal fractions, roughly an order of magnitude slower, and always overflow-checked (§2.13).

```csharp
0.1 + 0.2 == 0.3;         // false
0.1m + 0.2m == 0.3m;      // true
```

**Money is `decimal`. Measurements and statistics are `double`.** Never compare `double` values with `==`; compare against a tolerance appropriate to the magnitudes involved.

The rounding default surprises people: `Math.Round` uses **banker's rounding**, rounding halves to the nearest even number, so `Math.Round(2.5)` is `2` and `Math.Round(3.5)` is `4`. That is the statistically correct default and is not what a finance system usually wants:

```csharp
Math.Round(2.5);                                    // 2
Math.Round(2.5, MidpointRounding.AwayFromZero);     // 3
```

Also available when you need them: `Half` (.NET 5), `Int128`/`UInt128` (.NET 7), `BigInteger` for unbounded integers, and the `INumber<T>` hierarchy (§2.13) for generic arithmetic.

## 8.8 `System.Text.Json`

The built-in serialiser. It is faster and stricter than `Newtonsoft.Json`, which it replaced as the default, and the strictness is where migrations trip.

By default it serialises **public properties only**, using the declared names, and is case-sensitive on the way in. ASP.NET Core does not use those defaults: it uses `JsonSerializerDefaults.Web`, which is camelCase output, case-insensitive input, and numbers readable from strings. **So the same object serialised by your code and by your API can produce different JSON**, which is the first surprise.

```csharp
private static readonly JsonSerializerOptions Options = new(JsonSerializerDefaults.Web)
{
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
    Converters = { new JsonStringEnumConverter() }
};
```

Reuse that instance. `JsonSerializerOptions` builds and caches a metadata graph on first use, so constructing one per call is a serious and easily-missed performance bug; since .NET 8 an options object becomes read-only after first use, which turns the mistake into an exception rather than a slow leak.

Other defaults worth knowing: fields are ignored unless `IncludeFields` is set; enums serialise as numbers unless you add `JsonStringEnumConverter`; comments and trailing commas are rejected unless allowed; and `null` for a non-nullable property is accepted silently, which is §4.4's hole.

Two additions in .NET 10 are directly useful. `JsonSerializerOptions.Strict` is a preset that turns on the safe-by-default behaviours in one line, and `AllowDuplicateProperties = false` rejects JSON containing the same property twice, which is a real request-smuggling vector when two parsers in a chain disagree about which wins (№60).

**Source generation** is the other half. `[JsonSerializable]` on a partial `JsonSerializerContext` makes the compiler emit the serialisation code at build time:

```csharp
[JsonSerializable(typeof(Order))]
internal sealed partial class AppJsonContext : JsonSerializerContext;

JsonSerializer.Serialize(order, AppJsonContext.Default.Order);
```

That is faster, allocates less, and is **required** under Native AOT and trimming, because the reflection-based path cannot survive either (§1.7).

## 8.9 Files and streams

`Stream` is the abstraction: `FileStream`, `MemoryStream`, `NetworkStream`, the compression streams, all interchangeable. `File.ReadAllTextAsync`, `File.WriteAllLinesAsync` and friends cover the simple cases; drop to `Stream` when the data is large enough that materialising it is the problem.

Two portability points, because Linux containers are where this code ends up. **Always build paths with `Path.Combine`**, never string concatenation with `"\\"`. And **Linux file systems are case-sensitive**, so `Config.json` and `config.json` are different files: a project that works on Windows and fails in the container with a `FileNotFoundException` is almost always this.

`FileStreamOptions` gives you buffer size, `FileOptions.Asynchronous` and pre-allocation in one place, and matters when the file is large; the defaults are fine otherwise.

---

# Part 9 — The toolchain in practice

## 9.1 Solutions and project layout

A solution is a list of projects. It has no build semantics of its own beyond ordering and configuration mapping, and it exists mostly for the IDE.

The format changed. **`dotnet new sln` produces the new XML `.slnx` format by default as of the .NET 10 SDK**, and `dotnet sln migrate` converts an existing `.sln`. The new format is a readable, diffable XML tree with no GUIDs, which removes the single most reliable source of merge conflicts in a .NET repository. Both formats build; there is no reason to stay on the old one except tooling that has not caught up.

The layout that causes least friction:

```
src/
  Orders.Api/          Orders.Api.csproj
  Orders.Domain/       Orders.Domain.csproj
  Orders.Infrastructure/
tests/
  Orders.Domain.Tests/
  Orders.Api.IntegrationTests/
Directory.Build.props
Directory.Packages.props
.editorconfig
Orders.slnx
```

Project references form the dependency graph and the compiler enforces acyclicity, which makes them the cheapest architectural enforcement available: if `Domain` does not reference `Infrastructure`, no one can accidentally call into it (№42 on the dependency rule).

## 9.2 `Directory.Build.props`

MSBuild automatically imports the nearest `Directory.Build.props` from the project directory upwards, before the project file, and `Directory.Build.targets` after it. That is where settings shared by every project belong:

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <LangVersion>latest</LangVersion>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <AnalysisMode>Recommended</AnalysisMode>
    <InvariantGlobalization>true</InvariantGlobalization>
    <ContinuousIntegrationBuild Condition="'$(CI)' == 'true'">true</ContinuousIntegrationBuild>
  </PropertyGroup>
</Project>
```

Setting these once means a new project inherits the standards rather than being created without them, which is the whole point: **a rule that has to be copied into each new `.csproj` is a rule that will be missing from the project created next month.**

`<LangVersion>` is worth understanding: it defaults to the highest version the TFM supports, so `net10.0` implies C# 14. Setting `latest` or `preview` decouples them, and `preview` is what you need to try C# 15 features on a .NET 11 preview SDK. Pinning it to an older version is occasionally used to keep a codebase compilable by an older toolchain.

## 9.3 Central package management

Without it, every project names its own package versions and they drift. `Directory.Packages.props` at the repository root fixes the versions once:

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    <CentralPackageTransitivePinningEnabled>true</CentralPackageTransitivePinningEnabled>
  </PropertyGroup>
  <ItemGroup>
    <PackageVersion Include="Serilog.AspNetCore" Version="9.0.0" />
    <PackageVersion Include="FluentValidation" Version="12.0.0" />
  </ItemGroup>
</Project>
```

Projects then write `<PackageReference Include="Serilog.AspNetCore" />` with no version at all, and a version mismatch between two projects becomes impossible rather than merely unlikely. Transitive pinning extends it to dependencies of dependencies, which is how you force a fix for a vulnerable transitive package without waiting for the direct dependency to update it.

Add `<RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>` and commit `packages.lock.json` when you need restores to be reproducible: CI then uses `dotnet restore --locked-mode` and fails if the graph would change, which is the .NET equivalent of a committed lockfile everywhere else (№56).

## 9.4 Analysers and `.editorconfig`

The .NET SDK ships hundreds of analysers, on by default since .NET 5, producing `CA` diagnostics for correctness, security, performance and API design. `<AnalysisMode>` chooses how many are enabled: `Default` is a small set, `Recommended` is the useful middle, `All` will bury you.

`.editorconfig` is where severities and code style live, and both the IDE and the build read it:

```ini
[*.cs]
dotnet_diagnostic.CA1310.severity = error   # StartsWith without StringComparison (§8.2)
dotnet_diagnostic.CA2007.severity = none    # ConfigureAwait: irrelevant in ASP.NET Core (№80 §3.5)
csharp_style_namespace_declarations = file_scoped:error
csharp_prefer_braces = true:warning
```

`<EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>` makes the `IDE` style rules build failures rather than IDE-only squiggles, which is what stops style discussions in review.

Third-party analyser packages worth knowing: **Roslynator** (a large set of refactorings and diagnostics), **SonarAnalyzer.CSharp** (bug and code-smell rules), **Meziantou.Analyzer** (a strong opinionated set), and **Microsoft.VisualStudio.Threading.Analyzers** (async correctness, which pairs with №80).

> **The tell — analysers or review:** anything a machine can check should be an analyser at error severity, not a review comment. Nullable warnings, `StringComparison`, `ConfigureAwait`, `async void`, disposal, style. Review time is finite and should be spent on design, naming and whether the thing is worth building. If you find yourself making the same review comment twice, look for the rule ID.

## 9.5 Source generators

A source generator is a compiler plugin (an `IIncrementalGenerator`) that reads the compilation, including its syntax trees and its metadata, and adds more source to it during the build. It cannot modify existing code, only add, which is why the pattern is always `partial`: you declare the shape, the generator supplies the body.

They exist because the second organising idea has a cost. Everything the runtime knows can be discovered by reflection, and reflection is slow at startup, invisible to the linker, and impossible under Native AOT (§1.7). A generator does the same discovery at compile time and emits ordinary code, so it is faster, trimmable and AOT-safe.

The ones shipped in the box, all worth using:

| Attribute | Replaces |
|---|---|
| `[JsonSerializable]` | Reflection-based `System.Text.Json` (§8.8) |
| `[GeneratedRegex]` | `new Regex(...)` interpreted or run-time-compiled (.NET 7) |
| `[LoggerMessage]` | `logger.LogInformation(template, args)` on hot paths |
| `[LibraryImport]` | `[DllImport]` marshalling stubs (§9.7) |
| `[OptionsValidator]` | Reflection-based data annotation validation of options |

```csharp
internal static partial class Log
{
    [LoggerMessage(Level = LogLevel.Warning, Message = "Order {OrderId} rejected: {Reason}")]
    public static partial void OrderRejected(ILogger logger, int orderId, string reason);
}
```

The generated method does the level check first and formats nothing when the level is disabled.

Writing your own is a real but sharp option: generators run on every keystroke in the IDE, so an inefficient one makes the editor unusable, and the incremental API exists specifically so that unchanged inputs skip work. Read one of the shipped ones before writing your own.

## 9.6 Reflection, attributes and `dynamic`

Reflection is how you ask the runtime about types at run time: `typeof(Order).GetProperties()`, `Activator.CreateInstance(t)`, `method.Invoke(obj, args)`. Attributes are the declarative half: metadata attached to a declaration, readable by reflection, by analysers, or by generators.

Its costs are worth carrying. **A reflective member access is roughly two to three orders of magnitude slower than a direct one**, mostly in the lookup rather than the invoke, so caching the `PropertyInfo` recovers most of it. Beyond the speed, reflection is invisible to the trimmer and to Native AOT: nothing tells the linker that a type discovered by string name is used, so it gets removed and the failure appears only in the published build.

The escape hatches, in increasing order of preference: cache the `MemberInfo`; compile an expression tree or a delegate once and reuse it (§5.9); or move the whole thing to a source generator. **New code should reach for the generator.** The ecosystem is moving that way for exactly these reasons: object mappers, validators, mocking libraries and DI containers all have generator-based implementations now.

`dynamic` is a different mechanism, the Dynamic Language Runtime, which defers binding entirely to run time. You lose IntelliSense, compile-time checking, refactoring and speed. Its legitimate uses are COM interop and genuinely schemaless data, and even for the latter `JsonNode` or `JsonDocument` is better. Treat `dynamic` in application code as a smell.

## 9.7 Interop and `unsafe`

`unsafe` unlocks pointers, `fixed` (pinning a managed object so a pointer to it stays valid), and `sizeof` on arbitrary structs; it requires `<AllowUnsafeBlocks>true</AllowUnsafeBlocks>`. Most code that used to need it now wants `Span<T>` instead (§2.12), which is safe and usually as fast.

Calling native code is the `[LibraryImport]` source generator:

```csharp
[LibraryImport("libc", SetLastError = true)]
internal static partial int chmod(string path, int mode);
```

It generates the marshalling stub as C# at compile time rather than emitting it at run time, so it is faster, debuggable, AOT-compatible and warns about marshalling mistakes. The older `[DllImport]` still works and is what you will see in existing code. `[StructLayout(LayoutKind.Sequential)]` pins down field ordering when a struct crosses the boundary, which is the one place §2.3's layout caveat matters.

## 9.8 Testing

Three frameworks, all fine: **xUnit** (the de facto default in new .NET projects, now on v3), **NUnit**, **MSTest**. The differences are ergonomic rather than capability: xUnit constructs a fresh test class instance per test and has no `[SetUp]`, using the constructor and `IDisposable` instead, which is a cleaner model and the reason most greenfield projects pick it.

The test host is changing. **Microsoft.Testing.Platform** is replacing VSTest: tests build into a self-contained executable that runs directly, which is faster, works under Native AOT, and removes the `vstest.console` layer. `dotnet test` drives both, and the frameworks have shipped support; new projects should opt in.

Around the frameworks:

- **NSubstitute** or **Moq** for test doubles. Moq's 4.20 telemetry incident in 2023 pushed a lot of teams to NSubstitute; both work.
- **Testcontainers** for integration tests against a real database or broker in Docker. This is now the default answer for data-access tests and largely displaces in-memory database providers, which lie about behaviour (№44).
- **`WebApplicationFactory<Program>`** for in-process ASP.NET Core integration tests, which needs the `public partial class Program { }` line mentioned in §3.12.
- **Verify** for snapshot testing of complex output.

One licensing fact worth flagging, because it caught a lot of teams: **FluentAssertions moved to a paid commercial licence at version 8 in January 2025.** Version 7 remains under the old terms, and **AwesomeAssertions** is a community fork of the last Apache-licensed version with the same API. If you are starting a project, decide deliberately rather than installing the package everyone remembers; **Shouldly** is the other established alternative. This is the kind of thing to re-check rather than assume (№60 on licence and supply-chain review).

№44 covers what makes a test worth having; this section is only the inventory.

---

# Part 10 — When to use what

**A. `class`, `record`, `struct`, or `record struct`?**

Tell → `record`: the type is defined by its data and two instances with the same data are interchangeable (DTOs, domain values, events, messages, configuration). Tell → `class`: the type has identity, behaviour or a lifecycle (entities, services, anything in the container). Tell → `readonly record struct`: a small immutable value that will exist in large numbers or in a hot path. Tell → plain `struct`: essentially never, and if you write one, write `readonly struct`. Default: **`record` for data, `class` for behaviour, and seal both.**

**B. How does this type get equality?**

Tell → generated: it is a `record` or `record struct`, and you take what the compiler writes. Tell → hand-written `IEquatable<T>` plus `Equals` plus `GetHashCode` plus `==`: it is a plain type whose value equality is intrinsic. Tell → an `IEqualityComparer<T>` at the point of use: equality means different things in different contexts, or you are matching on a subset of fields. Tell → reference equality: the type has identity. Default: **make it a record and stop thinking about it.**

**C. Annotation, runtime check, or both?**

Tell → both: this is a public API boundary, or the value came from deserialisation, configuration, a database or reflection. Tell → annotation only: the value came from your own annotated code. Tell → `!`: you can state the invariant the compiler cannot see, in a sentence, in a comment. Default: **`ArgumentNullException.ThrowIfNull` at the boundary, annotations everywhere inside it, and nullable warnings promoted to errors.**

**D. What should this method return?**

Tell → `IReadOnlyList<T>` or a concrete `List<T>`: you have materialised the results, and the caller may enumerate more than once. Tell → `IEnumerable<T>`: the sequence is genuinely lazy or unbounded and you want the caller to control materialisation. Tell → `IAsyncEnumerable<T>`: the same, but items arrive over time (№80 §3.8). Default: **materialise and return `IReadOnlyList<T>`**; returning a lazy `IEnumerable<T>` from a method that has already done the work hides a fact the caller needs.

**E. `IQueryable` or materialised?**

Tell → let `IQueryable<T>` escape the data-access layer: never, in application code. Tell → keep it inside a repository or query object: always, so the shape of the SQL is decided where the schema is known. Tell → `AsEnumerable()` deliberately: you need an operation the provider cannot translate, and you have already filtered enough that pulling the rest is cheap. Default: **the data-access layer returns materialised results and owns every clause.**

**F. Which immutable-ish collection?**

Tell → `FrozenDictionary`/`FrozenSet`: built once at startup, read for the process lifetime, and reads are hot. Tell → `ImmutableArray<T>`: read constantly, replaced wholesale and rarely. Tell → `ImmutableList<T>`: genuinely edited, and versions must be shared cheaply. Tell → `.AsReadOnly()`: you still own the collection and are handing out a view. Default: **`ImmutableArray<T>` for shared snapshots, `FrozenDictionary` for startup-built lookups, and a plain `List<T>` when the thing never leaves the method.**

**G. Exception or return value?**

Tell → return value (`TryXxx`, nullable, a result type): the failure is a normal outcome for the immediate caller, or happens in a loop. Tell → exception: the failure means an assumption of your own code is false, and the caller has nothing sensible to do. Tell → exception with a filter: you will retry a specific, identifiable transient condition. Default: **expected outcomes in the signature, broken assumptions as exceptions, and nothing thrown per-item inside a loop.**

**H. Which string comparison?**

Tell → `StringComparison.Ordinal` or `OrdinalIgnoreCase`: identifiers, keys, paths, headers, protocol tokens, database values, anything a machine reads. Tell → `CurrentCulture`: a human will read the resulting order, and you have said so explicitly. Tell → `InvariantCulture`: you need stable culture-sensitive behaviour across machines, which is a rare and usually mistaken requirement. Default: **`Ordinal`, stated explicitly, with CA1307 and CA1310 as errors.**

**I. Which date type?**

Tell → `DateTimeOffset`: something happened at an instant. Tell → `DateOnly`/`TimeOnly`: a calendar date or a wall-clock time with no instant attached. Tell → `DateOnly` plus `TimeOnly` plus an IANA zone ID: a future local time whose offset is not yet decided. Tell → `DateTime`: an API you do not control demands one. Default: **`DateTimeOffset` for events, `DateOnly`/`TimeOnly` for calendars, and `TimeProvider` injected rather than `UtcNow` called.**

**J. `decimal` or `double`?**

Tell → `decimal`: money, and anything where a decimal fraction must be exact. Tell → `double`: measurements, statistics, geometry, anything where the input was approximate anyway. Tell → `Int128` or `BigInteger`: exact integers beyond 64 bits. Default: **`decimal` for money, `double` for everything else, and never `==` on a `double`.**

**K. Reflection, expression trees, or a source generator?**

Tell → source generator: the work can be done from information available at compile time, which covers serialisation, mapping, logging, validation, interop and most DI. Tell → a compiled expression tree: the shape is known only at run time but is reused many times. Tell → plain reflection: it happens once, at startup, and is not on any hot path. Default: **a generator for anything new, cached reflection for one-off startup work, and neither if Native AOT is a target.**

**L. Which deployment model?**

Tell → framework-dependent: you control the base image or the machine. Tell → self-contained: you do not, and cannot install a runtime. Tell → Native AOT: startup latency or memory is the binding constraint, and you have designed the dependency set around it from the beginning. Tell → ReadyToRun: startup matters but reflection does not permit AOT. Default: **framework-dependent, on a runtime container image, on the current LTS.**

---

# How to expand this

## Already in the library

Read these rather than duplicating them here.

| Document | What it gives this one |
|---|---|
| **№80 C# Threading and Asynchrony** | Everything concurrent, which is deliberately absent from this document: `Task`, `async`/`await` and its state machine, the thread pool, cancellation, the memory model, locks, channels and parallelism. §2.12's restriction on `ref struct` across `await`, §6.5's DI lifetimes, §3.8's delegates and §7.4's `ExceptionDispatchInfo` all point into it |
| **№14 Frameworks & DI** | The container model that §6.5 reads as a disposal contract: lifetimes, scopes, compile-time versus run-time resolution |
| **№30 Algorithms & Patterns** | The structure-to-cost mapping behind §5.2's table. The costs are language-independent; the table is only the .NET names for them |
| **№42 Architecture** | The dependency rule that §9.1's project layout enforces mechanically |
| **№44 Testing & Correctness** | What makes a test worth having, which §9.8 assumes and only inventories the tooling for |
| **№40 Clean Code & Refactoring** | Naming and API design, which §3.11's argument against implicit conversions and §7.2's against exceptions-as-control-flow are instances of |
| **№57 Observability** | Structured logging, which §8.3's message-template rule exists to preserve |
| **№60 Application Security** | Input validation at the boundary (§4.4), duplicate-property handling (§8.8), and the supply-chain case for §9.3 and §9.8 |
| **№20 / №21 Java Data-Access** | The unit-of-work and identity-map model that EF Core also implements, which §5.9's `IQueryable` boundary is the front end of |

## The .NET band, 80–89

This document is the second in the band. The remaining gaps, in the order they would pay off:

- **№82 ASP.NET Core.** Kestrel, the middleware pipeline, minimal APIs versus controllers, model binding, configuration and the host, `System.IO.Pipelines`. Both this document and №80 assume it: §6.5's scopes are request scopes, §8.8's `Web` defaults are its defaults. The largest remaining gap.
- **№83 EF Core.** Change tracking, the query pipeline, migrations, and the boundary §5.9 stops at. Pairs with №20 and №21, which describe the same problems from the JPA side.
- **№84 Azure.** The counterpart to №54 and №91, and the most directly useful of these for consultancy work.
- **№85 The .NET runtime.** GC generations and modes, tiered compilation and OSR, Native AOT, trimming, and the allocation price list that §2.4, §2.12, §5.4 and §9.6 all reason about without ever quoting.

## Candidates for deeper treatment

**Source generators and Roslyn**: §9.5 describes what they are and how to consume them, and says nothing about writing one, which is a subject in itself, syntax trees, semantic models, and the incremental pipeline. **LINQ providers and expression trees**: §5.9 explains the mechanism at the level you need to avoid the traps; building a provider is a different document. **The `Span<T>` and allocation-free style**: §2.12 introduces the type, and the discipline of writing an entire parsing or serialisation path with no allocation deserves its own treatment, most naturally as part of №85. **`System.Text.Json` in depth**: converters, polymorphic serialisation and the contract model are half a document, currently compressed into §8.8.

---

*Mechanisms described here (the value/reference split, boxing, reified generics and shared canonical instantiations, iterator and expression-tree lowering, the two-pass exception model, the dispose and finalisation contracts) are stable and have been for a decade or more; they will not drift. Version-specific claims were verified on 10 September 2026 against Microsoft Learn and the dotnet/core release notes rather than written from memory: .NET 10 is the current release and current LTS, shipped 11 November 2025 and supported to 14 November 2028, and .NET 8 (LTS) and .NET 9 (STS) both leave support on 10 November 2026. .NET 11 is at release candidate with general availability expected in November 2026; C# 15 is its default language version, and the union types and closed hierarchies referenced in §3.7 and §7.8 are release-candidate features that may still change, so re-check them against the .NET 11 release notes before building on them. The C# 14 feature list in §3.2, §3.10, §3.11 and §2.12 (the `field` keyword, extension members, user-defined compound assignment, implicit span conversions, unbound generic `nameof`, lambda parameter modifiers, partial constructors and events) was taken from the Microsoft Learn "What's new in C# 14" page. The LINQ availability table in §5.10 was checked method by method, and corrects a claim that circulates widely: `Enumerable.LeftJoin` and `Enumerable.RightJoin` are .NET 11 additions and are documented as prerelease; what shipped in the .NET 10 timeframe was EF Core 10's queryable `LeftJoin`/`RightJoin`. .NET 9's `CountBy`, `AggregateBy` and `Index` were confirmed against the .NET 9 libraries release notes. The `.slnx` default in §9.1 was confirmed against the SDK 10 breaking-change documentation for `dotnet new sln`. The FluentAssertions licence change in §9.8 was confirmed: version 8, January 2025, commercial licence required, with version 7 remaining under the previous terms and AwesomeAssertions existing as a community fork; this is the item here most likely to have moved again, and licence terms should be checked at the point of adoption rather than trusted from any document. Object header sizes and minimum object size in §2.3 are CoreCLR on 64-bit and are implementation details rather than specification guarantees. The behaviour claims in §2.10 (the three different outcomes of mutating a struct through a `List<T>` indexer, an array element and a `readonly` field), §5.5 (iterator argument validation deferring to enumeration) and §8.2 (the ordinal-versus-culture defaults per method) are the ones most worth confirming yourself in a scratch project, because they are the ones that read as implausible until you see them.*
