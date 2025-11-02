# virtual – Kompendium dla Developera .NET

## Wprowadzenie
**virtual** to słowo kluczowe C# umożliwiające **polimorfizm dyspozycyjny** (dispatch polymorphism). Metody virtualne pozwalają na zmianę implementacji w klasach pochodnych, przy czym rzeczywista metoda wywoływana jest określana w runtime na podstawie typu obiektu, a nie typu referencji. Wymaga to skomplikowanego mechanizmu za kulisami – **vtable** (virtual method table) i **virtual dispatch**.

---

## 1. Podstawowy mechanizm virtual

### Non-virtual vs virtual
```csharp
public class Animal
{
    // Non-virtual - binding w compile-time
    public void NonVirtualSound() => Console.WriteLine("Generic sound");
    
    // Virtual - binding w runtime
    public virtual void VirtualSound() => Console.WriteLine("Generic sound");
}

public class Dog : Animal
{
    // Override virtual method
    public override void VirtualSound() => Console.WriteLine("Woof!");
}

// Użycie
Animal animal = new Dog();
animal.NonVirtualSound();  // Generic sound (compile-time binding)
animal.VirtualSound();     // Woof! (runtime dispatch)
```

### Semantyka virtual dispatch
```csharp
// Compile-time: kompilator wie że animal to Animal
// Runtime: CLR sprawdza rzeczywisty typ (Dog)
// i wywołuje Dog.VirtualSound()

Animal animal = new Dog();
animal.VirtualSound();  // Woof!

// Wewnętrznie:
// 1. Sprawdź typ obiektu (Dog)
// 2. Wyszukaj VirtualSound w Dog's vtable
// 3. Jeśli nie ma, szukaj w Animal's vtable
// 4. Wywołaj znalezioną metodę
```

---

## 2. Virtual Method Table (vtable) – architektura

### Ukryta struktura
```csharp
public class Base
{
    public virtual void Method1() { }
    public virtual void Method2() { }
    public int Data;
}

public class Derived : Base
{
    public override void Method1() { }  // Override
    // Method2 - inherited virtual
}

// Wewnętrzna reprezentacja:
// Base object layout:
// [MethodTable ptr] -> Base's vtable
// [Field: Data]
//
// Base's vtable:
// [0] Object.GetHashCode
// [1] Object.Equals
// [2] Object.ToString
// [3] Base.Method1
// [4] Base.Method2
// ...
//
// Derived's vtable:
// [0] Object.GetHashCode
// [1] Object.Equals
// [2] Object.ToString
// [3] Derived.Method1 (overridden)
// [4] Base.Method2 (inherited)
// ...
```

### Object Header (SyncBlock Index i MethodTable)
```csharp
// Każdy managed object ma header:
// [Sync Block Index (4 bytes)] - dla locking, finalization
// [MethodTable Pointer (8 bytes na x64)] - wskazuje na vtable

object obj = new Dog();
// Wewnętrnie obj memory layout:
// Offset 0x00: SyncBlockIndex (for synchronization/GC)
// Offset 0x04: MethodTable* (points to Dog's vtable)
// Offset 0x0C: Instance fields
```

---

## 3. Virtual dispatch – low-level IL i CPU

### IL bytecode dla virtual call
```csharp
public void CallVirtual(Animal animal)
{
    animal.VirtualSound();
}

// Wygenerowany IL:
// ldarg.0              ; Load 'animal' reference
// callvirt Animal::VirtualSound()
// ret

// Non-virtual comparison:
// ldarg.0              ; Load 'animal' reference
// call Animal::NonVirtualSound()  ; Different opcode!
// ret
```

### CPU-level virtual dispatch
```
1. Load object address (w rejestrze rcx na x64)
2. Load MethodTable pointer z object header (offset 0)
3. Load function pointer z vtable (index * sizeof(void*))
4. Call function pointer (indirect call - jmp reg)

Simplified assembly:
mov   rcx, [animal]           ; Load object reference
mov   r11, [rcx]              ; Load MethodTable pointer
call  qword [r11 + methodIndex * 8]  ; Virtual call
```

---

## 4. Performance implications

### Virtual dispatch overhead
```csharp
[Benchmark]
public void NonVirtualCall()
{
    Animal animal = new Dog();
    for (int i = 0; i < 1_000_000; i++)
    {
        animal.NonVirtualSound();  // Direct call
    }
}

[Benchmark]
public void VirtualCall()
{
    Animal animal = new Dog();
    for (int i = 0; i < 1_000_000; i++)
    {
        animal.VirtualSound();  // Virtual dispatch
    }
}

// Wyniki (przykładowe):
// NonVirtualCall:  ~1.2 ms  (direct IL instruction)
// VirtualCall:     ~3.8 ms  (3-4x wolniej!)
```

