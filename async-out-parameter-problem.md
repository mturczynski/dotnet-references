# Async Method z out parametrem – Problem i Rozwiązania

## Wprowadzenie
Kod `public async Task<string> Foo(int id, out int userId)` jest **kompilowanym błędem**. C# nie pozwala na kombinację `async` metod z parametrami `out` (ani `ref`). To ograniczenie wynika z fundamentalnego konfliktu między semantyką `async/await` a semantyką `ref`/`out` parametrów.

---

## 1. Dlaczego to nie działa?

### Błąd kompilacji
```csharp
public async Task<string> Foo(int id, out int userId)
{
    // CS1988: Async methods cannot have out or ref parameters
}
```

### Przyczyny techniczne

#### State machine i async dispatch
```csharp
// Async metoda jest transformowana do state machine:
public class FooStateMachine
{
    private int _state = 0;
    private TaskAwaiter _awaiter;
    
    // out parametr byłby polem state machine?
    private int _userId;  // Gdzie go przechować?
    
    public void MoveNext()
    {
        switch (_state)
        {
            case 0:
                // Problem: out _userId - parametr musi być "ref"
                // do kontekstu callera, ale state machine nie ma dostępu
                break;
        }
    }
}
```

#### Stack frame problem
- **out parametr** wymaga referencji do zmiennej callera na jego stosie
- **async method** zawieszające się (await) zwalnia swój stack frame
- Gdy async wznawia, oryginalny stack frame callera już nie istnieje
- out parametr byłby wskaźnikiem na nieistniejącą pamięć (dangling pointer)

```csharp
// Pseudokod pokazujący problem:
async Task Foo(out int userId)
{
    // Stack frame A (caller context)
    int result = await GetValueAsync();  // Stack frame A zostaje zwalniona
    
    // Po await - stack frame A już nie istnieje!
    userId = result;  // Gdzie teraz jest userId?
    // BŁĄD - dangling pointer do caller's stack
}
```

---

## 2. Semantyka out vs async

### out – immediate write
```csharp
// out wymagany do write PRZED return
void GetUser(out int userId)
{
    userId = 42;  // Musi być przypisany
    // WYMAGANE: każda gałąź kodu musi przypisać
}

// Использование:
GetUser(out int id);  // id = 42 natychmiast dostępny
```

### async – deferred execution
```csharp
// async zawieszająca się operacja
async Task<int> GetUserAsync()
{
    var result = await FetchFromDatabaseAsync();
    return result;
}

// Użycie:
var task = GetUserAsync();  // Nie blokuje - zwraca Task
int id = await task;        // Czeka na rezultat
```

### Konflikt
```csharp
// Musiałoby być coś takiego (niemożliwe):
async Task GetUserAsync(out int userId)
{
    await SomethingAsync();  // Stack frame zwolniony
    userId = 42;             // Gdzie przypisać? Stack już nie istnieje!
}

// Calling:
var task = GetUserAsync(out int id);  // id co teraz zawiera?
// Jest undefined - metoda nie zakończyła się!
```

---

## 3. Dlaczego `ref` też nie działa?

```csharp
public async Task<string> Foo(int id, ref int userId)
{
    // CS1988: Async methods cannot have out or ref parameters
}
```

### Podobne problemy z `ref`
- **ref** wymaga stałej referencji do zmiennej w caller's context
- **async** zawieszająca się zmienia stack frame
- ref byłby dangling pointer po await
- ref byłby invalid w state machine

---

## 4. Rozwiązania

### ✅ Rozwiązanie 1: Zwróć wartość w Task<T>

**Problem:**
```csharp
// Nie działa:
async Task<string> Foo(int id, out int userId)
{
    userId = 42;
    return "result";
}
```

**Rozwiązanie:**
```csharp
public async Task<(string result, int userId)> FooAsync(int id)
{
    var userId = await GetUserIdAsync(id);
    return ("result", userId);
}

// Użycie:
var (result, userId) = await FooAsync(123);
```

