# Security Headers (Güvenlik Başlıkları)

## Genel Bakış
Güvenlik başlıkları, web uygulamalarının güvenliğini artırmak için kullanılan HTTP yanıt başlıklarıdır. Bu başlıklar, tarayıcıların uygulamayı nasıl işleyeceğini ve hangi güvenlik politikalarını uygulayacağını belirler.

## Temel Kavramlar

### 1. Content Security Policy (CSP)
```csharp
public class SecurityHeadersMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<SecurityHeadersMiddleware> _logger;

    public SecurityHeadersMiddleware(RequestDelegate next, ILogger<SecurityHeadersMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // CSP başlığı ekle
        context.Response.Headers.Add("Content-Security-Policy", 
            "default-src 'self'; " +
            "script-src 'self' 'unsafe-inline' 'unsafe-eval'; " +
            "style-src 'self' 'unsafe-inline'; " +
            "img-src 'self' data:; " +
            "font-src 'self'; " +
            "connect-src 'self'; " +
            "frame-ancestors 'none'; " +
            "form-action 'self'; " +
            "base-uri 'self'; " +
            "object-src 'none'");

        await _next(context);
    }
}
```

### 2. HTTP Strict Transport Security (HSTS)
```csharp
public class HstsMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<HstsMiddleware> _logger;

    public HstsMiddleware(RequestDelegate next, ILogger<HstsMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        if (context.Request.IsHttps)
        {
            // HSTS başlığı ekle
            context.Response.Headers.Add("Strict-Transport-Security", 
                "max-age=31536000; includeSubDomains; preload");
        }

        await _next(context);
    }
}
```

### 3. X-Frame-Options ve X-Content-Type-Options
```csharp
public class FrameOptionsMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<FrameOptionsMiddleware> _logger;

    public FrameOptionsMiddleware(RequestDelegate next, ILogger<FrameOptionsMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // X-Frame-Options başlığı ekle
        context.Response.Headers.Add("X-Frame-Options", "DENY");

        // X-Content-Type-Options başlığı ekle
        context.Response.Headers.Add("X-Content-Type-Options", "nosniff");

        await _next(context);
    }
}
```

### 4. Referrer Policy ve Permissions Policy
```csharp
public class ReferrerPolicyMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ReferrerPolicyMiddleware> _logger;

    public ReferrerPolicyMiddleware(RequestDelegate next, ILogger<ReferrerPolicyMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Referrer-Policy başlığı ekle
        context.Response.Headers.Add("Referrer-Policy", "strict-origin-when-cross-origin");

        // Permissions-Policy başlığı ekle
        context.Response.Headers.Add("Permissions-Policy", 
            "geolocation=(), " +
            "microphone=(), " +
            "camera=(), " +
            "payment=(), " +
            "usb=()");

        await _next(context);
    }
}
```

## Best Practices

### 1. Başlık Konfigürasyonu
- CSP politikaları
- HSTS ayarları
- Frame seçenekleri
- İçerik türü seçenekleri
- Referrer politikaları

### 2. Güvenlik Politikaları
- Kaynak kısıtlamaları
- İçerik güvenliği
- Çerçeve koruması
- MIME türü koruması
- İzin politikaları

### 3. Tarayıcı Uyumluluğu
- Tarayıcı desteği
- Geriye dönük uyumluluk
- Politika uygulaması
- Hata yönetimi
- Test stratejisi

## Sık Sorulan Sorular

### 1. Güvenlik başlıkları neden önemlidir?
- XSS koruması
- Clickjacking koruması
- MIME türü koruması
- Referrer bilgisi koruması
- SSL/TLS zorunluluğu

### 2. Güvenlik başlıkları nasıl yapılandırılır?
- CSP politikaları
- HSTS ayarları
- Frame seçenekleri
- İçerik türü seçenekleri
- Referrer politikaları

### 3. Güvenlik başlıkları zorlukları nelerdir?
- Politika karmaşıklığı
- Tarayıcı uyumluluğu
- Performans etkisi
- Hata ayıklama
- Test zorlukları

## Kaynaklar
- [Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP)
- [HTTP Strict Transport Security](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security)
- [X-Frame-Options](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options)
- [Referrer Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy)
- [Permissions Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Permissions-Policy) 