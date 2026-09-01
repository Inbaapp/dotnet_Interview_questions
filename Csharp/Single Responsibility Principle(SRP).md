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

 Class program
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
