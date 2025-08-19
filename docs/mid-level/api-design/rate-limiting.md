# API Rate Limiting

## Giriş

API Rate Limiting, API'lerin kötüye kullanımını önlemek, adil kullanım sağlamak ve sistem kaynaklarını korumak için kullanılan kritik bir güvenlik ve performans mekanizmasıdır. Mid-level geliştiriciler için rate limiting stratejilerini anlamak ve implement etmek önemlidir.

## Rate Limiting Nedir?

Rate Limiting, belirli bir zaman diliminde bir client'tan gelen istek sayısını sınırlayan mekanizmadır. Bu sayede:

- **API Abuse Prevention**: Kötü niyetli kullanıcıların sistemi aşırı yüklemesi önlenir
- **Resource Protection**: Sunucu kaynakları korunur
- **Fair Usage**: Tüm kullanıcılar için adil kullanım sağlanır
- **Cost Control**: API maliyetleri kontrol altında tutulur

## Rate Limiting Stratejileri

### 1. Fixed Window Counter
```csharp
public class FixedWindowRateLimiter : IRateLimiter
{
    private readonly IDictionary<string, WindowInfo> _windows;
    private readonly int _maxRequests;
    private readonly TimeSpan _windowSize;
    
    public FixedWindowRateLimiter(int maxRequests, TimeSpan windowSize)
    {
        _maxRequests = maxRequests;
        _windowSize = windowSize;
        _windows = new Dictionary<string, WindowInfo>();
    }
    
    public async Task<bool> IsAllowedAsync(string clientId)
    {
        var now = DateTime.UtcNow;
        var windowStart = new DateTime(now.Ticks - (now.Ticks % _windowSize.Ticks));
        
        if (!_windows.TryGetValue(clientId, out var window) || 
            window.WindowStart < windowStart)
        {
            // Yeni window başlat
            window = new WindowInfo { WindowStart = windowStart, RequestCount = 0 };
            _windows[clientId] = window;
        }
        
        if (window.RequestCount >= _maxRequests)
        {
            return false; // Rate limit exceeded
        }
        
        window.RequestCount++;
        return true;
    }
}

public class WindowInfo
{
    public DateTime WindowStart { get; set; }
    public int RequestCount { get; set; }
}
```

### 2. Sliding Window Counter
```csharp
public class SlidingWindowRateLimiter : IRateLimiter
{
    private readonly IDictionary<string, Queue<DateTime>> _requestTimestamps;
    private readonly int _maxRequests;
    private readonly TimeSpan _windowSize;
    
    public SlidingWindowRateLimiter(int maxRequests, TimeSpan windowSize)
    {
        _maxRequests = maxRequests;
        _windowSize = windowSize;
        _requestTimestamps = new Dictionary<string, Queue<DateTime>>();
    }
    
    public async Task<bool> IsAllowedAsync(string clientId)
    {
        var now = DateTime.UtcNow;
        var cutoff = now - _windowSize;
        
        if (!_requestTimestamps.TryGetValue(clientId, out var timestamps))
        {
            timestamps = new Queue<DateTime>();
            _requestTimestamps[clientId] = timestamps;
        }
        
        // Eski timestamp'leri temizle
        while (timestamps.Count > 0 && timestamps.Peek() < cutoff)
        {
            timestamps.Dequeue();
        }
        
        if (timestamps.Count >= _maxRequests)
        {
            return false; // Rate limit exceeded
        }
        
        timestamps.Enqueue(now);
        return true;
    }
}
```

### 3. Token Bucket Algorithm
```csharp
public class TokenBucketRateLimiter : IRateLimiter
{
    private readonly IDictionary<string, TokenBucket> _buckets;
    private readonly int _capacity;
    private readonly double _refillRate; // tokens per second
    
    public TokenBucketRateLimiter(int capacity, double refillRate)
    {
        _capacity = capacity;
        _refillRate = refillRate;
        _buckets = new Dictionary<string, TokenBucket>();
    }
    
    public async Task<bool> IsAllowedAsync(string clientId)
    {
        var now = DateTime.UtcNow;
        
        if (!_buckets.TryGetValue(clientId, out var bucket))
        {
            bucket = new TokenBucket { Tokens = _capacity, LastRefill = now };
            _buckets[clientId] = bucket;
        }
        
        // Token'ları yenile
        var timePassed = (now - bucket.LastRefill).TotalSeconds;
        var tokensToAdd = timePassed * _refillRate;
        bucket.Tokens = Math.Min(_capacity, bucket.Tokens + tokensToAdd);
        bucket.LastRefill = now;
        
        if (bucket.Tokens < 1)
        {
            return false; // Rate limit exceeded
        }
        
        bucket.Tokens--;
        return true;
    }
}

public class TokenBucket
{
    public double Tokens { get; set; }
    public DateTime LastRefill { get; set; }
}
```

## ASP.NET Core Middleware Implementation

