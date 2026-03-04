# Ödün Analizi (Trade-off Analysis)

## Genel Bakış

Ödün analizi (Trade-off Analysis), mimari kararların birden fazla kalite niteliği üzerindeki etkisini sistematik biçimde değerlendirme sürecidir. Her mimari karar; performans, güvenlik, sürdürülebilirlik, ölçeklenebilirlik ve maliyet gibi nitelikler arasında bir denge kurmak zorundadır. ATAM (Architecture Tradeoff Analysis Method), bu dengeyi yapılandırılmış bir çerçevede analiz etmeye olanak tanır. Senior geliştiriciler için ödün analizi, hem teknik kararları savunabilmek hem de paydaşlarla şeffaf iletişim kurmak açısından temel bir yetkinliktir.

## Mülakat Soruları ve Cevapları

### 1. Soru: ATAM (Architecture Tradeoff Analysis Method) nedir ve hangi adımlardan oluşur?

**Cevap:**
ATAM, SEI (Software Engineering Institute) tarafından geliştirilmiş sistematik bir mimari değerlendirme yöntemidir. Bir mimarinin kalite niteliklerini ne ölçüde desteklediğini, risk noktalarını ve ödünleri tespit etmek için yapılandırılmış bir süreç sunar.

ATAM dokuz adımdan oluşur:
1. **ATAM'ın Sunumu**: Değerlendirme süreci açıklanır
2. **İş Hedeflerinin Sunumu**: Mimariyi şekillendiren iş gereksinimleri ortaya konur
3. **Mimarinin Sunumu**: Mimari ekip tarafından açıklanır
4. **Mimari Yaklaşımların Belirlenmesi**: Kullanılan mimari taktikler listelenir
5. **Kalite Niteliği Ağacının Oluşturulması**: QA senaryoları önceliklendirilir
6. **Mimari Yaklaşımların Analizi**: Her yaklaşımın QA'lara etkisi değerlendirilir
7. **Beyin Fırtınası ve Risk Senaryoları**: Paydaşlardan senaryo toplanır
8. **Mimari Yaklaşımların Yeniden Analizi**: Yeni senaryolarla yaklaşımlar gözden geçirilir
9. **Sonuçların Sunumu**: Riskler, hassasiyet noktaları ve ödünler raporlanır

**Kritik ATAM kavramları:**
- **Risk noktası (Risk Point)**: Kalite niteliklerini olumsuz etkileyebilecek mimari karar
- **Hassasiyet noktası (Sensitivity Point)**: Bir kararın tek bir kalite niteliği üzerinde yüksek etkili olduğu yer
- **Ödün noktası (Tradeoff Point)**: Bir kararın birden fazla kalite niteliğini etkilediği yer

**Örnek Kod:**
```csharp
// ATAM Kalite Niteliği Senaryosu - C# modeli
// Senaryo: "Sisteme 10.000 eş zamanlı kullanıcı bağlandığında
//           95. yüzdelik yanıt süresi 500ms altında kalmalıdır"

public record QualityAttributeScenario(
    string Id,
    string QualityAttribute,       // Performans, Güvenlik, Sürdürülebilirlik vb.
    string Stimulus,               // Tetikleyici: 10.000 eş zamanlı kullanıcı
    string StimulusSource,         // Kaynak: Kullanıcı istekleri
    string Environment,            // Ortam: Normal işletim
    string Artifact,               // Etkilenen: API servisi
    string Response,               // Yanıt: İstekleri işle ve yanıt dön
    string ResponseMeasure,        // Ölçüm: P95 < 500ms
    Priority Priority              // H (Yüksek) / M (Orta) / L (Düşük)
);

public enum Priority { High, Medium, Low }

// ATAM değerlendirme matrisi
public class AtamEvaluationMatrix
{
    private readonly List<ArchitecturalApproach> _approaches = [];
    private readonly List<QualityAttributeScenario> _scenarios = [];

    public void AddApproach(ArchitecturalApproach approach) =>
        _approaches.Add(approach);

    public void AddScenario(QualityAttributeScenario scenario) =>
        _scenarios.Add(scenario);

    /// <summary>
    /// Her mimari yaklaşımın kalite niteliği senaryolarına etkisini değerlendirir.
    /// Sonuç: Risk, Hassasiyet veya Ödün noktası olarak sınıflandırılır.
    /// </summary>
    public AtamAnalysisResult Analyze()
    {
        var risks = new List<RiskPoint>();
        var sensitivities = new List<SensitivityPoint>();
        var tradeoffs = new List<TradeoffPoint>();

        foreach (var approach in _approaches)
        {
            var impacts = _scenarios
                .Select(s => new QaImpact(s, approach.GetImpact(s)))
                .ToList();

            var negativeImpacts = impacts.Where(i => i.Impact < 0).ToList();
            var positiveImpacts = impacts.Where(i => i.Impact > 0).ToList();

            // Risk noktası: Yüksek öncelikli senaryo olumsuz etkileniyor
            var riskScenarios = negativeImpacts
                .Where(i => i.Scenario.Priority == Priority.High)
                .ToList();

            if (riskScenarios.Any())
                risks.Add(new RiskPoint(approach, riskScenarios));

            // Ödün noktası: Hem olumlu hem olumsuz etki var
            if (positiveImpacts.Any() && negativeImpacts.Any())
                tradeoffs.Add(new TradeoffPoint(approach, positiveImpacts, negativeImpacts));

            // Hassasiyet noktası: Tek bir QA üzerinde yüksek etki
            var highImpacts = impacts.Where(i => Math.Abs(i.Impact) >= 8).ToList();
            if (highImpacts.Count == 1)
                sensitivities.Add(new SensitivityPoint(approach, highImpacts[0]));
        }

        return new AtamAnalysisResult(risks, sensitivities, tradeoffs);
    }
}

public record ArchitecturalApproach(string Name, string Description)
{
    public int GetImpact(QualityAttributeScenario scenario) => 0; // Gerçek implementasyonda doldurulur
}

public record QaImpact(QualityAttributeScenario Scenario, int Impact); // -10 (çok olumsuz) +10 (çok olumlu)
public record RiskPoint(ArchitecturalApproach Approach, List<QaImpact> AffectedScenarios);
public record SensitivityPoint(ArchitecturalApproach Approach, QaImpact HighImpact);
public record TradeoffPoint(ArchitecturalApproach Approach, List<QaImpact> Positives, List<QaImpact> Negatives);
public record AtamAnalysisResult(List<RiskPoint> Risks, List<SensitivityPoint> Sensitivities, List<TradeoffPoint> Tradeoffs);
```

