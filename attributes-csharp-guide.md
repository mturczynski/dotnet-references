# Atrybuty w C#

## Wprowadzenie
**Atrybuty** w C# to mechanizm deklaratywny umożliwiający dodawanie metadanych do elementów kodu (klas, metod, właściwości, parametrów, assembly). Są kompilowane do IL i dostępne w runtime przez **Reflection**, stanowiąc fundament dla frameworków jak ASP.NET Core, Entity Framework, czy bibliotek serialization.

---

## 1. Fundamenty atrybutów

### Definicja i składnia
```csharp
// Basic attribute usage
[Serializable]
public class User
{
    [Required]
    [MaxLength(100)]
    public string Name { get; set; }
    
    [JsonPropertyName("email_address")]
    public string Email { get; set; }
    
    [Obsolete("Use CreatedAt instead", true)]  // error on usage
    public DateTime Created { get; set; }
    
    public DateTime CreatedAt { get; set; }
}
```

### Miejsca stosowania AttributeTargets
```csharp
[assembly: AssemblyVersion("1.0.0.0")]       // Assembly
[module: CLSCompliant(true)]                 // Module

public class Example
{
    [field: NonSerialized]                   // Field (backing field)
    public string AutoProperty { get; set; }
    
    [method: MethodImpl(MethodImplOptions.NoInlining)]
    [return: MarshalAs(UnmanagedType.Bool)]   // Return value
    public bool Calculate([param: NotNull] string input) => true;
    
    [event: field: NonSerialized]            // Event field
    public event Action<string> OnChanged;
}
```

---

## 2. Implementacja custom attributes

### Podstawowa klasa atrybutu
```csharp
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method, 
                AllowMultiple = true, Inherited = true)]
public class LogExecutionAttribute : Attribute
{
    public string Category { get; set; }
    public LogLevel Level { get; set; } = LogLevel.Information;
    public bool IncludeParameters { get; set; }
    
    public LogExecutionAttribute(string category = "Default")
    {
        Category = category;
    }
}

// Usage
[LogExecution("UserService", Level = LogLevel.Debug, IncludeParameters = true)]
public class UserService
{
    [LogExecution("Authentication")]
    public async Task<bool> AuthenticateAsync(string username, string password)
    {
        // Implementation
        return true;
    }
}
```

### Zaawansowany attribute z validation
```csharp
[AttributeUsage(AttributeTargets.Property)]
public class ValidateEmailAttribute : ValidationAttribute
{
    private static readonly Regex EmailRegex = new(
        @"^[^@\s]+@[^@\s]+\.[^@\s]+$", 
        RegexOptions.Compiled | RegexOptions.IgnoreCase);
    
    public override bool IsValid(object value)
    {
        if (value == null) return true; // Let [Required] handle nulls
        
        return value is string email && EmailRegex.IsMatch(email);
    }
    
    public override string FormatErrorMessage(string name)
    {
        return $"The {name} field must be a valid email address.";
    }
}

// Usage
public class RegisterRequest
{
    [Required]
    [ValidateEmail]
    public string Email { get; set; }
}
```

---

## 3. Reflection i runtime access

### Odczyt metadanych
```csharp
public static class AttributeHelper
{
    public static T GetCustomAttribute<T>(Type type) where T : Attribute
    {
        return type.GetCustomAttribute<T>();
    }
    
    public static IEnumerable<T> GetCustomAttributes<T>(MemberInfo member) 
        where T : Attribute
    {
        return member.GetCustomAttributes<T>();
    }
    
    public static bool HasAttribute<T>(MemberInfo member) where T : Attribute
    {
        return member.IsDefined(typeof(T), inherit: true);
    }
}

// Usage example - validation framework
public static class Validator
{
    public static ValidationResult Validate(object instance)
    {
        var type = instance.GetType();
        var errors = new List<string>();
        
        foreach (var property in type.GetProperties())
        {
            var value = property.GetValue(instance);
            
            // Check Required attribute
            if (property.HasAttribute<RequiredAttribute>() && value == null)
            {
                errors.Add($"{property.Name} is required");
            }
            
            // Check custom validation attributes
            var validationAttribs = property.GetCustomAttributes<ValidationAttribute>();
            foreach (var attr in validationAttribs)
            {
                if (!attr.IsValid(value))
                {
                    errors.Add(attr.FormatErrorMessage(property.Name));
                }
            }
        }
        
        return new ValidationResult(errors);
    }
}
```

---

## 4. Interceptors i source generators z atrybutami

