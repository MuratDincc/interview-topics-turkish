# Tenant Çözümleme (Tenant Resolution)

## Genel Bakış

Tenant çözümleme (tenant resolution), gelen her HTTP isteğinin hangi tenant'a ait olduğunu belirleyen süreçtir. Bu bilgi, veri izolasyonu, yetkilendirme ve yapılandırma kararlarının temelini oluşturur. ASP.NET Core'da tenant çözümleme genellikle middleware katmanında gerçekleştirilir ve sonuç `IHttpContextAccessor` aracılığıyla uygulama genelinde paylaşılır.

Yaygın çözümleme stratejileri şunlardır: subdomain tabanlı, HTTP header tabanlı ve JWT claim tabanlı. Gerçek dünya uygulamalarında birden fazla strateji, öncelik sırasına göre zincirlenerek kullanılır.

## Temel Kavramlar

### 1. Tenant Context ve Servis Kayıtları

```csharp
// Tenant context arayüzü
public interface ITenantContext
{
    string TenantId { get; }
    string TenantName { get; }
    string Plan { get; }
    bool IsResolved { get; }
}

// Scoped yaşam döngüsü ile request başına yeni instance
public class TenantContext : ITenantContext
{
    public string TenantId { get; private set; } = string.Empty;
    public string TenantName { get; private set; } = string.Empty;
    public string Plan { get; private set; } = string.Empty;
    public bool IsResolved { get; private set; }

    public void Resolve(string tenantId, string tenantName, string plan)
    {
        if (string.IsNullOrWhiteSpace(tenantId))
            throw new ArgumentException("TenantId boş olamaz.", nameof(tenantId));

        TenantId = tenantId;
        TenantName = tenantName;
        Plan = plan;
        IsResolved = true;
    }
}

// DI kayıtları - Program.cs
// builder.Services.AddScoped<ITenantContext, TenantContext>();
// builder.Services.AddScoped<TenantContext>();

// Tenant model
public class Tenant
{
    public string Id { get; set; } = string.Empty;
    public string Name { get; set; } = string.Empty;
    public string Subdomain { get; set; } = string.Empty;
    public string Plan { get; set; } = string.Empty;
    public bool IsActive { get; set; }
}

// Tenant store arayüzü
public interface ITenantStore
{
    Task<Tenant?> GetByIdAsync(string tenantId, CancellationToken cancellationToken = default);
    Task<Tenant?> GetBySubdomainAsync(string subdomain, CancellationToken cancellationToken = default);
}
```

### 2. Subdomain Tabanlı Çözümleme

En yaygın SaaS pattern'ıdır. `acme.myapp.com` isteğinde `acme` subdomain'i tenant kimliği olarak kullanılır.

```csharp
public class SubdomainTenantResolver : ITenantResolver
{
    private readonly ITenantStore _tenantStore;
    private readonly ILogger<SubdomainTenantResolver> _logger;

    public SubdomainTenantResolver(
        ITenantStore tenantStore,
        ILogger<SubdomainTenantResolver> logger)
    {
        _tenantStore = tenantStore;
        _logger = logger;
    }

    public async Task<Tenant?> ResolveAsync(
        HttpContext httpContext,
        CancellationToken cancellationToken = default)
    {
        var host = httpContext.Request.Host.Host;

        if (string.IsNullOrWhiteSpace(host))
        {
            _logger.LogDebug("Host header bulunamadı, subdomain çözümleme atlanıyor.");
            return null;
        }

        var subdomain = ExtractSubdomain(host);

        if (subdomain is null)
        {
            _logger.LogDebug("Subdomain bulunamadı. Host: {Host}", host);
            return null;
        }

        _logger.LogDebug("Subdomain çözümleniyor: {Subdomain}", subdomain);

        var tenant = await _tenantStore.GetBySubdomainAsync(subdomain, cancellationToken);

        if (tenant is null)
        {
            _logger.LogWarning("Subdomain için tenant bulunamadı: {Subdomain}", subdomain);
        }

        return tenant;
    }

    private static string? ExtractSubdomain(string host)
    {
        // "acme.myapp.com" → "acme"
        // "myapp.com" → null (subdomain yok)
        // "localhost" → null
        var parts = host.Split('.');

        if (parts.Length < 3)
        {
            return null;
        }

        var subdomain = parts[0];

        // "www" subdomaini tenant değil
        if (subdomain.Equals("www", StringComparison.OrdinalIgnoreCase))
        {
            return null;
        }

        return subdomain;
    }
}
```

