# Garbage Collector w .NET

## Wprowadzenie
**Garbage Collector (GC)** to automatyczny system zarządzania pamięcią w .NET, odpowiedzialny za alokację obiektów na stercie (heap), śledzenie żywotności i dealokację pamięci, gdy obiekty są już nieosiągalne. Jest kluczowym elementem runtime'u, mającym wpływ na wydajność, latency i throughput aplikacji.

---

## 1. Fundamenty – Mark & Sweep algorytm

### Generational GC model

```csharp
// Pamięć zarządzana podzielona na generacje:
// Gen 0: nowe alokacje, szybkie, częste collections
// Gen 1: obiekty przetrwały Gen 0 collection
// Gen 2: obiekty przetrwały Gen 1, rzadko zbierane
```

### Proces collection

```
1. MARK PHASE (Concurrent/STW)
   - Root scan (stack, static fields, GC handles)
   - Object graph traversal
   - Mark reachable objects

2. SWEEP/COMPACT PHASE
   - Move live objects (heap compaction)
   - Free unreachable memory
   - Update references

3. SWEEP phase
   - Actually deallocate marked memory
   - Update allocation pointer
```

---

## 2. Generational hypothesis

### Koncepcja generacji

```csharp
public class Example
{
    public void AllocateMany()
    {
        for (int i = 0; i < 1000; i++)
        {
            var obj = new object();  // Alokacja w Gen 0
            // obj niedostępny po pętli - szybko do usunięcia
        }
        
        var longLived = new Dictionary<int, string>();  // Gen 0 initially
        // Przetrwa Gen 0 → Gen 1
        // Przetrwa Gen 1 → Gen 2
        // Będzie zbierany rzadko
    }
}

// Generational GC założenie:
// "Młode obiekty umierają częściej niż stare"
// Gen 0 collection: ~ms
// Gen 1 collection: ~10-100ms
// Gen 2 collection: ~100-1000ms+
```

---

## 3. Typy GC w .NET

### Workstation GC (domyślny dla client apps)

```csharp
// Pojedynczy thread zbierający
// Optymalizowany dla responsiveness

// Config:
// <RuntimeHostConfigurationOption>
//   <name>System.GC.IsServerGC</name>
//   <value>false</value>
// </RuntimeHostConfigurationOption>

// Charakterystyka:
// - STW (Stop-The-World) pause
// - Low memory overhead
// - Single-threaded collector
```

### Server GC (dla server apps, ASP.NET)

```csharp
// Parallel collector - thread per CPU core
// Optymalizowany dla throughput

// Config:
// <name>System.GC.IsServerGC</name>
// <value>true</value>

// Charakterystyka:
// - Parallel collection threads
// - Higher memory usage
// - Better throughput
// - Longer pause times
```

### Background GC (.NET Core+)

```csharp
// Concurrent collector - collection alongside application
// Drastycznie zmniejsza pause times

// Config:
// <name>System.GC.Concurrent</name>
// <value>true</value>

// Proces:
// - Concurrent mark phase (aplikacja działa)
// - Brief STW sweep
// - Concurrent compact (opcjonalnie)
```

---

## 4. Pressure i triggering

### GC Pressure sources

```csharp
// High allocation rate
public void HighAllocationPressure()
{
    for (int i = 0; i < 1_000_000; i++)
    {
        var list = new List<int>();  // Gen 0 pressure
        for (int j = 0; j < 100; j++)
            list.Add(j);
    }
    // Miliony alokacji w Gen 0 → frequent collections
}

// Large object allocations
public void LargeObjectHeapPressure()
{
    var largeArray = new byte[85000];  // > 85KB → LOH (Large Object Heap)
    // LOH collections raczej nie są compacted
    // Fragmentacja heap'u
}

// Pinned objects
public void PinnedObjectPressure()
{
    var obj = new object();
    var handle = GCHandle.Alloc(obj, GCHandleType.Pinned);
    // Pinned objects blokują compaction
    // Podem fragmentacja
    // handle.Free();  // Must free!
}
```

### Collection triggers

