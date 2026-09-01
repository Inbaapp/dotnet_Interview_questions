# Q2. Injecting DbContext Into a Singleton Background Service

## Interview Question

> You inject `DbContext` into a Singleton Background Service and get random errors. What's wrong?

## Short Answer

`BackgroundService` is registered as a **Singleton**, while `DbContext` is normally **Scoped** and is **not thread-safe**.

Directly injecting and retaining a `DbContext` inside a Singleton background service can cause:

- Lifetime mismatch
- Concurrent access to the same DbContext
- "A second operation was started..." errors
- Long-lived Change Tracking
- Increased memory usage
- Stale entity/state problems

The correct approach is to create a **new DI scope** for each unit of work using `IServiceScopeFactory`, or use `IDbContextFactory<T>` to create a fresh DbContext.

---

## 1. Remember the Three DI Lifetimes

```text
Transient → New instance whenever requested

Scoped    → One instance per scope/request

Singleton → One instance for the entire application
```

`DbContext` is normally **Scoped**.

A `BackgroundService` is normally **Singleton**.

```text
BackgroundService
       |
       | Singleton
       ↓
    DbContext
       |
       | Scoped
       ↓
    Database
```

---

## 2. What Is a Background Service?

A background service is a worker that runs continuously in the application.

```text
Application starts
       ↓
Background Service starts
       ↓
Check database
       ↓
Process jobs
       ↓
Wait
       ↓
Check database again
       ↓
Process jobs
       ↓
...
```

Example:

```csharp
public class OrderWorker : BackgroundService
{
    protected override async Task ExecuteAsync(
        CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // Process orders

            await Task.Delay(5000, stoppingToken);
        }
    }
}
```

Important:

> **A `BackgroundService` is registered as a Singleton.**

It normally lives for the lifetime of the application.

---

## 3. The Common Mistake

A developer might write:

```csharp
public class OrderWorker : BackgroundService
{
    private readonly AppDbContext _context;

    public OrderWorker(AppDbContext context)
    {
        _context = context;
    }

    protected override async Task ExecuteAsync(
        CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var orders = await _context.Orders.ToListAsync();

            // Process orders

            await Task.Delay(5000, stoppingToken);
        }
    }
}
```

At first glance, this looks reasonable.

But the lifetime relationship is wrong:

```text
OrderWorker
     ↓
Singleton
     ↓
DbContext
     ↓
Scoped
```

The Singleton background service is retaining a DbContext that is designed to be short-lived.

---

## 4. Why Is This Dangerous?

There are two major problems.

### Problem 1: Lifetime Mismatch

The BackgroundService can live for:

```text
Hours
Days
Weeks
```

But a DbContext is designed to be:

```text
Short-lived
```

You are effectively turning:

```text
Short-lived DbContext
```

into:

```text
Long-lived DbContext
```

That's a bad design.

---

## 5. Problem 2: DbContext Is NOT Thread-Safe

This is especially important.

Suppose the worker starts two operations:

```text
Worker
  |
  +---- Task 1 → DbContext
  |
  +---- Task 2 → DbContext
```

Both tasks are using the same DbContext.

That is dangerous because:

> **A `DbContext` instance is not thread-safe.**

You may get an error similar to:

```text
A second operation was started on this context instance
before a previous operation completed.
```

The exact wording can vary by EF Core version.

---

## 6. Why Does It Seem "Random"?

It usually isn't truly random.

Consider:

```csharp
var task1 = ProcessOrdersAsync();
var task2 = ProcessOrdersAsync();

await Task.WhenAll(task1, task2);
```

Both operations may use the same:

```csharp
_context
```

Conceptually:

```text
                 Same DbContext
                      |
             +--------+--------+
             |                 |
             ↓                 ↓
          Task 1            Task 2
             |                 |
             ↓                 ↓
        Database query   Database query
```

Sometimes Task 1 finishes before Task 2 starts.

```text
Task 1 → finishes
Task 2 → starts
```

No error.

Other times they overlap:

```text
Task 1 → Query starts
             |
             | still running
             ↓
Task 2 → Query starts
             |
             ↓
           ERROR
```

That's why developers may say:

> "It works sometimes, but randomly fails."

The underlying problem is a **race condition caused by concurrent access to the same DbContext**.

---

## 7. Another Problem: Change Tracking

Remember that DbContext performs Change Tracking.

A long-running worker might do:

```text
DbContext
   ↓
Process 100 orders
   ↓
Track entities
   ↓
Process another 100 orders
   ↓
Track more entities
   ↓
Process another 100
   ↓
Track more...
```

If the same DbContext remains alive for a long time, its ChangeTracker can accumulate a lot of state.

Possible consequences:

- Increased memory usage
- More change-tracking overhead
- Stale entity state
- Unexpected behavior
- Slower operations over time

This is another reason not to keep a DbContext alive for the entire lifetime of a background service.

---

## 8. Correct Solution: Create a Scope

The recommended approach is:

> **Create a new DI scope inside the background service and resolve the DbContext from that scope.**

```text
BackgroundService
   |
   | Singleton
   |
   +---- Create Scope
            |
            ↓
       DbContext #1
            |
            ↓
       Do database work
            |
            ↓
       Dispose scope
            |
            ↓
       DbContext disposed
```

Next iteration:

```text
BackgroundService
   |
   +---- Create NEW Scope
              |
              ↓
         DbContext #2
              |
              ↓
         Do database work
              |
              ↓
         Dispose
```

This gives each unit of work a fresh DbContext.

---

## 9. Correct Code Using `IServiceScopeFactory`

