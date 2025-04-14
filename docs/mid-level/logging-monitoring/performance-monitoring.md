# Performance Monitoring

## Giriş

Performance Monitoring (Performans İzleme), uygulamanın performans metriklerinin sürekli olarak izlenmesi ve analiz edilmesi sürecidir. .NET uygulamalarında performans sorunlarının tespiti, optimizasyon ve kapasite planlaması için kritik bir bileşendir.

## Performance Monitoring'ın Önemi

1. **Performans Optimizasyonu**
   - Bottleneck tespiti
   - Kaynak kullanımı analizi
   - Response time takibi
   - Throughput ölçümü

2. **Kullanıcı Deneyimi**
   - Uygulama yanıt süreleri
   - Sayfa yükleme süreleri
   - API latency
   - Error rate

3. **Sistem Sağlığı**
   - CPU kullanımı
   - Memory kullanımı
   - Disk I/O
   - Network trafiği

## Performance Monitoring Araçları

1. **Application Insights**
   - Performans metrikleri
   - Dependency tracking
   - Exception monitoring
   - Custom telemetri

2. **Prometheus + Grafana**
   - Time series veritabanı
   - Zengin metrik toplama
   - Güçlü görselleştirme
   - Alerting

3. **New Relic**
   - APM (Application Performance Monitoring)
   - Distributed tracing
   - Real-time monitoring
   - Custom dashboards

## Performance Monitoring Kullanımı

1. **Application Insights Entegrasyonu**
```csharp
// NuGet paketleri:
// Microsoft.ApplicationInsights.AspNetCore
// Microsoft.ApplicationInsights.PerfCounterCollector

public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddApplicationInsightsTelemetry(Configuration);
        services.AddApplicationInsightsTelemetryProcessor<CustomTelemetryProcessor>();
    }
}

public class CustomTelemetryProcessor : ITelemetryProcessor
{
    private readonly ITelemetryProcessor _next;

    public CustomTelemetryProcessor(ITelemetryProcessor next)
    {
        _next = next;
    }

    public void Process(ITelemetry item)
    {
        if (item is RequestTelemetry request)
        {
            request.Properties["CustomProperty"] = "Value";
        }
        _next.Process(item);
    }
}
```

2. **Prometheus Entegrasyonu**
```csharp
// NuGet paketleri:
// prometheus-net
// prometheus-net.AspNetCore

public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddPrometheusMetrics();
    }

    public void Configure(IApplicationBuilder app)
    {
        app.UseMetricServer();
        app.UseHttpMetrics();
    }
}

public class PerformanceMetrics
{
    private readonly Counter _requestCounter;
    private readonly Histogram _responseTimeHistogram;

    public PerformanceMetrics()
    {
        _requestCounter = Metrics.CreateCounter("http_requests_total", "Total HTTP requests");
        _responseTimeHistogram = Metrics.CreateHistogram("http_response_time_seconds", "HTTP response time");
    }

    public void TrackRequest(string path, double duration)
    {
        _requestCounter.Inc();
        _responseTimeHistogram.Observe(duration, new[] { path });
    }
}
```

3. **Custom Performance Monitoring**
```csharp
public class PerformanceMonitor
{
    private readonly ILogger _logger;
    private readonly Stopwatch _stopwatch;

    public PerformanceMonitor(ILogger<PerformanceMonitor> logger)
    {
        _logger = logger;
        _stopwatch = new Stopwatch();
    }

    public async Task<T> MonitorOperationAsync<T>(string operationName, Func<Task<T>> operation)
    {
        _stopwatch.Restart();
        try
        {
            var result = await operation();
            _stopwatch.Stop();
            
            _logger.LogInformation("Operation {OperationName} completed in {Duration}ms", 
                operationName, _stopwatch.ElapsedMilliseconds);
            
            return result;
        }
        catch (Exception ex)
        {
            _stopwatch.Stop();
            _logger.LogError(ex, "Operation {OperationName} failed after {Duration}ms", 
                operationName, _stopwatch.ElapsedMilliseconds);
            throw;
        }
    }
}
```

4. **Resource Monitoring**
```csharp
public class ResourceMonitor
{
    private readonly ILogger _logger;
    private readonly PerformanceCounter _cpuCounter;
    private readonly PerformanceCounter _memoryCounter;

    public ResourceMonitor(ILogger<ResourceMonitor> logger)
    {
        _logger = logger;
        _cpuCounter = new PerformanceCounter("Processor", "% Processor Time", "_Total");
        _memoryCounter = new PerformanceCounter("Memory", "Available MBytes");
    }

    public void LogResourceUsage()
    {
        var cpuUsage = _cpuCounter.NextValue();
        var availableMemory = _memoryCounter.NextValue();

        _logger.LogInformation("CPU Usage: {CpuUsage}%, Available Memory: {AvailableMemory}MB", 
            cpuUsage, availableMemory);
    }
}
```

## Performance Monitoring Best Practices

1. **Metrik Tasarımı**
   - Anlamlı metrik isimleri
   - Doğru metrik türleri
   - Context bilgileri
   - Sampling stratejileri

2. **Performans**
   - Hafif metrik toplama
   - Batch işlemler
   - Async operasyonlar
   - Buffer yönetimi

3. **Güvenlik**
   - Hassas veri filtreleme
   - Erişim kontrolü
   - Data retention
   - Audit logging

4. **Monitoring**
   - Alert kuralları
   - Dashboard tasarımı
   - Trend analizi
   - Capacity planning

## Mülakat Soruları

### Temel Sorular

