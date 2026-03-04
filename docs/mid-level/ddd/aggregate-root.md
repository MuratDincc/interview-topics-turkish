# Aggregate Root

## Genel Bakış

Aggregate Root, Domain-Driven Design'ın en temel yapı taşlarından biridir. Aggregate, birbiriyle ilişkili ve birlikte değişmesi gereken nesneler kümesidir; Aggregate Root ise bu kümenin dışarıya açılan tek giriş noktasıdır. Dış dünya yalnızca Aggregate Root ile etkileşime girer, iç nesnelere (child entity, value object) doğrudan erişemez. Bu yaklaşım, veri tutarlılığını (consistency) garanti eder ve iş kurallarının tek bir yerde uygulanmasını sağlar.

## Entity ve Value Object

### Entity

Entity, benzersiz bir kimliğe (identity) sahip olan ve yaşam döngüsü boyunca bu kimlikle takip edilen nesnedir. İki Entity'nin tüm özellikleri aynı olsa bile kimlikleri farklıysa eşit sayılmazlar.

**Entity özellikleri:**
- Benzersiz ID'ye sahiptir
- Zaman içinde değişebilir (mutable)
- Kimliği ile karşılaştırılır
- Yaşam döngüsü yönetilir (create, update, delete)

```csharp
public abstract class Entity
{
    public Guid Id { get; protected set; }

    protected Entity()
    {
        Id = Guid.NewGuid();
    }

    protected Entity(Guid id)
    {
        Id = id;
    }

    public override bool Equals(object? obj)
    {
        if (obj is not Entity other)
            return false;

        if (ReferenceEquals(this, other))
            return true;

        if (GetType() != other.GetType())
            return false;

        return Id == other.Id;
    }

    public override int GetHashCode()
    {
        return Id.GetHashCode();
    }

    public static bool operator ==(Entity? left, Entity? right)
    {
        if (left is null && right is null)
            return true;

        if (left is null || right is null)
            return false;

        return left.Equals(right);
    }

    public static bool operator !=(Entity? left, Entity? right)
    {
        return !(left == right);
    }
}
```

### Value Object

Value Object, kimliği olmayan ve değerleriyle tanımlanan değişmez (immutable) nesnedir. İki Value Object'in tüm değerleri aynıysa eşittirler. Örneğin `Money(100, "TRY")` ile başka bir `Money(100, "TRY")` eşdeğerdir.

**Value Object özellikleri:**
- Kimliği yoktur
- Değişmezdir (immutable) — değiştirmek yerine yeni nesne oluşturulur
- Değerleri ile karşılaştırılır
- Side effect içermez

```csharp
public abstract class ValueObject
{
    protected abstract IEnumerable<object> GetEqualityComponents();

    public override bool Equals(object? obj)
    {
        if (obj is null || obj.GetType() != GetType())
            return false;

        var other = (ValueObject)obj;
        return GetEqualityComponents().SequenceEqual(other.GetEqualityComponents());
    }

    public override int GetHashCode()
    {
        return GetEqualityComponents()
            .Select(x => x?.GetHashCode() ?? 0)
            .Aggregate((x, y) => x ^ y);
    }

    public static bool operator ==(ValueObject? left, ValueObject? right)
    {
        if (left is null && right is null)
            return true;

        if (left is null || right is null)
            return false;

        return left.Equals(right);
    }

    public static bool operator !=(ValueObject? left, ValueObject? right)
    {
        return !(left == right);
    }
}

// Örnek Value Object: Para
public sealed class Money : ValueObject
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        if (amount < 0)
            throw new ArgumentException("Tutar negatif olamaz.", nameof(amount));

        if (string.IsNullOrWhiteSpace(currency))
            throw new ArgumentException("Para birimi boş olamaz.", nameof(currency));

        Amount = amount;
        Currency = currency.ToUpperInvariant();
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException($"Farklı para birimleri toplanamaz: {Currency} ve {other.Currency}");

        return new Money(Amount + other.Amount, Currency);
    }

    public Money Subtract(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException($"Farklı para birimleri çıkarılamaz: {Currency} ve {other.Currency}");

        if (Amount < other.Amount)
            throw new InvalidOperationException("Sonuç negatif olamaz.");

        return new Money(Amount - other.Amount, Currency);
    }

    public Money Multiply(decimal multiplier)
    {
        return new Money(Amount * multiplier, Currency);
    }

    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return Amount;
        yield return Currency;
    }

    public override string ToString() => $"{Amount} {Currency}";
}

// Örnek Value Object: Adres
public sealed class Address : ValueObject
{
    public string Street { get; }
    public string City { get; }
    public string PostalCode { get; }
    public string Country { get; }

    public Address(string street, string city, string postalCode, string country)
    {
        Street = street ?? throw new ArgumentNullException(nameof(street));
        City = city ?? throw new ArgumentNullException(nameof(city));
        PostalCode = postalCode ?? throw new ArgumentNullException(nameof(postalCode));
        Country = country ?? throw new ArgumentNullException(nameof(country));
    }

    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return Street;
        yield return City;
        yield return PostalCode;
        yield return Country;
    }
}
```

