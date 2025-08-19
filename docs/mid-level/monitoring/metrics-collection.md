# Metrics Collection

## Giriş

Metrics Collection, uygulama performansını, sistem sağlığını ve business metrics'leri izlemek için kullanılan sistemdir. Mid-level geliştiriciler için metrics collection'ı anlamak, performance monitoring, alerting ve capacity planning için kritik öneme sahiptir. Bu dosya, custom metrics, performance counters, metrics aggregation ve monitoring konularını kapsar.

## Custom Metrics Implementation

### 1. Metrics Service
Custom metrics collection implementasyonu.

```csharp
public class MetricsService : IMetricsService
{
    private readonly ILogger<MetricsService> _logger;
    private readonly Dictionary<string, Counter> _counters;
    private readonly Dictionary<string, Gauge> _gauges;
    private readonly Dictionary<string, Histogram> _histograms;
    private readonly object _lock = new object();
    
    public MetricsService(ILogger<MetricsService> logger)
    {
        _logger = logger;
        _counters = new Dictionary<string, Counter>();
        _gauges = new Dictionary<string, Gauge>();
        _histograms = new Dictionary<string, Histogram>();
    }
    
    public void IncrementCounter(string name, Dictionary<string, object> labels = null)
    {
        lock (_lock)
        {
            if (!_counters.ContainsKey(name))
            {
                _counters[name] = new Counter(name);
            }
            
            _counters[name].Increment(labels);
            _logger.LogDebug("Incremented counter: {Name}", name);
        }
    }
    
    public void SetGauge(string name, double value, Dictionary<string, object> labels = null)
    {
        lock (_lock)
        {
            if (!_gauges.ContainsKey(name))
            {
                _gauges[name] = new Gauge(name);
            }
            
            _gauges[name].SetValue(value, labels);
            _logger.LogDebug("Set gauge: {Name} = {Value}", name, value);
        }
    }
    
    public void RecordHistogram(string name, double value, Dictionary<string, object> labels = null)
    {
        lock (_lock)
        {
            if (!_histograms.ContainsKey(name))
            {
                _histograms[name] = new Histogram(name);
            }
            
            _histograms[name].RecordValue(value, labels);
            _logger.LogDebug("Recorded histogram: {Name} = {Value}", name, value);
        }
    }
    
    public MetricsSnapshot GetSnapshot()
    {
        lock (_lock)
        {
            return new MetricsSnapshot
            {
                Counters = _counters.ToDictionary(kvp => kvp.Key, kvp => kvp.Value.GetSnapshot()),
                Gauges = _gauges.ToDictionary(kvp => kvp.Key, kvp => kvp.Value.GetSnapshot()),
                Histograms = _histograms.ToDictionary(kvp => kvp.Key, kvp => kvp.Value.GetSnapshot()),
                Timestamp = DateTime.UtcNow
            };
        }
    }
    
    public void Reset()
    {
        lock (_lock)
        {
            _counters.Clear();
            _gauges.Clear();
            _histograms.Clear();
            _logger.LogInformation("All metrics reset");
        }
    }
}

public class Counter
{
    private readonly string _name;
    private readonly Dictionary<string, long> _values;
    
    public Counter(string name)
    {
        _name = name;
        _values = new Dictionary<string, long>();
    }
    
    public void Increment(Dictionary<string, object> labels = null)
    {
        var key = GetLabelKey(labels);
        if (!_values.ContainsKey(key))
        {
            _values[key] = 0;
        }
        
        _values[key]++;
    }
    
    public CounterSnapshot GetSnapshot()
    {
        return new CounterSnapshot
        {
            Name = _name,
            Values = new Dictionary<string, long>(_values)
        };
    }
    
    private string GetLabelKey(Dictionary<string, object> labels)
    {
        if (labels == null || !labels.Any())
            return "default";
        
        return string.Join("|", labels.OrderBy(kvp => kvp.Key).Select(kvp => $"{kvp.Key}={kvp.Value}"));
    }
}

public class Gauge
{
    private readonly string _name;
    private readonly Dictionary<string, double> _values;
    
    public Gauge(string name)
    {
        _name = name;
        _values = new Dictionary<string, double>();
    }
    
    public void SetValue(double value, Dictionary<string, object> labels = null)
    {
        var key = GetLabelKey(labels);
        _values[key] = value;
    }
    
    public GaugeSnapshot GetSnapshot()
    {
        return new GaugeSnapshot
        {
            Name = _name,
            Values = new Dictionary<string, double>(_values)
        };
    }
    
    private string GetLabelKey(Dictionary<string, object> labels)
    {
        if (labels == null || !labels.Any())
            return "default";
        
        return string.Join("|", labels.OrderBy(kvp => kvp.Key).Select(kvp => $"{kvp.Key}={kvp.Value}"));
    }
}

public class Histogram
{
    private readonly string _name;
    private readonly Dictionary<string, List<double>> _values;
    
    public Histogram(string name)
    {
        _name = name;
        _values = new Dictionary<string, List<double>>();
    }
    
    public void RecordValue(double value, Dictionary<string, object> labels = null)
    {
        var key = GetLabelKey(labels);
        if (!_values.ContainsKey(key))
        {
            _values[key] = new List<double>();
        }
        
        _values[key].Add(value);
    }
    
    public HistogramSnapshot GetSnapshot()
    {
        var snapshots = new Dictionary<string, HistogramStats>();
        
        foreach (var kvp in _values)
        {
            var values = kvp.Value;
            if (values.Any())
            {
                snapshots[kvp.Key] = new HistogramStats
                {
                    Count = values.Count,
                    Sum = values.Sum(),
                    Min = values.Min(),
                    Max = values.Max(),
                    Average = values.Average(),
                    Percentile95 = CalculatePercentile(values, 95),
                    Percentile99 = CalculatePercentile(values, 99)
                };
            }
        }
        
        return new HistogramSnapshot
        {
            Name = _name,
            Stats = snapshots
        };
    }
    
    private double CalculatePercentile(List<double> values, int percentile)
    {
        if (!values.Any()) return 0;
        
        var sorted = values.OrderBy(v => v).ToList();
        var index = (int)Math.Ceiling(percentile / 100.0 * sorted.Count) - 1;
        return sorted[Math.Max(0, index)];
    }
    
    private string GetLabelKey(Dictionary<string, object> labels)
    {
        if (labels == null || !labels.Any())
            return "default";
        
        return string.Join("|", labels.OrderBy(kvp => kvp.Key).Select(kvp => $"{kvp.Key}={kvp.Value}"));
    }
}

public interface IMetricsService
{
    void IncrementCounter(string name, Dictionary<string, object> labels = null);
    void SetGauge(string name, double value, Dictionary<string, object> labels = null);
    void RecordHistogram(string name, double value, Dictionary<string, object> labels = null);
    MetricsSnapshot GetSnapshot();
    void Reset();
}

public class MetricsSnapshot
{
    public Dictionary<string, CounterSnapshot> Counters { get; set; } = new();
    public Dictionary<string, GaugeSnapshot> Gauges { get; set; } = new();
    public Dictionary<string, HistogramSnapshot> Histograms { get; set; } = new();
    public DateTime Timestamp { get; set; }
}

public class CounterSnapshot
{
    public string Name { get; set; }
    public Dictionary<string, long> Values { get; set; } = new();
}

public class GaugeSnapshot
{
    public string Name { get; set; }
    public Dictionary<string, double> Values { get; set; } = new();
}

public class HistogramSnapshot
{
    public string Name { get; set; }
    public Dictionary<string, HistogramStats> Stats { get; set; } = new();
}

public class HistogramStats
{
    public int Count { get; set; }
    public double Sum { get; set; }
    public double Min { get; set; }
    public double Max { get; set; }
    public double Average { get; set; }
    public double Percentile95 { get; set; }
    public double Percentile99 { get; set; }
}
```

