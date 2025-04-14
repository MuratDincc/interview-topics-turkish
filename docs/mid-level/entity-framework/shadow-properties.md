# Entity Framework - Shadow Properties

## Giriş

Entity Framework'te Shadow Properties (Gölge Özellikler), entity sınıfında tanımlanmayan ancak veritabanında bulunan özelliklerdir. Mid-level geliştiriciler için bu kavramın anlaşılması ve etkin kullanımı önemlidir.

## Shadow Properties'ın Önemi

1. **Veri Yönetimi**
   - Veritabanı kolonlarını entity'den gizleme
   - Daha iyi veri organizasyonu
   - Daha iyi veri bütünlüğü
   - Daha iyi veri erişimi

2. **Güvenlik**
   - Hassas verileri gizleme
   - Veri erişimini kontrol etme
   - Audit logging
   - Güvenlik kontrolleri

3. **Bakım**
   - Daha az kod tekrarı
   - Daha kolay test edilebilirlik
   - Daha iyi modülerlik
   - Daha kolay genişletilebilirlik

## Shadow Properties Özellikleri

1. **Temel Shadow Property**
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .Property<DateTime>("CreatedDate")
        .HasDefaultValueSql("GETDATE()");

    modelBuilder.Entity<Blog>()
        .Property<DateTime>("LastModifiedDate")
        .HasDefaultValueSql("GETDATE()");
}
```

2. **Shadow Property Kullanımı**
```csharp
public class BlogService
{
    private readonly DbContext _context;

    public BlogService(DbContext context)
    {
        _context = context;
    }

    public void UpdateBlog(int id, string title)
    {
        var blog = _context.Blogs.Find(id);
        if (blog != null)
        {
            blog.Title = title;
            _context.Entry(blog).Property("LastModifiedDate").CurrentValue = DateTime.UtcNow;
            _context.SaveChanges();
        }
    }

    public DateTime GetBlogCreatedDate(int id)
    {
        var blog = _context.Blogs.Find(id);
        return _context.Entry(blog).Property("CreatedDate").CurrentValue;
    }
}
```

3. **Shadow Property Validasyonu**
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .Property<DateTime>("CreatedDate")
        .IsRequired()
        .HasDefaultValueSql("GETDATE()");

    modelBuilder.Entity<Blog>()
        .Property<DateTime>("LastModifiedDate")
        .IsRequired()
        .HasDefaultValueSql("GETDATE()");

    modelBuilder.Entity<Blog>()
        .Property<string>("CreatedBy")
        .IsRequired()
        .HasMaxLength(50);
}
```

## Shadow Properties Kullanımı

1. **Entity İçinde Kullanım**
```csharp
public class Blog
{
    public int Id { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }
}

public class BlogService
{
    private readonly DbContext _context;

    public BlogService(DbContext context)
    {
        _context = context;
    }

    public void CreateBlog(string title, string content, string createdBy)
    {
        var blog = new Blog
        {
            Title = title,
            Content = content
        };

        _context.Add(blog);
        _context.Entry(blog).Property("CreatedBy").CurrentValue = createdBy;
        _context.SaveChanges();
    }

    public void UpdateBlog(int id, string title)
    {
        var blog = _context.Blogs.Find(id);
        if (blog != null)
        {
            blog.Title = title;
            _context.Entry(blog).Property("LastModifiedDate").CurrentValue = DateTime.UtcNow;
            _context.SaveChanges();
        }
    }
}
```

