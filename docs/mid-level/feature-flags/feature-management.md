# Feature Management

## Genel Bakış

`Microsoft.FeatureManagement` kütüphanesi, .NET ve ASP.NET Core uygulamalarında feature flag yönetimini standart hale getiren resmi Microsoft kütüphanesidir. Bu kütüphane; yapılandırma tabanlı flag tanımlarını, çeşitli yerleşik filtreleri ve genişletilebilir bir mimariyi bir arada sunar. Bu bölümde kütüphanenin tüm bileşenlerini, kurulumunu, kullanımını ve mülakat sorularını ele alacağız.

## Kurulum ve Temel Yapılandırma

### NuGet Paketleri

```bash
dotnet add package Microsoft.FeatureManagement.AspNetCore
# Azure App Configuration entegrasyonu için:
dotnet add package Microsoft.Azure.AppConfiguration.AspNetCore
```

### appsettings.json Yapılandırması

```json
{
  "FeatureManagement": {
    "NewDashboard": true,
    "BetaCheckout": false,
    "DarkMode": {
      "EnabledFor": [
        {
          "Name": "Percentage",
          "Parameters": {
            "Value": 30
          }
        }
      ]
    },
    "MaintenanceMode": {
      "EnabledFor": [
        {
          "Name": "TimeWindow",
          "Parameters": {
            "Start": "2024-03-01T00:00:00",
            "End": "2024-03-01T04:00:00"
          }
        }
      ]
    }
  }
}
```

### Program.cs ile Servis Kaydı

```csharp
var builder = WebApplication.CreateBuilder(args);

// Temel feature management kaydı
builder.Services.AddFeatureManagement();

// Özel filtrelerle birlikte kayıt
builder.Services.AddFeatureManagement()
    .AddFeatureFilter<PercentageFilter>()
    .AddFeatureFilter<TimeWindowFilter>()
    .AddFeatureFilter<TargetingFilter>()
    .AddFeatureFilter<CustomUserRoleFilter>();

var app = builder.Build();
app.Run();
```

## IFeatureManager Kullanımı

### Temel Sorgulama

```csharp
public class ProductService
{
    private readonly IFeatureManager _featureManager;
    private readonly ILogger<ProductService> _logger;

    public ProductService(
        IFeatureManager featureManager,
        ILogger<ProductService> logger)
    {
        _featureManager = featureManager;
        _logger = logger;
    }

    public async Task<IEnumerable<Product>> GetProductsAsync()
    {
        // Basit boolean flag kontrolü
        if (await _featureManager.IsEnabledAsync("NewProductListing"))
        {
            _logger.LogInformation("Yeni ürün listeleme algoritması kullanılıyor.");
            return await GetProductsWithNewAlgorithmAsync();
        }

        return await GetProductsWithLegacyAlgorithmAsync();
    }

    public async Task<ProductDetails> GetProductDetailsAsync(int productId)
    {
        var details = await FetchProductDetailsAsync(productId);

        // Birden fazla flag kontrolü
        if (await _featureManager.IsEnabledAsync("EnhancedProductImages"))
        {
            details.Images = await FetchHighResImagesAsync(productId);
        }

        if (await _featureManager.IsEnabledAsync("ProductRecommendations"))
        {
            details.Recommendations = await FetchRecommendationsAsync(productId);
        }

        return details;
    }

    private Task<IEnumerable<Product>> GetProductsWithNewAlgorithmAsync()
        => Task.FromResult(Enumerable.Empty<Product>());

    private Task<IEnumerable<Product>> GetProductsWithLegacyAlgorithmAsync()
        => Task.FromResult(Enumerable.Empty<Product>());

    private Task<ProductDetails> FetchProductDetailsAsync(int id)
        => Task.FromResult(new ProductDetails());

    private Task<IEnumerable<string>> FetchHighResImagesAsync(int id)
        => Task.FromResult(Enumerable.Empty<string>());

    private Task<IEnumerable<Product>> FetchRecommendationsAsync(int id)
        => Task.FromResult(Enumerable.Empty<Product>());
}
```

### Tüm Aktif Flag'lerin Listelenmesi

