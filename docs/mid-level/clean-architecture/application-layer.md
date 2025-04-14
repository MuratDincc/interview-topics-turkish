# Application Layer

## Genel Bakış
Application Layer, Clean Architecture'ın domain layer ile dış katmanlar arasında köprü görevi gören katmanıdır. Bu katman, use case'lerin implementasyonunu içerir ve domain layer'ı dış dünyadan izole eder.

## Temel Bileşenler

### 1. Use Cases
Use case'ler, uygulamanın temel işlevlerini tanımlar ve implemente eder.

**Örnek Kod:**
```csharp
public interface ICreateOrderUseCase
{
    Task<OrderDto> Execute(CreateOrderRequest request);
}

public class CreateOrderUseCase : ICreateOrderUseCase
{
    private readonly IOrderRepository _orderRepository;
    private readonly IProductRepository _productRepository;
    private readonly IUnitOfWork _unitOfWork;

    public CreateOrderUseCase(
        IOrderRepository orderRepository,
        IProductRepository productRepository,
        IUnitOfWork unitOfWork)
    {
        _orderRepository = orderRepository;
        _productRepository = productRepository;
        _unitOfWork = unitOfWork;
    }

    public async Task<OrderDto> Execute(CreateOrderRequest request)
    {
        var order = new Order(request.OrderNumber);
        
        foreach (var item in request.Items)
        {
            var product = await _productRepository.GetById(item.ProductId);
            order.AddItem(product, item.Quantity);
        }

        await _orderRepository.Add(order);
        await _unitOfWork.SaveChanges();

        return OrderDto.FromOrder(order);
    }
}
```

### 2. DTOs (Data Transfer Objects)
DTO'lar, katmanlar arası veri transferini sağlar.

**Örnek Kod:**
```csharp
public class OrderDto
{
    public Guid Id { get; set; }
    public string OrderNumber { get; set; }
    public DateTime OrderDate { get; set; }
    public OrderStatus Status { get; set; }
    public List<OrderItemDto> Items { get; set; }

    public static OrderDto FromOrder(Order order)
    {
        return new OrderDto
        {
            Id = order.Id,
            OrderNumber = order.OrderNumber,
            OrderDate = order.OrderDate,
            Status = order.Status,
            Items = order.Items.Select(OrderItemDto.FromOrderItem).ToList()
        };
    }
}

public class CreateOrderRequest
{
    public string OrderNumber { get; set; }
    public List<OrderItemRequest> Items { get; set; }
}
```

### 3. Interfaces
Interface'ler, domain layer ile diğer katmanlar arasındaki bağımlılıkları tanımlar.

**Örnek Kod:**
```csharp
public interface IOrderRepository
{
    Task<Order> GetById(Guid id);
    Task Add(Order order);
    Task Update(Order order);
    Task Delete(Order order);
}

public interface IUnitOfWork
{
    Task<int> SaveChanges();
    Task BeginTransaction();
    Task CommitTransaction();
    Task RollbackTransaction();
}
```

### 4. Mappings
Mappings, domain nesneleri ile DTO'lar arasındaki dönüşümleri sağlar.

**Örnek Kod:**
```csharp
public static class OrderMapping
{
    public static OrderDto ToDto(this Order order)
    {
        return new OrderDto
        {
            Id = order.Id,
            OrderNumber = order.OrderNumber,
            OrderDate = order.OrderDate,
            Status = order.Status,
            Items = order.Items.Select(i => i.ToDto()).ToList()
        };
    }

    public static Order ToDomain(this CreateOrderRequest request)
    {
        var order = new Order(request.OrderNumber);
        // Mapping logic
        return order;
    }
}
```

### 5. Validators
Validators, gelen isteklerin doğruluğunu kontrol eder.

**Örnek Kod:**
```csharp
public class CreateOrderRequestValidator : AbstractValidator<CreateOrderRequest>
{
    public CreateOrderRequestValidator()
    {
        RuleFor(x => x.OrderNumber)
            .NotEmpty()
            .MaximumLength(50);

        RuleFor(x => x.Items)
            .NotEmpty()
            .Must(items => items.Count > 0)
            .WithMessage("En az bir ürün seçilmelidir");

        RuleForEach(x => x.Items)
            .SetValidator(new OrderItemRequestValidator());
    }
}

public class OrderItemRequestValidator : AbstractValidator<OrderItemRequest>
{
    public OrderItemRequestValidator()
    {
        RuleFor(x => x.ProductId)
            .NotEmpty();

        RuleFor(x => x.Quantity)
            .GreaterThan(0);
    }
}
```

## Best Practices

### 1. Use Case Tasarımı
- Her use case tek bir sorumluluğa sahip olmalıdır
- Use case'ler interface'ler üzerinden tanımlanmalıdır
- Use case'ler bağımsız olarak test edilebilir olmalıdır
- Use case'ler domain logic içermemelidir

### 2. DTO Tasarımı
- DTO'lar immutable olmalıdır
- DTO'lar sadece gerekli verileri içermelidir
- DTO'lar validation logic içermemelidir
- DTO'lar mapping logic içermemelidir

### 3. Interface Tasarımı
- Interface'ler küçük ve öz olmalıdır
- Interface'ler bağımlılıkları minimize etmelidir
- Interface'ler domain layer'ı izole etmelidir
- Interface'ler test edilebilirliği artırmalıdır

### 4. Validation Stratejisi
- Validation logic use case'lerden ayrı tutulmalıdır
- Validation rules açık ve anlaşılır olmalıdır
- Validation hataları anlamlı mesajlar içermelidir
- Validation işlemleri asenkron olabilmelidir

## Sık Sorulan Sorular

### 1. Use case ve service arasındaki fark nedir?
- Use case'ler belirli bir iş akışını temsil eder
- Service'ler genel işlevleri sağlar
- Use case'ler daha spesifiktir
- Service'ler daha geneldir

### 2. DTO'lar neden kullanılmalıdır?
- Domain nesnelerini korumak için
- Veri transferini optimize etmek için
- Katmanlar arası bağımlılıkları azaltmak için
- API kontratlarını tanımlamak için

### 3. Validation nerede yapılmalıdır?
- Request validation application layer'da
- Domain validation domain layer'da
- Business rule validation domain layer'da
- Cross-cutting validation infrastructure layer'da

## Kaynaklar
- [Microsoft Application Layer](https://docs.microsoft.com/tr-tr/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/ddd-oriented-microservice#the-application-layer)
- [Clean Architecture - Application Layer](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Application Layer Best Practices](https://docs.microsoft.com/tr-tr/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/ddd-oriented-microservice#the-application-layer) 