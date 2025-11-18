# Generacje Garbage Collectora w .NET

## Wprowadzenie
**Generational Garbage Collection** to model zarządzania pamięcią oparty na hipotezie, że młode obiekty umierają częściej niż stare. .NET dzieli heapem na trzy główne generacje (Gen 0, Gen 1, Gen 2), z dodatkową Large Object Heap (LOH) dla dużych alokacji. Każda generacja ma inną częstość zbierania i strategie optymalizacji.

---

## 1. Trzy generacje – hierarchia i charakterystyka

### Gen 0 – Nursery (żłobek dla noworodków)

```csharp
public class Gen0Example
{
    public void AllocateGen0()
    {
        // Wszystkie nowe alokacje zaczynają się w Gen 0
        var temp1 = new byte[100];      // Gen 0
        var temp2 = new string("hello"); // Gen 0
        var list = new List<int>();    // Gen 0
        
        // Po wyjściu ze scope – są candidates do GC
    }
}

// Charakterystyka Gen 0:
// - Wszystkie NOWE obiekty
// - Allocation pointer (bump pointer) – szybka alokacja
// - Collectione CZĘSTO (co ~ms kiedy heap zapełni się)
// - Mały rozmiar (~512 KB w Workstation, ~4 MB w Server)
// - STW (Stop-The-World) pause: ~1-5ms
```

### Gen 1 – Eden space (pośredni)

```csharp
public class Gen1Example
{
    private static List<int> _survivingObject;  // Survives collections
    
    public void SurviveGen0ToGen1()
    {
        var tempObject = new byte[1000];
        _survivingObject = new List<int> { 1, 2, 3 };
        
        // Po Gen 0 collection:
        // - tempObject: deallocated (unreachable)
        // - _survivingObject: promoted to Gen 1 (survived collection)
    }
}

// Charakterystyka Gen 1:
// - Obiekty które przetrwały ≥1 Gen 0 collection
// - Pośrednia generacja (nie Gen 0, nie Gen 2)
// - Collectione rzadziej niż Gen 0 (~5-10x)
// - STW pause: ~5-50ms
// - Rozmiar: ~1-10 MB
```

### Gen 2 – Tenured space (stare obiekty)

```csharp
public class Gen2Example
{
    private static string _longLivedString;  // Will reach Gen 2
    private static Dictionary<int, object> _cache;  // Long-lived
    
    public void AllocateLongLived()
    {
        _longLivedString = "I will live forever in the app domain";
        _cache = new Dictionary<int, object>();
        
        // Po przetrwaniu Gen 0 i Gen 1 collections
        // Te obiekty promocje do Gen 2
    }
}

// Charakterystyka Gen 2:
// - Obiekty które przetrwały ≥1 Gen 1 collection
// - Longest-lived objects (application lifetime)
// - Collectione rzadko (background, triggered by memory pressure)
// - STW pause: ~100-1000ms+ (pełne GC może być bardzo drogie)
// - Rozmiar: Może być bardzo duży (cała pozostała pamięć)
// - Background GC: .NET Core+ może zbierać concurrently
```

---

## 2. Promotion process – jak obiekty się poruszają

### Promotion path

```csharp
public class PromotionExample
{
    public void DemonstratePromotion()
    {
        // Time: T0
        var temp = new byte[100];      // Alokacja w Gen 0
        var keeper = new List<int>();  // Alokacja w Gen 0
        
        // Time: T1 – Gen 0 collection triggered
        // Mark phase: temp – unreachable, keeper – reachable
        // Promotion: keeper → Gen 1
        // Deallocation: temp deallocated
        
        // Time: T2 – Few ms later, Gen 0 collection again
        // More objects allocated in Gen 0
        
        // Time: T3 – Gen 1 collection triggered
        // Promotion: keeper → Gen 2 (przetrwało Gen 1)
        // Gen 0 collected again as part of Gen 1 collection
        
        // Time: T4 – Gen 2 collection (full GC)
        // keeper remains in Gen 2
    }
}

// Promotion flow:
// Alokacja → Gen 0 → [survives Gen 0] → Gen 1 → [survives Gen 1] → Gen 2 → stays Gen 2
```

