# Yüksek Erişilebilirlik (High Availability)

## Genel Bakış
Yüksek Erişilebilirlik, bir sistemin kesintisiz çalışma ve hizmet verme yeteneğidir. Sistemlerin %99.9 ve üzeri uptime oranlarına sahip olması, hata toleransı ve felaket kurtarma stratejileri yüksek erişilebilirliği sağlamak için kullanılan temel yaklaşımlardır.

## Temel Kavramlar

### 1. Hata Toleransı (Fault Tolerance)
```csharp
public class FaultTolerantService
{
    private readonly ILogger<FaultTolerantService> _logger;
    private readonly IRetryPolicy _retryPolicy;
    private readonly ICircuitBreaker _circuitBreaker;

    public FaultTolerantService(
        ILogger<FaultTolerantService> logger,
        IRetryPolicy retryPolicy,
        ICircuitBreaker circuitBreaker)
    {
        _logger = logger;
        _retryPolicy = retryPolicy;
        _circuitBreaker = circuitBreaker;
    }

    public async Task<Result> ProcessRequestAsync(Request request)
    {
        try
        {
            return await _circuitBreaker.ExecuteAsync(async () =>
            {
                return await _retryPolicy.ExecuteAsync(async () =>
                {
                    // Kritik işlem
                    return await ProcessCriticalOperationAsync(request);
                });
            });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "İşlem başarısız oldu");
            return Result.Failure("İşlem başarısız oldu");
        }
    }

    private async Task<Result> ProcessCriticalOperationAsync(Request request)
    {
        // Kritik işlem mantığı
        await Task.Delay(100);
        return Result.Success();
    }
}

public interface ICircuitBreaker
{
    Task<T> ExecuteAsync<T>(Func<Task<T>> action);
}

public class CircuitBreaker : ICircuitBreaker
{
    private readonly ILogger<CircuitBreaker> _logger;
    private readonly int _failureThreshold;
    private readonly TimeSpan _resetTimeout;
    private int _failureCount;
    private DateTime _lastFailureTime;

    public CircuitBreaker(
        ILogger<CircuitBreaker> logger,
        int failureThreshold = 3,
        TimeSpan resetTimeout = default)
    {
        _logger = logger;
        _failureThreshold = failureThreshold;
        _resetTimeout = resetTimeout == default ? TimeSpan.FromMinutes(1) : resetTimeout;
    }

    public async Task<T> ExecuteAsync<T>(Func<Task<T>> action)
    {
        if (IsCircuitOpen())
        {
            _logger.LogWarning("Circuit breaker açık, işlem reddedildi");
            throw new CircuitBreakerOpenException();
        }

        try
        {
            var result = await action();
            Reset();
            return result;
        }
        catch (Exception ex)
        {
            _failureCount++;
            _lastFailureTime = DateTime.UtcNow;
            throw;
        }
    }

    private bool IsCircuitOpen()
    {
        if (_failureCount >= _failureThreshold)
        {
            if (DateTime.UtcNow - _lastFailureTime > _resetTimeout)
            {
                Reset();
                return false;
            }
            return true;
        }
        return false;
    }

    private void Reset()
    {
        _failureCount = 0;
        _lastFailureTime = DateTime.MinValue;
    }
}
```

### 2. Veri Replikasyonu (Data Replication)
```csharp
public class DataReplicationService
{
    private readonly ILogger<DataReplicationService> _logger;
    private readonly IReadOnlyList<IDatabase> _replicas;
    private readonly IConsistencyChecker _consistencyChecker;

    public DataReplicationService(
        ILogger<DataReplicationService> logger,
        IReadOnlyList<IDatabase> replicas,
        IConsistencyChecker consistencyChecker)
    {
        _logger = logger;
        _replicas = replicas;
        _consistencyChecker = consistencyChecker;
    }

    public async Task SaveDataAsync(Data data)
    {
        // Primary veritabanına kaydet
        await _replicas[0].SaveAsync(data);

        // Replikalara asenkron olarak yayınla
        var replicationTasks = _replicas.Skip(1).Select(replica =>
            Task.Run(async () =>
            {
                try
                {
                    await replica.SaveAsync(data);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Replikasyon hatası");
                }
            }));

        await Task.WhenAll(replicationTasks);

        // Tutarlılık kontrolü
        await _consistencyChecker.VerifyConsistencyAsync(data.Id);
    }

    public async Task<Data> GetDataAsync(string id)
    {
        // En yakın replikadan oku
        foreach (var replica in _replicas)
        {
            try
            {
                return await replica.GetAsync(id);
            }
            catch (Exception ex)
            {
                _logger.LogWarning(ex, "Replika okuma hatası");
            }
        }

        throw new DataNotFoundException(id);
    }
}
```

