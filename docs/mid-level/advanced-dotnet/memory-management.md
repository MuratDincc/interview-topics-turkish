# Memory Management

## Giriş

Memory Management, .NET uygulamalarında memory allocation, garbage collection ve memory optimization için kritik bir konudur. Mid-level geliştiriciler için memory management'ı anlamak, performance optimization, memory leaks prevention ve resource management için esastır. Bu dosya, garbage collection, memory allocation patterns, memory profiling ve optimization techniques konularını kapsar.

## Garbage Collection

### 1. GC Configuration
Garbage collection'ı optimize eden configuration.

```csharp
public class GarbageCollectionService
{
    private readonly ILogger<GarbageCollectionService> _logger;
    private readonly IConfiguration _configuration;
    
    public GarbageCollectionService(ILogger<GarbageCollectionService> logger, IConfiguration configuration)
    {
        _logger = logger;
        _configuration = configuration;
    }
    
    public void ConfigureGarbageCollection()
    {
        // Enable server GC for better performance on multi-core systems
        if (Environment.ProcessorCount > 1)
        {
            GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce;
            _logger.LogInformation("Server GC enabled for {ProcessorCount} processors", Environment.ProcessorCount);
        }
        
        // Configure GC latency mode
        var latencyMode = _configuration.GetValue<GCLatencyMode>("GarbageCollection:LatencyMode", GCLatencyMode.Interactive);
        GCSettings.LatencyMode = latencyMode;
        
        // Set memory pressure threshold
        var memoryPressureThreshold = _configuration.GetValue<double>("GarbageCollection:MemoryPressureThreshold", 0.8);
        GC.AddMemoryPressure((long)(GC.GetTotalMemory(false) * memoryPressureThreshold));
        
        _logger.LogInformation("GC configured with latency mode: {LatencyMode}", latencyMode);
    }
    
    public void ForceGarbageCollection()
    {
        var beforeMemory = GC.GetTotalMemory(false);
        var beforeGen0 = GC.CollectionCount(0);
        var beforeGen1 = GC.CollectionCount(1);
        var beforeGen2 = GC.CollectionCount(2);
        
        GC.Collect();
        GC.WaitForPendingFinalizers();
        GC.Collect();
        
        var afterMemory = GC.GetTotalMemory(false);
        var afterGen0 = GC.CollectionCount(0);
        var afterGen1 = GC.CollectionCount(1);
        var afterGen2 = GC.CollectionCount(2);
        
        _logger.LogInformation("Forced GC completed. Memory: {Before} -> {After} bytes, " +
            "Collections: Gen0: {Gen0Diff}, Gen1: {Gen1Diff}, Gen2: {Gen2Diff}",
            beforeMemory, afterMemory,
            afterGen0 - beforeGen0, afterGen1 - beforeGen1, afterGen2 - beforeGen2);
    }
    
    public MemoryInfo GetMemoryInfo()
    {
        return new MemoryInfo
        {
            TotalMemory = GC.GetTotalMemory(false),
            TotalMemoryForce = GC.GetTotalMemory(true),
            Gen0CollectionCount = GC.CollectionCount(0),
            Gen1CollectionCount = GC.CollectionCount(1),
            Gen2CollectionCount = GC.CollectionCount(2),
            MaxGeneration = GC.MaxGeneration,
            LatencyMode = GCSettings.LatencyMode,
            IsServerGC = GCSettings.IsServerGC,
            LargeObjectHeapCompactionMode = GCSettings.LargeObjectHeapCompactionMode
        };
    }
}

public class MemoryInfo
{
    public long TotalMemory { get; set; }
    public long TotalMemoryForce { get; set; }
    public int Gen0CollectionCount { get; set; }
    public int Gen1CollectionCount { get; set; }
    public int Gen2CollectionCount { get; set; }
    public int MaxGeneration { get; set; }
    public GCLatencyMode LatencyMode { get; set; }
    public bool IsServerGC { get; set; }
    public GCLargeObjectHeapCompactionMode LargeObjectHeapCompactionMode { get; set; }
}
```

### 2. Memory Pool Implementation
Memory allocation'ı optimize eden pool implementation.

