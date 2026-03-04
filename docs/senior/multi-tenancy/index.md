# Multi-Tenancy

## Genel Bakış

Multi-tenancy, tek bir uygulama örneğinin birden fazla müşteriye (tenant) hizmet verdiği bir mimari yaklaşımdır. Her tenant, verisini ve yapılandırmasını yalıtılmış biçimde kullanırken aynı altyapıyı paylaşır. Bu model, SaaS (Software as a Service) ürünlerinin temel taşını oluşturur ve kaynak kullanımını optimize ederken operasyonel maliyetleri düşürür.

.NET ekosisteminde multi-tenancy uygulamak; tenant çözümleme (resolution), veri izolasyonu, kimlik doğrulama, yetkilendirme ve yapılandırma yönetimi gibi birçok katmanı kapsar. Senior seviye mühendislerin bu katmanları doğru tasarlayabilmesi beklenir.

## Kapsanan Konular

### 1. Veri İzolasyonu (Data Isolation)
Tenant verilerini birbirinden ayırmanın üç temel stratejisi ve bunların EF Core ile uygulanması.

**Öğrenilecekler:**
- Database-per-tenant modeli
- Schema-per-tenant modeli
- Row-level isolation (paylaşımlı tablo) modeli
- EF Core Global Query Filters ile otomatik tenant filtreleme
- Tenant bazlı connection string yönetimi
- Veri izolasyonu güvenlik riskleri

### 2. Tenant Çözümleme (Tenant Resolution)
Gelen HTTP isteğinin hangi tenant'a ait olduğunun belirlenmesi.

**Öğrenilecekler:**
- Subdomain tabanlı çözümleme (`acme.myapp.com`)
- HTTP header tabanlı çözümleme (`X-Tenant-Id`)
- JWT claim tabanlı çözümleme
- ASP.NET Core Middleware ile tenant context yönetimi
- `IHttpContextAccessor` ile tenant bilgisine erişim
- Tenant bulunamadığında hata yönetimi

## Neden Önemli?

### 1. **SaaS Ürün Geliştirme**
- Tek bir deployment ile çok sayıda müşteriye hizmet
- Altyapı maliyetlerini müşteriler arasında paylaştırma
- Merkezi güncelleme ve bakım kolaylığı
- Hızlı onboarding süreçleri

### 2. **Veri Güvenliği ve Uyumluluk**
- Tenant verilerinin sızdırılmaması kritik bir güvenlik gereksinimidir
- GDPR, KVKK gibi düzenlemeler veri yalıtımını zorunlu kılabilir
- Denetim izlerinin (audit trail) tenant bazlı tutulması
- Veri konum gereksinimleri (data residency)

### 3. **Ölçeklenebilirlik**
- Yük artışında yalnızca ilgili tenant'ın kaynakları büyütülebilir
- Büyük tenant'lar için ayrı veritabanı sağlanabilir
- Tenant bazlı performans izleme ve kota yönetimi

### 4. **Operasyonel Verimlilik**
- Tek kod tabanı, tek deployment pipeline
- Tenant başına yapılandırma esnekliği (feature flags, limits)
- Merkezi monitoring ve log yönetimi

## Mimari Modeller

### Tam İzolasyon (Silo Modeli)
Her tenant için ayrı uygulama örneği ve veritabanı. En yüksek izolasyon, en yüksek maliyet.

```
Tenant A → App Instance A → Database A
Tenant B → App Instance B → Database B
```

### Havuz Modeli (Pool Modeli)
Tüm tenant'lar aynı uygulama örneğini ve veritabanını paylaşır. En düşük maliyet, en düşük izolasyon.

```
Tenant A ↘
Tenant B → Shared App Instance → Shared Database (tenant_id kolonu)
Tenant C ↗
```

### Köprü Modeli (Bridge Modeli)
Uygulama paylaşımlı, veritabanı tenant başına ayrı (database-per-tenant) ya da schema-per-tenant.

```
Tenant A ↘                    → Database A
Tenant B → Shared App Instance → Database B
Tenant C ↗                    → Database C
```

## Temel Kavramlar

### ITenantContext Arayüzü
```csharp
public interface ITenantContext
{
    string TenantId { get; }
    string TenantName { get; }
    bool IsResolved { get; }
}

public class TenantContext : ITenantContext
{
    public string TenantId { get; set; } = string.Empty;
    public string TenantName { get; set; } = string.Empty;
    public bool IsResolved { get; set; }
}
```

### Tenant Modeli
```csharp
public class Tenant
{
    public string Id { get; set; } = string.Empty;
    public string Name { get; set; } = string.Empty;
    public string Subdomain { get; set; } = string.Empty;
    public string ConnectionString { get; set; } = string.Empty;
    public bool IsActive { get; set; }
    public TenantPlan Plan { get; set; }
    public DateTime CreatedAt { get; set; }
}

public enum TenantPlan
{
    Free,
    Starter,
    Professional,
    Enterprise
}
```

