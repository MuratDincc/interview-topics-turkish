# Service Communication (Servis İletişimi)

## Genel Bakış
Mikroservisler arası iletişim, dağıtık sistemlerin temel taşıdır. Servisler arasında veri alışverişi ve senkronizasyon için farklı iletişim modelleri kullanılır.

## İletişim Modelleri

### 1. Senkron İletişim
Doğrudan ve anlık yanıt gerektiren iletişim türüdür.

#### REST API
```csharp
// Order Service
public class OrderController : ControllerBase
{
    private readonly HttpClient _httpClient;

    public OrderController(IHttpClientFactory httpClientFactory)
    {
        _httpClient = httpClientFactory.CreateClient("PaymentService");
    }

    [HttpPost]
    public async Task<IActionResult> CreateOrder([FromBody] CreateOrderRequest request)
    {
        // Payment Service'e istek gönder
        var paymentResponse = await _httpClient.PostAsJsonAsync(
            "/api/payments/process",
            new ProcessPaymentRequest
            {
                OrderId = request.OrderId,
                Amount = request.TotalAmount
            });

        if (!paymentResponse.IsSuccessStatusCode)
        {
            return BadRequest("Payment failed");
        }

        // Siparişi oluştur
        return Ok();
    }
}

// Payment Service
public class PaymentController : ControllerBase
{
    [HttpPost("process")]
    public async Task<IActionResult> ProcessPayment([FromBody] ProcessPaymentRequest request)
    {
        // Ödeme işlemini gerçekleştir
        return Ok();
    }
}
```

#### gRPC
```protobuf
// payment.proto
syntax = "proto3";

service PaymentService {
  rpc ProcessPayment (PaymentRequest) returns (PaymentResponse);
}

message PaymentRequest {
  string order_id = 1;
  double amount = 2;
}

message PaymentResponse {
  bool success = 1;
  string message = 2;
}
```

```csharp
// Order Service
public class OrderService
{
    private readonly PaymentService.PaymentServiceClient _paymentClient;

    public OrderService(PaymentService.PaymentServiceClient paymentClient)
    {
        _paymentClient = paymentClient;
    }

    public async Task CreateOrderAsync(CreateOrderRequest request)
    {
        var paymentRequest = new PaymentRequest
        {
            OrderId = request.OrderId,
            Amount = request.TotalAmount
        };

        var paymentResponse = await _paymentClient.ProcessPaymentAsync(paymentRequest);
        
        if (!paymentResponse.Success)
        {
            throw new PaymentFailedException(paymentResponse.Message);
        }
    }
}
```

### 2. Asenkron İletişim
Zaman bağımsız, mesaj tabanlı iletişim türüdür.

#### Message Queue (RabbitMQ)
```csharp
// Order Service
public class OrderService
{
    private readonly IMessagePublisher _publisher;

    public OrderService(IMessagePublisher publisher)
    {
        _publisher = publisher;
    }

    public async Task CreateOrderAsync(CreateOrderRequest request)
    {
        // Siparişi oluştur
        var order = new Order
        {
            Id = request.OrderId,
            TotalAmount = request.TotalAmount
        };

        // Sipariş oluşturuldu mesajını yayınla
        await _publisher.PublishAsync(new OrderCreatedEvent
        {
            OrderId = order.Id,
            Amount = order.TotalAmount,
            Timestamp = DateTime.UtcNow
        });
    }
}

// Payment Service
public class PaymentConsumer : IConsumer<OrderCreatedEvent>
{
    private readonly IPaymentProcessor _paymentProcessor;

    public PaymentConsumer(IPaymentProcessor paymentProcessor)
    {
        _paymentProcessor = paymentProcessor;
    }

    public async Task Consume(ConsumeContext<OrderCreatedEvent> context)
    {
        var message = context.Message;
        
        // Ödeme işlemini gerçekleştir
        await _paymentProcessor.ProcessPaymentAsync(
            message.OrderId,
            message.Amount
        );
    }
}
```

