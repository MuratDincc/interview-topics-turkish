# RPO ve RTO

## Genel Bakış

RPO (Recovery Point Objective - Kurtarma Noktası Hedefi) ve RTO (Recovery Time Objective - Kurtarma Süresi Hedefi), felaket kurtarma planlamasının iki temel metriğidir. RPO, bir kesinti anında kaybedilebilecek maksimum veri miktarını zaman cinsinden ifade ederken; RTO, sistemin kesinti sonrasında yeniden işler hale gelmesi için geçmesi gereken maksimum süreyi belirtir. Bu iki metrik, yedekleme stratejileri, replikasyon politikaları ve altyapı yatırımlarının boyutlandırılmasında belirleyici rol oynar.

Senior bir .NET geliştiricisi olarak RPO ve RTO değerlerini iş gereksinimleriyle uyumlu biçimde hesaplamak, uygun yedekleme stratejilerini tasarlamak ve C# tabanlı otomasyon çözümleri geliştirmek kritik yetkinliklerdir.

## Temel Kavramlar

### RPO (Recovery Point Objective) Nedir?

RPO, veri kaybının kabul edilebilir üst sınırıdır. Örneğin bir e-ticaret uygulamasında RPO = 15 dakika ise sistem en fazla 15 dakikalık sipariş verisini kaybedebilir; bu, yedeklerin en az 15 dakikada bir alınması gerektiği anlamına gelir.

| RPO Değeri | Yedekleme Stratejisi | Maliyet |
|---|---|---|
| 0 dakika | Senkron replikasyon | Çok Yüksek |
| 1-15 dakika | Asenkron replikasyon + log shipping | Yüksek |
| 1 saat | Saatlik artımlı yedekleme | Orta |
| 4-8 saat | Birkaç saatlik artımlı yedekleme | Düşük |
| 24 saat | Günlük tam yedekleme | Çok Düşük |

### RTO (Recovery Time Objective) Nedir?

RTO, kesinti sonrasında sistemin yeniden hizmet vermesi için geçmesi gereken maksimum süredir. Örneğin RTO = 2 saat ise sistem en fazla 2 saat içinde tam çalışır duruma getirilmelidir. RTO değeri; failover mekanizmaları, altyapı hazırlığı ve kurtarma prosedürlerinin otomasyonu ile doğrudan ilişkilidir.

| RTO Değeri | Standby Tipi | Açıklama |
|---|---|---|
| Saniyeler | Hot Standby / Active-Active | Anlık otomatik failover |
| Dakikalar | Warm Standby | Kısa hazırlık süresi |
| 1-4 saat | Yedek sunucu kurulumu | Manuel adımlar gerekir |
| 4-24 saat | Cold Standby | Uzun hazırlık süreci |
| 24+ saat | Fiziksel ortam kurtarma | Donanım temin süreci dahil |

### RPO ve RTO Hesaplama

```csharp
public class DisasterRecoveryMetrics
{
    /// <summary>
    /// İş gereksinimlerine göre RPO'yu hesaplar ve uygun yedekleme aralığını belirler.
    /// </summary>
    public static TimeSpan CalculateBackupInterval(TimeSpan rpo)
    {
        // Yedekleme aralığı RPO'dan küçük ya da eşit olmalıdır.
        // Güvenlik marjı olarak RPO'nun %80'i alınır.
        return TimeSpan.FromSeconds(rpo.TotalSeconds * 0.8);
    }

    /// <summary>
    /// Mevcut yedekleme politikasının RPO hedefini karşılayıp karşılamadığını doğrular.
    /// </summary>
    public static bool ValidateRpo(
        TimeSpan targetRpo,
        DateTime lastBackupTime,
        DateTime currentTime)
    {
        var timeSinceLastBackup = currentTime - lastBackupTime;
        return timeSinceLastBackup <= targetRpo;
    }

    /// <summary>
    /// Kurtarma süresini hesaplar ve RTO hedefiyle karşılaştırır.
    /// </summary>
    public static RecoveryAssessment AssessRecovery(
        TimeSpan targetRto,
        TimeSpan estimatedRecoveryTime)
    {
        return new RecoveryAssessment
        {
            TargetRto = targetRto,
            EstimatedRecoveryTime = estimatedRecoveryTime,
            IsCompliant = estimatedRecoveryTime <= targetRto,
            Gap = estimatedRecoveryTime - targetRto
        };
    }
}

public record RecoveryAssessment
{
    public TimeSpan TargetRto { get; init; }
    public TimeSpan EstimatedRecoveryTime { get; init; }
    public bool IsCompliant { get; init; }
    public TimeSpan Gap { get; init; }

    public override string ToString() =>
        IsCompliant
            ? $"RTO Uyumlu: Tahmini {EstimatedRecoveryTime.TotalMinutes:F0} dk <= Hedef {TargetRto.TotalMinutes:F0} dk"
            : $"RTO UYUMSUZ: Tahmini {EstimatedRecoveryTime.TotalMinutes:F0} dk > Hedef {TargetRto.TotalMinutes:F0} dk (Fark: {Gap.TotalMinutes:F0} dk)";
}
```

