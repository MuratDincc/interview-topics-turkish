# Entity Framework - Performance Optimization

## Giriş

Entity Framework'te performans optimizasyonu, uygulamanın veritabanı işlemlerini daha hızlı ve verimli bir şekilde gerçekleştirmesini sağlar. Mid-level geliştiriciler için performans optimizasyonu kritik bir konudur.

## Performance Optimization'un Önemi

1. **Uygulama Performansı**
   - Daha hızlı sorgu yanıt süreleri
   - Daha az kaynak kullanımı
   - Daha iyi kullanıcı deneyimi
   - Daha yüksek ölçeklenebilirlik

2. **Veritabanı Yükü**
   - Daha az veritabanı yükü
   - Daha verimli sorgu planları
   - Daha az network trafiği
   - Daha iyi kaynak yönetimi

3. **Maliyet Optimizasyonu**
   - Daha az donanım gereksinimi
   - Daha düşük işlem maliyetleri
   - Daha verimli kaynak kullanımı
   - Daha iyi ROI

## Performance Optimization Stratejileri

1. **Query Optimizasyonu**
```csharp
// Kötü örnek: N+1 problemi
var blogs = await _context.Blogs.ToListAsync();
foreach (var blog in blogs)
{
    var posts = await _context.Posts.Where(p => p.BlogId == blog.Id).ToListAsync();
}

// İyi örnek: Eager loading
var blogs = await _context.Blogs
    .Include(b => b.Posts)
    .ToListAsync();

// İyi örnek: Projection
var blogTitles = await _context.Blogs
    .Select(b => new { b.Id, b.Title })
    .ToListAsync();

// İyi örnek: Compiled query
private static readonly Func<ApplicationDbContext, int, Task<Blog>> GetBlogById =
    EF.CompileAsyncQuery((ApplicationDbContext context, int id) =>
        context.Blogs.FirstOrDefault(b => b.Id == id));
```

2. **Change Tracking Optimizasyonu**
```csharp
// Kötü örnek: Gereksiz change tracking
var blogs = await _context.Blogs.AsTracking().ToListAsync();

// İyi örnek: No tracking
var blogs = await _context.Blogs.AsNoTracking().ToListAsync();

// İyi örnek: Selective tracking
var blog = await _context.Blogs
    .AsNoTracking()
    .FirstOrDefaultAsync(b => b.Id == id);
_context.Entry(blog).State = EntityState.Modified;
```

3. **Bulk Operasyonlar**
```csharp
// Kötü örnek: Tek tek ekleme
foreach (var entity in entities)
{
    await _context.AddAsync(entity);
    await _context.SaveChangesAsync();
}

// İyi örnek: Bulk insert
await _context.BulkInsertAsync(entities);

// İyi örnek: Batch processing
var batchSize = 100;
for (var i = 0; i < entities.Count; i += batchSize)
{
    var batch = entities.Skip(i).Take(batchSize);
    await _context.BulkInsertAsync(batch);
}
```

4. **Caching Stratejileri**
```csharp
// Memory cache kullanımı
public class CachedRepository<T> where T : class
{
    private readonly IMemoryCache _cache;
    private readonly ApplicationDbContext _context;
    private readonly string _cacheKey;

    public CachedRepository(IMemoryCache cache, ApplicationDbContext context)
    {
        _cache = cache;
        _context = context;
        _cacheKey = typeof(T).Name;
    }

    public async Task<T> GetByIdAsync(int id)
    {
        var cacheKey = $"{_cacheKey}-{id}";
        return await _cache.GetOrCreateAsync(cacheKey, async entry =>
        {
            entry.SetAbsoluteExpiration(TimeSpan.FromMinutes(5));
            return await _context.Set<T>().FindAsync(id);
        });
    }
}

// Distributed cache kullanımı
public class DistributedCachedRepository<T> where T : class
{
    private readonly IDistributedCache _cache;
    private readonly ApplicationDbContext _context;
    private readonly string _cacheKey;

    public DistributedCachedRepository(IDistributedCache cache, ApplicationDbContext context)
    {
        _cache = cache;
        _context = context;
        _cacheKey = typeof(T).Name;
    }

    public async Task<T> GetByIdAsync(int id)
    {
        var cacheKey = $"{_cacheKey}-{id}";
        var cachedData = await _cache.GetStringAsync(cacheKey);
        
        if (cachedData != null)
        {
            return JsonSerializer.Deserialize<T>(cachedData);
        }

        var data = await _context.Set<T>().FindAsync(id);
        await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(data), new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
        });

        return data;
    }
}
```

5. **Index Yönetimi**
```csharp
// Index tanımlama
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .HasIndex(b => b.Title)
        .IsUnique();

    modelBuilder.Entity<Post>()
        .HasIndex(p => new { p.BlogId, p.PublishedDate })
        .IncludeProperties(p => new { p.Title, p.Content });
}

// Index kullanımı
var blogs = await _context.Blogs
    .Where(b => b.Title.StartsWith("A"))
    .OrderBy(b => b.Title)
    .ToListAsync();
```

## Performance Monitoring

1. **Query Logging**
```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder
        .UseSqlServer(connectionString)
        .EnableSensitiveDataLogging()
        .LogTo(Console.WriteLine, LogLevel.Information);
}
```

2. **Performance Metrics**
```csharp
public class PerformanceMonitor
{
    private readonly Stopwatch _stopwatch = new Stopwatch();

    public async Task<T> MeasureQuery<T>(Func<Task<T>> query)
    {
        _stopwatch.Start();
        var result = await query();
        _stopwatch.Stop();
        
        Console.WriteLine($"Query executed in {_stopwatch.ElapsedMilliseconds}ms");
        _stopwatch.Reset();
        
        return result;
    }
}
```

