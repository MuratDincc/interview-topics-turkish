# Veritabanı Optimizasyonu

## Genel Bakış
Veritabanı optimizasyonu, veritabanı sorgularının ve işlemlerinin daha hızlı ve verimli çalışması için yapılan iyileştirmelerdir. Bu optimizasyonlar, sorgu performansını artırır, kaynak kullanımını azaltır ve sistem ölçeklenebilirliğini iyileştirir.

## Temel Kavramlar

### 1. İndeksleme
```csharp
// Entity Framework Core'da indeks tanımlama
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Category { get; set; }
    public decimal Price { get; set; }
    public DateTime CreatedDate { get; set; }
}

public class ApplicationDbContext : DbContext
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Tek kolon indeksi
        modelBuilder.Entity<Product>()
            .HasIndex(p => p.Name);

        // Bileşik indeks
        modelBuilder.Entity<Product>()
            .HasIndex(p => new { p.Category, p.Price });

        // Benzersiz indeks
        modelBuilder.Entity<Product>()
            .HasIndex(p => p.Name)
            .IsUnique();

        // Filtrelenmiş indeks
        modelBuilder.Entity<Product>()
            .HasIndex(p => p.Price)
            .HasFilter("[Price] > 1000");
    }
}
```

### 2. Sorgu Optimizasyonu
```csharp
public class ProductRepository
{
    private readonly ApplicationDbContext _context;
    private readonly ILogger<ProductRepository> _logger;

    public ProductRepository(
        ApplicationDbContext context,
        ILogger<ProductRepository> logger)
    {
        _context = context;
        _logger = logger;
    }

    // N+1 problemi çözümü
    public async Task<List<Product>> GetProductsWithCategoryAsync()
    {
        return await _context.Products
            .Include(p => p.Category)  // Eager loading
            .ToListAsync();
    }

    // Select N+1 problemi çözümü
    public async Task<List<ProductDto>> GetProductDtosAsync()
    {
        return await _context.Products
            .Select(p => new ProductDto
            {
                Id = p.Id,
                Name = p.Name,
                CategoryName = p.Category.Name
            })
            .ToListAsync();
    }

    // Batch işlem örneği
    public async Task BulkInsertProductsAsync(List<Product> products)
    {
        await _context.BulkInsertAsync(products, options =>
        {
            options.BatchSize = 1000;
            options.AutoMapOutputDirection = false;
        });
    }
}
```

### 3. Bağlantı Yönetimi
```csharp
public class DatabaseConnectionManager
{
    private readonly string _connectionString;
    private readonly ILogger<DatabaseConnectionManager> _logger;

    public DatabaseConnectionManager(
        IConfiguration configuration,
        ILogger<DatabaseConnectionManager> logger)
    {
        _connectionString = configuration.GetConnectionString("DefaultConnection");
        _logger = logger;
    }

    public async Task<T> ExecuteWithRetryAsync<T>(Func<IDbConnection, Task<T>> operation)
    {
        var policy = Policy
            .Handle<SqlException>()
            .WaitAndRetryAsync(3, retryAttempt => 
                TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
                (exception, timeSpan, retryCount, context) =>
                {
                    _logger.LogWarning(
                        exception,
                        "Retry {RetryCount} after {TimeSpan} due to {Exception}",
                        retryCount,
                        timeSpan,
                        exception.Message);
                });

        return await policy.ExecuteAsync(async () =>
        {
            using var connection = new SqlConnection(_connectionString);
            await connection.OpenAsync();
            return await operation(connection);
        });
    }
}
```

## Best Practices

### 1. İndeks Tasarımı
- Sık sorgulanan kolonları indeksleyin
- Bileşik indeksleri doğru sırada oluşturun
- Gereksiz indekslerden kaçının
- İndeks bakımını düzenli yapın
- İndeks fragmentasyonunu izleyin

### 2. Sorgu Optimizasyonu
- N+1 problemini önleyin
- Gereksiz kolon seçiminden kaçının
- JOIN'leri optimize edin
- Subquery'leri dikkatli kullanın
- Batch işlemleri kullanın

### 3. Bağlantı Yönetimi
- Bağlantı havuzunu yapılandırın
- Bağlantı sürelerini optimize edin
- Retry mekanizması kullanın
- Bağlantı hatalarını yönetin
- Bağlantı izleme yapın

## Sık Sorulan Sorular

### 1. İndeksleme ne zaman kullanılmalıdır?
- Sık sorgulanan kolonlar için
- JOIN işlemlerinde kullanılan kolonlar için
- WHERE koşullarında kullanılan kolonlar için
- ORDER BY ve GROUP BY işlemlerinde kullanılan kolonlar için
- Benzersiz değer gerektiren kolonlar için

### 2. Sorgu performansı nasıl iyileştirilir?
- Execution plan analizi yapın
- İndeks kullanımını optimize edin
- Sorgu yapısını basitleştirin
- Gereksiz JOIN'lerden kaçının
- Batch işlemleri kullanın

### 3. Veritabanı bağlantıları nasıl yönetilir?
- Bağlantı havuzu boyutunu ayarlayın
- Bağlantı sürelerini yapılandırın
- Retry politikaları uygulayın
- Bağlantı hatalarını izleyin
- Bağlantı kaynaklarını temizleyin

## Kaynaklar
- [SQL Server Performance Tuning](https://docs.microsoft.com/tr-tr/sql/relational-databases/performance/sql-server-performance-tuning)
- [Entity Framework Core Performance](https://docs.microsoft.com/tr-tr/ef/core/performance/)
- [Database Connection Pooling](https://docs.microsoft.com/tr-tr/ef/core/miscellaneous/connection-pooling)
- [SQL Server Index Architecture](https://docs.microsoft.com/tr-tr/sql/relational-databases/indexes/indexes) 