### 3. Yük Dengeleme (Load Balancing)
```csharp
public class LoadBalancedService
{
    private readonly ILogger<LoadBalancedService> _logger;
    private readonly IHealthChecker _healthChecker;
    private readonly ILoadBalancer _loadBalancer;

    public LoadBalancedService(
        ILogger<LoadBalancedService> logger,
        IHealthChecker healthChecker,
        ILoadBalancer loadBalancer)
    {
        _logger = logger;
        _healthChecker = healthChecker;
        _loadBalancer = loadBalancer;
    }

    public async Task<Response> HandleRequestAsync(Request request)
    {
        // Sağlıklı sunucuları kontrol et
        var healthyServers = await _healthChecker.GetHealthyServersAsync();
        
        if (!healthyServers.Any())
        {
            throw new NoHealthyServersException();
        }

        // Yük dengeleyici üzerinden isteği yönlendir
        var server = await _loadBalancer.GetNextServerAsync(healthyServers);
        return await server.ProcessRequestAsync(request);
    }
}

public interface IHealthChecker
{
    Task<IReadOnlyList<Server>> GetHealthyServersAsync();
}

public class HealthChecker : IHealthChecker
{
    private readonly ILogger<HealthChecker> _logger;
    private readonly IReadOnlyList<Server> _servers;
    private readonly TimeSpan _healthCheckInterval;

    public HealthChecker(
        ILogger<HealthChecker> logger,
        IReadOnlyList<Server> servers,
        TimeSpan healthCheckInterval)
    {
        _logger = logger;
        _servers = servers;
        _healthCheckInterval = healthCheckInterval;
    }

    public async Task<IReadOnlyList<Server>> GetHealthyServersAsync()
    {
        var healthCheckTasks = _servers.Select(CheckServerHealthAsync);
        var results = await Task.WhenAll(healthCheckTasks);
        return results.Where(r => r.IsHealthy).Select(r => r.Server).ToList();
    }

    private async Task<(Server Server, bool IsHealthy)> CheckServerHealthAsync(Server server)
    {
        try
        {
            var isHealthy = await server.CheckHealthAsync();
            return (server, isHealthy);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Sağlık kontrolü hatası");
            return (server, false);
        }
    }
}
```

## Best Practices

### 1. Hata Yönetimi
- Circuit breaker pattern uygula
- Retry mekanizmaları kullan
- Fallback stratejileri belirle
- Hata izleme ve raporlama yap
- Otomatik kurtarma mekanizmaları kur

### 2. Veri Yönetimi
- Veri replikasyonu uygula
- Tutarlılık kontrolleri yap
- Veri yedekleme stratejileri belirle
- Veri senkronizasyonu sağla
- Veri bütünlüğünü koru

### 3. Sistem Yönetimi
- Yük dengeleme stratejileri uygula
- Sağlık kontrolleri yap
- Otomatik ölçeklendirme kullan
- Monitoring ve alerting kur
- Felaket kurtarma planları hazırla

## Sık Sorulan Sorular

### 1. Yüksek erişilebilirlik için hangi stratejiler kullanılmalıdır?
- Hata toleransı
- Veri replikasyonu
- Yük dengeleme
- Otomatik kurtarma
- Felaket kurtarma

### 2. Veri tutarlılığı nasıl sağlanır?
- Strong consistency
- Eventual consistency
- Read-your-writes consistency
- Monotonic reads
- Monotonic writes

### 3. Felaket kurtarma planı nasıl hazırlanır?
- Risk analizi yap
- Kurtarma stratejileri belirle
- Test senaryoları hazırla
- Dokümantasyon oluştur
- Düzenli testler yap

## Kaynaklar
- [Microsoft High Availability](https://docs.microsoft.com/tr-tr/azure/architecture/framework/resiliency/overview)
- [AWS High Availability](https://aws.amazon.com/architecture/well-architected/)
- [Google Cloud High Availability](https://cloud.google.com/architecture/framework/resiliency)
- [High Availability Patterns](https://docs.microsoft.com/tr-tr/azure/architecture/patterns/category/resiliency) 