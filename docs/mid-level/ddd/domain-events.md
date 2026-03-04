# Domain Events

## Genel Bakış

Domain Events (Domain Olayları), domain içinde gerçekleşen ve iş açısından anlamlı olayları temsil eden nesnelerdir. "Sipariş Onaylandı", "Ödeme Alındı", "Stok Tükendi" gibi olaylar Domain Events'e örnek verilebilir. Domain Events, Aggregate'ler arası koordinasyonu loosely coupled biçimde sağlar ve side effect'lerin yönetimini kolaylaştırır. MediatR kütüphanesi, .NET projelerinde Domain Events'i implement etmek için en yaygın kullanılan araçtır.

## Domain Events Temel Kavramlar

### Domain Event vs Integration Event

| Özellik | Domain Event | Integration Event |
|---------|--------------|-------------------|
| Kapsam | Tek Bounded Context içi | Bounded Context'ler arası |
| Transport | In-process (bellekte) | Message broker (Kafka, RabbitMQ) |
| Zamanlama | Transaction içinde | Transaction sonrası |
| Hedef | Aynı servis içi handler'lar | Farklı servisler |
| Örnek | `OrderConfirmedEvent` | `OrderConfirmedIntegrationEvent` |

### Event Tasarım İlkeleri

- **Geçmiş zaman kullanılır**: `OrderPlaced`, `PaymentReceived`, `InventoryReserved`
- **Değişmez (immutable)**: Event oluştuktan sonra değiştirilemez
- **İş anlamlı**: Teknik detay değil, iş olayını yansıtır
- **Yeterli veri içerir**: Handler'ların ihtiyaç duyduğu tüm bilgiler event'te bulunur

## C# Implementasyonu

### Temel Yapı

```csharp
// Domain Event temel interface
public interface IDomainEvent : INotification
{
    Guid EventId { get; }
    DateTime OccurredAt { get; }
}

// Base Domain Event sınıfı
public abstract record DomainEvent : IDomainEvent
{
    public Guid EventId { get; } = Guid.NewGuid();
    public DateTime OccurredAt { get; } = DateTime.UtcNow;
}

// Concrete event'ler — record kullanmak immutability sağlar
public record OrderPlacedEvent(
    Guid OrderId,
    Guid CustomerId,
    Money TotalAmount,
    IReadOnlyList<OrderItemSnapshot> Items) : DomainEvent;

public record OrderConfirmedEvent(
    Guid OrderId,
    Guid CustomerId,
    Money TotalAmount,
    DateTime ConfirmedAt) : DomainEvent;

public record OrderShippedEvent(
    Guid OrderId,
    string TrackingNumber,
    DateTime ShippedAt) : DomainEvent;

public record OrderCancelledEvent(
    Guid OrderId,
    string Reason,
    DateTime CancelledAt) : DomainEvent;

public record PaymentReceivedEvent(
    Guid OrderId,
    Guid PaymentId,
    Money Amount,
    string PaymentMethod) : DomainEvent;

// Snapshot value object — event'in anlık görüntüsü
public record OrderItemSnapshot(
    Guid ProductId,
    string ProductName,
    int Quantity,
    Money UnitPrice);
```

### Aggregate Root'ta Event Raising

