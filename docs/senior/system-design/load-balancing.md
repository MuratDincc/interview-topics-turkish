# Yük Dengeleme (Load Balancing)

## Genel Bakış
Yük dengeleme, gelen istekleri birden fazla sunucu arasında dağıtarak sistem kaynaklarının verimli kullanılmasını ve yüksek erişilebilirliği sağlayan bir tekniktir. Yük dengeleyiciler, istekleri farklı stratejilere göre dağıtarak sistem performansını optimize eder.

## Temel Kavramlar

### 1. Round Robin Yük Dengeleme
```csharp
public class RoundRobinLoadBalancer : ILoadBalancer
{
    private readonly ILogger<RoundRobinLoadBalancer> _logger;
    private readonly IReadOnlyList<Server> _servers;
    private int _currentIndex;
    private readonly object _lock = new object();

    public RoundRobinLoadBalancer(
        ILogger<RoundRobinLoadBalancer> logger,
        IReadOnlyList<Server> servers)
    {
        _logger = logger;
        _servers = servers;
        _currentIndex = 0;
    }

    public Task<Server> GetNextServerAsync()
    {
        lock (_lock)
        {
            var server = _servers[_currentIndex];
            _currentIndex = (_currentIndex + 1) % _servers.Count;
            return Task.FromResult(server);
        }
    }
}
```

### 2. Weighted Round Robin Yük Dengeleme
```csharp
public class WeightedRoundRobinLoadBalancer : ILoadBalancer
{
    private readonly ILogger<WeightedRoundRobinLoadBalancer> _logger;
    private readonly IReadOnlyList<WeightedServer> _servers;
    private int _currentIndex;
    private int _currentWeight;
    private readonly object _lock = new object();

    public WeightedRoundRobinLoadBalancer(
        ILogger<WeightedRoundRobinLoadBalancer> logger,
        IReadOnlyList<WeightedServer> servers)
    {
        _logger = logger;
        _servers = servers;
        _currentIndex = -1;
        _currentWeight = 0;
    }

    public Task<Server> GetNextServerAsync()
    {
        lock (_lock)
        {
            while (true)
            {
                _currentIndex = (_currentIndex + 1) % _servers.Count;
                if (_currentIndex == 0)
                {
                    _currentWeight = _currentWeight - 1;
                    if (_currentWeight <= 0)
                    {
                        _currentWeight = _servers.Max(s => s.Weight);
                    }
                }

                if (_servers[_currentIndex].Weight >= _currentWeight)
                {
                    return Task.FromResult(_servers[_currentIndex].Server);
                }
            }
        }
    }
}

public class WeightedServer
{
    public Server Server { get; }
    public int Weight { get; }

    public WeightedServer(Server server, int weight)
    {
        Server = server;
        Weight = weight;
    }
}
```

### 3. Least Connections Yük Dengeleme
```csharp
public class LeastConnectionsLoadBalancer : ILoadBalancer
{
    private readonly ILogger<LeastConnectionsLoadBalancer> _logger;
    private readonly IReadOnlyList<Server> _servers;
    private readonly ConcurrentDictionary<Server, int> _connectionCounts;
    private readonly object _lock = new object();

    public LeastConnectionsLoadBalancer(
        ILogger<LeastConnectionsLoadBalancer> logger,
        IReadOnlyList<Server> servers)
    {
        _logger = logger;
        _servers = servers;
        _connectionCounts = new ConcurrentDictionary<Server, int>();
    }

    public Task<Server> GetNextServerAsync()
    {
        lock (_lock)
        {
            var server = _servers
                .OrderBy(s => _connectionCounts.GetOrAdd(s, 0))
                .First();

            _connectionCounts.AddOrUpdate(server, 1, (_, count) => count + 1);
            return Task.FromResult(server);
        }
    }

    public Task ReleaseConnectionAsync(Server server)
    {
        lock (_lock)
        {
            _connectionCounts.AddOrUpdate(server, 0, (_, count) => Math.Max(0, count - 1));
            return Task.CompletedTask;
        }
    }
}
```

