# Entity Framework - Custom Migrations

## Giriş

Entity Framework'te Custom Migrations (Özel Migrasyonlar), veritabanı şemasını özelleştirmek ve veri dönüşümlerini yönetmek için kullanılan gelişmiş bir özelliktir. Mid-level geliştiriciler için bu kavramın anlaşılması ve etkin kullanımı önemlidir.

## Custom Migrations'ın Önemi

1. **Veri Yönetimi**
   - Veritabanı şemasını özelleştirme
   - Veri dönüşümlerini yönetme
   - Daha iyi veri bütünlüğü
   - Daha iyi veri erişimi

2. **Bakım**
   - Daha kolay şema yönetimi
   - Daha kolay veri dönüşümü
   - Daha iyi modülerlik
   - Daha kolay genişletilebilirlik

3. **Güvenlik**
   - Veri bütünlüğünü koruma
   - Veri dönüşümlerini güvenli yapma
   - Rollback stratejileri
   - Güvenlik kontrolleri

## Custom Migrations Özellikleri

1. **Temel Custom Migration**
```csharp
public partial class AddUserFullName : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AddColumn<string>(
            name: "FullName",
            table: "Users",
            nullable: true);

        migrationBuilder.Sql(
            @"UPDATE Users 
              SET FullName = FirstName + ' ' + LastName");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropColumn(
            name: "FullName",
            table: "Users");
    }
}
```

2. **Custom Migration Kullanımı**
```csharp
public class UserService
{
    private readonly DbContext _context;

    public UserService(DbContext context)
    {
        _context = context;
    }

    public void UpdateUserFullName(int userId, string fullName)
    {
        var user = _context.Users.Find(userId);
        if (user != null)
        {
            user.FullName = fullName;
            _context.SaveChanges();
        }
    }
}
```

3. **Custom Migration Validasyonu**
```csharp
public partial class AddUserFullName : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AddColumn<string>(
            name: "FullName",
            table: "Users",
            maxLength: 100,
            nullable: false,
            defaultValue: "");

        migrationBuilder.Sql(
            @"UPDATE Users 
              SET FullName = FirstName + ' ' + LastName
              WHERE FullName IS NULL");

        migrationBuilder.CreateIndex(
            name: "IX_Users_FullName",
            table: "Users",
            column: "FullName");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropIndex(
            name: "IX_Users_FullName",
            table: "Users");

        migrationBuilder.DropColumn(
            name: "FullName",
            table: "Users");
    }
}
```

## Custom Migrations Kullanımı

1. **Entity İçinde Kullanım**
```csharp
public class User
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string FullName { get; set; }
}

public class UserService
{
    private readonly DbContext _context;

    public UserService(DbContext context)
    {
        _context = context;
    }

    public void UpdateUserFullName(int userId, string fullName)
    {
        var user = _context.Users.Find(userId);
        if (user != null)
        {
            user.FullName = fullName;
            _context.SaveChanges();
        }
    }
}
```