### 3. HTTP Header Tabanlı Çözümleme

API'ler ve mobil uygulamalar için yaygın yöntemdir. İstemci her istekte `X-Tenant-Id: acme` başlığı gönderir.

```csharp
public class HeaderTenantResolver : ITenantResolver
{
    private const string TenantIdHeader = "X-Tenant-Id";

    private readonly ITenantStore _tenantStore;
    private readonly ILogger<HeaderTenantResolver> _logger;

    public HeaderTenantResolver(
        ITenantStore tenantStore,
        ILogger<HeaderTenantResolver> logger)
    {
        _tenantStore = tenantStore;
        _logger = logger;
    }

    public async Task<Tenant?> ResolveAsync(
        HttpContext httpContext,
        CancellationToken cancellationToken = default)
    {
        if (!httpContext.Request.Headers.TryGetValue(TenantIdHeader, out var tenantIdValues))
        {
            _logger.LogDebug("{Header} header bulunamadı.", TenantIdHeader);
            return null;
        }

        var tenantId = tenantIdValues.FirstOrDefault();

        if (string.IsNullOrWhiteSpace(tenantId))
        {
            _logger.LogDebug("{Header} header değeri boş.", TenantIdHeader);
            return null;
        }

        // Header injection saldırısına karşı temel doğrulama
        if (tenantId.Length > 100 || tenantId.Any(c => !char.IsLetterOrDigit(c) && c != '-' && c != '_'))
        {
            _logger.LogWarning("Geçersiz tenant ID formatı tespit edildi: {TenantId}", tenantId);
            return null;
        }

        _logger.LogDebug("Header üzerinden tenant çözümleniyor: {TenantId}", tenantId);

        var tenant = await _tenantStore.GetByIdAsync(tenantId, cancellationToken);

        if (tenant is null)
        {
            _logger.LogWarning("Header tenant ID için tenant bulunamadı: {TenantId}", tenantId);
        }

        return tenant;
    }
}
```

### 4. JWT Claim Tabanlı Çözümleme

Kimlik doğrulama token'ı içine gömülü tenant bilgisi ile çözümleme yapılır. En güvenli yöntemdir çünkü tenant ID imzalı token'dan alınır ve değiştirilemez.