```csharp
public abstract class AggregateRoot : Entity
{
    private readonly List<IDomainEvent> _domainEvents = new();

    public IReadOnlyList<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected void RaiseDomainEvent(IDomainEvent domainEvent)
    {
        _domainEvents.Add(domainEvent);
    }

    public void ClearDomainEvents()
    {
        _domainEvents.Clear();
    }
}

// Order Aggregate Root — event'leri doğal iş akışında fırlatır
public class Order : AggregateRoot
{
    private readonly List<OrderItem> _items = new();

    public string OrderNumber { get; private set; }
    public Guid CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money TotalAmount { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public Address ShippingAddress { get; private set; }

    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();

    protected Order() { }

    public static Order Place(Guid customerId, Address shippingAddress, List<OrderItemRequest> items)
    {
        ArgumentNullException.ThrowIfNull(shippingAddress);

        if (items is null || !items.Any())
            throw new DomainException("Sipariş en az bir kalem içermelidir.");

        var order = new Order
        {
            Id = Guid.NewGuid(),
            OrderNumber = GenerateOrderNumber(),
            CustomerId = customerId,
            Status = OrderStatus.Pending,
            ShippingAddress = shippingAddress,
            TotalAmount = new Money(0, "TRY"),
            CreatedAt = DateTime.UtcNow
        };

        foreach (var item in items)
        {
            order._items.Add(OrderItem.Create(item.ProductId, item.ProductName,
                item.Quantity, item.UnitPrice));
        }

        order.TotalAmount = order.CalculateTotal();

        // Sipariş verildi olayı
        order.RaiseDomainEvent(new OrderPlacedEvent(
            OrderId: order.Id,
            CustomerId: customerId,
            TotalAmount: order.TotalAmount,
            Items: order.Items.Select(i => new OrderItemSnapshot(
                i.ProductId, i.ProductName, i.Quantity, i.UnitPrice)).ToList()));

        return order;
    }

    public void Confirm()
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException("Yalnızca bekleyen siparişler onaylanabilir.");

        Status = OrderStatus.Confirmed;

        // Sipariş onaylandı olayı
        RaiseDomainEvent(new OrderConfirmedEvent(
            OrderId: Id,
            CustomerId: CustomerId,
            TotalAmount: TotalAmount,
            ConfirmedAt: DateTime.UtcNow));
    }

    public void MarkAsShipped(string trackingNumber)
    {
        if (Status != OrderStatus.Confirmed)
            throw new DomainException("Yalnızca onaylanmış siparişler kargoya verilebilir.");

        if (string.IsNullOrWhiteSpace(trackingNumber))
            throw new ArgumentException("Takip numarası boş olamaz.", nameof(trackingNumber));

        Status = OrderStatus.Shipped;

        // Sipariş gönderildi olayı
        RaiseDomainEvent(new OrderShippedEvent(
            OrderId: Id,
            TrackingNumber: trackingNumber,
            ShippedAt: DateTime.UtcNow));
    }

    public void Cancel(string reason)
    {
        if (Status is OrderStatus.Shipped or OrderStatus.Delivered)
            throw new DomainException("Gönderilmiş sipariş iptal edilemez.");

        var previousStatus = Status;
        Status = OrderStatus.Cancelled;

        // Sipariş iptal edildi olayı
        RaiseDomainEvent(new OrderCancelledEvent(
            OrderId: Id,
            Reason: reason,
            CancelledAt: DateTime.UtcNow));
    }

    public void RecordPayment(Guid paymentId, Money amount, string paymentMethod)
    {
        if (Status != OrderStatus.Confirmed)
            throw new DomainException("Yalnızca onaylanmış sipariş için ödeme kaydedilebilir.");

        // Ödeme alındı olayı
        RaiseDomainEvent(new PaymentReceivedEvent(
            OrderId: Id,
            PaymentId: paymentId,
            Amount: amount,
            PaymentMethod: paymentMethod));
    }

    private Money CalculateTotal()
    {
        return _items.Aggregate(
            new Money(0, "TRY"),
            (acc, item) => acc.Add(item.UnitPrice.Multiply(item.Quantity)));
    }

    private static string GenerateOrderNumber()
    {
        return $"ORD-{DateTime.UtcNow:yyyyMMdd}-{Guid.NewGuid().ToString("N")[..8].ToUpper()}";
    }
}
```

### MediatR ile Domain Event Dispatch

```csharp
// Unit of Work: transaction tamamlandıktan sonra event'leri dispatch eder
public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;
    private readonly IPublisher _publisher;  // MediatR IPublisher

    public UnitOfWork(AppDbContext context, IPublisher publisher)
    {
        _context = context;
        _publisher = publisher;
    }

    public async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        // 1. Veritabanına kaydet
        var result = await _context.SaveChangesAsync(cancellationToken);

        // 2. Aggregate'lerdeki event'leri topla
        var aggregates = _context.ChangeTracker
            .Entries<AggregateRoot>()
            .Where(e => e.Entity.DomainEvents.Any())
            .Select(e => e.Entity)
            .ToList();

        var domainEvents = aggregates
            .SelectMany(a => a.DomainEvents)
            .ToList();

        // 3. Event listelerini temizle (tekrar dispatch engellemek için)
        aggregates.ForEach(a => a.ClearDomainEvents());

        // 4. Event'leri MediatR aracılığıyla dispatch et
        foreach (var domainEvent in domainEvents)
        {
            await _publisher.Publish(domainEvent, cancellationToken);
        }

        return result;
    }
}
```