### 2. Soru: Kalite nitelikleri (quality attributes) arasındaki klasik ödünler nelerdir? Performans ile güvenlik arasındaki ödün nasıl yönetilir?

**Cevap:**
Mimari kararlar, kalite nitelikleri arasında kaçınılmaz ödünler oluşturur. En sık karşılaşılan ödünler:

| Nitelik A      | Nitelik B        | Ödün Açıklaması                                      |
|----------------|------------------|------------------------------------------------------|
| Performans     | Güvenlik         | Şifreleme/doğrulama işlemleri gecikme ekler         |
| Ölçeklenebilirlik | Tutarlılık    | Dağıtık sistemlerde eventual consistency             |
| Sürdürülebilirlik | Performans    | Soyutlama katmanları ek overhead getirir            |
| Esneklik       | Güvenilirlik     | Dinamik konfigürasyon karmaşıklık ve hata riski     |
| Maliyet        | Yüksek Erişilebilirlik | Çok bölgeli dağıtım altyapı maliyetini artırır |

Performans ile güvenlik arasındaki ödün yönetiminin anahtarı, neyin korunması gerektiğini önceliklendirmek ve güvenlik katmanlarını akıllıca konumlandırmaktır.

**Örnek Kod:**
```csharp
// Senaryo: JWT doğrulama - Performans vs Güvenlik ödünü

// YAKLAŞIM A: Her istekte tam JWT doğrulama (Yüksek Güvenlik, Düşük Performans)
public class StrictJwtValidationMiddleware
{
    private readonly RequestDelegate _next;
    private readonly JwtSecurityTokenHandler _handler;
    private readonly TokenValidationParameters _validationParams;

    public StrictJwtValidationMiddleware(
        RequestDelegate next,
        IOptions<JwtSettings> settings)
    {
        _next = next;
        _handler = new JwtSecurityTokenHandler();
        _validationParams = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            // Her istekte imzalama anahtarı için dış kaynak sorgusu
            IssuerSigningKeyResolver = (token, securityToken, kid, parameters) =>
                FetchSigningKeysFromAuthServer(kid) // ~50ms gecikme
        };
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Her istekte tam doğrulama: güvenli ama yavaş (~60ms ek gecikme)
        var token = ExtractToken(context.Request);
        _handler.ValidateToken(token, _validationParams, out _);
        await _next(context);
    }

    private IEnumerable<SecurityKey> FetchSigningKeysFromAuthServer(string kid)
        => []; // Dış sunucu çağrısı
    private string? ExtractToken(HttpRequest request) => null;
}

// YAKLAŞIM B: İmzalama anahtarı önbellekleme (Kabul Edilebilir Güvenlik, Yüksek Performans)
// Ödün: Anahtarın iptal edilmesi durumunda önbellekteki süre kadar gecikme riski

public class CachedJwtValidationMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IMemoryCache _keyCache;
    private readonly JwtSecurityTokenHandler _handler;
    private readonly ILogger<CachedJwtValidationMiddleware> _logger;

    // Ödün kararı: Anahtar önbellek süresi
    // Kısa süre (1dk): Güvenli, daha fazla ağ çağrısı
    // Uzun süre (1saat): Hızlı, iptal gecikmesi riski
    private static readonly TimeSpan KeyCacheDuration = TimeSpan.FromMinutes(5);

    public CachedJwtValidationMiddleware(
        RequestDelegate next,
        IMemoryCache keyCache,
        ILogger<CachedJwtValidationMiddleware> logger)
    {
        _next = next;
        _keyCache = keyCache;
        _handler = new JwtSecurityTokenHandler();
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var token = ExtractToken(context.Request);
        if (token is null)
        {
            context.Response.StatusCode = 401;
            return;
        }

        var validationParams = await GetValidationParamsAsync(token);
        _handler.ValidateToken(token, validationParams, out _);
        await _next(context);
    }

    private async Task<TokenValidationParameters> GetValidationParamsAsync(string token)
    {
        // İmzalama anahtarını önbellekten al veya getir
        var kid = ExtractKidFromToken(token);
        var signingKey = await _keyCache.GetOrCreateAsync(
            $"signing-key:{kid}",
            async entry =>
            {
                entry.AbsoluteExpirationRelativeToNow = KeyCacheDuration;
                _logger.LogInformation("İmzalama anahtarı önbellekleniyor: {Kid}", kid);
                return await FetchSigningKeyAsync(kid);
            });

        return new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = signingKey,
            ValidateLifetime = true,
            ValidateIssuer = true,
            ValidateAudience = true,
            // Ödün: Saat sapmasına tolerans - güvenlik zayıflar ama saat sorunları azalır
            ClockSkew = TimeSpan.FromMinutes(2)
        };
    }

    private string ExtractKidFromToken(string token) => "default";
    private Task<SecurityKey> FetchSigningKeyAsync(string kid) => Task.FromResult<SecurityKey>(null!);
    private string? ExtractToken(HttpRequest request) => null;
    private string ExtractKidFromToken2(string token) => "default";
}

// YAKLAŞIM C: Kritik endpoint'ler için tam doğrulama, diğerleri için önbellekli
// ADR kararı: Risk sınıflandırmasına göre seçici doğrulama

[AttributeUsage(AttributeTargets.Method | AttributeTargets.Class)]
public class StrictAuthAttribute : Attribute { }

public class SelectiveJwtMiddleware
{
    private readonly RequestDelegate _next;

    public SelectiveJwtMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var endpoint = context.GetEndpoint();
        var requiresStrictAuth = endpoint?.Metadata.GetMetadata<StrictAuthAttribute>() != null;

        if (requiresStrictAuth)
        {
            // Para transferi, şifre değişimi gibi kritik işlemler
            await ValidateWithFullVerification(context);
        }
        else
        {
            // Ürün listesi, arama gibi düşük riskli işlemler
            await ValidateWithCachedKeys(context);
        }

        await _next(context);
    }

    private Task ValidateWithFullVerification(HttpContext context) => Task.CompletedTask;
    private Task ValidateWithCachedKeys(HttpContext context) => Task.CompletedTask;
}
```