### Przyczyny overhead'u
1. **Indirect addressing** - musi załadować wskaźnik z vtable
2. **Branch prediction difficulty** - CPU nie wie gdzie skok
3. **L1 cache miss** - vtable mogą nie być w cache
4. **Instruction level parallelism** - indirect call blokuje pipeline

---

## 5. JIT optimization – Devirtualization

### Compile-time devirtualization
```csharp
// Sealed class - JIT wie że nie będzie override
public sealed class DogSealed : Animal
{
    public override void VirtualSound() => Console.WriteLine("Woof!");
}

public void OptimizedCall()
{
    DogSealed dog = new DogSealed();
    dog.VirtualSound();  // JIT devirtualizes - direct call!
}

// JIT widzi sealed i generuje:
// direct call do DogSealed.VirtualSound
// Bez vtable lookup!
```

### Tiered compilation devirtualization
```csharp
public void CallThroughReference(Animal animal)
{
    animal.VirtualSound();
}

// T0 JIT (interpreter or simple code):
// Full virtual dispatch - callvirt

// T1 JIT (optimizing compiler):
// Type feedback - jeśli zawsze Dog, może devirtualize
// Speculative optimization + guard check:
// if (animal.GetType() == typeof(Dog))
// {
//     call Dog.VirtualSound() directly
// }
// else
// {
//     fallback to virtual dispatch
// }
```

---

## 6. Abstract vs virtual

### Virtual metoda – ma implementację
```csharp
public class Animal
{
    public virtual void Sound()
    {
        Console.WriteLine("Generic animal sound");  // Default impl
    }
}

public class Dog : Animal
{
    // Może nie override - użyje Animal.Sound()
}

var dog = new Dog();
dog.Sound();  // Generic animal sound
```

### Abstract metoda – brak implementacji
```csharp
public abstract class Animal
{
    public abstract void Sound();  // Nie może mieć ciała
}

public class Dog : Animal
{
    public override void Sound() => Console.WriteLine("Woof!");  // Musi override
}

// Animal animal = new Animal();  // BŁĄD - nie można instancjonować
```

---

## 7. Virtual w kontekście interface

### Interface vs virtual class
```csharp
// Interface - nie ma vtable per se, ale similar mechanism
public interface IAnimal
{
    void Sound();
}

public class Dog : IAnimal
{
    public void Sound() => Console.WriteLine("Woof!");
}

// Call through interface:
IAnimal animal = new Dog();
animal.Sound();  // Virtual dispatch (interface dispatch)

// Wewnętrnie - different mechanism:
// - VTable lookup dla interface methods
// - Extra indirection (Interface Method Dispatch)
// - Może być wolniejsze niż direct virtual
```

---

## 8. Override vs new

### virtual + override
```csharp
public class Animal
{
    public virtual void Sound() => Console.WriteLine("Animal");
}

public class Dog : Animal
{
    public override void Sound() => Console.WriteLine("Woof!");
}

Animal animal = new Dog();
animal.Sound();  // Woof! - polymorphic
```

### virtual + new (typically mistake)
```csharp
public class Animal
{
    public virtual void Sound() => Console.WriteLine("Animal");
}

public class Dog : Animal
{
    public new void Sound() => Console.WriteLine("Woof!");  // new, nie override!
}

Animal animal = new Dog();
animal.Sound();  // Animal (nie Woof!)
Dog dog = new Dog();
dog.Sound();     // Woof!

// new - tylko "hiduje" metodę, nie overriduje
// Reference type określa który zaś się wybiera (compile-time)
```

---

## 9. Performance tuning strategies

### 1. Sealed classes
```csharp
public sealed class Dog : Animal  // JIT może devirtualize
{
    public override void Sound() => Console.WriteLine("Woof!");
}

// JIT optymalizuje do direct call
```

### 2. Prefer composition over virtual
```csharp
// Zamiast virtual:
public abstract class Logger
{
    public abstract void Log(string msg);
}

// Use strategy/composition:
public interface ILogWriter
{
    void Write(string msg);
}

public class Logger
{
    private readonly ILogWriter _writer;
    
    public Logger(ILogWriter writer) => _writer = writer;
    public void Log(string msg) => _writer.Write(msg);
}
```

### 3. Inline small virtual methods
```csharp
public class Base
{
    // Mały virtual - może być inlined nawet pomimo virtual
    public virtual int GetValue() => 42;
}

// JIT heuristics - inlines jeśli method small enough
// nawet jeśli virtual
```

### 4. Constrain with generics
```csharp
// Zamiast:
public void ProcessAnimals(Animal[] animals)
{
    foreach (var animal in animals)
        animal.Sound();  // Virtual dispatch per item
}

// Use generic (monomorphic):
public void ProcessAnimals<T>(T[] animals) where T : Animal
{
    foreach (var animal in animals)
        animal.Sound();  // JIT generates specialized code
}
```

---

## 10. Best practices

