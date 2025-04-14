# OpenTelemetry

## Giriş

OpenTelemetry, modern uygulamalar için açık kaynaklı bir gözlemlenebilirlik (observability) çerçevesidir. Telemetri verilerinin (metrikler, izler ve loglar) toplanması, işlenmesi ve aktarılması için standart bir API ve SDK sağlar. .NET uygulamalarında dağıtık izleme, metrik toplama ve log yönetimi için kullanılır.

## OpenTelemetry'nin Önemi

1. **Standartlaştırma**
   - Tek bir API
   - Farklı sağlayıcılarla uyumluluk
   - Dil bağımsız yaklaşım
   - Topluluk desteği

2. **Esneklik**
   - Özelleştirilebilir yapı
   - Farklı export seçenekleri
   - Zengin enstrümantasyon
   - Modüler mimari

3. **Dağıtık İzleme**
   - End-to-end tracing
   - Context propagation
   - Span ve trace yönetimi
   - Correlation

## OpenTelemetry Kullanımı

1. **Temel Kurulum**
```csharp
// NuGet paketleri:
// OpenTelemetry.Extensions.Hosting
// OpenTelemetry.Instrumentation.AspNetCore
// OpenTelemetry.Exporter.Console
// OpenTelemetry.Exporter.OpenTelemetryProtocol

public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddOpenTelemetry()
            .WithTracing(builder => builder
                .AddAspNetCoreInstrumentation()
                .AddHttpClientInstrumentation()
                .AddConsoleExporter())
            .WithMetrics(builder => builder
                .AddAspNetCoreInstrumentation()
                .AddConsoleExporter());
    }
}
```

2. **Özel Telemetri**
```csharp
public class TelemetryService
{
    private readonly Tracer _tracer;
    private readonly Meter _meter;

    public TelemetryService(TracerProvider tracerProvider, MeterProvider meterProvider)
    {
        _tracer = tracerProvider.GetTracer("MyApp");
        _meter = meterProvider.GetMeter("MyApp");
    }

    public async Task TrackOperationAsync(string operationName, Func<Task> operation)
    {
        using var span = _tracer.StartActiveSpan(operationName);
        try
        {
            await operation();
            span.SetStatus(Status.Ok);
        }
        catch (Exception ex)
        {
            span.SetStatus(Status.Error);
            span.RecordException(ex);
            throw;
        }
    }

    public void TrackMetric(string metricName, double value)
    {
        var counter = _meter.CreateCounter<double>(metricName);
        counter.Add(value);
    }
}
```

3. **Dağıtık İzleme**
```csharp
public class DistributedTracingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly Tracer _tracer;

    public DistributedTracingMiddleware(RequestDelegate next, TracerProvider tracerProvider)
    {
        _next = next;
        _tracer = tracerProvider.GetTracer("MyApp");
    }

    public async Task InvokeAsync(HttpContext context)
    {
        using var span = _tracer.StartActiveSpan("HTTP Request");
        try
        {
            span.SetAttribute("http.method", context.Request.Method);
            span.SetAttribute("http.url", context.Request.Path);

            await _next(context);

            span.SetAttribute("http.status_code", context.Response.StatusCode);
            span.SetStatus(Status.Ok);
        }
        catch (Exception ex)
        {
            span.SetStatus(Status.Error);
            span.RecordException(ex);
            throw;
        }
    }
}
```

4. **Metrik Toplama**
```csharp
public class MetricsCollector
{
    private readonly Meter _meter;
    private readonly Counter<double> _requestCounter;
    private readonly Histogram<double> _responseTimeHistogram;

    public MetricsCollector(MeterProvider meterProvider)
    {
        _meter = meterProvider.GetMeter("MyApp");
        _requestCounter = _meter.CreateCounter<double>("http_requests_total");
        _responseTimeHistogram = _meter.CreateHistogram<double>("http_response_time_seconds");
    }

    public void TrackRequest(string path, double duration)
    {
        _requestCounter.Add(1, new KeyValuePair<string, object>("path", path));
        _responseTimeHistogram.Record(duration, new KeyValuePair<string, object>("path", path));
    }
}
```

## OpenTelemetry Best Practices

1. **İzleme Tasarımı**
   - Anlamlı span isimleri
   - Doğru attribute kullanımı
   - Context propagation
   - Sampling stratejileri

2. **Performans**
   - Span sayısı optimizasyonu
   - Batch işlemler
   - Async operasyonlar
   - Buffer yönetimi

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

1. **OpenTelemetry nedir ve ne için kullanılır?**
   - **Cevap**: OpenTelemetry, modern uygulamalar için açık kaynaklı bir gözlemlenebilirlik çerçevesidir. Telemetri verilerinin toplanması, işlenmesi ve aktarılması için standart bir API ve SDK sağlar.

2. **OpenTelemetry'nin temel özellikleri nelerdir?**
   - **Cevap**:
     - Dağıtık izleme
     - Metrik toplama
     - Log yönetimi
     - Context propagation
     - Sampling

