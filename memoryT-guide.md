# Memory<T> – Odpowiednik Span<T> dla sterty w .NET

## Wprowadzenie
`Memory<T>` to **heap-friendly** odpowiednik `Span<T>`, wprowadzony w .NET Core 2.1/C# 7.2. Umożliwia reprezentację ciągłego obszaru pamięci (tablica, segmenty `MemoryPool<T>`, `Array`), ale w przeciwieństwie do `Span<T>`, może być przechowywany na heap, przekazywany do asynchronicznych metod oraz zapisywany w polach klas.

---

## 1. Podstawowe cechy

- **Heap allocation**: struktura (nie ref struct), może być fieldem, zwracana z metod async.
- **Access przez `Span<T>`**: operacje odczytu/zapisu przez metodę `.Span` lub `.Slice`.
- **Nullable**: można mieć `null` dla `Memory<T>?`, odpowiednik `default(Memory<T>)` to pusty segment.

```csharp
// Tworzenie Memory<T>
Memory<int> memFromArray = new int[100];
Memory<byte> memFromOwner = MemoryPool<byte>.Shared.Rent(256).Memory;

// Uzyskanie Span<T>
Span<int> span = memFromArray.Span;

// Slice
Memory<int> sliceMem = memFromArray.Slice(10, 20);
```

---

## 2. Zastosowania i korzyści

| Scenariusz                               | Korzyść                                                        |
|------------------------------------------|----------------------------------------------------------------|
| Asynchroniczne przetwarzanie I/O         | Możliwość przekazania bufora do `async`/`await`               |
| Przechowywanie stanów w obiektach         | `Memory<T>` można trzymać jako pole w klasie                   |
| Buffering i pooling                      | Współpraca z `MemoryPool<T>` i reusable buffers               |

- **No boxing**: brak konwersji do `object`.
- **Thread-safe**: segmenty pamięci mogą być udostępniane między wątkami.
- **Asynchroniczność**: bezpieczne użycie w asynchronicznych metodach.

---

## 3. Integracja z API

### Asynchroniczne I/O
```csharp
public async Task ReadDataAsync(Memory<byte> buffer)
{
    int bytesRead = await stream.ReadAsync(buffer);
    // buffer.Span[0..bytesRead] dostępne po odczycie
}
```

### Parsowanie danych
```csharp
public void ProcessData(Memory<char> memory)
{
    ReadOnlySpan<char> span = memory.Span;
    // Parsowanie CSV jak w Span<T>
}
```

---

## 4. Ograniczenia

| Ograniczenie                           | Opis                                                              |
|----------------------------------------|--------------------------------------------------------------------|
| **Alokacje**                          | `Memory<T>` sam w sobie to struktura, ale użycie `new T[]` alokuje tablicę |
| **Overhead**                          | Nieco większy narzut względem `Span<T>` (dwa pola: obiekt i start) |
| **Mutable buffer misuse**             | Konieczność ostrożnej pracy z `MemoryPool<T>` by unikać wycieków    |

---

## 5. Praktyczne przykłady

### Buforowanie w klasie
```csharp
public class DataBuffer
{
    private readonly Memory<byte> _buffer;

    public DataBuffer(int size)
    {
        _buffer = MemoryPool<byte>.Shared.Rent(size).Memory;
    }

    public async Task FillAsync(Stream stream)
    {
        await stream.ReadAsync(_buffer);
    }

    public void Process()
    {
        Span<byte> span = _buffer.Span;
        // Przetwarzaj dane
    }
}
```

### Łączenie segmentów pamięci
```csharp
public Memory<byte> Concat(Memory<byte> a, Memory<byte> b)
{
    byte[] result = new byte[a.Length + b.Length];
    a.Span.CopyTo(result);
    b.Span.CopyTo(result.AsSpan(a.Length));
    return result;
}
```

---

## Podsumowanie

`Memory<T>` to kluczowy element biblioteki **System.Memory**, umożliwiający **heap storage**, **asynchroniczne** i **bezpieczne** przekazywanie buforów. W połączeniu z `Span<T>`, `MemoryPool<T>` i **Pipelines** stanowi fundament wydajnego, zero-allocation przetwarzania danych w nowoczesnym .NET.

Developer powinien:  
- Wybierać `Memory<T>` dla asynchronicznych i długotrwałych scenariuszy,  
- Współpracować z `Span<T>` dla hot paths,  
- Wykorzystywać pooling (`MemoryPool<T>`) dla redukcji GC pressure.