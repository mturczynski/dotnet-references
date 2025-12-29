# Large Object Heap (LOH) – Comprehensive Guide

## Co to jest LOH?

**Large Object Heap (LOH)** to oddzielny region pamięci zarządzanej, dedykowany do przechowywania obiektów o rozmiarze **≥ 85 KB**. Separacja ta została wprowadzona w celu optymalizacji wydajności garbage collectora i redukcji fragmentacji dla małych obiektów.

```csharp
// Allocacja na LOH (≥ 85 KB)
byte[] largeBuffer = new byte[100_000];        // ~100 KB → LOH
string largeString = new string('x', 100_000); // ~200 KB → LOH

// Allocacja na Gen 0 (< 85 KB)
byte[] smallBuffer = new byte[1000];           // 1 KB → Gen 0
string smallString = "hello";                  // ~40 bytes → Gen 0
```

---

## Architektura i Mechanika LOH

### Dlaczego oddzielny heap?

1. **Generacyjne kolekcjonowanie nieefektywne dla dużych obiektów**
   - Duże obiekty żyją długo (tenured objects)
   - Markowanie ich za każdą Gen 0 kolekcję = marnowana praca
   - Kompaction dużych bloków = wysokie koszty CPU i I/O

2. **Fragmentation problemy**
   - Pinning dużych obiektów blokuje kompaction
   - Interleaving Gen 0/Gen 1 alokacji fragmentuje Gen 2
   - LOH osobno = czystsze zarządzanie blokami

3. **Performance charakterystyka**
   - Brak generacyjnego kolekcjonowania = mniej pauz w typowych scenariuszach
   - Fullkolekcje LOH tylko gdy LOH pressure przekroczy threshold

### Threshold 85 KB – dlaczego?

```csharp
// .NET Runtime ustala LOH threshold na 85 KB
// (internal define: MIN_OBJECT_SIZE dla LOH)

// Empiryczne obserwacje:
// - Poniżej 85 KB: Gen 0/Gen 2 compaction jest efektywna
// - Powyżej 85 KB: overhead kompaction > benefit generacyjnego kolekcjonowania
// - Próg zoptymalizowany dla typowych pattern'ów alokacji
```

---

## Brak Generacyjnego Kolekcjonowania LOH

### Konsekwencja: Full GC Trigger

```csharp
public class LOHAllocationDemo
{
    public static void Main()
    {
        // Scenario: LOH pod presją
        var lohObjects = new List<byte[]>();
        
        for (int i = 0; i < 100; i++)
        {
            // Każda alokacja trafia na LOH (≥85 KB)
            lohObjects.Add(new byte[100_000]);
            
            // Przy pewnym threshold → Full GC2 collection
            // (nie Gen 0, nie Gen 1 – pełna kolekcja!)
            
            if (i % 20 == 0)
                Console.WriteLine($"LOH pressure: {GC.GetTotalMemory(false) / (1024 * 1024)} MB");
        }
    }
}

// Output: Full GC2 pause observable
```

### Monitoring – kiedy LOH collection się triggerea:

```csharp
// Liczba Gen 2 collections = proxy dla LOH pressure
long gen2Before = GC.CollectionCount(2);

var loBuffer = new byte[1_000_000];  // 1 MB allocation
// ... pressure builds ...

long gen2After = GC.CollectionCount(2);
if (gen2After > gen2Before)
    Console.WriteLine("Full GC triggered due to LOH pressure!");
```

---

## Fragmentacja – Główny Problem LOH

### Dlaczego fragmentacja na LOH jest poważna?