```csharp
public class GCTriggers
{
    // 1. Allocation threshold exceeded
    var obj = new object();  // Gen 0 threshold → Gen 0 collection
    
    // 2. Explicit GC.Collect()
    GC.Collect();  // UNIKAJ - zniszczy performance!
    
    // 3. Low memory pressure (OS signal)
    // GC automatycznie zbiera Gen 2
    
    // 4. Finalizer queue pressure
    var withFinalizer = new ObjectWithFinalizer();  // Finalizer pressure
}

public class ObjectWithFinalizer
{
    ~ObjectWithFinalizer()  // Finalizer = pressure!
    {
        Console.WriteLine("Finalizing");
    }
}
```

---

## 5. Roots i reachability

### Root objects

```csharp
public class RootExample
{
    // Stack
    public void StackRoots()
    {
        var local = new object();  // root: local variable
        var obj = new Dictionary<string, object>();  // root
    }  // local, obj out of scope → no longer roots
    
    // Static fields
    public static object StaticRoot = new object();  // Persistent root
    
    // GC Handles
    public void HandleRoot()
    {
        var obj = new object();
        var handle = GCHandle.Alloc(obj);
        // obj teraz root'em dopóki handle nie będzie freed
        handle.Free();
    }
}

// Mark phase traversal:
// Roots → follow references → mark reachable
// Unmarked = garbage
```

### Reachability example

```csharp
public class ReachabilityExample
{
    public class Node
    {
        public Node Next { get; set; }
    }
    
    public void Example()
    {
        var node1 = new Node();
        var node2 = new Node();
        var node3 = new Node();
        
        node1.Next = node2;
        node2.Next = node3;
        
        node1 = null;  // Still reachable: node1 -> node2 -> node3
                       // node2 is referenced, so all reachable
        
        node2 = null;  // Still reachable: node2 via node1 (but node1 is null)
        
        node3 = null;  // If no one references node1/node2: all garbage
        
        // After GC: all collected if truly unreachable
    }
}
```

---

## 6. Finalizers i WeakReference

### Problemy z finalizatorami

```csharp
public class ProblematicFinalizer
{
    private byte[] _buffer = new byte[10_000];
    
    ~ProblematicFinalizer()  // UNIKAJ jeśli możliwe!
    {
        Console.WriteLine("Finalizing");
        // Finalizer creates pressure and delays collection
    }
}

// GC handling finalizers:
// 1. Object with finalizer → Gen 0 collection
// 2. If unreachable, moved to finalizer queue
// 3. Finalizer thread processes queue
// 4. Only then deallocated in Gen 2 collection
// Result: Object lives longer, creates Gen 2 pressure
```

### Proper cleanup pattern

```csharp
// Dispose pattern (nie finalizer!)
public class ProperCleanup : IDisposable
{
    private byte[] _buffer = new byte[10_000];
    private bool _disposed = false;
    
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);  // Tell GC nie uruchamiaj finalizer
    }
    
    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;
        
        if (disposing)
        {
            // Managed resources
            _buffer = null;
        }
        
        _disposed = true;
    }
    
    ~ProperCleanup()
    {
        Dispose(false);  // Finalizer as safety net only
    }
}

// Użycie
using (var obj = new ProperCleanup())
{
    // Use object
}  // Dispose() called immediately
```

### WeakReference

```csharp
public class WeakReferenceExample
{
    public void Example()
    {
        var obj = new byte[1_000_000];  // Strong reference
        var weak = new WeakReference<byte[]>(obj);  // Weak reference
        
        // obj is still strongly referenced
        byte[] retrieved;
        if (weak.TryGetTarget(out retrieved))
        {
            Console.WriteLine("Object still alive");
        }
        
        obj = null;  // No strong references
        
        // Next GC collection
        if (weak.TryGetTarget(out retrieved))
        {
            Console.WriteLine("Object collected");  // This branch
        }
    }
}

// Use case: caches where objects can be reclaimed
```

---

## 7. GC modes i configuration

### GC modes

```csharp
// 1. Throughput GC (default Server)
// - Maximizes app throughput
// - Longer pause times acceptable
// Config: <name>System.GC.Throughput</name>
// Result: ~500ms+ pauses for large heaps

// 2. Latency GC (default Workstation)
// - Minimizes pause time
// - Some throughput sacrifice
// Result: ~50-200ms pauses

// 3. Low latency GC (.NET 6+)
// - Concurrent Mark-Sweep
// - Sub-ms pauses
// Config: <name>System.GC.HeapCount</name>
// <value>4</value> (or 0 for default)
```