### MediatR Event Handler'ları

```csharp
// =============================================
// SIPARIŞ ONAYLANDI — Birden fazla handler
// =============================================

// Handler 1: Stok rezervasyonu
public class ReserveInventoryOnOrderConfirmedHandler
    : INotificationHandler<OrderConfirmedEvent>
{
    private readonly IInventoryService _inventoryService;
    private readonly ILogger<ReserveInventoryOnOrderConfirmedHandler> _logger;

    public ReserveInventoryOnOrderConfirmedHandler(
        IInventoryService inventoryService,
        ILogger<ReserveInventoryOnOrderConfirmedHandler> logger)
    {
        _inventoryService = inventoryService;
        _logger = logger;
    }

    public async Task Handle(OrderConfirmedEvent notification, CancellationToken cancellationToken)
    {
        _logger.LogInformation(
            "Sipariş {OrderId} için stok rezervasyonu başlatılıyor.",
            notification.OrderId);

        // Stok ayrıca rezerve edilir
        await _inventoryService.ReserveForOrderAsync(
            notification.OrderId, cancellationToken);

        _logger.LogInformation(
            "Sipariş {OrderId} için stok rezervasyonu tamamlandı.",
            notification.OrderId);
    }
}

// Handler 2: Müşteriye bildirim e-postası
public class SendOrderConfirmationEmailHandler
    : INotificationHandler<OrderConfirmedEvent>
{
    private readonly IEmailService _emailService;
    private readonly ICustomerRepository _customerRepository;
    private readonly ILogger<SendOrderConfirmationEmailHandler> _logger;

    public SendOrderConfirmationEmailHandler(
        IEmailService emailService,
        ICustomerRepository customerRepository,
        ILogger<SendOrderConfirmationEmailHandler> logger)
    {
        _emailService = emailService;
        _customerRepository = customerRepository;
        _logger = logger;
    }

    public async Task Handle(OrderConfirmedEvent notification, CancellationToken cancellationToken)
    {
        var customer = await _customerRepository.GetByIdAsync(
            notification.CustomerId, cancellationToken);

        if (customer is null)
        {
            _logger.LogWarning("Müşteri bulunamadı: {CustomerId}", notification.CustomerId);
            return;
        }

        await _emailService.SendOrderConfirmationAsync(
            to: customer.Email,
            orderId: notification.OrderId,
            totalAmount: notification.TotalAmount,
            confirmedAt: notification.ConfirmedAt,
            cancellationToken: cancellationToken);

        _logger.LogInformation(
            "Sipariş onay e-postası gönderildi: {Email}, Sipariş: {OrderId}",
            customer.Email, notification.OrderId);
    }
}

// Handler 3: Sadakat puanı güncelleme
public class UpdateLoyaltyPointsOnOrderConfirmedHandler
    : INotificationHandler<OrderConfirmedEvent>
{
    private readonly ICustomerRepository _customerRepository;
    private readonly IUnitOfWork _unitOfWork;

    public UpdateLoyaltyPointsOnOrderConfirmedHandler(
        ICustomerRepository customerRepository,
        IUnitOfWork unitOfWork)
    {
        _customerRepository = customerRepository;
        _unitOfWork = unitOfWork;
    }

    public async Task Handle(OrderConfirmedEvent notification, CancellationToken cancellationToken)
    {
        var customer = await _customerRepository.GetByIdAsync(
            notification.CustomerId, cancellationToken);

        if (customer is null) return;

        // Her 10 TRY için 1 puan
        var points = (int)(notification.TotalAmount.Amount / 10);
        customer.AddLoyaltyPoints(points);

        _customerRepository.Update(customer);
        await _unitOfWork.SaveChangesAsync(cancellationToken);
    }
}

// Handler 4: Sipariş iptalinde stok serbest bırakma
public class ReleaseInventoryOnOrderCancelledHandler
    : INotificationHandler<OrderCancelledEvent>
{
    private readonly IInventoryService _inventoryService;
    private readonly ILogger<ReleaseInventoryOnOrderCancelledHandler> _logger;

    public ReleaseInventoryOnOrderCancelledHandler(
        IInventoryService inventoryService,
        ILogger<ReleaseInventoryOnOrderCancelledHandler> logger)
    {
        _inventoryService = inventoryService;
        _logger = logger;
    }

    public async Task Handle(OrderCancelledEvent notification, CancellationToken cancellationToken)
    {
        _logger.LogInformation(
            "Sipariş {OrderId} iptal edildi. Stok serbest bırakılıyor. Neden: {Reason}",
            notification.OrderId, notification.Reason);

        await _inventoryService.ReleaseReservationAsync(
            notification.OrderId, cancellationToken);
    }
}
```