```
LOH Memory Layout PRZED fragmentacją:
┌─────────────────────────┬─────────────────────────┬─────────────────┐
│ Object 1 (100 KB)       │ Object 2 (100 KB)       │ Object 3 (50 KB)│
└─────────────────────────┴─────────────────────────┴─────────────────┘
                          ↑
                   Kontinualna wolna przestrzeń


LOH Memory Layout PO fragmentacji (z pinningiem/pruningiem):
┌─────────────────────────┬──────────┬─────────────────────────┬────┬────┬────┐
│ Object 1 (100 KB)       │ FREE 10KB│ Object 2 (100 KB)       │FREE│OBJ │FREE│
└─────────────────────────┴──────────┴─────────────────────────┴────┴────┴────┘
         ↓
Problem: Brak ciągłego bloku 100 KB dla nowej alokacji!
Mimo że suma FREE > 100 KB, alokacja FAIL – Full GC/Compaction trigger
```

### Fragmentacja – przyczyny:

```csharp
// 1. Pinning dużych obiektów
GCHandle pinnedHandle = GCHandle.Alloc(new byte[200_000], GCHandleType.Pinned);
// LOH nie może skompaktować (pinned object blocking)
pinnedHandle.Free();

// 2. Variable lifetime obiektów LOH
var obj1 = new byte[100_000];  // Żyje: 1s
var obj2 = new byte[100_000];  // Żyje: 10s
var obj3 = new byte[100_000];  // Żyje: 5s
// → obj1 umiera, obj2/obj3 żyją → fragmentacja!

// 3. Intermixed alokacje/dealokacje
for (int i = 0; i < 10; i++)
{
    var temp = new byte[100_000];
    // ... używaj ...
    // → po dealokacji: LOH ma dziury
}
```

---

## LOH Compaction (.NET 5+)

### Nowy feature: GCLoHCompactionMode

```csharp
// .NET 5.0+ – można kontrolować kompaction LOH
// Konfiguracja w runtime:

// 1. Default (ImmediateCompaction)
// Kompactuje LOH zawsze podczas Gen 2 collection
GCCollectionMode.Default;

// 2. ExplicitCompaction
// Manualna kompaction via GC.Collect() z flaga
GC.Collect(2, GCCollectionMode.Optimized, false, true); // true = compact LOH

// 3. Aggressive
GC.LowLatency();  // Zmienia profile – mniej kompaction
```

### Konfiguracja via environment/config:

```xml
<!-- runtimeconfig.json lub app.config -->
<configuration>
    <configPaths>
        <add key="System.GC.LOHCompactionMode" value="2" />
        <!-- 0 = Default (compact)
             1 = ImmediateCompaction (every Gen 2)
             2 = ExplicitCompaction (on demand)
             3 = OnlyOnDemand (no auto) -->
    </configPaths>
</configuration>
```

```csharp
// Benchmark: Compaction impact na fragmentation
public class LOHCompactionBenchmark
{
    [Benchmark]
    public void LOHWithDefaultCompaction()
    {
        for (int i = 0; i < 100; i++)
        {
            var buf = new byte[500_000];
            // ... use ...
        }
        GC.Collect(2);  // Trigger Gen 2 + LOH compaction
    }
    
    [Benchmark]
    public void LOHNoCompaction()
    {
        // Same allocation pattern
        // But no compaction → fragmentation accumulates
    }
}

// Result:
// LOHWithDefaultCompaction:   ~150ms pause
// LOHNoCompaction:             ~80ms pause (but fragmented)
```

---

## Performance Impact – Kiedy LOH staje się problemem

### GC Pause Time – LOH Pressure

```csharp
// Measurement: LOH pressure vs pause time
public class LOHPauseMeasurement
{
    public static void Main()
    {
        var sw = Stopwatch.StartNew();
        long collections = 0;
        
        for (int i = 0; i < 10_000; i++)
        {
            var gen2Before = GC.CollectionCount(2);
            
            var largeBuffer = new byte[200_000];  // LOH allocation
            
            var gen2After = GC.CollectionCount(2);
            if (gen2After > gen2Before)
            {
                sw.Stop();
                collections++;
                Console.WriteLine($"Full GC #{collections} at {sw.ElapsedMilliseconds}ms");
                sw.Start();
            }
        }
    }
}

// Expected output:
// Full GC #1 at ~150ms
// Full GC #2 at ~200ms (fragmentation worsens)
// ...
```

