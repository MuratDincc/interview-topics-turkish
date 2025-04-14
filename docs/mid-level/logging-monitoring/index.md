# Logging ve Monitoring

## Giriş

Logging ve Monitoring, modern uygulamaların vazgeçilmez bileşenleridir. Uygulama davranışını izleme, hata ayıklama ve performans optimizasyonu için kritik öneme sahiptir.

## Logging'in Önemi

1. **Hata Ayıklama**
   - Hata kaynaklarını belirleme
   - Sorunları hızlı tespit etme
   - Debugging sürecini kolaylaştırma

2. **İzlenebilirlik**
   - İş akışlarını takip etme
   - Kullanıcı davranışlarını analiz etme
   - Sistem durumunu anlama

3. **Güvenlik**
   - Güvenlik olaylarını kaydetme
   - Yetkisiz erişimleri tespit etme
   - Audit trail oluşturma

## Monitoring'in Önemi

1. **Performans İzleme**
   - Sistem kaynaklarını izleme
   - Yanıt sürelerini ölçme
   - Bottleneck'leri tespit etme

2. **Sağlık Kontrolü**
   - Sistem durumunu kontrol etme
   - Hizmet kullanılabilirliğini izleme
   - Proaktif sorun tespiti

3. **Kapasite Planlama**
   - Kaynak kullanımını analiz etme
   - Ölçeklendirme ihtiyaçlarını belirleme
   - Maliyet optimizasyonu

## Logging Türleri

1. **Structured Logging**
   - JSON formatında loglar
   - Kolay analiz edilebilir
   - Machine-readable format

2. **Unstructured Logging**
   - Serbest metin logları
   - İnsan tarafından okunabilir
   - Geleneksel format

3. **Event Logging**
   - Önemli olayların kaydı
   - İş mantığı olayları
   - Sistem olayları

## Monitoring Türleri

1. **Application Monitoring**
   - Performans metrikleri
   - Hata oranları
   - İş metrikleri

2. **Infrastructure Monitoring**
   - CPU kullanımı
   - Memory kullanımı
   - Disk I/O
   - Network trafiği

3. **Business Monitoring**
   - Kullanıcı davranışları
   - İş metrikleri
   - KPI'lar

## Logging ve Monitoring Best Practices

1. **Log Seviyeleri**
   - Debug
   - Info
   - Warning
   - Error
   - Critical

2. **Context Bilgisi**
   - Correlation ID
   - User context
   - Request context
   - Environment bilgisi

3. **Log Rotation**
   - Boyut limitleri
   - Zaman limitleri
   - Arşivleme stratejileri

4. **Alerting**
   - Threshold'lar
   - Notification kanalları
   - Escalation politikaları

## Mülakat Soruları

### Temel Sorular

1. **Logging nedir ve neden önemlidir?**
   - **Cevap**: Logging, uygulama davranışlarının ve olaylarının kaydedilmesidir. Hata ayıklama, izlenebilirlik ve güvenlik için kritik öneme sahiptir.

2. **Structured ve Unstructured logging arasındaki farklar nelerdir?**
   - **Cevap**:
     - Structured: JSON format, makine tarafından okunabilir
     - Unstructured: Serbest metin, insan tarafından okunabilir
     - Analiz kolaylığı
     - Arama ve filtreleme

3. **Log seviyeleri nelerdir ve ne zaman kullanılır?**
   - **Cevap**:
     - Debug: Geliştirme aşamasında
     - Info: Normal işlemler
     - Warning: Potansiyel sorunlar
     - Error: Hatalı durumlar
     - Critical: Sistem çöküşleri

4. **Monitoring nedir ve neden önemlidir?**
   - **Cevap**: Monitoring, sistem performansı ve sağlığının sürekli izlenmesidir. Proaktif sorun tespiti ve performans optimizasyonu için önemlidir.

5. **Application ve Infrastructure monitoring arasındaki farklar nelerdir?**
   - **Cevap**:
     - Application: Uygulama metrikleri, iş mantığı
     - Infrastructure: Sistem kaynakları, donanım
     - Farklı metrikler
     - Farklı araçlar

### Teknik Sorular

1. **Serilog kullanımı nasıl yapılır?**
   - **Cevap**:
```csharp
public class LoggingService
{
    private readonly ILogger _logger;

    public LoggingService()
    {
        _logger = new LoggerConfiguration()
            .MinimumLevel.Information()
            .WriteTo.Console()
            .WriteTo.File("logs/log.txt", rollingInterval: RollingInterval.Day)
            .CreateLogger();
    }

    public void LogInformation(string message, object data)
    {
        _logger.Information("{Message} {@Data}", message, data);
    }

    public void LogError(Exception ex, string message)
    {
        _logger.Error(ex, "{Message}", message);
    }
}
```