## Aggregate Root Implementasyonu

Aggregate Root, hem Entity'den türer hem de iç nesnelerin tutarlılığından sorumludur. Tüm iş kuralları Aggregate Root üzerinde uygulanır.

```csharp
public abstract class AggregateRoot : Entity
{
    private readonly List<IDomainEvent> _domainEvents = new();
    public IReadOnlyList<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected AggregateRoot() { }
    protected AggregateRoot(Guid id) : base(id) { }

    protected void AddDomainEvent(IDomainEvent domainEvent)
    {
        _domainEvents.Add(domainEvent);
    }

    public void ClearDomainEvents()
    {
        _domainEvents.Clear();
    }
}

// Sipariş Aggregate Root örneği
public class Order : AggregateRoot
{
    private readonly List<OrderItem> _items = new();

    public string OrderNumber { get; private set; }
    public Guid CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Address ShippingAddress { get; private set; }
    public Money TotalAmount { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? ConfirmedAt { get; private set; }
    public DateTime? ShippedAt { get; private set; }

    // Dışarıya salt okunur koleksiyon sunulur
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();

    // EF Core için protected constructor
    protected Order() { }

    // Factory method aracılığıyla oluşturma
    private Order(Guid id, string orderNumber, Guid customerId, Address shippingAddress)
        : base(id)
    {
        OrderNumber = orderNumber;
        CustomerId = customerId;
        ShippingAddress = shippingAddress;
        Status = OrderStatus.Pending;
        TotalAmount = new Money(0, "TRY");
        CreatedAt = DateTime.UtcNow;
    }

    // Factory method: iş kurallarını uygulayarak nesne oluşturur
    public static Order Create(Guid customerId, Address shippingAddress)
    {
        if (customerId == Guid.Empty)
            throw new ArgumentException("Müşteri ID boş olamaz.", nameof(customerId));

        ArgumentNullException.ThrowIfNull(shippingAddress);

        var orderNumber = GenerateOrderNumber();
        var order = new Order(Guid.NewGuid(), orderNumber, customerId, shippingAddress);

        order.AddDomainEvent(new OrderCreatedEvent(order.Id, customerId, DateTime.UtcNow));

        return order;
    }

    // Aggregate içi tutarlılığı sağlayan metotlar
    public void AddItem(Guid productId, string productName, int quantity, Money unitPrice)
    {
        if (Status != OrderStatus.Pending)
            throw new InvalidOperationException("Onaylanmış siparişe ürün eklenemez.");

        if (quantity <= 0)
            throw new ArgumentException("Miktar sıfırdan büyük olmalıdır.", nameof(quantity));

        // Aynı ürün varsa miktarı artır
        var existingItem = _items.FirstOrDefault(i => i.ProductId == productId);
        if (existingItem != null)
        {
            existingItem.IncreaseQuantity(quantity);
        }
        else
        {
            var item = OrderItem.Create(productId, productName, quantity, unitPrice);
            _items.Add(item);
        }

        RecalculateTotalAmount();
    }

    public void RemoveItem(Guid productId)
    {
        if (Status != OrderStatus.Pending)
            throw new InvalidOperationException("Onaylanmış siparişten ürün kaldırılamaz.");

        var item = _items.FirstOrDefault(i => i.ProductId == productId);
        if (item is null)
            throw new InvalidOperationException($"Sipariş kaleminde {productId} bulunamadı.");

        _items.Remove(item);
        RecalculateTotalAmount();
    }

    public void Confirm()
    {
        if (Status != OrderStatus.Pending)
            throw new InvalidOperationException("Yalnızca bekleyen siparişler onaylanabilir.");

        if (!_items.Any())
            throw new InvalidOperationException("Boş sipariş onaylanamaz.");

        Status = OrderStatus.Confirmed;
        ConfirmedAt = DateTime.UtcNow;

        AddDomainEvent(new OrderConfirmedEvent(Id, CustomerId, TotalAmount, DateTime.UtcNow));
    }

    public void Ship(string trackingNumber)
    {
        if (Status != OrderStatus.Confirmed)
            throw new InvalidOperationException("Yalnızca onaylanmış siparişler gönderilebilir.");

        if (string.IsNullOrWhiteSpace(trackingNumber))
            throw new ArgumentException("Takip numarası boş olamaz.", nameof(trackingNumber));

        Status = OrderStatus.Shipped;
        ShippedAt = DateTime.UtcNow;

        AddDomainEvent(new OrderShippedEvent(Id, trackingNumber, DateTime.UtcNow));
    }

    public void Cancel(string reason)
    {
        if (Status == OrderStatus.Shipped || Status == OrderStatus.Delivered)
            throw new InvalidOperationException("Gönderilmiş veya teslim edilmiş sipariş iptal edilemez.");

        Status = OrderStatus.Cancelled;

        AddDomainEvent(new OrderCancelledEvent(Id, reason, DateTime.UtcNow));
    }

    public void UpdateShippingAddress(Address newAddress)
    {
        if (Status != OrderStatus.Pending)
            throw new InvalidOperationException("Yalnızca bekleyen siparişlerin adresi değiştirilebilir.");

        ArgumentNullException.ThrowIfNull(newAddress);
        ShippingAddress = newAddress;
    }

    private void RecalculateTotalAmount()
    {
        TotalAmount = _items.Aggregate(
            new Money(0, "TRY"),
            (total, item) => total.Add(item.TotalPrice));
    }

    private static string GenerateOrderNumber()
    {
        return $"ORD-{DateTime.UtcNow:yyyyMMdd}-{Guid.NewGuid().ToString("N")[..8].ToUpper()}";
    }
}

// OrderItem — Child Entity (Aggregate içinde yaşar, dışarıdan doğrudan erişilemez)
public class OrderItem : Entity
{
    public Guid ProductId { get; private set; }
    public string ProductName { get; private set; }
    public int Quantity { get; private set; }
    public Money UnitPrice { get; private set; }
    public Money TotalPrice => UnitPrice.Multiply(Quantity);

    protected OrderItem() { }

    private OrderItem(Guid productId, string productName, int quantity, Money unitPrice)
        : base(Guid.NewGuid())
    {
        ProductId = productId;
        ProductName = productName;
        Quantity = quantity;
        UnitPrice = unitPrice;
    }

    internal static OrderItem Create(Guid productId, string productName, int quantity, Money unitPrice)
    {
        if (productId == Guid.Empty)
            throw new ArgumentException("Ürün ID boş olamaz.", nameof(productId));

        if (string.IsNullOrWhiteSpace(productName))
            throw new ArgumentException("Ürün adı boş olamaz.", nameof(productName));

        if (quantity <= 0)
            throw new ArgumentException("Miktar sıfırdan büyük olmalıdır.", nameof(quantity));

        ArgumentNullException.ThrowIfNull(unitPrice);

        return new OrderItem(productId, productName, quantity, unitPrice);
    }

    internal void IncreaseQuantity(int amount)
    {
        if (amount <= 0)
            throw new ArgumentException("Artış miktarı pozitif olmalıdır.", nameof(amount));

        Quantity += amount;
    }
}

public enum OrderStatus
{
    Pending,
    Confirmed,
    Shipped,
    Delivered,
    Cancelled
}
```

