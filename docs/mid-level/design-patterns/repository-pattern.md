# Repository Pattern

## Genel Bakış
Repository Pattern, veri erişim mantığını soyutlayarak, uygulama katmanı ile veri erişim katmanı arasında bir aracı görevi görür. Bu pattern, veri kaynağından veri almayı ve veri kaynağına veri göndermeyi yönetir.

## Mülakat Soruları ve Cevapları

### 1. Repository Pattern nedir ve ne zaman kullanılır?
**Cevap:**
Repository Pattern, veri erişim mantığını kapsüller ve veri kaynağından veri almayı ve veri kaynağına veri göndermeyi yönetir. Genellikle:
- Veri erişim mantığını soyutlamak için
- Test edilebilirliği artırmak için
- Veri kaynağı değişikliklerini yönetmek için
- Kod tekrarını azaltmak için kullanılır.

**Örnek Kod:**
```csharp
// Entity
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
}

// Repository interface
public interface IRepository<T> where T : class
{
    IEnumerable<T> GetAll();
    T GetById(int id);
    void Add(T entity);
    void Update(T entity);
    void Delete(T entity);
}

// Concrete repository
public class ProductRepository : IRepository<Product>
{
    private readonly DbContext _context;

    public ProductRepository(DbContext context)
    {
        _context = context;
    }

    public IEnumerable<Product> GetAll()
    {
        return _context.Products.ToList();
    }

    public Product GetById(int id)
    {
        return _context.Products.Find(id);
    }

    public void Add(Product product)
    {
        _context.Products.Add(product);
        _context.SaveChanges();
    }

    public void Update(Product product)
    {
        _context.Entry(product).State = EntityState.Modified;
        _context.SaveChanges();
    }

    public void Delete(Product product)
    {
        _context.Products.Remove(product);
        _context.SaveChanges();
    }
}
```

### 2. Generic Repository nasıl implemente edilir?
**Cevap:**
Generic Repository, farklı entity tipleri için tekrar kullanılabilir bir repository sağlar.

**Örnek Kod:**
```csharp
public class GenericRepository<T> : IRepository<T> where T : class
{
    private readonly DbContext _context;
    private readonly DbSet<T> _dbSet;

    public GenericRepository(DbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }

    public IEnumerable<T> GetAll()
    {
        return _dbSet.ToList();
    }

    public T GetById(int id)
    {
        return _dbSet.Find(id);
    }

    public void Add(T entity)
    {
        _dbSet.Add(entity);
        _context.SaveChanges();
    }

    public void Update(T entity)
    {
        _context.Entry(entity).State = EntityState.Modified;
        _context.SaveChanges();
    }

    public void Delete(T entity)
    {
        _dbSet.Remove(entity);
        _context.SaveChanges();
    }
}

// Kullanım
public class ProductService
{
    private readonly IRepository<Product> _productRepository;

    public ProductService(IRepository<Product> productRepository)
    {
        _productRepository = productRepository;
    }

    public IEnumerable<Product> GetAllProducts()
    {
        return _productRepository.GetAll();
    }
}
```

### 3. Repository Pattern'in avantajları nelerdir?
**Cevap:**
Repository Pattern'in avantajları:
- Veri erişim mantığının merkezileştirilmesi
- Test edilebilirliğin artması
- Kod tekrarının azalması
- Veri kaynağı değişikliklerinin kolay yönetimi
- İş mantığı ve veri erişim mantığının ayrılması

**Örnek Kod:**
```csharp
// Test edilebilirlik örneği
public class ProductServiceTests
{
    [Fact]
    public void GetAllProducts_ReturnsAllProducts()
    {
        // Arrange
        var mockRepository = new Mock<IRepository<Product>>();
        var products = new List<Product>
        {
            new Product { Id = 1, Name = "Product 1" },
            new Product { Id = 2, Name = "Product 2" }
        };
        mockRepository.Setup(r => r.GetAll()).Returns(products);
        var service = new ProductService(mockRepository.Object);

        // Act
        var result = service.GetAllProducts();

        // Assert
        Assert.Equal(2, result.Count());
    }
}
```

