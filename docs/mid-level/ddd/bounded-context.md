# Bounded Context

## Genel Bakış

Bounded Context (Sınırlı Bağlam), Domain-Driven Design'ın en kritik stratejik kavramlarından biridir. Büyük ve karmaşık sistemlerde aynı kavramlar (örn. "Müşteri", "Ürün") farklı alt sistemlerde farklı anlamlar taşıyabilir. Bounded Context, belirli bir domain modelinin geçerli olduğu açıkça tanımlanmış sınırdır. Bu sınır içinde Ubiquitous Language tutarlıdır ve model bütünlüğü korunur. Microservices mimarisinde her servis genellikle bir Bounded Context'e karşılık gelir.

## Context Mapping

Context Mapping, farklı Bounded Context'ler arasındaki ilişkileri ve entegrasyon şekillerini tanımlar. Aşağıdaki stratejiler kullanılır:

### İlişki Türleri

| Strateji | Açıklama |
|----------|----------|
| **Shared Kernel** | İki context aynı model parçasını paylaşır, birlikte değiştirilir |
| **Customer/Supplier** | Supplier (upstream) context, Customer (downstream) context'e hizmet verir |
| **Conformist** | Downstream, upstream'in modelini olduğu gibi benimser |
| **Anti-Corruption Layer** | Downstream, upstream'den gelen modeli kendi modeline dönüştürür |
| **Open Host Service** | Upstream, iyi tanımlanmış bir protokol/API sunar |
| **Published Language** | Ortak, iyi belgelenmiş bir dil/format kullanılır |
| **Separate Ways** | Context'ler entegrasyon kurmadan bağımsız çalışır |
| **Big Ball of Mud** | Sınır belirsiz, karmaşık eski sistemler |

## C# Implementasyonu

### Bounded Context Tanımlama

Bir e-ticaret sisteminde birden fazla Bounded Context bulunabilir:

```csharp
// =============================================
// SATIŞ BOUNDED CONTEXT (Sales Context)
// =============================================
namespace ECommerce.Sales.Domain
{
    // Sales Context'teki "Customer" modeli:
    // Satın alma geçmişi, sadakat puanları, tercihler odaklı
    public class Customer : AggregateRoot
    {
        public string Email { get; private set; }
        public string FullName { get; private set; }
        public int LoyaltyPoints { get; private set; }
        public CustomerTier Tier { get; private set; }
        private readonly List<OrderSummary> _orderHistory = new();

        public IReadOnlyList<OrderSummary> OrderHistory => _orderHistory.AsReadOnly();

        protected Customer() { }

        public static Customer Register(string email, string fullName)
        {
            var customer = new Customer
            {
                Id = Guid.NewGuid(),
                Email = email,
                FullName = fullName,
                LoyaltyPoints = 0,
                Tier = CustomerTier.Standard
            };
            customer.AddDomainEvent(new CustomerRegisteredEvent(customer.Id, email));
            return customer;
        }

        public void AddLoyaltyPoints(int points)
        {
            LoyaltyPoints += points;
            UpdateTier();
        }

        public decimal GetDiscountRate()
        {
            return Tier switch
            {
                CustomerTier.Standard => 0m,
                CustomerTier.Silver => 0.05m,
                CustomerTier.Gold => 0.10m,
                CustomerTier.Platinum => 0.15m,
                _ => 0m
            };
        }

        private void UpdateTier()
        {
            Tier = LoyaltyPoints switch
            {
                < 1000 => CustomerTier.Standard,
                < 5000 => CustomerTier.Silver,
                < 15000 => CustomerTier.Gold,
                _ => CustomerTier.Platinum
            };
        }
    }

    public enum CustomerTier { Standard, Silver, Gold, Platinum }
}

// =============================================
// DESTEK BOUNDED CONTEXT (Support Context)
// =============================================
namespace ECommerce.Support.Domain
{
    // Support Context'teki "Customer" modeli:
    // İletişim tercihler, bilet geçmişi, SLA bilgileri odaklı
    public class Customer : AggregateRoot
    {
        public string Email { get; private set; }
        public string FullName { get; private set; }
        public string PreferredContactMethod { get; private set; }
        public SupportTier SupportTier { get; private set; }
        private readonly List<SupportTicket> _tickets = new();

        public IReadOnlyList<SupportTicket> OpenTickets =>
            _tickets.Where(t => t.IsOpen).ToList().AsReadOnly();

        protected Customer() { }

        public SupportTicket OpenTicket(string subject, string description, Priority priority)
        {
            var ticket = SupportTicket.Create(Id, subject, description, priority, SupportTier);
            _tickets.Add(ticket);
            AddDomainEvent(new TicketOpenedEvent(Id, ticket.Id));
            return ticket;
        }

        public int GetOpenTicketCount() => _tickets.Count(t => t.IsOpen);
    }

    public enum SupportTier { Basic, Premium, Enterprise }
    public enum Priority { Low, Medium, High, Critical }
}

// =============================================
// MUHASEBE BOUNDED CONTEXT (Accounting Context)
// =============================================
namespace ECommerce.Accounting.Domain
{
    // Accounting Context'teki "Customer" modeli:
    // Fatura, ödeme, vergi bilgileri odaklı
    public class Customer : AggregateRoot
    {
        public string TaxNumber { get; private set; }
        public string CompanyName { get; private set; }
        public Address BillingAddress { get; private set; }
        public CreditLimit CreditLimit { get; private set; }
        public decimal OutstandingBalance { get; private set; }

        protected Customer() { }

        public bool CanPlaceOrder(decimal orderAmount)
        {
            return OutstandingBalance + orderAmount <= CreditLimit.Amount;
        }

        public void RecordPayment(decimal amount, string referenceNumber)
        {
            if (amount <= 0)
                throw new DomainException("Ödeme tutarı pozitif olmalıdır.");

            OutstandingBalance = Math.Max(0, OutstandingBalance - amount);
            AddDomainEvent(new PaymentRecordedEvent(Id, amount, referenceNumber));
        }
    }
}
```

