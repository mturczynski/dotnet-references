# DateTime, DateTimeOffset i TimeSpan w .NET C#

## Wprowadzenie
W .NET C# mamy trzy podstawowe typy do pracy z czasem:

- **DateTime** – punkt w czasie bez jednoznacznego odniesienia do strefy czasowej.
- **DateTimeOffset** – punkt w czasie z przesunięciem od UTC.
- **TimeSpan** – odstęp czasowy (duracja).

Ten dokument przedstawia ich różnice, wewnętrzne mechanizmy i rekomendowane scenariusze użycia.

---

## 1. Reprezentacja wewnętrzna

| Typ               | Wartość przechowywana                           | Rozmiar  | Zakres                                   |
|-------------------|-------------------------------------------------|----------|------------------------------------------|
| DateTime          | Ticks (Int64) + `Kind` (Local/UTC/Unspecified)  | 8 bajtów | ±10 000 lat od 1 AD do 31 12 9999 AD     |
| DateTimeOffset    | Ticks (Int64) + `Offset` (Int16)                | 10 bajtów| Ticks w zakresie DateTime; offset ±14h   |
| TimeSpan          | Ticks (Int64)                                   | 8 bajtów | ±10 000 dni                             |

---

## 2. DateTime vs DateTimeOffset

### DateTime

- **Kind**:
  - `Unspecified` – brak kontekstu strefy.
  - `Utc` – czas UTC.
  - `Local` – czas w lokalnej strefie maszyny.
- **Zastosowanie**:
  - Proste operacje na datach bez przesunięć.
  - Serializacja wewnątrz aplikacji, gdy kontekst strefy jest znany.
- **Pułapki**:
  - Nie definiuje jednoznacznie strefy, co prowadzi do nieprzewidywalnych konwersji.
  - Konwersje między `Local` i `Utc` muszą być ręcznie kontrolowane.

### DateTimeOffset

- Przechowuje punkt w czasie + stały offset od UTC.
- **Zalety**:
  - Jednoznaczne określenie momentu globalnego.
  - Bezpieczne logowanie i komunikacja w systemach rozproszonych.
- **Użycie**:
  - Aplikacje międzystrefowe.
  - Przechowywanie dat w bazach danych z pełnym kontekstem.
- **Metody**:
  - `ToUniversalTime()` zwraca `DateTimeOffset` z offsetem 0.
  - `UtcDateTime` zwraca `DateTime` w UTC.

**Porównanie:**

| Właściwość        | DateTime            | DateTimeOffset        |
|-------------------|---------------------|-----------------------|
| Strefa czasowa    | Kind (niepewne)     | Offset (pewne)        |
| Porównywalność    | Potrzebna normalizacja | Bezpośrednia porówna liczby ticks z offsetem |
| Serializacja      | Zależna od Kind     | Samowystarczalna      |

---

## 3. TimeSpan

- Reprezentuje **odstęp** między dwoma punktami w czasie.
- Operacje:
  - Dodawanie/odejmowanie do `DateTime` i `DateTimeOffset`.
  - Porównywanie, formatowanie (`c`, `g`, `G`).
- Zastosowania:
  - Mierzenie czasów trwania (profilowanie, timeouty).
  - Harmonogramy i interwały w timerach.

---

## 4. Rekomendacje i best practices

1. **Przechowywanie dat** w systemach rozproszonych: **DateTimeOffset**.
2. **Operacje lokalne** i interfejsy użytkownika: **DateTime** z `Kind=Local`.
3. **Porównania dat**: zawsze w UTC lub jako `DateTimeOffset`.
4. **Duracje**: **TimeSpan**.
5. **Unikaj** `DateTime` z `Kind=Unspecified` w publicznych API.

---

## 5. Przykłady kodu

```csharp
// DateTimeOffset
var dto = new DateTimeOffset(2025,8,29,22,0,0, TimeSpan.FromHours(2));
Console.WriteLine(dto.UtcDateTime);  // 2025-08-29T20:00:00Z

// Konwersja DateTime
DateTime dtLocal = DateTime.Now;
DateTime dtUtc = dtLocal.ToUniversalTime();

// TimeSpan
timeSpan = dto - DateTimeOffset.UtcNow;
Console.WriteLine(timeSpan.TotalHours);
```

---

## Podsumowanie
Wybór między **DateTime**, **DateTimeOffset** a **TimeSpan** powinien opierać się na:
- Kontekście strefy czasowej,
- Potrzebie jednoznaczności punktu w czasie,
- Operacjach na odstępach czasowych.

Stosowanie **DateTimeOffset** w systemach rozproszonych i **TimeSpan** do duracji gwarantuje przewidywalność i czytelność kodu.