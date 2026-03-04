# Cloud Cost Management

## Genel Bakış

Cloud Cost Management, bulut altyapısı harcamalarının sistematik olarak izlenmesi, analiz edilmesi ve optimize edilmesi sürecidir. Azure ve AWS gibi büyük bulut sağlayıcıları; maliyet görünürlüğü, bütçe yönetimi ve optimizasyon önerileri için kapsamlı araçlar sunar. Senior .NET geliştiricileri bu araçları hem portal üzerinden hem de programatik olarak kullanabilmelidir.

Etkili cloud cost management; right-sizing, reserved capacity, spot kullanımı ve auto-scaling stratejilerini bir arada ele alarak hem performans hem de maliyet hedeflerini dengelemeyi gerektirir.

## Mülakat Soruları ve Cevapları

### 1. Soru: Azure Cost Management ile programatik maliyet izleme nasıl yapılır?

**Cevap:**
Azure Cost Management, REST API ve .NET SDK aracılığıyla maliyet verilerine programatik erişim sağlar. `Azure.ResourceManager.CostManagement` NuGet paketi kullanılarak abonelik, kaynak grubu veya kaynak bazında sorgu yapılabilir. Maliyet verileri tarih aralığı, granülarite (günlük/aylık) ve boyut filtreleri ile sorgulanır.

Temel kullanım adımları:
- `DefaultAzureCredential` ile kimlik doğrulama
- `CostManagementExtensions` ile scope tanımlama
- `QueryDefinition` ile sorgu parametrelerini yapılandırma
- Sonuçları ayrıştırıp metrik sistemlerine aktarma

**Örnek Kod:**
```csharp
using Azure.Identity;
using Azure.ResourceManager;
using Azure.ResourceManager.CostManagement;
using Azure.ResourceManager.CostManagement.Models;

public class AzureCostMonitoringService
{
    private readonly ArmClient _armClient;
    private readonly ILogger<AzureCostMonitoringService> _logger;
    private readonly string _subscriptionId;

    public AzureCostMonitoringService(
        ILogger<AzureCostMonitoringService> logger,
        IConfiguration configuration)
    {
        _logger = logger;
        _subscriptionId = configuration["Azure:SubscriptionId"]
            ?? throw new ArgumentNullException("Azure:SubscriptionId");
        _armClient = new ArmClient(new DefaultAzureCredential());
    }

    public async Task<CostSummary> GetMonthlyCostAsync(int year, int month)
    {
        var startDate = new DateTimeOffset(year, month, 1, 0, 0, 0, TimeSpan.Zero);
        var endDate = startDate.AddMonths(1).AddDays(-1);

        var scope = $"/subscriptions/{_subscriptionId}";

        var queryDefinition = new QueryDefinition(ExportType.ActualCost)
        {
            TimePeriod = new QueryTimePeriod(startDate, endDate),
            Granularity = GranularityType.Monthly,
            Dataset = new QueryDataset
            {
                Aggregation =
                {
                    ["totalCost"] = new QueryAggregation("Cost", FunctionType.Sum)
                },
                Grouping =
                {
                    new QueryGrouping(QueryColumnType.Dimension, "ServiceName"),
                    new QueryGrouping(QueryColumnType.Dimension, "ResourceGroupName")
                }
            }
        };

        _logger.LogInformation(
            "Azure maliyet sorgusu başlatılıyor: {Start} - {End}",
            startDate.ToString("yyyy-MM-dd"),
            endDate.ToString("yyyy-MM-dd"));

        // Scope üzerinden sorguyu çalıştır
        var subscriptionResource = _armClient.GetSubscriptionResource(
            new ResourceIdentifier(scope));

        var results = new List<ServiceCost>();
        decimal totalCost = 0;

        // Gerçek API çağrısı için CostManagementExtensions kullanılır
        // Bu örnek sorgu yapısını göstermektedir
        _logger.LogInformation("Toplam aylık maliyet: {Total:C}", totalCost);

        return new CostSummary
        {
            Period = $"{year}-{month:D2}",
            TotalCostUsd = totalCost,
            ServiceBreakdown = results
        };
    }

    public async Task<IReadOnlyList<DailyCost>> GetDailyCostTrendAsync(
        DateTimeOffset startDate,
        DateTimeOffset endDate,
        string? resourceGroupName = null)
    {
        _logger.LogInformation(
            "Günlük maliyet trendi sorgulanıyor: {ResourceGroup}",
            resourceGroupName ?? "Tüm abonelik");

        var scope = resourceGroupName != null
            ? $"/subscriptions/{_subscriptionId}/resourceGroups/{resourceGroupName}"
            : $"/subscriptions/{_subscriptionId}";

        // Gerçek uygulamada Azure SDK QueryUsageAsync çağrısı yapılır
        // Burada veri modeli ve sorgu yapısı gösterilmektedir
        var dailyCosts = new List<DailyCost>();

        for (var date = startDate.Date; date <= endDate.Date; date = date.AddDays(1))
        {
            dailyCosts.Add(new DailyCost
            {
                Date = date,
                CostUsd = 0, // Gerçek API'den doldurulur
                ResourceGroup = resourceGroupName ?? "all"
            });
        }

        return dailyCosts;
    }
}

public record CostSummary
{
    public required string Period { get; init; }
    public decimal TotalCostUsd { get; init; }
    public List<ServiceCost> ServiceBreakdown { get; init; } = new();
}

public record ServiceCost
{
    public required string ServiceName { get; init; }
    public required string ResourceGroupName { get; init; }
    public decimal CostUsd { get; init; }
}

public record DailyCost
{
    public DateTime Date { get; init; }
    public decimal CostUsd { get; init; }
    public required string ResourceGroup { get; init; }
}
```