```csharp
public class FeatureDiagnosticsService
{
    private readonly IFeatureManager _featureManager;

    public FeatureDiagnosticsService(IFeatureManager featureManager)
    {
        _featureManager = featureManager;
    }

    public async Task<Dictionary<string, bool>> GetAllFeatureStatesAsync(
        IEnumerable<string> featureNames)
    {
        var result = new Dictionary<string, bool>();

        foreach (var featureName in featureNames)
        {
            result[featureName] = await _featureManager.IsEnabledAsync(featureName);
        }

        return result;
    }

    // IFeatureManager.GetFeatureNamesAsync() ile tüm tanımlı flag'ler
    public async Task<List<string>> GetAllDefinedFeaturesAsync()
    {
        var features = new List<string>();
        await foreach (var feature in _featureManager.GetFeatureNamesAsync())
        {
            features.Add(feature);
        }
        return features;
    }
}
```

## Feature Gate Attribute

### Controller Seviyesinde Kullanım

```csharp
using Microsoft.FeatureManagement.Mvc;

[ApiController]
[Route("api/[controller]")]
public class BetaController : ControllerBase
{
    // Tüm controller'ı flag'e bağla
    // Flag kapalıysa 404 döner
    [FeatureGate("BetaApi")]
    [HttpGet]
    public IActionResult GetBetaData()
    {
        return Ok(new { message = "Beta API'ye hoş geldiniz!" });
    }
}

[ApiController]
[Route("api/[controller]")]
public class CheckoutController : ControllerBase
{
    private readonly IFeatureManager _featureManager;

    public CheckoutController(IFeatureManager featureManager)
    {
        _featureManager = featureManager;
    }

    // Tekli action seviyesinde feature gate
    [FeatureGate("NewCheckoutFlow")]
    [HttpPost("new")]
    public async Task<IActionResult> NewCheckout([FromBody] CheckoutRequest request)
    {
        return Ok(new { message = "Yeni ödeme akışı tamamlandı." });
    }

    // Birden fazla flag ile VE mantığı (tüm flag'ler açık olmalı)
    [FeatureGate(RequirementType.All, "NewCheckoutFlow", "PaymentGatewayV2")]
    [HttpPost("premium")]
    public async Task<IActionResult> PremiumCheckout([FromBody] CheckoutRequest request)
    {
        return Ok(new { message = "Premium ödeme akışı tamamlandı." });
    }

    // Birden fazla flag ile VEYA mantığı (en az biri açık olmalı)
    [FeatureGate(RequirementType.Any, "ExpressCheckout", "OneClickBuy")]
    [HttpPost("fast")]
    public async Task<IActionResult> FastCheckout([FromBody] CheckoutRequest request)
    {
        return Ok(new { message = "Hızlı ödeme akışı tamamlandı." });
    }
}
```

### Feature Gate ile Özel 404 Sayfası

```csharp
// Program.cs
app.UseEndpoints(endpoints =>
{
    endpoints.MapControllers();
});

// Özel IDisabledFeaturesHandler implementasyonu
public class RedirectDisabledFeatureHandler : IDisabledFeaturesHandler
{
    public Task HandleDisabledFeatures(
        IEnumerable<string> features,
        ActionExecutingContext context)
    {
        // Devre dışı bir özelliğe erişildiğinde yönlendirme
        context.Result = new RedirectResult("/feature-unavailable");
        return Task.CompletedTask;
    }
}

// Program.cs'de kayıt
builder.Services.AddFeatureManagement()
    .UseDisabledFeaturesHandler(new RedirectDisabledFeatureHandler());
```

## Feature Filters

### Yerleşik Filtreler

#### PercentageFilter - Yüzde Tabanlı Filtre

```csharp
// appsettings.json
{
  "FeatureManagement": {
    "NewSearchAlgorithm": {
      "EnabledFor": [
        {
          "Name": "Microsoft.Percentage",
          "Parameters": {
            "Value": 25
          }
        }
      ]
    }
  }
}

// Program.cs
builder.Services.AddFeatureManagement()
    .AddFeatureFilter<PercentageFilter>();
```

