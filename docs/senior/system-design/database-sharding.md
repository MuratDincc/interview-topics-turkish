# Veritabanı Sharding

## Genel Bakış
Veritabanı sharding, büyük veritabanlarını daha küçük, yönetilebilir parçalara (shard) bölerek ölçeklenebilirliği artıran bir tekniktir. Her shard, veritabanının bir alt kümesini içerir ve bağımsız olarak çalışabilir.

## Temel Kavramlar

### 1. Hash-Based Sharding
```csharp
public class HashBasedShardingStrategy : IShardingStrategy
{
    private readonly ILogger<HashBasedShardingStrategy> _logger;
    private readonly IReadOnlyList<IDatabase> _shards;

    public HashBasedShardingStrategy(
        ILogger<HashBasedShardingStrategy> logger,
        IReadOnlyList<IDatabase> shards)
    {
        _logger = logger;
        _shards = shards;
    }

    public IDatabase GetShard(string key)
    {
        var hash = GetHash(key);
        var shardIndex = hash % _shards.Count;
        return _shards[shardIndex];
    }

    private int GetHash(string key)
    {
        return Math.Abs(key.GetHashCode());
    }
}

public class ShardedDatabaseService
{
    private readonly ILogger<ShardedDatabaseService> _logger;
    private readonly IShardingStrategy _shardingStrategy;

    public ShardedDatabaseService(
        ILogger<ShardedDatabaseService> logger,
        IShardingStrategy shardingStrategy)
    {
        _logger = logger;
        _shardingStrategy = shardingStrategy;
    }

    public async Task SaveDataAsync(Data data)
    {
        var shard = _shardingStrategy.GetShard(data.Id);
        await shard.SaveAsync(data);
    }

    public async Task<Data> GetDataAsync(string id)
    {
        var shard = _shardingStrategy.GetShard(id);
        return await shard.GetAsync(id);
    }
}
```

### 2. Range-Based Sharding
```csharp
public class RangeBasedShardingStrategy : IShardingStrategy
{
    private readonly ILogger<RangeBasedShardingStrategy> _logger;
    private readonly IReadOnlyList<ShardRange> _shardRanges;

    public RangeBasedShardingStrategy(
        ILogger<RangeBasedShardingStrategy> logger,
        IReadOnlyList<ShardRange> shardRanges)
    {
        _logger = logger;
        _shardRanges = shardRanges.OrderBy(s => s.StartValue).ToList();
    }

    public IDatabase GetShard(string key)
    {
        var value = GetNumericValue(key);
        var shard = _shardRanges.FirstOrDefault(s => 
            value >= s.StartValue && value < s.EndValue);

        if (shard == null)
        {
            throw new ShardNotFoundException(key);
        }

        return shard.Database;
    }

    private long GetNumericValue(string key)
    {
        return long.Parse(key);
    }
}

public class ShardRange
{
    public long StartValue { get; }
    public long EndValue { get; }
    public IDatabase Database { get; }

    public ShardRange(long startValue, long endValue, IDatabase database)
    {
        StartValue = startValue;
        EndValue = endValue;
        Database = database;
    }
}
```

### 3. Directory-Based Sharding
```csharp
public class DirectoryBasedShardingStrategy : IShardingStrategy
{
    private readonly ILogger<DirectoryBasedShardingStrategy> _logger;
    private readonly IDictionary<string, IDatabase> _shardMap;
    private readonly IDatabase _defaultShard;

    public DirectoryBasedShardingStrategy(
        ILogger<DirectoryBasedShardingStrategy> logger,
        IDictionary<string, IDatabase> shardMap,
        IDatabase defaultShard)
    {
        _logger = logger;
        _shardMap = shardMap;
        _defaultShard = defaultShard;
    }

    public IDatabase GetShard(string key)
    {
        if (_shardMap.TryGetValue(key, out var shard))
        {
            return shard;
        }

        _logger.LogWarning($"Shard bulunamadı, varsayılan shard kullanılıyor: {key}");
        return _defaultShard;
    }
}

public class ShardDirectoryService
{
    private readonly ILogger<ShardDirectoryService> _logger;
    private readonly IShardingStrategy _shardingStrategy;

    public ShardDirectoryService(
        ILogger<ShardDirectoryService> logger,
        IShardingStrategy shardingStrategy)
    {
        _logger = logger;
        _shardingStrategy = shardingStrategy;
    }

    public async Task UpdateShardMappingAsync(string key, IDatabase newShard)
    {
        // Shard mapping güncelleme işlemi
        await Task.Delay(100); // Simülasyon
        _logger.LogInformation($"Shard mapping güncellendi: {key}");
    }

    public async Task<IDatabase> GetShardAsync(string key)
    {
        return _shardingStrategy.GetShard(key);
    }
}
```