### Threshold Behavior

```csharp
// LOH collection trigger (empirical)
const long LOH_THRESHOLD = 100 * 1024 * 1024;  // ~100 MB
// When LOH heap size exceeds threshold → Full GC2

// Scenario: Memory pressure
long allocatedLOH = 0;

for (int i = 0; i < 10; i++)
{
    var obj = new byte[20_000_000];  // 20 MB each
    allocatedLOH += 20_000_000;
    
    Console.WriteLine($"LOH Allocated: {allocatedLOH / (1024 * 1024)} MB");
    
    if (allocatedLOH > LOH_THRESHOLD)
        Console.WriteLine("⚠️ Full GC2 likely triggered!");
}
```

---

## Monitoring LOH – Praktyczne Narzędzia

### GC Statistics

```csharp
public class LOHMonitoring
{
    public static void PrintLOHStats()
    {
        var info = GC.GetGCMemoryInfo();
        
        Console.WriteLine($"Total Committed: {info.TotalCommittedBytes / (1024 * 1024)} MB");
        Console.WriteLine($"Heap Size: {info.HeapSizeBytes / (1024 * 1024)} MB");
        
        // Collection counts by generation
        for (int gen = 0; gen <= 2; gen++)
            Console.WriteLine($"Gen {gen} Collections: {GC.CollectionCount(gen)}");
        
        // LOH statistics (requires .NET 5+)
        var heapInfo = GC.GetTotalMemory(false);
        Console.WriteLine($"Total Memory: {heapInfo / (1024 * 1024)} MB");
    }
}

// Output:
// Total Committed: 512 MB
// Heap Size: 256 MB
// Gen 0 Collections: 150
// Gen 1 Collections: 30
// Gen 2 Collections: 5  ← LOH collections manifest here
// Total Memory: 200 MB
```

### ETW Tracing (Windows)

```powershell
# Capture GC events including LOH
logman start gc_trace -p "Microsoft-Windows-DotNETRuntime" -o gc_trace.etl -ets
dotnet myapp.dll
logman stop gc_trace -ets

# Analyze with perfview
perfview /GCCollectOnly /MaxCollectSecs=30 myapp.exe
```

### PerfView – Visual Analysis

```
PerfView → GC Events → 
- Gen 2 collection pause times
- LOH fragmentation
- Compaction frequency
- Object survivorship patterns
```

---

## Best Practices – Minimalizacja LOH Pressure

### 1. Object Pooling dla LOH Alokacji

```csharp
// PROBLEM: Frequent LOH allocation
var buffer1 = new byte[200_000];
// use...
var buffer2 = new byte[200_000];
// use...
// → Repeated LOH collections

// SOLUTION: Object pooling
public class LOHBufferPool
{
    private static readonly ArrayPool<byte> _pool = ArrayPool<byte>.Shared;
    
    public void ProcessData()
    {
        byte[] buffer = _pool.Rent(200_000);
        try
        {
            // Use buffer
        }
        finally
        {
            _pool.Return(buffer);
        }
    }
}
```

### 2. Chunking – Break LOH into Gen 2 Objects

```csharp
// PROBLEM: Single 1 MB LOH allocation
byte[] gigantic = new byte[1_000_000];

// SOLUTION: Chunked Gen 2 allocation
public class ChunkedBuffer
{
    private const int CHUNK_SIZE = 80_000;  // < 85 KB threshold
    private readonly List<byte[]> _chunks;
    
    public ChunkedBuffer(int totalSize)
    {
        _chunks = new List<byte[]>();
        int remaining = totalSize;
        
        while (remaining > 0)
        {
            int size = Math.Min(CHUNK_SIZE, remaining);
            _chunks.Add(new byte[size]);
            remaining -= size;
        }
    }
}
```

### 3. Reuse & Pinning Awareness

