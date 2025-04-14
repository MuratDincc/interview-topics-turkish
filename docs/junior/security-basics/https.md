# HTTPS

## Genel Bakış
HTTPS (Hypertext Transfer Protocol Secure), HTTP protokolünün güvenli versiyonudur. SSL/TLS protokolleri kullanılarak veri iletiminin şifrelenmesini sağlar. Bu sayede verilerin güvenli bir şekilde iletilmesi ve gizliliğinin korunması sağlanır.

## Mülakat Soruları ve Cevapları

### 1. SSL/TLS nedir ve nasıl çalışır?
**Cevap:**
SSL (Secure Sockets Layer) ve TLS (Transport Layer Security), internet üzerinden güvenli veri iletişimi sağlayan kriptografik protokollerdir. Çalışma prensibi:
1. Handshake (El sıkışma)
2. Anahtar değişimi
3. Veri şifreleme
4. Veri iletimi

**Örnek Kod:**
```csharp
// SSL/TLS yapılandırması
public void ConfigureServices(IServiceCollection services)
{
    services.AddHttpsRedirection(options =>
    {
        options.RedirectStatusCode = StatusCodes.Status308PermanentRedirect;
        options.HttpsPort = 443;
    });

    services.AddHsts(options =>
    {
        options.Preload = true;
        options.IncludeSubDomains = true;
        options.MaxAge = TimeSpan.FromDays(365);
    });
}

// HTTPS zorunluluğu
[RequireHttps]
[ApiController]
[Route("api/[controller]")]
public class SecureController : ControllerBase
{
    [HttpGet]
    public IActionResult GetSecureData()
    {
        return Ok("Secure data");
    }
}
```

### 2. Sertifika yönetimi nasıl yapılır?
**Cevap:**
Sertifika yönetimi için:
- Sertifika oluşturma
- Sertifika doğrulama
- Sertifika yenileme
- Sertifika depolama

**Örnek Kod:**
```csharp
// Sertifika yapılandırması
public void ConfigureServices(IServiceCollection services)
{
    services.AddCertificateForwarding(options =>
    {
        options.CertificateHeader = "X-ARR-ClientCert";
        options.HeaderConverter = (headerValue) =>
        {
            X509Certificate2 certificate = null;
            if (!string.IsNullOrWhiteSpace(headerValue))
            {
                byte[] bytes = Convert.FromBase64String(headerValue);
                certificate = new X509Certificate2(bytes);
            }
            return certificate;
        };
    });
}

// Sertifika doğrulama
public class CertificateValidationService
{
    public bool ValidateCertificate(X509Certificate2 certificate)
    {
        // Sertifika geçerlilik süresi kontrolü
        if (DateTime.Now > certificate.NotAfter)
            return false;

        // Sertifika zinciri doğrulama
        var chain = new X509Chain();
        chain.ChainPolicy.RevocationMode = X509RevocationMode.Online;
        chain.ChainPolicy.RevocationFlag = X509RevocationFlag.ExcludeRoot;
        chain.ChainPolicy.VerificationFlags = X509VerificationFlags.NoFlag;
        chain.ChainPolicy.VerificationTime = DateTime.Now;
        chain.ChainPolicy.UrlRetrievalTimeout = new TimeSpan(0, 0, 30);

        return chain.Build(certificate);
    }
}
```

### 3. HTTPS yönlendirmesi nasıl yapılır?
**Cevap:**
HTTPS yönlendirmesi için:
- HTTP'den HTTPS'e yönlendirme
- HSTS (HTTP Strict Transport Security)
- Port yönlendirme
- Özel domain yönlendirme

**Örnek Kod:**
```csharp
// HTTPS yönlendirme yapılandırması
public void Configure(IApplicationBuilder app)
{
    app.UseHttpsRedirection();
    app.UseHsts();

    // Özel yönlendirme kuralları
    app.UseRewriter(new RewriteOptions()
        .AddRedirectToHttps(StatusCodes.Status301MovedPermanently, 443)
        .AddRedirect("^old-path/(.*)", "new-path/$1", StatusCodes.Status301MovedPermanently));
}

// HSTS yapılandırması
public void ConfigureServices(IServiceCollection services)
{
    services.AddHsts(options =>
    {
        options.Preload = true;
        options.IncludeSubDomains = true;
        options.MaxAge = TimeSpan.FromDays(365);
        options.ExcludedHosts.Add("example.com");
    });
}
```

