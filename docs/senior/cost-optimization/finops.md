# FinOps Pratikleri

## Genel Bakış

FinOps (Financial Operations), bulut harcamalarını bir disiplin olarak yöneten ve mühendislik, finans ile iş birimleri arasında köprü kuran bir çerçevedir. FinOps Foundation tarafından tanımlanan bu çerçeve; organizasyonların bulut maliyetlerinde görünürlük kazanmasını, harcamaları optimize etmesini ve sürekli iyileştirme kültürü oluşturmasını sağlar.

FinOps'un temel felsefesi şudur: "Her ekip kendi bulut harcamalarının sahibidir." Bu yaklaşım, merkezi maliyet kontrolünden ekip bazlı finansal hesap verebilirliğe geçişi ifade eder. Senior .NET geliştiricileri yalnızca teknik implementasyon değil, maliyet sahipliği kültürünü de benimsemelidir.

## Mülakat Soruları ve Cevapları

### 1. Soru: FinOps çerçevesi nedir ve üç ana fazı nasıl işler?

**Cevap:**
FinOps Foundation'ın tanımladığı çerçeve üç döngüsel fazdan oluşur:

**Inform (Bilgilendirme):**
- Harcamaların görünür kılınması; hangi takım, proje veya servisin ne kadar harcadığının bilinmesi
- Etiketleme, dashboard'lar ve maliyet dağılım raporları bu fazın araçlarıdır
- Amaç: "Nereye para gidiyor?" sorusuna yanıt vermek

**Optimize (İyileştirme):**
- Görünür hale gelen maliyetlerin azaltılması veya verimliliğinin artırılması
- Right-sizing, reserved capacity, idle resource temizliği bu fazda gerçekleşir
- Amaç: "Mevcut değeri daha ucuza üretebilir miyiz?"

**Operate (İşletim):**
- Sürekli maliyet yönetimi kültürünün oluşturulması
- Bütçe uyarıları, governance politikaları ve FinOps KPI'larının izlenmesi
- Amaç: "Bu süreç nasıl kalıcı hale getirilir?"

Fazlar birbirini döngüsel olarak besler; Operate fazından toplanan veriler yeni Inform döngüsünü başlatır.

**Örnek Kod:**
```csharp
// FinOps olgunluk seviyelerini değerlendiren bir araç
public class FinOpsMaturityAssessor
{
    private readonly ILogger<FinOpsMaturityAssessor> _logger;

    public FinOpsMaturityAssessor(ILogger<FinOpsMaturityAssessor> logger)
    {
        _logger = logger;
    }

    public FinOpsMaturityReport AssessOrganizationMaturity(
        OrganizationFinOpsState state)
    {
        var scores = new Dictionary<FinOpsCapability, MaturityLevel>();

        // Inform fazı: Görünürlük değerlendirmesi
        scores[FinOpsCapability.CostAllocation] =
            EvaluateCostAllocationMaturity(state);
        scores[FinOpsCapability.DataAnalysis] =
            EvaluateDataAnalysisMaturity(state);
        scores[FinOpsCapability.Reporting] =
            EvaluateReportingMaturity(state);

        // Optimize fazı: İyileştirme değerlendirmesi
        scores[FinOpsCapability.RateOptimization] =
            EvaluateRateOptimizationMaturity(state);
        scores[FinOpsCapability.UsageOptimization] =
            EvaluateUsageOptimizationMaturity(state);

        // Operate fazı: Süreç değerlendirmesi
        scores[FinOpsCapability.PolicyGovernance] =
            EvaluatePolicyGovernanceMaturity(state);
        scores[FinOpsCapability.FinOpsEducation] =
            EvaluateEducationMaturity(state);

        var overallMaturity = CalculateOverallMaturity(scores);

        _logger.LogInformation(
            "FinOps olgunluk değerlendirmesi tamamlandı: {Level}",
            overallMaturity);

        return new FinOpsMaturityReport
        {
            AssessedAt = DateTime.UtcNow,
            OrganizationName = state.OrganizationName,
            CapabilityScores = scores,
            OverallMaturityLevel = overallMaturity,
            Recommendations = GenerateRecommendations(scores)
        };
    }

    private static MaturityLevel EvaluateCostAllocationMaturity(
        OrganizationFinOpsState state)
    {
        if (!state.HasTaggingPolicy) return MaturityLevel.Crawl;
        if (state.TaggingCoveragePercent < 70) return MaturityLevel.Crawl;
        if (state.TaggingCoveragePercent < 90) return MaturityLevel.Walk;
        if (state.HasAutomatedTagEnforcement && state.HasCostCenters)
            return MaturityLevel.Run;
        return MaturityLevel.Walk;
    }

    private static MaturityLevel EvaluateRateOptimizationMaturity(
        OrganizationFinOpsState state)
    {
        if (state.ReservedInstanceCoveragePercent < 20) return MaturityLevel.Crawl;
        if (state.ReservedInstanceCoveragePercent < 60) return MaturityLevel.Walk;
        if (state.ReservedInstanceCoveragePercent >= 60 &&
            state.UsesSpotForEligibleWorkloads)
            return MaturityLevel.Run;
        return MaturityLevel.Walk;
    }

    private static MaturityLevel EvaluateUsageOptimizationMaturity(
        OrganizationFinOpsState state)
    {
        if (!state.HasRightSizingProcess) return MaturityLevel.Crawl;
        if (!state.HasOrphanedResourceCleanup) return MaturityLevel.Walk;
        if (state.HasAutomatedOptimization) return MaturityLevel.Run;
        return MaturityLevel.Walk;
    }

    private static MaturityLevel EvaluateDataAnalysisMaturity(
        OrganizationFinOpsState state) =>
        state.HasRealTimeDashboard
            ? (state.HasAnomalyDetection ? MaturityLevel.Run : MaturityLevel.Walk)
            : MaturityLevel.Crawl;

    private static MaturityLevel EvaluateReportingMaturity(
        OrganizationFinOpsState state) =>
        state.HasAutomatedReports
            ? (state.HasUnitEconomicsMetrics ? MaturityLevel.Run : MaturityLevel.Walk)
            : MaturityLevel.Crawl;

    private static MaturityLevel EvaluatePolicyGovernanceMaturity(
        OrganizationFinOpsState state) =>
        state.HasBudgetAlerts
            ? (state.HasAutomatedGovernancePolicies ? MaturityLevel.Run : MaturityLevel.Walk)
            : MaturityLevel.Crawl;

    private static MaturityLevel EvaluateEducationMaturity(
        OrganizationFinOpsState state) =>
        state.EngineersTrainedPercent > 80 ? MaturityLevel.Run :
        state.EngineersTrainedPercent > 40 ? MaturityLevel.Walk :
        MaturityLevel.Crawl;

    private static MaturityLevel CalculateOverallMaturity(
        Dictionary<FinOpsCapability, MaturityLevel> scores)
    {
        var avgScore = scores.Values.Average(l => (int)l);
        return avgScore >= 2.5 ? MaturityLevel.Run :
               avgScore >= 1.5 ? MaturityLevel.Walk :
               MaturityLevel.Crawl;
    }

    private static List<string> GenerateRecommendations(
        Dictionary<FinOpsCapability, MaturityLevel> scores)
    {
        var recommendations = new List<string>();

        foreach (var (capability, level) in scores.Where(s => s.Value < MaturityLevel.Walk))
        {
            recommendations.Add(capability switch
            {
                FinOpsCapability.CostAllocation =>
                    "Etiketleme politikası oluşturun ve Azure Policy ile zorunlu hale getirin",
                FinOpsCapability.RateOptimization =>
                    "Üretim iş yükleri için Reserved Instance analizi yapın",
                FinOpsCapability.UsageOptimization =>
                    "Aylık right-sizing ve orphaned resource temizleme süreci kurun",
                FinOpsCapability.Reporting =>
                    "Gerçek zamanlı maliyet dashboard'u oluşturun",
                FinOpsCapability.PolicyGovernance =>
                    "Takım başına aylık bütçe uyarıları tanımlayın",
                FinOpsCapability.FinOpsEducation =>
                    "Mühendislere FinOps temel eğitimi verin",
                _ => "FinOps pratiklerini güçlendirin"
            });
        }

        return recommendations;
    }
}

public record OrganizationFinOpsState
{
    public required string OrganizationName { get; init; }
    public bool HasTaggingPolicy { get; init; }
    public double TaggingCoveragePercent { get; init; }
    public bool HasAutomatedTagEnforcement { get; init; }
    public bool HasCostCenters { get; init; }
    public double ReservedInstanceCoveragePercent { get; init; }
    public bool UsesSpotForEligibleWorkloads { get; init; }
    public bool HasRightSizingProcess { get; init; }
    public bool HasOrphanedResourceCleanup { get; init; }
    public bool HasAutomatedOptimization { get; init; }
    public bool HasRealTimeDashboard { get; init; }
    public bool HasAnomalyDetection { get; init; }
    public bool HasAutomatedReports { get; init; }
    public bool HasUnitEconomicsMetrics { get; init; }
    public bool HasBudgetAlerts { get; init; }
    public bool HasAutomatedGovernancePolicies { get; init; }
    public double EngineersTrainedPercent { get; init; }
}

public record FinOpsMaturityReport
{
    public DateTime AssessedAt { get; init; }
    public required string OrganizationName { get; init; }
    public Dictionary<FinOpsCapability, MaturityLevel> CapabilityScores { get; init; } = new();
    public MaturityLevel OverallMaturityLevel { get; init; }
    public List<string> Recommendations { get; init; } = new();
}

public enum MaturityLevel { Crawl = 1, Walk = 2, Run = 3 }

public enum FinOpsCapability
{
    CostAllocation,
    DataAnalysis,
    Reporting,
    RateOptimization,
    UsageOptimization,
    PolicyGovernance,
    FinOpsEducation
}
```

