# API Versioning

## Genel Bakış
API versiyonlama, API'lerin zaman içinde değişimini yönetmek için kullanılan bir stratejidir. Bu, geriye dönük uyumluluğu korurken yeni özelliklerin eklenmesine ve mevcut özelliklerin değiştirilmesine olanak tanır.

## Mülakat Soruları ve Cevapları

### 1. API versiyonlama stratejileri nelerdir?
**Cevap:**
Temel API versiyonlama stratejileri:
- URL versiyonlama: `/api/v1/products`
- Header versiyonlama: `Accept: application/vnd.company.api.v1+json`
- Media Type versiyonlama: `Content-Type: application/vnd.company.api.v1+json`
- Query String versiyonlama: `/api/products?version=1`

**Örnek Kod:**
```csharp
// URL versiyonlama
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<IEnumerable<Product>>> GetProducts()
    {
        // ...
    }
}

// Header versiyonlama
[ApiVersion("1.0")]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<IEnumerable<Product>>> GetProducts()
    {
        // ...
    }
}
```

### 2. ASP.NET Core'da API versiyonlama nasıl yapılır?
**Cevap:**
ASP.NET Core'da API versiyonlama için:
- Microsoft.AspNetCore.Mvc.Versioning paketi kullanılır
- Startup.cs'de servis olarak eklenir
- Controller'larda ApiVersion attribute'u kullanılır
- Farklı versiyonlama stratejileri desteklenir

**Örnek Kod:**
```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddApiVersioning(options =>
    {
        options.DefaultApiVersion = new ApiVersion(1, 0);
        options.AssumeDefaultVersionWhenUnspecified = true;
        options.ReportApiVersions = true;
        options.ApiVersionReader = ApiVersionReader.Combine(
            new UrlSegmentApiVersionReader(),
            new HeaderApiVersionReader("x-api-version"),
            new QueryStringApiVersionReader("api-version")
        );
    });
}

// Controller
[ApiVersion("1.0")]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<IEnumerable<Product>>> GetProducts()
    {
        // ...
    }

    [HttpGet, MapToApiVersion("2.0")]
    public async Task<ActionResult<IEnumerable<ProductV2>>> GetProductsV2()
    {
        // ...
    }
}
```

### 3. Breaking changes nasıl yönetilir?
**Cevap:**
Breaking changes yönetimi için:
- Yeni versiyon oluşturulur
- Eski versiyon desteklenmeye devam eder
- Geçiş süreci planlanır
- Dokümantasyon güncellenir

**Örnek Kod:**
```csharp
// V1 Controller
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<IEnumerable<Product>>> GetProducts()
    {
        // Eski implementasyon
    }
}

// V2 Controller
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsV2Controller : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<IEnumerable<ProductV2>>> GetProducts()
    {
        // Yeni implementasyon
    }
}
```

### 4. Versiyon geçiş stratejileri nelerdir?
**Cevap:**
Versiyon geçiş stratejileri:
- Paralel çalışma
- Kademeli geçiş
- Zorunlu geçiş
- Otomatik yönlendirme

**Örnek Kod:**
```csharp
// Versiyon yönlendirme
public class ApiVersionRedirectMiddleware
{
    public async Task InvokeAsync(HttpContext context)
    {
        if (context.Request.Path.Value.Contains("/api/"))
        {
            var version = context.Request.Headers["x-api-version"].ToString();
            if (string.IsNullOrEmpty(version))
            {
                context.Request.Headers["x-api-version"] = "2.0";
            }
        }
        await _next(context);
    }
}

// Versiyon kontrolü
[ApiVersion("1.0", Deprecated = true)]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<IEnumerable<Product>>> GetProducts()
    {
        if (HttpContext.GetRequestedApiVersion() == new ApiVersion(1, 0))
        {
            // Deprecated versiyon için uyarı
            Response.Headers.Add("Warning", "299 - This API version is deprecated");
        }
        // ...
    }
}
```

### 5. API versiyonlama best practices nelerdir?
**Cevap:**
API versiyonlama best practices:
- Semantic versioning kullanın
- Versiyonları dokümante edin
- Geriye dönük uyumluluğu koruyun
- Versiyon geçiş planı oluşturun

**Örnek Kod:**
```csharp
// Semantic versioning
[ApiVersion("1.0.0")]
[ApiVersion("2.0.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsController : ControllerBase
{
    // ...
}

// Versiyon dokümantasyonu
/// <summary>
/// Products API v2.0
/// </summary>
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsV2Controller : ControllerBase
{
    /// <summary>
    /// Gets all products with enhanced features
    /// </summary>
    [HttpGet]
    [ProducesResponseType(typeof(IEnumerable<ProductV2>), 200)]
    [ProducesResponseType(400)]
    public async Task<ActionResult<IEnumerable<ProductV2>>> GetProducts()
    {
        // ...
    }
}
```

## Best Practices
1. **Versiyonlama Stratejisi**
   - Semantic versioning kullanın
   - Tutarlı bir strateji seçin
   - Versiyonları dokümante edin
   - Geçiş planı oluşturun

2. **Geriye Dönük Uyumluluk**
   - Breaking changes'den kaçının
   - Eski versiyonları destekleyin
   - Geçiş süresi tanıyın
   - Uyarılar ekleyin

3. **Dokümantasyon**
   - Versiyon değişikliklerini belgeleyin
   - Örnek istekleri ekleyin
   - Geçiş kılavuzu hazırlayın
   - Deprecated API'leri işaretleyin

## Kaynaklar
- [ASP.NET Core API Versioning](https://docs.microsoft.com/tr-tr/aspnet/core/web-api/versioning)
- [API Versioning Best Practices](https://www.mulesoft.com/resources/api/versioning-best-practices)
- [Semantic Versioning](https://semver.org/) 