```csharp
public class ClaimTenantResolver : ITenantResolver
{
    // JWT token içindeki tenant claim adı
    private const string TenantIdClaimType = "tenant_id";

    private readonly ITenantStore _tenantStore;
    private readonly ILogger<ClaimTenantResolver> _logger;

    public ClaimTenantResolver(
        ITenantStore tenantStore,
        ILogger<ClaimTenantResolver> logger)
    {
        _tenantStore = tenantStore;
        _logger = logger;
    }

    public async Task<Tenant?> ResolveAsync(
        HttpContext httpContext,
        CancellationToken cancellationToken = default)
    {
        var user = httpContext.User;

        if (user?.Identity?.IsAuthenticated != true)
        {
            _logger.LogDebug("Kullanıcı kimlik doğrulaması yapılmamış, claim çözümleme atlanıyor.");
            return null;
        }

        var tenantIdClaim = user.FindFirst(TenantIdClaimType)
            ?? user.FindFirst($"http://schemas.myapp.com/claims/{TenantIdClaimType}");

        if (tenantIdClaim is null)
        {
            _logger.LogWarning("JWT token'ında tenant_id claim bulunamadı. Sub: {Sub}",
                user.FindFirst("sub")?.Value);
            return null;
        }

        var tenantId = tenantIdClaim.Value;

        _logger.LogDebug("Claim üzerinden tenant çözümleniyor: {TenantId}", tenantId);

        var tenant = await _tenantStore.GetByIdAsync(tenantId, cancellationToken);

        if (tenant is null)
        {
            _logger.LogWarning("Claim tenant ID için tenant bulunamadı: {TenantId}", tenantId);
        }

        return tenant;
    }
}

// JWT'ye tenant claim ekleme (token üretimi sırasında)
public class TenantAwareJwtService
{
    private readonly IConfiguration _configuration;
    private readonly ILogger<TenantAwareJwtService> _logger;

    public TenantAwareJwtService(
        IConfiguration configuration,
        ILogger<TenantAwareJwtService> logger)
    {
        _configuration = configuration;
        _logger = logger;
    }

    public string GenerateToken(string userId, string username, string tenantId, string role)
    {
        var key = new SymmetricSecurityKey(
            System.Text.Encoding.UTF8.GetBytes(_configuration["Jwt:Key"]!));
        var credentials = new Microsoft.IdentityModel.Tokens.SigningCredentials(
            key, Microsoft.IdentityModel.Tokens.SecurityAlgorithms.HmacSha256);

        var claims = new List<System.Security.Claims.Claim>
        {
            new(System.IdentityModel.Tokens.Jwt.JwtRegisteredClaimNames.Sub, userId),
            new(System.IdentityModel.Tokens.Jwt.JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new(System.Security.Claims.ClaimTypes.Name, username),
            new(System.Security.Claims.ClaimTypes.Role, role),
            // Tenant claim - token imzası ile korunur
            new("tenant_id", tenantId)
        };

        var token = new System.IdentityModel.Tokens.Jwt.JwtSecurityToken(
            issuer: _configuration["Jwt:Issuer"],
            audience: _configuration["Jwt:Audience"],
            claims: claims,
            expires: DateTime.UtcNow.AddHours(1),
            signingCredentials: credentials);

        return new System.IdentityModel.Tokens.Jwt.JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

### 5. Zincirlenmiş (Composite) Tenant Resolver

Birden fazla stratejiyi öncelik sırasına göre dener; ilk başarılı sonucu kullanır.

```csharp
public interface ITenantResolver
{
    Task<Tenant?> ResolveAsync(HttpContext httpContext, CancellationToken cancellationToken = default);
}

public class CompositeTenantResolver : ITenantResolver
{
    private readonly IEnumerable<ITenantResolver> _resolvers;
    private readonly ILogger<CompositeTenantResolver> _logger;

    public CompositeTenantResolver(
        IEnumerable<ITenantResolver> resolvers,
        ILogger<CompositeTenantResolver> logger)
    {
        _resolvers = resolvers;
        _logger = logger;
    }

