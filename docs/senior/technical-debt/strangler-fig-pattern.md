# Strangler Fig Pattern

## Genel Bakış
Strangler Fig Pattern, Martin Fowler tarafından adlandırılmış bir yazılım migrasyonu stratejisidir. Adını, bir ağacın gövdesine sarılarak onu yavaş yavaş değiştiren strangler fig bitkisinden alır. Bu pattern, mevcut bir sistemi tamamen yıkıp sıfırdan yazmak yerine, işlevselliği kademeli olarak yeni bir sistemle değiştirerek riski minimize eder. Production sistemi kesintisiz çalışmaya devam ederken yeni bileşenler birer birer devreye alınır.

## Mülakat Soruları ve Cevapları

### 1. Strangler Fig Pattern nedir ve neden tercih edilir?
**Cevap:**
Strangler Fig Pattern, eski (legacy) bir sistemi tümüyle değiştirmek yerine kademeli olarak yeni bir sistemle sararak ilerleyen migration stratejisidir. Her iterasyonda eski sistemin bir işlev veya modülü yeni sistemde yeniden implementendir ve trafik yavaşça yeni sisteme yönlendirilir. Eski sistem zamanla körelir (strangles) ve nihayetinde tamamen kaldırılır.

Tercih edilme nedenleri:
- Big Bang migration'ın "hepsini bir anda değiştir" riskinden kaçınılır
- Her adımda test edilebilir ve geri alınabilir değişim yapılır
- Eski sistem fallback olarak çalışmaya devam eder
- Ekip yeni teknolojileri küçük adımlarla öğrenir
- İş değeri taşıyan özellikler önce taşınabilir

**Örnek Kod:**
```csharp
// Strangler Facade - eski ve yeni sistemi saran proxy katmanı
public class OrderFacade
{
    private readonly LegacyOrderService _legacyService;
    private readonly INewOrderService _newService;
    private readonly IFeatureToggleService _featureToggle;
    private readonly ILogger<OrderFacade> _logger;

    public OrderFacade(
        LegacyOrderService legacyService,
        INewOrderService newService,
        IFeatureToggleService featureToggle,
        ILogger<OrderFacade> logger)
    {
        _legacyService = legacyService;
        _newService = newService;
        _featureToggle = featureToggle;
        _logger = logger;
    }

    public async Task<OrderResult> CreateOrderAsync(CreateOrderRequest request)
    {
        // Feature toggle ile hangi sistemin kullanılacağına karar verilir
        if (await _featureToggle.IsEnabledAsync("UseNewOrderService", request.CustomerId))
        {
            _logger.LogInformation(
                "Routing order creation to new service for customer {CustomerId}",
                request.CustomerId);
            return await _newService.CreateOrderAsync(request);
        }

        _logger.LogInformation(
            "Routing order creation to legacy service for customer {CustomerId}",
            request.CustomerId);
        return await _legacyService.CreateOrderAsync(request);
    }

    public async Task<Order> GetOrderAsync(string orderId)
    {
        if (await _featureToggle.IsEnabledAsync("UseNewOrderRead"))
        {
            return await _newService.GetOrderAsync(orderId);
        }
        return await _legacyService.GetOrderAsync(orderId);
    }
}
```

### 2. Strangler Fig implementasyonunda Facade katmanı nasıl tasarlanır?
**Cevap:**
Facade katmanı, istemcilerin eski ve yeni sistemler arasındaki farkı görmeden çalışmasını sağlayan bir soyutlama katmanıdır. Şu sorumlulukları taşır: istek yönlendirme kararını vermek, eski ve yeni sistem arasındaki model dönüşümünü yapmak, hata durumunda fallback sağlamak ve yönlendirme kararlarını loglayarak görünürlük sunmak.

