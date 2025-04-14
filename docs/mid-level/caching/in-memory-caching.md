# In-Memory Caching

## Giriş

In-Memory Caching, verilerin uygulama belleğinde geçici olarak saklanması işlemidir. .NET'te `IMemoryCache` arayüzü ve `MemoryCache` sınıfı ile kolayca uygulanabilir.

## In-Memory Caching'in Önemi

1. **Performans**
   - Veritabanı yükünü azaltma
   - Yanıt sürelerini kısaltma
   - CPU kullanımını optimize etme

2. **Ölçeklenebilirlik**
   - Uygulama yükünü dağıtma
   - Kaynak kullanımını optimize etme
   - Sistem kapasitesini artırma

3. **Maliyet**
   - Veritabanı maliyetlerini azaltma
   - Ağ trafiğini azaltma
   - İşlem maliyetlerini düşürme

## In-Memory Caching Türleri

1. **MemoryCache**
   - Temel in-memory caching
   - Thread-safe
   - Expiration policies

2. **Distributed Memory Cache**
   - Birden fazla sunucu arasında paylaşım
   - Yüksek ölçeklenebilirlik
   - Fault tolerance

3. **Hybrid Cache**
   - Memory ve distributed cache kombinasyonu
   - Esnek yapı
   - Optimize edilmiş performans

## In-Memory Caching Kullanımı

1. **Temel Kullanım**
```csharp
public class CacheService
{
    private readonly IMemoryCache _cache;
    private readonly MemoryCacheEntryOptions _options;

    public CacheService(IMemoryCache cache)
    {
        _cache = cache;
        _options = new MemoryCacheEntryOptions()
            .SetSlidingExpiration(TimeSpan.FromMinutes(30))
            .SetAbsoluteExpiration(TimeSpan.FromHours(1));
    }

    public T GetOrCreate<T>(string key, Func<T> factory)
    {
        return _cache.GetOrCreate(key, entry =>
        {
            entry.SetOptions(_options);
            return factory();
        });
    }
}
```

2. **Async Kullanım**
```csharp
public class AsyncCacheService
{
    private readonly IMemoryCache _cache;
    private readonly MemoryCacheEntryOptions _options;

    public AsyncCacheService(IMemoryCache cache)
    {
        _cache = cache;
        _options = new MemoryCacheEntryOptions()
            .SetSlidingExpiration(TimeSpan.FromMinutes(30))
            .SetAbsoluteExpiration(TimeSpan.FromHours(1));
    }

    public async Task<T> GetOrCreateAsync<T>(string key, Func<Task<T>> factory)
    {
        if (_cache.TryGetValue(key, out T cachedValue))
        {
            return cachedValue;
        }

        var value = await factory();
        _cache.Set(key, value, _options);
        return value;
    }
}
```

3. **Cache Invalidation**
```csharp
public class CacheInvalidator
{
    private readonly IMemoryCache _cache;
    private readonly ILogger<CacheInvalidator> _logger;

    public CacheInvalidator(IMemoryCache cache, ILogger<CacheInvalidator> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public void Invalidate(string key)
    {
        _cache.Remove(key);
        _logger.LogInformation("Cache invalidated for key: {Key}", key);
    }

    public void InvalidateByPattern(string pattern)
    {
        var keys = GetKeysByPattern(pattern);
        foreach (var key in keys)
        {
            Invalidate(key);
        }
    }
}
```

## In-Memory Caching Best Practices

1. **Cache Key Tasarımı**
   - Anlamlı ve tutarlı isimlendirme
   - Namespace kullanımı
   - Versiyonlama

2. **Expiration Stratejileri**
   - Sliding expiration
   - Absolute expiration
   - Priority-based expiration

3. **Memory Yönetimi**
   - Size limitleri
   - Eviction politikaları
   - Memory pressure handling

4. **Error Handling**
   - Fallback mekanizmaları
   - Circuit breaker
   - Retry politikaları

## Mülakat Soruları

### Temel Sorular

1. **In-Memory Caching nedir ve neden önemlidir?**
   - **Cevap**: In-Memory Caching, verilerin uygulama belleğinde geçici olarak saklanmasıdır. Performans iyileştirmesi, ölçeklenebilirlik ve maliyet optimizasyonu sağlar.

2. **MemoryCache ve Distributed Cache arasındaki farklar nelerdir?**
   - **Cevap**:
     - MemoryCache: Tek sunucuda, hızlı, uygulama içi
     - Distributed Cache: Birden fazla sunucuda, ölçeklenebilir, paylaşımlı

3. **Cache expiration stratejileri nelerdir?**
   - **Cevap**:
     - Sliding expiration
     - Absolute expiration
     - Priority-based expiration
     - Size-based expiration

4. **Cache invalidation stratejileri nelerdir?**
   - **Cevap**:
     - Zaman tabanlı
     - Olay tabanlı
     - Manuel invalidation
     - Dependency-based

