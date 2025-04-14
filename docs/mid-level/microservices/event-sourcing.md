# Event Sourcing

## Genel Bakış
Event Sourcing, bir uygulamanın durumunu, gerçekleşen olayların (events) sıralı bir kaydı olarak saklayan bir mimari desendir. Her durum değişikliği bir olay olarak kaydedilir ve uygulamanın mevcut durumu bu olayların yeniden oynatılmasıyla elde edilir.

## Temel Özellikler

### 1. Event Modeli
```csharp
// Base Event
public abstract class Event
{
    public Guid Id { get; set; }
    public DateTime Timestamp { get; set; }
    public string EventType { get; set; }
    public int Version { get; set; }
}

// Order Events
public class OrderCreatedEvent : Event
{
    public Guid OrderId { get; set; }
    public Guid CustomerId { get; set; }
    public List<OrderItem> Items { get; set; }
    public decimal TotalAmount { get; set; }
}

public class OrderItemAddedEvent : Event
{
    public Guid OrderId { get; set; }
    public OrderItem Item { get; set; }
}

public class OrderStatusChangedEvent : Event
{
    public Guid OrderId { get; set; }
    public OrderStatus NewStatus { get; set; }
}

// Event Store Interface
public interface IEventStore
{
    Task SaveEventsAsync(Guid aggregateId, IEnumerable<Event> events, int expectedVersion);
    Task<IEnumerable<Event>> GetEventsAsync(Guid aggregateId);
    Task<IEnumerable<Event>> GetEventsAsync(Guid aggregateId, int fromVersion);
}
```

### 2. Aggregate Root
```csharp
public class OrderAggregate
{
    private readonly List<Event> _changes = new();
    private OrderState _state = new();

    public Guid Id => _state.Id;
    public int Version => _state.Version;

    public void CreateOrder(CreateOrderCommand command)
    {
        var @event = new OrderCreatedEvent
        {
            Id = Guid.NewGuid(),
            Timestamp = DateTime.UtcNow,
            EventType = nameof(OrderCreatedEvent),
            Version = _state.Version + 1,
            OrderId = command.OrderId,
            CustomerId = command.CustomerId,
            Items = command.Items,
            TotalAmount = command.TotalAmount
        };

        Apply(@event);
        _changes.Add(@event);
    }

    public void AddItem(AddItemCommand command)
    {
        var @event = new OrderItemAddedEvent
        {
            Id = Guid.NewGuid(),
            Timestamp = DateTime.UtcNow,
            EventType = nameof(OrderItemAddedEvent),
            Version = _state.Version + 1,
            OrderId = command.OrderId,
            Item = command.Item
        };

        Apply(@event);
        _changes.Add(@event);
    }

    private void Apply(Event @event)
    {
        switch (@event)
        {
            case OrderCreatedEvent e:
                _state = new OrderState
                {
                    Id = e.OrderId,
                    CustomerId = e.CustomerId,
                    Items = e.Items,
                    TotalAmount = e.TotalAmount,
                    Status = OrderStatus.Created,
                    Version = e.Version
                };
                break;

            case OrderItemAddedEvent e:
                _state.Items.Add(e.Item);
                _state.TotalAmount += e.Item.Price * e.Item.Quantity;
                _state.Version = e.Version;
                break;

            case OrderStatusChangedEvent e:
                _state.Status = e.NewStatus;
                _state.Version = e.Version;
                break;
        }
    }

    public IEnumerable<Event> GetUncommittedChanges() => _changes;
    public void ClearUncommittedChanges() => _changes.Clear();
}
```

### 3. Event Store Implementation
```csharp
public class EventStore : IEventStore
{
    private readonly IEventRepository _repository;
    private readonly ILogger<EventStore> _logger;

    public EventStore(
        IEventRepository repository,
        ILogger<EventStore> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    public async Task SaveEventsAsync(
        Guid aggregateId,
        IEnumerable<Event> events,
        int expectedVersion)
    {
        var existingEvents = await _repository.GetEventsAsync(aggregateId);
        if (existingEvents.Any() && existingEvents.Last().Version != expectedVersion)
        {
            throw new ConcurrencyException();
        }

        foreach (var @event in events)
        {
            await _repository.SaveEventAsync(@event);
        }

        _logger.LogInformation(
            "Saved {Count} events for aggregate {AggregateId}",
            events.Count(),
            aggregateId);
    }

    public async Task<IEnumerable<Event>> GetEventsAsync(Guid aggregateId)
    {
        return await _repository.GetEventsAsync(aggregateId);
    }

    public async Task<IEnumerable<Event>> GetEventsAsync(
        Guid aggregateId,
        int fromVersion)
    {
        return await _repository.GetEventsAsync(aggregateId, fromVersion);
    }
}
```

### 4. Event Handlers
```csharp
public class OrderEventHandler
{
    private readonly IOrderRepository _orderRepository;
    private readonly IEventPublisher _eventPublisher;

    public OrderEventHandler(
        IOrderRepository orderRepository,
        IEventPublisher eventPublisher)
    {
        _orderRepository = orderRepository;
        _eventPublisher = eventPublisher;
    }

    public async Task HandleAsync(OrderCreatedEvent @event)
    {
        var order = new Order
        {
            Id = @event.OrderId,
            CustomerId = @event.CustomerId,
            Items = @event.Items,
            TotalAmount = @event.TotalAmount,
            Status = OrderStatus.Created
        };

        await _orderRepository.SaveAsync(order);
        await _eventPublisher.PublishAsync(@event);
    }

    public async Task HandleAsync(OrderItemAddedEvent @event)
    {
        var order = await _orderRepository.GetByIdAsync(@event.OrderId);
        order.Items.Add(@event.Item);
        order.TotalAmount += @event.Item.Price * @event.Item.Quantity;

        await _orderRepository.SaveAsync(order);
        await _eventPublisher.PublishAsync(@event);
    }
}
```

## Best Practices

### 1. Event Tasarımı
- Event'leri immutable yapın
- Event'leri atomik tutun
- Event'leri anlamlı isimlendirin
- Event'leri versiyonlayın

### 2. Aggregate Tasarımı
- Aggregate'leri küçük tutun
- Business kurallarını Aggregate'lerde uygulayın
- Event'leri doğru sırada uygulayın
- Concurrency kontrolü yapın

### 3. Event Store
- Event'leri sıralı saklayın
- Event'leri immutable tutun
- Event'leri versiyonlayın
- Event'leri optimize edin

### 4. Event Handlers
- Event handler'ları idempotent yapın
- Event handler'ları asenkron yapın
- Event handler'ları izole edin
- Error handling yapın

## Sık Sorulan Sorular

### 1. Event Sourcing neden önemlidir?
- Audit trail sağlar
- Temporal query'leri destekler
- Event replay sağlar
- CQRS ile uyumludur

### 2. Event Sourcing ne zaman kullanılmalıdır?
- Audit gerektiren sistemlerde
- Temporal query gerektiren sistemlerde
- Event-driven mimarilerde
- CQRS ile birlikte

### 3. Event Sourcing'de hangi zorluklar vardır?
- Event store yönetimi
- Event versioning
- Event migration
- Performance optimization

## Kaynaklar
- [Microsoft Event Sourcing Pattern](https://docs.microsoft.com/tr-tr/azure/architecture/patterns/event-sourcing)
- [Event Sourcing Best Practices](https://docs.microsoft.com/tr-tr/azure/architecture/patterns/event-sourcing#considerations)
- [CQRS Pattern](https://docs.microsoft.com/tr-tr/azure/architecture/patterns/cqrs)
- [Event Store Documentation](https://eventstore.com/docs/) 