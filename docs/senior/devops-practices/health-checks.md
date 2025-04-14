# Health Checks

## Genel Bakış

Health checks, uygulamanın sağlık durumunu izlemek ve raporlamak için kullanılan bir mekanizmadır. Bu bölümde, health checks'in temel kavramlarını ve C# implementasyonlarını inceleyeceğiz.

## Temel Health Checks İşlemleri

### 1. Temel Health Check

```csharp
public class BasicHealthCheck : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            // Sağlık kontrolü
            return Task.FromResult(
                HealthCheckResult.Healthy("Uygulama sağlıklı çalışıyor."));
        }
        catch (Exception ex)
        {
            return Task.FromResult(
                HealthCheckResult.Unhealthy("Uygulama sağlıksız durumda.", ex));
        }
    }
}
```

### 2. Veritabanı Health Check

```csharp
public class DatabaseHealthCheck : IHealthCheck
{
    private readonly DbContext _dbContext;

    public DatabaseHealthCheck(DbContext dbContext)
    {
        _dbContext = dbContext;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            await _dbContext.Database.ExecuteSqlRawAsync("SELECT 1");
            return HealthCheckResult.Healthy("Veritabanı bağlantısı sağlıklı.");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Veritabanı bağlantısı sağlıksız.", ex);
        }
    }
}
```

### 3. Harici Servis Health Check

```csharp
public class ExternalServiceHealthCheck : IHealthCheck
{
    private readonly HttpClient _httpClient;

    public ExternalServiceHealthCheck(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var response = await _httpClient.GetAsync("health", cancellationToken);
            response.EnsureSuccessStatusCode();
            return HealthCheckResult.Healthy("Harici servis sağlıklı çalışıyor.");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Harici servis sağlıksız durumda.", ex);
        }
    }
}
```

## İleri Health Checks Algoritmaları

### 1. Özel Health Check Publisher

```csharp
public class CustomHealthCheckPublisher : IHealthCheckPublisher
{
    private readonly ILogger<CustomHealthCheckPublisher> _logger;

    public CustomHealthCheckPublisher(ILogger<CustomHealthCheckPublisher> logger)
    {
        _logger = logger;
    }

    public Task PublishAsync(HealthReport report, CancellationToken cancellationToken)
    {
        if (report.Status == HealthStatus.Healthy)
        {
            _logger.LogInformation("Uygulama sağlıklı: {Report}", report);
        }
        else
        {
            _logger.LogError("Uygulama sağlıksız: {Report}", report);
        }

        return Task.CompletedTask;
    }
}
```

### 2. Health Check UI

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddHealthChecks()
            .AddCheck<BasicHealthCheck>("basic")
            .AddCheck<DatabaseHealthCheck>("database")
            .AddCheck<ExternalServiceHealthCheck>("external-service");

        services.AddHealthChecksUI()
            .AddInMemoryStorage();
    }

    public void Configure(IApplicationBuilder app)
    {
        app.UseHealthChecks("/health", new HealthCheckOptions
        {
            ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
        });

        app.UseHealthChecksUI();
    }
}
```

### 3. Health Check Middleware

```csharp
public class HealthCheckMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IHealthCheckService _healthCheckService;

    public HealthCheckMiddleware(RequestDelegate next, IHealthCheckService healthCheckService)
    {
        _next = next;
        _healthCheckService = healthCheckService;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        if (context.Request.Path == "/health")
        {
            var report = await _healthCheckService.CheckHealthAsync();
            context.Response.ContentType = "application/json";
            await context.Response.WriteAsync(JsonSerializer.Serialize(report));
            return;
        }

        await _next(context);
    }
}
```

## Performans Analizi

| Algoritma | En İyi Durum | Ortalama Durum | En Kötü Durum | Bellek Kullanımı |
|-----------|-------------|----------------|---------------|------------------|
| Temel Health Check | O(1) | O(1) | O(1) | O(1) |
| Veritabanı Health Check | O(1) | O(1) | O(n) | O(1) |
| Harici Servis Health Check | O(1) | O(1) | O(n) | O(1) |

## Best Practices

1. Düzenli aralıklarla kontrol et
2. Kritik servisleri izle
3. Hata durumlarını logla
4. Performansı optimize et
5. Güvenliği sağla

## Örnek Uygulamalar

1. Uygulama izleme
2. Servis sağlığı
3. Kaynak kullanımı
4. Performans metrikleri
5. Hata tespiti

## Kaynaklar

- [Health Checks in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks)
- [Health Checks UI](https://github.com/Xabaril/AspNetCore.Diagnostics.HealthChecks)
- [Monitoring and Diagnostics](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/implement-resilient-applications/monitor-app-health) 