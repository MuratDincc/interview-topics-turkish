# Legacy Modernization

## Genel Bakış
Legacy Modernization (Eski Sistem Modernizasyonu), artık bakımı zor, ölçeklenemeyen veya yeni iş gereksinimlerine yanıt veremeyen sistemlerin güncel mimariye, teknolojiye ve en iyi pratiklere uygun hale getirilmesi sürecidir. .NET ekosisteminde bu süreç çoğunlukla .NET Framework'ten .NET 8'e geçiş, Web Forms veya WCF tabanlı uygulamaların ASP.NET Core Minimal API'ye dönüşümü ve monolitik uygulamaların mikroservis mimarisine bölünmesi biçiminde karşımıza çıkar.

Modernizasyon yalnızca teknik bir süreç değil; iş sürekliliğini korurken teknik borcu azaltma, geliştirici verimliliğini artırma ve sistemin yaşam süresini uzatma hedeflerini birleştiren stratejik bir programdır.

## Mülakat Soruları ve Cevapları

### 1. .NET Framework'ten .NET 8'e geçişte hangi adımlar izlenir?
**Cevap:**
Microsoft'un resmi upgrade yolu birkaç ana adımdan oluşur: (1) .NET Upgrade Assistant ve API Analyzer araçlarıyla uyumluluk değerlendirmesi yapılır, (2) üçüncü parti bağımlılıkların .NET 8 desteği kontrol edilir, (3) proje dosyaları yeni SDK stil formatına dönüştürülür, (4) platform-specific API'ler (WCF, Remoting, Web Forms) yeni karşılıklarıyla değiştirilir, (5) startup yapısı (`Global.asax`, `web.config`) `Program.cs` ve `appsettings.json`'a taşınır, (6) bağımlılık enjeksiyonu container'a geçirilir, (7) test kapsamı doğrulanır.

**Örnek Kod:**
```csharp
// ESKİ: .NET Framework - Global.asax ve WebApiConfig
// Global.asax.cs
public class WebApiApplication : System.Web.HttpApplication
{
    protected void Application_Start()
    {
        GlobalConfiguration.Configure(WebApiConfig.Register);
        FilterConfig.RegisterGlobalFilters(GlobalFilters.Filters);
        RouteConfig.RegisterRoutes(RouteTable.Routes);
    }
}

// WebApiConfig.cs
public static class WebApiConfig
{
    public static void Register(HttpConfiguration config)
    {
        config.MapHttpAttributeRoutes();
        config.Routes.MapHttpRoute(
            name: "DefaultApi",
            routeTemplate: "api/{controller}/{id}",
            defaults: new { id = RouteParameter.Optional }
        );
    }
}

// Bağımlılık yönetimi: Unity veya Autofac container
var container = new UnityContainer();
container.RegisterType<IOrderRepository, OrderRepository>();
GlobalConfiguration.Configuration.DependencyResolver =
    new UnityDependencyResolver(container);

// -------------------------------------------------------

// YENİ: .NET 8 - Program.cs Minimal Hosting
var builder = WebApplication.CreateBuilder(args);

// Yerleşik DI container
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddScoped<IOrderService, OrderService>();

// Middleware pipeline
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Yapılandırma: appsettings.json ve environment variables
builder.Services.Configure<DatabaseOptions>(
    builder.Configuration.GetSection("Database"));

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

// Minimal API endpoint örneği
app.MapGet("/api/health", () => Results.Ok(new { Status = "Healthy", Timestamp = DateTime.UtcNow }))
   .WithName("HealthCheck")
   .WithOpenApi();

app.Run();
```

### 2. WCF servislerinden gRPC veya REST API'ye geçiş nasıl yapılır?
**Cevap:**
WCF (Windows Communication Foundation) .NET Core/5+ üzerinde desteklenmez. Geçiş seçenekleri şunlardır: (a) HTTP/REST tabanlı iletişim için ASP.NET Core Web API, (b) yüksek performanslı binary iletişim için gRPC, (c) geriye dönük uyumluluk için `CoreWCF` kütüphanesi. İkili iletişim ve güçlü tip kontrolü gerektiren durumlar için gRPC önerilir; geniş client uyumluluğu için REST tercih edilir.