## Yedekleme Stratejileri

### Tam Yedekleme (Full Backup)

Tüm verinin eksiksiz kopyasının alınmasıdır. Geri yükleme en hızlı ve en basit şekilde yapılır; ancak yedekleme süresi ve depolama maliyeti en yüksek stratejidir. Genellikle haftalık ya da günlük olarak planlanır.

### Artımlı Yedekleme (Incremental Backup)

Yalnızca son yedeklemeden bu yana değişen verilerin alınmasıdır. Depolama alanı ve yedekleme süresi minimumdur; ancak geri yükleme sırasında tüm artımlı yedek zincirinin uygulanması gerektiğinden kurtarma daha uzun sürer.

### Diferansiyel Yedekleme (Differential Backup)

Son tam yedeklemeden bu yana değişen tüm verilerin alınmasıdır. Artımlı yedeklemeye göre daha fazla alan kaplar; ancak geri yükleme yalnızca son tam yedek + son diferansiyel yedek ile gerçekleştirilebileceğinden daha hızlıdır.

```csharp
public enum BackupType
{
    Full,
    Incremental,
    Differential
}

public class BackupStrategy
{
    public BackupType Type { get; init; }
    public TimeSpan Interval { get; init; }
    public int RetentionDays { get; init; }

    /// <summary>
    /// Örnek bir yedekleme politikası: haftalık tam + günlük diferansiyel + saatlik artımlı.
    /// </summary>
    public static IReadOnlyList<BackupStrategy> GetRecommendedPolicy(TimeSpan rpo) =>
        rpo.TotalHours switch
        {
            <= 1 => new[]
            {
                new BackupStrategy { Type = BackupType.Full,          Interval = TimeSpan.FromDays(7),    RetentionDays = 30 },
                new BackupStrategy { Type = BackupType.Differential,  Interval = TimeSpan.FromDays(1),    RetentionDays = 7  },
                new BackupStrategy { Type = BackupType.Incremental,   Interval = TimeSpan.FromHours(1),   RetentionDays = 2  }
            },
            <= 4 => new[]
            {
                new BackupStrategy { Type = BackupType.Full,          Interval = TimeSpan.FromDays(7),    RetentionDays = 30 },
                new BackupStrategy { Type = BackupType.Differential,  Interval = TimeSpan.FromHours(4),   RetentionDays = 7  }
            },
            _ => new[]
            {
                new BackupStrategy { Type = BackupType.Full,          Interval = TimeSpan.FromDays(1),    RetentionDays = 30 }
            }
        };
}
```

## C# ile Otomatik Yedekleme Servisi

### Yedekleme Orkestratörü

