# Entity Framework - Change Tracking

## Giriş

Entity Framework'te Change Tracking (Değişiklik İzleme), veritabanı nesnelerindeki değişiklikleri otomatik olarak takip eden ve yöneten bir mekanizmadır. Mid-level geliştiriciler için bu mekanizmanın anlaşılması ve etkin kullanımı kritik öneme sahiptir.

## Change Tracking'in Önemi

1. **Veri Tutarlılığı**
   - Otomatik değişiklik takibi
   - Veri bütünlüğü koruması
   - İlişkisel veri yönetimi
   - Transaction yönetimi

2. **Performans**
   - Optimize edilmiş güncellemeler
   - Gereksiz sorguların önlenmesi
   - Batch işlem desteği
   - Bellek kullanımı optimizasyonu

3. **Bakım**
   - Daha az kod
   - Daha kolay debug
   - Daha iyi test edilebilirlik
   - Daha kolay bakım

## Change Tracking Özellikleri

1. **Entity States**
```csharp
// Entity durumları
var blog = new Blog { Title = "Yeni Blog" };
var state = _context.Entry(blog).State; // Detached

_context.Blogs.Add(blog);
state = _context.Entry(blog).State; // Added

await _context.SaveChangesAsync();
state = _context.Entry(blog).State; // Unchanged

blog.Title = "Güncellenmiş Blog";
state = _context.Entry(blog).State; // Modified

_context.Blogs.Remove(blog);
state = _context.Entry(blog).State; // Deleted
```

2. **Change Detection**
```csharp
// Değişiklik tespiti
var blog = await _context.Blogs.FindAsync(1);
var originalValues = _context.Entry(blog).OriginalValues;
var currentValues = _context.Entry(blog).CurrentValues;

// Değişiklik kontrolü
var hasChanges = _context.ChangeTracker.HasChanges();

// Değişen entity'ler
var modifiedEntities = _context.ChangeTracker
    .Entries()
    .Where(e => e.State == EntityState.Modified)
    .ToList();

// Değişen property'ler
var modifiedProperties = _context.Entry(blog)
    .Properties
    .Where(p => p.IsModified)
    .ToList();
```

3. **Change Tracking Modları**
```csharp
// Change tracking modları
// 1. Snapshot Change Tracking
var options = new DbContextOptionsBuilder()
    .UseSqlServer(connectionString)
    .UseChangeTrackingStrategy(ChangeTrackingStrategy.Snapshot)
    .Options;

// 2. ChangedNotifications Change Tracking
var options = new DbContextOptionsBuilder()
    .UseSqlServer(connectionString)
    .UseChangeTrackingStrategy(ChangeTrackingStrategy.ChangedNotifications)
    .Options;

// 3. Change Tracking'i devre dışı bırakma
var blog = await _context.Blogs
    .AsNoTracking()
    .FirstOrDefaultAsync(b => b.Id == 1);
```

4. **Bulk Operations**
```csharp
// Toplu güncelleme
var blogs = await _context.Blogs.ToListAsync();
foreach (var blog in blogs)
{
    blog.Rating += 1;
}

// Change tracking'i geçici olarak devre dışı bırakma
_context.ChangeTracker.AutoDetectChangesEnabled = false;
try
{
    // Toplu işlemler
    foreach (var blog in blogs)
    {
        _context.Blogs.Update(blog);
    }
    await _context.SaveChangesAsync();
}
finally
{
    _context.ChangeTracker.AutoDetectChangesEnabled = true;
}
```

5. **Custom Change Tracking**
```csharp
// Özel change tracking
public class BlogChangeTracker : IEntityChangeTracker
{
    public void TrackChanges(DbContext context)
    {
        var entries = context.ChangeTracker.Entries();
        foreach (var entry in entries)
        {
            if (entry.State == EntityState.Modified)
            {
                var blog = entry.Entity as Blog;
                if (blog != null)
                {
                    blog.LastModified = DateTime.UtcNow;
                    blog.ModifiedBy = "System";
                }
            }
        }
    }
}

// Kullanımı
var changeTracker = new BlogChangeTracker();
changeTracker.TrackChanges(_context);
```

## Best Practices

1. **Change Tracking Tasarımı**
   - Uygun tracking modu seçimi
   - Gereksiz tracking'den kaçınma
   - Batch işlemlerde optimizasyon
   - Bellek kullanımı yönetimi