    public async Task<Tenant?> ResolveAsync(
        HttpContext httpContext,
        CancellationToken cancellationToken = default)
    {
        foreach (var resolver in _resolvers)
        {
            var tenant = await resolver.ResolveAsync(httpContext, cancellationToken);

            if (tenant is not null)
            {
                _logger.LogDebug(
                    "Tenant {Resolver} ile çözümlendi: {TenantId}",
                    resolver.GetType().Name, tenant.Id);
                return tenant;
            }
        }

        _logger.LogDebug("Hiçbir resolver tenant bulamadı.");
        return null;
    }
}
```

### 6. Tenant Resolution Middleware

Tüm çözümleme mantığını bir araya getiren ASP.NET Core middleware'i. Request pipeline'ının erken aşamasında çalışır.

```csharp
public class TenantResolutionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<TenantResolutionMiddleware> _logger;

    // Tenant gerektirmeyen yollar (health check, swagger, vb.)
    private static readonly HashSet<string> _excludedPaths = new(StringComparer.OrdinalIgnoreCase)
    {
        "/health",
        "/healthz",
        "/ready",
        "/metrics",
        "/swagger",
        "/favicon.ico"
    };

    public TenantResolutionMiddleware(
        RequestDelegate next,
        ILogger<TenantResolutionMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(
        HttpContext httpContext,
        ITenantResolver tenantResolver,
        TenantContext tenantContext,
        CancellationToken cancellationToken = default)
    {
        var path = httpContext.Request.Path.Value ?? string.Empty;

        // Dışlanan yolları atla
        if (IsExcludedPath(path))
        {
            await _next(httpContext);
            return;
        }

        var tenant = await tenantResolver.ResolveAsync(httpContext, cancellationToken);

        if (tenant is null)
        {
            _logger.LogWarning(
                "Tenant çözümlenemedi. Path: {Path}, Host: {Host}",
                path,
                httpContext.Request.Host.Value);

            httpContext.Response.StatusCode = StatusCodes.Status400BadRequest;
            await httpContext.Response.WriteAsJsonAsync(new
            {
                error = "Tenant belirlenemedi.",
                code = "TENANT_NOT_RESOLVED"
            }, cancellationToken);
            return;
        }

        if (!tenant.IsActive)
        {
            _logger.LogWarning("Pasif tenant erişim denemesi: {TenantId}", tenant.Id);

            httpContext.Response.StatusCode = StatusCodes.Status403Forbidden;
            await httpContext.Response.WriteAsJsonAsync(new
            {
                error = "Tenant hesabı aktif değil.",
                code = "TENANT_INACTIVE"
            }, cancellationToken);
            return;
        }

        // Tenant context'ini doldur - scoped servis olduğu için bu request boyunca geçerli
        tenantContext.Resolve(tenant.Id, tenant.Name, tenant.Plan);

        // Tenant bilgisini response header'a ekle (debug için - üretimde kaldırılabilir)
        httpContext.Response.Headers["X-Tenant-Id"] = tenant.Id;

        // Correlation için log scope ekle
        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["TenantId"] = tenant.Id,
            ["TenantName"] = tenant.Name
        }))
        {
            await _next(httpContext);
        }
    }

    private static bool IsExcludedPath(string path)
    {
        foreach (var excluded in _excludedPaths)
        {
            if (path.StartsWith(excluded, StringComparison.OrdinalIgnoreCase))
            {
                return true;
            }
        }
        return false;
    }
}

// Middleware extension metodu
public static class TenantResolutionMiddlewareExtensions
{
    public static IApplicationBuilder UseTenantResolution(this IApplicationBuilder app)
    {
        return app.UseMiddleware<TenantResolutionMiddleware>();
    }
}
```

### 7. Program.cs Yapılandırması

```csharp
// Program.cs - Multi-tenancy DI yapılandırması
var builder = WebApplication.CreateBuilder(args);

// Tenant context - Scoped: her request için yeni instance
builder.Services.AddScoped<TenantContext>();
builder.Services.AddScoped<ITenantContext>(sp => sp.GetRequiredService<TenantContext>());

// Tenant store - üretime göre implementasyon seçin
builder.Services.AddScoped<ITenantStore, DatabaseTenantStore>();

// Tenant resolver'ları - öncelik sırasına göre kaydet
// 1. Önce claim (güvenli), 2. Subdomain, 3. Header (fallback)
builder.Services.AddScoped<ITenantResolver, CompositeTenantResolver>();
builder.Services.AddScoped<ClaimTenantResolver>();
builder.Services.AddScoped<SubdomainTenantResolver>();
builder.Services.AddScoped<HeaderTenantResolver>();

// CompositeTenantResolver için resolver listesini yapılandır
builder.Services.AddScoped<IEnumerable<ITenantResolver>>(sp => new ITenantResolver[]
{
    sp.GetRequiredService<ClaimTenantResolver>(),
    sp.GetRequiredService<SubdomainTenantResolver>(),
    sp.GetRequiredService<HeaderTenantResolver>()
});

// Tenant-aware DbContext
builder.Services.AddDbContext<TenantAwareDbContext>((sp, options) =>
{
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection"));
});

var app = builder.Build();

// Middleware sırası önemli:
// 1. Routing
// 2. Authentication (JWT doğrulama)
// 3. Tenant resolution (claim resolver için auth gerekli)
// 4. Authorization
app.UseRouting();
app.UseAuthentication();
app.UseTenantResolution();   // Authentication'dan sonra, Authorization'dan önce
app.UseAuthorization();