2. **Application Insights entegrasyonu nasıl yapılır?**
   - **Cevap**:
```csharp
public class ApplicationInsightsService
{
    private readonly TelemetryClient _telemetryClient;

    public ApplicationInsightsService(TelemetryClient telemetryClient)
    {
        _telemetryClient = telemetryClient;
    }

    public void TrackRequest(string name, TimeSpan duration, bool success)
    {
        _telemetryClient.TrackRequest(name, DateTimeOffset.Now, duration, "200", success);
    }

    public void TrackException(Exception ex)
    {
        _telemetryClient.TrackException(ex);
    }

    public void TrackMetric(string name, double value)
    {
        _telemetryClient.TrackMetric(name, value);
    }
}
```

3. **OpenTelemetry kullanımı nasıl yapılır?**
   - **Cevap**:
```csharp
public class OpenTelemetryService
{
    private readonly Tracer _tracer;
    private readonly Meter _meter;

    public OpenTelemetryService()
    {
        var tracerProvider = Sdk.CreateTracerProviderBuilder()
            .AddSource("MyApplication")
            .AddConsoleExporter()
            .Build();

        var meterProvider = Sdk.CreateMeterProviderBuilder()
            .AddMeter("MyApplication")
            .AddConsoleExporter()
            .Build();

        _tracer = tracerProvider.GetTracer("MyApplication");
        _meter = meterProvider.GetMeter("MyApplication");
    }

    public async Task TrackOperation(string name, Func<Task> operation)
    {
        using var span = _tracer.StartActiveSpan(name);
        try
        {
            await operation();
        }
        catch (Exception ex)
        {
            span.RecordException(ex);
            throw;
        }
    }
}
```

4. **Log aggregation nasıl yapılır?**
   - **Cevap**:
```csharp
public class LogAggregationService
{
    private readonly ILogger _logger;
    private readonly ElasticClient _elasticClient;

    public LogAggregationService(ILogger logger, ElasticClient elasticClient)
    {
        _logger = logger;
        _elasticClient = elasticClient;
    }

    public async Task IndexLog(LogEntry log)
    {
        try
        {
            await _elasticClient.IndexDocumentAsync(log);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to index log: {@Log}", log);
        }
    }

    public async Task<IEnumerable<LogEntry>> SearchLogs(string query)
    {
        var response = await _elasticClient.SearchAsync<LogEntry>(s => s
            .Query(q => q
                .QueryString(qs => qs
                    .Query(query)
                )
            )
        );

        return response.Documents;
    }
}
```

5. **Performance monitoring nasıl yapılır?**
   - **Cevap**:
```csharp
public class PerformanceMonitor
{
    private readonly ILogger _logger;
    private readonly Stopwatch _stopwatch;

    public PerformanceMonitor(ILogger logger)
    {
        _logger = logger;
        _stopwatch = new Stopwatch();
    }

    public async Task<T> MeasureOperation<T>(string operationName, Func<Task<T>> operation)
    {
        _stopwatch.Restart();
        try
        {
            var result = await operation();
            return result;
        }
        finally
        {
            _stopwatch.Stop();
            _logger.LogInformation(
                "Operation {OperationName} completed in {ElapsedMilliseconds}ms",
                operationName,
                _stopwatch.ElapsedMilliseconds
            );
        }
    }
}
```

### İleri Seviye Sorular

1. **Distributed tracing nasıl uygulanır?**
   - **Cevap**:
     - Correlation ID
     - Trace context
     - Span propagation
     - Context injection
     - Trace visualization

2. **Log sampling stratejileri nelerdir?**
   - **Cevap**:
     - Rate-based sampling
     - Adaptive sampling
     - Priority-based sampling
     - Dynamic sampling
     - Cost-based sampling

3. **Alerting stratejileri nelerdir?**
   - **Cevap**:
     - Threshold-based alerts
     - Anomaly detection
     - Trend analysis
     - Composite alerts
     - Alert routing

4. **Log retention stratejileri nelerdir?**
   - **Cevap**:
     - Time-based retention
     - Size-based retention
     - Cost-based retention
     - Compliance requirements
     - Archive strategies

5. **Monitoring scalability nasıl sağlanır?**
   - **Cevap**:
     - Sampling
     - Aggregation
     - Downsampling
     - Data partitioning
     - Storage optimization 