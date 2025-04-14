# Cross-Cutting Concerns

## Genel Bakış
Cross-Cutting Concerns, uygulamanın farklı katmanlarında tekrar eden ve merkezi olarak yönetilmesi gereken işlevleri ifade eder. Bu işlevler, uygulamanın temel iş mantığından bağımsız olarak tüm katmanlarda kullanılır.

## Temel Bileşenler

### 1. Logging
Uygulama genelinde loglama işlemleri.

**Örnek Kod:**
```csharp
public interface ILoggerService
{
    void LogInformation(string message);
    void LogWarning(string message);
    void LogError(string message, Exception exception = null);
    void LogDebug(string message);
}

public class SerilogLoggerService : ILoggerService
{
    private readonly ILogger _logger;

    public SerilogLoggerService(ILogger logger)
    {
        _logger = logger;
    }

    public void LogInformation(string message)
    {
        _logger.LogInformation(message);
    }

    public void LogWarning(string message)
    {
        _logger.LogWarning(message);
    }

    public void LogError(string message, Exception exception = null)
    {
        _logger.LogError(exception, message);
    }

    public void LogDebug(string message)
    {
        _logger.LogDebug(message);
    }
}

// Logging middleware
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILoggerService _logger;

    public RequestLoggingMiddleware(RequestDelegate next, ILoggerService logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var startTime = DateTime.UtcNow;
        
        try
        {
            await _next(context);
            var duration = DateTime.UtcNow - startTime;
            
            _logger.LogInformation(
                "Request {Method} {Path} completed in {Duration}ms",
                context.Request.Method,
                context.Request.Path,
                duration.TotalMilliseconds
            );
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Request failed");
            throw;
        }
    }
}
```

### 2. Exception Handling
Merkezi hata yönetimi.

**Örnek Kod:**
```csharp
public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILoggerService _logger;
    private readonly IWebHostEnvironment _environment;

    public GlobalExceptionHandler(
        ILoggerService logger,
        IWebHostEnvironment environment)
    {
        _logger = logger;
        _environment = environment;
    }

    public async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        _logger.LogError(exception, "An unhandled exception occurred");

        var response = new ErrorResponse
        {
            Message = _environment.IsDevelopment() 
                ? exception.Message 
                : "An error occurred while processing your request",
            Details = _environment.IsDevelopment() 
                ? exception.ToString() 
                : null
        };

        context.Response.ContentType = "application/json";
        context.Response.StatusCode = GetStatusCode(exception);

        await context.Response.WriteAsJsonAsync(response);
    }

    private static int GetStatusCode(Exception exception)
    {
        return exception switch
        {
            ValidationException => StatusCodes.Status400BadRequest,
            NotFoundException => StatusCodes.Status404NotFound,
            UnauthorizedAccessException => StatusCodes.Status401Unauthorized,
            _ => StatusCodes.Status500InternalServerError
        };
    }
}
```

### 3. Caching
Önbellekleme işlemleri.

**Örnek Kod:**
```csharp
public interface ICacheService
{
    Task<T> GetAsync<T>(string key);
    Task SetAsync<T>(string key, T value, TimeSpan? expiration = null);
    Task RemoveAsync(string key);
    Task<bool> ExistsAsync(string key);
}

public class DistributedCacheService : ICacheService
{
    private readonly IDistributedCache _cache;
    private readonly ILoggerService _logger;

    public DistributedCacheService(
        IDistributedCache cache,
        ILoggerService logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public async Task<T> GetAsync<T>(string key)
    {
        try
        {
            var value = await _cache.GetStringAsync(key);
            return value == null ? default : JsonSerializer.Deserialize<T>(value);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Cache read error");
            return default;
        }
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan? expiration = null)
    {
        try
        {
            var options = new DistributedCacheEntryOptions();
            if (expiration.HasValue)
            {
                options.SetAbsoluteExpiration(expiration.Value);
            }

            var serializedValue = JsonSerializer.Serialize(value);
            await _cache.SetStringAsync(key, serializedValue, options);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Cache write error");
        }
    }
}
```

### 4. Validation
Merkezi doğrulama işlemleri.

