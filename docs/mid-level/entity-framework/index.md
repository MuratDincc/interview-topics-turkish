# Entity Framework - Genel Bakış

## Giriş

Entity Framework (EF), .NET uygulamalarında veritabanı işlemlerini gerçekleştirmek için kullanılan bir ORM (Object-Relational Mapping) framework'üdür. Mid-level geliştiriciler için EF'in ileri seviye özellikleri ve kullanım senaryoları önemlidir.

## Mid-Level Entity Framework Konuları

1. **Performance Optimization**
   - Query optimizasyonu
   - Change tracking optimizasyonu
   - Bulk operasyonlar
   - Caching stratejileri
   - Index yönetimi

2. **Advanced Querying**
   - LINQ to Entities
   - Raw SQL queries
   - Stored procedures
   - Database functions
   - Complex queries

3. **Change Tracking**
   - Entity states
   - Change tracking strategies
   - Bulk updates
   - Concurrency control
   - Audit logging

4. **Bulk Operations**
   - Bulk insert
   - Bulk update
   - Bulk delete
   - Batch processing
   - Performance optimization

5. **Concurrency**
   - Optimistic concurrency
   - Pessimistic concurrency
   - Concurrency tokens
   - Conflict resolution
   - Retry strategies

6. **Raw SQL**
   - SQL injection önleme
   - Parametre kullanımı
   - Stored procedure çağrıları
   - View kullanımı
   - Custom SQL

7. **Interceptors**
   - Command interception
   - Connection interception
   - Transaction interception
   - Custom interception
   - Logging ve auditing

8. **Value Objects**
   - Value object tanımlama
   - Owned entity types
   - Value conversion
   - Query optimization
   - Immutability

9. **Complex Types**
   - Complex type tanımlama
   - Value conversion
   - Query optimization
   - Mapping stratejileri
   - Validation

10. **Shadow Properties**
    - Shadow property tanımlama
    - Audit fields
    - Soft delete
    - Tenant isolation
    - Security

11. **Global Query Filters**
    - Filter tanımlama
    - Multi-tenant uygulamalar
    - Soft delete
    - Security filters
    - Performance optimization

12. **Database Functions**
    - Built-in functions
    - Custom functions
    - Scalar functions
    - Table-valued functions
    - Performance considerations

13. **Custom Migrations**
    - Migration oluşturma
    - Custom SQL
    - Data seeding
    - Version control
    - Rollback stratejileri

14. **Multiple Databases**
    - Connection string yönetimi
    - Context factory pattern
    - Database provider seçimi
    - Migration yönetimi
    - Transaction yönetimi

15. **Distributed Transactions**
    - TransactionScope kullanımı
    - MSDTC yapılandırması
    - Two-phase commit
    - Error handling
    - Retry mekanizmaları

## Entity Framework Best Practices

1. **Performans**
   - Query optimizasyonu
   - Change tracking stratejileri
   - Bulk operasyonlar
   - Caching
   - Index yönetimi

2. **Güvenlik**
   - SQL injection önleme
   - Parametre kullanımı
   - Input validation
   - Access control
   - Audit logging

3. **Bakım**
   - Code-first yaklaşımı
   - Migration yönetimi
   - Version control
   - Documentation
   - Testing

4. **Monitoring**
   - Query logging
   - Performance metrics
   - Error tracking
   - Audit logging
   - Health checks

## Mülakat Soruları

### Temel Sorular

1. **Entity Framework nedir ve neden kullanılır?**
   - **Cevap**: Entity Framework, .NET uygulamalarında veritabanı işlemlerini gerçekleştirmek için kullanılan bir ORM framework'üdür. Veritabanı işlemlerini nesne yönelimli bir şekilde yapmayı sağlar, kod tekrarını azaltır ve veritabanı bağımsızlığı sağlar.

2. **Entity Framework'un temel özellikleri nelerdir?**
   - **Cevap**:
     - Object-Relational Mapping
     - LINQ desteği
     - Change tracking
     - Migrations
     - Concurrency control
     - Transaction yönetimi

