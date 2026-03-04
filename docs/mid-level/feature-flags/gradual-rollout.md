# Gradual Rollout & A/B Testing

## Genel Bakış

Gradual rollout (kademeli yayılım) ve A/B testing, modern yazılım ekiplerinin yeni özellikleri güvenle yayınlayabilmesi ve veri odaklı ürün kararları alabilmesi için kritik pratiklerdir. Bu bölümde; yüzde tabanlı kademeli açılım, kullanıcı segmentasyonu, canary release deseni, A/B test tasarımı, istatistiksel analiz ve .NET'te bu sistemlerin nasıl implement edildiğini kapsamlı örneklerle inceleyeceğiz.

## Gradual Rollout Nedir?

Gradual rollout; yeni bir özelliği tüm kullanıcı tabanına bir anda açmak yerine, giderek artan yüzde dilimleriyle kademeli olarak sunma stratejisidir. Bu yaklaşım risk azaltma, erken geri bildirim toplama ve sistemin gerçek yük altında izlenmesine olanak tanır.

```
Kademeli Açılım Takvimi Örneği:
─────────────────────────────────────────────────────
Gün 1:   %1   → İç ekip ve test kullanıcıları
Gün 3:   %5   → Erken erişim programı üyeleri
Gün 7:   %20  → Düşük trafikli bölgeler
Gün 14:  %50  → Genel kullanıcı tabanının yarısı
Gün 21: %100  → Tüm kullanıcılar
─────────────────────────────────────────────────────
Her aşamada: Hata oranı, P95 gecikme, kullanıcı şikayetleri izlenir
```

## Yüzde Tabanlı Rollout Implementasyonu

### Temel Hash Tabanlı Kullanıcı Dağılımı

```csharp
public class PercentageBasedRolloutService
{
    private readonly ILogger<PercentageBasedRolloutService> _logger;

    public PercentageBasedRolloutService(ILogger<PercentageBasedRolloutService> logger)
    {
        _logger = logger;
    }

    /// <summary>
    /// Kullanıcının belirtilen yüzdelik dilime girip girmediğini belirler.
    /// Hash tabanlı yaklaşım, aynı kullanıcının her zaman aynı grupta kalmasını sağlar.
    /// </summary>
    public bool IsUserInRollout(string userId, string featureName, double percentage)
    {
        if (percentage <= 0) return false;
        if (percentage >= 100) return true;

        // Kullanıcı ID ve özellik adını birleştirerek tutarlı hash üret
        var key = $"{featureName}:{userId}";
        var hash = ComputeStableHash(key);

        // Hash'i 0-100 aralığına normalize et
        var normalizedValue = Math.Abs(hash) % 100;

        var result = normalizedValue < percentage;

        _logger.LogDebug(
            "Rollout kontrolü - Kullanıcı: {UserId}, Özellik: {Feature}, " +
            "Yüzde: {Percentage}%, Hash: {Hash}, Sonuç: {Result}",
            userId, featureName, percentage, normalizedValue, result);

        return result;
    }

    private static int ComputeStableHash(string input)
    {
        // FNV-1a hash algoritması - dağıtım kalitesi iyi, hızlı
        unchecked
        {
            const int fnvPrime = 16777619;
            int hash = unchecked((int)2166136261);

            foreach (char c in input)
            {
                hash ^= c;
                hash *= fnvPrime;
            }

            return hash;
        }
    }
}
```

### Aşamalı Rollout Orkestratörü

```csharp
public class RolloutOrchestrator
{
    private readonly IRolloutConfigRepository _configRepository;
    private readonly PercentageBasedRolloutService _rolloutService;
    private readonly IRolloutMetricsCollector _metricsCollector;
    private readonly ILogger<RolloutOrchestrator> _logger;

    public RolloutOrchestrator(
        IRolloutConfigRepository configRepository,
        PercentageBasedRolloutService rolloutService,
        IRolloutMetricsCollector metricsCollector,
        ILogger<RolloutOrchestrator> logger)
    {
        _configRepository = configRepository;
        _rolloutService = rolloutService;
        _metricsCollector = metricsCollector;
        _logger = logger;
    }

    public async Task<RolloutDecision> EvaluateAsync(string featureName, string userId)
    {
        var config = await _configRepository.GetAsync(featureName);

        if (config == null || !config.IsEnabled)
        {
            return RolloutDecision.Disabled(featureName);
        }

        // Kill switch kontrolü
        if (config.IsKillSwitchActive)
        {
            _logger.LogWarning("Kill switch aktif: {Feature}", featureName);
            return RolloutDecision.KillSwitched(featureName);
        }

        // Whitelist kontrolü (her zaman dahil edilecek kullanıcılar)
        if (config.WhitelistUserIds.Contains(userId))
        {
            _metricsCollector.RecordEvaluation(featureName, userId, "whitelist", true);
            return RolloutDecision.Enabled(featureName, "whitelist");
        }

        // Blacklist kontrolü (her zaman hariç tutulacak kullanıcılar)
        if (config.BlacklistUserIds.Contains(userId))
        {
            _metricsCollector.RecordEvaluation(featureName, userId, "blacklist", false);
            return RolloutDecision.Disabled(featureName, "blacklist");
        }

        // Grup tabanlı kontrol
        var userGroups = await GetUserGroupsAsync(userId);
        foreach (var groupRule in config.GroupRules)
        {
            if (userGroups.Contains(groupRule.GroupName))
            {
                var groupResult = _rolloutService.IsUserInRollout(
                    userId, featureName, groupRule.Percentage);

                _metricsCollector.RecordEvaluation(
                    featureName, userId, $"group:{groupRule.GroupName}", groupResult);

                return groupResult
                    ? RolloutDecision.Enabled(featureName, $"group:{groupRule.GroupName}")
                    : RolloutDecision.Disabled(featureName, $"group:{groupRule.GroupName}");
            }
        }

        // Genel yüzde kontrolü
        var generalResult = _rolloutService.IsUserInRollout(
            userId, featureName, config.DefaultPercentage);

        _metricsCollector.RecordEvaluation(featureName, userId, "default", generalResult);

        return generalResult
            ? RolloutDecision.Enabled(featureName, "default")
            : RolloutDecision.Disabled(featureName, "default");
    }

    private Task<HashSet<string>> GetUserGroupsAsync(string userId)
    {
        // Gerçek implementasyonda veritabanından veya önbellekten okunur
        return Task.FromResult(new HashSet<string>());
    }
}

public record RolloutDecision(
    string FeatureName,
    bool IsEnabled,
    string Reason)
{
    public static RolloutDecision Enabled(string feature, string reason = "enabled")
        => new(feature, true, reason);

    public static RolloutDecision Disabled(string feature, string reason = "disabled")
        => new(feature, false, reason);

    public static RolloutDecision KillSwitched(string feature)
        => new(feature, false, "kill_switch");
}

public class RolloutConfig
{
    public string FeatureName { get; set; } = string.Empty;
    public bool IsEnabled { get; set; }
    public bool IsKillSwitchActive { get; set; }
    public double DefaultPercentage { get; set; }
    public List<string> WhitelistUserIds { get; set; } = new();
    public List<string> BlacklistUserIds { get; set; } = new();
    public List<GroupRolloutRule> GroupRules { get; set; } = new();
}

public class GroupRolloutRule
{
    public string GroupName { get; set; } = string.Empty;
    public double Percentage { get; set; }
}
```

