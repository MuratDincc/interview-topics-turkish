# Log Aggregation

## Giriş

Log Aggregation, farklı kaynaklardan gelen log'ları toplayan, işleyen ve analiz eden sistemdir. Mid-level geliştiriciler için log aggregation'ı anlamak, centralized logging, log analysis ve troubleshooting için kritik öneme sahiptir. Bu dosya, log collection, log processing, log storage ve log analysis konularını kapsar.

## Log Collection Service

### 1. Centralized Logging
Merkezi log collection implementasyonu.

```csharp
public class LogAggregationService : ILogAggregationService
{
    private readonly ILogger<LogAggregationService> _logger;
    private readonly IConfiguration _configuration;
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly ConcurrentQueue<LogEntry> _logQueue;
    private readonly Timer _flushTimer;
    private readonly int _batchSize;
    private readonly string _logEndpoint;
    
    public LogAggregationService(ILogger<LogAggregationService> logger, IConfiguration configuration, 
        IHttpClientFactory httpClientFactory)
    {
        _logger = logger;
        _configuration = configuration;
        _httpClientFactory = httpClientFactory;
        _logQueue = new ConcurrentQueue<LogEntry>();
        
        _batchSize = _configuration.GetValue<int>("LogAggregation:BatchSize", 100);
        _logEndpoint = _configuration["LogAggregation:Endpoint"] ?? "http://localhost:5000/api/logs";
        
        var flushInterval = _configuration.GetValue<int>("LogAggregation:FlushIntervalSeconds", 30);
        _flushTimer = new Timer(FlushLogs, null, TimeSpan.FromSeconds(flushInterval), TimeSpan.FromSeconds(flushInterval));
    }
    
    public void AddLog(LogEntry logEntry)
    {
        try
        {
            // Enrich log entry with additional context
            logEntry.Timestamp = DateTime.UtcNow;
            logEntry.HostName = Environment.MachineName;
            logEntry.ProcessId = Environment.ProcessId;
            logEntry.ApplicationName = _configuration["Application:Name"] ?? "Unknown";
            logEntry.ApplicationVersion = _configuration["Application:Version"] ?? "1.0.0";
            
            _logQueue.Enqueue(logEntry);
            
            // Flush immediately if queue is full
            if (_logQueue.Count >= _batchSize)
            {
                _ = Task.Run(FlushLogsAsync);
            }
            
            _logger.LogDebug("Log entry added to queue: {Level}, {Message}", logEntry.Level, logEntry.Message);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error adding log entry to queue");
        }
    }
    
    public void AddLog(LogLevel level, string message, string category = null, 
        Dictionary<string, object> properties = null, Exception exception = null)
    {
        var logEntry = new LogEntry
        {
            Level = level.ToString(),
            Message = message,
            Category = category ?? "General",
            Properties = properties ?? new Dictionary<string, object>(),
            Exception = exception?.ToString(),
            StackTrace = exception?.StackTrace
        };
        
        AddLog(logEntry);
    }
    
    private async void FlushLogs(object state)
    {
        await FlushLogsAsync();
    }
    
    private async Task FlushLogsAsync()
    {
        try
        {
            var logsToSend = new List<LogEntry>();
            
            // Dequeue logs from queue
            while (logsToSend.Count < _batchSize && _logQueue.TryDequeue(out var logEntry))
            {
                logsToSend.Add(logEntry);
            }
            
            if (logsToSend.Any())
            {
                await SendLogsAsync(logsToSend);
                _logger.LogDebug("Flushed {Count} log entries", logsToSend.Count);
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error flushing logs");
        }
    }
    
    private async Task SendLogsAsync(List<LogEntry> logs)
    {
        try
        {
            var client = _httpClientFactory.CreateClient("LogAggregation");
            
            var request = new HttpRequestMessage(HttpMethod.Post, _logEndpoint)
            {
                Content = new StringContent(JsonSerializer.Serialize(logs), Encoding.UTF8, "application/json")
            };
            
            var response = await client.SendAsync(request);
            
            if (!response.IsSuccessStatusCode)
            {
                var errorContent = await response.Content.ReadAsStringAsync();
                _logger.LogWarning("Failed to send logs. Status: {Status}, Error: {Error}", 
                    response.StatusCode, errorContent);
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error sending logs to aggregation service");
            
            // Re-queue failed logs for retry
            foreach (var log in logs)
            {
                _logQueue.Enqueue(log);
            }
        }
    }
    
    public async Task<List<LogEntry>> SearchLogsAsync(LogSearchCriteria criteria)
    {
        try
        {
            var client = _httpClientFactory.CreateClient("LogAggregation");
            
            var queryString = BuildQueryString(criteria);
            var request = new HttpRequestMessage(HttpMethod.Get, $"{_logEndpoint}/search?{queryString}");
            
            var response = await client.SendAsync(request);
            
            if (response.IsSuccessStatusCode)
            {
                var content = await response.Content.ReadAsStringAsync();
                var searchResults = JsonSerializer.Deserialize<List<LogEntry>>(content);
                
                _logger.LogDebug("Log search completed. Found {Count} results", searchResults?.Count ?? 0);
                return searchResults ?? new List<LogEntry>();
            }
            else
            {
                var errorContent = await response.Content.ReadAsStringAsync();
                _logger.LogWarning("Log search failed. Status: {Status}, Error: {Error}", 
                    response.StatusCode, errorContent);
                return new List<LogEntry>();
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error searching logs");
            return new List<LogEntry>();
        }
    }
    
    private string BuildQueryString(LogSearchCriteria criteria)
    {
        var queryParams = new List<string>();
        
        if (!string.IsNullOrEmpty(criteria.Level))
            queryParams.Add($"level={Uri.EscapeDataString(criteria.Level)}");
        
        if (!string.IsNullOrEmpty(criteria.Category))
            queryParams.Add($"category={Uri.EscapeDataString(criteria.Category)}");
        
        if (!string.IsNullOrEmpty(criteria.SearchTerm))
            queryParams.Add($"search={Uri.EscapeDataString(criteria.SearchTerm)}");
        
        if (criteria.FromDate.HasValue)
            queryParams.Add($"from={criteria.FromDate.Value:yyyy-MM-ddTHH:mm:ssZ}");
        
        if (criteria.ToDate.HasValue)
            queryParams.Add($"to={criteria.ToDate.Value:yyyy-MM-ddTHH:mm:ssZ}");
        
        if (criteria.Limit.HasValue)
            queryParams.Add($"limit={criteria.Limit.Value}");
        
        return string.Join("&", queryParams);
    }
    
    public void Dispose()
    {
        _flushTimer?.Dispose();
    }
}

public interface ILogAggregationService
{
    void AddLog(LogEntry logEntry);
    void AddLog(LogLevel level, string message, string category = null, 
        Dictionary<string, object> properties = null, Exception exception = null);
    Task<List<LogEntry>> SearchLogsAsync(LogSearchCriteria criteria);
}

public class LogEntry
{
    public string Id { get; set; } = Guid.NewGuid().ToString();
    public DateTime Timestamp { get; set; }
    public string Level { get; set; }
    public string Message { get; set; }
    public string Category { get; set; }
    public string HostName { get; set; }
    public int ProcessId { get; set; }
    public string ApplicationName { get; set; }
    public string ApplicationVersion { get; set; }
    public Dictionary<string, object> Properties { get; set; } = new();
    public string Exception { get; set; }
    public string StackTrace { get; set; }
    public string CorrelationId { get; set; }
    public string RequestId { get; set; }
}

public class LogSearchCriteria
{
    public string Level { get; set; }
    public string Category { get; set; }
    public string SearchTerm { get; set; }
    public DateTime? FromDate { get; set; }
    public DateTime? ToDate { get; set; }
    public int? Limit { get; set; }
}
```