## Best Practices

1. **Query Tasarımı**
   - Eager loading kullanımı
   - Projection kullanımı
   - Compiled query kullanımı
   - Batch processing
   - Pagination

2. **Change Tracking**
   - No tracking kullanımı
   - Selective tracking
   - Bulk operasyonlar
   - Batch updates
   - Optimistic concurrency

3. **Caching**
   - Memory cache
   - Distributed cache
   - Cache invalidation
   - Cache duration
   - Cache size

4. **Indexing**
   - Index tasarımı
   - Index maintenance
   - Index fragmentation
   - Index statistics
   - Query plan analysis

## Mülakat Soruları

### Temel Sorular

1. **Entity Framework'te N+1 problemi nedir ve nasıl çözülür?**
   - **Cevap**: N+1 problemi, bir ana sorgu ve her bir kayıt için ayrı bir sorgu yapılması durumudur. Eager loading, projection veya compiled query kullanılarak çözülebilir.

2. **Entity Framework'te change tracking nedir ve nasıl optimize edilir?**
   - **Cevap**: Change tracking, entity'lerdeki değişikliklerin takip edilmesi sürecidir. AsNoTracking() kullanarak veya selective tracking ile optimize edilebilir.

3. **Entity Framework'te bulk operasyonlar nasıl yapılır?**
   - **Cevap**: Bulk operasyonlar, toplu veri işlemleri için kullanılır. BulkInsert, BulkUpdate, BulkDelete gibi metodlar veya batch processing ile yapılabilir.

4. **Entity Framework'te caching stratejileri nelerdir?**
   - **Cevap**: Memory cache, distributed cache, query result caching gibi stratejiler kullanılabilir.

5. **Entity Framework'te index yönetimi nasıl yapılır?**
   - **Cevap**: OnModelCreating metodunda HasIndex kullanılarak veya migration'lar ile yapılabilir.

### Teknik Sorular

1. **Query optimizasyonu için hangi stratejileri kullanırsınız?**
   - **Cevap**:
```csharp
// Eager loading
var blogs = await _context.Blogs
    .Include(b => b.Posts)
    .ToListAsync();

// Projection
var blogTitles = await _context.Blogs
    .Select(b => new { b.Id, b.Title })
    .ToListAsync();

// Compiled query
private static readonly Func<ApplicationDbContext, int, Task<Blog>> GetBlogById =
    EF.CompileAsyncQuery((ApplicationDbContext context, int id) =>
        context.Blogs.FirstOrDefault(b => b.Id == id));
```

2. **Change tracking optimizasyonu nasıl yapılır?**
   - **Cevap**:
```csharp
// No tracking
var blogs = await _context.Blogs.AsNoTracking().ToListAsync();

// Selective tracking
var blog = await _context.Blogs
    .AsNoTracking()
    .FirstOrDefaultAsync(b => b.Id == id);
_context.Entry(blog).State = EntityState.Modified;
```

3. **Bulk operasyonlar nasıl yapılır?**
   - **Cevap**:
```csharp
// Bulk insert
await _context.BulkInsertAsync(entities);

// Batch processing
var batchSize = 100;
for (var i = 0; i < entities.Count; i += batchSize)
{
    var batch = entities.Skip(i).Take(batchSize);
    await _context.BulkInsertAsync(batch);
}
```

4. **Caching nasıl implemente edilir?**
   - **Cevap**:
```csharp
public class CachedRepository<T> where T : class
{
    private readonly IMemoryCache _cache;
    private readonly ApplicationDbContext _context;

    public async Task<T> GetByIdAsync(int id)
    {
        var cacheKey = $"{typeof(T).Name}-{id}";
        return await _cache.GetOrCreateAsync(cacheKey, async entry =>
        {
            entry.SetAbsoluteExpiration(TimeSpan.FromMinutes(5));
            return await _context.Set<T>().FindAsync(id);
        });
    }
}
```

5. **Index yönetimi nasıl yapılır?**
   - **Cevap**:
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .HasIndex(b => b.Title)
        .IsUnique();

    modelBuilder.Entity<Post>()
        .HasIndex(p => new { p.BlogId, p.PublishedDate })
        .IncludeProperties(p => new { p.Title, p.Content });
}
```

### İleri Seviye Sorular

1. **Entity Framework'te query plan cache nasıl yönetilir?**
   - **Cevap**:
     - Query plan cache stratejileri
     - Plan cache invalidation
     - Plan cache monitoring
     - Plan cache optimization
     - Plan cache troubleshooting

2. **Entity Framework'te memory leak nasıl önlenir?**
   - **Cevap**:
     - Change tracking yönetimi
     - Connection pooling
     - Resource cleanup
     - Memory profiling
     - Garbage collection

3. **Entity Framework'te distributed sistemlerde performans nasıl optimize edilir?**
   - **Cevap**:
     - Caching stratejileri
     - Query routing
     - Load balancing
     - Data partitioning
     - Replication

4. **Entity Framework'te high concurrency senaryolarında performans nasıl optimize edilir?**
   - **Cevap**:
     - Optimistic concurrency
     - Pessimistic concurrency
     - Retry mekanizmaları
     - Queue yönetimi
     - Batch processing

5. **Entity Framework'te monitoring ve profiling nasıl yapılır?**
   - **Cevap**:
     - Query logging
     - Performance metrics
     - Resource monitoring
     - Profiling tools
     - Health checks 