### Promotion heuristics

```csharp
public class PromotionHeuristics
{
    public void DemonstrateSurvivalRate()
    {
        var generation = 0;
        
        // Gen 0: Typically 90% objects die, 10% survive
        for (int i = 0; i < 100; i++)
        {
            var obj = new object();
            // obj unreachable after loop – deallocated in Gen 0
        }
        
        // Gen 1: Typically 50% objects die, 50% survive → Gen 2
        var list1 = new List<int>();
        list1.Add(1);
        // If unreachable → deallocated
        // If reachable → promoted to Gen 2
        
        // Gen 2: ~1-2% objects die per collection
        // Most objects stay until application exit
    }
}

// Survival rates (empirically):
// Gen 0 → Gen 1: ~10% (90% deallocated)
// Gen 1 → Gen 2: ~50% (50% deallocated)
// Gen 2 → stays: ~95% (5% deallocated)
```

---

## 3. Collection process – co się dzieje w każdej generacji

### Gen 0 Collection

```csharp
public class Gen0CollectionFlow
{
    public void ExplainGen0Collection()
    {
        // Step 1: Allocation causes Gen 0 collection
        try
        {
            for (int i = 0; i < 100_000; i++)
            {
                var obj = new byte[1000];  // ~100 MB allocation attempts
                // Gen 0 heap fills up
            }
        }
        catch (OutOfMemoryException)
        {
            // Gen 0 collection triggered before reaching here
        }
        
        // Step 2: Stop-The-World (STW)
        // All managed threads paused
        
        // Step 3: Mark phase
        // - Scan stack roots
        // - Follow references from live objects
        // - Mark reachable objects
        
        // Step 4: Sweep phase
        // - Deallocate unmarked objects
        // - Compact Gen 0 (move live objects to beginning)
        
        // Step 5: Resume application
        // STW pause: ~1-5ms typical
    }
}

// Gen 0 collection timeline:
// Allocation → Threshold → STW → Mark → Compact → Resume
// Frequency: Every ~1-10MB allocation (varies by system)
```

### Gen 1 Collection (also triggers Gen 0)

```csharp
public class Gen1CollectionFlow
{
    public void ExplainGen1Collection()
    {
        // Gen 1 collection = Gen 0 + Gen 1 collection together
        // Gen 0 promoted objects + Gen 1 objects are collected
        
        // Step 1: Gen 1 heap pressure
        // Objects promoted from Gen 0 accumulate in Gen 1
        
        // Step 2: STW pause (longer than Gen 0)
        // Pause time: ~5-50ms
        
        // Step 3: Mark phase
        // - Scan roots (stack, statics, gc handles)
        // - Traverse object graph through Gen 0 + Gen 1
        // - Gen 2 objects marked but not traversed (unless referenced from Gen 0/1)
        
        // Step 4: Sweep + Compact
        // - Move live Gen 0 + Gen 1 objects
        // - Promote survivors to Gen 1 / Gen 2
        
        // Step 5: Gen 0 emptied
        // New allocations start in Gen 0 again
    }
}

// Gen 1 collection triggeruje Gen 0 collection jako part of process
// Ratio: Gen 1 collection ~1 per 10 Gen 0 collections (approximate)
```

### Gen 2 Collection (Full GC)

```csharp
public class Gen2CollectionFlow
{
    public void ExplainGen2Collection()
    {
        // Gen 2 collection = Full GC (Gen 0 + Gen 1 + Gen 2)
        
        // Step 1: Trigger conditions
        // a) Memory pressure (low memory notification from OS)
        // b) Gen 2 heap size threshold exceeded
        // c) Explicit GC.Collect(2) call (should avoid)
        // d) Application shutdown
        
        // Step 2: STW or Concurrent (depends on GC mode)
        // Workstation: STW, pause time: ~100-500ms
        // Server: Parallel + STW, pause time: ~500-2000ms
        // Background: Mostly concurrent, pause: ~5-50ms
        
        // Step 3: Mark phase (expensive)
        // - Entire heap traversal (Gen 0, Gen 1, Gen 2)
        // - Mark all reachable objects
        // - Most expensive part
        
        // Step 4: Sweep + Compact
        // - Deallocate all unmarked objects
        // - Compact entire heap (move objects)
        // - Update all references
        
        // Step 5: Resume
        // Longest pause time, significant application impact
    }
}

// Gen 2 collection impact:
// - Rarest but most expensive
// - Can pause application for 100ms+ in large systems
// - Full heap compaction (can reduce fragmentation)
```