---

### 2. Soru: Right-sizing analizi nedir ve nasıl otomatize edilir?

**Cevap:**
Right-sizing, sanal makinelerin, container'ların ve diğer bulut kaynaklarının gerçek kullanım ihtiyacına uygun boyuta getirilmesidir. Kaynaklar çoğunlukla peak yük için aşırı boyutlandırılır; ancak ortalama kullanım çok daha düşük kalır. Bu da gereksiz maliyete yol açar.

Right-sizing analizi için:
- Son 7-30 günlük CPU, bellek, disk I/O ve ağ metrikleri toplanır
- P95 veya P99 değerleri baz alınarak gerçek maksimum ihtiyaç hesaplanır
- Mevcut kapasite ile bu ihtiyaç karşılaştırılır
- Önerilen boyut, Azure Advisor veya özel analiz ile belirlenir

Azure Monitor Metrics API ile bu analiz C# kodu ile otomatize edilebilir.

**Örnek Kod:**
```csharp
using Azure.Identity;
using Azure.Monitor.Query;
using Azure.Monitor.Query.Models;

public class RightSizingAnalyzer
{
    private readonly MetricsQueryClient _metricsClient;
    private readonly ILogger<RightSizingAnalyzer> _logger;

    public RightSizingAnalyzer(ILogger<RightSizingAnalyzer> logger)
    {
        _logger = logger;
        _metricsClient = new MetricsQueryClient(new DefaultAzureCredential());
    }

    public async Task<RightSizingRecommendation> AnalyzeVirtualMachineAsync(
        string resourceId,
        string vmSize,
        int analysisDays = 14)
    {
        var endTime = DateTimeOffset.UtcNow;
        var startTime = endTime.AddDays(-analysisDays);

        _logger.LogInformation(
            "Right-sizing analizi başlatılıyor. VM: {ResourceId}, Süre: {Days} gün",
            resourceId, analysisDays);

        var metricsResponse = await _metricsClient.QueryResourceAsync(
            resourceId,
            new[] { "Percentage CPU", "Available Memory Bytes", "Network In Total" },
            new MetricsQueryOptions
            {
                TimeRange = new QueryTimeRange(startTime, endTime),
                Granularity = TimeSpan.FromHours(1)
            });

        var cpuMetric = metricsResponse.Value.Metrics
            .FirstOrDefault(m => m.Name == "Percentage CPU");

        var memoryMetric = metricsResponse.Value.Metrics
            .FirstOrDefault(m => m.Name == "Available Memory Bytes");

        double avgCpu = 0, maxCpu = 0, p95Cpu = 0;
        double avgMemoryAvailableGb = 0;

        if (cpuMetric?.TimeSeries?.Any() == true)
        {
            var cpuValues = cpuMetric.TimeSeries
                .SelectMany(ts => ts.Values)
                .Where(v => v.Average.HasValue)
                .Select(v => v.Average!.Value)
                .ToList();

            if (cpuValues.Any())
            {
                avgCpu = cpuValues.Average();
                maxCpu = cpuValues.Max();
                p95Cpu = CalculatePercentile(cpuValues, 95);
            }
        }

        if (memoryMetric?.TimeSeries?.Any() == true)
        {
            var memValues = memoryMetric.TimeSeries
                .SelectMany(ts => ts.Values)
                .Where(v => v.Average.HasValue)
                .Select(v => v.Average!.Value / (1024 * 1024 * 1024)) // Bytes -> GB
                .ToList();

            if (memValues.Any())
                avgMemoryAvailableGb = memValues.Average();
        }

        var recommendation = DetermineRecommendation(
            currentVmSize: vmSize,
            avgCpuPercent: avgCpu,
            p95CpuPercent: p95Cpu,
            avgAvailableMemoryGb: avgMemoryAvailableGb);

        _logger.LogInformation(
            "Right-sizing sonucu: Mevcut={Current}, Öneri={Recommended}, " +
            "Tahmini Tasarruf={Savings:P0}",
            recommendation.CurrentSize,
            recommendation.RecommendedSize,
            recommendation.EstimatedSavingsPercent / 100.0);

        return recommendation;
    }

    private static RightSizingRecommendation DetermineRecommendation(
        string currentVmSize,
        double avgCpuPercent,
        double p95CpuPercent,
        double avgAvailableMemoryGb)
    {
        // Basitleştirilmiş boyutlandırma mantığı
        // Gerçek uygulamada Azure VM SKU kataloğu kullanılır
        var isUndersized = p95CpuPercent > 85 || avgAvailableMemoryGb < 2;
        var isOversized = avgCpuPercent < 10 && p95CpuPercent < 20;

        if (isOversized)
        {
            return new RightSizingRecommendation
            {
                ResourceId = currentVmSize,
                CurrentSize = currentVmSize,
                RecommendedSize = GetSmallerSize(currentVmSize),
                RecommendationReason = $"Ortalama CPU kullanımı düşük ({avgCpuPercent:F1}%)",
                EstimatedSavingsPercent = 30,
                AnalysisPeriodDays = 14,
                AvgCpuPercent = avgCpuPercent,
                P95CpuPercent = p95CpuPercent
            };
        }

        if (isUndersized)
        {
            return new RightSizingRecommendation
            {
                ResourceId = currentVmSize,
                CurrentSize = currentVmSize,
                RecommendedSize = GetLargerSize(currentVmSize),
                RecommendationReason = $"P95 CPU kullanımı yüksek ({p95CpuPercent:F1}%)",
                EstimatedSavingsPercent = -15, // Maliyet artışı
                AnalysisPeriodDays = 14,
                AvgCpuPercent = avgCpuPercent,
                P95CpuPercent = p95CpuPercent
            };
        }

        return new RightSizingRecommendation
        {
            ResourceId = currentVmSize,
            CurrentSize = currentVmSize,
            RecommendedSize = currentVmSize,
            RecommendationReason = "Mevcut boyut optimum",
            EstimatedSavingsPercent = 0,
            AnalysisPeriodDays = 14,
            AvgCpuPercent = avgCpuPercent,
            P95CpuPercent = p95CpuPercent
        };
    }

    private static double CalculatePercentile(List<double> values, int percentile)
    {
        if (!values.Any()) return 0;
        var sorted = values.OrderBy(v => v).ToList();
        var index = (int)Math.Ceiling(percentile / 100.0 * sorted.Count) - 1;
        return sorted[Math.Max(0, Math.Min(index, sorted.Count - 1))];
    }

    private static string GetSmallerSize(string currentSize) =>
        currentSize switch
        {
            "Standard_D4s_v3" => "Standard_D2s_v3",
            "Standard_D8s_v3" => "Standard_D4s_v3",
            "Standard_D16s_v3" => "Standard_D8s_v3",
            _ => currentSize
        };

    private static string GetLargerSize(string currentSize) =>
        currentSize switch
        {
            "Standard_D2s_v3" => "Standard_D4s_v3",
            "Standard_D4s_v3" => "Standard_D8s_v3",
            "Standard_D8s_v3" => "Standard_D16s_v3",
            _ => currentSize
        };
}

public record RightSizingRecommendation
{
    public required string ResourceId { get; init; }
    public required string CurrentSize { get; init; }
    public required string RecommendedSize { get; init; }
    public required string RecommendationReason { get; init; }
    public double EstimatedSavingsPercent { get; init; }
    public int AnalysisPeriodDays { get; init; }
    public double AvgCpuPercent { get; init; }
    public double P95CpuPercent { get; init; }
}
```