### Anti-Corruption Layer (ACL)

ACL, bir Bounded Context'in başka bir context'in modeliyle doğrudan kirlenmesini engeller. Upstream'den gelen veriyi kendi domain modeline çevirir.

```csharp
// Dış sistem (eski CRM) modeli — kontrol edemeyiz
namespace LegacyCRM
{
    public class CrmCustomerDto
    {
        public int CustId { get; set; }
        public string CustCode { get; set; }
        public string FirstName { get; set; }
        public string LastName { get; set; }
        public string EmailAddr { get; set; }
        public int CustType { get; set; }   // 1=Bireysel, 2=Kurumsal, 3=VIP
        public string TaxNo { get; set; }
        public bool IsActive { get; set; }
        public DateTime RegisterDate { get; set; }
        public decimal CreditLmt { get; set; }
    }
}

// ---- ACL: Sales Context ----
namespace ECommerce.Sales.Infrastructure.AntiCorruption
{
    // ACL Interface: Sales Context'in ihtiyaç duyduğu sözleşme
    public interface ICustomerTranslator
    {
        Sales.Domain.Customer Translate(LegacyCRM.CrmCustomerDto crmCustomer);
    }

    // ACL Implementasyonu: CRM modelini Sales domain modeline dönüştürür
    public class CrmToSalesCustomerTranslator : ICustomerTranslator
    {
        public Sales.Domain.Customer Translate(LegacyCRM.CrmCustomerDto crmCustomer)
        {
            ArgumentNullException.ThrowIfNull(crmCustomer);

            if (!crmCustomer.IsActive)
                throw new DomainException($"Pasif müşteri aktarılamaz: {crmCustomer.CustCode}");

            var fullName = $"{crmCustomer.FirstName} {crmCustomer.LastName}".Trim();
            var tier = MapCustomerType(crmCustomer.CustType);

            return Sales.Domain.Customer.Reconstruct(
                id: Guid.NewGuid(),
                email: crmCustomer.EmailAddr,
                fullName: fullName,
                tier: tier,
                loyaltyPoints: 0);
        }

        private static Sales.Domain.CustomerTier MapCustomerType(int custType)
        {
            return custType switch
            {
                1 => Sales.Domain.CustomerTier.Standard,
                2 => Sales.Domain.CustomerTier.Silver,
                3 => Sales.Domain.CustomerTier.Platinum,
                _ => Sales.Domain.CustomerTier.Standard
            };
        }
    }

    // ACL için Adapter Service
    public class CrmCustomerAdapter
    {
        private readonly ICrmApiClient _crmApiClient;
        private readonly ICustomerTranslator _translator;
        private readonly ILogger<CrmCustomerAdapter> _logger;

        public CrmCustomerAdapter(
            ICrmApiClient crmApiClient,
            ICustomerTranslator translator,
            ILogger<CrmCustomerAdapter> logger)
        {
            _crmApiClient = crmApiClient;
            _translator = translator;
            _logger = logger;
        }

        public async Task<Sales.Domain.Customer?> FindCustomerByEmailAsync(string email)
        {
            try
            {
                // Dış sisteme istek
                var crmCustomer = await _crmApiClient.FindByEmailAsync(email);
                if (crmCustomer is null) return null;

                // ACL çevirisi: dış model → domain modeli
                return _translator.Translate(crmCustomer);
            }
            catch (CrmApiException ex)
            {
                _logger.LogError(ex, "CRM API hatası: {Email}", email);
                // Dış sistemin exception'ını domain exception'a çevir
                throw new DomainException($"Müşteri bilgisi alınamadı: {email}", ex);
            }
        }
    }
}

// ---- ACL: Integration Events aracılığıyla Context'ler arası iletişim ----
namespace ECommerce.Sales.Infrastructure.AntiCorruption
{
    // Support Context'ten gelen event (farklı bir context'in modeli)
    public record SupportTicketClosedIntegrationEvent(
        Guid SupportCustomerId,
        Guid TicketId,
        string Resolution,
        DateTime ClosedAt);

    // ACL: Support event'ini Sales domain event'ine çeviren handler
    public class SupportTicketClosedEventHandler
        : IIntegrationEventHandler<SupportTicketClosedIntegrationEvent>
    {
        private readonly ICustomerRepository _customerRepository;
        private readonly IUnitOfWork _unitOfWork;

        public SupportTicketClosedEventHandler(
            ICustomerRepository customerRepository,
            IUnitOfWork unitOfWork)
        {
            _customerRepository = customerRepository;
            _unitOfWork = unitOfWork;
        }

        public async Task Handle(SupportTicketClosedIntegrationEvent integrationEvent)
        {
            // Support Context'in customer ID'si ile Sales Context'teki customer'ı bul
            // (ID mapping gerekebilir — bir context'in ID'si diğerinden farklı olabilir)
            var customer = await _customerRepository
                .GetByExternalSupportIdAsync(integrationEvent.SupportCustomerId);

            if (customer is null) return;

            // Support context'teki olayı Sales context'e anlamlı şekilde çevir:
            // Ticket kapandı → Müşteri memnuniyeti puanı ekle
            customer.AddLoyaltyPoints(50);
            _customerRepository.Update(customer);
            await _unitOfWork.SaveChangesAsync();
        }
    }
}
```