## Canary Release Deseni

```csharp
public class CanaryReleaseService
{
    private readonly IFeatureManager _featureManager;
    private readonly IMetricsClient _metricsClient;
    private readonly IAlertingService _alertingService;
    private readonly ILogger<CanaryReleaseService> _logger;

    public CanaryReleaseService(
        IFeatureManager featureManager,
        IMetricsClient metricsClient,
        IAlertingService alertingService,
        ILogger<CanaryReleaseService> logger)
    {
        _featureManager = featureManager;
        _metricsClient = metricsClient;
        _alertingService = alertingService;
        _logger = logger;
    }

    public async Task<ServiceResponse<T>> ExecuteWithCanaryAsync<T>(
        string featureName,
        Func<Task<T>> stableImplementation,
        Func<Task<T>> canaryImplementation,
        string userId)
    {
        var stopwatch = Stopwatch.StartNew();
        bool isCanary = await _featureManager.IsEnabledAsync(featureName);

        string version = isCanary ? "canary" : "stable";

        try
        {
            T result = isCanary
                ? await canaryImplementation()
                : await stableImplementation();

            stopwatch.Stop();

            // Başarı metriği kaydet
            _metricsClient.RecordHistogram(
                $"feature.{featureName}.duration_ms",
                stopwatch.ElapsedMilliseconds,
                new Dictionary<string, string>
                {
                    ["version"] = version,
                    ["status"] = "success"
                });

            return ServiceResponse<T>.Success(result, version);
        }
        catch (Exception ex)
        {
            stopwatch.Stop();

            _logger.LogError(ex,
                "Canary implementasyonunda hata - Özellik: {Feature}, Versiyon: {Version}",
                featureName, version);

            // Hata metriği kaydet
            _metricsClient.IncrementCounter(
                $"feature.{featureName}.errors",
                new Dictionary<string, string>
                {
                    ["version"] = version,
                    ["error_type"] = ex.GetType().Name
                });

            // Canary hata oranı çok yüksekse uyarı gönder
            if (isCanary)
            {
                await CheckCanaryHealthAsync(featureName);
            }

            throw;
        }
    }

    private async Task CheckCanaryHealthAsync(string featureName)
    {
        var errorRate = await _metricsClient.GetErrorRateAsync(
            $"feature.{featureName}.errors",
            timeWindow: TimeSpan.FromMinutes(5));

        if (errorRate > 0.05) // %5 hata eşiği
        {
            _logger.LogCritical(
                "Canary hata oranı eşiği aşıldı! Özellik: {Feature}, Hata oranı: {Rate:P2}",
                featureName, errorRate);

            await _alertingService.SendAlertAsync(new Alert
            {
                Severity = AlertSeverity.Critical,
                Title = $"Canary Hata Eşiği Aşıldı: {featureName}",
                Message = $"Son 5 dakikada hata oranı {errorRate:P2} oldu. " +
                          $"Lütfen özelliği devre dışı bırakmayı değerlendirin.",
                Tags = new Dictionary<string, string>
                {
                    ["feature"] = featureName,
                    ["error_rate"] = errorRate.ToString("F4")
                }
            });
        }
    }
}

public record ServiceResponse<T>(T? Data, bool IsSuccess, string Version, string? Error)
{
    public static ServiceResponse<T> Success(T data, string version)
        => new(data, true, version, null);

    public static ServiceResponse<T> Failure(string error, string version)
        => new(default, false, version, error);
}
```

## A/B Testing Implementasyonu

### A/B Test Altyapısı

