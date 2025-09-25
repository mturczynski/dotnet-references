# Value Types vs Reference Types

## Fundamentalna różnica w alokacji pamięci

**Value types** przechowują dane bezpośrednio, natomiast **reference types** przechowują referencję (adres w pamięci) do danych. Ta różnica determinuje wszystkie pozostałe aspekty ich zachowania w aplikacjach .NET.

### Stack vs Heap - gdzie żyją nasze dane

#### Value Types - Stack Allocation
- **Lokalizacja**: Głównie na stosie (stack) dla zmiennych lokalnych
- **Rozmiar**: Ograniczony rozmiarem stosu (domyślnie ~1MB na wątek)
- **Dostęp**: Bardzo szybki - bezpośredni dostęp do danych
- **Zarządzanie**: Automatyczne - dane są usuwane po wyjściu z zakresu

```csharp
void ExampleMethod()
{
    int value = 42;        // Przechowywane bezpośrednio na stosie
    Point point = new Point(10, 20);  // Cała struktura na stosie
    // Po wyjściu z metody - automatyczne zwolnienie pamięci
}
```

#### Reference Types - Heap Allocation
- **Lokalizacja**: Obiekt na stercie (heap), referencja na stosie
- **Rozmiar**: Praktycznie nieograniczony (GB danych)
- **Dostęp**: Wolniejszy - wymaga dereferncji (skok do adresu)
- **Zarządzanie**: Garbage Collector

```csharp
void ExampleMethod()
{
    var person = new Person("Jan");  // Obiekt na stercie, referencja na stosie
    // GC będzie później zarządzał pamięcią obiektu Person
}
```

## Semantyka kopiowania - Copy by Value vs Copy by Reference

### Value Types - Copy by Value
Każda zmienna ma **własną kopię danych**:

```csharp
struct Point
{
    public int X, Y;
    public Point(int x, int y) { X = x; Y = y; }
}

Point p1 = new Point(5, 10);
Point p2 = p1;    // Kopiowanie całej struktury
p2.X = 20;        // p1.X pozostaje 5

Console.WriteLine($"p1.X = {p1.X}, p2.X = {p2.X}");
// Wynik: p1.X = 5, p2.X = 20
```

### Reference Types - Copy by Reference
Wiele zmiennych może **wskazywać na ten sam obiekt**:

```csharp
class Person
{
    public string Name { get; set; }
    public Person(string name) { Name = name; }
}

Person person1 = new Person("Anna");
Person person2 = person1;    // Kopiowanie referencji, nie obiektu
person2.Name = "Ewa";        // Zmiana wpływa na oba obiekty

Console.WriteLine($"person1.Name = {person1.Name}");
// Wynik: person1.Name = Ewa
```

## Boxing i Unboxing - Performance Critical Concept

### Boxing - konwersja Value Type → Reference Type

Boxing zachodzi automatycznie przy konwersji do `object` lub interfejsów:

```csharp
int number = 42;              // Value type na stosie
object boxed = number;        // Boxing - alokacja na stercie!

// Nieoczekiwane boxing w kolekcjach:
ArrayList list = new ArrayList();
list.Add(42);                 // Boxing każdej wartości!

// Boxing w String.Format:
string message = String.Format("Value: {0}", 42);  // Boxing!
```

### Unboxing - konwersja Reference Type → Value Type

```csharp
object boxed = 42;
int unboxed = (int)boxed;     // Explicit cast wymagany

// Niepoprawny unboxing:
double invalid = (double)boxed;  // InvalidCastException!
```

### Koszt wydajnościowy Boxing/Unboxing

```csharp
// ❌ Wolne - boxing w każdej iteracji
ArrayList oldList = new ArrayList();
for (int i = 0; i < 1000000; i++)
{
    oldList.Add(i);  // 1M alokacji na stercie!
}

// ✅ Szybkie - bez boxing
List<int> newList = new List<int>();
for (int i = 0; i < 1000000; i++)
{
    newList.Add(i);  // Bez dodatkowych alokacji
}
```

## Wyjątki od reguły - kiedy Value Types żyją na Heap

### 1. Pola w klasach (Reference Types)

```csharp
class Container
{
    public int Number;      // Value type, ale na stercie jako część obiektu
    public Point Location;  // Struktura na stercie jako pole klasy
}

Container container = new Container();
// container.Number i container.Location są na stercie!
```

### 2. Elementy tablic

```csharp
int[] numbers = new int[1000];  // Tablica na stercie
// Wszystkie elementy int[] również na stercie, nie na stosie
```

### 3. Boxed Value Types

```csharp
int value = 42;
object boxed = value;  // Boxing - kopia value type na stercie
```

### 4. Captured Variables w Lambda/Closures

```csharp
int captured = 42;
Action action = () => Console.WriteLine(captured);
// 'captured' zostanie przeniesiona na stertę dla closure
```

## Garbage Collection i Value Types

### GC nie zarządza Value Types bezpośrednio

```csharp
void StackExample()
{
    int localVar = 42;        // Nie wymaga GC
    Point point = new Point(1, 2);  // Nie wymaga GC
    // Automatyczne zwolnienie przy wyjściu z metody
}
```

### Ale GC zarządza nimi pośrednio

```csharp
class DataContainer
{
    public int Value;         // GC zbiera całą klasę wraz z tym polem
    public decimal Price;     // Zbierane razem z obiektem klasy
}
```

## Nowoczesne techniki zarządzania pamięcią

### Span&lt;T&gt; - Zero-allocation slicing

```csharp
// ❌ Tradycyjne podejście - alokacje
string text = "Hello World";
string substring = text.Substring(6, 5);  // Nowa alokacja!

// ✅ Span<T> - zero allocation
ReadOnlySpan<char> span = text.AsSpan();
ReadOnlySpan<char> slice = span.Slice(6, 5);  // Bez alokacji!
```