### ✅ Rozwiązanie 2: Tuple return

```csharp
public async Task<(string result, int userId, string additionalData)> FooAsync(int id)
{
    var userData = await FetchUserAsync(id);
    return (userData.Name, userData.Id, userData.Email);
}

// Użycie
var (name, id, email) = await FooAsync(123);
```

### ✅ Rozwiązanie 3: Custom DTO/Record

```csharp
public record FooResult(string Result, int UserId, DateTime CreatedAt);

public async Task<FooResult> FooAsync(int id)
{
    var user = await FetchUserAsync(id);
    return new FooResult(
        Result: "Processed",
        UserId: user.Id,
        CreatedAt: DateTime.UtcNow
    );
}

// Użycie
var result = await FooAsync(123);
Console.WriteLine($"{result.Result}: {result.UserId}");
```

### ✅ Rozwiązanie 4: Callback/Action

```csharp
public async Task<string> FooAsync(int id, Action<int> onUserId)
{
    var userId = await GetUserIdAsync(id);
    onUserId?.Invoke(userId);  // Callback
    return "result";
}

// Użycie
int capturedUserId = 0;
var result = await FooAsync(123, userId => capturedUserId = userId);
```

### ✅ Rozwiązanie 5: Property/State object

```csharp
public class FooContext
{
    public int UserId { get; set; }
    public string Result { get; set; }
}

public async Task<FooContext> FooAsync(int id, FooContext context)
{
    context.UserId = await GetUserIdAsync(id);
    context.Result = "processed";
    return context;
}

// Użycie
var context = new FooContext();
var result = await FooAsync(123, context);
Console.WriteLine(context.UserId);
```

---

## 5. Porównanie rozwiązań

