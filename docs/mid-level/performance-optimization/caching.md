# Caching

## Genel Bakış
Caching (önbellekleme), sık kullanılan verilerin hızlı erişilebilir bir yerde saklanması ve gerektiğinde buradan alınması işlemidir. Bu sayede veritabanı sorguları, hesaplamalar veya dış servis çağrıları gibi maliyetli işlemler tekrarlanmaz ve uygulama performansı artırılır.

## Temel Kavramlar

### 1. Cache Türleri
- **In-Memory Cache**: Verilerin uygulama belleğinde saklanması
- **Distributed Cache**: Verilerin birden fazla sunucu arasında paylaşılması
- **Response Cache**: HTTP yanıtlarının önbelleğe alınması
- **Output Cache**: Sayfa çıktılarının önbelleğe alınması

### 2. Cache Stratejileri
```csharp
// In-Memory Cache örneği
public class ProductService
{
    private readonly IMemoryCache _cache;
    private readonly IProductRepository _repository;
    private readonly ILogger<ProductService> _logger;

    public ProductService(
        IMemoryCache cache,
        IProductRepository repository,
        ILogger<ProductService> logger)
    {
        _cache = cache;
        _repository = repository;
        _logger = logger;
    }

    public async Task<Product> GetProductAsync(int id)
    {
        var cacheKey = $"product_{id}";
        
        if (_cache.TryGetValue(cacheKey, out Product cachedProduct))
        {
            _logger.LogInformation("Product {Id} retrieved from cache", id);
            return cachedProduct;
        }

        var product = await _repository.GetByIdAsync(id);
        
        var cacheOptions = new MemoryCacheEntryOptions()
            .SetSlidingExpiration(TimeSpan.FromMinutes(30))
            .SetAbsoluteExpiration(TimeSpan.FromHours(1))
            .SetPriority(CacheItemPriority.Normal)
            .RegisterPostEvictionCallback((key, value, reason, state) =>
            {
                _logger.LogInformation("Cache entry {Key} was evicted due to {Reason}", key, reason);
            });

        _cache.Set(cacheKey, product, cacheOptions);
        _logger.LogInformation("Product {Id} added to cache", id);
        
        return product;
    }
}

// Distributed Cache örneği
public class DistributedCacheService
{
    private readonly IDistributedCache _cache;
    private readonly ILogger<DistributedCacheService> _logger;

    public DistributedCacheService(
        IDistributedCache cache,
        ILogger<DistributedCacheService> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public async Task<T> GetOrSetAsync<T>(
        string key,
        Func<Task<T>> factory,
        TimeSpan? expiration = null)
    {
        var cachedValue = await _cache.GetStringAsync(key);
        if (cachedValue != null)
        {
            _logger.LogInformation("Value for key {Key} retrieved from cache", key);
            return JsonSerializer.Deserialize<T>(cachedValue);
        }

        var value = await factory();
        var serializedValue = JsonSerializer.Serialize(value);
        
        var options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = expiration ?? TimeSpan.FromMinutes(30)
        };

        await _cache.SetStringAsync(key, serializedValue, options);
        _logger.LogInformation("Value for key {Key} added to cache", key);
        
        return value;
    }
}
```

### 3. Cache Invalidation
```csharp
public class CacheInvalidator
{
    private readonly IMemoryCache _cache;
    private readonly ILogger<CacheInvalidator> _logger;

    public CacheInvalidator(
        IMemoryCache cache,
        ILogger<CacheInvalidator> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public void InvalidateProduct(int productId)
    {
        var cacheKey = $"product_{productId}";
        _cache.Remove(cacheKey);
        _logger.LogInformation("Cache entry {Key} invalidated", cacheKey);
    }

    public void InvalidateByPattern(string pattern)
    {
        var keys = _cache.GetKeys<string>()
            .Where(k => k.StartsWith(pattern));
            
        foreach (var key in keys)
        {
            _cache.Remove(key);
            _logger.LogInformation("Cache entry {Key} invalidated by pattern", key);
        }
    }
}
```

## Best Practices

### 1. Cache Tasarımı
- Uygun cache stratejisini seçin
- Cache key'lerini anlamlı tasarlayın
- Cache sürelerini optimize edin
- Cache boyutunu yönetin
- Cache invalidation stratejisi belirleyin

### 2. Performans Optimizasyonu
- Cache hit oranını artırın
- Cache miss maliyetini azaltın
- Cache coherency'yi sağlayın
- Cache partitioning kullanın
- Cache warming uygulayın

### 3. Güvenlik
- Hassas verileri cache'lemeyin
- Cache poisoning'a karşı korunun
- Cache key'lerini güvenli oluşturun
- Cache erişimini kontrol edin
- Cache verilerini şifreleyin

## Sık Sorulan Sorular

### 1. Ne zaman cache kullanılmalıdır?
- Sık erişilen veriler için
- Hesaplama maliyeti yüksek işlemler için
- Dış servis çağrıları için
- Statik içerik için
- Session verileri için

### 2. Hangi cache çözümleri kullanılabilir?
- In-Memory: IMemoryCache, MemoryCache
- Distributed: Redis, NCache, Memcached
- Response: ResponseCache, OutputCache
- Browser: LocalStorage, SessionStorage

### 3. Cache kullanırken nelere dikkat edilmelidir?
- Cache invalidation stratejisi
- Cache boyutu yönetimi
- Cache coherency
- Cache security
- Cache monitoring

## Kaynaklar
- [Microsoft Caching Best Practices](https://docs.microsoft.com/tr-tr/aspnet/core/performance/caching/memory)
- [Distributed Caching in ASP.NET Core](https://docs.microsoft.com/tr-tr/aspnet/core/performance/caching/distributed)
- [Response Caching in ASP.NET Core](https://docs.microsoft.com/tr-tr/aspnet/core/performance/caching/response)
- [Redis Documentation](https://redis.io/documentation) 