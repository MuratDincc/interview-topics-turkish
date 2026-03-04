# Failover Stratejileri

## Genel Bakış

Failover, bir sistemin birincil bileşeni arıza yaptığında trafiğin otomatik ya da manuel olarak yedek bileşene yönlendirilmesi işlemidir. Etkili bir failover stratejisi; RTO ve RPO hedefleri, maliyet kısıtları, uygulama karmaşıklığı ve ekip operasyonel kapasitesi dikkate alınarak seçilir. Senior backend geliştiriciler farklı stratejilerin avantaj ve dezavantajlarını derinlemesine kavramalı; C# ekosistemine özgü araçlarla (Polly, ASP.NET Core Health Checks, EF Core) bu stratejileri implement edebilmelidir.

Bu bölümde Active-Passive ve Active-Active mimariler, Hot/Warm/Cold Standby yaklaşımları, veritabanı failover senaryoları ve Circuit Breaker pattern'in failover ile entegrasyonu ele alınmaktadır.

## Failover Mimarileri

### Active-Passive (Aktif-Pasif) Mimari

Bir birincil (active) sistem tüm trafiği karşılarken, yedek (passive) sistem hazır bekler. Birincil sistem devre dışı kalınca failover tetiklenir ve yedek sistem aktif hale geçer.

**Avantajlar:**
- Daha düşük maliyet (yedek sistem trafiği karşılamaz)
- Daha basit veri senkronizasyonu
- Çakışan yazma işlemleri sorun oluşturmaz (split-brain riski düşük)

**Dezavantajlar:**
- Failover süresi boyunca kısa kesinti yaşanabilir (RTO > 0)
- Yedek sistem kapasitesi atıl kalır
- Yedek sistemin hazırlık durumu sürekli izlenmelidir

```csharp
public class ActivePassiveLoadBalancer
{
    private readonly ILogger<ActivePassiveLoadBalancer> _logger;
    private readonly IHealthChecker _healthChecker;
    private readonly List<ServerEndpoint> _servers;
    private int _activeServerIndex = 0;
    private readonly SemaphoreSlim _failoverLock = new(1, 1);

    public ActivePassiveLoadBalancer(
        ILogger<ActivePassiveLoadBalancer> logger,
        IHealthChecker healthChecker,
        IEnumerable<ServerEndpoint> servers)
    {
        _logger = logger;
        _healthChecker = healthChecker;
        _servers = servers.ToList();
    }

    public ServerEndpoint GetActiveServer() => _servers[_activeServerIndex];

    public async Task<bool> TriggerFailoverAsync(CancellationToken cancellationToken = default)
    {
        // Yalnızca bir failover aynı anda çalışsın
        if (!await _failoverLock.WaitAsync(TimeSpan.Zero))
        {
            _logger.LogWarning("Failover zaten devam ediyor, yeni istek atlandı.");
            return false;
        }

        try
        {
            var currentActive = _servers[_activeServerIndex];
            _logger.LogWarning(
                "Failover tetiklendi. Mevcut aktif sunucu: {Server}", currentActive.Address);

            // Sağlıklı bir yedek bul
            for (int i = 1; i < _servers.Count; i++)
            {
                var candidateIndex = (_activeServerIndex + i) % _servers.Count;
                var candidate = _servers[candidateIndex];

                var isHealthy = await _healthChecker.CheckAsync(candidate, cancellationToken);
                if (!isHealthy)
                {
                    _logger.LogWarning(
                        "Yedek sunucu sağlıksız, atlanıyor: {Server}", candidate.Address);
                    continue;
                }

                // Yeni aktif sunucuyu ata
                _activeServerIndex = candidateIndex;
                _logger.LogInformation(
                    "Failover tamamlandı. Yeni aktif sunucu: {Server}", candidate.Address);
                return true;
            }

            _logger.LogError("Sağlıklı yedek sunucu bulunamadı! Failover başarısız.");
            return false;
        }
        finally
        {
            _failoverLock.Release();
        }
    }
}

public record ServerEndpoint(string Address, int Port, string Region);
```

### Active-Active (Aktif-Aktif) Mimari

Birden fazla sistem aynı anda trafik karşılar. Bir sistem devre dışı kalınca diğerleri tüm yükü üstlenir; kesinti süresi neredeyse sıfırdır.

**Avantajlar:**
- Neredeyse sıfır kesinti süresi (RTO ~ 0)
- Tüm sunucu kapasitesi aktif olarak kullanılır
- Doğal yük dengeleme

**Dezavantajlar:**
- Veri tutarlılığı karmaşıktır (split-brain riski)
- Çakışan yazma işlemleri çözüme kavuşturulmalıdır
- Daha yüksek operasyonel karmaşıklık ve maliyet

