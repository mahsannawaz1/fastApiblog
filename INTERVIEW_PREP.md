# Interview Preparation Guide — .NET / React / Angular / Databases

> Tailored for **Muhammad Ahsan Nawaz** — .NET Backend Engineer @ Techverx (EF Core, Dapper, PostgreSQL, RabbitMQ/MassTransit, SignalR, multi-tenant, distributed locking).

This guide answers every question you listed, gives interview-ready talking points, and adds CV-style follow-ups the panel will likely throw at you.

---

## Table of Contents
1. [C# Language Basics](#1-c-language-basics)
2. [OOP](#2-oop)
3. [Coding Problems (with C# solutions)](#3-coding-problems)
4. [SQL & Databases](#4-sql--databases)
5. [.NET / CLR / ASP.NET Core](#5-net--clr--aspnet-core)
6. [Dependency Injection (Deep Dive)](#6-dependency-injection)
7. [Async / Await](#7-async--await)
8. [SOLID & Design Patterns](#8-solid--design-patterns)
9. [LINQ & Collections](#9-linq--collections)
10. [Entity Framework Core](#10-entity-framework-core)
11. [Azure (App Service, Functions, Managed Identity, Secrets)](#11-azure)
12. [React / Angular](#12-react--angular)
13. [Resume-Driven Questions](#13-resume-driven-questions)
14. [Self-rating script](#14-self-rating-script)

---

## 1. C# Language Basics

### `ref` vs `in` vs `out`
All three pass arguments **by reference** (not by value), but differ in direction and initialization:

| Keyword | Caller must initialize? | Callee must assign? | Callee can read? | Callee can modify? |
|---|---|---|---|---|
| `ref` | ✅ Yes | ❌ No | ✅ | ✅ |
| `out` | ❌ No | ✅ Yes (before return) | ❌ (until assigned) | ✅ |
| `in` | ✅ Yes | ❌ No | ✅ | ❌ Read-only |

```csharp
void DoRef(ref int x) { x++; }            // bidirectional
void DoOut(out int x) { x = 10; }         // output only
void DoIn(in int x)  { /* x is readonly */ } // input only, pass-by-ref (perf for large structs)
```

**When to use `in`?** Large `readonly struct`s (e.g., `Matrix4x4`) — avoids defensive copy.
**`out` classic use:** `TryParse`, `Dictionary.TryGetValue`.

### `const` vs `readonly`
| | `const` | `readonly` |
|---|---|---|
| When assigned | Compile time | Compile time **or** in constructor |
| Implicitly `static` | ✅ | ❌ (can be instance or static) |
| Allowed types | Primitives, `string`, `null` for refs | Any type |
| Compiled into callers | ✅ (baked at consumer) | ❌ (resolved at runtime) |
| Versioning risk | High — changing the value requires recompiling consumers | Safe to change |

> Pro tip: For values that are conceptually constant but might change (like `Pi`-ish business values), use `static readonly`. Only use `const` for *truly* immutable literals like `MaxRetryCount = 3`.

---

## 2. OOP

### Four Pillars
1. **Encapsulation** — bundle data + behavior; expose only what's needed (access modifiers, properties).
2. **Inheritance** — `class Dog : Animal` reuses base behavior.
3. **Polymorphism** — same interface, different behavior. Two kinds:
   - *Compile-time* → overloading.
   - *Runtime* → overriding via `virtual`/`override`.
4. **Abstraction** — expose *what* the object does, hide *how* (interfaces/abstract classes).

### Overloading vs Overriding
- **Overloading** — same method name, different signature, same class (or via inheritance). Resolved at **compile time**.
- **Overriding** — derived class redefines a `virtual`/`abstract` method. Resolved at **runtime** via vtable.

```csharp
class Shape { public virtual double Area() => 0; }
class Circle : Shape {
    public double R { get; }
    public Circle(double r) => R = r;
    public override double Area() => Math.PI * R * R; // overriding
    public double Area(double scale) => Area() * scale; // overloading
}
```

### Composition vs Inheritance — "Favor composition over inheritance"
- **Inheritance (IS-A)**: `Car : Vehicle`. Tight coupling, base changes ripple down, single base class in C#.
- **Composition (HAS-A)**: `Car` *has an* `Engine`. Flexible — swap implementations at runtime.

> Interview line: *"Inheritance is good for true behavioral hierarchies, but I default to composition because it gives me runtime flexibility, easier mocking in tests, and doesn't lock me into a single base class."*

### Diamond Problem
Multiple inheritance ambiguity:
```
   A
  / \
 B   C
  \ /
   D    // which A.Method() does D inherit?
```
- C++ has it. **C# avoids it** by disallowing multiple class inheritance.
- C# *interfaces* can multi-inherit. Since C# 8, interfaces can have **default implementations** — re-introducing a mild diamond. Resolution rule: if conflicting defaults, the most-derived interface wins; otherwise the implementing class must explicitly disambiguate:
```csharp
interface IA { void M() => Console.WriteLine("A"); }
interface IB : IA { void IA.M() => Console.WriteLine("B"); }
interface IC : IA { void IA.M() => Console.WriteLine("C"); }
class D : IB, IC { void IA.M() => Console.WriteLine("D"); } // must resolve
```

### Interface vs Abstract Class
| | Interface | Abstract class |
|---|---|---|
| Multiple inheritance | ✅ | ❌ |
| Fields | ❌ (only props/methods, static fields since C# 8) | ✅ |
| Constructors | ❌ | ✅ |
| Default implementations | ✅ (C# 8+) | ✅ |
| Use when | Capability/contract (`IDisposable`, `IRepository`) | Shared base behavior + state (`Stream`, `DbContext`) |

> Rule of thumb: *Interface for "can do"*, *abstract class for "is a"*.

---

## 3. Coding Problems

### 3.1 Count number of `1`s

**(a) Input is a string** — count occurrences of the character `'1'`:
```csharp
int CountOnesInString(string s) {
    int count = 0;
    foreach (char c in s) if (c == '1') count++;
    return count;
}
```

**(b) Input is a number** — count set bits in its binary representation (no string conversion):
```csharp
int CountOnesInNumber(int n) {
    int count = 0;
    while (n != 0) {
        count += n & 1;   // check LSB
        n >>>= 1;         // unsigned right shift (handles negatives)
    }
    return count;
}
// Brian Kernighan optimization:
int CountOnesFast(int n) {
    int c = 0;
    while (n != 0) { n &= (n - 1); c++; } // clears lowest set bit each iteration
    return c;
}
```
Mention: `BitOperations.PopCount((uint)n)` is the framework method (since .NET Core 3.0).

### 3.2 Linked List Cycle Detection — Floyd's Tortoise & Hare
```csharp
public class ListNode { public int Val; public ListNode? Next; }

bool HasCycle(ListNode? head) {
    var slow = head;
    var fast = head;
    while (fast != null && fast.Next != null) {
        slow = slow!.Next;
        fast = fast.Next.Next;
        if (slow == fast) return true;
    }
    return false;
}
```
Time **O(n)**, Space **O(1)**. To find the **cycle start**, reset one pointer to head and advance both one step at a time until they meet.

### 3.3 First Non-Repeating Character
```csharp
char? FirstNonRepeating(string s) {
    var counts = new Dictionary<char, int>();
    foreach (var c in s) counts[c] = counts.GetValueOrDefault(c) + 1;
    foreach (var c in s) if (counts[c] == 1) return c;
    return null;
}
```
Two passes, **O(n)**. For ASCII you can use `int[128]` for O(1) space.

### 3.4 Missing element in `[1,2,3,5,6,7]` — no loop / no recursion

**Sum formula (Gauss):**
```csharp
int MissingBySum(int[] arr, int n) {
    long expected = (long)n * (n + 1) / 2;
    long actual = arr.Sum();          // LINQ - if "no loop" means no explicit loop, this is OK
    return (int)(expected - actual);
}
```
> If the interviewer also forbids `Sum`, mention that ultimately *something* must iterate; the *spirit* of the question is "no explicit loop in your code".

**XOR approach:**
```csharp
int MissingByXor(int[] arr, int n) {
    int x = 0;
    for (int i = 1; i <= n; i++) x ^= i;
    foreach (var v in arr) x ^= v;
    return x;
}
```
**Why XOR works:**
- `a ^ a = 0`, `a ^ 0 = a`, XOR is commutative & associative.
- XOR everything from `1..n` with every element of the array. Each present number cancels itself out (appears twice in the XOR chain). The only number that doesn't cancel is the missing one.
- Advantages: no overflow risk (unlike sum on huge n), constant extra space.

### 3.5 Swap k elements in array — `[1,2,3,4,5,6]`, k=2 → `[5,6,3,4,1,2]`
Looks like swapping the **first k** with the **last k**:
```csharp
void SwapKEnds(int[] a, int k) {
    if (k < 0 || 2 * k > a.Length) throw new ArgumentException();
    for (int i = 0; i < k; i++) {
        (a[i], a[a.Length - k + i]) = (a[a.Length - k + i], a[i]);
    }
}
```
Time **O(k)**, in-place.

### 3.6 FizzBuzz-style — return two arrays from one function (multiples of 3, 5, and 8)
Interviewer's hint: define a small DTO/record and return it.

```csharp
public record MultiplesResult(int[] MultiplesOf3, int[] MultiplesOf5And8);

MultiplesResult GetMultiples(int upTo) {
    var threes = new List<int>();
    var fivesAndEights = new List<int>(); // multiples of BOTH 5 and 8 → multiples of 40
    for (int i = 1; i <= upTo; i++) {
        if (i % 3 == 0) threes.Add(i);
        if (i % 5 == 0 && i % 8 == 0) fivesAndEights.Add(i);
    }
    return new MultiplesResult(threes.ToArray(), fivesAndEights.ToArray());
}
```
Alternatives to mention:
- `out` parameter for the second array.
- `ValueTuple`: `(int[] a, int[] b) GetMultiples(...)`.
- The class/record approach is best because it's *self-documenting*.

---

## 4. SQL & Databases

### 4.1 Student & Course query "using all clauses of SELECT"
Classic full-clauses query (`SELECT ... FROM ... JOIN ... WHERE ... GROUP BY ... HAVING ... ORDER BY ... TOP/LIMIT`):

```sql
SELECT  TOP 5
        s.StudentId,
        s.Name,
        COUNT(c.CourseId)        AS CoursesTaken,
        AVG(c.Credits * 1.0)     AS AvgCredits
FROM    Student s
JOIN    StudentCourse sc ON sc.StudentId = s.StudentId
JOIN    Course c         ON c.CourseId   = sc.CourseId
WHERE   s.IsActive = 1
        AND c.Credits >= 3
GROUP BY s.StudentId, s.Name
HAVING   COUNT(c.CourseId) > 2
ORDER BY AvgCredits DESC, s.Name ASC;
```

**Logical processing order** (memorize this — interviewers love it):
`FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → TOP/OFFSET-FETCH`

Note: `WHERE` filters rows *before* grouping; `HAVING` filters *after* grouping (it can use aggregates).

### 4.2 Procedures vs Functions (SQL Server / Oracle PL/SQL — relevant to your Techlogix work)

| | Stored Procedure | Function |
|---|---|---|
| Return value | Optional, can return multiple result sets via `SELECT` | Must return a single value (scalar) or table |
| Use in SELECT | ❌ Cannot embed in queries | ✅ Can be used in `SELECT`, `WHERE`, `JOIN` |
| DML (INSERT/UPDATE/DELETE) | ✅ Allowed | ❌ Not allowed in pure functions (side-effect free) |
| Transactions | ✅ Can start/commit/rollback | ❌ Cannot manage transactions |
| Calls another | Can call functions and other procs | Can call other functions, **not** procs |
| Try/catch | ✅ | Limited |
| Output params | ✅ Multiple `OUT`/`OUTPUT` params | Single return value (or table) |

> Tie-in to your CV: *"At Techlogix I worked in FLEXCUBE on PL/SQL packages — procedures handled account-creation workflows with DML and transaction control, while functions were used for reusable validations like checking IBAN format."*

### 4.3 Indexing

**Clustered vs Non-clustered**
- **Clustered**: defines the *physical order* of rows in the table. The leaf level **is** the data. **Only one per table** (because data can be physically sorted only one way). Usually on the primary key.
- **Non-clustered**: a separate B-tree whose leaf level contains the index key + a *row locator* (clustered key, or RID for heaps). You can have many.

**Why only one clustered index?**
A clustered index dictates physical row order on disk; you can't sort the same rows two different ways simultaneously.

**Composite / multi-column clustered index** — yes, supported. Useful when most queries filter by a fixed column order (e.g., `(TenantId, CreatedAt)` in a multi-tenant app — directly relevant to your Techverx work).

**Leftmost prefix rule**
For an index on `(A, B, C)`, the index can efficiently serve queries filtering on:
- `A`
- `A, B`
- `A, B, C`

It *cannot* efficiently serve `B` alone, or `B, C`. Reason: the B-tree is sorted first by `A`, then by `B` within equal-`A` rows, etc. Without `A`, you can't seek.

**Covering index**
A non-clustered index that contains *all* columns needed by the query — either in the key or as `INCLUDE` columns:
```sql
CREATE NONCLUSTERED INDEX IX_Orders_Customer
    ON Orders(CustomerId)
    INCLUDE (OrderDate, TotalAmount);
```
Query `SELECT OrderDate, TotalAmount FROM Orders WHERE CustomerId = @id` is **covered** — no need to jump back to the clustered index.

**Key Lookup / Bookmark Lookup**
When a non-clustered index finds a matching row but doesn't contain all needed columns, SQL Server performs a *lookup* into the clustered index (key lookup) or heap (RID/bookmark lookup) to fetch the missing columns. Lookups are expensive on large result sets — the fix is to make the index **covering** (`INCLUDE` the needed columns).

### 4.4 CTE (Common Table Expression)
A named, temporary result set, scoped to one statement:
```sql
WITH ActiveStudents AS (
    SELECT StudentId, Name FROM Student WHERE IsActive = 1
)
SELECT * FROM ActiveStudents WHERE Name LIKE 'A%';
```

**CTE vs Subquery vs Temp Table**

| Aspect | CTE | Subquery | Temp Table (`#t`) |
|---|---|---|---|
| Scope | Single statement | Single statement | Whole session |
| Readability | High (named) | Low when nested | High |
| Reusable in same query | ✅ Can reference itself / multiple times (but re-executed unless materialized) | ❌ | ✅ (single materialization) |
| Stats / indexes | ❌ no stats | ❌ | ✅ has stats, indexable |
| Best for | Recursive queries, breaking complex logic | One-off filters | Large intermediate results reused multiple times |

**Recursive CTE use cases**
Organizational charts, bill-of-materials, folder hierarchies, transitive relationships, generating number/date series.

```sql
WITH OrgChart AS (
    SELECT EmpId, ManagerId, Name, 0 AS Level
    FROM Employee WHERE ManagerId IS NULL
    UNION ALL
    SELECT e.EmpId, e.ManagerId, e.Name, oc.Level + 1
    FROM Employee e JOIN OrgChart oc ON e.ManagerId = oc.EmpId
)
SELECT * FROM OrgChart;
```

### 4.5 Window Functions: `ROW_NUMBER()` & `LAG()`

```sql
-- ROW_NUMBER: rank rows per partition
SELECT UserId, LoginDate,
       ROW_NUMBER() OVER (PARTITION BY UserId ORDER BY LoginDate) AS rn
FROM Logins;

-- LAG: peek at the previous row's value
SELECT UserId, LoginDate,
       LAG(LoginDate) OVER (PARTITION BY UserId ORDER BY LoginDate) AS PrevLogin
FROM Logins;
```

### 4.6 Gaps & Islands — Longest Consecutive Login Streak per User

Classic technique: subtract `ROW_NUMBER()` from the date. Consecutive dates produce the **same difference**, identifying an "island".

```sql
WITH Numbered AS (
    SELECT UserId, LoginDate,
           DATEADD(day,
                   -ROW_NUMBER() OVER (PARTITION BY UserId ORDER BY LoginDate),
                   LoginDate) AS GroupKey
    FROM (SELECT DISTINCT UserId, CAST(LoginAt AS date) AS LoginDate FROM Logins) d
),
Islands AS (
    SELECT UserId, GroupKey,
           COUNT(*)        AS StreakLength,
           MIN(LoginDate)  AS StreakStart,
           MAX(LoginDate)  AS StreakEnd
    FROM Numbered
    GROUP BY UserId, GroupKey
)
SELECT UserId,
       MAX(StreakLength) AS LongestStreak
FROM Islands
GROUP BY UserId;
```
Why it works: for consecutive dates `(d, d+1, d+2)` and ROW_NUMBER `(1,2,3)`, `d - 1 = (d+1) - 2 = (d+2) - 3` → same group. A gap breaks the equality and starts a new island.

### 4.7 Query Optimization Checklist
- Use **EXPLAIN / Actual Execution Plan**; look for scans where seeks are expected.
- **Index** correctly: leading columns must match filter; consider `INCLUDE` for covering.
- Avoid **SELECT \***; project only needed columns.
- Replace **scalar UDFs** with inline TVFs (scalar UDFs serialize execution).
- Use **set-based** operations; avoid cursors.
- Beware **parameter sniffing** — use `OPTION (RECOMPILE)` or `OPTIMIZE FOR UNKNOWN` when plan is unstable.
- **SARGable** predicates: `WHERE Col = @x` (good) vs `WHERE FN(Col) = @x` (bad — kills index seek).
- Up-to-date **statistics**; auto-update on.
- Watch for **implicit conversions** (e.g., `NVARCHAR` column compared to `VARCHAR` parameter → scan).
- **Pagination**: `OFFSET/FETCH` is OK for small offsets; for large, use keyset pagination (`WHERE Id > @lastId`).
- For EF: project with `Select`, avoid `Include` chains when not needed, batch with `AsNoTracking`, use `Split queries` for cartesian explosion.

---

## 5. .NET / CLR / ASP.NET Core

### C# vs .NET
- **C#** is a *language* (syntax, semantics, type rules) — compiled by the Roslyn compiler to **CIL** (Common Intermediate Language).
- **.NET** is the *platform/runtime/BCL* — includes the **CLR** (Common Language Runtime) which executes CIL, plus thousands of libraries.

### Can we use languages other than C# in .NET?
Yes — any CLI-compliant language compiles to CIL. Officially supported: **C#**, **F#**, **VB.NET**. Historically: C++/CLI, IronPython, IronRuby, etc.

### CLR
The runtime that executes managed code. Responsibilities:
- **JIT compilation** of CIL → native code at runtime (or AOT via NativeAOT/ReadyToRun).
- **Garbage Collection** (generational: Gen 0, 1, 2 + LOH; server vs workstation GC).
- **Type safety**, exception handling, assembly loading.
- **Thread management**, security (CAS historically, now mostly OS-level).

### CIL (Common Intermediate Language)
- CPU-independent bytecode emitted by C#/F#/VB compilers.
- JIT'd to machine code on first execution of each method.
- View with `ildasm` or `dotnet-ildasm`/SharpLab.

### MVC
Architectural pattern in ASP.NET Core:
- **Model** — domain data and business logic.
- **View** — UI rendering (Razor in MVC; not used in Web API).
- **Controller** — receives HTTP request, orchestrates model + view, returns response.

Separation enables testability and parallel development.

### RESTful APIs
Constraints (Roy Fielding):
1. **Client-Server** — separation of concerns.
2. **Stateless** — each request carries all info; server keeps no client session.
3. **Cacheable** — responses declare cacheability.
4. **Uniform Interface** — resources identified by URIs, manipulated via standard verbs (GET/POST/PUT/PATCH/DELETE), self-descriptive messages, HATEOAS (often skipped in practice).
5. **Layered System** — intermediaries (proxies, gateways) are transparent.
6. **Code on Demand** (optional).

Practical tips:
- Use **nouns**, not verbs: `/orders/123` not `/getOrder?id=123`.
- Proper status codes: 200, 201, 204, 400, 401, 403, 404, 409, 422, 500.
- **Idempotency**: PUT/DELETE idempotent; POST is not.

### Middleware
Components forming the HTTP **request pipeline**. Each can:
- Inspect/modify the request.
- Pass to the next via `await next()`.
- Inspect/modify the response on the way back.

```csharp
app.Use(async (ctx, next) => {
    var sw = Stopwatch.StartNew();
    await next();
    logger.LogInformation("{path} took {ms}ms", ctx.Request.Path, sw.ElapsedMilliseconds);
});
```

**Order matters** — Exception handler → HTTPS redirection → Static files → Routing → Auth → Authorization → Endpoints.

Difference from a filter: middleware is global to the pipeline; filters are MVC-specific (action, exception, authorization, resource, result filters).

### `String` vs `StringBuilder`
- `string` is **immutable**. Every concatenation allocates a new string + copies. Doing `s += x` in a loop is O(n²).
- `StringBuilder` maintains a mutable internal buffer; append is amortized O(1).

Rule of thumb:
- **< ~5 concatenations** or compile-time known → use `string` (or interpolation `$"..."`).
- **Loops / dynamic many appends** → `StringBuilder`.
- For very hot paths, use `string.Create` or `Span<char>`.

### `Interface` vs `Abstract class` — *see §2*.

### N+1 problem in EF / LINQ
You issue one query to get N parent rows, then iterate and trigger one extra query *per parent* to load a related collection → **1 + N** queries.

```csharp
var blogs = db.Blogs.ToList();
foreach (var b in blogs) {
    Console.WriteLine(b.Posts.Count); // N extra queries if lazy-loading enabled!
}
```

**Fixes:**
- `Include` (eager): `db.Blogs.Include(b => b.Posts).ToList();`
- **Projection**: `db.Blogs.Select(b => new { b.Title, PostCount = b.Posts.Count() }).ToList();`
- **Split queries** to avoid cartesian explosion on multiple Includes: `.AsSplitQuery()`.
- Disable lazy loading proxies in production-critical paths.

---

## 6. Dependency Injection

### Service lifetimes in ASP.NET Core

| Lifetime | Instance per | Use when |
|---|---|---|
| **Transient** | Every resolution | Lightweight, stateless services (mappers, small utilities) |
| **Scoped** | Per HTTP request (per scope) | DbContext, per-request state, UnitOfWork |
| **Singleton** | App lifetime, shared across requests | Caches, config, logger factories, thread-safe stateless services |

### Can a Singleton depend on a Scoped service?
**Not directly — it causes a "captive dependency"**. The scoped service gets captured by the singleton and effectively lives forever, which:
- Bypasses per-request isolation.
- Causes thread-safety issues (scoped services are usually *not* thread-safe).
- For `DbContext`, **leaks tracked entities forever**, causes memory growth, and `DbContext` itself isn't thread-safe → exceptions.

ASP.NET Core's default DI validates this in **Development** and throws `InvalidOperationException` when you build the host with `ValidateScopes = true` (default in dev).

### What if a Singleton depends on DbContext (scoped)?
- DbContext gets captured.
- Concurrent requests hit it → `InvalidOperationException: A second operation was started on this context instance before a previous operation completed`.
- Change tracker accumulates entities → memory leak.
- Stale data across requests.

### How to safely use a scoped dependency from a Singleton?
Inject **`IServiceScopeFactory`** (or `IServiceProvider`) and create a scope on demand:

```csharp
public class BackgroundProcessor {
    private readonly IServiceScopeFactory _scopeFactory;
    public BackgroundProcessor(IServiceScopeFactory scopeFactory)
        => _scopeFactory = scopeFactory;

    public async Task ProcessAsync() {
        using var scope = _scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        // use db within the scope only
    }
}
```

This is the standard pattern in `IHostedService` / `BackgroundService` (very relevant to your RabbitMQ/MassTransit work — consumers create scopes per message).

### Circular dependency
```csharp
class A { public A(B b) {} }
class B { public B(A a) {} }
services.AddTransient<A>();
services.AddTransient<B>();
// → InvalidOperationException: A circular dependency was detected
```

The DI container detects the cycle while building the object graph and throws. Fixes:
- **Redesign** — usually a sign that responsibilities are wrong; extract a third type both depend on.
- **Lazy resolution** with `Lazy<T>` (requires registering `Lazy<T>` factory) or `Func<T>`.
- **Property injection** as a last resort (anti-pattern for required deps).

### DI container internals (object graph construction)
1. **Registration**: each service descriptor (type + lifetime + factory).
2. **Resolution request**: container looks up the descriptor for the requested type.
3. **Constructor selection**: picks the constructor with the most parameters it can satisfy. *(ASP.NET Core default container is strict; Autofac is more permissive.)*
4. **Recursive resolve** of each constructor parameter — building the *object graph*.
5. **Lifetime check**: singleton/scoped/transient cache lookup; scope validation.
6. **Disposal**: container tracks `IDisposable`/`IAsyncDisposable` instances and disposes them when the scope (or root container) is disposed.

### Types of DI
1. **Constructor injection** — preferred; dependencies are required and visible.
2. **Property/setter injection** — for optional deps.
3. **Method injection** — pass per-call.

> Interview tip: When asked "scenarios where to use which lifetime":
> - **Singleton**: `IMemoryCache`, `IHttpClientFactory`, `ILogger<T>` infra, configuration, custom thread-safe in-memory state.
> - **Scoped**: `DbContext`, `IUnitOfWork`, current-user/tenant context, request-scoped caches.
> - **Transient**: validators, mappers, command/query handlers — lightweight, no shared state.

---

## 7. Async / Await

### `.Result` vs `await`
- `await` is **non-blocking**: the method suspends, the thread is released; when the task completes the continuation resumes (often on the captured sync context).
- `.Result` (and `.Wait()`) **block** the calling thread until the task finishes, *and* unwrap exceptions into `AggregateException`.

```csharp
var result = await func(); // good — non-blocking
var result = func().Result; // bad — blocks the thread + deadlock risk in UI/old ASP.NET
```

### Why can `.Result` cause deadlocks?
Classic scenario (older ASP.NET / WinForms / WPF):
1. Caller is on a UI/request thread with a `SynchronizationContext` that allows only **one thread at a time**.
2. `await SomethingAsync()` (inside the called method) tries to resume on that same context.
3. But the context is **blocked** because the caller is sitting on `.Result` waiting.
4. → deadlock.

**ASP.NET Core does not have a synchronization context**, so this specific deadlock is rare in Core — but `.Result` is still bad because it ties up a thread-pool thread (causing **thread starvation**) and breaks `IAsyncDisposable`/cancellation semantics.

### Why is blocking async code bad in ASP.NET Core?
- Thread-pool threads are a finite resource. Blocked threads can't serve other requests → throughput collapses (**thread starvation / pool exhaustion**).
- Async I/O frees the thread *during* the wait; blocking defeats the entire purpose.
- Symptoms: 503s, slow responses under load, queue growth.

### Running multiple async methods in parallel
```csharp
// Sequential — total time ≈ tA + tB + tC
var a = await GetA();
var b = await GetB();
var c = await GetC();

// Parallel — total time ≈ max(tA, tB, tC)
var tA = GetA();
var tB = GetB();
var tC = GetC();
await Task.WhenAll(tA, tB, tC);
var (a, b, c) = (tA.Result, tB.Result, tC.Result); // safe after WhenAll
```

Caveats:
- Don't parallelize on the **same DbContext** — it's not thread-safe.
- For external HTTP / independent DBs / queue calls — parallelize freely.

### Does calling an async method start execution immediately?
**Yes** — it starts running synchronously up to the **first `await` that returns an incomplete task**. At that point control returns to the caller; the rest is scheduled as a continuation.

### Predict the output
```csharp
Console.WriteLine("1");
var task = LongTask();     // runs synchronously until first incomplete await
Console.WriteLine("2");
await task;
Console.WriteLine("3");
```
Output: **1, 2, 3** — but interleaved with anything `LongTask` prints synchronously *before* its first awaited I/O. If `LongTask` writes `"start"` synchronously and `"end"` after `await Task.Delay(...)`:
```
1
start
2
end
3
```

### `AggregateException` vs actual exception
- When you `await` a task, the runtime **unwraps** the first inner exception — you catch the actual exception type (`HttpRequestException`, etc.).
- When you call `.Result`, `.Wait()`, or `Task.WaitAll`, exceptions surface as **`AggregateException`** with all failures in `InnerExceptions`.
- Use `await task` whenever possible. For multi-task: `await Task.WhenAll(t1, t2)` — still only throws the *first* exception, but `((Task)Task.WhenAll(...)).Exception` gives you the AggregateException with all.

### Synchronization context
- An abstraction representing *where* continuations should run after an `await`.
- UI frameworks (WinForms, WPF): single UI thread context.
- Classic ASP.NET: request context.
- **ASP.NET Core: none** (null context) — continuations run on any thread-pool thread.
- `ConfigureAwait(false)` says: *"I don't need to resume on the captured context."* Recommended in **library code** to avoid deadlocks and minor perf gains. In application code (especially ASP.NET Core) it doesn't matter much.

### Thread starvation
Happens when the thread pool can't grow fast enough to serve incoming work:
- Caused by blocking calls on async code (`.Result`, `Task.Run` on CPU work in tight loops), tying up sync DB calls, or `Thread.Sleep`.
- Pool ramps up slowly (default ~1 thread/sec after min threads), so under burst load you see **response time spikes** and requests pile up.
- Diagnose with `dotnet-counters monitor` (`threadpool-thread-count`, `threadpool-queue-length`).

---

## 8. SOLID & Design Patterns

### SOLID with one-liners
- **S — Single Responsibility**: a class should have one reason to change.
- **O — Open/Closed**: open for extension, closed for modification.
- **L — Liskov Substitution**: subclasses must be usable wherever the base is expected without surprises (no narrowing preconditions, no broader exceptions).
- **I — Interface Segregation**: many small interfaces > one fat one. Clients shouldn't depend on methods they don't use.
- **D — Dependency Inversion**: depend on abstractions, not concretions.

### Which SOLID is violated in the discount service?
```csharp
decimal GetDiscount(string customerType, decimal total) {
    if (customerType == "Regular") return total * 0.05m;
    if (customerType == "Premium") return total * 0.10m;
    if (customerType == "VIP")     return total * 0.20m;
    return 0;
}
```
- **OCP** is the primary violation: adding a new customer type means modifying the method.
- **SRP** is also at risk: the class knows *every* discount rule.

### Refactor via Strategy Pattern + OCP

```csharp
public interface IDiscountStrategy {
    bool AppliesTo(Customer c);
    decimal Calculate(decimal total);
}

public class RegularDiscount : IDiscountStrategy {
    public bool AppliesTo(Customer c) => c.Type == CustomerType.Regular;
    public decimal Calculate(decimal total) => total * 0.05m;
}
public class PremiumDiscount : IDiscountStrategy { /* ... 0.10 ... */ }
public class VipDiscount     : IDiscountStrategy { /* ... 0.20 ... */ }

public class DiscountService {
    private readonly IEnumerable<IDiscountStrategy> _strategies;
    public DiscountService(IEnumerable<IDiscountStrategy> strategies) => _strategies = strategies;

    public decimal Calculate(Customer c, decimal total)
        => _strategies.FirstOrDefault(s => s.AppliesTo(c))?.Calculate(total) ?? 0;
}

// Registration:
services.AddSingleton<IDiscountStrategy, RegularDiscount>();
services.AddSingleton<IDiscountStrategy, PremiumDiscount>();
services.AddSingleton<IDiscountStrategy, VipDiscount>();
services.AddSingleton<DiscountService>();
```
Now adding **Platinum** = adding one class + one registration. No modification to existing code.

### Does ISP apply here?
Not really — there's only one operation (`Calculate`). ISP would matter if the interface had `Calculate`, `LogDiscount`, `EmailCustomer`, etc., where not all strategies need every method. Mention this to show depth.

### SRP vs OCP vs ISP — quick practical examples
- **SRP**: split a `UserService` that does auth, profile updates, and email sending into three services.
- **OCP**: payment gateway abstraction (`IPaymentGateway`) — adding a new gateway (Stripe, PayPal) doesn't change `CheckoutService`.
- **ISP**: instead of one `IRepository<T>` with 15 methods, expose `IReadOnlyRepository<T>` and `IWritableRepository<T>` so query-only consumers don't depend on write methods.

---

## 9. LINQ & Collections

### `IEnumerable<T>` vs `IQueryable<T>`

| | `IEnumerable<T>` | `IQueryable<T>` |
|---|---|---|
| Lives in | `System.Collections.Generic` | `System.Linq` |
| Execution | In-memory (LINQ to Objects) | Builds an **expression tree** → translated by provider (LINQ to EF, LINQ to SQL) |
| Where filters run | In your process | On the database server |
| Use when | Data is already in memory (collections, arrays) | Querying an external store (EF, OData) |
| Deferred? | ✅ | ✅ |

**Subtle trap:**
```csharp
db.Orders.Where(o => o.Total > 100)      // IQueryable — SQL: WHERE Total > 100
   .AsEnumerable()                        // ← switch to in-memory
   .Where(o => DateOnly.FromDateTime(o.CreatedAt) == today); // runs in C#
```
Anything after `AsEnumerable()` / `ToList()` runs client-side.

### When should you use `IQueryable`?
- Whenever the source is a **database/remote provider** and you want filters/sorts/projections pushed down.
- For composable query layers (repositories returning `IQueryable` for callers to refine).

### If no filters needed, `IEnumerable` or `IQueryable`?
- If you'll *materialize and iterate anyway* in memory → `IEnumerable` (simpler, no provider overhead).
- Generally: expose `IQueryable` from the data layer until the boundary; **materialize before returning to the API layer** to prevent leaky abstractions (and to avoid the API filtering the query unintentionally).

### Deferred execution
LINQ queries don't execute when defined — only when **enumerated** (`foreach`, `ToList`, `Count`, `First`, etc.). This enables:
- Composable query building.
- Push-down to the DB.
- But also re-execution if you enumerate twice — call `.ToList()` to cache.

```csharp
var q = db.Orders.Where(o => o.Total > 100); // no SQL yet
foreach (var o in q) {...} // SQL runs here
foreach (var o in q) {...} // SQL runs again!
```

### Expression trees vs delegates
- A **delegate** (`Func<T,bool>`) is a compiled method pointer; you can invoke it but not introspect it.
- An **expression tree** (`Expression<Func<T,bool>>`) is a *data structure* representing the code. LINQ providers walk the tree and translate it (e.g., to SQL).

```csharp
Func<int,bool>             d = x => x > 5;      // compiled lambda
Expression<Func<int,bool>> e = x => x > 5;      // tree of nodes
```
That's why `IQueryable.Where` takes an `Expression<...>` (so EF can translate to SQL), while `IEnumerable.Where` takes a `Func<...>`.

### N+1 in LINQ
Same problem as EF N+1: enumerating in a way that triggers per-item lookups. Avoid by:
- Joining/grouping upfront.
- Using `Include`/projection in EF.
- For in-memory: build a lookup with `ToDictionary` / `ToLookup` once.

---

## 10. Entity Framework Core

### Eager vs Lazy Loading
- **Eager**: `Include` related data in the same SQL query (or split query). Predictable, single round-trip.
- **Lazy**: navigations load on access via proxies (requires `UseLazyLoadingProxies`, virtual navs). Convenient but easy to trigger N+1.
- **Explicit**: `db.Entry(blog).Collection(b => b.Posts).LoadAsync()` — load on demand.

### When to use eager loading
- You know upfront which navigations you'll need.
- You want a deterministic number of queries.
- API endpoints where the shape is fixed.

### When is lazy loading dangerous?
- Inside loops → N+1.
- Across async boundaries (proxy access is synchronous → blocks).
- After `DbContext` is disposed → `ObjectDisposedException`.
- In serialization (web APIs) → can load the entire graph and stack-overflow.

### Projection vs Include
- **Include** pulls the *whole* related entity (all columns).
- **Projection** (`Select(b => new BlogDto { Title = b.Title, Posts = b.Posts.Select(p => p.Title) })`) pulls only what you need — usually **faster and lighter**.
- Projections also bypass change tracking automatically.

### EF Optimizations (Techverx-style answers)
- `AsNoTracking()` for read-only queries.
- Projection DTOs instead of full entities.
- `AsSplitQuery()` to fight cartesian explosion when you `Include` multiple collections.
- Compiled queries (`EF.CompileAsyncQuery`) for hot paths.
- Batch updates: `ExecuteUpdateAsync` / `ExecuteDeleteAsync` (EF 7+) — no tracking, single SQL statement.
- Indexes (DB-side) on FK + filter columns.
- `Take`/`Skip` server-side; never paginate in memory.
- Pool DbContexts via `AddDbContextPool`.
- For high-throughput inserts: **Dapper** or `BulkInsert` libraries (you already use Dapper at Techverx — call this out).
- Disable EF tracking globally for query-heavy services.

---

## 11. Azure

### Azure Functions vs App Service

| | Azure Functions | App Service |
|---|---|---|
| Model | Event-driven, serverless | Always-on web hosting |
| Scaling | Per-execution, scale-to-zero (Consumption); Premium for warm | Manual or autoscale based on metrics |
| Pricing | Pay per execution + memory-seconds | Pay per allocated plan time |
| Cold starts | Yes (Consumption); none on Premium/App Service Plan | No |
| Max runtime | 5 min default / 10 (Consumption); unlimited on Premium | Unlimited |
| Triggers | HTTP, queue, blob, timer, Event Grid, Service Bus, etc. | HTTP only (you bring your own) |
| Use when | Spiky workloads, integrations, background jobs, lightweight APIs | Long-running web apps, full Web APIs, predictable load |

> Tie-in: *"For our RabbitMQ-driven workflow services at Techverx, App Service is a better fit because we need a long-lived consumer; if we had spiky, on-demand jobs, Functions with a queue trigger would be lighter."*

### Azure Managed Identity
An identity automatically provisioned in Azure AD for your Azure resource (App Service, Function, VM, etc.). Two flavors:
- **System-assigned**: tied to the resource lifecycle.
- **User-assigned**: independent identity that can be attached to multiple resources.

You grant the identity RBAC roles on Azure resources (Key Vault, Storage, SQL). Your code uses `DefaultAzureCredential` (Azure.Identity) — **no secrets in code or config**.

```csharp
var client = new SecretClient(new Uri("https://my-kv.vault.azure.net/"),
                              new DefaultAzureCredential());
var secret = await client.GetSecretAsync("DbConnectionString");
```

Even SQL connection strings can be passwordless:
```
Server=tcp:my.database.windows.net; Database=app; Authentication=Active Directory Default;
```

### How to save secrets in a .NET application
**Local development:**
- **User Secrets** (`dotnet user-secrets`) — stored outside the project (`~/.microsoft/usersecrets/...`), so not committed.
- `appsettings.Development.json` is OK only for non-sensitive values.

**Production:**
- **Azure Key Vault** + Managed Identity (preferred).
- AWS Secrets Manager / HashiCorp Vault if not on Azure.
- Environment variables for container apps / Kubernetes (mounted from secret stores).

**Never:** commit secrets to git, store them in `appsettings.json` shipped to prod, log them.

Configuration providers stack:
```csharp
builder.Configuration
    .AddJsonFile("appsettings.json")
    .AddJsonFile($"appsettings.{env}.json", optional: true)
    .AddUserSecrets<Program>(optional: true) // dev only
    .AddEnvironmentVariables()
    .AddAzureKeyVault(new Uri(vaultUri), new DefaultAzureCredential());
```

---

## 12. React / Angular

### Virtual DOM
- An in-memory tree representing the UI.
- React renders to the virtual DOM, then **diffs** against the previous snapshot and applies the **minimum patch** to the real DOM.
- Why? Real DOM mutations are expensive (reflow/repaint). Virtual DOM batches and minimizes them.
- "Reconciliation" + the **Fiber** scheduler enable interruptible rendering and priorities (concurrent rendering since React 18).

### Hooks (interview essentials)
- **`useState`** — local state.
- **`useEffect`** — side effects (subscriptions, fetches). Cleanup via return function. Deps array controls re-run.
- **`useMemo`** — memoize expensive computed values.
- **`useCallback`** — memoize functions (stable references for child memoization).
- **`useRef`** — mutable container that doesn't trigger re-render; also DOM refs.
- **`useContext`** — read context.
- **`useReducer`** — state via reducer pattern.
- **Custom hooks** — extract reusable stateful logic; must start with `use`.

**Rules of hooks:** call at top level, only inside React functions, never conditionally.

### State management options
- **Local state** (`useState`, `useReducer`) — component-local.
- **Lifted state / Context API** — share across subtree, low-frequency updates (theme, auth).
- **Redux / Redux Toolkit** — predictable global store, time-travel debug, middleware (thunks, sagas).
- **Zustand / Jotai** — minimal, no boilerplate, atomic stores.
- **Server state**: **TanStack Query (React Query)** / SWR — caching, dedup, refetch, mutations. Don't put server data in Redux.

> Interview line: *"I separate **client UI state** (Zustand/Context) from **server state** (React Query) — putting server data in Redux is a common anti-pattern that conflates the two."*

### Angular state management (in case asked)
- Component state + `@Input`/`@Output`.
- Services with `BehaviorSubject` (RxJS).
- **NgRx** (Redux-style) for large apps.
- Signals (Angular 16+) — fine-grained reactivity, similar to Solid/Vue.

---

## 13. Resume-Driven Questions

The panel will mine your CV. Have crisp 30–60 second stories for each bullet.

### Techverx — .NET Backend
Likely questions & how to answer:

**Q: Tell me about the multi-tenant architecture you worked on.**
> "We use a discriminator column + tenant-aware DbContext interceptor. Every query is scoped by `TenantId` automatically. For data isolation we have an `ITenantContext` (scoped) populated from the JWT in middleware; the DbContext reads it in `OnModelCreating` via global query filters: `modelBuilder.Entity<X>().HasQueryFilter(e => e.TenantId == _tenant.Id)`. Indexes are composite `(TenantId, ...)` to keep queries efficient."

Be ready for follow-ups:
- Hard delete vs soft delete with filters.
- Cross-tenant admin endpoints (`IgnoreQueryFilters()`).
- Separate-database vs shared-database tradeoffs.

**Q: Distributed locking — how does it work?**
> "We had a workflow that could be triggered from multiple consumers; we needed only one to win. I used a Redis-based distributed lock (e.g., RedLock.NET or a simple `SET NX PX` pattern with a unique token, releasing via Lua script to avoid releasing someone else's lock). Lock acquisition has a timeout and renewal for long jobs."

Follow-ups:
- Why not DB-row lock? (Doesn't scale, blocks readers.)
- What happens if Redis goes down? (Multiple acquirers possible — at-least-once semantics.)
- Lease/heartbeat for long-running locks.

**Q: SignalR real-time updates — design?**
> "We use SignalR with the Azure SignalR Service / Redis backplane for scale-out. Connections are grouped by `TenantId` and user; on workflow events from MassTransit consumers, we publish to a group via `IHubContext<T>`. Auth is JWT on the negotiate request."

**Q: RabbitMQ + MassTransit patterns?**
- Publish/subscribe with topic exchanges.
- Consumer concurrency, prefetch, retry filters (`UseMessageRetry`), redelivery with `UseDelayedRedelivery`, fault queues.
- Sagas / state machines for long-running workflows.
- Outbox pattern to avoid losing messages on DB commit failures.

**Q: How do you optimize DB performance?** (you wrote "API/DB optimization" on your CV)
- Profile with `EXPLAIN ANALYZE` on PostgreSQL.
- Add appropriate indexes (`(TenantId, Status, CreatedAt)`-style composites).
- Replace EF for hot paths with **Dapper** (you do this).
- Pagination via keyset, not offset, on large tables.
- Batch operations (`ExecuteUpdateAsync`).
- Cache hot reads in Valkey (you use Valkey).

**Q: Grafana / Loki — what do you log and how?**
- Structured logging via Serilog (`{TenantId} {CorrelationId} {UserId}`).
- Logs shipped via Promtail/OTel to Loki.
- Dashboards on error rate, latency p95, queue depth.
- LogQL queries to filter (`{app="workflow"} |= "Error" | json | tenant_id="xyz"`).

### Techlogix — Oracle / FLEXCUBE
**Q: What did you do in FLEXCUBE?**
> "I worked on PL/SQL packages, procedures, triggers, and functions powering banking workflows — account opening and cheque book issuance. Typical task: a request comes through the FLEXCUBE workflow, we validate against multiple tables, debit charges, update statuses, and produce GL entries. I learned a lot about transactional boundaries, package state, and how procedures use `AUTONOMOUS_TRANSACTION` for audit-style writes that survive rollback."

Be ready for:
- Difference between **packages** and standalone procedures (encapsulation, persistent state, dependency reduction).
- **Triggers**: row-level vs statement-level; `BEFORE` vs `AFTER`; mutating-table errors.
- **PL/SQL exceptions**: `EXCEPTION WHEN OTHERS THEN ...`, named exceptions, `RAISE_APPLICATION_ERROR`.
- **Cursors**: implicit vs explicit; bulk collect for performance.

### PixelAI (Video Translator)
**Q: Architecture overview?**
> "Full-stack MERN. Upload goes to Node/Express; video is queued to a Python worker pipeline: Whisper for ASR → speaker diarization → MarianMT for translation → a TTS model for target audio → lip-sync alignment (e.g., Wav2Lip). Each stage publishes progress events back to the UI. Storage on S3/GCS, with signed URLs."

Likely follow-ups:
- How do you handle long-running jobs? (Job queue, polling/WebSockets for progress.)
- Cost concerns? (Model inference is expensive; cache by content hash.)
- Failure recovery? (Idempotent stages, retry from last successful checkpoint.)

### Freelance (.NET Core Marketplace)
- Razor Pages or MVC? Be specific.
- Auth (Identity, roles for freelancer vs client).
- EF Core relationships (Job → Proposals, Job → Client, Proposal → Freelancer).
- What would you improve now? (Move to API + SPA, add escrow, add payments, add search.)

### GameHub
- RAWG API integration, debouncing search input.
- Chakra UI theming, dark mode.
- Performance: lazy loading images, infinite scroll with React Query.
- TypeScript: typing API responses with discriminated unions.

---

## 14. Self-rating script

Interviewers love "rate yourself 0–10". Honest framing wins.

**.NET**: *"I'd say a solid **7**. I work daily with .NET Core, EF Core, Dapper, and ASP.NET Web APIs at Techverx, building production microservices with multi-tenancy, distributed locking, and SignalR. I'm comfortable with DI lifetimes, async patterns, EF optimization, and middleware. I'm still leveling up on advanced topics like NativeAOT, source generators, and high-perf `Span<T>`-based code, which is why I'd hold off on 8."*

**React**: *"Maybe a **6**. I built PixelAI's frontend and GameHub in React/TypeScript with hooks and Chakra UI. I'm solid on hooks, virtual DOM, state management (Context + React Query), and component patterns. I haven't shipped large Redux apps, so I'd not call myself 8+."*

**Angular**: *"**3–4** — I understand the conceptual model (components, services, RxJS, DI similar to .NET) but haven't shipped production Angular. Happy to ramp up quickly if needed."*

> Honesty + concrete examples beats inflated numbers every time.

---

## Final tips for the day of the interview

1. **Think aloud** during coding problems. Interviewers grade reasoning more than the perfect answer.
2. For each design question, ask **clarifying questions** first (data size, read/write ratio, consistency requirements).
3. When stuck, **propose a simple solution first**, then say "we could optimize by…".
4. Always tie back to your CV when relevant: *"This reminds me of something we did at Techverx where…"*. It steers questions to your strength.
5. Have **2–3 questions** ready for the interviewer (team structure, on-call, observability stack, biggest engineering challenge).

Good luck — you've got this.