```csharp
public interface IBackupService
{
    Task<BackupResult> CreateBackupAsync(BackupType type, CancellationToken cancellationToken = default);
    Task<RestoreResult> RestoreBackupAsync(string backupId, CancellationToken cancellationToken = default);
    Task<IReadOnlyList<BackupMetadata>> ListBackupsAsync(CancellationToken cancellationToken = default);
}

public class BackupOrchestrator : IBackupService
{
    private readonly ILogger<BackupOrchestrator> _logger;
    private readonly IBackupStorage _storage;
    private readonly IDatabaseExporter _dbExporter;
    private readonly IBackupValidator _validator;
    private readonly BackupConfiguration _config;

    public BackupOrchestrator(
        ILogger<BackupOrchestrator> logger,
        IBackupStorage storage,
        IDatabaseExporter dbExporter,
        IBackupValidator validator,
        BackupConfiguration config)
    {
        _logger = logger;
        _storage = storage;
        _dbExporter = dbExporter;
        _validator = validator;
        _config = config;
    }

    public async Task<BackupResult> CreateBackupAsync(
        BackupType type,
        CancellationToken cancellationToken = default)
    {
        var backupId = Guid.NewGuid().ToString("N");
        var startTime = DateTime.UtcNow;

        _logger.LogInformation(
            "Yedekleme başlatılıyor. BackupId: {BackupId}, Tip: {Type}",
            backupId, type);

        try
        {
            // 1. Veri dışa aktarımı
            var exportedData = await _dbExporter.ExportAsync(type, cancellationToken);

            // 2. Sıkıştırma ve şifreleme
            var processedData = await ProcessBackupDataAsync(exportedData, cancellationToken);

            // 3. Depolamaya yükleme
            var storagePath = await _storage.UploadAsync(backupId, processedData, cancellationToken);

            // 4. Metadata kaydetme
            var metadata = new BackupMetadata
            {
                Id = backupId,
                Type = type,
                CreatedAt = startTime,
                CompletedAt = DateTime.UtcNow,
                StoragePath = storagePath,
                SizeBytes = processedData.Length,
                DatabaseVersion = await _dbExporter.GetDatabaseVersionAsync(cancellationToken)
            };

            await _storage.SaveMetadataAsync(metadata, cancellationToken);

            // 5. Yedek doğrulama
            var validationResult = await _validator.ValidateAsync(backupId, cancellationToken);
            if (!validationResult.IsValid)
            {
                _logger.LogError(
                    "Yedek doğrulama başarısız. BackupId: {BackupId}, Hata: {Error}",
                    backupId, validationResult.ErrorMessage);

                return BackupResult.Failure(backupId, validationResult.ErrorMessage);
            }

            var duration = DateTime.UtcNow - startTime;
            _logger.LogInformation(
                "Yedekleme tamamlandı. BackupId: {BackupId}, Süre: {Duration:mm\\:ss}",
                backupId, duration);

            return BackupResult.Success(backupId, metadata);
        }
        catch (OperationCanceledException)
        {
            _logger.LogWarning("Yedekleme iptal edildi. BackupId: {BackupId}", backupId);
            throw;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Yedekleme başarısız oldu. BackupId: {BackupId}", backupId);
            return BackupResult.Failure(backupId, ex.Message);
        }
    }

    public async Task<RestoreResult> RestoreBackupAsync(
        string backupId,
        CancellationToken cancellationToken = default)
    {
        _logger.LogWarning(
            "GERİ YÜKLEME BAŞLATILIYOR. BackupId: {BackupId}", backupId);

        var startTime = DateTime.UtcNow;

        try
        {
            // 1. Metadata kontrolü
            var metadata = await _storage.GetMetadataAsync(backupId, cancellationToken)
                ?? throw new BackupNotFoundException(backupId);

            // 2. Yedek dosyasını indir
            var backupData = await _storage.DownloadAsync(backupId, cancellationToken);

            // 3. Şifreyi çöz ve aç
            var rawData = await DecompressAndDecryptAsync(backupData, cancellationToken);

            // 4. Veritabanını geri yükle
            await _dbExporter.ImportAsync(rawData, cancellationToken);

            var duration = DateTime.UtcNow - startTime;
            _logger.LogInformation(
                "Geri yükleme tamamlandı. BackupId: {BackupId}, Süre: {Duration:mm\\:ss}",
                backupId, duration);

            return RestoreResult.Success(backupId, duration);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex,
                "Geri yükleme başarısız. BackupId: {BackupId}", backupId);
            return RestoreResult.Failure(backupId, ex.Message);
        }
    }

    public async Task<IReadOnlyList<BackupMetadata>> ListBackupsAsync(
        CancellationToken cancellationToken = default)
    {
        return await _storage.ListMetadataAsync(cancellationToken);
    }

    private async Task<byte[]> ProcessBackupDataAsync(
        byte[] data,
        CancellationToken cancellationToken)
    {
        // Sıkıştırma
        using var inputStream = new MemoryStream(data);
        using var outputStream = new MemoryStream();
        using (var gzip = new System.IO.Compression.GZipStream(
            outputStream, System.IO.Compression.CompressionLevel.Optimal))
        {
            await inputStream.CopyToAsync(gzip, cancellationToken);
        }
        return outputStream.ToArray();
    }

    private async Task<byte[]> DecompressAndDecryptAsync(
        byte[] data,
        CancellationToken cancellationToken)
    {
        using var inputStream = new MemoryStream(data);
        using var outputStream = new MemoryStream();
        using var gzip = new System.IO.Compression.GZipStream(
            inputStream, System.IO.Compression.CompressionMode.Decompress);
        await gzip.CopyToAsync(outputStream, cancellationToken);
        return outputStream.ToArray();
    }
}

public record BackupResult
{
    public string BackupId { get; init; } = string.Empty;
    public bool IsSuccess { get; init; }
    public string? ErrorMessage { get; init; }
    public BackupMetadata? Metadata { get; init; }

    public static BackupResult Success(string id, BackupMetadata metadata) =>
        new() { BackupId = id, IsSuccess = true, Metadata = metadata };

    public static BackupResult Failure(string id, string error) =>
        new() { BackupId = id, IsSuccess = false, ErrorMessage = error };
}

public record RestoreResult
{
    public string BackupId { get; init; } = string.Empty;
    public bool IsSuccess { get; init; }
    public TimeSpan Duration { get; init; }
    public string? ErrorMessage { get; init; }

    public static RestoreResult Success(string id, TimeSpan duration) =>
        new() { BackupId = id, IsSuccess = true, Duration = duration };

    public static RestoreResult Failure(string id, string error) =>
        new() { BackupId = id, IsSuccess = false, ErrorMessage = error };
}

public record BackupMetadata
{
    public string Id { get; init; } = string.Empty;
    public BackupType Type { get; init; }
    public DateTime CreatedAt { get; init; }
    public DateTime CompletedAt { get; init; }
    public string StoragePath { get; init; } = string.Empty;
    public long SizeBytes { get; init; }
    public string DatabaseVersion { get; init; } = string.Empty;
}
```

