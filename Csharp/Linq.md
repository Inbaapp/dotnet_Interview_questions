# LINQ Interview Questions

### Q1: What is LINQ?
**Answer:** LINQ (Language Integrated Query) is a set of features in C# that provides query capabilities directly in the language syntax.

**Example:**
```csharp
var numbers = new List<int> { 1, 2, 3, 4, 5 };
var evenNumbers = from n in numbers
                  where n % 2 == 0
                  select n;