```csharp
public class MemoryPool<T> : IDisposable where T : class
{
    private readonly ConcurrentQueue<T> _pool;
    private readonly Func<T> _factory;
    private readonly Action<T> _reset;
    private readonly int _maxSize;
    private int _currentSize;
    
    public MemoryPool(Func<T> factory, Action<T> reset = null, int maxSize = 100)
    {
        _pool = new ConcurrentQueue<T>();
        _factory = factory ?? throw new ArgumentNullException(nameof(factory));
        _reset = reset;
        _maxSize = maxSize;
    }
    
    public T Rent()
    {
        if (_pool.TryDequeue(out T item))
        {
            Interlocked.Decrement(ref _currentSize);
            return item;
        }
        
        return _factory();
    }
    
    public void Return(T item)
    {
        if (item == null) return;
        
        if (_currentSize < _maxSize)
        {
            _reset?.Invoke(item);
            _pool.Enqueue(item);
            Interlocked.Increment(ref _currentSize);
        }
    }
    
    public void Dispose()
    {
        while (_pool.TryDequeue(out _)) { }
    }
}

public class ArrayPoolService
{
    private readonly ArrayPool<byte> _bytePool;
    private readonly ArrayPool<char> _charPool;
    private readonly ILogger<ArrayPoolService> _logger;
    
    public ArrayPoolService(ILogger<ArrayPoolService> logger)
    {
        _bytePool = ArrayPool<byte>.Shared;
        _charPool = ArrayPool<char>.Shared;
        _logger = logger;
    }
    
    public async Task<string> ProcessLargeDataAsync(Stream stream)
    {
        var buffer = _bytePool.Rent(8192);
        var charBuffer = _charPool.Rent(8192);
        
        try
        {
            var result = new StringBuilder();
            int bytesRead;
            
            while ((bytesRead = await stream.ReadAsync(buffer, 0, buffer.Length)) > 0)
            {
                var charCount = Encoding.UTF8.GetChars(buffer, 0, bytesRead, charBuffer, 0);
                result.Append(charBuffer, 0, charCount);
            }
            
            return result.ToString();
        }
        finally
        {
            _bytePool.Return(buffer);
            _charPool.Return(charBuffer);
        }
    }
    
    public async Task<byte[]> CompressDataAsync(byte[] data)
    {
        var compressedBuffer = _bytePool.Rent(data.Length);
        
        try
        {
            using var outputStream = new MemoryStream();
            using var gzipStream = new GZipStream(outputStream, CompressionMode.Compress);
            
            await gzipStream.WriteAsync(data, 0, data.Length);
            await gzipStream.FlushAsync();
            
            return outputStream.ToArray();
        }
        finally
        {
            _bytePool.Return(compressedBuffer);
        }
    }
}
```

## Memory Profiling

### 1. Memory Profiler
Memory usage'ı izleyen profiler.

```csharp
public class MemoryProfiler : IDisposable
{
    private readonly ILogger<MemoryProfiler> _logger;
    private readonly Timer _timer;
    private readonly List<MemorySnapshot> _snapshots;
    private readonly object _lock = new object();
    
    public MemoryProfiler(ILogger<MemoryProfiler> logger, TimeSpan interval)
    {
        _logger = logger;
        _snapshots = new List<MemorySnapshot>();
        _timer = new Timer(CollectSnapshot, null, TimeSpan.Zero, interval);
    }
    
    private void CollectSnapshot(object state)
    {
        var snapshot = new MemorySnapshot
        {
            Timestamp = DateTime.UtcNow,
            TotalMemory = GC.GetTotalMemory(false),
            Gen0CollectionCount = GC.CollectionCount(0),
            Gen1CollectionCount = GC.CollectionCount(1),
            Gen2CollectionCount = GC.CollectionCount(2),
            WorkingSet = Environment.WorkingSet,
            PrivateMemory = Environment.WorkingSet
        };
        
        lock (_lock)
        {
            _snapshots.Add(snapshot);
            
            // Keep only last 1000 snapshots
            if (_snapshots.Count > 1000)
            {
                _snapshots.RemoveAt(0);
            }
        }
        
        // Check for memory leaks
        CheckForMemoryLeaks(snapshot);
    }
    
    private void CheckForMemoryLeaks(MemorySnapshot current)
    {
        if (_snapshots.Count < 10) return;
        
        var recentSnapshots = _snapshots.TakeLast(10).ToList();
        var memoryGrowth = recentSnapshots.Last().TotalMemory - recentSnapshots.First().TotalMemory;
        var timeSpan = recentSnapshots.Last().Timestamp - recentSnapshots.First().Timestamp;
        
        if (memoryGrowth > 100 * 1024 * 1024 && timeSpan.TotalMinutes > 5) // 100MB growth in 5 minutes
        {
            _logger.LogWarning("Potential memory leak detected. Memory growth: {Growth} bytes in {TimeSpan}",
                memoryGrowth, timeSpan);
        }
    }
    
    public MemoryAnalysis AnalyzeMemoryUsage()
    {
        lock (_lock)
        {
            if (_snapshots.Count < 2) return new MemoryAnalysis();
            
            var first = _snapshots.First();
            var last = _snapshots.Last();
            var timeSpan = last.Timestamp - first.Timestamp;
            
            var memoryGrowth = last.TotalMemory - first.TotalMemory;
            var growthRate = memoryGrowth / timeSpan.TotalMinutes; // bytes per minute
            
            var gcFrequency = (last.Gen2CollectionCount - first.Gen2CollectionCount) / timeSpan.TotalMinutes;
            
            return new MemoryAnalysis
            {
                TotalMemoryGrowth = memoryGrowth,
                MemoryGrowthRate = growthRate,
                GarbageCollectionFrequency = gcFrequency,
                AverageMemoryUsage = _snapshots.Average(s => s.TotalMemory),
                PeakMemoryUsage = _snapshots.Max(s => s.TotalMemory),
                AnalysisDuration = timeSpan
            };
        }
    }
    
    public void Dispose()
    {
        _timer?.Dispose();
    }
}

public class MemorySnapshot
{
    public DateTime Timestamp { get; set; }
    public long TotalMemory { get; set; }
    public int Gen0CollectionCount { get; set; }
    public int Gen1CollectionCount { get; set; }
    public int Gen2CollectionCount { get; set; }
    public long WorkingSet { get; set; }
    public long PrivateMemory { get; set; }
}

public class MemoryAnalysis
{
    public long TotalMemoryGrowth { get; set; }
    public double MemoryGrowthRate { get; set; }
    public double GarbageCollectionFrequency { get; set; }
    public double AverageMemoryUsage { get; set; }
    public long PeakMemoryUsage { get; set; }
    public TimeSpan AnalysisDuration { get; set; }
}
```

