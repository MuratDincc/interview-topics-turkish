# Caching Stratejileri

## Giriş

Caching (önbellekleme), uygulama performansını artırmak için sık kullanılan verilerin geçici olarak saklanması işlemidir. .NET uygulamalarında farklı caching stratejileri ve yaklaşımları bulunmaktadır.

## Caching'in Önemi

1. **Performans İyileştirmesi**
   - Veritabanı yükünü azaltma
   - Yanıt sürelerini kısaltma
   - Sistem kaynaklarını verimli kullanma

2. **Ölçeklenebilirlik**
   - Yük dengeleme
   - Sistem yükünü dağıtma
   - Kaynak kullanımını optimize etme

3. **Maliyet Optimizasyonu**
   - Veritabanı maliyetlerini azaltma
   - Bant genişliği kullanımını optimize etme
   - İşlem maliyetlerini düşürme

## Caching Türleri

1. **In-Memory Caching**
   - Uygulama içi önbellekleme
   - Distributed Cache
   - Memory Cache

2. **Distributed Caching**
   - Redis
   - NCache
   - Memcached

3. **Response Caching**
   - HTTP Response Caching
   - Output Caching
   - Response Compression

4. **Database Caching**
   - Query Result Caching
   - Stored Procedure Caching
   - Materialized Views

## Caching Stratejileri

1. **Cache-Aside**
   - Veri isteğe bağlı olarak önbelleğe alınır
   - Uygulama önbelleği yönetir
   - Esnek ve kontrol edilebilir

2. **Read-Through**
   - Önbellek veritabanından okur
   - Uygulama önbelleğe erişir
   - Şeffaf ve basit

3. **Write-Through**
   - Veri hem önbelleğe hem veritabanına yazılır
   - Veri tutarlılığı sağlar
   - Performans maliyeti vardır

4. **Write-Behind**
   - Veri önce önbelleğe yazılır
   - Veritabanına asenkron yazılır
   - Yüksek performans sağlar

## Caching Best Practices

1. **Cache Key Tasarımı**
   - Anlamlı ve tutarlı isimlendirme
   - Versiyonlama
   - Namespace kullanımı

2. **Cache Invalidation**
   - Zaman tabanlı
   - Olay tabanlı
   - Manuel invalidation

3. **Cache Size Yönetimi**
   - Memory limitleri
   - Eviction politikaları
   - Monitoring

4. **Error Handling**
   - Fallback mekanizmaları
   - Circuit breaker
   - Retry politikaları

## Mülakat Soruları

### Temel Sorular

1. **Caching nedir ve neden önemlidir?**
   - **Cevap**: Caching, sık kullanılan verilerin geçici olarak saklanması işlemidir. Performans iyileştirmesi, ölçeklenebilirlik ve maliyet optimizasyonu sağlar.

2. **In-Memory ve Distributed Caching arasındaki farklar nelerdir?**
   - **Cevap**:
     - In-Memory: Tek sunucuda, hızlı, uygulama içi
     - Distributed: Birden fazla sunucuda, ölçeklenebilir, paylaşımlı

3. **Cache invalidation stratejileri nelerdir?**
   - **Cevap**:
     - Zaman tabanlı (TTL)
     - Olay tabanlı
     - Manuel invalidation
     - Dependency-based

4. **Cache-aside pattern nedir?**
   - **Cevap**:
     - Veri isteğe bağlı önbelleğe alınır
     - Uygulama önbelleği yönetir
     - Esnek ve kontrol edilebilir
     - Yaygın kullanılan pattern

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

2. **Distributed Cache kullanımı nasıl yapılır?**
   - **Cevap**:
```csharp
public class DistributedCacheService
{
    private readonly IDistributedCache _cache;
    private readonly DistributedCacheEntryOptions _options;

    public DistributedCacheService(IDistributedCache cache)
    {
        _cache = cache;
        _options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1),
            SlidingExpiration = TimeSpan.FromMinutes(30)
        };
    }

    public async Task<T> GetOrCreateAsync<T>(string key, Func<Task<T>> factory)
    {
        var cached = await _cache.GetAsync(key);
        if (cached != null)
        {
            return JsonSerializer.Deserialize<T>(cached);
        }

        var result = await factory();
        await _cache.SetAsync(key, JsonSerializer.SerializeToUtf8Bytes(result), _options);
        return result;
    }
}
```

3. **Cache invalidation nasıl yapılır?**
   - **Cevap**:
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