### Rate Limiting Middleware
```csharp
public class RateLimitingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IRateLimiter _rateLimiter;
    private readonly ILogger<RateLimitingMiddleware> _logger;
    
    public RateLimitingMiddleware(
        RequestDelegate next, 
        IRateLimiter rateLimiter,
        ILogger<RateLimitingMiddleware> logger)
    {
        _next = next;
        _rateLimiter = rateLimiter;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var clientId = GetClientId(context);
        
        if (await _rateLimiter.IsAllowedAsync(clientId))
        {
            await _next(context);
        }
        else
        {
            _logger.LogWarning("Rate limit exceeded for client: {ClientId}", clientId);
            
            context.Response.StatusCode = 429; // Too Many Requests
            context.Response.Headers.Add("Retry-After", "60");
            
            var response = new
            {
                Error = "Rate limit exceeded",
                Message = "Too many requests. Please try again later.",
                RetryAfter = 60
            };
            
            await context.Response.WriteAsJsonAsync(response);
        }
    }
    
    private string GetClientId(HttpContext context)
    {
        // API key'den client ID al
        if (context.Request.Headers.TryGetValue("X-API-Key", out var apiKey))
        {
            return apiKey.ToString();
        }
        
        // IP adresinden client ID al
        var ipAddress = context.Connection.RemoteIpAddress?.ToString() ?? "unknown";
        return ipAddress;
    }
}

// Extension method
public static class RateLimitingMiddlewareExtensions
{
    public static IApplicationBuilder UseRateLimiting(this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<RateLimitingMiddleware>();
    }
}
```

### Program.cs'de Kullanım
```csharp
var builder = WebApplication.CreateBuilder(args);

// Rate limiting services
builder.Services.AddSingleton<IRateLimiter>(provider =>
{
    var configuration = provider.GetRequiredService<IConfiguration>();
    var maxRequests = configuration.GetValue<int>("RateLimiting:MaxRequests", 100);
    var windowSize = configuration.GetValue<int>("RateLimiting:WindowSizeSeconds", 60);
    
    return new SlidingWindowRateLimiter(maxRequests, TimeSpan.FromSeconds(windowSize));
});

var app = builder.Build();

// Rate limiting middleware
app.UseRateLimiting();

app.Run();
```

## Distributed Rate Limiting

### Redis-based Rate Limiter
```csharp
public class RedisRateLimiter : IRateLimiter
{
    private readonly IConnectionMultiplexer _redis;
    private readonly string _keyPrefix;
    private readonly int _maxRequests;
    private readonly TimeSpan _windowSize;
    
    public RedisRateLimiter(
        IConnectionMultiplexer redis, 
        int maxRequests, 
        TimeSpan windowSize,
        string keyPrefix = "rate_limit")
    {
        _redis = redis;
        _maxRequests = maxRequests;
        _windowSize = windowSize;
        _keyPrefix = keyPrefix;
    }
    
    public async Task<bool> IsAllowedAsync(string clientId)
    {
        var db = _redis.GetDatabase();
        var key = $"{_keyPrefix}:{clientId}";
        var now = DateTimeOffset.UtcNow.ToUnixTimeSeconds();
        var windowStart = now - (long)_windowSize.TotalSeconds;
        
        // Lua script ile atomic operation
        var script = @"
            local key = KEYS[1]
            local window_start = tonumber(ARGV[1])
            local max_requests = tonumber(ARGV[2])
            local current_time = tonumber(ARGV[3])
            
            -- Eski timestamp'leri temizle
            redis.call('ZREMRANGEBYSCORE', key, '-inf', window_start)
            
            -- Mevcut request sayısını al
            local current_count = redis.call('ZCARD', key)
            
            if current_count >= max_requests then
                return 0
            end
            
            -- Yeni request'i ekle
            redis.call('ZADD', key, current_time, current_time .. '-' .. math.random())
            
            -- TTL ayarla
            redis.call('EXPIRE', key, 3600)
            
            return 1
        ";
        
        var result = await db.ScriptEvaluateAsync(script, 
            new RedisKey[] { key }, 
            new RedisValue[] { windowStart, _maxRequests, now });
        
        return (int)result == 1;
    }
}
```

### Configuration
```json
{
  "RateLimiting": {
    "MaxRequests": 100,
    "WindowSizeSeconds": 60,
    "Redis": {
      "ConnectionString": "localhost:6379",
      "KeyPrefix": "api_rate_limit"
    }
  }
}
```

## Rate Limiting Policies