#### TimeWindowFilter - Zaman Penceresi Filtresi

```csharp
// appsettings.json
{
  "FeatureManagement": {
    "HolidaySale": {
      "EnabledFor": [
        {
          "Name": "Microsoft.TimeWindow",
          "Parameters": {
            "Start": "2024-12-24T00:00:00+03:00",
            "End": "2024-12-26T23:59:59+03:00"
          }
        }
      ]
    }
  }
}
```

#### TargetingFilter - Kullanıcı Hedefleme Filtresi

```csharp
// appsettings.json
{
  "FeatureManagement": {
    "EarlyAccess": {
      "EnabledFor": [
        {
          "Name": "Microsoft.Targeting",
          "Parameters": {
            "Audience": {
              "Users": ["alice@example.com", "bob@example.com"],
              "Groups": [
                {
                  "Name": "BetaTesters",
                  "RolloutPercentage": 100
                },
                {
                  "Name": "PremiumUsers",
                  "RolloutPercentage": 50
                }
              ],
              "DefaultRolloutPercentage": 0
            }
          }
        }
      ]
    }
  }
}

// ITargetingContextAccessor implementasyonu
public class HttpContextTargetingContextAccessor : ITargetingContextAccessor
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public HttpContextTargetingContextAccessor(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    public ValueTask<TargetingContext> GetContextAsync()
    {
        var httpContext = _httpContextAccessor.HttpContext;
        var user = httpContext?.User;

        var targetingContext = new TargetingContext
        {
            UserId = user?.Identity?.Name ?? "anonymous",
            Groups = GetUserGroups(user)
        };

        return new ValueTask<TargetingContext>(targetingContext);
    }

    private IEnumerable<string> GetUserGroups(ClaimsPrincipal? user)
    {
        if (user == null) return Enumerable.Empty<string>();

        var groups = new List<string>();

        if (user.IsInRole("BetaTesters")) groups.Add("BetaTesters");
        if (user.IsInRole("PremiumUsers")) groups.Add("PremiumUsers");
        if (user.IsInRole("Employees")) groups.Add("Employees");

        return groups;
    }
}

// Program.cs
builder.Services.AddHttpContextAccessor();
builder.Services.AddSingleton<ITargetingContextAccessor, HttpContextTargetingContextAccessor>();
builder.Services.AddFeatureManagement()
    .WithTargeting<HttpContextTargetingContextAccessor>()
    .AddFeatureFilter<TargetingFilter>();
```

### Özel Feature Filter Implementasyonu

```csharp
// Filtre parametreleri için POCO
public class SubscriptionFilterSettings
{
    public List<string> AllowedPlans { get; set; } = new();
    public bool AllowTrial { get; set; }
}

// Özel filtre implementasyonu
[FilterAlias("SubscriptionLevel")]
public class SubscriptionLevelFilter : IFeatureFilter
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    private readonly ILogger<SubscriptionLevelFilter> _logger;

    public SubscriptionLevelFilter(
        IHttpContextAccessor httpContextAccessor,
        ILogger<SubscriptionLevelFilter> logger)
    {
        _httpContextAccessor = httpContextAccessor;
        _logger = logger;
    }

    public Task<bool> EvaluateAsync(FeatureFilterEvaluationContext context)
    {
        var settings = context.Parameters
            .Get<SubscriptionFilterSettings>()
            ?? new SubscriptionFilterSettings();

        var httpContext = _httpContextAccessor.HttpContext;
        if (httpContext == null) return Task.FromResult(false);

        var user = httpContext.User;
        if (!user.Identity?.IsAuthenticated ?? true)
        {
            return Task.FromResult(false);
        }

        var userPlan = user.FindFirstValue("subscription_plan") ?? "free";
        var isTrial = user.FindFirstValue("is_trial") == "true";

        if (isTrial && settings.AllowTrial)
        {
            _logger.LogDebug("Deneme kullanıcısına özellik erişimi verildi: {Feature}",
                context.FeatureName);
            return Task.FromResult(true);
        }

        var isAllowed = settings.AllowedPlans.Contains(userPlan,
            StringComparer.OrdinalIgnoreCase);

        _logger.LogDebug(
            "Abonelik filtresi sonucu - Kullanıcı planı: {Plan}, İzin verilen: {Allowed}",
            userPlan, isAllowed);

        return Task.FromResult(isAllowed);
    }
}

// appsettings.json
{
  "FeatureManagement": {
    "AdvancedReporting": {
      "EnabledFor": [
        {
          "Name": "SubscriptionLevel",
          "Parameters": {
            "AllowedPlans": ["Professional", "Enterprise"],
            "AllowTrial": true
          }
        }
      ]
    }
  }
}

// Program.cs
builder.Services.AddHttpContextAccessor();
builder.Services.AddFeatureManagement()
    .AddFeatureFilter<SubscriptionLevelFilter>();
```

