# Entity Framework - Bulk Operations

## Giriş

Entity Framework'te Bulk Operations (Toplu İşlemler), büyük veri setleri üzerinde verimli bir şekilde toplu ekleme, güncelleme ve silme işlemleri yapmayı sağlayan tekniklerdir. Mid-level geliştiriciler için bu tekniklerin anlaşılması ve etkin kullanımı kritik öneme sahiptir.

## Bulk Operations'ın Önemi

1. **Performans**
   - Daha hızlı işlem süreleri
   - Daha az veritabanı yükü
   - Daha az network trafiği
   - Daha iyi kaynak kullanımı

2. **Ölçeklenebilirlik**
   - Büyük veri setleriyle çalışma
   - Yüksek hacimli işlemler
   - Batch processing
   - Paralel işlemler

3. **Bakım**
   - Daha az kod
   - Daha kolay debug
   - Daha iyi test edilebilirlik
   - Daha kolay bakım

## Bulk Operations Teknikleri

1. **Bulk Insert**
```csharp
// Temel bulk insert
var blogs = new List<Blog>();
for (int i = 0; i < 1000; i++)
{
    blogs.Add(new Blog { Title = $"Blog {i}" });
}

// 1. Yöntem: AddRange
_context.Blogs.AddRange(blogs);
await _context.SaveChangesAsync();

// 2. Yöntem: BulkExtensions
await _context.BulkInsertAsync(blogs);

// 3. Yöntem: ExecuteSqlRaw
var sql = "INSERT INTO Blogs (Title) VALUES (@p0)";
var parameters = blogs.Select(b => new SqlParameter("@p0", b.Title)).ToArray();
await _context.Database.ExecuteSqlRawAsync(sql, parameters);
```

2. **Bulk Update**
```csharp
// Temel bulk update
var blogs = await _context.Blogs.ToListAsync();
foreach (var blog in blogs)
{
    blog.Rating += 1;
}

// 1. Yöntem: UpdateRange
_context.Blogs.UpdateRange(blogs);
await _context.SaveChangesAsync();

// 2. Yöntem: BulkExtensions
await _context.BulkUpdateAsync(blogs);

// 3. Yöntem: ExecuteSqlRaw
var sql = "UPDATE Blogs SET Rating = Rating + 1";
await _context.Database.ExecuteSqlRawAsync(sql);
```

3. **Bulk Delete**
```csharp
// Temel bulk delete
var blogs = await _context.Blogs
    .Where(b => b.Rating < 3)
    .ToListAsync();

// 1. Yöntem: RemoveRange
_context.Blogs.RemoveRange(blogs);
await _context.SaveChangesAsync();

// 2. Yöntem: BulkExtensions
await _context.BulkDeleteAsync(blogs);

// 3. Yöntem: ExecuteSqlRaw
var sql = "DELETE FROM Blogs WHERE Rating < 3";
await _context.Database.ExecuteSqlRawAsync(sql);
```

4. **Batch Processing**
```csharp
// Batch işleme
public async Task ProcessInBatches<T>(List<T> items, int batchSize, Func<List<T>, Task> processBatch)
{
    for (int i = 0; i < items.Count; i += batchSize)
    {
        var batch = items.Skip(i).Take(batchSize).ToList();
        await processBatch(batch);
    }
}

// Kullanımı
var blogs = new List<Blog>();
// ... blogs doldurulur

await ProcessInBatches(blogs, 100, async batch =>
{
    _context.Blogs.AddRange(batch);
    await _context.SaveChangesAsync();
});
```

5. **Bulk Operations Optimizasyonu**
```csharp
// Change tracking optimizasyonu
_context.ChangeTracker.AutoDetectChangesEnabled = false;
try
{
    // Bulk işlemler
    await _context.SaveChangesAsync();
}
finally
{
    _context.ChangeTracker.AutoDetectChangesEnabled = true;
}

// Transaction optimizasyonu
using var transaction = await _context.Database.BeginTransactionAsync();
try
{
    // Bulk işlemler
    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}

// Batch size optimizasyonu
var batchSize = 1000; // Veritabanı ve network kapasitesine göre ayarlanır
```

## Best Practices

