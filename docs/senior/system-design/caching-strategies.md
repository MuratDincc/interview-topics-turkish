# Önbellekleme Stratejileri (Caching Strategies)

## Genel Bakış
Önbellekleme, sık erişilen verilerin hızlı erişilebilir bir depoda saklanarak sistem performansını artıran bir tekniktir. Farklı önbellekleme stratejileri, veri tutarlılığı, ölçeklenebilirlik ve performans gereksinimlerine göre kullanılır.

## Temel Kavramlar

### 1. In-Memory Caching
```csharp
public class InMemoryCacheService : ICacheService
{
    private readonly ILogger<InMemoryCacheService> _logger;
    private readonly IMemoryCache _cache;
    private readonly TimeSpan _defaultExpiration;

    public InMemoryCacheService(
        ILogger<InMemoryCacheService> logger,
        IMemoryCache cache,
        IConfiguration configuration)
    {
        _logger = logger;
        _cache = cache;
        _defaultExpiration = TimeSpan.FromMinutes(
            configuration.GetValue<int>("Cache:DefaultExpirationMinutes", 30));
    }

    public async Task<T> GetOrSetAsync<T>(string key, Func<Task<T>> factory)
    {
        if (_cache.TryGetValue(key, out T cachedValue))
        {
            return cachedValue;
        }

        var value = await factory();
        var cacheEntryOptions = new MemoryCacheEntryOptions()
            .SetAbsoluteExpiration(_defaultExpiration)
            .SetPriority(CacheItemPriority.Normal)
            .RegisterPostEvictionCallback((key, value, reason, state) =>
            {
                _logger.LogInformation($"Cache entry {key} was evicted due to {reason}");
            });

        _cache.Set(key, value, cacheEntryOptions);
        return value;
    }

    public Task RemoveAsync(string key)
    {
        _cache.Remove(key);
        return Task.CompletedTask;
    }
}
```

### 2. Distributed Caching
```csharp
public class RedisCacheService : ICacheService
{
    private readonly ILogger<RedisCacheService> _logger;
    private readonly IDistributedCache _cache;
    private readonly DistributedCacheEntryOptions _defaultOptions;

    public RedisCacheService(
        ILogger<RedisCacheService> logger,
        IDistributedCache cache,
        IConfiguration configuration)
    {
        _logger = logger;
        _cache = cache;
        _defaultOptions = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(
                configuration.GetValue<int>("Cache:DefaultExpirationMinutes", 30)),
            SlidingExpiration = TimeSpan.FromMinutes(5)
        };
    }

    public async Task<T> GetOrSetAsync<T>(string key, Func<Task<T>> factory)
    {
        var cachedValue = await _cache.GetStringAsync(key);
        if (cachedValue != null)
        {
            return JsonSerializer.Deserialize<T>(cachedValue);
        }

        var value = await factory();
        var serializedValue = JsonSerializer.Serialize(value);
        await _cache.SetStringAsync(key, serializedValue, _defaultOptions);
        return value;
    }

    public Task RemoveAsync(string key)
    {
        return _cache.RemoveAsync(key);
    }
}
```

### 3. Cache-Aside Pattern
```csharp
public class CacheAsideService
{
    private readonly ILogger<CacheAsideService> _logger;
    private readonly ICacheService _cache;
    private readonly IRepository _repository;

    public CacheAsideService(
        ILogger<CacheAsideService> logger,
        ICacheService cache,
        IRepository repository)
    {
        _logger = logger;
        _cache = cache;
        _repository = repository;
    }

    public async Task<T> GetDataAsync<T>(string key)
    {
        try
        {
            // Önbellekten kontrol et
            var cachedValue = await _cache.GetAsync<T>(key);
            if (cachedValue != null)
            {
                return cachedValue;
            }

            // Veritabanından al
            var value = await _repository.GetAsync<T>(key);
            
            // Önbelleğe kaydet
            await _cache.SetAsync(key, value);
            
            return value;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Veri alma hatası");
            throw;
        }
    }

    public async Task UpdateDataAsync<T>(string key, T value)
    {
        try
        {
            // Veritabanını güncelle
            await _repository.UpdateAsync(key, value);
            
            // Önbelleği güncelle
            await _cache.SetAsync(key, value);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Veri güncelleme hatası");
            throw;
        }
    }
}
```

### 4. Write-Through Caching
```csharp
public class WriteThroughCacheService
{
    private readonly ILogger<WriteThroughCacheService> _logger;
    private readonly ICacheService _cache;
    private readonly IRepository _repository;

    public WriteThroughCacheService(
        ILogger<WriteThroughCacheService> logger,
        ICacheService cache,
        IRepository repository)
    {
        _logger = logger;
        _cache = cache;
        _repository = repository;
    }

    public async Task<T> GetDataAsync<T>(string key)
    {
        try
        {
            // Önbellekten kontrol et
            var cachedValue = await _cache.GetAsync<T>(key);
            if (cachedValue != null)
            {
                return cachedValue;
            }

            // Veritabanından al ve önbelleğe kaydet
            var value = await _repository.GetAsync<T>(key);
            await _cache.SetAsync(key, value);
            
            return value;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Veri alma hatası");
            throw;
        }
    }

    public async Task UpdateDataAsync<T>(string key, T value)
    {
        try
        {
            // Önce önbelleği güncelle
            await _cache.SetAsync(key, value);
            
            // Sonra veritabanını güncelle
            await _repository.UpdateAsync(key, value);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Veri güncelleme hatası");
            throw;
        }
    }
}
```

## Best Practices

### 1. Önbellekleme Stratejileri
- Veri erişim desenlerini analiz et
- Uygun önbellek boyutunu belirle
- Önbellek süresini optimize et
- Önbellek temizleme stratejisi belirle
- Önbellek tutarlılığını sağla

### 2. Performans Optimizasyonu
- Önbellek hit oranını izle
- Önbellek boyutunu monitörle
- Önbellek süresini ayarla
- Önbellek sıkıştırması kullan
- Önbellek dağıtımını optimize et

### 3. Monitoring ve Logging
- Önbellek istatistiklerini topla
- Önbellek hatalarını izle
- Önbellek performansını takip et
- Önbellek kullanımını raporla
- Otomatik alarmlar kur

## Sık Sorulan Sorular

### 1. Hangi önbellekleme stratejisi ne zaman kullanılmalıdır?
- In-Memory: Tek sunucu, hızlı erişim
- Distributed: Çoklu sunucu, ölçeklenebilirlik
- Cache-Aside: Basit, esnek
- Write-Through: Tutarlılık önemli
- Write-Behind: Performans önemli

### 2. Önbelleklemede veri tutarlılığı nasıl sağlanır?
- TTL (Time-To-Live) kullan
- Versiyonlama uygula
- Etiketleme yap
- Temizleme stratejisi belirle
- Senkronizasyon mekanizması kur

### 3. Önbelleklemede karşılaşılan zorluklar nelerdir?
- Veri tutarlılığı
- Önbellek ıskalama
- Önbellek kirlenmesi
- Bellek yönetimi
- Ölçeklendirme

## Kaynaklar
- [Microsoft Caching Patterns](https://docs.microsoft.com/tr-tr/azure/architecture/patterns/category/performance)
- [Redis Caching](https://redis.io/topics/caching)
- [Caching Best Practices](https://docs.microsoft.com/tr-tr/azure/architecture/best-practices/caching)
- [Distributed Caching](https://docs.microsoft.com/tr-tr/aspnet/core/performance/caching/distributed) 