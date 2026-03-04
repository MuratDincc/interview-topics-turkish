# Veri İzolasyonu (Data Isolation)

## Genel Bakış

Multi-tenancy mimarisinde veri izolasyonu, farklı tenant'lara ait verilerin birbirinden güvenli biçimde ayrılmasını sağlayan tasarım stratejileridir. Yanlış veya eksik izolasyon, ciddi güvenlik ihlallerine ve yasal yükümlülüklere yol açabilir. Üç temel strateji mevcuttur: database-per-tenant, schema-per-tenant ve row-level isolation. Her stratejinin izolasyon gücü, maliyet profili ve operasyonel karmaşıklığı farklıdır.

## Temel Kavramlar

### 1. Database-Per-Tenant Stratejisi

Her tenant için fiziksel olarak ayrı bir veritabanı oluşturulur. Uygulama, ilgili tenant'ın connection string'ini dinamik olarak seçerek bağlantı kurar.

**Avantajlar:**
- En güçlü veri izolasyonu
- Tenant başına yedekleme ve geri yükleme
- Tenant kaldırıldığında veritabanı silinir, veri artığı kalmaz
- Her tenant için ayrı performans ayarı

**Dezavantajlar:**
- Yüksek kaynak tüketimi (çok sayıda veritabanı bağlantı havuzu)
- Şema migrasyonları tüm tenant veritabanlarında çalıştırılmalıdır
- Çapraz tenant sorgulama karmaşıktır

```csharp
// Tenant'a özgü connection string yönetimi
public interface ITenantConnectionStringResolver
{
    Task<string> ResolveAsync(string tenantId, CancellationToken cancellationToken = default);
}

public class TenantConnectionStringResolver : ITenantConnectionStringResolver
{
    private readonly ITenantStore _tenantStore;
    private readonly ILogger<TenantConnectionStringResolver> _logger;

    public TenantConnectionStringResolver(
        ITenantStore tenantStore,
        ILogger<TenantConnectionStringResolver> logger)
    {
        _tenantStore = tenantStore;
        _logger = logger;
    }

    public async Task<string> ResolveAsync(string tenantId, CancellationToken cancellationToken = default)
    {
        var tenant = await _tenantStore.GetByIdAsync(tenantId, cancellationToken);

        if (tenant is null)
        {
            _logger.LogError("Tenant bulunamadı: {TenantId}", tenantId);
            throw new TenantNotFoundException(tenantId);
        }

        if (!tenant.IsActive)
        {
            _logger.LogWarning("Pasif tenant erişim denemesi: {TenantId}", tenantId);
            throw new TenantInactiveException(tenantId);
        }

        return tenant.ConnectionString;
    }
}

// Database-per-tenant DbContext fabrikası
public class TenantDbContextFactory
{
    private readonly ITenantConnectionStringResolver _connectionStringResolver;
    private readonly ILoggerFactory _loggerFactory;

    public TenantDbContextFactory(
        ITenantConnectionStringResolver connectionStringResolver,
        ILoggerFactory loggerFactory)
    {
        _connectionStringResolver = connectionStringResolver;
        _loggerFactory = loggerFactory;
    }

    public async Task<AppDbContext> CreateAsync(string tenantId, CancellationToken cancellationToken = default)
    {
        var connectionString = await _connectionStringResolver.ResolveAsync(tenantId, cancellationToken);

        var optionsBuilder = new DbContextOptionsBuilder<AppDbContext>();
        optionsBuilder
            .UseSqlServer(connectionString)
            .UseLoggerFactory(_loggerFactory);

        return new AppDbContext(optionsBuilder.Options);
    }
}
```

### 2. Schema-Per-Tenant Stratejisi

Aynı veritabanı sunucusunda her tenant için ayrı bir SQL şeması oluşturulur. `acme.Orders`, `globex.Orders` gibi şema-prefix'li tablo isimleri kullanılır.