3. **OpenTelemetry'nin avantajları nelerdir?**
   - **Cevap**:
     - Standartlaştırma
     - Esneklik
     - Topluluk desteği
     - Dil bağımsızlık
     - Zengin enstrümantasyon

4. **Span ve Trace nedir?**
   - **Cevap**: Span, bir işlemin başlangıcı ve bitişi arasındaki süreyi temsil eder. Trace ise bir isteğin tüm yaşam döngüsünü kapsayan span'lerin toplamıdır.

5. **Context Propagation nedir?**
   - **Cevap**: Context Propagation, dağıtık sistemlerde trace bilgisinin servisler arasında taşınması sürecidir. Genellikle HTTP header'ları veya mesaj kuyrukları üzerinden yapılır.

### Teknik Sorular

1. **OpenTelemetry nasıl yapılandırılır?**
   - **Cevap**:
```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddOpenTelemetry()
            .WithTracing(builder => builder
                .AddAspNetCoreInstrumentation()
                .AddHttpClientInstrumentation()
                .AddConsoleExporter())
            .WithMetrics(builder => builder
                .AddAspNetCoreInstrumentation()
                .AddConsoleExporter());
    }
}
```

2. **Özel telemetri nasıl gönderilir?**
   - **Cevap**:
```csharp
public class TelemetryService
{
    private readonly Tracer _tracer;
    private readonly Meter _meter;

    public TelemetryService(TracerProvider tracerProvider, MeterProvider meterProvider)
    {
        _tracer = tracerProvider.GetTracer("MyApp");
        _meter = meterProvider.GetMeter("MyApp");
    }

    public async Task TrackOperationAsync(string operationName, Func<Task> operation)
    {
        using var span = _tracer.StartActiveSpan(operationName);
        try
        {
            await operation();
            span.SetStatus(Status.Ok);
        }
        catch (Exception ex)
        {
            span.SetStatus(Status.Error);
            span.RecordException(ex);
            throw;
        }
    }
}
```

3. **Dağıtık izleme nasıl yapılır?**
   - **Cevap**:
```csharp
public class DistributedTracingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly Tracer _tracer;

    public DistributedTracingMiddleware(RequestDelegate next, TracerProvider tracerProvider)
    {
        _next = next;
        _tracer = tracerProvider.GetTracer("MyApp");
    }

    public async Task InvokeAsync(HttpContext context)
    {
        using var span = _tracer.StartActiveSpan("HTTP Request");
        try
        {
            span.SetAttribute("http.method", context.Request.Method);
            span.SetAttribute("http.url", context.Request.Path);

            await _next(context);

            span.SetAttribute("http.status_code", context.Response.StatusCode);
            span.SetStatus(Status.Ok);
        }
        catch (Exception ex)
        {
            span.SetStatus(Status.Error);
            span.RecordException(ex);
            throw;
        }
    }
}
```

4. **Metrik toplama nasıl yapılır?**
   - **Cevap**:
```csharp
public class MetricsCollector
{
    private readonly Meter _meter;
    private readonly Counter<double> _requestCounter;
    private readonly Histogram<double> _responseTimeHistogram;

    public MetricsCollector(MeterProvider meterProvider)
    {
        _meter = meterProvider.GetMeter("MyApp");
        _requestCounter = _meter.CreateCounter<double>("http_requests_total");
        _responseTimeHistogram = _meter.CreateHistogram<double>("http_response_time_seconds");
    }

    public void TrackRequest(string path, double duration)
    {
        _requestCounter.Add(1, new KeyValuePair<string, object>("path", path));
        _responseTimeHistogram.Record(duration, new KeyValuePair<string, object>("path", path));
    }
}
```

5. **Sampling nasıl yapılandırılır?**
   - **Cevap**:
```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddOpenTelemetry()
        .WithTracing(builder => builder
            .SetSampler(new ParentBasedSampler(new TraceIdRatioBasedSampler(0.1)))
            .AddAspNetCoreInstrumentation()
            .AddConsoleExporter());
}
```

### İleri Seviye Sorular

1. **OpenTelemetry performans optimizasyonu nasıl yapılır?**
   - **Cevap**:
     - Sampling stratejileri
     - Span sayısı optimizasyonu
     - Batch işlemler
     - Async operasyonlar
     - Buffer yönetimi

2. **OpenTelemetry güvenliği nasıl sağlanır?**
   - **Cevap**:
     - Hassas veri filtreleme
     - PII veri yönetimi
     - Erişim kontrolü
     - Data retention
     - Audit logging

3. **OpenTelemetry monitoring ve alerting nasıl yapılır?**
   - **Cevap**:
     - Alert kuralları
     - Dashboard tasarımı
     - Metric aggregation
     - Trend analizi
     - Notification kanalları

4. **OpenTelemetry ile log aggregation nasıl yapılır?**
   - **Cevap**:
     - Log entegrasyonu
     - Log analizi
     - Log enrichment
     - Log rotasyonu
     - Log arşivleme

5. **OpenTelemetry ile distributed tracing nasıl yapılır?**
   - **Cevap**:
     - Context propagation
     - Span ve trace yönetimi
     - Correlation
     - End-to-end tracing
     - Performance analysis 