**Örnek Kod:**
```csharp
// ESKİ: WCF Servis Sözleşmesi
[ServiceContract]
public interface IOrderService
{
    [OperationContract]
    OrderDto GetOrder(string orderId);

    [OperationContract]
    OrderDto CreateOrder(CreateOrderRequest request);
}

[DataContract]
public class OrderDto
{
    [DataMember] public string Id { get; set; }
    [DataMember] public string CustomerName { get; set; }
    [DataMember] public decimal TotalAmount { get; set; }
    [DataMember] public DateTime CreatedAt { get; set; }
}

// -------------------------------------------------------

// YENİ A: ASP.NET Core Web API (REST)
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;

    public OrdersController(IOrderService orderService)
    {
        _orderService = orderService;
    }

    [HttpGet("{orderId}")]
    [ProducesResponseType(typeof(OrderDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<OrderDto>> GetOrder(string orderId)
    {
        var order = await _orderService.GetOrderAsync(orderId);
        return order is null ? NotFound() : Ok(order);
    }

    [HttpPost]
    [ProducesResponseType(typeof(OrderDto), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<OrderDto>> CreateOrder([FromBody] CreateOrderRequest request)
    {
        var order = await _orderService.CreateOrderAsync(request);
        return CreatedAtAction(nameof(GetOrder), new { orderId = order.Id }, order);
    }
}

// -------------------------------------------------------

// YENİ B: gRPC Servis (orders.proto dosyasından üretilen)
// orders.proto:
// syntax = "proto3";
// service OrderService {
//   rpc GetOrder (GetOrderRequest) returns (OrderResponse);
//   rpc CreateOrder (CreateOrderRequest) returns (OrderResponse);
// }

public class OrderGrpcService : OrderService.OrderServiceBase
{
    private readonly IOrderService _orderService;
    private readonly ILogger<OrderGrpcService> _logger;

    public OrderGrpcService(IOrderService orderService, ILogger<OrderGrpcService> logger)
    {
        _orderService = orderService;
        _logger = logger;
    }

    public override async Task<OrderResponse> GetOrder(
        GetOrderRequest request,
        ServerCallContext context)
    {
        var order = await _orderService.GetOrderAsync(request.OrderId);

        if (order is null)
            throw new RpcException(new Status(StatusCode.NotFound, $"Order {request.OrderId} not found"));

        return new OrderResponse
        {
            Id = order.Id,
            CustomerName = order.CustomerName,
            TotalAmount = (double)order.TotalAmount,
            CreatedAt = Timestamp.FromDateTime(order.CreatedAt.ToUniversalTime())
        };
    }

    public override async Task<OrderResponse> CreateOrder(
        CreateOrderRequest request,
        ServerCallContext context)
    {
        var order = await _orderService.CreateOrderAsync(new Domain.CreateOrderRequest
        {
            CustomerName = request.CustomerName,
            Items = request.Items.Select(i => new Domain.OrderItem
            {
                ProductId = i.ProductId,
                Quantity = i.Quantity,
                UnitPrice = (decimal)i.UnitPrice
            }).ToList()
        });

        _logger.LogInformation("Order {OrderId} created via gRPC", order.Id);

        return new OrderResponse
        {
            Id = order.Id,
            CustomerName = order.CustomerName,
            TotalAmount = (double)order.TotalAmount,
            CreatedAt = Timestamp.FromDateTime(order.CreatedAt.ToUniversalTime())
        };
    }
}
```

### 3. Monolitten mikroservise geçişte hangi stratejiler kullanılır?
**Cevap:**
Monolith-to-microservices geçişi en riskli migration türlerinden biridir. Başlıca stratejiler: (1) Modüler Monolit: önce monolit içinde bounded context sınırları belirlenir ve modüller arası bağımlılıklar azaltılır; (2) Extract Service: en az bağımlılığa sahip modüller önce servise çıkarılır; (3) Strangler Fig ile servis bazlı extraction; (4) Event-driven decomposition: servisleri birbirine bağlamak için domain event'leri kullanılır. En sık yapılan hata, doğrudan büyük parçalar koparmak yerine küçük, iş değeri taşıyan servislerden başlamamaktır.