**Avantajlar:**
- Veritabanı bağlantı havuzu paylaşılabilir
- Fiziksel izolasyon database-per-tenant'a yakın
- Şema başına erişim kontrolü uygulanabilir

**Dezavantajlar:**
- SQL Server ve PostgreSQL destekler; bazı motorlar desteklemeyebilir
- Şema sayısı arttıkça yönetim karmaşıklaşır
- EF Core ile şema yönetimi özel yapılandırma gerektirir

```csharp
// Schema-per-tenant için DbContext yapılandırması
public class SchemaAwareDbContext : DbContext
{
    private readonly string _schema;

    public SchemaAwareDbContext(DbContextOptions options, string schema) : base(options)
    {
        _schema = schema;
    }

    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Customer> Customers => Set<Customer>();
    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Tüm entity'leri ilgili şemaya yönlendir
        modelBuilder.HasDefaultSchema(_schema);

        modelBuilder.Entity<Order>(entity =>
        {
            entity.ToTable("Orders", _schema);
            entity.HasKey(o => o.Id);
        });

        modelBuilder.Entity<Customer>(entity =>
        {
            entity.ToTable("Customers", _schema);
            entity.HasKey(c => c.Id);
        });

        modelBuilder.Entity<Product>(entity =>
        {
            entity.ToTable("Products", _schema);
            entity.HasKey(p => p.Id);
        });
    }
}

// Schema-per-tenant DbContext fabrikası
public class SchemaTenantDbContextFactory
{
    private readonly string _masterConnectionString;
    private readonly ITenantStore _tenantStore;
    private readonly ILoggerFactory _loggerFactory;

    public SchemaTenantDbContextFactory(
        string masterConnectionString,
        ITenantStore tenantStore,
        ILoggerFactory loggerFactory)
    {
        _masterConnectionString = masterConnectionString;
        _tenantStore = tenantStore;
        _loggerFactory = loggerFactory;
    }

    public async Task<SchemaAwareDbContext> CreateAsync(
        string tenantId,
        CancellationToken cancellationToken = default)
    {
        var tenant = await _tenantStore.GetByIdAsync(tenantId, cancellationToken)
            ?? throw new TenantNotFoundException(tenantId);

        // Şema adını tenant subdomain'inden türet (güvenli karakter kontrolü)
        var schema = SanitizeSchemaName(tenant.Subdomain);

        var optionsBuilder = new DbContextOptionsBuilder<SchemaAwareDbContext>();
        optionsBuilder
            .UseSqlServer(_masterConnectionString)
            .UseLoggerFactory(_loggerFactory);

        return new SchemaAwareDbContext(optionsBuilder.Options, schema);
    }

    private static string SanitizeSchemaName(string subdomain)
    {
        // Şema adında yalnızca alfanümerik ve alt çizgiye izin ver
        return System.Text.RegularExpressions.Regex.Replace(subdomain, @"[^a-zA-Z0-9_]", "_").ToLowerInvariant();
    }
}

// Tenant şeması oluşturma servisi
public class TenantSchemaProvisioningService
{
    private readonly string _masterConnectionString;
    private readonly ILogger<TenantSchemaProvisioningService> _logger;

    public TenantSchemaProvisioningService(
        string masterConnectionString,
        ILogger<TenantSchemaProvisioningService> logger)
    {
        _masterConnectionString = masterConnectionString;
        _logger = logger;
    }

    public async Task ProvisionSchemaAsync(string schemaName, CancellationToken cancellationToken = default)
    {
        _logger.LogInformation("Tenant şeması oluşturuluyor: {Schema}", schemaName);

        await using var connection = new SqlConnection(_masterConnectionString);
        await connection.OpenAsync(cancellationToken);

        // Şemayı oluştur
        var createSchemaCmd = connection.CreateCommand();
        createSchemaCmd.CommandText = $"IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = @schema) EXEC('CREATE SCHEMA [{schemaName}]')";
        createSchemaCmd.Parameters.AddWithValue("@schema", schemaName);
        await createSchemaCmd.ExecuteNonQueryAsync(cancellationToken);

        _logger.LogInformation("Tenant şeması oluşturuldu: {Schema}", schemaName);
    }
}
```

