# Entity Framework - Database Functions

## Giriş

Entity Framework'te Database Functions (Veritabanı Fonksiyonları), veritabanı seviyesinde çalışan ve LINQ sorgularında kullanılabilen fonksiyonlardır. Mid-level geliştiriciler için bu kavramın anlaşılması ve etkin kullanımı önemlidir.

## Database Functions'ın Önemi

1. **Veri Yönetimi**
   - Veritabanı seviyesinde işlem yapma
   - Daha iyi veri organizasyonu
   - Daha iyi veri bütünlüğü
   - Daha iyi veri erişimi

2. **Performans**
   - Veritabanı seviyesinde optimizasyon
   - Daha az veri transferi
   - Daha hızlı sorgular
   - Daha iyi kaynak kullanımı

3. **Bakım**
   - Daha az kod tekrarı
   - Daha kolay test edilebilirlik
   - Daha iyi modülerlik
   - Daha kolay genişletilebilirlik

## Database Functions Özellikleri

1. **Temel Database Function**
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.HasDbFunction(typeof(MyDbContext)
        .GetMethod(nameof(CalculateAge), new[] { typeof(DateTime) }))
        .HasName("CalculateAge");

    modelBuilder.HasDbFunction(typeof(MyDbContext)
        .GetMethod(nameof(FormatDate), new[] { typeof(DateTime), typeof(string) }))
        .HasName("FormatDate");
}

public static int CalculateAge(DateTime birthDate)
{
    throw new NotSupportedException();
}

public static string FormatDate(DateTime date, string format)
{
    throw new NotSupportedException();
}
```

2. **Database Function Kullanımı**
```csharp
public class UserService
{
    private readonly DbContext _context;

    public UserService(DbContext context)
    {
        _context = context;
    }

    public List<User> GetUsersByAge(int age)
    {
        return _context.Users
            .Where(u => MyDbContext.CalculateAge(u.BirthDate) == age)
            .ToList();
    }

    public List<User> GetUsersWithFormattedBirthDate(string format)
    {
        return _context.Users
            .Select(u => new
            {
                u.Id,
                u.Name,
                FormattedBirthDate = MyDbContext.FormatDate(u.BirthDate, format)
            })
            .ToList();
    }
}
```

3. **Database Function Validasyonu**
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.HasDbFunction(typeof(MyDbContext)
        .GetMethod(nameof(CalculateAge), new[] { typeof(DateTime) }))
        .HasName("CalculateAge")
        .HasParameter("birthDate")
        .HasStoreType("date");

    modelBuilder.HasDbFunction(typeof(MyDbContext)
        .GetMethod(nameof(FormatDate), new[] { typeof(DateTime), typeof(string) }))
        .HasName("FormatDate")
        .HasParameter("date")
        .HasStoreType("datetime")
        .HasParameter("format")
        .HasStoreType("nvarchar(50)");
}
```

## Database Functions Kullanımı

1. **Entity İçinde Kullanım**
```csharp
public class User
{
    public int Id { get; set; }
    public string Name { get; set; }
    public DateTime BirthDate { get; set; }
}

public class UserService
{
    private readonly DbContext _context;

    public UserService(DbContext context)
    {
        _context = context;
    }

    public List<User> GetUsersByAge(int age)
    {
        return _context.Users
            .Where(u => MyDbContext.CalculateAge(u.BirthDate) == age)
            .ToList();
    }

    public List<User> GetUsersWithFormattedBirthDate(string format)
    {
        return _context.Users
            .Select(u => new
            {
                u.Id,
                u.Name,
                FormattedBirthDate = MyDbContext.FormatDate(u.BirthDate, format)
            })
            .ToList();
    }
}
```

2. **DbContext Konfigürasyonu**
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.HasDbFunction(typeof(MyDbContext)
        .GetMethod(nameof(CalculateAge), new[] { typeof(DateTime) }))
        .HasName("CalculateAge")
        .HasParameter("birthDate")
        .HasStoreType("date");

    modelBuilder.HasDbFunction(typeof(MyDbContext)
        .GetMethod(nameof(FormatDate), new[] { typeof(DateTime), typeof(string) }))
        .HasName("FormatDate")
        .HasParameter("date")
        .HasStoreType("datetime")
        .HasParameter("format")
        .HasStoreType("nvarchar(50)");
}
```

3. **Database Function Dönüşümleri**
```csharp
public static class UserExtensions
{
    public static IQueryable<User> FilterByAge(this IQueryable<User> query, int age)
    {
        return query.Where(u => MyDbContext.CalculateAge(u.BirthDate) == age);
    }

