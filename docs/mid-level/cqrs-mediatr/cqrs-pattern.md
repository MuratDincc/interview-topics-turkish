# CQRS Pattern

## Genel Bakış
CQRS (Command Query Responsibility Segregation), bir uygulamadaki okuma (Query) ve yazma (Command) operasyonlarının birbirinden ayrı modeller ve handler'larla yönetilmesini öneren mimari bir pattern'dir. Greg Young tarafından popülerleştirilen bu yaklaşım, karmaşık iş kurallarına sahip sistemlerde kod okunabilirliğini, ölçeklenebilirliği ve test edilebilirliği önemli ölçüde artırır. CQRS, özellikle Domain-Driven Design (DDD) ve Event Sourcing ile birlikte kullanıldığında güçlü bir mimari zemin oluşturur.

## Mülakat Soruları ve Cevapları

### 1. Soru: CQRS nedir ve temel prensibi neden faydalıdır?
**Cevap:**
CQRS, Command (yazma/değiştirme işlemleri) ve Query (okuma işlemleri) sorumluluklarını birbirinden ayıran bir mimari pattern'dir. Geleneksel CRUD yaklaşımında tek bir model hem okuma hem yazma için kullanılır; bu durum zaman içinde modelin şişmesine ve karmaşıklaşmasına yol açar. CQRS'te ise Command tarafı iş kurallarını ve domain mantığını yönetirken, Query tarafı okuma için optimize edilmiş, denormalize veri modelleri sunar. Bu ayrım sayesinde her iki taraf bağımsız olarak evrilebilir, ölçeklendirilebilir ve test edilebilir.

Temel faydaları:
- Okuma ve yazma tarafları bağımsız optimize edilebilir
- Her handler tek bir sorumluluğa odaklanır (SRP)
- Yüksek trafikte okuma tarafı ayrıca ölçeklendirilebilir
- Audit log ve Event Sourcing entegrasyonu kolaylaşır

**Örnek Kod:**
```csharp
// Geleneksel yaklaşım — tek model her şeyi yönetir
public class ProductService
{
    private readonly AppDbContext _context;

    public ProductService(AppDbContext context)
    {
        _context = context;
    }

    // Okuma ve yazma aynı modeli kullanıyor
    public async Task<Product> GetByIdAsync(int id)
        => await _context.Products.FindAsync(id);

    public async Task UpdateAsync(Product product)
    {
        _context.Products.Update(product);
        await _context.SaveChangesAsync();
    }
}

// CQRS yaklaşımı — Command ve Query ayrılmış
// --- Command tarafı ---
public record UpdateProductCommand(int Id, string Name, decimal Price) : IRequest<Unit>;

public class UpdateProductCommandHandler : IRequestHandler<UpdateProductCommand, Unit>
{
    private readonly AppDbContext _context;

    public UpdateProductCommandHandler(AppDbContext context)
    {
        _context = context;
    }

    public async Task<Unit> Handle(UpdateProductCommand request, CancellationToken cancellationToken)
    {
        var product = await _context.Products.FindAsync(request.Id, cancellationToken);
        if (product is null)
            throw new NotFoundException(nameof(Product), request.Id);

        product.UpdateDetails(request.Name, request.Price);
        await _context.SaveChangesAsync(cancellationToken);
        return Unit.Value;
    }
}

// --- Query tarafı ---
public record GetProductByIdQuery(int Id) : IRequest<ProductDto>;

public class GetProductByIdQueryHandler : IRequestHandler<GetProductByIdQuery, ProductDto>
{
    private readonly IReadDbContext _readContext;

    public GetProductByIdQueryHandler(IReadDbContext readContext)
    {
        _readContext = readContext;
    }

    public async Task<ProductDto> Handle(GetProductByIdQuery request, CancellationToken cancellationToken)
    {
        return await _readContext.Products
            .Where(p => p.Id == request.Id)
            .Select(p => new ProductDto(p.Id, p.Name, p.Price, p.CategoryName))
            .FirstOrDefaultAsync(cancellationToken)
            ?? throw new NotFoundException(nameof(Product), request.Id);
    }
}
```

---

### 2. Soru: Command ve Query'yi nasıl tasarlarsınız? Aralarındaki farklar nelerdir?
**Cevap:**
Command, sistemde bir değişiklik yaratan işlemi temsil eder: kayıt oluşturma, güncelleme, silme gibi. Command'lar genellikle `Unit` ya da işleme özgü minimal bir sonuç döndürür (oluşturulan kaydın ID'si gibi). Query ise sistemi değiştirmeden veri okuyan işlemi temsil eder ve zengin bir DTO döndürür. Command'lar iş kurallarını içerebilirken, Query'ler doğrudan veri katmanına erişebilir ve okuma için optimize edilmiş projeksiyonlar kullanır.