```csharp
// Deney (experiment) tanımı
public class Experiment
{
    public string Id { get; set; } = string.Empty;
    public string Name { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public ExperimentStatus Status { get; set; }
    public List<ExperimentVariant> Variants { get; set; } = new();
    public ExperimentMetrics SuccessMetrics { get; set; } = new();
    public DateTime StartDate { get; set; }
    public DateTime? EndDate { get; set; }
    public double MinimumDetectableEffect { get; set; } = 0.05; // %5 etki
    public double StatisticalSignificanceThreshold { get; set; } = 0.95; // %95 güven
}

public class ExperimentVariant
{
    public string Name { get; set; } = string.Empty; // "control", "treatment_a"
    public string DisplayName { get; set; } = string.Empty;
    public double TrafficAllocation { get; set; } // 0.0 - 1.0
    public Dictionary<string, object> Parameters { get; set; } = new();
}

public enum ExperimentStatus { Draft, Running, Paused, Concluded }

public class ExperimentMetrics
{
    public string PrimaryMetric { get; set; } = string.Empty; // "conversion_rate"
    public List<string> GuardrailMetrics { get; set; } = new(); // "error_rate", "latency"
}

// A/B test servisi
public class AbTestingService
{
    private readonly IExperimentRepository _experimentRepository;
    private readonly IExperimentAssignmentRepository _assignmentRepository;
    private readonly IExperimentEventRepository _eventRepository;
    private readonly ILogger<AbTestingService> _logger;

    public AbTestingService(
        IExperimentRepository experimentRepository,
        IExperimentAssignmentRepository assignmentRepository,
        IExperimentEventRepository eventRepository,
        ILogger<AbTestingService> logger)
    {
        _experimentRepository = experimentRepository;
        _assignmentRepository = assignmentRepository;
        _eventRepository = eventRepository;
        _logger = logger;
    }

    /// <summary>
    /// Kullanıcıyı bir deney varyantına atar ve atamayı kaydeder.
    /// Aynı kullanıcı için her zaman aynı varyant döner (sticky assignment).
    /// </summary>
    public async Task<ExperimentAssignment> AssignVariantAsync(
        string experimentId,
        string userId)
    {
        // Mevcut atama var mı kontrol et (sticky assignment)
        var existingAssignment = await _assignmentRepository
            .GetAsync(experimentId, userId);

        if (existingAssignment != null)
        {
            return existingAssignment;
        }

        var experiment = await _experimentRepository.GetAsync(experimentId);

        if (experiment == null || experiment.Status != ExperimentStatus.Running)
        {
            return ExperimentAssignment.Control(experimentId, userId);
        }

        // Deterministik varyant seçimi
        var variant = SelectVariant(experiment, userId);
        var assignment = new ExperimentAssignment
        {
            ExperimentId = experimentId,
            UserId = userId,
            VariantName = variant.Name,
            AssignedAt = DateTimeOffset.UtcNow,
            Parameters = variant.Parameters
        };

        // Atamayı kaydet
        await _assignmentRepository.SaveAsync(assignment);

        _logger.LogInformation(
            "Deney ataması - Deney: {Experiment}, Kullanıcı: {User}, Varyant: {Variant}",
            experimentId, userId, variant.Name);

        return assignment;
    }

    private ExperimentVariant SelectVariant(Experiment experiment, string userId)
    {
        // Hash tabanlı deterministik dağılım
        var key = $"{experiment.Id}:{userId}";
        var hash = Math.Abs(key.GetHashCode());
        var position = (hash % 1000) / 1000.0; // 0.0 - 1.0 arası değer

        double cumulativeAllocation = 0;
        foreach (var variant in experiment.Variants)
        {
            cumulativeAllocation += variant.TrafficAllocation;
            if (position < cumulativeAllocation)
            {
                return variant;
            }
        }

        // Fallback: kontrol grubu
        return experiment.Variants.First(v => v.Name == "control");
    }

    /// <summary>
    /// Dönüşüm veya hedef olayı kaydeder.
    /// </summary>
    public async Task TrackEventAsync(
        string experimentId,
        string userId,
        string eventName,
        double value = 1.0,
        Dictionary<string, object>? metadata = null)
    {
        var assignment = await _assignmentRepository.GetAsync(experimentId, userId);
        if (assignment == null) return;

        var experimentEvent = new ExperimentEvent
        {
            ExperimentId = experimentId,
            UserId = userId,
            VariantName = assignment.VariantName,
            EventName = eventName,
            Value = value,
            OccurredAt = DateTimeOffset.UtcNow,
            Metadata = metadata ?? new Dictionary<string, object>()
        };

        await _eventRepository.SaveAsync(experimentEvent);
    }
}

public class ExperimentAssignment
{
    public string ExperimentId { get; set; } = string.Empty;
    public string UserId { get; set; } = string.Empty;
    public string VariantName { get; set; } = string.Empty;
    public DateTimeOffset AssignedAt { get; set; }
    public Dictionary<string, object> Parameters { get; set; } = new();

    public static ExperimentAssignment Control(string experimentId, string userId)
        => new()
        {
            ExperimentId = experimentId,
            UserId = userId,
            VariantName = "control",
            AssignedAt = DateTimeOffset.UtcNow
        };
}
```

### A/B Test Sonuç Analizi

