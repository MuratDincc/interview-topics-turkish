# MediatR Pipeline

## Genel Bakış
MediatR, .NET ekosisteminde mediator pattern'ini uygulayan, Jimmy Bogard tarafından geliştirilen açık kaynaklı bir kütüphanedir. Temel amacı, bileşenler arasındaki doğrudan bağımlılıkları ortadan kaldırarak isteklerin (request) ve bu istekleri işleyen handler'ların birbirinden bağımsız olmasını sağlamaktır. MediatR'ın en güçlü özelliklerinden biri Pipeline Behavior'lardır; bu yapı sayesinde doğrulama, loglama, transaction yönetimi ve hata yakalama gibi cross-cutting concern'ler merkezi bir noktada, her handler'ı değiştirmek zorunda kalmadan uygulanabilir. MediatR ayrıca Notification (bildirim) mekanizmasıyla birden fazla handler'a mesaj yayınlamayı da destekler.

## Mülakat Soruları ve Cevapları

### 1. Soru: MediatR nedir ve IRequest / IRequestHandler nasıl çalışır?
**Cevap:**
MediatR, mediator design pattern'ini uygular; gönderici (sender) ve alıcı (handler) arasındaki doğrudan bağı keserek her ikisini de birbirinden bağımsız kılar. `IRequest<TResponse>` interface'i, bir isteği temsil eden marker'dır. `IRequestHandler<TRequest, TResponse>` ise o isteği karşılayan handler'ın implement edeceği interface'dir. `IMediator.Send()` çağrıldığında MediatR, isteğin tipine göre doğru handler'ı bulur, pipeline'dan geçirir ve handler'ı çağırır. Bu mekanizma sayesinde gönderici, handler'ı hiç tanımak zorunda değildir.

**Örnek Kod:**
```csharp
// 1. Paket kurulumu
// dotnet add package MediatR
// dotnet add package MediatR.Extensions.Microsoft.DependencyInjection

// 2. Program.cs — MediatR kayıt
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssembly(typeof(Program).Assembly));

// 3. IRequest implementasyonu — hem Command hem Query için kullanılır
public record GetProductQuery(int ProductId) : IRequest<ProductDto>;

// Unit döndüren istek — yanıt gerekmediğinde
public record DeleteProductCommand(int ProductId) : IRequest<Unit>;

// 4. IRequestHandler implementasyonu
public class GetProductQueryHandler : IRequestHandler<GetProductQuery, ProductDto>
{
    private readonly AppDbContext _db;
    private readonly ILogger<GetProductQueryHandler> _logger;

    public GetProductQueryHandler(AppDbContext db, ILogger<GetProductQueryHandler> logger)
    {
        _db = db;
        _logger = logger;
    }

    // Handle metodu: her zaman async, CancellationToken alır
    public async Task<ProductDto> Handle(
        GetProductQuery request,
        CancellationToken cancellationToken)
    {
        _logger.LogInformation("Ürün sorgulanıyor: {ProductId}", request.ProductId);

        var product = await _db.Products
            .Where(p => p.Id == request.ProductId && !p.IsDeleted)
            .Select(p => new ProductDto(p.Id, p.Name, p.Price, p.Stock))
            .FirstOrDefaultAsync(cancellationToken);

        if (product is null)
            throw new NotFoundException($"Ürün bulunamadı: {request.ProductId}");

        return product;
    }
}

// 5. Controller'da kullanım
[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    private readonly IMediator _mediator;

    public ProductsController(IMediator mediator) => _mediator = mediator;

    [HttpGet("{id}")]
    public async Task<IActionResult> Get(int id)
    {
        // IMediator.Send() — handler'ı doğrudan bilmeden isteği iletir
        var product = await _mediator.Send(new GetProductQuery(id));
        return Ok(product);
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(int id)
    {
        await _mediator.Send(new DeleteProductCommand(id));
        return NoContent();
    }
}
```

---