**Örnek Kod:**
```csharp
// Yönlendirme stratejisi arayüzü
public interface IRoutingStrategy
{
    Task<bool> ShouldUseNewSystemAsync(RoutingContext context);
}

// Yüzde bazlı yönlendirme stratejisi
public class PercentageBasedRoutingStrategy : IRoutingStrategy
{
    private readonly IConfiguration _configuration;

    public PercentageBasedRoutingStrategy(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public Task<bool> ShouldUseNewSystemAsync(RoutingContext context)
    {
        var percentage = _configuration.GetValue<int>("Migration:NewSystemPercentage", 0);
        var hash = Math.Abs(context.RequestId.GetHashCode()) % 100;
        return Task.FromResult(hash < percentage);
    }
}

// Müşteri segmentine göre yönlendirme stratejisi
public class CustomerSegmentRoutingStrategy : IRoutingStrategy
{
    private readonly ICustomerRepository _customerRepository;

    public CustomerSegmentRoutingStrategy(ICustomerRepository customerRepository)
    {
        _customerRepository = customerRepository;
    }

    public async Task<bool> ShouldUseNewSystemAsync(RoutingContext context)
    {
        if (string.IsNullOrEmpty(context.CustomerId))
            return false;

        var customer = await _customerRepository.GetAsync(context.CustomerId);
        // Pilot müşteriler veya belirli bir segmentteki müşteriler yeni sisteme yönlendirilir
        return customer?.IsPilotCustomer == true || customer?.Segment == CustomerSegment.Premium;
    }
}

// Composite strateji - birden fazla stratejiyi birleştirir
public class CompositeRoutingStrategy : IRoutingStrategy
{
    private readonly IEnumerable<IRoutingStrategy> _strategies;

    public CompositeRoutingStrategy(IEnumerable<IRoutingStrategy> strategies)
    {
        _strategies = strategies;
    }

    public async Task<bool> ShouldUseNewSystemAsync(RoutingContext context)
    {
        // Herhangi bir strateji true dönerse yeni sisteme yönlendir
        foreach (var strategy in _strategies)
        {
            if (await strategy.ShouldUseNewSystemAsync(context))
                return true;
        }
        return false;
    }
}

public record RoutingContext(string RequestId, string? CustomerId = null, string? Feature = null);
```

### 3. Reverse proxy ile Strangler Fig nasıl uygulanır?
**Cevap:**
Reverse proxy yaklaşımında, istemcilerden gelen HTTP istekleri bir proxy katmanında incelenerek URL yolu, başlık veya başka kriterlere göre eski veya yeni servise iletilir. Bu yaklaşım kod değişikliği gerektirmez; altyapı seviyesinde çalışır. .NET'te YARP (Yet Another Reverse Proxy) kütüphanesi bu amaçla kullanılabilir.

