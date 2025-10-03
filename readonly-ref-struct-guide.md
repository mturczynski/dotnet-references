# readonly struct, ref struct vs struct w C#

## Wprowadzenie
C# 7.2 wprowadził **readonly struct** i **ref struct** jako specjalizowane warianty zwykłej struktury. Te modyfikatory znacząco zmieniają semantykę, ograniczenia i możliwości optymalizacji, będąc kluczowymi dla high-performance scenariuszy i zero-allocation programming.

---

## 1. readonly struct - niezmienność gwarantowana przez kompilator

### Charakterystyka
- **Immutability**: wszystkie pola muszą być `readonly` lub auto-implemented properties z tylko `get`
- **Defensive copying elimination**: kompilator nie generuje defensive copies przy wywołaniach metod
- **Performance**: eliminuje ukryte kopie w read-only contexts

### Implementacja
```csharp
public readonly struct Point3D
{
    public readonly double X;
    public readonly double Y; 
    public readonly double Z;
    
    public Point3D(double x, double y, double z)
    {
        X = x;
        Y = y;
        Z = z;
    }
    
    // Metody mogą tylko czytać pola
    public double Magnitude => Math.Sqrt(X * X + Y * Y + Z * Z);
    
    public Point3D Normalize()
    {
        var mag = Magnitude;
        return new Point3D(X / mag, Y / mag, Z / mag);
    }
    
    // BŁĄD - nie można modyfikować pól
    // public void Scale(double factor) { X *= factor; }
}
```

### Defensive copying problem
```csharp
// Regular struct - hidden defensive copies
public struct MutablePoint
{
    public int X;
    public int Y;
    
    public readonly int GetSum() => X + Y;
}

void RegularStructProblem()
{
    var points = new MutablePoint[] { new(1, 2), new(3, 4) };
    
    // Każde wywołanie GetSum() tworzy defensive copy!
    foreach (ref readonly var point in points.AsSpan())
    {
        var sum = point.GetSum(); // Hidden copy created
    }
}

// readonly struct - no defensive copies
public readonly struct ImmutablePoint
{
    public readonly int X;
    public readonly int Y;
    
    public ImmutablePoint(int x, int y) => (X, Y) = (x, y);
    public int GetSum() => X + Y; // No copy needed
}
```

### Korzyści performance
```csharp
// Benchmark comparison
[Benchmark]
public int RegularStructSum()
{
    var sum = 0;
    foreach (ref readonly var point in _mutablePoints.AsSpan())
    {
        sum += point.GetSum(); // Defensive copy on each call
    }
    return sum;
}

[Benchmark]
public int ReadonlyStructSum()
{
    var sum = 0;
    foreach (ref readonly var point in _immutablePoints.AsSpan())
    {
        sum += point.GetSum(); // No defensive copies
    }
    return sum;
}
// ReadonlyStructSum może być 2-3x szybszy
```

---

## 2. ref struct - stack-only semantics

### Charakterystyka
- **Stack allocation only**: nie może być alokowany na heap
- **No boxing**: nie można konwertować do `object` ani interfejsu
- **Temporary lifetime**: idealny dla short-lived, high-performance operations
- **Span-like semantics**: fundament dla `Span<T>`, `ReadOnlySpan<T>`

### Implementacja
```csharp
public ref struct StackOnlyBuffer
{
    private readonly Span<byte> _buffer;
    
    public StackOnlyBuffer(Span<byte> buffer)
    {
        _buffer = buffer;
    }
    
    public ref byte this[int index] => ref _buffer[index];
    
    public void Clear() => _buffer.Clear();
    
    public Span<byte> Slice(int start, int length) => _buffer.Slice(start, length);
}

// Usage
void ProcessData()
{
    Span<byte> stackBuffer = stackalloc byte[1024]; // Stack allocation
    var buffer = new StackOnlyBuffer(stackBuffer);
    
    // Proces data without heap allocations
    buffer[0] = 42;
    buffer.Clear();
    
    // buffer automatycznie "znika" po wyjściu z scope
}
```