### Outbox Pattern ile Güvenilir Event Dispatch

Outbox Pattern, domain event'lerin kaybolmamasını (at-least-once delivery) garantiler. Event, veritabanı transaction'ı ile birlikte kaydedilir; arka planda bir işleyici event'leri yayınlar.

```csharp
// Outbox mesajı — veritabanında saklanır
public class OutboxMessage
{
    public Guid Id { get; init; } = Guid.NewGuid();
    public string Type { get; init; }
    public string Payload { get; init; }
    public DateTime CreatedAt { get; init; } = DateTime.UtcNow;
    public DateTime? ProcessedAt { get; set; }
    public string? Error { get; set; }
    public int RetryCount { get; set; }
}

// Unit of Work: Event'leri outbox tablosuna yazar
public class OutboxUnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;

    public OutboxUnitOfWork(AppDbContext context)
    {
        _context = context;
    }

    public async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        // Aggregate'lerden event'leri topla
        var aggregates = _context.ChangeTracker
            .Entries<AggregateRoot>()
            .Where(e => e.Entity.DomainEvents.Any())
            .Select(e => e.Entity)
            .ToList();

        // Event'leri Outbox tablosuna dönüştür
        var outboxMessages = aggregates
            .SelectMany(a => a.DomainEvents)
            .Select(e => new OutboxMessage
            {
                Type = e.GetType().AssemblyQualifiedName!,
                Payload = JsonSerializer.Serialize(e, e.GetType())
            })
            .ToList();

        aggregates.ForEach(a => a.ClearDomainEvents());

        // Aggregate değişiklikleri ve outbox mesajları aynı transaction içinde kaydedilir
        await _context.OutboxMessages.AddRangeAsync(outboxMessages, cancellationToken);
        return await _context.SaveChangesAsync(cancellationToken);
    }
}

// Background Service: Outbox mesajlarını periyodik işler
public class OutboxProcessorService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<OutboxProcessorService> _logger;
    private readonly TimeSpan _interval = TimeSpan.FromSeconds(5);

    public OutboxProcessorService(
        IServiceScopeFactory scopeFactory,
        ILogger<OutboxProcessorService> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await ProcessPendingMessagesAsync(stoppingToken);
            await Task.Delay(_interval, stoppingToken);
        }
    }

    private async Task ProcessPendingMessagesAsync(CancellationToken cancellationToken)
    {
        using var scope = _scopeFactory.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        var publisher = scope.ServiceProvider.GetRequiredService<IPublisher>();

        var pendingMessages = await context.OutboxMessages
            .Where(m => m.ProcessedAt == null && m.RetryCount < 3)
            .OrderBy(m => m.CreatedAt)
            .Take(20)
            .ToListAsync(cancellationToken);

        foreach (var message in pendingMessages)
        {
            try
            {
                var eventType = Type.GetType(message.Type)!;
                var domainEvent = (IDomainEvent)JsonSerializer.Deserialize(message.Payload, eventType)!;

                await publisher.Publish(domainEvent, cancellationToken);

                message.ProcessedAt = DateTime.UtcNow;
                _logger.LogInformation("Outbox mesajı işlendi: {MessageId}", message.Id);
            }
            catch (Exception ex)
            {
                message.RetryCount++;
                message.Error = ex.Message;
                _logger.LogError(ex, "Outbox mesajı işlenemedi: {MessageId}", message.Id);
            }
        }

        await context.SaveChangesAsync(cancellationToken);
    }
}
```

### DI Kayıt ve Yapılandırma