**Örnek Kod:**
```csharp
// Program.cs - YARP ile Strangler Fig Reverse Proxy
using Yarp.ReverseProxy;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

builder.Services.AddSingleton<IMigrationStateService, MigrationStateService>();

var app = builder.Build();

// Özel middleware: migration durumuna göre rota seçimi
app.Use(async (context, next) =>
{
    var migrationService = context.RequestServices.GetRequiredService<IMigrationStateService>();
    var path = context.Request.Path.Value ?? string.Empty;

    // /api/orders/* yolları migration durumuna göre yönlendirilir
    if (path.StartsWith("/api/orders", StringComparison.OrdinalIgnoreCase))
    {
        var feature = path.Split('/').ElementAtOrDefault(3) ?? "list";
        var useNewSystem = await migrationService.IsFeatureMigratedAsync(feature);

        // Yönlendirme kümesini dinamik olarak ayarla
        context.Items["cluster"] = useNewSystem ? "new-order-service" : "legacy-order-service";
    }

    await next();
});

app.MapReverseProxy();
app.Run();

// appsettings.json içindeki YARP konfigürasyonu:
// {
//   "ReverseProxy": {
//     "Routes": {
//       "legacy-route": {
//         "ClusterId": "legacy-order-service",
//         "Match": { "Path": "/api/orders/{**catch-all}" }
//       }
//     },
//     "Clusters": {
//       "legacy-order-service": {
//         "Destinations": {
//           "destination1": { "Address": "http://legacy-app:8080/" }
//         }
//       },
//       "new-order-service": {
//         "Destinations": {
//           "destination1": { "Address": "http://new-app:8080/" }
//         }
//       }
//     }
//   }
// }

// Migration durum servisi
public interface IMigrationStateService
{
    Task<bool> IsFeatureMigratedAsync(string featureName);
    Task MarkFeatureAsMigratedAsync(string featureName);
    Task<IReadOnlyDictionary<string, bool>> GetAllFeaturesAsync();
}

public class MigrationStateService : IMigrationStateService
{
    private readonly IDistributedCache _cache;
    private readonly IConfiguration _configuration;

    public MigrationStateService(IDistributedCache cache, IConfiguration configuration)
    {
        _cache = cache;
        _configuration = configuration;
    }

    public async Task<bool> IsFeatureMigratedAsync(string featureName)
    {
        var cacheKey = $"migration:feature:{featureName}";
        var cached = await _cache.GetStringAsync(cacheKey);

        if (cached is not null)
            return bool.Parse(cached);

        // Konfigürasyondan varsayılan değeri oku
        var isMigrated = _configuration.GetValue<bool>($"Migration:Features:{featureName}", false);
        await _cache.SetStringAsync(cacheKey, isMigrated.ToString(),
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5) });

        return isMigrated;
    }

    public async Task MarkFeatureAsMigratedAsync(string featureName)
    {
        var cacheKey = $"migration:feature:{featureName}";
        await _cache.SetStringAsync(cacheKey, "true",
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5) });
    }

    public Task<IReadOnlyDictionary<string, bool>> GetAllFeaturesAsync()
    {
        // Gerçek implementasyonda veritabanından okunur
        var features = new Dictionary<string, bool>
        {
            ["create"] = true,
            ["read"] = true,
            ["update"] = false,
            ["delete"] = false
        };
        return Task.FromResult<IReadOnlyDictionary<string, bool>>(features);
    }
}
```

### 4. Strangler Fig pattern ile API Gateway entegrasyonu nasıl yapılır?
**Cevap:**
API Gateway, mikro servis mimarisinin giriş noktasıdır ve Strangler Fig implementasyonunda trafik yönlendirme kararlarının merkezi konumudur. Yeni bir endpoint implemente edildiğinde gateway konfigürasyonu güncellenerek ilgili istekler yeni servise yönlendirilir; eski endpoint silinmez, yalnızca trafiği almayı bırakır.