1. **Bulk Operations Tasarımı**
   - Uygun batch size seçimi
   - Transaction yönetimi
   - Error handling
   - Retry mekanizmaları

2. **Performans**
   - Batch size optimizasyonu
   - Change tracking optimizasyonu
   - Index kullanımı
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

1. **Entity Framework'te Bulk Operations nedir?**
   - **Cevap**: Bulk Operations, büyük veri setleri üzerinde verimli bir şekilde toplu ekleme, güncelleme ve silme işlemleri yapmayı sağlayan tekniklerdir.

2. **Entity Framework'te bulk insert nasıl yapılır?**
   - **Cevap**: AddRange, BulkExtensions veya ExecuteSqlRaw kullanılarak yapılabilir.

3. **Entity Framework'te bulk update nasıl yapılır?**
   - **Cevap**: UpdateRange, BulkExtensions veya ExecuteSqlRaw kullanılarak yapılabilir.

4. **Entity Framework'te bulk delete nasıl yapılır?**
   - **Cevap**: RemoveRange, BulkExtensions veya ExecuteSqlRaw kullanılarak yapılabilir.

5. **Entity Framework'te batch processing nedir?**
   - **Cevap**: Büyük veri setlerini daha küçük parçalara bölerek işleme tekniğidir.

### Teknik Sorular

1. **Bulk insert nasıl optimize edilir?**
   - **Cevap**:
```csharp
_context.ChangeTracker.AutoDetectChangesEnabled = false;
try
{
    _context.Blogs.AddRange(blogs);
    await _context.SaveChangesAsync();
}
finally
{
    _context.ChangeTracker.AutoDetectChangesEnabled = true;
}
```

2. **Batch processing nasıl implemente edilir?**
   - **Cevap**:
```csharp
public async Task ProcessInBatches<T>(List<T> items, int batchSize, Func<List<T>, Task> processBatch)
{
    for (int i = 0; i < items.Count; i += batchSize)
    {
        var batch = items.Skip(i).Take(batchSize).ToList();
        await processBatch(batch);
    }
}
```

3. **Transaction yönetimi nasıl yapılır?**
   - **Cevap**:
```csharp
using var transaction = await _context.Database.BeginTransactionAsync();
try
{
    // Bulk işlemler
    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

4. **Bulk operations'da error handling nasıl yapılır?**
   - **Cevap**:
```csharp
try
{
    // Bulk işlemler
}
catch (DbUpdateException ex)
{
    // Hata yönetimi
}
catch (Exception ex)
{
    // Genel hata yönetimi
}
```

5. **Bulk operations'da retry mekanizması nasıl implemente edilir?**
   - **Cevap**:
```csharp
public async Task RetryBulkOperation<T>(Func<Task> operation, int maxRetries = 3)
{
    for (int i = 0; i < maxRetries; i++)
    {
        try
        {
            await operation();
            return;
        }
        catch (Exception ex)
        {
            if (i == maxRetries - 1) throw;
            await Task.Delay(1000 * (i + 1));
        }
    }
}
```

### İleri Seviye Sorular

1. **Entity Framework'te bulk operations performansı nasıl optimize edilir?**
   - **Cevap**:
     - Batch size optimizasyonu
     - Change tracking optimizasyonu
     - Index kullanımı
     - Resource yönetimi
     - Parallel processing

2. **Entity Framework'te distributed sistemlerde bulk operations nasıl yönetilir?**
   - **Cevap**:
     - Distributed transactions
     - Data partitioning
     - Replication
     - Consistency
     - Conflict resolution

3. **Entity Framework'te high concurrency senaryolarında bulk operations nasıl yönetilir?**
   - **Cevap**:
     - Optimistic concurrency
     - Pessimistic concurrency
     - Retry mekanizmaları
     - Queue yönetimi
     - Batch processing

4. **Entity Framework'te bulk operations monitoring ve profiling nasıl yapılır?**
   - **Cevap**:
     - Performance metrics
     - Resource monitoring
     - Profiling tools
     - Health checks
     - Logging

5. **Entity Framework'te custom bulk operations stratejileri nasıl geliştirilir?**
   - **Cevap**:
     - Custom batch processing
     - Custom error handling
     - Custom retry mekanizmaları
     - Custom monitoring
     - Custom optimization 