### Open Host Service ve Published Language

```csharp
// =============================================
// ÜRÜN KATALOĞU BOUNDED CONTEXT — Open Host Service
// =============================================
namespace ECommerce.Catalog.Api
{
    // Published Language: Diğer context'lerin kullanacağı DTO şeması
    // Bu DTO, Catalog Context'in resmi "yayınlanmış dili"dir
    public record ProductPublicDto(
        Guid ProductId,
        string Sku,
        string Name,
        string Description,
        decimal Price,
        string Currency,
        int StockQuantity,
        string Category,
        bool IsAvailable);

    // Open Host Service: Catalog context'in dışarıya sunduğu API
    [ApiController]
    [Route("api/v1/products")]
    public class ProductsController : ControllerBase
    {
        private readonly IProductQueryService _queryService;

        public ProductsController(IProductQueryService queryService)
        {
            _queryService = queryService;
        }

        [HttpGet("{id}")]
        [ProducesResponseType(typeof(ProductPublicDto), StatusCodes.Status200OK)]
        [ProducesResponseType(StatusCodes.Status404NotFound)]
        public async Task<IActionResult> GetProduct(Guid id)
        {
            var product = await _queryService.GetProductByIdAsync(id);
            if (product is null) return NotFound();

            // Domain model → Published Language dönüşümü
            var dto = MapToPublicDto(product);
            return Ok(dto);
        }

        [HttpGet("search")]
        public async Task<IActionResult> SearchProducts(
            [FromQuery] string? category,
            [FromQuery] decimal? minPrice,
            [FromQuery] decimal? maxPrice,
            [FromQuery] bool onlyAvailable = true)
        {
            var products = await _queryService.SearchAsync(category, minPrice, maxPrice, onlyAvailable);
            return Ok(products.Select(MapToPublicDto));
        }

        private static ProductPublicDto MapToPublicDto(Product product)
        {
            return new ProductPublicDto(
                ProductId: product.Id,
                Sku: product.Sku.Value,
                Name: product.Name,
                Description: product.Description,
                Price: product.Price.Amount,
                Currency: product.Price.Currency,
                StockQuantity: product.Stock.Quantity,
                Category: product.Category.Name,
                IsAvailable: product.IsAvailable);
        }
    }
}

// Sales Context, Catalog'un Open Host Service'ini tüketir
namespace ECommerce.Sales.Infrastructure.ExternalServices
{
    public interface IProductCatalogService
    {
        Task<ProductInfo?> GetProductAsync(Guid productId);
    }

    // Sales Context'in kendi ihtiyaçlarına göre şekillendirilmiş model
    public record ProductInfo(Guid ProductId, string Name, Money Price, bool IsAvailable);

    // ACL: Catalog'un Public DTO'sunu Sales domain modeline çevirir
    public class CatalogProductService : IProductCatalogService
    {
        private readonly HttpClient _httpClient;
        private readonly ILogger<CatalogProductService> _logger;

        public CatalogProductService(HttpClient httpClient, ILogger<CatalogProductService> logger)
        {
            _httpClient = httpClient;
            _logger = logger;
        }

        public async Task<ProductInfo?> GetProductAsync(Guid productId)
        {
            try
            {
                // Catalog'un Open Host Service'ine HTTP isteği
                var response = await _httpClient.GetFromJsonAsync<CatalogProductDto>(
                    $"api/v1/products/{productId}");

                if (response is null) return null;

                // Published Language → Sales domain model dönüşümü
                return new ProductInfo(
                    ProductId: response.ProductId,
                    Name: response.Name,
                    Price: new Money(response.Price, response.Currency),
                    IsAvailable: response.IsAvailable);
            }
            catch (HttpRequestException ex)
            {
                _logger.LogError(ex, "Catalog servisi hatası: {ProductId}", productId);
                throw new ExternalServiceException("Ürün bilgisi alınamadı.", ex);
            }
        }

        // Catalog servisinin döndürdüğü DTO (Published Language)
        private record CatalogProductDto(
            Guid ProductId, string Sku, string Name, string Description,
            decimal Price, string Currency, int StockQuantity,
            string Category, bool IsAvailable);
    }
}
```

