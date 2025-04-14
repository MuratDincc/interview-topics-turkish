# Entity Framework Core

## Genel Bakış
Entity Framework Core (EF Core), .NET uygulamalarında veritabanı işlemlerini gerçekleştirmek için kullanılan bir Object-Relational Mapping (ORM) aracıdır. Veritabanı işlemlerini nesne yönelimli bir yaklaşımla gerçekleştirmeyi sağlar.

## Mülakat Soruları ve Cevapları

### 1. Entity Framework Core nedir ve neden kullanılır?
**Cevap:**
Entity Framework Core, veritabanı işlemlerini nesne yönelimli bir yaklaşımla gerçekleştirmeyi sağlayan bir ORM aracıdır. Kullanım nedenleri:
- Veritabanı işlemlerini kolaylaştırır
- LINQ desteği sunar
- Cross-platform çalışabilir
- Code First yaklaşımını destekler

**Örnek Kod:**
```csharp
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    public DbSet<Product> Products { get; set; }
    public DbSet<Category> Categories { get; set; }
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public int CategoryId { get; set; }
    public Category Category { get; set; }
}
```

### 2. DbContext ve DbSet nedir?
**Cevap:**
DbContext:
- Veritabanı bağlantısını yönetir
- Entity'lerin yaşam döngüsünü kontrol eder
- Change tracking sağlar
- Transaction yönetimi yapar

DbSet:
- Entity koleksiyonlarını temsil eder
- CRUD operasyonlarını sağlar
- LINQ sorgularına olanak tanır

**Örnek Kod:**
```csharp
public class ProductService
{
    private readonly ApplicationDbContext _context;

    public ProductService(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<List<Product>> GetAllProducts()
    {
        return await _context.Products
            .Include(p => p.Category)
            .ToListAsync();
    }

    public async Task AddProduct(Product product)
    {
        _context.Products.Add(product);
        await _context.SaveChangesAsync();
    }
}
```

### 3. Lazy Loading ve Eager Loading arasındaki farklar nelerdir?
**Cevap:**
Lazy Loading:
- İlişkili veriler ihtiyaç duyulduğunda yüklenir
- Performans açısından daha maliyetli
- N+1 problemi oluşturabilir
- Proxies gerektirir

Eager Loading:
- İlişkili veriler ana sorguyla birlikte yüklenir
- Daha performanslı
- Include metodu ile kullanılır
- Tek sorgu ile veri çeker

**Örnek Kod:**
```csharp
// Lazy Loading
public class Product
{
    public virtual Category Category { get; set; }
}

// Eager Loading
var products = await _context.Products
    .Include(p => p.Category)
    .ToListAsync();
```

### 4. Change Tracking nedir ve nasıl çalışır?
**Cevap:**
Change Tracking:
- Entity'lerdeki değişiklikleri takip eder
- SaveChanges çağrıldığında değişiklikleri veritabanına yansıtır
- EntityState ile durumları yönetir
- Performans optimizasyonu sağlar

**Örnek Kod:**
```csharp
public async Task UpdateProduct(Product product)
{
    var existingProduct = await _context.Products.FindAsync(product.Id);
    
    if (existingProduct != null)
    {
        // Change tracking otomatik olarak değişiklikleri takip eder
        existingProduct.Name = product.Name;
        existingProduct.Price = product.Price;
        
        await _context.SaveChangesAsync();
    }
}
```

### 5. Entity Framework Core'da performans optimizasyonu nasıl yapılır?
**Cevap:**
Performans optimizasyonu için:
- Eager Loading kullanın
- Select ile sadece ihtiyaç duyulan alanları çekin
- AsNoTracking kullanın
- Batch işlemleri için AddRange kullanın

**Örnek Kod:**
```csharp
// Sadece ihtiyaç duyulan alanları çekme
var products = await _context.Products
    .Select(p => new { p.Id, p.Name })
    .ToListAsync();

// Change tracking'i devre dışı bırakma
var products = await _context.Products
    .AsNoTracking()
    .ToListAsync();

// Batch insert
await _context.Products.AddRangeAsync(products);
await _context.SaveChangesAsync();
```

## Best Practices
1. **DbContext Yönetimi**
   - Scoped lifetime kullanın
   - Connection pooling'i etkinleştirin
   - Dispose pattern'i uygulayın
   - DbContext'i thread-safe kullanın

2. **Sorgu Optimizasyonu**
   - N+1 probleminden kaçının
   - Gereksiz Include'lardan kaçının
   - Index'leri doğru kullanın
   - Raw SQL sorgularını dikkatli kullanın

3. **Migration Yönetimi**
   - Migration'ları version control'de tutun
   - Seed data'yı düzenli güncelleyin
   - Rollback stratejisi belirleyin
   - Production migration'larını test edin

## Kaynaklar
- [Entity Framework Core Dokümantasyonu](https://docs.microsoft.com/tr-tr/ef/core/)
- [Entity Framework Core Performance](https://docs.microsoft.com/tr-tr/ef/core/performance/)
- [Entity Framework Core Best Practices](https://docs.microsoft.com/tr-tr/ef/core/performance/efficient-querying) 