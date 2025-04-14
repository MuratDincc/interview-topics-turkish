# Log Aggregation

## Giriş

Log Aggregation (Log Toplama), dağıtık sistemlerde logların merkezi bir noktada toplanması, depolanması ve analiz edilmesi sürecidir. Modern uygulamalarda, özellikle mikroservis mimarilerinde, logların etkili bir şekilde yönetilmesi için kritik bir bileşendir.

## Log Aggregation'ın Önemi

1. **Merkezi Yönetim**
   - Tüm logların tek noktada toplanması
   - Kolay erişim ve analiz
   - Tutarlı log formatı
   - Merkezi yapılandırma

2. **Performans ve Ölçeklenebilirlik**
   - Dağıtık log toplama
   - Yük dengeleme
   - Buffer yönetimi
   - Batch işlemler

3. **Analiz ve İzleme**
   - Gerçek zamanlı analiz
   - Trend analizi
   - Anomali tespiti
   - Alerting

## Log Aggregation Araçları

1. **ELK Stack (Elasticsearch, Logstash, Kibana)**
   - Elasticsearch: Log depolama ve arama
   - Logstash: Log toplama ve işleme
   - Kibana: Görselleştirme ve analiz

2. **Fluentd**
   - Hafif ve esnek
   - Zengin plugin ekosistemi
   - Yüksek performans
   - Kolay yapılandırma

3. **Graylog**
   - Gerçek zamanlı analiz
   - Alerting
   - Role-based erişim
   - Dashboard özellikleri

## Log Aggregation Kullanımı

1. **ELK Stack Kurulumu**
```csharp
// NuGet paketleri:
// Serilog
// Serilog.Sinks.Elasticsearch
// Serilog.Sinks.Console

public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        Log.Logger = new LoggerConfiguration()
            .MinimumLevel.Information()
            .WriteTo.Console()
            .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(new Uri("http://localhost:9200"))
            {
                AutoRegisterTemplate = true,
                IndexFormat = "myapp-logs-{0:yyyy.MM.dd}"
            })
            .CreateLogger();

        services.AddLogging(builder => builder.AddSerilog());
    }
}
```

2. **Fluentd Entegrasyonu**
```csharp
public class FluentdLogger
{
    private readonly ILogger _logger;

    public FluentdLogger(ILogger<FluentdLogger> logger)
    {
        _logger = logger;
    }

    public void LogInformation(string message, object data = null)
    {
        _logger.LogInformation("{Message} {@Data}", message, data);
    }

    public void LogError(Exception ex, string message, object data = null)
    {
        _logger.LogError(ex, "{Message} {@Data}", message, data);
    }
}
```

3. **Graylog Entegrasyonu**
```csharp
public class GraylogLogger
{
    private readonly ILogger _logger;

    public GraylogLogger(ILogger<GraylogLogger> logger)
    {
        _logger = logger;
    }

    public void LogWithContext(string message, string correlationId, object data = null)
    {
        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["CorrelationId"] = correlationId,
            ["Data"] = data
        }))
        {
            _logger.LogInformation(message);
        }
    }
}
```

4. **Log Enrichment**
```csharp
public class LogEnricher
{
    private readonly ILogger _logger;

    public LogEnricher(ILogger<LogEnricher> logger)
    {
        _logger = logger;
    }

    public void LogWithEnrichment(string message, HttpContext context)
    {
        var enrichedData = new
        {
            RequestPath = context.Request.Path,
            UserAgent = context.Request.Headers["User-Agent"],
            ClientIp = context.Connection.RemoteIpAddress,
            UserId = context.User?.FindFirst("sub")?.Value
        };

        _logger.LogInformation("{Message} {@EnrichedData}", message, enrichedData);
    }
}
```

## Log Aggregation Best Practices

1. **Log Tasarımı**
   - Yapılandırılmış loglar
   - Anlamlı log seviyeleri
   - Context bilgileri
   - Unique identifier'lar

2. **Performans**
   - Asenkron loglama
   - Batch işlemler
   - Buffer yönetimi
   - Sampling stratejileri

3. **Güvenlik**
   - Hassas veri filtreleme
   - PII veri yönetimi
   - Erişim kontrolü
   - Log rotasyonu

4. **Monitoring**
   - Log hacmi takibi
   - Error rate monitoring
   - Performance metrics
   - Alert kuralları

## Mülakat Soruları

### Temel Sorular

1. **Log Aggregation nedir ve neden önemlidir?**
   - **Cevap**: Log Aggregation, dağıtık sistemlerde logların merkezi bir noktada toplanması, depolanması ve analiz edilmesi sürecidir. Özellikle mikroservis mimarilerinde logların etkili yönetimi için kritiktir.

2. **Popüler Log Aggregation araçları nelerdir?**
   - **Cevap**:
     - ELK Stack (Elasticsearch, Logstash, Kibana)
     - Fluentd
     - Graylog
     - Splunk
     - Datadog

