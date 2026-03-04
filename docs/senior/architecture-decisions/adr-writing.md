# ADR Yazımı

## Genel Bakış

ADR (Architecture Decision Record) yazımı, mimari kararları yapılandırılmış ve sürdürülebilir bir biçimde belgeleme sanatıdır. İyi yazılmış bir ADR; kararın arka planını, değerlendirilen alternatifleri, seçilen yaklaşımı ve bu seçimin sistemin geleceği üzerindeki olumlu/olumsuz etkilerini açıkça ortaya koyar. Bu sayfa, ADR şablonlarını, numaralandırma yaklaşımlarını, araçları ve C# uygulamalarına etkilerini kapsamlı biçimde ele alır.

## Mülakat Soruları ve Cevapları

### 1. Soru: ADR şablonunun temel bileşenleri nelerdir ve her bileşen ne anlama gelir?

**Cevap:**
Bir ADR tipik olarak beş ana bölümden oluşur:

- **Başlık (Title)**: Kararı kısaca ve anlaşılır biçimde özetleyen bir cümle. Genellikle "ADR-XXXX: [Karar Adı]" formatında numaralandırılır.
- **Durum (Status)**: Kararın yaşam döngüsünü gösterir. `Proposed`, `Accepted`, `Deprecated`, `Superseded by ADR-XXXX` değerlerini alabilir.
- **Bağlam (Context)**: Kararın alınmasını zorunlu kılan teknik veya iş gereksinimlerini açıklar. "Neden bu konuda karar almak zorundayız?" sorusuna yanıt verir.
- **Karar (Decision)**: Alınan kararı ve kısa gerekçesini açıklar. "Şunu yapıyoruz çünkü..." formatında yazılır.
- **Sonuçlar (Consequences)**: Kararın olumlu ve olumsuz etkilerini, kabul edilen ödünleri ve gelecekte ortaya çıkabilecek sorumlulukları listeler.

Bazı genişletilmiş şablonlar (MADR gibi) ek olarak şunları içerir:
- **Değerlendirilen Alternatifler (Considered Options)**: Neden bu karar seçildi, diğerleri neden reddedildi?
- **Karar Kriterleri (Decision Drivers)**: Kararı yönlendiren temel gereksinimler.
- **Katılımcılar (Participants)**: Kararı alan veya onaylayan kişiler.

**Örnek Kod:**
```markdown
# ADR-0003: Servisler Arası İletişimde gRPC Kullanımı

## Durum

Accepted

## Bağlam

Sistemimiz, birden fazla .NET mikroservisi içermektedir. Servisler arası iletişimde
şu anda REST/JSON kullanılmaktadır. Yük testleri, yüksek frekanslı dahili servis
çağrılarında JSON serileştirme maliyetinin önemli gecikmelere neden olduğunu
göstermiştir. Ortalama dahili çağrı süresi 18ms olup bunun %40'ı serileştirme
giderinden kaynaklanmaktadır.

Ek kısıtlamalar:
- Tüm servisler .NET 8 kullanmaktadır
- İstemci tarafında JavaScript/TypeScript servisleri de mevcuttur (bunlar REST'i koruyacak)
- Ekip gRPC konusunda sınırlı deneyime sahiptir

## Karar

Dahili .NET-to-.NET servis iletişiminde REST/JSON yerine gRPC kullanacağız.
Dış istemcilere açık API'ler ve JavaScript/TypeScript servisleri REST/JSON ile
iletişimini sürdürecektir.

Gerekçe: Protobuf, JSON'a kıyasla %60-80 daha küçük payload ve daha hızlı
serileştirme sağlamaktadır. HTTP/2 çoklama desteği bağlantı sayısını azaltmaktadır.
Güçlü tip güvenliği sözleşme ihlallerini derleme zamanında yakalar.

## Sonuçlar

Olumlu:
- Dahili servis gecikmeleri ~40% azalacak (benchmark sonuçlarına dayanarak)
- Güçlü tip güvenliği ve derleme zamanı sözleşme doğrulaması
- HTTP/2 akış desteği gerçek zamanlı senaryolara olanak tanıyacak

Olumsuz / Kabul edilen ödünler:
- Ekip gRPC öğrenmek zorunda kalacak (tahmini 2 sprint onboarding süresi)
- .proto dosyalarının bakımı ek operasyonel yük getirecek
- gRPC, REST'e kıyasla tarayıcıdan doğrudan çağrılabilirliği sınırlamaktadır
- Hata ayıklama araçları (Postman vb.) ek yapılandırma gerektirecek

İzleme kararları:
- İlk 3 ay sonra gecikme metriklerini gözden geçir
- Gerekirse gRPC-Web ile tarayıcı desteğini değerlendir
```