**Örnek Kod:**
```csharp
// Özel API Gateway middleware - Strangler Fig yönlendirme mantığı
public class StranglerFigGatewayMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IRoutingRegistry _routingRegistry;
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly ILogger<StranglerFigGatewayMiddleware> _logger;

    public StranglerFigGatewayMiddleware(
        RequestDelegate next,
        IRoutingRegistry routingRegistry,
        IHttpClientFactory httpClientFactory,
        ILogger<StranglerFigGatewayMiddleware> logger)
    {
        _next = next;
        _routingRegistry = routingRegistry;
        _httpClientFactory = httpClientFactory;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var routeKey = BuildRouteKey(context.Request);
        var targetService = await _routingRegistry.ResolveAsync(routeKey);

        if (targetService is null)
        {
            await _next(context);
            return;
        }

        _logger.LogInformation(
            "Routing {Method} {Path} to {Service}",
            context.Request.Method,
            context.Request.Path,
            targetService.Name);

        await ProxyRequestAsync(context, targetService);
    }

    private async Task ProxyRequestAsync(HttpContext context, ServiceEndpoint target)
    {
        var client = _httpClientFactory.CreateClient(target.Name);
        var targetUri = BuildTargetUri(context.Request, target.BaseAddress);

        using var requestMessage = CreateProxyRequest(context.Request, targetUri);

        try
        {
            using var response = await client.SendAsync(
                requestMessage,
                HttpCompletionOption.ResponseHeadersRead,
                context.RequestAborted);

            await CopyResponseAsync(context.Response, response);
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "Error proxying request to {Service}", target.Name);

            // Yeni servisten hata gelirse eski servise fallback
            if (target.EnableFallback)
            {
                _logger.LogWarning("Falling back to legacy service for {Path}", context.Request.Path);
                await FallbackToLegacyAsync(context, target.FallbackAddress!);
            }
            else
            {
                context.Response.StatusCode = StatusCodes.Status502BadGateway;
            }
        }
    }

    private static string BuildRouteKey(HttpRequest request)
        => $"{request.Method}:{request.Path.Value?.TrimEnd('/')}";

    private static Uri BuildTargetUri(HttpRequest request, string baseAddress)
    {
        var uriBuilder = new UriBuilder(baseAddress)
        {
            Path = request.Path,
            Query = request.QueryString.Value ?? string.Empty
        };
        return uriBuilder.Uri;
    }

    private static HttpRequestMessage CreateProxyRequest(HttpRequest request, Uri targetUri)
    {
        var message = new HttpRequestMessage(new HttpMethod(request.Method), targetUri);

        foreach (var header in request.Headers)
        {
            if (!header.Key.Equals("Host", StringComparison.OrdinalIgnoreCase))
                message.Headers.TryAddWithoutValidation(header.Key, header.Value.ToArray());
        }

        if (request.ContentLength > 0)
            message.Content = new StreamContent(request.Body);

        return message;
    }

    private static async Task CopyResponseAsync(HttpResponse response, HttpResponseMessage proxyResponse)
    {
        response.StatusCode = (int)proxyResponse.StatusCode;

        foreach (var header in proxyResponse.Headers)
            response.Headers[header.Key] = header.Value.ToArray();

        await proxyResponse.Content.CopyToAsync(response.Body);
    }

    private Task FallbackToLegacyAsync(HttpContext context, string fallbackAddress)
    {
        // Eski sisteme yeniden yönlendirme mantığı
        context.Response.Redirect(fallbackAddress + context.Request.Path, permanent: false);
        return Task.CompletedTask;
    }
}

// Yönlendirme kayıt defteri
public interface IRoutingRegistry
{
    Task<ServiceEndpoint?> ResolveAsync(string routeKey);
    Task RegisterAsync(string routePattern, ServiceEndpoint endpoint);
    Task UnregisterAsync(string routePattern);
}

public record ServiceEndpoint(
    string Name,
    string BaseAddress,
    bool EnableFallback = false,
    string? FallbackAddress = null);
```

### 5. Strangler Fig'de eski ve yeni sistemler arasında veri tutarlılığı nasıl sağlanır?
**Cevap:**
İki sistemin paralel çalıştığı dönemde veri tutarlılığı en kritik zorluktur. Çift yazma (dual write), event-driven senkronizasyon ve change data capture (CDC) yaklaşımları kullanılır. Geçiş tamamlandıktan sonra tek veri kaynağına (single source of truth) geçilir.

