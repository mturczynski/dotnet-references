# Span<T> w .NET

## Wprowadzenie
`Span<T>` to **ref struct** wprowadzony w C# 7.2/.NET Core 2.1, umożliwiający **stack-only**, **zero-allocation** reprezentację ciągłego obszaru pamięci zawierającego elementy typu `T`. Zaprojektowany z myślą o wydajności, eliminuje konieczność alokacji na heap i minimalizuje kopiowanie danych.

---

## 1. Charakterystyka i semantyka

- **ref struct**: alokowany wyłącznie na stosie, nie może być boxed ani przechowywany w heapowych strukturach.
- **Read/Write**: wspiera odczyt i zapis elementów.
- **Slice**: szybkie wycinki bez kopiowania.
- **Interop**: może reprezentować dowolne ciągłe dane (tablica, `stackalloc`, `Memory<T>`).

```csharp
// Tworzenie Span<T>
int[] array = new int[100];
Span<int> spanFromArray = array;

Span<byte> spanFromStack = stackalloc byte[256];
ReadOnlySpan<char> spanFromString = "Hello, World!";

// Slice
Span<int> slice = spanFromArray.Slice(10, 20);

// Modyfikacja
slice[0] = 42;
```

---

## 2. Zastosowania i korzyści

| Scenariusz                               | Korzyść                                                          |
|------------------------------------------|------------------------------------------------------------------|
| Parsowanie danych (CSV, JSON, XML)        | Przetwarzanie bez alokacji, minimalizacja GC pressure             |
| Operacje na buforach (I/O, sieć)         | Zero-copy slicing, stackalloc dla małych buforów                 |
| High-performance algorithms (Signal Processing) | SIMD, `Span<T>` w połączeniu z `Vector<T>`                      |
| Interoperacyjność (Interop, P/Invoke)    | Bezpieczne referencje do niezarządzanych buforów                 |

- **Zero-copy slicing**: `Slice` nie tworzy kopii danych.  
- **Stackalloc**: alokacja bufora na stosie za pomocą `stackalloc`.  
- **Brak boxingu**: nie można przekazać do `object` czy `dynamic`.

---

## 3. Ograniczenia i pułapki

| Ograniczenie                           | Opis                                                                  |
|----------------------------------------|----------------------------------------------------------------------|
| **Heap storage**                       | Nie można przechowywać w polach klas ani w kolekcjach heapowych       |
| **Async/Iterator**                     | Nie można używać jako parametr/metoda async lub iterator (yield)      |
| **Boxing**                             | Brak możliwości boxing do `object`, interfejsów, delegate             |
| **Ref safety**                         | Nie może być przekazywany poza kontekst stosu (np. po wyjściu z metody) |
| **Multi-dimensional slices**           | Brak wbudowanego wsparcia; trzeba używać `Span2D` od Community Toolkit |

---

## 4. Przykłady użycia

### Parsowanie CSV bez alokacji
```csharp
public IEnumerable<ReadOnlySpan<char>> Tokenize(ReadOnlySpan<char> input)
{
    int start = 0;
    while (start < input.Length)
    {
        int comma = input.Slice(start).IndexOf(',');
        if (comma == -1) comma = input.Length - start;
        yield return input.Slice(start, comma);
        start += comma + 1;
    }
}
```

### Operacje na buforze sieciowym
```csharp
public void ProcessData(Span<byte> buffer)
{
    // Odczyt nagłówka
    var header = buffer.Slice(0, 4);
    int length = BinaryPrimitives.ReadInt32LittleEndian(header);
    
    // Przetwarzanie payload
    var payload = buffer.Slice(4, length);
    // ...
}

// Wywołanie z stackalloc
Span<byte> networkBuffer = stackalloc byte[1024];
ReceiveData(networkBuffer);
ProcessData(networkBuffer);
```

---

## 5. Performance considerations

- **JSON parsing**: `Span<byte>` w `Utf8JsonReader` eliminuje alokacje stringów.  
- **Regex**: `ReadOnlySpan<char>` w `Regex.Match` reduces allocations.  
- **SIMD**: `Span<T>` współpracuje z `Vector<T>` dla wektoryzacji.

Benchmark wykazuje znaczące redukcje alokacji GC i przyspieszenie w gorących ścieżkach.

---

## 6. Best Practices

1. **Prefer `Span<T>` w gorących ścieżkach**: parsing, I/O, obliczenia.  
2. **Używaj `stackalloc` dla tymczasowych buforów** < 1 KB.  
3. **Konwertuj tablice do `Span<T>`** zamiast `ArraySegment<T>`.  
4. **Nigdy nie przechowuj `Span<T>` w polach klasy** – powoduje błędy kompilacji.  
5. **Rozważ `Memory<T>`** jeśli potrzebujesz heap storage i async.

---

## 7. Podsumowanie

`Span<T>` to kluczowy element high-performance .NET:
- **Zero-allocation** i **stack-only** semantics.  
- **Brak kopiowania** dzięki slicing.  
- **Optymalizacje** dla parsing, I/O i obliczeń.  
- **Ograniczenia**: brak heap storage, async, boxing.

Developer .NET powinien:
- Wykorzystywać `Span<T>` dla redukcji GC pressure.
- Łączyć z `stackalloc`, `Vector<T>`, `Utf8JsonReader`.
- Mądrze dobierać `Memory<T>` tam, gdzie `Span<T>` jest niewystarczający.
