# Entity Framework - Concurrency

## Giriş

Entity Framework'te Concurrency (Eşzamanlılık), aynı veri üzerinde birden fazla kullanıcının veya işlemin eşzamanlı olarak çalışmasını yönetmeyi sağlayan bir mekanizmadır. Mid-level geliştiriciler için bu mekanizmanın anlaşılması ve etkin kullanımı kritik öneme sahiptir.

## Concurrency'nin Önemi

1. **Veri Tutarlılığı**
   - Veri çakışmalarını önleme
   - Veri bütünlüğünü koruma
   - İlişkisel veri yönetimi
   - Transaction yönetimi

2. **Performans**
   - Eşzamanlı işlem desteği
   - Kaynak kullanımı optimizasyonu
   - Sistem yanıt süreleri
   - Ölçeklenebilirlik

3. **Bakım**
   - Daha az kod
   - Daha kolay debug
   - Daha iyi test edilebilirlik
   - Daha kolay bakım

## Concurrency Teknikleri

1. **Optimistic Concurrency**
```csharp
// Timestamp/RowVersion ile
public class Blog
{
    public int Id { get; set; }
    public string Title { get; set; }
    [Timestamp]
    public byte[] RowVersion { get; set; }
}

// Concurrency token ile
public class Blog
{
    public int Id { get; set; }
    public string Title { get; set; }
    [ConcurrencyCheck]
    public string ConcurrencyToken { get; set; }
}

// Kullanımı
try
{
    var blog = await _context.Blogs.FindAsync(1);
    blog.Title = "Güncellenmiş Blog";
    await _context.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)
{
    // Concurrency çakışması yönetimi
    var entry = ex.Entries.Single();
    var databaseValues = await entry.GetDatabaseValuesAsync();
    var clientValues = entry.CurrentValues;
    
    // Çakışma çözümü
    entry.OriginalValues.SetValues(databaseValues);
    // veya
    entry.CurrentValues.SetValues(databaseValues);
}
```

2. **Pessimistic Concurrency**
```csharp
// Transaction ile
using var transaction = await _context.Database.BeginTransactionAsync(
    IsolationLevel.Serializable);
try
{
    var blog = await _context.Blogs
        .FromSqlRaw("SELECT * FROM Blogs WITH (UPDLOCK) WHERE Id = {0}", 1)
        .FirstOrDefaultAsync();
        
    blog.Title = "Güncellenmiş Blog";
    await _context.SaveChangesAsync();
    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}

// Stored procedure ile
var blog = await _context.Blogs
    .FromSqlRaw("EXEC GetBlogWithLock @Id = {0}", 1)
    .FirstOrDefaultAsync();
```

3. **Concurrency Resolution**
```csharp
// Client wins
public async Task UpdateBlogClientWins(Blog blog)
{
    try
    {
        await _context.SaveChangesAsync();
    }
    catch (DbUpdateConcurrencyException ex)
    {
        var entry = ex.Entries.Single();
        entry.OriginalValues.SetValues(await entry.GetDatabaseValuesAsync());
        await _context.SaveChangesAsync();
    }
}

// Store wins
public async Task UpdateBlogStoreWins(Blog blog)
{
    try
    {
        await _context.SaveChangesAsync();
    }
    catch (DbUpdateConcurrencyException ex)
    {
        var entry = ex.Entries.Single();
        var databaseValues = await entry.GetDatabaseValuesAsync();
        entry.CurrentValues.SetValues(databaseValues);
        await _context.SaveChangesAsync();
    }
}

// Merge
public async Task UpdateBlogMerge(Blog blog)
{
    try
    {
        await _context.SaveChangesAsync();
    }
    catch (DbUpdateConcurrencyException ex)
    {
        var entry = ex.Entries.Single();
        var databaseValues = await entry.GetDatabaseValuesAsync();
        var clientValues = entry.CurrentValues;
        
        // Özel merge mantığı
        foreach (var property in clientValues.Properties)
        {
            var databaseValue = databaseValues[property];
            var clientValue = clientValues[property];
            
            if (clientValue != databaseValue)
            {
                // Merge stratejisi
                clientValues[property] = MergeValues(clientValue, databaseValue);
            }
        }
        
        await _context.SaveChangesAsync();
    }
}
```

4. **Custom Concurrency**
```csharp
// Özel concurrency kontrolü
public class BlogConcurrencyHandler : IConcurrencyHandler<Blog>
{
    public async Task HandleConcurrencyAsync(
        DbContext context,
        Blog entity,
        DbUpdateConcurrencyException ex)
    {
        var entry = ex.Entries.Single();
        var databaseValues = await entry.GetDatabaseValuesAsync();
        
        // Özel concurrency stratejisi
        if (ShouldOverride(databaseValues, entity))
        {
            entry.OriginalValues.SetValues(databaseValues);
            await context.SaveChangesAsync();
        }
        else
        {
            throw new ConcurrencyException("Concurrency çakışması");
        }
    }
}

// Kullanımı
try
{
    await _context.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)
{
    var handler = new BlogConcurrencyHandler();
    await handler.HandleConcurrencyAsync(_context, blog, ex);
}
```

