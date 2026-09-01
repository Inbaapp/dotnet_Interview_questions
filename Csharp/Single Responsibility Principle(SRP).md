# Solid Principles - Single Responsibility Priniciple Interview Questions

### Q1: What is Single Responsibility Priniciple?

**Answer:** A Class should have only one reason to change.

     in simple word
        - one class = one Responsibility
        - Don't put Multiple responsibility in the same class

## 👎Bad Example (Violates SRP)

**Example:**

```csharp
 
 public class Invoiceservice
 {
    public void CalculateTotal(){
        Console.WriteLine("Calculating invoice total..");
    }

    public void SaveToDatabase(){
        Console.WriteLine("Saving invoice to Database..");
    }

    public void SendEmail(){
        Console.WriteLine("Sending Invoice email..");
    }
 }

 class program
 {
    public static void main(){

        var obj = new Invoiceservice();
        obj.CalculateTotal();
        obj.SaveToDatabase();
        obj.SendEmail();

    }
 }

```

**Problem:**

    -- Tax calculation changes
    -- Database changes 
    -- Email format chnages

you must modify the same class. This violates SRP

## 👍Good Example (Violates SRP)

**Example:**

```csharp

public class InvoiceCalculator
{
    public void CalculateTotal(){
        Console.WriteLine("Calculating invoice total..");
    }
}

public class InvoiceRepository
{
    public void SaveToDatabase(){
        Console.WriteLine("Saving invoice to Database..");
    }
}

public class EmailService
{
    public void SendEmail(){
        Console.WriteLine("Sending Invoice email..");
    }
}

class program
{
    public static void main()
    {
       var calculator = new InvoiceCalculator();
       var repository = new InvoiceRepository();
       var emailservice = new EmailService();

        calculator.CalculateTotal();
        repository.SaveToDatabase();
        emailservice.EmailService();
    }
}

```

**Benefits**

    - Easier maintenance
    - Easier testing
    - Better Readability
    - Less Imapct when Requiremts changes
    - Resuable components

**Easy way to remember**

one class = one job = one Reason to change


✅ In short:    *SRP in .NET ensures that each class has a single responsibility and only one reason to change, making your codebase cleaner, easier to test, and more maintainable.*

🔑 Why SRP Matters in .NET
- Maintainability: Changes in one responsibility don’t break unrelated functionality.

- Testability: Easier to write unit tests since each class has a clear purpose.

- Cohesion: Classes remain tightly focused on a single concern.

- Loose Coupling: Reduces dependencies between unrelated parts of the system.

- Scalability: Supports Clean Architecture and microservices design.


📌 Best Practices for SRP in .NET
- Keep classes focused on one business responsibility.

- Separate business logic, data access, logging, and notifications.

- Use Dependency Injection for external services.

- Refactor when a class starts gaining multiple reasons to change.

- Balance SRP—avoid over-splitting into too many tiny classes