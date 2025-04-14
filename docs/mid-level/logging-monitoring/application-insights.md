# Application Insights

## Giriş

Application Insights, Microsoft'un bulut tabanlı bir uygulama performans yönetimi (APM) ve izleme hizmetidir. .NET uygulamalarında performans izleme, hata tespiti ve kullanıcı davranışı analizi için kullanılır. Azure ekosistemiyle tam entegrasyon sağlar.

## Application Insights'ın Önemi

1. **Performans İzleme**
   - Uygulama yanıt süreleri
   - Bağımlılık performansı
   - Kaynak kullanımı
   - Exception tracking

2. **Kullanılabilirlik**
   - Uptime monitoring
   - Web testleri
   - Alerting
   - SLA takibi

3. **Kullanıcı Analizi**
   - Kullanıcı davranışları
   - Kullanım istatistikleri
   - Conversion analizi
   - Kullanıcı segmentasyonu

## Application Insights Kullanımı

1. **Temel Kurulum**
```csharp
// NuGet paketleri:
// Microsoft.ApplicationInsights.AspNetCore
// Microsoft.ApplicationInsights.PerfCounterCollector

public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddApplicationInsightsTelemetry(Configuration);
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        app.UseApplicationInsightsRequestTelemetry();
        app.UseApplicationInsightsExceptionTelemetry();
    }
}
```

2. **Özel Telemetri**
```csharp
public class TelemetryService
{
    private readonly TelemetryClient _telemetryClient;

    public TelemetryService(TelemetryClient telemetryClient)
    {
        _telemetryClient = telemetryClient;
    }

    public void TrackCustomEvent(string eventName, IDictionary<string, string> properties = null)
    {
        _telemetryClient.TrackEvent(eventName, properties);
    }

    public void TrackMetric(string metricName, double value)
    {
        _telemetryClient.TrackMetric(metricName, value);
    }

    public void TrackDependency(string dependencyType, string target, string dependencyName, 
        DateTimeOffset startTime, TimeSpan duration, bool success)
    {
        _telemetryClient.TrackDependency(dependencyType, target, dependencyName, 
            startTime, duration, success);
    }
}
```

3. **Exception Tracking**
```csharp
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly TelemetryClient _telemetryClient;

    public ExceptionHandlingMiddleware(RequestDelegate next, TelemetryClient telemetryClient)
    {
        _next = next;
        _telemetryClient = telemetryClient;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _telemetryClient.TrackException(ex);
            throw;
        }
    }
}
```

4. **Performance Tracking**
```csharp
public class PerformanceTracker
{
    private readonly TelemetryClient _telemetryClient;

    public PerformanceTracker(TelemetryClient telemetryClient)
    {
        _telemetryClient = telemetryClient;
    }

    public async Task<T> TrackOperationAsync<T>(string operationName, Func<Task<T>> operation)
    {
        var startTime = DateTimeOffset.UtcNow;
        var timer = System.Diagnostics.Stopwatch.StartNew();

        try
        {
            var result = await operation();
            timer.Stop();

            _telemetryClient.TrackDependency("Custom", operationName, 
                startTime, timer.Elapsed, true);

            return result;
        }
        catch (Exception ex)
        {
            timer.Stop();
            _telemetryClient.TrackDependency("Custom", operationName, 
                startTime, timer.Elapsed, false);
            throw;
        }
    }
}
```

## Application Insights Best Practices

1. **Telemetri Tasarımı**
   - Anlamlı event isimleri
   - Ölçülebilir metrikler
   - Context bilgileri
   - Sampling stratejileri

2. **Performans**
   - Telemetri hacmi yönetimi
   - Sampling kullanımı
   - Batch işlemler
   - Async operasyonlar

3. **Güvenlik**
   - Hassas veri filtreleme
   - PII veri yönetimi
   - Erişim kontrolü
   - Data retention

4. **Monitoring**
   - Alert kuralları
   - Dashboard tasarımı
   - Metric aggregation
   - Trend analizi

## Mülakat Soruları

### Temel Sorular

1. **Application Insights nedir ve ne için kullanılır?**
   - **Cevap**: Application Insights, Microsoft'un bulut tabanlı bir APM ve izleme hizmetidir. Performans izleme, hata tespiti ve kullanıcı davranışı analizi için kullanılır.

