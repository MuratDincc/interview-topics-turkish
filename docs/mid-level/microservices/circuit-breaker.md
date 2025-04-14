# Circuit Breaker

## Genel Bakış
Circuit Breaker, mikroservis mimarisinde hata toleransını artırmak için kullanılan bir tasarım desenidir. Bir servis başarısız olduğunda, belirli bir süre boyunca istekleri engelleyerek sistemin daha fazla hasar görmesini önler.

## Temel Özellikler

### 1. Circuit Breaker State Machine
```csharp
public enum CircuitBreakerState
{
    Closed,
    Open,
    HalfOpen
}

public class CircuitBreaker
{
    private readonly int _failureThreshold;
    private readonly TimeSpan _resetTimeout;
    private readonly ILogger<CircuitBreaker> _logger;

    private CircuitBreakerState _state = CircuitBreakerState.Closed;
    private int _failureCount = 0;
    private DateTime _lastFailureTime;
    private readonly object _lock = new object();

    public CircuitBreaker(
        int failureThreshold,
        TimeSpan resetTimeout,
        ILogger<CircuitBreaker> logger)
    {
        _failureThreshold = failureThreshold;
        _resetTimeout = resetTimeout;
        _logger = logger;
    }

    public async Task<T> ExecuteAsync<T>(Func<Task<T>> action)
    {
        if (_state == CircuitBreakerState.Open)
        {
            if (DateTime.UtcNow - _lastFailureTime >= _resetTimeout)
            {
                _state = CircuitBreakerState.HalfOpen;
            }
            else
            {
                throw new CircuitBreakerOpenException();
            }
        }

        try
        {
            var result = await action();
            OnSuccess();
            return result;
        }
        catch (Exception ex)
        {
            OnFailure();
            throw;
        }
    }

    private void OnSuccess()
    {
        lock (_lock)
        {
            _state = CircuitBreakerState.Closed;
            _failureCount = 0;
        }
    }

    private void OnFailure()
    {
        lock (_lock)
        {
            _failureCount++;
            _lastFailureTime = DateTime.UtcNow;

            if (_failureCount >= _failureThreshold)
            {
                _state = CircuitBreakerState.Open;
                _logger.LogWarning("Circuit breaker opened");
            }
        }
    }
}
```

### 2. Polly ile Circuit Breaker
```csharp
// Circuit Breaker Policy
public class CircuitBreakerPolicy
{
    private readonly IAsyncPolicy<HttpResponseMessage> _policy;

    public CircuitBreakerPolicy()
    {
        _policy = Policy<HttpResponseMessage>
            .Handle<HttpRequestException>()
            .OrResult(r => !r.IsSuccessStatusCode)
            .CircuitBreakerAsync(
                exceptionsAllowedBeforeBreaking: 3,
                durationOfBreak: TimeSpan.FromSeconds(30),
                onBreak: (ex, duration) =>
                {
                    // Circuit açıldığında yapılacak işlemler
                },
                onReset: () =>
                {
                    // Circuit kapandığında yapılacak işlemler
                });
    }

    public async Task<HttpResponseMessage> ExecuteAsync(
        Func<Task<HttpResponseMessage>> action)
    {
        return await _policy.ExecuteAsync(action);
    }
}

// HttpClient Factory ile Kullanım
public class ResilientHttpClient
{
    private readonly HttpClient _httpClient;
    private readonly CircuitBreakerPolicy _circuitBreaker;

    public ResilientHttpClient(
        HttpClient httpClient,
        CircuitBreakerPolicy circuitBreaker)
    {
        _httpClient = httpClient;
        _circuitBreaker = circuitBreaker;
    }

    public async Task<HttpResponseMessage> GetAsync(string url)
    {
        return await _circuitBreaker.ExecuteAsync(
            () => _httpClient.GetAsync(url));
    }
}
```