#### Event Sourcing
```csharp
// Order Service
public class OrderAggregate
{
    private readonly List<IDomainEvent> _events = new();
    private OrderState _state = new();

    public void CreateOrder(CreateOrderCommand command)
    {
        var @event = new OrderCreatedEvent
        {
            OrderId = command.OrderId,
            CustomerId = command.CustomerId,
            Items = command.Items,
            Timestamp = DateTime.UtcNow
        };

        Apply(@event);
        _events.Add(@event);
    }

    private void Apply(OrderCreatedEvent @event)
    {
        _state = new OrderState
        {
            OrderId = @event.OrderId,
            CustomerId = @event.CustomerId,
            Items = @event.Items,
            Status = OrderStatus.Created
        };
    }
}

// Payment Service
public class PaymentEventHandler
{
    private readonly IPaymentProcessor _paymentProcessor;

    public PaymentEventHandler(IPaymentProcessor paymentProcessor)
    {
        _paymentProcessor = paymentProcessor;
    }

    public async Task Handle(OrderCreatedEvent @event)
    {
        await _paymentProcessor.ProcessPaymentAsync(
            @event.OrderId,
            @event.TotalAmount
        );
    }
}
```

## İletişim Desenleri

### 1. Request/Response
- Senkron iletişim için kullanılır
- Anlık yanıt gerektiren durumlar
- REST ve gRPC ile uygulanır

### 2. Publish/Subscribe
- Asenkron iletişim için kullanılır
- Birden fazla alıcı olabilir
- Message Queue ile uygulanır

### 3. Event-Driven
- Olay tabanlı iletişim
- Loose coupling sağlar
- Event sourcing ile uygulanır

## Best Practices

### 1. API Tasarımı
- RESTful prensipleri takip edin
- API versiyonlama kullanın
- Swagger/OpenAPI dokümantasyonu ekleyin
- Rate limiting uygulayın

### 2. Mesajlaşma
- Mesaj formatını standardize edin
- Dead letter queue kullanın
- Retry mekanizması ekleyin
- Mesaj boyutunu optimize edin

### 3. Hata Yönetimi
- Circuit breaker pattern kullanın
- Timeout değerlerini ayarlayın
- Fallback mekanizmaları ekleyin
- Hata loglama yapın

### 4. Güvenlik
- SSL/TLS kullanın
- API authentication uygulayın
- Input validation yapın
- Rate limiting uygulayın

## Sık Sorulan Sorular

### 1. Senkron ve asenkron iletişim arasındaki fark nedir?
- Senkron: Anlık yanıt gerektiren durumlar
- Asenkron: Zaman bağımsız, mesaj tabanlı iletişim
- Senkron: Doğrudan servis çağrısı
- Asenkron: Mesaj kuyruğu üzerinden iletişim

### 2. Hangi iletişim modeli ne zaman kullanılmalıdır?
- Senkron: Anlık yanıt gerektiren işlemler
- Asenkron: Zaman bağımsız işlemler
- Event-driven: Loose coupling gerektiren durumlar
- Message queue: Yüksek hacimli işlemler

### 3. Servisler arası iletişimde güvenlik nasıl sağlanmalıdır?
- SSL/TLS kullanılmalı
- API authentication uygulanmalı
- Input validation yapılmalı
- Rate limiting uygulanmalı

## Kaynaklar
- [Microsoft Service Communication](https://docs.microsoft.com/tr-tr/dotnet/architecture/microservices/architect-microservice-container-applications/communication-in-microservice-architecture)
- [gRPC Documentation](https://grpc.io/docs/)
- [RabbitMQ Documentation](https://www.rabbitmq.com/documentation.html)
- [Event Sourcing Pattern](https://docs.microsoft.com/tr-tr/azure/architecture/patterns/event-sourcing) 