### 3. Row-Level Isolation (Paylaşımlı Tablo) Stratejisi

Tüm tenant'lar aynı tabloları paylaşır; her satıra `TenantId` kolonu eklenerek tenant bazlı filtreleme yapılır.

**Avantajlar:**
- En düşük altyapı maliyeti
- Yönetimi basit (tek şema, tek veritabanı)
- Çapraz tenant analizleri kolayca yapılabilir

**Dezavantajlar:**
- Yazılımsal izolasyon gerektirir; hata maliyeti yüksek
- Büyük tenant'lar performansı etkileyebilir (noisy neighbor)
- Her sorguya `WHERE TenantId = ?` eklenmesini sağlamak kritik

```csharp
// Row-level isolation için temel entity
public abstract class TenantEntity
{
    public Guid Id { get; set; }
    public string TenantId { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? UpdatedAt { get; set; }
}

public class Order : TenantEntity
{
    public string OrderNumber { get; set; } = string.Empty;
    public decimal TotalAmount { get; set; }
    public OrderStatus Status { get; set; }
    public List<OrderItem> Items { get; set; } = new();
}

public enum OrderStatus { Pending, Confirmed, Shipped, Delivered, Cancelled }

public class OrderItem
{
    public Guid Id { get; set; }
    public Guid OrderId { get; set; }
    public string ProductName { get; set; } = string.Empty;
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}
```

### 4. EF Core Global Query Filters ile Otomatik Tenant Filtreleme

EF Core'un `HasQueryFilter` özelliği, her sorguda otomatik olarak `WHERE TenantId = @currentTenantId` koşulunu ekler. Bu sayede geliştiricilerin her sorguda `TenantId` filtresi eklemeyi unutması önlenir.

```csharp
// Tenant-aware DbContext
public class TenantAwareDbContext : DbContext
{
    private readonly ITenantContext _tenantContext;
    private readonly ILogger<TenantAwareDbContext> _logger;

    public TenantAwareDbContext(
        DbContextOptions<TenantAwareDbContext> options,
        ITenantContext tenantContext,
        ILogger<TenantAwareDbContext> logger) : base(options)
    {
        _tenantContext = tenantContext;
        _logger = logger;
    }

    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Customer> Customers => Set<Customer>();
    public DbSet<Invoice> Invoices => Set<Invoice>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Tüm TenantEntity türevlerine otomatik filtre uygula
        foreach (var entityType in modelBuilder.Model.GetEntityTypes())
        {
            if (typeof(TenantEntity).IsAssignableFrom(entityType.ClrType))
            {
                var method = typeof(TenantAwareDbContext)
                    .GetMethod(nameof(SetTenantFilter), System.Reflection.BindingFlags.NonPublic | System.Reflection.BindingFlags.Instance)!
                    .MakeGenericMethod(entityType.ClrType);

                method.Invoke(this, new object[] { modelBuilder });
            }
        }

        modelBuilder.Entity<Order>(entity =>
        {
            entity.HasIndex(o => o.TenantId);
            entity.HasIndex(o => new { o.TenantId, o.Status });
        });

        modelBuilder.Entity<Customer>(entity =>
        {
            entity.HasIndex(c => c.TenantId);
        });
    }

    private void SetTenantFilter<TEntity>(ModelBuilder modelBuilder)
        where TEntity : TenantEntity
    {
        modelBuilder.Entity<TEntity>()
            .HasQueryFilter(e => e.TenantId == _tenantContext.TenantId);
    }

    public override int SaveChanges()
    {
        EnforceTenantId();
        return base.SaveChanges();
    }

    public override Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        EnforceTenantId();
        return base.SaveChangesAsync(cancellationToken);
    }

    // Yeni eklenen entity'lere TenantId'yi otomatik ata
    private void EnforceTenantId()
    {
        if (!_tenantContext.IsResolved)
        {
            throw new InvalidOperationException("Tenant context çözümlenmeden veritabanı işlemi yapılamaz.");
        }

        var entries = ChangeTracker.Entries<TenantEntity>()
            .Where(e => e.State == EntityState.Added);

        foreach (var entry in entries)
        {
            if (string.IsNullOrEmpty(entry.Entity.TenantId))
            {
                entry.Entity.TenantId = _tenantContext.TenantId;
            }
            else if (entry.Entity.TenantId != _tenantContext.TenantId)
            {
                _logger.LogCritical(
                    "Farklı tenant ID'ye sahip entity ekleme girişimi! Context: {ContextTenant}, Entity: {EntityTenant}",
                    _tenantContext.TenantId, entry.Entity.TenantId);
                throw new TenantSecurityException(
                    $"Tenant çakışması tespit edildi. Mevcut tenant: {_tenantContext.TenantId}");
            }
        }
    }
}
```