---

### 2. Soru: Kaynak etiketleme (tagging) stratejisi nasıl tasarlanır ve C# ile zorunlu hale getirilir?

**Cevap:**
Etiketleme stratejisi, cost allocation'ın temelidir. Etkili bir tagging stratejisi hem maliyet dağılımını hem de operasyonel yönetimi (kimin neyin sahibi olduğu, hangi ortamda çalıştığı) desteklemelidir.

Zorunlu etiket kategorileri:
- **İş birimine göre:** `cost-center`, `department`, `team`, `project`
- **Teknik bağlam:** `environment`, `application`, `component`, `version`
- **Operasyonel:** `owner`, `managed-by`, `created-by`, `expiry-date`
- **Maliyet:** `billing-code`, `budget-owner`

Azure'da etiketleme zorunluluğu Azure Policy ile, AWS'de Service Control Policies (SCP) ile uygulanır. IaC (Terraform, Bicep) araçları ile etiketler otomatik uygulanabilir.

**Örnek Kod:**
```csharp
// Etiket doğrulama ve uygulama servisi
public class TaggingComplianceService
{
    private readonly ILogger<TaggingComplianceService> _logger;

    private static readonly IReadOnlySet<string> RequiredTags = new HashSet<string>
    {
        "environment",
        "team",
        "project",
        "cost-center",
        "owner",
        "application"
    };

    private static readonly IReadOnlyDictionary<string, IReadOnlyList<string>> AllowedTagValues =
        new Dictionary<string, IReadOnlyList<string>>
        {
            ["environment"] = new[] { "production", "staging", "development", "testing" },
            ["cost-center"] = new[] { "engineering", "data", "platform", "security", "product" }
        };

    public TaggingComplianceService(ILogger<TaggingComplianceService> logger)
    {
        _logger = logger;
    }

    public TagComplianceResult ValidateResourceTags(
        string resourceId,
        IReadOnlyDictionary<string, string> tags)
    {
        var violations = new List<TagViolation>();
        var warnings = new List<string>();

        // Zorunlu etiket kontrolü
        foreach (var requiredTag in RequiredTags)
        {
            if (!tags.ContainsKey(requiredTag) || string.IsNullOrWhiteSpace(tags[requiredTag]))
            {
                violations.Add(new TagViolation(
                    TagName: requiredTag,
                    ViolationType: TagViolationType.MissingRequiredTag,
                    Message: $"Zorunlu '{requiredTag}' etiketi eksik"));
            }
        }

        // İzin verilen değer kontrolü
        foreach (var (tagName, allowedValues) in AllowedTagValues)
        {
            if (tags.TryGetValue(tagName, out var value) &&
                !allowedValues.Contains(value.ToLower()))
            {
                violations.Add(new TagViolation(
                    TagName: tagName,
                    ViolationType: TagViolationType.InvalidValue,
                    Message: $"'{tagName}' için geçersiz değer: '{value}'. " +
                             $"İzin verilenler: {string.Join(", ", allowedValues)}"));
            }
        }

        // Sona erme tarihi kontrolü
        if (tags.TryGetValue("expiry-date", out var expiryStr) &&
            DateTime.TryParse(expiryStr, out var expiryDate))
        {
            if (expiryDate < DateTime.UtcNow)
            {
                warnings.Add($"Kaynak sona erme tarihi geçmiş: {expiryDate:yyyy-MM-dd}. " +
                             "Kaynağın hâlâ gerekli olup olmadığını doğrulayın.");
            }
        }

        // Owner e-posta formatı kontrolü
        if (tags.TryGetValue("owner", out var owner) &&
            !IsValidEmail(owner))
        {
            violations.Add(new TagViolation(
                TagName: "owner",
                ViolationType: TagViolationType.InvalidFormat,
                Message: $"'owner' etiketi geçerli bir e-posta adresi olmalıdır: '{owner}'"));
        }

        var isCompliant = !violations.Any();

        if (!isCompliant)
        {
            _logger.LogWarning(
                "Etiket uyumsuzluğu tespit edildi. Kaynak: {ResourceId}, " +
                "İhlal sayısı: {Count}",
                resourceId, violations.Count);
        }

        return new TagComplianceResult
        {
            ResourceId = resourceId,
            IsCompliant = isCompliant,
            Violations = violations,
            Warnings = warnings,
            CheckedAt = DateTime.UtcNow
        };
    }

    public IReadOnlyDictionary<string, string> GenerateStandardTags(
        StandardTagRequest request)
    {
        return new Dictionary<string, string>
        {
            ["environment"] = request.Environment.ToLower(),
            ["team"] = request.TeamName.ToLower().Replace(" ", "-"),
            ["project"] = request.ProjectName.ToLower().Replace(" ", "-"),
            ["cost-center"] = request.CostCenter.ToLower(),
            ["owner"] = request.OwnerEmail.ToLower(),
            ["application"] = request.ApplicationName.ToLower().Replace(" ", "-"),
            ["created-by"] = request.CreatedBy,
            ["created-at"] = DateTime.UtcNow.ToString("yyyy-MM-dd"),
            ["managed-by"] = "terraform", // veya "bicep", "manual"
            ["billing-code"] = request.BillingCode
        };
    }

    public async Task<BatchTaggingReport> EnforceTagsOnResourceGroupAsync(
        string subscriptionId,
        string resourceGroupName,
        IReadOnlyDictionary<string, string> requiredTags,
        bool dryRun = true)
    {
        _logger.LogInformation(
            "Toplu etiket uygulama başlatılıyor: {ResourceGroup}, DryRun: {DryRun}",
            resourceGroupName, dryRun);

        // Gerçek uygulamada Azure Resource Manager API çağrısı yapılır
        // Bu örnek mantığı ve raporlamayı göstermektedir
        var results = new List<TagApplicationResult>();

        // Simüle edilmiş kaynak listesi
        var resources = new[]
        {
            new { Id = $"/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Compute/virtualMachines/vm-01", Type = "VM" },
            new { Id = $"/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Storage/storageAccounts/storage01", Type = "Storage" }
        };

        foreach (var resource in resources)
        {
            if (!dryRun)
            {
                // Gerçek etiket uygulaması burada yapılır
                _logger.LogInformation(
                    "Etiket uygulandı: {ResourceId}",
                    resource.Id);
            }

            results.Add(new TagApplicationResult
            {
                ResourceId = resource.Id,
                ResourceType = resource.Type,
                TagsApplied = requiredTags.Keys.ToList(),
                WasDryRun = dryRun,
                AppliedAt = DateTime.UtcNow
            });
        }

        return new BatchTaggingReport
        {
            ResourceGroupName = resourceGroupName,
            TotalResourcesProcessed = results.Count,
            SuccessfullyTagged = results.Count,
            Failed = 0,
            IsDryRun = dryRun,
            Results = results
        };
    }

    private static bool IsValidEmail(string email) =>
        email.Contains('@') && email.Contains('.') && email.Length > 5;
}

public record TagComplianceResult
{
    public required string ResourceId { get; init; }
    public bool IsCompliant { get; init; }
    public List<TagViolation> Violations { get; init; } = new();
    public List<string> Warnings { get; init; } = new();
    public DateTime CheckedAt { get; init; }
}

public record TagViolation(
    string TagName,
    TagViolationType ViolationType,
    string Message);

public enum TagViolationType
{
    MissingRequiredTag,
    InvalidValue,
    InvalidFormat,
    ExpiredResource
}

public record StandardTagRequest
{
    public required string Environment { get; init; }
    public required string TeamName { get; init; }
    public required string ProjectName { get; init; }
    public required string CostCenter { get; init; }
    public required string OwnerEmail { get; init; }
    public required string ApplicationName { get; init; }
    public required string CreatedBy { get; init; }
    public required string BillingCode { get; init; }
}

public record BatchTaggingReport
{
    public required string ResourceGroupName { get; init; }
    public int TotalResourcesProcessed { get; init; }
    public int SuccessfullyTagged { get; init; }
    public int Failed { get; init; }
    public bool IsDryRun { get; init; }
    public List<TagApplicationResult> Results { get; init; } = new();
}

public record TagApplicationResult
{
    public required string ResourceId { get; init; }
    public required string ResourceType { get; init; }
    public List<string> TagsApplied { get; init; } = new();
    public bool WasDryRun { get; init; }
    public DateTime AppliedAt { get; init; }
}
```

