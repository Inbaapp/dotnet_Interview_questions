# EF Core DbContext Lifetime, Change Tracking, and EntityState

## 1. What is `DbContext`?

Think of a school library.

The library has many books, and you need a librarian to:

- Find books
- Tell you what is available
- Keep track of books
- Help you borrow or return books

In Entity Framework Core, **`DbContext` is like the librarian**.

It is the main object EF Core uses to communicate with the database.

```text
Your Application
       |
       v
   DbContext
       |
       v
    Database
```

Example:

```csharp
using var context = new AppDbContext();

var users = context.Users.ToList();
```

---

# 2. What does DbContext "Lifetime" mean?

Lifetime simply means:

> **How long should one `DbContext` object stay alive?**

For example:

```csharp
var context = new AppDbContext();
```

You created a DbContext.

At some point, it needs to be disposed.

```text
Create DbContext
       ↓
Use it
       ↓
Finish the work
       ↓
Dispose DbContext
```

The important question is whether the same DbContext should live for a short time or for the whole application.

---

# 3. The common ASP.NET Core lifetime: Scoped

In ASP.NET Core, `DbContext` is normally registered as **Scoped**.

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));
```

`AddDbContext()` registers it as Scoped by default.

### What does Scoped mean?

Usually:

> **One HTTP request → One DbContext**

Example:

```text
Request 1
    ↓
DbContext #1
    ↓
Database work
    ↓
Request ends
    ↓
DbContext disposed
```

Then another request gets another DbContext:

```text
Request 2
    ↓
DbContext #2
    ↓
Database work
    ↓
Request ends
    ↓
DbContext disposed
```

So:

```text
Request 1 → DbContext #1
Request 2 → DbContext #2
Request 3 → DbContext #3
```

---

# 4. Why shouldn't DbContext live forever?

`DbContext` does more than send SQL to the database.

It also **remembers entities** that it is tracking.

For example:

```csharp
var user = context.Users.First();
```

EF Core can remember something like:

```text
User #10
Name = John
City = Chennai
```

If the DbContext lives for a very long time and keeps tracking more and more entities:

```text
User 1
User 2
User 3
...
User 100,000
```

the context can consume more memory and spend more time managing tracked entities.

Long-lived DbContexts can also lead to:

- More memory usage
- More change-tracking overhead
- Stale data/state problems
- Difficult concurrency behavior
- Hard-to-manage application state

Therefore, DbContext is designed to be **short-lived**.

---

# 5. DbContext as a Unit of Work

A very important EF Core concept is **Unit of Work**.

A Unit of Work means:

> Do a related group of database operations, save them, and finish.

For example, creating an order:

```text
Create Order
    ↓
Add Order Items
    ↓
Calculate Total
    ↓
Save Everything
    ↓
Done
```

Example:

```csharp
context.Orders.Add(order);
context.OrderItems.Add(item1);
context.OrderItems.Add(item2);

context.SaveChanges();
```

Think of:

> **DbContext = a workspace for one database job.**

---

# 6. What is Change Tracking?

Imagine you have a notebook.

You ask your friend for John's details:

```text
John
Age: 25
City: Chennai
```

You write them in your notebook.

Later, John's city changes:

```text
Before:
City = Chennai

After:
City = Bangalore
```

Because you were watching John, you know something changed.

EF Core's `DbContext` does something similar.

When EF Core queries an entity, the DbContext normally starts tracking it.

```text
Database
   ↓
DbContext
   ↓
User object
   ↓
DbContext remembers the entity
```

This is called **Change Tracking**.

---

# 7. Simple Change Tracking example

Suppose the database contains:

```text
Users table

Id    Name     City
------------------------
1     John     Chennai
```

Query:

```csharp
var user = context.Users.First();
```

EF Core gives us:

```text
Id   = 1
Name = John
City = Chennai
```

The DbContext tracks this object.

Think:

```text
DbContext remembers:

Original:
Name = John
City = Chennai
```

---

# 8. Modify the entity

Now:

```csharp
user.City = "Bangalore";
```

The current object becomes:

```text
Id   = 1
Name = John
City = Bangalore
```

EF Core can compare:

```text
Original:
City = Chennai