**Örnek Kod:**
```csharp
// --- Command örnekleri ---

// Oluşturma komutu — yeni oluşturulan kaydın ID'sini döndürür
public record CreateOrderCommand(
    int CustomerId,
    List<OrderLineDto> Lines
) : IRequest<int>;

// Güncelleme komutu — yalnızca başarı/başarısızlık bilgisi döner
public record CancelOrderCommand(int OrderId, string Reason) : IRequest<Unit>;

// Silme komutu
public record DeleteProductCommand(int ProductId) : IRequest<Unit>;

// --- Query örnekleri ---

// Tekil kayıt sorgusu
public record GetOrderByIdQuery(int OrderId) : IRequest<OrderDetailDto>;

// Liste sorgusu — sayfalama ile
public record GetOrdersListQuery(
    int Page,
    int PageSize,
    int? CustomerId = null,
    OrderStatus? Status = null
) : IRequest<PagedResult<OrderSummaryDto>>;

// --- DTO tasarımı ---
// Read Model: denormalize, okuma için optimize
public record OrderDetailDto(
    int Id,
    string CustomerFullName,
    string CustomerEmail,
    DateTime OrderDate,
    OrderStatus Status,
    decimal TotalAmount,
    List<OrderLineDto> Lines
);

public record OrderSummaryDto(
    int Id,
    string CustomerFullName,
    DateTime OrderDate,
    OrderStatus Status,
    decimal TotalAmount
);

// Write Model (Domain entity): iş kurallarını korur
public class Order
{
    public int Id { get; private set; }
    public int CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public List<OrderLine> Lines { get; private set; } = new();

    public void Cancel(string reason)
    {
        if (Status == OrderStatus.Delivered)
            throw new DomainException("Teslim edilmiş sipariş iptal edilemez.");

        Status = OrderStatus.Cancelled;
        AddDomainEvent(new OrderCancelledEvent(Id, reason));
    }
}
```

---

### 3. Soru: CQRS'te Read Model ve Write Model neden ayrılır? Pratik avantajları nelerdir?
**Cevap:**
Write Model, domain mantığını, invariant'ları ve iş kurallarını koruyan zengin bir nesne modelidir. Bu model normalizeye uygun, tutarlılığa odaklı yapıdadır. Read Model ise kullanıcı arayüzü ya da API tüketicilerinin ihtiyaçlarına göre şekillendirilmiş, denormalize ve sorgu-optimized bir yapıdır. Bu ayrım sayesinde:
- Yazma tarafında veri bütünlüğü korunurken okuma tarafında N+1 sorgu problemi ve JOIN yükü azaltılır
- Okuma tarafı için ayrı bir veritabanı (örn. Redis, Elasticsearch, okuma replikası) kullanılabilir
- Her iki taraf bağımsız olarak değiştirilip deploy edilebilir

**Örnek Kod:**
```csharp
// Write Model — normalize, domain kurallarını korur
public class Customer
{
    public int Id { get; private set; }
    public string FirstName { get; private set; }
    public string LastName { get; private set; }
    public string Email { get; private set; }
    public List<Address> Addresses { get; private set; } = new();

    public void ChangeEmail(string newEmail)
    {
        if (string.IsNullOrWhiteSpace(newEmail))
            throw new DomainException("E-posta boş olamaz.");
        Email = newEmail;
    }
}

// Read Model — denormalize, JOIN'siz hızlı sorgu
public record CustomerDetailDto(
    int Id,
    string FullName,         // FirstName + LastName birleştirilmiş
    string Email,
    string DefaultAddress,   // Ayrı Address tablosundan flatten edilmiş
    int TotalOrderCount,
    decimal TotalSpent
);

// Query Handler — Read Model için optimize edilmiş sorgu
public class GetCustomerDetailQueryHandler
    : IRequestHandler<GetCustomerDetailQuery, CustomerDetailDto>
{
    private readonly IReadDbContext _db;

    public GetCustomerDetailQueryHandler(IReadDbContext db) => _db = db;

    public async Task<CustomerDetailDto> Handle(
        GetCustomerDetailQuery request,
        CancellationToken cancellationToken)
    {
        // Read Model: tek sorguda tüm veriler, JOIN ile
        return await _db.Customers
            .Where(c => c.Id == request.CustomerId)
            .Select(c => new CustomerDetailDto(
                c.Id,
                c.FirstName + " " + c.LastName,
                c.Email,
                c.Addresses.Where(a => a.IsDefault).Select(a => a.FullAddress).FirstOrDefault() ?? "",
                c.Orders.Count,
                c.Orders.Sum(o => o.TotalAmount)
            ))
            .FirstOrDefaultAsync(cancellationToken)
            ?? throw new NotFoundException("Müşteri bulunamadı.");
    }
}

// Command Handler — Write Model üzerinden iş kurallarıyla güncelleme
public class ChangeCustomerEmailCommandHandler
    : IRequestHandler<ChangeCustomerEmailCommand, Unit>
{
    private readonly AppDbContext _db;

    public ChangeCustomerEmailCommandHandler(AppDbContext db) => _db = db;

    public async Task<Unit> Handle(
        ChangeCustomerEmailCommand request,
        CancellationToken cancellationToken)
    {
        var customer = await _db.Customers.FindAsync(request.CustomerId, cancellationToken)
            ?? throw new NotFoundException("Müşteri bulunamadı.");

        customer.ChangeEmail(request.NewEmail); // Domain kuralı burada çalışır
        await _db.SaveChangesAsync(cancellationToken);
        return Unit.Value;
    }
}
```