### 2. Soru: IPipelineBehavior nedir ve nasıl implement edilir?
**Cevap:**
`IPipelineBehavior<TRequest, TResponse>`, MediatR'da bir isteğin handler'a ulaşmadan önce ve sonra ara işlem yapılmasını sağlayan interface'dir. ASP.NET Core middleware'ine benzer şekilde çalışır: her behavior bir sonraki davranışı veya handler'ı `next()` çağrısıyla tetikler. Birden fazla behavior kayıt edilebildiğinde bunlar kayıt sırasına göre bir zincir (pipeline) oluşturur. Bu yapı sayesinde loglama, doğrulama, transaction yönetimi ve önbellekleme gibi cross-cutting concern'ler her handler'a tek tek eklenmek zorunda kalınmadan merkezi olarak uygulanır.

**Örnek Kod:**
```csharp
// Genel IPipelineBehavior yapısı
public class LoggingBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;

    public LoggingBehavior(ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    {
        _logger = logger;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next, // Sonraki davranış veya handler
        CancellationToken cancellationToken)
    {
        var requestName = typeof(TRequest).Name;
        _logger.LogInformation("İstek başladı: {RequestName} {@Request}", requestName, request);

        var stopwatch = Stopwatch.StartNew();
        try
        {
            // Handler'ı (veya sonraki behavior'ı) çağır
            var response = await next();

            stopwatch.Stop();
            _logger.LogInformation(
                "İstek tamamlandı: {RequestName} — {ElapsedMs}ms",
                requestName, stopwatch.ElapsedMilliseconds);

            return response;
        }
        catch (Exception ex)
        {
            stopwatch.Stop();
            _logger.LogError(ex,
                "İstek başarısız: {RequestName} — {ElapsedMs}ms",
                requestName, stopwatch.ElapsedMilliseconds);
            throw;
        }
    }
}

// Performans uyarı behavior'ı — yavaş istekleri loglayan versiyon
public class PerformanceBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private const int SlowRequestThresholdMs = 500;
    private readonly ILogger<PerformanceBehavior<TRequest, TResponse>> _logger;
    private readonly ICurrentUserService _currentUser;

    public PerformanceBehavior(
        ILogger<PerformanceBehavior<TRequest, TResponse>> logger,
        ICurrentUserService currentUser)
    {
        _logger = logger;
        _currentUser = currentUser;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var timer = Stopwatch.StartNew();
        var response = await next();
        timer.Stop();

        if (timer.ElapsedMilliseconds > SlowRequestThresholdMs)
        {
            _logger.LogWarning(
                "Yavaş istek tespit edildi: {RequestName} — {ElapsedMs}ms — Kullanıcı: {UserId} — {@Request}",
                typeof(TRequest).Name,
                timer.ElapsedMilliseconds,
                _currentUser.UserId,
                request);
        }

        return response;
    }
}

// DI kaydı — sıralama önemlidir!
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(PerformanceBehavior<,>));
```

---

### 3. Soru: MediatR ile FluentValidation doğrulama pipeline'ı nasıl kurulur?
**Cevap:**
Validation Pipeline Behavior, handler'a ulaşmadan önce isteği doğrulayan bir katman ekler. FluentValidation kütüphanesiyle birlikte kullanıldığında her Command veya Query için güçlü, okunabilir doğrulama kuralları yazılabilir. Doğrulama başarısız olursa handler hiç çalışmaz; hata bilgileri `ValidationException` ile üst katmana iletilir. Bu yaklaşım, handler'lar içinde tekrarlayan doğrulama kodunu ortadan kaldırır ve her isteğin validator'ını ayrı test etmeyi mümkün kılar.