## Mülakat Soruları

### 1. Soru: Bounded Context nedir ve neden önemlidir?

**Cevap:**

Bounded Context, belirli bir domain modelinin ve Ubiquitous Language'ın geçerli olduğu açıkça tanımlanmış sınırdır. Önem nedenleri:

- **Model bütünlüğü**: Aynı kavramın farklı alt sistemlerde farklı anlamı olabilir; her context kendi modelini korur
- **Ölçeklenebilirlik**: Büyük sistemleri bağımsız geliştirilebilir parçalara böler
- **Ekip bağımsızlığı**: Farklı takımlar farklı context'ler üzerinde bağımsız çalışabilir
- **Microservices temeli**: Her context, potansiyel olarak bağımsız bir servis olabilir

**Örnek:** Bir e-ticaret sisteminde "Ürün":
- **Catalog Context**: Açıklama, resim, kategori, özellikler
- **Inventory Context**: Stok miktarı, depo konumu, yenileme eşiği
- **Pricing Context**: Fiyat, indirim kuralları, kampanyalar
- **Shipping Context**: Ağırlık, boyut, kargo kısıtlamaları

**Örnek Kod:**

```csharp
// Her context kendi "Product" tanımına sahip
namespace Catalog.Domain { public class Product { public string Name; public string Description; /* ... */ } }
namespace Inventory.Domain { public class Product { public int StockQuantity; public string WarehouseLocation; /* ... */ } }
namespace Pricing.Domain { public class Product { public Money BasePrice; public List<DiscountRule> Discounts; /* ... */ } }
```

---

### 2. Soru: Anti-Corruption Layer ne zaman kullanılır?

**Cevap:**

ACL şu durumlarda kullanılır:

1. **Eski sistemlerle entegrasyon**: Legacy sistemin modeli domain'inizi "kirletmemesi" gerektiğinde
2. **Farklı domain modelleri**: İki context'in kavramları örtüşmüyor veya farklı anlam taşıyorsa
3. **Upstream değişim riski**: Dış sistemin değişmesinden etkilenmek istemiyorsanız
4. **Conformist olmak istemiyorsanız**: Upstream'in modelini olduğu gibi benimsemek yerine kendi modelinizi korumak istiyorsanız

**Örnek Kod:**

