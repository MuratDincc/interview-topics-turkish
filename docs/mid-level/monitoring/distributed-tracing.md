# Distributed Tracing

## Giriş

Distributed Tracing, microservices architecture'da request'lerin farklı servisler arasında nasıl akış gösterdiğini izleyen ve analiz eden teknolojidir. Mid-level geliştiriciler için distributed tracing'i anlamak, performance monitoring, debugging ve system observability için kritik öneme sahiptir. Bu dosya, OpenTelemetry, correlation IDs, span management ve tracing visualization konularını kapsar.

## OpenTelemetry Implementation

### 1. Tracing Service
Distributed tracing implementasyonu.

```csharp
public class TracingService : ITracingService
{
    private readonly ILogger<TracingService> _logger;
    private readonly ActivitySource _activitySource;
    private readonly IConfiguration _configuration;
    
    public TracingService(ILogger<TracingService> logger, IConfiguration configuration)
    {
        _logger = logger;
        _configuration = configuration;
        _activitySource = new ActivitySource("MyApplication");
    }
    
    public Activity StartActivity(string operationName, string operationType = "internal")
    {
        var activity = _activitySource.StartActivity(operationName);
        
        if (activity != null)
        {
            activity.SetTag("operation.type", operationType);
            activity.SetTag("service.name", "MyApplication");
            activity.SetTag("service.version", "1.0.0");
            
            _logger.LogDebug("Started activity: {OperationName}, TraceId: {TraceId}", 
                operationName, activity.TraceId);
        }
        
        return activity;
    }
    
    public Activity StartActivityWithParent(string operationName, ActivityContext parentContext, 
        string operationType = "internal")
    {
        var activity = _activitySource.StartActivity(operationName, ActivityKind.Internal, parentContext);
        
        if (activity != null)
        {
            activity.SetTag("operation.type", operationType);
            activity.SetTag("service.name", "MyApplication");
            activity.SetTag("service.version", "1.0.0");
            
            _logger.LogDebug("Started child activity: {OperationName}, TraceId: {TraceId}, ParentSpanId: {ParentSpanId}", 
                operationName, activity.TraceId, parentContext.SpanId);
        }
        
        return activity;
    }
    
    public void AddTag(Activity activity, string key, string value)
    {
        if (activity != null && !string.IsNullOrEmpty(key))
        {
            activity.SetTag(key, value);
            _logger.LogDebug("Added tag to activity: {Key} = {Value}", key, value);
        }
    }
    
    public void AddEvent(Activity activity, string eventName, Dictionary<string, object> attributes = null)
    {
        if (activity != null && !string.IsNullOrEmpty(eventName))
        {
            var activityAttributes = attributes?.ToDictionary(kvp => kvp.Key, kvp => kvp.Value?.ToString()) 
                ?? new Dictionary<string, string>();
            
            activity.AddEvent(new ActivityEvent(eventName, default, activityAttributes));
            
            _logger.LogDebug("Added event to activity: {EventName}", eventName);
        }
    }
    
    public void SetStatus(Activity activity, ActivityStatusCode status, string description = null)
    {
        if (activity != null)
        {
            activity.SetStatus(status, description);
            
            _logger.LogDebug("Set activity status: {Status}, Description: {Description}", 
                status, description ?? "N/A");
        }
    }
    
    public string GetTraceId(Activity activity)
    {
        return activity?.TraceId.ToString() ?? string.Empty;
    }
    
    public string GetSpanId(Activity activity)
    {
        return activity?.SpanId.ToString() ?? string.Empty;
    }
    
    public ActivityContext GetCurrentContext()
    {
        return Activity.Current?.Context ?? default;
    }
    
    public void InjectTraceContext(HttpRequestMessage request, Activity activity)
    {
        if (activity != null && request != null)
        {
            var traceParent = $"00-{activity.TraceId}-{activity.SpanId}-{activity.ActivityTraceFlags}";
            request.Headers.Add("traceparent", traceParent);
            
            if (!string.IsNullOrEmpty(activity.TraceStateString))
            {
                request.Headers.Add("tracestate", activity.TraceStateString);
            }
            
            _logger.LogDebug("Injected trace context: {TraceParent}", traceParent);
        }
    }
    
    public ActivityContext ExtractTraceContext(HttpRequestMessage request)
    {
        try
        {
            if (request.Headers.TryGetValues("traceparent", out var traceParentValues))
            {
                var traceParent = traceParentValues.FirstOrDefault();
                if (!string.IsNullOrEmpty(traceParent))
                {
                    var parts = traceParent.Split('-');
                    if (parts.Length == 4)
                    {
                        var traceId = ActivityTraceId.CreateFromString(parts[1]);
                        var spanId = ActivitySpanId.CreateFromString(parts[2]);
                        var traceFlags = (ActivityTraceFlags)Convert.ToByte(parts[3], 16);
                        
                        var traceState = string.Empty;
                        if (request.Headers.TryGetValues("tracestate", out var traceStateValues))
                        {
                            traceState = traceStateValues.FirstOrDefault() ?? string.Empty;
                        }
                        
                        var context = new ActivityContext(traceId, spanId, traceFlags, traceState);
                        
                        _logger.LogDebug("Extracted trace context: TraceId: {TraceId}, SpanId: {SpanId}", 
                            traceId, spanId);
                        
                        return context;
                    }
                }
            }
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Error extracting trace context");
        }
        
        return default;
    }
}

public interface ITracingService
{
    Activity StartActivity(string operationName, string operationType = "internal");
    Activity StartActivityWithParent(string operationName, ActivityContext parentContext, string operationType = "internal");
    void AddTag(Activity activity, string key, string value);
    void AddEvent(Activity activity, string eventName, Dictionary<string, object> attributes = null);
    void SetStatus(Activity activity, ActivityStatusCode status, string description = null);
    string GetTraceId(Activity activity);
    string GetSpanId(Activity activity);
    ActivityContext GetCurrentContext();
    void InjectTraceContext(HttpRequestMessage request, Activity activity);
    ActivityContext ExtractTraceContext(HttpRequestMessage request);
}
```