## Mülakat Soruları

### Temel Sorular

1. **Garbage Collection nasıl çalışır?**
   - **Cevap**: Generational collection, mark-and-sweep algorithm, automatic memory management.

2. **Memory pools neden kullanılır?**
   - **Cevap**: Reduce allocation overhead, improve performance, reduce GC pressure.

3. **Memory leaks nasıl tespit edilir?**
   - **Cevap**: Memory profiling, growth analysis, GC monitoring, tools like dotMemory.

4. **Large Object Heap nedir?**
   - **Cevap**: 85KB+ objects, separate GC generation, compaction challenges.

5. **Memory optimization techniques nelerdir?**
   - **Cevap**: Object pooling, value types, structs, array pooling, GC tuning.

### Teknik Sorular

1. **GC.Collect() ne zaman kullanılır?**
   - **Cevap**: Rarely, only in specific scenarios like memory pressure, testing.

2. **Memory pressure nasıl yönetilir?**
   - **Cevap**: GC.AddMemoryPressure(), monitoring, proactive cleanup.

3. **ArrayPool nasıl implement edilir?**
   - **Cevap**: Rent/Return pattern, buffer reuse, automatic cleanup.

4. **Memory profiling nasıl yapılır?**
   - **Cevap**: Custom profilers, performance counters, third-party tools.

5. **Memory leaks nasıl önlenir?**
   - **Cevap**: Proper disposal, weak references, event unsubscription, resource cleanup.

## Best Practices

1. **Memory Allocation**
   - Object pooling kullanın
   - Value types tercih edin
   - ArrayPool implement edin
   - Large allocations minimize edin

2. **Garbage Collection**
   - GC.Collect() avoid edin
   - Server GC enable edin
   - Latency mode optimize edin
   - Memory pressure monitor edin

3. **Memory Monitoring**
   - Custom profilers implement edin
   - Memory growth track edin
   - GC frequency monitor edin
   - Leak detection automate edin

4. **Performance Optimization**
   - Allocation patterns optimize edin
   - Memory access patterns improve edin
   - Cache locality optimize edin
   - Memory fragmentation minimize edin

5. **Resource Management**
   - IDisposable implement edin
   - Using statements kullanın
   - Finalizers avoid edin
   - Resource cleanup automate edin

## Kaynaklar

- [Garbage Collection](https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/)
- [Memory Management](https://docs.microsoft.com/en-us/dotnet/standard/memory-and-spans/)
- [ArrayPool](https://docs.microsoft.com/en-us/dotnet/api/system.buffers.arraypool-1)
- [Memory Profiling](https://docs.microsoft.com/en-us/dotnet/framework/debug-trace-profile/memory-profiling)
- [Performance Best Practices](https://docs.microsoft.com/en-us/dotnet/framework/performance/)