```csharp
public class ActiveActiveRouter
{
    private readonly ILogger<ActiveActiveRouter> _logger;
    private readonly IHealthChecker _healthChecker;
    private readonly IReadOnlyList<ServerEndpoint> _servers;
    private readonly object _lockObj = new();
    private int _roundRobinIndex = 0;

    public ActiveActiveRouter(
        ILogger<ActiveActiveRouter> logger,
        IHealthChecker healthChecker,
        IEnumerable<ServerEndpoint> servers)
    {
        _logger = logger;
        _healthChecker = healthChecker;
        _servers = servers.ToList().AsReadOnly();
    }

    /// <summary>
    /// Round-robin algoritmasıyla sağlıklı bir sunucu döndürür.
    /// Sağlıksız sunucular otomatik olarak devre dışı bırakılır.
    /// </summary>
    public async Task<ServerEndpoint?> GetNextServerAsync(
        CancellationToken cancellationToken = default)
    {
        for (int attempt = 0; attempt < _servers.Count; attempt++)
        {
            ServerEndpoint candidate;
            lock (_lockObj)
            {
                var index = _roundRobinIndex % _servers.Count;
                _roundRobinIndex++;
                candidate = _servers[index];
            }

            var isHealthy = await _healthChecker.CheckAsync(candidate, cancellationToken);
            if (isHealthy)
                return candidate;

            _logger.LogWarning(
                "Sunucu sağlıksız, bir sonraki deneniyor: {Server}", candidate.Address);
        }

        _logger.LogError("Hiçbir sağlıklı sunucu bulunamadı!");
        return null;
    }
}

/// <summary>
/// Active-Active senaryolarında çakışan yazmaları çözmek için Last-Write-Wins stratejisi.
/// </summary>
public class ConflictResolver<T> where T : IVersioned
{
    public T Resolve(T local, T remote)
    {
        // En son güncellenen kaydı kabul et (Last-Write-Wins)
        return local.UpdatedAt >= remote.UpdatedAt ? local : remote;
    }
}

public interface IVersioned
{
    DateTime UpdatedAt { get; }
}
```

## Standby Seviyeleri

### Hot Standby

Yedek sistem birincil sistemle sürekli senkronize çalışır ve anlık failover yapılabilir. Hem veri hem de uygulama katmanı hazırdır; trafik saniyeler içinde yedek sisteme yönlendirilebilir.

- **RTO:** Saniyeler
- **RPO:** Sıfır veya sıfıra yakın
- **Maliyet:** En yüksek (yedek sistem tam kapasitede çalışır)
- **Kullanım:** Finansal sistemler, e-ticaret, kritik altyapı

### Warm Standby

Yedek sistem çalışır durumdadır ve belirli aralıklarla verilerle güncellenir; ancak tam kapasite ile trafik karşılamaya hazır değildir. Failover için kısa bir hazırlık süresi gerekir.

- **RTO:** Dakikalar (5-30 dakika)
- **RPO:** Dakikalar (replikasyon aralığına bağlı)
- **Maliyet:** Orta
- **Kullanım:** Kurumsal uygulamalar, dahili servisler

### Cold Standby

Yedek sistem pasiftir; veriler periyodik olarak yedeklenir ancak sistem sürekli çalışmaz. Failover için sistemin sıfırdan ayağa kaldırılması gerekir.

- **RTO:** Saatler
- **RPO:** Saatler veya günler
- **Maliyet:** En düşük
- **Kullanım:** Arşiv sistemleri, geliştirme ortamları, düşük kritikiyetli uygulamalar

```csharp
public enum StandbyMode { Hot, Warm, Cold }

public class StandbySystemManager
{
    private readonly ILogger<StandbySystemManager> _logger;
    private readonly IInfrastructureProvisioner _provisioner;
    private readonly IDataReplicator _replicator;

    public StandbySystemManager(
        ILogger<StandbySystemManager> logger,
        IInfrastructureProvisioner provisioner,
        IDataReplicator replicator)
    {
        _logger = logger;
        _provisioner = provisioner;
        _replicator = replicator;
    }

    public Task<TimeSpan> EstimateFailoverTimeAsync(StandbyMode mode) =>
        mode switch
        {
            StandbyMode.Hot  => Task.FromResult(TimeSpan.FromSeconds(10)),
            StandbyMode.Warm => Task.FromResult(TimeSpan.FromMinutes(15)),
            StandbyMode.Cold => Task.FromResult(TimeSpan.FromHours(2)),
            _                => throw new ArgumentOutOfRangeException(nameof(mode))
        };

    public async Task ActivateStandbyAsync(
        StandbyMode mode,
        CancellationToken cancellationToken = default)
    {
        _logger.LogInformation("Standby aktivasyonu başlatılıyor. Mod: {Mode}", mode);

        switch (mode)
        {
            case StandbyMode.Hot:
                // Sadece trafik yönlendirmesini değiştir
                await ActivateHotStandbyAsync(cancellationToken);
                break;

            case StandbyMode.Warm:
                // Eksik verileri senkronize et, sonra trafiği yönlendir
                await SynchronizeMissingDataAsync(cancellationToken);
                await ActivateWarmStandbyAsync(cancellationToken);
                break;

            case StandbyMode.Cold:
                // Altyapıyı kur, son yedeği geri yükle, başlat
                await _provisioner.ProvisionInfrastructureAsync(cancellationToken);
                await _replicator.RestoreLatestBackupAsync(cancellationToken);
                await ActivateColdStandbyAsync(cancellationToken);
                break;
        }

        _logger.LogInformation("Standby aktivasyonu tamamlandı. Mod: {Mode}", mode);
    }

    private async Task ActivateHotStandbyAsync(CancellationToken cancellationToken)
    {
        // DNS veya load balancer kuralını güncelle
        _logger.LogInformation("Hot standby: Trafik yönlendirmesi güncelleniyor.");
        await Task.Delay(TimeSpan.FromSeconds(5), cancellationToken); // Simülasyon
    }

    private async Task SynchronizeMissingDataAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Warm standby: Eksik veriler senkronize ediliyor.");
        await Task.Delay(TimeSpan.FromMinutes(5), cancellationToken); // Simülasyon
    }

    private async Task ActivateWarmStandbyAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Warm standby: Sistem aktive ediliyor.");
        await Task.Delay(TimeSpan.FromMinutes(2), cancellationToken); // Simülasyon
    }

    private async Task ActivateColdStandbyAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Cold standby: Sistem başlatılıyor.");
        await Task.Delay(TimeSpan.FromHours(1), cancellationToken); // Simülasyon
    }
}
```