```csharp
// Program.cs veya DI extension method
public static class DomainEventsServiceCollectionExtensions
{
    public static IServiceCollection AddDomainEventsInfrastructure(
        this IServiceCollection services,
        Assembly[] assemblies)
    {
        // MediatR kayıt — handler'ları otomatik tarar
        services.AddMediatR(cfg =>
        {
            cfg.RegisterServicesFromAssemblies(assemblies);

            // Pipeline behavior: event logging
            cfg.AddOpenBehavior(typeof(DomainEventLoggingBehavior<,>));
        });

        // Unit of Work
        services.AddScoped<IUnitOfWork, OutboxUnitOfWork>();

        // Outbox işleyici
        services.AddHostedService<OutboxProcessorService>();

        return services;
    }
}

// Pipeline Behavior: Event dispatch'i loglama
public class DomainEventLoggingBehavior<TEvent, TResponse>
    : IPipelineBehavior<TEvent, TResponse>
    where TEvent : IDomainEvent
{
    private readonly ILogger<DomainEventLoggingBehavior<TEvent, TResponse>> _logger;

    public DomainEventLoggingBehavior(
        ILogger<DomainEventLoggingBehavior<TEvent, TResponse>> logger)
    {
        _logger = logger;
    }

    public async Task<TResponse> Handle(
        TEvent request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var eventName = typeof(TEvent).Name;

        _logger.LogInformation(
            "Domain event işleniyor: {EventName} (EventId: {EventId}, OccurredAt: {OccurredAt})",
            eventName, request.EventId, request.OccurredAt);

        var response = await next();

        _logger.LogInformation(
            "Domain event tamamlandı: {EventName} (EventId: {EventId})",
            eventName, request.EventId);

        return response;
    }
}
```

## Mülakat Soruları

### 1. Soru: Domain Event nedir ve neden kullanılır?

**Cevap:**

Domain Event, domain içinde gerçekleşen ve iş açısından anlamlı bir olayı temsil eden değişmez nesnedir. Kullanım nedenleri:

- **Loosely Coupling**: Aggregate'ler birbirini doğrudan çağırmak yerine event'ler aracılığıyla iletişim kurar
- **Single Responsibility**: Her handler tek bir sorumluluğa odaklanır
- **Açık-Kapalı Prensibi**: Yeni side effect eklemek için mevcut kodu değiştirmek gerekmez, yeni handler yazılır
- **Audit Trail**: Domain olaylarının loglanması kolaylaşır
- **Eventual Consistency**: Aggregate'ler arası tutarlılık asenkron sağlanabilir

**Örnek Kod:**

```csharp
// Event olmadan — sıkı bağımlılık
public class OrderService
{
    private readonly IInventoryService _inventory;
    private readonly IEmailService _email;
    private readonly ILoyaltyService _loyalty;

    public async Task ConfirmOrderAsync(Guid orderId)
    {
        var order = await GetOrderAsync(orderId);
        order.Confirm();
        // OrderService her side effect'ten haberdar olmak zorunda
        await _inventory.ReserveAsync(orderId);   // Sıkı bağımlılık
        await _email.SendConfirmationAsync(orderId); // Sıkı bağımlılık
        await _loyalty.AddPointsAsync(orderId);      // Sıkı bağımlılık
    }
}

// Event ile — loosely coupled
public class OrderService
{
    public async Task ConfirmOrderAsync(Guid orderId)
    {
        var order = await GetOrderAsync(orderId);
        order.Confirm(); // OrderConfirmedEvent raise edilir
        await _unitOfWork.SaveChangesAsync(); // Event'ler dispatch edilir
        // OrderService hiçbir side effect'ten haberdar değil — her handler bağımsız
    }
}
```

---

### 2. Soru: Domain Event ile Integration Event arasındaki fark nedir?

**Cevap:**

| Özellik | Domain Event | Integration Event |
|---------|--------------|-------------------|
| Kapsam | Tek Bounded Context | Bounded Context'ler arası |
| Transport | In-process (MediatR) | Message broker (Kafka, RabbitMQ) |
| Zamanlama | Transaction içinde veya hemen sonrası | Transaction commit'ten sonra |
| Güvenilirlik | Best-effort veya Outbox | At-least-once delivery |
| Model | Domain nesneleri | Serileştirilebilir DTO |