### Dynamic adaptation (GC in .NET 9)

```csharp
// .NET 9 feature: GC adapts to runtime behavior
// - Measures pause times
// - Adjusts gen thresholds dynamically
// - Balances throughput vs latency
// Result: Automatic optimization without config

// Example:
// If pauses exceed threshold → reduce Gen 2 collection frequency
// If memory usage high → increase Gen 0 threshold
```

---

## 8. Memory leaks w GC'd systemach

### Logical memory leak

```csharp
public class LogicalMemoryLeak
{
    private static List<byte[]> _cache = new();
    
    public void AddToCache(byte[] data)
    {
        _cache.Add(data);  // Strong reference retained forever
    }
}

// Problem:
// - Data never garbage collected
// - But GC thinks it's reachable (it is!)
// - Leak but not from GC perspective
// Solution: Implement proper cache eviction/cleanup
```

### Event handler leak

```csharp
public class EventHandlerLeak
{
    public event EventHandler SomethingHappened;
    
    public void Subscribe(Action handler)
    {
        SomethingHappened += (s, e) => handler();
        // If handler holds reference to 'this'
        // and 'this' is long-lived → leak
    }
}

// Solution: Proper unsubscribe
public void Unsubscribe(Action handler)
{
    SomethingHappened -= (s, e) => handler();  // Can't unsubscribe anonymous!
    // Use named method instead
}
```

---

## 9. Performance tuning

### GC configuration (.NET Core+)

```xml
<!-- .csproj or runtimeconfig.json -->
<runtimeOptions>
  <configProperties>
    <!-- Server GC for throughput -->
    <property name="System.GC.IsServerGC" value="true" />
    
    <!-- Reduce Gen 2 collection frequency -->
    <property name="System.GC.HeapCount" value="8" />
    
    <!-- Increase LOH threshold -->
    <property name="System.GC.LOHThreshold" value="131072" />
    
    <!-- Concurrent GC for low latency -->
    <property name="System.GC.Concurrent" value="true" />
    
    <!-- Aggressive tiering for JIT -->
    <property name="System.TieredCompilation" value="true" />
  </configProperties>
</runtimeOptions>
```

### Monitoring GC

```csharp
public class GCMonitoring
{
    public void PrintGCStats()
    {
        // Generation sizes
        var gen0Size = GC.GetTotalMemory(false);
        
        // Collection counts
        long gen0Collections = GC.CollectionCount(0);
        long gen1Collections = GC.CollectionCount(1);
        long gen2Collections = GC.CollectionCount(2);
        
        Console.WriteLine($"Gen0 collections: {gen0Collections}");
        Console.WriteLine($"Gen1 collections: {gen1Collections}");
        Console.WriteLine($"Gen2 collections: {gen2Collections}");
        
        // Get memory info (.NET 5+)
        GCMemoryInfo info = GC.GetGCMemoryInfo();
        Console.WriteLine($"Heap size: {info.HeapSizeBytes}");
        Console.WriteLine($"Fragmented: {info.FragmentedBytes}");
    }
}
```

### Reducing allocations (best strategy)

```csharp
// ❌ High allocation
public void BadAllocation()
{
    for (int i = 0; i < 1_000_000; i++)
    {
        var list = new List<int>();
        for (int j = 0; j < 10; j++)
            list.Add(j);
    }
}

// ✅ Reuse objects
public void GoodAllocation()
{
    var list = new List<int>(10);
    for (int i = 0; i < 1_000_000; i++)
    {
        list.Clear();
        for (int j = 0; j < 10; j++)
            list.Add(j);
    }
}

// ✅ Stack allocation
public void StackAllocation()
{
    Span<int> span = stackalloc int[10];
    for (int i = 0; i < 1_000_000; i++)
    {
        // Use span - no heap allocation
    }
}
```

---

## 10. Best practices

### ✅ Recommended