2. **Application Insights'ın temel özellikleri nelerdir?**
   - **Cevap**:
     - Performans izleme
     - Exception tracking
     - Kullanıcı analizi
     - Uptime monitoring
     - Alerting

3. **Application Insights'ın avantajları nelerdir?**
   - **Cevap**:
     - Azure entegrasyonu
     - Zengin telemetri
     - Güçlü analiz
     - Kolay kurulum
     - Ölçeklenebilirlik

4. **Telemetri nedir?**
   - **Cevap**: Telemetri, uygulama davranışı ve performansı hakkında toplanan verilerdir. Event, metric, dependency ve exception gibi farklı türleri vardır.

5. **Sampling nedir?**
   - **Cevap**: Sampling, telemetri verilerinin belirli bir yüzdesini toplama işlemidir. Performans ve maliyet optimizasyonu için kullanılır.

### Teknik Sorular

1. **Application Insights nasıl yapılandırılır?**
   - **Cevap**:
```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddApplicationInsightsTelemetry(Configuration);
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        app.UseApplicationInsightsRequestTelemetry();
        app.UseApplicationInsightsExceptionTelemetry();
    }
}
```

2. **Özel telemetri nasıl gönderilir?**
   - **Cevap**:
```csharp
public class TelemetryService
{
    private readonly TelemetryClient _telemetryClient;

    public TelemetryService(TelemetryClient telemetryClient)
    {
        _telemetryClient = telemetryClient;
    }

    public void TrackCustomEvent(string eventName, IDictionary<string, string> properties = null)
    {
        _telemetryClient.TrackEvent(eventName, properties);
    }

    public void TrackMetric(string metricName, double value)
    {
        _telemetryClient.TrackMetric(metricName, value);
    }
}
```

3. **Exception tracking nasıl yapılır?**
   - **Cevap**:
```csharp
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly TelemetryClient _telemetryClient;

    public ExceptionHandlingMiddleware(RequestDelegate next, TelemetryClient telemetryClient)
    {
        _next = next;
        _telemetryClient = telemetryClient;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _telemetryClient.TrackException(ex);
            throw;
        }
    }
}
```

4. **Performance tracking nasıl yapılır?**
   - **Cevap**:
```csharp
public class PerformanceTracker
{
    private readonly TelemetryClient _telemetryClient;

    public PerformanceTracker(TelemetryClient telemetryClient)
    {
        _telemetryClient = telemetryClient;
    }

    public async Task<T> TrackOperationAsync<T>(string operationName, Func<Task<T>> operation)
    {
        var startTime = DateTimeOffset.UtcNow;
        var timer = System.Diagnostics.Stopwatch.StartNew();

        try
        {
            var result = await operation();
            timer.Stop();

            _telemetryClient.TrackDependency("Custom", operationName, 
                startTime, timer.Elapsed, true);

            return result;
        }
        catch (Exception ex)
        {
            timer.Stop();
            _telemetryClient.TrackDependency("Custom", operationName, 
                startTime, timer.Elapsed, false);
            throw;
        }
    }
}
```

5. **Sampling nasıl yapılandırılır?**
   - **Cevap**:
```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddApplicationInsightsTelemetry(options =>
    {
        options.EnableAdaptiveSampling = true;
        options.EnableQuickPulseMetricStream = true;
    });
}
```

### İleri Seviye Sorular

1. **Application Insights performans optimizasyonu nasıl yapılır?**
   - **Cevap**:
     - Sampling stratejileri
     - Telemetri hacmi yönetimi
     - Batch işlemler
     - Async operasyonlar
     - Cache kullanımı

2. **Application Insights güvenliği nasıl sağlanır?**
   - **Cevap**:
     - Hassas veri filtreleme
     - PII veri yönetimi
     - Erişim kontrolü
     - Data retention
     - Audit logging

3. **Application Insights monitoring ve alerting nasıl yapılır?**
   - **Cevap**:
     - Alert kuralları
     - Dashboard tasarımı
     - Metric aggregation
     - Trend analizi
     - Notification kanalları

4. **Application Insights ile log aggregation nasıl yapılır?**
   - **Cevap**:
     - Log entegrasyonu
     - Log analizi
     - Log enrichment
     - Log rotasyonu
     - Log arşivleme

5. **Application Insights ile distributed tracing nasıl yapılır?**
   - **Cevap**:
     - Correlation ID
     - Operation context
     - Dependency tracking
     - End-to-end tracing
     - Performance analysis 