---

### 4. Soru: CQRS'te Eventual Consistency nedir ve nasıl yönetilir?
**Cevap:**
Eventual Consistency (nihai tutarlılık), özellikle Write ve Read Model'lerin farklı veri depolarında tutulduğu senaryolarda ortaya çıkar. Bir Command işlendiğinde Write tarafındaki değişiklik anında gerçekleşir, ancak Read Model bu değişikliği asenkron bir süreç aracılığıyla (domain event, message bus gibi) alır. Bu kısa gecikme süresinde kullanıcı eski veriyi görebilir. Yönetim stratejileri şunlardır: Optimistic UI (arayüzde anlık güncelleme varsayımı), polling, WebSocket ile anlık bildirim veya eventual consistency'i kullanıcıya açıkça bildirme.

**Örnek Kod:**
```csharp
// Domain Event tanımı
public record ProductPriceChangedEvent(int ProductId, decimal OldPrice, decimal NewPrice);

// Command Handler — Write Model günceller ve event fırlatır
public class UpdateProductPriceCommandHandler
    : IRequestHandler<UpdateProductPriceCommand, Unit>
{
    private readonly AppDbContext _writeDb;
    private readonly IEventBus _eventBus;

    public UpdateProductPriceCommandHandler(AppDbContext writeDb, IEventBus eventBus)
    {
        _writeDb = writeDb;
        _eventBus = eventBus;
    }

    public async Task<Unit> Handle(
        UpdateProductPriceCommand request,
        CancellationToken cancellationToken)
    {
        var product = await _writeDb.Products.FindAsync(request.ProductId, cancellationToken)
            ?? throw new NotFoundException("Ürün bulunamadı.");

        var oldPrice = product.Price;
        product.UpdatePrice(request.NewPrice);
        await _writeDb.SaveChangesAsync(cancellationToken);

        // Read Model'i güncelleyecek event'i yayınla (asenkron)
        await _eventBus.PublishAsync(
            new ProductPriceChangedEvent(product.Id, oldPrice, request.NewPrice),
            cancellationToken);

        return Unit.Value;
    }
}

// Read Model Projector — event'i alarak okuma tarafını günceller
public class ProductPriceChangedProjector
    : INotificationHandler<ProductPriceChangedEvent>
{
    private readonly IReadDbContext _readDb;
    private readonly ILogger<ProductPriceChangedProjector> _logger;

    public ProductPriceChangedProjector(
        IReadDbContext readDb,
        ILogger<ProductPriceChangedProjector> logger)
    {
        _readDb = readDb;
        _logger = logger;
    }

    public async Task Handle(
        ProductPriceChangedEvent notification,
        CancellationToken cancellationToken)
    {
        // Read Model'deki ürün fiyatını güncelle
        var readModel = await _readDb.ProductReadModels
            .FindAsync(notification.ProductId, cancellationToken);

        if (readModel is null)
        {
            _logger.LogWarning(
                "Read Model bulunamadı: ProductId={ProductId}", notification.ProductId);
            return;
        }

        readModel.Price = notification.NewPrice;
        readModel.LastUpdatedAt = DateTime.UtcNow;
        await _readDb.SaveChangesAsync(cancellationToken);

        _logger.LogInformation(
            "Read Model güncellendi: ProductId={ProductId}, YeniFiyat={Price}",
            notification.ProductId, notification.NewPrice);
    }
}

// Eventual Consistency ile API yanıtı
[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    private readonly IMediator _mediator;

    public ProductsController(IMediator mediator) => _mediator = mediator;

    [HttpPut("{id}/price")]
    public async Task<IActionResult> UpdatePrice(int id, UpdateProductPriceRequest request)
    {
        await _mediator.Send(new UpdateProductPriceCommand(id, request.NewPrice));

        // Eventual consistency bilgisini header ile ilet
        Response.Headers.Append("X-Consistency-Model", "eventual");
        return Accepted(new
        {
            Message = "Fiyat güncelleme isteği alındı. Değişiklikler kısa süre içinde yansıyacak.",
            ProductId = id
        });
    }
}
```