3. **Entity Framework'te Code-First ve Database-First yaklaşımları arasındaki farklar nelerdir?**
   - **Cevap**: Code-First yaklaşımında önce entity sınıfları oluşturulur ve veritabanı bu sınıflardan türetilir. Database-First yaklaşımında ise önce veritabanı oluşturulur ve entity sınıfları veritabanından türetilir.

4. **Entity Framework'te lazy loading ve eager loading nedir?**
   - **Cevap**: Lazy loading, ilişkili verilerin ihtiyaç duyulduğunda yüklenmesidir. Eager loading ise ilişkili verilerin ana sorgu ile birlikte yüklenmesidir.

5. **Entity Framework'te change tracking nedir?**
   - **Cevap**: Change tracking, entity'lerdeki değişikliklerin takip edilmesi ve veritabanına yansıtılması sürecidir.

### Teknik Sorular

1. **Entity Framework'te performans optimizasyonu nasıl yapılır?**
   - **Cevap**:
```csharp
// Örnek: Eager loading
var blog = await _context.Blogs
    .Include(b => b.Posts)
    .FirstOrDefaultAsync(b => b.Id == id);

// Örnek: Projection
var blogTitles = await _context.Blogs
    .Select(b => new { b.Id, b.Title })
    .ToListAsync();

// Örnek: Raw SQL
var blogs = await _context.Blogs
    .FromSqlRaw("SELECT * FROM Blogs WHERE Rating > {0}", 3)
    .ToListAsync();
```

2. **Entity Framework'te transaction yönetimi nasıl yapılır?**
   - **Cevap**:
```csharp
using (var transaction = await _context.Database.BeginTransactionAsync())
{
    try
    {
        // İşlemler
        await _context.SaveChangesAsync();
        await transaction.CommitAsync();
    }
    catch
    {
        await transaction.RollbackAsync();
        throw;
    }
}
```

3. **Entity Framework'te concurrency kontrolü nasıl yapılır?**
   - **Cevap**:
```csharp
public class Blog
{
    public int Id { get; set; }
    public string Title { get; set; }
    [Timestamp]
    public byte[] RowVersion { get; set; }
}

// Güncelleme
try
{
    _context.Entry(blog).OriginalValues["RowVersion"] = originalRowVersion;
    await _context.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException)
{
    // Concurrency çakışması
}
```

4. **Entity Framework'te bulk operasyonlar nasıl yapılır?**
   - **Cevap**:
```csharp
// Bulk insert
await _context.BulkInsertAsync(entities);

// Bulk update
await _context.BulkUpdateAsync(entities);

// Bulk delete
await _context.BulkDeleteAsync(entities);
```

5. **Entity Framework'te custom migration nasıl oluşturulur?**
   - **Cevap**:
```csharp
public partial class CustomMigration : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.Sql("CREATE PROCEDURE ...");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.Sql("DROP PROCEDURE ...");
    }
}
```

### İleri Seviye Sorular

1. **Entity Framework'te distributed transaction nasıl yönetilir?**
   - **Cevap**:
     - TransactionScope kullanımı
     - MSDTC yapılandırması
     - Two-phase commit
     - Error handling
     - Retry mekanizmaları

2. **Entity Framework'te multiple database nasıl yönetilir?**
   - **Cevap**:
     - Connection string yönetimi
     - Context factory pattern
     - Database provider seçimi
     - Migration yönetimi
     - Transaction yönetimi

3. **Entity Framework'te interceptors nasıl kullanılır?**
   - **Cevap**:
     - Command interception
     - Connection interception
     - Transaction interception
     - Custom interception
     - Logging ve auditing

4. **Entity Framework'te value objects ve complex types nasıl kullanılır?**
   - **Cevap**:
     - Value object tanımlama
     - Complex type tanımlama
     - Owned entity types
     - Value conversion
     - Query optimization

5. **Entity Framework'te global query filters nasıl kullanılır?**
   - **Cevap**:
     - Filter tanımlama
     - Multi-tenant uygulamalar
     - Soft delete
     - Security filters
     - Performance optimization 