### 3. Soru: Karar matrisi (decision matrix) nasıl oluşturulur ve ağırlıklandırma nasıl yapılır?

**Cevap:**
Karar matrisi, birden fazla alternatif arasında seçim yaparken değerlendirme kriterlerini sistematik ve sayısal olarak karşılaştıran bir araçtır. Ağırlıklı karar matrisi, her kriterin önem derecesini de hesaba katar.

**Adımlar:**
1. Alternatifleri listele (A1, A2, A3...)
2. Değerlendirme kriterlerini belirle
3. Her kritere ağırlık ata (toplamı 100 veya 1.0 olmalı)
4. Her alternatifi her kritere göre puanla (1-5 veya 1-10)
5. Ağırlıklı puan hesapla (ağırlık × puan)
6. Toplam ağırlıklı puana göre sırala

**Örnek Kod:**
```csharp
// Mesajlaşma altyapısı seçimi: RabbitMQ vs Apache Kafka vs Azure Service Bus

public class WeightedDecisionMatrix
{
    private readonly List<DecisionCriterion> _criteria;
    private readonly List<Alternative> _alternatives;

    public WeightedDecisionMatrix(
        List<DecisionCriterion> criteria,
        List<Alternative> alternatives)
    {
        var totalWeight = criteria.Sum(c => c.Weight);
        if (Math.Abs(totalWeight - 1.0) > 0.001)
            throw new ArgumentException("Kriter ağırlıklarının toplamı 1.0 olmalıdır.");

        _criteria = criteria;
        _alternatives = alternatives;
    }

    public DecisionMatrixResult Evaluate(Dictionary<(string Alternative, string Criterion), int> scores)
    {
        var results = _alternatives.Select(alt =>
        {
            var weightedScore = _criteria.Sum(criterion =>
            {
                var score = scores.TryGetValue((alt.Name, criterion.Name), out var s) ? s : 0;
                return criterion.Weight * score;
            });

            return new AlternativeScore(alt, weightedScore);
        })
        .OrderByDescending(r => r.WeightedScore)
        .ToList();

        return new DecisionMatrixResult(results, results.First());
    }
}

public record DecisionCriterion(string Name, double Weight, string Description);
public record Alternative(string Name, string Description);
public record AlternativeScore(Alternative Alternative, double WeightedScore);
public record DecisionMatrixResult(List<AlternativeScore> Rankings, AlternativeScore Recommended);

// Kullanım örneği: Mesajlaşma altyapısı seçimi
public static class MessagingDecisionExample
{
    public static DecisionMatrixResult EvaluateMessagingOptions()
    {
        // Değerlendirme kriterleri ve ağırlıkları
        var criteria = new List<DecisionCriterion>
        {
            new("Performans",          0.25, "Saniye başına mesaj kapasitesi ve gecikme"),
            new("Operasyonel Kolaylık", 0.20, "Kurulum, bakım ve izleme"),
            new("Ölçeklenebilirlik",    0.20, "Yüksek hacimde büyüyebilme"),
            new("Ekip Deneyimi",        0.15, "Mevcut ekip yetkinliği"),
            new("Maliyet",              0.10, "Lisans ve altyapı maliyeti"),
            new("Mesaj Sırası Güvencesi", 0.10, "Kesinlikle-bir-kez teslim")
        };

        var alternatives = new List<Alternative>
        {
            new("RabbitMQ",           "AMQP protokolü, esnek routing"),
            new("Apache Kafka",       "Distributed log, yüksek throughput"),
            new("Azure Service Bus",  "Yönetilen PaaS, Azure ekosistemi")
        };

        var matrix = new WeightedDecisionMatrix(criteria, alternatives);

        // Her alternatifte kriterler 1-5 arası puanlandı (5 = en iyi)
        var scores = new Dictionary<(string, string), int>
        {
            // RabbitMQ
            { ("RabbitMQ", "Performans"),            4 }, // Yüksek throughput, düşük gecikme
            { ("RabbitMQ", "Operasyonel Kolaylık"),  4 }, // Yönetim arayüzü mevcut
            { ("RabbitMQ", "Ölçeklenebilirlik"),     3 }, // Kümeleme mümkün ama karmaşık
            { ("RabbitMQ", "Ekip Deneyimi"),         5 }, // Ekip deneyimli
            { ("RabbitMQ", "Maliyet"),               5 }, // Açık kaynak
            { ("RabbitMQ", "Mesaj Sırası Güvencesi"), 4 }, // Durable queue desteği

            // Apache Kafka
            { ("Apache Kafka", "Performans"),            5 }, // Milyonlarca mesaj/sn
            { ("Apache Kafka", "Operasyonel Kolaylık"),  2 }, // Karmaşık yapılandırma
            { ("Apache Kafka", "Ölçeklenebilirlik"),     5 }, // Yatay ölçeklendirme mükemmel
            { ("Apache Kafka", "Ekip Deneyimi"),         2 }, // Sınırlı deneyim
            { ("Apache Kafka", "Maliyet"),               3 }, // Açık kaynak ama altyapı maliyeti
            { ("Apache Kafka", "Mesaj Sırası Güvencesi"), 5 }, // Partition bazlı sıra garantisi

            // Azure Service Bus
            { ("Azure Service Bus", "Performans"),            3 }, // Yeterli ama Kafka kadar değil
            { ("Azure Service Bus", "Operasyonel Kolaylık"),  5 }, // Tam yönetilen servis
            { ("Azure Service Bus", "Ölçeklenebilirlik"),     4 }, // Otomatik ölçeklendirme
            { ("Azure Service Bus", "Ekip Deneyimi"),         3 }, // Orta düzey deneyim
            { ("Azure Service Bus", "Maliyet"),               2 }, // Kullanıma göre ücret
            { ("Azure Service Bus", "Mesaj Sırası Güvencesi"), 5 }  // FIFO desteği
        };

        var result = matrix.Evaluate(scores);

        // RabbitMQ: 4.15, Azure Service Bus: 3.75, Kafka: 3.65 (varsayımsal)
        // Ekip deneyimi ağırlığı %15 olduğunda Kafka'nın performans avantajı azalır

        return result;
    }
}
```

