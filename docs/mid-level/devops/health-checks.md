# Health Checks

## Giriş

Health Checks, modern software systems'de uygulamanın ve bağımlılıklarının sağlık durumunu sürekli olarak izleyen ve raporlayan mekanizmalardır. Mid-level geliştiriciler için health checks implementasyonu, production environment'larda proactive monitoring ve issue detection için kritik öneme sahiptir. Bu dosya, ASP.NET Core health checks, custom health checks, health check middleware ve health check monitoring konularını kapsar.

## ASP.NET Core Health Checks

### 1. Basic Health Check Setup
ASP.NET Core'da health checks'in temel kurulumu.

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // Add health checks
        services.AddHealthChecks()
            .AddCheck("self", () => HealthCheckResult.Healthy())
            .AddCheck("database", () => CheckDatabaseHealth())
            .AddCheck("redis", () => CheckRedisHealth())
            .AddCheck("external_api", () => CheckExternalApiHealth())
            .AddCheck("disk_space", () => CheckDiskSpaceHealth());
        
        // Add health check UI (optional)
        services.AddHealthChecksUI(setup =>
        {
            setup.SetEvaluationTimeInSeconds(30);
            setup.MaximumHistoryEntriesPerEndpoint(60);
        })
        .AddInMemoryStorage();
    }
    
    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        // Health check endpoint
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapHealthChecks("/health", new HealthCheckOptions
            {
                Predicate = _ => true,
                ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse,
                ResultStatusCodes =
                {
                    [HealthStatus.Healthy] = StatusCodes.Status200OK,
                    [HealthStatus.Degraded] = StatusCodes.Status200OK,
                    [HealthStatus.Unhealthy] = StatusCodes.Status503ServiceUnavailable,
                },
                AllowCachingResponses = false
            });
            
            // Health check UI
            endpoints.MapHealthChecksUI(setup =>
            {
                setup.UIPath = "/health-ui";
                setup.AddCustomStylesheet("./wwwroot/css/health-checks.css");
            });
        });
    }
    
    private HealthCheckResult CheckDatabaseHealth()
    {
        try
        {
            // Implement database health check logic
            return HealthCheckResult.Healthy("Database is accessible");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Database health check failed", ex);
        }
    }
    
    private HealthCheckResult CheckRedisHealth()
    {
        try
        {
            // Implement Redis health check logic
            return HealthCheckResult.Healthy("Redis is accessible");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Redis health check failed", ex);
        }
    }
    
    private HealthCheckResult CheckExternalApiHealth()
    {
        try
        {
            // Implement external API health check logic
            return HealthCheckResult.Healthy("External API is accessible");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("External API health check failed", ex);
        }
    }
    
    private HealthCheckResult CheckDiskSpaceHealth()
    {
        try
        {
            var drive = new DriveInfo(Path.GetPathRoot(Environment.CurrentDirectory));
            var freeSpacePercentage = (double)drive.AvailableFreeSpace / drive.TotalSize * 100;
            
            if (freeSpacePercentage > 20)
            {
                return HealthCheckResult.Healthy($"Disk space is sufficient: {freeSpacePercentage:F1}% free");
            }
            else if (freeSpacePercentage > 10)
            {
                return HealthCheckResult.Degraded($"Disk space is low: {freeSpacePercentage:F1}% free");
            }
            else
            {
                return HealthCheckResult.Unhealthy($"Disk space is critical: {freeSpacePercentage:F1}% free");
            }
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Disk space health check failed", ex);
        }
    }
}
```

### 2. Custom Health Check Implementation
Özel health check'ler için interface ve implementation.

```csharp
public interface ICustomHealthCheck
{
    string Name { get; }
    string Description { get; }
    Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default);
}

public abstract class BaseHealthCheck : ICustomHealthCheck
{
    public abstract string Name { get; }
    public abstract string Description { get; }
    
    protected readonly ILogger Logger;
    protected readonly IConfiguration Configuration;
    
    protected BaseHealthCheck(ILogger logger, IConfiguration configuration)
    {
        Logger = logger;
        Configuration = configuration;
    }
    