## Veritabanı Failover Stratejileri

### SQL Server Always On Availability Groups

SQL Server Always On, senkron ve asenkron replikasyon modlarını destekleyen yüksek erişilebilirlik çözümüdür. Senkron modda veri kaybı yoktur; asenkron modda ise coğrafi uzaklık nedeniyle küçük bir gecikme kabul edilir.

```csharp
public class SqlServerFailoverConnectionFactory
{
    private readonly SqlServerFailoverConfiguration _config;
    private readonly ILogger<SqlServerFailoverConnectionFactory> _logger;

    public SqlServerFailoverConnectionFactory(
        SqlServerFailoverConfiguration config,
        ILogger<SqlServerFailoverConnectionFactory> logger)
    {
        _config = config;
        _logger = logger;
    }

    /// <summary>
    /// Always On Availability Group'u destekleyen bağlantı dizesi oluşturur.
    /// ApplicationIntent=ReadOnly; replica'dan okuma yapar.
    /// </summary>
    public string CreateConnectionString(bool readOnly = false)
    {
        var builder = new Microsoft.Data.SqlClient.SqlConnectionStringBuilder
        {
            // Listener adresi - otomatik failover'ı destekler
            DataSource              = _config.AvailabilityGroupListener,
            InitialCatalog          = _config.DatabaseName,
            IntegratedSecurity      = false,
            UserID                  = _config.Username,
            Password                = _config.Password,
            MultiSubnetFailover     = true,   // Çoklu subnet failover desteği
            ConnectTimeout          = 30,
            ApplicationIntent       = readOnly
                ? Microsoft.Data.SqlClient.ApplicationIntent.ReadOnly
                : Microsoft.Data.SqlClient.ApplicationIntent.ReadWrite
        };

        return builder.ConnectionString;
    }
}

public class SqlServerFailoverConfiguration
{
    public string AvailabilityGroupListener { get; init; } = string.Empty;
    public string DatabaseName { get; init; } = string.Empty;
    public string Username { get; init; } = string.Empty;
    public string Password { get; init; } = string.Empty;
}

// Entity Framework Core ile CQRS + Read Replica
public class ReadWriteSplitDbContextFactory
{
    private readonly SqlServerFailoverConnectionFactory _connectionFactory;

    public ReadWriteSplitDbContextFactory(SqlServerFailoverConnectionFactory connectionFactory)
    {
        _connectionFactory = connectionFactory;
    }

    public AppDbContext CreateWriteContext()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer(_connectionFactory.CreateConnectionString(readOnly: false))
            .EnableSensitiveDataLogging(false)
            .Options;
        return new AppDbContext(options);
    }

    public AppDbContext CreateReadContext()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer(_connectionFactory.CreateConnectionString(readOnly: true))
            .EnableSensitiveDataLogging(false)
            .Options;
        return new AppDbContext(options);
    }
}
```

### PostgreSQL Streaming Replication ile Failover

```csharp
public class PostgreSqlFailoverService
{
    private readonly ILogger<PostgreSqlFailoverService> _logger;
    private readonly PostgreSqlFailoverConfiguration _config;
    private string _currentPrimary;
    private readonly List<string> _replicas;

    public PostgreSqlFailoverService(
        ILogger<PostgreSqlFailoverService> logger,
        PostgreSqlFailoverConfiguration config)
    {
        _logger = logger;
        _config = config;
        _currentPrimary = config.PrimaryConnectionString;
        _replicas = new List<string>(config.ReplicaConnectionStrings);
    }

    public string GetPrimaryConnectionString() => _currentPrimary;

    public IReadOnlyList<string> GetReplicaConnectionStrings() =>
        _replicas.AsReadOnly();

    public async Task<bool> PromoteReplicaAsync(
        string replicaConnectionString,
        CancellationToken cancellationToken = default)
    {
        _logger.LogWarning(
            "Replica promote ediliyor. Replica: {Replica}", replicaConnectionString);

        try
        {
            // pg_ctl promote veya pg_promote() fonksiyonu ile replica'yı primary'ye yükselt
            await using var connection = new Npgsql.NpgsqlConnection(replicaConnectionString);
            await connection.OpenAsync(cancellationToken);

            await using var command = connection.CreateCommand();
            command.CommandText = "SELECT pg_promote()";
            var result = await command.ExecuteScalarAsync(cancellationToken);

            if (result is true)
            {
                _logger.LogInformation(
                    "Replica başarıyla primary'ye yükseltildi: {Replica}",
                    replicaConnectionString);

                // Bağlantı bilgilerini güncelle
                _replicas.Remove(replicaConnectionString);
                _currentPrimary = replicaConnectionString;
                return true;
            }

            _logger.LogError("pg_promote() başarısız döndü.");
            return false;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex,
                "Replica promote işlemi başarısız: {Replica}", replicaConnectionString);
            return false;
        }
    }
}

public class PostgreSqlFailoverConfiguration
{
    public string PrimaryConnectionString { get; init; } = string.Empty;
    public IReadOnlyList<string> ReplicaConnectionStrings { get; init; } =
        Array.Empty<string>();
}
```