## Repository ile Aggregate Erişimi

DDD'de Repository yalnızca Aggregate Root için tanımlanır. Child entity'lere ayrı Repository oluşturulmaz.

```csharp
// Repository interface — yalnızca Aggregate Root için
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<Order?> GetByOrderNumberAsync(string orderNumber, CancellationToken cancellationToken = default);
    Task<IEnumerable<Order>> GetByCustomerIdAsync(Guid customerId, CancellationToken cancellationToken = default);
    Task AddAsync(Order order, CancellationToken cancellationToken = default);
    void Update(Order order);
    void Remove(Order order);
}

// EF Core implementasyonu
public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;

    public OrderRepository(AppDbContext context)
    {
        _context = context;
    }

    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
    {
        // Aggregate tüm child entity'leriyle birlikte yüklenir
        return await _context.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id, cancellationToken);
    }

    public async Task<Order?> GetByOrderNumberAsync(string orderNumber, CancellationToken cancellationToken = default)
    {
        return await _context.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.OrderNumber == orderNumber, cancellationToken);
    }

    public async Task<IEnumerable<Order>> GetByCustomerIdAsync(
        Guid customerId,
        CancellationToken cancellationToken = default)
    {
        return await _context.Orders
            .Include(o => o.Items)
            .Where(o => o.CustomerId == customerId)
            .ToListAsync(cancellationToken);
    }

    public async Task AddAsync(Order order, CancellationToken cancellationToken = default)
    {
        await _context.Orders.AddAsync(order, cancellationToken);
    }

    public void Update(Order order)
    {
        _context.Orders.Update(order);
    }

    public void Remove(Order order)
    {
        _context.Orders.Remove(order);
    }
}
```