**Örnek Kod:**
```csharp
// AŞAMA 1: Monolit içinde bounded context sınırlarını belirle
// Kötü: Her şey birbirine bağlı
public class MonolithicOrderService
{
    private readonly AppDbContext _dbContext;
    private readonly IEmailSender _emailSender;
    private readonly IInventoryChecker _inventoryChecker;
    private readonly IPaymentProcessor _paymentProcessor;
    private readonly IShippingCalculator _shippingCalculator;
    private readonly ILoyaltyPoints _loyaltyPoints;

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        // Tek serviste tüm iş mantığı - sıkı bağımlı
        await _inventoryChecker.CheckAsync(request.Items);
        var payment = await _paymentProcessor.ProcessAsync(request.PaymentInfo);
        var shipping = await _shippingCalculator.CalculateAsync(request.Address);
        var order = new Order { /* ... */ };
        _dbContext.Orders.Add(order);
        await _dbContext.SaveChangesAsync();
        await _loyaltyPoints.AddAsync(request.CustomerId, order.TotalAmount);
        await _emailSender.SendOrderConfirmationAsync(request.CustomerEmail, order);
        return order;
    }
}

// -------------------------------------------------------

// AŞAMA 2: Modüler monolit - bounded context'ler arası iletişim event ile
// Order domain
public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly IDomainEventDispatcher _eventDispatcher;

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        // Yalnızca order domain mantığı burada
        var order = Order.Create(request.CustomerId, request.Items);
        await _repository.SaveAsync(order);

        // Diğer domain'leri domain event ile haberdar et - doğrudan bağımlılık yok
        await _eventDispatcher.DispatchAsync(new OrderCreatedEvent(
            order.Id,
            order.CustomerId,
            order.Items,
            order.TotalAmount));

        return order;
    }
}

// Inventory domain - kendi servisi olarak çıkarılmaya hazır
public class InventoryEventHandler : IEventHandler<OrderCreatedEvent>
{
    private readonly IInventoryRepository _repository;

    public async Task HandleAsync(OrderCreatedEvent @event)
    {
        foreach (var item in @event.Items)
        {
            await _repository.ReserveStockAsync(item.ProductId, item.Quantity);
        }
    }
}

// -------------------------------------------------------

// AŞAMA 3: Inventory domain'i bağımsız servise çıkar
// Yeni Inventory Microservice - ASP.NET Core Web API
[ApiController]
[Route("api/[controller]")]
public class InventoryController : ControllerBase
{
    private readonly IInventoryService _inventoryService;

    public InventoryController(IInventoryService inventoryService)
    {
        _inventoryService = inventoryService;
    }

    [HttpPost("reserve")]
    public async Task<ActionResult> Reserve([FromBody] ReserveStockRequest request)
    {
        var result = await _inventoryService.ReserveAsync(request.ProductId, request.Quantity);
        return result.IsSuccess ? Ok() : Conflict(result.Error);
    }
}

// Order servisi artık HTTP veya message bus ile inventory'e iletişim kurar
public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly IMessageBus _messageBus; // RabbitMQ, Azure Service Bus vb.

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        var order = Order.Create(request.CustomerId, request.Items);
        await _repository.SaveAsync(order);

        // Servise mesaj gönder (asenkron, uygulamayı ayırır)
        await _messageBus.PublishAsync(new OrderCreatedIntegrationEvent
        {
            OrderId = order.Id,
            CustomerId = order.CustomerId,
            Items = order.Items.Select(i => new OrderItemDto
            {
                ProductId = i.ProductId,
                Quantity = i.Quantity
            }).ToList(),
            TotalAmount = order.TotalAmount,
            OccurredAt = DateTime.UtcNow
        });

        return order;
    }
}
```

### 4. .NET Framework'te kullanılan Entity Framework 6'dan EF Core 8'e nasıl geçilir?
**Cevap:**
EF6'dan EF Core'a geçiş breaking change'ler içerir. Namespace değişir (`System.Data.Entity` -> `Microsoft.EntityFrameworkCore`), `ObjectContext` desteklenmez, bazı fluent API yöntemleri farklıdır ve migration komutları değişir. EF Core daha yüksek performans, daha iyi sorgu çevirisi ve NoSQL desteği sunar. Kademeli geçiş için önce sınırlı bir aggregate'i EF Core'a taşıyıp paralel çalıştırmak önerilir.