    public abstract Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default);
    
    protected HealthCheckResult Success(string message = null, IDictionary<string, object> data = null)
    {
        return HealthCheckResult.Healthy(message, data);
    }
    
    protected HealthCheckResult Warning(string message, IDictionary<string, object> data = null)
    {
        return HealthCheckResult.Degraded(message, data);
    }
    
    protected HealthCheckResult Failure(string message, Exception exception = null, IDictionary<string, object> data = null)
    {
        return HealthCheckResult.Unhealthy(message, exception, data);
    }
    
    protected void LogHealthCheckStart()
    {
        Logger.LogDebug("Starting health check: {HealthCheckName}", Name);
    }
    
    protected void LogHealthCheckSuccess(string message = null)
    {
        Logger.LogDebug("Health check {HealthCheckName} completed successfully: {Message}", Name, message);
    }
    
    protected void LogHealthCheckWarning(string message)
    {
        Logger.LogWarning("Health check {HealthCheckName} completed with warning: {Message}", Name, message);
    }
    
    protected void LogHealthCheckFailure(string message, Exception exception = null)
    {
        Logger.LogError(exception, "Health check {HealthCheckName} failed: {Message}", Name, message);
    }
}

public class DatabaseHealthCheck : BaseHealthCheck
{
    public override string Name => "database";
    public override string Description => "Checks if the database is accessible and responding";
    
    private readonly IDbConnectionFactory _connectionFactory;
    private readonly TimeSpan _timeout;
    
    public DatabaseHealthCheck(
        ILogger<DatabaseHealthCheck> logger,
        IConfiguration configuration,
        IDbConnectionFactory connectionFactory)
        : base(logger, configuration)
    {
        _connectionFactory = connectionFactory;
        _timeout = TimeSpan.FromSeconds(
            configuration.GetValue<int>("HealthChecks:Database:TimeoutSeconds", 10));
    }
    
    public override async Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        LogHealthCheckStart();
        
        try
        {
            using var cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
            cts.CancelAfter(_timeout);
            
            using var connection = await _connectionFactory.CreateConnectionAsync();
            await connection.OpenAsync(cts.Token);
            
            // Test basic query
            using var command = connection.CreateCommand();
            command.CommandText = "SELECT 1";
            command.CommandTimeout = (int)_timeout.TotalSeconds;
            
            var result = await command.ExecuteScalarAsync(cts.Token);
            
            if (result != null && result.ToString() == "1")
            {
                var data = new Dictionary<string, object>
                {
                    ["connection_string"] = MaskConnectionString(connection.ConnectionString),
                    ["database_type"] = connection.GetType().Name,
                    ["timeout"] = _timeout.TotalSeconds
                };
                
                LogHealthCheckSuccess("Database is accessible and responding");
                return Success("Database is accessible and responding", data);
            }
            else
            {
                LogHealthCheckWarning("Database query returned unexpected result");
                return Warning("Database query returned unexpected result");
            }
        }
        catch (OperationCanceledException) when (cancellationToken.IsCancellationRequested)
        {
            LogHealthCheckFailure("Database health check was cancelled");
            return Failure("Database health check was cancelled");
        }
        catch (OperationCanceledException)
        {
            LogHealthCheckFailure("Database health check timed out");
            return Failure("Database health check timed out", data: new Dictionary<string, object>
            {
                ["timeout"] = _timeout.TotalSeconds
            });
        }
        catch (Exception ex)
        {
            LogHealthCheckFailure("Database health check failed", ex);
            return Failure("Database health check failed", ex, new Dictionary<string, object>
            {
                ["error_type"] = ex.GetType().Name,
                ["error_message"] = ex.Message
            });
        }
    }
    
    private string MaskConnectionString(string connectionString)
    {
        if (string.IsNullOrEmpty(connectionString))
            return string.Empty;
        
        // Mask sensitive information in connection string
        var masked = connectionString;
        
        // Mask password
        var passwordMatch = Regex.Match(connectionString, @"Password=([^;]+)", RegexOptions.IgnoreCase);
        if (passwordMatch.Success)
        {
            masked = masked.Replace(passwordMatch.Groups[1].Value, "***");
        }
        
        // Mask user ID
        var userIdMatch = Regex.Match(connectionString, @"User Id=([^;]+)", RegexOptions.IgnoreCase);
        if (userIdMatch.Success)
        {
            masked = masked.Replace(userIdMatch.Groups[1].Value, "***");
        }
        
        return masked;
    }
}

public class RedisHealthCheck : BaseHealthCheck
{
    public override string Name => "redis";
    public override string Description => "Checks if Redis cache is accessible and responding";
    