```csharp
public class AbTestAnalysisService
{
    private readonly IExperimentEventRepository _eventRepository;
    private readonly ILogger<AbTestAnalysisService> _logger;

    public AbTestAnalysisService(
        IExperimentEventRepository eventRepository,
        ILogger<AbTestAnalysisService> logger)
    {
        _eventRepository = eventRepository;
        _logger = logger;
    }

    public async Task<ExperimentResults> AnalyzeAsync(
        string experimentId,
        string metricName,
        DateTimeOffset from,
        DateTimeOffset to)
    {
        var events = await _eventRepository.GetAsync(experimentId, metricName, from, to);

        var variantGroups = events
            .GroupBy(e => e.VariantName)
            .ToDictionary(g => g.Key, g => g.ToList());

        var variantStats = new Dictionary<string, VariantStatistics>();

        foreach (var (variantName, variantEvents) in variantGroups)
        {
            var uniqueUsers = variantEvents.Select(e => e.UserId).Distinct().Count();
            var conversions = variantEvents.Count(e => e.Value > 0);
            var conversionRate = uniqueUsers > 0
                ? (double)conversions / uniqueUsers
                : 0;

            variantStats[variantName] = new VariantStatistics
            {
                VariantName = variantName,
                TotalUsers = uniqueUsers,
                Conversions = conversions,
                ConversionRate = conversionRate,
                AverageValue = variantEvents.Any()
                    ? variantEvents.Average(e => e.Value)
                    : 0
            };
        }

        // İstatistiksel anlamlılık hesabı (control vs treatment)
        var significance = CalculateStatisticalSignificance(variantStats);

        return new ExperimentResults
        {
            ExperimentId = experimentId,
            MetricName = metricName,
            AnalyzedFrom = from,
            AnalyzedTo = to,
            VariantStats = variantStats,
            StatisticalSignificance = significance,
            IsStatisticallySignificant = significance >= 0.95,
            Winner = DetermineWinner(variantStats, significance)
        };
    }

    private double CalculateStatisticalSignificance(
        Dictionary<string, VariantStatistics> stats)
    {
        if (!stats.ContainsKey("control") || stats.Count < 2)
            return 0;

        var control = stats["control"];
        var treatment = stats.Values.First(v => v.VariantName != "control");

        // Z-test for proportions (iki oran testi)
        var p1 = control.ConversionRate;
        var p2 = treatment.ConversionRate;
        var n1 = control.TotalUsers;
        var n2 = treatment.TotalUsers;

        if (n1 == 0 || n2 == 0) return 0;

        var pooledProportion = (p1 * n1 + p2 * n2) / (n1 + n2);
        var standardError = Math.Sqrt(
            pooledProportion * (1 - pooledProportion) * (1.0 / n1 + 1.0 / n2));

        if (standardError == 0) return 0;

        var zScore = Math.Abs((p2 - p1) / standardError);

        // Z-score'u p-value'ya çevir (normal dağılım CDF)
        var pValue = 2 * (1 - NormalCdf(zScore));
        return 1 - pValue; // Güven düzeyi
    }

    private static double NormalCdf(double z)
    {
        // Standart normal dağılım CDF yaklaşımı (Abramowitz & Stegun)
        const double a1 = 0.254829592;
        const double a2 = -0.284496736;
        const double a3 = 1.421413741;
        const double a4 = -1.453152027;
        const double a5 = 1.061405429;
        const double p = 0.3275911;

        var sign = z < 0 ? -1 : 1;
        z = Math.Abs(z) / Math.Sqrt(2);

        var t = 1.0 / (1.0 + p * z);
        var y = 1.0 - (((((a5 * t + a4) * t) + a3) * t + a2) * t + a1) * t
                * Math.Exp(-z * z);

        return 0.5 * (1.0 + sign * y);
    }

    private static string? DetermineWinner(
        Dictionary<string, VariantStatistics> stats,
        double significance)
    {
        if (significance < 0.95) return null; // İstatistiksel anlamlılık yok

        return stats
            .OrderByDescending(kv => kv.Value.ConversionRate)
            .First()
            .Key;
    }
}

public class ExperimentResults
{
    public string ExperimentId { get; set; } = string.Empty;
    public string MetricName { get; set; } = string.Empty;
    public DateTimeOffset AnalyzedFrom { get; set; }
    public DateTimeOffset AnalyzedTo { get; set; }
    public Dictionary<string, VariantStatistics> VariantStats { get; set; } = new();
    public double StatisticalSignificance { get; set; }
    public bool IsStatisticallySignificant { get; set; }
    public string? Winner { get; set; }
}

public class VariantStatistics
{
    public string VariantName { get; set; } = string.Empty;
    public int TotalUsers { get; set; }
    public int Conversions { get; set; }
    public double ConversionRate { get; set; }
    public double AverageValue { get; set; }
    public double LiftOverControl { get; set; }
}
```

## Azure App Configuration ile Gradual Rollout

```csharp
// Program.cs - Azure App Configuration entegrasyonu
var builder = WebApplication.CreateBuilder(args);

builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(builder.Configuration["AppConfig:ConnectionString"])
           .UseFeatureFlags(flagOptions =>
           {
               // Her 30 saniyede bir flag değerlerini yenile
               flagOptions.CacheExpirationInterval = TimeSpan.FromSeconds(30);

               // Belirli label'ları kullan (ortam bazlı)
               flagOptions.Label = builder.Environment.EnvironmentName;
           });
});

builder.Services.AddAzureAppConfiguration();
builder.Services.AddFeatureManagement();

var app = builder.Build();
app.UseAzureAppConfiguration(); // Middleware ile dinamik yenileme

// Program.cs içindeki middleware sırası önemlidir
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.Run();
```

## Feature Flag ile Çok Değişkenli Test