### Zamanlanmış Yedekleme (Background Service)

```csharp
public class ScheduledBackupService : BackgroundService
{
    private readonly ILogger<ScheduledBackupService> _logger;
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly BackupScheduleConfiguration _schedule;

    public ScheduledBackupService(
        ILogger<ScheduledBackupService> logger,
        IServiceScopeFactory scopeFactory,
        BackupScheduleConfiguration schedule)
    {
        _logger = logger;
        _scopeFactory = scopeFactory;
        _schedule = schedule;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Zamanlanmış yedekleme servisi başlatıldı.");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await RunScheduledBackupsAsync(stoppingToken);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                _logger.LogError(ex, "Zamanlanmış yedekleme döngüsünde hata oluştu.");
            }

            // Bir dakika bekleyip tekrar kontrol et
            await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
        }
    }

    private async Task RunScheduledBackupsAsync(CancellationToken cancellationToken)
    {
        var now = DateTime.UtcNow;

        foreach (var schedule in _schedule.Schedules)
        {
            if (!ShouldRunBackup(schedule, now))
                continue;

            _logger.LogInformation(
                "Zamanlanmış yedekleme tetiklendi. Tip: {Type}", schedule.BackupType);

            await using var scope = _scopeFactory.CreateAsyncScope();
            var backupService = scope.ServiceProvider.GetRequiredService<IBackupService>();

            var result = await backupService.CreateBackupAsync(
                schedule.BackupType, cancellationToken);

            if (!result.IsSuccess)
            {
                _logger.LogError(
                    "Zamanlanmış yedekleme başarısız. Tip: {Type}, Hata: {Error}",
                    schedule.BackupType, result.ErrorMessage);
            }
        }
    }

    private static bool ShouldRunBackup(BackupScheduleEntry schedule, DateTime now)
    {
        if (schedule.LastRunTime == null)
            return true;

        return now - schedule.LastRunTime.Value >= schedule.Interval;
    }
}

public class BackupScheduleConfiguration
{
    public IReadOnlyList<BackupScheduleEntry> Schedules { get; init; } =
        Array.Empty<BackupScheduleEntry>();
}

public class BackupScheduleEntry
{
    public BackupType BackupType { get; set; }
    public TimeSpan Interval { get; set; }
    public DateTime? LastRunTime { get; set; }
}
```

## C# ile Health Check ve RPO İzleme

### Yedekleme Sağlık Kontrolü

```csharp
public class BackupHealthCheck : IHealthCheck
{
    private readonly IBackupService _backupService;
    private readonly BackupHealthConfiguration _config;
    private readonly ILogger<BackupHealthCheck> _logger;

    public BackupHealthCheck(
        IBackupService backupService,
        BackupHealthConfiguration config,
        ILogger<BackupHealthCheck> logger)
    {
        _backupService = backupService;
        _config = config;
        _logger = logger;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var backups = await _backupService.ListBackupsAsync(cancellationToken);
            var lastBackup = backups
                .OrderByDescending(b => b.CreatedAt)
                .FirstOrDefault();

            if (lastBackup is null)
            {
                return HealthCheckResult.Unhealthy(
                    "Hiç yedek bulunamadı.",
                    data: new Dictionary<string, object>
                    {
                        ["status"] = "no_backups"
                    });
            }

            var timeSinceLastBackup = DateTime.UtcNow - lastBackup.CreatedAt;
            var rpoViolated = timeSinceLastBackup > _config.MaxAllowedRpo;

            var data = new Dictionary<string, object>
            {
                ["last_backup_id"]   = lastBackup.Id,
                ["last_backup_time"] = lastBackup.CreatedAt.ToString("O"),
                ["time_since_backup_minutes"] = (int)timeSinceLastBackup.TotalMinutes,
                ["rpo_target_minutes"] = (int)_config.MaxAllowedRpo.TotalMinutes,
                ["rpo_violated"] = rpoViolated
            };

            if (rpoViolated)
            {
                _logger.LogWarning(
                    "RPO ihlali tespit edildi! Son yedekten bu yana {Minutes} dakika geçti, hedef: {Target} dakika.",
                    (int)timeSinceLastBackup.TotalMinutes,
                    (int)_config.MaxAllowedRpo.TotalMinutes);

                return HealthCheckResult.Degraded(
                    $"RPO ihlali: Son yedekten {timeSinceLastBackup.TotalMinutes:F0} dakika geçti " +
                    $"(Hedef: {_config.MaxAllowedRpo.TotalMinutes:F0} dakika).",
                    data: data);
            }

            return HealthCheckResult.Healthy(
                $"Yedekleme sağlıklı. Son yedek: {timeSinceLastBackup.TotalMinutes:F0} dakika önce.",
                data: data);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Yedekleme sağlık kontrolü başarısız.");
            return HealthCheckResult.Unhealthy(
                "Yedekleme sağlık kontrolü sırasında hata oluştu.",
                exception: ex);
        }
    }
}

public class BackupHealthConfiguration
{
    public TimeSpan MaxAllowedRpo { get; set; } = TimeSpan.FromHours(1);
}

// Startup / Program.cs kayıt örneği
public static class BackupHealthCheckExtensions
{
    public static IHealthChecksBuilder AddBackupHealthCheck(
        this IHealthChecksBuilder builder,
        TimeSpan maxAllowedRpo)
    {
        builder.Services.Configure<BackupHealthConfiguration>(config =>
        {
            config.MaxAllowedRpo = maxAllowedRpo;
        });

        return builder.AddCheck<BackupHealthCheck>(
            name: "backup-rpo",
            failureStatus: HealthStatus.Degraded,
            tags: new[] { "disaster-recovery", "backup" });
    }
}
```