### 2. Soru: ADR numaralandırma sistemi nasıl tasarlanır? Monorepo ile çoklu repo senaryolarındaki farklar nelerdir?

**Cevap:**
ADR numaralandırmasında temel prensip, numaraların **değişmez (immutable)** olmasıdır. Bir ADR iptal edilse bile numarası başka bir ADR'ye verilmez; bunun yerine yeni bir ADR yazılarak eski ADR "Superseded" durumuna alınır.

**Tek repo / monolith için:**
`docs/adr/0001-use-postgresql.md` biçiminde sıralı dört haneli sayı kullanılır.

**Monorepo için:**
Her alt proje kendi AD dizisini yönetir:
- `services/order-service/docs/adr/0001-use-redis-cache.md`
- `services/payment-service/docs/adr/0001-use-stripe.md`
- `platform/docs/adr/0001-kubernetes-deployment.md`

**Kurumsal çok-repo ortamı için:**
Merkezi bir ADR deposu, servis önekiyle numaralandırma yapar:
- `ORD-0001`, `PAY-0001`, `PLAT-0001`

**Örnek Kod:**
```csharp
// ADR-0003'te alınan karar (gRPC kullanımı) C# koduna nasıl yansır:

// Proje yapısı
// src/
//   OrderService/
//     docs/
//       adr/
//         0001-use-entity-framework.md
//         0002-use-mediatr-pattern.md
//         0003-use-grpc-internal-communication.md  <-- bu ADR

// gRPC servis tanımı (.proto dosyası - ADR-0003 kararının ürünü)
// Protos/order.proto
/*
syntax = "proto3";
option csharp_namespace = "OrderService.Grpc";

service OrderGrpc {
  rpc GetOrder (GetOrderRequest) returns (OrderResponse);
  rpc CreateOrder (CreateOrderRequest) returns (OrderResponse);
  rpc ListOrders (ListOrdersRequest) returns (stream OrderResponse);
}

message GetOrderRequest { string order_id = 1; }
message OrderResponse {
  string order_id = 1;
  string status = 2;
  double total_amount = 3;
}
*/

// OrderService/Services/OrderGrpcService.cs
// Bu dosyanın başına ADR referansı ekleme pratiği
/// <summary>
/// gRPC ile order sorgulama servisi.
/// Bkz: docs/adr/0003-use-grpc-internal-communication.md
/// </summary>
public class OrderGrpcService : OrderGrpc.OrderGrpcBase
{
    private readonly IOrderRepository _orderRepository;
    private readonly ILogger<OrderGrpcService> _logger;

    public OrderGrpcService(
        IOrderRepository orderRepository,
        ILogger<OrderGrpcService> logger)
    {
        _orderRepository = orderRepository;
        _logger = logger;
    }

    public override async Task<OrderResponse> GetOrder(
        GetOrderRequest request,
        ServerCallContext context)
    {
        var order = await _orderRepository.GetByIdAsync(request.OrderId);
        if (order is null)
        {
            throw new RpcException(new Status(StatusCode.NotFound,
                $"Sipariş bulunamadı: {request.OrderId}"));
        }

        return new OrderResponse
        {
            OrderId = order.Id.ToString(),
            Status = order.Status.ToString(),
            TotalAmount = (double)order.TotalAmount
        };
    }

    public override async Task ListOrders(
        ListOrdersRequest request,
        IServerStreamWriter<OrderResponse> responseStream,
        ServerCallContext context)
    {
        // ADR-0003: HTTP/2 server-side streaming özelliği
        await foreach (var order in _orderRepository.GetAllStreamAsync(context.CancellationToken))
        {
            await responseStream.WriteAsync(new OrderResponse
            {
                OrderId = order.Id.ToString(),
                Status = order.Status.ToString(),
                TotalAmount = (double)order.TotalAmount
            });
        }
    }
}

// Program.cs - gRPC servis kaydı
public static class ServiceCollectionExtensions
{
    // ADR-0003: Dahili servisler gRPC, dış API REST kullanır
    public static IServiceCollection AddOrderServices(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddGrpc(options =>
        {
            options.EnableDetailedErrors = false; // Üretimde kapalı
            options.MaxReceiveMessageSize = 1 * 1024 * 1024; // 1 MB
        });

        services.AddGrpcReflection(); // Geliştirme ortamı için

        return services;
    }
}
```

