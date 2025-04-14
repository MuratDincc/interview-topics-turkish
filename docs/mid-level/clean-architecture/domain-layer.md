# Domain Layer

## Genel Bakış
Domain Layer, Clean Architecture'ın en iç katmanıdır ve iş mantığının merkezini oluşturur. Bu katman, uygulamanın temel iş kurallarını ve varlıklarını içerir. Dış dünyadan tamamen bağımsızdır ve diğer katmanlara bağımlılığı yoktur.

## Temel Bileşenler

### 1. Entities
Entity'ler, iş mantığının temel yapı taşlarıdır. Her entity benzersiz bir kimliğe sahiptir ve iş kurallarını içerir.

**Örnek Kod:**
```csharp
public class Order : Entity
{
    public Guid Id { get; private set; }
    public string OrderNumber { get; private set; }
    public DateTime OrderDate { get; private set; }
    public OrderStatus Status { get; private set; }
    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();

    public Order(string orderNumber)
    {
        Id = Guid.NewGuid();
        OrderNumber = orderNumber;
        OrderDate = DateTime.UtcNow;
        Status = OrderStatus.Pending;
    }

    public void AddItem(Product product, int quantity)
    {
        if (quantity <= 0)
            throw new DomainException("Quantity must be greater than zero");

        var item = new OrderItem(product, quantity);
        _items.Add(item);
    }

    public void Complete()
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException("Order is not in pending status");

        Status = OrderStatus.Completed;
    }
}
```

### 2. Value Objects
Value Objects, kimliği olmayan ve değerleriyle tanımlanan nesnelerdir.

**Örnek Kod:**
```csharp
public class Address : ValueObject
{
    public string Street { get; private set; }
    public string City { get; private set; }
    public string Country { get; private set; }
    public string PostalCode { get; private set; }

    public Address(string street, string city, string country, string postalCode)
    {
        Street = street;
        City = city;
        Country = country;
        PostalCode = postalCode;
    }

    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return Street;
        yield return City;
        yield return Country;
        yield return PostalCode;
    }
}
```

### 3. Domain Events
Domain Events, domain içinde gerçekleşen önemli olayları temsil eder.

**Örnek Kod:**
```csharp
public class OrderCompletedEvent : IDomainEvent
{
    public Guid OrderId { get; }
    public DateTime OccurredOn { get; }

    public OrderCompletedEvent(Guid orderId)
    {
        OrderId = orderId;
        OccurredOn = DateTime.UtcNow;
    }
}
```

### 4. Domain Services
Domain Services, entity'lerin kapsamı dışındaki iş mantığını içerir.

**Örnek Kod:**
```csharp
public interface IOrderService
{
    bool CanPlaceOrder(Customer customer, Product product);
    decimal CalculateDiscount(Order order);
}

public class OrderService : IOrderService
{
    public bool CanPlaceOrder(Customer customer, Product product)
    {
        return customer.IsActive && product.IsAvailable;
    }

    public decimal CalculateDiscount(Order order)
    {
        if (order.Items.Count > 10)
            return order.Total * 0.1m;
        
        return 0;
    }
}
```

### 5. Domain Exceptions
Domain Exceptions, domain katmanında oluşabilecek özel hataları temsil eder.

**Örnek Kod:**
```csharp
public class DomainException : Exception
{
    public DomainException(string message) : base(message)
    {
    }

    public DomainException(string message, Exception innerException) 
        : base(message, innerException)
    {
    }
}
```

## Best Practices

### 1. Entity Tasarımı
- Entity'ler immutable olmalıdır
- Property'ler private set olmalıdır
- İş kuralları entity içinde uygulanmalıdır
- Validation logic entity içinde olmalıdır

### 2. Value Object Tasarımı
- Immutable olmalıdır
- Equality comparison override edilmelidir
- Validation logic içermelidir
- Business rules içerebilir

### 3. Domain Event Tasarımı
- Event'ler immutable olmalıdır
- Event'ler sadece gerekli verileri içermelidir
- Event'ler anlamlı isimlere sahip olmalıdır
- Event'ler geçmiş zaman kipinde olmalıdır

### 4. Domain Service Tasarımı
- Stateless olmalıdır
- Interface'ler üzerinden kullanılmalıdır
- Sadece domain logic içermelidir
- Infrastructure bağımlılığı olmamalıdır

## Sık Sorulan Sorular

### 1. Entity ve Value Object arasındaki fark nedir?
- Entity'lerin kimliği vardır, Value Object'lerin yoktur
- Entity'ler değişebilir, Value Object'ler immutable'dır
- Entity'ler referans eşitliği ile, Value Object'ler değer eşitliği ile karşılaştırılır

### 2. Domain Service ne zaman kullanılmalıdır?
- İş mantığı birden fazla entity'yi ilgilendirdiğinde
- İş mantığı entity'lerin dışında olduğunda
- Stateless operasyonlar gerektiğinde
- Complex business rules uygulandığında

### 3. Domain Event'ler nasıl kullanılmalıdır?
- Önemli domain değişikliklerini bildirmek için
- Sistem parçaları arasında loose coupling sağlamak için
- Event sourcing pattern uygulandığında
- CQRS pattern uygulandığında

## Kaynaklar
- [Domain-Driven Design - Eric Evans](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215)
- [Microsoft Domain Layer](https://docs.microsoft.com/tr-tr/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/ddd-oriented-microservice)
- [Domain Layer Best Practices](https://docs.microsoft.com/tr-tr/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/ddd-oriented-microservice#the-domain-layer) 