### 4. Soru: Performans ile sürdürülebilirlik (maintainability) arasındaki ödünü C# örnekleriyle açıklayınız.

**Cevap:**
Performans optimizasyonları genellikle kodu daha karmaşık ve bakımı daha zor hale getirir. Tersi de geçerlidir: temiz, okunabilir kod her zaman en yüksek performansı vermez. Bu ödün, yazılım mühendisliğinin en sık karşılaşılan ikilemleri arasındadır.

Temel ödün alanları:
- **Soyutlama vs performans**: Arayüz ve sanal metod çağrıları JIT optimizasyonunu zorlaştırır
- **Okunabilirlik vs bellek verimliliği**: LINQ zincirleri okunabilir ama ara koleksiyonlar oluşturur
- **Bağımsızlık vs önbellekleme**: Saf fonksiyonlar test edilebilir ama memoization eklemek karmaşıklaştırır
- **Genel amaçlı vs özel amaçlı**: Generic çözümler yeniden kullanılabilir ama özel çözümler daha hızlı

**Örnek Kod:**
```csharp
// SENARYO: Büyük veri setinde filtreleme

// YAKLAŞIM A: Temiz, okunabilir LINQ (Yüksek Sürdürülebilirlik, Orta Performans)
public class ReadableOrderProcessor
{
    public IEnumerable<OrderSummary> GetHighValuePendingOrders(
        IEnumerable<Order> orders,
        decimal minimumAmount)
    {
        // Açık ve anlaşılır - her geliştirici anlayabilir
        return orders
            .Where(o => o.Status == OrderStatus.Pending)
            .Where(o => o.TotalAmount >= minimumAmount)
            .OrderByDescending(o => o.TotalAmount)
            .Take(100)
            .Select(o => new OrderSummary(o.Id, o.TotalAmount, o.CreatedAt));
        // Ödün: Her Where ayrı bir iteration, OrderByDescending tüm koleksiyonu sıralar
    }
}

// YAKLAŞIM B: Performans odaklı - manuel iterasyon (Düşük Sürdürülebilirlik, Yüksek Performans)
public class OptimizedOrderProcessor
{
    public List<OrderSummary> GetHighValuePendingOrders(
        ReadOnlySpan<Order> orders,     // Span - yığın bellek, GC baskısı yok
        decimal minimumAmount)
    {
        // En fazla 100 sonuç, küçük liste
        var result = new List<OrderSummary>(capacity: 100);

        // Tek geçişte filtrele - LINQ'nun iki Where'i gibi iki geçiş yok
        foreach (ref readonly var order in orders)
        {
            if (order.Status != OrderStatus.Pending) continue;
            if (order.TotalAmount < minimumAmount) continue;

            result.Add(new OrderSummary(order.Id, order.TotalAmount, order.CreatedAt));
            if (result.Count >= 100) break; // Erken çıkış
        }

        // Yerinde sıralama - ek koleksiyon oluşturulmaz
        result.Sort((a, b) => b.Amount.CompareTo(a.Amount));
        return result;

        // Ödün: Daha hızlı ama LINQ'nun açıklığını yitirdi,
        // olası mantık hataları test olmadan gözden kaçabilir
    }
}

// YAKLAŞIM C: Dengeli yaklaşım - Açık olmayan parçaları izole et (ADR kararı)
// Bu yaklaşım genellikle en iyi ödün dengesini sunar

public class BalancedOrderProcessor
{
    // Dış arayüz temiz ve okunabilir
    public IReadOnlyList<OrderSummary> GetHighValuePendingOrders(
        IEnumerable<Order> orders,
        decimal minimumAmount,
        int maxResults = 100)
    {
        return FilterAndRankOrders(orders, minimumAmount, maxResults);
    }

    // Performans kritik iç metot - açıklayan yorumlarla
    private static List<OrderSummary> FilterAndRankOrders(
        IEnumerable<Order> source,
        decimal minimumAmount,
        int maxResults)
    {
        // Not: LINQ yerine manuel iterasyon - benchmark sonucu %35 iyileşme
        // Bkz: docs/benchmarks/order-processing-2025.md
        var result = new List<OrderSummary>(capacity: Math.Min(maxResults, 128));

        foreach (var order in source)
        {
            if (order.Status != OrderStatus.Pending || order.TotalAmount < minimumAmount)
                continue;

            // Sıralı ekleme - sonradan Sort() yerine
            var summary = new OrderSummary(order.Id, order.TotalAmount, order.CreatedAt);
            var insertIndex = result.BinarySearch(summary, OrderSummaryComparer.Instance);
            result.Insert(insertIndex < 0 ? ~insertIndex : insertIndex, summary);

            if (result.Count > maxResults)
                result.RemoveAt(result.Count - 1); // En düşük değeri çıkar
        }

        return result;
    }
}

// Birim test edilebilir sıralama mantığı - sürdürülebilirlik için izole
public class OrderSummaryComparer : IComparer<OrderSummary>
{
    public static readonly OrderSummaryComparer Instance = new();

    public int Compare(OrderSummary? x, OrderSummary? y)
    {
        if (x is null && y is null) return 0;
        if (x is null) return 1;
        if (y is null) return -1;
        return y.Amount.CompareTo(x.Amount); // Azalan sıra
    }
}

public record Order(Guid Id, OrderStatus Status, decimal TotalAmount, DateTime CreatedAt);
public enum OrderStatus { Pending, Processing, Completed, Cancelled }
public record OrderSummary(Guid Id, decimal Amount, DateTime CreatedAt);
```