    public static IQueryable<User> FormatBirthDate(this IQueryable<User> query, string format)
    {
        return query.Select(u => new
        {
            u.Id,
            u.Name,
            FormattedBirthDate = MyDbContext.FormatDate(u.BirthDate, format)
        });
    }
}
```

## Best Practices

1. **Database Function Tasarımı**
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

1. **Entity Framework'te Database Function nedir?**
   - **Cevap**: Database Function, veritabanı seviyesinde çalışan ve LINQ sorgularında kullanılabilen fonksiyonlardır.

2. **Entity Framework'te Database Function ve Normal Function arasındaki fark nedir?**
   - **Cevap**: Normal Function'lar uygulama seviyesinde çalışır, Database Function'lar veritabanı seviyesinde çalışır.

3. **Entity Framework'te Database Function nasıl konfigüre edilir?**
   - **Cevap**: DbContext.OnModelCreating metodunda HasDbFunction metodu kullanılarak konfigüre edilir.

4. **Entity Framework'te Database Function ne zaman kullanılır?**
   - **Cevap**: Veritabanı seviyesinde işlem yapmak, performansı artırmak veya veri bütünlüğünü sağlamak için kullanılır.

5. **Entity Framework'te Database Function performansı nasıl etkiler?**
   - **Cevap**: Veritabanı seviyesinde çalıştığı için performansı artırabilir ancak veritabanı yükünü de artırabilir.

### Teknik Sorular

1. **Temel Database Function nasıl oluşturulur?**
   - **Cevap**:
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.HasDbFunction(typeof(MyDbContext)
        .GetMethod(nameof(CalculateAge), new[] { typeof(DateTime) }))
        .HasName("CalculateAge");
}

public static int CalculateAge(DateTime birthDate)
{
    throw new NotSupportedException();
}
```

2. **Database Function nasıl kullanılır?**
   - **Cevap**:
```csharp
public List<User> GetUsersByAge(int age)
{
    return _context.Users
        .Where(u => MyDbContext.CalculateAge(u.BirthDate) == age)
        .ToList();
}
```

3. **Database Function validasyonu nasıl yapılır?**
   - **Cevap**:
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.HasDbFunction(typeof(MyDbContext)
        .GetMethod(nameof(CalculateAge), new[] { typeof(DateTime) }))
        .HasName("CalculateAge")
        .HasParameter("birthDate")
        .HasStoreType("date");
}
```

4. **Database Function nasıl özelleştirilir?**
   - **Cevap**:
```csharp
public static class UserExtensions
{
    public static IQueryable<User> FilterByAge(this IQueryable<User> query, int age)
    {
        return query.Where(u => MyDbContext.CalculateAge(u.BirthDate) == age);
    }
}
```

5. **Database Function nasıl test edilir?**
   - **Cevap**:
```csharp
[Test]
public void CalculateAge_ShouldReturnCorrectAge()
{
    var birthDate = new DateTime(1990, 1, 1);
    var expectedAge = DateTime.Now.Year - birthDate.Year;
    var actualAge = MyDbContext.CalculateAge(birthDate);
    Assert.AreEqual(expectedAge, actualAge);
}
```

### İleri Seviye Sorular

1. **Entity Framework'te Database Function performansı nasıl optimize edilir?**
   - **Cevap**:
     - Query optimizasyonu
     - Index kullanımı
     - Caching stratejileri
     - Lazy loading
     - Materialization optimizasyonu

2. **Entity Framework'te distributed sistemlerde Database Function nasıl yönetilir?**
   - **Cevap**:
     - Consistency
     - Replication
     - Sharding
     - Partitioning
     - Caching

3. **Entity Framework'te high concurrency senaryolarında Database Function nasıl yönetilir?**
   - **Cevap**:
     - Thread safety
     - Atomic operations
     - Locking stratejileri
     - Isolation levels
     - Conflict resolution

4. **Entity Framework'te Database Function monitoring ve profiling nasıl yapılır?**
   - **Cevap**:
     - Query profiling
     - Performance metrics
     - Resource monitoring
     - Profiling tools
     - Health checks

5. **Entity Framework'te custom Database Function stratejileri nasıl geliştirilir?**
   - **Cevap**:
     - Custom function handling
     - Custom validation
     - Custom optimization
     - Custom caching
     - Custom materialization 