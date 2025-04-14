# Distributed Locking

## Genel Bakış

Distributed locking, dağıtık sistemlerde kaynaklara erişimi senkronize etmek için kullanılan bir mekanizmadır. Bu bölümde, distributed locking'in temel kavramlarını ve C# implementasyonlarını inceleyeceğiz.

## Temel Distributed Locking İşlemleri

### 1. Redis ile Distributed Lock

```csharp
public class RedisDistributedLock
{
    private readonly IDatabase _redis;
    private readonly string _lockKey;
    private readonly string _lockValue;
    private readonly TimeSpan _expiry;

    public RedisDistributedLock(IDatabase redis, string lockKey, TimeSpan expiry)
    {
        _redis = redis;
        _lockKey = lockKey;
        _lockValue = Guid.NewGuid().ToString();
        _expiry = expiry;
    }

    public async Task<bool> AcquireLockAsync()
    {
        return await _redis.StringSetAsync(_lockKey, _lockValue, _expiry, When.NotExists);
    }

    public async Task ReleaseLockAsync()
    {
        var script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        await _redis.ScriptEvaluateAsync(script, new RedisKey[] { _lockKey }, new RedisValue[] { _lockValue });
    }
}
```

### 2. ZooKeeper ile Distributed Lock

```csharp
public class ZooKeeperDistributedLock
{
    private readonly ZooKeeper _zooKeeper;
    private readonly string _lockPath;
    private string _currentNode;

    public ZooKeeperDistributedLock(ZooKeeper zooKeeper, string lockPath)
    {
        _zooKeeper = zooKeeper;
        _lockPath = lockPath;
    }

    public async Task AcquireLockAsync()
    {
        _currentNode = await _zooKeeper.CreateAsync(
            $"{_lockPath}/lock-",
            Array.Empty<byte>(),
            ZooDefs.Ids.OPEN_ACL_UNSAFE,
            CreateMode.EPHEMERAL_SEQUENTIAL);

        var children = await _zooKeeper.GetChildrenAsync(_lockPath, false);
        var sortedNodes = children.OrderBy(x => x).ToList();
        var currentNodeIndex = sortedNodes.IndexOf(_currentNode.Split('/').Last());

        if (currentNodeIndex > 0)
        {
            var previousNode = $"{_lockPath}/{sortedNodes[currentNodeIndex - 1]}";
            var watcher = new LockWatcher();
            await _zooKeeper.ExistsAsync(previousNode, watcher);
            await watcher.WaitAsync();
        }
    }

    public async Task ReleaseLockAsync()
    {
        await _zooKeeper.DeleteAsync(_currentNode);
    }
}
```

## İleri Distributed Locking Algoritmaları

### 1. Lease-based Locking

```csharp
public class LeaseBasedLock
{
    private readonly IDistributedCache _cache;
    private readonly string _lockKey;
    private readonly string _lockValue;
    private readonly TimeSpan _leaseTime;
    private Timer _renewalTimer;

    public LeaseBasedLock(IDistributedCache cache, string lockKey, TimeSpan leaseTime)
    {
        _cache = cache;
        _lockKey = lockKey;
        _lockValue = Guid.NewGuid().ToString();
        _leaseTime = leaseTime;
    }

    public async Task<bool> AcquireLockAsync()
    {
        var options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = _leaseTime
        };

        if (await _cache.GetStringAsync(_lockKey) == null)
        {
            await _cache.SetStringAsync(_lockKey, _lockValue, options);
            StartLeaseRenewal();
            return true;
        }

        return false;
    }

    private void StartLeaseRenewal()
    {
        _renewalTimer = new Timer(async _ =>
        {
            await _cache.SetStringAsync(_lockKey, _lockValue, new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = _leaseTime
            });
        }, null, _leaseTime / 2, _leaseTime / 2);
    }

    public async Task ReleaseLockAsync()
    {
        _renewalTimer?.Dispose();
        await _cache.RemoveAsync(_lockKey);
    }
}
```

### 2. Redlock Algorithm

```csharp
public class Redlock
{
    private readonly List<IDatabase> _redisInstances;
    private readonly int _quorum;
    private readonly TimeSpan _lockTimeToLive;

    public Redlock(List<IDatabase> redisInstances, TimeSpan lockTimeToLive)
    {
        _redisInstances = redisInstances;
        _quorum = redisInstances.Count / 2 + 1;
        _lockTimeToLive = lockTimeToLive;
    }

    public async Task<bool> LockAsync(string resource, string value)
    {
        var startTime = DateTime.UtcNow;
        var acquiredLocks = 0;

        foreach (var redis in _redisInstances)
        {
            if (await redis.StringSetAsync(resource, value, _lockTimeToLive, When.NotExists))
            {
                acquiredLocks++;
            }
        }

        var elapsedTime = DateTime.UtcNow - startTime;
        if (acquiredLocks >= _quorum && elapsedTime < _lockTimeToLive)
        {
            return true;
        }

        await UnlockAsync(resource, value);
        return false;
    }

    public async Task UnlockAsync(string resource, string value)
    {
        foreach (var redis in _redisInstances)
        {
            var script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
            await redis.ScriptEvaluateAsync(script, new RedisKey[] { resource }, new RedisValue[] { value });
        }
    }
}
```

## Performans Analizi

| Algoritma | En İyi Durum | Ortalama Durum | En Kötü Durum | Bellek Kullanımı |
|-----------|-------------|----------------|---------------|------------------|
| Redis Lock | O(1) | O(1) | O(n) | O(1) |
| ZooKeeper Lock | O(log n) | O(log n) | O(n) | O(1) |
| Lease-based Lock | O(1) | O(1) | O(n) | O(1) |
| Redlock | O(n) | O(n) | O(n) | O(1) |

## Best Practices

1. Deadlock'ları önle
2. Lock süresini optimize et
3. Hata durumlarını yönet
4. Performansı izle
5. Yedeklilik sağla

## Örnek Uygulamalar

1. Dağıtık işlemler
2. Kaynak yönetimi
3. Veri tutarlılığı
4. Ölçeklendirme
5. Yüksek erişilebilirlik

## Kaynaklar

- [Distributed Locks with Redis](https://redis.io/topics/distlock)
- [ZooKeeper Recipes](https://zookeeper.apache.org/doc/current/recipes.html)
- [Distributed Systems](https://www.tutorialspoint.com/distributed_system/index.htm) 