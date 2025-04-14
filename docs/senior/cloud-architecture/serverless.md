# Serverless Architecture

## Genel Bakış
Serverless mimari, geliştiricilerin sunucu yönetimi olmadan uygulama geliştirmesine ve çalıştırmasına olanak tanıyan bir bulut bilişim modelidir. Bu modelde, uygulamalar olay odaklı (event-driven) olarak çalışır ve sadece kullanıldığında kaynak tüketir.

## Temel Kavramlar

### 1. Azure Functions
```csharp
public class AzureFunctionService
{
    private readonly ILogger<AzureFunctionService> _logger;

    public AzureFunctionService(ILogger<AzureFunctionService> logger)
    {
        _logger = logger;
    }

    [FunctionName("ProcessOrder")]
    public async Task<IActionResult> ProcessOrder(
        [HttpTrigger(AuthorizationLevel.Function, "post", Route = null)] HttpRequest req)
    {
        _logger.LogInformation("Order processing started");

        string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
        var order = JsonSerializer.Deserialize<Order>(requestBody);

        // Sipariş işleme mantığı
        await ProcessOrderAsync(order);

        return new OkObjectResult("Order processed successfully");
    }

    private async Task ProcessOrderAsync(Order order)
    {
        // Sipariş işleme işlemleri
        await Task.Delay(1000); // Simüle edilmiş işlem
        _logger.LogInformation($"Order {order.Id} processed");
    }
}
```

### 2. AWS Lambda
```csharp
public class AWSLambdaService
{
    private readonly ILogger<AWSLambdaService> _logger;

    public AWSLambdaService(ILogger<AWSLambdaService> logger)
    {
        _logger = logger;
    }

    [LambdaFunction]
    public async Task<APIGatewayProxyResponse> ProcessRequest(
        APIGatewayProxyRequest request,
        ILambdaContext context)
    {
        _logger.LogInformation("Request processing started");

        var body = JsonSerializer.Deserialize<RequestData>(request.Body);

        // İstek işleme mantığı
        var result = await ProcessRequestAsync(body);

        return new APIGatewayProxyResponse
        {
            StatusCode = 200,
            Body = JsonSerializer.Serialize(result)
        };
    }

    private async Task<ResponseData> ProcessRequestAsync(RequestData data)
    {
        // İstek işleme işlemleri
        await Task.Delay(1000); // Simüle edilmiş işlem
        return new ResponseData { Success = true };
    }
}
```

### 3. Event-Driven Architecture
```csharp
public class EventDrivenService
{
    private readonly ILogger<EventDrivenService> _logger;
    private readonly IEventGridClient _eventGridClient;

    public EventDrivenService(
        ILogger<EventDrivenService> logger,
        IEventGridClient eventGridClient)
    {
        _logger = logger;
        _eventGridClient = eventGridClient;
    }

    public async Task PublishEventAsync(EventData data)
    {
        var @event = new EventGridEvent
        {
            Id = Guid.NewGuid().ToString(),
            EventType = data.EventType,
            Data = data,
            EventTime = DateTime.UtcNow,
            Subject = data.Subject,
            DataVersion = "1.0"
        };

        await _eventGridClient.PublishEventsAsync(
            "topic-name",
            new List<EventGridEvent> { @event });

        _logger.LogInformation($"Event published: {data.EventType}");
    }

    [FunctionName("ProcessEvent")]
    public async Task ProcessEvent(
        [EventGridTrigger] EventGridEvent eventGridEvent)
    {
        _logger.LogInformation($"Event received: {eventGridEvent.EventType}");

        var data = eventGridEvent.Data.ToObjectFromJson<EventData>();
        await ProcessEventDataAsync(data);
    }

    private async Task ProcessEventDataAsync(EventData data)
    {
        // Olay işleme mantığı
        await Task.Delay(1000); // Simüle edilmiş işlem
        _logger.LogInformation($"Event processed: {data.EventType}");
    }
}
```

### 4. Serverless Monitoring
```csharp
public class ServerlessMonitoringService
{
    private readonly ILogger<ServerlessMonitoringService> _logger;
    private readonly IApplicationInsightsClient _appInsightsClient;

    public ServerlessMonitoringService(
        ILogger<ServerlessMonitoringService> logger,
        IApplicationInsightsClient appInsightsClient)
    {
        _logger = logger;
        _appInsightsClient = appInsightsClient;
    }

    public async Task TrackFunctionExecutionAsync(
        string functionName,
        string operationId,
        TimeSpan duration,
        bool success)
    {
        var telemetry = new RequestTelemetry
        {
            Name = functionName,
            Id = operationId,
            Duration = duration,
            Success = success,
            Timestamp = DateTime.UtcNow
        };

        await _appInsightsClient.TrackRequestAsync(telemetry);
        _logger.LogInformation($"Function execution tracked: {functionName}");
    }

    public async Task TrackExceptionAsync(
        string functionName,
        Exception exception)
    {
        var telemetry = new ExceptionTelemetry
        {
            Exception = exception,
            Message = exception.Message,
            Timestamp = DateTime.UtcNow
        };

        await _appInsightsClient.TrackExceptionAsync(telemetry);
        _logger.LogError(exception, $"Exception in function: {functionName}");
    }
}
```

## Best Practices

### 1. Function Tasarımı
- Stateless fonksiyonlar
- Kısa çalışma süreleri
- Bağımlılık enjeksiyonu
- Hata yönetimi
- Retry politikaları

### 2. Performans Optimizasyonu
- Cold start azaltma
- Memory optimizasyonu
- Timeout yönetimi
- Concurrent execution
- Caching stratejileri

### 3. Güvenlik
- IAM yapılandırması
- API Gateway güvenliği
- Environment variables
- Secret management
- Network isolation

## Sık Sorulan Sorular

### 1. Serverless mimarinin avantajları nelerdir?
- Otomatik ölçeklendirme
- Maliyet optimizasyonu
- Operasyonel yük azaltma
- Hızlı geliştirme
- Yüksek erişilebilirlik

### 2. Serverless mimarinin dezavantajları nelerdir?
- Cold start sorunu
- Uzun süreli işlemler için uygun değil
- Debug zorluğu
- Vendor lock-in riski
- Karmaşık uygulamalar için uygun değil

### 3. Serverless mimari ne zaman kullanılmalıdır?
- Event-driven uygulamalar
- API endpoints
- Background jobs
- Microservices
- Batch processing

## Kaynaklar
- [Azure Functions Documentation](https://docs.microsoft.com/tr-tr/azure/azure-functions/)
- [AWS Lambda Documentation](https://docs.aws.amazon.com/lambda/)
- [Serverless Best Practices](https://docs.microsoft.com/tr-tr/azure/architecture/serverless/best-practices)
- [Event-Driven Architecture](https://docs.microsoft.com/tr-tr/azure/architecture/guide/architecture-styles/event-driven) 