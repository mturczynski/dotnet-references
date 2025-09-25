# Pytania rekrutacyjne dla .NET Senior Developera

## 1. Pytania ogólne i o doświadczenie
- Jakie masz hobby?
- Opisz swoją dotychczasową karierę.
- Ostatnie projekty, w których pracowałeś:
  - Co to było?
  - Jakie technologie?
  - Kto ustalał architekturę?
  - Jak wyglądał zespół?
  - Kto definiował wymagania do zadań i jak detalicznie były opisywane?
  - Jaka była Twoja rola?
- W jaki sposób wspieraliście produkcję (on-call)?
- Jaka jest Twoja największa porażka w karierze?
- Rozwiązanie, z którego jesteś dumny?
- Co najbardziej Cię irytuje w projektach/firmach?
- Konflikt personalny – opis i rozwiązanie.
- Dlaczego chcesz zmienić pracę?

## 2. Programowanie .NET i C#
### Typy i struktury

- [Jaka jest najnowsza wersja .NET? Jakie nowe featury weszły w tej i ostatnich wersjach?](dotnet-latest-summary.md)
- [Czym się różni struktura od klasy?](struct-vs-class.md)
    - [Value vs reference type](value-vs-reference-types.md)
- [Czym jest mechanizm closure w C#?](closure-mechanism.md)
- [Czym różnią się DateTime, DateTimeOffset i TimeSpan?](datetime-vs-offset-timespan.md)
- [Czym się różni const od readonly fielda?](const-vs-readonly.md)
- [Czym się różni klasa abstrakcyjna od interfejsu?](abstract-vs-interface.md)
- [Czym są typy generyczne? Constraints, kowariancja vs kontrawariancja](generics-constraints-variance.md)
- [Czym są atrybuty w C#?](attributes-csharp-guide.md)
- [Co robią słowa kluczowe out, ref, in?](ref-out-in-keywords.md)

## 3. Wielowątkowość i async/await
- Po co async/await i Task?
- I/O-bound vs CPU-bound.
- async/await i wątki.
- Concurrency vs parallelism.
- SynchronizationContext.
- Kanały (Channels).
- Kolekcje thread-safe, pułapki ConcurrentDictionary.
- Semaphore, mutex, sekcja krytyczna.
- Obiekty niemutowalne (immutable).
- Thread.Sleep vs Task.Delay; Task.WaitAll vs Task.WhenAll.
- await Task vs Task.ContinueWith().
- volatile.
- ValueTask – zalety.

## 4. ASP.NET Core
- Middleware i request pipeline.
- Minimal API.
- Cykl życia serwisów IoC – Transient, Scoped, Singleton.
- Options pattern: IOptions<T>, IOptionsSnapshot<T>, IOptionsMonitor<T>.

## 5. Entity Framework
- Wielowątkowość.
- Typy dziedziczenia (TPH, TPT, TPC).
- Optymalizacja zapytań – projekcje, AsNoTracking, single/split query.
- DbSet<T> vs Set<T>.
- Query Filters, ComplexProperty vs OwnProperty.
- First() vs Single() – różnice.
- Tworzenie DbContext bez DbSetów.
- Zabezpieczenie bazy, pula połączeń.

## 6. Architektura i wzorce
- OOP: abstrakcja, hermetyzacja, polimorfizm, dziedziczenie.
- Dziedziczenie vs kompozycja.
- CQRS, outbox/inbox pattern.
- Thundering Herd problem.
- Rate limiting: fixed, sliding window, leaky bucket.
- Idempotency, circuit breaker, Bloom filter.
- Sagi (Choreography vs Orchestration).
- Event Sourcing.

## 7. Messaging i mikroserwisy
- Kolejki vs strumienie (RabbitMQ vs Kafka).
- Exchange vs queue, typy exchange (direct, topic, fanout, headers).
- Obsługa odrzuceń i dead-letter w RabbitMQ.
- MassTransit.
- Mikroserwisy – zalety, wady, komunikacja, caching.

## 8. Bazy relacyjne i transakcje
- JOIN (INNER, LEFT), WHERE vs HAVING.
- Spójność danych, przykłady niespójności.
- ACID i transakcje.
- Poziomy izolacji transakcji i problemy.
- Lost updates – operacje atomowe, SELECT FOR UPDATE, compare-and-set.
- Serializability, locki vs SSI.
- Ochrona przed awarią: WAL, backupy, replikacja.
- Point-in-time recovery.

## 9. Git i zarządzanie kodem
- Branchowanie: trunk-based, git-flow.
- Pull-rebase vs pull-merge.
- Squash.
- Git LFS – duże pliki binarne.
- Revert, cherry-pick, patch.

## 10. REST API i wzorce integracyjne
- HTTP verbs, HATEOAS.
- RPC (gRPC) vs message passing.
- OLTP vs OLAP.
- Eventual consistency, CAP Theorem, split-brain.
- Actor Model.

## 11. Bezpieczeństwo
- Token sesji vs JWT – zalety, wady, bezpieczeństwo JWT.
- Przechowywanie tokenów w cookies.
- Funkcje hashujące: MD5, SHA-1, SHA-2, salt+pepper, HMAC.
- Poufność/integralność przy transmisji – szyfry.
- Generatory liczb losowych: Random, Random.Shared, RandomNumberGenerator.
- CSPRNG.
- OWASP Top 10: Injection, Broken Auth, XSS, XXE, CSRF, click-jacking itd.
- Przechowywanie kluczy API.

## 12. Praca zespołowa, leadership i organizacja
- Waterfall vs Agile – wady/zalety.
- Estymacja zadań.
- Zarządzanie długiem technologicznym.
- Konflikty w zespole, rozwiązywanie.
- Delegowanie, balans hands-on i liderowania.
- Współpraca z innymi ekspertami.

## 13. Prawdopodobieństwo
- Prawdopodobieństwo orła po serii wyników.
- Mnożnik w kasynie (&2% marża).
- Dystrybuanta – wyjaśnienie.

## 14. Zadania i snippety
- Przykłady snippetów C#, SQL.
- Analiza kodu (closures, async/await, virtual/override, itd.).

---
