# Profiling

## Genel Bakış
Profiling (performans analizi), bir uygulamanın çalışma zamanındaki davranışını ve performansını analiz etme sürecidir. Bu analiz, performans sorunlarını tespit etmek, darboğazları belirlemek ve optimizasyon fırsatlarını ortaya çıkarmak için kullanılır.

## Temel Kavramlar

### 1. CPU Profiling
```csharp
public class CpuProfiler
{
    private readonly ILogger<CpuProfiler> _logger;
    private readonly Stopwatch _stopwatch = new();

    public CpuProfiler(ILogger<CpuProfiler> logger)
    {
        _logger = logger;
    }

    public async Task<T> ProfileAsync<T>(string operationName, Func<Task<T>> operation)
    {
        _stopwatch.Restart();
        var startCpuTime = Process.GetCurrentProcess().TotalProcessorTime;

        try
        {
            return await operation();
        }
        finally
        {
            _stopwatch.Stop();
            var endCpuTime = Process.GetCurrentProcess().TotalProcessorTime;
            var cpuTime = endCpuTime - startCpuTime;

            _logger.LogInformation(
                "Operation {OperationName} took {ElapsedMilliseconds}ms (CPU: {CpuTime}ms)",
                operationName,
                _stopwatch.ElapsedMilliseconds,
                cpuTime.TotalMilliseconds);
        }
    }

    public void ProfileAction(string operationName, Action operation)
    {
        _stopwatch.Restart();
        var startCpuTime = Process.GetCurrentProcess().TotalProcessorTime;

        try
        {
            operation();
        }
        finally
        {
            _stopwatch.Stop();
            var endCpuTime = Process.GetCurrentProcess().TotalProcessorTime;
            var cpuTime = endCpuTime - startCpuTime;

            _logger.LogInformation(
                "Operation {OperationName} took {ElapsedMilliseconds}ms (CPU: {CpuTime}ms)",
                operationName,
                _stopwatch.ElapsedMilliseconds,
                cpuTime.TotalMilliseconds);
        }
    }
}
```

### 2. Memory Profiling
```csharp
public class MemoryProfiler
{
    private readonly ILogger<MemoryProfiler> _logger;

    public MemoryProfiler(ILogger<MemoryProfiler> logger)
    {
        _logger = logger;
    }

    public void LogMemoryUsage(string context)
    {
        var process = Process.GetCurrentProcess();
        var memoryUsage = process.WorkingSet64;
        var privateMemory = process.PrivateMemorySize64;
        var virtualMemory = process.VirtualMemorySize64;

        _logger.LogInformation(
            "Memory Usage ({Context}): Working Set: {WorkingSet}MB, Private: {PrivateMemory}MB, Virtual: {VirtualMemory}MB",
            context,
            memoryUsage / 1024 / 1024,
            privateMemory / 1024 / 1024,
            virtualMemory / 1024 / 1024);
    }

    public void LogGarbageCollectionStats()
    {
        var gen0Collections = GC.CollectionCount(0);
        var gen1Collections = GC.CollectionCount(1);
        var gen2Collections = GC.CollectionCount(2);
        var totalMemory = GC.GetTotalMemory(false);

        _logger.LogInformation(
            "GC Stats: Gen0: {Gen0}, Gen1: {Gen1}, Gen2: {Gen2}, Total Memory: {TotalMemory}MB",
            gen0Collections,
            gen1Collections,
            gen2Collections,
            totalMemory / 1024 / 1024);
    }
}
```

### 3. Performance Counter
```csharp
public class PerformanceMonitor
{
    private readonly ILogger<PerformanceMonitor> _logger;
    private readonly List<PerformanceCounter> _counters = new();

    public PerformanceMonitor(ILogger<PerformanceMonitor> logger)
    {
        _logger = logger;
        InitializeCounters();
    }

    private void InitializeCounters()
    {
        // CPU kullanımı
        _counters.Add(new PerformanceCounter(
            "Processor",
            "% Processor Time",
            "_Total"));

        // Bellek kullanımı
        _counters.Add(new PerformanceCounter(
            "Memory",
            "Available MBytes"));

        // Disk I/O
        _counters.Add(new PerformanceCounter(
            "PhysicalDisk",
            "% Disk Time",
            "_Total"));
    }

    public void LogPerformanceMetrics()
    {
        foreach (var counter in _counters)
        {
            try
            {
                var value = counter.NextValue();
                _logger.LogInformation(
                    "Performance Counter {CounterName}: {Value}",
                    counter.CounterName,
                    value);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error reading counter {CounterName}", counter.CounterName);
            }
        }
    }
}
```

## Best Practices

### 1. Profiling Stratejisi
- Profiling hedeflerini belirleyin
- Doğru araçları seçin
- Test senaryolarını hazırlayın
- Baseline ölçümleri alın
- Sonuçları dokümante edin

### 2. Performans Metrikleri
- CPU kullanımını izleyin
- Bellek kullanımını takip edin
- I/O operasyonlarını ölçün
- Ağ trafiğini analiz edin
- Response time'ı ölçün

### 3. Analiz ve Raporlama
- Darboğazları tespit edin
- Trend analizi yapın
- Öneriler geliştirin
- Raporları paylaşın
- İyileştirmeleri takip edin

## Sık Sorulan Sorular

### 1. Hangi profiling araçları kullanılabilir?
- Visual Studio Profiler
- dotTrace
- ANTS Performance Profiler
- PerfView
- Application Insights

### 2. Profiling ne zaman yapılmalıdır?
- Performans sorunları yaşandığında
- Yeni özellikler eklendiğinde
- Düzenli performans kontrollerinde
- Kapasite planlaması yaparken
- Optimizasyon öncesi ve sonrası

### 3. Profiling sonuçları nasıl yorumlanır?
- Baseline ile karşılaştırın
- Trend analizi yapın
- Darboğazları belirleyin
- Öneriler geliştirin
- Sonuçları dokümante edin

## Kaynaklar
- [Visual Studio Profiling Tools](https://docs.microsoft.com/tr-tr/visualstudio/profiling/)
- [Application Performance Monitoring](https://docs.microsoft.com/tr-tr/azure/azure-monitor/app/app-insights-overview)
- [Performance Counters in .NET](https://docs.microsoft.com/tr-tr/dotnet/framework/debug-trace-profile/performance-counters)
- [Memory Profiling in .NET](https://docs.microsoft.com/tr-tr/dotnet/standard/garbage-collection/memory-management-and-gc) 