### ITenantStore Arayüzü
```csharp
public interface ITenantStore
{
    Task<Tenant?> GetByIdAsync(string tenantId, CancellationToken cancellationToken = default);
    Task<Tenant?> GetBySubdomainAsync(string subdomain, CancellationToken cancellationToken = default);
    Task<IReadOnlyList<Tenant>> GetAllActiveAsync(CancellationToken cancellationToken = default);
}
```

## Mülakat Soruları

### 1. Soru: Multi-tenancy nedir ve neden tercih edilir?

**Cevap:** Multi-tenancy, tek bir yazılım örneğinin birden fazla müşteriye (tenant) hizmet verdiği mimari yaklaşımdır. Tercih edilme sebepleri şunlardır: altyapı maliyetleri paylaşılır, kod tabanı tekleşir, merkezi güncelleme sağlanır ve yeni müşteri onboarding'i hızlanır. Özellikle SaaS ürünlerde temel tercih nedeni, ölçek ekonomisinden yararlanmaktır.

### 2. Soru: Single-tenancy ile multi-tenancy arasındaki fark nedir?

**Cevap:** Single-tenancy'de her müşteri için ayrı bir uygulama ve veritabanı kurulumu yapılır; bu en yüksek izolasyonu sağlar ancak maliyeti yüksektir. Multi-tenancy'de kaynaklar paylaşılır; bu maliyet avantajı getirir ancak doğru veri izolasyonu tasarımını zorunlu kılar. Single-tenancy genellikle finans, sağlık gibi yüksek düzenleyici gereksinimleri olan sektörlerde tercih edilir.

### 3. Soru: Veri izolasyon stratejileri nelerdir?

**Cevap:** Üç ana strateji vardır:
- **Database-per-tenant**: Her tenant'ın kendi veritabanı bulunur. En güçlü izolasyon, en yüksek kaynak kullanımı.
- **Schema-per-tenant**: Aynı veritabanı sunucusunda her tenant için ayrı şema. Orta düzey izolasyon.
- **Row-level isolation**: Tek veritabanında tüm tablolara `TenantId` kolonu eklenir. En düşük maliyet, yazılımsal izolasyon gerektirir.

### 4. Soru: Tenant çözümleme (resolution) nedir?

**Cevap:** Her HTTP isteğinin hangi tenant'a ait olduğunun belirlenmesi sürecidir. Yaygın yöntemler: subdomaine göre (`acme.app.com`), HTTP header'a göre (`X-Tenant-Id: acme`), URL path segment'e göre (`/tenants/acme/api/...`) veya JWT token içindeki claim'e göre çözümleme.

### 5. Soru: EF Core'da multi-tenancy nasıl uygulanır?

**Cevap:** EF Core'da `Global Query Filters` özelliği ile her `DbSet` sorgusuna otomatik olarak `WHERE TenantId = @currentTenantId` koşulu eklenir. `DbContext` constructor'ında `ITenantContext` inject edilerek filtreler yapılandırılır. Böylece her sorgu otomatik olarak o anki tenant'a kısıtlanır ve tenant verisi sızıntısı önlenir.

### 6. Soru: Multi-tenancy'de karşılaşılan yaygın güvenlik riskleri nelerdir?

**Cevap:**
- **Tenant sızıntısı**: Yanlış filtreleme sonucu farklı tenant verilerinin görülmesi.
- **Tenant spoofing**: Kötü niyetli kullanıcının başka tenant ID'si ile istek atması.
- **Shared secret exposure**: Ortak kullanılan şifreleme anahtarlarının tüm tenant'ları etkilemesi.
- **Cache poisoning**: Önbellekte tenant-agnostik key kullanılması.

Çözümler: JWT claim doğrulaması, Global Query Filters, tenant-scope cache key'leri ve audit logging.

## Best Practices

### 1. Her Katmanda Tenant Farkındalığı
- HTTP katmanı: Middleware ile tenant çözümleme
- İş mantığı katmanı: Servisler `ITenantContext` inject eder
- Veri katmanı: EF Core Global Query Filters veya ayrı DbContext

### 2. Tenant Bilgisini Scope'a Bağlayın
- `ITenantContext`'i `Scoped` servis olarak kaydedin
- Her istek için yeni bir tenant context oluşturulmasını sağlayın
- Background job'larda tenant context'ini açıkça set edin

### 3. Varsayılan Olarak Reddedin
- Tenant çözümleme başarısız olduğunda isteği reddedin
- Null ya da boş tenant ID ile sorgu çalıştırmayın

### 4. Tenant Verilerini Önbelleklerken Dikkatli Olun
- Cache key'lere `tenantId` ekleyin
- Farklı tenant'ların cache'lerinin birbirini etkilememesini sağlayın

## Kaynaklar

- [Microsoft Multi-tenant SaaS patterns](https://docs.microsoft.com/en-us/azure/azure-sql/database/saas-tenancy-app-design-patterns)
- [Finbuckle.MultiTenant - .NET Kütüphanesi](https://www.finbuckle.com/MultiTenant)
- [EF Core Global Query Filters](https://docs.microsoft.com/en-us/ef/core/querying/filters)
- [ASP.NET Core Middleware](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/)