---

### 3. Soru: Reserved Instances ile Spot Instances'ı hangi senaryolarda kullanmalıyız?

**Cevap:**
Reserved Instances (RI) ve Spot/Low-Priority Instances, farklı iş yükü özelliklerine göre kullanılır:

**Reserved Instances:**
- 1 veya 3 yıllık taahhüt karşılığında %30-72 indirim (Azure) veya %40-75 indirim (AWS)
- 7/24 çalışan, öngörülebilir iş yükleri için idealdir (veritabanları, API sunucuları, temel servisler)
- Scope olarak paylaşımlı (tüm abonelik) veya tek kaynak grubu seçilebilir
- Instance flexibility ile aynı VM ailesinin farklı boyutlarına uygulanabilir

**Spot Instances:**
- Azure'da "Spot VM", AWS'de "Spot Instance" olarak bilinir
- %60-90 maliyet avantajı sağlar, ancak kapasite geri alınabilir
- Hata toleranslı ve checkpoint destekli iş yükleri için uygundur: batch işleme, ML model eğitimi, rendering, test ortamları
- Preemption (geri alma) durumunda workload'ın güvenli kapanması sağlanmalıdır

**Hibrit strateji:** Production ortamı için RI tabanı, trafik patlamalarını karşılamak için on-demand ve test/batch için spot kombinasyonu ideal denge sağlar.

**Örnek Kod:**
```csharp
public class InstanceStrategySelector
{
    private readonly ILogger<InstanceStrategySelector> _logger;

    public InstanceStrategySelector(ILogger<InstanceStrategySelector> logger)
    {
        _logger = logger;
    }

    public InstanceStrategy DetermineOptimalStrategy(WorkloadProfile workload)
    {
        _logger.LogInformation(
            "İş yükü analiz ediliyor: {WorkloadName}, Tür: {Type}",
            workload.Name, workload.Type);

        // Kritik, kesintisiz iş yükleri
        if (workload.RequiresHighAvailability && workload.IsAlwaysOn)
        {
            return new InstanceStrategy
            {
                BaseCapacity = InstanceType.Reserved,
                BurstCapacity = InstanceType.OnDemand,
                RecommendedReservationTerm = ReservationTerm.OneYear,
                EstimatedMonthlySavingsPercent = 40,
                Rationale = "7/24 çalışan kritik iş yükü - Reserved Instance önerilir"
            };
        }

        // Hata toleranslı, kesintili iş yükleri
        if (workload.IsFaultTolerant && workload.SupportsCheckpointing)
        {
            return new InstanceStrategy
            {
                BaseCapacity = InstanceType.Spot,
                BurstCapacity = InstanceType.OnDemand,
                RecommendedReservationTerm = ReservationTerm.None,
                EstimatedMonthlySavingsPercent = 70,
                Rationale = "Kesintili iş yükü - Spot Instance ile maksimum tasarruf"
            };
        }

        // Geliştirme/test ortamları
        if (workload.Type == WorkloadType.Development ||
            workload.Type == WorkloadType.Testing)
        {
            return new InstanceStrategy
            {
                BaseCapacity = InstanceType.Spot,
                BurstCapacity = InstanceType.OnDemand,
                RecommendedReservationTerm = ReservationTerm.None,
                EstimatedMonthlySavingsPercent = 65,
                Rationale = "Dev/Test ortamı - Spot + otomatik kapatma önerilir"
            };
        }

        // Hibrit: tahmin edilebilir taban + esnek burst
        return new InstanceStrategy
        {
            BaseCapacity = InstanceType.Reserved,
            BurstCapacity = InstanceType.Spot,
            RecommendedReservationTerm = ReservationTerm.OneYear,
            EstimatedMonthlySavingsPercent = 55,
            Rationale = "Karma iş yükü - Reserved taban + Spot burst hibrit stratejisi"
        };
    }
}

public record WorkloadProfile
{
    public required string Name { get; init; }
    public WorkloadType Type { get; init; }
    public bool RequiresHighAvailability { get; init; }
    public bool IsAlwaysOn { get; init; }
    public bool IsFaultTolerant { get; init; }
    public bool SupportsCheckpointing { get; init; }
}

public record InstanceStrategy
{
    public InstanceType BaseCapacity { get; init; }
    public InstanceType BurstCapacity { get; init; }
    public ReservationTerm RecommendedReservationTerm { get; init; }
    public double EstimatedMonthlySavingsPercent { get; init; }
    public required string Rationale { get; init; }
}

public enum WorkloadType { Production, Development, Testing, Batch, Analytics }
public enum InstanceType { OnDemand, Reserved, Spot }
public enum ReservationTerm { None, OneYear, ThreeYear }
```