**Örnek Kod:**
```csharp
// Dual Write servisi - her iki sisteme eş zamanlı yazma
public class DualWriteOrderRepository : IOrderRepository
{
    private readonly IOrderRepository _legacyRepository;
    private readonly IOrderRepository _newRepository;
    private readonly ILogger<DualWriteOrderRepository> _logger;
    private readonly IMigrationStateService _migrationState;

    public DualWriteOrderRepository(
        IOrderRepository legacyRepository,
        IOrderRepository newRepository,
        ILogger<DualWriteOrderRepository> logger,
        IMigrationStateService migrationState)
    {
        _legacyRepository = legacyRepository;
        _newRepository = newRepository;
        _logger = logger;
        _migrationState = migrationState;
    }

    public async Task<Order> SaveAsync(Order order)
    {
        var isDualWriteEnabled = await _migrationState.IsFeatureMigratedAsync("dual-write-orders");

        if (!isDualWriteEnabled)
        {
            // Yalnızca legacy sisteme yaz
            return await _legacyRepository.SaveAsync(order);
        }

        // Her iki sisteme de yaz; legacy birincil, yeni sistem ikincil
        var legacyResult = await _legacyRepository.SaveAsync(order);

        try
        {
            // Yeni sisteme yazma başarısız olsa dahi legacy işlemi tamamdır
            var newOrder = MapToNewOrder(order);
            await _newRepository.SaveAsync(newOrder);
            _logger.LogDebug("Dual write successful for order {OrderId}", order.Id);
        }
        catch (Exception ex)
        {
            // Yeni sisteme yazma hatası loglama ile geçilir; sonradan reconcile edilir
            _logger.LogError(ex,
                "Dual write to new system failed for order {OrderId}. Will reconcile later.",
                order.Id);
            await EnqueueForReconciliationAsync(order.Id);
        }

        return legacyResult;
    }

    public async Task<Order?> GetAsync(string orderId)
    {
        var useNewForRead = await _migrationState.IsFeatureMigratedAsync("new-system-reads");

        if (useNewForRead)
        {
            var newOrder = await _newRepository.GetAsync(orderId);
            if (newOrder is not null) return newOrder;

            // Yeni sistemde bulunamazsa legacy'e fallback
            _logger.LogWarning("Order {OrderId} not found in new system, falling back to legacy", orderId);
        }

        return await _legacyRepository.GetAsync(orderId);
    }

    private static Order MapToNewOrder(Order order)
    {
        // Domain model dönüşümü burada yapılır
        return order with { Source = "migrated" };
    }

    private async Task EnqueueForReconciliationAsync(string orderId)
    {
        // Mesaj kuyruğuna reconciliation görevi eklenir
        await Task.CompletedTask; // Gerçek implementasyonda message bus kullanılır
    }
}

// Reconciliation servisi - tutarsızlıkları gidermek için arka plan işi
public class DataReconciliationService : BackgroundService
{
    private readonly IOrderRepository _legacyRepository;
    private readonly IOrderRepository _newRepository;
    private readonly ILogger<DataReconciliationService> _logger;

    public DataReconciliationService(
        IOrderRepository legacyRepository,
        IOrderRepository newRepository,
        ILogger<DataReconciliationService> logger)
    {
        _legacyRepository = legacyRepository;
        _newRepository = newRepository;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await ReconcileAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromMinutes(15), stoppingToken);
        }
    }

    private async Task ReconcileAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Starting reconciliation run");

        var legacyOrders = await _legacyRepository.GetRecentOrdersAsync(TimeSpan.FromHours(1));

        foreach (var legacyOrder in legacyOrders)
        {
            if (cancellationToken.IsCancellationRequested) break;

            var newOrder = await _newRepository.GetAsync(legacyOrder.Id);
            if (newOrder is null)
            {
                _logger.LogWarning("Order {OrderId} missing in new system, syncing", legacyOrder.Id);
                await _newRepository.SaveAsync(legacyOrder);
            }
        }

        _logger.LogInformation("Reconciliation run completed");
    }
}
```

### 6. Strangler Fig geri alma (rollback) stratejisi nasıl uygulanır?
**Cevap:**
Her migration adımı için geri alma planı tanımlanmalıdır. Feature flag'ler anlık rollback imkânı sağlar; veritabanı değişiklikleri için expand-contract pattern kullanılır; yeni servis eski servisle aynı API sözleşmesini korumalıdır. Canary deployment ile küçük trafik yüzdeleriyle test edilir; sorun çıkarsa yüzde sıfıra çekilir.