```csharp
// ANTI-PATTERN: Pinning LOH objects
GCHandle pinnedLOH = GCHandle.Alloc(new byte[500_000], GCHandleType.Pinned);
// ... blocks compaction ...
// Later...
pinnedLOH.Free();

// BETTER: Pin before LOH allocation (if necessary)
public class PinningStrategy
{
    // Pre-allocate and pin Gen 2 object
    private static readonly GCHandle _pinned;
    
    static PinningStrategy()
    {
        var buffer = new byte[100_000];  // Force Gen 2 residency first
        GC.Collect(2);
        _pinned = GCHandle.Alloc(buffer, GCHandleType.Pinned);
    }
}
```

### 4. Explicit Compaction (если необходимо)

```csharp
public class ExplicitLOHCompaction
{
    public static void CompactLOH()
    {
        // Only on .NET 5+
        #if NET5_0_OR_GREATER
        GC.Collect(2, GCCollectionMode.Optimized, false, true); // true = compact
        #else
        GC.Collect(2);  // Best effort
        #endif
    }
    
    // Use sparingly – only when:
    // 1. Known LOH fragmentation spike
    // 2. App entering lower-traffic period
    // 3. Measured pause acceptable
}
```

---

## Anti-Patterns – Co Unikać

### ❌ Frequent GC.Collect()

```csharp
// BAD: Forces unnecessary LOH compaction
for (int i = 0; i < 1000; i++)
{
    var obj = new byte[200_000];
    // use...
    GC.Collect();  // ← Triggers full GC (including LOH)
}

// Consequence: Constant pause spikes, thrashing cache
```

### ❌ Pinning bez Potrzeby

```csharp
// BAD: Prevents LOH compaction indefinitely
GCHandle handle = GCHandle.Alloc(new byte[300_000], GCHandleType.Pinned);
// Object cannot move → blocks compaction for entire LOH segment
```

### ❌ Mixed LOH Lifetime

```csharp
// BAD: Objects with different lifetimes
var permanent = new byte[200_000];  // Lives entire app
var temporary = new byte[200_000];  // Lives 10ms
var cache = new byte[200_000];      // Lives 1min
// → Interleaved allocation/deallocation → fragmentation

// BETTER: Segregate
static class LOHSegmentation
{
    public static class Permanent
    {
        public static byte[] LongLivedBuffer = new byte[200_000];
    }
    
    public class Temporary : IDisposable
    {
        private byte[] _buffer = new byte[200_000];
        // Controlled lifecycle
    }
}
```

---

## Real-World Scenarios

### Scenario 1: Image Processing Pipeline

```csharp
public class ImageProcessor
{
    // PROBLEM: Each frame = LOH allocation
    public void ProcessVideoFrame(byte[] frameData)
    {
        byte[] decompressed = Decompress(frameData);  // ≥ 85 KB → LOH
        byte[] filtered = ApplyFilter(decompressed);   // ≥ 85 KB → LOH
        byte[] resized = Resize(filtered);             // ≥ 85 KB → LOH
        SaveFrame(resized);
    }
    
    // SOLUTION: Object pooling
    private readonly ArrayPool<byte> _pool = ArrayPool<byte>.Shared;
    
    public void ProcessVideoFrameOptimized(byte[] frameData)
    {
        byte[] decompressed = _pool.Rent(frameData.Length * 3);
        byte[] filtered = _pool.Rent(decompressed.Length);
        byte[] resized = _pool.Rent(frameData.Length);
        
        try
        {
            Decompress(frameData, decompressed);
            ApplyFilter(decompressed, filtered);
            Resize(filtered, resized);
            SaveFrame(resized);
        }
        finally
        {
            _pool.Return(decompressed);
            _pool.Return(filtered);
            _pool.Return(resized);
        }
    }
}
```

### Scenario 2: Network Buffer Management

