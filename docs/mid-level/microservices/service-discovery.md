# Service Discovery (Servis Keşfi)

## Genel Bakış
Service Discovery, mikroservis mimarisinde servislerin birbirlerini dinamik olarak bulabilmesini sağlayan bir mekanizmadır. Servislerin konumlarını ve durumlarını merkezi olarak yönetir.

## Temel Özellikler

### 1. Servis Kaydı
```csharp
// Consul Service Registration
public class ConsulServiceRegistration
{
    private readonly IConsulClient _consulClient;
    private readonly IConfiguration _configuration;

    public ConsulServiceRegistration(
        IConsulClient consulClient,
        IConfiguration configuration)
    {
        _consulClient = consulClient;
        _configuration = configuration;
    }

    public async Task RegisterServiceAsync()
    {
        var serviceId = $"{_configuration["Service:Name"]}-{Guid.NewGuid()}";
        var serviceRegistration = new AgentServiceRegistration
        {
            ID = serviceId,
            Name = _configuration["Service:Name"],
            Address = _configuration["Service:Host"],
            Port = int.Parse(_configuration["Service:Port"]),
            Check = new AgentServiceCheck
            {
                HTTP = $"http://{_configuration["Service:Host"]}:{_configuration["Service:Port"]}/health",
                Interval = TimeSpan.FromSeconds(10),
                Timeout = TimeSpan.FromSeconds(5),
                DeregisterCriticalServiceAfter = TimeSpan.FromMinutes(1)
            }
        };

        await _consulClient.Agent.ServiceRegister(serviceRegistration);
    }
}

// Health Check Endpoint
public class HealthCheckController : ControllerBase
{
    [HttpGet("health")]
    public IActionResult Check()
    {
        return Ok(new { status = "healthy" });
    }
}
```

### 2. Servis Keşfi
```csharp
// Consul Service Discovery
public class ConsulServiceDiscovery
{
    private readonly IConsulClient _consulClient;

    public ConsulServiceDiscovery(IConsulClient consulClient)
    {
        _consulClient = consulClient;
    }

    public async Task<string> GetServiceUrlAsync(string serviceName)
    {
        var services = await _consulClient.Health.Service(serviceName);
        if (!services.Response.Any())
        {
            throw new ServiceNotFoundException(serviceName);
        }

        var service = services.Response.First();
        return $"http://{service.Service.Address}:{service.Service.Port}";
    }
}

// Service Discovery Middleware
public class ServiceDiscoveryMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IServiceDiscovery _serviceDiscovery;

    public ServiceDiscoveryMiddleware(
        RequestDelegate next,
        IServiceDiscovery serviceDiscovery)
    {
        _next = next;
        _serviceDiscovery = serviceDiscovery;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var serviceName = context.Request.Headers["X-Service-Name"];
        if (!string.IsNullOrEmpty(serviceName))
        {
            try
            {
                var serviceUrl = await _serviceDiscovery.GetServiceUrlAsync(serviceName);
                context.Request.Headers["X-Service-Url"] = serviceUrl;
            }
            catch (ServiceNotFoundException)
            {
                context.Response.StatusCode = 503;
                return;
            }
        }

        await _next(context);
    }
}
```

### 3. Health Checking
```csharp
// Health Check Service
public class HealthCheckService : IHealthCheckService
{
    private readonly IEnumerable<IHealthCheck> _healthChecks;

    public HealthCheckService(IEnumerable<IHealthCheck> healthChecks)
    {
        _healthChecks = healthChecks;
    }

    public async Task<HealthCheckResult> CheckHealthAsync()
    {
        var results = new List<HealthCheckResult>();
        
        foreach (var healthCheck in _healthChecks)
        {
            try
            {
                var result = await healthCheck.CheckHealthAsync();
                results.Add(result);
            }
            catch (Exception ex)
            {
                results.Add(new HealthCheckResult(
                    healthCheck.Name,
                    HealthStatus.Unhealthy,
                    ex.Message));
            }
        }

        var status = results.All(r => r.Status == HealthStatus.Healthy)
            ? HealthStatus.Healthy
            : HealthStatus.Unhealthy;

        return new HealthCheckResult(status, results);
    }
}

// Database Health Check
public class DatabaseHealthCheck : IHealthCheck
{
    private readonly IDbConnection _dbConnection;

    public DatabaseHealthCheck(IDbConnection dbConnection)
    {
        _dbConnection = dbConnection;
    }

    public async Task<HealthCheckResult> CheckHealthAsync()
    {
        try
        {
            await _dbConnection.OpenAsync();
            await _dbConnection.CloseAsync();
            return new HealthCheckResult("Database", HealthStatus.Healthy);
        }
        catch (Exception ex)
        {
            return new HealthCheckResult(
                "Database",
                HealthStatus.Unhealthy,
                ex.Message);
        }
    }
}
```

