# Migrations

## Genel Bakış
Migrations, Entity Framework Core'da veritabanı şemasını yönetmek için kullanılan bir sistemdir. Veritabanı değişikliklerini kod tabanlı bir yaklaşımla yönetmeyi ve versiyonlamayı sağlar.

## Mülakat Soruları ve Cevapları

### 1. Migration nedir ve neden kullanılır?
**Cevap:**
Migration, veritabanı şemasındaki değişiklikleri kod tabanlı olarak yönetmeyi sağlayan bir sistemdir. Kullanım nedenleri:
- Veritabanı değişikliklerinin versiyonlanması
- Takım çalışmasında değişikliklerin senkronizasyonu
- Geri alma (rollback) imkanı
- Otomatik şema güncelleme

**Örnek Kod:**
```csharp
// Migration oluşturma
dotnet ef migrations add InitialCreate

// Migration uygulama
dotnet ef database update

// Migration geri alma
dotnet ef database update PreviousMigration
```

### 2. Migration nasıl oluşturulur ve uygulanır?
**Cevap:**
Migration oluşturma ve uygulama adımları:
1. Entity sınıflarında değişiklik yapma
2. Migration oluşturma
3. Migration'ı inceleme
4. Veritabanına uygulama

**Örnek Kod:**
```csharp
// Entity sınıfı
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
}

// Migration oluşturma komutu
dotnet ef migrations add AddProductTable

// Migration dosyası
public partial class AddProductTable : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "Products",
            columns: table => new
            {
                Id = table.Column<int>(nullable: false)
                    .Annotation("SqlServer:Identity", "1, 1"),
                Name = table.Column<string>(nullable: false),
                Price = table.Column<decimal>(nullable: false)
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_Products", x => x.Id);
            });
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(name: "Products");
    }
}
```

### 3. Migration stratejileri nelerdir?
**Cevap:**
Temel Migration stratejileri:
- Code First: Entity sınıflarından veritabanı oluşturma
- Database First: Veritabanından entity sınıfları oluşturma
- Hybrid: Her iki yaklaşımın kombinasyonu

**Örnek Kod:**
```csharp
// Code First yaklaşımı
public class ApplicationDbContext : DbContext
{
    public DbSet<Product> Products { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>()
            .HasKey(p => p.Id);
            
        modelBuilder.Entity<Product>()
            .Property(p => p.Name)
            .IsRequired()
            .HasMaxLength(100);
    }
}

// Database First yaklaşımı
dotnet ef dbcontext scaffold "Server=.;Database=MyDb;Trusted_Connection=True;" Microsoft.EntityFrameworkCore.SqlServer -o Models
```

### 4. Migration'lar nasıl yönetilir?
**Cevap:**
Migration yönetimi için:
- Migration'ları version control'de tutun
- Migration'ları isimlendirin
- Migration'ları test edin
- Rollback stratejisi belirleyin

**Örnek Kod:**
```csharp
// Migration listesi görüntüleme
dotnet ef migrations list

// Migration silme
dotnet ef migrations remove

// Belirli bir migration'a geri dönme
dotnet ef database update 20240101000000_InitialCreate

// Tüm migration'ları geri alma
dotnet ef database update 0
```

### 5. Migration'ları production ortamında nasıl yönetirsiniz?
**Cevap:**
Production ortamında migration yönetimi için:
- Migration'ları script olarak oluşturun
- Script'leri inceleyin
- Yedekleme yapın
- Rollback planı hazırlayın

**Örnek Kod:**
```csharp
// Migration script oluşturma
dotnet ef migrations script -o migration.sql

// Belirli migration'lar arası script oluşturma
dotnet ef migrations script FromMigration ToMigration -o migration.sql

// Idempotent script oluşturma
dotnet ef migrations script --idempotent -o migration.sql
```

## Best Practices
1. **Migration Yönetimi**
   - Migration'ları düzenli oluşturun
   - Migration'ları isimlendirin
   - Migration'ları test edin
   - Rollback stratejisi belirleyin

2. **Veritabanı Değişiklikleri**
   - Büyük değişiklikleri parçalayın
   - Veri kaybına dikkat edin
   - Index'leri optimize edin
   - Performans etkisini değerlendirin

3. **Takım Çalışması**
   - Migration'ları senkronize edin
   - Çakışmaları yönetin
   - Code review yapın
   - Deployment stratejisi belirleyin

## Kaynaklar
- [Entity Framework Core Migrations](https://docs.microsoft.com/tr-tr/ef/core/managing-schemas/migrations/)
- [Managing Migrations](https://docs.microsoft.com/tr-tr/ef/core/managing-schemas/migrations/managing)
- [Applying Migrations](https://docs.microsoft.com/tr-tr/ef/core/managing-schemas/migrations/applying) 