---

### 5. Soru: CQRS'i Clean Architecture ile nasıl entegre edersiniz?
**Cevap:**
Clean Architecture'da CQRS en iyi Application katmanında uygulanır. Command ve Query'ler Application katmanındaki handler'larla yönetilir. Domain katmanı iş kurallarını ve entity'leri barındırır; Infrastructure katmanı ise veritabanı ve dış servis erişimlerini sağlar. Presentation (API) katmanı yalnızca MediatR üzerinden komutları ve sorguları gönderir; doğrudan servis çağrısı yapmaz. Bu yapı sayesinde her katmanın bağımlılıkları net olarak belirlenir ve test edilebilirlik en üst düzeye çıkar.

**Örnek Kod:**
```csharp
// Proje yapısı:
// MyApp.Domain         — Entity'ler, domain event'ler, interface'ler
// MyApp.Application    — Command'lar, Query'ler, Handler'lar, DTO'lar
// MyApp.Infrastructure — DbContext, Repository implementasyonları
// MyApp.Api            — Controller'lar, Program.cs

// --- Domain katmanı ---
// MyApp.Domain/Entities/Order.cs
public class Order
{
    public int Id { get; private set; }
    public OrderStatus Status { get; private set; }
    private readonly List<OrderLine> _lines = new();
    public IReadOnlyCollection<OrderLine> Lines => _lines.AsReadOnly();

    public static Order Create(int customerId, List<(int ProductId, int Qty, decimal Price)> lines)
    {
        if (!lines.Any())
            throw new DomainException("Sipariş en az bir ürün içermelidir.");

        var order = new Order { Status = OrderStatus.Pending };
        foreach (var (productId, qty, price) in lines)
            order._lines.Add(new OrderLine(productId, qty, price));

        return order;
    }
}

// --- Application katmanı ---
// MyApp.Application/Orders/Commands/CreateOrder/CreateOrderCommand.cs
public record CreateOrderCommand(
    int CustomerId,
    List<CreateOrderLineDto> Lines
) : IRequest<int>;

// MyApp.Application/Orders/Commands/CreateOrder/CreateOrderCommandHandler.cs
public class CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, int>
{
    private readonly IAppDbContext _context;

    public CreateOrderCommandHandler(IAppDbContext context) => _context = context;

    public async Task<int> Handle(
        CreateOrderCommand request,
        CancellationToken cancellationToken)
    {
        var lines = request.Lines
            .Select(l => (l.ProductId, l.Quantity, l.UnitPrice))
            .ToList();

        var order = Order.Create(request.CustomerId, lines);
        _context.Orders.Add(order);
        await _context.SaveChangesAsync(cancellationToken);
        return order.Id;
    }
}

// MyApp.Application/Orders/Queries/GetOrderById/GetOrderByIdQuery.cs
public record GetOrderByIdQuery(int OrderId) : IRequest<OrderDetailDto>;

// MyApp.Application/Orders/Queries/GetOrderById/GetOrderByIdQueryHandler.cs
public class GetOrderByIdQueryHandler : IRequestHandler<GetOrderByIdQuery, OrderDetailDto>
{
    private readonly IAppDbContext _context;

    public GetOrderByIdQueryHandler(IAppDbContext context) => _context = context;

    public async Task<OrderDetailDto> Handle(
        GetOrderByIdQuery request,
        CancellationToken cancellationToken)
    {
        return await _context.Orders
            .Where(o => o.Id == request.OrderId)
            .Select(o => new OrderDetailDto(
                o.Id,
                o.Customer.FirstName + " " + o.Customer.LastName,
                o.Status,
                o.Lines.Sum(l => l.Quantity * l.UnitPrice)
            ))
            .FirstOrDefaultAsync(cancellationToken)
            ?? throw new NotFoundException("Sipariş bulunamadı.");
    }
}

// --- Presentation katmanı ---
// MyApp.Api/Controllers/OrdersController.cs
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly IMediator _mediator;

    public OrdersController(IMediator mediator) => _mediator = mediator;

    [HttpPost]
    public async Task<IActionResult> Create(CreateOrderRequest request)
    {
        var orderId = await _mediator.Send(new CreateOrderCommand(
            request.CustomerId, request.Lines));
        return CreatedAtAction(nameof(GetById), new { id = orderId }, new { Id = orderId });
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetById(int id)
    {
        var order = await _mediator.Send(new GetOrderByIdQuery(id));
        return Ok(order);
    }
}
```

