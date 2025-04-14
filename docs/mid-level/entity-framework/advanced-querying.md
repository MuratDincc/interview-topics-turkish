# Entity Framework - Advanced Querying

## Giriş

Entity Framework'te ileri seviye sorgulama teknikleri, veritabanı işlemlerini daha verimli ve esnek bir şekilde gerçekleştirmeyi sağlar. Mid-level geliştiriciler için bu teknikler kritik öneme sahiptir.

## Advanced Querying'in Önemi

1. **Performans**
   - Daha verimli sorgular
   - Daha az veritabanı yükü
   - Daha iyi sorgu planları
   - Daha hızlı yanıt süreleri

2. **Esneklik**
   - Karmaşık sorgular
   - Dinamik sorgular
   - Özelleştirilmiş sorgular
   - Verimli veri filtreleme

3. **Bakım**
   - Daha temiz kod
   - Daha kolay debug
   - Daha iyi test edilebilirlik
   - Daha kolay bakım

## Advanced Querying Teknikleri

1. **LINQ to Entities**
```csharp
// Temel sorgu
var blogs = await _context.Blogs
    .Where(b => b.Rating > 3)
    .OrderByDescending(b => b.CreatedDate)
    .ToListAsync();

// Include ile ilişkili veriler
var blogs = await _context.Blogs
    .Include(b => b.Posts)
        .ThenInclude(p => p.Comments)
    .ToListAsync();

// Projection
var blogTitles = await _context.Blogs
    .Select(b => new BlogDto
    {
        Id = b.Id,
        Title = b.Title,
        PostCount = b.Posts.Count
    })
    .ToListAsync();

// Group by
var blogGroups = await _context.Blogs
    .GroupBy(b => b.Category)
    .Select(g => new
    {
        Category = g.Key,
        Count = g.Count(),
        AverageRating = g.Average(b => b.Rating)
    })
    .ToListAsync();
```

2. **Raw SQL Queries**
```csharp
// Temel raw SQL
var blogs = await _context.Blogs
    .FromSqlRaw("SELECT * FROM Blogs WHERE Rating > {0}", 3)
    .ToListAsync();

// Stored procedure
var blogs = await _context.Blogs
    .FromSqlRaw("EXEC GetTopRatedBlogs @MinRating = {0}", 3)
    .ToListAsync();

// View kullanımı
var blogStats = await _context.BlogStats
    .FromSqlRaw("SELECT * FROM vw_BlogStatistics")
    .ToListAsync();

// Parametreli sorgu
var blogs = await _context.Blogs
    .FromSqlInterpolated($"SELECT * FROM Blogs WHERE Rating > {minRating} AND Category = {category}")
    .ToListAsync();
```

3. **Compiled Queries**
```csharp
// Compiled query tanımlama
private static readonly Func<ApplicationDbContext, int, Task<Blog>> GetBlogById =
    EF.CompileAsyncQuery((ApplicationDbContext context, int id) =>
        context.Blogs.FirstOrDefault(b => b.Id == id));

// Compiled query kullanımı
var blog = await GetBlogById(_context, 1);

// Parametreli compiled query
private static readonly Func<ApplicationDbContext, int, string, Task<Blog>> GetBlogByTitle =
    EF.CompileAsyncQuery((ApplicationDbContext context, int id, string title) =>
        context.Blogs.FirstOrDefault(b => b.Id == id && b.Title == title));
```

4. **Dynamic Queries**
```csharp
// Dinamik sorgu oluşturma
public IQueryable<Blog> GetFilteredBlogs(BlogFilter filter)
{
    var query = _context.Blogs.AsQueryable();

    if (filter.Category != null)
    {
        query = query.Where(b => b.Category == filter.Category);
    }

    if (filter.MinRating.HasValue)
    {
        query = query.Where(b => b.Rating >= filter.MinRating.Value);
    }

    if (filter.StartDate.HasValue)
    {
        query = query.Where(b => b.CreatedDate >= filter.StartDate.Value);
    }

    return query;
}

// Expression kullanımı
public IQueryable<Blog> GetFilteredBlogs(Expression<Func<Blog, bool>> filter)
{
    return _context.Blogs.Where(filter);
}
```

5. **Complex Queries**
```csharp
// Subquery
var blogs = await _context.Blogs
    .Where(b => b.Posts.Any(p => p.Comments.Count > 10))
    .ToListAsync();

// Join
var blogPosts = await _context.Blogs
    .Join(_context.Posts,
        b => b.Id,
        p => p.BlogId,
        (b, p) => new { Blog = b, Post = p })
    .ToListAsync();

// Union
var allContent = await _context.Blogs
    .Select(b => new { Type = "Blog", Title = b.Title })
    .Union(_context.Posts.Select(p => new { Type = "Post", Title = p.Title }))
    .ToListAsync();
```

## Best Practices

1. **Query Tasarımı**
   - Eager loading kullanımı
   - Projection kullanımı
   - Compiled query kullanımı
   - Batch processing
   - Pagination