### stackalloc - Stack allocation for arrays

```csharp
// ❌ Heap allocation
byte[] heapArray = new byte[1024];

// ✅ Stack allocation (dla małych tablic)
Span<byte> stackArray = stackalloc byte[1024];
// Automatycznie zwolnione po wyjściu z metody
```

### Memory&lt;T&gt; - Asynchronous-safe memory

```csharp
async Task ProcessDataAsync()
{
    // Span<T> nie działa z async/await
    Memory<byte> buffer = new byte[1024];
    await SomeAsyncOperation(buffer);  // OK
    
    // Span<byte> span = stackalloc byte[1024];  // Błąd kompilacji!
    // await SomeAsyncOperation(span);
}
```

## Performance Implications - praktyczne wskazówki

### 1. Wielkość struktur ma znaczenie

```csharp
// ✅ Małe struktury (≤16 bajtów) - idealne
public readonly struct Point2D
{
    public readonly int X, Y;  // 8 bajtów total
}

// ❌ Duże struktury - kosztowne kopiowanie
public struct HugeStruct
{
    public decimal Field1, Field2, Field3, Field4;  // 64 bajty!
    // Każde przekazanie = kopiowanie 64 bajtów
}
```

### 2. Implementuj IEquatable&lt;T&gt; dla struktur

```csharp
// ❌ Wolne - używa refleksji
public struct SlowPoint
{
    public int X, Y;
    // Domyślny Equals używa refleksji!
}

// ✅ Szybkie - zoptymalizowane porównania
public struct FastPoint : IEquatable<FastPoint>
{
    public int X, Y;
    
    public bool Equals(FastPoint other) => X == other.X && Y == other.Y;
    public override bool Equals(object obj) => obj is FastPoint p && Equals(p);
    public override int GetHashCode() => HashCode.Combine(X, Y);
}
```

### 3. Używaj generic collections

```csharp
// ❌ Boxing w każdej operacji
ArrayList list = new ArrayList();
Hashtable hash = new Hashtable();

// ✅ Type-safe i bez boxing
List<int> list = new List<int>();
Dictionary<string, int> hash = new Dictionary<string, int>();
```

## Memory Layout i Padding

### Struct padding wpływa na rozmiar

```csharp
// ❌ Źle ułożone pola - 40 bajtów przez padding
struct BadLayout
{
    public byte B1;    // 1 bajt + 7 padding
    public double D1;  // 8 bajtów  
    public byte B2;    // 1 bajt + 7 padding
    public double D2;  // 8 bajtów
    public byte B3;    // 1 bajt + 7 padding
    // Total: 40 bajtów
}

// ✅ Dobrze ułożone pola - 24 bajty
struct GoodLayout
{
    public double D1;  // 8 bajtów
    public double D2;  // 8 bajtów
    public byte B1;    // 1 bajt
    public byte B2;    // 1 bajt
    public byte B3;    // 1 bajt + 5 padding
    // Total: 24 bajty
}
```

### Kontrola layoutu z atrybutami

```csharp
[StructLayout(LayoutKind.Explicit)]
public struct ExplicitLayout
{
    [FieldOffset(0)]
    public int IntValue;
    
    [FieldOffset(0)]
    public float FloatValue;  // Aliasing - to samo miejsce w pamięci
}
```

## Wzorce projektowe i best practices

### 1. Immutable Value Types

```csharp
public readonly struct Money
{
    public readonly decimal Amount;
    public readonly string Currency;
    
    public Money(decimal amount, string currency)
    {
        Amount = amount;
        Currency = currency;
    }
    
    // Immutable operations return new instances
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Currency mismatch");
            
        return new Money(Amount + other.Amount, Currency);
    }
}
```

### 2. Factory Methods dla Value Types

```csharp
public readonly struct Color
{
    private readonly uint _value;
    
    private Color(uint value) => _value = value;
    
    public static Color FromRgb(byte r, byte g, byte b)
        => new Color((uint)(r << 16 | g << 8 | b));
    
    public static readonly Color Red = FromRgb(255, 0, 0);
    public static readonly Color Green = FromRgb(0, 255, 0);
    public static readonly Color Blue = FromRgb(0, 0, 255);
}
```

### 3. Ref Returns dla performance-critical code

```csharp
public struct Matrix4x4
{
    private float[,] _elements;
    
    // Ref return pozwala na modyfikację bez kopiowania
    public ref float this[int row, int col] => ref _elements[row, col];
}

Matrix4x4 matrix = new Matrix4x4();
matrix[0, 0] = 1.0f;  // Bezpośrednia modyfikacja bez kopiowania
```

## Podsumowanie - kiedy co używać

### Używaj Value Types gdy:
- Reprezentujesz proste wartości (Point, Color, Money)
- Rozmiar ≤ 16 bajtów
- Immutable data
- Semantyka wartościowa jest naturalna
- Potrzebujesz high-performance w kolekcjach

### Używaj Reference Types gdy:
- Złożone obiekty biznesowe
- Potrzebujesz dziedziczenia/polimorfizmu
- Duże struktury danych
- Długotrwałe obiekty współdzielone
- Identity semantics (dwa obiekty = ten sam obiekt)

### Record Types (C# 9+) - najlepsze z obu światów:

```csharp
// Record struct - immutable value type z automatycznymi implementacjami
public readonly record struct Point3D(double X, double Y, double Z);

// Record class - immutable reference type  
public record Person(string Name, int Age);
```

Zrozumienie różnic między value types i reference types to fundament wysokowydajnego programowania w .NET. Świadome decyzje dotyczące wyboru typu danych mogą znacząco wpłynąć na wydajność, zużycie pamięci i zachowanie aplikacji.