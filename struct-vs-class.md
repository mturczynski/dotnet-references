# Struktura vs Klasa w .NET C#

## Wprowadzenie
W .NET C# zarówno **struktury** (`struct`) jak i **klasy** (`class`) umożliwiają definiowanie własnych typów danych. Mimo podobnej składni istnieją istotne różnice w ich semantyce, zachowaniu w pamięci i scenariuszach użycia.

---

## 1. Typ wartościowy vs typ referencyjny

- **Struktura** jest **typem wartościowym**, co oznacza:
  - Przechowywana jest bezpośrednio na stosie (ang. stack) lub **inline** w tablicach/obiektach zawierających.
  - Przekazywana przez **kopię** przy przypisaniu lub przekazaniu jako argument.
  - Domyślnie nie wymaga garbage collectora; żywotność zależy od zakresu.

- **Klasa** jest **typem referencyjnym**, co oznacza:
  - Przechowywana na stercie (ang. heap).
  - Przekazywana przez **referencję** (wskaźnik).
  - Żywotność kontrolowana przez **GC**.

---

## 2. Inheritance i polimorfizm

- **Struktury**:
  - Nie wspierają dziedziczenia od innych struktur ani klas (poza `System.ValueType`).
  - Mogą implementować interfejsy.
  - Brak możliwości polimorfizmu opartego na typie bazowym.

- **Klasy**:
  - Wspierają pełne dziedziczenie (single inheritance).
  - Umożliwiają polimorfizm poprzez metody wirtualne (`virtual`/`override`) i abstrakcję.
  - Pozwalają na hierarchie typów i wzorce projektowe jak fabryka, strategia.

---

## 3. Konstruktor, inicjalizacja i domyślne wartości

- **Struktura**:
  - Automatycznie generowany **konstruktor bezparametrowy**, nie można go przesłonić.
  - Pola niejawnie inicjowane do wartości domyślnych (`0`, `false`, `null` dla referencji).
  - Możliwość definiowania konstruktorów z parametrami od C# 10.

- **Klasa**:
  - Definiowanie dowolnej liczby konstruktorów (parametryzowanych i bezparametrowych).
  - Jeśli zdefiniowany konstruktor, kompilator nie wygeneruje bezparametrowego domyślnie.

---

## 4. Koszt pamięci i wydajność

- **Struktura**:
  - Brak dodatkowej alokacji na stercie (jeśli używana lokalnie).
  - Przy dużych strukturach (np. >16–32 bajtów) koszty kopiowania mogą przewyższyć korzyści.
  - Unika fragmentacji sterty i obciążeń GC.

- **Klasa**:
  - Wymaga alokacji na stercie i późniejszego zwolnienia przez GC.
  - Idealna do dużych, zmiennych w czasie obiektów i sieci zależności (DI).

---

## 5. Boxing i unboxing

- Przypisanie struktury do zmiennej typu `object` lub interfejsu powoduje **boxing** (opakowanie w obiekt na stercie).
- Konwersja z powrotem do struktury to **unboxing**, kosztowna operacja.
- Klasy nie wymagają boxing/unboxing, ponieważ są referencjami.

---

## 6. Kiedy używać struktury, a kiedy klasy?

- **Struktury**:
  - Małe, niezmienne typy danych (np. `Point`, `DateTime`, `Guid`).
  - Brak potrzeby dziedziczenia.
  - Często używane w intensywnych obliczeniach (wektory, macierze) dla uniknięcia GC.

- **Klasy**:
  - Domenowe modele z bogatą logiką, stanem i zależnościami.
  - Wymagana hierarchia dziedziczenia lub rozszerzalność.
  - Zarządzanie cyklem życia przy pomocy DI, wzorców projektowych.

---

## 7. Przykład porównawczy
```csharp
// Struktura
public struct Point
{
    public int X { get; }
    public int Y { get; }

    public Point(int x, int y)
    {
        X = x;
        Y = y;
    }
}

// Klasa
public class Person
{
    public string Name { get; set; }
    public DateTime BirthDate { get; set; }

    public Person(string name, DateTime birthDate)
    {
        Name = name;
        BirthDate = birthDate;
    }

    public virtual int CalculateAge() =>
        (int)((DateTime.Now - BirthDate).TotalDays / 365.25);
}
```

---

## Podsumowanie
Wybór między **strukturą** a **klasą** zależy od: charakteru danych (wartość vs referencja), kosztów wydajnościowych (alokacja, kopiowanie), oraz potrzeb w zakresie dziedziczenia i polimorfizmu. Warto stosować struktury w scenariuszach małych, niezmiennych typów i klasy tam, gdzie wymagana jest elastyczność i bogata semantyka obiektowa.