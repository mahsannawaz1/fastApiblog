# C# and the .NET Ecosystem — Deep Dive

A complete walkthrough of how your C# code goes from a `.cs` file to instructions executing on a CPU, and every important piece of machinery along the way. Read this once end-to-end; it will make every interview question about "CLR, JIT, assemblies, GC, runtime" feel obvious.

## Table of Contents
1. [The Big Picture in One Diagram](#1-the-big-picture-in-one-diagram)
2. [C# vs .NET vs CLR vs BCL — terminology straightened out](#2-c-vs-net-vs-clr-vs-bcl)
3. [The History of .NET (so you know what people mean)](#3-the-history-of-net)
4. [The Compilation Pipeline: `.cs` → CIL → Native](#4-the-compilation-pipeline)
5. [Assemblies: what they really are](#5-assemblies)
6. [Metadata and Reflection](#6-metadata-and-reflection)
7. [The CLR — Common Language Runtime](#7-the-clr)
8. [JIT Compilation in detail](#8-jit-compilation)
9. [Ahead-of-Time (AOT): ReadyToRun, NativeAOT](#9-ahead-of-time-aot)
10. [The Type System and Memory Model](#10-the-type-system-and-memory-model)
11. [Garbage Collection](#11-garbage-collection)
12. [Application Startup: how a .NET app actually launches](#12-application-startup)
13. [Hosting models, runtimes, and SDKs](#13-hosting-models-runtimes-and-sdks)
14. [Putting it together — example walkthrough](#14-putting-it-together)
15. [Interview cheat-sheet](#15-interview-cheat-sheet)

---

## 1. The Big Picture in One Diagram

```
        ┌────────────────┐     compile     ┌──────────────┐
        │  YourCode.cs   │ ───────────────▶│   Roslyn     │
        └────────────────┘   (csc / dotnet) │  Compiler    │
                                            └──────┬───────┘
                                                   │ emits
                                                   ▼
                                 ┌──────────────────────────────┐
                                 │   YourApp.dll  (assembly)    │
                                 │   ├── PE Header              │
                                 │   ├── CLI Header             │
                                 │   ├── CIL bytecode (methods) │
                                 │   ├── Metadata tables        │
                                 │   └── Manifest (refs, ver)   │
                                 └──────────────┬───────────────┘
                                                │ loaded by
                                                ▼
                                 ┌──────────────────────────────┐
                                 │            CLR               │
                                 │  ─ AssemblyLoadContext       │
                                 │  ─ Class loader / verifier   │
                                 │  ─ JIT (RyuJIT) ─────┐       │
                                 │  ─ GC                │       │
                                 │  ─ Exception engine  │       │
                                 │  ─ Threading / SyncCtx│      │
                                 └──────────────────────┼───────┘
                                                        │ JIT-compiles per-method
                                                        ▼
                                 ┌──────────────────────────────┐
                                 │  Native machine code (x64,   │
                                 │  arm64) executed by the CPU  │
                                 └──────────────────────────────┘
```

Every concept in the rest of this document is one of those boxes.

---

## 2. C# vs .NET vs CLR vs BCL

These four terms get mixed up constantly. Pin them down:

| Term | What it actually is | Analogy |
|---|---|---|
| **C#** | A programming **language** — a specification (ECMA-334) defining syntax and semantics. Implemented by the Roslyn compiler. | Like "English" — a set of rules. |
| **CIL / IL / MSIL** | **Common Intermediate Language** — a CPU-independent bytecode that C# (and F#, VB) compiles to. Defined by ECMA-335. | Like "machine-readable English in IPA phonetics" — portable representation. |
| **CLR** | **Common Language Runtime** — the virtual machine that loads assemblies, JIT-compiles CIL to native code, runs GC, manages threads, handles exceptions, etc. | Like the "JVM" of .NET. |
| **BCL / FCL** | **Base Class Library** (a.k.a. Framework Class Library) — the standard library shipped with .NET: `System.*`, `System.Collections.*`, `System.IO`, `System.Net.*`, `System.Linq`, etc. | Like Python's stdlib. |
| **.NET** | The **platform** — CLR + BCL + tooling + SDK. When someone says ".NET 8", they mean a specific version of this whole platform. | Like "Java SE 21" — runtime + libs + tools. |
| **CLI** | **Common Language Infrastructure** — the ECMA-335 spec that defines the runtime, type system, metadata, and CIL. Any compliant impl (CoreCLR, Mono) can run CLI programs. | Like "JVM spec" — the standard. |
| **CTS** | **Common Type System** — the part of the CLI that defines what types look like (`Int32`, `String`, classes, structs, etc.) so multiple languages agree. | Shared type rules. |
| **CLS** | **Common Language Specification** — a subset of CTS that all CLR-targeting languages must agree on for interop. | "Lingua franca" subset. |

So when you say *"I write C# and target .NET 8"*, you're saying: *"I use the C# language; my code compiles to CIL; my CIL runs on the CoreCLR runtime + the .NET 8 BCL."*

> **Interview line:** *"C# is the language; .NET is the platform; CLR is the runtime; CIL is what compiled C# looks like before the JIT turns it into machine code."*

### Can other languages run on .NET?
Yes — any language whose compiler emits valid CIL + metadata. Officially:
- **C#** — Roslyn
- **F#** — functional ML-family language
- **VB.NET** — Visual Basic
- Historically: C++/CLI (managed C++), IronPython, IronRuby, J#, A#, etc.

The reason they interoperate is the **CTS/CLS**: an `int` in C# is the same `System.Int32` value type as an `Integer` in VB.NET, so calling between them is free.

---

## 3. The History of .NET

You'll sometimes see ".NET Framework", ".NET Core", ".NET 5/6/7/8/9" thrown around. Quick chronology so you know what they mean:

| Year | Release | Notes |
|---|---|---|
| 2002 | .NET Framework 1.0 | Windows-only, closed source |
| 2005 | .NET Framework 2.0 | Generics, nullable types |
| 2007–2015 | .NET Framework 3.5 → 4.x | LINQ (3.5), TPL/`async` (4.5), ongoing |
| 2016 | **.NET Core 1.0** | Cross-platform rewrite, open source, modular |
| 2019 | .NET Core 3.x | Windows desktop on Core (WPF, WinForms) |
| 2020 | **.NET 5** | Unified — "Core" dropped from name. **Same codebase forward.** |
| 2021 | .NET 6 LTS | Minimal APIs, hot reload |
| 2022 | .NET 7 | Performance focus, NativeAOT preview-stable |
| 2023 | .NET 8 LTS | Big perf gains, full NativeAOT for ASP.NET Core |
| 2024 | .NET 9 | Latest at time of writing |

**Key takeaway:** Modern code uses **".NET" (5+)**, which runs on the **CoreCLR** runtime everywhere (Windows/Linux/macOS, x64/arm64). The legacy ".NET Framework" (4.8.x) is Windows-only and frozen — mentioned only because lots of enterprise code still runs on it.

There's also:
- **Mono** — the original Linux implementation of the CLR; still used by Unity and historically Xamarin.
- **CoreRT / NativeAOT** — a different runtime that compiles your whole app ahead of time to native code, no JIT.

---

## 4. The Compilation Pipeline

### Step 1 — You write source code
```csharp
// File: Hello.cs
using System;

public class Program {
    public static void Main() {
        Console.WriteLine($"2 + 2 = {Add(2, 2)}");
    }
    static int Add(int a, int b) => a + b;
}
```

### Step 2 — Roslyn compiles it
You run `dotnet build` (or invoke `csc` directly). This launches the **Roslyn** C# compiler:

```
dotnet build  →  csc.dll  →  Roslyn pipeline:
                              ├── 1. Lexer       (tokens)
                              ├── 2. Parser      (syntax tree)
                              ├── 3. Binder      (semantic model, symbols)
                              ├── 4. Lowering    (foreach → for, async → state machine, etc.)
                              ├── 5. Flow analysis (definite assignment, nullability)
                              ├── 6. Codegen     (emit CIL + metadata)
                              └── 7. PE writer   (write the .dll/.exe on disk)
```

Roslyn is itself written in C# and exposed as an API — that's why **source generators**, **analyzers**, and **IDE features** (refactorings, IntelliSense) are all built on the same compiler. You can host Roslyn in your own program.

**Lowering is interesting.** Several C# features don't exist at the CIL level — Roslyn rewrites them to simpler constructs:
- `foreach (var x in list)` → a `while` loop on an enumerator
- `async/await` → a generated **state machine class** with `MoveNext()`
- `yield return` → a generated **iterator state machine**
- LINQ query syntax → method-call syntax (`from x in xs where ...` → `xs.Where(...)`)
- `lock (obj) { ... }` → `try { Monitor.Enter(...); ... } finally { Monitor.Exit(...); }`
- Local functions, lambdas with captures → generated classes/structs

This is why if you decompile your DLL you'll see a class like `<Main>d__0` for an async method.

### Step 3 — Output is an Assembly (`.dll` or `.exe`)
The compiler writes a **portable executable (PE) file** containing:
- CIL bytecode for every method
- Metadata tables describing every type, method, field, parameter, attribute
- A **manifest** listing the assembly's identity, version, references to other assemblies
- Optionally, embedded resources (strings, images), debug info (PDB), strong name

### Step 4 — At runtime, the CLR loads it
When you do `dotnet Hello.dll`, the .NET host (`dotnet.exe` / `apphost`) bootstraps the CLR, which:
1. Loads your assembly via an `AssemblyLoadContext`.
2. Loads every assembly it depends on (transitive closure).
3. Finds your entry point (`Main`).
4. Calls it. The first time `Main` runs, the JIT compiles its CIL to native code; subsequent calls reuse the cached native code.

That's the whole pipeline. The rest of this document zooms in on each box.

### Side-by-side: what the CIL looks like
For the `Add` method above, Roslyn emits (simplified):
```
.method private static int32 Add(int32 a, int32 b) cil managed
{
    .maxstack 8
    ldarg.0       // push 'a'
    ldarg.1       // push 'b'
    add           // pop two, push sum
    ret           // return top of stack
}
```
CIL is a **stack-based** instruction set — there are no registers, just an evaluation stack per method. The JIT translates this to native code with real registers.

Tools to inspect this:
- `ildasm` (Windows SDK) or **`dotnet-ildasm`**
- **ILSpy** / **dnSpy** / **dotPeek** — decompilers (very useful to see what `async` lowering produces)
- **sharplab.io** — paste C# code, see CIL and JITted assembly online

---

## 5. Assemblies

An **assembly** is the unit of deployment, versioning, and security in .NET. Concretely it's a file on disk — usually a `.dll`, sometimes a `.exe`.

### Physical layout (PE format)
.NET assemblies extend the standard Windows **Portable Executable (PE)** format (the same format used by `notepad.exe`):

```
┌────────────────────────────┐
│  DOS Header ("MZ")          │   Legacy Windows compatibility stub
├────────────────────────────┤
│  PE Header                  │   File type, target architecture, sections
├────────────────────────────┤
│  Section Headers            │
├────────────────────────────┤
│  .text section              │
│   ├─ CLI Header             │   The pointer into managed-land
│   ├─ CIL method bodies      │   Your compiled methods
│   ├─ Metadata               │   Tables of types/methods/fields/...
│   └─ Strong-name signature  │   (optional)
├────────────────────────────┤
│  .rsrc section              │   Embedded resources
├────────────────────────────┤
│  .reloc section             │   Base relocations
└────────────────────────────┘
```

The **CLI Header** is what makes this a *managed* assembly — its presence tells the OS loader "hand this off to the CLR, not to the native loader."

### Logical structure
Inside the metadata and CIL, an assembly contains:
- **Manifest** — identity (name, version, culture, public-key token), references to other assemblies, list of modules and files.
- **Types** — every class, struct, interface, enum, delegate.
- **Members** — methods, fields, properties, events for each type.
- **Method bodies** — CIL bytecode + exception handlers.
- **Custom attributes** — `[Obsolete]`, `[Serializable]`, your own `[Authorize]`.
- **Resources** — strings, images, embedded files.
- **PDB** (usually separate `.pdb` file or embedded) — debug info mapping CIL offsets to source lines.

### Assembly identity
An assembly is uniquely identified by:
```
Name, Version, Culture, PublicKeyToken
```
e.g., `System.Text.Json, Version=8.0.0.0, Culture=neutral, PublicKeyToken=cc7b13ffcd2ddd51`

This is how the runtime decides which copy of a library to load when multiple are present.

### `dll` vs `exe`
- A `.dll` exposes types/methods for use by other assemblies.
- An `.exe` is a `.dll` with an **entry point** (`Main`) and a small native bootstrap so the OS can launch it.

In modern .NET (5+), the runtime is often **not** baked into the executable — `dotnet Hello.dll` runs your DLL via the shared host. You can also publish:
- **Framework-dependent**: small output, needs a .NET runtime installed.
- **Self-contained**: bundles a private copy of the runtime (bigger output, no install needed).
- **Single-file**: everything zipped into one `.exe` (extracts at run time or executes in-place).
- **NativeAOT**: real native binary, no CLR/JIT (more in §9).

### How an assembly is created
Two paths:
1. **`dotnet build`** runs MSBuild, which invokes `csc.dll` (the Roslyn compiler) with all the right arguments and references, producing `bin/Debug/net8.0/MyApp.dll`.
2. **Directly**: `csc Hello.cs` → `Hello.exe`. Useful for understanding what's going on, never used in real projects.

### Strong names and signing
A **strong-named** assembly has a public/private key pair embedded — its identity includes a public-key token, ensuring uniqueness and tamper-detection. Less critical in modern .NET (Authenticode + NuGet signature are preferred), but you'll still see it on framework assemblies.

### Loading: `AssemblyLoadContext`
Each .NET process has at least one **AssemblyLoadContext (ALC)**:
- **Default** ALC — loads your app + framework.
- Custom ALCs — used for plugin systems where you want to load + unload assemblies, or load conflicting versions of a library side-by-side.

This replaces the old "AppDomain" concept from .NET Framework. Unloadable ALCs are the modern way to do plugin isolation in .NET Core+.

---

## 6. Metadata and Reflection

Every assembly carries a complete description of itself — that's metadata. This is why **reflection** works: you can inspect types, methods, attributes, and call them dynamically at runtime *without source code*.

```csharp
var asm  = typeof(Program).Assembly;
foreach (var t in asm.GetTypes())
    Console.WriteLine(t.FullName);

var addMethod = typeof(Program).GetMethod("Add", BindingFlags.NonPublic | BindingFlags.Static);
var sum = (int)addMethod.Invoke(null, new object[] { 3, 4 });
```

Metadata is also why these "magical" features work without code generation by your hand:
- **Attributes**: `[Authorize]`, `[HttpGet]`, `[Required]` — frameworks scan metadata at startup.
- **ASP.NET routing**: discovers controllers via reflection.
- **EF Core**: discovers `DbSet<T>` properties.
- **JSON serializers**: read property names via reflection (faster paths use source generators now).
- **DI containers**: pick constructors via reflection.

**The cost:** reflection is much slower than direct calls. Modern .NET ships **source generators** as a compile-time alternative — they generate code at build time, so you get the convenience of reflection with the speed of hand-written calls. `System.Text.Json`, `LoggerMessage`, and ASP.NET Core 8 minimal APIs all use them.

---

## 7. The CLR

The CLR is the engine that runs your CIL. It's a native program written in C++ (sources live in [`dotnet/runtime`](https://github.com/dotnet/runtime)). When you launch a .NET app, the CLR is loaded into your process and stays there for the lifetime of the process.

### CLR subsystems

```
┌─────────────────────────── CLR ───────────────────────────┐
│                                                            │
│  AssemblyLoadContext       Class Loader      Verifier      │
│  (finds & loads .dlls)    (lays out types)  (CIL safety)   │
│                                                            │
│  JIT compiler (RyuJIT)     ─────────►   Native code cache  │
│                                                            │
│  Garbage Collector         Exception Engine                │
│  (generational, SOH/LOH)   (two-pass, SEH-based)           │
│                                                            │
│  Threading & Sync          Interop (P/Invoke, COM)         │
│  (ThreadPool, Tasks)       (managed ↔ native bridge)       │
│                                                            │
│  Profiling & Diagnostics   Security                        │
│  (ETW, EventPipe)          (verification, sandbox-lite)    │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

Let's walk through them.

### 7.1 AssemblyLoadContext (loader)
Resolves "I need `System.Text.Json` version 8.0.0" to a physical file on disk, loads it into the process, parses its metadata. Manages probing paths (your `bin/`, the framework's shared folder, NuGet packages).

### 7.2 Class loader
When a type is first used, the loader walks its metadata and lays it out in memory:
- Computes field offsets (respects `[StructLayout]`).
- Builds a **MethodTable** (a.k.a. EEClass) — the runtime structure that holds the type's vtable, interface map, type info, generic instantiations, etc. Every reference-type object has a hidden pointer to its MethodTable as its first machine word.
- Resolves base classes and interfaces.
- Initializes static fields lazily via `.cctor`.

### 7.3 Verifier
On platforms that enforce verification, the CLR checks that incoming CIL is **type-safe** — no jumping into the middle of methods, no using arbitrary memory as objects, no calling private methods of other types. Untrusted code historically ran "verified"; today verification is mostly a sanity check, and `unsafe` code skips it.

### 7.4 JIT compiler — see §8.
### 7.5 Garbage collector — see §11.
### 7.6 Exception engine
.NET exceptions use **two-pass** unwinding on top of OS-level structured exception handling (SEH on Windows, libunwind on Linux). On `throw`:
1. **First pass**: walk the stack looking for a matching `catch` (or `filter`). Finds the handler frame.
2. **Second pass**: walk the stack again, running every `finally`/`fault` block from the throw site up to the handler.

`when` clauses (exception filters) run during the first pass *without* unwinding — that's why they preserve the original stack in debuggers.

### 7.7 Threading
- Every .NET process has a managed **ThreadPool** with a worker-thread pool and an I/O completion port pool.
- The pool auto-grows when work piles up but ramps slowly (one extra thread/sec by default) — this is why blocking async causes the "starvation" pathology we discussed in the interview prep.
- High-level abstractions (`Task`, `Parallel`, PLINQ, `async/await`) all sit on top of the ThreadPool.

### 7.8 Interop
- **P/Invoke**: call native C functions (`[DllImport("kernel32.dll")] static extern void Sleep(int ms);`). The CLR marshals managed types to native ABIs.
- **COM interop** on Windows for legacy.
- The reverse direction (native code calling managed) is possible via UnmanagedCallersOnly + NativeAOT.

### 7.9 Diagnostics
- **EventPipe** is the modern cross-platform diagnostics protocol — exposes counters, traces, GC events.
- Tools: `dotnet-counters`, `dotnet-trace`, `dotnet-dump`, `dotnet-gcdump`, `PerfView`.
- ETW on Windows is still supported.

---

## 8. JIT Compilation

The **JIT (Just-In-Time)** compiler converts CIL into native machine code at runtime. The default JIT in modern .NET is called **RyuJIT** — written in C++, available for x64, arm64, x86, arm32.

### When does the JIT run?
The first time a method is called, its CIL gets compiled to native code. Subsequent calls use the cached native code directly. This is per-method, not per-assembly.

In more detail:
1. CLR sees a call to method `M`. The vtable slot for `M` points to a tiny stub.
2. The stub calls the JIT, which compiles `M`'s CIL to native code.
3. The JIT writes the native code to a code-heap and patches the vtable slot to point directly at it.
4. Future calls jump straight to the native code.

### Cost vs benefit
- **Cost**: first call is slower ("JIT warm-up"); JIT code lives in memory.
- **Benefit**: the JIT knows the *exact* CPU it's running on. It can use SSE4/AVX/AVX-512, branch hints, and other CPU-specific tricks that an ahead-of-time compiler can't always assume. It also knows runtime data — like which classes are actually loaded — and can use that for **devirtualization** and inlining.

### Tiered Compilation (since .NET Core 3.0, on by default)
The JIT has two tiers:
- **Tier 0 (quick JIT)** — compiles fast, minimal optimization. Used for cold/first calls so startup is snappy.
- **Tier 1 (optimized JIT)** — kicks in for methods that are called frequently ("hot"). The runtime tracks call counts; once a method passes a threshold, it's re-JITted with full optimizations and the call site is patched to use the new code.

This gives you both **fast startup** (Tier 0) and **peak throughput** (Tier 1).

You can disable it with `DOTNET_TieredCompilation=0` for benchmarking, but rarely should.

**On-stack replacement (OSR)** — since .NET 7, long-running methods (think a tight loop in `Main` that takes minutes) can be promoted from Tier 0 to Tier 1 *mid-execution*, without waiting for the next call.

### What optimizations does the JIT do?
Roughly the same ones a C++ compiler does, scaled to "we have a few milliseconds":
- **Inlining** small methods.
- **Constant folding & propagation**.
- **Dead code elimination**.
- **Loop unrolling** (mild) and loop-invariant hoisting.
- **Register allocation** (linear-scan).
- **Bounds-check elimination** for array accesses it can prove are safe.
- **Devirtualization** — when the JIT sees that a virtual call always goes to one type, it replaces it with a direct call (sometimes guarded). Crucial for LINQ/async perf.
- **Escape analysis** (limited) — stack-allocating objects that don't escape (`Span<T>` benefits hugely).
- **Auto-vectorization** with SIMD (`Vector<T>` / hardware intrinsics).
- **Dynamic Profile-Guided Optimization (PGO)** — Tier 0 instruments code to gather actual runtime profile data, which Tier 1 uses. Enabled by default in .NET 6+.

You can see exactly what the JIT did with **DOTNET_JitDisasm** or via **sharplab.io**.

### CoreCLR's native code cache
- Stored in process memory (a private "code-heap").
- Not shared across processes (unless you use ReadyToRun — see §9).
- Lives for the lifetime of the AssemblyLoadContext.

### So... is the JIT good or bad?
- **Good** for general apps — adaptive, CPU-aware, supports dynamic loading.
- **Bad** for: cold-start-sensitive scenarios (Lambda, CLI tools), code-signed environments without write+exec memory (iOS), and ultra-tiny memory budgets.

That's where AOT comes in.

---

## 9. Ahead-of-Time (AOT)

You can compile CIL to native code **at build time** to avoid JIT cost.

### ReadyToRun (R2R) — the gentle AOT
- Ships pre-compiled native code **alongside** the CIL in the same assembly.
- The JIT can fall back / re-optimize at runtime (it's a hint, not a replacement).
- All modern framework DLLs ship as R2R — this is why .NET 8 startup is fast.
- Enable for your own app via `<PublishReadyToRun>true</PublishReadyToRun>`.
- Trade-off: larger binaries, but faster cold start with no behavior change.

### NativeAOT — the aggressive AOT
- Compiles your *entire* app to a real native executable (Linux ELF / Windows PE / macOS Mach-O) using a different runtime (CoreRT-derived).
- **No JIT.** **No CIL.** Tiny runtime overhead.
- Massive startup improvements (think 5ms vs 200ms), huge memory wins, perfect for serverless / containers.
- Restrictions:
  - **No `Assembly.Load` / dynamic code generation** (no JIT).
  - Reflection is heavily trimmed — you need attribution (`[DynamicDependency]`) for things you reflect on.
  - Some libraries don't support trimming.
- Since .NET 8, ASP.NET Core has full NativeAOT support for minimal APIs.

In interviews, mentioning NativeAOT signals you're current with the platform.

---

## 10. The Type System and Memory Model

### Value types vs reference types
- **Value types** (`struct`, `enum`, primitives like `int`, `double`, `bool`, `DateTime`, `Guid`): instances **contain** their data. Copied on assignment/parameter pass. Live on the **stack** when local, or **inline** inside the containing object.
- **Reference types** (`class`, `interface`, `delegate`, arrays, `string`): variables hold a **reference (pointer)** to data on the **heap**. Assignment copies the reference, not the object.

```csharp
struct PointS { public int X, Y; }
class  PointC { public int X, Y; }

var a = new PointS { X = 1 };  var b = a;  b.X = 99;   // a.X is still 1
var c = new PointC { X = 1 };  var d = c;  d.X = 99;   // c.X is now 99
```

### Boxing / unboxing
Storing a value type in a reference-typed slot (e.g., `object o = 42;`) **allocates** a small heap object containing the value — that's **boxing**. Unboxing reverses it. Boxing is a perf trap; generics + `Span<T>` exist largely to avoid it.

### Stack vs heap (a common interview question)
- **Stack**: per-thread, fast LIFO allocation, freed automatically when method returns. Holds local value-type variables, parameters, return addresses.
- **Heap**: shared, managed by GC, slower allocation but flexible lifetime. Holds reference-type objects.
- Distinction is *not* about value vs reference per se — a `struct` that's a *field* of a class lives on the heap (inline inside the class). What matters is **where the slot lives**.

### `string` is special
`string` is a reference type but **immutable** — every "modification" allocates a new string. The CLR also **interns** literal strings (deduplicates them in a global table) so `"hello" == "hello"` is cheap.

### `null`, generics, and value types
- `int x = null;` is illegal — value types can't be null.
- `int? x = null;` works — `Nullable<int>` is a struct that wraps a value + a `HasValue` flag.
- Generics with value types get **separate compiled versions per `T`** (avoiding boxing). Generics with reference types share a single compiled version.

---

## 11. Garbage Collection

The CLR's GC reclaims memory you no longer reference. Highlights:

### Generations
- **Gen 0** — newly allocated short-lived objects. Collected most often (cheap).
- **Gen 1** — survivors of one Gen 0 collection. Buffer between Gen 0 and Gen 2.
- **Gen 2** — long-lived objects. Collected rarely (expensive).
- **Large Object Heap (LOH)** — objects ≥ 85,000 bytes. Treated as Gen 2 by default (since .NET 5, LOH compaction is opt-in).
- **POH (Pinned Object Heap)** since .NET 5 — for pinned objects (e.g., buffers for native interop) to avoid fragmenting other generations.

### The collection process (simplified)
1. **Mark** — walk roots (statics, local variables on every thread's stack, GC handles) and mark every reachable object.
2. **Sweep / Compact** — reclaim unreachable objects. For Gen 0/1 the GC **compacts** (moves surviving objects together) to keep allocation fast. For LOH, traditionally swept without compaction.
3. **Promote** survivors to the next generation.

Most objects die young (the "generational hypothesis"), so most collections only scan Gen 0 — that's why GC is amortized cheap in idiomatic code.

### Workstation vs Server GC
- **Workstation GC** — one heap, one GC thread. Default for client apps; lower latency for small workloads.
- **Server GC** — one heap *per CPU*, GC runs in parallel across cores. Default for ASP.NET; much higher throughput.
- Configure with `<ServerGarbageCollection>true</ServerGarbageCollection>` or `DOTNET_gcServer=1`.

### Background GC
Gen 2 collections can run concurrently with your app threads (with brief pauses), so you don't get long stop-the-world stalls.

### Finalizers and IDisposable
- A class with a **finalizer** (`~MyClass()`) survives one extra GC cycle so its finalizer can run. Avoid finalizers unless you own native resources.
- **`IDisposable`** is the deterministic-cleanup pattern — `using var x = new ...;` guarantees `Dispose()` runs as soon as the scope ends, no GC involvement.
- Use **`SafeHandle`** for native handles instead of writing a finalizer yourself.

### What you can control
- Allocate less. Reuse buffers (`ArrayPool<T>`).
- Prefer `Span<T>`, `Memory<T>`, `stackalloc` for short-lived sequences.
- Avoid LINQ in hot loops (delegates allocate).
- Avoid boxing — use generics, avoid `object`, avoid `Enum.HasFlag` pre-.NET 6.
- Use `ValueTask<T>` where appropriate to avoid `Task` allocations in synchronously-completing async methods.

---

## 12. Application Startup

What happens when you type `dotnet MyApp.dll`:

1. The OS launches the `dotnet` host (a small native executable).
2. The host reads `MyApp.runtimeconfig.json` to figure out which version of the runtime to use (e.g., .NET 8.0.x), then loads `coreclr.dll` / `libcoreclr.so`.
3. CoreCLR boots: initializes JIT, GC, ThreadPool, default `AssemblyLoadContext`.
4. CoreCLR loads `MyApp.dll`, then walks its references, loading `System.Private.CoreLib.dll`, `System.Runtime.dll`, etc., from the shared framework directory (`/usr/share/dotnet/shared/Microsoft.NETCore.App/8.0.x`).
5. CoreCLR finds the entry point (`Program.Main`) via metadata, runs static constructors of any types it touches, JITs `Main`, calls it.
6. Your `Main` runs — for an ASP.NET Core app, this calls `WebApplication.CreateBuilder(args).Build().Run()`, which spins up Kestrel and starts the HTTP pipeline.
7. The process stays alive until you return from `Main` or call `Environment.Exit()`.

For a published **self-contained** app, steps 1–3 happen with the bundled runtime under `publish/` instead of a system install.

For a **NativeAOT** binary, none of the above — your `.exe` *is* the entry point, and the runtime is statically linked.

---

## 13. Hosting Models, Runtimes, and SDKs

The .NET install layout (Linux example):
```
/usr/share/dotnet/
├── dotnet                              # the host executable
├── shared/
│   ├── Microsoft.NETCore.App/8.0.5/    # the runtime + BCL
│   ├── Microsoft.AspNetCore.App/8.0.5/ # ASP.NET Core libs
│   └── Microsoft.WindowsDesktop.App/.. # (Windows only)
└── sdk/8.0.300/                        # build-time tooling (csc, MSBuild, dotnet CLI)
```

- **SDK** = build tools. Only needed for developers. Contains compilers, MSBuild, project templates.
- **Runtime** = `Microsoft.NETCore.App`. Needed by any .NET app at runtime.
- **ASP.NET Core runtime** = adds `Microsoft.AspNetCore.App` on top.

Two big publish profiles:
- **Framework-dependent** (`dotnet publish`): produces small DLLs that need a runtime installed on the target machine.
- **Self-contained** (`-r linux-x64 --self-contained`): bundles the runtime inside your output.

Single-file publish wraps everything into one executable. AOT compiles to a true native binary.

---

## 14. Putting It Together — Example Walkthrough

Let's trace a single line:

```csharp
var greeting = $"Hello, {name}!";
Console.WriteLine(greeting);
```

1. **Roslyn lexes** the source into tokens (`var`, identifier, `=`, interpolated-string-start, etc.).
2. **Roslyn parses** into a syntax tree (`LocalDeclarationStatement` with an `InterpolatedStringExpression`).
3. **Binding/semantic analysis** resolves `name` to a local, infers `greeting` as `string`, resolves `Console.WriteLine` to the `string`-overload in `System.Console`.
4. **Lowering** rewrites `$"Hello, {name}!"` to either `string.Concat("Hello, ", name, "!")` or `DefaultInterpolatedStringHandler.AppendFormatted(...)` depending on .NET version.
5. **Codegen** emits CIL: a few `ldstr`, `ldloc`, `call`, `stloc`, `call` instructions.
6. **PE writer** serializes the assembly to `bin/Debug/net8.0/MyApp.dll`.
7. **`dotnet MyApp.dll`** launches the host → CoreCLR loads → loads `System.Console`, `System.Private.CoreLib`.
8. **Class loader** lays out `Console`'s static fields.
9. **JIT** compiles `Main` on first call. Inside it sees `Console.WriteLine` — JITs that too if not already done.
10. The native code executes: pushes registers, calls `Concat`, gets a heap-allocated `string` pointer back, passes it to `WriteLine`, which calls the OS write syscall.
11. The string object becomes unreachable after `Main` returns; the next Gen 0 GC reclaims it.

Every box in the diagram at the top got touched in those 11 steps.

---

## 15. Interview Cheat-Sheet

When asked "explain CLR and JIT" you can recite:

1. **C# is a language; .NET is the platform; CLR is the runtime; CIL is the bytecode.**
2. **Compile**: Roslyn turns `.cs` → CIL + metadata in a PE-format **assembly** (`.dll`/`.exe`).
3. **Load**: at runtime the CLR loads the assembly via an `AssemblyLoadContext`, walks metadata, resolves dependencies.
4. **JIT**: methods are compiled to native code lazily on first call. **Tiered Compilation** uses a fast Tier 0 for cold paths and re-optimized Tier 1 for hot paths; **PGO** uses runtime data; **OSR** can promote a long-running method mid-execution.
5. **Execute**: native code runs directly on the CPU; the CLR continues to provide GC, exceptions, threading, interop.
6. **GC**: generational (Gen 0/1/2 + LOH + POH), workstation vs server modes, background concurrent collection.
7. **Alternatives**: **ReadyToRun** pre-compiles for faster startup but keeps the JIT; **NativeAOT** removes the JIT entirely for true ahead-of-time native builds.
8. **Metadata + reflection** is why DI, EF, ASP.NET routing, attributes, and serializers all "just work" — modern .NET supplements that with **source generators** to avoid runtime reflection.

Memorize that 8-bullet sequence and you can answer every "explain the .NET ecosystem" question fluently.

---

## Further Reading
- Book: **"CLR via C#"** by Jeffrey Richter — the canonical deep dive.
- Repos: [dotnet/runtime](https://github.com/dotnet/runtime), [dotnet/roslyn](https://github.com/dotnet/roslyn).
- Tools: SharpLab, ILSpy, dotPeek, PerfView, `dotnet-counters`, `dotnet-trace`.
- ECMA specs: 334 (C#), 335 (CLI).