app.MapControllers();
app.Run();
```

### 8. Önbellekli Tenant Store

Her request'te veritabanı sorgusu yapmak yerine tenant bilgilerini önbelleğe alarak performansı artırma.

```csharp
public class CachedTenantStore : ITenantStore
{
    private readonly ITenantStore _inner;
    private readonly IMemoryCache _cache;
    private readonly ILogger<CachedTenantStore> _logger;

    private static readonly TimeSpan CacheDuration = TimeSpan.FromMinutes(5);

    public CachedTenantStore(
        ITenantStore inner,
        IMemoryCache cache,
        ILogger<CachedTenantStore> logger)
    {
        _inner = inner;
        _cache = cache;
        _logger = logger;
    }

    public async Task<Tenant?> GetByIdAsync(string tenantId, CancellationToken cancellationToken = default)
    {
        var cacheKey = $"tenant:id:{tenantId}";

        if (_cache.TryGetValue(cacheKey, out Tenant? cached))
        {
            _logger.LogDebug("Tenant önbellekten alındı: {TenantId}", tenantId);
            return cached;
        }

        var tenant = await _inner.GetByIdAsync(tenantId, cancellationToken);

        if (tenant is not null)
        {
            _cache.Set(cacheKey, tenant, new MemoryCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = CacheDuration,
                SlidingExpiration = TimeSpan.FromMinutes(2),
                Size = 1
            });
        }

        return tenant;
    }

    public async Task<Tenant?> GetBySubdomainAsync(string subdomain, CancellationToken cancellationToken = default)
    {
        var cacheKey = $"tenant:subdomain:{subdomain.ToLowerInvariant()}";

        if (_cache.TryGetValue(cacheKey, out Tenant? cached))
        {
            _logger.LogDebug("Tenant subdomain önbellekten alındı: {Subdomain}", subdomain);
            return cached;
        }

        var tenant = await _inner.GetBySubdomainAsync(subdomain, cancellationToken);

        if (tenant is not null)
        {
            _cache.Set(cacheKey, tenant, new MemoryCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = CacheDuration,
                Size = 1
            });
        }

        return tenant;
    }

    // Tenant güncellendiğinde önbelleği temizle
    public void InvalidateTenant(string tenantId, string? subdomain = null)
    {
        _cache.Remove($"tenant:id:{tenantId}");

        if (subdomain is not null)
        {
            _cache.Remove($"tenant:subdomain:{subdomain.ToLowerInvariant()}");
        }

        _logger.LogInformation("Tenant önbelleği temizlendi: {TenantId}", tenantId);
    }
}
```

### 9. Controller'da Tenant Bilgisine Erişim

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly ITenantContext _tenantContext;
    private readonly ITenantRepository<Order> _orderRepository;
    private readonly ILogger<OrdersController> _logger;

    public OrdersController(
        ITenantContext tenantContext,
        ITenantRepository<Order> orderRepository,
        ILogger<OrdersController> logger)
    {
        _tenantContext = tenantContext;
        _orderRepository = orderRepository;
        _logger = logger;
    }

    [HttpGet]
    public async Task<IActionResult> GetOrders(CancellationToken cancellationToken)
    {
        // TenantContext middleware tarafından doldurulmuştur
        _logger.LogInformation(
            "Siparişler listeleniyor. Tenant: {TenantId}", _tenantContext.TenantId);

        var orders = await _orderRepository.GetAllAsync(cancellationToken);
        return Ok(orders);
    }

    [HttpGet("{id:guid}")]
    public async Task<IActionResult> GetOrder(Guid id, CancellationToken cancellationToken)
    {
        var order = await _orderRepository.GetByIdAsync(id, cancellationToken);

        if (order is null)
        {
            return NotFound();
        }

        return Ok(order);
    }

    [HttpPost]
    public async Task<IActionResult> CreateOrder(
        [FromBody] CreateOrderRequest request,
        CancellationToken cancellationToken)
    {
        if (!_tenantContext.IsResolved)
        {
            return BadRequest("Tenant context çözümlenmemiş.");
        }

        var order = new Order
        {
            Id = Guid.NewGuid(),
            OrderNumber = $"ORD-{DateTime.UtcNow:yyyyMMdd}-{Guid.NewGuid().ToString()[..8].ToUpper()}",
            TotalAmount = request.TotalAmount,
            Status = OrderStatus.Pending
        };

        await _orderRepository.AddAsync(order, cancellationToken);

        return CreatedAtAction(nameof(GetOrder), new { id = order.Id }, order);
    }
}

public record CreateOrderRequest(decimal TotalAmount);
```