2. **DbContext Konfigürasyonu**
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<User>(entity =>
    {
        entity.Property(e => e.FullName)
            .IsRequired()
            .HasMaxLength(100);

        entity.HasIndex(e => e.FullName);
    });
}
```

3. **Custom Migration Dönüşümleri**
```csharp
public static class UserExtensions
{
    public static void UpdateFullName(this User user)
    {
        user.FullName = $"{user.FirstName} {user.LastName}";
    }
}
```

## Best Practices

1. **Custom Migration Tasarımı**
   - Single Responsibility
   - Immutability
   - Validation
   - Business logic

2. **Güvenlik**
   - Data integrity
   - Rollback stratejileri
   - Access control
   - Audit logging

3. **Performans**
   - Batch processing
   - Index kullanımı
   - Transaction yönetimi
   - Timeout yönetimi

4. **Bakım**
   - Code organization
   - Documentation
   - Testing
   - Versioning

## Mülakat Soruları

### Temel Sorular

1. **Entity Framework'te Custom Migration nedir?**
   - **Cevap**: Custom Migration, veritabanı şemasını özelleştirmek ve veri dönüşümlerini yönetmek için kullanılan gelişmiş bir özelliktir.

2. **Entity Framework'te Custom Migration ve Normal Migration arasındaki fark nedir?**
   - **Cevap**: Normal Migration'lar otomatik olarak oluşturulur, Custom Migration'lar geliştirici tarafından özelleştirilir.

3. **Entity Framework'te Custom Migration nasıl oluşturulur?**
   - **Cevap**: Migration sınıfı oluşturularak ve Up/Down metodları override edilerek oluşturulur.

4. **Entity Framework'te Custom Migration ne zaman kullanılır?**
   - **Cevap**: Veritabanı şemasını özelleştirmek, veri dönüşümlerini yönetmek veya özel işlemler yapmak için kullanılır.

5. **Entity Framework'te Custom Migration performansı nasıl etkiler?**
   - **Cevap**: Veritabanı işlemlerini etkileyebilir ancak veri bütünlüğünü sağlamayı kolaylaştırır.

### Teknik Sorular

1. **Temel Custom Migration nasıl oluşturulur?**
   - **Cevap**:
```csharp
public partial class AddUserFullName : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AddColumn<string>(
            name: "FullName",
            table: "Users",
            nullable: true);
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropColumn(
            name: "FullName",
            table: "Users");
    }
}
```

2. **Custom Migration nasıl kullanılır?**
   - **Cevap**:
```csharp
public void UpdateUserFullName(int userId, string fullName)
{
    var user = _context.Users.Find(userId);
    if (user != null)
    {
        user.FullName = fullName;
        _context.SaveChanges();
    }
}
```

3. **Custom Migration validasyonu nasıl yapılır?**
   - **Cevap**:
```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.AddColumn<string>(
        name: "FullName",
        table: "Users",
        maxLength: 100,
        nullable: false,
        defaultValue: "");

    migrationBuilder.CreateIndex(
        name: "IX_Users_FullName",
        table: "Users",
        column: "FullName");
}
```

4. **Custom Migration nasıl özelleştirilir?**
   - **Cevap**:
```csharp
public static class UserExtensions
{
    public static void UpdateFullName(this User user)
    {
        user.FullName = $"{user.FirstName} {user.LastName}";
    }
}
```

5. **Custom Migration nasıl test edilir?**
   - **Cevap**:
```csharp
[Test]
public void AddUserFullName_ShouldAddFullNameColumn()
{
    var migration = new AddUserFullName();
    var migrationBuilder = new MigrationBuilder("Test");
    
    migration.Up(migrationBuilder);
    
    Assert.IsTrue(migrationBuilder.Operations.Any(o => 
        o is AddColumnOperation operation && 
        operation.Name == "FullName"));
}
```

### İleri Seviye Sorular

1. **Entity Framework'te Custom Migration performansı nasıl optimize edilir?**
   - **Cevap**:
     - Batch processing
     - Index kullanımı
     - Transaction yönetimi
     - Timeout yönetimi
     - Resource kullanımı

2. **Entity Framework'te distributed sistemlerde Custom Migration nasıl yönetilir?**
   - **Cevap**:
     - Consistency
     - Replication
     - Sharding
     - Partitioning
     - Caching

3. **Entity Framework'te high concurrency senaryolarında Custom Migration nasıl yönetilir?**
   - **Cevap**:
     - Locking stratejileri
     - Transaction isolation
     - Deadlock önleme
     - Retry mekanizmaları
     - Conflict resolution

4. **Entity Framework'te Custom Migration monitoring ve profiling nasıl yapılır?**
   - **Cevap**:
     - Performance metrics
     - Resource monitoring
     - Profiling tools
     - Health checks
     - Logging

5. **Entity Framework'te custom Custom Migration stratejileri nasıl geliştirilir?**
   - **Cevap**:
     - Custom migration handling
     - Custom validation
     - Custom optimization
     - Custom rollback
     - Custom testing 