---

### 3. Soru: Showback ve Chargeback modelleri nasıl implemente edilir?

**Cevap:**
**Showback:** Maliyetleri iş birimlerine görünür kılar ancak fiili fatura oluşturmaz. Kültürel farkındalık yaratmak ve maliyet sahipliği alışkanlığı kazandırmak için idealdir. FinOps yolculuğunun ilk aşamasında kullanılır.

**Chargeback:** Maliyetleri iş birimlerine fiilen fatura eder. P&L hesapları ayrı tutulan departmanlar veya iç faturalama modelleri olan organizasyonlarda uygulanır. Yüksek hesap verebilirlik sağlar; ancak sürtüşme yaratabilir.

**Karma model:** Paylaşılan servisler (shared Kubernetes cluster, merkezi log servisi) için maliyet dağıtım (cost distribution) modeli kullanılır. Kullanım oranına göre maliyet paylaştırılır.

**Örnek Kod:**
```csharp
public class CostAllocationEngine
{
    private readonly ILogger<CostAllocationEngine> _logger;
    private readonly ICostDataRepository _costRepository;

    public CostAllocationEngine(
        ILogger<CostAllocationEngine> logger,
        ICostDataRepository costRepository)
    {
        _logger = logger;
        _costRepository = costRepository;
    }

    public async Task<ShowbackReport> GenerateShowbackReportAsync(
        DateTimeOffset periodStart,
        DateTimeOffset periodEnd)
    {
        _logger.LogInformation(
            "Showback raporu oluşturuluyor: {Start:yyyy-MM} - {End:yyyy-MM}",
            periodStart, periodEnd);

        var rawCosts = await _costRepository.GetCostsByPeriodAsync(periodStart, periodEnd);

        // Doğrudan atanabilir maliyetler (etiketlenmiş kaynaklar)
        var directCosts = rawCosts
            .Where(c => c.TeamTag != null)
            .GroupBy(c => c.TeamTag!)
            .ToDictionary(
                g => g.Key,
                g => g.Sum(c => c.AmountUsd));

        // Paylaşılan servis maliyetleri (etiketlenmemiş veya shared)
        var sharedCosts = rawCosts
            .Where(c => c.TeamTag == null || c.IsSharedService)
            .Sum(c => c.AmountUsd);

        // Paylaşılan maliyetlerin kullanım oranına göre dağıtımı
        var teamUsageWeights = await _costRepository
            .GetTeamUsageWeightsAsync(periodStart, periodEnd);

        var allocatedSharedCosts = DistributeSharedCosts(
            sharedCosts, teamUsageWeights);

        // Toplam showback raporu oluştur
        var teamReports = new List<TeamCostSummary>();

        foreach (var team in directCosts.Keys.Union(allocatedSharedCosts.Keys))
        {
            directCosts.TryGetValue(team, out var direct);
            allocatedSharedCosts.TryGetValue(team, out var shared);

            teamReports.Add(new TeamCostSummary
            {
                TeamName = team,
                DirectCostUsd = direct,
                AllocatedSharedCostUsd = shared,
                TotalCostUsd = direct + shared,
                Period = periodStart.ToString("yyyy-MM"),
                CostBreakdown = await GetTeamCostBreakdownAsync(
                    team, periodStart, periodEnd)
            });
        }

        var totalCost = teamReports.Sum(t => t.TotalCostUsd);

        _logger.LogInformation(
            "Showback raporu tamamlandı. Toplam: {Total:C}, " +
            "Takım sayısı: {TeamCount}, Paylaşılan maliyet: {Shared:C}",
            totalCost, teamReports.Count, sharedCosts);

        return new ShowbackReport
        {
            PeriodStart = periodStart,
            PeriodEnd = periodEnd,
            TotalCostUsd = totalCost,
            TotalDirectCostUsd = directCosts.Values.Sum(),
            TotalSharedCostUsd = sharedCosts,
            TeamSummaries = teamReports.OrderByDescending(t => t.TotalCostUsd).ToList()
        };
    }

    public async Task<ChargebackInvoice> GenerateChargebackInvoiceAsync(
        string teamName,
        DateTimeOffset periodStart,
        DateTimeOffset periodEnd,
        string currency = "USD")
    {
        _logger.LogInformation(
            "Chargeback faturası oluşturuluyor: {Team}, {Period:yyyy-MM}",
            teamName, periodStart);

        var teamCosts = await _costRepository
            .GetCostsByTeamAndPeriodAsync(teamName, periodStart, periodEnd);

        var lineItems = teamCosts
            .GroupBy(c => new { c.ServiceName, c.Category })
            .Select(g => new InvoiceLineItem
            {
                ServiceName = g.Key.ServiceName,
                Category = g.Key.Category,
                Quantity = g.Sum(c => c.UsageQuantity),
                UnitCostUsd = g.Average(c => c.UnitPrice),
                TotalCostUsd = g.Sum(c => c.AmountUsd),
                Description = BuildLineItemDescription(g.Key.ServiceName, g.ToList())
            })
            .OrderByDescending(l => l.TotalCostUsd)
            .ToList();

        var totalCost = lineItems.Sum(l => l.TotalCostUsd);

        // Maliyet merkezi kodu ile fatura oluştur
        return new ChargebackInvoice
        {
            InvoiceNumber = $"CLOUD-{periodStart:yyyyMM}-{teamName.ToUpper().Replace(" ", "")}",
            TeamName = teamName,
            PeriodStart = periodStart,
            PeriodEnd = periodEnd,
            Currency = currency,
            SubtotalUsd = totalCost,
            TaxUsd = 0, // Dahili faturalama genellikle vergisiz
            TotalUsd = totalCost,
            LineItems = lineItems,
            GeneratedAt = DateTime.UtcNow,
            DueDate = periodEnd.AddDays(30),
            Notes = "Bu fatura dahili bulut maliyet dağıtımı içindir."
        };
    }

    private static Dictionary<string, decimal> DistributeSharedCosts(
        decimal totalSharedCost,
        IReadOnlyDictionary<string, double> usageWeights)
    {
        if (!usageWeights.Any()) return new Dictionary<string, decimal>();

        var totalWeight = usageWeights.Values.Sum();
        return usageWeights.ToDictionary(
            kv => kv.Key,
            kv => totalSharedCost * (decimal)(kv.Value / totalWeight));
    }

    private async Task<List<CostCategory>> GetTeamCostBreakdownAsync(
        string teamName,
        DateTimeOffset start,
        DateTimeOffset end)
    {
        var costs = await _costRepository.GetCostsByTeamAndPeriodAsync(teamName, start, end);
        return costs
            .GroupBy(c => c.Category)
            .Select(g => new CostCategory
            {
                Category = g.Key,
                TotalCostUsd = g.Sum(c => c.AmountUsd),
                PercentOfTotal = 0 // Hesaplanır
            })
            .ToList();
    }

    private static string BuildLineItemDescription(
        string serviceName, List<CostRecord> records) =>
        $"{serviceName}: {records.Count} kaynak, " +
        $"{records.Sum(r => r.UsageQuantity):F1} birim kullanım";
}

public record ShowbackReport
{
    public DateTimeOffset PeriodStart { get; init; }
    public DateTimeOffset PeriodEnd { get; init; }
    public decimal TotalCostUsd { get; init; }
    public decimal TotalDirectCostUsd { get; init; }
    public decimal TotalSharedCostUsd { get; init; }
    public List<TeamCostSummary> TeamSummaries { get; init; } = new();
}

public record TeamCostSummary
{
    public required string TeamName { get; init; }
    public decimal DirectCostUsd { get; init; }
    public decimal AllocatedSharedCostUsd { get; init; }
    public decimal TotalCostUsd { get; init; }
    public required string Period { get; init; }
    public List<CostCategory> CostBreakdown { get; init; } = new();
}

public record ChargebackInvoice
{
    public required string InvoiceNumber { get; init; }
    public required string TeamName { get; init; }
    public DateTimeOffset PeriodStart { get; init; }
    public DateTimeOffset PeriodEnd { get; init; }
    public required string Currency { get; init; }
    public decimal SubtotalUsd { get; init; }
    public decimal TaxUsd { get; init; }
    public decimal TotalUsd { get; init; }
    public List<InvoiceLineItem> LineItems { get; init; } = new();
    public DateTime GeneratedAt { get; init; }
    public DateTimeOffset DueDate { get; init; }
    public required string Notes { get; init; }
}

public record InvoiceLineItem
{
    public required string ServiceName { get; init; }
    public required string Category { get; init; }
    public double Quantity { get; init; }
    public decimal UnitCostUsd { get; init; }
    public decimal TotalCostUsd { get; init; }
    public required string Description { get; init; }
}

public record CostCategory
{
    public required string Category { get; init; }
    public decimal TotalCostUsd { get; init; }
    public double PercentOfTotal { get; init; }
}

// Veri modelleri
public record CostRecord
{
    public required string ServiceName { get; init; }
    public required string Category { get; init; }
    public string? TeamTag { get; init; }
    public bool IsSharedService { get; init; }
    public decimal AmountUsd { get; init; }
    public double UsageQuantity { get; init; }
    public decimal UnitPrice { get; init; }
}

public interface ICostDataRepository
{
    Task<List<CostRecord>> GetCostsByPeriodAsync(
        DateTimeOffset start, DateTimeOffset end);
    Task<List<CostRecord>> GetCostsByTeamAndPeriodAsync(
        string teamName, DateTimeOffset start, DateTimeOffset end);
    Task<IReadOnlyDictionary<string, double>> GetTeamUsageWeightsAsync(
        DateTimeOffset start, DateTimeOffset end);
}
```