1. **Performance Monitoring nedir ve neden önemlidir?**
   - **Cevap**: Performance Monitoring, uygulamanın performans metriklerinin sürekli olarak izlenmesi ve analiz edilmesi sürecidir. Performans sorunlarının tespiti, optimizasyon ve kapasite planlaması için kritiktir.

2. **Popüler Performance Monitoring araçları nelerdir?**
   - **Cevap**:
     - Application Insights
     - Prometheus + Grafana
     - New Relic
     - Datadog
     - Dynatrace

3. **Performance Monitoring'ın temel bileşenleri nelerdir?**
   - **Cevap**:
     - Metrik toplama
     - Veri depolama
     - Görselleştirme
     - Alerting
     - Analiz

4. **APM nedir?**
   - **Cevap**: APM (Application Performance Monitoring), uygulama performansının end-to-end izlenmesi ve yönetilmesi için kullanılan araçlar ve teknikler bütünüdür.

5. **Bottleneck nedir?**
   - **Cevap**: Bottleneck, sistemin performansını sınırlayan ve darboğaz oluşturan bileşen veya kaynaktır.

### Teknik Sorular

1. **Application Insights nasıl yapılandırılır?**
   - **Cevap**:
```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddApplicationInsightsTelemetry(Configuration);
        services.AddApplicationInsightsTelemetryProcessor<CustomTelemetryProcessor>();
    }
}
```

2. **Prometheus metrikleri nasıl tanımlanır?**
   - **Cevap**:
```csharp
public class PerformanceMetrics
{
    private readonly Counter _requestCounter;
    private readonly Histogram _responseTimeHistogram;

    public PerformanceMetrics()
    {
        _requestCounter = Metrics.CreateCounter("http_requests_total", "Total HTTP requests");
        _responseTimeHistogram = Metrics.CreateHistogram("http_response_time_seconds", "HTTP response time");
    }

    public void TrackRequest(string path, double duration)
    {
        _requestCounter.Inc();
        _responseTimeHistogram.Observe(duration, new[] { path });
    }
}
```

3. **Custom performance monitoring nasıl yapılır?**
   - **Cevap**:
```csharp
public class PerformanceMonitor
{
    private readonly ILogger _logger;
    private readonly Stopwatch _stopwatch;

    public PerformanceMonitor(ILogger<PerformanceMonitor> logger)
    {
        _logger = logger;
        _stopwatch = new Stopwatch();
    }

    public async Task<T> MonitorOperationAsync<T>(string operationName, Func<Task<T>> operation)
    {
        _stopwatch.Restart();
        try
        {
            var result = await operation();
            _stopwatch.Stop();
            
            _logger.LogInformation("Operation {OperationName} completed in {Duration}ms", 
                operationName, _stopwatch.ElapsedMilliseconds);
            
            return result;
        }
        catch (Exception ex)
        {
            _stopwatch.Stop();
            _logger.LogError(ex, "Operation {OperationName} failed after {Duration}ms", 
                operationName, _stopwatch.ElapsedMilliseconds);
            throw;
        }
    }
}
```

4. **Resource monitoring nasıl yapılır?**
   - **Cevap**:
```csharp
public class ResourceMonitor
{
    private readonly ILogger _logger;
    private readonly PerformanceCounter _cpuCounter;
    private readonly PerformanceCounter _memoryCounter;

    public ResourceMonitor(ILogger<ResourceMonitor> logger)
    {
        _logger = logger;
        _cpuCounter = new PerformanceCounter("Processor", "% Processor Time", "_Total");
        _memoryCounter = new PerformanceCounter("Memory", "Available MBytes");
    }

    public void LogResourceUsage()
    {
        var cpuUsage = _cpuCounter.NextValue();
        var availableMemory = _memoryCounter.NextValue();

        _logger.LogInformation("CPU Usage: {CpuUsage}%, Available Memory: {AvailableMemory}MB", 
            cpuUsage, availableMemory);
    }
}
```

5. **Performance alerting nasıl yapılandırılır?**
   - **Cevap**:
```csharp
public class PerformanceAlert
{
    private readonly ILogger _logger;
    private readonly double _threshold;

    public PerformanceAlert(ILogger<PerformanceAlert> logger, double threshold)
    {
        _logger = logger;
        _threshold = threshold;
    }

    public void CheckPerformance(double metricValue, string metricName)
    {
        if (metricValue > _threshold)
        {
            _logger.LogWarning("Performance alert: {MetricName} exceeded threshold. Value: {Value}", 
                metricName, metricValue);
        }
    }
}
```

### İleri Seviye Sorular

1. **Performance Monitoring performans optimizasyonu nasıl yapılır?**
   - **Cevap**:
     - Hafif metrik toplama
     - Batch işlemler
     - Async operasyonlar
     - Buffer yönetimi
     - Sampling stratejileri

2. **Performance Monitoring güvenliği nasıl sağlanır?**
   - **Cevap**:
     - Hassas veri filtreleme
     - Erişim kontrolü
     - Data retention
     - Audit logging
     - Encryption

3. **Performance Monitoring ile capacity planning nasıl yapılır?**
   - **Cevap**:
     - Trend analizi
     - Resource utilization
     - Growth projection
     - Scaling strategies
     - Cost optimization

4. **Performance Monitoring ile distributed tracing nasıl entegre edilir?**
   - **Cevap**:
     - Correlation ID
     - Context propagation
     - Span ve trace yönetimi
     - End-to-end tracing
     - Performance analysis

5. **Performance Monitoring ile anomaly detection nasıl yapılır?**
   - **Cevap**:
     - Statistical analysis
     - Machine learning
     - Pattern recognition
     - Threshold optimization
     - Alert management 