    private readonly IConnectionMultiplexer _redis;
    private readonly TimeSpan _timeout;
    
    public RedisHealthCheck(
        ILogger<RedisHealthCheck> logger,
        IConfiguration configuration,
        IConnectionMultiplexer redis)
        : base(logger, configuration)
    {
        _redis = redis;
        _timeout = TimeSpan.FromSeconds(
            configuration.GetValue<int>("HealthChecks:Redis:TimeoutSeconds", 5));
    }
    
    public override async Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        LogHealthCheckStart();
        
        try
        {
            using var cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
            cts.CancelAfter(_timeout);
            
            var database = _redis.GetDatabase();
            
            // Test Redis connection with PING command
            var pingResult = await database.PingAsync();
            
            if (pingResult.TotalMilliseconds > 0)
            {
                var data = new Dictionary<string, object>
                {
                    ["ping_time_ms"] = pingResult.TotalMilliseconds,
                    ["endpoint_count"] = _redis.GetEndPoints().Length,
                    ["timeout"] = _timeout.TotalSeconds
                };
                
                LogHealthCheckSuccess($"Redis is accessible (PING: {pingResult.TotalMilliseconds:F2}ms)");
                return Success($"Redis is accessible (PING: {pingResult.TotalMilliseconds:F2}ms)", data);
            }
            else
            {
                LogHealthCheckWarning("Redis PING returned unexpected result");
                return Warning("Redis PING returned unexpected result");
            }
        }
        catch (OperationCanceledException) when (cancellationToken.IsCancellationRequested)
        {
            LogHealthCheckFailure("Redis health check was cancelled");
            return Failure("Redis health check was cancelled");
        }
        catch (OperationCanceledException)
        {
            LogHealthCheckFailure("Redis health check timed out");
            return Failure("Redis health check timed out", data: new Dictionary<string, object>
            {
                ["timeout"] = _timeout.TotalSeconds
            });
        }
        catch (Exception ex)
        {
            LogHealthCheckFailure("Redis health check failed", ex);
            return Failure("Redis health check failed", ex, new Dictionary<string, object>
            {
                ["error_type"] = ex.GetType().Name,
                ["error_message"] = ex.Message
            });
        }
    }
}

public class ExternalApiHealthCheck : BaseHealthCheck
{
    public override string Name => "external_api";
    public override string Description => "Checks if external API is accessible and responding";
    
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly string _apiUrl;
    private readonly TimeSpan _timeout;
    private readonly int _expectedStatusCode;
    
    public ExternalApiHealthCheck(
        ILogger<ExternalApiHealthCheck> logger,
        IConfiguration configuration,
        IHttpClientFactory httpClientFactory)
        : base(logger, configuration)
    {
        _httpClientFactory = httpClientFactory;
        _apiUrl = configuration["HealthChecks:ExternalApi:Url"];
        _timeout = TimeSpan.FromSeconds(
            configuration.GetValue<int>("HealthChecks:ExternalApi:TimeoutSeconds", 10));
        _expectedStatusCode = configuration.GetValue<int>("HealthChecks:ExternalApi:ExpectedStatusCode", 200);
    }
    
    public override async Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        LogHealthCheckStart();
        
        if (string.IsNullOrEmpty(_apiUrl))
        {
            LogHealthCheckWarning("External API URL not configured");
            return Warning("External API URL not configured");
        }
        