## Circuit Breaker Pattern ile Failover

Circuit Breaker, bağımlı bir servisteki arıza tüm sisteme yayılmadan önce isteği keserek hem sistemi korur hem de failover mekanizmasını tetikler.

### Polly ile Circuit Breaker Implementasyonu

```csharp
using Polly;
using Polly.CircuitBreaker;

public class ResilientServiceClient
{
    private readonly ILogger<ResilientServiceClient> _logger;
    private readonly HttpClient _httpClient;
    private readonly AsyncCircuitBreakerPolicy<HttpResponseMessage> _circuitBreakerPolicy;
    private readonly AsyncRetryPolicy<HttpResponseMessage> _retryPolicy;

    public ResilientServiceClient(
        ILogger<ResilientServiceClient> logger,
        HttpClient httpClient)
    {
        _logger = logger;
        _httpClient = httpClient;

        // Circuit Breaker: 5 başarısız istek sonrası 30 saniye devre dışı bırak
        _circuitBreakerPolicy = Policy
            .HandleResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode)
            .Or<HttpRequestException>()
            .Or<TaskCanceledException>()
            .AdvancedCircuitBreakerAsync(
                failureThreshold: 0.5,                         // %50 hata oranı
                samplingDuration: TimeSpan.FromSeconds(30),    // 30 saniye örnekleme
                minimumThroughput: 5,                          // Minimum 5 istek
                durationOfBreak: TimeSpan.FromSeconds(30),     // 30 saniye açık kal
                onBreak: (result, duration) =>
                {
                    _logger.LogError(
                        "Circuit Breaker AÇILDI. Servis {Duration} süre devre dışı.",
                        duration);
                },
                onReset: () =>
                {
                    _logger.LogInformation("Circuit Breaker KAPANDI. Servis tekrar aktif.");
                },
                onHalfOpen: () =>
                {
                    _logger.LogInformation(
                        "Circuit Breaker YARI AÇIK. Test isteği gönderiliyor.");
                });

        // Retry: 3 deneme, üstel geri çekilme ile
        _retryPolicy = Policy
            .HandleResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode)
            .Or<HttpRequestException>()
            .WaitAndRetryAsync(
                retryCount: 3,
                sleepDurationProvider: attempt =>
                    TimeSpan.FromSeconds(Math.Pow(2, attempt)), // 2, 4, 8 saniye
                onRetry: (result, delay, attempt, context) =>
                {
                    _logger.LogWarning(
                        "İstek yeniden deneniyor. Deneme: {Attempt}, Bekleme: {Delay}",
                        attempt, delay);
                });
    }

    public async Task<HttpResponseMessage> SendWithResilienceAsync(
        HttpRequestMessage request,
        string? fallbackEndpoint = null,
        CancellationToken cancellationToken = default)
    {
        try
        {
            // Retry + Circuit Breaker kombinasyonu
            return await _retryPolicy.WrapAsync(_circuitBreakerPolicy)
                .ExecuteAsync(() => _httpClient.SendAsync(
                    CloneRequest(request), cancellationToken));
        }
        catch (BrokenCircuitException)
        {
            _logger.LogWarning("Circuit açık! Fallback endpoint deneniyor.");

            if (fallbackEndpoint is null)
                throw;

            // Fallback endpoint'e yönlendir
            var fallbackRequest = CloneRequest(request);
            fallbackRequest.RequestUri = new Uri(fallbackEndpoint);
            return await _httpClient.SendAsync(fallbackRequest, cancellationToken);
        }
    }

    private static HttpRequestMessage CloneRequest(HttpRequestMessage request)
    {
        var clone = new HttpRequestMessage(request.Method, request.RequestUri);
        foreach (var header in request.Headers)
            clone.Headers.TryAddWithoutValidation(header.Key, header.Value);
        return clone;
    }
}
```

### Polly ile Tam Failover Pipeline'ı