5. **Cache coherency nedir?**
   - **Cevap**:
     - Önbellek tutarlılığı
     - Veri senkronizasyonu
     - Stale data önleme
     - Consistency modelleri

### Teknik Sorular

1. **MemoryCache kullanımı nasıl yapılır?**
   - **Cevap**:
```csharp
public class MemoryCacheService
{
    private readonly IMemoryCache _cache;
    private readonly MemoryCacheEntryOptions _options;

    public MemoryCacheService(IMemoryCache cache)
    {
        _cache = cache;
        _options = new MemoryCacheEntryOptions()
            .SetSlidingExpiration(TimeSpan.FromMinutes(30))
            .SetAbsoluteExpiration(TimeSpan.FromHours(1));
    }

    public T GetOrCreate<T>(string key, Func<T> factory)
    {
        return _cache.GetOrCreate(key, entry =>
        {
            entry.SetOptions(_options);
            return factory();
        });
    }
}
```

2. **Cache monitoring nasıl yapılır?**
   - **Cevap**:
```csharp
public class CacheMonitor
{
    private readonly IMemoryCache _cache;
    private readonly ILogger<CacheMonitor> _logger;

    public CacheMonitor(IMemoryCache cache, ILogger<CacheMonitor> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public void LogCacheStatistics()
    {
        var stats = new
        {
            HitCount = _cache.GetCurrentStatistics()?.TotalHits ?? 0,
            MissCount = _cache.GetCurrentStatistics()?.TotalMisses ?? 0,
            CurrentSize = _cache.GetCurrentStatistics()?.CurrentEntryCount ?? 0
        };

        _logger.LogInformation("Cache Statistics: {@Stats}", stats);
    }
}
```

3. **Cache fallback stratejisi nasıl uygulanır?**
   - **Cevap**:
```csharp
public class CacheWithFallback
{
    private readonly IMemoryCache _cache;
    private readonly ILogger<CacheWithFallback> _logger;

    public CacheWithFallback(IMemoryCache cache, ILogger<CacheWithFallback> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public async Task<T> GetWithFallback<T>(string key, Func<Task<T>> factory, TimeSpan cacheDuration)
    {
        try
        {
            if (_cache.TryGetValue(key, out T cachedValue))
            {
                return cachedValue;
            }

            var value = await factory();
            _cache.Set(key, value, cacheDuration);
            return value;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Cache operation failed for key: {Key}", key);
            return await factory();
        }
    }
}
```

4. **Cache size yönetimi nasıl yapılır?**
   - **Cevap**:
```csharp
public class CacheSizeManager
{
    private readonly IMemoryCache _cache;
    private readonly ILogger<CacheSizeManager> _logger;
    private readonly long _maxSize;

    public CacheSizeManager(IMemoryCache cache, ILogger<CacheSizeManager> logger, long maxSize)
    {
        _cache = cache;
        _logger = logger;
        _maxSize = maxSize;
    }

    public void EnsureCacheSize()
    {
        var currentSize = _cache.GetCurrentStatistics()?.CurrentSize ?? 0;
        if (currentSize > _maxSize)
        {
            _logger.LogWarning("Cache size exceeded limit: {CurrentSize}/{MaxSize}", currentSize, _maxSize);
            EvictOldestItems();
        }
    }

    private void EvictOldestItems()
    {
        // Implement oldest items eviction logic
    }
}
```

5. **Cache warming nasıl yapılır?**
   - **Cevap**:
```csharp
public class CacheWarmer
{
    private readonly IMemoryCache _cache;
    private readonly ILogger<CacheWarmer> _logger;

    public CacheWarmer(IMemoryCache cache, ILogger<CacheWarmer> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public async Task WarmCache<T>(string key, Func<Task<T>> factory, TimeSpan cacheDuration)
    {
        try
        {
            var value = await factory();
            _cache.Set(key, value, cacheDuration);
            _logger.LogInformation("Cache warmed for key: {Key}", key);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to warm cache for key: {Key}", key);
        }
    }
}
```

### İleri Seviye Sorular

1. **Cache coherency sorunları nasıl çözülür?**
   - **Cevap**:
     - Strong consistency modelleri
     - Eventual consistency
     - Versioning
     - Distributed locking
     - Cache synchronization

2. **Cache warming stratejileri nelerdir?**
   - **Cevap**:
     - Startup warming
     - Background warming
     - Predictive warming
     - Scheduled warming
     - On-demand warming

3. **Cache partitioning nasıl yapılır?**
   - **Cevap**:
     - Key-based partitioning
     - Hash-based partitioning
     - Range partitioning
     - Directory partitioning
     - Consistent hashing

4. **Cache monitoring ve alerting nasıl yapılır?**
   - **Cevap**:
     - Performance metrics
     - Hit/miss ratios
     - Memory usage
     - Latency monitoring
     - Custom alerts

5. **Cache security nasıl sağlanır?**
   - **Cevap**:
     - Encryption
     - Access control
     - Data isolation
     - Secure communication
     - Audit logging 