```csharp
// IVariantFeatureManager ile A/B/C/D testi
public class MultiVariantCheckoutService
{
    private readonly IVariantFeatureManager _variantFeatureManager;
    private readonly AbTestingService _abTestingService;
    private readonly ILogger<MultiVariantCheckoutService> _logger;

    public MultiVariantCheckoutService(
        IVariantFeatureManager variantFeatureManager,
        AbTestingService abTestingService,
        ILogger<MultiVariantCheckoutService> logger)
    {
        _variantFeatureManager = variantFeatureManager;
        _abTestingService = abTestingService;
        _logger = logger;
    }

    public async Task<CheckoutPageConfig> GetCheckoutConfigAsync(string userId)
    {
        // Varyant ataması al
        var variant = await _variantFeatureManager
            .GetVariantAsync("CheckoutFlowExperiment", CancellationToken.None);

        var config = variant?.Name switch
        {
            "one_page" => new CheckoutPageConfig
            {
                Layout = "single_page",
                ShowProgressBar = false,
                EnableExpressShipping = true
            },
            "stepped" => new CheckoutPageConfig
            {
                Layout = "multi_step",
                ShowProgressBar = true,
                EnableExpressShipping = false
            },
            "minimal" => new CheckoutPageConfig
            {
                Layout = "minimal",
                ShowProgressBar = false,
                EnableExpressShipping = false,
                HideUpsells = true
            },
            _ => new CheckoutPageConfig
            {
                Layout = "default",
                ShowProgressBar = true,
                EnableExpressShipping = false
            }
        };

        _logger.LogDebug(
            "Kullanıcı {UserId} için checkout varyantı: {Variant}",
            userId, variant?.Name ?? "control");

        return config;
    }

    public async Task TrackCheckoutStartedAsync(string userId)
    {
        await _abTestingService.TrackEventAsync(
            "CheckoutFlowExperiment",
            userId,
            "checkout_started");
    }

    public async Task TrackOrderCompletedAsync(string userId, decimal orderValue)
    {
        await _abTestingService.TrackEventAsync(
            "CheckoutFlowExperiment",
            userId,
            "order_completed",
            value: (double)orderValue);
    }
}

public class CheckoutPageConfig
{
    public string Layout { get; set; } = "default";
    public bool ShowProgressBar { get; set; } = true;
    public bool EnableExpressShipping { get; set; } = false;
    public bool HideUpsells { get; set; } = false;
}
```

## Metrik İzleme ve Otomatik Rollback

```csharp
public class AutoRollbackService : BackgroundService
{
    private readonly IRolloutConfigRepository _configRepository;
    private readonly IMetricsClient _metricsClient;
    private readonly IAlertingService _alertingService;
    private readonly ILogger<AutoRollbackService> _logger;

    private static readonly RollbackRule[] Rules =
    {
        new("error_rate", threshold: 0.05, window: TimeSpan.FromMinutes(5)),
        new("p95_latency_ms", threshold: 2000, window: TimeSpan.FromMinutes(10)),
        new("payment_failure_rate", threshold: 0.02, window: TimeSpan.FromMinutes(3))
    };

    public AutoRollbackService(
        IRolloutConfigRepository configRepository,
        IMetricsClient metricsClient,
        IAlertingService alertingService,
        ILogger<AutoRollbackService> logger)
    {
        _configRepository = configRepository;
        _metricsClient = metricsClient;
        _alertingService = alertingService;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await EvaluateRollbackConditionsAsync();
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Rollback değerlendirmesinde hata oluştu");
            }

            await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
        }
    }

    private async Task EvaluateRollbackConditionsAsync()
    {
        var activeRollouts = await _configRepository.GetActiveRolloutsAsync();

        foreach (var rollout in activeRollouts)
        {
            foreach (var rule in Rules)
            {
                var metricValue = await _metricsClient.GetMetricValueAsync(
                    $"feature.{rollout.FeatureName}.{rule.MetricName}",
                    rule.Window);

                if (metricValue > rule.Threshold)
                {
                    _logger.LogCritical(
                        "Rollback tetiklendi! Özellik: {Feature}, Metrik: {Metric}, " +
                        "Değer: {Value:F4}, Eşik: {Threshold:F4}",
                        rollout.FeatureName, rule.MetricName, metricValue, rule.Threshold);

                    // Kill switch'i aktif et
                    await _configRepository.ActivateKillSwitchAsync(rollout.FeatureName);

                    await _alertingService.SendAlertAsync(new Alert
                    {
                        Severity = AlertSeverity.Critical,
                        Title = $"Otomatik Rollback: {rollout.FeatureName}",
                        Message = $"'{rule.MetricName}' metriği eşiği aştı " +
                                  $"({metricValue:P2} > {rule.Threshold:P2}). " +
                                  $"Özellik otomatik olarak devre dışı bırakıldı.",
                        Tags = new Dictionary<string, string>
                        {
                            ["feature"] = rollout.FeatureName,
                            ["metric"] = rule.MetricName,
                            ["value"] = metricValue.ToString("F4")
                        }
                    });

                    break; // Bu özellik için diğer kuralları kontrol etmeye gerek yok
                }
            }
        }
    }
}

public record RollbackRule(string MetricName, double Threshold, TimeSpan Window);
```

## Mülakat Soruları

### 1. Soru

**Yüzde tabanlı kademeli rollout'ta neden rastgele sayı yerine hash fonksiyonu kullanılır?**