### 2. Performance Metrics Collector
Performance metrics'leri toplayan servis.

```csharp
public class PerformanceMetricsCollector : BackgroundService
{
    private readonly ILogger<PerformanceMetricsCollector> _logger;
    private readonly IMetricsService _metricsService;
    private readonly IConfiguration _configuration;
    private readonly Timer _timer;
    
    public PerformanceMetricsCollector(ILogger<PerformanceMetricsCollector> logger, 
        IMetricsService metricsService, IConfiguration configuration)
    {
        _logger = logger;
        _metricsService = metricsService;
        _configuration = configuration;
        
        var interval = _configuration.GetValue<int>("Metrics:CollectionIntervalSeconds", 60);
        _timer = new Timer(CollectMetrics, null, TimeSpan.Zero, TimeSpan.FromSeconds(interval));
    }
    
    protected override Task ExecuteAsync(CancellationToken stoppingToken)
    {
        return Task.CompletedTask;
    }
    
    private async void CollectMetrics(object state)
    {
        try
        {
            await CollectSystemMetricsAsync();
            await CollectApplicationMetricsAsync();
            await CollectBusinessMetricsAsync();
            
            _logger.LogDebug("Performance metrics collected successfully");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error collecting performance metrics");
        }
    }
    
    private async Task CollectSystemMetricsAsync()
    {
        try
        {
            // CPU usage
            var cpuUsage = await GetCpuUsageAsync();
            _metricsService.SetGauge("system.cpu.usage_percent", cpuUsage);
            
            // Memory usage
            var memoryInfo = GC.GetTotalMemory(false);
            var workingSet = Environment.WorkingSet;
            var memoryUsagePercent = (double)workingSet / memoryInfo * 100;
            
            _metricsService.SetGauge("system.memory.total_bytes", memoryInfo);
            _metricsService.SetGauge("system.memory.working_set_bytes", workingSet);
            _metricsService.SetGauge("system.memory.usage_percent", memoryUsagePercent);
            
            // Process metrics
            var process = Process.GetCurrentProcess();
            _metricsService.SetGauge("system.process.cpu_time_ms", process.TotalProcessorTime.TotalMilliseconds);
            _metricsService.SetGauge("system.process.thread_count", process.Threads.Count);
            _metricsService.SetGauge("system.process.handle_count", process.HandleCount);
            
            await Task.CompletedTask;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error collecting system metrics");
        }
    }
    
    private async Task CollectApplicationMetricsAsync()
    {
        try
        {
            // GC metrics
            var gen0Collections = GC.CollectionCount(0);
            var gen1Collections = GC.CollectionCount(1);
            var gen2Collections = GC.CollectionCount(2);
            
            _metricsService.SetGauge("application.gc.gen0_collections", gen0Collections);
            _metricsService.SetGauge("application.gc.gen1_collections", gen1Collections);
            _metricsService.SetGauge("application.gc.gen2_collections", gen2Collections);
            
            // Thread pool metrics
            ThreadPool.GetAvailableThreads(out var workerThreads, out var completionPortThreads);
            ThreadPool.GetMaxThreads(out var maxWorkerThreads, out var maxCompletionPortThreads);
            
            _metricsService.SetGauge("application.threadpool.available_worker_threads", workerThreads);
            _metricsService.SetGauge("application.threadpool.available_completion_port_threads", completionPortThreads);
            _metricsService.SetGauge("application.threadpool.max_worker_threads", maxWorkerThreads);
            _metricsService.SetGauge("application.threadpool.max_completion_port_threads", maxCompletionPortThreads);
            
            await Task.CompletedTask;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error collecting application metrics");
        }
    }
    
    private async Task CollectBusinessMetricsAsync()
    {
        try
        {
            // Business-specific metrics can be added here
            // For example: active users, request rates, business transactions
            
            await Task.CompletedTask;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error collecting business metrics");
        }
    }
    
    private async Task<double> GetCpuUsageAsync()
    {
        try
        {
            var process = Process.GetCurrentProcess();
            var startTime = process.TotalProcessorTime;
            var startCpuTime = Environment.TickCount;
            
            await Task.Delay(100); // Wait 100ms to get CPU usage
            
            var endTime = process.TotalProcessorTime;
            var endCpuTime = Environment.TickCount;
            
            var cpuUsedMs = (endTime - startTime).TotalMilliseconds;
            var totalMsPassed = endCpuTime - startCpuTime;
            var cpuUsageTotal = cpuUsedMs / (Environment.ProcessorCount * totalMsPassed);
            
            return cpuUsageTotal * 100;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting CPU usage");
            return 0;
        }
    }
    
    public override void Dispose()
    {
        _timer?.Dispose();
        base.Dispose();
    }
}
```