```csharp
public class FailoverHttpClientFactory
{
    private readonly ILogger<FailoverHttpClientFactory> _logger;

    public FailoverHttpClientFactory(ILogger<FailoverHttpClientFactory> logger)
    {
        _logger = logger;
    }

    /// <summary>
    /// Primary + Fallback endpoint'lerle tam failover destekli HTTP politikası oluşturur.
    /// </summary>
    public IAsyncPolicy<HttpResponseMessage> CreateFailoverPolicy(
        string primaryUrl,
        string fallbackUrl)
    {
        // Birincil servis circuit breaker'ı
        var primaryCircuitBreaker = Policy
            .HandleResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode)
            .Or<Exception>()
            .CircuitBreakerAsync(
                handledEventsAllowedBeforeBreaking: 3,
                durationOfBreak: TimeSpan.FromMinutes(1));

        // Fallback politikası: birincil başarısız olunca yedek URL'e git
        var fallbackPolicy = Policy<HttpResponseMessage>
            .Handle<BrokenCircuitException>()
            .Or<HttpRequestException>()
            .FallbackAsync(
                fallbackAction: async (context, cancellationToken) =>
                {
                    _logger.LogWarning(
                        "Birincil servis başarısız, fallback devreye giriyor: {Url}",
                        fallbackUrl);

                    using var client = new HttpClient();
                    return await client.GetAsync(fallbackUrl, cancellationToken);
                },
                onFallbackAsync: (result, context) =>
                {
                    _logger.LogWarning(
                        "Fallback tetiklendi. Neden: {Reason}",
                        result.Exception?.Message ?? result.Result?.StatusCode.ToString());
                    return Task.CompletedTask;
                });

        return Policy.WrapAsync(fallbackPolicy, primaryCircuitBreaker);
    }
}

// DI kayıt örneği (Program.cs)
public static class FailoverHttpClientExtensions
{
    public static IServiceCollection AddFailoverHttpClient(
        this IServiceCollection services,
        string primaryUrl,
        string fallbackUrl)
    {
        services.AddHttpClient("resilient-client")
            .AddPolicyHandler((provider, _) =>
            {
                var factory = provider.GetRequiredService<FailoverHttpClientFactory>();
                return factory.CreateFailoverPolicy(primaryUrl, fallbackUrl);
            });

        return services;
    }
}
```

## DNS Tabanlı Failover

DNS tabanlı failover, yük dengeleme ve trafik yönlendirmesini DNS katmanında gerçekleştirir. AWS Route 53 Health Checks veya Azure Traffic Manager gibi servisler sağlık kontrolüne göre otomatik olarak DNS kayıtlarını günceller.

```csharp
public class DnsFailoverMonitor : BackgroundService
{
    private readonly ILogger<DnsFailoverMonitor> _logger;
    private readonly IHealthChecker _healthChecker;
    private readonly IDnsManager _dnsManager;
    private readonly DnsFailoverConfiguration _config;

    public DnsFailoverMonitor(
        ILogger<DnsFailoverMonitor> logger,
        IHealthChecker healthChecker,
        IDnsManager dnsManager,
        DnsFailoverConfiguration config)
    {
        _logger = logger;
        _healthChecker = healthChecker;
        _dnsManager = dnsManager;
        _config = config;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("DNS Failover Monitor başlatıldı.");

        while (!stoppingToken.IsCancellationRequested)
        {
            await CheckAndFailoverAsync(stoppingToken);
            await Task.Delay(_config.CheckInterval, stoppingToken);
        }
    }

    private async Task CheckAndFailoverAsync(CancellationToken cancellationToken)
    {
        var primaryHealthy = await _healthChecker.CheckAsync(
            new ServerEndpoint(_config.PrimaryEndpoint, 443, "primary"),
            cancellationToken);

        if (primaryHealthy)
        {
            // Primary sağlıklı; DNS'in primary'ye işaret ettiğinden emin ol
            var currentDnsTarget = await _dnsManager.GetCurrentTargetAsync(
                _config.DomainName, cancellationToken);

            if (currentDnsTarget != _config.PrimaryEndpoint)
            {
                _logger.LogInformation(
                    "Primary sağlıklı, DNS primary'ye geri döndürülüyor: {Domain}",
                    _config.DomainName);

                await _dnsManager.UpdateDnsRecordAsync(
                    _config.DomainName,
                    _config.PrimaryEndpoint,
                    ttlSeconds: 60,
                    cancellationToken);
            }
        }
        else
        {
            _logger.LogWarning(
                "Primary sağlıksız! DNS failover başlatılıyor: {Domain} -> {Fallback}",
                _config.DomainName, _config.FallbackEndpoint);

            await _dnsManager.UpdateDnsRecordAsync(
                _config.DomainName,
                _config.FallbackEndpoint,
                ttlSeconds: 30, // Kısa TTL: failover daha hızlı yayılır
                cancellationToken);
        }
    }
}

public class DnsFailoverConfiguration
{
    public string DomainName { get; init; } = string.Empty;
    public string PrimaryEndpoint { get; init; } = string.Empty;
    public string FallbackEndpoint { get; init; } = string.Empty;
    public TimeSpan CheckInterval { get; init; } = TimeSpan.FromSeconds(30);
}
```

## Sağlık Kontrolü (Health Check) ile Otomatik Failover