## Mülakat Soruları

### 1. Soru: Aggregate Root nedir ve neden gereklidir?

**Cevap:**

Aggregate Root, birbiriyle ilişkili nesneler kümesinin (Aggregate) dış dünyaya açılan tek giriş noktasıdır. Gereklilik nedenleri şunlardır:

- **Tutarlılık Garantisi**: Tüm değişiklikler root üzerinden geçtiğinden iş kuralları tek noktada uygulanır
- **Kapsülleme**: İç nesnelere doğrudan erişim engellenerek invariant'lar korunur
- **Transaction Sınırı**: Bir Aggregate, tek bir transaction içinde atomik olarak değiştirilir
- **Basit Repository**: Yalnızca Aggregate Root için Repository tanımlanır, karmaşıklık azalır

**Örnek Kod:**

```csharp
// YANLIŞ — Child entity'e doğrudan erişim
public class OrderService
{
    public void AddItemDirectly(Order order, OrderItem item)
    {
        // İş kuralları uygulanmıyor, tutarlılık bozulabilir
        order.Items.Add(item); // Derleme hatası — Items salt okunur
    }
}

// DOĞRU — Aggregate Root üzerinden erişim
public class OrderService
{
    public void AddItem(Order order, Guid productId, string productName,
        int quantity, Money unitPrice)
    {
        // İş kuralları Order.AddItem içinde uygulanıyor
        order.AddItem(productId, productName, quantity, unitPrice);
    }
}
```

---

### 2. Soru: Entity ile Value Object arasındaki fark nedir? Örneklerle açıklayınız.

**Cevap:**

| Özellik | Entity | Value Object |
|---------|--------|--------------|
| Kimlik | Benzersiz ID | Yok |
| Karşılaştırma | ID ile | Değerler ile |
| Değişkenlik | Mutable | Immutable |
| Yaşam döngüsü | Takip edilir | Takip edilmez |
| Örnek | Order, Customer, Product | Money, Address, DateRange |

**Örnek Kod:**

```csharp
// Entity örneği: Customer
public class Customer : Entity
{
    public string Email { get; private set; }
    public string FullName { get; private set; }

    private Customer() { }

    public Customer(string email, string fullName) : base(Guid.NewGuid())
    {
        Email = email;
        FullName = fullName;
    }

    public void UpdateEmail(string newEmail)
    {
        // Entity mutable olabilir; kimliği değişmez
        Email = newEmail;
    }
}

// Value Object örneği: EmailAddress
public sealed class EmailAddress : ValueObject
{
    public string Value { get; }

    public EmailAddress(string value)
    {
        if (string.IsNullOrWhiteSpace(value) || !value.Contains('@'))
            throw new ArgumentException("Geçersiz e-posta adresi.", nameof(value));

        Value = value.ToLowerInvariant();
    }

    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return Value;
    }

    // Değiştirmek yerine yeni nesne döndür (immutability)
    public EmailAddress ChangeToSubdomain(string subdomain)
    {
        var parts = Value.Split('@');
        return new EmailAddress($"{parts[0]}@{subdomain}.{parts[1]}");
    }
}

// Kullanım karşılaştırması
var customer1 = new Customer("ali@example.com", "Ali Yılmaz");
var customer2 = new Customer("ali@example.com", "Ali Yılmaz");
Console.WriteLine(customer1 == customer2); // false — farklı ID'ler

var email1 = new EmailAddress("ali@example.com");
var email2 = new EmailAddress("ali@example.com");
Console.WriteLine(email1 == email2); // true — aynı değerler
```