### 5. Global Query Filter'ı Bypass Etme (Dikkatli Kullanım)

Yalnızca admin servisleri veya migration araçları gibi özel senaryolarda tüm tenant verilerine erişmek gerekebilir.

```csharp
public class AdminOrderReportService
{
    private readonly TenantAwareDbContext _dbContext;
    private readonly ILogger<AdminOrderReportService> _logger;

    public AdminOrderReportService(
        TenantAwareDbContext dbContext,
        ILogger<AdminOrderReportService> logger)
    {
        _dbContext = dbContext;
        _logger = logger;
    }

    // UYARI: Bu metot tüm tenant verilerini döndürür, yalnızca admin bağlamında kullanın
    public async Task<IReadOnlyList<TenantOrderSummary>> GetAllTenantsOrderSummaryAsync(
        CancellationToken cancellationToken = default)
    {
        _logger.LogWarning("Global query filter bypass edildi - admin raporu");

        return await _dbContext.Orders
            .IgnoreQueryFilters()  // Global Query Filter'ı devre dışı bırak
            .GroupBy(o => o.TenantId)
            .Select(g => new TenantOrderSummary
            {
                TenantId = g.Key,
                OrderCount = g.Count(),
                TotalRevenue = g.Sum(o => o.TotalAmount)
            })
            .ToListAsync(cancellationToken);
    }
}

public class TenantOrderSummary
{
    public string TenantId { get; set; } = string.Empty;
    public int OrderCount { get; set; }
    public decimal TotalRevenue { get; set; }
}
```

### 6. Repository Pattern ile Tenant İzolasyonu