**Örnek Kod:**
```csharp
// dotnet add package FluentValidation
// dotnet add package FluentValidation.DependencyInjectionExtensions

// 1. Command tanımı
public record CreateProductCommand(
    string Name,
    string Description,
    decimal Price,
    int Stock,
    int CategoryId
) : IRequest<int>;

// 2. FluentValidation Validator
public class CreateProductCommandValidator : AbstractValidator<CreateProductCommand>
{
    private readonly AppDbContext _db;

    public CreateProductCommandValidator(AppDbContext db)
    {
        _db = db;

        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Ürün adı boş olamaz.")
            .MaximumLength(200).WithMessage("Ürün adı 200 karakterden uzun olamaz.")
            .MustAsync(BeUniqueName).WithMessage("Bu ürün adı zaten kullanımda.");

        RuleFor(x => x.Description)
            .NotEmpty().WithMessage("Açıklama boş olamaz.")
            .MaximumLength(2000).WithMessage("Açıklama 2000 karakterden uzun olamaz.");

        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("Fiyat sıfırdan büyük olmalıdır.")
            .LessThanOrEqualTo(999999.99m).WithMessage("Fiyat maksimum değeri aşıyor.");

        RuleFor(x => x.Stock)
            .GreaterThanOrEqualTo(0).WithMessage("Stok negatif olamaz.");

        RuleFor(x => x.CategoryId)
            .GreaterThan(0).WithMessage("Geçersiz kategori.")
            .MustAsync(CategoryExist).WithMessage("Belirtilen kategori bulunamadı.");
    }

    private async Task<bool> BeUniqueName(string name, CancellationToken ct)
        => !await _db.Products.AnyAsync(p => p.Name == name, ct);

    private async Task<bool> CategoryExist(int categoryId, CancellationToken ct)
        => await _db.Categories.AnyAsync(c => c.Id == categoryId, ct);
}

// 3. Validation Pipeline Behavior
public class ValidationBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
    {
        _validators = validators;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        if (!_validators.Any())
            return await next(); // Validator yoksa direkt geç

        var context = new ValidationContext<TRequest>(request);

        // Tüm validator'ları paralel çalıştır
        var validationResults = await Task.WhenAll(
            _validators.Select(v => v.ValidateAsync(context, cancellationToken)));

        // Hataları topla
        var failures = validationResults
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Any())
            throw new ValidationException(failures);

        return await next();
    }
}

// 4. DI kaydı
builder.Services.AddValidatorsFromAssembly(typeof(Program).Assembly);
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));

// 5. Global exception handler ile ValidationException yakalama
// Program.cs veya Middleware
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var exception = context.Features.Get<IExceptionHandlerFeature>()?.Error;

        if (exception is ValidationException validationEx)
        {
            context.Response.StatusCode = 400;
            context.Response.ContentType = "application/json";

            var errors = validationEx.Errors
                .GroupBy(e => e.PropertyName)
                .ToDictionary(
                    g => g.Key,
                    g => g.Select(e => e.ErrorMessage).ToArray());

            await context.Response.WriteAsJsonAsync(new { Errors = errors });
        }
    });
});
```

---