## IFeatureDefinitionProvider - Özel Tanım Kaynağı

```csharp
// Veritabanından flag tanımlarını okuyan provider
public class DatabaseFeatureDefinitionProvider : IFeatureDefinitionProvider
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly IMemoryCache _cache;
    private readonly TimeSpan _cacheDuration = TimeSpan.FromMinutes(5);

    public DatabaseFeatureDefinitionProvider(
        IServiceScopeFactory scopeFactory,
        IMemoryCache cache)
    {
        _scopeFactory = scopeFactory;
        _cache = cache;
    }

    public async IAsyncEnumerable<FeatureDefinition> GetAllFeatureDefinitionsAsync()
    {
        var definitions = await GetCachedDefinitionsAsync();
        foreach (var definition in definitions)
        {
            yield return definition;
        }
    }

    public async Task<FeatureDefinition?> GetFeatureDefinitionAsync(string featureName)
    {
        var definitions = await GetCachedDefinitionsAsync();
        return definitions.FirstOrDefault(d => d.Name == featureName);
    }

    private async Task<List<FeatureDefinition>> GetCachedDefinitionsAsync()
    {
        return await _cache.GetOrCreateAsync("feature_definitions", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = _cacheDuration;

            using var scope = _scopeFactory.CreateScope();
            var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();

            var flags = await dbContext.FeatureFlags
                .Where(f => f.IsActive)
                .ToListAsync();

            return flags.Select(f => new FeatureDefinition
            {
                Name = f.Name,
                EnabledFor = f.IsGloballyEnabled
                    ? new List<FeatureFilterConfiguration>()
                    : new List<FeatureFilterConfiguration>
                    {
                        new FeatureFilterConfiguration
                        {
                            Name = "Percentage",
                            Parameters = new ConfigurationBuilder()
                                .AddInMemoryCollection(new Dictionary<string, string?>
                                {
                                    ["Value"] = f.RolloutPercentage.ToString()
                                })
                                .Build()
                        }
                    }
            }).ToList();
        }) ?? new List<FeatureDefinition>();
    }
}

// Veritabanı entity
public class FeatureFlagEntity
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public bool IsActive { get; set; }
    public bool IsGloballyEnabled { get; set; }
    public int RolloutPercentage { get; set; }
    public DateTime? ExpiresAt { get; set; }
    public string? Description { get; set; }
}
```

## Middleware ile Feature Flag Kontrolü

```csharp
public class FeatureFlagMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<FeatureFlagMiddleware> _logger;

    public FeatureFlagMiddleware(
        RequestDelegate next,
        ILogger<FeatureFlagMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context, IFeatureManager featureManager)
    {
        // Bakım modu kontrolü
        if (await featureManager.IsEnabledAsync("MaintenanceMode"))
        {
            var path = context.Request.Path.Value ?? string.Empty;

            // Sağlık kontrol endpoint'i ve admin paneline izin ver
            if (!path.StartsWith("/health") && !path.StartsWith("/admin"))
            {
                _logger.LogWarning("Bakım modu aktif. İstek engellendi: {Path}", path);

                context.Response.StatusCode = StatusCodes.Status503ServiceUnavailable;
                context.Response.Headers["Retry-After"] = "3600";
                await context.Response.WriteAsJsonAsync(new
                {
                    error = "Hizmet bakımda",
                    message = "Sistemimiz şu anda bakım modundadır. Lütfen daha sonra tekrar deneyin.",
                    retryAfter = 3600
                });
                return;
            }
        }

        // API sürüm kontrolü
        if (context.Request.Path.StartsWithSegments("/api/v2"))
        {
            if (!await featureManager.IsEnabledAsync("ApiV2"))
            {
                context.Response.StatusCode = StatusCodes.Status404NotFound;
                await context.Response.WriteAsJsonAsync(new
                {
                    error = "API V2 henüz kullanılabilir değil"
                });
                return;
            }
        }

        await _next(context);
    }
}
```