---

### 4. Soru: Unit economics metrikleri nasıl hesaplanır ve takip edilir?

**Cevap:**
Unit economics, maliyet etkinliğini ölçmek için kullanılan iş metriklerini bulut maliyetleriyle ilişkilendirir. Teknik maliyetleri iş değerine bağlar ve ürün kararlarını yönlendirmede kritik rol oynar.

Yaygın unit economics metrikleri:
- **API isteği başına maliyet:** Toplam altyapı maliyeti / Toplam API isteği sayısı
- **Müşteri başına aylık maliyet:** Toplam maliyet / Aktif müşteri sayısı
- **İşlem başına maliyet:** Toplam maliyet / Tamamlanan işlem sayısı
- **1000 mesaj başına maliyet:** Toplam mesaj servisi maliyeti / Mesaj sayısı (binde)

Bu metrikler zaman içindeki eğilimlerle izlenerek verimlilik artışı veya maliyet artışı tespit edilir.

**Örnek Kod:**
```csharp
public class UnitEconomicsCalculator
{
    private readonly ILogger<UnitEconomicsCalculator> _logger;
    private readonly ICostDataRepository _costRepository;
    private readonly IBusinessMetricsRepository _metricsRepository;

    public UnitEconomicsCalculator(
        ILogger<UnitEconomicsCalculator> logger,
        ICostDataRepository costRepository,
        IBusinessMetricsRepository metricsRepository)
    {
        _logger = logger;
        _costRepository = costRepository;
        _metricsRepository = metricsRepository;
    }

    public async Task<UnitEconomicsReport> CalculateMonthlyMetricsAsync(
        int year, int month)
    {
        var periodStart = new DateTimeOffset(year, month, 1, 0, 0, 0, TimeSpan.Zero);
        var periodEnd = periodStart.AddMonths(1).AddSeconds(-1);

        _logger.LogInformation(
            "Unit economics hesaplanıyor: {Year}-{Month:D2}",
            year, month);

        var costs = await _costRepository.GetCostsByPeriodAsync(periodStart, periodEnd);
        var totalCost = costs.Sum(c => c.AmountUsd);

        var businessMetrics = await _metricsRepository.GetMetricsAsync(periodStart, periodEnd);

        // API isteği başına maliyet
        decimal costPerApiRequest = businessMetrics.TotalApiRequests > 0
            ? totalCost / (decimal)businessMetrics.TotalApiRequests
            : 0;

        // Müşteri başına aylık maliyet
        decimal costPerActiveCustomer = businessMetrics.ActiveCustomers > 0
            ? totalCost / businessMetrics.ActiveCustomers
            : 0;

        // İşlem başına maliyet
        decimal costPerTransaction = businessMetrics.TotalTransactions > 0
            ? totalCost / (decimal)businessMetrics.TotalTransactions
            : 0;

        // 1000 mesaj başına maliyet
        var messagingCost = costs
            .Where(c => c.Category == "Messaging")
            .Sum(c => c.AmountUsd);
        decimal costPer1000Messages = businessMetrics.TotalMessagesProcessed > 0
            ? messagingCost / ((decimal)businessMetrics.TotalMessagesProcessed / 1000)
            : 0;

        // Önceki ay ile karşılaştırma
        var previousPeriodStart = periodStart.AddMonths(-1);
        var previousPeriodEnd = periodStart.AddSeconds(-1);
        var previousMetrics = await _metricsRepository
            .GetMetricsAsync(previousPeriodStart, previousPeriodEnd);
        var previousCosts = await _costRepository
            .GetCostsByPeriodAsync(previousPeriodStart, previousPeriodEnd);

        var previousTotalCost = previousCosts.Sum(c => c.AmountUsd);
        decimal previousCostPerCustomer = previousMetrics.ActiveCustomers > 0
            ? previousTotalCost / previousMetrics.ActiveCustomers
            : 0;

        var customerCostTrend = previousCostPerCustomer > 0
            ? (costPerActiveCustomer - previousCostPerCustomer) / previousCostPerCustomer * 100
            : 0;

        var report = new UnitEconomicsReport
        {
            Period = $"{year}-{month:D2}",
            TotalInfrastructureCostUsd = totalCost,
            Metrics = new UnitCostMetrics
            {
                CostPerApiRequestUsd = costPerApiRequest,
                CostPerActiveCustomerUsd = costPerActiveCustomer,
                CostPerTransactionUsd = costPerTransaction,
                CostPer1000MessagesUsd = costPer1000Messages
            },
            BusinessVolume = new BusinessVolumeSnapshot
            {
                TotalApiRequests = businessMetrics.TotalApiRequests,
                ActiveCustomers = businessMetrics.ActiveCustomers,
                TotalTransactions = businessMetrics.TotalTransactions,
                TotalMessagesProcessed = businessMetrics.TotalMessagesProcessed
            },
            MonthOverMonthTrends = new UnitCostTrends
            {
                CostPerCustomerChangePercent = customerCostTrend,
                TotalCostChangePercent = previousTotalCost > 0
                    ? (totalCost - previousTotalCost) / previousTotalCost * 100
                    : 0
            }
        };

        _logger.LogInformation(
            "Unit economics: Müşteri başına {PerCustomer:C}, " +
            "API isteği başına {PerRequest:C5}, " +
            "Aylık değişim: {Trend:+0.0;-0.0;0}%",
            costPerActiveCustomer,
            costPerApiRequest,
            customerCostTrend);

        return report;
    }

    public async Task<List<UnitEconomicsTrend>> GetTrendAnalysisAsync(int months = 12)
    {
        var trends = new List<UnitEconomicsTrend>();
        var now = DateTime.UtcNow;

        for (int i = months - 1; i >= 0; i--)
        {
            var targetDate = now.AddMonths(-i);
            var report = await CalculateMonthlyMetricsAsync(
                targetDate.Year, targetDate.Month);

            trends.Add(new UnitEconomicsTrend
            {
                Period = report.Period,
                CostPerCustomerUsd = report.Metrics.CostPerActiveCustomerUsd,
                CostPerApiRequestUsd = report.Metrics.CostPerApiRequestUsd,
                CostPerTransactionUsd = report.Metrics.CostPerTransactionUsd,
                TotalCostUsd = report.TotalInfrastructureCostUsd,
                ActiveCustomers = report.BusinessVolume.ActiveCustomers
            });
        }

        return trends;
    }
}

public record UnitEconomicsReport
{
    public required string Period { get; init; }
    public decimal TotalInfrastructureCostUsd { get; init; }
    public required UnitCostMetrics Metrics { get; init; }
    public required BusinessVolumeSnapshot BusinessVolume { get; init; }
    public required UnitCostTrends MonthOverMonthTrends { get; init; }
}

public record UnitCostMetrics
{
    public decimal CostPerApiRequestUsd { get; init; }
    public decimal CostPerActiveCustomerUsd { get; init; }
    public decimal CostPerTransactionUsd { get; init; }
    public decimal CostPer1000MessagesUsd { get; init; }
}

public record BusinessVolumeSnapshot
{
    public long TotalApiRequests { get; init; }
    public decimal ActiveCustomers { get; init; }
    public long TotalTransactions { get; init; }
    public long TotalMessagesProcessed { get; init; }
}

public record UnitCostTrends
{
    public decimal CostPerCustomerChangePercent { get; init; }
    public decimal TotalCostChangePercent { get; init; }
}

public record UnitEconomicsTrend
{
    public required string Period { get; init; }
    public decimal CostPerCustomerUsd { get; init; }
    public decimal CostPerApiRequestUsd { get; init; }
    public decimal CostPerTransactionUsd { get; init; }
    public decimal TotalCostUsd { get; init; }
    public decimal ActiveCustomers { get; init; }
}

public record BusinessMetrics
{
    public long TotalApiRequests { get; init; }
    public decimal ActiveCustomers { get; init; }
    public long TotalTransactions { get; init; }
    public long TotalMessagesProcessed { get; init; }
}

public interface IBusinessMetricsRepository
{
    Task<BusinessMetrics> GetMetricsAsync(DateTimeOffset start, DateTimeOffset end);
}
```

