# Const vs Readonly Fields w C#

## Wprowadzenie
W C# mamy dwa główne mechanizmy definiowania niezmiennych wartości: **const** i **readonly**. Mimo że oba służą do tworzenia wartości, które nie powinny się zmieniać po inicjalizacji, mają fundamentalnie różne semantyki, zachowanie w czasie kompilacji i wykonania oraz scenariusze użycia.

---

## 1. Podstawowe różnice

| Właściwość              | const                           | readonly                        |
|------------------------|---------------------------------|---------------------------------|
| **Czas inicjalizacji** | Compile-time                   | Runtime (konstruktor)          |
| **Miejsce przechowywania** | Inline w kod IL             | W pamięci jako pole obiektu     |
| **Modyfikatory dostępu** | Domyślnie static              | Instance lub static             |
| **Typy dozwolone**     | Tylko primitive + string + null | Wszystkie typy                 |
| **Inicjalizacja**      | Tylko literały                 | Dowolne wyrażenia runtime      |

---

## 2. Const - wartości stałe w czasie kompilacji

### Charakterystyka
- Wartości są **wkompilowywane** bezpośrednio do IL kodu.
- **Implicitly static** - nie można użyć `static` modifier.
- Muszą być inicjalizowane **literałami** znanymi w czasie kompilacji.

### Przykład
```csharp
public class MathConstants 
{
    public const double Pi = 3.14159;
    public const string Version = "1.0.0";
    public const int MaxItems = 100;
    
    // BŁĄD - nie można użyć runtime expression
    // public const DateTime Now = DateTime.Now; 
}
```

### Wygenerowany IL
```csharp
// Użycie:
double area = radius * radius * MathConstants.Pi;

// Kompiluje się do:
double area = radius * radius * 3.14159;  // inline value
```

---

## 3. Readonly - wartości ustalone w runtime

### Charakterystyka
- Inicjalizacja w **konstruktorze** lub przy deklaracji.
- Można używać **dowolnych wyrażeń** runtime.
- Wspiera zarówno **instance** jak i **static** readonly fields.
- Wartość przechowywana w **obiekcie**, nie inline.

### Przykład
```csharp
public class Configuration
{
    // Static readonly - inicjalizacja przy deklaracji lub static constructor
    public static readonly DateTime AppStartTime = DateTime.Now;
    public static readonly Guid ApplicationId = Guid.NewGuid();
    
    // Instance readonly - inicjalizacja w konstruktorze
    public readonly string ConnectionString;
    public readonly ILogger Logger;
    
    public Configuration(string connStr, ILogger logger)
    {
        ConnectionString = connStr;  // OK w konstruktorze
        Logger = logger;
        
        // ConnectionString = "new value";  // BŁĄD poza konstruktorem
    }
}
```

---

## 4. Versioning i Binary Compatibility

### Problem z const
```csharp
// Assembly A v1.0
public class Constants
{
    public const string ApiVersion = "1.0";
}

// Assembly B - używa Assembly A
public void CallApi()
{
    Console.WriteLine(Constants.ApiVersion);  // "1.0" wkompilowane
}
```

Jeśli zaktualizujemy Assembly A do wersji 2.0 i zmienimy const:
```csharp
// Assembly A v2.0
public const string ApiVersion = "2.0";  // Nowa wartość
```

Assembly B **nadal będzie wypisywać "1.0"** bez rekompilacji!

### Rozwiązanie z readonly
```csharp
// Assembly A
public static readonly string ApiVersion = "2.0";

// Assembly B automatycznie użyje nową wartość po restart aplikacji
```

---

## 5. Performance i Memory Impact

### Const
- **Zero overhead** - wartości inline.
- Brak alokacji pamięci na runtime.
- Idealne dla prawdziwych stałych matematycznych.

### Readonly
- **Pole w obiekcie** - zajmuje pamięć.
- Jeden dodatkowy memory access przy odczycie.
- Static readonly - jedna wartość dla całej aplikacji.

---

## 6. Best Practices i rekomendacje

### Używaj const gdy:
- Wartość jest **prawdziwie stałą** (Pi, stałe matematyczne).
- Wartość **nigdy się nie zmieni** w kolejnych wersjach.
- Potrzebna **maksymalna wydajność**.
- Pracujesz z **primitive types** lub strings.

### Używaj readonly gdy:
- Wartość jest **konfiguracyjna** lub może się zmienić.
- Potrzebujesz **complex objects** lub **runtime calculations**.
- Chcesz **binary compatibility** między wersjami.
- Wartość zależy od **dependency injection** lub external sources.

### Przykłady rekomendowanych zastosowań
```csharp
// CONST - prawdziwe stałe
public const int SecondsInMinute = 60;
public const double GoldenRatio = 1.618033988749;

// READONLY - configuration i runtime values
public static readonly string Environment = 
    ConfigurationManager.AppSettings["Environment"];
    
public readonly IHttpClient HttpClient;
public readonly TimeSpan RequestTimeout;
```

---

## 7. Zaawansowane scenariusze

### Readonly collections
```csharp
public static readonly IReadOnlyList<string> SupportedFormats = 
    new List<string> { "json", "xml", "csv" }.AsReadOnly();

// Lub z C# 12
public static readonly string[] SupportedFormats = ["json", "xml", "csv"];
```

### Readonly structs (C# 7.2+)
```csharp
public readonly struct Point
{
    public readonly int X;
    public readonly int Y;
    
    public Point(int x, int y) => (X, Y) = (x, y);
}
```

---

## Podsumowanie

Wybór między **const** a **readonly** to kluczowa decyzja architekturalna:

- **Const** dla prawdziwych stałych, maksymalnej wydajności i primitive values.
- **Readonly** dla wartości konfiguracyjnych, complex objects i binary compatibility.