---

### 6. Soru: CQRS'te hata yönetimi nasıl yapılır? Result pattern kullanımı neden önerilir?
**Cevap:**
CQRS handler'larından exception fırlatmak yerine Result pattern kullanmak tercih edilir; bu yaklaşım, akış kontrolünü exception mekanizmasına bağlamaktan kaçınır ve hata durumlarını açıkça tip güvenli bir şekilde ifade eder. Result pattern, başarılı durumda değer, başarısız durumda hata bilgisi döndürür. Bu sayede handler'lar her zaman deterministik bir sonuç üretir ve çağıran taraf hata durumunu açık olarak işler.

**Örnek Kod:**
```csharp
// Generic Result type
public class Result<T>
{
    public bool IsSuccess { get; private init; }
    public T? Value { get; private init; }
    public string? Error { get; private init; }
    public List<string> ValidationErrors { get; private init; } = new();

    private Result() { }

    public static Result<T> Success(T value)
        => new() { IsSuccess = true, Value = value };

    public static Result<T> Failure(string error)
        => new() { IsSuccess = false, Error = error };

    public static Result<T> ValidationFailure(List<string> errors)
        => new() { IsSuccess = false, ValidationErrors = errors };
}

// Command — Result döndüren versiyon
public record RegisterUserCommand(
    string Email,
    string Password,
    string FullName
) : IRequest<Result<int>>;

// Handler — exception yerine Result döndürür
public class RegisterUserCommandHandler
    : IRequestHandler<RegisterUserCommand, Result<int>>
{
    private readonly AppDbContext _db;

    public RegisterUserCommandHandler(AppDbContext db) => _db = db;

    public async Task<Result<int>> Handle(
        RegisterUserCommand request,
        CancellationToken cancellationToken)
    {
        // İş kuralı kontrolü
        var emailExists = await _db.Users.AnyAsync(
            u => u.Email == request.Email, cancellationToken);

        if (emailExists)
            return Result<int>.Failure("Bu e-posta adresi zaten kayıtlı.");

        var user = new User(request.Email, request.FullName, request.Password);
        _db.Users.Add(user);
        await _db.SaveChangesAsync(cancellationToken);

        return Result<int>.Success(user.Id);
    }
}

// Controller — Result'u HTTP yanıtına dönüştürür
[HttpPost("register")]
public async Task<IActionResult> Register(RegisterUserRequest request)
{
    var result = await _mediator.Send(
        new RegisterUserCommand(request.Email, request.Password, request.FullName));

    if (!result.IsSuccess)
    {
        if (result.ValidationErrors.Any())
            return BadRequest(new { Errors = result.ValidationErrors });

        return Conflict(new { Error = result.Error });
    }

    return CreatedAtAction(nameof(GetById), new { id = result.Value }, null);
}
```

---

### 7. Soru: CQRS'in dezavantajları ve dikkat edilmesi gereken durumlar nelerdir?
**Cevap:**
CQRS güçlü bir pattern olmakla birlikte her proje için doğru tercih olmayabilir. Küçük ve basit CRUD uygulamalarına uygulandığında gereksiz karmaşıklık ve boilerplate kod yüküne yol açar. Dikkat edilmesi gereken başlıca noktalar şunlardır: eventual consistency yönetiminin zorluğu, iki ayrı modelin bakım yükü, takımın pattern hakkında yeterli bilgiye sahip olması gerekliliği ve proje büyüklüğüne uygunluk değerlendirmesi.