---

## 4. Large Object Heap (LOH) – specjalna obsługa

### LOH characteristics

```csharp
public class LOHExample
{
    public void AllocateLOH()
    {
        // Objects >= 85 KB go to Large Object Heap (LOH)
        var largeArray = new byte[85_000];      // LOH
        var largeString = new string('x', 100_000);  // LOH
        
        // Small objects stay in regular Gen 0/1/2
        var smallArray = new byte[100];         // Gen 0
        var smallString = "hello";              // Gen 0
    }
}

// LOH rules:
// - Threshold: >= 85 KB (85000 bytes)
// - LOH is NOT compacted (fragmentation risk!)
// - LOH collected during Gen 2 collections
// - No promotion; LOH objects collected immediately if unreachable
// - Pinned objects often end up in LOH
```

### LOH performance implications

```csharp
public class LOHFragmentation
{
    private List<byte[]> _allocations = new();
    
    public void CauseFragmentation()
    {
        // Allocate many large objects
        for (int i = 0; i < 1000; i++)
        {
            _allocations.Add(new byte[100_000]);  // Each goes to LOH
        }
        
        // Remove every other allocation
        for (int i = _allocations.Count - 1; i >= 0; i -= 2)
        {
            _allocations.RemoveAt(i);
        }
        
        // LOH now fragmented: [used] [free] [used] [free] ...
        // Can't allocate contiguous large object without Gen 2 collection + compact
        // (or LOH compaction in .NET 5+)
    }
}

// LOH fragmentation problem:
// - No compaction by default
// - Gaps between live objects
// - New allocations may fail despite total free space
// - Solution: Use ArrayPool<T> to reduce LOH pressure
```

### LOH compaction (.NET 5+)

```csharp
public class LOHCompaction
{
    public void CompactLOH()
    {
        // .NET 5+: Can explicitly compact LOH
        GCCollectionMode mode = GCCollectionMode.Optimized;
        
        // Normal Gen 2 collection (may not compact LOH)
        GC.Collect(2, GCCollectionMode.Default);
        
        // Optimized: Prefers compacting LOH
        GC.Collect(2, GCCollectionMode.Optimized);
        
        // After compact: LOH defragmented, but expensive operation
    }
}
```

---

## 5. Relationship between generations

### Card marking – efficiency optimization

```csharp
public class CardMarking
{
    public class Node
    {
        public Node Next { get; set; }
    }
    
    public void DemonstrateCardMarking()
    {
        // Gen 2 object holding reference to Gen 0 object
        var gen2Object = new Node();
        
        // Gen 0 collection – do we need to scan Gen 2?
        // NO – Card marking tracks Gen2 → Gen0 references
        
        gen2Object.Next = new Node();  // Reference to Gen 0 object
        // Card marking records this reference
        // Gen 0 collection only scans marked cards, not entire Gen 2
        
        // Result: Faster Gen 0 collections despite Gen 2 existing
    }
}

// Card marking mechanism:
// - Divides Gen 2 into "cards" (128 bytes each)
// - When Gen 2 object gets reference to younger generation
// - That card is marked
// - Gen 0/1 collection only scans marked cards, not entire Gen 2
// - Vastly improves Gen 0 collection performance
```

### Ephemeral generations

```
Heap layout:
┌─────────────────────────────────────────────────────┐
│ Gen 2                                               │
│ (tenured, large, rarely moved)                      │
├─────────────────────────────────────────────────────┤
│ Gen 1 (ephemeral)                                   │
│ (intermediate, frequently promoted)                 │
├─────────────────────────────────────────────────────┤
│ Gen 0 (ephemeral)                                   │
│ (nursery, short-lived, frequently collected)        │
└─────────────────────────────────────────────────────┘

Gen 0 + Gen 1 = Ephemeral generations
- Frequently collected together
- Managed differently than Gen 2
- Gen 0 = allocation space for new objects
```

