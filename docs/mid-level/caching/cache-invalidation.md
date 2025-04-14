# Cache Invalidation

## Giriş

Cache Invalidation, önbellekteki verilerin geçerliliğini yitirdiğinde veya güncellendiğinde, bu verilerin önbellekten kaldırılması veya güncellenmesi işlemidir. Doğru cache invalidation stratejileri, veri tutarlılığını sağlamak için kritik öneme sahiptir.

## Cache Invalidation'in Önemi

1. **Veri Tutarlılığı**
   - Güncel veri sağlama
   - Stale data önleme
   - Veri senkronizasyonu

2. **Performans**
   - Gereksiz önbellek kullanımını önleme
   - Bellek optimizasyonu
   - Sistem kaynaklarının verimli kullanımı

3. **Güvenilirlik**
   - Doğru veri sağlama
   - Hata önleme
   - Sistem stabilitesi

## Cache Invalidation Stratejileri

1. **Time-Based Invalidation**
   - Absolute expiration
   - Sliding expiration
   - Hybrid expiration

2. **Event-Based Invalidation**
   - Veri değişikliği
   - Sistem olayları
   - Kullanıcı aksiyonları

3. **Dependency-Based Invalidation**
   - Veri bağımlılıkları
   - İlişkisel veriler
   - Cascade invalidation

4. **Manual Invalidation**
   - Kullanıcı tetiklemeli
   - Admin kontrollü
   - Sistem yönetimi

## Cache Invalidation Kullanımı

1. **Time-Based Invalidation**
```csharp
public class TimeBasedCache
{
    private readonly IMemoryCache _cache;
    private readonly ILogger<TimeBasedCache> _logger;

    public TimeBasedCache(IMemoryCache cache, ILogger<TimeBasedCache> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public void SetWithExpiration<T>(string key, T value, TimeSpan expiration)
    {
        var options = new MemoryCacheEntryOptions()
            .SetAbsoluteExpiration(expiration)
            .RegisterPostEvictionCallback((key, value, reason, state) =>
            {
                _logger.LogInformation("Cache entry {Key} was evicted due to {Reason}", key, reason);
            });

        _cache.Set(key, value, options);
    }
}
```

2. **Event-Based Invalidation**
```csharp
public class EventBasedCache
{
    private readonly IMemoryCache _cache;
    private readonly ILogger<EventBasedCache> _logger;

    public EventBasedCache(IMemoryCache cache, ILogger<EventBasedCache> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public void InvalidateOnEvent(string key, IObservable<object> eventStream)
    {
        eventStream.Subscribe(_ =>
        {
            _cache.Remove(key);
            _logger.LogInformation("Cache entry {Key} was invalidated due to event", key);
        });
    }
}
```

3. **Dependency-Based Invalidation**
```csharp
public class DependencyBasedCache
{
    private readonly IMemoryCache _cache;
    private readonly ILogger<DependencyBasedCache> _logger;

    public DependencyBasedCache(IMemoryCache cache, ILogger<DependencyBasedCache> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public void InvalidateDependencies(string key, IEnumerable<string> dependencies)
    {
        foreach (var dependency in dependencies)
        {
            _cache.Remove(dependency);
            _logger.LogInformation("Cache entry {Dependency} was invalidated due to dependency on {Key}", dependency, key);
        }
    }
}
```