### 2. HTTP Client Tracing
HTTP client'lar için tracing middleware.

```csharp
public class TracingHttpClientHandler : DelegatingHandler
{
    private readonly ITracingService _tracingService;
    private readonly ILogger<TracingHttpClientHandler> _logger;
    
    public TracingHttpClientHandler(ITracingService tracingService, ILogger<TracingHttpClientHandler> logger)
    {
        _tracingService = tracingService;
        _logger = logger;
    }
    
    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        var operationName = $"{request.Method} {request.RequestUri?.Host}{request.RequestUri?.AbsolutePath}";
        var activity = _tracingService.StartActivity(operationName, "http_client");
        
        try
        {
            if (activity != null)
            {
                _tracingService.AddTag(activity, "http.method", request.Method.ToString());
                _tracingService.AddTag(activity, "http.url", request.RequestUri?.ToString());
                _tracingService.AddTag(activity, "http.target", request.RequestUri?.AbsolutePath);
                _tracingService.AddTag(activity, "http.host", request.RequestUri?.Host);
                
                // Inject trace context
                _tracingService.InjectTraceContext(request, activity);
            }
            
            var stopwatch = Stopwatch.StartNew();
            var response = await base.SendAsync(request, cancellationToken);
            stopwatch.Stop();
            
            if (activity != null)
            {
                _tracingService.AddTag(activity, "http.status_code", ((int)response.StatusCode).ToString());
                _tracingService.AddTag(activity, "http.response_time_ms", stopwatch.ElapsedMilliseconds.ToString());
                
                var status = response.IsSuccessStatusCode ? ActivityStatusCode.Ok : ActivityStatusCode.Error;
                _tracingService.SetStatus(activity, status);
            }
            
            return response;
        }
        catch (Exception ex)
        {
            if (activity != null)
            {
                _tracingService.AddTag(activity, "error", "true");
                _tracingService.AddTag(activity, "error.message", ex.Message);
                _tracingService.SetStatus(activity, ActivityStatusCode.Error, ex.Message);
            }
            
            _logger.LogError(ex, "HTTP request failed: {OperationName}", operationName);
            throw;
        }
        finally
        {
            activity?.Dispose();
        }
    }
}
```

## Correlation ID Management

### 1. Correlation Service
Request correlation ID'lerini yöneten servis.