## Mülakat Soruları

### 1. Soru

**Microsoft.FeatureManagement kütüphanesini ASP.NET Core'a nasıl entegre edersiniz?**

**Cevap:** Entegrasyon üç adımda gerçekleştirilir: NuGet paketi kurulumu, servis kaydı ve yapılandırma. `Microsoft.FeatureManagement.AspNetCore` paketi yüklenir. `Program.cs`'de `builder.Services.AddFeatureManagement()` çağrısı yapılır; gerekirse özel filtreler `.AddFeatureFilter<T>()` ile zincirlenir. Flag tanımları `appsettings.json`'daki `FeatureManagement` bölümünde veya özel bir `IFeatureDefinitionProvider` aracılığıyla sağlanır. Ardından `IFeatureManager` bağımlılık enjeksiyonu ile herhangi bir servise veya controller'a enjekte edilerek `IsEnabledAsync("FlagAdi")` ile sorgulanır.

**Örnek Kod:**

```csharp
// Program.cs
builder.Services.AddFeatureManagement()
    .AddFeatureFilter<PercentageFilter>()
    .AddFeatureFilter<TimeWindowFilter>();

// Servis kullanımı
public class OrderService
{
    private readonly IFeatureManager _featureManager;

    public OrderService(IFeatureManager featureManager)
    {
        _featureManager = featureManager;
    }

    public async Task<Order> CreateOrderAsync(OrderRequest request)
    {
        if (await _featureManager.IsEnabledAsync("ExpressProcessing"))
        {
            return await ProcessExpressAsync(request);
        }
        return await ProcessStandardAsync(request);
    }

    private Task<Order> ProcessExpressAsync(OrderRequest r) => Task.FromResult(new Order());
    private Task<Order> ProcessStandardAsync(OrderRequest r) => Task.FromResult(new Order());
}
```

### 2. Soru

**IFeatureFilter arayüzünü implement ederek özel bir filtre nasıl yazılır?**

**Cevap:** Özel filtre yazmak için `IFeatureFilter` arayüzü implement edilir ve `EvaluateAsync` metodu override edilir. Sınıf, filtrenin konfigürasyonda hangi isimle tanımlanacağını belirten `[FilterAlias]` attribute'u ile işaretlenir. `FeatureFilterEvaluationContext.Parameters` üzerinden `Get<T>()` metodu ile JSON konfigürasyonundan güçlü tipleme ile parametreler okunur. Filtre mantığı burada çalışır ve `true`/`false` döndürülür. Son olarak filtre, `Program.cs`'de `.AddFeatureFilter<MyFilter>()` ile kaydedilir.

**Örnek Kod:**

```csharp
[FilterAlias("BusinessHours")]
public class BusinessHoursFilter : IFeatureFilter
{
    public class Settings
    {
        public TimeOnly OpenTime { get; set; } = new TimeOnly(9, 0);
        public TimeOnly CloseTime { get; set; } = new TimeOnly(18, 0);
        public DayOfWeek[] WorkDays { get; set; } =
            [DayOfWeek.Monday, DayOfWeek.Tuesday, DayOfWeek.Wednesday,
             DayOfWeek.Thursday, DayOfWeek.Friday];
    }

    public Task<bool> EvaluateAsync(FeatureFilterEvaluationContext context)
    {
        var settings = context.Parameters.Get<Settings>() ?? new Settings();
        var now = DateTimeOffset.Now;
        var currentTime = TimeOnly.FromDateTime(now.DateTime);

        var isWorkDay = settings.WorkDays.Contains(now.DayOfWeek);
        var isBusinessHour = currentTime >= settings.OpenTime
                          && currentTime <= settings.CloseTime;

        return Task.FromResult(isWorkDay && isBusinessHour);
    }
}
```