---

### 4. Soru: Auto-scaling stratejileri maliyet optimizasyonunu nasıl destekler?

**Cevap:**
Auto-scaling, talep ile kapasite arasındaki dengeyi dinamik olarak yönetir. Doğru yapılandırılmış auto-scaling politikaları:
- Trafik düşük olduğunda kaynakları azaltarak boşa harcamayı önler
- Trafik artışlarında manüel müdahale gerektirmeden kapasiteyi artırır
- Öngörülü (predictive) scaling ile schedule-based önceden hazırlık sağlar

Azure'da VMSS (Virtual Machine Scale Sets), AKS (Kubernetes) ve App Service Plan scale-out destekler. Kubernetes'te HPA (Horizontal Pod Autoscaler), KEDA (event-driven) ve cluster autoscaler bir arada kullanılabilir.

**Örnek Kod:**
```csharp
using Azure.Identity;
using Azure.ResourceManager;
using Azure.ResourceManager.Monitor;
using Azure.ResourceManager.Monitor.Models;

public class AutoScalingCostOptimizer
{
    private readonly ArmClient _armClient;
    private readonly ILogger<AutoScalingCostOptimizer> _logger;

    public AutoScalingCostOptimizer(ILogger<AutoScalingCostOptimizer> logger)
    {
        _logger = logger;
        _armClient = new ArmClient(new DefaultAzureCredential());
    }

    public async Task<AutoScaleSettingResource> ConfigureCostAwareScalingAsync(
        string resourceGroupId,
        string targetResourceId,
        CostAwareScalingConfig config)
    {
        _logger.LogInformation(
            "Maliyet odaklı auto-scaling yapılandırılıyor: MinInstances={Min}, " +
            "MaxInstances={Max}",
            config.MinInstances,
            config.MaxInstances);

        var resourceGroup = _armClient.GetResourceGroupResource(
            new ResourceIdentifier(resourceGroupId));

        var autoScaleData = new AutoScaleSettingData("West Europe")
        {
            IsEnabled = true,
            TargetResourceId = new ResourceIdentifier(targetResourceId),
            Profiles =
            {
                // İş saatleri profili - daha yüksek kapasite
                new AutoScaleProfile(
                    name: "business-hours",
                    capacity: new ScaleCapacity(
                        minimum: config.BusinessHoursMinInstances.ToString(),
                        maximum: config.MaxInstances.ToString(),
                        defaultProperty: config.BusinessHoursMinInstances.ToString()),
                    rules: new List<ScaleRule>
                    {
                        // CPU > %70 ise ölçek artır
                        new ScaleRule(
                            metricTrigger: new MetricTrigger(
                                metricName: "CpuPercentage",
                                metricResourceId: new ResourceIdentifier(targetResourceId),
                                timeGrain: TimeSpan.FromMinutes(1),
                                statistic: MetricStatisticType.Average,
                                timeWindow: TimeSpan.FromMinutes(5),
                                timeAggregation: TimeAggregationType.Average,
                                @operator: ComparisonOperationType.GreaterThan,
                                threshold: 70),
                            scaleAction: new ScaleAction(
                                direction: ScaleDirection.Increase,
                                scaleType: ScaleType.ChangeCount,
                                cooldown: TimeSpan.FromMinutes(5))
                            {
                                Value = "2"
                            }),
                        // CPU < %30 ise ölçek azalt
                        new ScaleRule(
                            metricTrigger: new MetricTrigger(
                                metricName: "CpuPercentage",
                                metricResourceId: new ResourceIdentifier(targetResourceId),
                                timeGrain: TimeSpan.FromMinutes(1),
                                statistic: MetricStatisticType.Average,
                                timeWindow: TimeSpan.FromMinutes(10),
                                timeAggregation: TimeAggregationType.Average,
                                @operator: ComparisonOperationType.LessThan,
                                threshold: 30),
                            scaleAction: new ScaleAction(
                                direction: ScaleDirection.Decrease,
                                scaleType: ScaleType.ChangeCount,
                                cooldown: TimeSpan.FromMinutes(10))
                            {
                                Value = "1"
                            })
                    })
                {
                    // Çalışma saatleri: Pazartesi-Cuma 08:00-18:00
                    Recurrence = new Recurrence(
                        frequency: RecurrenceFrequency.Week,
                        schedule: new RecurrentSchedule(
                            timeZone: "Turkey Standard Time",
                            days: new List<string> { "Monday", "Tuesday", "Wednesday", "Thursday", "Friday" },
                            hours: new List<int> { 8 },
                            minutes: new List<int> { 0 }))
                },

                // Gece/hafta sonu profili - minimum kapasite
                new AutoScaleProfile(
                    name: "off-hours",
                    capacity: new ScaleCapacity(
                        minimum: config.MinInstances.ToString(),
                        maximum: config.OffHoursMaxInstances.ToString(),
                        defaultProperty: config.MinInstances.ToString()),
                    rules: new List<ScaleRule>())
                {
                    Recurrence = new Recurrence(
                        frequency: RecurrenceFrequency.Week,
                        schedule: new RecurrentSchedule(
                            timeZone: "Turkey Standard Time",
                            days: new List<string> { "Monday", "Tuesday", "Wednesday", "Thursday", "Friday" },
                            hours: new List<int> { 18 },
                            minutes: new List<int> { 0 }))
                }
            }
        };

        var autoScaleSettings = resourceGroup.GetAutoScaleSettings();
        var result = await autoScaleSettings.CreateOrUpdateAsync(
            Azure.WaitUntil.Completed,
            "cost-aware-scaling",
            autoScaleData);

        _logger.LogInformation(
            "Auto-scaling yapılandırması tamamlandı. Tahmini tasarruf: {Savings:P0}",
            config.EstimatedSavingsPercent / 100.0);

        return result.Value;
    }
}

public record CostAwareScalingConfig
{
    public int MinInstances { get; init; } = 1;
    public int MaxInstances { get; init; } = 10;
    public int BusinessHoursMinInstances { get; init; } = 2;
    public int OffHoursMaxInstances { get; init; } = 3;
    public double EstimatedSavingsPercent { get; init; } = 35;
}
```

