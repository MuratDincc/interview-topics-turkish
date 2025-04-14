# Redis Kullanımı

## Giriş

Redis (Remote Dictionary Server), açık kaynaklı, in-memory veri yapısı deposu olarak kullanılan bir NoSQL veritabanıdır. .NET uygulamalarında önbellekleme, oturum yönetimi, mesaj kuyruğu ve gerçek zamanlı analitik gibi çeşitli senaryolarda kullanılır.

## Redis'in Önemi

1. **Performans**
   - Yüksek hızlı veri erişimi
   - Düşük latency
   - Yüksek throughput

2. **Ölçeklenebilirlik**
   - Yatay ölçeklenebilirlik
   - Cluster desteği
   - Replikasyon

3. **Esneklik**
   - Çoklu veri yapıları
   - Zengin komut seti
   - Genişletilebilir mimari

## Redis Veri Tipleri

1. **String**
   - Metin ve sayısal değerler
   - Binary-safe
   - Atomic operasyonlar

2. **Hash**
   - Alan-değer çiftleri
   - Nesne temsili
   - Kısmi güncelleme

3. **List**
   - Sıralı koleksiyonlar
   - Queue/Stack yapıları
   - Blocking operasyonlar

4. **Set**
   - Benzersiz elemanlar
   - Küme operasyonları
   - Random eleman seçimi

5. **Sorted Set**
   - Sıralı benzersiz elemanlar
   - Skor bazlı sıralama
   - Range sorguları

## Redis Kullanımı (.NET)

1. **Temel Kurulum**
```csharp
// NuGet paketi: StackExchange.Redis
public class RedisService
{
    private readonly IConnectionMultiplexer _redis;
    private readonly ILogger<RedisService> _logger;

    public RedisService(IConnectionMultiplexer redis, ILogger<RedisService> logger)
    {
        _redis = redis;
        _logger = logger;
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan? expiry = null)
    {
        var db = _redis.GetDatabase();
        var serializedValue = JsonSerializer.Serialize(value);
        await db.StringSetAsync(key, serializedValue, expiry);
    }

    public async Task<T> GetAsync<T>(string key)
    {
        var db = _redis.GetDatabase();
        var value = await db.StringGetAsync(key);
        return value.HasValue ? JsonSerializer.Deserialize<T>(value) : default;
    }
}
```

2. **Hash Kullanımı**
```csharp
public class RedisHashService
{
    private readonly IConnectionMultiplexer _redis;
    private readonly ILogger<RedisHashService> _logger;

    public RedisHashService(IConnectionMultiplexer redis, ILogger<RedisHashService> logger)
    {
        _redis = redis;
        _logger = logger;
    }

    public async Task SetHashAsync<T>(string key, string field, T value)
    {
        var db = _redis.GetDatabase();
        var serializedValue = JsonSerializer.Serialize(value);
        await db.HashSetAsync(key, field, serializedValue);
    }

    public async Task<T> GetHashAsync<T>(string key, string field)
    {
        var db = _redis.GetDatabase();
        var value = await db.HashGetAsync(key, field);
        return value.HasValue ? JsonSerializer.Deserialize<T>(value) : default;
    }
}
```

3. **List Kullanımı**
```csharp
public class RedisListService
{
    private readonly IConnectionMultiplexer _redis;
    private readonly ILogger<RedisListService> _logger;

    public RedisListService(IConnectionMultiplexer redis, ILogger<RedisListService> logger)
    {
        _redis = redis;
        _logger = logger;
    }

    public async Task PushAsync<T>(string key, T value)
    {
        var db = _redis.GetDatabase();
        var serializedValue = JsonSerializer.Serialize(value);
        await db.ListRightPushAsync(key, serializedValue);
    }

    public async Task<T> PopAsync<T>(string key)
    {
        var db = _redis.GetDatabase();
        var value = await db.ListLeftPopAsync(key);
        return value.HasValue ? JsonSerializer.Deserialize<T>(value) : default;
    }
}
```