```csharp
public class OrderWorker : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public OrderWorker(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    protected override async Task ExecuteAsync(
        CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            using var scope = _scopeFactory.CreateScope();

            var context =
                scope.ServiceProvider
                     .GetRequiredService<AppDbContext>();

            var orders = await context.Orders
                                      .ToListAsync(stoppingToken);

            // Process orders

            await Task.Delay(
                TimeSpan.FromSeconds(5),
                stoppingToken);
        }
    }
}
```

Now the lifetime is correct:

```text
BackgroundService
      |
      | Singleton
      ↓
IServiceScopeFactory
      |
      +---- Scope #1
      |       |
      |       ↓
      |   DbContext #1
      |
      +---- Scope #2
              |
              ↓
          DbContext #2
```

---

## 10. Why Does `IServiceScopeFactory` Solve It?

The background service creates a new DI scope:

```csharp
using var scope = _scopeFactory.CreateScope();
```

Inside that scope:

```csharp
var context =
    scope.ServiceProvider
         .GetRequiredService<AppDbContext>();
```

ASP.NET Core creates the scoped DbContext.

When the scope is disposed, the DbContext is disposed as well.

```text
Scope created
     ↓
DbContext created
     ↓
Database work
     ↓
Scope disposed
     ↓
DbContext disposed
```

---

## 11. Another Good Solution: `IDbContextFactory`

For background processing, another excellent approach is:

```csharp
IDbContextFactory<AppDbContext>
```

Register it:

```csharp
builder.Services.AddDbContextFactory<AppDbContext>(options =>
    options.UseSqlServer(connectionString));
```

Then:

```csharp
public class OrderWorker : BackgroundService
{
    private readonly IDbContextFactory<AppDbContext> _contextFactory;

    public OrderWorker(
        IDbContextFactory<AppDbContext> contextFactory)
    {
        _contextFactory = contextFactory;
    }

    protected override async Task ExecuteAsync(
        CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await using var context =
                await _contextFactory.CreateDbContextAsync(
                    stoppingToken);

            var orders = await context.Orders
                                      .ToListAsync(stoppingToken);

            // Process orders

            await Task.Delay(
                TimeSpan.FromSeconds(5),
                stoppingToken);
        }
    }
}
```

Every iteration gets a new DbContext:

```text
Iteration 1 → DbContext #1 → Dispose

Iteration 2 → DbContext #2 → Dispose

Iteration 3 → DbContext #3 → Dispose
```

---

## 12. Which Approach Should You Use?

Both are valid.

| Approach | Good for |
|---|---|
| `IServiceScopeFactory` | Background services using multiple scoped dependencies |
| `IDbContextFactory<T>` | Creating short-lived DbContexts specifically for background/database work |

### Practical guideline

If your worker needs several scoped services:

```text
BackgroundService
      ↓
Create Scope
      ↓
Service
      ↓
Repository
      ↓
DbContext
```

Use:

```csharp
IServiceScopeFactory
```

If the worker mainly needs DbContexts:

```text
BackgroundService
      ↓
IDbContextFactory
      ↓
DbContext
```

`IDbContextFactory<T>` is a clean option.

---

## 13. Important Mistake to Avoid

Do NOT try to fix the problem by making DbContext Singleton:

```csharp
builder.Services.AddSingleton<AppDbContext>();
```

This is not the correct solution.

You are hiding the original problem instead of fixing it.

Remember:

```text
DbContext
   ↓
Not thread-safe
   ↓
Short-lived
   ↓
Usually Scoped
```

---

## 14. Interview Answer

### Question

> You inject DbContext into a Singleton BackgroundService and get random errors. What's wrong?

### Strong Answer

> `BackgroundService` is registered as a Singleton, while `DbContext` is normally Scoped and is not thread-safe. Injecting and retaining the DbContext directly in the singleton can create a lifetime mismatch and cause the same DbContext to be reused for too long. If multiple operations run concurrently, they may access the same DbContext at the same time, causing errors such as "A second operation was started on this context instance before a previous operation completed." It can also lead to excessive change tracking and stale state. The correct approach is to create a scope using `IServiceScopeFactory` for each unit of work, or use `IDbContextFactory<T>` to create a fresh DbContext when needed.

---

## 15. Mental Model

### Wrong

```text
             BackgroundService
                  Singleton
                     |
                     |
              ❌ Don't do this
                     |
                     ↓
                DbContext
                  Scoped
```

### Correct — Scope

```text
BackgroundService
    Singleton
       |
       ↓
IServiceScopeFactory
       |
       ↓
Create Scope
       |
       ↓
DbContext
       |
       ↓
Database Work
       |
       ↓
Dispose Scope
```

### Correct — Factory

```text
BackgroundService
    Singleton
       |
       ↓
IDbContextFactory
       |
       ↓
Create DbContext
       |
       ↓
Database Work
       |
       ↓
Dispose DbContext
```

---

## 16. One Sentence to Remember

> **Singleton BackgroundService + Scoped DbContext = Don't inject the DbContext directly; create a scope or use `IDbContextFactory` and use a fresh DbContext for each unit of work.**

---

## Quick Revision

```text
BackgroundService
       ↓
Singleton
       ↓
Cannot safely hold a DbContext for its whole lifetime
       ↓
DbContext is Scoped + NOT thread-safe
       ↓
Create a new scope OR use IDbContextFactory
       ↓
Fresh DbContext
       ↓
Do database work
       ↓
Dispose
```

## Key Interview Keywords

- Singleton
- Scoped
- Dependency Injection
- Lifetime mismatch
- `BackgroundService`
- `DbContext`
- Not thread-safe
- Change Tracking
- Race condition
- `IServiceScopeFactory`
- `IDbContextFactory`
- Unit of Work
- Dispose