### 2. Log Processing Pipeline
Log'ları işleyen pipeline.

```csharp
public class LogProcessingPipeline : ILogProcessingPipeline
{
    private readonly ILogger<LogProcessingPipeline> _logger;
    private readonly IEnumerable<ILogProcessor> _processors;
    private readonly IConfiguration _configuration;
    
    public LogProcessingPipeline(ILogger<LogProcessingPipeline> logger, 
        IEnumerable<ILogProcessor> processors, IConfiguration configuration)
    {
        _logger = logger;
        _processors = processors;
        _configuration = configuration;
    }
    
    public async Task<ProcessedLogEntry> ProcessLogAsync(LogEntry logEntry)
    {
        try
        {
            var processedLog = new ProcessedLogEntry(logEntry);
            
            foreach (var processor in _processors.OrderBy(p => p.Priority))
            {
                try
                {
                    await processor.ProcessAsync(processedLog);
                    _logger.LogDebug("Log processed by {ProcessorType}", processor.GetType().Name);
                }
                catch (Exception ex)
                {
                    _logger.LogWarning(ex, "Error processing log with {ProcessorType}", processor.GetType().Name);
                }
            }
            
            return processedLog;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error in log processing pipeline");
            throw;
        }
    }
    
    public async Task<List<ProcessedLogEntry>> ProcessLogsAsync(List<LogEntry> logEntries)
    {
        var processedLogs = new List<ProcessedLogEntry>();
        
        foreach (var logEntry in logEntries)
        {
            try
            {
                var processedLog = await ProcessLogAsync(logEntry);
                processedLogs.Add(processedLog);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing log entry: {Id}", logEntry.Id);
            }
        }
        
        return processedLogs;
    }
}

public interface ILogProcessingPipeline
{
    Task<ProcessedLogEntry> ProcessLogAsync(LogEntry logEntry);
    Task<List<ProcessedLogEntry>> ProcessLogsAsync(List<LogEntry> logEntries);
}

public interface ILogProcessor
{
    int Priority { get; }
    Task ProcessAsync(ProcessedLogEntry logEntry);
}

public class ProcessedLogEntry
{
    public LogEntry OriginalLog { get; }
    public Dictionary<string, object> EnrichedData { get; }
    public List<string> Tags { get; }
    public Dictionary<string, object> Metrics { get; }
    
    public ProcessedLogEntry(LogEntry originalLog)
    {
        OriginalLog = originalLog;
        EnrichedData = new Dictionary<string, object>();
        Tags = new List<string>();
        Metrics = new Dictionary<string, object>();
    }
}

// Sample processors
public class LogEnrichmentProcessor : ILogProcessor
{
    public int Priority => 1;
    
    public Task ProcessAsync(ProcessedLogEntry logEntry)
    {
        // Add environment information
        logEntry.EnrichedData["Environment"] = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") ?? "Unknown";
        logEntry.EnrichedData["MachineName"] = Environment.MachineName;
        logEntry.EnrichedData["OSVersion"] = Environment.OSVersion.ToString();
        
        // Add performance metrics
        logEntry.Metrics["MemoryUsageMB"] = GC.GetTotalMemory(false) / (1024 * 1024);
        logEntry.Metrics["ThreadCount"] = Process.GetCurrentProcess().Threads.Count;
        
        return Task.CompletedTask;
    }
}

public class LogTaggingProcessor : ILogProcessor
{
    public int Priority => 2;
    
    public Task ProcessAsync(ProcessedLogEntry logEntry)
    {
        // Add tags based on log level
        if (logEntry.OriginalLog.Level == "Error" || logEntry.OriginalLog.Level == "Critical")
        {
            logEntry.Tags.Add("high-priority");
        }
        
        // Add tags based on category
        if (logEntry.OriginalLog.Category?.Contains("Security") == true)
        {
            logEntry.Tags.Add("security");
        }
        
        // Add tags based on content
        if (logEntry.OriginalLog.Message?.Contains("Exception") == true)
        {
            logEntry.Tags.Add("exception");
        }
        
        return Task.CompletedTask;
    }
}
```