### 3. Soru: Ne zaman ADR yazmalıyız, ne zaman yazmamalıyız? Pratik bir rehber nasıl oluşturulur?

**Cevap:**
ADR yazma kararı kendisi de bir ödün analizidir. ADR yazmak zaman alır; dolayısıyla her karar için ADR yazmak bürokratik yük oluşturur. Aşağıdaki kriterler pratik bir karar rehberi sağlar.

**ADR yazılması gereken durumlar:**
- Geri döndürülmesi zor veya pahalı kararlar (veritabanı değişimi, mesajlaşma altyapısı seçimi)
- Birden fazla servisi veya ekibi etkileyen kararlar
- Ekip içinde gerçek bir tartışmaya neden olan kararlar
- Kabul edilmesi gerekmeyen ama bilinçli olarak kabul edilen ödünler
- Gelecekte "Neden böyle yaptık?" diye sorulacak kararlar

**ADR yazılmaması gereken durumlar:**
- Kod içi uygulama ayrıntıları (hangi sınıfın hangi metodu çağıracağı)
- Standart best practice'lerin uygulanması
- Geçici veya deneysel kararlar (spike/proof-of-concept)
- Kolayca geri alınabilecek değişiklikler

**Örnek Kod:**
```csharp
// ADR Karar Rehberi - Kod tabanına entegre edilmiş açıklamalar

// SENARYO 1: ADR GEREKTİREN KARAR
// Durum: Hangfire vs Quartz.NET seçimi - ikisi de geçerli, ödünler mevcut
// Bu karar için ADR-0007 yazıldı

// ADR-0007: Arka Plan İşleri için Hangfire Kullanımı
// Bağlam: Çok sayıda arka plan görevi ve yeniden deneme (retry) gereksinimi
// Karar: Hangfire (Quartz.NET yerine)
// Gerekçe: Dashboard, SQL Server entegrasyonu, daha az yapılandırma

// [Karar noktası - ADR-0007]
builder.Services.AddHangfire(config =>
{
    config.SetDataCompatibilityLevel(CompatibilityLevel.Version_180)
          .UseSimpleAssemblyNameTypeSerializer()
          .UseRecommendedSerializerSettings()
          .UseSqlServerStorage(
              builder.Configuration.GetConnectionString("HangfireConnection"),
              new SqlServerStorageOptions
              {
                  CommandBatchMaxTimeout = TimeSpan.FromMinutes(5),
                  SlidingInvisibilityTimeout = TimeSpan.FromMinutes(5),
                  QueuePollInterval = TimeSpan.Zero,
                  UseRecommendedIsolationLevel = true,
                  DisableGlobalLocks = true
              });
});

builder.Services.AddHangfireServer(options =>
{
    options.WorkerCount = Environment.ProcessorCount * 2;
    options.Queues = ["critical", "default", "low-priority"];
});

// SENARYO 2: ADR GEREKTİRMEYEN KARAR
// Durum: IOrderService içinde private helper metod adı seçimi
// Bu standart bir uygulama detayıdır, ADR gerekmez

public class OrderService : IOrderService
{
    // Bu metod adı seçimi ADR gerektirmez
    private decimal CalculateDiscountedTotal(Order order, decimal discountRate)
    {
        return order.Subtotal * (1 - discountRate);
    }
}

// SENARYO 3: ADR KARAR KONTROL LİSTESİ - Kayıt olarak tutulabilir
public static class AdrDecisionGuide
{
    /// <summary>
    /// Bir karar için ADR gerekip gerekmediğini değerlendirir.
    /// Herhangi bir soruya "evet" yanıtı ADR yazılması gerektiğini gösterir.
    /// </summary>
    public static bool RequiresAdr(
        bool isHardToReverse,          // Geri döndürmek >1 sprint alır mı?
        bool affectsMultipleTeams,     // Birden fazla ekibi etkiliyor mu?
        bool causedDebate,             // Toplantıda tartışma oldu mu?
        bool hasSignificantTradeoffs,  // Önemli ödünler var mı?
        bool affectsNonFunctional      // Kalite niteliklerini etkiliyor mu?
    )
    {
        return isHardToReverse
            || affectsMultipleTeams
            || causedDebate
            || hasSignificantTradeoffs
            || affectsNonFunctional;
    }
}
```