### 3. Soru

**FeatureGate attribute'u ile IFeatureManager.IsEnabledAsync() kullanımı arasındaki tercih kriterleri nelerdir?**

**Cevap:** `[FeatureGate]` attribute'u controller action'larını declarative (bildirimsel) bir şekilde korumak için idealdir; boilerplate kod miktarını azaltır ve flag kapalıysa otomatik olarak 404 döndürür. Ancak yalnızca MVC/Controller katmanında çalışır. `IFeatureManager.IsEnabledAsync()` ise servis katmanı, middleware, background job veya herhangi bir C# kodu içinde kullanılabilen imperative (zorunlu) yaklaşımdır. Kısmi özellik değişikliği, farklı davranışlar veya flag durumuna göre loglama gibi ince kontrol gerektiren durumlarda `IFeatureManager` tercih edilmelidir. Genel kural: UI/API erişim kısıtlaması için attribute, iş mantığı dallı yürütme için doğrudan sorgulama kullanılır.

**Örnek Kod:**

```csharp
// Attribute yaklaşımı - tüm endpoint'i kapat
[FeatureGate("NewReportingModule")]
[HttpGet("reports/advanced")]
public IActionResult GetAdvancedReports() => Ok();

// Imperative yaklaşım - kısmi davranış değişikliği
[HttpGet("reports")]
public async Task<IActionResult> GetReports()
{
    var report = await _reportService.GetBasicReportAsync();

    if (await _featureManager.IsEnabledAsync("NewReportingModule"))
    {
        report.AdvancedMetrics = await _reportService.GetAdvancedMetricsAsync();
        report.Visualizations = await _reportService.GetVisualizationsAsync();
    }

    return Ok(report);
}
```

### 4. Soru

**TargetingFilter ile belirli kullanıcı gruplarına özellik nasıl açılır?**

**Cevap:** TargetingFilter; bireysel kullanıcılar, kullanıcı grupları ve her grup için ayrı yüzde değerleri tanımlanmasına olanak tanır. Bunun çalışması için `ITargetingContextAccessor` arayüzü implement edilmeli ve mevcut kullanıcının kimliği ile grup üyelikleri bu context'e aktarılmalıdır. Konfigürasyondaki `Users` listesi tam eşleşme, `Groups` bölümü grup bazlı kısmi yayılım ve `DefaultRolloutPercentage` tüm diğer kullanıcılara uygulanacak yüzdeyi belirtir.

**Örnek Kod:**

```csharp
public class ClaimsPrincipalTargetingContextAccessor : ITargetingContextAccessor
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public ClaimsPrincipalTargetingContextAccessor(IHttpContextAccessor accessor)
        => _httpContextAccessor = accessor;

    public ValueTask<TargetingContext> GetContextAsync()
    {
        var user = _httpContextAccessor.HttpContext?.User;

        return new ValueTask<TargetingContext>(new TargetingContext
        {
            UserId = user?.FindFirstValue(ClaimTypes.Email) ?? "anonymous",
            Groups = user?.Claims
                .Where(c => c.Type == ClaimTypes.Role)
                .Select(c => c.Value)
                .ToList() ?? new List<string>()
        });
    }
}

// appsettings.json içeriği:
// "Audience": {
//   "Users": ["vip@example.com"],
//   "Groups": [{ "Name": "Employees", "RolloutPercentage": 100 }],
//   "DefaultRolloutPercentage": 5
// }
```

### 5. Soru

**Feature flag'lerin birim testlerde nasıl mock'landığı açıklayınız.**

**Cevap:** Birim testlerde `IFeatureManager` arayüzü `Moq` veya `NSubstitute` gibi kütüphanelerle kolayca mock'lanabilir. `IsEnabledAsync` metodu istenen flag adı için belirli bir değer döndürecek şekilde kurulur. Bu yaklaşım, flag açık ve kapalı durumlarının her ikisi için de test yazılmasına olanak tanır. Ayrıca `Microsoft.FeatureManagement` kütüphanesi bir `ConfigurationFeatureManager` yardımcı sınıfı sunar ve test koleksiyonuyla kurulabilir.