### 4. Health Check ve Yük Dengeleme
```csharp
public class HealthCheckLoadBalancer : ILoadBalancer
{
    private readonly ILogger<HealthCheckLoadBalancer> _logger;
    private readonly IReadOnlyList<Server> _servers;
    private readonly IHealthChecker _healthChecker;
    private readonly TimeSpan _healthCheckInterval;
    private readonly ConcurrentDictionary<Server, bool> _healthyServers;
    private readonly Timer _healthCheckTimer;

    public HealthCheckLoadBalancer(
        ILogger<HealthCheckLoadBalancer> logger,
        IReadOnlyList<Server> servers,
        IHealthChecker healthChecker,
        TimeSpan healthCheckInterval)
    {
        _logger = logger;
        _servers = servers;
        _healthChecker = healthChecker;
        _healthCheckInterval = healthCheckInterval;
        _healthyServers = new ConcurrentDictionary<Server, bool>();

        // İlk sağlık kontrolü
        _ = CheckHealthAsync();

        // Periyodik sağlık kontrolü
        _healthCheckTimer = new Timer(async _ => await CheckHealthAsync(), 
            null, _healthCheckInterval, _healthCheckInterval);
    }

    public async Task<Server> GetNextServerAsync()
    {
        var healthyServers = _healthyServers
            .Where(kvp => kvp.Value)
            .Select(kvp => kvp.Key)
            .ToList();

        if (!healthyServers.Any())
        {
            throw new NoHealthyServersException();
        }

        // Round Robin stratejisi ile sağlıklı sunuculardan birini seç
        var index = new Random().Next(healthyServers.Count);
        return healthyServers[index];
    }

    private async Task CheckHealthAsync()
    {
        foreach (var server in _servers)
        {
            try
            {
                var isHealthy = await _healthChecker.CheckHealthAsync(server);
                _healthyServers[server] = isHealthy;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Sağlık kontrolü hatası");
                _healthyServers[server] = false;
            }
        }
    }
}
```

## Best Practices

### 1. Yük Dengeleme Stratejileri
- Uygulama ihtiyaçlarına göre strateji seç
- Sunucu kapasitelerini dikkate al
- Sağlık kontrollerini düzenli yap
- Otomatik ölçeklendirme ile entegre et
- Sticky session desteği sağla

### 2. Performans Optimizasyonu
- Önbellekleme stratejileri uygula
- SSL/TLS terminasyonu yap
- Gzip sıkıştırma kullan
- Keep-alive bağlantıları yönet
- Bağlantı havuzu oluştur

### 3. Monitoring ve Logging
- Performans metriklerini topla
- Hata oranlarını izle
- Yanıt sürelerini takip et
- Kaynak kullanımını monitörle
- Otomatik alarmlar kur

## Sık Sorulan Sorular

### 1. Hangi yük dengeleme stratejisi ne zaman kullanılmalıdır?
- Round Robin: Basit ve eşit yük dağılımı
- Weighted Round Robin: Sunucu kapasiteleri farklı olduğunda
- Least Connections: Uzun süren bağlantılar olduğunda
- IP Hash: Oturum tutarlılığı gerektiğinde
- Least Response Time: Performans optimizasyonu gerektiğinde

### 2. Yük dengeleyici seçiminde nelere dikkat edilmelidir?
- Ölçeklenebilirlik
- Yüksek erişilebilirlik
- SSL/TLS desteği
- Monitoring yetenekleri
- Maliyet

### 3. Yük dengelemede karşılaşılan zorluklar nelerdir?
- Oturum yönetimi
- Veri tutarlılığı
- SSL terminasyonu
- Sağlık kontrolü
- Ölçeklendirme

## Kaynaklar
- [Microsoft Load Balancing](https://docs.microsoft.com/tr-tr/azure/architecture/guide/technology-choices/load-balancing-overview)
- [AWS Load Balancing](https://aws.amazon.com/elasticloadbalancing/)
- [NGINX Load Balancing](https://www.nginx.com/resources/glossary/load-balancing/)
- [Load Balancing Algorithms](https://www.nginx.com/resources/glossary/load-balancing-algorithms/) 