**Cevap:** Rastgele sayı kullanıldığında, aynı kullanıcı her sayfa yüklemesinde farklı bir gruba düşer ve bu tutarsız bir deneyime yol açar; bir seferinde yeni özelliği görür, bir seferinde görmez. Hash tabanlı yaklaşım ise kullanıcı ID'si ve özellik adından deterministik, tutarlı bir değer üretir. Böylece aynı kullanıcı her zaman aynı gruba atanır (sticky assignment). Bu yaklaşım aynı zamanda veritabanında kullanıcı atamalarını saklamadan çalışır ve ölçeklenebilirdir. Özellik adının hash'e dahil edilmesi ise farklı deneyler için aynı kullanıcının her seferinde aynı gruba düşmemesini (örneğin her deneyimde daima kontrol grubunda olmamasını) sağlar.

**Örnek Kod:**

```csharp
public bool IsUserInRollout(string userId, string featureName, double percentage)
{
    // Özellik adını dahil et: farklı deney için farklı dağılım
    var key = $"{featureName}:{userId}";

    // FNV hash: hızlı, iyi dağılım, aynı input -> aynı output
    var hash = ComputeFnvHash(key);
    var bucket = Math.Abs(hash) % 100;

    // Deterministik: aynı kullanıcı her zaman aynı bucket'ta
    return bucket < percentage;
}
```

### 2. Soru

**A/B testinde istatistiksel anlamlılık neden önemlidir ve nasıl hesaplanır?**

**Cevap:** Küçük örneklem boyutlarıyla gözlemlenen farklılıklar tesadüfi olabilir. İstatistiksel anlamlılık, gözlemlenen etkinin rastgele şansa bağlı olma olasılığının ne kadar düşük olduğunu gösterir. Genellikle %95 güven düzeyi (p < 0.05) kullanılır; bu, gözlemlenen farkın rastgele olma ihtimalinin %5'ten az olduğunu gösterir. İki oran (dönüşüm oranı) karşılaştırması için z-testi uygulanır: kontrol ve treatment gruplarının oranları ve örneklem büyüklükleri kullanılarak z-skoru hesaplanır, bu skor p-değerine çevrilir. Yeterli örneklem büyüklüğüne ulaşmadan testi erken sonlandırmak (peeking problem) yanlış pozitif sonuçlara yol açar.

**Örnek Kod:**

```csharp
// İki oran testi için minimum örneklem büyüklüğü hesabı
public static int CalculateRequiredSampleSize(
    double baselineConversionRate,
    double minimumDetectableEffect,
    double alpha = 0.05,    // %5 anlamlılık düzeyi
    double power = 0.80)    // %80 istatistiksel güç
{
    // z-değerleri: alpha/2 = 1.96, power = 0.84
    double zAlpha = 1.96; // two-tailed
    double zBeta = 0.84;

    double p1 = baselineConversionRate;
    double p2 = baselineConversionRate * (1 + minimumDetectableEffect);
    double pooled = (p1 + p2) / 2;

    double n = Math.Pow(zAlpha * Math.Sqrt(2 * pooled * (1 - pooled))
             + zBeta * Math.Sqrt(p1 * (1 - p1) + p2 * (1 - p2)), 2)
             / Math.Pow(p2 - p1, 2);

    return (int)Math.Ceiling(n);
}
// Örnek: %5 dönüşüm oranı, %20 rölatif iyileşme -> her grup için ~3842 kullanıcı
```

### 3. Soru

**Canary release sırasında hangi metrikleri izlemeli ve otomatik rollback için eşikler nasıl belirlenir?**

**Cevap:** İzlenmesi gereken temel metrik kategorileri şunlardır: Hata metrikleri (HTTP 5xx oranı, exception sayısı, uygulama hata oranı), performans metrikleri (P95/P99 yanıt süresi, veritabanı sorgu süresi), iş metrikleri (dönüşüm oranı, ödeme başarı oranı, sepet terk oranı) ve sistem metrikleri (CPU, bellek, bağlantı havuzu kullanımı). Eşikler belirlenirken son 7-30 günlük baseline (temel çizgi) hesaplanır ve anlamlı sapma üst sınırı belirlenir. Rollback kararı için tek bir ağır metrik (kritik hata artışı) veya birden fazla metriğin birlikte kötüleşmesi kural olarak tanımlanır. Erken aşamalarda daha hassas eşikler, kullanıcı tabanı büyüdükçe daha gevşek eşikler kullanılabilir.

**Örnek Kod:**

```csharp
public class RollbackThresholds
{
    // Canary kullanıcı sayısına göre adaptif eşikler
    public static double GetErrorRateThreshold(int canaryUserCount)
    {
        return canaryUserCount switch
        {
            < 100  => 0.10,  // %10 - çok küçük örneklem, geniş tolerans
            < 1000 => 0.05,  // %5  - küçük örneklem
            < 10000 => 0.02, // %2  - orta örneklem
            _      => 0.01   // %1  - büyük örneklem, hassas kontrol
        };
    }
}
```

### 4. Soru

**A/B testinde "peeking problem" nedir ve nasıl önlenir?**

**Cevap:** Peeking problem, test devam ederken istatistiksel anlamlılığa ulaşılır ulaşılmaz testi sonlandırma eğilimidir. Bu yanlış pozitif oranını dramatik biçimde artırır; çünkü her kontrol noktasında şans eseri anlamlı görünen bir sonuç elde etme olasılığı birikmektedir. Örneğin, günlük kontrol edilen 30 günlük bir testte gerçekte hiçbir etki olmasa bile yanlış pozitif bulma olasılığı %5'ten %30'a çıkabilir. Çözümler şunlardır: önceden belirlenen örneklem büyüklüğüne ulaşana kadar testi sürdürmek, Bonferroni düzeltmesi ile çok kontrollü testlerde alpha seviyesini ayarlamak ve Sequential Testing veya Bayesian yaklaşımlar kullanmak. Pratikte testin başlamadan önce minimum süre ve örneklem büyüklüğü planlanmalı ve bu plana sadık kalınmalıdır.