2. **Performans**
   - Index kullanımı
   - Query optimizasyonu
   - Caching stratejileri
   - Batch processing
   - Resource yönetimi

3. **Güvenlik**
   - SQL injection önleme
   - Parametre kullanımı
   - Input validation
   - Access control
   - Audit logging

4. **Bakım**
   - Kod organizasyonu
   - Documentation
   - Testing
   - Version control
   - Error handling

## Mülakat Soruları

### Temel Sorular

1. **Entity Framework'te LINQ to Entities nedir?**
   - **Cevap**: LINQ to Entities, Entity Framework'te veritabanı sorgularını nesne yönelimli bir şekilde yazmayı sağlayan bir teknolojidir.

2. **Entity Framework'te raw SQL sorguları ne zaman kullanılır?**
   - **Cevap**: Karmaşık sorgular, stored procedure'ler, view'lar veya performans gerektiren durumlarda kullanılır.

3. **Entity Framework'te compiled query nedir?**
   - **Cevap**: Compiled query, sorgunun bir kez derlenip tekrar tekrar kullanılmasını sağlayan bir optimizasyon tekniğidir.

4. **Entity Framework'te dynamic query nedir?**
   - **Cevap**: Runtime'da oluşturulan ve değişen sorgulardır. Expression'lar veya IQueryable kullanılarak oluşturulur.

5. **Entity Framework'te complex query nedir?**
   - **Cevap**: Subquery, join, union gibi karmaşık veritabanı işlemlerini içeren sorgulardır.

### Teknik Sorular

1. **LINQ to Entities ile nasıl karmaşık sorgu yazılır?**
   - **Cevap**:
```csharp
var blogs = await _context.Blogs
    .Include(b => b.Posts)
        .ThenInclude(p => p.Comments)
    .Where(b => b.Rating > 3)
    .GroupBy(b => b.Category)
    .Select(g => new
    {
        Category = g.Key,
        Count = g.Count(),
        AverageRating = g.Average(b => b.Rating)
    })
    .ToListAsync();
```

2. **Raw SQL sorguları nasıl yazılır?**
   - **Cevap**:
```csharp
var blogs = await _context.Blogs
    .FromSqlRaw("SELECT * FROM Blogs WHERE Rating > {0}", 3)
    .ToListAsync();

var blogs = await _context.Blogs
    .FromSqlInterpolated($"SELECT * FROM Blogs WHERE Rating > {minRating}")
    .ToListAsync();
```

3. **Compiled query nasıl oluşturulur?**
   - **Cevap**:
```csharp
private static readonly Func<ApplicationDbContext, int, Task<Blog>> GetBlogById =
    EF.CompileAsyncQuery((ApplicationDbContext context, int id) =>
        context.Blogs.FirstOrDefault(b => b.Id == id));

var blog = await GetBlogById(_context, 1);
```

4. **Dynamic query nasıl oluşturulur?**
   - **Cevap**:
```csharp
public IQueryable<Blog> GetFilteredBlogs(BlogFilter filter)
{
    var query = _context.Blogs.AsQueryable();

    if (filter.Category != null)
    {
        query = query.Where(b => b.Category == filter.Category);
    }

    if (filter.MinRating.HasValue)
    {
        query = query.Where(b => b.Rating >= filter.MinRating.Value);
    }

    return query;
}
```

5. **Complex query nasıl yazılır?**
   - **Cevap**:
```csharp
var blogs = await _context.Blogs
    .Where(b => b.Posts.Any(p => p.Comments.Count > 10))
    .Join(_context.Posts,
        b => b.Id,
        p => p.BlogId,
        (b, p) => new { Blog = b, Post = p })
    .ToListAsync();
```

### İleri Seviye Sorular

1. **Entity Framework'te query plan cache nasıl yönetilir?**
   - **Cevap**:
     - Query plan cache stratejileri
     - Plan cache invalidation
     - Plan cache monitoring
     - Plan cache optimization
     - Plan cache troubleshooting

2. **Entity Framework'te query performansı nasıl optimize edilir?**
   - **Cevap**:
     - Index kullanımı
     - Query optimizasyonu
     - Caching stratejileri
     - Batch processing
     - Resource yönetimi

3. **Entity Framework'te distributed sistemlerde query yönetimi nasıl yapılır?**
   - **Cevap**:
     - Query routing
     - Load balancing
     - Data partitioning
     - Replication
     - Consistency

4. **Entity Framework'te high concurrency senaryolarında query yönetimi nasıl yapılır?**
   - **Cevap**:
     - Optimistic concurrency
     - Pessimistic concurrency
     - Retry mekanizmaları
     - Queue yönetimi
     - Batch processing

5. **Entity Framework'te query monitoring ve profiling nasıl yapılır?**
   - **Cevap**:
     - Query logging
     - Performance metrics
     - Resource monitoring
     - Profiling tools
     - Health checks 