        try
        {
            using var cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
            cts.CancelAfter(_timeout);
            
            using var client = _httpClientFactory.CreateClient("HealthCheck");
            client.Timeout = _timeout;
            
            var stopwatch = Stopwatch.StartNew();
            var response = await client.GetAsync(_apiUrl, cts.Token);
            stopwatch.Stop();
            
            var responseTime = stopwatch.ElapsedMilliseconds;
            
            if (response.StatusCode == (HttpStatusCode)_expectedStatusCode)
            {
                var data = new Dictionary<string, object>
                {
                    ["response_time_ms"] = responseTime,
                    ["status_code"] = (int)response.StatusCode,
                    ["timeout"] = _timeout.TotalSeconds,
                    ["url"] = _apiUrl
                };
                
                LogHealthCheckSuccess($"External API is accessible (Response time: {responseTime}ms)");
                return Success($"External API is accessible (Response time: {responseTime}ms)", data);
            }
            else
            {
                var data = new Dictionary<string, object>
                {
                    ["expected_status_code"] = _expectedStatusCode,
                    ["actual_status_code"] = (int)response.StatusCode,
                    ["response_time_ms"] = responseTime,
                    ["url"] = _apiUrl
                };
                
                LogHealthCheckWarning($"External API returned unexpected status code: {response.StatusCode}");
                return Warning($"External API returned unexpected status code: {response.StatusCode}", data);
            }
        }
        catch (OperationCanceledException) when (cancellationToken.IsCancellationRequested)
        {
            LogHealthCheckFailure("External API health check was cancelled");
            return Failure("External API health check was cancelled");
        }
        catch (OperationCanceledException)
        {
            LogHealthCheckFailure("External API health check timed out");
            return Failure("External API health check timed out", data: new Dictionary<string, object>
            {
                ["timeout"] = _timeout.TotalSeconds,
                ["url"] = _apiUrl
            });
        }
        catch (Exception ex)
        {
            LogHealthCheckFailure("External API health check failed", ex);
            return Failure("External API health check failed", ex, new Dictionary<string, object>
            {
                ["error_type"] = ex.GetType().Name,
                ["error_message"] = ex.Message,
                ["url"] = _apiUrl
            });
        }
    }
}
```

## Health Check Middleware

### 1. Custom Health Check Middleware
Health check'leri özelleştiren middleware.

```csharp
public class HealthCheckMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<HealthCheckMiddleware> _logger;
    private readonly IHealthCheckService _healthCheckService;
    private readonly HealthCheckOptions _options;
    
    public HealthCheckMiddleware(
        RequestDelegate next,
        ILogger<HealthCheckMiddleware> logger,
        IHealthCheckService healthCheckService,
        IOptions<HealthCheckOptions> options)
    {
        _next = next;
        _logger = logger;
        _healthCheckService = healthCheckService;
        _options = options.Value;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        if (context.Request.Path.StartsWithSegments("/health"))
        {
            await HandleHealthCheckAsync(context);
            return;
        }
        
        await _next(context);
    }
    
    private async Task HandleHealthCheckAsync(HttpContext context)
    {
        try
        {
            var startTime = DateTime.UtcNow;
            var stopwatch = Stopwatch.StartNew();
            
            // Run health checks
            var healthReport = await _healthCheckService.CheckHealthAsync(
                predicate: _options.Predicate,
                cancellationToken: context.RequestAborted);
            
            stopwatch.Stop();
            
            // Add timing information
            var responseData = new
            {
                status = healthReport.Status.ToString().ToLowerInvariant(),
                totalDuration = healthReport.TotalDuration.TotalMilliseconds,
                checks = healthReport.Entries.Select(e => new
                {
                    name = e.Key,
                    status = e.Value.Status.ToString().ToLowerInvariant(),
                    description = e.Value.Description,
                    duration = e.Value.Duration.TotalMilliseconds,
                    tags = e.Value.Tags,
                    data = e.Value.Data
                }),
                timestamp = startTime,
                responseTime = stopwatch.ElapsedMilliseconds
            };
            
            // Set response status code
            context.Response.StatusCode = GetStatusCode(healthReport.Status);
            context.Response.ContentType = "application/json";
            
            // Add custom headers
            context.Response.Headers.Add("X-Health-Check-Timestamp", startTime.ToString("O"));
            context.Response.Headers.Add("X-Health-Check-Duration", stopwatch.ElapsedMilliseconds.ToString());
            context.Response.Headers.Add("X-Health-Check-Status", healthReport.Status.ToString());
            
            // Write response
            var json = JsonSerializer.Serialize(responseData, new JsonSerializerOptions
            {
                WriteIndented = true,
                PropertyNamingPolicy = JsonNamingPolicy.CamelCase
            });
            
            await context.Response.WriteAsync(json, context.RequestAborted);
            
            // Log health check result
            LogHealthCheckResult(healthReport, stopwatch.ElapsedMilliseconds);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error during health check");
            
            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            context.Response.ContentType = "application/json";
            
            var errorResponse = new
            {
                status = "error",
                message = "Health check failed",
                error = ex.Message,
                timestamp = DateTime.UtcNow
            };
            
            var json = JsonSerializer.Serialize(errorResponse, new JsonSerializerOptions
            {
                WriteIndented = true,
                PropertyNamingPolicy = JsonNamingPolicy.CamelCase
            });
            
            await context.Response.WriteAsync(json, context.RequestAborted);
        }
    }
    
    private int GetStatusCode(HealthStatus status)
    {
        return status switch
        {
            HealthStatus.Healthy => StatusCodes.Status200OK,
            HealthStatus.Degraded => StatusCodes.Status200OK,
            HealthStatus.Unhealthy => StatusCodes.Status503ServiceUnavailable,
            _ => StatusCodes.Status200OK
        };
    }
    
    private void LogHealthCheckResult(HealthReport healthReport, long responseTime)
    {
        var healthyCount = healthReport.Entries.Count(e => e.Value.Status == HealthStatus.Healthy);
        var unhealthyCount = healthReport.Entries.Count(e => e.Value.Status == HealthStatus.Unhealthy);
        var degradedCount = healthReport.Entries.Count(e => e.Value.Status == HealthStatus.Degraded);
        
        if (healthReport.Status == HealthStatus.Healthy)
        {
            _logger.LogInformation("Health check completed successfully. Status: {Status}, " +
                "Healthy: {HealthyCount}, Unhealthy: {UnhealthyCount}, Degraded: {DegradedCount}, " +
                "Total Duration: {TotalDuration}ms, Response Time: {ResponseTime}ms",
                healthReport.Status, healthyCount, unhealthyCount, degradedCount,
                healthReport.TotalDuration.TotalMilliseconds, responseTime);
        }
        else if (healthReport.Status == HealthStatus.Degraded)
        {
            _logger.LogWarning("Health check completed with degraded status. Status: {Status}, " +
                "Healthy: {HealthyCount}, Unhealthy: {UnhealthyCount}, Degraded: {DegradedCount}, " +
                "Total Duration: {TotalDuration}ms, Response Time: {ResponseTime}ms",
                healthReport.Status, healthyCount, unhealthyCount, degradedCount,
                healthReport.TotalDuration.TotalMilliseconds, responseTime);
        }
        else
        {
            _logger.LogError("Health check completed with unhealthy status. Status: {Status}, " +
                "Healthy: {HealthyCount}, Unhealthy: {UnhealthyCount}, Degraded: {DegradedCount}, " +
                "Total Duration: {TotalDuration}ms, Response Time: {ResponseTime}ms",
                healthReport.Status, healthyCount, unhealthyCount, degradedCount,
                healthReport.TotalDuration.TotalMilliseconds, responseTime);
        }
    }
}