### RTO Simülasyon ve Ölçüm Aracı

```csharp
public class RtoSimulator
{
    private readonly IBackupService _backupService;
    private readonly ILogger<RtoSimulator> _logger;

    public RtoSimulator(IBackupService backupService, ILogger<RtoSimulator> logger)
    {
        _backupService = backupService;
        _logger = logger;
    }

    /// <summary>
    /// Gerçek geri yükleme süreci simülasyonu yapar ve RTO hedefiyle karşılaştırır.
    /// </summary>
    public async Task<RtoSimulationResult> SimulateRecoveryAsync(
        string backupId,
        TimeSpan targetRto,
        CancellationToken cancellationToken = default)
    {
        _logger.LogInformation(
            "RTO simülasyonu başlatılıyor. BackupId: {BackupId}, Hedef: {Rto} dk",
            backupId, targetRto.TotalMinutes);

        var stopwatch = System.Diagnostics.Stopwatch.StartNew();
        var steps = new List<RecoveryStep>();

        try
        {
            // Adım 1: Yedek indirme
            var downloadStart = stopwatch.Elapsed;
            _logger.LogInformation("Adım 1: Yedek dosyası indiriliyor...");
            // Gerçek senaryoda: await _storage.DownloadAsync(backupId, cancellationToken);
            await Task.Delay(TimeSpan.FromSeconds(5), cancellationToken); // Simülasyon
            steps.Add(new RecoveryStep("Yedek İndirme", stopwatch.Elapsed - downloadStart));

            // Adım 2: Geri yükleme
            var restoreStart = stopwatch.Elapsed;
            _logger.LogInformation("Adım 2: Veritabanı geri yükleniyor...");
            var restoreResult = await _backupService.RestoreBackupAsync(backupId, cancellationToken);
            if (!restoreResult.IsSuccess)
                throw new InvalidOperationException($"Geri yükleme başarısız: {restoreResult.ErrorMessage}");
            steps.Add(new RecoveryStep("Veritabanı Geri Yükleme", stopwatch.Elapsed - restoreStart));

            // Adım 3: Sağlık kontrolü
            var healthStart = stopwatch.Elapsed;
            _logger.LogInformation("Adım 3: Sistem sağlık kontrolü yapılıyor...");
            await Task.Delay(TimeSpan.FromSeconds(2), cancellationToken); // Simülasyon
            steps.Add(new RecoveryStep("Sağlık Kontrolü", stopwatch.Elapsed - healthStart));

            stopwatch.Stop();

            var assessment = DisasterRecoveryMetrics.AssessRecovery(targetRto, stopwatch.Elapsed);

            _logger.LogInformation(
                "RTO simülasyonu tamamlandı. {Assessment}", assessment);

            return new RtoSimulationResult
            {
                BackupId = backupId,
                TotalDuration = stopwatch.Elapsed,
                TargetRto = targetRto,
                IsCompliant = assessment.IsCompliant,
                Steps = steps
            };
        }
        catch (Exception ex)
        {
            stopwatch.Stop();
            _logger.LogError(ex, "RTO simülasyonu başarısız oldu.");
            return new RtoSimulationResult
            {
                BackupId = backupId,
                TotalDuration = stopwatch.Elapsed,
                TargetRto = targetRto,
                IsCompliant = false,
                ErrorMessage = ex.Message,
                Steps = steps
            };
        }
    }
}

public record RtoSimulationResult
{
    public string BackupId { get; init; } = string.Empty;
    public TimeSpan TotalDuration { get; init; }
    public TimeSpan TargetRto { get; init; }
    public bool IsCompliant { get; init; }
    public string? ErrorMessage { get; init; }
    public IReadOnlyList<RecoveryStep> Steps { get; init; } = Array.Empty<RecoveryStep>();
}

public record RecoveryStep(string Name, TimeSpan Duration);
```