```csharp
// 1. Use pooling dla high-allocation scenarios
private static ArrayPool<byte> _pool = ArrayPool<byte>.Shared;

public void EfficientBufferUsage()
{
    byte[] buffer = _pool.Rent(1024);
    try
    {
        // Use buffer
    }
    finally
    {
        _pool.Return(buffer);  // Return to pool
    }
}

// 2. Prefer stack allocation dla small data
public void ProcessData()
{
    Span<byte> buffer = stackalloc byte[256];
    // No GC pressure
}

// 3. Implement IDisposable properly
public sealed class Resource : IDisposable
{
    private IntPtr _handle;
    
    public void Dispose()
    {
        if (_handle != IntPtr.Zero)
        {
            // Cleanup native resources
            Marshal.FreeHGlobal(_handle);
            _handle = IntPtr.Zero;
        }
        GC.SuppressFinalize(this);
    }
}

// 4. Monitor GC events in production
GC.RegisterForFullGCNotification(10, 10);
var thread = new Thread(WaitForGC);
thread.IsBackground = true;
thread.Start();

void WaitForGC()
{
    while (true)
    {
        if (GC.WaitForFullGCApproach() == GCNotificationStatus.Succeeded)
        {
            // Prepare for GC pause
            Console.WriteLine("Full GC approaching");
        }
    }
}
```

### ❌ Anti-patterns

```csharp
// Don't: Call GC.Collect() in production
public void BadPractice()
{
    GC.Collect();  // NEVER without profiling!
    GC.WaitForPendingFinalizers();
    GC.Collect();
}

// Don't: Create finalizers unnecessarily
public class BadFinalizer
{
    ~BadFinalizer()  // Avoid unless unmanaged resources!
    {
    }
}

// Don't: Hold onto large objects unnecessarily
public void ResourceLeak()
{
    var largeArray = new byte[100_000_000];
    // Never cleared - keeps growing memory usage
}

// Don't: Create many event handlers without cleanup
public void EventLeak()
{
    MyEvent += (s, e) => Console.WriteLine("handler");
    // Anonymous handler never unsubscribed
}
```

---

## 11. GC pauses – real-world impact

### Pause time expectations

```
Workstation GC:
- Gen 0: ~1-5ms
- Gen 1: ~5-50ms
- Gen 2: ~100-1000ms (full GC)

Server GC (parallel):
- Gen 0: ~2-10ms
- Gen 1: ~20-100ms
- Gen 2: ~500-2000ms (full GC, high throughput)

Background GC:
- Gen 0: ~1-3ms
- Gen 1: ~5-20ms
- Gen 2: ~5-50ms (concurrent, minimal pauses)
```

### Measuring GC pauses

```csharp
public class GCPauseMeasurement
{
    private long _lastGCCount;
    
    public void MeasurePause()
    {
        _lastGCCount = GC.CollectionCount(2);
        
        var sw = Stopwatch.StartNew();
        
        // Do work that triggers GC
        var list = new List<byte[]>();
        for (int i = 0; i < 1000; i++)
            list.Add(new byte[10000]);
        
        sw.Stop();
        
        long currentGCCount = GC.CollectionCount(2);
        if (currentGCCount > _lastGCCount)
        {
            Console.WriteLine($"GC occurred during {sw.ElapsedMilliseconds}ms operation");
        }
    }
}
```

---

## Podsumowanie

**Garbage Collector** w .NET to automatyczny system zarządzania pamięcią:

- **Generational model**: Gen 0 → Gen 1 → Gen 2, szybkie zbieranie młodych obiektów
- **Mark & Sweep**: Oznacz osiągalne, usuń resztę
- **Concurrent**: Background GC zmniejsza pauses
- **Modes**: Workstation (latency), Server (throughput), Background (low latency)

**Kluczowe mechanizmy:**
- Roots (stack, static, handles) → object graph traversal
- Reachability determines lifetime
- Compaction zmienia adresy obiektów
- LOH (Large Object Heap) dla dużych alokacji

**Developer powinien:**
- Rozumieć generacyjny model i impact na wydajność
- Minimalizować alokacje (pooling, stackalloc, Span<T>)
- Prawidłowo implementować Dispose pattern
- Monitorować GC w produkcji
- Unikać common leaks (event handlers, static caches)
- Profilować zamiast guessować

GC to kompleksowy system – wiedza o jego działaniu jest kluczowa do optymalizacji wydajności .NET applications, szczególnie dla high-throughput, low-latency scenariuszy.