### 10. Özel Durum Sınıfları

```csharp
public class TenantNotFoundException : Exception
{
    public string TenantId { get; }

    public TenantNotFoundException(string tenantId)
        : base($"Tenant bulunamadı: {tenantId}")
    {
        TenantId = tenantId;
    }
}

public class TenantInactiveException : Exception
{
    public string TenantId { get; }

    public TenantInactiveException(string tenantId)
        : base($"Tenant hesabı aktif değil: {tenantId}")
    {
        TenantId = tenantId;
    }
}

public class TenantSecurityException : Exception
{
    public TenantSecurityException(string message) : base(message) { }
}
```

## Best Practices

### 1. Middleware Sıralaması Kritiktir
- Authentication middleware'i, tenant resolution'dan **önce** çalışmalıdır (claim resolver için)
- Tenant resolution, authorization'dan **önce** olmalıdır
- Yanlış sıralama, claim resolver'ın kullanıcı kimliğine erişememesine yol açar

### 2. Güvenli Tenant ID Doğrulaması
- Header tabanlı çözümlemede tenant ID'yi whitelist regex ile doğrulayın
- Subdomain'den alınan değerleri SQL/path injection'a karşı sanitize edin
- Claim tabanlı çözümleme en güvenlidir; token imzası sayesinde değiştirilemez

### 3. Dışlanan Yolları Doğru Yapılandırın
- Health check, metrics ve swagger endpointleri tenant gerektirmemeli
- Tenant çözümlenemediğinde doğrudan 400/403 döndürün; downstream servislerin null tenant ile çalışmasına izin vermeyin

### 4. Önbellekleme Stratejisi
- Tenant bilgilerini kısa süreli önbelleğe alın (1-5 dakika)
- Tenant güncellendiğinde (plan değişimi, pasif hale getirilme) önbelleği invalidate edin
- Redis ile distributed cache kullanın (çok instance ortamlar için)

### 5. Gözlemlenebilirlik (Observability)
- Her log kaydına `TenantId` ekleyin (Serilog enricher veya log scope)
- Tenant bazlı metrik toplayın (istek sayısı, hata oranı)
- Başarısız tenant çözümleme girişimlerini alert olarak işaretleyin

## Sık Sorulan Sorular

### 1. Soru: Subdomain, header ve claim çözümleme yöntemleri arasında güvenlik açısından fark nedir?

**Cevap:**
- **Claim tabanlı**: En güvenli. JWT token, sunucu tarafında imzalandığı için tenant ID tahrif edilemez. Ancak token yenileme sürecinde tenant değişikliği yönetilmelidir.
- **Header tabanlı**: Orta güvenlik. İstemci header'ı değiştirebilir, bu nedenle header'daki tenant ID'nin veritabanında doğrulanması şarttır. "Tenant spoofing" saldırısına açıktır.
- **Subdomain tabanlı**: DNS üzerinde kontrol sahibi olunduğu varsayımıyla güvenlidir; ancak DNS hijacking senaryolarında risk taşır.

Üretimde genellikle claim + subdomain kombinasyonu tercih edilir: subdomain, doğru tenant sayfasını yükler; claim ise API çağrılarında kullanılır.

**Örnek Kod:**
```csharp
// Güvenli: Claim resolver önce çalışır, subdomain fallback
builder.Services.AddScoped<IEnumerable<ITenantResolver>>(sp => new ITenantResolver[]
{
    sp.GetRequiredService<ClaimTenantResolver>(),    // JWT claim (en güvenli)
    sp.GetRequiredService<SubdomainTenantResolver>() // Subdomain fallback
});
```

### 2. Soru: Background job ve worker servislerinde tenant context nasıl yönetilir?