**Örnek Kod:**

```csharp
// Domain Event: Aynı process içinde
public record OrderConfirmedEvent(
    Guid OrderId, Guid CustomerId, Money TotalAmount) : DomainEvent;

// Integration Event: Farklı servisler arası — serileştirilebilir primitive tipler
public record OrderConfirmedIntegrationEvent(
    Guid OrderId,
    Guid CustomerId,
    decimal TotalAmount,
    string Currency,
    DateTime ConfirmedAt);

// Domain Event → Integration Event dönüşümü
public class PublishIntegrationEventOnOrderConfirmedHandler
    : INotificationHandler<OrderConfirmedEvent>
{
    private readonly IIntegrationEventBus _eventBus;

    public PublishIntegrationEventOnOrderConfirmedHandler(IIntegrationEventBus eventBus)
    {
        _eventBus = eventBus;
    }

    public async Task Handle(OrderConfirmedEvent notification, CancellationToken ct)
    {
        // Domain modeli → Integration event DTO dönüşümü
        var integrationEvent = new OrderConfirmedIntegrationEvent(
            OrderId: notification.OrderId,
            CustomerId: notification.CustomerId,
            TotalAmount: notification.TotalAmount.Amount,
            Currency: notification.TotalAmount.Currency,
            ConfirmedAt: notification.OccurredAt);

        await _eventBus.PublishAsync(integrationEvent, ct);
    }
}
```

---

### 3. Soru: Domain Event'ler ne zaman dispatch edilmeli: transaction öncesi mi sonrası mı?

**Cevap:**

**Transaction öncesi dispatch:**
- Event handler hataları transaction'ı geri alabilir
- Ancak event handler'daki dış sistem çağrıları (e-posta, SMS) geri alınamaz

**Transaction sonrası dispatch (önerilen):**
- Veri tutarlılığı garantilenir, önce kayıt sonra event
- Ancak uygulama çökmesi durumunda event'ler kaybolabilir

**Outbox Pattern ile transaction sonrası + güvenilir dispatch:**
- Event, domain aggregate değişikliğiyle aynı transaction'da outbox tablosuna yazılır
- Background service outbox'ı işleyerek event'leri publish eder
- At-least-once delivery garantisi sağlanır

**Örnek Kod:**

```csharp
// Transaction sonrası dispatch — SaveChanges sonrası
public class UnitOfWork : IUnitOfWork
{
    public async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        // 1. Önce veritabanına kaydet
        var result = await _context.SaveChangesAsync(ct);

        // 2. Kayıt başarılıysa event'leri dispatch et
        var events = CollectDomainEvents();
        foreach (var domainEvent in events)
        {
            await _publisher.Publish(domainEvent, ct);
        }

        return result;
    }
}

// Outbox Pattern: En güvenilir yöntem
public class OutboxUnitOfWork : IUnitOfWork
{
    public async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        // Domain değişiklikleri + outbox mesajları aynı transaction'da kaydedilir
        var events = CollectDomainEvents();
        var outboxMessages = events.Select(ToOutboxMessage).ToList();
        await _context.OutboxMessages.AddRangeAsync(outboxMessages, ct);

        // Tek atomik işlem
        return await _context.SaveChangesAsync(ct);
        // Background service outbox'ı periyodik işler
    }
}
```

---

### 4. Soru: MediatR ile Domain Events nasıl implement edilir?

**Cevap:**

MediatR'de `INotification` interface'i ve `INotificationHandler<T>` kullanılır. Domain events `INotification`'dan türer, handler'lar `INotificationHandler<TEvent>` implemente eder. `IPublisher.Publish()` tüm handler'ları paralel çalıştırır.

**Örnek Kod:**