**Örnek Kod:**
```csharp
// ESKİ: Entity Framework 6
// using System.Data.Entity;

public class LegacyOrderDbContext : DbContext
{
    public LegacyOrderDbContext() : base("OrderConnection") { }

    public DbSet<Order> Orders { get; set; }
    public DbSet<OrderItem> OrderItems { get; set; }

    protected override void OnModelCreating(DbModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Order>()
            .HasKey(o => o.Id)
            .Property(o => o.Id).HasDatabaseGeneratedOption(DatabaseGeneratedOption.Identity);

        modelBuilder.Entity<Order>()
            .HasMany(o => o.Items)
            .WithRequired(i => i.Order)
            .HasForeignKey(i => i.OrderId);

        // EF6 - Timestamp/RowVersion için farklı sözdizimi
        modelBuilder.Entity<Order>()
            .Property(o => o.RowVersion)
            .IsRowVersion();
    }
}

// EF6 sorgu
using (var context = new LegacyOrderDbContext())
{
    var orders = context.Orders
        .Include(o => o.Items)
        .Where(o => o.CustomerId == customerId && !o.IsDeleted)
        .OrderByDescending(o => o.CreatedAt)
        .Take(20)
        .ToList(); // Senkron - EF6'da async desteklenmiyordu (eski sürümler)
}

// -------------------------------------------------------

// YENİ: EF Core 8
// using Microsoft.EntityFrameworkCore;

public class OrderDbContext : DbContext
{
    public OrderDbContext(DbContextOptions<OrderDbContext> options) : base(options) { }

    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Order>(entity =>
        {
            entity.HasKey(o => o.Id);
            entity.Property(o => o.Id).ValueGeneratedOnAdd();

            entity.HasMany(o => o.Items)
                  .WithOne(i => i.Order)
                  .HasForeignKey(i => i.OrderId)
                  .OnDelete(DeleteBehavior.Cascade);

            // EF Core 8 - soft delete için global query filter
            entity.HasQueryFilter(o => !o.IsDeleted);

            // Concurrency token
            entity.Property(o => o.RowVersion).IsRowVersion();

            // EF Core 8 - JSON kolon desteği
            entity.OwnsOne(o => o.ShippingAddress, sa =>
            {
                sa.ToJson();
            });
        });

        modelBuilder.Entity<OrderItem>(entity =>
        {
            entity.HasKey(i => i.Id);
            // EF Core 8 - owned entity ile value object
            entity.OwnsOne(i => i.Money, m =>
            {
                m.Property(x => x.Amount).HasColumnName("UnitPrice").HasPrecision(18, 2);
                m.Property(x => x.Currency).HasColumnName("Currency").HasMaxLength(3);
            });
        });
    }
}

// Program.cs bağımlılık kaydı
builder.Services.AddDbContext<OrderDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("Orders"),
        sqlOptions =>
        {
            sqlOptions.MigrationsAssembly(typeof(OrderDbContext).Assembly.FullName);
            sqlOptions.EnableRetryOnFailure(maxRetryCount: 5);
            sqlOptions.CommandTimeout(30);
        })
    .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTrackingWithIdentityResolution));

// Async sorgu - EF Core native async desteği
public async Task<IReadOnlyList<OrderSummaryDto>> GetRecentOrdersAsync(
    string customerId,
    CancellationToken cancellationToken = default)
{
    return await _context.Orders
        .AsNoTracking()
        .Where(o => o.CustomerId == customerId)
        .OrderByDescending(o => o.CreatedAt)
        .Take(20)
        .Select(o => new OrderSummaryDto(
            o.Id,
            o.OrderNumber,
            o.TotalAmount,
            o.Status,
            o.CreatedAt))
        .ToListAsync(cancellationToken);
}
```