```csharp
public class AutomaticFailoverService : BackgroundService
{
    private readonly ILogger<AutomaticFailoverService> _logger;
    private readonly IHealthCheckService _healthCheckService;
    private readonly ActivePassiveLoadBalancer _loadBalancer;
    private readonly FailoverMetrics _metrics;

    public AutomaticFailoverService(
        ILogger<AutomaticFailoverService> logger,
        IHealthCheckService healthCheckService,
        ActivePassiveLoadBalancer loadBalancer,
        FailoverMetrics metrics)
    {
        _logger = logger;
        _healthCheckService = healthCheckService;
        _loadBalancer = loadBalancer;
        _metrics = metrics;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Otomatik failover servisi başlatıldı.");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await MonitorAndFailoverAsync(stoppingToken);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                _logger.LogError(ex, "Failover izleme döngüsünde hata.");
            }

            await Task.Delay(TimeSpan.FromSeconds(10), stoppingToken);
        }
    }

    private async Task MonitorAndFailoverAsync(CancellationToken cancellationToken)
    {
        var healthReport = await _healthCheckService.CheckHealthAsync(
            predicate: check => check.Tags.Contains("primary"),
            cancellationToken);

        var primaryHealthy = healthReport.Status == HealthStatus.Healthy;

        _metrics.RecordHealthStatus(primaryHealthy);

        if (!primaryHealthy)
        {
            _logger.LogWarning(
                "Birincil sistem sağlıksız. Durum: {Status}. Failover başlatılıyor.",
                healthReport.Status);

            var failoverSuccess = await _loadBalancer.TriggerFailoverAsync(cancellationToken);

            _metrics.RecordFailoverAttempt(failoverSuccess);

            if (failoverSuccess)
            {
                _logger.LogInformation("Otomatik failover başarıyla tamamlandı.");
            }
            else
            {
                _logger.LogError("Otomatik failover başarısız oldu! Manuel müdahale gerekiyor.");
            }
        }
    }
}

public class FailoverMetrics
{
    private int _failoverAttempts = 0;
    private int _successfulFailovers = 0;

    public void RecordHealthStatus(bool isHealthy) { /* Prometheus metriği yayınla */ }

    public void RecordFailoverAttempt(bool success)
    {
        Interlocked.Increment(ref _failoverAttempts);
        if (success)
            Interlocked.Increment(ref _successfulFailovers);
    }

    public double FailoverSuccessRate =>
        _failoverAttempts == 0 ? 0 :
        (double)_successfulFailovers / _failoverAttempts * 100;
}
```

## Mülakat Soruları

### 1. Soru
**Active-Passive ve Active-Active mimariler arasındaki temel farklar nelerdir? Hangisini ne zaman tercih edersiniz?**

**Cevap:** Active-Passive mimaride yalnızca birincil sistem trafik karşılar; yedek sistem hazır bekler. Birincil arızalanınca failover tetiklenir ve kısa bir kesinti yaşanabilir. Maliyet açısından daha verimlidir çünkü yedek sistem atıl kapasitedir. Veri tutarlılığı yönetimi daha basittir.

Active-Active mimaride tüm sistemler aynı anda trafik karşılar; bir sistem devre dışı kalınca diğerleri tüm yükü üstlenir, neredeyse sıfır kesinti yaşanır. Tüm kapasite aktif kullanıldığından kaynak israfı yoktur; ancak veri tutarlılığı (split-brain, çakışan yazmalar) daha karmaşık yönetim gerektirir ve maliyet daha yüksektir.

Tercih: Sıkı SLA gereksinimleri ve kesintiye hiç tolerans yoksa Active-Active; daha düşük maliyet ve basit operasyon öncelikliyse Active-Passive tercih edilir.

**Örnek Kod:**
```csharp
// Active-Active: Her sunucu trafik alır, sağlık kontrolü ile yönlendirme
services.AddHttpClient("payment-service")
    .AddPolicyHandler(Policy<HttpResponseMessage>
        .Handle<Exception>()
        .FallbackAsync(async (ctx, ct) =>
        {
            // Birincil başarısız, ikincil endpoint'e yönlendir
            using var fallbackClient = new HttpClient();
            return await fallbackClient.GetAsync("https://payment-secondary.example.com/api", ct);
        }));
```

### 2. Soru
**Circuit Breaker pattern nedir ve failover ile nasıl birlikte kullanılır?**

**Cevap:** Circuit Breaker, bir bağımlılıkta hata oranı belirli bir eşiği aşınca isteklerin o bağımlılığa iletilmesini geçici olarak durdurur. Üç durumu vardır: Closed (normal çalışma), Open (istekler engellenir) ve Half-Open (test isteği gönderilir). Failover ile birlikte kullanımda: Circuit açık durumdayken istekler otomatik olarak fallback servise veya yedek endpoint'e yönlendirilir. Bu sayede hem birincil servisin kurtarılmasına zaman tanınır hem de sistemin genel sağlığı korunur. Polly kütüphanesi .NET ekosisteminde bu pattern'ı kolaylıkla uygulamayı sağlar.

**Örnek Kod:**
```csharp
// Polly ile Circuit Breaker + Fallback kombinasyonu
var policy = Policy<HttpResponseMessage>
    .Handle<Exception>()
    .FallbackAsync(fallbackAction: (ctx, ct) =>
        Task.FromResult(new HttpResponseMessage(HttpStatusCode.ServiceUnavailable)))
    .WrapAsync(
        Policy<HttpResponseMessage>
            .Handle<Exception>()
            .CircuitBreakerAsync(
                handledEventsAllowedBeforeBreaking: 5,
                durationOfBreak: TimeSpan.FromSeconds(30)));
```

### 3. Soru
**Split-brain problemi nedir ve nasıl önlenir?**