### 4. SSL/TLS güvenlik ayarları nasıl yapılandırılır?
**Cevap:**
SSL/TLS güvenlik ayarları için:
- Şifreleme algoritmaları
- Protokol versiyonları
- Anahtar uzunlukları
- Güvenlik politikaları

**Örnek Kod:**
```csharp
// SSL/TLS güvenlik ayarları
public void ConfigureServices(IServiceCollection services)
{
    services.AddHttpsRedirection(options =>
    {
        options.RedirectStatusCode = StatusCodes.Status308PermanentRedirect;
        options.HttpsPort = 443;
    });

    services.Configure<KestrelServerOptions>(options =>
    {
        options.ConfigureHttpsDefaults(httpsOptions =>
        {
            httpsOptions.SslProtocols = SslProtocols.Tls12 | SslProtocols.Tls13;
            httpsOptions.ClientCertificateMode = ClientCertificateMode.RequireCertificate;
            httpsOptions.CheckCertificateRevocation = true;
        });
    });
}

// Özel SSL/TLS politikası
public class SslPolicy
{
    public static bool ValidateServerCertificate(
        object sender,
        X509Certificate certificate,
        X509Chain chain,
        SslPolicyErrors sslPolicyErrors)
    {
        if (sslPolicyErrors == SslPolicyErrors.None)
            return true;

        // Özel doğrulama kuralları
        if (sslPolicyErrors == SslPolicyErrors.RemoteCertificateChainErrors)
        {
            // Zincir doğrulama
            return chain.ChainStatus
                .All(status => status.Status == X509ChainStatusFlags.NoError);
        }

        return false;
    }
}
```

### 5. HTTPS performans optimizasyonu nasıl yapılır?
**Cevap:**
HTTPS performans optimizasyonu için:
- Session resumption
- OCSP stapling
- HTTP/2 kullanımı
- Önbellek yönetimi

**Örnek Kod:**
```csharp
// HTTPS performans optimizasyonu
public void ConfigureServices(IServiceCollection services)
{
    services.Configure<KestrelServerOptions>(options =>
    {
        options.ConfigureHttpsDefaults(httpsOptions =>
        {
            // Session resumption
            httpsOptions.SslProtocols = SslProtocols.Tls12 | SslProtocols.Tls13;
            httpsOptions.AllowResume = true;

            // OCSP stapling
            httpsOptions.UseOcspStapling = true;

            // HTTP/2
            options.Listen(IPAddress.Any, 443, listenOptions =>
            {
                listenOptions.UseHttps(httpsOptions =>
                {
                    httpsOptions.HttpProtocols = HttpProtocols.Http2;
                });
            });
        });
    });
}

// Önbellek yönetimi
public class HttpsCacheMiddleware
{
    public async Task InvokeAsync(HttpContext context)
    {
        context.Response.Headers[HeaderNames.CacheControl] = "public, max-age=31536000";
        await _next(context);
    }
}
```

## Best Practices
1. **Güvenlik**
   - En güncel SSL/TLS versiyonlarını kullanın
   - Güçlü şifreleme algoritmaları seçin
   - Sertifikaları düzenli olarak yenileyin
   - HSTS kullanın

2. **Performans**
   - Session resumption kullanın
   - OCSP stapling etkinleştirin
   - HTTP/2 kullanın
   - Önbellek yönetimini optimize edin

3. **Yapılandırma**
   - Güvenli varsayılan ayarlar kullanın
   - Düzenli güvenlik taramaları yapın
   - Sertifika yönetimini otomatikleştirin
   - Hata loglamayı etkinleştirin

## Kaynaklar
- [ASP.NET Core HTTPS](https://docs.microsoft.com/tr-tr/aspnet/core/security/enforcing-ssl)
- [SSL/TLS Best Practices](https://www.ssl.com/guide/ssl-best-practices/)
- [Let's Encrypt Documentation](https://letsencrypt.org/docs/) 