### Ograniczenia ref struct
```csharp
public ref struct RestrictedStruct
{
    // OK - można mieć pola value types i inne ref structs
    private int _value;
    private Span<byte> _span;
    
    // BŁĄD - nie można mieć pól reference types
    // private string _text; // Compiler error
    // private object _obj;  // Compiler error
}

// Ograniczenia użycia
class UsageRestrictions
{
    // BŁĄD - nie można jako pole klasy
    // private StackOnlyBuffer _buffer;
    
    // BŁĄD - nie można w generic type parameters
    // private List<StackOnlyBuffer> _buffers;
    
    // BŁĄD - nie można boxing
    // object boxed = new StackOnlyBuffer(...);
    
    // BŁĄD - nie można jako async method parameter/return
    // public async Task ProcessAsync(StackOnlyBuffer buffer) { }
    
    // OK - można jako local variable i method parameters (non-async)
    public void Process(StackOnlyBuffer buffer)
    {
        var localBuffer = buffer; // OK
    }
}
```

---

## 3. Kombinacja: readonly ref struct

### Ultimate performance struct
```csharp
public readonly ref struct ReadOnlySpanWrapper<T>
{
    private readonly ReadOnlySpan<T> _span;
    
    public ReadOnlySpanWrapper(ReadOnlySpan<T> span)
    {
        _span = span;
    }
    
    public int Length => _span.Length;
    
    public ref readonly T this[int index] => ref _span[index];
    
    // Immutable + stack-only = maximum performance
    public ReadOnlySpanWrapper<T> Slice(int start, int length) => 
        new(_span.Slice(start, length));
}
```

---

## 4. Praktyczne zastosowania

### Span<T> i Memory<T> ecosystem
```csharp
// Span<T> to ref struct
public readonly ref struct Span<T>
{
    private readonly ref T _pointer;
    private readonly int _length;
    
    // Zero-allocation slicing
    public Span<T> Slice(int start, int length) => new(ref Unsafe.Add(ref _pointer, start), length);
}

// Praktyczne użycie
void ProcessLargeArray()
{
    var data = new int[1_000_000];
    
    // Slice without allocation
    Span<int> slice = data.AsSpan(500_000, 100_000);
    
    // Process slice with zero allocations
    foreach (ref var item in slice)
    {
        item *= 2;
    }
}
```

### Custom high-performance structs
```csharp
public readonly ref struct StringTokenizer
{
    private readonly ReadOnlySpan<char> _text;
    private readonly char _separator;
    
    public StringTokenizer(ReadOnlySpan<char> text, char separator)
    {
        _text = text;
        _separator = separator;
    }
    
    public Enumerator GetEnumerator() => new(_text, _separator);
    
    public ref struct Enumerator
    {
        private ReadOnlySpan<char> _remaining;
        private readonly char _separator;
        private ReadOnlySpan<char> _current;
        
        public Enumerator(ReadOnlySpan<char> text, char separator)
        {
            _remaining = text;
            _separator = separator;
            _current = default;
        }
        
        public ReadOnlySpan<char> Current => _current;
        
        public bool MoveNext()
        {
            if (_remaining.IsEmpty) return false;
            
            var index = _remaining.IndexOf(_separator);
            if (index == -1)
            {
                _current = _remaining;
                _remaining = default;
            }
            else
            {
                _current = _remaining.Slice(0, index);
                _remaining = _remaining.Slice(index + 1);
            }
            
            return true;
        }
    }
}

// Zero-allocation string tokenization
void TokenizeString()
{
    var text = "one,two,three,four".AsSpan();
    
    foreach (var token in new StringTokenizer(text, ','))
    {
        Console.WriteLine(token.ToString()); // ToString() allocates, but tokenization doesn't
    }
}
```

---

## 5. Porównanie kompilacji IL

### Regular struct vs readonly struct
```csharp
// Regular struct method call
public struct RegularStruct
{
    public int Value;
    public readonly int GetValue() => Value;
}

// IL dla wywołania z readonly context:
// ldarg.0
// ldobj RegularStruct  // Defensive copy!
// call GetValue

// readonly struct method call
public readonly struct ReadonlyStruct
{
    public readonly int Value;
    public int GetValue() => Value;
}

// IL dla wywołania z readonly context:
// ldarg.0
// call GetValue  // No defensive copy needed
```

---

## 6. Best Practices i wzorce