### 4. Load Balancing
```csharp
// Load Balancer
public class RoundRobinLoadBalancer : ILoadBalancer
{
    private readonly IServiceDiscovery _serviceDiscovery;
    private readonly ConcurrentDictionary<string, int> _serviceIndices = new();

    public RoundRobinLoadBalancer(IServiceDiscovery serviceDiscovery)
    {
        _serviceDiscovery = serviceDiscovery;
    }

    public async Task<string> GetServiceUrlAsync(string serviceName)
    {
        var services = await _serviceDiscovery.GetServicesAsync(serviceName);
        if (!services.Any())
        {
            throw new ServiceNotFoundException(serviceName);
        }

        var index = _serviceIndices.AddOrUpdate(
            serviceName,
            0,
            (_, current) => (current + 1) % services.Count);

        return services[index];
    }
}

// Load Balancing Middleware
public class LoadBalancingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILoadBalancer _loadBalancer;

    public LoadBalancingMiddleware(
        RequestDelegate next,
        ILoadBalancer loadBalancer)
    {
        _next = next;
        _loadBalancer = loadBalancer;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var serviceName = context.Request.Headers["X-Service-Name"];
        if (!string.IsNullOrEmpty(serviceName))
        {
            try
            {
                var serviceUrl = await _loadBalancer.GetServiceUrlAsync(serviceName);
                context.Request.Headers["X-Service-Url"] = serviceUrl;
            }
            catch (ServiceNotFoundException)
            {
                context.Response.StatusCode = 503;
                return;
            }
        }

        await _next(context);
    }
}
```

## Best Practices

### 1. Servis Kaydı
- Otomatik kayıt yapın
- Health check ekleyin
- Deregistration yapın
- Metadata ekleyin

### 2. Servis Keşfi
- Caching kullanın
- Fallback mekanizması ekleyin
- Timeout değerlerini ayarlayın
- Retry mekanizması ekleyin

### 3. Health Checking
- Düzenli kontrol yapın
- Kritik servisleri izleyin
- Detaylı raporlama yapın
- Alerting ekleyin

### 4. Load Balancing
- Farklı stratejiler kullanın
- Session affinity uygulayın
- Circuit breaker ekleyin
- Monitoring yapın

## Sık Sorulan Sorular

### 1. Service Discovery neden önemlidir?
- Dinamik servis bulmayı sağlar
- Yük dengelemeyi kolaylaştırır
- Servis izolasyonunu sağlar
- Scaling'i kolaylaştırır

### 2. Hangi Service Discovery çözümleri kullanılabilir?
- Consul
- Eureka
- etcd
- ZooKeeper
- Kubernetes Service Discovery

### 3. Service Discovery'de hangi güvenlik önlemleri alınmalıdır?
- SSL/TLS kullanılmalı
- Authentication uygulanmalı
- Authorization yapılmalı
- Rate limiting uygulanmalı

## Kaynaklar
- [Microsoft Service Discovery](https://docs.microsoft.com/tr-tr/dotnet/architecture/microservices/architect-microservice-container-applications/service-discovery)
- [Consul Documentation](https://www.consul.io/docs)
- [Eureka Documentation](https://github.com/Netflix/eureka/wiki)
- [Kubernetes Service Discovery](https://kubernetes.io/docs/concepts/services-networking/service-discovery/) 