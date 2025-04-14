# Entity Framework - Multiple Databases

## Giriş

Entity Framework'te Multiple Databases (Çoklu Veritabanları), farklı veritabanlarını aynı uygulamada yönetmek için kullanılan gelişmiş bir özelliktir. Mid-level geliştiriciler için bu kavramın anlaşılması ve etkin kullanımı önemlidir.

## Multiple Databases'ın Önemi

1. **Veri Yönetimi**
   - Farklı veritabanlarını yönetme
   - Veri bölümleme
   - Daha iyi veri organizasyonu
   - Daha iyi veri erişimi

2. **Performans**
   - Yük dengeleme
   - Ölçeklenebilirlik
   - Daha hızlı sorgular
   - Daha iyi kaynak kullanımı

3. **Güvenlik**
   - Veri izolasyonu
   - Erişim kontrolü
   - Güvenlik kontrolleri
   - Audit logging

## Multiple Databases Özellikleri

1. **Temel Multiple Database Konfigürasyonu**
```csharp
public class MainDbContext : DbContext
{
    public MainDbContext(DbContextOptions<MainDbContext> options)
        : base(options)
    {
    }

    public DbSet<User> Users { get; set; }
    public DbSet<Order> Orders { get; set; }
}

public class LogDbContext : DbContext
{
    public LogDbContext(DbContextOptions<LogDbContext> options)
        : base(options)
    {
    }

    public DbSet<Log> Logs { get; set; }
}

public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddDbContext<MainDbContext>(options =>
            options.UseSqlServer(Configuration.GetConnectionString("MainDb")));

        services.AddDbContext<LogDbContext>(options =>
            options.UseSqlServer(Configuration.GetConnectionString("LogDb")));
    }
}
```

2. **Multiple Database Kullanımı**
```csharp
public class UserService
{
    private readonly MainDbContext _mainContext;
    private readonly LogDbContext _logContext;

    public UserService(MainDbContext mainContext, LogDbContext logContext)
    {
        _mainContext = mainContext;
        _logContext = logContext;
    }

    public async Task CreateUser(User user)
    {
        await _mainContext.Users.AddAsync(user);
        await _mainContext.SaveChangesAsync();

        await _logContext.Logs.AddAsync(new Log
        {
            Message = $"User created: {user.Id}",
            CreatedAt = DateTime.UtcNow
        });
        await _logContext.SaveChangesAsync();
    }
}
```

3. **Multiple Database Validasyonu**
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<User>(entity =>
    {
        entity.HasKey(e => e.Id);
        entity.Property(e => e.Name).IsRequired().HasMaxLength(100);
        entity.Property(e => e.Email).IsRequired().HasMaxLength(100);
    });

    modelBuilder.Entity<Log>(entity =>
    {
        entity.HasKey(e => e.Id);
        entity.Property(e => e.Message).IsRequired();
        entity.Property(e => e.CreatedAt).IsRequired();
    });
}
```

## Multiple Databases Kullanımı

1. **Entity İçinde Kullanım**
```csharp
public class User
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
}

public class Log
{
    public int Id { get; set; }
    public string Message { get; set; }
    public DateTime CreatedAt { get; set; }
}

public class UserService
{
    private readonly MainDbContext _mainContext;
    private readonly LogDbContext _logContext;

    public UserService(MainDbContext mainContext, LogDbContext logContext)
    {
        _mainContext = mainContext;
        _logContext = logContext;
    }

    public async Task CreateUser(User user)
    {
        await _mainContext.Users.AddAsync(user);
        await _mainContext.SaveChangesAsync();

        await _logContext.Logs.AddAsync(new Log
        {
            Message = $"User created: {user.Id}",
            CreatedAt = DateTime.UtcNow
        });
        await _logContext.SaveChangesAsync();
    }
}
```

2. **DbContext Konfigürasyonu**
```csharp
public class MainDbContext : DbContext
{
    public MainDbContext(DbContextOptions<MainDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<User>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Name).IsRequired().HasMaxLength(100);
            entity.Property(e => e.Email).IsRequired().HasMaxLength(100);
        });
    }

    public DbSet<User> Users { get; set; }
}