### ✅ Recommended patterns
```csharp
// 1. readonly struct for immutable value types
public readonly struct Money
{
    public readonly decimal Amount;
    public readonly string Currency;
    
    public Money(decimal amount, string currency) => 
        (Amount, Currency) = (amount, currency);
    
    public Money Add(Money other)
    {
        if (Currency != other.Currency) 
            throw new InvalidOperationException("Currency mismatch");
        return new Money(Amount + other.Amount, Currency);
    }
}

// 2. ref struct for high-performance temporary operations
public ref struct JsonWriter
{
    private Span<byte> _buffer;
    private int _position;
    
    public JsonWriter(Span<byte> buffer)
    {
        _buffer = buffer;
        _position = 0;
    }
    
    public void WriteProperty(ReadOnlySpan<char> name, int value)
    {
        // Write directly to buffer without allocations
        WriteString(name);
        WriteColon();
        WriteInt32(value);
    }
}

// 3. readonly ref struct for read-only high-performance operations
public readonly ref struct ConfigurationReader
{
    private readonly ReadOnlySpan<char> _configData;
    
    public ConfigurationReader(ReadOnlySpan<char> data) => _configData = data;
    
    public ReadOnlySpan<char> GetValue(ReadOnlySpan<char> key)
    {
        // Parse configuration without allocations
        return ParseValue(key);
    }
}
```

### ❌ Anti-patterns
```csharp
// Don't: Mutable readonly struct fields via properties
public readonly struct BadReadonlyStruct
{
    private readonly List<int> _list; // Reference type field is readonly, but content is mutable
    
    public BadReadonlyStruct(List<int> list) => _list = list;
    
    public void AddItem(int item) => _list.Add(item); // Mutates content!
}

// Don't: ref struct in async methods
public ref struct AsyncNotAllowed
{
    // This cannot be used in async methods
}

// Don't: Large readonly structs without ref
public readonly struct TooBigStruct
{
    // 100+ bytes struct - should consider ref passing
    public readonly int Field1, Field2, /*... Field25 */;
}
```

---

## 7. Performance implications i measurements

### Memory allocation comparison
```csharp
[MemoryDiagnoser]
public class StructPerformanceComparison
{
    private readonly RegularStruct[] _regularStructs;
    private readonly ReadonlyStruct[] _readonlyStructs;
    
    [Benchmark(Baseline = true)]
    public int ProcessRegularStructs()
    {
        var sum = 0;
        foreach (ref readonly var item in _regularStructs.AsSpan())
        {
            sum += item.Calculate(); // Defensive copies
        }
        return sum;
    }
    
    [Benchmark]
    public int ProcessReadonlyStructs()
    {
        var sum = 0;
        foreach (ref readonly var item in _readonlyStructs.AsSpan())
        {
            sum += item.Calculate(); // No defensive copies
        }
        return sum;
    }
    
    [Benchmark]
    public void ProcessWithRefStruct()
    {
        Span<byte> buffer = stackalloc byte[1024]; // Stack allocation
        var processor = new RefStructProcessor(buffer);
        processor.Process(); // Zero heap allocations
    }
}
```

---

## 8. Interop z .NET Core/5+ APIs

### Integration z System.Memory
```csharp
public readonly ref struct MemoryProcessor<T>
{
    private readonly ReadOnlySpan<T> _memory;
    
    public MemoryProcessor(ReadOnlyMemory<T> memory) => 
        _memory = memory.Span;
    
    public MemoryProcessor(T[] array) => 
        _memory = array.AsSpan();
    
    public MemoryProcessor(ReadOnlySpan<T> span) => 
        _memory = span;
    
    // Unified processing regardless of source
    public int CountNonDefault()
    {
        var count = 0;
        for (var i = 0; i < _memory.Length; i++)
        {
            if (!EqualityComparer<T>.Default.Equals(_memory[i], default(T)))
                count++;
        }
        return count;
    }
}
```

---

## Podsumowanie

| Typ | Mutability | Allocation | Defensive Copies | Use Cases |
|-----|------------|------------|------------------|-----------|
| **struct** | Mutable | Stack/Heap | Yes (readonly contexts) | General value types |
| **readonly struct** | Immutable | Stack/Heap | No | Immutable value types, math, coordinates |
| **ref struct** | Mutable | Stack only | Depends on fields | High-performance temporary operations |
| **readonly ref struct** | Immutable | Stack only | No | Ultimate performance for read operations |

**readonly struct** eliminuje defensive copying i gwarantuje immutability, idealny dla value objects.

**ref struct** zapewnia stack-only semantics dla zero-allocation scenarios, fundament dla Span<T>/Memory<T>.

**readonly ref struct** łączy oba podejścia dla maksymalnej wydajności w read-only, short-lived operations.

Developer .NET powinien wykorzystywać te typy w high-performance scenarios, szczególnie w hot paths, parsingu, matematyce i przetwarzaniu dużych danych gdzie garbage collection może być problematyczny.