### 4. Soru: ADR araçları nelerdir ve bunlar C# projelerine nasıl entegre edilir?

**Cevap:**
ADR araçları iki kategoride incelenir: **komut satırı araçları** (ADR oluşturma ve yönetme için) ve **görselleştirme/arama araçları** (ADR'leri gezinebilir hale getirmek için).

Popüler araçlar:
- **adr-tools** (Bash tabanlı, klasik Nygard şablonu)
- **Log4brains** (Node.js, web arayüzü oluşturur, MkDocs ile entegre edilebilir)
- **ADR Manager** (VS Code eklentisi)
- **dotnet-adr** (.NET Global Tool, C# projeleri için özel)

C# projelerinde ADR'leri kod tabanıyla ilişkilendirmek için yorumlara `// See: docs/adr/XXXX` referansı ekleme pratiği yaygındır.

**Örnek Kod:**
```csharp
// dotnet-adr kurulumu ve kullanımı (terminal komutları)
// dotnet tool install -g adr-cli
// adr init          --> docs/adr/ klasörü oluşturur
// adr new "Use PostgreSQL as primary database"
// adr list          --> mevcut ADR'leri listeler
// adr generate toc  --> içindekiler tablosu oluşturur

// ADR-0001: PostgreSQL Birincil Veritabanı Olarak Kullanımı
// Bu kararın C# koduna etkisi:

// appsettings.json - ADR-0001 kararının yapılandırması
/*
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Database=myapp;Username=app;Password=secret"
  }
}
*/

// DbContext yapılandırması - ADR-0001 referansı
/// <remarks>
/// Veritabanı seçimi için bkz. docs/adr/0001-use-postgresql.md
/// PostgreSQL'e özgü özellikler (JSONB, Array vb.) burada kullanılabilir.
/// </remarks>
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options) { }

    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // ADR-0001: PostgreSQL JSONB tipi - SQL Server'da bu mümkün değildi
        modelBuilder.Entity<Order>()
            .Property(o => o.Metadata)
            .HasColumnType("jsonb");

        // ADR-0001: PostgreSQL dizileri
        modelBuilder.Entity<Product>()
            .Property(p => p.Tags)
            .HasColumnType("text[]");

        base.OnModelCreating(modelBuilder);
    }
}

// ServiceCollection uzantısı - ADR-0001 kararının servis kaydı
public static class DatabaseServiceExtensions
{
    // ADR-0001: Npgsql EF Core provider kullanımı
    public static IServiceCollection AddDatabase(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<ApplicationDbContext>(options =>
        {
            options.UseNpgsql(
                configuration.GetConnectionString("DefaultConnection"),
                npgsqlOptions =>
                {
                    // ADR-0001: Dayanıklılık politikası
                    npgsqlOptions.EnableRetryOnFailure(
                        maxRetryCount: 5,
                        maxRetryDelay: TimeSpan.FromSeconds(30),
                        errorCodesToAdd: null);

                    npgsqlOptions.CommandTimeout(30);
                });

            // Üretimde kapalı - performans için
            options.EnableSensitiveDataLogging(false);
            options.EnableDetailedErrors(false);
        });

        return services;
    }
}

// ADR gözden geçirme zamanını takip eden basit bir kayıt mekanizması
public record AdrReference(
    string AdrId,
    string FilePath,
    DateOnly DecisionDate,
    DateOnly? ReviewDate = null)
{
    public bool IsOverdueForReview(int reviewPeriodMonths = 12)
    {
        var deadline = ReviewDate ?? DecisionDate.AddMonths(reviewPeriodMonths);
        return DateOnly.FromDateTime(DateTime.UtcNow) > deadline;
    }
}
```

### 5. Soru: ADR yaşam döngüsü nasıl yönetilir? Superseded ve Deprecated arasındaki fark nedir?

**Cevap:**
ADR yaşam döngüsü dört temel durumdan oluşur:

- **Proposed**: ADR önerilmiş, henüz tartışma sürecinde.
- **Accepted**: Karar alınmış ve uygulamaya konulmuş.
- **Deprecated**: Karar artık geçerli değil ama yerine başka bir karar yazılmamış (sistem evrildi, bağlam tamamen değişti).
- **Superseded by ADR-XXXX**: Karar daha iyi bir kararla değiştirildi. Eski ADR silinmez; tarihsel kayıt olarak tutulur.

**Temel fark:** Deprecated — doğal sona erme; Superseded — bilinçli değiştirme.

**Örnek Kod:**
```csharp
// ADR'nin superseded edilmesi senaryosu:

// ----- docs/adr/0004-use-in-memory-cache.md -----
/*
# ADR-0004: Tek Sunucu Önbelleği için In-Memory Cache Kullanımı

## Durum
Superseded by ADR-0012

## Bağlam
Uygulama başlangıçta tek bir sunucuda çalışıyordu. IMemoryCache performans
gereksinimlerini karşılamak için yeterliydi.

## Karar
ASP.NET Core IMemoryCache kullanımı.

## Sonuçlar
Olumlu: Basit yapılandırma, sıfır altyapı maliyeti
Olumsuz: [ADR-0012 bağlamında] Yatay ölçeklendirme kararıyla birden fazla
sunucu örneğinde önbellek tutarsızlığı sorununa yol açtı.
*/

// ----- docs/adr/0012-use-redis-distributed-cache.md -----
/*
# ADR-0012: Dağıtık Önbellek için Redis Kullanımı

## Durum
Accepted

## Bağlam
ADR-0004 kapsamında In-Memory Cache kullanılıyordu. Kubernetes üzerinde
yatay ölçeklendirme (3+ pod) kararı (ADR-0010) alındıktan sonra In-Memory
Cache, pod'lar arası tutarsız önbellek durumuna yol açtı. Üretim ortamında
önbellek çakışmalarından kaynaklanan hata oranı %2.3'e ulaştı.

## Karar
ADR-0004'teki IMemoryCache yerine Redis ile IDistributedCache kullanımına geçiyoruz.

## Sonuçlar
Olumlu: Pod'lar arası tutarlı önbellek, yüksek erişilebilirlik
Olumsuz: Redis altyapısı bakım maliyeti, ağ gecikmesi ekleniyor
*/

// IMemoryCache (ADR-0004 - eski) -> IDistributedCache (ADR-0012 - yeni)

// ESKİ UYGULAMA (ADR-0004)
public class LegacyCatalogService
{
    private readonly IMemoryCache _cache;  // ADR-0004 kararı

    public LegacyCatalogService(IMemoryCache cache)
    {
        _cache = cache;
    }

    public async Task<Product?> GetProductAsync(int id)
    {
        if (_cache.TryGetValue($"product:{id}", out Product? product))
            return product;

        product = await FetchFromDatabaseAsync(id);
        _cache.Set($"product:{id}", product, TimeSpan.FromMinutes(10));
        return product;
    }

    private Task<Product?> FetchFromDatabaseAsync(int id) => Task.FromResult<Product?>(null);
}

// YENİ UYGULAMA (ADR-0012)
public class CatalogService
{
    private readonly IDistributedCache _cache;  // ADR-0012 kararı
    private readonly ILogger<CatalogService> _logger;

    public CatalogService(
        IDistributedCache cache,
        ILogger<CatalogService> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public async Task<Product?> GetProductAsync(int id, CancellationToken ct = default)
    {
        var cacheKey = $"product:{id}";
        var cachedBytes = await _cache.GetAsync(cacheKey, ct);

        if (cachedBytes is not null)
        {
            _logger.LogDebug("Önbellek isabeti: {CacheKey}", cacheKey);
            return JsonSerializer.Deserialize<Product>(cachedBytes);
        }

        var product = await FetchFromDatabaseAsync(id, ct);
        if (product is not null)
        {
            var serialized = JsonSerializer.SerializeToUtf8Bytes(product);
            await _cache.SetAsync(
                cacheKey,
                serialized,
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10),
                    SlidingExpiration = TimeSpan.FromMinutes(5)
                },
                ct);
        }

        return product;
    }

    private Task<Product?> FetchFromDatabaseAsync(int id, CancellationToken ct) =>
        Task.FromResult<Product?>(null);
}

// ServiceCollection - ADR-0012 kaydı
public static class CacheServiceExtensions
{
    // ADR-0012: Redis dağıtık önbellek
    public static IServiceCollection AddDistributedCaching(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddStackExchangeRedisCache(options =>
        {
            options.Configuration = configuration.GetConnectionString("Redis");
            options.InstanceName = "myapp:";
        });

        return services;
    }
}
```

### 6. Soru: Markdown tabanlı ADR şablonları nasıl standartlaştırılır? MADR şablonu nedir?

**Cevap:**
MADR (Markdown Architectural Decision Records), Michael Nygard'ın orijinal şablonundan esinlenerek geliştirilen ve daha fazla yapı sunan modern bir ADR formatıdır. "Considered Options" ve "Decision Outcome" bölümleri, ödün analizini sistematik hale getirir.

MADR formatının öne çıkan özellikleri:
- Değerlendirilen alternatifleri açıkça listeler
- Her alternatifin artılarını ve eksilerini belgeler
- Karar çıktısını (seçilen seçenek + gerekçe) ayrı bir bölümde gösterir
- Makine tarafından işlenebilir (otomatik indeks oluşturma için)

**Örnek Kod:**
```markdown
---
status: "accepted"
date: 2025-11-10
decision-makers: ["Ahmet Yılmaz", "Fatma Kaya", "Mehmet Demir"]
consulted: ["DevOps Ekibi", "Güvenlik Ekibi"]
informed: ["Tüm Mühendislik"]
---

# ADR-0008: API Gateway Katmanı için Yeniden Denenebilir İstekler

## Bağlam ve Sorun

Mikroservis mimarimizde API Gateway, downstream servislere istek iletmektedir.
Geçici ağ hataları veya servis yeniden başlatmaları sırasında kullanıcılar 5xx
hataları almaktadır. Son 30 günlük veriye göre %0.8 başarısız istek oranı
mevcuttur ve bunların %65'i 1 saniye içinde yeniden deneme ile başarılı olmaktadır.

## Karar Kriterleri

* Kullanıcıya görünür başarısızlık oranının azaltılması
* Downstream servislerde aşırı yük oluşturulmaması
* Sistemin idempotent olmayan isteklerde çift-işlem riskinin yönetimi

## Değerlendirilen Seçenekler

* **Seçenek A**: Polly ile yeniden deneme (exponential backoff + jitter)
* **Seçenek B**: YARP reverse proxy yerleşik yeniden deneme
* **Seçenek C**: İstemci tarafında yeniden deneme (frontend/mobile)
* **Seçenek D**: Yeniden deneme yok, sadece devre kesici

## Karar Çıktısı

Seçilen seçenek: **Seçenek A - Polly ile yeniden deneme**, çünkü
Gateway katmanında merkezi kontrol sağlar, tüm downstream servisler
için tek yapılandırma noktası oluşturur ve YARP'tan bağımsızdır.

### Olumlu Sonuçlar

* Geçici hatalarda otomatik kurtarma, kullanıcı deneyimi iyileşir
* Merkezi yeniden deneme mantığı tüm servislere tek yerden uygulanır

### Olumsuz Sonuçlar

* İdempotent olmayan POST isteklerinde çift-işlem riski (hafifletme: sadece GET/PUT yeniden denen)
* Toplam gecikme artabilir (hafifletme: jitter ile yayılan backoff)
```

```csharp
// ADR-0008 kararının C# YARP + Polly implementasyonu

// Program.cs
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

// Polly yeniden deneme politikası - ADR-0008
builder.Services.AddResiliencePipeline("api-gateway-retry", pipelineBuilder =>
{
    pipelineBuilder
        .AddRetry(new RetryStrategyOptions
        {
            // Yalnızca idempotent metodlar için yeniden dene - ADR-0008 kararı
            ShouldHandle = new PredicateBuilder()
                .Handle<HttpRequestException>()
                .HandleResult<HttpResponseMessage>(r =>
                    r.StatusCode >= System.Net.HttpStatusCode.InternalServerError
                    && r.RequestMessage?.Method != HttpMethod.Post),

            MaxRetryAttempts = 3,
            Delay = TimeSpan.FromMilliseconds(200),
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true // Thundering herd sorununu önler
        })
        .AddCircuitBreaker(new CircuitBreakerStrategyOptions
        {
            FailureRatio = 0.5,
            MinimumThroughput = 10,
            SamplingDuration = TimeSpan.FromSeconds(30),
            BreakDuration = TimeSpan.FromSeconds(15)
        })
        .AddTimeout(TimeSpan.FromSeconds(10));
});

// Gateway middleware - ADR-0008 politikasının uygulanması
public class RetryMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ResiliencePipeline<HttpResponseMessage> _pipeline;

    public RetryMiddleware(
        RequestDelegate next,
        ResiliencePipelineProvider<string> pipelineProvider)
    {
        _next = next;
        _pipeline = pipelineProvider.GetPipeline<HttpResponseMessage>("api-gateway-retry");
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // ADR-0008: POST istekleri yeniden denenmez (idempotency)
        if (context.Request.Method == HttpMethods.Post)
        {
            await _next(context);
            return;
        }

        await _pipeline.ExecuteAsync(async ct =>
        {
            await _next(context);
            return new HttpResponseMessage(
                (System.Net.HttpStatusCode)context.Response.StatusCode);
        }, context.RequestAborted);
    }
}
```

## Best Practices

### 1. ADR Yazım Kalitesi
- Her ADR tek bir kararı belgeleyin; birden fazla kararı tek ADR'de toplamaktan kaçının
- Bağlam bölümünde ölçüm verilerine (metrikler, benchmark sonuçları) yer verin
- Gelecekteki okuyucuyu hedef alın; "şu an herkes biliyor" varsayımından kaçının
- Kod değişiklikleriyle eş zamanlı ADR oluşturun; geriye dönük ADR yazmak bilgi kaybına yol açar
- Yorumlara `// See: docs/adr/XXXX` referansı ekleyerek kod ile ADR arasında köprü kurun

### 2. Ekip Süreçleri
- ADR pull request'leri en az iki teknik lider tarafından gözden geçirilmeli
- Sprint planlamasında ADR yazımı için zaman ayırın
- Yeni ekip üyelerine ADR okuma onboarding sürecine dahil edilmeli
- Her üç ayda bir ADR gözden geçirme (ADR health check) toplantısı düzenleyin

### 3. Araç Entegrasyonu
- ADR'leri `docs/adr/` altında versiyonlayın ve ana kod tabanıyla birlikte gözden geçirin
- CI/CD pipeline'da ADR lint kontrolü ekleyin (gerekli bölümlerin varlığını doğrulayan)
- MkDocs veya Docusaurus ile ADR'leri aranabilir dokümantasyon sitesine dahil edin

## Kaynaklar

- [Michael Nygard - Documenting Architecture Decisions](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions)
- [MADR Şablonu](https://adr.github.io/madr/)
- [adr-tools CLI](https://github.com/npryce/adr-tools)
- [Log4brains - ADR Web Arayüzü](https://github.com/thomvaill/log4brains)
- [Polly .NET Resilience Library](https://www.thepollyproject.org/)
- [YARP Reverse Proxy](https://microsoft.github.io/reverse-proxy/)
- [Microsoft - Architecture Decision Records](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/)