public class LogDbContext : DbContext
{
    public LogDbContext(DbContextOptions<LogDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Log>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Message).IsRequired();
            entity.Property(e => e.CreatedAt).IsRequired();
        });
    }

    public DbSet<Log> Logs { get; set; }
}
```

3. **Multiple Database Dönüşümleri**
```csharp
public static class UserExtensions
{
    public static async Task LogUserCreation(this User user, LogDbContext context)
    {
        await context.Logs.AddAsync(new Log
        {
            Message = $"User created: {user.Id}",
            CreatedAt = DateTime.UtcNow
        });
        await context.SaveChangesAsync();
    }
}
```

## Best Practices

1. **Multiple Database Tasarımı**
   - Single Responsibility
   - Immutability
   - Validation
   - Business logic

2. **Güvenlik**
   - Data integrity
   - Access control
   - Audit logging
   - Encryption

3. **Performans**
   - Connection pooling
   - Query optimization
   - Caching
   - Lazy loading

4. **Bakım**
   - Code organization
   - Documentation
   - Testing
   - Versioning

## Mülakat Soruları

### Temel Sorular

1. **Entity Framework'te Multiple Database nedir?**
   - **Cevap**: Multiple Database, farklı veritabanlarını aynı uygulamada yönetmek için kullanılan gelişmiş bir özelliktir.

2. **Entity Framework'te Multiple Database ve Single Database arasındaki fark nedir?**
   - **Cevap**: Single Database'de tüm veriler tek bir veritabanında tutulur, Multiple Database'de farklı veritabanları kullanılır.

3. **Entity Framework'te Multiple Database nasıl konfigüre edilir?**
   - **Cevap**: Farklı DbContext sınıfları oluşturularak ve her biri için ayrı connection string tanımlanarak konfigüre edilir.

4. **Entity Framework'te Multiple Database ne zaman kullanılır?**
   - **Cevap**: Farklı veritabanları kullanmak, veri bölümleme yapmak veya performansı artırmak için kullanılır.

5. **Entity Framework'te Multiple Database performansı nasıl etkiler?**
   - **Cevap**: Yük dengeleme ve ölçeklenebilirlik sağlayabilir ancak connection yönetimi gerektirir.

### Teknik Sorular

1. **Temel Multiple Database nasıl oluşturulur?**
   - **Cevap**:
```csharp
public class MainDbContext : DbContext
{
    public MainDbContext(DbContextOptions<MainDbContext> options)
        : base(options)
    {
    }

    public DbSet<User> Users { get; set; }
}

public class LogDbContext : DbContext
{
    public LogDbContext(DbContextOptions<LogDbContext> options)
        : base(options)
    {
    }

    public DbSet<Log> Logs { get; set; }
}
```

2. **Multiple Database nasıl kullanılır?**
   - **Cevap**:
```csharp
public class UserService
{
    private readonly MainDbContext _mainContext;
    private readonly LogDbContext _logContext;

    public UserService(MainDbContext mainContext, LogDbContext logContext)
    {
        _mainContext = mainContext;
        _logContext = logContext;
    }

    public async Task CreateUser(User user)
    {
        await _mainContext.Users.AddAsync(user);
        await _mainContext.SaveChangesAsync();

        await _logContext.Logs.AddAsync(new Log
        {
            Message = $"User created: {user.Id}",
            CreatedAt = DateTime.UtcNow
        });
        await _logContext.SaveChangesAsync();
    }
}
```

3. **Multiple Database validasyonu nasıl yapılır?**
   - **Cevap**:
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<User>(entity =>
    {
        entity.HasKey(e => e.Id);
        entity.Property(e => e.Name).IsRequired().HasMaxLength(100);
    });
}
```

4. **Multiple Database nasıl özelleştirilir?**
   - **Cevap**:
```csharp
public static class UserExtensions
{
    public static async Task LogUserCreation(this User user, LogDbContext context)
    {
        await context.Logs.AddAsync(new Log
        {
            Message = $"User created: {user.Id}",
            CreatedAt = DateTime.UtcNow
        });
        await context.SaveChangesAsync();
    }
}
```

5. **Multiple Database nasıl test edilir?**
   - **Cevap**:
```csharp
[Test]
public async Task CreateUser_ShouldCreateUserAndLog()
{
    var mainOptions = new DbContextOptionsBuilder<MainDbContext>()
        .UseInMemoryDatabase("MainDb")
        .Options;

    var logOptions = new DbContextOptionsBuilder<LogDbContext>()
        .UseInMemoryDatabase("LogDb")
        .Options;

    using (var mainContext = new MainDbContext(mainOptions))
    using (var logContext = new LogDbContext(logOptions))
    {
        var service = new UserService(mainContext, logContext);
        var user = new User { Name = "Test User" };

        await service.CreateUser(user);

        Assert.IsTrue(mainContext.Users.Any(u => u.Name == "Test User"));
        Assert.IsTrue(logContext.Logs.Any(l => l.Message.Contains("User created")));
    }
}
```

### İleri Seviye Sorular

1. **Entity Framework'te Multiple Database performansı nasıl optimize edilir?**
   - **Cevap**:
     - Connection pooling
     - Query optimizasyonu
     - Caching stratejileri
     - Lazy loading
     - Materialization optimizasyonu

2. **Entity Framework'te distributed sistemlerde Multiple Database nasıl yönetilir?**
   - **Cevap**:
     - Consistency
     - Replication
     - Sharding
     - Partitioning
     - Caching

3. **Entity Framework'te high concurrency senaryolarında Multiple Database nasıl yönetilir?**
   - **Cevap**:
     - Locking stratejileri
     - Transaction isolation
     - Deadlock önleme
     - Retry mekanizmaları
     - Conflict resolution

4. **Entity Framework'te Multiple Database monitoring ve profiling nasıl yapılır?**
   - **Cevap**:
     - Performance metrics
     - Resource monitoring
     - Profiling tools
     - Health checks
     - Logging

5. **Entity Framework'te custom Multiple Database stratejileri nasıl geliştirilir?**
   - **Cevap**:
     - Custom connection handling
     - Custom validation
     - Custom optimization
     - Custom caching
     - Custom materialization 