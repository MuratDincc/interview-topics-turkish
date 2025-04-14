# Distributed Caching

## Giriş

Distributed Caching, verilerin birden fazla sunucu arasında paylaşılan bir önbellek sisteminde saklanmasıdır. .NET'te Redis, NCache gibi dağıtık önbellek sistemleri kullanılarak uygulanabilir.

## Distributed Caching'in Önemi

1. **Ölçeklenebilirlik**
   - Yatay ölçeklenebilirlik
   - Yük dengeleme
   - Yüksek erişilebilirlik

2. **Performans**
   - Düşük gecikme süreleri
   - Yüksek verimlilik
   - Ölçeklenebilir performans

3. **Güvenilirlik**
   - Fault tolerance
   - Veri tutarlılığı
   - Yüksek erişilebilirlik

## Distributed Caching Türleri

1. **Redis**
   - Açık kaynak
   - Yüksek performans
   - Zengin veri tipleri
   - Persistence desteği

2. **NCache**
   - .NET için optimize
   - Yerel önbellek desteği
   - Otomatik ölçekleme
   - Yüksek güvenilirlik

3. **Memcached**
   - Basit ve hızlı
   - Düşük bellek kullanımı
   - Yüksek ölçeklenebilirlik
   - Kolay entegrasyon

## Distributed Caching Kullanımı

1. **Redis Kullanımı**
```csharp
public class RedisCacheService
{
    private readonly IConnectionMultiplexer _redis;
    private readonly IDatabase _database;

    public RedisCacheService(IConnectionMultiplexer redis)
    {
        _redis = redis;
        _database = redis.GetDatabase();
    }

    public async Task<T> GetOrCreateAsync<T>(string key, Func<Task<T>> factory, TimeSpan? expiry = null)
    {
        var value = await _database.StringGetAsync(key);
        if (!value.IsNull)
        {
            return JsonSerializer.Deserialize<T>(value);
        }

        var newValue = await factory();
        await _database.StringSetAsync(key, JsonSerializer.Serialize(newValue), expiry);
        return newValue;
    }
}
```

2. **NCache Kullanımı**
```csharp
public class NCacheService
{
    private readonly ICache _cache;

    public NCacheService(ICache cache)
    {
        _cache = cache;
    }

    public T GetOrCreate<T>(string key, Func<T> factory, CacheItemPriority priority = CacheItemPriority.Default)
    {
        var item = _cache.Get<T>(key);
        if (item != null)
        {
            return item;
        }

        var newItem = factory();
        var cacheItem = new CacheItem(newItem)
        {
            Priority = priority
        };
        _cache.Insert(key, cacheItem);
        return newItem;
    }
}
```

3. **Memcached Kullanımı**
```csharp
public class MemcachedService
{
    private readonly IMemcachedClient _client;

    public MemcachedService(IMemcachedClient client)
    {
        _client = client;
    }

    public async Task<T> GetOrCreateAsync<T>(string key, Func<Task<T>> factory, TimeSpan? expiry = null)
    {
        var value = await _client.GetAsync<T>(key);
        if (value != null)
        {
            return value;
        }

        var newValue = await factory();
        await _client.SetAsync(key, newValue, expiry ?? TimeSpan.FromMinutes(30));
        return newValue;
    }
}
```

## Distributed Caching Best Practices

1. **Veri Tasarımı**
   - Veri parçalama
   - Veri sıkıştırma
   - Veri serileştirme
   - Veri boyutu optimizasyonu

2. **Ölçekleme Stratejileri**
   - Sharding
   - Replication
   - Partitioning
   - Load balancing

3. **Güvenlik**
   - Şifreleme
   - Erişim kontrolü
   - Ağ güvenliği
   - Veri izolasyonu

4. **Monitoring**
   - Performans metrikleri
   - Sağlık kontrolleri
   - Kapasite planlama
   - Hata izleme

## Mülakat Soruları

### Temel Sorular

1. **Distributed Caching nedir ve neden önemlidir?**
   - **Cevap**: Distributed Caching, verilerin birden fazla sunucu arasında paylaşılan bir önbellek sisteminde saklanmasıdır. Ölçeklenebilirlik, performans ve güvenilirlik sağlar.

2. **Redis ve Memcached arasındaki farklar nelerdir?**
   - **Cevap**:
     - Redis: Zengin veri tipleri, persistence, atomic operasyonlar
     - Memcached: Basit, hızlı, düşük bellek kullanımı

3. **Cache coherency nedir ve nasıl sağlanır?**
   - **Cevap**:
     - Veri tutarlılığı
     - Replication stratejileri
     - Consistency modelleri
     - Conflict resolution

4. **Cache invalidation stratejileri nelerdir?**
   - **Cevap**:
     - Time-based
     - Event-based
     - Manual
     - Dependency-based