**Cevap:** Split-brain, dağıtık sistemlerde birden fazla düğümün kendisini birincil (primary/leader) olarak kabul etmesi ve birbirinden bağımsız yazma işlemleri yapması durumudur. Bu durum veri tutarsızlığına ve çakışan güncellemelere yol açar. Önleme yöntemleri: (1) Quorum tabanlı seçim: seçim için düğümlerin çoğunluğunun (N/2 + 1) onayı gereklidir; böylece iki ayrı "çoğunluk" oluşamaz, (2) Fencing tokens: her primary, isteklerini benzersiz bir token ile imzalar; eski primary'nin tokeni geçersizleşince yazmaları reddedilir, (3) STONITH (Shoot The Other Node In The Head): arızalı düğümü zorla kapatma, (4) Network partition tespiti ve servis duraklatma. Raft ve Paxos gibi dağıtık konsensüs algoritmaları split-brain sorununu yapısal olarak çözer.

**Örnek Kod:**
```csharp
public class LeaderElectionService
{
    private readonly IDistributedLock _distributedLock;
    private readonly ILogger<LeaderElectionService> _logger;

    public LeaderElectionService(
        IDistributedLock distributedLock,
        ILogger<LeaderElectionService> logger)
    {
        _distributedLock = distributedLock;
        _logger = logger;
    }

    // Redis veya ZooKeeper tabanlı distributed lock ile tek leader garantisi
    public async Task<bool> TryBecomeLeaderAsync(
        string nodeId,
        CancellationToken cancellationToken = default)
    {
        // Yalnızca bir düğüm lock'u alabilir; diğerleri follower kalır
        var acquired = await _distributedLock.TryAcquireAsync(
            key: "leader-election",
            holder: nodeId,
            expiry: TimeSpan.FromSeconds(30),
            cancellationToken);

        if (acquired)
            _logger.LogInformation("Bu düğüm leader seçildi: {NodeId}", nodeId);

        return acquired;
    }
}
```

### 4. Soru
**Veritabanı failover sırasında veri tutarlılığı nasıl sağlanır?**

**Cevap:** Veritabanı failover sırasında veri tutarlılığı için şu yaklaşımlar kullanılır: (1) Senkron replikasyon: yazma işlemi tüm replikalara uygulanana kadar tamamlanmaz; sıfır veri kaybı garantisi verir ancak yazma latency'si artar, (2) Fencing: eski primary devreye girmeye çalışırsa okuma/yazma yetkileri iptal edilir, (3) WAL (Write-Ahead Log) shipping: PostgreSQL'de transaction logları replica'ya gönderilir; replica bu logları uygulayarak primary ile senkron kalır, (4) Conflict resolution: Active-Active senaryolarında Last-Write-Wins, timestamp tabanlı veya application-level çakışma çözümleme stratejileri uygulanır, (5) Read-after-write consistency: kullanıcı kendi yazdığı veriyi okurken primary'den okumaya yönlendirme yapılır.

**Örnek Kod:**
```csharp
// EF Core ile okuma/yazma ayrımı ve tutarlılık garantisi
public class ConsistentDataService
{
    private readonly ReadWriteSplitDbContextFactory _contextFactory;

    public ConsistentDataService(ReadWriteSplitDbContextFactory contextFactory)
    {
        _contextFactory = contextFactory;
    }

    // Yazma: primary'ye git
    public async Task<Order> CreateOrderAsync(CreateOrderCommand command)
    {
        await using var writeCtx = _contextFactory.CreateWriteContext();
        var order = Order.Create(command);
        writeCtx.Orders.Add(order);
        await writeCtx.SaveChangesAsync();
        return order;
    }

    // Okuma: replica'dan oku (eventual consistency kabul edilebilir)
    public async Task<IReadOnlyList<Order>> GetOrdersAsync(string userId)
    {
        await using var readCtx = _contextFactory.CreateReadContext();
        return await readCtx.Orders
            .Where(o => o.UserId == userId)
            .ToListAsync();
    }
}
```

### 5. Soru
**DNS tabanlı failover'ın TTL problemi nedir ve nasıl yönetilir?**

**Cevap:** DNS tabanlı failover, DNS kaydının TTL (Time-To-Live) süresi boyunca önbellekte tutulur; bu süre dolmadan yeni IP adresi yayılmaz. Yüksek TTL değerleri failover süresini uzatır. Yönetim stratejileri: (1) Rutin operasyonlarda düşük TTL (60 saniye) kullanmak, failover güncellemesinin hızlıca yayılmasını sağlar, (2) DR tatbikatı öncesinde TTL değeri düşürülür; tatbikat sonrası tekrar yükseltilir, (3) AWS Route 53 Health Checks ile TTL = 60 saniye kombinasyonu pratik bir standart olarak yaygınlaşmıştır, (4) İstemci tarafında DNS önbellek süresini (OS/uygulama düzeyi) dikkate almak gerekir; bazı istemciler TTL'yi görmezden gelebilir.

**Örnek Kod:**
```csharp
// HttpClient ile DNS yenileme - SocketsHttpHandler
var handler = new SocketsHttpHandler
{
    // DNS değişikliklerinin HTTP client'a yansıması için bağlantı ömrünü sınırla
    PooledConnectionLifetime = TimeSpan.FromMinutes(1)
};

var httpClient = new HttpClient(handler);

// DI ile kayıt
builder.Services.AddHttpClient("failover-aware-client")
    .ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
    {
        // Failover sonrası yeni IP adresinin kullanılmasını sağlar
        PooledConnectionLifetime = TimeSpan.FromMinutes(1),
        ConnectTimeout = TimeSpan.FromSeconds(10)
    });
```

### 6. Soru
**Failover sonrasında fallback (geri dönüş) stratejisi nasıl yönetilmelidir?**