---

### 5. Soru: FinOps kültürünü organizasyona nasıl yaygınlaştırırsınız?

**Cevap:**
FinOps kültürünün organizasyona yayılması yalnızca araç ve süreç değil, zihniyet dönüşümü gerektirir. Başarılı FinOps kültürünün unsurları:

**Yapısal değişiklikler:**
- FinOps pratisyeni rolü veya merkezi FinOps ekibi oluşturulması
- Her takımın kendi maliyet dashboarduna erişimi
- Sprint review'lerde maliyet metriklerinin düzenli gözden geçirilmesi
- Maliyet optimizasyonunun mühendislik KPI'larına eklenmesi

**Davranışsal değişiklikler:**
- "Cloud-first, cost-aware" tasarım prensiplerinin benimsenmesi
- Mühendislerin mimari kararlarında maliyet etkisini hesaba katması
- Başarılı optimizasyon hikayelerinin paylaşılması
- Maliyet israfının kişiselleştirilmeden, sistem sorunu olarak ele alınması

**Teknik enabler'lar:**
- Self-service maliyet görünürlük araçları
- Otomatik anomali bildirimleri
- IaC şablonlarında maliyet tahmini
- CI/CD pipeline'larına entegre maliyet kontrolleri

**Örnek Kod:**
```csharp
// FinOps kültürünü destekleyen maliyet farkındalık servisi
// ASP.NET Core middleware olarak kullanılabilir
public class CostAwarenessMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ICostTracker _costTracker;
    private readonly ILogger<CostAwarenessMiddleware> _logger;

    // Farklı operasyonlar için yaklaşık birim maliyetler (USD)
    private static readonly IReadOnlyDictionary<string, decimal> OperationCosts =
        new Dictionary<string, decimal>
        {
            ["database-query"] = 0.000001m,     // 1 DB sorgusu
            ["storage-read"] = 0.000004m,        // 1 GB okuma
            ["storage-write"] = 0.000018m,       // 1 GB yazma
            ["external-api-call"] = 0.0001m,     // Dış API çağrısı
            ["cache-miss"] = 0.000002m,          // Cache miss -> DB
            ["message-publish"] = 0.0000004m     // Mesaj kuyruğu
        };

    public CostAwarenessMiddleware(
        RequestDelegate next,
        ICostTracker costTracker,
        ILogger<CostAwarenessMiddleware> logger)
    {
        _next = next;
        _costTracker = costTracker;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var requestStart = DateTime.UtcNow;
        var initialCostEstimate = await _costTracker.GetCurrentRequestCostAsync();

        await _next(context);

        var duration = DateTime.UtcNow - requestStart;
        var finalCostEstimate = await _costTracker.GetCurrentRequestCostAsync();
        var requestCost = finalCostEstimate - initialCostEstimate;

        // Geliştirme ortamında maliyet bilgisini response header'a ekle
        if (context.Request.Headers.ContainsKey("X-Show-Cost-Estimate"))
        {
            context.Response.Headers["X-Estimated-Request-Cost-USD"] =
                requestCost.ToString("F8");
            context.Response.Headers["X-Request-Duration-Ms"] =
                duration.TotalMilliseconds.ToString("F0");
        }

        // Yüksek maliyetli istekleri logla
        if (requestCost > 0.001m) // 0.1 cent eşiği
        {
            _logger.LogWarning(
                "Yüksek maliyetli istek tespit edildi. " +
                "Path: {Path}, Cost: {Cost:F6} USD, Duration: {Duration:F0}ms",
                context.Request.Path,
                requestCost,
                duration.TotalMilliseconds);
        }

        // Maliyet metriği olarak kaydet (Prometheus/Application Insights)
        await _costTracker.RecordRequestCostAsync(new RequestCostRecord
        {
            Path = context.Request.Path,
            Method = context.Request.Method,
            StatusCode = context.Response.StatusCode,
            EstimatedCostUsd = requestCost,
            DurationMs = duration.TotalMilliseconds,
            RecordedAt = requestStart
        });
    }
}

// Extension method ile kolay kayıt
public static class CostAwarenessMiddlewareExtensions
{
    public static IApplicationBuilder UseCostAwareness(
        this IApplicationBuilder builder) =>
        builder.UseMiddleware<CostAwarenessMiddleware>();
}

// FinOps KPI dashboard servisi
public class FinOpsKpiDashboardService
{
    private readonly ILogger<FinOpsKpiDashboardService> _logger;
    private readonly ICostDataRepository _costRepository;
    private readonly IBusinessMetricsRepository _metricsRepository;

    public FinOpsKpiDashboardService(
        ILogger<FinOpsKpiDashboardService> logger,
        ICostDataRepository costRepository,
        IBusinessMetricsRepository metricsRepository)
    {
        _logger = logger;
        _costRepository = costRepository;
        _metricsRepository = metricsRepository;
    }

    public async Task<FinOpsKpiSnapshot> GetCurrentKpisAsync()
    {
        var now = DateTimeOffset.UtcNow;
        var monthStart = new DateTimeOffset(now.Year, now.Month, 1, 0, 0, 0, TimeSpan.Zero);

        var costs = await _costRepository.GetCostsByPeriodAsync(monthStart, now);
        var totalMtdCost = costs.Sum(c => c.AmountUsd);

        // Etiketleme kapsamı hesapla (etiketlenmiş vs. toplam kaynak)
        var taggedCosts = costs.Where(c => c.TeamTag != null).Sum(c => c.AmountUsd);
        var taggingCoverage = totalMtdCost > 0
            ? taggedCosts / totalMtdCost * 100
            : 0;

        return new FinOpsKpiSnapshot
        {
            GeneratedAt = DateTime.UtcNow,
            MonthToDateCostUsd = totalMtdCost,
            TaggingCoveragePercent = (double)taggingCoverage,
            // Diğer KPI'lar gerçek verilerle doldurulur
            ReservedInstanceCoveragePercent = 72.5,
            OrphanedResourceCount = 3,
            OrphanedResourceEstimatedMonthlyCostUsd = 87.50m,
            BudgetUtilizationPercent = 68.3,
            FinOpsMaturityLevel = "Walk"
        };
    }
}

public record FinOpsKpiSnapshot
{
    public DateTime GeneratedAt { get; init; }
    public decimal MonthToDateCostUsd { get; init; }
    public double TaggingCoveragePercent { get; init; }
    public double ReservedInstanceCoveragePercent { get; init; }
    public int OrphanedResourceCount { get; init; }
    public decimal OrphanedResourceEstimatedMonthlyCostUsd { get; init; }
    public double BudgetUtilizationPercent { get; init; }
    public required string FinOpsMaturityLevel { get; init; }
}

public record RequestCostRecord
{
    public required string Path { get; init; }
    public required string Method { get; init; }
    public int StatusCode { get; init; }
    public decimal EstimatedCostUsd { get; init; }
    public double DurationMs { get; init; }
    public DateTime RecordedAt { get; init; }
}

public interface ICostTracker
{
    Task<decimal> GetCurrentRequestCostAsync();
    Task RecordRequestCostAsync(RequestCostRecord record);
}
```