```csharp
public class CorrelationService : ICorrelationService
{
    private readonly ILogger<CorrelationService> _logger;
    private readonly AsyncLocal<string> _correlationId;
    private readonly AsyncLocal<string> _requestId;
    
    public CorrelationService(ILogger<CorrelationService> logger)
    {
        _logger = logger;
        _correlationId = new AsyncLocal<string>();
        _requestId = new AsyncLocal<string>();
    }
    
    public string GetCorrelationId()
    {
        var correlationId = _correlationId.Value;
        
        if (string.IsNullOrEmpty(correlationId))
        {
            correlationId = GenerateCorrelationId();
            _correlationId.Value = correlationId;
            
            _logger.LogDebug("Generated new correlation ID: {CorrelationId}", correlationId);
        }
        
        return correlationId;
    }
    
    public void SetCorrelationId(string correlationId)
    {
        if (!string.IsNullOrEmpty(correlationId))
        {
            _correlationId.Value = correlationId;
            _logger.LogDebug("Set correlation ID: {CorrelationId}", correlationId);
        }
    }
    
    public string GetRequestId()
    {
        var requestId = _requestId.Value;
        
        if (string.IsNullOrEmpty(requestId))
        {
            requestId = GenerateRequestId();
            _requestId.Value = requestId;
            
            _logger.LogDebug("Generated new request ID: {RequestId}", requestId);
        }
        
        return requestId;
    }
    
    public void SetRequestId(string requestId)
    {
        if (!string.IsNullOrEmpty(requestId))
        {
            _requestId.Value = requestId;
            _logger.LogDebug("Set request ID: {RequestId}", requestId);
        }
    }
    
    public void Clear()
    {
        _correlationId.Value = null;
        _requestId.Value = null;
        
        _logger.LogDebug("Cleared correlation and request IDs");
    }
    
    private string GenerateCorrelationId()
    {
        return $"corr-{Guid.NewGuid():N}";
    }
    
    private string GenerateRequestId()
    {
        return $"req-{Guid.NewGuid():N}";
    }
}

public interface ICorrelationService
{
    string GetCorrelationId();
    void SetCorrelationId(string correlationId);
    string GetRequestId();
    void SetRequestId(string requestId);
    void Clear();
}
```

## Mülakat Soruları

### Temel Sorular

1. **Distributed Tracing nedir?**
   - **Cevap**: Microservices arası request flow'u izleme ve analiz etme teknolojisi.

2. **OpenTelemetry nedir?**
   - **Cevap**: Observability için open standard, tracing, metrics, logging.

3. **Correlation ID nedir?**
   - **Cevap**: Request'leri farklı servisler arasında takip etmek için kullanılan unique identifier.

4. **Span ve Trace farkı nedir?**
   - **Cevap**: Span: tek operasyon, Trace: span'lerin collection'ı.

5. **Distributed tracing ne zaman kullanılır?**
   - **Cevap**: Microservices, performance monitoring, debugging, system observability.

### Teknik Sorular

1. **OpenTelemetry nasıl implement edilir?**
   - **Cevap**: ActivitySource, Activity, tags, events, status.

2. **Trace context nasıl propagate edilir?**
   - **Cevap**: HTTP headers, traceparent, tracestate, context injection.

3. **Correlation ID nasıl yönetilir?**
   - **Cevap**: AsyncLocal, middleware, header propagation.

4. **Tracing performance nasıl optimize edilir?**
   - **Cevap**: Sampling, filtering, async operations, resource cleanup.

5. **Tracing data nasıl analyze edilir?**
   - **Cevap**: Jaeger, Zipkin, custom dashboards, metrics aggregation.

## Best Practices

1. **Tracing Implementation**
   - Consistent naming conventions kullanın
   - Meaningful tags ekleyin
   - Error handling implement edin
   - Performance impact minimize edin

2. **Correlation Management**
   - Unique ID generation sağlayın
   - Header propagation implement edin
   - Async context management yapın
   - Cleanup procedures ekleyin

3. **Performance Optimization**
   - Sampling strategies uygulayın
   - Async operations kullanın
   - Resource disposal yapın
   - Monitoring ekleyin

4. **Data Management**
   - Structured logging implement edin
   - Metrics collection yapın
   - Data retention policies uygulayın
   - Privacy considerations göz önünde bulundurun

5. **Integration**
   - OpenTelemetry standards kullanın
   - Multiple backends support edin
   - Custom exporters implement edin
   - Testing strategies ekleyin

## Kaynaklar

- [OpenTelemetry](https://opentelemetry.io/)
- [Distributed Tracing](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/distributed-tracing)
- [Activity API](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.activity)
- [Correlation IDs](https://docs.microsoft.com/en-us/azure/azure-monitor/app/correlation)
- [Jaeger Tracing](https://www.jaegertracing.io/)