4. **Pub/Sub Kullanımı**
```csharp
public class RedisPubSubService
{
    private readonly IConnectionMultiplexer _redis;
    private readonly ILogger<RedisPubSubService> _logger;

    public RedisPubSubService(IConnectionMultiplexer redis, ILogger<RedisPubSubService> logger)
    {
        _redis = redis;
        _logger = logger;
    }

    public async Task PublishAsync<T>(string channel, T message)
    {
        var subscriber = _redis.GetSubscriber();
        var serializedMessage = JsonSerializer.Serialize(message);
        await subscriber.PublishAsync(channel, serializedMessage);
    }

    public void Subscribe<T>(string channel, Action<T> handler)
    {
        var subscriber = _redis.GetSubscriber();
        subscriber.Subscribe(channel, (redisChannel, message) =>
        {
            var deserializedMessage = JsonSerializer.Deserialize<T>(message);
            handler(deserializedMessage);
        });
    }
}
```

## Redis Best Practices

1. **Bağlantı Yönetimi**
   - Connection pooling
   - Bağlantı multiplexing
   - Timeout yönetimi
   - Retry politikaları

2. **Veri Tasarımı**
   - Uygun veri tipleri
   - Key naming conventions
   - Veri boyutu optimizasyonu
   - TTL kullanımı

3. **Performans**
   - Pipeline kullanımı
   - Batch işlemler
   - Memory optimizasyonu
   - Monitoring

4. **Güvenlik**
   - Authentication
   - SSL/TLS
   - Network izolasyonu
   - Access control

## Redis Monitoring

1. **Redis CLI**
   - INFO komutu
   - MONITOR komutu
   - SLOWLOG komutu
   - MEMORY komutu

2. **RedisInsight**
   - Real-time monitoring
   - Performance metrics
   - Memory analysis
   - Key space analysis

3. **Prometheus + Grafana**
   - Custom metrics
   - Alerting
   - Dashboard
   - Trend analysis

## Mülakat Soruları

### Temel Sorular

1. **Redis nedir ve ne için kullanılır?**
   - **Cevap**: Redis, in-memory veri yapısı deposu olarak kullanılan bir NoSQL veritabanıdır. Önbellekleme, oturum yönetimi, mesaj kuyruğu ve gerçek zamanlı analitik gibi senaryolarda kullanılır.

2. **Redis'in temel veri tipleri nelerdir?**
   - **Cevap**:
     - String
     - Hash
     - List
     - Set
     - Sorted Set

3. **Redis'in avantajları nelerdir?**
   - **Cevap**:
     - Yüksek performans
     - Ölçeklenebilirlik
     - Esneklik
     - Zengin veri tipleri
     - Atomic operasyonlar

4. **Redis'te persistence nasıl sağlanır?**
   - **Cevap**:
     - RDB (Redis Database)
     - AOF (Append Only File)
     - Hybrid yaklaşım

5. **Redis Cluster nedir?**
   - **Cevap**: Redis Cluster, verileri birden fazla Redis node'u arasında dağıtan ve yüksek kullanılabilirlik sağlayan bir yapıdır.

### Teknik Sorular

1. **Redis'te string veri tipi nasıl kullanılır?**
   - **Cevap**:
```csharp
public class RedisStringService
{
    private readonly IConnectionMultiplexer _redis;
    private readonly ILogger<RedisStringService> _logger;

    public RedisStringService(IConnectionMultiplexer redis, ILogger<RedisStringService> logger)
    {
        _redis = redis;
        _logger = logger;
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan? expiry = null)
    {
        var db = _redis.GetDatabase();
        var serializedValue = JsonSerializer.Serialize(value);
        await db.StringSetAsync(key, serializedValue, expiry);
    }

    public async Task<T> GetAsync<T>(string key)
    {
        var db = _redis.GetDatabase();
        var value = await db.StringGetAsync(key);
        return value.HasValue ? JsonSerializer.Deserialize<T>(value) : default;
    }
}
```

2. **Redis'te hash veri tipi nasıl kullanılır?**
   - **Cevap**:
```csharp
public class RedisHashService
{
    private readonly IConnectionMultiplexer _redis;
    private readonly ILogger<RedisHashService> _logger;

    public RedisHashService(IConnectionMultiplexer redis, ILogger<RedisHashService> logger)
    {
        _redis = redis;
        _logger = logger;
    }

    public async Task SetHashAsync<T>(string key, string field, T value)
    {
        var db = _redis.GetDatabase();
        var serializedValue = JsonSerializer.Serialize(value);
        await db.HashSetAsync(key, field, serializedValue);
    }

    public async Task<T> GetHashAsync<T>(string key, string field)
    {
        var db = _redis.GetDatabase();
        var value = await db.HashGetAsync(key, field);
        return value.HasValue ? JsonSerializer.Deserialize<T>(value) : default;
    }
}
```

3. **Redis'te pub/sub pattern nasıl implemente edilir?**
   - **Cevap**:
```csharp
public class RedisPubSubService
{
    private readonly IConnectionMultiplexer _redis;
    private readonly ILogger<RedisPubSubService> _logger;

    public RedisPubSubService(IConnectionMultiplexer redis, ILogger<RedisPubSubService> logger)
    {
        _redis = redis;
        _logger = logger;
    }

    public async Task PublishAsync<T>(string channel, T message)
    {
        var subscriber = _redis.GetSubscriber();
        var serializedMessage = JsonSerializer.Serialize(message);
        await subscriber.PublishAsync(channel, serializedMessage);
    }

    public void Subscribe<T>(string channel, Action<T> handler)
    {
        var subscriber = _redis.GetSubscriber();
        subscriber.Subscribe(channel, (redisChannel, message) =>
        {
            var deserializedMessage = JsonSerializer.Deserialize<T>(message);
            handler(deserializedMessage);
        });
    }
}
```

4. **Redis'te transaction nasıl kullanılır?**
   - **Cevap**:
```csharp
public class RedisTransactionService
{
    private readonly IConnectionMultiplexer _redis;
    private readonly ILogger<RedisTransactionService> _logger;

    public RedisTransactionService(IConnectionMultiplexer redis, ILogger<RedisTransactionService> logger)
    {
        _redis = redis;
        _logger = logger;
    }

    public async Task<bool> ExecuteTransactionAsync(string key1, string key2, string value)
    {
        var db = _redis.GetDatabase();
        var tran = db.CreateTransaction();

        var set1 = tran.StringSetAsync(key1, value);
        var set2 = tran.StringSetAsync(key2, value);

        return await tran.ExecuteAsync();
    }
}
```

5. **Redis'te pipeline nasıl kullanılır?**
   - **Cevap**:
```csharp
public class RedisPipelineService
{
    private readonly IConnectionMultiplexer _redis;
    private readonly ILogger<RedisPipelineService> _logger;

    public RedisPipelineService(IConnectionMultiplexer redis, ILogger<RedisPipelineService> logger)
    {
        _redis = redis;
        _logger = logger;
    }

    public async Task<List<string>> ExecutePipelineAsync(List<string> keys)
    {
        var db = _redis.GetDatabase();
        var batch = db.CreateBatch();
        var tasks = new List<Task<RedisValue>>();

        foreach (var key in keys)
        {
            tasks.Add(batch.StringGetAsync(key));
        }

        batch.Execute();
        var results = await Task.WhenAll(tasks);
        return results.Select(r => r.ToString()).ToList();
    }
}
```

### İleri Seviye Sorular

1. **Redis Cluster'da veri nasıl dağıtılır?**
   - **Cevap**:
     - Hash slot dağıtımı
     - Consistent hashing
     - Replica sharding
     - Failover mekanizmaları

2. **Redis'te memory optimizasyonu nasıl yapılır?**
   - **Cevap**:
     - Memory profiling
     - Key compression
     - Data structure seçimi
     - Memory limits
     - Eviction politikaları

3. **Redis'te güvenlik nasıl sağlanır?**
   - **Cevap**:
     - Authentication
     - SSL/TLS
     - Network izolasyonu
     - Access control
     - Audit logging

4. **Redis'te monitoring ve alerting nasıl yapılır?**
   - **Cevap**:
     - Redis CLI
     - RedisInsight
     - Prometheus + Grafana
     - Custom metrics
     - Alerting rules

5. **Redis'te backup ve recovery nasıl yapılır?**
   - **Cevap**:
     - RDB backup
     - AOF backup
     - Point-in-time recovery
     - Disaster recovery
     - Replication 