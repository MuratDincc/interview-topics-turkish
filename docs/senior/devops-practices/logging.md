# Logging (Günlük Kaydı)

## Genel Bakış
Logging, uygulama ve sistem davranışlarının kaydedilmesi, analiz edilmesi ve izlenmesi sürecidir. Etkili bir logging stratejisi, sorunların tespit edilmesi, performans optimizasyonu ve güvenlik denetimleri için kritik öneme sahiptir.

## Temel Kavramlar

### 1. Yapılandırılmış Logging
```csharp
public class StructuredLoggingService
{
    private readonly ILogger<StructuredLoggingService> _logger;

    public StructuredLoggingService(ILogger<StructuredLoggingService> logger)
    {
        _logger = logger;
    }

    public void LogUserAction(UserAction action)
    {
        _logger.LogInformation(
            "User {UserId} performed {ActionType} on {ResourceType} at {Timestamp}",
            action.UserId,
            action.ActionType,
            action.ResourceType,
            action.Timestamp
        );
    }

    public void LogError(Exception ex, string context)
    {
        _logger.LogError(
            ex,
            "Error occurred in {Context}. Error: {ErrorMessage}",
            context,
            ex.Message
        );
    }
}
```

### 2. Log Seviyeleri ve Filtreleme
```csharp
public class LogLevelService
{
    private readonly ILogger<LogLevelService> _logger;

    public LogLevelService(ILogger<LogLevelService> logger)
    {
        _logger = logger;
    }

    public void LogWithDifferentLevels(string message)
    {
        // Debug seviyesi - detaylı bilgi
        _logger.LogDebug($"Debug: {message}");

        // Information seviyesi - genel bilgi
        _logger.LogInformation($"Info: {message}");

        // Warning seviyesi - uyarı
        _logger.LogWarning($"Warning: {message}");

        // Error seviyesi - hata
        _logger.LogError($"Error: {message}");

        // Critical seviyesi - kritik hata
        _logger.LogCritical($"Critical: {message}");
    }
}
```

### 3. Log Toplama ve Analiz
```csharp
public class LogAggregationService
{
    private readonly ILogger<LogAggregationService> _logger;
    private readonly ILogCollector _logCollector;

    public LogAggregationService(ILogger<LogAggregationService> logger, ILogCollector logCollector)
    {
        _logger = logger;
        _logCollector = logCollector;
    }

    public async Task AnalyzeLogsAsync(DateTime startTime, DateTime endTime)
    {
        // Logları topla
        var logs = await _logCollector.GetLogsAsync(startTime, endTime);

        // Hata analizi
        var errorLogs = logs.Where(l => l.Level == LogLevel.Error).ToList();
        _logger.LogInformation($"Found {errorLogs.Count} error logs");

        // Performans analizi
        var slowRequests = logs.Where(l => l.Duration > TimeSpan.FromSeconds(1)).ToList();
        _logger.LogInformation($"Found {slowRequests.Count} slow requests");

        // Kullanıcı aktivite analizi
        var userActivities = logs.GroupBy(l => l.UserId)
            .Select(g => new { UserId = g.Key, Count = g.Count() })
            .ToList();
    }
}
```

### 4. Log Rotasyonu ve Arşivleme
```csharp
public class LogRotationService
{
    private readonly ILogger<LogRotationService> _logger;
    private readonly ILogStorage _logStorage;

    public LogRotationService(ILogger<LogRotationService> logger, ILogStorage logStorage)
    {
        _logger = logger;
        _logStorage = logStorage;
    }

    public async Task RotateLogsAsync()
    {
        // Eski logları arşivle
        var oldLogs = await _logStorage.GetOldLogsAsync(TimeSpan.FromDays(30));
        await _logStorage.ArchiveLogsAsync(oldLogs);

        // Log dosyalarını döndür
        await _logStorage.RotateLogFilesAsync();

        _logger.LogInformation("Log rotation completed successfully");
    }
}
```

## Best Practices

### 1. Log Stratejisi
- Log seviyelerinin doğru kullanımı
- Yapılandırılmış log formatı
- Anlamlı log mesajları
- Bağlam bilgisi
- Performans etkisi

### 2. Güvenlik ve Gizlilik
- Hassas veri filtreleme
- Log şifreleme
- Erişim kontrolü
- Veri saklama politikaları
- Uyumluluk gereksinimleri

### 3. Performans ve Ölçeklenebilirlik
- Asenkron logging
- Toplu log gönderimi
- Log sıkıştırma
- Depolama optimizasyonu
- Kaynak kullanımı

## Sık Sorulan Sorular

### 1. Logging neden önemlidir?
- Sorun tespiti
- Performans analizi
- Güvenlik denetimi
- Kullanıcı davranışı analizi
- Uyumluluk gereksinimleri

### 2. Hangi bilgiler loglanmalıdır?
- Hata mesajları
- Performans metrikleri
- Kullanıcı işlemleri
- Sistem olayları
- Güvenlik olayları

### 3. Logging zorlukları nelerdir?
- Veri hacmi
- Performans etkisi
- Depolama maliyeti
- Veri analizi
- Güvenlik endişeleri

## Kaynaklar
- [Serilog Documentation](https://serilog.net/)
- [NLog Documentation](https://nlog-project.org/)
- [ELK Stack Documentation](https://www.elastic.co/guide/index.html)
- [Microsoft Logging Documentation](https://docs.microsoft.com/tr-tr/aspnet/core/fundamentals/logging/) 