### 5. Veritabanı migration sırasında zero-downtime nasıl sağlanır?
**Cevap:**
Zero-downtime veritabanı değişiklikleri için Expand-Contract pattern kullanılır. Adımlar: (1) Expand — yeni kolon/tablo eklenir, hem eski hem yeni kod çalışabilir; (2) Migrate — mevcut veriler yeni yapıya kopyalanır; (3) Cutover — uygulama yeni yapıyı kullanmaya başlar; (4) Contract — eski yapı kaldırılır. Bu süreç birden fazla deployment gerektirir ama production'da kesinti olmaz.

**Örnek Kod:**
```csharp
// ADIM 1: EXPAND - Yeni kolon ekle (null olabilir, eski kod etkilenmez)
public partial class AddFullNameToCustomers : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // Yeni kolon nullable olarak eklenir
        migrationBuilder.AddColumn<string>(
            name: "FullName",
            table: "Customers",
            type: "nvarchar(200)",
            nullable: true); // Kritik: null olmalı, eski kod bu kolonu bilmiyor

        // İndeks henüz eklenmez, veri doldurulduktan sonra eklenecek
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropColumn(name: "FullName", table: "Customers");
    }
}

// ADIM 2: Uygulama kodunu her iki kolonu yazacak şekilde güncelle
public class CustomerService
{
    private readonly AppDbContext _context;

    public async Task UpdateCustomerAsync(UpdateCustomerRequest request)
    {
        var customer = await _context.Customers.FindAsync(request.Id);
        if (customer is null) throw new NotFoundException(nameof(Customer), request.Id);

        // Hem eski hem yeni kolon yazılıyor (geçiş dönemi)
        customer.FirstName = request.FirstName;
        customer.LastName = request.LastName;
        customer.FullName = $"{request.FirstName} {request.LastName}"; // Yeni kolon

        await _context.SaveChangesAsync();
    }
}

// ADIM 3: Mevcut verileri yeni kolona doldur (backfill migration)
public partial class BackfillFullName : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // Mevcut kayıtları güncelle - büyük tablolarda batch olarak yapılmalı
        migrationBuilder.Sql(@"
            UPDATE Customers
            SET FullName = TRIM(FirstName + ' ' + LastName)
            WHERE FullName IS NULL
              AND (FirstName IS NOT NULL OR LastName IS NOT NULL);
        ");

        // Backfill tamamlandıktan sonra NOT NULL constraint eklenebilir
        migrationBuilder.AlterColumn<string>(
            name: "FullName",
            table: "Customers",
            type: "nvarchar(200)",
            nullable: false,
            defaultValue: "");

        // İndeks eklenir
        migrationBuilder.CreateIndex(
            name: "IX_Customers_FullName",
            table: "Customers",
            column: "FullName");
    }
}

// ADIM 4: Uygulama yalnızca yeni kolonu okumaya başlar
public class CustomerReadService
{
    private readonly AppDbContext _context;

    public async Task<CustomerDto?> GetCustomerAsync(string id)
    {
        return await _context.Customers
            .AsNoTracking()
            .Where(c => c.Id == id)
            .Select(c => new CustomerDto(
                c.Id,
                c.FullName, // Artık yalnızca yeni kolon
                c.Email,
                c.CreatedAt))
            .FirstOrDefaultAsync();
    }
}

// ADIM 5: CONTRACT - Eski kolonları kaldır (ayrı deployment)
public partial class RemoveLegacyNameColumns : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropColumn(name: "FirstName", table: "Customers");
        migrationBuilder.DropColumn(name: "LastName", table: "Customers");
    }
}
```

### 6. Anti-Corruption Layer (ACL) implementasyonu nasıl yapılır?
**Cevap:**
ACL, yeni domain modelini eski sistemin modeline karşı koruyan bir çeviri katmanıdır. Yeni servisin kendi terminolojisini ve modelini kullanmasını sağlarken eski sistemle entegrasyonu sürdürür. Repository veya service adapter olarak implemente edilir. Eski sistem kaldırıldığında yalnızca ACL değiştirilir; domain modeli etkilenmez.

