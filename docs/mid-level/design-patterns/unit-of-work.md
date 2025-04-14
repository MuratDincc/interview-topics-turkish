# Unit of Work Pattern

## Genel Bakış
Unit of Work Pattern, bir iş birimi içinde yapılan tüm veritabanı işlemlerini tek bir transaction altında yönetir. Bu pattern, veritabanı işlemlerinin tutarlılığını sağlar ve transaction yönetimini merkezileştirir.

## Mülakat Soruları ve Cevapları

### 1. Unit of Work Pattern nedir ve ne zaman kullanılır?
**Cevap:**
Unit of Work Pattern, veritabanı işlemlerini tek bir transaction altında yönetmek için kullanılır. Genellikle:
- Birden fazla repository'yi tek bir transaction altında yönetmek için
- Veritabanı işlemlerinin tutarlılığını sağlamak için
- Transaction yönetimini merkezileştirmek için
- Repository Pattern ile birlikte kullanılır.

**Örnek Kod:**
```csharp
public interface IUnitOfWork : IDisposable
{
    IRepository<Product> Products { get; }
    IRepository<Order> Orders { get; }
    IRepository<Customer> Customers { get; }
    int SaveChanges();
    Task<int> SaveChangesAsync();
}

public class UnitOfWork : IUnitOfWork
{
    private readonly DbContext _context;
    private IRepository<Product> _products;
    private IRepository<Order> _orders;
    private IRepository<Customer> _customers;

    public UnitOfWork(DbContext context)
    {
        _context = context;
    }

    public IRepository<Product> Products
    {
        get
        {
            if (_products == null)
                _products = new GenericRepository<Product>(_context);
            return _products;
        }
    }

    public IRepository<Order> Orders
    {
        get
        {
            if (_orders == null)
                _orders = new GenericRepository<Order>(_context);
            return _orders;
        }
    }

    public IRepository<Customer> Customers
    {
        get
        {
            if (_customers == null)
                _customers = new GenericRepository<Customer>(_context);
            return _customers;
        }
    }

    public int SaveChanges()
    {
        return _context.SaveChanges();
    }

    public async Task<int> SaveChangesAsync()
    {
        return await _context.SaveChangesAsync();
    }

    public void Dispose()
    {
        _context.Dispose();
    }
}
```

### 2. Unit of Work Pattern'in avantajları nelerdir?
**Cevap:**
Unit of Work Pattern'in avantajları:
- Transaction yönetiminin merkezileştirilmesi
- Veritabanı işlemlerinin tutarlılığının sağlanması
- Kod tekrarının azalması
- Test edilebilirliğin artması
- Repository'ler arası işlemlerin kolay yönetimi

**Örnek Kod:**
```csharp
public class OrderService
{
    private readonly IUnitOfWork _unitOfWork;

    public OrderService(IUnitOfWork unitOfWork)
    {
        _unitOfWork = unitOfWork;
    }

    public async Task CreateOrderAsync(Order order, List<Product> products)
    {
        try
        {
            // Tüm işlemler tek bir transaction altında
            _unitOfWork.Orders.Add(order);
            
            foreach (var product in products)
            {
                product.Stock -= order.Quantity;
                _unitOfWork.Products.Update(product);
            }

            await _unitOfWork.SaveChangesAsync();
        }
        catch (Exception)
        {
            // Hata durumunda tüm değişiklikler geri alınır
            throw;
        }
    }
}
```

### 3. Unit of Work Pattern'de transaction yönetimi nasıl yapılır?
**Cevap:**
Transaction yönetimi için:
- DbContext transaction'ları
- TransactionScope kullanımı
- Manuel transaction yönetimi
- Nested transaction'lar

**Örnek Kod:**
```csharp
public class UnitOfWork : IUnitOfWork
{
    private readonly DbContext _context;
    private IDbContextTransaction _transaction;

    public UnitOfWork(DbContext context)
    {
        _context = context;
    }

    public void BeginTransaction()
    {
        _transaction = _context.Database.BeginTransaction();
    }

    public void Commit()
    {
        try
        {
            _context.SaveChanges();
            _transaction?.Commit();
        }
        catch
        {
            Rollback();
            throw;
        }
    }

    public void Rollback()
    {
        _transaction?.Rollback();
    }

    public async Task CommitAsync()
    {
        try
        {
            await _context.SaveChangesAsync();
            await _transaction?.CommitAsync();
        }
        catch
        {
            await RollbackAsync();
            throw;
        }
    }

    public async Task RollbackAsync()
    {
        if (_transaction != null)
            await _transaction.RollbackAsync();
    }
}

// Kullanım
public class OrderService
{
    private readonly IUnitOfWork _unitOfWork;

    public async Task ProcessOrderAsync(Order order)
    {
        _unitOfWork.BeginTransaction();
        try
        {
            // İşlemler
            await _unitOfWork.CommitAsync();
        }
        catch
        {
            await _unitOfWork.RollbackAsync();
            throw;
        }
    }
}
```