3. **Log Aggregation'ın temel bileşenleri nelerdir?**
   - **Cevap**:
     - Log toplama
     - Log depolama
     - Log analizi
     - Görselleştirme
     - Alerting

4. **Yapılandırılmış loglama nedir?**
   - **Cevap**: Yapılandırılmış loglama, logların belirli bir formatta (genellikle JSON) yazılmasıdır. Bu, logların daha kolay analiz edilmesini ve işlenmesini sağlar.

5. **Log rotasyonu nedir?**
   - **Cevap**: Log rotasyonu, log dosyalarının belirli bir boyuta veya süreye ulaştığında arşivlenmesi ve yeni dosyaların oluşturulması işlemidir.

### Teknik Sorular

1. **ELK Stack nasıl yapılandırılır?**
   - **Cevap**:
```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        Log.Logger = new LoggerConfiguration()
            .MinimumLevel.Information()
            .WriteTo.Console()
            .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(new Uri("http://localhost:9200"))
            {
                AutoRegisterTemplate = true,
                IndexFormat = "myapp-logs-{0:yyyy.MM.dd}"
            })
            .CreateLogger();

        services.AddLogging(builder => builder.AddSerilog());
    }
}
```

2. **Log enrichment nasıl yapılır?**
   - **Cevap**:
```csharp
public class LogEnricher
{
    private readonly ILogger _logger;

    public LogEnricher(ILogger<LogEnricher> logger)
    {
        _logger = logger;
    }

    public void LogWithEnrichment(string message, HttpContext context)
    {
        var enrichedData = new
        {
            RequestPath = context.Request.Path,
            UserAgent = context.Request.Headers["User-Agent"],
            ClientIp = context.Connection.RemoteIpAddress,
            UserId = context.User?.FindFirst("sub")?.Value
        };

        _logger.LogInformation("{Message} {@EnrichedData}", message, enrichedData);
    }
}
```

3. **Asenkron loglama nasıl yapılır?**
   - **Cevap**:
```csharp
public class AsyncLogger
{
    private readonly ILogger _logger;
    private readonly Channel<string> _logChannel;

    public AsyncLogger(ILogger<AsyncLogger> logger)
    {
        _logger = logger;
        _logChannel = Channel.CreateUnbounded<string>();
        StartProcessing();
    }

    private async void StartProcessing()
    {
        await foreach (var message in _logChannel.Reader.ReadAllAsync())
        {
            _logger.LogInformation(message);
        }
    }

    public async Task LogAsync(string message)
    {
        await _logChannel.Writer.WriteAsync(message);
    }
}
```

4. **Log sampling nasıl yapılandırılır?**
   - **Cevap**:
```csharp
public class SampledLogger
{
    private readonly ILogger _logger;
    private readonly Random _random;
    private readonly double _samplingRate;

    public SampledLogger(ILogger<SampledLogger> logger, double samplingRate = 0.1)
    {
        _logger = logger;
        _random = new Random();
        _samplingRate = samplingRate;
    }

    public void LogWithSampling(string message)
    {
        if (_random.NextDouble() < _samplingRate)
        {
            _logger.LogInformation(message);
        }
    }
}
```

5. **Log rotasyonu nasıl yapılandırılır?**
   - **Cevap**:
```csharp
public class RotatingLogger
{
    private readonly ILogger _logger;

    public RotatingLogger(ILogger<RotatingLogger> logger)
    {
        _logger = logger;
    }

    public void ConfigureLogRotation(string logPath, long maxSize, int maxFiles)
    {
        Log.Logger = new LoggerConfiguration()
            .WriteTo.File(logPath,
                rollingInterval: RollingInterval.Day,
                rollOnFileSizeLimit: true,
                fileSizeLimitBytes: maxSize,
                retainedFileCountLimit: maxFiles)
            .CreateLogger();
    }
}
```

### İleri Seviye Sorular

1. **Log Aggregation performans optimizasyonu nasıl yapılır?**
   - **Cevap**:
     - Asenkron loglama
     - Batch işlemler
     - Buffer yönetimi
     - Sampling stratejileri
     - Compression

2. **Log Aggregation güvenliği nasıl sağlanır?**
   - **Cevap**:
     - Hassas veri filtreleme
     - PII veri yönetimi
     - Erişim kontrolü
     - Log rotasyonu
     - Audit logging

3. **Log Aggregation monitoring ve alerting nasıl yapılır?**
   - **Cevap**:
     - Log hacmi takibi
     - Error rate monitoring
     - Performance metrics
     - Alert kuralları
     - Dashboard tasarımı

4. **Log Aggregation ile distributed tracing nasıl entegre edilir?**
   - **Cevap**:
     - Correlation ID
     - Context propagation
     - Span ve trace yönetimi
     - End-to-end tracing
     - Performance analysis

5. **Log Aggregation ile log analizi nasıl yapılır?**
   - **Cevap**:
     - Pattern recognition
     - Anomali tespiti
     - Trend analizi
     - Machine learning
     - Predictive analytics 