```csharp
// NuGet: MediatR

// 1. Event tanımı
public record ProductStockDepletedEvent(
    Guid ProductId,
    string ProductName,
    int CurrentStock) : DomainEvent;

// 2. Handler'lar — birden fazla olabilir
public class NotifyPurchasingOnStockDepletedHandler
    : INotificationHandler<ProductStockDepletedEvent>
{
    private readonly IPurchasingService _purchasingService;

    public NotifyPurchasingOnStockDepletedHandler(IPurchasingService purchasingService)
    {
        _purchasingService = purchasingService;
    }

    public async Task Handle(
        ProductStockDepletedEvent notification,
        CancellationToken cancellationToken)
    {
        await _purchasingService.CreateReplenishmentOrderAsync(
            notification.ProductId,
            quantity: 100,
            cancellationToken);
    }
}

public class LogStockDepletionHandler
    : INotificationHandler<ProductStockDepletedEvent>
{
    private readonly ILogger<LogStockDepletionHandler> _logger;

    public LogStockDepletionHandler(ILogger<LogStockDepletionHandler> logger)
    {
        _logger = logger;
    }

    public Task Handle(
        ProductStockDepletedEvent notification,
        CancellationToken cancellationToken)
    {
        _logger.LogWarning(
            "Stok tükendi: {ProductName} (ID: {ProductId}), Mevcut: {CurrentStock}",
            notification.ProductName, notification.ProductId, notification.CurrentStock);

        return Task.CompletedTask;
    }
}

// 3. DI Kayıt
services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssembly(typeof(Program).Assembly));

// 4. Dispatch — UnitOfWork.SaveChangesAsync içinde otomatik
await _publisher.Publish(new ProductStockDepletedEvent(productId, "Laptop", 0));
```

---

### 5. Soru: Domain Events ile eventual consistency nasıl sağlanır?

**Cevap:**

Eventual consistency, birden fazla Aggregate'in farklı zamanlarda tutarlı hale gelmesini ifade eder. Domain Events aracılığıyla sağlanır:

1. Aggregate 1 değişir ve event raise eder
2. Event dispatch edilir
3. Handler, Aggregate 2'yi günceller
4. Sistem sonunda tutarlı hale gelir

**Dikkat edilecekler:**
- Event handler'lar idempotent olmalı (aynı event birden fazla işlenirse sonuç değişmemeli)
- Hata durumunda retry mekanizması bulunmalı
- Tutarsız ara durum kabul edilmeli

**Örnek Kod:**

```csharp
// Order Confirmed → Inventory Reserved akışı (eventual consistency)
public class Order : AggregateRoot
{
    public ReservationStatus InventoryStatus { get; private set; } = ReservationStatus.Pending;

    public void MarkInventoryReserved()
    {
        if (Status != OrderStatus.Confirmed)
            throw new DomainException("Yalnızca onaylanmış sipariş için stok rezerve edilebilir.");

        InventoryStatus = ReservationStatus.Reserved;
        RaiseDomainEvent(new InventoryReservedEvent(Id));
    }

    public void MarkInventoryFailed(string reason)
    {
        // Stok rezervasyonu başarısız — siparişi iptal et
        InventoryStatus = ReservationStatus.Failed;
        Cancel($"Stok rezervasyonu başarısız: {reason}");
    }
}

// Handler: Idempotent — aynı event birden fazla gelse bile güvenli
public class ReserveInventoryHandler : INotificationHandler<OrderConfirmedEvent>
{
    private readonly IInventoryRepository _inventoryRepository;
    private readonly IOrderRepository _orderRepository;
    private readonly IUnitOfWork _unitOfWork;

    public async Task Handle(OrderConfirmedEvent notification, CancellationToken ct)
    {
        var order = await _orderRepository.GetByIdAsync(notification.OrderId, ct);
        if (order is null) return;

        // Idempotency: Zaten rezerve edildiyse tekrar yapma
        if (order.InventoryStatus == ReservationStatus.Reserved) return;

        try
        {
            // Stok rezervasyonu
            foreach (var item in order.Items)
            {
                await _inventoryRepository.ReserveAsync(item.ProductId, item.Quantity, ct);
            }

            order.MarkInventoryReserved();
        }
        catch (InsufficientStockException ex)
        {
            order.MarkInventoryFailed(ex.Message);
        }

        _orderRepository.Update(order);
        await _unitOfWork.SaveChangesAsync(ct);
    }
}

public enum ReservationStatus { Pending, Reserved, Failed }
```

---

### 6. Soru: Domain Event handler'ları test nasıl yazılır?

**Cevap:**

Domain Event handler'ları bağımsız sınıflar olduğundan unit test yazmak kolaydır. Mock kullanarak bağımlılıklar izole edilir.