2. **DbContext Konfigürasyonu**
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>(entity =>
    {
        entity.Property<DateTime>("CreatedDate")
            .IsRequired()
            .HasDefaultValueSql("GETDATE()");

        entity.Property<DateTime>("LastModifiedDate")
            .IsRequired()
            .HasDefaultValueSql("GETDATE()");

        entity.Property<string>("CreatedBy")
            .IsRequired()
            .HasMaxLength(50);

        entity.Property<string>("LastModifiedBy")
            .IsRequired()
            .HasMaxLength(50);
    });
}
```

3. **Shadow Property Dönüşümleri**
```csharp
public static class BlogExtensions
{
    public static BlogDto ToDto(this Blog blog, DbContext context)
    {
        return new BlogDto
        {
            Id = blog.Id,
            Title = blog.Title,
            Content = blog.Content,
            CreatedDate = context.Entry(blog).Property<DateTime>("CreatedDate").CurrentValue,
            CreatedBy = context.Entry(blog).Property<string>("CreatedBy").CurrentValue,
            LastModifiedDate = context.Entry(blog).Property<DateTime>("LastModifiedDate").CurrentValue,
            LastModifiedBy = context.Entry(blog).Property<string>("LastModifiedBy").CurrentValue
        };
    }
}
```

## Best Practices

1. **Shadow Property Tasarımı**
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
   - Memory usage
   - Query optimization
   - Lazy loading
   - Caching

4. **Bakım**
   - Code organization
   - Documentation
   - Testing
   - Versioning

## Mülakat Soruları

### Temel Sorular

1. **Entity Framework'te Shadow Property nedir?**
   - **Cevap**: Shadow Property, entity sınıfında tanımlanmayan ancak veritabanında bulunan özelliklerdir.

2. **Entity Framework'te Shadow Property ve Normal Property arasındaki fark nedir?**
   - **Cevap**: Normal Property'ler entity sınıfında tanımlanır ve doğrudan erişilebilir, Shadow Property'ler entity sınıfında tanımlanmaz ve DbContext üzerinden erişilir.

3. **Entity Framework'te Shadow Property nasıl konfigüre edilir?**
   - **Cevap**: DbContext.OnModelCreating metodunda Property<T> metodu kullanılarak konfigüre edilir.

4. **Entity Framework'te Shadow Property ne zaman kullanılır?**
   - **Cevap**: Veritabanı kolonlarını entity'den gizlemek, hassas verileri gizlemek veya audit logging için kullanılır.

5. **Entity Framework'te Shadow Property performansı nasıl etkiler?**
   - **Cevap**: Memory kullanımını azaltabilir ancak erişim için ek kod gerektirir.

### Teknik Sorular

1. **Temel Shadow Property nasıl oluşturulur?**
   - **Cevap**:
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .Property<DateTime>("CreatedDate")
        .HasDefaultValueSql("GETDATE()");
}
```

2. **Shadow Property nasıl kullanılır?**
   - **Cevap**:
```csharp
public void UpdateBlog(int id, string title)
{
    var blog = _context.Blogs.Find(id);
    if (blog != null)
    {
        blog.Title = title;
        _context.Entry(blog).Property("LastModifiedDate").CurrentValue = DateTime.UtcNow;
        _context.SaveChanges();
    }
}
```

3. **Shadow Property validasyonu nasıl yapılır?**
   - **Cevap**:
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .Property<string>("CreatedBy")
        .IsRequired()
        .HasMaxLength(50);
}
```

4. **Shadow Property değeri nasıl okunur?**
   - **Cevap**:
```csharp
public DateTime GetBlogCreatedDate(int id)
{
    var blog = _context.Blogs.Find(id);
    return _context.Entry(blog).Property("CreatedDate").CurrentValue;
}
```

5. **Shadow Property değeri nasıl güncellenir?**
   - **Cevap**:
```csharp
public void UpdateBlog(int id, string title)
{
    var blog = _context.Blogs.Find(id);
    if (blog != null)
    {
        blog.Title = title;
        _context.Entry(blog).Property("LastModifiedDate").CurrentValue = DateTime.UtcNow;
        _context.SaveChanges();
    }
}
```

### İleri Seviye Sorular

1. **Entity Framework'te Shadow Property performansı nasıl optimize edilir?**
   - **Cevap**:
     - Memory kullanımı optimizasyonu
     - Query optimizasyonu
     - Lazy loading
     - Caching stratejileri
     - Index kullanımı

2. **Entity Framework'te distributed sistemlerde Shadow Property nasıl yönetilir?**
   - **Cevap**:
     - Serialization
     - Versioning
     - Migration
     - Consistency
     - Conflict resolution

3. **Entity Framework'te high concurrency senaryolarında Shadow Property nasıl yönetilir?**
   - **Cevap**:
     - Immutability
     - Thread safety
     - Atomic operations
     - Versioning
     - Conflict resolution

4. **Entity Framework'te Shadow Property monitoring ve profiling nasıl yapılır?**
   - **Cevap**:
     - Memory profiling
     - Performance metrics
     - Resource monitoring
     - Profiling tools
     - Health checks

5. **Entity Framework'te custom Shadow Property stratejileri nasıl geliştirilir?**
   - **Cevap**:
     - Custom property handling
     - Custom validation
     - Custom serialization
     - Custom conversion
     - Custom optimization 