### 5. Soru: Ölçeklenebilirlik ile veri tutarlılığı arasındaki ödün nasıl analiz edilir? CAP teoremi pratikte ne anlama gelir?

**Cevap:**
CAP teoremi (Consistency, Availability, Partition Tolerance) dağıtık sistemlerde üç niteliği aynı anda tam olarak sağlamanın mümkün olmadığını belirtir. Ağ bölünmesi (partition) pratikte kaçınılmaz olduğundan, gerçek seçim **CP** (tutarlılık + bölünme toleransı) veya **AP** (erişilebilirlik + bölünme toleransı) arasındadır.

**Pratik anlamları:**
- **CP sistemi**: Ağ bölünmesinde bazı istekler reddedilir (hata döner), ancak dönen veri her zaman günceldir. Örnek: Para transferi.
- **AP sistemi**: Ağ bölünmesinde eski veri dönebilir (eventual consistency), ancak sistem asla hata vermez. Örnek: Ürün kataloğu, öneri sistemi.

**Örnek Kod:**
```csharp
// CP vs AP seçiminin C# implementasyona etkisi

// SENARYO: Banka transferi - CP gerektirir
// "Bakiye %100 doğru olmalı, hizmet kesilmesi kabul edilebilir"

public class StronglyConsistentTransferService
{
    private readonly IDbContextFactory<BankContext> _contextFactory;
    private readonly IDistributedLock _distributedLock;
    private readonly ILogger<StronglyConsistentTransferService> _logger;

    public StronglyConsistentTransferService(
        IDbContextFactory<BankContext> contextFactory,
        IDistributedLock distributedLock,
        ILogger<StronglyConsistentTransferService> logger)
    {
        _contextFactory = contextFactory;
        _distributedLock = distributedLock;
        _logger = logger;
    }

    public async Task<TransferResult> TransferAsync(
        Guid fromAccountId,
        Guid toAccountId,
        decimal amount,
        CancellationToken ct)
    {
        // CP kararı: Dağıtık kilit - sadece bir node işlemi yürütür
        // Ödün: Diğer node'lar kilidi bekler (gecikme artar)
        var lockKey = $"transfer:{string.Join("_", new[] { fromAccountId, toAccountId }.OrderBy(id => id))}";

        await using var lockHandle = await _distributedLock.AcquireAsync(lockKey, TimeSpan.FromSeconds(30), ct);
        if (lockHandle is null)
        {
            _logger.LogWarning("Kilit alınamadı: {LockKey}", lockKey);
            throw new ServiceUnavailableException("Transfer servisi şu an meşgul, lütfen tekrar deneyin.");
        }

        await using var context = await _contextFactory.CreateDbContextAsync(ct);
        await using var transaction = await context.Database.BeginTransactionAsync(
            System.Data.IsolationLevel.Serializable, ct); // En güçlü izolasyon seviyesi

        try
        {
            var fromAccount = await context.Accounts
                .Where(a => a.Id == fromAccountId)
                .FirstOrDefaultAsync(ct)
                ?? throw new AccountNotFoundException(fromAccountId);

            var toAccount = await context.Accounts
                .Where(a => a.Id == toAccountId)
                .FirstOrDefaultAsync(ct)
                ?? throw new AccountNotFoundException(toAccountId);

            if (fromAccount.Balance < amount)
                throw new InsufficientFundsException(fromAccountId, amount, fromAccount.Balance);

            fromAccount.Balance -= amount;
            toAccount.Balance += amount;

            context.TransferAudits.Add(new TransferAudit
            {
                FromAccountId = fromAccountId,
                ToAccountId = toAccountId,
                Amount = amount,
                ExecutedAt = DateTime.UtcNow
            });

            await context.SaveChangesAsync(ct);
            await transaction.CommitAsync(ct);

            return TransferResult.Success(fromAccount.Balance);
        }
        catch
        {
            await transaction.RollbackAsync(ct);
            throw;
        }
    }
}

// SENARYO: Ürün kataloğu - AP tercih edilir
// "Anlık tutarsızlık kabul edilebilir, sistem her zaman yanıt vermeli"

public class EventuallyConsistentCatalogService
{
    private readonly IReadModelRepository _readRepo;   // Önbellekli okuma modeli
    private readonly ICommandBus _commandBus;          // CQRS write tarafı
    private readonly ILogger<EventuallyConsistentCatalogService> _logger;

    public EventuallyConsistentCatalogService(
        IReadModelRepository readRepo,
        ICommandBus commandBus,
        ILogger<EventuallyConsistentCatalogService> logger)
    {
        _readRepo = readRepo;
        _commandBus = commandBus;
        _logger = logger;
    }

    // AP kararı: Önbellekten oku, eski veri olabilir ama her zaman yanıt ver
    public async Task<ProductDto?> GetProductAsync(Guid productId, CancellationToken ct)
    {
        try
        {
            // Önce okuma modelinden dene (eventual consistency)
            return await _readRepo.GetProductAsync(productId, ct);
        }
        catch (ReadModelUnavailableException ex)
        {
            // AP kararı: Okuma modeli erişilemez olsa bile hata verme
            _logger.LogWarning(ex, "Okuma modeli erişilemez, yavaş yol deneniyor");
            return await FallbackToSlowPathAsync(productId, ct);
        }
    }

    // Fiyat güncellemesi: Asenkron - immediate consistency gerekmez
    public async Task UpdatePriceAsync(Guid productId, decimal newPrice, CancellationToken ct)
    {
        // Komut kuyruğa alınır, eventual consistency ile tüm node'lara yayılır
        // Ödün: Bazı kullanıcılar kısa süreliğine eski fiyatı görebilir
        await _commandBus.PublishAsync(
            new UpdateProductPriceCommand(productId, newPrice),
            ct);

        _logger.LogInformation("Fiyat güncelleme komutu kuyruğa alındı: {ProductId} -> {Price}",
            productId, newPrice);
    }

    private Task<ProductDto?> FallbackToSlowPathAsync(Guid productId, CancellationToken ct)
        => Task.FromResult<ProductDto?>(null);
}

// Domain nesneleri
public class Account { public Guid Id { get; set; } public decimal Balance { get; set; } }
public class TransferAudit
{
    public Guid FromAccountId { get; set; }
    public Guid ToAccountId { get; set; }
    public decimal Amount { get; set; }
    public DateTime ExecutedAt { get; set; }
}
public class BankContext : DbContext
{
    public BankContext(DbContextOptions<BankContext> options) : base(options) { }
    public DbSet<Account> Accounts => Set<Account>();
    public DbSet<TransferAudit> TransferAudits => Set<TransferAudit>();
}
public record TransferResult(bool IsSuccess, decimal NewBalance)
{
    public static TransferResult Success(decimal balance) => new(true, balance);
}
public record ProductDto(Guid Id, string Name, decimal Price);
public record UpdateProductPriceCommand(Guid ProductId, decimal NewPrice);
public class AccountNotFoundException(Guid id) : Exception($"Hesap bulunamadı: {id}");
public class InsufficientFundsException(Guid id, decimal requested, decimal available)
    : Exception($"Yetersiz bakiye. Hesap: {id}, İstenen: {requested}, Mevcut: {available}");
public class ServiceUnavailableException(string message) : Exception(message);
public class ReadModelUnavailableException : Exception { }
public interface IReadModelRepository { Task<ProductDto?> GetProductAsync(Guid id, CancellationToken ct); }
public interface ICommandBus { Task PublishAsync<T>(T command, CancellationToken ct); }
public interface IDistributedLock { Task<IAsyncDisposable?> AcquireAsync(string key, TimeSpan timeout, CancellationToken ct); }
```