5. **Cache partitioning stratejileri nelerdir?**
   - **Cevap**:
     - Hash-based
     - Range-based
     - Directory-based
     - Consistent hashing

### Teknik Sorular

1. **Redis ile distributed caching nasıl uygulanır?**
   - **Cevap**:
```csharp
public class RedisDistributedCache
{
    private readonly IConnectionMultiplexer _redis;
    private readonly IDatabase _database;

    public RedisDistributedCache(IConnectionMultiplexer redis)
    {
        _redis = redis;
        _database = redis.GetDatabase();
    }

    public async Task<T> GetOrCreateAsync<T>(string key, Func<Task<T>> factory, TimeSpan? expiry = null)
    {
        var value = await _database.StringGetAsync(key);
        if (!value.IsNull)
        {
            return JsonSerializer.Deserialize<T>(value);
        }

        var newValue = await factory();
        await _database.StringSetAsync(key, JsonSerializer.Serialize(newValue), expiry);
        return newValue;
    }
}
```

2. **Cache monitoring nasıl yapılır?**
   - **Cevap**:
```csharp
public class CacheMonitor
{
    private readonly IConnectionMultiplexer _redis;
    private readonly ILogger<CacheMonitor> _logger;

    public CacheMonitor(IConnectionMultiplexer redis, ILogger<CacheMonitor> logger)
    {
        _redis = redis;
        _logger = logger;
    }

    public async Task LogCacheStatistics()
    {
        var server = _redis.GetServer(_redis.GetEndPoints().First());
        var info = await server.InfoAsync();
        
        var stats = new
        {
            ConnectedClients = info.FirstOrDefault(x => x.Key == "connected_clients")?.Value,
            UsedMemory = info.FirstOrDefault(x => x.Key == "used_memory")?.Value,
            TotalCommands = info.FirstOrDefault(x => x.Key == "total_commands_processed")?.Value
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
    private readonly IDistributedCache _cache;
    private readonly ILogger<CacheWithFallback> _logger;

    public CacheWithFallback(IDistributedCache cache, ILogger<CacheWithFallback> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public async Task<T> GetWithFallback<T>(string key, Func<Task<T>> factory, TimeSpan cacheDuration)
    {
        try
        {
            var cachedValue = await _cache.GetStringAsync(key);
            if (cachedValue != null)
            {
                return JsonSerializer.Deserialize<T>(cachedValue);
            }

            var value = await factory();
            await _cache.SetStringAsync(key, JsonSerializer.Serialize(value), new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = cacheDuration
            });
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
    private readonly IConnectionMultiplexer _redis;
    private readonly ILogger<CacheSizeManager> _logger;
    private readonly long _maxSize;

    public CacheSizeManager(IConnectionMultiplexer redis, ILogger<CacheSizeManager> logger, long maxSize)
    {
        _redis = redis;
        _logger = logger;
        _maxSize = maxSize;
    }

    public async Task EnsureCacheSize()
    {
        var server = _redis.GetServer(_redis.GetEndPoints().First());
        var info = await server.InfoAsync();
        var usedMemory = long.Parse(info.First(x => x.Key == "used_memory").Value);

        if (usedMemory > _maxSize)
        {
            _logger.LogWarning("Cache size exceeded limit: {UsedMemory}/{MaxSize}", usedMemory, _maxSize);
            await EvictOldestItems();
        }
    }

    private async Task EvictOldestItems()
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
    private readonly IDistributedCache _cache;
    private readonly ILogger<CacheWarmer> _logger;

    public CacheWarmer(IDistributedCache cache, ILogger<CacheWarmer> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public async Task WarmCache<T>(string key, Func<Task<T>> factory, TimeSpan cacheDuration)
    {
        try
        {
            var value = await factory();
            await _cache.SetStringAsync(key, JsonSerializer.Serialize(value), new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = cacheDuration
            });
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

1. **Distributed locking nasıl uygulanır?**
   - **Cevap**:
     - Redis RedLock
     - ZooKeeper
     - Etcd
     - Consul
     - Custom distributed locks

2. **Cache coherency sorunları nasıl çözülür?**
   - **Cevap**:
     - Strong consistency
     - Eventual consistency
     - Read-your-writes
     - Monotonic reads
     - Consistent prefix

3. **Cache partitioning stratejileri nelerdir?**
   - **Cevap**:
     - Hash-based
     - Range-based
     - Directory-based
     - Consistent hashing
     - Dynamic partitioning

4. **Cache monitoring ve alerting nasıl yapılır?**
   - **Cevap**:
     - Prometheus
     - Grafana
     - ELK Stack
     - Custom metrics
     - Alert rules

5. **Cache security nasıl sağlanır?**
   - **Cevap**:
     - TLS/SSL
     - Authentication
     - Authorization
     - Network isolation
     - Audit logging 