---

## 6. Monitoring generational collections

### Collection counts

```csharp
public class MonitoringGenerations
{
    public void PrintCollectionStats()
    {
        int gen0Collects = GC.CollectionCount(0);
        int gen1Collects = GC.CollectionCount(1);
        int gen2Collects = GC.CollectionCount(2);
        
        Console.WriteLine($"Gen 0 collections: {gen0Collects}");
        Console.WriteLine($"Gen 1 collections: {gen1Collects}");
        Console.WriteLine($"Gen 2 collections: {gen2Collects}");
        
        // Ratio analysis
        double ratio01 = gen0Collects > 0 ? (double)gen1Collects / gen0Collects : 0;
        double ratio12 = gen1Collects > 0 ? (double)gen2Collects / gen1Collects : 0;
        
        Console.WriteLine($"Gen 0 to Gen 1 ratio: ~1:{1/ratio01}");
        Console.WriteLine($"Gen 1 to Gen 2 ratio: ~1:{1/ratio12}");
        
        // Expected ratios:
        // Gen 0:Gen 1 ≈ 10:1 (Gen 1 collection every 10 Gen 0)
        // Gen 1:Gen 2 ≈ 5:1 (Gen 2 collection every 5 Gen 1)
    }
}
```

### Memory per generation

```csharp
public class GenerationMemory
{
    public void PrintGenerationMemory()
    {
        // .NET 5+
        GCMemoryInfo info = GC.GetGCMemoryInfo();
        
        Console.WriteLine($"Heap size: {info.HeapSizeBytes} bytes");
        Console.WriteLine($"Fragmented: {info.FragmentedBytes} bytes");
        Console.WriteLine($"Total committed: {info.TotalCommittedBytes} bytes");
        
        // Older .NET (approximate per generation)
        long totalMemory = GC.GetTotalMemory(false);
        long gen0Size ≈ totalMemory * 0.1;   // ~10% for Gen 0
        long gen1Size ≈ totalMemory * 0.1;   // ~10% for Gen 1
        long gen2Size ≈ totalMemory * 0.8;   // ~80% for Gen 2
    }
}
```

---

## 7. Best practices per generation

### Minimize Gen 0 pressure

```csharp
// ❌ High Gen 0 pressure
public void BadGen0Usage()
{
    for (int i = 0; i < 1_000_000; i++)
    {
        var list = new List<int>();
        for (int j = 0; j < 10; j++)
            list.Add(j);
    }
}

// ✅ Reduce Gen 0 allocations
public void GoodGen0Usage()
{
    var list = new List<int>(10);
    for (int i = 0; i < 1_000_000; i++)
    {
        list.Clear();
        for (int j = 0; j < 10; j++)
            list.Add(j);
    }
}
```

### Minimize Gen 2 collections

```csharp
// ❌ Causes Gen 2 collection
public void BadGen2Pressure()
{
    var cache = new Dictionary<string, byte[]>();
    
    for (int i = 0; i < 100_000; i++)
    {
        var key = $"item_{i}";
        cache[key] = new byte[10_000];  // Objects promoted to Gen 2
    }
    // Cache grows infinitely, Gen 2 bloats
}

// ✅ Manage Gen 2 pressure
public void GoodGen2Pressure()
{
    // Use object pooling
    var pool = ArrayPool<byte>.Shared;
    
    var cache = new Dictionary<string, byte[]>(10_000);  // Bounded size
    
    for (int i = 0; i < 100_000; i++)
    {
        var key = $"item_{i}";
        var bytes = pool.Rent(10_000);
        
        if (cache.Count > 10_000)
        {
            var oldestKey = cache.Keys.First();
            pool.Return(cache[oldestKey]);
            cache.Remove(oldestKey);
        }
        
        cache[key] = bytes;
    }
}
```

---

## 8. Practical scenarios and implications

### Short-lived objects (Gen 0)