```csharp
public interface ITenantRepository<TEntity> where TEntity : TenantEntity
{
    Task<TEntity?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<IReadOnlyList<TEntity>> GetAllAsync(CancellationToken cancellationToken = default);
    Task<TEntity> AddAsync(TEntity entity, CancellationToken cancellationToken = default);
    Task UpdateAsync(TEntity entity, CancellationToken cancellationToken = default);
    Task DeleteAsync(Guid id, CancellationToken cancellationToken = default);
}

public class TenantRepository<TEntity> : ITenantRepository<TEntity>
    where TEntity : TenantEntity
{
    private readonly TenantAwareDbContext _dbContext;
    private readonly ITenantContext _tenantContext;
    private readonly ILogger<TenantRepository<TEntity>> _logger;

    public TenantRepository(
        TenantAwareDbContext dbContext,
        ITenantContext tenantContext,
        ILogger<TenantRepository<TEntity>> logger)
    {
        _dbContext = dbContext;
        _tenantContext = tenantContext;
        _logger = logger;
    }

    public async Task<TEntity?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
    {
        // Global Query Filter zaten tenant filtresi uygular; ek kontrol ekliyoruz
        var entity = await _dbContext.Set<TEntity>()
            .FirstOrDefaultAsync(e => e.Id == id, cancellationToken);

        if (entity is not null && entity.TenantId != _tenantContext.TenantId)
        {
            // Bu durum Global Query Filter düzgün çalışıyorsa yaşanmamalı
            _logger.LogCritical(
                "Güvenlik ihlali tespit edildi! Entity TenantId: {EntityTenant}, Context TenantId: {ContextTenant}",
                entity.TenantId, _tenantContext.TenantId);
            throw new TenantSecurityException("Farklı tenant kaydına erişim denemesi.");
        }

        return entity;
    }

    public async Task<IReadOnlyList<TEntity>> GetAllAsync(CancellationToken cancellationToken = default)
    {
        return await _dbContext.Set<TEntity>().ToListAsync(cancellationToken);
    }

    public async Task<TEntity> AddAsync(TEntity entity, CancellationToken cancellationToken = default)
    {
        entity.TenantId = _tenantContext.TenantId;
        entity.CreatedAt = DateTime.UtcNow;

        await _dbContext.Set<TEntity>().AddAsync(entity, cancellationToken);
        await _dbContext.SaveChangesAsync(cancellationToken);

        return entity;
    }

    public async Task UpdateAsync(TEntity entity, CancellationToken cancellationToken = default)
    {
        if (entity.TenantId != _tenantContext.TenantId)
        {
            throw new TenantSecurityException("Farklı tenant kaydı güncelleme denemesi.");
        }

        entity.UpdatedAt = DateTime.UtcNow;
        _dbContext.Set<TEntity>().Update(entity);
        await _dbContext.SaveChangesAsync(cancellationToken);
    }

    public async Task DeleteAsync(Guid id, CancellationToken cancellationToken = default)
    {
        var entity = await GetByIdAsync(id, cancellationToken);

        if (entity is null)
        {
            return;
        }

        _dbContext.Set<TEntity>().Remove(entity);
        await _dbContext.SaveChangesAsync(cancellationToken);
    }
}
```

### 7. Çoklu Tenant Migrasyonu

Database-per-tenant senaryosunda migration'ların tüm tenant veritabanlarına uygulanması gerekir.

```csharp
public class TenantMigrationService
{
    private readonly ITenantStore _tenantStore;
    private readonly TenantDbContextFactory _dbContextFactory;
    private readonly ILogger<TenantMigrationService> _logger;

    public TenantMigrationService(
        ITenantStore tenantStore,
        TenantDbContextFactory dbContextFactory,
        ILogger<TenantMigrationService> logger)
    {
        _tenantStore = tenantStore;
        _dbContextFactory = dbContextFactory;
        _logger = logger;
    }

    public async Task MigrateAllTenantsAsync(CancellationToken cancellationToken = default)
    {
        var tenants = await _tenantStore.GetAllActiveAsync(cancellationToken);

        _logger.LogInformation("{Count} tenant için migration başlatılıyor.", tenants.Count);

        var failedTenants = new List<string>();

        foreach (var tenant in tenants)
        {
            try
            {
                await MigrateTenantAsync(tenant.Id, cancellationToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Tenant migration başarısız: {TenantId}", tenant.Id);
                failedTenants.Add(tenant.Id);
            }
        }

        if (failedTenants.Count > 0)
        {
            _logger.LogError("Migration başarısız olan tenant'lar: {FailedTenants}",
                string.Join(", ", failedTenants));
        }
        else
        {
            _logger.LogInformation("Tüm tenant migration'ları başarıyla tamamlandı.");
        }
    }

    private async Task MigrateTenantAsync(string tenantId, CancellationToken cancellationToken)
    {
        _logger.LogInformation("Tenant migration başlıyor: {TenantId}", tenantId);

        await using var dbContext = await _dbContextFactory.CreateAsync(tenantId, cancellationToken);
        await dbContext.Database.MigrateAsync(cancellationToken);

        _logger.LogInformation("Tenant migration tamamlandı: {TenantId}", tenantId);
    }
}
```