**Örnek Kod:**
```csharp
// Eski sistem domain modeli - kötü tasarlanmış, kurtarılacak
public class LegacyCustomerRecord
{
    public int CustNo { get; set; }
    public string CustName { get; set; } = string.Empty;
    public string Addr1 { get; set; } = string.Empty;
    public string Addr2 { get; set; } = string.Empty;
    public string PostCode { get; set; } = string.Empty;
    public string CountryCode { get; set; } = string.Empty;
    public int StatusCode { get; set; } // 1=Active, 2=Suspended, 3=Closed
    public DateTime RegDate { get; set; }
    public string Email1 { get; set; } = string.Empty;
}

// Yeni domain modeli - temiz, anlamlı
public record Customer(
    CustomerId Id,
    FullName Name,
    Address Address,
    Email PrimaryEmail,
    CustomerStatus Status,
    DateTime RegisteredAt);

public record CustomerId(string Value);
public record FullName(string First, string Last);
public record Address(string Line1, string? Line2, string PostCode, Country Country);
public record Email(string Value);
public enum CustomerStatus { Active, Suspended, Closed }

// ACL: Eski sisteme bağlanan adapter
public interface ICustomerRepository
{
    Task<Customer?> GetByIdAsync(CustomerId id);
    Task<IReadOnlyList<Customer>> GetActiveCustomersAsync();
    Task SaveAsync(Customer customer);
}

// ACL implementasyonu - eski legacy DB'ye erişir, yeni modele çevirir
public class LegacyCustomerRepositoryAdapter : ICustomerRepository
{
    private readonly LegacyDbContext _legacyContext;
    private readonly ILogger<LegacyCustomerRepositoryAdapter> _logger;

    public LegacyCustomerRepositoryAdapter(
        LegacyDbContext legacyContext,
        ILogger<LegacyCustomerRepositoryAdapter> logger)
    {
        _legacyContext = legacyContext;
        _logger = logger;
    }

    public async Task<Customer?> GetByIdAsync(CustomerId id)
    {
        if (!int.TryParse(id.Value, out var legacyId))
        {
            _logger.LogWarning("Invalid legacy customer ID format: {Id}", id.Value);
            return null;
        }

        var legacy = await _legacyContext.Customers
            .AsNoTracking()
            .FirstOrDefaultAsync(c => c.CustNo == legacyId);

        return legacy is null ? null : TranslateToDomain(legacy);
    }

    public async Task<IReadOnlyList<Customer>> GetActiveCustomersAsync()
    {
        var activeRecords = await _legacyContext.Customers
            .AsNoTracking()
            .Where(c => c.StatusCode == 1) // Eski: 1 = Active
            .ToListAsync();

        return activeRecords.Select(TranslateToDomain).ToList();
    }

    public async Task SaveAsync(Customer customer)
    {
        if (!int.TryParse(customer.Id.Value, out var legacyId))
            throw new InvalidOperationException($"Cannot save customer with ID {customer.Id.Value} to legacy system");

        var legacy = await _legacyContext.Customers.FindAsync(legacyId);

        if (legacy is null)
        {
            legacy = new LegacyCustomerRecord { CustNo = legacyId };
            _legacyContext.Customers.Add(legacy);
        }

        // Yeni domain modelinden legacy modeline çeviri
        TranslateFromDomain(customer, legacy);
        await _legacyContext.SaveChangesAsync();
    }

    // Domain -> Legacy çevirisi (ACL'nin asıl işi)
    private static Customer TranslateToDomain(LegacyCustomerRecord legacy)
    {
        // İsim ayrıştırma: "SOYAD, AD" formatından ad ve soyad çıkarılır
        var nameParts = legacy.CustName.Split(',', 2);
        var lastName = nameParts[0].Trim();
        var firstName = nameParts.Length > 1 ? nameParts[1].Trim() : string.Empty;

        // Ülke kodu çevirisi
        var country = legacy.CountryCode switch
        {
            "TR" => Country.Turkey,
            "US" => Country.UnitedStates,
            "GB" => Country.UnitedKingdom,
            _ => Country.Unknown
        };

        // Durum kodu çevirisi
        var status = legacy.StatusCode switch
        {
            1 => CustomerStatus.Active,
            2 => CustomerStatus.Suspended,
            3 => CustomerStatus.Closed,
            _ => CustomerStatus.Active
        };

        return new Customer(
            Id: new CustomerId(legacy.CustNo.ToString()),
            Name: new FullName(firstName, lastName),
            Address: new Address(legacy.Addr1, legacy.Addr2, legacy.PostCode, country),
            PrimaryEmail: new Email(legacy.Email1),
            Status: status,
            RegisteredAt: legacy.RegDate);
    }

    private static void TranslateFromDomain(Customer customer, LegacyCustomerRecord legacy)
    {
        legacy.CustName = $"{customer.Name.Last}, {customer.Name.First}";
        legacy.Addr1 = customer.Address.Line1;
        legacy.Addr2 = customer.Address.Line2 ?? string.Empty;
        legacy.PostCode = customer.Address.PostCode;
        legacy.CountryCode = customer.Address.Country switch
        {
            Country.Turkey => "TR",
            Country.UnitedStates => "US",
            Country.UnitedKingdom => "GB",
            _ => "XX"
        };
        legacy.StatusCode = customer.Status switch
        {
            CustomerStatus.Active => 1,
            CustomerStatus.Suspended => 2,
            CustomerStatus.Closed => 3,
            _ => 1
        };
        legacy.Email1 = customer.PrimaryEmail.Value;
    }
}

public enum Country { Turkey, UnitedStates, UnitedKingdom, Unknown }
```