```csharp
public class TcpBufferManager
{
    // PROBLEM: Large receive buffers → LOH
    public async Task HandleConnectionAsync(TcpClient client)
    {
        byte[] receiveBuffer = new byte[1_000_000];  // 1 MB → LOH
        // ...
        // Connection ends, buffer deallocates
        // LOH fragmentation increases
    }
    
    // SOLUTION: Reuse & pool
    private static readonly MemoryPool<byte> _pool = MemoryPool<byte>.Shared;
    
    public async Task HandleConnectionOptimizedAsync(TcpClient client)
    {
        using (var memory = _pool.Rent(1_000_000))
        {
            // Use memory
            // Automatically returned on dispose
        }
    }
}
```

### Scenario 3: Serialization (JSON/Protobuf)

```csharp
public class SerializationBuffering
{
    // PROBLEM: Large JSON buffer
    public string SerializeLargeObject(object obj)
    {
        var buffer = new byte[5_000_000];  // 5 MB → LOH
        // Serialize...
        return Encoding.UTF8.GetString(buffer);
    }
    
    // SOLUTION: Use MemoryStream (heap-friendly)
    public string SerializeLargeObjectOptimized(object obj)
    {
        using var stream = new MemoryStream();
        using var writer = new Utf8JsonWriter(stream);
        // Serialize...
        return Encoding.UTF8.GetString(stream.ToArray());
    }
}
```

---

## Integration z GC Modes

### Server GC + LOH

```
Server GC (multi-threaded):
- Każdy thread ma własny Gen 0/Gen 1
- Gen 2 i LOH współdzielone
- LOH fragmentacja bardziej widoczna (contention)

Best practice: Monitor LOH stats per HeapIndex
```

### Background GC + LOH

```csharp
// Background GC zmniejsza pause time
// Ale zwiększa LOH pressure (background thread nie może kompaktować)

var info = GC.GetGCMemoryInfo();
foreach (var heap in info.HeapInfo)
{
    Console.WriteLine($"Heap {heap.Index}: {heap.TotalCommittedBytes / (1024 * 1024)} MB");
    // Monitor LOH per heap
}
```

---

## Podsumowanie – Kluczowe Punkty

| Aspekt | Opis |
|--------|------|
| **Threshold** | ≥ 85 KB → LOH allocation |
| **Generacje** | Brak generacyjnego GC dla LOH (full collection) |
| **Fragmentation** | Główny problem – zmniejsza throughput, zwiększa pause time |
| **Compaction** | .NET 5+: można kontrolować (GCLoHCompactionMode) |
| **Monitoring** | GC.CollectionCount(2), GC.GetGCMemoryInfo(), ETW tracing |
| **Best Practice** | ArrayPool<T>, chunking, reuse, avoid pinning |
| **Anti-Pattern** | Frequent GC.Collect(), unnecessary pinning, mixed lifetime |
| **Tuning** | GCLoHCompactionMode=ExplicitCompaction dla low-latency apps |

---

## Praktyczne Porady

1. **Profil przed optymalizacją**
   - Użyj PerfView, ETW do zmierzenia LOH pressure
   - Nie gaś się na spekulacjach

2. **ArrayPool<T> jako default**
   - Prawie zawsze lepsza od manual pooling
   - Thread-safe, optimized

3. **Unikaj "premature GC.Collect()"**
   - Rzadko pomaga, zwykle pogarsza
   - Trust GC heuristics

4. **Monitoruj pauz time w produkcji**
   - Gen 2 pause = proxy dla LOH issues
   - Ustaw alerty na Gen 2 > 100ms

5. **Chunk jeśli możesz**
   - 85 KB < data < 200 KB? Perturbuj alokacje
   - Zmniejsza LOH pressure

6. **Pinning to nuclear option**
   - Ostateczność dla P/Invoke
   - Zawsze rozważ `fixed` statement zamiast GCHandle

---

## Referencje

- [Microsoft Docs: Large Object Heap](https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/large-object-heap)
- [.NET 5 LOH Compaction](https://github.com/dotnet/runtime/issues/35547)
- [Maoni Stephens – GC Performance Blog](https://maoni0.wordpress.com/)
- [Stephen Toub – Memory<T> Series](https://github.com/dotnet/runtime/issues/new)