4. **Manual Invalidation**
```csharp
public class ManualCacheInvalidator
{
    private readonly IMemoryCache _cache;
    private readonly ILogger<ManualCacheInvalidator> _logger;

    public ManualCacheInvalidator(IMemoryCache cache, ILogger<ManualCacheInvalidator> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public void Invalidate(string key)
    {
        _cache.Remove(key);
        _logger.LogInformation("Cache entry {Key} was manually invalidated", key);
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

## Cache Invalidation Best Practices

1. **Strateji Seçimi**
   - Veri türüne göre strateji
   - Kullanım senaryosuna göre strateji
   - Performans gereksinimlerine göre strateji

2. **Monitoring**
   - Invalidation oranları
   - Cache hit/miss oranları
   - Performans metrikleri
   - Hata izleme

3. **Error Handling**
   - Graceful degradation
   - Fallback mekanizmaları
   - Retry politikaları
   - Logging

4. **Performance**
   - Batch invalidation
   - Asenkron invalidation
   - Lazy invalidation
   - Optimize edilmiş algoritmalar

## Mülakat Soruları

### Temel Sorular

1. **Cache Invalidation nedir ve neden önemlidir?**
   - **Cevap**: Cache Invalidation, önbellekteki verilerin geçerliliğini yitirdiğinde veya güncellendiğinde, bu verilerin önbellekten kaldırılması veya güncellenmesi işlemidir. Veri tutarlılığı, performans ve güvenilirlik için kritik öneme sahiptir.

2. **Farklı cache invalidation stratejileri nelerdir?**
   - **Cevap**:
     - Time-Based
     - Event-Based
     - Dependency-Based
     - Manual

3. **Cache coherency nedir?**
   - **Cevap**:
     - Veri tutarlılığı
     - Senkronizasyon
     - Consistency modelleri
     - Conflict resolution

4. **Stale data nedir ve nasıl önlenir?**
   - **Cevap**:
     - Eski veri
     - Invalidation stratejileri
     - TTL kullanımı
     - Versioning

5. **Cache warming nedir?**
   - **Cevap**:
     - Önceden yükleme
     - Performans optimizasyonu
     - Cold start önleme
     - Kullanım senaryoları

### Teknik Sorular

1. **Time-based cache invalidation nasıl uygulanır?**
   - **Cevap**:
```csharp
public class TimeBasedInvalidation
{
    private readonly IMemoryCache _cache;
    private readonly ILogger<TimeBasedInvalidation> _logger;

    public TimeBasedInvalidation(IMemoryCache cache, ILogger<TimeBasedInvalidation> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public void SetWithExpiration<T>(string key, T value, TimeSpan expiration)
    {
        var options = new MemoryCacheEntryOptions()
            .SetAbsoluteExpiration(expiration)
            .RegisterPostEvictionCallback((key, value, reason, state) =>
            {
                _logger.LogInformation("Cache entry {Key} was evicted due to {Reason}", key, reason);
            });

        _cache.Set(key, value, options);
    }
}
```

2. **Event-based cache invalidation nasıl uygulanır?**
   - **Cevap**:
```csharp
public class EventBasedInvalidation
{
    private readonly IMemoryCache _cache;
    private readonly ILogger<EventBasedInvalidation> _logger;

    public EventBasedInvalidation(IMemoryCache cache, ILogger<EventBasedInvalidation> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public void InvalidateOnEvent(string key, IObservable<object> eventStream)
    {
        eventStream.Subscribe(_ =>
        {
            _cache.Remove(key);
            _logger.LogInformation("Cache entry {Key} was invalidated due to event", key);
        });
    }
}
```

3. **Dependency-based cache invalidation nasıl uygulanır?**
   - **Cevap**:
```csharp
public class DependencyBasedInvalidation
{
    private readonly IMemoryCache _cache;
    private readonly ILogger<DependencyBasedInvalidation> _logger;

    public DependencyBasedInvalidation(IMemoryCache cache, ILogger<DependencyBasedInvalidation> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public void InvalidateDependencies(string key, IEnumerable<string> dependencies)
    {
        foreach (var dependency in dependencies)
        {
            _cache.Remove(dependency);
            _logger.LogInformation("Cache entry {Dependency} was invalidated due to dependency on {Key}", dependency, key);
        }
    }
}
```

4. **Cache monitoring nasıl yapılır?**
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

5. **Cache fallback stratejisi nasıl uygulanır?**
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

### İleri Seviye Sorular

1. **Distributed cache invalidation nasıl yapılır?**
   - **Cevap**:
     - Pub/Sub pattern
     - Event sourcing
     - Message queues
     - Distributed locks
     - Consensus algorithms

2. **Cache coherency sorunları nasıl çözülür?**
   - **Cevap**:
     - Strong consistency
     - Eventual consistency
     - Read-your-writes
     - Monotonic reads
     - Consistent prefix

3. **Cache warming stratejileri nelerdir?**
   - **Cevap**:
     - Startup warming
     - Background warming
     - Predictive warming
     - Scheduled warming
     - On-demand warming

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