### 4. Repository Pattern ile Unit of Work Pattern nasıl birlikte kullanılır?
**Cevap:**
Unit of Work Pattern, birden fazla repository'yi tek bir transaction altında yönetmek için kullanılır.

**Örnek Kod:**
```csharp
public interface IUnitOfWork : IDisposable
{
    IRepository<Product> Products { get; }
    IRepository<Order> Orders { get; }
    int SaveChanges();
}

public class UnitOfWork : IUnitOfWork
{
    private readonly DbContext _context;
    private IRepository<Product> _products;
    private IRepository<Order> _orders;

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

    public int SaveChanges()
    {
        return _context.SaveChanges();
    }

    public void Dispose()
    {
        _context.Dispose();
    }
}

// Kullanım
public class OrderService
{
    private readonly IUnitOfWork _unitOfWork;

    public OrderService(IUnitOfWork unitOfWork)
    {
        _unitOfWork = unitOfWork;
    }

    public void CreateOrder(Order order, List<Product> products)
    {
        _unitOfWork.Orders.Add(order);
        foreach (var product in products)
        {
            _unitOfWork.Products.Update(product);
        }
        _unitOfWork.SaveChanges();
    }
}
```

### 5. Repository Pattern'de performans optimizasyonu nasıl yapılır?
**Cevap:**
Repository Pattern'de performans optimizasyonu için:
- Lazy loading kullanımı
- Eager loading stratejileri
- Caching mekanizmaları
- Batch işlemleri
- Asenkron operasyonlar

**Örnek Kod:**
```csharp
public interface IRepository<T> where T : class
{
    Task<IEnumerable<T>> GetAllAsync();
    Task<T> GetByIdAsync(int id);
    Task AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(T entity);
}

public class ProductRepository : IRepository<Product>
{
    private readonly DbContext _context;
    private readonly MemoryCache _cache;

    public ProductRepository(DbContext context)
    {
        _context = context;
        _cache = new MemoryCache(new MemoryCacheOptions());
    }

    public async Task<IEnumerable<Product>> GetAllAsync()
    {
        const string cacheKey = "products_all";
        if (_cache.TryGetValue(cacheKey, out IEnumerable<Product> products))
        {
            return products;
        }

        products = await _context.Products
            .Include(p => p.Category)
            .ToListAsync();

        var cacheOptions = new MemoryCacheEntryOptions()
            .SetSlidingExpiration(TimeSpan.FromMinutes(5));

        _cache.Set(cacheKey, products, cacheOptions);
        return products;
    }

    public async Task AddAsync(Product product)
    {
        await _context.Products.AddAsync(product);
        await _context.SaveChangesAsync();
        _cache.Remove("products_all");
    }
}
```

## Best Practices
1. **Repository Tasarımı**
   - Interface'leri doğru tanımlayın
   - Generic repository kullanın
   - Dependency Injection kullanın
   - SOLID prensiplerini takip edin

2. **Performans**
   - Lazy loading kullanın
   - Caching mekanizmaları ekleyin
   - Batch işlemleri yapın
   - Asenkron operasyonlar kullanın

3. **Test Edilebilirlik**
   - Mock repository'ler oluşturun
   - Unit testler yazın
   - Integration testler yapın
   - Test coverage'ı takip edin

## Kaynaklar
- [Microsoft Repository Pattern](https://docs.microsoft.com/tr-tr/previous-versions/msp-n-p/ff649690(v=pandp.10))
- [Repository Pattern Best Practices](https://docs.microsoft.com/tr-tr/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-implementation)
- [Implementing Repository Pattern in C#](https://docs.microsoft.com/tr-tr/aspnet/mvc/overview/older-versions/getting-started-with-ef-5-using-mvc-4/implementing-the-repository-and-unit-of-work-patterns-in-an-asp-net-mvc-application) 