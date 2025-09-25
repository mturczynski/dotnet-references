# Klasa abstrakcyjna vs Interfejs w C#

## Wprowadzenie
W C# zarówno **klasy abstrakcyjne** jak i **interfejsy** służą do definiowania kontraktów i abstrakcji, ale mają fundamentalnie różne cele i możliwości. Zrozumienie tych różnic jest kluczowe dla właściwego projektowania architektury aplikacji .NET.

---

## 1. Podstawowe różnice

| Właściwość | Klasa abstrakcyjna | Interfejs |
|------------|-------------------|-----------|
| **Dziedziczenie** | Single inheritance | Multiple inheritance |
| **Implementacja** | Może zawierać implementację | Tylko sygnatury (do C# 8.0) |
| **Pola** | Wszystkie typy pól | Brak (tylko properties) |
| **Konstruktory** | Tak | Nie |
| **Modyfikatory dostępu** | public, protected, private, internal | public (domyślnie) |
| **Static members** | Tak | Tak (od C# 8.0) |
| **Wersjonowanie** | Łatwiejsze dodawanie członków | Breaking changes |

---

## 2. Klasa abstrakcyjna - partial implementation

### Charakterystyka
- **Szablon** z częściową implementacją dla klas pochodnych.
- **Template Method Pattern** - definiuje algorytm z customizable steps.
- **Shared state** - wspólne pola i właściwości.
- **Konstruktory** dla inicjalizacji wspólnego stanu.

### Przykład
```csharp
public abstract class DatabaseRepository<T> where T : class
{
    protected readonly string ConnectionString;
    protected readonly ILogger Logger;
    
    protected DatabaseRepository(string connectionString, ILogger logger)
    {
        ConnectionString = connectionString ?? throw new ArgumentNullException();
        Logger = logger;
    }
    
    // Template method - definuje workflow
    public async Task<T> SaveAsync(T entity)
    {
        Logger.LogInformation($"Saving {typeof(T).Name}");
        
        ValidateEntity(entity);           // abstract - must implement
        var result = await PersistAsync(entity);  // abstract - must implement
        OnEntitySaved(entity);            // virtual - can override
        
        return result;
    }
    
    protected abstract void ValidateEntity(T entity);
    protected abstract Task<T> PersistAsync(T entity);
    
    protected virtual void OnEntitySaved(T entity) 
    {
        Logger.LogInformation($"{typeof(T).Name} saved successfully");
    }
}

public class UserRepository : DatabaseRepository<User>
{
    public UserRepository(string connectionString, ILogger logger) 
        : base(connectionString, logger) { }
    
    protected override void ValidateEntity(User user)
    {
        if (string.IsNullOrEmpty(user.Email))
            throw new ValidationException("Email is required");
    }
    
    protected override async Task<User> PersistAsync(User user)
    {
        // SQL implementation
        using var connection = new SqlConnection(ConnectionString);
        // ... implementation
        return user;
    }
}
```

---

## 3. Interfejs - pure contract

### Charakterystyka (tradycyjne, pre-C# 8.0)
- **Pure contract** - tylko sygnatury metod.
- **Multiple inheritance** - klasa może implementować wiele interfejsów.
- **Loose coupling** - dependency inversion.
- **Testability** - łatwe mockowanie.

### Przykład
```csharp
public interface IPaymentProcessor
{
    Task<PaymentResult> ProcessAsync(decimal amount, string currency);
    Task<bool> RefundAsync(string transactionId);
    event EventHandler<PaymentProcessedEventArgs> PaymentProcessed;
}

public interface INotificationService
{
    Task SendAsync(string recipient, string message);
}

// Klasa może implementować multiple interfaces
public class StripePaymentService : IPaymentProcessor, INotificationService
{
    public async Task<PaymentResult> ProcessAsync(decimal amount, string currency)
    {
        // Stripe API implementation
        var result = await CallStripeApi(amount, currency);
        PaymentProcessed?.Invoke(this, new PaymentProcessedEventArgs(result));
        return result;
    }
    
    public async Task<bool> RefundAsync(string transactionId)
    {
        // Refund implementation
        return await ProcessRefund(transactionId);
    }
    
    public async Task SendAsync(string recipient, string message)
    {
        // Email notification implementation
        await SendEmail(recipient, message);
    }
    
    public event EventHandler<PaymentProcessedEventArgs> PaymentProcessed;
}
```

---

## 4. Nowości C# 8.0+ - Default Interface Methods

### Interface z implementacją domyślną
```csharp
public interface ILogger
{
    void Log(string message);
    
    // Default implementation - nie breaking change
    void LogWithTimestamp(string message)
    {
        Log($"[{DateTime.UtcNow:yyyy-MM-dd HH:mm:ss}] {message}");
    }
    
    // Static methods
    static ILogger CreateConsoleLogger() => new ConsoleLogger();
}

public class FileLogger : ILogger
{
    public void Log(string message)
    {
        File.AppendAllText("log.txt", message + Environment.NewLine);
    }
    
    // LogWithTimestamp używa default implementation
}
```

---

## 5. Kiedy używać którego?

### Klasa abstrakcyjna - gdy:
- **Wspólny kod** między implementacjami (Template Method Pattern).
- **Shared state** - pola, właściwości, konstruktory.
- **Ewolucja w czasie** - łatwiejsze dodawanie funkcjonalności.
- **Silne powiązanie** między klasami pochodymi.
- **Framework design** - np. ASP.NET Core controllers.

### Interfejs - gdy:
- **Pure contract** bez implementacji.
- **Multiple inheritance** - klasa implementuje wiele ról.
- **Loose coupling** - dependency injection, testability.
- **API design** - publiczne kontrakty.
- **Cross-cutting concerns** - IDisposable, IComparable.

---

## 6. Kombinowanie obu podejść

```csharp
// Interface definiuje kontrakt
public interface IDocumentProcessor
{
    Task<ProcessResult> ProcessAsync(Document document);
    bool CanProcess(string fileExtension);
}

// Abstract class zapewnia shared functionality  
public abstract class BaseDocumentProcessor : IDocumentProcessor
{
    protected readonly ILogger Logger;
    protected readonly IFileSystem FileSystem;
    
    protected BaseDocumentProcessor(ILogger logger, IFileSystem fileSystem)
    {
        Logger = logger;
        FileSystem = fileSystem;
    }
    
    // Template method
    public async Task<ProcessResult> ProcessAsync(Document document)
    {
        Logger.LogInformation($"Processing {document.Name}");
        
        if (!CanProcess(document.Extension))
            throw new NotSupportedException($"Cannot process {document.Extension}");
            
        var result = await ProcessDocumentAsync(document);
        
        Logger.LogInformation($"Processed {document.Name} successfully");
        return result;
    }
    
    public abstract bool CanProcess(string fileExtension);
    protected abstract Task<ProcessResult> ProcessDocumentAsync(Document document);
}

// Konkretne implementacje
public class PdfProcessor : BaseDocumentProcessor
{
    public PdfProcessor(ILogger logger, IFileSystem fileSystem) 
        : base(logger, fileSystem) { }
    
    public override bool CanProcess(string fileExtension) => 
        fileExtension.Equals(".pdf", StringComparison.OrdinalIgnoreCase);
        
    protected override async Task<ProcessResult> ProcessDocumentAsync(Document document)
    {
        // PDF-specific processing
        return await ProcessPdf(document);
    }
}
```

---

## 7. Design Patterns i Best Practices

### Strategy Pattern z interfejsami
```csharp
public interface ICompressionStrategy
{
    byte[] Compress(byte[] data);
    byte[] Decompress(byte[] compressedData);
}

public class CompressionContext
{
    private readonly ICompressionStrategy _strategy;
    
    public CompressionContext(ICompressionStrategy strategy)
    {
        _strategy = strategy;
    }
    
    public byte[] Process(byte[] data) => _strategy.Compress(data);
}
```

### Template Method z klasami abstrakcyjnymi
```csharp
public abstract class DataImporter
{
    public async Task<ImportResult> ImportAsync(string source)
    {
        var data = await ReadDataAsync(source);      // abstract
        var validated = ValidateData(data);          // virtual
        var transformed = TransformData(validated);  // abstract
        return await SaveDataAsync(transformed);     // abstract
    }
    
    protected abstract Task<RawData> ReadDataAsync(string source);
    protected abstract Task<ImportResult> SaveDataAsync(ProcessedData data);
    protected abstract ProcessedData TransformData(RawData data);
    
    protected virtual RawData ValidateData(RawData data)
    {
        // Default validation logic
        return data;
    }
}
```

---

## Podsumowanie

**Klasy abstrakcyjne** i **interfejsy** służą różnym celom w architekturze .NET:

- **Klasa abstrakcyjna**: Template Method, shared implementation, strong coupling.
- **Interfejs**: Pure contract, multiple inheritance, loose coupling, dependency injection.

.NET Developer powinien używać ich komplementarnie:
- **Interfejsy** dla definiowania publicznych kontraktów i dependency injection.
- **Klasy abstrakcyjne** dla wspólnej implementacji i template patterns.

Nowoczesny C# (8.0+) z default interface methods rozmywa te granice, ale fundamentalne różnice w semantyce i zastosowaniu pozostają aktualne.