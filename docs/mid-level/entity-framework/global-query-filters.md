# Entity Framework - Global Query Filters

## Giriş

Entity Framework'te Global Query Filters (Küresel Sorgu Filtreleri), tüm sorgulara otomatik olarak uygulanan filtrelerdir. Mid-level geliştiriciler için bu kavramın anlaşılması ve etkin kullanımı önemlidir.

## Global Query Filters'ın Önemi

1. **Veri Yönetimi**
   - Tüm sorgulara otomatik filtre uygulama
   - Daha iyi veri organizasyonu
   - Daha iyi veri bütünlüğü
   - Daha iyi veri erişimi

2. **Güvenlik**
   - Veri erişimini kontrol etme
   - Hassas verileri filtreleme
   - Yetkilendirme
   - Güvenlik kontrolleri

3. **Bakım**
   - Daha az kod tekrarı
   - Daha kolay test edilebilirlik
   - Daha iyi modülerlik
   - Daha kolay genişletilebilirlik

## Global Query Filters Özellikleri

1. **Temel Global Query Filter**
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .HasQueryFilter(b => !b.IsDeleted);

    modelBuilder.Entity<Post>()
        .HasQueryFilter(p => p.IsPublished);
}
```

2. **Global Query Filter Kullanımı**
```csharp
public class BlogService
{
    private readonly DbContext _context;

    public BlogService(DbContext context)
    {
        _context = context;
    }

    public List<Blog> GetActiveBlogs()
    {
        return _context.Blogs.ToList();
    }

    public List<Blog> GetInactiveBlogs()
    {
        return _context.Blogs.IgnoreQueryFilters().ToList();
    }
}
```

3. **Global Query Filter Validasyonu**
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>(entity =>
    {
        entity.HasQueryFilter(b => !b.IsDeleted);
        entity.Property(b => b.IsDeleted).IsRequired();
    });

    modelBuilder.Entity<Post>(entity =>
    {
        entity.HasQueryFilter(p => p.IsPublished);
        entity.Property(p => p.IsPublished).IsRequired();
    });
}
```

## Global Query Filters Kullanımı

1. **Entity İçinde Kullanım**
```csharp
public class Blog
{
    public int Id { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }
    public bool IsDeleted { get; set; }
}

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }
    public bool IsPublished { get; set; }
}

public class BlogService
{
    private readonly DbContext _context;

    public BlogService(DbContext context)
    {
        _context = context;
    }

    public List<Blog> GetActiveBlogs()
    {
        return _context.Blogs.ToList();
    }

    public List<Blog> GetAllBlogs()
    {
        return _context.Blogs.IgnoreQueryFilters().ToList();
    }
}
```

2. **DbContext Konfigürasyonu**
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>(entity =>
    {
        entity.HasQueryFilter(b => !b.IsDeleted);
        entity.Property(b => b.IsDeleted).IsRequired();
    });

    modelBuilder.Entity<Post>(entity =>
    {
        entity.HasQueryFilter(p => p.IsPublished);
        entity.Property(p => p.IsPublished).IsRequired();
    });
}
```

3. **Global Query Filter Dönüşümleri**
```csharp
public static class BlogExtensions
{
    public static IQueryable<Blog> FilterByStatus(this IQueryable<Blog> query, bool includeDeleted)
    {
        if (includeDeleted)
        {
            return query.IgnoreQueryFilters();
        }
        return query;
    }
}
```

## Best Practices

1. **Global Query Filter Tasarımı**
   - Single Responsibility
   - Immutability
   - Validation
   - Business logic

2. **Güvenlik**
   - Input validation
   - Data integrity
   - Access control
   - Audit logging

3. **Performans**
   - Query optimization
   - Index kullanımı
   - Caching
   - Lazy loading

4. **Bakım**
   - Code organization
   - Documentation
   - Testing
   - Versioning

## Mülakat Soruları

### Temel Sorular

1. **Entity Framework'te Global Query Filter nedir?**
   - **Cevap**: Global Query Filter, tüm sorgulara otomatik olarak uygulanan filtrelerdir.

2. **Entity Framework'te Global Query Filter ve Normal Filter arasındaki fark nedir?**
   - **Cevap**: Normal Filter'lar her sorgu için ayrı ayrı uygulanır, Global Query Filter'lar tüm sorgulara otomatik olarak uygulanır.

3. **Entity Framework'te Global Query Filter nasıl konfigüre edilir?**
   - **Cevap**: DbContext.OnModelCreating metodunda HasQueryFilter metodu kullanılarak konfigüre edilir.

4. **Entity Framework'te Global Query Filter ne zaman kullanılır?**
   - **Cevap**: Tüm sorgulara otomatik filtre uygulamak, veri erişimini kontrol etmek veya hassas verileri filtrelemek için kullanılır.

5. **Entity Framework'te Global Query Filter performansı nasıl etkiler?**
   - **Cevap**: Query optimizasyonunu etkileyebilir ancak veri erişimini kontrol etmeyi kolaylaştırır.

### Teknik Sorular

1. **Temel Global Query Filter nasıl oluşturulur?**
   - **Cevap**:
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .HasQueryFilter(b => !b.IsDeleted);
}
```

2. **Global Query Filter nasıl kullanılır?**
   - **Cevap**:
```csharp
public List<Blog> GetActiveBlogs()
{
    return _context.Blogs.ToList();
}

public List<Blog> GetAllBlogs()
{
    return _context.Blogs.IgnoreQueryFilters().ToList();
}
```

3. **Global Query Filter validasyonu nasıl yapılır?**
   - **Cevap**:
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>(entity =>
    {
        entity.HasQueryFilter(b => !b.IsDeleted);
        entity.Property(b => b.IsDeleted).IsRequired();
    });
}
```

4. **Global Query Filter nasıl devre dışı bırakılır?**
   - **Cevap**:
```csharp
public List<Blog> GetAllBlogs()
{
    return _context.Blogs.IgnoreQueryFilters().ToList();
}
```

5. **Global Query Filter nasıl özelleştirilir?**
   - **Cevap**:
```csharp
public static class BlogExtensions
{
    public static IQueryable<Blog> FilterByStatus(this IQueryable<Blog> query, bool includeDeleted)
    {
        if (includeDeleted)
        {
            return query.IgnoreQueryFilters();
        }
        return query;
    }
}
```

### İleri Seviye Sorular

1. **Entity Framework'te Global Query Filter performansı nasıl optimize edilir?**
   - **Cevap**:
     - Query optimizasyonu
     - Index kullanımı
     - Caching stratejileri
     - Lazy loading
     - Materialization optimizasyonu

2. **Entity Framework'te distributed sistemlerde Global Query Filter nasıl yönetilir?**
   - **Cevap**:
     - Consistency
     - Replication
     - Sharding
     - Partitioning
     - Caching

3. **Entity Framework'te high concurrency senaryolarında Global Query Filter nasıl yönetilir?**
   - **Cevap**:
     - Thread safety
     - Atomic operations
     - Locking stratejileri
     - Isolation levels
     - Conflict resolution

4. **Entity Framework'te Global Query Filter monitoring ve profiling nasıl yapılır?**
   - **Cevap**:
     - Query profiling
     - Performance metrics
     - Resource monitoring
     - Profiling tools
     - Health checks

5. **Entity Framework'te custom Global Query Filter stratejileri nasıl geliştirilir?**
   - **Cevap**:
     - Custom filter handling
     - Custom validation
     - Custom optimization
     - Custom caching
     - Custom materialization 