### 4. Unit of Work Pattern'de performans optimizasyonu nasıl yapılır?
**Cevap:**
Performans optimizasyonu için:
- Batch işlemleri
- Asenkron operasyonlar
- Caching mekanizmaları
- Lazy loading
- Change tracking optimizasyonu

**Örnek Kod:**
```csharp
public class UnitOfWork : IUnitOfWork
{
    private readonly DbContext _context;
    private readonly MemoryCache _cache;

    public UnitOfWork(DbContext context)
    {
        _context = context;
        _cache = new MemoryCache(new MemoryCacheOptions());
        
        // Change tracking optimizasyonu
        _context.ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
    }

    public async Task<int> SaveChangesAsync()
    {
        // Batch işlemleri için
        _context.ChangeTracker.AutoDetectChangesEnabled = false;
        
        try
        {
            var result = await _context.SaveChangesAsync();
            
            // Cache temizleme
            _cache.Remove("products_all");
            _cache.Remove("orders_all");
            
            return result;
        }
        finally
        {
            _context.ChangeTracker.AutoDetectChangesEnabled = true;
        }
    }
}
```

### 5. Unit of Work Pattern ile Repository Pattern nasıl birlikte kullanılır?
**Cevap:**
Unit of Work ve Repository Pattern'lerin birlikte kullanımı:
- Unit of Work repository'leri yönetir
- Repository'ler veri erişimini sağlar
- Transaction yönetimi Unit of Work'te yapılır
- Dependency Injection ile entegrasyon

**Örnek Kod:**
```csharp
public class OrderService
{
    private readonly IUnitOfWork _unitOfWork;

    public OrderService(IUnitOfWork unitOfWork)
    {
        _unitOfWork = unitOfWork;
    }

    public async Task ProcessOrderAsync(Order order)
    {
        try
        {
            // Ürün stok kontrolü
            var product = await _unitOfWork.Products.GetByIdAsync(order.ProductId);
            if (product.Stock < order.Quantity)
                throw new Exception("Yetersiz stok");

            // Sipariş oluşturma
            _unitOfWork.Orders.Add(order);

            // Stok güncelleme
            product.Stock -= order.Quantity;
            _unitOfWork.Products.Update(product);

            // Müşteri puanı güncelleme
            var customer = await _unitOfWork.Customers.GetByIdAsync(order.CustomerId);
            customer.Points += order.TotalAmount / 10;
            _unitOfWork.Customers.Update(customer);

            // Tüm değişiklikleri kaydet
            await _unitOfWork.SaveChangesAsync();
        }
        catch (Exception)
        {
            // Hata durumunda tüm değişiklikler geri alınır
            throw;
        }
    }
}
```

## Best Practices
1. **Transaction Yönetimi**
   - Transaction'ları doğru yönetin
   - Hata durumlarını kontrol edin
   - Nested transaction'lardan kaçının
   - Transaction scope'larını sınırlayın

2. **Performans**
   - Batch işlemleri kullanın
   - Asenkron operasyonlar tercih edin
   - Change tracking'i optimize edin
   - Caching mekanizmaları ekleyin

3. **Test Edilebilirlik**
   - Mock Unit of Work oluşturun
   - Unit testler yazın
   - Integration testler yapın
   - Test coverage'ı takip edin

## Kaynaklar
- [Microsoft Unit of Work Pattern](https://docs.microsoft.com/tr-tr/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-implementation)
- [Implementing Unit of Work Pattern](https://docs.microsoft.com/tr-tr/aspnet/mvc/overview/older-versions/getting-started-with-ef-5-using-mvc-4/implementing-the-repository-and-unit-of-work-patterns-in-an-asp-net-mvc-application)
- [Unit of Work Best Practices](https://docs.microsoft.com/tr-tr/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-implementation#implementing-the-unit-of-work-pattern) 