### 3. Fallback Mekanizması
```csharp
// Fallback Policy
public class FallbackPolicy
{
    private readonly IAsyncPolicy<HttpResponseMessage> _policy;

    public FallbackPolicy()
    {
        _policy = Policy<HttpResponseMessage>
            .Handle<Exception>()
            .FallbackAsync(
                fallbackAction: async (context) =>
                {
                    // Fallback yanıtı döndür
                    return new HttpResponseMessage(HttpStatusCode.OK)
                    {
                        Content = new StringContent("Fallback response")
                    };
                },
                onFallbackAsync: async (response, context) =>
                {
                    // Fallback durumunda yapılacak işlemler
                });
    }

    public async Task<HttpResponseMessage> ExecuteAsync(
        Func<Task<HttpResponseMessage>> action)
    {
        return await _policy.ExecuteAsync(action);
    }
}

// Circuit Breaker ve Fallback Birlikte
public class ResilientService
{
    private readonly CircuitBreakerPolicy _circuitBreaker;
    private readonly FallbackPolicy _fallback;

    public ResilientService(
        CircuitBreakerPolicy circuitBreaker,
        FallbackPolicy fallback)
    {
        _circuitBreaker = circuitBreaker;
        _fallback = fallback;
    }

    public async Task<HttpResponseMessage> GetDataAsync()
    {
        return await _fallback.ExecuteAsync(
            () => _circuitBreaker.ExecuteAsync(
                () => _httpClient.GetAsync("api/data")));
    }
}
```

### 4. Monitoring ve Logging
```csharp
// Circuit Breaker Monitor
public class CircuitBreakerMonitor
{
    private readonly ILogger<CircuitBreakerMonitor> _logger;
    private readonly IMetricsCollector _metrics;

    public CircuitBreakerMonitor(
        ILogger<CircuitBreakerMonitor> logger,
        IMetricsCollector metrics)
    {
        _logger = logger;
        _metrics = metrics;
    }

    public void OnCircuitOpened(string serviceName)
    {
        _logger.LogWarning(
            "Circuit breaker opened for service {ServiceName}",
            serviceName);
        
        _metrics.IncrementCounter(
            "circuit_breaker_opened",
            new Dictionary<string, string>
            {
                { "service", serviceName }
            });
    }

    public void OnCircuitClosed(string serviceName)
    {
        _logger.LogInformation(
            "Circuit breaker closed for service {ServiceName}",
            serviceName);
        
        _metrics.IncrementCounter(
            "circuit_breaker_closed",
            new Dictionary<string, string>
            {
                { "service", serviceName }
            });
    }
}
```

## Best Practices

### 1. Circuit Breaker Yapılandırması
- Uygun eşik değerleri belirleyin
- Timeout sürelerini ayarlayın
- Half-open state kullanın
- Monitoring ekleyin

### 2. Fallback Stratejisi
- Anlamlı fallback yanıtları döndürün
- Cache kullanın
- Stale data yönetin
- Graceful degradation uygulayın

### 3. Monitoring
- Circuit durumlarını izleyin
- Metrikleri toplayın
- Alerting yapılandırın
- Logging yapın

### 4. Testing
- Failure senaryolarını test edin
- Timeout senaryolarını test edin
- Recovery senaryolarını test edin
- Load testing yapın

## Sık Sorulan Sorular

### 1. Circuit Breaker neden önemlidir?
- Hata toleransını artırır
- Sistem kaynaklarını korur
- Kaskad hataları önler
- Recovery sürecini yönetir

### 2. Circuit Breaker ne zaman kullanılmalıdır?
- Dış servis çağrılarında
- Kritik işlemlerde
- Yüksek yük altında
- Network bağlantılarında

### 3. Circuit Breaker'da hangi parametreler ayarlanmalıdır?
- Failure threshold
- Reset timeout
- Half-open timeout
- Sampling duration

## Kaynaklar
- [Microsoft Circuit Breaker Pattern](https://docs.microsoft.com/tr-tr/azure/architecture/patterns/circuit-breaker)
- [Polly Documentation](https://github.com/App-vNext/Polly)
- [Resilience Patterns](https://docs.microsoft.com/tr-tr/dotnet/architecture/microservices/implement-resilient-applications/)
- [Circuit Breaker Best Practices](https://docs.microsoft.com/tr-tr/azure/architecture/patterns/circuit-breaker#considerations) 