### 4. Shard Rebalancing
```csharp
public class ShardRebalancingService
{
    private readonly ILogger<ShardRebalancingService> _logger;
    private readonly IShardingStrategy _shardingStrategy;
    private readonly IReadOnlyList<IDatabase> _shards;

    public ShardRebalancingService(
        ILogger<ShardRebalancingService> logger,
        IShardingStrategy shardingStrategy,
        IReadOnlyList<IDatabase> shards)
    {
        _logger = logger;
        _shardingStrategy = shardingStrategy;
        _shards = shards;
    }

    public async Task RebalanceShardsAsync()
    {
        foreach (var shard in _shards)
        {
            var data = await shard.GetAllDataAsync();
            foreach (var item in data)
            {
                var targetShard = _shardingStrategy.GetShard(item.Id);
                if (targetShard != shard)
                {
                    await targetShard.SaveAsync(item);
                    await shard.DeleteAsync(item.Id);
                }
            }
        }
    }

    public async Task AddNewShardAsync(IDatabase newShard)
    {
        // Yeni shard ekleme ve rebalancing işlemi
        await Task.Delay(100); // Simülasyon
        _logger.LogInformation("Yeni shard eklendi ve rebalancing tamamlandı");
    }
}
```

## Best Practices

### 1. Sharding Stratejileri
- Veri dağılımını analiz et
- Shard boyutunu optimize et
- Shard sayısını belirle
- Shard key seçimini yap
- Shard mapping stratejisini belirle

### 2. Performans Optimizasyonu
- Shard lokasyonunu optimize et
- Çapraz shard sorgularını minimize et
- Shard yükünü dengele
- Önbellekleme stratejisi uygula
- İndeksleme stratejisini belirle

### 3. Monitoring ve Logging
- Shard performansını izle
- Shard boyutunu takip et
- Shard yükünü monitörle
- Hata oranlarını izle
- Otomatik alarmlar kur

## Sık Sorulan Sorular

### 1. Hangi sharding stratejisi ne zaman kullanılmalıdır?
- Hash-Based: Eşit dağılım gerektiğinde
- Range-Based: Sıralı veri erişimi olduğunda
- Directory-Based: Esnek shard mapping gerektiğinde
- Composite: Karmaşık gereksinimler olduğunda

### 2. Sharding'de veri tutarlılığı nasıl sağlanır?
- Transaction yönetimi
- Veri replikasyonu
- Tutarlılık kontrolleri
- Senkronizasyon mekanizmaları
- Hata toleransı

### 3. Sharding'de karşılaşılan zorluklar nelerdir?
- Çapraz shard sorguları
- Veri tutarlılığı
- Shard rebalancing
- Yedekleme ve kurtarma
- Monitoring ve yönetim

## Kaynaklar
- [Microsoft Database Sharding](https://docs.microsoft.com/tr-tr/azure/architecture/patterns/sharding)
- [Database Sharding Strategies](https://www.mongodb.com/basics/sharding)
- [Sharding Best Practices](https://docs.microsoft.com/tr-tr/azure/architecture/best-practices/data-partitioning)
- [Sharding Patterns](https://docs.microsoft.com/tr-tr/azure/architecture/patterns/category/data-management) 