public class HealthCheckOptions
{
    public Func<HealthCheckRegistration, bool> Predicate { get; set; } = _ => true;
    public bool AllowCachingResponses { get; set; } = false;
    public TimeSpan CacheExpiration { get; set; } = TimeSpan.FromMinutes(5);
}
```

### 2. Health Check Response Caching
Health check response'larını cache'leyen middleware.

```csharp
public class HealthCheckCachingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<HealthCheckCachingMiddleware> _logger;
    private readonly IMemoryCache _cache;
    private readonly TimeSpan _cacheExpiration;
    
    public HealthCheckCachingMiddleware(
        RequestDelegate next,
        ILogger<HealthCheckCachingMiddleware> logger,
        IMemoryCache cache,
        IConfiguration configuration)
    {
        _next = next;
        _logger = logger;
        _cache = cache;
        _cacheExpiration = TimeSpan.FromSeconds(
            configuration.GetValue<int>("HealthChecks:Caching:ExpirationSeconds", 30));
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        if (context.Request.Path.StartsWithSegments("/health"))
        {
            await HandleCachedHealthCheckAsync(context);
            return;
        }
        
        await _next(context);
    }
    
    private async Task HandleCachedHealthCheckAsync(HttpContext context)
    {
        var cacheKey = $"health_check_{context.Request.QueryString}";
        
        // Check if we have a cached response
        if (_cache.TryGetValue(cacheKey, out CachedHealthCheckResponse cachedResponse))
        {
            // Check if cache is still valid
            if (DateTime.UtcNow < cachedResponse.ExpiresAt)
            {
                _logger.LogDebug("Returning cached health check response. Cache expires at {ExpiresAt}", 
                    cachedResponse.ExpiresAt);
                
                context.Response.StatusCode = cachedResponse.StatusCode;
                context.Response.ContentType = cachedResponse.ContentType;
                
                foreach (var header in cachedResponse.Headers)
                {
                    context.Response.Headers[header.Key] = header.Value;
                }
                
                await context.Response.WriteAsync(cachedResponse.Body, context.RequestAborted);
                return;
            }
            else
            {
                _logger.LogDebug("Cached health check response expired at {ExpiresAt}", cachedResponse.ExpiresAt);
                _cache.Remove(cacheKey);
            }
        }
        
        // Capture the response
        var originalBodyStream = context.Response.Body;
        using var memoryStream = new MemoryStream();
        context.Response.Body = memoryStream;
        
        try
        {
            // Call the next middleware
            await _next(context);
            
            // Read the response
            memoryStream.Position = 0;
            var responseBody = await new StreamReader(memoryStream).ReadToEndAsync();
            
            // Cache the response if it's successful
            if (context.Response.StatusCode == StatusCodes.Status200OK || 
                context.Response.StatusCode == StatusCodes.Status503ServiceUnavailable)
            {
                var cachedResponse = new CachedHealthCheckResponse
                {
                    StatusCode = context.Response.StatusCode,
                    ContentType = context.Response.ContentType,
                    Headers = context.Response.Headers.ToDictionary(h => h.Key, h => h.Value.ToString()),
                    Body = responseBody,
                    ExpiresAt = DateTime.UtcNow.Add(_cacheExpiration)
                };
                
                _cache.Set(cacheKey, cachedResponse, _cacheExpiration);
                
                _logger.LogDebug("Cached health check response. Cache expires at {ExpiresAt}", 
                    cachedResponse.ExpiresAt);
            }
            
            // Copy the response back to the original stream
            memoryStream.Position = 0;
            await memoryStream.CopyToAsync(originalBodyStream);
        }
        finally
        {
            context.Response.Body = originalBodyStream;
        }
    }
}

