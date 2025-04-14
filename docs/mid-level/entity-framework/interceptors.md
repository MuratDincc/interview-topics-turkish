# Entity Framework - Interceptors

## Giriş

Entity Framework'te Interceptors (Kesiciler), veritabanı işlemleri sırasında belirli noktalarda müdahale etmeyi sağlayan güçlü bir özelliktir. Mid-level geliştiriciler için bu özelliğin anlaşılması ve etkin kullanımı önemlidir.

## Interceptors'ın Önemi

1. **Esneklik**
   - İşlem öncesi/sonrası müdahale
   - Özel iş mantığı ekleme
   - Davranış değiştirme
   - Genişletilebilirlik

2. **Güvenlik**
   - Veri doğrulama
   - Yetkilendirme
   - Audit logging
   - Güvenlik kontrolleri

3. **Bakım**
   - Merkezi kontrol
   - Kod tekrarını önleme
   - Kolay test edilebilirlik
   - Modüler yapı

## Interceptor Tipleri

1. **Command Interceptor**
```csharp
public class CustomCommandInterceptor : DbCommandInterceptor
{
    public override InterceptionResult<DbDataReader> ReaderExecuting(
        DbCommand command,
        CommandEventData eventData,
        InterceptionResult<DbDataReader> result)
    {
        // Command öncesi işlemler
        LogCommand(command);
        return result;
    }

    public override ValueTask<InterceptionResult<DbDataReader>> ReaderExecutingAsync(
        DbCommand command,
        CommandEventData eventData,
        InterceptionResult<DbDataReader> result,
        CancellationToken cancellationToken = default)
    {
        // Async command öncesi işlemler
        LogCommand(command);
        return new ValueTask<InterceptionResult<DbDataReader>>(result);
    }
}
```

2. **Connection Interceptor**
```csharp
public class CustomConnectionInterceptor : DbConnectionInterceptor
{
    public override InterceptionResult ConnectionOpening(
        DbConnection connection,
        ConnectionEventData eventData,
        InterceptionResult result)
    {
        // Connection öncesi işlemler
        LogConnection(connection);
        return result;
    }

    public override ValueTask<InterceptionResult> ConnectionOpeningAsync(
        DbConnection connection,
        ConnectionEventData eventData,
        InterceptionResult result,
        CancellationToken cancellationToken = default)
    {
        // Async connection öncesi işlemler
        LogConnection(connection);
        return new ValueTask<InterceptionResult>(result);
    }
}
```

3. **Transaction Interceptor**
```csharp
public class CustomTransactionInterceptor : DbTransactionInterceptor
{
    public override InterceptionResult TransactionStarting(
        DbConnection connection,
        TransactionStartingEventData eventData,
        InterceptionResult result)
    {
        // Transaction başlangıcı işlemleri
        LogTransactionStart();
        return result;
    }

    public override ValueTask<InterceptionResult> TransactionStartingAsync(
        DbConnection connection,
        TransactionStartingEventData eventData,
        InterceptionResult result,
        CancellationToken cancellationToken = default)
    {
        // Async transaction başlangıcı işlemleri
        LogTransactionStart();
        return new ValueTask<InterceptionResult>(result);
    }
}
```

4. **SaveChanges Interceptor**
```csharp
public class CustomSaveChangesInterceptor : SaveChangesInterceptor
{
    public override InterceptionResult<int> SavingChanges(
        DbContextEventData eventData,
        InterceptionResult<int> result)
    {
        // SaveChanges öncesi işlemler
        ValidateChanges(eventData.Context);
        return result;
    }

    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        // Async SaveChanges öncesi işlemler
        ValidateChanges(eventData.Context);
        return new ValueTask<InterceptionResult<int>>(result);
    }
}
```

## Interceptor Kullanımı

1. **DbContext Konfigürasyonu**
```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder
        .AddInterceptors(new CustomCommandInterceptor())
        .AddInterceptors(new CustomConnectionInterceptor())
        .AddInterceptors(new CustomTransactionInterceptor())
        .AddInterceptors(new CustomSaveChangesInterceptor());
}
```

2. **Dependency Injection ile Kullanım**
```csharp
services.AddDbContext<ApplicationDbContext>(options =>
{
    options
        .AddInterceptors(new CustomCommandInterceptor())
        .AddInterceptors(new CustomConnectionInterceptor())
        .AddInterceptors(new CustomTransactionInterceptor())
        .AddInterceptors(new CustomSaveChangesInterceptor());
});
```

3. **Özel Interceptor Örnekleri**
```csharp
// Audit Logging Interceptor
public class AuditLoggingInterceptor : SaveChangesInterceptor
{
    public override InterceptionResult<int> SavingChanges(
        DbContextEventData eventData,
        InterceptionResult<int> result)
    {
        var entries = eventData.Context.ChangeTracker.Entries();
        foreach (var entry in entries)
        {
            if (entry.State == EntityState.Added || 
                entry.State == EntityState.Modified || 
                entry.State == EntityState.Deleted)
            {
                LogAudit(entry);
            }
        }
        return result;
    }
}

// Soft Delete Interceptor
public class SoftDeleteInterceptor : SaveChangesInterceptor
{
    public override InterceptionResult<int> SavingChanges(
        DbContextEventData eventData,
        InterceptionResult<int> result)
    {
        var entries = eventData.Context.ChangeTracker.Entries()
            .Where(e => e.State == EntityState.Deleted);
        
        foreach (var entry in entries)
        {
            if (entry.Entity is ISoftDelete softDelete)
            {
                softDelete.IsDeleted = true;
                entry.State = EntityState.Modified;
            }
        }
        return result;
    }
}
```