## Best Practices

### 1. İzolasyon Stratejisi Seçimi
- Yüksek güvenlik gereksinimleri → Database-per-tenant
- Orta ölçekli SaaS → Schema-per-tenant
- Düşük maliyetli büyük ölçek → Row-level isolation
- Karma yaklaşım: Büyük tenant'lara database, küçüklere shared schema

### 2. Row-Level Isolation Güvenliği
- Global Query Filters'ı hiçbir zaman devre dışı bırakmayın (üretimde)
- `IgnoreQueryFilters()` kullanımını audit logging ile izleyin
- Tüm entity'ler `TenantEntity` base sınıfından miras almalı
- Integration testlerde tenant sızıntısı senaryolarını test edin

### 3. Veritabanı İndeksleme
- Row-level isolation'da her tabloda `TenantId` üzerine composite index ekleyin
- `(TenantId, CreatedAt)`, `(TenantId, Status)` gibi sık kullanılan sorgular için composite index
- Partition key olarak `TenantId` kullanmayı değerlendirin

### 4. Bağlantı Yönetimi
- Database-per-tenant'da connection pool boyutunu sınırlayın
- Bağlantı havuzu konfigürasyonunu tenant plan'ına göre belirleyin (Enterprise daha büyük havuz)

## Sık Sorulan Sorular

### 1. Soru: EF Core Global Query Filters güvenli midir? Bypass edilebilir mi?

**Cevap:** Global Query Filters, EF Core LINQ sorguları üzerinde çalışır ve LINQ → SQL dönüşümünde otomatik olarak `WHERE TenantId = @id` koşulunu ekler. Ancak şu senaryolarda bypass edilebilir:
- `IgnoreQueryFilters()` çağrısı
- Raw SQL (`FromSqlRaw`, `ExecuteSqlRaw`) ile doğrudan sorgu
- Stored procedure çağrıları

Bu nedenle repository katmanında ikincil doğrulama kontrolü eklemek ve raw SQL kullanımını kısıtlamak önerilir.

**Örnek Kod:**
```csharp
// Güvensiz - Global Query Filter'ı bypass eder
var orders = await _dbContext.Orders
    .IgnoreQueryFilters()
    .Where(o => o.Status == OrderStatus.Pending)
    .ToListAsync();

// Güvenli - Global Query Filter aktif
var orders = await _dbContext.Orders
    .Where(o => o.Status == OrderStatus.Pending)
    .ToListAsync();
```

### 2. Soru: Database-per-tenant'ta migration nasıl yönetilir?

**Cevap:** Her tenant'ın veritabanında migration'ların ayrı ayrı çalıştırılması gerekir. Bu süreç `IHostedService` veya deployment pipeline'ının bir parçası olarak otomatize edilir. Migration başarısız olan tenant'lar loglanır ve rollback mekanizması hazırlanır. Yeni tenant ekleme sürecine migration adımı entegre edilir.

**Örnek Kod:**
```csharp
// Uygulama başlarken tüm tenant migration'larını çalıştır
public class MigrationHostedService : IHostedService
{
    private readonly TenantMigrationService _migrationService;

    public MigrationHostedService(TenantMigrationService migrationService)
    {
        _migrationService = migrationService;
    }

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        await _migrationService.MigrateAllTenantsAsync(cancellationToken);
    }

    public Task StopAsync(CancellationToken cancellationToken) => Task.CompletedTask;
}
```

### 3. Soru: Row-level isolation'da performans sorunları nasıl çözülür?

**Cevap:** Büyük tenant'ların küçük tenant performansını olumsuz etkilemesine "noisy neighbor" problemi denir. Çözüm yolları:
- `TenantId` üzerine partition veya composite index
- Büyüyen tenant'ları ayrı veritabanına taşıma (tenant off-boarding)
- Tenant bazlı sorgu kota limitleri
- Read replica'ları büyük tenant'lara yönlendirme