### 6. Soru: Maliyet-fayda analizi (cost-benefit analysis) mimari kararlarda nasıl uygulanır? Sayısal bir örnek veriniz.

**Cevap:**
Maliyet-fayda analizi, bir mimari kararın finansal ve operasyonel değerini objektif biçimde değerlendirmek için kullanılır. Yazılım mimarisinde maliyetler; geliştirme süresi, öğrenme eğrisi, altyapı giderleri ve operasyonel yük şeklinde ortaya çıkar. Faydalar ise; performans iyileştirmesi, hata azalması, geliştirici verimliliği ve iş değeri olarak ölçülür.

**Örnek Kod:**
```csharp
// Mimari karar için maliyet-fayda analizi modeli
// Senaryo: Monolith -> Mikroservis geçişinin ROI hesabı

public class ArchitectureCostBenefitAnalysis
{
    public ArchitectureCostBenefitResult Analyze(
        ArchitectureMigrationScenario scenario)
    {
        var costs = CalculateTotalCosts(scenario);
        var benefits = CalculateTotalBenefits(scenario);
        var roi = CalculateRoi(costs, benefits, scenario.TimeHorizonMonths);

        return new ArchitectureCostBenefitResult(
            TotalCost: costs,
            TotalBenefit: benefits,
            NetPresentValue: benefits.Total - costs.Total,
            Roi: roi,
            PaybackPeriodMonths: CalculatePaybackPeriod(costs, benefits),
            Recommendation: roi > 0.20m
                ? "Geçiş önerilir"
                : "Mevcut mimariyi koru veya kademeli modernizasyonu değerlendir"
        );
    }

    private CostBreakdown CalculateTotalCosts(ArchitectureMigrationScenario s)
    {
        // Doğrudan maliyetler
        var developmentCost = s.DeveloperCount * s.AverageDailyCost * s.EstimatedDevelopmentDays;
        var trainingCost = s.DeveloperCount * s.TrainingDaysPerDeveloper * s.AverageDailyCost;
        var infrastructureCost = s.MonthlyInfrastructureCostIncrease * s.TimeHorizonMonths;

        // Dolaylı maliyetler
        var productivityLossDuringMigration = developmentCost * 0.25m; // %25 verimlilik kaybı
        var riskContingency = developmentCost * 0.15m; // %15 risk tamponu

        return new CostBreakdown(
            Development: developmentCost,
            Training: trainingCost,
            Infrastructure: infrastructureCost,
            ProductivityLoss: productivityLossDuringMigration,
            RiskContingency: riskContingency
        );
    }

    private BenefitBreakdown CalculateTotalBenefits(ArchitectureMigrationScenario s)
    {
        // Ölçülebilir faydalar (aylık tasarruf × süre)
        var deploymentSpeedupBenefit =
            s.CurrentDeploymentHoursPerRelease
            * s.MicroserviceDeploymentSpeedupFactor
            * s.AverageDailyCost / 8 // saatlik maliyet
            * s.ReleasesPerMonth
            * s.TimeHorizonMonths;

        var incidentReductionBenefit =
            s.CurrentMonthlyIncidentHours
            * s.ExpectedIncidentReductionPercent
            * s.AverageDailyCost / 8
            * s.TimeHorizonMonths;

        var scalingCostReduction =
            s.CurrentMonthlyScalingCost
            * s.ExpectedScalingCostReductionPercent
            * s.TimeHorizonMonths;

        // Zor ölçülür ama tahmin edilebilir faydalar
        var developerProductivityGain =
            s.DeveloperCount
            * s.AverageDailyCost
            * s.ExpectedProductivityGainDaysPerMonth
            * s.TimeHorizonMonths;

        return new BenefitBreakdown(
            DeploymentSpeedup: deploymentSpeedupBenefit,
            IncidentReduction: incidentReductionBenefit,
            ScalingCostSavings: scalingCostReduction,
            DeveloperProductivity: developerProductivityGain
        );
    }

    private decimal CalculateRoi(CostBreakdown costs, BenefitBreakdown benefits, int months)
    {
        var totalCost = costs.Total;
        var totalBenefit = benefits.Total;
        return totalCost > 0
            ? (totalBenefit - totalCost) / totalCost
            : 0;
    }

    private int CalculatePaybackPeriod(CostBreakdown costs, BenefitBreakdown benefits)
    {
        var monthlyBenefit = benefits.Total / 24; // 24 aylık ufuk varsayımı
        return monthlyBenefit > 0
            ? (int)Math.Ceiling(costs.Total / monthlyBenefit)
            : int.MaxValue;
    }
}

public record ArchitectureMigrationScenario(
    int DeveloperCount,
    decimal AverageDailyCost,           // TL cinsinden
    int EstimatedDevelopmentDays,
    int TrainingDaysPerDeveloper,
    decimal MonthlyInfrastructureCostIncrease,
    int TimeHorizonMonths,
    decimal CurrentDeploymentHoursPerRelease,
    decimal MicroserviceDeploymentSpeedupFactor, // 0.5 = %50 daha hızlı
    int ReleasesPerMonth,
    decimal CurrentMonthlyIncidentHours,
    decimal ExpectedIncidentReductionPercent,    // 0.3 = %30 azalma
    decimal CurrentMonthlyScalingCost,
    decimal ExpectedScalingCostReductionPercent,
    decimal ExpectedProductivityGainDaysPerMonth
);

public record CostBreakdown(
    decimal Development,
    decimal Training,
    decimal Infrastructure,
    decimal ProductivityLoss,
    decimal RiskContingency)
{
    public decimal Total => Development + Training + Infrastructure + ProductivityLoss + RiskContingency;
}

public record BenefitBreakdown(
    decimal DeploymentSpeedup,
    decimal IncidentReduction,
    decimal ScalingCostSavings,
    decimal DeveloperProductivity)
{
    public decimal Total => DeploymentSpeedup + IncidentReduction + ScalingCostSavings + DeveloperProductivity;
}

public record ArchitectureCostBenefitResult(
    CostBreakdown TotalCost,
    BenefitBreakdown TotalBenefit,
    decimal NetPresentValue,
    decimal Roi,
    int PaybackPeriodMonths,
    string Recommendation);

// Gerçek bir senaryoda kullanım
public static class CostBenefitExample
{
    public static ArchitectureCostBenefitResult RunMicroserviceMigrationAnalysis()
    {
        var analyzer = new ArchitectureCostBenefitAnalysis();

        var scenario = new ArchitectureMigrationScenario(
            DeveloperCount: 8,
            AverageDailyCost: 3000,                       // 3.000 TL/gün/geliştirici
            EstimatedDevelopmentDays: 180,                // 9 ay
            TrainingDaysPerDeveloper: 10,
            MonthlyInfrastructureCostIncrease: 15_000,   // TL/ay
            TimeHorizonMonths: 24,                        // 2 yıllık ROI
            CurrentDeploymentHoursPerRelease: 4,
            MicroserviceDeploymentSpeedupFactor: 0.75m,  // %75 daha hızlı
            ReleasesPerMonth: 8,
            CurrentMonthlyIncidentHours: 40,
            ExpectedIncidentReductionPercent: 0.35m,      // %35 azalma
            CurrentMonthlyScalingCost: 20_000,
            ExpectedScalingCostReductionPercent: 0.20m,   // %20 tasarruf
            ExpectedProductivityGainDaysPerMonth: 2       // gün/geliştirici/ay
        );

        return analyzer.Analyze(scenario);
        // Beklenen sonuç: Toplam maliyet ~5.6M TL, Toplam fayda ~7.2M TL
        // ROI ~%28, Geri ödeme süresi ~18 ay -> Geçiş önerilir
    }
}
```