---

### 5. Soru: Kullanılmayan bulut kaynaklarını tespit etmek için nasıl bir sistem kurulur?

**Cevap:**
Kullanılmayan kaynaklar (orphaned resources) bulut maliyetlerinin önemli bir bölümünü oluşturabilir. Tipik israf kaynakları:
- VM'den bağımsız hale gelmiş diskler (unattached disks)
- Silinmiş VM'lere ait statik IP adresleri
- Boş load balancer'lar
- Trafik almayan Application Gateway örnekleri
- Silinmiş servislere ait depolama hesapları

Azure Resource Graph ile tüm abonelik boyutunda bu kaynakları programatik olarak taramak mümkündür.

**Örnek Kod:**
```csharp
using Azure.Identity;
using Azure.ResourceManager.ResourceGraph;
using Azure.ResourceManager.ResourceGraph.Models;

public class OrphanedResourceScanner
{
    private readonly ResourceGraphClient _resourceGraphClient;
    private readonly ILogger<OrphanedResourceScanner> _logger;
    private readonly string _subscriptionId;

    public OrphanedResourceScanner(
        ILogger<OrphanedResourceScanner> logger,
        IConfiguration configuration)
    {
        _logger = logger;
        _subscriptionId = configuration["Azure:SubscriptionId"]!;
        _resourceGraphClient = new ResourceGraphClient(new DefaultAzureCredential());
    }

    public async Task<OrphanedResourceReport> ScanForOrphanedResourcesAsync()
    {
        _logger.LogInformation("Sahipsiz kaynak taraması başlatılıyor...");

        var report = new OrphanedResourceReport
        {
            ScannedAt = DateTime.UtcNow,
            SubscriptionId = _subscriptionId
        };

        // Bağlı olmayan managed diskler
        report.UnattachedDisks = await FindUnattachedManagedDisksAsync();

        // Kullanılmayan statik IP adresleri
        report.UnusedPublicIPs = await FindUnusedPublicIPsAsync();

        // Boş ağ arayüzleri
        report.UnattachedNetworkInterfaces = await FindUnattachedNICsAsync();

        // Boş kaynak grupları
        report.EmptyResourceGroups = await FindEmptyResourceGroupsAsync();

        report.TotalEstimatedMonthlyCostUsd =
            report.UnattachedDisks.Sum(d => d.EstimatedMonthlyCostUsd) +
            report.UnusedPublicIPs.Sum(ip => ip.EstimatedMonthlyCostUsd) +
            report.UnattachedNetworkInterfaces.Sum(n => n.EstimatedMonthlyCostUsd);

        _logger.LogInformation(
            "Tarama tamamlandı. Toplam israf: {Cost:C}/ay, " +
            "Disk: {Disks}, IP: {IPs}, NIC: {NICs}",
            report.TotalEstimatedMonthlyCostUsd,
            report.UnattachedDisks.Count,
            report.UnusedPublicIPs.Count,
            report.UnattachedNetworkInterfaces.Count);

        return report;
    }

    private async Task<List<OrphanedResource>> FindUnattachedManagedDisksAsync()
    {
        var query = """
            Resources
            | where type == 'microsoft.compute/disks'
            | where managedBy == ''
            | where properties.diskState == 'Unattached'
            | project id, name, resourceGroup, location,
                      diskSizeGB = properties.diskSku.name,
                      skuName = sku.name,
                      createdAt = properties.timeCreated
            | order by name asc
            """;

        return await ExecuteResourceGraphQueryAsync(
            query,
            estimatedCostPerUnit: 5.0m, // Ortalama 5 USD/ay disk başına
            resourceType: "Unattached Managed Disk");
    }

    private async Task<List<OrphanedResource>> FindUnusedPublicIPsAsync()
    {
        var query = """
            Resources
            | where type == 'microsoft.network/publicipaddresses'
            | where isnull(properties.ipConfiguration)
            | where properties.publicIPAllocationMethod == 'Static'
            | project id, name, resourceGroup, location,
                      sku = sku.name,
                      createdAt = properties.provisioningState
            | order by name asc
            """;

        return await ExecuteResourceGraphQueryAsync(
            query,
            estimatedCostPerUnit: 3.0m, // Statik IP ~3 USD/ay
            resourceType: "Unused Static Public IP");
    }

    private async Task<List<OrphanedResource>> FindUnattachedNICsAsync()
    {
        var query = """
            Resources
            | where type == 'microsoft.network/networkinterfaces'
            | where isnull(properties.virtualMachine)
            | project id, name, resourceGroup, location
            | order by name asc
            """;

        return await ExecuteResourceGraphQueryAsync(
            query,
            estimatedCostPerUnit: 0m, // NIC'in doğrudan maliyeti yok ama temiz mimari için önemli
            resourceType: "Unattached Network Interface");
    }

    private async Task<List<OrphanedResource>> FindEmptyResourceGroupsAsync()
    {
        var query = """
            ResourceContainers
            | where type == 'microsoft.resources/subscriptions/resourcegroups'
            | where not(id in (
                (Resources | project resourceGroup = tolower(resourceGroup) | distinct resourceGroup)
            ))
            | project id, name, location
            """;

        return await ExecuteResourceGraphQueryAsync(
            query,
            estimatedCostPerUnit: 0m,
            resourceType: "Empty Resource Group");
    }

    private async Task<List<OrphanedResource>> ExecuteResourceGraphQueryAsync(
        string query,
        decimal estimatedCostPerUnit,
        string resourceType)
    {
        var request = new ResourceQueryContent(query)
        {
            Subscriptions = { _subscriptionId }
        };

        try
        {
            var response = await _resourceGraphClient.ResourcesAsync(request);
            var resources = new List<OrphanedResource>();

            if (response?.Value?.Data != null)
            {
                foreach (var row in response.Value.Data.ToObjectFromJson<List<Dictionary<string, object>>>()
                         ?? Enumerable.Empty<Dictionary<string, object>>())
                {
                    resources.Add(new OrphanedResource
                    {
                        ResourceId = row.GetValueOrDefault("id")?.ToString() ?? string.Empty,
                        Name = row.GetValueOrDefault("name")?.ToString() ?? string.Empty,
                        ResourceGroup = row.GetValueOrDefault("resourceGroup")?.ToString() ?? string.Empty,
                        Location = row.GetValueOrDefault("location")?.ToString() ?? string.Empty,
                        ResourceType = resourceType,
                        EstimatedMonthlyCostUsd = estimatedCostPerUnit
                    });
                }
            }

            return resources;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Resource Graph sorgusu başarısız: {ResourceType}", resourceType);
            return new List<OrphanedResource>();
        }
    }
}

public record OrphanedResourceReport
{
    public DateTime ScannedAt { get; init; }
    public required string SubscriptionId { get; init; }
    public List<OrphanedResource> UnattachedDisks { get; init; } = new();
    public List<OrphanedResource> UnusedPublicIPs { get; init; } = new();
    public List<OrphanedResource> UnattachedNetworkInterfaces { get; init; } = new();
    public List<OrphanedResource> EmptyResourceGroups { get; init; } = new();
    public decimal TotalEstimatedMonthlyCostUsd { get; init; }
}

public record OrphanedResource
{
    public required string ResourceId { get; init; }
    public required string Name { get; init; }
    public required string ResourceGroup { get; init; }
    public required string Location { get; init; }
    public required string ResourceType { get; init; }
    public decimal EstimatedMonthlyCostUsd { get; init; }
}
```