5. **Concurrency Monitoring**
```csharp
// Concurrency izleme
public class ConcurrencyMonitor
{
    private readonly DbContext _context;
    
    public ConcurrencyMonitor(DbContext context)
    {
        _context = context;
    }
    
    public async Task MonitorConcurrencyAsync()
    {
        var concurrencyConflicts = _context.ChangeTracker
            .Entries()
            .Where(e => e.State == EntityState.Modified)
            .Select(e => new
            {
                Entity = e.Entity,
                OriginalValues = e.OriginalValues,
                CurrentValues = e.CurrentValues
            })
            .ToList();
            
        // Concurrency izleme işlemleri
    }
}
```

## Best Practices

1. **Concurrency Tasarımı**
   - Uygun concurrency stratejisi seçimi
   - Concurrency token yönetimi
   - Transaction yönetimi
   - Error handling

2. **Performans**
   - Concurrency optimizasyonu
   - Lock stratejileri
   - Resource yönetimi
   - Query optimizasyonu

3. **Güvenlik**
   - Concurrency doğrulama
   - Access control
   - Audit logging
   - Data validation

4. **Bakım**
   - Kod organizasyonu
   - Documentation
   - Testing
   - Monitoring

## Mülakat Soruları

### Temel Sorular

1. **Entity Framework'te concurrency nedir?**
   - **Cevap**: Concurrency, aynı veri üzerinde birden fazla kullanıcının veya işlemin eşzamanlı olarak çalışmasını yönetmeyi sağlayan bir mekanizmadır.

2. **Entity Framework'te optimistic concurrency nedir?**
   - **Cevap**: Optimistic concurrency, veri üzerinde kilit tutmadan, değişikliklerin kaydedilmesi sırasında çakışma kontrolü yapan bir stratejidir.

3. **Entity Framework'te pessimistic concurrency nedir?**
   - **Cevap**: Pessimistic concurrency, veri üzerinde kilit tutarak, diğer işlemlerin erişimini engelleyen bir stratejidir.

4. **Entity Framework'te concurrency token nedir?**
   - **Cevap**: Concurrency token, veri değişikliklerini takip etmek için kullanılan bir alandır.

5. **Entity Framework'te concurrency çakışması nasıl yönetilir?**
   - **Cevap**: DbUpdateConcurrencyException yakalanarak ve uygun strateji uygulanarak yönetilir.

### Teknik Sorular

1. **Optimistic concurrency nasıl implemente edilir?**
   - **Cevap**:
```csharp
public class Blog
{
    [Timestamp]
    public byte[] RowVersion { get; set; }
}

try
{
    await _context.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)
{
    var entry = ex.Entries.Single();
    entry.OriginalValues.SetValues(await entry.GetDatabaseValuesAsync());
    await _context.SaveChangesAsync();
}
```

2. **Pessimistic concurrency nasıl implemente edilir?**
   - **Cevap**:
```csharp
using var transaction = await _context.Database.BeginTransactionAsync(
    IsolationLevel.Serializable);
try
{
    var blog = await _context.Blogs
        .FromSqlRaw("SELECT * FROM Blogs WITH (UPDLOCK) WHERE Id = {0}", 1)
        .FirstOrDefaultAsync();
        
    await _context.SaveChangesAsync();
    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

3. **Concurrency resolution stratejileri nelerdir?**
   - **Cevap**:
```csharp
// Client wins
entry.OriginalValues.SetValues(await entry.GetDatabaseValuesAsync());

// Store wins
entry.CurrentValues.SetValues(await entry.GetDatabaseValuesAsync());

// Merge
foreach (var property in clientValues.Properties)
{
    // Merge stratejisi
}
```

4. **Custom concurrency nasıl implemente edilir?**
   - **Cevap**:
```csharp
public class CustomConcurrencyHandler : IConcurrencyHandler<Blog>
{
    public async Task HandleConcurrencyAsync(
        DbContext context,
        Blog entity,
        DbUpdateConcurrencyException ex)
    {
        // Özel concurrency stratejisi
    }
}
```

5. **Concurrency monitoring nasıl yapılır?**
   - **Cevap**:
```csharp
var concurrencyConflicts = _context.ChangeTracker
    .Entries()
    .Where(e => e.State == EntityState.Modified)
    .Select(e => new
    {
        Entity = e.Entity,
        OriginalValues = e.OriginalValues,
        CurrentValues = e.CurrentValues
    })
    .ToList();
```

### İleri Seviye Sorular

1. **Entity Framework'te concurrency performansı nasıl optimize edilir?**
   - **Cevap**:
     - Concurrency stratejisi optimizasyonu
     - Lock stratejileri
     - Resource yönetimi
     - Query optimizasyonu
     - Caching stratejileri

2. **Entity Framework'te distributed sistemlerde concurrency nasıl yönetilir?**
   - **Cevap**:
     - Distributed transactions
     - Data partitioning
     - Replication
     - Consistency
     - Conflict resolution

3. **Entity Framework'te high concurrency senaryolarında nasıl yönetilir?**
   - **Cevap**:
     - Optimistic concurrency
     - Pessimistic concurrency
     - Retry mekanizmaları
     - Queue yönetimi
     - Batch processing

4. **Entity Framework'te concurrency monitoring ve profiling nasıl yapılır?**
   - **Cevap**:
     - Concurrency metrics
     - Resource monitoring
     - Profiling tools
     - Health checks
     - Logging

5. **Entity Framework'te custom concurrency stratejileri nasıl geliştirilir?**
   - **Cevap**:
     - IConcurrencyHandler implementasyonu
     - Custom resolution stratejileri
     - Custom monitoring
     - Custom validation
     - Custom persistence 