**Örnek Kod:**
```csharp
// CQRS uygun OLMAYAN senaryo — basit kullanıcı yönetimi
// Bu kadar basit bir CRUD için CQRS gereksiz karmaşıklık ekler
public record CreateSimpleUserCommand(string Name, string Email) : IRequest<int>;
public class CreateSimpleUserHandler : IRequestHandler<CreateSimpleUserCommand, int>
{
    // Sadece veritabanına kayıt, iş kuralı yok — CQRS overkill
}

// CQRS uygun senaryo — karmaşık iş kuralları
public record ProcessLoanApplicationCommand(
    int ApplicantId,
    decimal RequestedAmount,
    int TermMonths,
    List<int> CollateralIds
) : IRequest<Result<LoanApplicationDto>>;

public class ProcessLoanApplicationHandler
    : IRequestHandler<ProcessLoanApplicationCommand, Result<LoanApplicationDto>>
{
    private readonly ICreditScoreService _creditService;
    private readonly ICollateralValuationService _valuationService;
    private readonly AppDbContext _db;
    private readonly IEventBus _eventBus;

    public ProcessLoanApplicationHandler(
        ICreditScoreService creditService,
        ICollateralValuationService valuationService,
        AppDbContext db,
        IEventBus eventBus)
    {
        _creditService = creditService;
        _valuationService = valuationService;
        _db = db;
        _eventBus = eventBus;
    }

    public async Task<Result<LoanApplicationDto>> Handle(
        ProcessLoanApplicationCommand request,
        CancellationToken cancellationToken)
    {
        // Karmaşık iş kuralları — CQRS burada gerçek değer sağlar
        var creditScore = await _creditService.GetScoreAsync(request.ApplicantId);
        if (creditScore < 600)
            return Result<LoanApplicationDto>.Failure("Kredi notu yetersiz.");

        var totalCollateralValue = await _valuationService
            .CalculateTotalValueAsync(request.CollateralIds);

        if (totalCollateralValue < request.RequestedAmount * 0.8m)
            return Result<LoanApplicationDto>.Failure("Teminat değeri yetersiz.");

        var application = LoanApplication.Create(
            request.ApplicantId, request.RequestedAmount,
            request.TermMonths, request.CollateralIds, creditScore);

        _db.LoanApplications.Add(application);
        await _db.SaveChangesAsync(cancellationToken);
        await _eventBus.PublishAsync(new LoanApplicationSubmittedEvent(application.Id));

        return Result<LoanApplicationDto>.Success(LoanApplicationDto.From(application));
    }
}
```

---

## Best Practices

### 1. **Klasör Yapısı**
Handler'ları feature'a göre organize edin; `Commands/` ve `Queries/` olarak ayırın. Her handler kendi alt klasöründe ilgili DTO ve validator ile birlikte bulunmalıdır.

```
Application/
  Orders/
    Commands/
      CreateOrder/
        CreateOrderCommand.cs
        CreateOrderCommandHandler.cs
        CreateOrderCommandValidator.cs
    Queries/
      GetOrderById/
        GetOrderByIdQuery.cs
        GetOrderByIdQueryHandler.cs
        OrderDetailDto.cs
```

### 2. **Command Tasarım Kuralları**
- Command'lar immutable olmalıdır (record tercih edilir)
- Command'lar yalnızca gerekli verileri içermelidir
- Command adları eylem bildiren fiillerle başlamalıdır: `CreateOrder`, `CancelOrder`
- Büyük veri listesi içeren Command'larda sayfalama veya toplu işlem pattern'leri değerlendirilmelidir

### 3. **Query Tasarım Kuralları**
- Query'ler yan etkisiz (side-effect free) olmalıdır
- Read Model'ler kullanıcı arayüzü ihtiyaçlarına göre şekillendirilmelidir
- N+1 sorgu problemini önlemek için projection ve `Select` kullanılmalıdır
- Büyük veri setleri için sayfalama zorunlu tutulmalıdır

### 4. **Handler Sorumlulukları**
- Handler'lar thin (ince) tutulmalı; karmaşık iş mantığı domain nesnelerine ya da domain servislerine taşınmalıdır
- Her handler bağımsız test edilebilir olmalıdır
- Handler'larda doğrudan HTTP bağımlılığı olmamalıdır

## Kaynaklar

- [CQRS - Martin Fowler](https://martinfowler.com/bliki/CQRS.html)
- [Microsoft CQRS Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [Greg Young — CQRS Documents](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf)
- [CQRS with Clean Architecture](https://jasontaylor.dev/clean-architecture-getting-started/)
- [Event Sourcing ve CQRS](https://docs.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)