```csharp
// Dış ödeme sistemi (Stripe) modeli
public class StripePaymentIntent
{
    public string Id { get; set; }
    public long Amount { get; set; }         // Kuruş cinsinden (100 = 1 TRY)
    public string Currency { get; set; }     // "try" (lowercase)
    public string Status { get; set; }       // "succeeded", "requires_payment_method", vb.
    public string CustomerId { get; set; }   // Stripe customer ID
}

// ACL: Stripe modelini domain modeline çevirir
public class StripePaymentTranslator
{
    public Payment Translate(StripePaymentIntent stripeIntent)
    {
        var amount = new Money(stripeIntent.Amount / 100m, stripeIntent.Currency.ToUpper());
        var status = stripeIntent.Status switch
        {
            "succeeded" => PaymentStatus.Completed,
            "requires_payment_method" => PaymentStatus.Pending,
            "canceled" => PaymentStatus.Cancelled,
            _ => PaymentStatus.Failed
        };

        return new Payment(
            externalId: stripeIntent.Id,
            amount: amount,
            status: status);
    }
}
```

---

### 3. Soru: Context Mapping stratejileri nelerdir? Customer/Supplier ve Conformist arasındaki fark nedir?

**Cevap:**

**Customer/Supplier**: Upstream (Supplier) context, Downstream (Customer) context'in ihtiyaçlarını dinler ve buna göre API'sini şekillendirir. İki taraf arasında müzakere vardır.

**Conformist**: Upstream context, downstream'in ihtiyaçlarını dikkate almaz. Downstream, upstream'in modelini olduğu gibi benimser. Büyük, değiştirilemez sistemlerde (örn. 3rd party API) karşılaşılır.

**Örnek Kod:**

```csharp
// Customer/Supplier: Supplier'ın sağladığı, Customer'ın şekillendirmeye katkı sunduğu API
namespace InventoryContext.Api
{
    // Customer (Sales) Context'in ihtiyaçlarına göre tasarlanmış endpoint
    [HttpGet("stock-check")]
    public async Task<StockCheckResult> CheckStock(
        [FromQuery] Guid[] productIds)  // Sales'in istediği toplu kontrol
    {
        var results = await _stockService.CheckMultipleAsync(productIds);
        return new StockCheckResult(results);
    }
}

// Conformist: 3rd party vergi servisi — modeli değiştiremeyiz, olduğu gibi kullanırız
public class TaxCalculationConformist
{
    private readonly ITaxApiClient _taxApiClient;

    public async Task<decimal> CalculateTax(Order order)
    {
        // 3rd party API'nin beklediği formata uyuyoruz (Conformist)
        var request = new TaxApiRequest
        {
            TotalAmount = (double)order.TotalAmount.Amount,
            CountryCode = "TR",
            TaxNumber = order.CustomerTaxNumber
        };

        var response = await _taxApiClient.CalculateAsync(request);
        return (decimal)response.TaxAmount;
    }
}
```

---

### 4. Soru: Bounded Context'ler arası iletişim nasıl sağlanır?

**Cevap:**

İki temel yöntem vardır:

**Senkron (Sync)**: REST API veya gRPC ile doğrudan çağrı. Basit ama sıkı bağımlılık yaratır.

**Asenkron (Async)**: Message broker (RabbitMQ, Kafka) üzerinden Integration Events. Loosely coupled, daha dayanıklı.

**Örnek Kod:**

```csharp
// Asenkron iletişim: Integration Events aracılığıyla
// Sales Context bir event yayınlar
public class OrderConfirmedIntegrationEvent
{
    public Guid OrderId { get; init; }
    public Guid CustomerId { get; init; }
    public List<OrderItemDto> Items { get; init; }
    public decimal TotalAmount { get; init; }
    public DateTime ConfirmedAt { get; init; }
}

// Sales Context'te event yayınlama
public class OrderConfirmedDomainEventHandler
    : INotificationHandler<OrderConfirmedEvent>
{
    private readonly IIntegrationEventBus _eventBus;

    public OrderConfirmedDomainEventHandler(IIntegrationEventBus eventBus)
    {
        _eventBus = eventBus;
    }

    public async Task Handle(OrderConfirmedEvent notification, CancellationToken ct)
    {
        // Domain event → Integration event çevirisi
        var integrationEvent = new OrderConfirmedIntegrationEvent
        {
            OrderId = notification.OrderId,
            CustomerId = notification.CustomerId,
            Items = notification.Items.Select(i => new OrderItemDto(i.ProductId, i.Quantity)).ToList(),
            TotalAmount = notification.TotalAmount.Amount,
            ConfirmedAt = notification.ConfirmedAt
        };

        await _eventBus.PublishAsync(integrationEvent);
    }
}

// Inventory Context event'i dinler ve stok düşer
public class OrderConfirmedIntegrationEventHandler
    : IIntegrationEventHandler<OrderConfirmedIntegrationEvent>
{
    private readonly IInventoryService _inventoryService;

    public OrderConfirmedIntegrationEventHandler(IInventoryService inventoryService)
    {
        _inventoryService = inventoryService;
    }

    public async Task Handle(OrderConfirmedIntegrationEvent integrationEvent)
    {
        foreach (var item in integrationEvent.Items)
        {
            await _inventoryService.ReserveStockAsync(item.ProductId, item.Quantity);
        }
    }
}
```