**Örnek Kod:**
```csharp
// Feature flag ile anlık rollback kontrolü
public class FeatureFlagService : IFeatureToggleService
{
    private readonly IDistributedCache _cache;
    private readonly IConfiguration _configuration;
    private readonly ILogger<FeatureFlagService> _logger;

    public FeatureFlagService(
        IDistributedCache cache,
        IConfiguration configuration,
        ILogger<FeatureFlagService> logger)
    {
        _cache = cache;
        _configuration = configuration;
        _logger = logger;
    }

    public async Task<bool> IsEnabledAsync(string featureName, string? contextId = null)
    {
        var cacheKey = $"feature:{featureName}:{contextId ?? "global"}";
        var cached = await _cache.GetStringAsync(cacheKey);

        if (cached is not null)
            return bool.Parse(cached);

        var enabled = EvaluateFeature(featureName, contextId);
        await _cache.SetStringAsync(cacheKey, enabled.ToString(),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromSeconds(30) // Kısa TTL: hızlı rollback
            });

        return enabled;
    }

    private bool EvaluateFeature(string featureName, string? contextId)
    {
        var section = _configuration.GetSection($"FeatureFlags:{featureName}");
        if (!section.Exists()) return false;

        var enabled = section.GetValue<bool>("Enabled");
        if (!enabled) return false;

        // Yüzde bazlı rollout
        var rolloutPercentage = section.GetValue<int>("RolloutPercentage", 100);
        if (rolloutPercentage < 100 && contextId is not null)
        {
            var hash = Math.Abs(contextId.GetHashCode()) % 100;
            return hash < rolloutPercentage;
        }

        return enabled;
    }

    // Anlık rollback: tüm flag'i devre dışı bırak
    public async Task DisableFeatureAsync(string featureName)
    {
        _logger.LogWarning("Disabling feature flag {FeatureName} - ROLLBACK initiated", featureName);

        var pattern = $"feature:{featureName}:*";
        // Gerçek implementasyonda tüm cache key'leri temizlenir
        await _cache.RemoveAsync($"feature:{featureName}:global");
    }
}

// appsettings.json örnek feature flag konfigürasyonu:
// {
//   "FeatureFlags": {
//     "UseNewOrderService": {
//       "Enabled": true,
//       "RolloutPercentage": 25
//     },
//     "UseNewPaymentService": {
//       "Enabled": false,
//       "RolloutPercentage": 0
//     }
//   }
// }
```

### 7. Strangler Fig'de shadow mode (gölge mod) testi nedir?
**Cevap:**
Shadow mode, bir isteği hem eski hem de yeni sisteme aynı anda göndererek sonuçları karşılaştırma tekniğidir. Gerçek trafik yalnızca eski sistemden döner; yeni sistemin yanıtı karşılaştırma ve loglama için kullanılır. Bu sayede yeni sistem production yüküyle test edilirken kullanıcı etkilenmez.