```csharp
// LINQ query results, temporary calculations
public IEnumerable<int> CalculateTemporaryResults(int[] input)
{
    // LINQ allocates enumerator, buffers – all Gen 0
    return input
        .Where(x => x > 0)
        .Select(x => x * 2)
        .ToList()  // Dies after method returns, collected in Gen 0
        .Where(x => x < 100)
        .ToArray();
}

// Impact: Frequent Gen 0 collections, but low cost (~1-5ms)
```

### Medium-lived objects (Gen 1)

```csharp
// Request-scoped objects in ASP.NET
public class RequestHandler
{
    public async Task HandleRequest()
    {
        var requestData = new RequestData();  // Gen 0
        var processor = new DataProcessor(requestData);  // Gen 0
        
        var result = await processor.ProcessAsync();  // Operations in Gen 0
        
        // After request completes, if survived Gen 0 collections
        // Promoted to Gen 1, but deallocated before Gen 2
    }
}

// Impact: Moderate frequency, moderate cost (~5-50ms)
```

### Long-lived objects (Gen 2)

```csharp
// Caches, singletons, application state
public class ApplicationCache
{
    private static Dictionary<string, object> _cache = new();  // Gen 2
    
    public void CacheValue(string key, object value)
    {
        _cache[key] = value;  // Promoted to Gen 2
        // Stays in Gen 2 for application lifetime
    }
}

// Impact: Low frequency but high cost when Gen 2 collected (~100-1000ms)
// Strategy: Minimize Gen 2 pressure through cache eviction policies
```

---

## 9. Generational pressure characteristics

### Typical collection patterns

```
Application startup:
- High Gen 0 activity (initialization)
- Few Gen 1 collections
- Rare Gen 2 collections

Steady state (server app):
- Continuous Gen 0 collections
- Regular Gen 1 collections (background)
- Occasional Gen 2 collections (memory pressure)

Spikes (batch processing):
- Intense Gen 0 activity
- Possible Gen 2 collection (if memory exhausted)
- Can cause latency spike visible to users
```

### Impact on application

```csharp
public class PerformanceImpact
{
    [Benchmark]
    public void StableMemoryUsage()
    {
        // Gen 0 collections: ~100/sec
        // Gen 1 collections: ~10/sec
        // Gen 2 collections: ~0.1/sec
        // Avg latency impact: minimal (~5-10ms every second)
        
        for (int i = 0; i < 1_000; i++)
        {
            var obj = new object();
        }
    }
    
    [Benchmark]
    public void UnstableMemoryGrowth()
    {
        // Gen 0 collections: ~50/sec
        // Gen 1 collections: ~5/sec
        // Gen 2 collections: ~1/sec (significant!)
        // Avg latency impact: high (~100-500ms every second)
        
        var list = new List<object>();
        for (int i = 0; i < 1_000_000; i++)
        {
            list.Add(new object());  // Unbounded growth
        }
    }
}
```

---

## Podsumowanie

**Generacje Garbage Collectora:**

| Generacja | Rozmiar | Frequency | Pause Time | Purpose |
|-----------|---------|-----------|-----------|---------|
| **Gen 0** | ~500KB-4MB | Frequent (~ms) | 1-5ms | Nursery dla nowych obiektów |
| **Gen 1** | ~1-10MB | Moderate (~sec) | 5-50ms | Pośredni stage dla survivals |
| **Gen 2** | ~rest of heap | Rare | 100-1000ms+ | Long-lived objects |
| **LOH** | Large objects | With Gen 2 | Significant | >=85KB objects |

**Kluczowe mechanizmy:**
- **Promotion**: Gen 0 → Gen 1 → Gen 2 dla survivorów
- **Card marking**: Optimization dla inter-generational references
- **Ephemeral generations**: Gen 0 + Gen 1 zbierane razem
- **LOH fragmentation**: Risk bez compaction

**Developer powinien:**
- Minimalizować Gen 0 pressure (pooling, Span<T>, stackalloc)
- Razem kontrolować Gen 2 pressure (cache policies, weak references)
- Monitorować collection ratios i pause times
- Rozumieć impact na latency w produkcji
- Profilować high-allocation scenarios

Generacyjny model GC w .NET to sophisticated optimization – property understanding pozwala na effective tuning aplikacji dla performance i latency.