---

### 5. Soru: Shared Kernel ne zaman tercih edilir, ne zaman kaçınılmalıdır?

**Cevap:**

**Shared Kernel kullanma durumları:**
- İki context sık değiştirilen ortak bir modeli paylaşıyorsa
- Her iki context'i geliştiren ekip küçük ve koordinasyonu kolaysa
- Kod tekrarının maliyeti entegrasyon maliyetinden yüksekse

**Shared Kernel'den kaçınma durumları:**
- Context'ler farklı ekipler tarafından geliştiriliyorsa
- Paylaşılan model sık değişiyorsa (yüksek bağımlılık riski)
- Context'ler farklı hızda evriliyorsa

**Örnek Kod:**

```csharp
// Shared Kernel: Hem Sales hem Accounting'in paylaştığı temel tipler
// Bu kütüphane, her iki context'in referans aldığı ortak kütüphanedir
namespace ECommerce.SharedKernel
{
    // Para birimi — her context için aynı tanım
    public sealed class Money : ValueObject
    {
        public decimal Amount { get; }
        public string Currency { get; }
        public Money(decimal amount, string currency) { Amount = amount; Currency = currency; }
        protected override IEnumerable<object> GetEqualityComponents()
        { yield return Amount; yield return Currency; }
    }

    // Ortak base class'lar
    public abstract class Entity { /* ... */ }
    public abstract class AggregateRoot : Entity { /* ... */ }
    public abstract class ValueObject { /* ... */ }
    public interface IDomainEvent { /* ... */ }

    // DİKKAT: Shared Kernel mümkün olduğunca minimal tutulmalı
    // Business logic eklenmemeli — sadece altyapısal tipler
}
```

---

### 6. Soru: Microservices ile Bounded Context ilişkisi nedir?

**Cevap:**

Her Bounded Context, bir microservice için iyi bir aday olmakla birlikte bire bir eşleme zorunlu değildir:

- **1 Context = 1 Microservice**: En yaygın başlangıç noktası, net sorumluluk sınırları
- **1 Context = N Microservice**: Bir context içinde teknik ölçekleme gerektiğinde (örn. okuma ve yazma servisleri)
- **N Context = 1 Microservice**: Context'ler çok küçükse başlangıçta birleştirilebilir (modular monolith), sonra ayrılır

**Örnek Kod:**

```csharp
// Modüler Monolith: Bounded Context'ler ayrı projeler ama tek process
// ECommerce.sln
// ├── ECommerce.Sales.Domain
// ├── ECommerce.Sales.Application
// ├── ECommerce.Sales.Infrastructure
// ├── ECommerce.Inventory.Domain
// ├── ECommerce.Inventory.Application
// ├── ECommerce.Inventory.Infrastructure
// └── ECommerce.WebApi  ← Tek uygulama girişi

// Program.cs — Modüler monolith'te context'leri kaydet
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddSalesContext(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<SalesDbContext>(opts =>
            opts.UseNpgsql(configuration.GetConnectionString("SalesDb")));

        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<ICustomerRepository, CustomerRepository>();
        services.AddMediatR(typeof(Sales.Application.AssemblyMarker));
        return services;
    }

    public static IServiceCollection AddInventoryContext(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<InventoryDbContext>(opts =>
            opts.UseNpgsql(configuration.GetConnectionString("InventoryDb")));

        services.AddScoped<IStockRepository, StockRepository>();
        services.AddMediatR(typeof(Inventory.Application.AssemblyMarker));
        return services;
    }
}
```