## Mülakat Soruları

### 1. Soru
**RPO ve RTO nedir? Bir proje için bu değerleri nasıl belirlersiniz?**

**Cevap:** RPO (Recovery Point Objective), bir felaket sonrasında kabul edilebilecek maksimum veri kaybını zaman cinsinden ifade eder; yedekleme sıklığını belirler. RTO (Recovery Time Objective) ise sistemin kesintiden sonra tekrar hizmete girmesi için izin verilen maksimum süredir; altyapı hazırlığını ve kurtarma sürecinin otomasyonunu belirler.

Değerleri belirlerken şu adımlar izlenir: (1) Sistemin iş üzerindeki etkisi analiz edilir (kritik mi, destekleyici mi?), (2) Kesinti başına saatlik gelir kaybı hesaplanır, (3) Regülasyon gereksinimleri incelenir, (4) Mevcut altyapı kapasitesi değerlendirilir ve (5) RPO/RTO hedeflerine ulaşmak için gereken maliyet ile iş riski karşılaştırılır. Örneğin bir bankacılık uygulaması için RPO = 0 ve RTO = dakikalar uygunken, bir blog uygulaması için RPO = 24 saat ve RTO = birkaç saat kabul edilebilir.

**Örnek Kod:**
```csharp
public class RpoRtoPlanner
{
    public DrRequirements PlanRequirements(BusinessImpactAnalysis bia)
    {
        // Kritik sistemler için daha sıkı hedefler
        var rpo = bia.RevenuePerHour > 10_000m
            ? TimeSpan.FromMinutes(15)   // Yüksek gelirli sistem
            : TimeSpan.FromHours(1);     // Düşük öncelikli sistem

        var rto = bia.IsCustomerFacing
            ? TimeSpan.FromMinutes(30)   // Müşteriye dönük sistem
            : TimeSpan.FromHours(4);     // İç sistem

        return new DrRequirements { Rpo = rpo, Rto = rto };
    }
}

public record BusinessImpactAnalysis
{
    public decimal RevenuePerHour { get; init; }
    public bool IsCustomerFacing { get; init; }
    public bool HasRegulatoryRequirements { get; init; }
}

public record DrRequirements
{
    public TimeSpan Rpo { get; init; }
    public TimeSpan Rto { get; init; }
}
```

### 2. Soru
**Tam, artımlı ve diferansiyel yedekleme arasındaki farklar nelerdir? Hangisini ne zaman tercih edersiniz?**

**Cevap:** Tam yedekleme tüm verinin eksiksiz kopyasını alır; geri yükleme en hızlıdır fakat en fazla alan ve süre gerektirir. Artımlı yedekleme yalnızca son yedekten bu yana değişen verileri alır; depolama ve süre minimumdur ancak geri yükleme için tüm zincir (Pazartesi tam + Salı artımlı + Çarşamba artımlı...) uygulanmalıdır. Diferansiyel yedekleme ise son tam yedekten sonra değişen tüm veriyi alır; geri yükleme için yalnızca son tam + son diferansiyel yeterlidir.

Tercih: RPO çok düşükse artımlı yedekleme (sık çalıştırılabilir), RTO düşükse diferansiyel (geri yükleme daha hızlı), ikisi bir arada isteniyorsa haftalık tam + günlük diferansiyel + saatlik artımlı kombinasyonu en yaygın yaklaşımdır.

**Örnek Kod:**
```csharp
public class HybridBackupPolicy
{
    public BackupType DetermineBackupType(DateTime now, DateTime lastFullBackup)
    {
        // Pazar günü tam yedek
        if (now.DayOfWeek == DayOfWeek.Sunday)
            return BackupType.Full;

        // Günde bir diferansiyel (son tam yedekten sonra)
        if (now.Hour == 2 && (now - lastFullBackup).TotalDays >= 1)
            return BackupType.Differential;

        // Saatlik artımlı
        return BackupType.Incremental;
    }
}
```

### 3. Soru
**Yedeklerin gerçekten çalışıp çalışmadığını nasıl doğrularsınız?**

**Cevap:** Yedek almak yetmez; geri yüklenebilirlik de test edilmelidir. Doğrulama yaklaşımları: (1) Checksum doğrulama: yedek dosyasının hash değeri kaydedilerek bütünlük kontrolü yapılır, (2) Periyodik geri yükleme testi: izole bir test ortamına (staging) yedek geri yüklenerek veri doğrulanır, (3) Otomatik smoke test: geri yükleme sonrası kritik tablolarda satır sayısı ve özet veriler kontrol edilir, (4) CI/CD entegrasyonu: her hafta otomatik olarak DR testinin çalıştırılması. Sadece yedek almak yetmez; "yedekten geri dönebiliyorum" kanıtlanmalıdır.