### 7. .NET Upgrade Assistant nasıl kullanılır ve hangi işlemleri otomatik yapar?
**Cevap:**
.NET Upgrade Assistant, Microsoft'un resmi migration aracıdır. Proje dosyasını SDK stiline dönüştürür, `PackageReference` formatına geçer, eski API kullanımlarını tespit eder ve mümkünse otomatik düzeltmeler önerir. `try-convert` aracıyla proje formatını dönüştürür; API analyzer ile çalışmayan API'leri listeler. Araç tüm sorunları çözmez; platform-specific kod ve üçüncü parti bağımlılıklar manuel inceleme gerektirir.

**Örnek Kod:**
```csharp
// Upgrade Assistant kurulumu ve kullanımı (terminal komutları):
// dotnet tool install -g upgrade-assistant
// upgrade-assistant analyze MyApp.sln
// upgrade-assistant upgrade MyApp.sln --target-tfm-support LTS

// Upgrade Assistant'ın dönüştürdüğü proje dosyası örneği:

// ESKİ: .NET Framework csproj formatı (packages.config kullanan)
// <Project ToolsVersion="15.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
//   <Import Project="$(MSBuildExtensionsPath)\$(MSBuildToolsVersion)\Microsoft.Common.props" />
//   <PropertyGroup>
//     <TargetFrameworkVersion>v4.8</TargetFrameworkVersion>
//   </PropertyGroup>
//   <ItemGroup>
//     <Reference Include="Newtonsoft.Json, Version=13.0.0.0, ...">
//       <HintPath>packages\Newtonsoft.Json.13.0.1\lib\net45\Newtonsoft.Json.dll</HintPath>
//     </Reference>
//   </ItemGroup>
// </Project>

// YENİ: SDK stil csproj formatı (Upgrade Assistant dönüşümü sonrası)
// <Project Sdk="Microsoft.NET.Sdk.Web">
//   <PropertyGroup>
//     <TargetFramework>net8.0</TargetFramework>
//     <Nullable>enable</Nullable>
//     <ImplicitUsings>enable</ImplicitUsings>
//     <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
//   </PropertyGroup>
//   <ItemGroup>
//     <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
//   </ItemGroup>
// </Project>

// API Analyzer - uyumsuz API kullanımını tespit eden attribute
// Bazı .NET Framework API'leri .NET 8'de platform-specific olarak işaretlenmiştir:

// [SupportedOSPlatform("windows")] attribute'u olan API'ler yalnızca Windows'ta çalışır
// RegistryKey, Environment.UserDomainName vb.

#if WINDOWS
[SupportedOSPlatform("windows")]
public class WindowsRegistryService
{
    public string? GetInstallPath()
    {
        using var key = Registry.LocalMachine.OpenSubKey(@"SOFTWARE\MyApp");
        return key?.GetValue("InstallPath") as string;
    }
}
#endif

// Platform bağımsız alternatif
public class ConfigurationService
{
    private readonly IConfiguration _configuration;

    public ConfigurationService(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public string GetInstallPath()
    {
        // appsettings.json veya environment variable'dan oku - platform bağımsız
        return _configuration["App:InstallPath"]
            ?? Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData), "MyApp");
    }
}

// HttpClient migration: WebClient/HttpWebRequest -> HttpClient
// ESKİ: WebClient (obsolete)
public string FetchDataLegacy(string url)
{
    using var client = new WebClient();
    return client.DownloadString(url); // Senkron, thread-blocking
}

// YENİ: IHttpClientFactory ile typed HttpClient
public class ExternalApiClient
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<ExternalApiClient> _logger;

    public ExternalApiClient(HttpClient httpClient, ILogger<ExternalApiClient> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }

    public async Task<string> FetchDataAsync(string endpoint, CancellationToken ct = default)
    {
        try
        {
            var response = await _httpClient.GetAsync(endpoint, ct);
            response.EnsureSuccessStatusCode();
            return await response.Content.ReadAsStringAsync(ct);
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "Failed to fetch data from {Endpoint}", endpoint);
            throw;
        }
    }
}

// Program.cs - typed HttpClient kaydı
builder.Services.AddHttpClient<ExternalApiClient>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["ExternalApi:BaseUrl"]!);
    client.DefaultRequestHeaders.Add("User-Agent", "MyApp/1.0");
    client.Timeout = TimeSpan.FromSeconds(30);
})
.AddPolicyHandler(GetRetryPolicy())
.AddPolicyHandler(GetCircuitBreakerPolicy());

static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
    => HttpPolicyExtensions
        .HandleTransientHttpError()
        .WaitAndRetryAsync(3, retryAttempt =>
            TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));

static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
    => HttpPolicyExtensions
        .HandleTransientHttpError()
        .CircuitBreakerAsync(5, TimeSpan.FromSeconds(30));
```