| Rozwiązanie | Pros | Cons |
|------------|------|------|
| **Tuple** | Szybkie, nowoczesne | Ograniczone do 8 elementów (C# 7.0) |
| **Record** | Type-safe, Named properties | Extra allocacja |
| **Callback** | Elastyczne | Complexity, hard to read |
| **Property object** | Reusable | Mutable, side effects |
| **Task<T>** | Simple | Tylko jedna wartość |

---

## 6. Real-world example

### ❌ Błędny kod
```csharp
public async Task<User> FetchUserAsync(int id, out string cacheKey)
{
    cacheKey = GenerateCacheKey(id);  // CS1988 error
    return await _database.GetUserAsync(id);
}
```

### ✅ Poprawny kod (multiple approaches)

**Approach 1: Tuple**
```csharp
public async Task<(User user, string cacheKey)> FetchUserAsync(int id)
{
    var cacheKey = GenerateCacheKey(id);
    var user = await _database.GetUserAsync(id);
    return (user, cacheKey);
}

// Użycie
var (user, cacheKey) = await FetchUserAsync(123);
await _cache.SetAsync(cacheKey, user);
```

**Approach 2: Record**
```csharp
public record UserFetchResult(User User, string CacheKey);

public async Task<UserFetchResult> FetchUserAsync(int id)
{
    var cacheKey = GenerateCacheKey(id);
    var user = await _database.GetUserAsync(id);
    return new UserFetchResult(user, cacheKey);
}

// Użycie
var result = await FetchUserAsync(123);
await _cache.SetAsync(result.CacheKey, result.User);
```

**Approach 3: Encapsulated in class**
```csharp
public class UserService
{
    public User User { get; private set; }
    public string CacheKey { get; private set; }
    
    public async Task FetchUserAsync(int id)
    {
        CacheKey = GenerateCacheKey(id);
        User = await _database.GetUserAsync(id);
    }
}

// Użycie
var service = new UserService();
await service.FetchUserAsync(123);
var cacheKey = service.CacheKey;
```

---

## 7. Best practices dla async multi-value returns

### ✅ Kiedy zwrócić tuple
```csharp
// Proste, few properties, ad-hoc
public async Task<(string name, int age)> GetPersonAsync(int id)
{
    return ("John", 30);
}
```

### ✅ Kiedy zwrócić record
```csharp
// Structured result, reusable, domain model
public record PersonData(string Name, int Age, string Email);

public async Task<PersonData> GetPersonAsync(int id)
{
    return new PersonData("John", 30, "john@example.com");
}
```

### ✅ Kiedy zwrócić DTO class
```csharp
// API contracts, serialization, complex object
[DataContract]
public class PersonApiResponse
{
    [DataMember]
    public string Name { get; set; }
    [DataMember]
    public int Age { get; set; }
}

public async Task<PersonApiResponse> GetPersonAsync(int id)
{
    return new PersonApiResponse { Name = "John", Age = 30 };
}
```

---

## 8. Uwagi o wydajności

### Allocations przy tuple return
```csharp
// ValueTuple (C# 7.0+) - stackalloc jeśli możliwe
public async Task<(int id, string name)> GetDataAsync()
{
    return (1, "test");  // ValueTuple - usually stack
}

// Tuple (legacy) - heap allocation
public async Task<Tuple<int, string>> GetDataAsync()
{
    return Tuple.Create(1, "test");  // Heap alloc
}
```

### Memory allocations
```csharp
// ValueTuple - zwykle optimized by JIT
var (id, name) = await GetDataAsync();  // Minimal overhead

// Record - może być inlined dla małych typów
var result = await GetPersonAsync();    // Possibility of stack alloc
```

---

## 9. Best practice pattern dla async APIs

```csharp
// Context/Result pattern - best practice
public class GetUserContext
{
    public int UserId { get; set; }
    public User User { get; set; }
    public TimeSpan ExecutionTime { get; set; }
    public string CacheKey { get; set; }
}

public interface IUserService
{
    Task<GetUserContext> GetUserAsync(int id);
}

public class UserService : IUserService
{
    public async Task<GetUserContext> GetUserAsync(int id)
    {
        var sw = Stopwatch.StartNew();
        var user = await _database.GetUserAsync(id);
        sw.Stop();
        
        return new GetUserContext
        {
            UserId = id,
            User = user,
            ExecutionTime = sw.Elapsed,
            CacheKey = GenerateCacheKey(id)
        };
    }
}

// Użycie
var context = await _userService.GetUserAsync(123);
await _cache.SetAsync(context.CacheKey, context.User);
Console.WriteLine($"Fetched in {context.ExecutionTime.TotalMilliseconds}ms");
```

---

## 10. Dlaczego C# tak to projektuje?

### Async state machine constraints
- async metoda generuje state machine implementujący IAsyncStateMachine
- State machine przechowuje local variables i state
- out/ref parametry wymagają external reference
- Reference byłyby invalid po suspension/resumption

### Design decisions
- **Bezpieczeństwo**: lepiej wymagać return value niż ryzykować dangling pointers
- **Predictability**: jawne return types, nie side-effects przez parametry
- **Consistency**: functional-style return value, nie imperative out/ref

---

## Podsumowanie

**Problem:** `async Task<T>` + `out`/`ref` parametry = kompilacja error

**Przyczyny:**
- Stack frame zwalniany przy await
- out/ref byłby dangling pointer
- State machine nie może przechowywać external references

**Rozwiązania (ranked):**
1. **Tuple return** – proste i nowoczesne
2. **Record** – type-safe, sformatowane
3. **DTO/Result class** – dla złożonych scenariuszy
4. **Property object** – gdy potrzeba state
5. **Callback** – zaawansowane przypadki

**Best practice:** Zwracaj wartości przez `Task<T>`, preferuj tuples i records dla wielokrotnych zwrotów.

Developer .NET powinien rozumieć to ograniczenie i automatycznie stosować idiomatic rozwiązania bez walczenia z systemem typów.