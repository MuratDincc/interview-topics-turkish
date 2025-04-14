# CORS

## Genel Bakış
CORS (Cross-Origin Resource Sharing), farklı kaynaklardan (origin) gelen isteklerin güvenli bir şekilde yönetilmesini sağlayan bir mekanizmadır. Web tarayıcıları, güvenlik nedeniyle varsayılan olarak farklı kaynaklardan gelen istekleri engeller. CORS, bu kısıtlamaları yönetmek için kullanılan bir protokoldür.

## Mülakat Soruları ve Cevapları

### 1. CORS nedir ve neden önemlidir?
**Cevap:**
CORS, web uygulamalarının farklı kaynaklardan gelen istekleri güvenli bir şekilde yönetmesini sağlar. Önemi:
- Güvenlik: Same-Origin Policy'yi yönetir
- Esneklik: Farklı domainler arası iletişimi sağlar
- Kontrol: Hangi kaynaklardan istek kabul edileceğini belirler

**Örnek Kod:**
```csharp
// CORS yapılandırması
public void ConfigureServices(IServiceCollection services)
{
    services.AddCors(options =>
    {
        options.AddPolicy("AllowSpecificOrigin",
            builder => builder.WithOrigins("https://example.com")
                            .AllowAnyMethod()
                            .AllowAnyHeader());
    });
}

// CORS middleware kullanımı
public void Configure(IApplicationBuilder app)
{
    app.UseCors("AllowSpecificOrigin");
}
```

### 2. CORS politikaları nasıl yapılandırılır?
**Cevap:**
CORS politikaları için:
- Origin belirleme
- HTTP metodları
- Header'lar
- Credentials
- Preflight istekleri

**Örnek Kod:**
```csharp
// Detaylı CORS politikası
public void ConfigureServices(IServiceCollection services)
{
    services.AddCors(options =>
    {
        options.AddPolicy("CustomPolicy", builder =>
        {
            builder.WithOrigins("https://api.example.com")
                   .WithMethods("GET", "POST", "PUT", "DELETE")
                   .WithHeaders("Authorization", "Content-Type")
                   .AllowCredentials()
                   .SetPreflightMaxAge(TimeSpan.FromMinutes(10));
        });
    });
}

// Farklı ortamlar için CORS
public void ConfigureServices(IServiceCollection services, IWebHostEnvironment env)
{
    services.AddCors(options =>
    {
        if (env.IsDevelopment())
        {
            options.AddPolicy("DevelopmentPolicy",
                builder => builder.AllowAnyOrigin()
                                .AllowAnyMethod()
                                .AllowAnyHeader());
        }
        else
        {
            options.AddPolicy("ProductionPolicy",
                builder => builder.WithOrigins("https://production.com")
                                .WithMethods("GET", "POST")
                                .WithHeaders("Authorization"));
        }
    });
}
```

### 3. Preflight istekleri nedir ve nasıl yönetilir?
**Cevap:**
Preflight istekleri, karmaşık CORS isteklerinden önce gönderilen OPTIONS istekleridir. Yönetimi için:
- OPTIONS isteklerini işleme
- CORS header'larını ayarlama
- Cache kontrolü
- Timeout yönetimi

**Örnek Kod:**
```csharp
// Preflight istek yönetimi
public void ConfigureServices(IServiceCollection services)
{
    services.AddCors(options =>
    {
        options.AddPolicy("PreflightPolicy", builder =>
        {
            builder.WithOrigins("https://api.example.com")
                   .WithMethods("GET", "POST", "OPTIONS")
                   .WithHeaders("Authorization", "Content-Type")
                   .SetPreflightMaxAge(TimeSpan.FromMinutes(10));
        });
    });
}

// Preflight middleware
public class PreflightMiddleware
{
    public async Task InvokeAsync(HttpContext context)
    {
        if (context.Request.Method == "OPTIONS")
        {
            context.Response.Headers.Add("Access-Control-Allow-Origin", "https://api.example.com");
            context.Response.Headers.Add("Access-Control-Allow-Methods", "GET, POST, OPTIONS");
            context.Response.Headers.Add("Access-Control-Allow-Headers", "Authorization, Content-Type");
            context.Response.Headers.Add("Access-Control-Max-Age", "600");
            context.Response.StatusCode = 204;
            return;
        }
        await _next(context);
    }
}
```