---

### 3. Soru: Aggregate sınırları nasıl belirlenir?

**Cevap:**

Aggregate sınırları belirlenirken şu sorular sorulmalıdır:

1. **Hangi nesneler birlikte tutarlı olmalı?** — Aynı iş kurallarının koruması altında olan nesneler
2. **Hangi nesneler birlikte değişiyor?** — Aynı use case içinde birlikte güncellenen nesneler
3. **Hangi nesneler bağımsız yaşam döngüsüne sahip?** — Bağımsız olanlar ayrı Aggregate olmalı

**İyi pratikler:**
- Aggregate'leri küçük tutun (1-4 nesne ideal)
- Aggregate'ler arası ilişkiyi ID referansı ile kurun, nesne referansıyla değil
- Bir Aggregate'e bir transaction'da dokunun

**Örnek Kod:**

```csharp
// YANLIŞ — Çok büyük Aggregate (God Object anti-pattern)
public class Order : AggregateRoot
{
    public Customer Customer { get; private set; }        // Ayrı Aggregate olmalı
    public List<Product> Products { get; private set; }   // Ayrı Aggregate olmalı
    public Payment Payment { get; private set; }          // Ayrı Aggregate olabilir
    public List<OrderItem> Items { get; private set; }
    // ...
}

// DOĞRU — ID referansı ile ilişki
public class Order : AggregateRoot
{
    public Guid CustomerId { get; private set; }    // Referans, nesne değil
    public List<OrderItem> Items { get; private set; } // Order'a ait, birlikte değişir
    // ...
}
```

---

### 4. Soru: Aggregate içinde iş kuralları (invariant) nasıl korunur?

**Cevap:**

Invariant'lar, bir Aggregate'in her zaman geçerli durumda olmasını sağlayan kural kümeleridir. Koruma yöntemleri:

- **Private setter kullanımı**: Dış kod property'leri doğrudan değiştiremez
- **Factory method**: Geçerli başlangıç durumu garanti edilir
- **Domain method'ları**: Durum değişikliklerini kapsüller ve kuralları uygular
- **Guard clause**: Metot başlarında önkoşullar kontrol edilir

**Örnek Kod:**

```csharp
public class BankAccount : AggregateRoot
{
    public string AccountNumber { get; private set; }
    public Money Balance { get; private set; }
    public bool IsActive { get; private set; }
    private const decimal MaxDailyWithdrawalLimit = 10_000m;

    protected BankAccount() { }

    public static BankAccount Open(string accountNumber, Money initialDeposit)
    {
        if (initialDeposit.Amount < 50)
            throw new DomainException("Minimum açılış bakiyesi 50 TRY olmalıdır.");

        var account = new BankAccount
        {
            Id = Guid.NewGuid(),
            AccountNumber = accountNumber,
            Balance = initialDeposit,
            IsActive = true
        };

        account.AddDomainEvent(new AccountOpenedEvent(account.Id, initialDeposit));
        return account;
    }

    public void Deposit(Money amount)
    {
        // Invariant: Hesap aktif olmalı
        if (!IsActive)
            throw new DomainException("Kapalı hesaba para yatırılamaz.");

        if (amount.Amount <= 0)
            throw new DomainException("Yatırma miktarı pozitif olmalıdır.");

        Balance = Balance.Add(amount);
        AddDomainEvent(new MoneyDepositedEvent(Id, amount, Balance));
    }

    public void Withdraw(Money amount)
    {
        if (!IsActive)
            throw new DomainException("Kapalı hesaptan para çekilemez.");

        if (amount.Amount <= 0)
            throw new DomainException("Çekme miktarı pozitif olmalıdır.");

        // Invariant: Bakiye yetersiz olmamalı
        if (Balance.Amount < amount.Amount)
            throw new DomainException("Yetersiz bakiye.");

        // Invariant: Günlük limit aşılmamalı
        if (amount.Amount > MaxDailyWithdrawalLimit)
            throw new DomainException($"Günlük çekim limiti {MaxDailyWithdrawalLimit} TRY'dir.");

        Balance = Balance.Subtract(amount);
        AddDomainEvent(new MoneyWithdrawnEvent(Id, amount, Balance));
    }

    public void Close()
    {
        if (!IsActive)
            throw new DomainException("Hesap zaten kapalı.");

        if (Balance.Amount > 0)
            throw new DomainException("Bakiyeli hesap kapatılamaz. Önce bakiyeyi sıfırlayın.");

        IsActive = false;
        AddDomainEvent(new AccountClosedEvent(Id, DateTime.UtcNow));
    }
}
```