## Best Practices

1. **Interceptor Tasarımı**
   - Single Responsibility
   - Dependency Injection
   - Error handling
   - Performance

2. **Güvenlik**
   - Input validation
   - Authorization
   - Audit logging
   - Security checks

3. **Performans**
   - Minimal müdahale
   - Async operations
   - Resource yönetimi
   - Caching

4. **Bakım**
   - Kod organizasyonu
   - Documentation
   - Testing
   - Monitoring

## Mülakat Soruları

### Temel Sorular

1. **Entity Framework'te Interceptor nedir?**
   - **Cevap**: Interceptor, veritabanı işlemleri sırasında belirli noktalarda müdahale etmeyi sağlayan bir özelliktir.

2. **Entity Framework'te kaç çeşit Interceptor vardır?**
   - **Cevap**: Command Interceptor, Connection Interceptor, Transaction Interceptor ve SaveChanges Interceptor.

3. **Entity Framework'te Interceptor nasıl kaydedilir?**
   - **Cevap**: DbContext.OnConfiguring metodunda veya Dependency Injection ile AddInterceptors metodu kullanılarak.

4. **Entity Framework'te Interceptor ne zaman kullanılır?**
   - **Cevap**: Audit logging, soft delete, validation, authorization gibi cross-cutting concerns'ler için kullanılır.

5. **Entity Framework'te Interceptor performansı nasıl etkiler?**
   - **Cevap**: Her işlem için ek yük getirir, bu nedenle dikkatli kullanılmalıdır.

### Teknik Sorular

1. **Command Interceptor nasıl oluşturulur?**
   - **Cevap**:
```csharp
public class CustomCommandInterceptor : DbCommandInterceptor
{
    public override InterceptionResult<DbDataReader> ReaderExecuting(
        DbCommand command,
        CommandEventData eventData,
        InterceptionResult<DbDataReader> result)
    {
        LogCommand(command);
        return result;
    }
}
```

2. **SaveChanges Interceptor nasıl oluşturulur?**
   - **Cevap**:
```csharp
public class CustomSaveChangesInterceptor : SaveChangesInterceptor
{
    public override InterceptionResult<int> SavingChanges(
        DbContextEventData eventData,
        InterceptionResult<int> result)
    {
        ValidateChanges(eventData.Context);
        return result;
    }
}
```

3. **Soft Delete Interceptor nasıl oluşturulur?**
   - **Cevap**:
```csharp
public class SoftDeleteInterceptor : SaveChangesInterceptor
{
    public override InterceptionResult<int> SavingChanges(
        DbContextEventData eventData,
        InterceptionResult<int> result)
    {
        var entries = eventData.Context.ChangeTracker.Entries()
            .Where(e => e.State == EntityState.Deleted);
        
        foreach (var entry in entries)
        {
            if (entry.Entity is ISoftDelete softDelete)
            {
                softDelete.IsDeleted = true;
                entry.State = EntityState.Modified;
            }
        }
        return result;
    }
}
```

4. **Audit Logging Interceptor nasıl oluşturulur?**
   - **Cevap**:
```csharp
public class AuditLoggingInterceptor : SaveChangesInterceptor
{
    public override InterceptionResult<int> SavingChanges(
        DbContextEventData eventData,
        InterceptionResult<int> result)
    {
        var entries = eventData.Context.ChangeTracker.Entries();
        foreach (var entry in entries)
        {
            if (entry.State == EntityState.Added || 
                entry.State == EntityState.Modified || 
                entry.State == EntityState.Deleted)
            {
                LogAudit(entry);
            }
        }
        return result;
    }
}
```

5. **Interceptor'lar nasıl test edilir?**
   - **Cevap**:
```csharp
[Test]
public void TestCommandInterceptor()
{
    var interceptor = new CustomCommandInterceptor();
    var command = new Mock<DbCommand>();
    var eventData = new CommandEventData(null, null, null);
    var result = new InterceptionResult<DbDataReader>();

    var actual = interceptor.ReaderExecuting(command.Object, eventData, result);
    Assert.That(actual, Is.EqualTo(result));
}
```

### İleri Seviye Sorular

1. **Entity Framework'te Interceptor performansı nasıl optimize edilir?**
   - **Cevap**:
     - Minimal müdahale
     - Async operations
     - Resource yönetimi
     - Caching stratejileri
     - Conditional execution

2. **Entity Framework'te distributed sistemlerde Interceptor nasıl yönetilir?**
   - **Cevap**:
     - Distributed transactions
     - Event sourcing
     - Message queues
     - Consistency
     - Conflict resolution

3. **Entity Framework'te high concurrency senaryolarında Interceptor nasıl yönetilir?**
   - **Cevap**:
     - Optimistic concurrency
     - Pessimistic concurrency
     - Retry mekanizmaları
     - Queue yönetimi
     - Batch processing

4. **Entity Framework'te Interceptor monitoring ve profiling nasıl yapılır?**
   - **Cevap**:
     - Performance metrics
     - Resource monitoring
     - Profiling tools
     - Health checks
     - Logging

5. **Entity Framework'te custom Interceptor stratejileri nasıl geliştirilir?**
   - **Cevap**:
     - Custom event handling
     - Custom validation
     - Custom logging
     - Custom monitoring
     - Custom optimization 