# Bellek Yönetimi

## Genel Bakış
Bellek yönetimi, uygulamanın bellek kullanımını optimize etmek ve bellek sızıntılarını önlemek için yapılan işlemlerdir. .NET'te otomatik bellek yönetimi (Garbage Collection) kullanılır, ancak bazı durumlarda manuel bellek yönetimi de gerekebilir.

## Temel Kavramlar

### 1. Garbage Collection
```csharp
public class MemoryAnalyzer
{
    private readonly ILogger<MemoryAnalyzer> _logger;

    public MemoryAnalyzer(ILogger<MemoryAnalyzer> logger)
    {
        _logger = logger;
    }

    public void AnalyzeMemoryUsage()
    {
        // Bellek kullanımını analiz et
        var process = Process.GetCurrentProcess();
        var memoryUsage = process.WorkingSet64;
        var peakMemoryUsage = process.PeakWorkingSet64;
        
        _logger.LogInformation(
            "Current Memory Usage: {MemoryUsage}MB, Peak: {PeakMemoryUsage}MB",
            memoryUsage / 1024 / 1024,
            peakMemoryUsage / 1024 / 1024);

        // GC istatistiklerini al
        var totalMemory = GC.GetTotalMemory(false);
        var maxGeneration = GC.MaxGeneration;
        
        _logger.LogInformation(
            "Total Memory: {TotalMemory}MB, Max Generation: {MaxGeneration}",
            totalMemory / 1024 / 1024,
            maxGeneration);
    }

    public void ForceGarbageCollection()
    {
        // GC'yi manuel olarak tetikle
        GC.Collect();
        GC.WaitForPendingFinalizers();
        
        _logger.LogInformation("Garbage collection completed");
    }
}
```

### 2. Bellek Havuzu (Object Pool)
```csharp
public class ObjectPool<T> where T : class, new()
{
    private readonly ConcurrentBag<T> _objects;
    private readonly int _maxSize;
    private int _currentSize;

    public ObjectPool(int maxSize = 100)
    {
        _objects = new ConcurrentBag<T>();
        _maxSize = maxSize;
        _currentSize = 0;
    }

    public T Get()
    {
        if (_objects.TryTake(out T item))
        {
            return item;
        }

        if (_currentSize < _maxSize)
        {
            Interlocked.Increment(ref _currentSize);
            return new T();
        }

        throw new InvalidOperationException("Pool limit reached");
    }

    public void Return(T item)
    {
        if (_currentSize <= _maxSize)
        {
            _objects.Add(item);
        }
    }
}

// Kullanım örneği
public class ConnectionPool
{
    private readonly ObjectPool<SqlConnection> _pool;

    public ConnectionPool()
    {
        _pool = new ObjectPool<SqlConnection>(10);
    }

    public async Task<SqlConnection> GetConnectionAsync()
    {
        var connection = _pool.Get();
        if (connection.State != ConnectionState.Open)
        {
            await connection.OpenAsync();
        }
        return connection;
    }

    public void ReturnConnection(SqlConnection connection)
    {
        _pool.Return(connection);
    }
}
```

### 3. IDisposable Pattern
```csharp
public class ResourceManager : IDisposable
{
    private bool _disposed;
    private readonly ILogger<ResourceManager> _logger;
    private readonly List<IDisposable> _resources;

    public ResourceManager(ILogger<ResourceManager> logger)
    {
        _logger = logger;
        _resources = new List<IDisposable>();
    }

    public void AddResource(IDisposable resource)
    {
        _resources.Add(resource);
    }

    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                foreach (var resource in _resources)
                {
                    try
                    {
                        resource.Dispose();
                    }
                    catch (Exception ex)
                    {
                        _logger.LogError(ex, "Error disposing resource");
                    }
                }
                _resources.Clear();
            }
            _disposed = true;
        }
    }

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
}
```

## Best Practices

### 1. Bellek Kullanımı
- Büyük nesneleri dikkatli kullanın
- Bellek sızıntılarını önleyin
- IDisposable pattern'i uygulayın
- Object pooling kullanın
- Bellek fragmentasyonunu izleyin

### 2. Garbage Collection
- GC'yi manuel tetiklemekten kaçının
- Finalizer'ları dikkatli kullanın
- Weak references kullanın
- Large Object Heap'ı yönetin
- GC modlarını anlayın

### 3. Bellek İzleme
- Memory profiler kullanın
- Bellek kullanımını loglayın
- Bellek sızıntılarını tespit edin
- Performans metriklerini izleyin
- Alarm mekanizmaları kurun

## Sık Sorulan Sorular

### 1. Bellek sızıntısı nedir ve nasıl önlenir?
- Event handler'ları düzgün temizleyin
- IDisposable nesneleri dispose edin
- Static koleksiyonları dikkatli kullanın
- Timer'ları düzgün yönetin
- Weak references kullanın

### 2. Garbage Collection nasıl çalışır?
- Generations (0, 1, 2)
- Mark and Sweep algoritması
- Finalization queue
- Large Object Heap
- GC modları (Workstation, Server)

### 3. Bellek optimizasyonu için hangi araçlar kullanılabilir?
- Visual Studio Memory Profiler
- dotMemory
- PerfView
- CLR Profiler
- Windows Performance Monitor

## Kaynaklar
- [.NET Garbage Collection](https://docs.microsoft.com/tr-tr/dotnet/standard/garbage-collection/)
- [Memory Management Best Practices](https://docs.microsoft.com/tr-tr/dotnet/standard/garbage-collection/memory-management-and-gc)
- [Object Pooling in .NET](https://docs.microsoft.com/tr-tr/dotnet/standard/collections/thread-safe/object-pooling)
- [IDisposable Pattern](https://docs.microsoft.com/tr-tr/dotnet/standard/garbage-collection/implementing-dispose) 