**Örnek Kod:**
```csharp
public class BackupVerificationService
{
    private readonly IBackupService _backupService;
    private readonly IDatabaseValidator _dbValidator;
    private readonly ILogger<BackupVerificationService> _logger;

    public BackupVerificationService(
        IBackupService backupService,
        IDatabaseValidator dbValidator,
        ILogger<BackupVerificationService> logger)
    {
        _backupService = backupService;
        _dbValidator = dbValidator;
        _logger = logger;
    }

    public async Task<VerificationResult> VerifyLatestBackupAsync(
        CancellationToken cancellationToken = default)
    {
        var backups = await _backupService.ListBackupsAsync(cancellationToken);
        var latest = backups.OrderByDescending(b => b.CreatedAt).FirstOrDefault();

        if (latest is null)
            return VerificationResult.Failure("Doğrulanacak yedek bulunamadı.");

        // Test ortamına geri yükle
        var restoreResult = await _backupService.RestoreBackupAsync(
            latest.Id, cancellationToken);

        if (!restoreResult.IsSuccess)
            return VerificationResult.Failure($"Geri yükleme başarısız: {restoreResult.ErrorMessage}");

        // Veri bütünlüğü kontrolü
        var validationPassed = await _dbValidator.ValidateIntegrityAsync(cancellationToken);
        if (!validationPassed)
            return VerificationResult.Failure("Veri bütünlüğü doğrulaması başarısız.");

        _logger.LogInformation(
            "Yedek doğrulama başarılı. BackupId: {BackupId}", latest.Id);

        return VerificationResult.Success(latest.Id);
    }
}

public record VerificationResult
{
    public bool IsValid { get; init; }
    public string? BackupId { get; init; }
    public string? ErrorMessage { get; init; }

    public static VerificationResult Success(string backupId) =>
        new() { IsValid = true, BackupId = backupId };

    public static VerificationResult Failure(string error) =>
        new() { IsValid = false, ErrorMessage = error };
}
```

### 4. Soru
**Bir uygulamanın RPO'sunun 0'a yakın olması gerekiyorsa ne yaparsınız?**

**Cevap:** RPO = 0 veya sıfıra yakın hedefi gerçekleştirmek için senkron replikasyon kullanılır. Bu yaklaşımda yazma işlemi, tüm replikalara başarıyla uygulanmadan tamamlanmış sayılmaz (strong consistency). Uygulamalar: SQL Server Always On Synchronous Commit, PostgreSQL Synchronous Streaming Replication, Azure SQL Geo-Replication (sync mode). Dezavantajı, her yazma için replikasyon gecikmesi kadar ek latency oluşmasıdır; bu nedenle RPO = 0 hedefi yalnızca gerçekten kritik iş verisi için seçilmeli, mümkün olduğunda RPO = birkaç dakika daha maliyet etkin olabilir.

**Örnek Kod:**
```csharp
// Entity Framework Core ile read/write ayrımı (CQRS + replikasyon)
public class ReplicatedDbContextFactory
{
    private readonly string _primaryConnectionString;
    private readonly string _replicaConnectionString;

    public ReplicatedDbContextFactory(
        string primaryConnectionString,
        string replicaConnectionString)
    {
        _primaryConnectionString = primaryConnectionString;
        _replicaConnectionString = replicaConnectionString;
    }

    public AppDbContext CreateWriteContext()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer(_primaryConnectionString)  // Yazma: primary
            .Options;
        return new AppDbContext(options);
    }

    public AppDbContext CreateReadContext()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer(_replicaConnectionString)  // Okuma: replica
            .Options;
        return new AppDbContext(options);
    }
}
```

### 5. Soru
**RPO/RTO ihlallerini gerçek zamanlı olarak nasıl izlersiniz?**

**Cevap:** İzleme için birden fazla katman kullanılır: (1) ASP.NET Core Health Checks API ile `/health` endpoint'i üzerinden RPO uyumluluğu raporlanır, (2) Prometheus/Grafana gibi izleme araçlarıyla son yedek zamanı ve süreleri metrik olarak yayınlanır, (3) Azure Monitor / AWS CloudWatch alarmları ile RPO ihlali tespit edildiğinde otomatik uyarı gönderilir, (4) PagerDuty/OpsGenie gibi on-call sistemlerine entegrasyon yapılarak ilgili ekip anında bildirim alır. Kritik nokta: ihlal yaşandığında öğrenmek değil, ihlal olmadan önce uyarı almak hedeflenmelidir.

**Örnek Kod:**
```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddBackupHealthCheck(maxAllowedRpo: TimeSpan.FromHours(1))
    .AddCheck("database", () =>
    {
        // Veritabanı bağlantı kontrolü
        return HealthCheckResult.Healthy();
    });

builder.Services.AddHealthChecksUI()
    .AddInMemoryStorage();

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecksUI(options => options.UIPath = "/health-ui");
```