---

### 6. Soru: AWS Cost Explorer ile maliyet analizi nasıl yapılır?

**Cevap:**
AWS Cost Explorer, harcama trendlerini görüntülemek, tahminler oluşturmak ve maliyet dağılımını analiz etmek için kullanılan bir AWS servisine dayalı araçtır. AWS SDK for .NET (AWSSDK.CostExplorer) paketi ile programatik erişim sağlanır.

Temel yetenekler:
- Servis, bölge, hesap veya etiket bazında maliyet dağılımı
- Kullanım türü ve satın alma seçeneği bazında gruplama
- Anomali tespiti ve uyarılar
- Tasarruf planı ve RI kapsama analizi
- 12 aylık tahmin

**Örnek Kod:**
```csharp
using Amazon.CostExplorer;
using Amazon.CostExplorer.Model;

public class AwsCostAnalysisService
{
    private readonly IAmazonCostExplorer _costExplorerClient;
    private readonly ILogger<AwsCostAnalysisService> _logger;

    public AwsCostAnalysisService(
        IAmazonCostExplorer costExplorerClient,
        ILogger<AwsCostAnalysisService> logger)
    {
        _costExplorerClient = costExplorerClient;
        _logger = logger;
    }

    public async Task<AwsCostReport> GetServiceCostBreakdownAsync(
        DateTime startDate,
        DateTime endDate)
    {
        _logger.LogInformation(
            "AWS servis maliyet dağılımı sorgulanıyor: {Start} - {End}",
            startDate.ToString("yyyy-MM-dd"),
            endDate.ToString("yyyy-MM-dd"));

        var request = new GetCostAndUsageRequest
        {
            TimePeriod = new DateInterval
            {
                Start = startDate.ToString("yyyy-MM-dd"),
                End = endDate.ToString("yyyy-MM-dd")
            },
            Granularity = Granularity.MONTHLY,
            Metrics = new List<string> { "BlendedCost", "UsageQuantity" },
            GroupBy = new List<GroupDefinition>
            {
                new GroupDefinition
                {
                    Type = GroupDefinitionType.DIMENSION,
                    Key = "SERVICE"
                },
                new GroupDefinition
                {
                    Type = GroupDefinitionType.DIMENSION,
                    Key = "REGION"
                }
            }
        };

        var response = await _costExplorerClient.GetCostAndUsageAsync(request);

        var serviceCosts = new List<AwsServiceCost>();

        foreach (var result in response.ResultsByTime)
        {
            foreach (var group in result.Groups)
            {
                var serviceName = group.Keys.ElementAtOrDefault(0) ?? "Unknown";
                var region = group.Keys.ElementAtOrDefault(1) ?? "Unknown";

                if (group.Metrics.TryGetValue("BlendedCost", out var costMetric))
                {
                    if (decimal.TryParse(costMetric.Amount, out var amount) && amount > 0)
                    {
                        serviceCosts.Add(new AwsServiceCost
                        {
                            ServiceName = serviceName,
                            Region = region,
                            Period = result.TimePeriod.Start,
                            BlendedCostUsd = amount,
                            Unit = costMetric.Unit
                        });
                    }
                }
            }
        }

        var totalCost = serviceCosts.Sum(s => s.BlendedCostUsd);

        _logger.LogInformation(
            "AWS maliyet analizi tamamlandı. Toplam: {Total:C}, Servis sayısı: {Count}",
            totalCost, serviceCosts.Select(s => s.ServiceName).Distinct().Count());

        return new AwsCostReport
        {
            StartDate = startDate,
            EndDate = endDate,
            TotalBlendedCostUsd = totalCost,
            ServiceCosts = serviceCosts
                .OrderByDescending(s => s.BlendedCostUsd)
                .ToList()
        };
    }

    public async Task<SavingsPlanCoverageReport> GetSavingsPlanCoverageAsync()
    {
        var request = new GetSavingsPlansCoverageRequest
        {
            TimePeriod = new DateInterval
            {
                Start = DateTime.UtcNow.AddMonths(-1).ToString("yyyy-MM-dd"),
                End = DateTime.UtcNow.ToString("yyyy-MM-dd")
            },
            Granularity = Granularity.MONTHLY
        };

        var response = await _costExplorerClient.GetSavingsPlansCoverageAsync(request);

        var coverages = response.SavingsPlansCoverages.Select(c => new CoverageDetail
        {
            Period = c.TimePeriod.Start,
            CoveragePercent = double.TryParse(
                c.Coverage?.CoveragePercentage, out var pct) ? pct : 0,
            OnDemandCostUsd = decimal.TryParse(
                c.Coverage?.OnDemandCost, out var od) ? od : 0,
            SpendCoveredBySavingsPlanUsd = decimal.TryParse(
                c.Coverage?.SpendCoveredBySavingsPlans, out var sp) ? sp : 0
        }).ToList();

        return new SavingsPlanCoverageReport
        {
            Coverages = coverages,
            AverageCoveragePercent = coverages.Any()
                ? coverages.Average(c => c.CoveragePercent)
                : 0
        };
    }
}

public record AwsCostReport
{
    public DateTime StartDate { get; init; }
    public DateTime EndDate { get; init; }
    public decimal TotalBlendedCostUsd { get; init; }
    public List<AwsServiceCost> ServiceCosts { get; init; } = new();
}

public record AwsServiceCost
{
    public required string ServiceName { get; init; }
    public required string Region { get; init; }
    public required string Period { get; init; }
    public decimal BlendedCostUsd { get; init; }
    public required string Unit { get; init; }
}

public record SavingsPlanCoverageReport
{
    public List<CoverageDetail> Coverages { get; init; } = new();
    public double AverageCoveragePercent { get; init; }
}

public record CoverageDetail
{
    public required string Period { get; init; }
    public double CoveragePercent { get; init; }
    public decimal OnDemandCostUsd { get; init; }
    public decimal SpendCoveredBySavingsPlanUsd { get; init; }
}
```