---

### 6. Soru: FinOps governance politikaları ve otomasyonu nasıl kurulur?

**Cevap:**
FinOps governance, maliyetleri proaktif olarak yönetmek için politikalar, guardrail'ler ve otomasyonlar bütünüdür. Manuel müdahale gerektirmeden maliyet israfını önler.

Temel governance bileşenleri:
- **Azure Policy / AWS Service Control Policies:** Etiketleme zorunluluğu, izin verilen bölgeler, VM boyutu kısıtlamaları
- **Otomatik kapatma:** Dev/Test ortamlarının mesai dışında otomatik kapatılması
- **Bütçe aksiyon grupları:** Bütçe aşımında webhook tetikleme veya kaynak durdurma
- **Cost anomaly detection:** Beklenmedik maliyet artışlarında otomatik bildirim

**Örnek Kod:**
```csharp
public class FinOpsGovernanceAutomation
{
    private readonly ILogger<FinOpsGovernanceAutomation> _logger;
    private readonly INotificationService _notificationService;

    // İzin verilen VM boyutları (governance politikası)
    private static readonly IReadOnlySet<string> AllowedVmSizes =
        new HashSet<string>
        {
            // Dev/Test için izin verilenlerin listesi
            "Standard_B1s", "Standard_B2s", "Standard_B4ms",
            // Üretim için
            "Standard_D2s_v3", "Standard_D4s_v3", "Standard_D8s_v3",
            "Standard_D16s_v3", "Standard_E4s_v3", "Standard_E8s_v3"
        };

    // Maliyet limitlerini aşan ortam/boyut kombinasyonları
    private static readonly IReadOnlyDictionary<string, decimal> EnvironmentMonthlyLimits =
        new Dictionary<string, decimal>
        {
            ["development"] = 500m,
            ["staging"] = 1500m,
            ["production"] = 10000m
        };

    public FinOpsGovernanceAutomation(
        ILogger<FinOpsGovernanceAutomation> logger,
        INotificationService notificationService)
    {
        _logger = logger;
        _notificationService = notificationService;
    }

    public async Task<GovernanceCheckResult> RunGovernanceChecksAsync(
        ResourceProvisioningRequest request)
    {
        var violations = new List<GovernanceViolation>();

        // VM boyutu kontrolü
        if (request.VmSize != null &&
            !AllowedVmSizes.Contains(request.VmSize))
        {
            violations.Add(new GovernanceViolation(
                Rule: "allowed-vm-sizes",
                Severity: ViolationSeverity.Block,
                Message: $"VM boyutu '{request.VmSize}' izin verilen listesinde değil. " +
                         $"İzin verilenler: {string.Join(", ", AllowedVmSizes.Take(5))}...",
                SuggestedFix: "Standart_D4s_v3 veya daha küçük bir boyut seçin"));
        }

        // Zorunlu etiket kontrolü
        var requiredTags = new[] { "environment", "team", "cost-center", "owner" };
        foreach (var tag in requiredTags)
        {
            if (!request.Tags.ContainsKey(tag))
            {
                violations.Add(new GovernanceViolation(
                    Rule: "required-tags",
                    Severity: ViolationSeverity.Block,
                    Message: $"Zorunlu '{tag}' etiketi eksik",
                    SuggestedFix: $"'{tag}' etiketini ekleyin"));
            }
        }

        // Ortam maliyet limiti kontrolü
        if (request.Tags.TryGetValue("environment", out var env) &&
            EnvironmentMonthlyLimits.TryGetValue(env.ToLower(), out var limit))
        {
            var estimatedMonthlyCost = EstimateResourceMonthlyCost(request);
            if (estimatedMonthlyCost > limit * 0.1m) // Tek kaynak limitin %10'unu geçemez
            {
                violations.Add(new GovernanceViolation(
                    Rule: "resource-cost-limit",
                    Severity: ViolationSeverity.Warn,
                    Message: $"Kaynak tahmini aylık maliyeti ({estimatedMonthlyCost:C}) " +
                             $"{env} ortamı için yüksek",
                    SuggestedFix: "Daha küçük bir kaynak boyutu değerlendirin"));
            }
        }

        // Üretim dışı ortamlarda yüksek kullanılabilirlik kontrolü
        if (request.Tags.TryGetValue("environment", out var envCheck) &&
            envCheck.ToLower() != "production" &&
            request.RequestsHighAvailability)
        {
            violations.Add(new GovernanceViolation(
                Rule: "ha-non-production",
                Severity: ViolationSeverity.Warn,
                Message: $"{envCheck} ortamında yüksek kullanılabilirlik gereksiz maliyet yaratır",
                SuggestedFix: "Üretim dışı ortamlar için tek instance kullanın"));
        }

        var blockingViolations = violations
            .Where(v => v.Severity == ViolationSeverity.Block)
            .ToList();

        if (blockingViolations.Any())
        {
            _logger.LogWarning(
                "Governance ihlali: {Resource}, İhlal sayısı: {Count}",
                request.ResourceName, blockingViolations.Count);

            await _notificationService.SendGovernanceAlertAsync(
                recipientEmail: request.Tags.GetValueOrDefault("owner", "finops@company.com"),
                resourceName: request.ResourceName,
                violations: blockingViolations);
        }

        return new GovernanceCheckResult
        {
            IsApproved = !blockingViolations.Any(),
            Violations = violations,
            CheckedAt = DateTime.UtcNow,
            ResourceName = request.ResourceName
        };
    }

    private static decimal EstimateResourceMonthlyCost(
        ResourceProvisioningRequest request) =>
        request.VmSize switch
        {
            "Standard_D2s_v3" => 100m,
            "Standard_D4s_v3" => 200m,
            "Standard_D8s_v3" => 400m,
            "Standard_D16s_v3" => 800m,
            _ => 150m
        };
}

public record ResourceProvisioningRequest
{
    public required string ResourceName { get; init; }
    public string? VmSize { get; init; }
    public Dictionary<string, string> Tags { get; init; } = new();
    public bool RequestsHighAvailability { get; init; }
    public required string RequestedBy { get; init; }
}

public record GovernanceCheckResult
{
    public bool IsApproved { get; init; }
    public List<GovernanceViolation> Violations { get; init; } = new();
    public DateTime CheckedAt { get; init; }
    public required string ResourceName { get; init; }
}

public record GovernanceViolation(
    string Rule,
    ViolationSeverity Severity,
    string Message,
    string SuggestedFix);

public enum ViolationSeverity { Info, Warn, Block }

public interface INotificationService
{
    Task SendGovernanceAlertAsync(
        string recipientEmail,
        string resourceName,
        List<GovernanceViolation> violations);
}
```

