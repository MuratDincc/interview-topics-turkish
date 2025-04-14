# Entity Framework - Raw SQL

## Giriş

Entity Framework'te Raw SQL (Ham SQL), LINQ sorguları yerine doğrudan SQL sorguları yazmayı ve çalıştırmayı sağlayan bir özelliktir. Mid-level geliştiriciler için bu özelliğin anlaşılması ve etkin kullanımı kritik öneme sahiptir.

## Raw SQL'in Önemi

1. **Performans**
   - Daha hızlı sorgu çalıştırma
   - Daha az kaynak kullanımı
   - Daha iyi sorgu optimizasyonu
   - Karmaşık sorgular için uygunluk

2. **Esneklik**
   - Özel SQL özellikleri kullanımı
   - Stored procedure'ler
   - View'lar
   - Database-specific özellikler

3. **Bakım**
   - Daha az kod
   - Daha kolay debug
   - Daha iyi test edilebilirlik
   - Daha kolay bakım

## Raw SQL Teknikleri

1. **FromSqlRaw**
```csharp
// Temel sorgu
var blogs = await _context.Blogs
    .FromSqlRaw("SELECT * FROM Blogs WHERE Rating > {0}", 3)
    .ToListAsync();

// Parametreli sorgu
var blogs = await _context.Blogs
    .FromSqlRaw("SELECT * FROM Blogs WHERE Rating > {0} AND Category = {1}", 3, "Technology")
    .ToListAsync();

// Stored procedure
var blogs = await _context.Blogs
    .FromSqlRaw("EXEC GetTopRatedBlogs @MinRating = {0}", 3)
    .ToListAsync();

// View kullanımı
var blogStats = await _context.BlogStats
    .FromSqlRaw("SELECT * FROM vw_BlogStatistics")
    .ToListAsync();
```

2. **FromSqlInterpolated**
```csharp
// String interpolation ile
var minRating = 3;
var category = "Technology";
var blogs = await _context.Blogs
    .FromSqlInterpolated($"SELECT * FROM Blogs WHERE Rating > {minRating} AND Category = {category}")
    .ToListAsync();

// Parametreli stored procedure
var blogs = await _context.Blogs
    .FromSqlInterpolated($"EXEC GetTopRatedBlogs @MinRating = {minRating}")
    .ToListAsync();
```

3. **ExecuteSqlRaw**
```csharp
// Temel sorgu
await _context.Database.ExecuteSqlRawAsync(
    "UPDATE Blogs SET Rating = Rating + 1 WHERE Category = {0}", "Technology");

// Parametreli sorgu
await _context.Database.ExecuteSqlRawAsync(
    "UPDATE Blogs SET Rating = Rating + 1 WHERE Category = {0} AND Rating < {1}", 
    "Technology", 5);

// Stored procedure
await _context.Database.ExecuteSqlRawAsync(
    "EXEC UpdateBlogRatings @Category = {0}, @Increment = {1}", 
    "Technology", 1);
```

4. **ExecuteSqlInterpolated**
```csharp
// String interpolation ile
var category = "Technology";
var increment = 1;
await _context.Database.ExecuteSqlInterpolatedAsync(
    $"UPDATE Blogs SET Rating = Rating + {increment} WHERE Category = {category}");

// Parametreli stored procedure
await _context.Database.ExecuteSqlInterpolatedAsync(
    $"EXEC UpdateBlogRatings @Category = {category}, @Increment = {increment}");
```

5. **Raw SQL ile LINQ**
```csharp
// Raw SQL + LINQ
var blogs = await _context.Blogs
    .FromSqlRaw("SELECT * FROM Blogs WHERE Rating > {0}", 3)
    .Where(b => b.Category == "Technology")
    .OrderByDescending(b => b.CreatedDate)
    .ToListAsync();

// Raw SQL + Include
var blogs = await _context.Blogs
    .FromSqlRaw("SELECT * FROM Blogs WHERE Rating > {0}", 3)
    .Include(b => b.Posts)
    .ToListAsync();
```

## Best Practices

1. **Raw SQL Tasarımı**
   - SQL injection önleme
   - Parametre kullanımı
   - Query optimizasyonu
   - Error handling