## Mülakat Soruları

### Temel Sorular

1. **Log aggregation nedir?**
   - **Cevap**: Farklı kaynaklardan log'ları toplama, işleme ve analiz etme.

2. **Centralized logging nedir?**
   - **Cevap**: Tüm log'ları merkezi bir sistemde toplama ve yönetme.

3. **Log processing pipeline nedir?**
   - **Cevap**: Log'ları sırayla işleyen ve zenginleştiren sistem.

4. **Structured logging nedir?**
   - **Cevap**: JSON formatında structured data olarak log yazma.

5. **Log correlation nedir?**
   - **Cevap**: Request'leri farklı servisler arasında takip etme.

### Teknik Sorular

1. **Log aggregation nasıl implement edilir?**
   - **Cevap**: Centralized service, batching, async processing, retry logic.

2. **Log processing nasıl optimize edilir?**
   - **Cevap**: Parallel processing, caching, filtering, compression.

3. **Log storage nasıl yapılır?**
   - **Cevap**: Time-series databases, Elasticsearch, data retention, indexing.

4. **Log search nasıl implement edilir?**
   - **Cevap**: Full-text search, filtering, pagination, real-time search.

5. **Log analysis nasıl yapılır?**
   - **Cevap**: Aggregation queries, trend analysis, anomaly detection, reporting.

## Best Practices

1. **Log Collection**
   - Structured logging kullanın
   - Batch processing implement edin
   - Error handling ekleyin
   - Performance monitoring yapın

2. **Log Processing**
   - Pipeline architecture kullanın
   - Priority-based processing implement edin
   - Error isolation sağlayın
   - Monitoring ekleyin

3. **Log Storage**
   - Efficient indexing implement edin
   - Data retention policies uygulayın
   - Compression strategies kullanın
   - Backup procedures ekleyin

4. **Log Analysis**
   - Real-time monitoring implement edin
   - Alerting systems kurun
   - Dashboard creation yapın
   - Trend analysis ekleyin

5. **Performance & Scalability**
   - Async processing kullanın
   - Horizontal scaling implement edin
   - Caching strategies uygulayın
   - Load balancing ekleyin

## Kaynaklar

- [Log Aggregation](https://docs.microsoft.com/en-us/azure/architecture/patterns/log-aggregation)
- [Structured Logging](https://docs.microsoft.com/en-us/dotnet/core/extensions/logging)
- [Serilog](https://serilog.net/)
- [ELK Stack](https://www.elastic.co/what-is/elk-stack)
- [Log Processing](https://docs.microsoft.com/en-us/azure/azure-monitor/logs/log-query-overview)