**Cevap:** Background job'larda HTTP context olmadığı için middleware çalışmaz. Tenant context'i açıkça set etmek gerekir. Job'un hangi tenant için çalıştığı, job parametresi veya veritabanından okunarak belirlenir ve `TenantContext.Resolve()` el ile çağrılır.

**Örnek Kod:**
```csharp
public class TenantAwareBackgroundJob
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<TenantAwareBackgroundJob> _logger;

    public TenantAwareBackgroundJob(
        IServiceScopeFactory scopeFactory,
        ILogger<TenantAwareBackgroundJob> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    public async Task ExecuteForTenantAsync(string tenantId, CancellationToken cancellationToken)
    {
        // Her tenant için ayrı DI scope oluştur
        using var scope = _scopeFactory.CreateScope();

        var tenantContext = scope.ServiceProvider.GetRequiredService<TenantContext>();
        var tenantStore = scope.ServiceProvider.GetRequiredService<ITenantStore>();

        var tenant = await tenantStore.GetByIdAsync(tenantId, cancellationToken)
            ?? throw new TenantNotFoundException(tenantId);

        // Tenant context'ini el ile set et
        tenantContext.Resolve(tenant.Id, tenant.Name, tenant.Plan);

        _logger.LogInformation("Background job çalıştırılıyor. Tenant: {TenantId}", tenantId);

        var orderRepo = scope.ServiceProvider.GetRequiredService<ITenantRepository<Order>>();
        var orders = await orderRepo.GetAllAsync(cancellationToken);

        // İşlemler...
    }
}
```

### 3. Soru: Tenant çözümleme başarısız olduğunda ne yapılmalıdır?

**Cevap:** Strateji uygulamanın tipine göre değişir:
- **API**: 400 Bad Request veya 401 Unauthorized döndürün.
- **Web uygulaması**: Tenant seçim sayfasına veya hata sayfasına yönlendirin.
- **Webhook receiver**: İstek loglayın, 400 döndürün.

Asla null tenant ile işleme devam etmeyin; bu, Global Query Filter'ı etkisiz bırakır ve tenant sızıntısına yol açar.

**Örnek Kod:**
```csharp
if (tenant is null)
{
    httpContext.Response.StatusCode = StatusCodes.Status400BadRequest;
    await httpContext.Response.WriteAsJsonAsync(new
    {
        error = "Tenant belirlenemedi. Lütfen geçerli bir subdomain veya X-Tenant-Id header değeri sağlayın.",
        code = "TENANT_NOT_RESOLVED"
    }, cancellationToken);
    return; // Pipeline'ı durdur
}
```

### 4. Soru: Aynı kullanıcı birden fazla tenant'a üye olabilir mi?

**Cevap:** Evet. Bu senaryoda JWT claim içine `tenant_id` koymak yerine, kullanıcının her oturum açışında hangi tenant için giriş yaptığını belirleyen bir akış tasarlanır. Token, o oturum için seçilen tenant'ın ID'sini taşır. Alternatif olarak, token içine kullanıcının erişebildiği tüm tenant'ların listesi konulur ve request başına aktif tenant header ile bildirilir; middleware bu kombinasyonu doğrular.

**Örnek Kod:**
```csharp
// Çoklu tenant üyeliği: token'da authorized tenants listesi
// "authorized_tenants": ["acme", "globex"]
// Header: X-Active-Tenant: acme

public class MultiTenantClaimResolver : ITenantResolver
{
    public async Task<Tenant?> ResolveAsync(HttpContext httpContext, CancellationToken cancellationToken = default)
    {
        var user = httpContext.User;
        if (user?.Identity?.IsAuthenticated != true) return null;

        // İstenen aktif tenant
        var requestedTenantId = httpContext.Request.Headers["X-Active-Tenant"].FirstOrDefault();
        if (string.IsNullOrWhiteSpace(requestedTenantId)) return null;

        // Kullanıcının yetkili olduğu tenant'lar
        var authorizedTenants = user.FindAll("authorized_tenants").Select(c => c.Value).ToHashSet();

        if (!authorizedTenants.Contains(requestedTenantId))
        {
            return null; // Yetkisiz tenant erişimi
        }

        return await _tenantStore.GetByIdAsync(requestedTenantId, cancellationToken);
    }
}
```