### ✅ Recommended
```csharp
// 1. Use virtual dla extension points, nie dla implementation details
public abstract class DataProcessor
{
    public void Process(byte[] data)
    {
        Validate(data);        // May be virtual for subclass customization
        var result = Parse(data);  // Core logic - consider non-virtual
        Store(result);         // Non-virtual
    }
    
    protected virtual void Validate(byte[] data) { }
    protected abstract T Parse(byte[] data);
}

// 2. Seal derived classes
public sealed class JsonProcessor : DataProcessor
{
    protected override void Validate(byte[] data) { }
    protected override T Parse(byte[] data) { }
}

// 3. Template method pattern
public abstract class Reader
{
    public void Read(Stream stream)
    {
        var data = stream.ReadAllBytes();      // Non-virtual (common)
        var parsed = ParseData(data);          // Virtual (customization)
        ApplyFilters(parsed);                  // Virtual (optional customization)
    }
    
    protected abstract T ParseData(byte[] data);
    protected virtual void ApplyFilters(T data) { }
}
```

### ❌ Anti-patterns
```csharp
// Don't: Over-virtualize trivial methods
public class Container
{
    public virtual int Count => items.Count;  // Usually no need for virtual
    public virtual void Clear() => items.Clear();  // Not an extension point
}

// Don't: Virtual in high-frequency paths without profiling
public class Vector
{
    public virtual float Magnitude
    {
        get => Math.Sqrt(X * X + Y * Y);  // Called millions of times
    }
    // Performance-critical - avoid virtual here!
}

// Don't: Virtual on sealed class
public sealed class Dog : Animal
{
    public virtual void Sound() { }  // Why? Sealed prevents override!
}
```

---

## 11. Wgląd w CLR – MethodTable structure

```csharp
// Uproszczona struktura MethodTable:
struct MethodTable
{
    // Metadata flags
    uint Flags;                           // Bit flags (sealed, abstract, interface, etc)
    uint BaseClassOffset;                 // Offset to base class MethodTable
    ushort NumVirtuals;                   // Number of virtual methods
    ushort NumInterfaces;                 // Number of interface implementations
    
    // Virtual method slots (array)
    IntPtr[] VirtualMethods;              // [0] = Method1, [1] = Method2, etc.
    
    // Interface dispatch data
    InterfaceMap[] InterfaceMaps;
};

// Dla każdego objectu:
object header:
    uint SyncBlockIndex;
    IntPtr MethodTablePtr;  // Points to type's MethodTable
```

---

## 12. Benchmark porównawczy

```csharp
[MemoryDiagnoser]
public class VirtualBenchmarks
{
    private sealed class DogSealed : Animal { /* ... */ }
    private class DogNonSealed : Animal { /* ... */ }
    
    [Benchmark(Baseline = true)]
    public void DirectMethodCall()
    {
        var dog = new DogSealed();
        for (int i = 0; i < 1000; i++)
            dog.VirtualSound();  // Direct - JIT can devirtualize
    }
    
    [Benchmark]
    public void VirtualThroughReference()
    {
        Animal dog = new DogSealed();
        for (int i = 0; i < 1000; i++)
            dog.VirtualSound();  // Virtual dispatch required
    }
    
    [Benchmark]
    public void VirtualThroughNonSealedClass()
    {
        Animal dog = new DogNonSealed();
        for (int i = 0; i < 1000; i++)
            dog.VirtualSound();  // Virtual - cannot devirtualize
    }
}

// Spodziewane rezultaty:
// DirectMethodCall:                  ~1.2 ms (baseline)
// VirtualThroughReference:           ~3.8 ms (3x wolniej)
// VirtualThroughNonSealedClass:      ~4.1 ms (3.4x wolniej)
```

---

## Podsumowanie

**virtual** w .NET to kluczowy mechanizm polimorfizmu:

- **Kompilator** generuje `callvirt` IL opcode
- **Runtime** wyszukuje metodę w vtable obiektu
- **CPU** wykonuje indirect call
- **Performance cost** – 2-4x wolniej niż direct call

**Mechanizm:**
1. Object header zawiera MethodTable pointer
2. MethodTable zawiera vtable (array metod)
3. Virtual dispatch: załaduj MethodTable → załaduj slot → indirect call

**Optymalizacje JIT:**
- **Sealed classes** – devirtualization
- **Tiered compilation** – speculative inlining
- **Type feedback** – runtime optimization

**Best practices:**
- Używaj virtual dla extension points, nie details
- Seal derived classes by default
- Profiluj performance-critical code
- Rozważ composition/dependency injection zamiast virtual
- Understand virtual dispatch cost w gorących ścieżkach

Developer powinien rozumieć zarówno high-level semantykę virtual, jak i low-level mechanizm vtable dispatch dla świadomych decyzji architektonicznych i optymalizacyjnych.