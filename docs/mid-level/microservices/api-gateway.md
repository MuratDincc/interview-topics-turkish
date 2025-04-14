# API Gateway

## Genel Bakış
API Gateway, mikroservis mimarisinde tüm istemci isteklerinin tek giriş noktası olarak hizmet veren bir bileşendir. İstekleri ilgili mikroservislere yönlendirir ve merkezi olarak yönetilen işlevler sağlar.

## Temel Özellikler

### 1. Yönlendirme ve Yük Dengeleme
```csharp
// Ocelot Configuration
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/orders/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "orderservice",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/orders/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ],
      "LoadBalancerOptions": {
        "Type": "RoundRobin"
      }
    },
    {
      "DownstreamPathTemplate": "/api/payments/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "paymentservice",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/payments/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST" ]
    }
  ]
}
```

### 2. Kimlik Doğrulama ve Yetkilendirme
```csharp
// Ocelot Authentication Configuration
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/orders/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "orderservice",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/orders/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ],
      "AuthenticationOptions": {
        "AuthenticationProviderKey": "Bearer",
        "AllowedScopes": ["orders.read", "orders.write"]
      }
    }
  ]
}

// JWT Authentication Middleware
public class JwtAuthenticationMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IConfiguration _configuration;

    public JwtAuthenticationMiddleware(
        RequestDelegate next,
        IConfiguration configuration)
    {
        _next = next;
        _configuration = configuration;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var token = context.Request.Headers["Authorization"]
            .ToString()
            .Replace("Bearer ", "");

        if (string.IsNullOrEmpty(token))
        {
            context.Response.StatusCode = 401;
            return;
        }

        try
        {
            var tokenHandler = new JwtSecurityTokenHandler();
            var key = Encoding.ASCII.GetBytes(_configuration["Jwt:Secret"]);
            
            tokenHandler.ValidateToken(token, new TokenValidationParameters
            {
                ValidateIssuerSigningKey = true,
                IssuerSigningKey = new SymmetricSecurityKey(key),
                ValidateIssuer = false,
                ValidateAudience = false,
                ClockSkew = TimeSpan.Zero
            }, out SecurityToken validatedToken);

            await _next(context);
        }
        catch
        {
            context.Response.StatusCode = 401;
        }
    }
}
```

### 3. Rate Limiting
```csharp
// Ocelot Rate Limiting Configuration
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/orders/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "orderservice",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/orders/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ],
      "RateLimitOptions": {
        "ClientWhitelist": [],
        "EnableRateLimiting": true,
        "Period": "1s",
        "PeriodTimespan": 1,
        "Limit": 10
      }
    }
  ]
}

// Custom Rate Limiting Middleware
public class RateLimitingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IMemoryCache _cache;
    private readonly int _limit;
    private readonly TimeSpan _period;

    public RateLimitingMiddleware(
        RequestDelegate next,
        IMemoryCache cache,
        IConfiguration configuration)
    {
        _next = next;
        _cache = cache;
        _limit = configuration.GetValue<int>("RateLimit:Limit");
        _period = TimeSpan.FromSeconds(
            configuration.GetValue<int>("RateLimit:PeriodSeconds"));
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var key = $"{context.Request.Path}-{context.Connection.RemoteIpAddress}";
        var counter = _cache.GetOrCreate(key, entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = _period;
            return 0;
        });

        if (counter >= _limit)
        {
            context.Response.StatusCode = 429;
            return;
        }

        _cache.Set(key, counter + 1);
        await _next(context);
    }
}
```

### 4. Caching
```csharp
// Ocelot Caching Configuration
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/products/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "productservice",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/products/{everything}",
      "UpstreamHttpMethod": [ "GET" ],
      "FileCacheOptions": {
        "TtlSeconds": 60,
        "Region": "products"
      }
    }
  ]
}

// Custom Caching Middleware
public class ResponseCachingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IMemoryCache _cache;

    public ResponseCachingMiddleware(
        RequestDelegate next,
        IMemoryCache cache)
    {
        _next = next;
        _cache = cache;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        if (context.Request.Method != HttpMethods.Get)
        {
            await _next(context);
            return;
        }

        var cacheKey = context.Request.Path + context.Request.QueryString;
        if (_cache.TryGetValue(cacheKey, out var cachedResponse))
        {
            await context.Response.WriteAsync(cachedResponse.ToString());
            return;
        }

        var originalBodyStream = context.Response.Body;
        using var memoryStream = new MemoryStream();
        context.Response.Body = memoryStream;

        await _next(context);

        memoryStream.Position = 0;
        var responseBody = await new StreamReader(memoryStream).ReadToEndAsync();
        
        _cache.Set(cacheKey, responseBody, TimeSpan.FromMinutes(5));
        
        memoryStream.Position = 0;
        await memoryStream.CopyToAsync(originalBodyStream);
        context.Response.Body = originalBodyStream;
    }
}
```

## Best Practices

### 1. Güvenlik
- SSL/TLS kullanın
- JWT authentication uygulayın
- Rate limiting uygulayın
- IP whitelisting yapın

### 2. Performans
- Response caching kullanın
- Load balancing yapın
- Circuit breaker uygulayın
- Timeout değerlerini ayarlayın

### 3. Monitoring
- Request/response logging yapın
- Error tracking uygulayın
- Performance metrics toplayın
- Health checks yapın

### 4. Scalability
- Horizontal scaling yapın
- Stateless tasarım kullanın
- Service discovery kullanın
- Load balancing uygulayın

## Sık Sorulan Sorular

### 1. API Gateway neden önemlidir?
- Merkezi yönetim sağlar
- Güvenlik önlemlerini tek noktada toplar
- İstekleri yönlendirir ve dengelemeyi sağlar
- Monitoring ve logging'i kolaylaştırır

### 2. Hangi API Gateway çözümleri kullanılabilir?
- Ocelot
- Kong
- Traefik
- Nginx
- AWS API Gateway

### 3. API Gateway'de hangi güvenlik önlemleri alınmalıdır?
- SSL/TLS kullanılmalı
- JWT authentication uygulanmalı
- Rate limiting yapılmalı
- IP whitelisting uygulanmalı

## Kaynaklar
- [Microsoft API Gateway](https://docs.microsoft.com/tr-tr/dotnet/architecture/microservices/architect-microservice-container-applications/direct-client-to-microservice-communication-versus-the-api-gateway-pattern)
- [Ocelot Documentation](https://ocelot.readthedocs.io/)
- [Kong Documentation](https://docs.konghq.com/)
- [Traefik Documentation](https://doc.traefik.io/traefik/) 