**Örnek Kod:**
```csharp
// Shadow mode servis decorator'ı
public class ShadowModeOrderService : INewOrderService
{
    private readonly INewOrderService _newService;
    private readonly LegacyOrderService _legacyService;
    private readonly ILogger<ShadowModeOrderService> _logger;
    private readonly IFeatureToggleService _featureToggle;

    public ShadowModeOrderService(
        INewOrderService newService,
        LegacyOrderService legacyService,
        ILogger<ShadowModeOrderService> logger,
        IFeatureToggleService featureToggle)
    {
        _newService = newService;
        _legacyService = legacyService;
        _logger = logger;
        _featureToggle = featureToggle;
    }

    public async Task<OrderResult> CreateOrderAsync(CreateOrderRequest request)
    {
        var shadowEnabled = await _featureToggle.IsEnabledAsync("ShadowModeOrderCreate");

        if (!shadowEnabled)
            return await _legacyService.CreateOrderAsync(request);

        // Legacy servis gerçek sonucu döner
        var legacyTask = _legacyService.CreateOrderAsync(request);

        // Yeni servis arka planda çalışır (fire-and-forget değil, ama sonucu kullanılmaz)
        var newServiceTask = RunShadowAsync(request);

        var legacyResult = await legacyTask;

        // Shadow görevi sonucunu logla ama beklemeden devam et
        _ = newServiceTask.ContinueWith(t =>
        {
            if (t.IsCompletedSuccessfully)
                CompareShadowResults(legacyResult, t.Result, request.OrderNumber);
            else
                _logger.LogError(t.Exception, "Shadow mode error for order {OrderNumber}", request.OrderNumber);
        }, TaskScheduler.Default);

        return legacyResult;
    }

    private async Task<OrderResult> RunShadowAsync(CreateOrderRequest request)
    {
        try
        {
            return await _newService.CreateOrderAsync(request);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Shadow service failed for order {OrderNumber}", request.OrderNumber);
            throw;
        }
    }

    private void CompareShadowResults(OrderResult legacyResult, OrderResult newResult, string orderNumber)
    {
        if (legacyResult.OrderId != newResult.OrderId ||
            legacyResult.TotalAmount != newResult.TotalAmount)
        {
            _logger.LogWarning(
                "Shadow mode DIVERGENCE for order {OrderNumber}. " +
                "Legacy: Id={LegacyId} Amount={LegacyAmount}. " +
                "New: Id={NewId} Amount={NewAmount}",
                orderNumber,
                legacyResult.OrderId, legacyResult.TotalAmount,
                newResult.OrderId, newResult.TotalAmount);
        }
        else
        {
            _logger.LogDebug(
                "Shadow mode MATCH for order {OrderNumber}",
                orderNumber);
        }
    }
}
```

## Best Practices

### 1. **Küçük ve Bağımsız Adımlar**
- Her migration adımı bağımsız olarak test edilebilir olmalıdır
- Bir özellik tamamen taşınmadan diğerine geçilmemelidir
- Her adım için "done" kriterleri önceden tanımlanmalıdır
- Adım başarısız olursa otomatik rollback mekanizması devreye girmelidir
- Progress dashboard ile ilerleme görünür kılınmalıdır

### 2. **Gözlemlenebilirlik**
- Her yönlendirme kararı loglanmalıdır
- Eski ve yeni sistem hata oranları karşılaştırmalı izlenmelidir
- Latency farkları alert olarak tanımlanmalıdır
- Shadow mode divergence metrikleri takip edilmelidir
- Distributed tracing ile her isteğin hangi sisteme gittiği izlenmelidir

### 3. **API Sözleşmesi Uyumu**
- Yeni sistem, eski sistemin API sözleşmesini birebir karşılamalıdır
- Contract testing (Pact gibi araçlarla) otomatize edilmelidir
- Backward compatibility kırılmadan versiyonlama yapılmalıdır
- Sözleşme testleri CI/CD pipeline'ına entegre edilmelidir
- Consumer-driven contract testing uygulanmalıdır

### 4. **Veri Tutarlılığı**
- Dual write sırasında idempotent operasyonlar tasarlanmalıdır
- Reconciliation süreci düzenli çalıştırılmalı ve alarm kurulmalıdır
- Veri tutarsızlıkları production'da alarm üretecek şekilde izlenmelidir
- Change Data Capture (CDC) araçları değerlendirilmelidir
- Migration tamamlandıktan sonra cleanup görevi çalıştırılmalıdır

## Kaynaklar

- [Martin Fowler - Strangler Fig Application](https://martinfowler.com/bliki/StranglerFigApplication.html)
- [Microsoft YARP (Yet Another Reverse Proxy)](https://microsoft.github.io/reverse-proxy/)
- [Feature Flags Best Practices](https://martinfowler.com/articles/feature-toggles.html)
- [Microsoft Migration Patterns](https://docs.microsoft.com/en-us/azure/architecture/patterns/strangler-fig)
- [Dual Write Pattern](https://debezium.io/blog/2019/02/19/reliable-microservices-data-exchange-with-the-outbox-pattern/)
- [Shadow Mode Testing](https://shopify.engineering/shadow-mode-testing)
