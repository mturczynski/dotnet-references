# Typy generyczne w C# - Constraints, Kowariancja i Kontrawariancja

## Wprowadzenie
**Typy generyczne** w C# umożliwiają definiowanie klas, interfejsów i metod z parametrami typu, które są określane dopiero w momencie użycia. Zapewniają **type safety** w compile-time przy zachowaniu **reusability** kodu. Zaawansowane mechanizmy jak constraints i variance dodatkowo zwiększają ich możliwości.

---

## 1. Podstawy typów generycznych

### Definicja i składnia
```csharp
// Generic class
public class Repository<T> where T : class
{
    private readonly List<T> _items = new();
    
    public void Add(T item) => _items.Add(item);
    public T GetById(int id) => _items[id];
    public IEnumerable<T> GetAll() => _items.AsReadOnly();
}

// Generic method
public static T CloneObject<T>(T source) where T : ICloneable
{
    return (T)source.Clone();
}

// Multiple type parameters
public interface IMapper<TSource, TDestination>
{
    TDestination Map(TSource source);
    IEnumerable<TDestination> Map(IEnumerable<TSource> sources);
}
```

### Korzyści
- **Type Safety** - błędy w compile-time zamiast runtime
- **Performance** - brak boxing/unboxing dla value types
- **Code Reusability** - jeden kod dla wielu typów
- **IntelliSense** - pełne wsparcie IDE

---

## 2. Generic Constraints - ograniczenia typów

### Rodzaje constraints

| Constraint | Opis | Przykład |
|------------|------|----------|
| `where T : class` | Typ referencyjny | `Repository<T> where T : class` |
| `where T : struct` | Typ wartościowy | `Nullable<T> where T : struct` |
| `where T : new()` | Konstruktor bezparametrowy | `Factory<T> where T : new()` |
| `where T : BaseClass` | Dziedziczenie z klasy | `where T : Entity` |
| `where T : IInterface` | Implementacja interfejsu | `where T : IComparable<T>` |
| `where T : U` | Relacja między parametrami | `where T : U` |

### Praktyczne przykłady
```csharp
// Multiple constraints
public class EntityRepository<T> 
    where T : class, IEntity, new()
{
    public T Create()
    {
        var entity = new T();  // new() constraint
        entity.Id = Guid.NewGuid();  // IEntity constraint
        return entity;  // class constraint
    }
}

// Constraint inheritance
public abstract class BaseService<T> where T : IEntity
{
    protected abstract void Process(T entity);
}

public class UserService : BaseService<User>  // User must implement IEntity
{
    protected override void Process(User user)
    {
        // Implementation
    }
}

// Complex constraints
public class SortedCollection<T> where T : IComparable<T>, IEquatable<T>
{
    private readonly SortedSet<T> _items = new();
    
    public void Add(T item)
    {
        _items.Add(item);  // Uses IComparable<T>
    }
    
    public bool Contains(T item)
    {
        return _items.Contains(item);  // Uses IEquatable<T>
    }
}
```

---

## 3. Variance - Kowariancja i Kontrawariancja

### Podstawowe pojęcia
- **Invariance** (domyślnie) - `Generic<Base>` i `Generic<Derived>` to różne typy
- **Covariance** (`out`) - `Generic<Derived>` można przypisać do `Generic<Base>`
- **Contravariance** (`in`) - `Generic<Base>` można przypisać do `Generic<Derived>`

### Covariance - `out` keyword

**Zasada**: Parametr typu używany tylko jako **output** (return type).

```csharp
// IEnumerable<T> jest covariant
public interface IEnumerable<out T>
{
    IEnumerator<T> GetEnumerator();
}

// Praktyczny przykład
IEnumerable<string> strings = new List<string> { "hello", "world" };
IEnumerable<object> objects = strings;  // OK - covariance

// Custom covariant interface
public interface IProducer<out T>
{
    T Produce();
    IEnumerable<T> ProduceMany();
    // T Consume(T item);  // ERROR - T in input position
}

public class StringProducer : IProducer<string>
{
    public string Produce() => "Hello";
    public IEnumerable<string> ProduceMany() => new[] { "A", "B", "C" };
}

// Usage
IProducer<string> stringProducer = new StringProducer();
IProducer<object> objectProducer = stringProducer;  // OK - covariance
object result = objectProducer.Produce();  // Returns string as object
```

### Contravariance - `in` keyword

**Zasada**: Parametr typu używany tylko jako **input** (parameter type).

```csharp
// Action<T> jest contravariant
public delegate void Action<in T>(T obj);

// Praktyczny przykład
Action<object> objectHandler = obj => Console.WriteLine(obj.ToString());
Action<string> stringHandler = objectHandler;  // OK - contravariance
stringHandler("Hello");  // Calls objectHandler with string

// Custom contravariant interface
public interface IConsumer<in T>
{
    void Consume(T item);
    void ConsumeMany(IEnumerable<T> items);
    // T Produce();  // ERROR - T in output position
}

public class ObjectConsumer : IConsumer<object>
{
    public void Consume(object item) => Console.WriteLine($"Processing: {item}");
    public void ConsumeMany(IEnumerable<object> items) => 
        items.ToList().ForEach(Consume);
}

// Usage
IConsumer<object> objectConsumer = new ObjectConsumer();
IConsumer<string> stringConsumer = objectConsumer;  // OK - contravariance
stringConsumer.Consume("Hello");  // Passes string to object consumer
```