## Mülakat Soruları

### Temel Sorular

1. **Metrics collection nedir?**
   - **Cevap**: Uygulama ve sistem performansını izlemek için veri toplama.

2. **Counter, Gauge, Histogram farkları nelerdir?**
   - **Cevap**: Counter: artan değer, Gauge: anlık değer, Histogram: dağılım.

3. **Performance metrics nelerdir?**
   - **Cevap**: CPU, memory, response time, throughput, error rate.

4. **Business metrics nedir?**
   - **Cevap**: İş süreçlerini ölçen metrikler (user count, transaction rate).

5. **Metrics aggregation nasıl yapılır?**
   - **Cevap**: Sum, average, percentiles, time-based grouping.

### Teknik Sorular

1. **Custom metrics nasıl implement edilir?**
   - **Cevap**: IMetricsService interface, Counter/Gauge/Histogram classes.

2. **Performance counters nasıl kullanılır?**
   - **Cevap**: System.Diagnostics.PerformanceCounter, custom counters.

3. **Metrics sampling nasıl yapılır?**
   - **Cevap**: Time-based sampling, rate limiting, adaptive sampling.

4. **Metrics storage nasıl yapılır?**
   - **Cevap**: Time-series databases, Prometheus, InfluxDB.

5. **Metrics visualization nasıl yapılır?**
   - **Cevap**: Grafana, custom dashboards, real-time monitoring.

## Best Practices

1. **Metrics Design**
   - Meaningful names kullanın
   - Consistent labeling implement edin
   - Cardinality control edin
   - Documentation sağlayın

2. **Performance Optimization**
   - Efficient collection implement edin
   - Sampling strategies uygulayın
   - Background collection kullanın
   - Resource usage minimize edin

3. **Data Management**
   - Retention policies uygulayın
   - Data aggregation yapın
   - Storage optimization sağlayın
   - Backup strategies implement edin

4. **Monitoring & Alerting**
   - Threshold-based alerting kurun
   - Trend analysis implement edin
   - Anomaly detection ekleyin
   - Escalation procedures tanımlayın

5. **Integration**
   - Standard formats kullanın
   - Multiple backends support edin
   - Export capabilities ekleyin
   - Testing strategies implement edin

## Kaynaklar

- [Metrics Collection](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/metrics)
- [Performance Counters](https://docs.microsoft.com/en-us/dotnet/framework/debug-trace-profile/performance-counters)
- [OpenTelemetry Metrics](https://opentelemetry.io/docs/concepts/signals/metrics/)
- [Prometheus Metrics](https://prometheus.io/docs/concepts/metric_types/)
- [Grafana Dashboards](https://grafana.com/docs/grafana/latest/dashboards/)