2. **Performans**
   - Query optimizasyonu
   - Index kullanımı
   - Batch processing
   - Resource yönetimi

3. **Güvenlik**
   - SQL injection önleme
   - Parametre kullanımı
   - Access control
   - Audit logging

4. **Bakım**
   - Kod organizasyonu
   - Documentation
   - Testing
   - Monitoring

## Mülakat Soruları

### Temel Sorular

1. **Entity Framework'te Raw SQL nedir?**
   - **Cevap**: Raw SQL, LINQ sorguları yerine doğrudan SQL sorguları yazmayı ve çalıştırmayı sağlayan bir özelliktir.

2. **Entity Framework'te FromSqlRaw nedir?**
   - **Cevap**: FromSqlRaw, entity'leri döndüren SQL sorgularını çalıştırmak için kullanılan bir metottur.

3. **Entity Framework'te ExecuteSqlRaw nedir?**
   - **Cevap**: ExecuteSqlRaw, sonuç döndürmeyen SQL sorgularını çalıştırmak için kullanılan bir metottur.

4. **Entity Framework'te SQL injection nasıl önlenir?**
   - **Cevap**: Parametre kullanımı ve FromSqlInterpolated/ExecuteSqlInterpolated kullanılarak önlenir.

5. **Entity Framework'te Raw SQL ne zaman kullanılır?**
   - **Cevap**: Karmaşık sorgular, stored procedure'ler, view'lar veya performans gerektiren durumlarda kullanılır.

### Teknik Sorular

1. **FromSqlRaw nasıl kullanılır?**
   - **Cevap**:
```csharp
var blogs = await _context.Blogs
    .FromSqlRaw("SELECT * FROM Blogs WHERE Rating > {0}", 3)
    .ToListAsync();
```

2. **FromSqlInterpolated nasıl kullanılır?**
   - **Cevap**:
```csharp
var minRating = 3;
var blogs = await _context.Blogs
    .FromSqlInterpolated($"SELECT * FROM Blogs WHERE Rating > {minRating}")
    .ToListAsync();
```

3. **ExecuteSqlRaw nasıl kullanılır?**
   - **Cevap**:
```csharp
await _context.Database.ExecuteSqlRawAsync(
    "UPDATE Blogs SET Rating = Rating + 1 WHERE Category = {0}", "Technology");
```

4. **Raw SQL ile LINQ nasıl birleştirilir?**
   - **Cevap**:
```csharp
var blogs = await _context.Blogs
    .FromSqlRaw("SELECT * FROM Blogs WHERE Rating > {0}", 3)
    .Where(b => b.Category == "Technology")
    .OrderByDescending(b => b.CreatedDate)
    .ToListAsync();
```

5. **Stored procedure nasıl çağrılır?**
   - **Cevap**:
```csharp
var blogs = await _context.Blogs
    .FromSqlRaw("EXEC GetTopRatedBlogs @MinRating = {0}", 3)
    .ToListAsync();
```

### İleri Seviye Sorular

1. **Entity Framework'te Raw SQL performansı nasıl optimize edilir?**
   - **Cevap**:
     - Query optimizasyonu
     - Index kullanımı
     - Batch processing
     - Resource yönetimi
     - Caching stratejileri

2. **Entity Framework'te distributed sistemlerde Raw SQL nasıl yönetilir?**
   - **Cevap**:
     - Distributed transactions
     - Data partitioning
     - Replication
     - Consistency
     - Conflict resolution

3. **Entity Framework'te high concurrency senaryolarında Raw SQL nasıl yönetilir?**
   - **Cevap**:
     - Optimistic concurrency
     - Pessimistic concurrency
     - Retry mekanizmaları
     - Queue yönetimi
     - Batch processing

4. **Entity Framework'te Raw SQL monitoring ve profiling nasıl yapılır?**
   - **Cevap**:
     - Query logging
     - Performance metrics
     - Resource monitoring
     - Profiling tools
     - Health checks

5. **Entity Framework'te custom Raw SQL stratejileri nasıl geliştirilir?**
   - **Cevap**:
     - Custom SQL builder
     - Custom parameter handling
     - Custom error handling
     - Custom monitoring
     - Custom optimization 