Current:
City = Bangalore

        ↓

Something changed!
```

The entity's state becomes:

```text
Modified
```

---

# 9. `SaveChanges()`

When you call:

```csharp
context.SaveChanges();
```

EF Core looks at the tracked changes and sends the required SQL to the database.

For example:

```sql
UPDATE Users
SET City = 'Bangalore'
WHERE Id = 1;
```

Complete flow:

```text
1. Get entity
       ↓
2. DbContext tracks entity
       ↓
3. Change entity
       ↓
4. DbContext detects change
       ↓
5. SaveChanges()
       ↓
6. UPDATE database
```

---

# 10. Changing the C# object is NOT the same as updating the database

This is very important.

When you do:

```csharp
user.Name = "Robert";
```

you changed the C# object.

You have NOT necessarily updated the database yet.

Think:

```text
C# object
Name = Robert

Database
Name = John
```

Then:

```csharp
context.SaveChanges();
```

causes EF Core to send the change to the database.

```text
C# object
Robert
   |
   | SaveChanges()
   ↓
Database
Robert
```

Remember:

> **Changing the object and saving the database are two different things.**

---

# 11. What is EntityState?

`EntityState` tells EF Core what it thinks about an entity.

There are five important states:

```text
Added
Modified
Deleted
Unchanged
Detached
```

Think of the DbContext as a teacher watching students.

The teacher wants to know:

- Is this a new student?
- Did this student's information change?
- Should this student be removed?
- Did nothing change?
- Is the teacher not watching this student?

That is the purpose of EntityState.

---

# 12. `Unchanged`

### Meaning

> **The entity is being tracked, but nothing has changed.**

Example:

```csharp
var user = context.Users.First(x => x.Id == 1);
```

Initially:

```text
User
Name = John

State = Unchanged
```

Because:

```text
Database:
John

C# object:
John

Same → Nothing changed
```

Check the state:

```csharp
var state = context.Entry(user).State;

Console.WriteLine(state);
```

Output:

```text
Unchanged
```

### Easy way to remember

> **Unchanged = "I'm watching this entity, but nothing changed."**

`SaveChanges()` normally does nothing for an unchanged entity.

---

# 13. `Modified`

### Meaning

> **The entity already exists, but something has changed.**

Example:

```csharp
user.Name = "Robert";
```

Original:

```text
Name = John
```

Current:

```text
Name = Robert
```

EF Core changes the state to:

```text
Modified
```

Check:

```csharp
Console.WriteLine(context.Entry(user).State);
```

Output:

```text
Modified
```

Then:

```csharp
context.SaveChanges();
```

EF Core generates an `UPDATE`, similar to:

```sql
UPDATE Users
SET Name = 'Robert'
WHERE Id = 1;
```

### Easy way to remember

> **Modified = "This entity existed before, but somebody changed it."**

---

# 14. `Added`

### Meaning

> **This is a new entity that needs to be inserted into the database.**

Example:

```csharp
var user = new User
{
    Name = "Arun"
};

context.Users.Add(user);
```

State:

```text
Added
```

Then:

```csharp
context.SaveChanges();
```

EF Core generates an `INSERT`, similar to:

```sql
INSERT INTO Users (Name)
VALUES ('Arun');
```

### Easy way to remember

> **Added = "This is new and needs to be inserted."**

---

# 15. `Deleted`

### Meaning

> **The entity exists, but we want to remove it from the database.**

Example:

```csharp
var user = context.Users.First(x => x.Id == 1);

context.Users.Remove(user);
```

State:

```text
Deleted
```

Then:

```csharp
context.SaveChanges();
```

EF Core generates:

```sql
DELETE FROM Users
WHERE Id = 1;
```

### Easy way to remember

> **Deleted = "Remove this entity from the database."**

---

# 16. `Detached`

This one is slightly different.

### Meaning

> **The DbContext is not tracking the entity.**

Example:

```csharp
var user = new User
{
    Name = "Arun"
};
```

If you don't attach or add the object to the DbContext:

```csharp
context.Users.Add(user);
```

was never called.

The DbContext is not tracking it.

State:

```text
Detached
```

You can check:

```csharp
Console.WriteLine(context.Entry(user).State);
```

### Easy way to remember

> **Detached = "DbContext isn't watching this entity."**

Think:

```text
Teacher
  |
  +---- John     ← watching
  +---- David    ← watching
  |
  +---- Arun     ← NOT watching
