# Middleware

## Genel Bakış
Middleware, ASP.NET Core uygulamalarında HTTP isteklerini işleyen ve yanıtları oluşturan bileşenlerdir. Bu bileşenler, istek pipeline'ında sıralı olarak çalışır ve her biri belirli bir işlevi yerine getirir.

## Middleware Nedir?
- HTTP isteklerini işleyen bileşenler
- Pipeline içinde sıralı çalışma
- Request ve response manipülasyonu
- Özelleştirilebilir yapı

## Middleware Pipeline
```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // Exception handling middleware
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }
    else
    {
        app.UseExceptionHandler("/Error");
    }

    // Static files middleware
    app.UseStaticFiles();

    // Routing middleware
    app.UseRouting();

    // Authentication middleware
    app.UseAuthentication();

    // Authorization middleware
    app.UseAuthorization();

    // Endpoint middleware
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

## Middleware Türleri

### 1. Built-in Middleware
```csharp
// Exception handling
app.UseExceptionHandler("/Error");

// Static files
app.UseStaticFiles();

// Routing
app.UseRouting();

// Authentication
app.UseAuthentication();

// Authorization
app.UseAuthorization();

// CORS
app.UseCors("MyPolicy");

// Compression
app.UseResponseCompression();
```

### 2. Custom Middleware
```csharp
public class CustomMiddleware
{
    private readonly RequestDelegate _next;

    public CustomMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Request işleme
        await _next(context);
        // Response işleme
    }
}

// Extension method
public static class CustomMiddlewareExtensions
{
    public static IApplicationBuilder UseCustomMiddleware(
        this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<CustomMiddleware>();
    }
}

// Kullanımı
app.UseCustomMiddleware();
```

## Middleware Yaşam Döngüsü

### 1. Oluşturma
- Constructor injection
- Singleton yaşam döngüsü
- RequestDelegate parametresi

### 2. Çalışma
- Invoke/InvokeAsync metodu
- Request işleme
- Next middleware çağrısı
- Response işleme

### 3. Sonlandırma
- Dispose pattern
- Kaynak temizleme
- Exception handling

## Best Practices

### 1. Middleware Sıralaması
- Exception handling en başta
- Static files routing'den önce
- Authentication authorization'dan önce
- Endpoint en sonda

### 2. Performans
- Async/await kullanımı
- Gereksiz middleware'lerden kaçınma
- Response compression
- Caching stratejileri

### 3. Güvenlik
- HTTPS yönlendirmesi
- Security headers
- CORS politikaları
- Rate limiting

## Örnek Uygulamalar

### 1. Request Logging Middleware
```csharp
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;

    public RequestLoggingMiddleware(
        RequestDelegate next,
        ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var startTime = DateTime.UtcNow;
        
        await _next(context);
        
        var endTime = DateTime.UtcNow;
        var duration = endTime - startTime;
        
        _logger.LogInformation(
            "Request {method} {url} => {statusCode} ({duration}ms)",
            context.Request.Method,
            context.Request.Path,
            context.Response.StatusCode,
            duration.TotalMilliseconds);
    }
}
```

### 2. Custom Header Middleware
```csharp
public class CustomHeaderMiddleware
{
    private readonly RequestDelegate _next;
    private readonly string _headerName;
    private readonly string _headerValue;

    public CustomHeaderMiddleware(
        RequestDelegate next,
        string headerName,
        string headerValue)
    {
        _next = next;
        _headerName = headerName;
        _headerValue = headerValue;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        context.Response.Headers.Add(_headerName, _headerValue);
        await _next(context);
    }
}
```

## Sık Sorulan Sorular

### 1. Middleware sıralaması neden önemlidir?
- İşlem sırası belirler
- Performansı etkiler
- Güvenliği sağlar
- Hata yönetimini kolaylaştırır

### 2. Custom middleware ne zaman kullanılmalıdır?
- Özel işlemler gerektiğinde
- Cross-cutting concerns için
- Request/response manipülasyonu için
- Logging ve monitoring için

### 3. Middleware performansını nasıl optimize edebilirim?
- Gereksiz middleware'leri kaldırın
- Async/await kullanın
- Response compression kullanın
- Caching stratejileri uygulayın

## Kaynaklar
- [ASP.NET Core Middleware](https://docs.microsoft.com/tr-tr/aspnet/core/fundamentals/middleware/)
- [Custom Middleware](https://docs.microsoft.com/tr-tr/aspnet/core/fundamentals/middleware/write)
- [Middleware Order](https://docs.microsoft.com/tr-tr/aspnet/core/fundamentals/middleware/#middleware-order) 