---

### 5. Soru: Aggregate'ler arası ilişki nasıl kurulur?

**Cevap:**

Aggregate'ler arası doğrudan nesne referansı yerine ID referansı kullanılır. Bu yaklaşım:

- Her Aggregate'in bağımsız yüklenmesini sağlar
- Transaction sınırlarını net tanımlar
- Aggregate büyümesini engeller
- Eventual consistency'ye kapı açar

**Örnek Kod:**

```csharp
// YANLIŞ — Nesne referansı (Aggregate sınırı ihlali)
public class Order : AggregateRoot
{
    public Customer Customer { get; private set; }  // Customer ayrı Aggregate
    // Order yüklenirken Customer da yüklenmek zorunda
}

// DOĞRU — ID referansı
public class Order : AggregateRoot
{
    public Guid CustomerId { get; private set; }  // Yalnızca ID

    // Müşteri bilgisine ihtiyaç duyulduğunda ayrıca yüklenir
}

// Application Service'de gerektiğinde ilgili Aggregate yüklenir
public class OrderApplicationService
{
    private readonly IOrderRepository _orderRepository;
    private readonly ICustomerRepository _customerRepository;

    public OrderApplicationService(
        IOrderRepository orderRepository,
        ICustomerRepository customerRepository)
    {
        _orderRepository = orderRepository;
        _customerRepository = customerRepository;
    }

    public async Task ConfirmOrderAsync(Guid orderId)
    {
        var order = await _orderRepository.GetByIdAsync(orderId)
            ?? throw new NotFoundException($"Sipariş bulunamadı: {orderId}");

        // Müşteri gerekiyorsa ayrı repository'den yüklenir
        var customer = await _customerRepository.GetByIdAsync(order.CustomerId)
            ?? throw new NotFoundException($"Müşteri bulunamadı: {order.CustomerId}");

        if (!customer.IsVerified)
            throw new DomainException("Doğrulanmamış müşteri siparişi onaylanamaz.");

        order.Confirm();
        _orderRepository.Update(order);
    }
}
```

---

### 6. Soru: Value Object tasarımında dikkat edilmesi gereken noktalar nelerdir?

**Cevap:**

Value Object tasarımında temel prensipler:

1. **Immutability**: Tüm property'ler `init` veya constructor'dan sonra değiştirilemez
2. **Structural Equality**: `GetEqualityComponents` doğru implemente edilmeli
3. **Self-validation**: Constructor'da tüm iş kuralları validate edilmeli
4. **Side-effect free**: Metotlar yeni Value Object döndürmeli, mevcut nesneyi değiştirmemeli
5. **Anlamlı davranış**: Domain logic, Value Object'e taşınabilir

**Örnek Kod:**

```csharp
public sealed class DateRange : ValueObject
{
    public DateTime StartDate { get; }
    public DateTime EndDate { get; }
    public int DurationInDays => (EndDate - StartDate).Days;

    public DateRange(DateTime startDate, DateTime endDate)
    {
        if (endDate <= startDate)
            throw new ArgumentException("Bitiş tarihi başlangıç tarihinden sonra olmalıdır.");

        StartDate = startDate;
        EndDate = endDate;
    }

    public bool Overlaps(DateRange other)
    {
        return StartDate < other.EndDate && EndDate > other.StartDate;
    }

    public bool Contains(DateTime date)
    {
        return date >= StartDate && date <= EndDate;
    }

    public DateRange Extend(int days)
    {
        // Yeni nesne döndür — mevcut nesneyi değiştirme
        return new DateRange(StartDate, EndDate.AddDays(days));
    }

    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return StartDate;
        yield return EndDate;
    }

    public override string ToString()
    {
        return $"{StartDate:dd.MM.yyyy} - {EndDate:dd.MM.yyyy} ({DurationInDays} gün)";
    }
}

// Kullanım
var reservation = new DateRange(
    new DateTime(2024, 6, 1),
    new DateTime(2024, 6, 7));

var extended = reservation.Extend(3);  // Yeni nesne: 6/1 - 6/10
Console.WriteLine(reservation == extended);  // false — farklı değerler
Console.WriteLine(reservation.Overlaps(extended));  // true — çakışıyor
```