**Örnek Kod:**

```csharp
public class ExperimentGuard
{
    private readonly IExperimentRepository _repo;

    public ExperimentGuard(IExperimentRepository repo) => _repo = repo;

    public async Task<ExperimentReadinessResult> CheckReadinessAsync(string experimentId)
    {
        var experiment = await _repo.GetAsync(experimentId);
        var stats = await _repo.GetCurrentStatsAsync(experimentId);

        var requiredSamplePerVariant = CalculateRequiredSampleSize(
            experiment.BaselineConversionRate,
            experiment.MinimumDetectableEffect);

        var minDaysElapsed = (DateTimeOffset.UtcNow - experiment.StartDate).Days >= 7;
        var sufficientSample = stats.Values.All(v => v.TotalUsers >= requiredSamplePerVariant);

        return new ExperimentReadinessResult
        {
            IsReadyForConclusion = minDaysElapsed && sufficientSample,
            RequiredUsersPerVariant = requiredSamplePerVariant,
            DaysElapsed = (int)(DateTimeOffset.UtcNow - experiment.StartDate).TotalDays,
            Message = !minDaysElapsed
                ? "En az 7 gün beklenmeli (haftalık mevsimsellik etkisi)"
                : !sufficientSample
                    ? $"Her varyant için en az {requiredSamplePerVariant} kullanıcı gerekli"
                    : "Test sonuçlandırmaya hazır"
        };
    }

    private static int CalculateRequiredSampleSize(double baseline, double mde)
        => (int)Math.Ceiling(16 * baseline * (1 - baseline) / Math.Pow(mde * baseline, 2));
}

public class ExperimentReadinessResult
{
    public bool IsReadyForConclusion { get; set; }
    public int RequiredUsersPerVariant { get; set; }
    public int DaysElapsed { get; set; }
    public string Message { get; set; } = string.Empty;
}
```

### 5. Soru

**Kullanıcı segmentasyonu ve hedefleme stratejileri nelerdir? .NET'te nasıl implement edilir?**

**Cevap:** Kullanıcı segmentasyonu; demografik özellikler (konum, dil, cihaz), davranışsal özellikler (son 30 gündeki aktivite, satın alma geçmişi), hesap özellikleri (plan türü, kayıt tarihi, yaş) ve teknik özellikler (tarayıcı, OS, uygulama versiyonu) gibi kriterlere göre yapılabilir. .NET'te segmentasyon genellikle custom `IFeatureFilter` implementasyonu ile gerçekleştirilir; kullanıcı profili önbellekten veya veritabanından alınır ve segment kriterleri değerlendirilir. Birden fazla segment tanımlanabilir ve öncelik sıralaması uygulanabilir. Karmaşık segmentasyon için kural motoru (örn. JSON tabanlı kural tanımları) kullanılması tavsiye edilir.

**Örnek Kod:**

```csharp
[FilterAlias("UserSegment")]
public class UserSegmentFilter : IFeatureFilter
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    private readonly IUserProfileService _userProfileService;

    public UserSegmentFilter(
        IHttpContextAccessor httpContextAccessor,
        IUserProfileService userProfileService)
    {
        _httpContextAccessor = httpContextAccessor;
        _userProfileService = userProfileService;
    }

    public async Task<bool> EvaluateAsync(FeatureFilterEvaluationContext context)
    {
        var settings = context.Parameters.Get<UserSegmentSettings>()
            ?? new UserSegmentSettings();

        var userId = _httpContextAccessor.HttpContext?.User?.Identity?.Name;
        if (string.IsNullOrEmpty(userId)) return false;

        var profile = await _userProfileService.GetProfileAsync(userId);
        if (profile == null) return false;

        // Koşulların tümü sağlanmalı (AND mantığı)
        return CheckPlanType(profile, settings)
            && CheckAccountAge(profile, settings)
            && CheckRegion(profile, settings);
    }

    private static bool CheckPlanType(UserProfile p, UserSegmentSettings s)
        => s.AllowedPlans.Count == 0 || s.AllowedPlans.Contains(p.SubscriptionPlan);

    private static bool CheckAccountAge(UserProfile p, UserSegmentSettings s)
        => (DateTime.UtcNow - p.CreatedAt).Days >= s.MinAccountAgeDays;

    private static bool CheckRegion(UserProfile p, UserSegmentSettings s)
        => s.AllowedRegions.Count == 0 || s.AllowedRegions.Contains(p.Region);
}

public class UserSegmentSettings
{
    public List<string> AllowedPlans { get; set; } = new();
    public int MinAccountAgeDays { get; set; } = 0;
    public List<string> AllowedRegions { get; set; } = new();
}
```

### 6. Soru

**Mikro servis mimarisinde feature flag yönetimi nasıl yapılır? Tutarlılık nasıl sağlanır?**

**Cevap:** Mikro servis mimarisinde her servisin kendi flag yapılandırmasına sahip olması tutarsızlığa yol açar; örneğin aynı özellik bir serviste açık başka bir serviste kapalı olabilir. Bu sorunu çözmek için merkezi bir flag yönetim hizmeti (Azure App Configuration, LaunchDarkly, Unleash) kullanılmalıdır. Her servis bu merkezi hizmetten flag değerlerini okur ve kısa süreli (30-60 saniye) önbellek tutar. Mesaj kuyruğu veya event bus aracılığıyla anlık flag değişikliği yayını yapılabilir. Distributed tracing ile flag durumu request context'ine eklenmeli ve log'lara yansıtılmalıdır. Özellikle kullanıcı deneyimi açısından kritik olan flag'lerde servisler arasında tutarlılık için kullanıcı atamalarının merkezi olarak saklanması tercih edilir.