**Cevap:** Fallback (failback), yedek sistemden birincil sisteme geri dönüş sürecidir ve dikkatli yönetilmezse yeni bir kesintiye neden olabilir. Güvenli fallback için şu adımlar izlenmelidir: (1) Birincil sistemin tamamen sağlıklı olduğu doğrulanır; kademeli yük testi yapılır, (2) Veri senkronizasyonu: yedek sistemde oluşan veriler birincil sisteme aktarılır ve tutarlılık kontrol edilir, (3) Kademeli trafik geçişi: tüm trafik aniden değil, yüzde yüzde (canary veya blue-green gibi) birincile aktarılır, (4) Rollback planı hazır tutulur: fallback sırasında sorun çıkarsa yedek sisteme hızla geri dönülebilmelidir, (5) Tüm adımlar tatbikatla test edilmiş olmalıdır.

**Örnek Kod:**
```csharp
public class GradualFailbackService
{
    private readonly ILogger<GradualFailbackService> _logger;
    private volatile int _primaryTrafficPercent = 0;

    public GradualFailbackService(ILogger<GradualFailbackService> logger)
    {
        _logger = logger;
    }

    /// <summary>
    /// Trafiği kademeli olarak birincil sisteme geri taşır.
    /// Her adımda %10 trafik artışı ve 5 dakika izleme süresi.
    /// </summary>
    public async Task ExecuteGradualFailbackAsync(
        CancellationToken cancellationToken = default)
    {
        for (int percent = 10; percent <= 100; percent += 10)
        {
            _logger.LogInformation(
                "Failback: Birincil sisteme trafik yüzdesi artırılıyor -> %{Percent}",
                percent);

            _primaryTrafficPercent = percent;

            // 5 dakika izle
            await Task.Delay(TimeSpan.FromMinutes(5), cancellationToken);

            var primaryHealthy = await CheckPrimaryHealthAsync(cancellationToken);
            if (!primaryHealthy)
            {
                _logger.LogError(
                    "Birincil sistem %{Percent} trafikte sağlıksız! Failback durduruluyor.",
                    percent);
                _primaryTrafficPercent = 0; // Yedek sisteme geri dön
                return;
            }
        }

        _logger.LogInformation("Failback tamamlandı. Tüm trafik birincil sisteme aktarıldı.");
    }

    public bool ShouldUsePrimary()
    {
        return Random.Shared.Next(0, 100) < _primaryTrafficPercent;
    }

    private Task<bool> CheckPrimaryHealthAsync(CancellationToken cancellationToken)
    {
        // Gerçek sağlık kontrolü burada yapılır
        return Task.FromResult(true);
    }
}
```

## Best Practices

### 1. **Otomatik Failover'ı Test Edin**
- Yılda en az iki kez tam DR tatbikatı ile failover sürecini gerçek ortamda test edin.
- Chaos Engineering araçlarıyla (Netflix Chaos Monkey, Azure Chaos Studio) kontrolsüz arızaları simüle edin.
- Otomatik failover testlerini CI/CD pipeline'ına entegre ederek sürekli doğrulamayı sağlayın.

### 2. **İzleme ve Uyarı**
- Birincil sistem sağlık durumu, failover süresi ve başarı oranı gibi metrikleri Prometheus/Grafana ile izleyin.
- Failover tetiklendiğinde anında bildirim gönderilecek uyarı kuralları tanımlayın.
- Postmortem analizi için tüm failover olaylarını detaylı biçimde kayıt altına alın.

### 3. **Runbook Hazırlığı**
- Her failover senaryosu için adım adım runbook hazırlayın.
- Runbook'ları mümkün olduğunca otomatize edin; insan hatasını azaltın.
- Runbook'ları düzenli olarak güncelleyin ve tatbikatlarda doğrulayın.

### 4. **Kapasite Planlaması**
- Yedek sistem, tüm yükü tek başına taşıyabilecek kapasitede tasarlanmalıdır.
- Active-Active mimaride kapasite dengesi sürekli izlenmelidir.
- Otomatik ölçekleme (auto-scaling) ile ani yük artışlarına hazırlıklı olun.

### 5. **Maliyet ve RTO Dengesi**
- Hot standby her zaman gerekli değildir; iş gereksinimlerine göre doğru standby tipi seçilmelidir.
- Bulut sağlayıcıların yönetilen DR servisleri (Azure Site Recovery, AWS Elastic Disaster Recovery) değerlendirilmelidir.
- Kullanılmayan DR kaynaklarını periyodik olarak gözden geçirin.

## Kaynaklar

- [Polly .NET Resilience Library](https://github.com/App-vNext/Polly)
- [Microsoft Azure Traffic Manager](https://docs.microsoft.com/en-us/azure/traffic-manager/traffic-manager-overview)
- [AWS Route 53 Health Checks](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover.html)
- [SQL Server Always On Availability Groups](https://docs.microsoft.com/en-us/sql/database-engine/availability-groups/windows/always-on-availability-groups-sql-server)
- [PostgreSQL Streaming Replication](https://www.postgresql.org/docs/current/warm-standby.html)
- [Azure Chaos Studio](https://docs.microsoft.com/en-us/azure/chaos-studio/chaos-studio-overview)
- [Martin Fowler - Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html)