**Örnek Kod:**
```csharp
// Composite index ile sorgu performansı artırma
modelBuilder.Entity<Order>()
    .HasIndex(o => new { o.TenantId, o.Status, o.CreatedAt })
    .HasDatabaseName("IX_Orders_TenantId_Status_CreatedAt");
```

### 4. Soru: Tenant silindiğinde veri nasıl temizlenir?

**Cevap:**
- **Database-per-tenant**: Veritabanı tamamen silinir. Hızlı ve temiz.
- **Schema-per-tenant**: Şema `DROP SCHEMA ... CASCADE` ile silinir.
- **Row-level isolation**: `DELETE FROM ... WHERE TenantId = @id` sorguları çalıştırılır. Büyük tablolarda bu işlem uzun sürebilir; batch silme tercih edilir.

**Örnek Kod:**
```csharp
public class TenantDeletionService
{
    private readonly TenantAwareDbContext _dbContext;

    public TenantDeletionService(TenantAwareDbContext dbContext)
    {
        _dbContext = dbContext;
    }

    public async Task DeleteTenantDataAsync(string tenantId, CancellationToken cancellationToken = default)
    {
        const int batchSize = 1000;

        // Batch silme - büyük tablolar için performans dostu
        int deleted;
        do
        {
            deleted = await _dbContext.Orders
                .IgnoreQueryFilters()
                .Where(o => o.TenantId == tenantId)
                .Take(batchSize)
                .ExecuteDeleteAsync(cancellationToken);
        }
        while (deleted == batchSize);
    }
}
```

### 5. Soru: Tenant'ın bağlantı bilgileri nerede saklanmalıdır?

**Cevap:** Tenant bağlantı bilgileri hassas verilerdir ve aşağıdaki seçenekler değerlendirilir:
- **Azure Key Vault / AWS Secrets Manager**: Üretim için önerilen. Connection string'ler şifreli saklanır.
- **Merkezi "master" veritabanı**: Tenant katalog bilgileri (subdomain, şema adı vb.) burada; actual connection string Key Vault'ta.
- **Ortam değişkenleri**: Küçük ölçekli projeler için kabul edilebilir.

Connection string'ler asla kaynak kodunda ya da version control'de bulunmamalıdır.

### 6. Soru: Tenant'lar arası veri paylaşımı nasıl sağlanır?

**Cevap:** Bazı senaryolarda tenant'ların belirli verileri (örneğin ürün kataloğu) paylaşması gerekir. Bu durum için:
- **Shared schema**: Tenant-agnostik tablolar (örneğin `dbo.ProductCatalog`) ayrı şemada tutulur.
- **TenantId = null veya "shared"**: Row-level isolation'da paylaşılan satırlar için özel bir sentinel değer kullanılır.
- **Global Query Filter güncelleme**: Filtre `e.TenantId == _tenantContext.TenantId || e.TenantId == "shared"` şeklinde genişletilir.

**Örnek Kod:**
```csharp
// Paylaşılan ve tenant'a özgü verileri birlikte sorgulayan filtre
private void SetTenantOrSharedFilter<TEntity>(ModelBuilder modelBuilder)
    where TEntity : TenantEntity
{
    modelBuilder.Entity<TEntity>()
        .HasQueryFilter(e =>
            e.TenantId == _tenantContext.TenantId ||
            e.TenantId == "shared");
}
```

## Kaynaklar

- [EF Core Global Query Filters](https://docs.microsoft.com/en-us/ef/core/querying/filters)
- [Microsoft SaaS Data Isolation Patterns](https://docs.microsoft.com/en-us/azure/azure-sql/database/saas-tenancy-app-design-patterns)
- [Finbuckle.MultiTenant EF Core](https://www.finbuckle.com/MultiTenant/Docs/EFCore)
- [PostgreSQL Row Security Policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- [Azure SQL Elastic Pools](https://docs.microsoft.com/en-us/azure/azure-sql/database/elastic-pool-overview)