### 5. Soru: Tenant çözümleme performansını nasıl ölçersiniz?

**Cevap:** Her request'te tenant çözümleme süresi ölçülmeli ve threshold aşıldığında uyarı verilmelidir. `Stopwatch` ile süre ölçümü, Application Insights / OpenTelemetry ile metrik toplama ve önbellek hit/miss oranı izlenmesi temel yaklaşımlardır.

**Örnek Kod:**
```csharp
public class InstrumentedTenantResolver : ITenantResolver
{
    private readonly ITenantResolver _inner;
    private readonly ILogger<InstrumentedTenantResolver> _logger;

    private static readonly System.Diagnostics.ActivitySource ActivitySource =
        new("MyApp.TenantResolution");

    public InstrumentedTenantResolver(ITenantResolver inner, ILogger<InstrumentedTenantResolver> logger)
    {
        _inner = inner;
        _logger = logger;
    }

    public async Task<Tenant?> ResolveAsync(HttpContext httpContext, CancellationToken cancellationToken = default)
    {
        using var activity = ActivitySource.StartActivity("TenantResolution");
        var sw = System.Diagnostics.Stopwatch.StartNew();

        try
        {
            var tenant = await _inner.ResolveAsync(httpContext, cancellationToken);
            sw.Stop();

            activity?.SetTag("tenant.id", tenant?.Id ?? "not_resolved");
            activity?.SetTag("tenant.resolution_ms", sw.ElapsedMilliseconds);

            if (sw.ElapsedMilliseconds > 100)
            {
                _logger.LogWarning("Yavaş tenant çözümleme: {ElapsedMs}ms", sw.ElapsedMilliseconds);
            }

            return tenant;
        }
        catch (Exception ex)
        {
            activity?.SetStatus(System.Diagnostics.ActivityStatusCode.Error, ex.Message);
            throw;
        }
    }
}
```

### 6. Soru: Tenant çözümleme ile rate limiting nasıl entegre edilir?

**Cevap:** Tenant bazlı rate limiting için tenant çözümleme, rate limiting middleware'inden önce tamamlanmalıdır. ASP.NET Core 7+'da yerleşik rate limiter'ın `partitionKey` olarak `TenantId` kullanılması önerilir. Her tenant'ın planına göre farklı limitler atanabilir.

**Örnek Kod:**
```csharp
// Program.cs - Tenant bazlı rate limiting
builder.Services.AddRateLimiter(options =>
{
    options.AddPolicy("TenantPolicy", httpContext =>
    {
        // Middleware çalıştıktan sonra TenantContext doldurulmuş olmalı
        var tenantContext = httpContext.RequestServices.GetService<ITenantContext>();
        var tenantId = tenantContext?.TenantId ?? "anonymous";
        var plan = tenantContext?.Plan ?? "free";

        // Plan bazlı limit belirleme
        var limit = plan switch
        {
            "enterprise" => 10000,
            "professional" => 1000,
            "starter" => 100,
            _ => 10 // free
        };

        return RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: tenantId,
            factory: _ => new System.Threading.RateLimiting.FixedWindowRateLimiterOptions
            {
                PermitLimit = limit,
                Window = TimeSpan.FromMinutes(1)
            });
    });
});

// Pipeline sırası: TenantResolution → RateLimiter
app.UseTenantResolution();
app.UseRateLimiter();
```

## Kaynaklar

- [ASP.NET Core Middleware](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/)
- [Finbuckle.MultiTenant - Tenant Resolution](https://www.finbuckle.com/MultiTenant/Docs/TenantResolution)
- [ASP.NET Core Rate Limiting](https://docs.microsoft.com/en-us/aspnet/core/performance/rate-limit)
- [JWT Claims Best Practices](https://datatracker.ietf.org/doc/html/rfc7519#section-4)
- [IMemoryCache in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/performance/caching/memory)
- [OpenTelemetry .NET](https://opentelemetry.io/docs/languages/dotnet/)