public class CachedHealthCheckResponse
{
    public int StatusCode { get; set; }
    public string ContentType { get; set; }
    public Dictionary<string, string> Headers { get; set; } = new Dictionary<string, string>();
    public string Body { get; set; }
    public DateTime ExpiresAt { get; set; }
}
```

## Health Check Monitoring

### 1. Health Check Metrics Collection
Health check'lerden metrics toplayan servis.

```csharp
public class HealthCheckMetricsCollector : BackgroundService
{
    private readonly ILogger<HealthCheckMetricsCollector> _logger;
    private readonly IHealthCheckService _healthCheckService;
    private readonly IMetricsService _metricsService;
    private readonly IConfiguration _configuration;
    private readonly Timer _timer;
    
    public HealthCheckMetricsCollector(
        ILogger<HealthCheckMetricsCollector> logger,
        IHealthCheckService healthCheckService,
        IMetricsService metricsService,
        IConfiguration configuration)
    {
        _logger = logger;
        _healthCheckService = healthCheckService;
        _metricsService = metricsService;
        _configuration = configuration;
        
        var interval = _configuration.GetValue<int>("HealthChecks:Metrics:CollectionIntervalSeconds", 60);
        _timer = new Timer(CollectMetrics, null, TimeSpan.Zero, TimeSpan.FromSeconds(interval));
    }
    