### 4. Soru: MediatR'da Transaction Pipeline Behavior nasıl implement edilir?
**Cevap:**
Transaction Pipeline Behavior, Command'ların bir veritabanı transaction'ı içinde çalışmasını sağlar. Behavior başlamadan önce transaction açar, handler başarıyla tamamlandıysa commit eder; bir exception oluşursa rollback yapar. Bu yapı sayesinde her Command handler'ına ayrı ayrı transaction kodu eklemek gerekmez. Genellikle bu behavior yalnızca Command'lara uygulanır (Query'ler veriyi değiştirmediğinden transaction gerektirmez); bunun için marker interface kullanılabilir.

**Örnek Kod:**
```csharp
// Command'ları işaretlemek için marker interface
public interface ICommand<TResponse> : IRequest<TResponse> { }
public interface ICommand : IRequest<Unit> { }

// Command örnekleri
public record TransferFundsCommand(
    int FromAccountId,
    int ToAccountId,
    decimal Amount
) : ICommand<Unit>;

// Transaction Behavior — yalnızca ICommand için çalışır
public class TransactionBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : ICommand<TResponse>
{
    private readonly AppDbContext _dbContext;
    private readonly ILogger<TransactionBehavior<TRequest, TResponse>> _logger;

    public TransactionBehavior(
        AppDbContext dbContext,
        ILogger<TransactionBehavior<TRequest, TResponse>> logger)
    {
        _dbContext = dbContext;
        _logger = logger;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var requestName = typeof(TRequest).Name;
        _logger.LogInformation("Transaction başlatılıyor: {RequestName}", requestName);

        await using var transaction = await _dbContext.Database
            .BeginTransactionAsync(cancellationToken);

        try
        {
            var response = await next();

            await transaction.CommitAsync(cancellationToken);
            _logger.LogInformation("Transaction commit edildi: {RequestName}", requestName);

            return response;
        }
        catch (Exception ex)
        {
            await transaction.RollbackAsync(cancellationToken);
            _logger.LogError(ex, "Transaction rollback yapıldı: {RequestName}", requestName);
            throw;
        }
    }
}

// Command Handler — transaction management tamamen pipeline'da, handler temiz kalır
public class TransferFundsCommandHandler : IRequestHandler<TransferFundsCommand, Unit>
{
    private readonly AppDbContext _db;

    public TransferFundsCommandHandler(AppDbContext db) => _db = db;

    public async Task<Unit> Handle(
        TransferFundsCommand request,
        CancellationToken cancellationToken)
    {
        var fromAccount = await _db.Accounts.FindAsync(request.FromAccountId, cancellationToken)
            ?? throw new NotFoundException("Kaynak hesap bulunamadı.");

        var toAccount = await _db.Accounts.FindAsync(request.ToAccountId, cancellationToken)
            ?? throw new NotFoundException("Hedef hesap bulunamadı.");

        if (fromAccount.Balance < request.Amount)
            throw new DomainException("Yetersiz bakiye.");

        fromAccount.Debit(request.Amount);
        toAccount.Credit(request.Amount);

        await _db.SaveChangesAsync(cancellationToken);
        return Unit.Value;
    }
}

// Sıralı DI kaydı: Logging → Validation → Transaction → Handler
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(TransactionBehavior<,>));
```

---

### 5. Soru: MediatR INotification ve INotificationHandler nasıl çalışır? Ne zaman kullanılır?
**Cevap:**
`INotification`, birden fazla handler'ın dinleyebildiği, yayın/abonelik (publish/subscribe) modeline dayalı mesajlar için kullanılır. `IMediator.Publish()` çağrıldığında ilgili tüm `INotificationHandler<T>` handler'ları çalıştırılır. Bu mekanizma özellikle domain event'lerin yayınlanması, e-posta gönderimi, cache invalidation, audit log gibi ana iş akışından bağımsız yan işlemler için idealdir. `IRequest.Send()`'den farkı şudur: Request tek bir handler ile eşleşir ve yanıt döndürür; Notification birden fazla handler'a iletilir ve yanıt döndürmez.

**Örnek Kod:**
```csharp
// 1. Notification tanımı — bir domain event
public record OrderPlacedNotification(
    int OrderId,
    int CustomerId,
    decimal TotalAmount,
    string CustomerEmail
) : INotification;

// 2. Birden fazla handler — hepsi aynı notification'ı dinler

// Handler 1: E-posta gönder
public class SendOrderConfirmationEmailHandler
    : INotificationHandler<OrderPlacedNotification>
{
    private readonly IEmailService _emailService;
    private readonly ILogger<SendOrderConfirmationEmailHandler> _logger;

    public SendOrderConfirmationEmailHandler(
        IEmailService emailService,
        ILogger<SendOrderConfirmationEmailHandler> logger)
    {
        _emailService = emailService;
        _logger = logger;
    }

    public async Task Handle(
        OrderPlacedNotification notification,
        CancellationToken cancellationToken)
    {
        _logger.LogInformation(
            "Sipariş onay e-postası gönderiliyor: OrderId={OrderId}", notification.OrderId);

        await _emailService.SendAsync(new EmailMessage(
            To: notification.CustomerEmail,
            Subject: $"Siparişiniz alındı — #{notification.OrderId}",
            Body: $"Toplam tutar: {notification.TotalAmount:C2}. Siparişiniz için teşekkürler!"
        ));
    }
}

// Handler 2: Analytics servisine kaydet
public class TrackOrderAnalyticsHandler
    : INotificationHandler<OrderPlacedNotification>
{
    private readonly IAnalyticsService _analytics;

    public TrackOrderAnalyticsHandler(IAnalyticsService analytics)
    {
        _analytics = analytics;
    }

    public async Task Handle(
        OrderPlacedNotification notification,
        CancellationToken cancellationToken)
    {
        await _analytics.TrackEventAsync("order_placed", new
        {
            OrderId = notification.OrderId,
            CustomerId = notification.CustomerId,
            Amount = notification.TotalAmount
        });
    }
}

// Handler 3: Stok rezervasyonunu tetikle
public class ReserveStockHandler : INotificationHandler<OrderPlacedNotification>
{
    private readonly IStockService _stockService;

    public ReserveStockHandler(IStockService stockService)
    {
        _stockService = stockService;
    }

    public async Task Handle(
        OrderPlacedNotification notification,
        CancellationToken cancellationToken)
    {
        await _stockService.ReserveForOrderAsync(notification.OrderId, cancellationToken);
    }
}

// 3. Command Handler — notification yayınlar
public class PlaceOrderCommandHandler : IRequestHandler<PlaceOrderCommand, int>
{
    private readonly AppDbContext _db;
    private readonly IMediator _mediator;

    public PlaceOrderCommandHandler(AppDbContext db, IMediator mediator)
    {
        _db = db;
        _mediator = mediator;
    }

    public async Task<int> Handle(
        PlaceOrderCommand request,
        CancellationToken cancellationToken)
    {
        var order = Order.Create(request.CustomerId, request.Lines);
        _db.Orders.Add(order);
        await _db.SaveChangesAsync(cancellationToken);

        // Notification yayınla — tüm handler'lar bildirilir
        await _mediator.Publish(
            new OrderPlacedNotification(
                order.Id,
                order.CustomerId,
                order.TotalAmount,
                request.CustomerEmail),
            cancellationToken);

        return order.Id;
    }
}
```

---

### 6. Soru: MediatR pipeline'ında önbellekleme (caching) behavior'ı nasıl implement edilir?
**Cevap:**
Caching Pipeline Behavior, Query'lerin sonuçlarını önbelleğe alarak tekrarlayan veritabanı sorgularını önler. Bu behavior genellikle yalnızca Query'lere uygulanır; Command'lara uygulanmaz çünkü Command'lar veriyi değiştirir. Cacheable Query'ler bir marker interface ile işaretlenir ve cache key ile TTL (time-to-live) bilgisini sağlar. Behavior önce önbelleği kontrol eder; veri varsa direkt döndürür, yoksa handler'ı çalıştırıp sonucu önbelleğe alır.

**Örnek Kod:**
```csharp
// Cacheable query'leri işaretleyen interface
public interface ICacheableQuery
{
    string CacheKey { get; }
    TimeSpan CacheDuration { get; }
}

// Cacheable Query örneği
public record GetProductCatalogQuery(
    int CategoryId,
    int Page,
    int PageSize
) : IRequest<PagedResult<ProductSummaryDto>>, ICacheableQuery
{
    public string CacheKey
        => $"product-catalog:cat={CategoryId}:page={Page}:size={PageSize}";
    public TimeSpan CacheDuration => TimeSpan.FromMinutes(10);
}

// Cache Behavior
public class CachingBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IDistributedCache _cache;
    private readonly ILogger<CachingBehavior<TRequest, TResponse>> _logger;

    public CachingBehavior(
        IDistributedCache cache,
        ILogger<CachingBehavior<TRequest, TResponse>> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        // Yalnızca ICacheableQuery uygulayan istekler için çalış
        if (request is not ICacheableQuery cacheableRequest)
            return await next();

        var cacheKey = cacheableRequest.CacheKey;

        // Önbellekte var mı?
        var cachedBytes = await _cache.GetAsync(cacheKey, cancellationToken);
        if (cachedBytes is not null)
        {
            _logger.LogDebug("Önbellekten okundu: {CacheKey}", cacheKey);
            return JsonSerializer.Deserialize<TResponse>(cachedBytes)!;
        }

        // Yoksa handler'ı çalıştır
        var response = await next();

        // Sonucu önbelleğe yaz
        var serialized = JsonSerializer.SerializeToUtf8Bytes(response);
        await _cache.SetAsync(cacheKey, serialized, new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = cacheableRequest.CacheDuration
        }, cancellationToken);

        _logger.LogDebug(
            "Önbelleğe yazıldı: {CacheKey} — TTL: {Duration}",
            cacheKey, cacheableRequest.CacheDuration);

        return response;
    }
}

// Cache invalidation — Command başarılıysa cache'i temizle
public class InvalidateCacheBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : ICommand<TResponse>
{
    private readonly IDistributedCache _cache;
    private readonly ICacheInvalidationService _invalidationService;

    public InvalidateCacheBehavior(
        IDistributedCache cache,
        ICacheInvalidationService invalidationService)
    {
        _cache = cache;
        _invalidationService = invalidationService;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var response = await next();

        // Handler başarılı olduysa ilgili cache key'leri temizle
        var keysToInvalidate = _invalidationService.GetKeysToInvalidate(request);
        foreach (var key in keysToInvalidate)
            await _cache.RemoveAsync(key, cancellationToken);

        return response;
    }
}
```

---

### 7. Soru: MediatR'da tam bir pipeline zinciri nasıl kurulur ve pipeline sıralaması neden önemlidir?
**Cevap:**
MediatR'da pipeline sıralaması kritik öneme sahiptir çünkü behavior'lar DI'a kaydedilme sırasına göre zincir oluşturur. Tipik bir üretim pipeline'ı şu sırayla kurulur: Logging (her şeyi logla) → Exception Handling (hataları yakala) → Validation (geçersiz istekleri erkenden reddet) → Transaction (Command'lar için transaction aç) → Caching (Query'ler için cache kontrol et) → Handler (asıl işi yap). Loglama en dışta olmalıdır ki hem request başlangıcı hem sonucu kaydedilsin. Validation, transaction'dan önce gelmelidir ki geçersiz istekler için boşuna transaction açılmasın.

**Örnek Kod:**
```csharp
// Eksiksiz pipeline kurulumu — Program.cs
var builder = WebApplication.CreateBuilder(args);

// MediatR kaydı
builder.Services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(typeof(Program).Assembly);
});

// FluentValidation kaydı
builder.Services.AddValidatorsFromAssembly(typeof(Program).Assembly);

// Pipeline Behavior'lar — sıralama çok önemli!
// 1. En dışta: Logging — tüm istekleri logla
builder.Services.AddTransient(
    typeof(IPipelineBehavior<,>),
    typeof(LoggingBehavior<,>));

// 2. Exception handling — handler'dan gelen exception'ları yakala
builder.Services.AddTransient(
    typeof(IPipelineBehavior<,>),
    typeof(ExceptionHandlingBehavior<,>));

// 3. Validation — geçersiz istekleri elerken transaction açma
builder.Services.AddTransient(
    typeof(IPipelineBehavior<,>),
    typeof(ValidationBehavior<,>));

// 4. Transaction — yalnızca Command'lar için
builder.Services.AddTransient(
    typeof(IPipelineBehavior<,>),
    typeof(TransactionBehavior<,>));

// 5. Caching — yalnızca Query'ler için
builder.Services.AddTransient(
    typeof(IPipelineBehavior<,>),
    typeof(CachingBehavior<,>));

// Global Exception Handler Behavior
public class ExceptionHandlingBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly ILogger<ExceptionHandlingBehavior<TRequest, TResponse>> _logger;

    public ExceptionHandlingBehavior(
        ILogger<ExceptionHandlingBehavior<TRequest, TResponse>> logger)
    {
        _logger = logger;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        try
        {
            return await next();
        }
        catch (NotFoundException ex)
        {
            _logger.LogWarning(ex, "Kayıt bulunamadı: {RequestName}", typeof(TRequest).Name);
            throw; // Controller/middleware'e bırak
        }
        catch (DomainException ex)
        {
            _logger.LogWarning(ex, "Domain hatası: {RequestName}", typeof(TRequest).Name);
            throw;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Beklenmedik hata: {RequestName}", typeof(TRequest).Name);
            throw;
        }
    }
}

// Gerçek pipeline akışı — bir CreateOrderCommand geldiğinde:
// Request → LoggingBehavior.before
//         → ExceptionHandlingBehavior.try
//           → ValidationBehavior (kuralları kontrol et)
//             → TransactionBehavior.BeginTransaction
//               → CreateOrderCommandHandler.Handle (asıl iş)
//             → TransactionBehavior.Commit
//           (ValidationException durumunda buraya gelinmez)
//         → ExceptionHandlingBehavior.catch (hata varsa)
//         → LoggingBehavior.after (süre loglanır)
// Response

// Servis kaydı doğrulama için basit test
public class PipelineOrderVerificationTest
{
    [Fact]
    public async Task Pipeline_ShouldExecuteInCorrectOrder()
    {
        var executionOrder = new List<string>();

        // ... mock behavior'lar executionOrder'a log ekler
        // assertion: ["Logging", "Validation", "Transaction", "Handler"]
    }
}
```

---

## Best Practices

### 1. **Pipeline Tasarım Kuralları**
- Behavior'lar ince (thin) ve tek sorumlu olmalıdır
- Generic constraint'lerle behavior'ların hangi istek türlerine uygulandığı kısıtlanmalıdır
- Her behavior ayrı birim testiyle doğrulanmalıdır
- Behavior'lar birbirinden bağımsız olmalı; önceki behavior'ın sonucuna bağlı olmamalıdır

### 2. **Performans Optimizasyonu**
- Handler'lar her zaman async yazılmalıdır
- CancellationToken her metoda aktarılmalıdır
- Gereksiz serileştirme/deserializasyon cache behavior'larında minimize edilmelidir
- Validator'lar `MustAsync` kullanırken veritabanı bağlantı havuzuna dikkat edilmelidir

### 3. **Test Stratejisi**
- Handler'lar bağımsız unit test ile test edilmelidir
- Pipeline behavior'ları ayrı ayrı test edilmelidir
- Entegrasyon testlerinde tüm pipeline birlikte doğrulanmalıdır
- MediatR'ı mock'lamak yerine gerçek mediator ile test etmek daha güvenilirdir

### 4. **Organizasyon**
- Behavior'lar kendi namespace/klasörlerinde tutulmalıdır
- Her feature'ın Command ve Query'leri ayrı klasörlerde olmalıdır
- Shared DTO'lar merkezi bir yerde tutulmalıdır
- Notification handler'ları ilgili feature klasöründe organize edilmelidir

## Kaynaklar

- [MediatR GitHub](https://github.com/jbogard/MediatR)
- [MediatR Pipeline Behaviors](https://github.com/jbogard/MediatR/wiki/Behaviors)
- [FluentValidation Docs](https://docs.fluentvalidation.net/en/latest/)
- [MediatR ile Clean Architecture](https://jasontaylor.dev/clean-architecture-getting-started/)
- [Jimmy Bogard Blog](https://jimmybogard.com/)