### Kombinowanie variance
```csharp
// Func<in T, out TResult> - contravariant in T, covariant in TResult
public delegate TResult Func<in T, out TResult>(T arg);

Func<object, string> objectToString = obj => obj.ToString();
Func<string, object> stringToObject = objectToString;  // OK - beide variance
```

---

## 4. Zaawansowane wzorce i scenariusze

### Generic Method Type Inference
```csharp
public static class Extensions
{
    // Compiler infers T from usage
    public static T GetValueOrDefault<T>(this Dictionary<string, object> dict, 
        string key, T defaultValue = default(T))
    {
        return dict.TryGetValue(key, out var value) && value is T t 
            ? t 
            : defaultValue;
    }
}

// Usage - T inferred as int
var dict = new Dictionary<string, object> { ["count"] = 42 };
int count = dict.GetValueOrDefault("count", 0);  // T = int inferred
```

### Self-referencing generics
```csharp
// Fluent interface pattern
public abstract class FluentBuilder<T> where T : FluentBuilder<T>
{
    protected abstract T This { get; }
    
    public T WithName(string name)
    {
        // Set name
        return This;
    }
    
    public T WithDescription(string description)
    {
        // Set description  
        return This;
    }
}

public class UserBuilder : FluentBuilder<UserBuilder>
{
    protected override UserBuilder This => this;
    
    public UserBuilder WithEmail(string email)
    {
        // Set email
        return this;
    }
    
    public User Build() => new User(/* parameters */);
}

// Usage with full fluent chaining
var user = new UserBuilder()
    .WithName("John")
    .WithDescription("User")
    .WithEmail("john@example.com")  // UserBuilder-specific method
    .Build();
```

### Generic Delegates i Events
```csharp
public class EventPublisher<T>
{
    public event Action<T> ItemAdded;
    public event Func<T, bool> ItemValidating;
    
    public void AddItem(T item)
    {
        var isValid = ItemValidating?.Invoke(item) ?? true;
        if (isValid)
        {
            ItemAdded?.Invoke(item);
        }
    }
}

// Usage with variance
EventPublisher<string> stringPublisher = new();
stringPublisher.ItemAdded += (object obj) => Console.WriteLine(obj);  // Contravariance
```

---

## 5. Performance i IL Code

### Boxing avoidance
```csharp
// Before generics - boxing for value types
ArrayList list = new ArrayList();
list.Add(42);  // Boxing int to object
int value = (int)list[0];  // Unboxing

// With generics - no boxing
List<int> genericList = new List<int>();
genericList.Add(42);  // No boxing
int genericValue = genericList[0];  // No unboxing
```

### JIT compilation
- **Generic type** = jeden IL kod dla wszystkich typów referencyjnych
- **Value types** = osobne native code dla każdego typu
- **Shared generics** dla reference types, **specialized** dla value types

---

## 6. Best Practices i Anti-patterns

### ✅ Best Practices
```csharp
// Meaningful constraint names
public interface IRepository<TEntity> where TEntity : class, IEntity
{
    Task<TEntity> GetByIdAsync(Guid id);
}

// Prefer generic methods over generic overloads
public static T ConvertTo<T>(object value) where T : IConvertible
{
    return (T)Convert.ChangeType(value, typeof(T));
}

// Use variance appropriately
public interface IEventHandler<in TEvent> where TEvent : IEvent
{
    Task HandleAsync(TEvent eventArgs);
}
```

### ❌ Anti-patterns
```csharp
// Too many type parameters
public class BadService<T1, T2, T3, T4, T5> { }  // Avoid

// Unnecessary constraints
public void Process<T>(T item) where T : class  // Remove if not used

// Breaking variance rules
public interface IBadVariance<out T>
{
    void BadMethod(T input);  // ERROR - input with 'out'
}
```

---

## 7. .NET Framework vs .NET Core/5+ różnice

### Nullable Reference Types (C# 8.0+)
```csharp
// C# 8.0+ with nullable reference types
public class Repository<T> where T : class?  // T can be nullable reference
{
    public T? FindOptional(int id) => default(T);
    public T GetRequired(int id) => FindOptional(id) ?? throw new NotFoundException();
}
```

### Generic Attributes (C# 11+)
```csharp
// Generic attributes
public class GenericAttribute<T> : Attribute
{
    public T Value { get; }
    public GenericAttribute(T value) => Value = value;
}

[Generic<string>("Hello")]
[Generic<int>(42)]
public class AttributeTarget { }
```

---

## Podsumowanie

**Typy generyczne** w C# to potężny mechanizm zapewniający:

- **Type Safety** bez utraty **Performance**
- **Code Reusability** z zachowaniem specificity przez **Constraints**
- **Flexibility** w projektowaniu API przez **Variance**

Developer .NET powinien:
- Rozumieć mechanizmy variance i ich ograniczenia
- Skutecznie używać constraints dla type safety
- Projektować API z myślą o variance gdzie to możliwe
- Znać performance implications generics vs non-generic code

Generics to fundament nowoczesnego C# - od `List<T>` przez LINQ po dependency injection w ASP.NET Core.