### 4. CORS güvenlik riskleri nelerdir ve nasıl önlenir?
**Cevap:**
CORS güvenlik riskleri:
- Açık origin politikaları
- Hassas veri sızıntısı
- CSRF saldırıları
- Header enjeksiyonu

**Örnek Kod:**
```csharp
// Güvenli CORS yapılandırması
public void ConfigureServices(IServiceCollection services)
{
    services.AddCors(options =>
    {
        options.AddPolicy("SecurePolicy", builder =>
        {
            builder.WithOrigins("https://trusted-domain.com")
                   .WithMethods("GET", "POST")
                   .WithHeaders("Content-Type")
                   .DisallowCredentials()
                   .SetPreflightMaxAge(TimeSpan.FromMinutes(5));
        });
    });
}

// CORS güvenlik kontrolü
public class CorsSecurityMiddleware
{
    public async Task InvokeAsync(HttpContext context)
    {
        var origin = context.Request.Headers["Origin"].ToString();
        if (!IsTrustedOrigin(origin))
        {
            context.Response.StatusCode = 403;
            return;
        }

        context.Response.Headers.Add("Access-Control-Allow-Origin", origin);
        context.Response.Headers.Add("Access-Control-Allow-Methods", "GET, POST");
        context.Response.Headers.Add("Access-Control-Allow-Headers", "Content-Type");
        await _next(context);
    }

    private bool IsTrustedOrigin(string origin)
    {
        var trustedOrigins = new[] { "https://trusted-domain.com" };
        return trustedOrigins.Contains(origin);
    }
}
```

### 5. CORS performans optimizasyonu nasıl yapılır?
**Cevap:**
CORS performans optimizasyonu için:
- Preflight önbellekleme
- Header optimizasyonu
- Origin kontrolü
- Middleware sıralaması

**Örnek Kod:**
```csharp
// Performans odaklı CORS yapılandırması
public void ConfigureServices(IServiceCollection services)
{
    services.AddCors(options =>
    {
        options.AddPolicy("PerformancePolicy", builder =>
        {
            builder.WithOrigins("https://api.example.com")
                   .WithMethods("GET", "POST")
                   .WithHeaders("Content-Type")
                   .SetPreflightMaxAge(TimeSpan.FromHours(1))
                   .SetIsOriginAllowedToAllowWildcardSubdomains();
        });
    });
}

// Önbellek yönetimi
public class CorsCacheMiddleware
{
    public async Task InvokeAsync(HttpContext context)
    {
        if (context.Request.Method == "OPTIONS")
        {
            context.Response.Headers.Add("Cache-Control", "public, max-age=3600");
        }
        await _next(context);
    }
}
```

## Best Practices
1. **Güvenlik**
   - Spesifik origin'leri belirtin
   - Gereksiz header'ları kısıtlayın
   - Credentials kullanımını sınırlayın
   - Preflight önbellekleme süresini optimize edin

2. **Performans**
   - Preflight isteklerini önbelleğe alın
   - Gereksiz header'ları kaldırın
   - Origin kontrolünü optimize edin
   - Middleware sıralamasını doğru yapın

3. **Yapılandırma**
   - Ortama göre farklı politikalar kullanın
   - Header'ları minimumda tutun
   - Timeout değerlerini optimize edin
   - Hata yönetimini etkinleştirin

## Kaynaklar
- [ASP.NET Core CORS](https://docs.microsoft.com/tr-tr/aspnet/core/security/cors)
- [MDN CORS](https://developer.mozilla.org/tr/docs/Web/HTTP/CORS)
- [CORS Best Practices](https://www.moesif.com/blog/technical/cors/Authoritative-Guide-to-CORS-Cross-Origin-Resource-Sharing-for-REST-APIs/) 