**Örnek Kod:**

```csharp
public class OrderServiceTests
{
    [Fact]
    public async Task CreateOrder_WhenExpressProcessingEnabled_UsesExpressPath()
    {
        // Arrange
        var mockFeatureManager = new Mock<IFeatureManager>();
        mockFeatureManager
            .Setup(fm => fm.IsEnabledAsync("ExpressProcessing"))
            .ReturnsAsync(true);

        var service = new OrderService(mockFeatureManager.Object);

        // Act
        var result = await service.CreateOrderAsync(new OrderRequest());

        // Assert
        Assert.Equal(ProcessingType.Express, result.ProcessingType);
    }

    [Fact]
    public async Task CreateOrder_WhenExpressProcessingDisabled_UsesStandardPath()
    {
        // Arrange
        var mockFeatureManager = new Mock<IFeatureManager>();
        mockFeatureManager
            .Setup(fm => fm.IsEnabledAsync("ExpressProcessing"))
            .ReturnsAsync(false);

        var service = new OrderService(mockFeatureManager.Object);

        // Act
        var result = await service.CreateOrderAsync(new OrderRequest());

        // Assert
        Assert.Equal(ProcessingType.Standard, result.ProcessingType);
    }

    [Theory]
    [InlineData(true, "v2")]
    [InlineData(false, "v1")]
    public async Task GetApiVersion_ReturnsCorrectVersion(bool flagEnabled, string expectedVersion)
    {
        var mockFeatureManager = new Mock<IFeatureManager>();
        mockFeatureManager
            .Setup(fm => fm.IsEnabledAsync("ApiV2"))
            .ReturnsAsync(flagEnabled);

        var service = new VersionService(mockFeatureManager.Object);
        var version = await service.GetCurrentVersionAsync();

        Assert.Equal(expectedVersion, version);
    }
}
```

### 6. Soru

**IFeatureDefinitionProvider ile özel bir yapılandırma kaynağı (veritabanı veya uzak API) nasıl kullanılır?**

**Cevap:** `IFeatureDefinitionProvider` arayüzü implement edilerek flag tanımları herhangi bir kaynaktan okunabilir. `GetAllFeatureDefinitionsAsync()` ve `GetFeatureDefinitionAsync()` metotları override edilir. Performans açısından tanımlar önbelleğe alınmalı ve belirli aralıklarla yenilenir. Bu provider, `Program.cs`'de `AddFeatureManagement()` zincirinin sonunda `.UseCustomProvider<T>()` veya doğrudan `services.AddSingleton<IFeatureDefinitionProvider>()` ile kaydedilir.

**Örnek Kod:**

```csharp
public class RemoteApiFeatureDefinitionProvider : IFeatureDefinitionProvider
{
    private readonly HttpClient _httpClient;
    private readonly IMemoryCache _cache;
    private const string CacheKey = "remote_feature_definitions";

    public RemoteApiFeatureDefinitionProvider(HttpClient httpClient, IMemoryCache cache)
    {
        _httpClient = httpClient;
        _cache = cache;
    }

    public async IAsyncEnumerable<FeatureDefinition> GetAllFeatureDefinitionsAsync()
    {
        var definitions = await GetDefinitionsAsync();
        foreach (var def in definitions)
            yield return def;
    }

    public async Task<FeatureDefinition?> GetFeatureDefinitionAsync(string featureName)
    {
        var definitions = await GetDefinitionsAsync();
        return definitions.FirstOrDefault(d => d.Name == featureName);
    }

    private async Task<List<FeatureDefinition>> GetDefinitionsAsync()
    {
        return await _cache.GetOrCreateAsync(CacheKey, async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(2);

            var response = await _httpClient.GetFromJsonAsync<List<RemoteFlag>>(
                "/api/feature-flags");

            return response?.Select(r => new FeatureDefinition
            {
                Name = r.Name,
                EnabledFor = r.IsEnabled
                    ? new List<FeatureFilterConfiguration>()
                    : new List<FeatureFilterConfiguration>
                    {
                        new() { Name = "AlwaysOff" }
                    }
            }).ToList() ?? new List<FeatureDefinition>();
        }) ?? new List<FeatureDefinition>();
    }

    private class RemoteFlag
    {
        public string Name { get; set; } = string.Empty;
        public bool IsEnabled { get; set; }
    }
}

// Program.cs
builder.Services.AddSingleton<IFeatureDefinitionProvider, RemoteApiFeatureDefinitionProvider>();
builder.Services.AddFeatureManagement();
```