---

### 7. Soru: Bütçe uyarıları ve anomali tespiti nasıl yapılandırılır?

**Cevap:**
Bütçe uyarıları, harcamalar önceden belirlenen eşiklere ulaştığında otomatik bildirim gönderir. Azure'da `Microsoft.CostManagement/budgets` API'si; AWS'de `AWS Budgets` servisi bu işlevi sağlar. Anomali tespiti ise ML tabanlı olarak beklenmedik maliyet artışlarını tespit eder.

Azure bütçe türleri:
- Cost budget: Belirli bir harcama miktarına göre
- Usage budget: Kullanım miktarına göre
- Uyarı eşikleri: Gerçek harcama veya tahmin bazlı (%50, %75, %90, %100, %110 gibi)

**Örnek Kod:**
```csharp
using Azure.Identity;
using Azure.ResourceManager;
using Azure.ResourceManager.CostManagement;
using Azure.ResourceManager.CostManagement.Models;

public class BudgetAlertConfigurationService
{
    private readonly ArmClient _armClient;
    private readonly ILogger<BudgetAlertConfigurationService> _logger;

    public BudgetAlertConfigurationService(ILogger<BudgetAlertConfigurationService> logger)
    {
        _logger = logger;
        _armClient = new ArmClient(new DefaultAzureCredential());
    }

    public async Task<BudgetResource> CreateTeamBudgetAsync(
        string subscriptionId,
        TeamBudgetConfig config)
    {
        _logger.LogInformation(
            "Takım bütçesi oluşturuluyor: {Team}, Limit: {Limit:C}/ay",
            config.TeamName, config.MonthlyLimitUsd);

        var scope = $"/subscriptions/{subscriptionId}";
        var subscriptionResource = _armClient.GetSubscriptionResource(
            new ResourceIdentifier(scope));

        var budgets = subscriptionResource.GetBudgets();

        var budgetData = new BudgetData
        {
            Category = BudgetCategory.Cost,
            Amount = config.MonthlyLimitUsd,
            TimeGrain = BudgetTimeGrainType.Monthly,
            TimePeriod = new BudgetTimePeriod(
                startDate: new DateTimeOffset(DateTime.UtcNow.Year, DateTime.UtcNow.Month, 1, 0, 0, 0, TimeSpan.Zero),
                endDate: new DateTimeOffset(DateTime.UtcNow.Year + 2, 12, 31, 23, 59, 59, TimeSpan.Zero)),

            // Etiket filtresi ile takıma özel bütçe
            Filter = new BudgetFilter
            {
                Tags =
                {
                    new BudgetFilterProperties
                    {
                        Name = "team",
                        Values = { config.TeamName }
                    }
                }
            },

            // Çoklu uyarı eşikleri
            Notifications =
            {
                // %50 harcamada bilgi amaçlı uyarı
                ["budget-50pct"] = new BudgetNotification(
                    isEnabled: true,
                    operator: BudgetOperatorType.GreaterThan,
                    threshold: 50,
                    contactEmails: config.NotificationEmails)
                {
                    ThresholdType = ThresholdType.Actual,
                    ContactRoles = { "Owner", "Contributor" }
                },

                // %80 harcamada kritik uyarı
                ["budget-80pct"] = new BudgetNotification(
                    isEnabled: true,
                    operator: BudgetOperatorType.GreaterThan,
                    threshold: 80,
                    contactEmails: config.NotificationEmails)
                {
                    ThresholdType = ThresholdType.Actual,
                    ContactRoles = { "Owner" }
                },

                // %100 tahmin uyarısı (henüz ulaşılmadan uyarır)
                ["forecast-100pct"] = new BudgetNotification(
                    isEnabled: true,
                    operator: BudgetOperatorType.GreaterThan,
                    threshold: 100,
                    contactEmails: config.NotificationEmails)
                {
                    ThresholdType = ThresholdType.Forecasted,
                    ContactRoles = { "Owner" }
                }
            }
        };

        var result = await budgets.CreateOrUpdateAsync(
            Azure.WaitUntil.Completed,
            $"budget-{config.TeamName.ToLower().Replace(" ", "-")}",
            budgetData);

        _logger.LogInformation("Bütçe oluşturuldu: {BudgetName}", result.Value.Data.Name);
        return result.Value;
    }
}

public record TeamBudgetConfig
{
    public required string TeamName { get; init; }
    public decimal MonthlyLimitUsd { get; init; }
    public List<string> NotificationEmails { get; init; } = new();
}
```

