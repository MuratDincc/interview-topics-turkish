# Performance

## Genel Bakış
Veritabanı performansı, uygulamanın genel performansını doğrudan etkileyen kritik bir faktördür. Entity Framework Core ve veritabanı işlemlerinde performans optimizasyonu, uygulamanın ölçeklenebilirliği ve kullanıcı deneyimi açısından önemlidir.

## Mülakat Soruları ve Cevapları

### 1. Veritabanı performansını etkileyen faktörler nelerdir?
**Cevap:**
Veritabanı performansını etkileyen temel faktörler:
- Sorgu optimizasyonu
- Index kullanımı
- Connection pooling
- Batch işlemleri
- Caching stratejileri

**Örnek Kod:**
```csharp
// Performanslı sorgu örneği
public async Task<List<Product>> GetProductsAsync()
{
    return await _context.Products
        .AsNoTracking() // Change tracking'i devre dışı bırak
        .Where(p => p.IsActive)
        .Select(p => new Product
        {
            Id = p.Id,
            Name = p.Name,
            Price = p.Price
        })
        .ToListAsync();
}

// Connection pooling örneği
services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection"),
        sqlOptions =>
        {
            sqlOptions.EnableRetryOnFailure(
                maxRetryCount: 5,
                maxRetryDelay: TimeSpan.FromSeconds(30),
                errorNumbersToAdd: null);
        }));
```

### 2. N+1 problemi nedir ve nasıl çözülür?
**Cevap:**
N+1 problemi:
- Ana sorgu sonrası her kayıt için ek sorgu yapılması
- Performans sorunlarına yol açar
- Eager Loading veya Projection ile çözülür

**Örnek Kod:**
```csharp
// N+1 problemi
var orders = await _context.Orders.ToListAsync();
foreach (var order in orders)
{
    // Her order için ayrı sorgu
    var customer = await _context.Customers.FindAsync(order.CustomerId);
}

// Eager Loading çözümü
var orders = await _context.Orders
    .Include(o => o.Customer)
    .ToListAsync();

// Projection çözümü
var orders = await _context.Orders
    .Select(o => new OrderDto
    {
        Id = o.Id,
        CustomerName = o.Customer.Name,
        OrderDate = o.OrderDate
    })
    .ToListAsync();
```

### 3. Index'ler nasıl kullanılmalıdır?
**Cevap:**
Index kullanımı için:
- Sık sorgulanan alanlara index ekleyin
- Composite index'leri doğru sıralayın
- Gereksiz index'lerden kaçının
- Index bakımını düzenli yapın

**Örnek Kod:**
```csharp
// Index tanımlama
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Product>()
        .HasIndex(p => p.Name)
        .IsUnique();

    modelBuilder.Entity<Order>()
        .HasIndex(o => new { o.CustomerId, o.OrderDate })
        .IsClustered(false);
}

// Index kullanımı
var products = await _context.Products
    .Where(p => p.Name.StartsWith("A"))
    .ToListAsync();
```

### 4. Batch işlemleri nasıl optimize edilir?
**Cevap:**
Batch işlem optimizasyonu için:
- AddRange/UpdateRange kullanın
- SaveChanges çağrılarını azaltın
- Bulk insert/update kullanın
- Transaction kullanın

**Örnek Kod:**
```csharp
// Batch insert
public async Task AddProductsAsync(List<Product> products)
{
    await _context.Products.AddRangeAsync(products);
    await _context.SaveChangesAsync();
}

// Bulk insert
public async Task BulkInsertProductsAsync(List<Product> products)
{
    using (var transaction = await _context.Database.BeginTransactionAsync())
    {
        try
        {
            await _context.BulkInsertAsync(products);
            await transaction.CommitAsync();
        }
        catch (Exception)
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}
```

### 5. Caching stratejileri nelerdir?
**Cevap:**
Caching stratejileri:
- Memory Cache
- Distributed Cache
- Query Cache
- Second Level Cache

**Örnek Kod:**
```csharp
// Memory Cache
public class ProductService
{
    private readonly IMemoryCache _cache;
    private readonly ApplicationDbContext _context;

    public async Task<List<Product>> GetProductsAsync()
    {
        return await _cache.GetOrCreateAsync("products", async entry =>
        {
            entry.SlidingExpiration = TimeSpan.FromMinutes(30);
            return await _context.Products.ToListAsync();
        });
    }
}

// Distributed Cache
public class ProductService
{
    private readonly IDistributedCache _cache;
    private readonly ApplicationDbContext _context;

    public async Task<List<Product>> GetProductsAsync()
    {
        var cachedProducts = await _cache.GetStringAsync("products");
        if (cachedProducts != null)
        {
            return JsonSerializer.Deserialize<List<Product>>(cachedProducts);
        }

        var products = await _context.Products.ToListAsync();
        await _cache.SetStringAsync("products", 
            JsonSerializer.Serialize(products),
            new DistributedCacheEntryOptions
            {
                SlidingExpiration = TimeSpan.FromMinutes(30)
            });

        return products;
    }
}
```

## Best Practices
1. **Sorgu Optimizasyonu**
   - Gereksiz Include'lardan kaçının
   - Select ile projeksiyon yapın
   - AsNoTracking kullanın
   - Raw SQL sorgularını dikkatli kullanın

2. **Index Yönetimi**
   - Sık sorgulanan alanlara index ekleyin
   - Composite index'leri doğru sıralayın
   - Index bakımını düzenli yapın
   - Index fragmentasyonunu kontrol edin

3. **Caching Stratejisi**
   - Uygun cache stratejisi seçin
   - Cache invalidation stratejisi belirleyin
   - Cache boyutunu yönetin
   - Cache hit/miss oranlarını izleyin

## Kaynaklar
- [Entity Framework Core Performance](https://docs.microsoft.com/tr-tr/ef/core/performance/)
- [Query Performance](https://docs.microsoft.com/tr-tr/ef/core/querying/related-data/eager)
- [Caching in .NET](https://docs.microsoft.com/tr-tr/aspnet/core/performance/caching/overview) 