### Marker attribute dla source generator
```csharp
[AttributeUsage(AttributeTargets.Method)]
public class GenerateBuilderAttribute : Attribute
{
    public bool GenerateToString { get; set; } = true;
    public bool GenerateEquals { get; set; } = true;
}

// Usage
[GenerateBuilder(GenerateEquals = false)]
public partial class User
{
    public string Name { get; set; }
    public string Email { get; set; }
    public DateTime CreatedAt { get; set; }
}

// Generated by source generator:
// public partial class User
// {
//     public static UserBuilder Builder() => new UserBuilder();
//     
//     public override string ToString() => 
//         $"User {{ Name = {Name}, Email = {Email}, CreatedAt = {CreatedAt} }}";
// }
```

### Interceptor attribute (C# 12)
```csharp
[AttributeUsage(AttributeTargets.Method, AllowMultiple = true)]
public class InterceptsLocationAttribute : Attribute
{
    public InterceptsLocationAttribute(string filePath, int line, int column)
    {
        FilePath = filePath;
        Line = line;
        Column = column;
    }
    
    public string FilePath { get; }
    public int Line { get; }
    public int Column { get; }
}
```

---

## 5. Framework integration patterns

### ASP.NET Core attributes
```csharp
[ApiController]
[Route("api/[controller]")]
[Authorize(Roles = "Admin")]
public class UsersController : ControllerBase
{
    [HttpGet]
    [ResponseCache(Duration = 300)]
    [ProducesResponseType(typeof(IEnumerable<UserDto>), StatusCodes.Status200OK)]
    public async Task<ActionResult<IEnumerable<UserDto>>> GetUsers()
    {
        // Implementation
        return Ok();
    }
    
    [HttpPost]
    [ValidateAntiForgeryToken]
    [ServiceFilter(typeof(ValidationFilter))]
    public async Task<ActionResult<UserDto>> CreateUser([FromBody] CreateUserRequest request)
    {
        // Implementation
        return CreatedAtAction(nameof(GetUser), new { id = 1 }, new UserDto());
    }
    
    [HttpGet("{id:int}")]
    [ResponseCache(VaryByHeader = "User-Agent", Duration = 600)]
    public async Task<ActionResult<UserDto>> GetUser([FromRoute] int id)
    {
        // Implementation
        return Ok();
    }
}
```

### Entity Framework attributes
```csharp
[Table("Users")]
[Index(nameof(Email), IsUnique = true)]
public class User
{
    [Key]
    [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
    public int Id { get; set; }
    
    [Required]
    [MaxLength(100)]
    [Column(TypeName = "nvarchar")]
    public string Name { get; set; }
    
    [Required]
    [MaxLength(255)]
    [Column("EmailAddress")]
    public string Email { get; set; }
    
    [Column(TypeName = "datetime2")]
    [DatabaseGenerated(DatabaseGeneratedOption.Computed)]
    public DateTime CreatedAt { get; set; }
    
    [NotMapped]
    public string FullDisplayName => $"{Name} <{Email}>";
    
    [ForeignKey(nameof(DepartmentId))]
    public Department Department { get; set; }
    
    public int DepartmentId { get; set; }
}
```

---

## 6. Performance considerations

### Reflection caching
```csharp
public static class AttributeCache
{
    private static readonly ConcurrentDictionary<Type, Dictionary<string, Attribute[]>> 
        _typeCache = new();
    
    public static T[] GetCachedAttributes<T>(Type type, string memberName) 
        where T : Attribute
    {
        var typeAttributes = _typeCache.GetOrAdd(type, t =>
        {
            var members = t.GetMembers(BindingFlags.Public | BindingFlags.Instance);
            return members.ToDictionary(
                m => m.Name,
                m => m.GetCustomAttributes().ToArray()
            );
        });
        
        if (typeAttributes.TryGetValue(memberName, out var attributes))
        {
            return attributes.OfType<T>().ToArray();
        }
        
        return Array.Empty<T>();
    }
}
```

### Compile-time attribute processing
```csharp
// Source generator approach for better performance
[System.CodeDom.Compiler.GeneratedCode("AttributeProcessor", "1.0")]
public static partial class UserValidation
{
    // Generated at compile time instead of reflection at runtime
    public static ValidationResult ValidateUser(User user)
    {
        var errors = new List<string>();
        
        if (string.IsNullOrEmpty(user.Name))
            errors.Add("Name is required");
        
        if (user.Name?.Length > 100)
            errors.Add("Name must not exceed 100 characters");
            
        // ... more validations
        
        return new ValidationResult(errors);
    }
}
```

---

## 7. Advanced scenarios