```

---

# 17. All five EntityStates

| EntityState | Simple meaning | `SaveChanges()` usually results in |
|---|---|---|
| `Detached` | DbContext isn't tracking it | Nothing |
| `Unchanged` | Tracked, but nothing changed | Nothing |
| `Added` | New entity | `INSERT` |
| `Modified` | Existing entity was changed | `UPDATE` |
| `Deleted` | Existing entity should be removed | `DELETE` |

The most important mapping to memorize:

```text
Added     → INSERT
Modified  → UPDATE
Deleted   → DELETE
Unchanged → Nothing
Detached  → Not tracked
```

---

# 18. EntityState lifecycle

An entity can move from one state to another.

## Existing entity

When loaded:

```text
Database
   ↓
Load entity
   ↓
Unchanged
```

After changing it:

```text
Unchanged
    ↓
Change property
    ↓
Modified
```

After saving:

```text
Modified
    ↓
SaveChanges()
    ↓
Database updated
    ↓
Unchanged
```

Why does it become `Unchanged` again?

Because after saving:

```text
Database = Robert
C# object = Robert
```

There is no longer a difference.

---

# 19. Added lifecycle

```text
New C# object
     ↓
Add()
     ↓
Added
     ↓
SaveChanges()
     ↓
INSERT
     ↓
Unchanged
```

Example:

```csharp
var user = new User
{
    Name = "Arun"
};

context.Users.Add(user);

context.SaveChanges();
```

After a successful save, the new entity is generally considered `Unchanged`.

---

# 20. Deleted lifecycle

```text
Unchanged
    ↓
Remove()
    ↓
Deleted
    ↓
SaveChanges()
    ↓
DELETE
    ↓
No longer tracked
```

After deletion, the entity generally becomes `Detached` from that DbContext.

---

# 21. How to inspect EntityState

You can inspect one entity:

```csharp
var user = context.Users.First();

user.Name = "Robert";

var state = context.Entry(user).State;

Console.WriteLine(state);
```

Output:

```text
Modified
```

---

# 22. How to inspect all tracked entities

You can use `ChangeTracker`:

```csharp
var entries = context.ChangeTracker.Entries();

foreach (var entry in entries)
{
    Console.WriteLine(
        $"{entry.Entity.GetType().Name} - {entry.State}");
}
```

Possible output:

```text
User - Modified
Order - Unchanged
Product - Added
```

This is useful when debugging EF Core.

---

# 23. What is `ChangeTracker`?

`ChangeTracker` is the part of the DbContext that keeps information about tracked entities.

Think:

```text
DbContext
    |
    +---- ChangeTracker
              |
              +---- User 1 → Unchanged
              +---- User 2 → Modified
              +---- Order  → Added
```

It keeps track of:

- Which entities are being tracked
- Their EntityState
- Original/current values
- Changes that need to be saved

---

# 24. `AsNoTracking()`

Not every query needs Change Tracking.

Suppose you're displaying products on a website:

```text
Products page
```

You only need to read the products.

You are not going to modify them.

Normally:

```csharp
var products = context.Products.ToList();
```

EF Core tracks the returned entities.

For read-only queries, you can use:

```csharp
var products = context.Products
                      .AsNoTracking()
                      .ToList();
```

`AsNoTracking()` means:

> **"Give me the data, but don't track these entities."**

Conceptually:

```text
Normal query:

Database
   ↓
DbContext
   ↓
Product
   ↓
TRACKED
```

With `AsNoTracking()`:

```text
Database
   ↓
DbContext
   ↓
Product
   ↓
NOT TRACKED
```

---

# 25. When should you use `AsNoTracking()`?

## Read-only operation

Good use:

```csharp
var users = context.Users
                   .AsNoTracking()
                   .ToList();
```

For example:

- Product listing
- Reports
- Dashboard data
- Search results
- Read-only API endpoints

## Read + modify

Normal tracking is useful:

```csharp
var user = context.Users
                  .First(x => x.Id == id);

