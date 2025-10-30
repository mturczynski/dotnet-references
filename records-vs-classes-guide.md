# Rekordy (record) w C#

## Wprowadzenie
**Record** (C# 9.0+) to referencyjny typ danych zaprojektowany z myślą o **value-based semantics** zamiast identity-based. Rekordy oferują skróconą składnię, immutability (domyślnie), wbudowane value equality i copy semantics (`with`), będąc idealnym wyborem dla DTOs, domain models i value objects.

---

## 1. Podstawowa składnia i różnice

### Record vs Class - definicja
```csharp
// Positional record (C# 9.0+)
public record Person(string FirstName, string LastName, int Age);

// Odpowiednik klasy (verbose)
public class PersonClass
{
    public string FirstName { get; init; }
    public string LastName { get; init; }
    public int Age { get; init; }
    
    public PersonClass(string firstName, string lastName, int age)
    {
        FirstName = firstName;
        LastName = lastName;
        Age = age;
    }
    
    // Value equality (manualne)
    public override bool Equals(object obj) =>
        obj is PersonClass other &&
        FirstName == other.FirstName &&
        LastName == other.LastName &&
        Age == other.Age;
    
    public override int GetHashCode() =>
        HashCode.Combine(FirstName, LastName, Age);
    
    public override string ToString() =>
        $"PersonClass {{ FirstName = {FirstName}, LastName = {LastName}, Age = {Age} }}";
}

// Record automatycznie generuje:
// - Primary constructor
// - init-only properties
// - Value-based Equals/GetHashCode
// - ToString() z property display
// - Deconstruction
// - Clone method (protected)
```

---

## 2. Kluczowe różnice vs klasy

| Aspekt | Class | Record |
|--------|-------|--------|
| **Equality** | Reference equality (domyślnie) | Value equality (automatycznie) |
| **ToString()** | `Type.ToString()` | Sformatowane z properties |
| **Immutability** | Mutable (domyślnie) | init-only properties |
| **Copy semantics** | Brak | `with` expression |
| **Deconstruction** | Manualne | Automatyczne |
| **Inheritance** | Tak | Tak (record może dziedziczyć record) |
| **Type** | Reference type | Reference type (domyślnie) |

---

## 3. Value equality - kluczowa różnica

### Reference equality (class)
```csharp
var person1 = new PersonClass("John", "Doe", 30);
var person2 = new PersonClass("John", "Doe", 30);

Console.WriteLine(person1 == person2);        // False (różne referencje)
Console.WriteLine(person1.Equals(person2));   // False (bez override)
```

### Value equality (record)
```csharp
var person1 = new Person("John", "Doe", 30);
var person2 = new Person("John", "Doe", 30);

Console.WriteLine(person1 == person2);        // True (value equality)
Console.WriteLine(person1.Equals(person2));   // True
Console.WriteLine(person1.GetHashCode() == person2.GetHashCode()); // True
```

### Wygenerowane Equals implementation
```csharp
// Kompilator generuje coś podobnego do:
public virtual bool Equals(Person other)
{
    return other != null &&
           EqualityContract == other.EqualityContract &&
           FirstName == other.FirstName &&
           LastName == other.LastName &&
           Age == other.Age;
}

protected virtual Type EqualityContract => typeof(Person);
```

---

## 4. with Expression - non-destructive mutation

### Problem z klasami
```csharp
var person = new PersonClass("John", "Doe", 30);
// Aby zmienić Age, trzeba stworzyć nową instancję ręcznie
var older = new PersonClass(person.FirstName, person.LastName, 31);
```

### with expression w recordach
```csharp
var person = new Person("John", "Doe", 30);
var older = person with { Age = 31 };  // Kopia z modyfikacją

Console.WriteLine(person);  // Person { FirstName = John, LastName = Doe, Age = 30 }
Console.WriteLine(older);   // Person { FirstName = John, LastName = Doe, Age = 31 }
```

### Wygenerowana clone metoda
```csharp
// Kompilator generuje protected copy constructor
protected Person(Person original)
{
    FirstName = original.FirstName;
    LastName = original.LastName;
    Age = original.Age;
}

// with używa tego konstruktora + object initializer
```

---

## 5. Deconstruction

### Automatyczne dla positional records
```csharp
var person = new Person("John", "Doe", 30);

// Deconstruction
var (firstName, lastName, age) = person;
Console.WriteLine($"{firstName} {lastName} is {age}"); // John Doe is 30

// Discard niepotrzebne wartości
var (first, _, _) = person;
```

### Wygenerowana metoda Deconstruct
```csharp
// Kompilator generuje:
public void Deconstruct(out string FirstName, out string LastName, out int Age)
{
    FirstName = this.FirstName;
    LastName = this.LastName;
    Age = this.Age;
}
```

---

## 6. Record struct (C# 10.0+)

### Value type records
```csharp
// Record struct - value type z value semantics
public record struct Point(int X, int Y);

// vs regular struct
public struct PointStruct
{
    public int X { get; init; }
    public int Y { get; init; }
    
    public PointStruct(int x, int y) => (X, Y) = (x, y);
    
    // Manualne Equals, GetHashCode, ToString
}

// Użycie
Point p1 = new(10, 20);
Point p2 = p1 with { X = 15 };  // with expression działa!

Console.WriteLine(p1);  // Point { X = 10, Y = 20 }
Console.WriteLine(p2);  // Point { X = 15, Y = 20 }
```

### readonly record struct
```csharp
// readonly record struct - guaranteed immutability
public readonly record struct ImmutablePoint(int X, int Y);

// Wszystkie properties są readonly
// Kompilator wymusza immutability
```

---

## 7. Inheritance i polymorphism

### Record inheritance
```csharp
// Base record
public record Person(string FirstName, string LastName);

// Derived record
public record Employee(string FirstName, string LastName, string EmployeeId) 
    : Person(FirstName, LastName);

// Usage
var employee = new Employee("John", "Doe", "EMP001");
Console.WriteLine(employee);  
// Employee { FirstName = John, LastName = Doe, EmployeeId = EMP001 }

// Polymorphic equality
Person person = new Person("John", "Doe");
Employee emp = new Employee("John", "Doe", "EMP001");
Console.WriteLine(person == emp);  // False - różne typy (EqualityContract)
```

### Sealed records (best practice)
```csharp
// Zalecane dla performance i predictability
public sealed record Person(string FirstName, string LastName, int Age);

// Kompilator może zoptymalizować Equals (brak virtual dispatch)
```

---

## 8. Zaawansowane scenariusze

### Custom validation
```csharp
public record Person(string FirstName, string LastName, int Age)
{
    // Custom constructor z walidacją
    public Person(string FirstName, string LastName, int Age) : this()
    {
        if (string.IsNullOrWhiteSpace(FirstName))
            throw new ArgumentException("FirstName cannot be empty");
        if (Age < 0)
            throw new ArgumentException("Age must be positive");
            
        this.FirstName = FirstName;
        this.LastName = LastName;
        this.Age = Age;
    }
}
```

### Additional properties
```csharp
public record Person(string FirstName, string LastName, int Age)
{
    // Additional computed property
    public string FullName => $"{FirstName} {LastName}";
    
    // Additional mutable property (not recommended)
    public List<string> Nicknames { get; set; } = new();
    
    // Additional init-only property
    public DateTime BirthDate { get; init; }
}
```

### Explicit property definitions
```csharp
public record Person(string FirstName, string LastName)
{
    // Override positional parameter property
    private string _firstName = FirstName;
    public string FirstName 
    { 
        get => _firstName;
        init => _firstName = value?.Trim() ?? throw new ArgumentNullException();
    }
    
    // LastName remains auto-generated
}
```

---

## 9. Praktyczne zastosowania

### DTOs i API responses
```csharp
// Clean API contracts
public record UserDto(int Id, string Username, string Email, DateTime CreatedAt);

public record ApiResponse<T>(bool Success, T Data, string Message);

// Usage
var response = new ApiResponse<UserDto>(
    true,
    new UserDto(1, "john_doe", "john@example.com", DateTime.UtcNow),
    "User retrieved successfully"
);
```

### Domain events
```csharp
public abstract record DomainEvent(Guid EventId, DateTime OccurredAt);

public record UserRegisteredEvent(
    Guid EventId, 
    DateTime OccurredAt,
    int UserId,
    string Email
) : DomainEvent(EventId, OccurredAt);

public record OrderPlacedEvent(
    Guid EventId,
    DateTime OccurredAt, 
    int OrderId,
    decimal TotalAmount
) : DomainEvent(EventId, OccurredAt);
```

### Value Objects (DDD)
```csharp
public record Money(decimal Amount, string Currency)
{
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Currency mismatch");
        return this with { Amount = Amount + other.Amount };
    }
    
    public static Money operator +(Money left, Money right) => left.Add(right);
}

public record Address(
    string Street,
    string City,
    string PostalCode,
    string Country
);
```

### Configuration objects
```csharp
public record DatabaseConfig(
    string ConnectionString,
    int MaxPoolSize,
    int CommandTimeout
);

public record AppSettings(
    DatabaseConfig Database,
    string ApiKey,
    bool EnableLogging
);
```

---

## 10. Performance considerations

### Memory allocation
```csharp
// Record classes - heap allocation (reference type)
var person = new Person("John", "Doe", 30);  // Heap

// Record structs - stack allocation (value type)
var point = new Point(10, 20);  // Stack (if local variable)
```

### Equality comparison overhead
```csharp
// Records have automatic value equality
// Performance cost: każde pole musi być porównane

// Benchmark (przykładowe)
// Record with 5 properties: ~15-20ns per comparison
// Class with reference equality: ~2-3ns
// Sealed record optimization: ~10-12ns

// Użyj sealed dla lepszej performance
public sealed record Person(string FirstName, string LastName, int Age);
```

---

## 11. Best Practices

### ✅ Kiedy używać records

1. **DTOs i API contracts** - immutability i value equality out of the box
2. **Value objects** w DDD - semantyka wartości zamiast tożsamości
3. **Domain events** - immutable snapshots stanów
4. **Configuration** - read-only settings
5. **Temporary data structures** - parsing results, query results

### ✅ Recommended patterns
```csharp
// 1. Sealed by default dla performance
public sealed record UserDto(int Id, string Username);

// 2. Validation w konstruktorze
public record Email
{
    public string Value { get; }
    
    public Email(string value)
    {
        if (!IsValidEmail(value))
            throw new ArgumentException("Invalid email");
        Value = value;
    }
    
    private static bool IsValidEmail(string email) => /* validation */;
}

// 3. Immutability - unikaj mutable properties
public record Person(string Name, int Age)
{
    // ❌ Avoid
    // public List<string> Tags { get; set; } = new();
    
    // ✅ Better
    public IReadOnlyList<string> Tags { get; init; } = Array.Empty<string>();
}
```

### ❌ Anti-patterns

```csharp
// Don't: Duże obiekty jako records (overhead Equals/GetHashCode)
public record HugeDataRecord(
    string Field1, string Field2, /* ... */ string Field100
); // 100 fields - slow equality checks

// Don't: Mutable collections jako properties
public record BadRecord(string Name)
{
    public List<int> MutableList { get; set; } = new(); // Breaks immutability
}

// Don't: Business logic w recordach (używaj klas dla entity)
public record Entity(int Id, string Name)
{
    // ❌ Complex business logic doesn't belong here
    public void ComplexOperation() { /* ... */ }
}
```

---

## 12. Migracja z klas do records

### Before (class)
```csharp
public class PersonClass
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public int Age { get; set; }
}
```

### After (record)
```csharp
public record Person(string FirstName, string LastName, int Age);

// Lub jeśli potrzebujesz mutability:
public record Person
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public int Age { get; set; }
}
```

---

## Podsumowanie

**Records** w C# oferują:

- **Value-based equality** - automatyczne porównanie przez wartość
- **Immutability** - init-only properties domyślnie
- **with expression** - non-destructive mutation
- **Automatic ToString()** - czytelny output
- **Deconstruction** - wygodne rozpakowanie
- **Concise syntax** - minimalna boilerplate

**Kluczowe różnice vs klasy:**
- Semantyka wartości zamiast referencji
- Domyślna immutability
- Skrócona składnia
- Reference type (domyślnie) z value semantics

**Developer .NET powinien:**
- Używać records dla DTOs, value objects, domain events
- Preferować sealed records dla performance
- Zachować immutability (unikać mutable properties)
- Rozważyć record struct dla małych value types
- Nie nadużywać dla entity z bogatą logiką biznesową

Records są idealnym narzędziem dla functional-style programming i domain modeling w C#, redukując boilerplate przy zachowaniu type safety i czytelności.