**Örnek Kod:**
```csharp
public interface IValidator<T>
{
    Task<ValidationResult> ValidateAsync(T instance);
}

public class FluentValidator<T> : IValidator<T>
{
    private readonly IEnumerable<IValidationRule<T>> _rules;

    public FluentValidator(IEnumerable<IValidationRule<T>> rules)
    {
        _rules = rules;
    }

    public async Task<ValidationResult> ValidateAsync(T instance)
    {
        var errors = new List<ValidationError>();
        
        foreach (var rule in _rules)
        {
            var result = await rule.ValidateAsync(instance);
            if (!result.IsValid)
            {
                errors.AddRange(result.Errors);
            }
        }

        return new ValidationResult(errors);
    }
}

// Validation middleware
public class ValidationMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IValidatorFactory _validatorFactory;

    public ValidationMiddleware(
        RequestDelegate next,
        IValidatorFactory validatorFactory)
    {
        _next = next;
        _validatorFactory = validatorFactory;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        if (context.Request.Method == HttpMethods.Post || 
            context.Request.Method == HttpMethods.Put)
        {
            var model = await context.Request.ReadFromJsonAsync<object>();
            var validator = _validatorFactory.GetValidator(model.GetType());
            
            var result = await validator.ValidateAsync(model);
            if (!result.IsValid)
            {
                context.Response.StatusCode = StatusCodes.Status400BadRequest;
                await context.Response.WriteAsJsonAsync(result);
                return;
            }
        }

        await _next(context);
    }
}
```

### 5. Security
Güvenlik ile ilgili işlemler.

**Örnek Kod:**
```csharp
public interface ISecurityService
{
    Task<string> GenerateJwtToken(User user);
    Task<bool> ValidateToken(string token);
    Task<User> GetUserFromToken(string token);
}

public class JwtSecurityService : ISecurityService
{
    private readonly JwtSettings _jwtSettings;
    private readonly ILoggerService _logger;

    public JwtSecurityService(
        IOptions<JwtSettings> jwtSettings,
        ILoggerService logger)
    {
        _jwtSettings = jwtSettings.Value;
        _logger = logger;
    }

    public async Task<string> GenerateJwtToken(User user)
    {
        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
            new Claim(ClaimTypes.Name, user.Username),
            new Claim(ClaimTypes.Role, user.Role)
        };

        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_jwtSettings.Secret));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: _jwtSettings.Issuer,
            audience: _jwtSettings.Audience,
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(_jwtSettings.ExpiryMinutes),
            signingCredentials: credentials
        );

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

## Best Practices

### 1. Logging Stratejisi
- Log seviyelerini doğru kullanın
- Anlamlı log mesajları yazın
- Structured logging kullanın
- Log rotasyonu yapılandırın

### 2. Exception Handling
- Merkezi exception handling kullanın
- Özel exception tipleri tanımlayın
- Hata mesajlarını anlamlı yapın
- Production'da detaylı hata göstermeyin

### 3. Caching Stratejisi
- Cache key'lerini anlamlı seçin
- Cache sürelerini optimize edin
- Cache invalidation stratejisi belirleyin
- Distributed cache kullanın

### 4. Validation Yaklaşımı
- Fluent validation kullanın
- Validation kurallarını merkezi yönetin
- Custom validators yazın
- Validation hatalarını anlamlı yapın

### 5. Security Önlemleri
- JWT kullanın
- Role-based authorization uygulayın
- Input validation yapın
- CORS politikası belirleyin

## Sık Sorulan Sorular

### 1. Cross-Cutting Concerns neden önemlidir?
- Kod tekrarını önler
- Merkezi yönetim sağlar
- Bakımı kolaylaştırır
- Tutarlılık sağlar

### 2. Logging nasıl yapılandırılmalıdır?
- Log seviyeleri belirlenmelidir
- Structured logging kullanılmalıdır
- Log rotasyonu yapılandırılmalıdır
- Log analizi yapılmalıdır

### 3. Caching stratejisi nasıl belirlenmelidir?
- Cache key'leri anlamlı olmalıdır
- Cache süreleri optimize edilmelidir
- Cache invalidation stratejisi belirlenmelidir
- Distributed cache kullanılmalıdır

## Kaynaklar
- [Microsoft Cross-Cutting Concerns](https://docs.microsoft.com/tr-tr/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/ddd-oriented-microservice#cross-cutting-concerns)
- [Clean Architecture - Cross-Cutting Concerns](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Cross-Cutting Concerns Best Practices](https://docs.microsoft.com/tr-tr/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/ddd-oriented-microservice#cross-cutting-concerns) 