2. **Performans**
   - AutoDetectChanges optimizasyonu
   - Batch işlem stratejileri
   - Bellek yönetimi
   - Query optimizasyonu

3. **Güvenlik**
   - Değişiklik doğrulama
   - Audit logging
   - Concurrency kontrolü
   - Transaction yönetimi

4. **Bakım**
   - Kod organizasyonu
   - Documentation
   - Testing
   - Error handling

## Mülakat Soruları

### Temel Sorular

1. **Entity Framework'te Change Tracking nedir?**
   - **Cevap**: Change Tracking, veritabanı nesnelerindeki değişiklikleri otomatik olarak takip eden ve yöneten bir mekanizmadır.

2. **Entity Framework'te entity state'ler nelerdir?**
   - **Cevap**: Detached, Added, Unchanged, Modified ve Deleted state'leri vardır.

3. **Entity Framework'te change tracking modları nelerdir?**
   - **Cevap**: Snapshot ve ChangedNotifications modları vardır.

4. **Entity Framework'te change tracking nasıl devre dışı bırakılır?**
   - **Cevap**: AsNoTracking() metodu veya AutoDetectChangesEnabled = false ile devre dışı bırakılabilir.

5. **Entity Framework'te değişiklikler nasıl tespit edilir?**
   - **Cevap**: ChangeTracker.Entries() veya Entry(entity).Properties ile tespit edilir.

### Teknik Sorular

1. **Entity state'leri nasıl kontrol edilir?**
   - **Cevap**:
```csharp
var state = _context.Entry(entity).State;
var states = _context.ChangeTracker.Entries()
    .Select(e => new { Entity = e.Entity, State = e.State })
    .ToList();
```

2. **Değişiklikleri nasıl izlenir?**
   - **Cevap**:
```csharp
var originalValues = _context.Entry(entity).OriginalValues;
var currentValues = _context.Entry(entity).CurrentValues;
var modifiedProperties = _context.Entry(entity)
    .Properties
    .Where(p => p.IsModified)
    .ToList();
```

3. **Change tracking modu nasıl değiştirilir?**
   - **Cevap**:
```csharp
var options = new DbContextOptionsBuilder()
    .UseSqlServer(connectionString)
    .UseChangeTrackingStrategy(ChangeTrackingStrategy.Snapshot)
    .Options;
```

4. **Toplu işlemlerde change tracking nasıl optimize edilir?**
   - **Cevap**:
```csharp
_context.ChangeTracker.AutoDetectChangesEnabled = false;
try
{
    // Toplu işlemler
    await _context.SaveChangesAsync();
}
finally
{
    _context.ChangeTracker.AutoDetectChangesEnabled = true;
}
```

5. **Özel change tracking nasıl implemente edilir?**
   - **Cevap**:
```csharp
public class CustomChangeTracker : IEntityChangeTracker
{
    public void TrackChanges(DbContext context)
    {
        var entries = context.ChangeTracker.Entries();
        foreach (var entry in entries)
        {
            if (entry.State == EntityState.Modified)
            {
                // Özel işlemler
            }
        }
    }
}
```

### İleri Seviye Sorular

1. **Entity Framework'te change tracking performansı nasıl optimize edilir?**
   - **Cevap**:
     - AutoDetectChanges optimizasyonu
     - Batch işlem stratejileri
     - Bellek yönetimi
     - Query optimizasyonu
     - Index kullanımı

2. **Entity Framework'te distributed sistemlerde change tracking nasıl yönetilir?**
   - **Cevap**:
     - Change tracking stratejileri
     - Replication
     - Consistency
     - Conflict resolution
     - Transaction yönetimi

3. **Entity Framework'te high concurrency senaryolarında change tracking nasıl yönetilir?**
   - **Cevap**:
     - Optimistic concurrency
     - Pessimistic concurrency
     - Retry mekanizmaları
     - Queue yönetimi
     - Batch processing

4. **Entity Framework'te change tracking monitoring ve profiling nasıl yapılır?**
   - **Cevap**:
     - Change tracking logging
     - Performance metrics
     - Resource monitoring
     - Profiling tools
     - Health checks

5. **Entity Framework'te custom change tracking stratejileri nasıl geliştirilir?**
   - **Cevap**:
     - IEntityChangeTracker implementasyonu
     - Custom state management
     - Custom change detection
     - Custom validation
     - Custom persistence 