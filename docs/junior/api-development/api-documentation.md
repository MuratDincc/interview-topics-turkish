# API Documentation

## Genel Bakış
API dokümantasyonu, API'lerin nasıl kullanılacağını, endpoint'lerin ne işe yaradığını ve nasıl entegre edileceğini açıklayan teknik bir belgedir. İyi bir API dokümantasyonu, geliştiricilerin API'yi hızlı ve doğru bir şekilde kullanmasını sağlar.

## Mülakat Soruları ve Cevapları

### 1. Swagger/OpenAPI nedir ve nasıl kullanılır?
**Cevap:**
Swagger/OpenAPI, RESTful API'leri tanımlamak, oluşturmak ve dokümante etmek için kullanılan bir araçtır. ASP.NET Core'da Swashbuckle paketi ile entegre edilir.

**Örnek Kod:**
```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddSwaggerGen(c =>
    {
        c.SwaggerDoc("v1", new OpenApiInfo
        {
            Title = "Product API",
            Version = "v1",
            Description = "Product management API",
            Contact = new OpenApiContact
            {
                Name = "API Support",
                Email = "support@company.com"
            }
        });
    });
}

public void Configure(IApplicationBuilder app)
{
    app.UseSwagger();
    app.UseSwaggerUI(c =>
    {
        c.SwaggerEndpoint("/swagger/v1/swagger.json", "Product API V1");
    });
}

// Controller
[ApiController]
[Route("api/[controller]")]
[Produces("application/json")]
public class ProductsController : ControllerBase
{
    /// <summary>
    /// Gets all products
    /// </summary>
    /// <returns>List of products</returns>
    /// <response code="200">Returns the list of products</response>
    /// <response code="400">If the request is invalid</response>
    [HttpGet]
    [ProducesResponseType(typeof(IEnumerable<Product>), 200)]
    [ProducesResponseType(400)]
    public async Task<ActionResult<IEnumerable<Product>>> GetProducts()
    {
        // ...
    }
}
```

### 2. XML dokümantasyon nasıl oluşturulur?
**Cevap:**
XML dokümantasyonu için:
- Proje ayarlarında XML dokümantasyonu etkinleştirilir
- /// yorumları kullanılır
- Swagger'a XML dosyası eklenir

**Örnek Kod:**
```csharp
// Proje dosyası (.csproj)
<PropertyGroup>
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    <NoWarn>$(NoWarn);1591</NoWarn>
</PropertyGroup>

// Controller
/// <summary>
/// Products API controller
/// </summary>
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    /// <summary>
    /// Gets a specific product by id
    /// </summary>
    /// <param name="id">The product id</param>
    /// <returns>A product</returns>
    /// <response code="200">Returns the requested product</response>
    /// <response code="404">If the product is not found</response>
    [HttpGet("{id}")]
    [ProducesResponseType(typeof(Product), 200)]
    [ProducesResponseType(404)]
    public async Task<ActionResult<Product>> GetProduct(int id)
    {
        // ...
    }
}
```

### 3. API dokümantasyonunda neler bulunmalıdır?
**Cevap:**
API dokümantasyonunda bulunması gerekenler:
- Genel API açıklaması
- Authentication bilgileri
- Endpoint açıklamaları
- Request/Response örnekleri
- Hata kodları
- Rate limiting bilgileri

**Örnek Kod:**
```csharp
// Startup.cs
services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Product API",
        Version = "v1",
        Description = "Product management API with authentication",
        Contact = new OpenApiContact
        {
            Name = "API Support",
            Email = "support@company.com"
        },
        License = new OpenApiLicense
        {
            Name = "API License",
            Url = new Uri("https://company.com/license")
        }
    });

    // Authentication şeması
    c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Description = "JWT Authorization header using the Bearer scheme",
        Name = "Authorization",
        In = ParameterLocation.Header,
        Type = SecuritySchemeType.ApiKey
    });

    // Rate limiting bilgisi
    c.OperationFilter<RateLimitOperationFilter>();
});
```

### 4. API dokümantasyonu nasıl test edilir?
**Cevap:**
API dokümantasyonu testi için:
- Swagger UI üzerinden test
- Postman koleksiyonları
- Otomatik test araçları
- Dokümantasyon doğrulama

**Örnek Kod:**
```csharp
// Test controller
[ApiController]
[Route("api/[controller]")]
public class DocumentationTestsController : ControllerBase
{
    /// <summary>
    /// Tests API documentation
    /// </summary>
    /// <remarks>
    /// Sample request:
    ///
    ///     POST /api/documentationtests
    ///     {
    ///        "testCase": "string",
    ///        "expectedResult": "string"
    ///     }
    ///
    /// </remarks>
    [HttpPost]
    [ProducesResponseType(typeof(TestResult), 200)]
    [ProducesResponseType(400)]
    public async Task<ActionResult<TestResult>> TestDocumentation(TestRequest request)
    {
        // Test logic
    }
}

// Test model
public class TestRequest
{
    /// <summary>
    /// Test case description
    /// </summary>
    /// <example>Authentication test</example>
    public string TestCase { get; set; }

    /// <summary>
    /// Expected result
    /// </summary>
    /// <example>Success</example>
    public string ExpectedResult { get; set; }
}
```

### 5. API dokümantasyonu nasıl güncellenir?
**Cevap:**
API dokümantasyonu güncelleme:
- Versiyon kontrolü
- Değişiklik logu
- Otomatik güncelleme
- Dokümantasyon review

**Örnek Kod:**
```csharp
// Versioned documentation
services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Product API",
        Version = "v1",
        Description = "Product management API - Version 1"
    });

    c.SwaggerDoc("v2", new OpenApiInfo
    {
        Title = "Product API",
        Version = "v2",
        Description = "Product management API - Version 2"
    });
});

// Change log
/// <summary>
/// Products API controller
/// </summary>
/// <remarks>
/// Version History:
/// - 1.0: Initial version
/// - 1.1: Added pagination
/// - 2.0: Added new features
/// </remarks>
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // ...
}
```

## Best Practices
1. **Dokümantasyon Kalitesi**
   - Açık ve anlaşılır olun
   - Örnekler ekleyin
   - Güncel tutun
   - Tutarlı olun

2. **Dokümantasyon Araçları**
   - Swagger/OpenAPI kullanın
   - XML dokümantasyonu ekleyin
   - Test araçları kullanın
   - CI/CD entegrasyonu yapın

3. **Dokümantasyon Yönetimi**
   - Versiyon kontrolü yapın
   - Değişiklik logu tutun
   - Review süreci oluşturun
   - Otomatik güncelleme kullanın

## Kaynaklar
- [ASP.NET Core Web API Documentation](https://docs.microsoft.com/tr-tr/aspnet/core/web-api/advanced/swagger)
- [OpenAPI Specification](https://swagger.io/specification/)
- [API Documentation Best Practices](https://swagger.io/blog/api-documentation/best-practices-for-api-documentation/) 