**Örnek Kod:**

```csharp
// Merkezi flag servisi client'ı
public class CentralizedFeatureFlagClient : IFeatureFlagClient
{
    private readonly HttpClient _httpClient;
    private readonly IMemoryCache _cache;
    private readonly ILogger<CentralizedFeatureFlagClient> _logger;

    public CentralizedFeatureFlagClient(
        HttpClient httpClient,
        IMemoryCache cache,
        ILogger<CentralizedFeatureFlagClient> logger)
    {
        _httpClient = httpClient;
        _cache = cache;
        _logger = logger;
    }

    public async Task<bool> IsEnabledAsync(string flagName, string userId)
    {
        var cacheKey = $"flag:{flagName}:{userId}";

        return await _cache.GetOrCreateAsync(cacheKey, async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromSeconds(30);

            try
            {
                var response = await _httpClient.GetFromJsonAsync<FlagResponse>(
                    $"/api/flags/{flagName}/evaluate?userId={userId}");

                return response?.IsEnabled ?? false;
            }
            catch (Exception ex)
            {
                _logger.LogWarning(ex,
                    "Flag servisi erişilemez, varsayılan değer kullanılıyor: {Flag}",
                    flagName);

                // Fail-safe: merkezi servis erişilemezse flag kapalı kabul et
                entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromSeconds(5);
                return false;
            }
        });
    }
}
```

### 7. Soru

**Feature flag ve A/B test altyapısı için integration test nasıl yazılır?**

**Cevap:** Integration testlerde gerçek flag davranışını test etmek için `WebApplicationFactory<T>` içinde `AddFeatureManagement()` yapılandırması overrride edilir veya flag değerleri test için sabit bir `IFeatureDefinitionProvider` ile belirlenir. A/B test mantığını test ederken deterministik kullanıcı ID'leri kullanılarak hangi kullanıcının hangi varyanta düştüğü öngörülebilir hale getirilir. Hash fonksiyonunun tutarlılığı ayrıca birim testle doğrulanmalıdır. Canary karar mantığı için hem treatment hem control senaryoları test edilmeli, metrik izleme ve rollback mantığının doğruluğu mock servislerle doğrulanmalıdır.

**Örnek Kod:**

```csharp
public class FeatureFlagIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public FeatureFlagIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }

    [Fact]
    public async Task Checkout_WhenNewFlowEnabled_ReturnsNewLayout()
    {
        var client = _factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureAppConfiguration((ctx, config) =>
            {
                config.AddInMemoryCollection(new Dictionary<string, string?>
                {
                    ["FeatureManagement:NewCheckoutFlow"] = "true"
                });
            });
        }).CreateClient();

        var response = await client.GetAsync("/api/checkout/config");
        response.EnsureSuccessStatusCode();

        var config = await response.Content
            .ReadFromJsonAsync<CheckoutPageConfig>();

        Assert.Equal("one_page", config!.Layout);
    }

    [Fact]
    public async Task Rollout_SameUserAlwaysGetsConsistentVariant()
    {
        var service = new PercentageBasedRolloutService(
            Mock.Of<ILogger<PercentageBasedRolloutService>>());

        const string userId = "user-12345";
        const string featureName = "TestFeature";
        const double percentage = 50.0;

        // Aynı kullanıcı için 100 kez kontrol et
        var results = Enumerable.Range(0, 100)
            .Select(_ => service.IsUserInRollout(userId, featureName, percentage))
            .Distinct()
            .ToList();

        // Tüm sonuçlar aynı olmalı (deterministik)
        Assert.Single(results);
    }
}
```

## Best Practices

### 1. **Deney Tasarımı**
- Her deney için tek bir birincil metrik belirleyin
- Guardrail metrikler tanımlayarak regresyonu önleyin
- Minimum tespit edilebilir etkiyi (MDE) önceden belirleyin
- Mevsimsellik etkisini göz önünde bulundurun (haftanın günü, ay)

### 2. **Kullanıcı Deneyimi**
- Kullanıcı bir deney sırasında varyant değiştirmemeli (sticky assignment)
- Aynı kullanıcı birbiriyle çelişen iki deneye dahil olmamalı
- Önemli UI değişiklikleri için yeterli ısınma süresi bırakın

### 3. **Veri Kalitesi**
- Bot trafiğini filtrelayın
- Yalnızca özelliğe maruz kalan kullanıcıları analiz edin (intent-to-treat)
- Outlier değerleri winsorize edin

### 4. **Operasyonel Hazırlık**
- Her rollout için geri dönüş (rollback) planı hazırlayın
- Otomatik rollback eşiklerini ve koşullarını önceden belirleyin
- Tüm değişiklikleri audit log'a kaydedin

## Kaynaklar

- [Microsoft Feature Management Dokümantasyonu](https://docs.microsoft.com/en-us/azure/azure-app-configuration/feature-management-overview)
- [Experimentation Platform at Microsoft](https://www.microsoft.com/en-us/research/group/experimentation-platform-exp/)
- [Trustworthy Online Controlled Experiments (Kohavi, Tang, Xu)](https://experimentguide.com/)
- [Optimizely A/B Testing Guide](https://www.optimizely.com/optimization-glossary/ab-testing/)
- [LaunchDarkly - Progressive Delivery](https://launchdarkly.com/progressive-delivery/)