## Best Practices

### 1. ATAM Uygulaması
- Değerlendirme sürecine hem teknik hem de iş paydaşlarını dahil edin
- Kalite niteliği senaryolarını ölçülebilir biçimde yazın ("hızlı" değil, "500ms altında")
- Risk noktalarını ciddiye alın ve her biri için hafifletme stratejisi belirleyin
- Ödün noktalarını ADR olarak belgeleyin; yalnızca seçilen çözümü değil, neden seçildiğini yazın

### 2. Karar Matrisi Kullanımı
- Kriterleri ve ağırlıkları karardan önce belirleyin; sonuçtan etkilenmeyin
- Ağırlıklandırmayı paydaşlarla birlikte yapın; farklı grupların öncelikleri farklıdır
- Puanlama için somut kanıtlara başvurun (benchmark, referans müşteri, pilot uygulama)
- Matrisi kararın gerekçesiyle birlikte ADR'ye ekleyin

### 3. Maliyet-Fayda Analizi
- Ölçülemeyen faydaları (geliştirici mutluluğu, işe alım kolaylığı) ayrıca listeleyin
- Gizli maliyetleri (öğrenme eğrisi, araç değişimi, ekip direnci) hesaba katın
- Birden fazla senaryo analiz edin (iyimser, gerçekçi, kötümser)
- ROI hesaplamalarını 6 ayda bir gerçek verilerle güncelleyin

### 4. Dokümantasyon
- Analiz sonuçlarını ADR'de "Sonuçlar" bölümünde ödün tablosu olarak gösterin
- Reddedilen alternatiflerin neden reddedildiğini belgeleyin
- Gelecekteki karar gözden geçirme için tetikleyici koşulları belirtin (ör. "Kullanıcı sayısı 1M'u geçerse bu karar yeniden değerlendirilmeli")

## Kaynaklar

- [ATAM - SEI Carnegie Mellon University](https://resources.sei.cmu.edu/library/asset-view.cfm?assetid=513908)
- [Documenting Software Architectures - Bass, Clements, Kazman](https://www.informit.com/store/documenting-software-architectures-views-and-beyond-9780321552686)
- [CAP Theorem - Eric Brewer](https://people.eecs.berkeley.edu/~brewer/cs262b-2004/PODC-keynote.pdf)
- [Designing Data-Intensive Applications - Martin Kleppmann](https://dataintensive.net/)
- [Polly .NET Resilience Library](https://www.thepollyproject.org/)
- [Microsoft Architecture Guides](https://learn.microsoft.com/en-us/azure/architecture/)
- [Building Evolutionary Architectures - Ford, Parsons, Kua](https://www.buildingevolutionaryarchitectures.com/)