user.Name = "David";

context.SaveChanges();
```

Here EF Core needs to track the entity so it can detect and save the change.

---

# 26. DbContext Lifetime + Change Tracking + EntityState

Now connect everything.

```text
                 DbContext
                     |
          +----------+----------+
          |                     |
          ↓                     ↓
   ChangeTracker          Database access
          |
          ↓
     EntityState
          |
    +-----+-----+---------+----------+
    |           |         |          |
    ↓           ↓         ↓          ↓
 Unchanged   Modified   Added     Deleted
                |         |          |
                ↓         ↓          ↓
              UPDATE    INSERT     DELETE
                \         |         /
                 \        |        /
                  ↓       ↓       ↓
                    SaveChanges()
                         |
                         ↓
                      Database
```

And DbContext lifetime controls how long this tracking information lives.

```text
HTTP Request
     ↓
DbContext created
     ↓
Entities tracked
     ↓
Changes made
     ↓
SaveChanges()
     ↓
Request ends
     ↓
DbContext disposed
     ↓
Tracked state goes away
```

---

# 27. Easy real-world analogy

Imagine a teacher and students.

### DbContext

The **teacher**.

### Entity

A **student**.

### ChangeTracker

The teacher's **notebook**.

### EntityState

The label the teacher puts next to each student.

```text
Student       State
-------------------------
John          Unchanged
David         Modified
Arun          Added
Ravi          Deleted
Kumar         Detached
```

### SaveChanges()

The teacher says:

> "Okay, apply all the changes."

EF Core then performs:

```text
Added     → INSERT
Modified  → UPDATE
Deleted   → DELETE
Unchanged → Nothing
Detached  → Nothing
```

---

# 28. Strong interview answer

### Question:

**What is Change Tracking in EF Core?**

### Answer:

> Change Tracking is the mechanism used by EF Core's `DbContext` to keep track of entities and their state. When we query an entity, EF Core normally tracks it. If we modify the entity, the ChangeTracker detects the change and the entity becomes `Modified`. When `SaveChanges()` is called, EF Core generates the required `INSERT`, `UPDATE`, or `DELETE` SQL based on the entity states.

For read-only queries:

> We can use `AsNoTracking()` to avoid tracking overhead and improve performance.

---

# 29. Strong interview answer for EntityState

### Question:

**What are the different EntityState values in EF Core?**

### Answer:

> EF Core has five main EntityState values: `Added`, `Modified`, `Deleted`, `Unchanged`, and `Detached`.
>
> - `Added` means the entity is new and will normally result in an `INSERT`.
> - `Modified` means an existing entity has changed and will normally result in an `UPDATE`.
> - `Deleted` means the entity is marked for deletion and will normally result in a `DELETE`.
> - `Unchanged` means the entity is tracked but has no changes.
> - `Detached` means the DbContext is not tracking the entity.

---

# 30. Important interview question: Why is DbContext usually Scoped?

A good answer:

> `DbContext` is normally registered as Scoped in ASP.NET Core, meaning one DbContext instance is normally used within one HTTP request. This fits the Unit of Work pattern because related database operations can be performed and saved together. DbContext is also not thread-safe, so we should not normally share one DbContext instance across multiple concurrent requests or make it Singleton.

---

# 31. Final cheat sheet

```text
DbContext
    ↓
Short-lived workspace for database operations
    ↓
Usually Scoped in ASP.NET Core
    ↓
Tracks entities
    ↓
ChangeTracker
    ↓
EntityState
```

### EntityState

```text
Added
   ↓
INSERT

Modified
   ↓
UPDATE

Deleted
   ↓
DELETE

Unchanged
   ↓
Nothing

Detached
   ↓
Not tracked
```

### Change Tracking

```text
Load entity
    ↓
Unchanged
    ↓
Change property
    ↓
Modified
    ↓
SaveChanges()
    ↓
UPDATE
    ↓
Unchanged
```

### Read-only query

```csharp
context.Users
       .AsNoTracking()
       .ToList();
```

### Most important rule

> **DbContext is a short-lived Unit of Work. It tracks entities, assigns EntityState, detects changes, and `SaveChanges()` uses those states to synchronize the database.**
