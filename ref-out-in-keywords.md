# out, ref, in w C#

## Wprowadzenie
Słowa kluczowe `out`, `ref` oraz `in` służą do przekazywania argumentów przez referencję do metod, co daje kontrolę nad mutowalnością i semantyką przekazywanych danych na poziomie ABI (Application Binary Interface) i wydajności. Mają wpływ zarówno na składnię, bytecode IL, jak i na optymalizację pamięci w zaawansowanych scenariuszach.

---

## 1. ref – przekazywanie przez referencję (czytanie/zapisywanie)

- **Definicja:** Pozwala na przekazanie zmiennej przez referencję umożliwiając zarówno odczyt, jak i zapis w metodzie.
- **Wymogi:** Zmienna musi być zainicjowana przed wywołaniem (parameter passing by alias).

```csharp
public void Increment(ref int value)
{
    value++;
}

int number = 10;
Increment(ref number); // number == 11
```

- **Zastosowania:**
  - Mutacja struktury (np. struct zamiast klasy dla performance)
  - optymalizacja unikająca kopii dużych value types,
  - przekazywanie wskaźnika do zmiennej.
- **Performance:** pozwala na modyfikację bez boxingu/unboxingu, co jest szczególnie istotne dla dużych struktur.

---

## 2. out – przez referencję, tylko zapis (write-only output semantics)

- **Definicja:** Pozwala na przekazanie zmiennej, której wartość zostanie ustawiona przez metodę – wymagane przypisanie wartości przed powrotem z metody.
- **Wymogi:** Nie trzeba inicjalizować zmiennej przed wywołaniem; metoda musi ją przypisać.

```csharp
public bool TryParseInt(string input, out int result)
{
    return int.TryParse(input, out result);
}

if (TryParseInt("42", out int value))
{
    // value == 42
}
```

- **Scenariusze:**
  - Wzorce Try-Parse (`TryGet`, `TryParse`)
  - wielokrotne zwroty wartości z metody (de facto "multiple return")
- **IL:** Argument jest przekazywany przez wskaźnik z semantyką write-only

---

## 3. in – przez referencję, tylko do odczytu (read-only by ref)

- **Definicja:** Od C# 7.2, pozwala na przekazanie dużych struktur przez referencję, ale z gwarancją, że metoda nie zmodyfikuje danych.
- **Semantyka:** Chwilowa alias dla zmiennej, ale TYLKO do odczytu (kompilator wymusza brak mutacji nawet na structach z mutable fields).

```csharp
public bool IsOrigin(in Point3D point)
{
    // point.X = 5; // błąd kompilacji
    return point.X == 0 && point.Y == 0 && point.Z == 0;
}

Point3D p = new Point3D(0, 0, 0);
IsOrigin(in p);
```

- **Performance:**
  - Eliminacja niepotrzebnych kopii dużych struktur,
  - Współdziała z `readonly struct` dla pełnej niezmienności,
  - Wpływa na wygenerowany kod IL (ref readonly argument).

---

## 4. ref readonly, ref struct, in dla zaawansowanych

- **ref readonly:** metoda zwraca referencję tylko do odczytu (np. slice tablic bez kopii)
- **ref struct, readonly struct:** typy ograniczone do stack-only semantics, często z `Span<T>`/`ReadOnlySpan<T>`
- Przekazywanie przez referencję jest kluczowe w nowoczesnych API .NET (np. `Span<T>`, `Memory<T>`, linia SIMD, ImageSharp, matematyka).

---

## 5. Porównanie – tabela

| Słowo kluczowe | Wymagana inicjalizacja? | Czy można czytać? | Czy można pisać? | Przechowywanie w IL |
|----------------|------------------------|------------------|------------------|---------------------|
| ref            | Tak                    | Tak              | Tak              | ByRef (mutowalny)   |
| out            | Nie                    | Nie (przed assign)| Tak              | ByRef (write-only)  |
| in             | Tak                    | Tak              | Nie              | ByRef (readonly)    |


---

## 6. Best practices
- Używaj `ref`/`in` do przekazywania dużych struktur w gorących ścieżkach/performance (
np. algorytmy, obliczenia, streaming, collections API)
- `out` do semantyki Try-Parse i „wielu wyników”
- W nowoczesnych strukturach (Span, SIMD, slicing) zawsze preferuj przekazywanie przez referencję.
- Do komunikacji między wątkami preferuj immutable types (ref/in – tylko lokalnie/stack).

## Podsumowanie
Słowa kluczowe `out`, `ref`, `in` pozwalają na kontrolowany dostęp do zmiennych przez referencję i powinny być stosowane w świadomy sposób – zarówno dla czytelności API, jak i wydajności (dotyczy głównie value types – struct). Użycie ich w kodzie produkcyjnym wymaga znajomości semantyki IL i konsekwencji w projektowaniu kontraktów/metod.