    protected override Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // Service is managed by timer
        return Task.CompletedTask;
    }
    
    private async void CollectMetrics(object state)
    {
        try
        {
            await CollectHealthCheckMetricsAsync();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error collecting health check metrics");
        }
    }
    
    private async Task CollectHealthCheckMetricsAsync()
    {
        try
        {
            var startTime = DateTime.UtcNow;
            var stopwatch = Stopwatch.StartNew();
            
            // Run health checks
            var healthReport = await _healthCheckService.CheckHealthAsync();
            stopwatch.Stop();
            
            // Record overall metrics
            await RecordOverallMetricsAsync(healthReport, stopwatch.ElapsedMilliseconds);
            
            // Record individual check metrics
            await RecordIndividualCheckMetricsAsync(healthReport);
            
            // Record trend metrics
            await RecordTrendMetricsAsync(healthReport, startTime);
            
            _logger.LogDebug("Health check metrics collected successfully. Duration: {Duration}ms", 
                stopwatch.ElapsedMilliseconds);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error collecting health check metrics");
            
            // Record failure metric
            await _metricsService.RecordCounterAsync("health_check_metrics_collection_failures", 1);
        }
    }
    
    private async Task RecordOverallMetricsAsync(HealthReport healthReport, long responseTime)
    {
        var totalChecks = healthReport.Entries.Count;
        var healthyChecks = healthReport.Entries.Count(e => e.Value.Status == HealthStatus.Healthy);
        var unhealthyChecks = healthReport.Entries.Count(e => e.Value.Status == HealthStatus.Unhealthy);
        var degradedChecks = healthReport.Entries.Count(e => e.Value.Status == HealthStatus.Degraded);
        
        // Record counts
        await _metricsService.RecordGaugeAsync("health_checks_total", totalChecks);
        await _metricsService.RecordGaugeAsync("health_checks_healthy", healthyChecks);
        await _metricsService.RecordGaugeAsync("health_checks_unhealthy", unhealthyChecks);
        await _metricsService.RecordGaugeAsync("health_checks_degraded", degradedChecks);
        
        // Record percentages
        var healthyPercentage = totalChecks > 0 ? (double)healthyChecks / totalChecks * 100 : 0;
        var unhealthyPercentage = totalChecks > 0 ? (double)unhealthyChecks / totalChecks * 100 : 0;
        var degradedPercentage = totalChecks > 0 ? (double)degradedChecks / totalChecks * 100 : 0;
        
        await _metricsService.RecordGaugeAsync("health_checks_healthy_percentage", healthyPercentage);
        await _metricsService.RecordGaugeAsync("health_checks_unhealthy_percentage", unhealthyPercentage);
        await _metricsService.RecordGaugeAsync("health_checks_degraded_percentage", degradedPercentage);
        
        // Record overall status
        var overallStatus = healthReport.Status == HealthStatus.Healthy ? 1 : 0;
        await _metricsService.RecordGaugeAsync("health_checks_overall_status", overallStatus);
        
        // Record response time
        await _metricsService.RecordHistogramAsync("health_checks_response_time", responseTime);
        
        // Record total duration
        await _metricsService.RecordHistogramAsync("health_checks_total_duration", 
            healthReport.TotalDuration.TotalMilliseconds);
    }
    
    private async Task RecordIndividualCheckMetricsAsync(HealthReport healthReport)
    {
        foreach (var entry in healthReport.Entries)
        {
            var checkName = entry.Key;
            var check = entry.Value;
            
            // Record individual check status
            var status = check.Status == HealthStatus.Healthy ? 1 : 0;
            await _metricsService.RecordGaugeAsync("health_check_status", status, 
                new Dictionary<string, object> { ["check_name"] = checkName });
            
            // Record individual check duration
            await _metricsService.RecordHistogramAsync("health_check_duration", 
                check.Duration.TotalMilliseconds,
                new Dictionary<string, object> { ["check_name"] = checkName });
            
            // Record individual check tags
            foreach (var tag in check.Tags)
            {
                await _metricsService.RecordGaugeAsync("health_check_tag", 1,
                    new Dictionary<string, object> 
                    { 
                        ["check_name"] = checkName,
                        ["tag"] = tag 
                    });
            }
        }
    }
    
    private async Task RecordTrendMetricsAsync(HealthReport healthReport, DateTime timestamp)
    {
        // Record time-based metrics
        var hourOfDay = timestamp.Hour;
        var dayOfWeek = (int)timestamp.DayOfWeek;
        
        await _metricsService.RecordGaugeAsync("health_checks_by_hour", 
            healthReport.Entries.Count(e => e.Value.Status == HealthStatus.Healthy),
            new Dictionary<string, object> { ["hour"] = hourOfDay });
        
        await _metricsService.RecordGaugeAsync("health_checks_by_day", 
            healthReport.Entries.Count(e => e.Value.Status == HealthStatus.Healthy),
            new Dictionary<string, object> { ["day"] = dayOfWeek });
        
        // Record consecutive failures for critical checks
        var criticalChecks = healthReport.Entries
            .Where(e => IsCriticalHealthCheck(e.Key))
            .ToList();
        
        foreach (var check in criticalChecks)
        {
            if (check.Value.Status == HealthStatus.Unhealthy)
            {
                var failureKey = $"consecutive_failures_{check.Key}";
                var currentFailures = await GetConsecutiveFailuresAsync(failureKey);
                var newFailures = currentFailures + 1;
                
                await SetConsecutiveFailuresAsync(failureKey, newFailures);
                
                await _metricsService.RecordGaugeAsync("health_check_consecutive_failures", newFailures,
                    new Dictionary<string, object> { ["check_name"] = check.Key });
            }
            else
            {
                // Reset consecutive failures on success
                var failureKey = $"consecutive_failures_{check.Key}";
                await SetConsecutiveFailuresAsync(failureKey, 0);
                
                await _metricsService.RecordGaugeAsync("health_check_consecutive_failures", 0,
                    new Dictionary<string, object> { ["check_name"] = check.Key });
            }
        }
    }
    
    private bool IsCriticalHealthCheck(string checkName)
    {
        var criticalChecks = new[]
        {
            "database",
            "redis",
            "external_api"
        };
        
        return criticalChecks.Any(check => checkName.Contains(check, StringComparison.OrdinalIgnoreCase));
    }
    
    private async Task<int> GetConsecutiveFailuresAsync(string key)
    {
        // Implementation to get consecutive failures from cache or database
        return await Task.FromResult(0);
    }
    
    private async Task SetConsecutiveFailuresAsync(string key, int value)
    {
        // Implementation to set consecutive failures in cache or database
        await Task.CompletedTask;
    }
    
    public override void Dispose()
    {
        _timer?.Dispose();
        base.Dispose();
    }
}
```

## Mülakat Soruları

### Temel Sorular

1. **Health Checks nedir ve neden önemlidir?**
   - **Cevap**: Uygulama ve bağımlılıklarının sağlık durumunu izleyen mekanizmalar. Proactive monitoring, issue detection, load balancer integration için kritik.

2. **ASP.NET Core'da health checks nasıl implement edilir?**
   - **Cevap**: `AddHealthChecks()`, custom health checks, health check endpoints, health check UI.

3. **Health check status'ları nelerdir?**
   - **Cevap**: Healthy (sağlıklı), Unhealthy (sağlıksız), Degraded (bozulmuş).

4. **Health check caching neden kullanılır?**
   - **Cevap**: Performance improvement, reduced load on dependencies, faster response times.

5. **Health check metrics neden toplanır?**
   - **Cevap**: Trend analysis, alerting, performance monitoring, capacity planning.

### Teknik Sorular

1. **Custom health check nasıl implement edilir?**
   - **Cevap**: `IHealthCheck` interface implement, custom logic, error handling, timeout management.

2. **Health check middleware nasıl özelleştirilir?**
   - **Cevap**: Custom middleware, response formatting, custom headers, logging.

3. **Health check response caching nasıl implement edilir?**
   - **Cevap**: Memory cache, response capture, cache invalidation, performance optimization.

4. **Health check metrics collection nasıl yapılır?**
   - **Cevap**: Background service, metrics aggregation, trend analysis, alerting integration.

5. **Health check timeout management nasıl yapılır?**
   - **Cevap**: CancellationToken, timeout configuration, graceful degradation, error handling.

## Best Practices

1. **Health Check Implementation**
   - Comprehensive health checks implement edin
   - Timeout management yapın
   - Error handling implement edin
   - Performance impact minimize edin

2. **Health Check Configuration**
   - Environment-specific configuration kullanın
   - Configurable timeouts sağlayın
   - Health check dependencies minimize edin
   - Graceful degradation implement edin

3. **Health Check Monitoring**
   - Metrics collection yapın
   - Trend analysis implement edin
   - Alerting kurun
   - Performance monitoring sağlayın

4. **Health Check Security**
   - Authentication implement edin
   - Rate limiting yapın
   - Sensitive information expose etmeyin
   - Access control sağlayın

5. **Health Check Performance**
   - Response caching kullanın
   - Async operations implement edin
   - Resource usage optimize edin
   - Load testing yapın

## Kaynaklar

- [ASP.NET Core Health Checks](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks)
- [Health Check UI](https://github.com/Xabaril/AspNetCore.Diagnostics.HealthChecks)
- [Health Check Best Practices](https://microservices.io/patterns/observability/health-check-api.html)
- [Health Check Monitoring](https://prometheus.io/docs/guides/health-check/)
- [Health Check Patterns](https://docs.aws.amazon.com/whitepapers/latest/implementing-health-checks-for-aws-elastic-beanstalk-environments/health-check-patterns.html)