**Örnek Kod:**

```csharp
// xUnit + NSubstitute ile handler testi
public class SendOrderConfirmationEmailHandlerTests
{
    private readonly IEmailService _emailService;
    private readonly ICustomerRepository _customerRepository;
    private readonly SendOrderConfirmationEmailHandler _handler;

    public SendOrderConfirmationEmailHandlerTests()
    {
        _emailService = Substitute.For<IEmailService>();
        _customerRepository = Substitute.For<ICustomerRepository>();
        _handler = new SendOrderConfirmationEmailHandler(
            _emailService,
            _customerRepository,
            NullLogger<SendOrderConfirmationEmailHandler>.Instance);
    }

    [Fact]
    public async Task Handle_ValidEvent_SendsConfirmationEmail()
    {
        // Arrange
        var customerId = Guid.NewGuid();
        var orderId = Guid.NewGuid();
        var customer = Customer.Reconstruct(customerId, "test@example.com", "Test Kullanıcı");
        var totalAmount = new Money(250m, "TRY");

        _customerRepository.GetByIdAsync(customerId, Arg.Any<CancellationToken>())
            .Returns(customer);

        var @event = new OrderConfirmedEvent(
            OrderId: orderId,
            CustomerId: customerId,
            TotalAmount: totalAmount,
            ConfirmedAt: DateTime.UtcNow);

        // Act
        await _handler.Handle(@event, CancellationToken.None);

        // Assert
        await _emailService.Received(1).SendOrderConfirmationAsync(
            to: "test@example.com",
            orderId: orderId,
            totalAmount: totalAmount,
            confirmedAt: Arg.Any<DateTime>(),
            cancellationToken: Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task Handle_CustomerNotFound_DoesNotSendEmail()
    {
        // Arrange
        var customerId = Guid.NewGuid();
        _customerRepository.GetByIdAsync(customerId, Arg.Any<CancellationToken>())
            .Returns((Customer?)null);

        var @event = new OrderConfirmedEvent(
            OrderId: Guid.NewGuid(),
            CustomerId: customerId,
            TotalAmount: new Money(100m, "TRY"),
            ConfirmedAt: DateTime.UtcNow);

        // Act — exception fırlatmamalı
        await _handler.Handle(@event, CancellationToken.None);

        // Assert — e-posta gönderilmemeli
        await _emailService.DidNotReceiveWithAnyArgs()
            .SendOrderConfirmationAsync(default!, default, default!, default, default);
    }
}

// Aggregate Root event testi
public class OrderDomainEventTests
{
    [Fact]
    public void Confirm_PendingOrder_RaisesOrderConfirmedEvent()
    {
        // Arrange
        var order = Order.Place(
            customerId: Guid.NewGuid(),
            shippingAddress: new Address("Atatürk Cad.", "İstanbul", "34000", "TR"),
            items: new List<OrderItemRequest>
            {
                new(Guid.NewGuid(), "Laptop", 1, new Money(15000m, "TRY"))
            });

        order.ClearDomainEvents(); // Place event'lerini temizle

        // Act
        order.Confirm();

        // Assert
        var events = order.DomainEvents;
        Assert.Single(events);
        var confirmedEvent = Assert.IsType<OrderConfirmedEvent>(events[0]);
        Assert.Equal(order.Id, confirmedEvent.OrderId);
        Assert.Equal(new Money(15000m, "TRY"), confirmedEvent.TotalAmount);
    }

    [Fact]
    public void Cancel_ShippedOrder_ThrowsDomainException()
    {
        // Arrange
        var order = CreateShippedOrder();

        // Act & Assert
        Assert.Throws<DomainException>(() => order.Cancel("Test"));
        Assert.Empty(order.DomainEvents.OfType<OrderCancelledEvent>());
    }

    private static Order CreateShippedOrder()
    {
        var order = Order.Place(
            Guid.NewGuid(),
            new Address("Test Sok.", "Ankara", "06000", "TR"),
            new List<OrderItemRequest> { new(Guid.NewGuid(), "Klavye", 2, new Money(500m, "TRY")) });
        order.Confirm();
        order.MarkAsShipped("TRK-123456");
        order.ClearDomainEvents();
        return order;
    }
}
```