---

## Best Practices

### 1. **Inform Fazı: Görünürlük Önce**
- FinOps yolculuğuna etiketleme politikası ve maliyet dağılım raporu oluşturarak başlayın
- İlk 30 günde hedef: tüm harcamaların %80'ini takımlara atayabilmek
- Takım liderlerine haftalık otomatik maliyet özeti gönderin
- Anomali tespiti için ilk hafta baseline metriklerini toplayın

### 2. **Optimize Fazı: Hızlı Kazanımlar**
- İlk 60 günde: boşta çalışan dev/test VM'lerini kapatın (kolay tasarruf)
- Bağlı olmayan diskleri ve kullanılmayan IP'leri temizleyin
- Dev/Test ortamları için mesai dışı otomatik kapatma politikası kurun
- Üretim iş yüklerinin %50'si için Reserved Instance analizi yapın

### 3. **Operate Fazı: Sürekli İyileştirme**
- Aylık FinOps review toplantısı planlayın
- Unit economics metriklerini ürün roadmap kararlarına dahil edin
- Mühendisleri FinOps Certified Practitioner eğitimine yönlendirin
- Her çeyrek için maliyet optimizasyon hedefi belirleyin

### 4. **Governance ve Otomasyon**
- IaC şablonlarında maliyet tahmini adımı ekleyin
- Pull request sürecine maliyet etkisi analizi entegre edin
- Azure Policy ile etiketleme ve bölge kısıtlamalarını zorunlu tutun
- FinOps KPI'larını mühendislik performans değerlendirmelerine ekleyin

## Kaynaklar

- [FinOps Foundation](https://www.finops.org/)
- [FinOps Framework](https://www.finops.org/framework/)
- [FinOps Certified Practitioner](https://learn.finops.org/)
- [Azure Cost Management Best Practices](https://docs.microsoft.com/tr-tr/azure/cost-management-billing/costs/cost-mgt-best-practices)
- [AWS Well-Architected Cost Optimization](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html)
- [Cloud FinOps (O'Reilly Book)](https://www.oreilly.com/library/view/cloud-finops/9781492054610/)
- [Azure FinOps Review](https://docs.microsoft.com/tr-tr/azure/architecture/framework/cost/overview)
- [CNCF FinOps Landscape](https://landscape.cncf.io/card-mode?category=finops&grouping=category)