## Best Practices

### 1. **Migration Öncesi Hazırlık**
- Mevcut sistemin tam test kapsamını oluşturun (golden master testleri)
- Bağımlılık grafiğini belgelendirin ve döngüsel bağımlılıkları kırın
- Tüm üçüncü parti paketlerin .NET 8 desteğini doğrulayın
- Platform-specific kodların envanterini çıkarın
- Performans ve yük testleri için baseline metrikler toplayın

### 2. **Aşamalı Migration Yönetimi**
- Her migration adımı tek bir şeyi değiştirmelidir
- CI/CD pipeline her adımda testleri çalıştırmalıdır
- Feature branch stratejisiyle uzun süreli migration kollarından kaçının
- Database migration'ları uygulama deployment'ından ayrı yönetin
- Her başarılı adımı takım ile kutlayın; motivasyonu koruyun

### 3. **Kalite Güvencesi**
- Test kapsamını migration sürecinde %80 altına düşürmeyin
- Mutation testing ile test kalitesini doğrulayın
- Contract testlerini otomatize edin
- Performance regression testlerini CI'a ekleyin
- Static analysis araçlarını (Roslyn analyzers, SonarQube) zorunlu hale getirin

### 4. **Ekip ve Organizasyon**
- Migration çalışmalarını iş özellikleriyle birleştirerek "yenilik yaparken taşıyın"
- Knowledge siloları oluşturmayın; çiftli programlama ve kod incelemeleri yapın
- Migration kararlarını Architecture Decision Record (ADR) ile belgeleyin
- Paydaşlara düzenli ilerleme raporları sunun
- Teknik borç azaltımını sprint velocity ile ölçün

## Kaynaklar

- [Microsoft .NET Upgrade Assistant](https://dotnet.microsoft.com/en-us/platform/upgrade-assistant)
- [Porting from .NET Framework to .NET](https://docs.microsoft.com/en-us/dotnet/core/porting/)
- [Anti-Corruption Layer Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/anti-corruption-layer)
- [Expand Contract Pattern](https://martinfowler.com/bliki/ParallelChange.html)
- [Monolith to Microservices - Sam Newman](https://samnewman.io/books/monolith-to-microservices/)
- [EF Core Migration from EF6](https://docs.microsoft.com/en-us/ef/efcore-and-ef6/porting/)
- [CoreWCF - WCF on .NET Core](https://github.com/CoreWCF/CoreWCF)
- [YARP Reverse Proxy](https://microsoft.github.io/reverse-proxy/)
