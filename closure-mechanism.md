# Mechanizm Closure w C#

## Wprowadzenie
W języku C# **closure** to technika umożliwiająca funkcjom lub wyrażeniom lambda przechowywanie i dostęp do zmiennych z zewnętrznego kontekstu, nawet po zakończeniu życia tego kontekstu. Mechanizm ten jest kluczowy dla funkcjonalnego stylu programowania i zaawansowanych wzorców asynchronicznych.

---

## 1. Definicja i semantyka

- **Closure** to połączenie:
  - Funkcji (wyrażenia lambda lub delegata),
  - Ze środowiskiem, w którym funkcja została zdefiniowana.

- Zmienne lokalne, parametry lub zmienne w bloku definiującym lambdę są *zamykane* (captured) i długowieczne, tzn. istnieją tak długo, jak długo istnieje referencja do delegata.

---

## 2. Przechwytywanie zmiennych

- C# przechwytuje zmienne **przez odniesienie** (reference capture), a nie przez wartość:

```csharp
Func<int> CreateCounter()
{
    int count = 0;               // zmienna lokalna
    return () => ++count;        // lambda zamyka count
}

var counter1 = CreateCounter();
Console.WriteLine(counter1());  // 1
Console.WriteLine(counter1());  // 2

var counter2 = CreateCounter();
Console.WriteLine(counter2());  // 1
Console.WriteLine(counter1());  // 3 (wspólne środowisko counter1)
```

W powyższym przykładzie każdy delegat trzyma własną kopię zmiennej `count` w wygenerowanej klasie pomocniczej.

---

## 3. Pod spodem: klasy pomocnicze kompilerowe

- Kompilator C# generuje **ukryte klasy** (closure classes) dla każdej zmiennej zamkniętej przez lambdę.
- Pola tych klas odpowiadają przechwyconym zmiennym. Każde wywołanie metody tworzy nową instancję tej klasy, o ile metoda definiująca lambdę jest instancją metody.

Ilustracja generowanego kodu:

```csharp
private class DisplayClass
{
    public int count;
    public int Lambda() => ++count;
}

Func<int> CreateCounter()
{
    var c = new DisplayClass();
    c.count = 0;
    return new Func<int>(c.Lambda);
}
```

---

## 4. Potencjalne pułapki i best practices

- **Problem utrzymania referencji**: przechwycone zmienne żyją tak długo, jak delegat. Może to prowadzić do wycieków pamięci, gdy zamknięcie żyje dłużej niż zakładano.
- **Zmienne iteracyjne** w pętlach:

```csharp
var actions = new List<Action>();
for (int i = 0; i < 3; i++)
{
    actions.Add(() => Console.WriteLine(i));
}

foreach (var action in actions)
    action();  // wypisze '3' trzy razy
```

Aby poprawić:
```csharp
for (int i = 0; i < 3; i++)
{
    int copy = i;
    actions.Add(() => Console.WriteLine(copy));
}
```

- **Unikanie niezamierzonych efektów ubocznych**: w miarę możliwości stosować *immutability* lub kopiować wartości.

---

## 5. Zastosowania zaawansowane

1. **Asynchroniczne API**: przekazywanie kontekstu do kontynuacji (`async/await`).
2. **Wzorce Rx (Reactive Extensions)**: definiowanie transformacji strumieni.
3. **Konfiguracja i DSL**: API fluente pozwalają na budowanie zagnieżdżonych konfiguracji.

---

## Podsumowanie
Mechanizm **closure** w C# pozwala na tworzenie funkcji zachowujących stan zewnętrzny w bezpieczny sposób, otwierając drzwi do stylu funkcyjnego i bardziej deklaratywnych wzorców programowania. Świadomość działania zamknięć — zwłaszcza generowanych klas pomocniczych i przechowywania zmiennych przez referencję — jest niezbędna dla senior developerów w celu unikania pułapek związanych z zarządzaniem pamięcią i nieoczekiwanym zachowaniem.