### 6. Soru
**3-2-1 yedekleme kuralı nedir ve neden önemlidir?**

**Cevap:** 3-2-1 kuralı şu prensibi ifade eder: (3) Verinin en az 3 kopyası olmalı, (2) Bu kopyalar en az 2 farklı depolama medyasında saklanmalı (örn. disk + bulut), (1) Kopyaların en az 1 tanesi farklı bir fiziksel konumda (offsite) bulunmalı. Bu kural; tek bir arıza noktasından kaynaklanan toplam veri kaybını önler. Örneğin: birincil veritabanı (1. kopya) + aynı veri merkezi NAS yedek (2. kopya, farklı medya) + farklı region bulut depolama (3. kopya, offsite). Ransomware saldırıları göz önüne alındığında offsite kopyanın ağdan izole olması da kritik önem taşır (air-gapped backup).

**Örnek Kod:**
```csharp
public class ThreeTwoOneBackupPolicy
{
    private readonly ILocalBackupStorage _localStorage;
    private readonly INetworkAttachedStorage _nasStorage;
    private readonly ICloudStorage _cloudStorage;
    private readonly ILogger<ThreeTwoOneBackupPolicy> _logger;

    public ThreeTwoOneBackupPolicy(
        ILocalBackupStorage localStorage,
        INetworkAttachedStorage nasStorage,
        ICloudStorage cloudStorage,
        ILogger<ThreeTwoOneBackupPolicy> logger)
    {
        _localStorage = localStorage;
        _nasStorage = nasStorage;
        _cloudStorage = cloudStorage;
        _logger = logger;
    }

    public async Task StoreBackupAsync(
        string backupId,
        byte[] backupData,
        CancellationToken cancellationToken = default)
    {
        var tasks = new List<Task>
        {
            StoreWithLoggingAsync(
                "Kopya 1 (Yerel Disk)",
                () => _localStorage.SaveAsync(backupId, backupData, cancellationToken)),

            StoreWithLoggingAsync(
                "Kopya 2 (NAS - Farklı Medya)",
                () => _nasStorage.SaveAsync(backupId, backupData, cancellationToken)),

            StoreWithLoggingAsync(
                "Kopya 3 (Bulut - Offsite)",
                () => _cloudStorage.SaveAsync(backupId, backupData, cancellationToken))
        };

        await Task.WhenAll(tasks);
        _logger.LogInformation("3-2-1 yedekleme politikası tamamlandı. BackupId: {Id}", backupId);
    }

    private async Task StoreWithLoggingAsync(string location, Func<Task> storeAction)
    {
        try
        {
            await storeAction();
            _logger.LogInformation("{Location}: Yedek başarıyla kaydedildi.", location);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "{Location}: Yedek kaydedilemedi!", location);
            throw;
        }
    }
}
```

## Best Practices

### 1. **Yedek Rotasyonu ve Saklama Politikası**
- Günlük yedekler 7 gün, haftalık yedekler 4 hafta, aylık yedekler 12 ay saklanmalıdır.
- Eski yedeklerin otomatik silinmesi depolama maliyetini optimize eder.
- Yasal gereksinimler saklama sürelerini uzatabilir (örn. 7 yıl).

### 2. **Yedek Şifreleme**
- Yedeklerdeki hassas veriler hem aktarım sırasında hem de depoda şifrelenmelidir.
- Şifreleme anahtarları yedek dosyasından ayrı saklanmalıdır.
- AES-256 standart şifreleme olarak kullanılabilir.

### 3. **Coğrafi Dağıtım**
- Yedekler en az 2 farklı coğrafi bölgede saklanmalıdır.
- Bölgesel afetlere (deprem, sel) karşı yeterli mesafe önemlidir.
- Bulut sağlayıcıların çoklu bölge depolama servisleri (AWS S3 Cross-Region Replication, Azure GRS) kullanılabilir.

### 4. **Otomasyon ve İzleme**
- Manuel yedekleme prosedürleri insan hatasına açıktır; otomasyon zorunludur.
- Her yedekleme sonucu loglanmalı ve başarısız yedeklemeler için uyarı gönderilmelidir.
- Yedekleme metrikleri izleme panosunda görünür olmalıdır.

## Kaynaklar

- [Microsoft Azure Backup Documentation](https://docs.microsoft.com/en-us/azure/backup/)
- [AWS Backup Service](https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html)
- [ASP.NET Core Health Checks](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks)
- [SQL Server Backup and Restore](https://docs.microsoft.com/en-us/sql/relational-databases/backup-restore/back-up-and-restore-of-sql-server-databases)
- [Polly Resilience Library](https://github.com/App-vNext/Polly)