### 7. Soru

**Feature flag yönetiminde dikkat edilmesi gereken güvenlik ve erişim kontrol pratikleri nelerdir?**

**Cevap:** Feature flag sistemi, yanlış yapılandırıldığında güvenlik açıklarına yol açabilir. Dikkat edilmesi gereken başlıca konular: Flag konfigürasyonu yazma erişimini yalnızca yetkili kişilerle sınırlayın; rol tabanlı erişim kontrolü (RBAC) uygulayın. Flag değişikliklerini audit log'a kaydedin; kim, ne zaman, ne değiştirildi bilgisi saklanmalıdır. Hassas özellikleri (ödeme, kimlik doğrulama) flag'e bağlarken ek doğrulama katmanları kullanın. Flag değerlerini asla istemci tarafında güvenilir veri olarak ele almayın; sunucu taraflı doğrulama şarttır. Ortam bazlı flag ayrımı yapın; dev flag'leri üretimde kullanmayın.

**Örnek Kod:**

```csharp
// Flag değişikliklerini audit log'a kaydeden decorator
public class AuditedFeatureManager : IFeatureManager
{
    private readonly IFeatureManager _inner;
    private readonly IAuditLogger _auditLogger;
    private readonly IHttpContextAccessor _httpContextAccessor;

    public AuditedFeatureManager(
        IFeatureManager inner,
        IAuditLogger auditLogger,
        IHttpContextAccessor httpContextAccessor)
    {
        _inner = inner;
        _auditLogger = auditLogger;
        _httpContextAccessor = httpContextAccessor;
    }

    public async Task<bool> IsEnabledAsync(string feature)
    {
        var result = await _inner.IsEnabledAsync(feature);
        var userId = _httpContextAccessor.HttpContext?.User?.Identity?.Name;

        await _auditLogger.LogAsync(new AuditEntry
        {
            Action = "FeatureChecked",
            FeatureName = feature,
            Result = result,
            UserId = userId,
            Timestamp = DateTimeOffset.UtcNow
        });

        return result;
    }

    public IAsyncEnumerable<string> GetFeatureNamesAsync()
        => _inner.GetFeatureNamesAsync();

    public Task<bool> IsEnabledAsync<TContext>(string feature, TContext context)
        => _inner.IsEnabledAsync(feature, context);
}
```

## Best Practices

### 1. **Flag Organizasyonu**
- `appsettings.json`'ı ortam bazlı ayırın: `appsettings.Development.json`, `appsettings.Production.json`
- Hassas flag'ler için Azure App Configuration veya AWS Parameter Store kullanın
- Flag adlarını Pascal Case ve açıklayıcı tutun

### 2. **Performans**
- Sık erişilen flag değerlerini önbelleğe alın
- Uzak API'den okunan tanımlar için kısa TTL'li cache uygulayın
- `IAsyncEnumerable` kullanarak flag listelerini akış şeklinde işleyin

### 3. **Temizlik**
- Her flag için bitiş tarihi ve sorumlu kişi tanımlayın
- Özellik tamamlandığında flag'i ve dal yapısını koddan kaldırın
- Eski flag'leri otomatik tespit eden CI kontrolleri ekleyin

## Kaynaklar

- [Microsoft.FeatureManagement GitHub](https://github.com/microsoft/FeatureManagement-Dotnet)
- [Azure App Configuration Feature Flags](https://docs.microsoft.com/en-us/azure/azure-app-configuration/concept-feature-management)
- [FeatureManagement NuGet Paketi](https://www.nuget.org/packages/Microsoft.FeatureManagement.AspNetCore)