### Different Policies for Different Endpoints
```csharp
public class RateLimitingPolicy
{
    public string Endpoint { get; set; }
    public int MaxRequests { get; set; }
    public TimeSpan WindowSize { get; set; }
    public string ClientType { get; set; } // "free", "premium", "enterprise"
}

public class PolicyBasedRateLimiter : IRateLimiter
{
    private readonly IDictionary<string, IRateLimiter> _limiters;
    private readonly IDictionary<string, RateLimitingPolicy> _policies;
    
    public PolicyBasedRateLimiter(IEnumerable<RateLimitingPolicy> policies)
    {
        _policies = policies.ToDictionary(p => p.Endpoint);
        _limiters = new Dictionary<string, IRateLimiter>();
        
        foreach (var policy in policies)
        {
            _limiters[policy.Endpoint] = new SlidingWindowRateLimiter(
                policy.MaxRequests, 
                policy.WindowSize);
        }
    }
    
    public async Task<bool> IsAllowedAsync(string clientId, string endpoint)
    {
        if (!_limiters.TryGetValue(endpoint, out var limiter))
        {
            return true; // No rate limiting for this endpoint
        }
        
        var key = $"{clientId}:{endpoint}";
        return await limiter.IsAllowedAsync(key);
    }
}
```

## Response Headers

### Rate Limit Headers
```csharp
public class RateLimitHeadersMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IRateLimiter _rateLimiter;
    
    public async Task InvokeAsync(HttpContext context)
    {
        var clientId = GetClientId(context);
        var endpoint = context.Request.Path.Value;
        
        // Rate limit bilgilerini al
        var limitInfo = await _rateLimiter.GetLimitInfoAsync(clientId, endpoint);
        
        // Response headers ekle
        context.Response.Headers.Add("X-RateLimit-Limit", limitInfo.Limit.ToString());
        context.Response.Headers.Add("X-RateLimit-Remaining", limitInfo.Remaining.ToString());
        context.Response.Headers.Add("X-RateLimit-Reset", limitInfo.ResetTime.ToUnixTimeSeconds().ToString());
        
        await _next(context);
    }
}
```

## Mülakat Soruları

### Temel Sorular

1. **Rate limiting nedir ve neden önemlidir?**
   - **Cevap**: API kullanımını sınırlayan mekanizma. API abuse prevention, resource protection ve fair usage için kritik.

2. **Fixed Window vs Sliding Window arasındaki fark nedir?**
   - **Cevap**: Fixed window belirli zaman dilimlerinde sayar, sliding window sürekli hareket eden pencere kullanır. Sliding window daha adil.

3. **Token Bucket algorithm nasıl çalışır?**
   - **Cevap**: Bucket'da token'lar saklanır, her request bir token kullanır. Token'lar sürekli yenilenir, burst traffic'e izin verir.

4. **Rate limiting'de client identification nasıl yapılır?**
   - **Cevap**: API key, IP address, user ID, session token gibi yöntemlerle. Context'e göre en uygun yöntem seçilir.

5. **Distributed rate limiting neden gerekli?**
   - **Cevap**: Multiple server instance'larda rate limit state'i paylaşmak için. Redis gibi shared storage kullanılır.

### Teknik Sorular

1. **Rate limiting'de race condition nasıl önlenir?**
   - **Cevap**: Atomic operations, distributed locks, Lua scripts (Redis'te) kullanarak. Concurrent request'lerde consistency sağlanır.

2. **Rate limit exceeded durumunda ne yapılır?**
   - **Cevap**: 429 status code, Retry-After header, rate limit bilgileri. Client'a ne zaman tekrar deneyebileceği bilgisi verilir.

3. **Different rate limits for different users nasıl implement edilir?**
   - **Cevap**: Policy-based approach, user tiers (free/premium/enterprise), endpoint-specific limits. Configuration-driven design.

4. **Rate limiting performance'ı nasıl optimize edilir?**
   - **Cevap**: Caching, efficient data structures, background cleanup, Redis pipelining. Memory ve CPU kullanımı optimize edilir.

5. **Rate limiting bypass nasıl önlenir?**
   - **Cevap**: Multiple identification methods, IP rotation detection, behavioral analysis, rate limit evasion detection.

## Best Practices

1. **Policy Design**
   - Endpoint-specific limits belirleyin
   - User tier-based limits uygulayın
   - Burst traffic'e izin verin
   - Graceful degradation sağlayın

2. **Implementation**
   - Atomic operations kullanın
   - Efficient data structures seçin
   - Background cleanup yapın
   - Monitoring ekleyin

3. **Monitoring**
   - Rate limit metrics toplayın
   - Client behavior analiz edin
   - Alerting kurun
   - Performance izleyin

4. **Security**
   - Multiple identification methods kullanın
   - Bypass detection implement edin
   - Rate limit evasion önleyin
   - Audit logging yapın

## Kaynaklar

- [ASP.NET Core Rate Limiting](https://docs.microsoft.com/en-us/aspnet/core/performance/rate-limit)
- [Redis Rate Limiting](https://redis.io/commands/incr/)
- [Rate Limiting Strategies](https://cloud.google.com/architecture/rate-limiting-strategies-techniques)
- [API Rate Limiting Best Practices](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html)
- [Rate Limiting Algorithms](https://en.wikipedia.org/wiki/Rate_limiting)