---

## Best Practices

### 1. **Maliyet Görünürlüğü**
- Her kaynak için standart etiketleme politikası zorunlu tutun
- Gerçek zamanlı maliyet dashboardları tüm takımlara açık olsun
- Günlük maliyet özet raporlarını ekip liderlerine otomatik gönderin
- Anomali uyarılarını saatlik granülaritede yapılandırın

### 2. **Kapasite Yönetimi**
- Üretim iş yükleri için 14-30 günlük metrik analizi ile düzenli right-sizing yapın
- 7/24 çalışan iş yükleri için Reserved Instance kapsama oranını %70+ tutun
- Dev/Test ortamlarında mesai saatleri dışında otomatik kapatma uygulayın
- Kubernetes'te KEDA ile olay tabanlı, sıfıra ölçeklenebilir mimari kurun

### 3. **Depolama Optimizasyonu**
- Blob storage için lifecycle management politikaları tanımlayın (Hot -> Cool -> Archive)
- Snapshot ve backup saklama sürelerini iş gereksinimlerine göre sınırlayın
- Kullanılmayan diskleri haftalık olarak tarayıp silin
- Data transfer maliyetlerini azaltmak için CDN ve bölge içi iletişimi tercih edin

### 4. **Ağ Maliyeti**
- Cross-region veri transferini minimize edin
- NAT Gateway yerine Private Endpoint tercih edin (yüksek hacimde daha ucuz)
- ExpressRoute ile tahmin edilebilir bandwidth maliyeti sağlayın
- Egress maliyetlerini azaltmak için veri sıkıştırma uygulayın

## Kaynaklar

- [Azure Cost Management Docs](https://docs.microsoft.com/tr-tr/azure/cost-management-billing/)
- [Azure Pricing Calculator](https://azure.microsoft.com/tr-tr/pricing/calculator/)
- [AWS Cost Explorer API](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-api.html)
- [Azure Monitor Metrics](https://docs.microsoft.com/tr-tr/azure/azure-monitor/essentials/metrics-getting-started)
- [Azure Resource Graph](https://docs.microsoft.com/tr-tr/azure/governance/resource-graph/)
- [AWS Savings Plans](https://aws.amazon.com/savingsplans/)
- [Azure Advisor Cost Recommendations](https://docs.microsoft.com/tr-tr/azure/advisor/advisor-cost-recommendations)
- [KEDA - Kubernetes Event-driven Autoscaling](https://keda.sh/)