### Generic attributes (C# 11+)
```csharp
public class GenericAttribute<T> : Attribute where T : class
{
    public Type TargetType => typeof(T);
    public T DefaultValue { get; set; }
    
    public GenericAttribute(T defaultValue = default)
    {
        DefaultValue = defaultValue;
    }
}

// Usage
[Generic<string>("DefaultName")]
public class User
{
    public string Name { get; set; }
}

// Access
var attr = typeof(User).GetCustomAttribute<GenericAttribute<string>>();
var defaultName = attr?.DefaultValue; // "DefaultName"
```

### Conditional compilation attributes
```csharp
public class ConditionalMethodAttribute : Attribute
{
    public string Condition { get; }
    
    public ConditionalMethodAttribute(string condition)
    {
        Condition = condition;
    }
}

[Conditional("DEBUG")]
[ConditionalMethod("TRACE_ENABLED")]
public static void TraceMessage(string message)
{
    Console.WriteLine($"TRACE: {message}");
}
```

---

## 8. Best practices

### ✅ Recommended patterns
```csharp
// 1. Meaningful names and clear purpose
[AttributeUsage(AttributeTargets.Method)]
public class CachingPolicyAttribute : Attribute
{
    public TimeSpan Duration { get; set; } = TimeSpan.FromMinutes(5);
    public string[] VaryByHeaders { get; set; } = Array.Empty<string>();
}

// 2. Validation in constructor
[AttributeUsage(AttributeTargets.Property)]
public class RangeValidationAttribute : Attribute
{
    public int Min { get; }
    public int Max { get; }
    
    public RangeValidationAttribute(int min, int max)
    {
        if (min > max)
            throw new ArgumentException("Min cannot be greater than max");
            
        Min = min;
        Max = max;
    }
}

// 3. Fluent configuration support
public class ApiEndpointAttribute : Attribute
{
    public string Route { get; set; }
    public string[] HttpMethods { get; set; } = { "GET" };
    public bool RequireAuth { get; set; }
    public string[] Roles { get; set; } = Array.Empty<string>();
    
    public ApiEndpointAttribute(string route)
    {
        Route = route ?? throw new ArgumentNullException(nameof(route));
    }
    
    public ApiEndpointAttribute WithMethods(params string[] methods)
    {
        HttpMethods = methods;
        return this;
    }
    
    public ApiEndpointAttribute RequireAuthorization(params string[] roles)
    {
        RequireAuth = true;
        Roles = roles;
        return this;
    }
}
```

### ❌ Anti-patterns
```csharp
// Don't: Too many parameters
public class BadAttribute : Attribute
{
    public BadAttribute(string a, int b, bool c, double d, string e, int f) { }
}

// Don't: Mutable state without proper initialization
public class MutableAttribute : Attribute
{
    public List<string> Items { get; set; }  // Can be null
}

// Don't: Heavy computation in constructor
public class ExpensiveAttribute : Attribute
{
    public ExpensiveAttribute()
    {
        // Expensive database call or complex computation
        ExpensiveOperation();
    }
}
```

---

## 9. Framework examples w praktyce

### Custom validation attribute
```csharp
public class BusinessRuleAttribute : ValidationAttribute
{
    private readonly string _ruleName;
    private readonly IServiceProvider _serviceProvider;
    
    public BusinessRuleAttribute(string ruleName)
    {
        _ruleName = ruleName;
    }
    
    protected override ValidationResult IsValid(object value, ValidationContext context)
    {
        var ruleEngine = context.GetService<IBusinessRuleEngine>();
        var result = ruleEngine.Validate(_ruleName, value, context.ObjectInstance);
        
        return result.IsValid 
            ? ValidationResult.Success 
            : new ValidationResult(result.ErrorMessage);
    }
}

// Usage
public class Order
{
    [BusinessRule("ValidateOrderAmount")]
    public decimal Amount { get; set; }
    
    [BusinessRule("ValidateCustomerCredit")]
    public int CustomerId { get; set; }
}
```

---

## Podsumowanie

**Atrybuty** w C# są potężnym mechanizmem declarative programming umożliwiającym:

- **Separation of concerns** - oddzielenie metadanych od logiki biznesowej
- **Framework integration** - podstawa dla ASP.NET Core, EF Core, validation
- **Code generation** - współpraca z source generators i analyzers
- **AOP patterns** - cross-cutting concerns przez interceptory

Developer .NET powinien:
- Rozumieć związek między atrybutami a reflection/performance
- Projektować attributes z myślą o reusability i type safety  
- Wykorzystywać source generators zamiast reflection gdzie możliwe
- Znać wzorce integracji z popularnymi frameworkami

Atrybuty to klucz do declarative, metadata-driven architectures w nowoczesnym .NET.