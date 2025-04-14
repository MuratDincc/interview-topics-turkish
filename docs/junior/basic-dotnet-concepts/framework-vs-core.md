# .NET Framework vs .NET Core

## Genel Bakış
Bu bölümde, .NET Framework ve .NET Core arasındaki temel farkları, kullanım senaryolarını ve geçiş stratejilerini inceleyeceğiz.

## Mülakat Soruları ve Cevapları

### 1. .NET Framework ve .NET Core arasındaki temel mimari farklar nelerdir?
**Cevap:**
- **.NET Framework:**
  - Monolitik yapı
  - Windows'a bağımlı
  - GAC (Global Assembly Cache) kullanır
  - System.Web bağımlılığı
  - IIS gerektirir

- **.NET Core:**
  - Modüler yapı
  - Cross-platform
  - NuGet paket yönetimi
  - Self-contained deployment
  - Kestrel web sunucusu

**Örnek Kod:**
```csharp
// .NET Framework Web.config
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.web>
    <compilation debug="true" targetFramework="4.7.2" />
  </system.web>
</configuration>

// .NET Core appsettings.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  }
}
```

### 2. .NET Core'un performans avantajları nelerdir?
**Cevap:**
- Daha hızlı başlangıç süresi
- Daha az memory kullanımı
- Daha iyi ölçeklenebilirlik
- Asenkron operasyonlarda daha iyi performans
- Container desteği ile daha iyi kaynak yönetimi

**Örnek Kod:**
```csharp
// .NET Core'da performans optimizasyonu
public class PerformanceExample
{
    // Span<T> kullanımı
    public void ProcessData(ReadOnlySpan<byte> data)
    {
        // Düşük seviyeli memory erişimi
    }

    // ValueTask kullanımı
    public async ValueTask<int> GetDataAsync()
    {
        // Asenkron operasyon optimizasyonu
        return await Task.FromResult(42);
    }
}
```

### 3. .NET Framework'ten .NET Core'a geçiş stratejileri nelerdir?
**Cevap:**
1. Analiz aşaması:
   - Bağımlılıkların tespiti
   - Windows-specific kodların belirlenmesi
   - Third-party kütüphanelerin kontrolü

2. Geçiş planı:
   - Adım adım geçiş
   - Side-by-side deployment
   - Feature flag kullanımı

3. Test stratejisi:
   - Unit testlerin güncellenmesi
   - Integration testlerinin yazılması
   - Performance testlerinin yapılması

**Örnek Kod:**
```csharp
// .NET Framework'ten .NET Core'a geçiş örneği
public class MigrationExample
{
    // Eski .NET Framework kodu
    public void OldMethod()
    {
        using (var connection = new SqlConnection("connectionString"))
        {
            // ADO.NET kullanımı
        }
    }

    // Yeni .NET Core kodu
    public async Task NewMethod()
    {
        using var connection = new SqlConnection("connectionString");
        await connection.OpenAsync();
        // Dapper veya Entity Framework Core kullanımı
    }
}
```

### 4. .NET Core'un container desteği nasıl çalışır?
**Cevap:**
- Docker desteği
- Microservice mimarisi için uygunluk
- Container orchestration (Kubernetes) entegrasyonu
- Environment-based configuration
- Health check desteği

**Örnek Kod:**
```csharp
// .NET Core Docker desteği
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}

// Dockerfile örneği
FROM mcr.microsoft.com/dotnet/aspnet:5.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443
```

### 5. .NET Core'da dependency injection nasıl çalışır?
**Cevap:**
- Built-in DI container
- Service lifetime yönetimi (Singleton, Scoped, Transient)
- Constructor injection
- Interface-based programming
- Configuration binding

**Örnek Kod:**
```csharp
// Startup.cs
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // Singleton service
        services.AddSingleton<ICacheService, CacheService>();
        
        // Scoped service
        services.AddScoped<IUserService, UserService>();
        
        // Transient service
        services.AddTransient<IEmailService, EmailService>();
    }
}

// Controller kullanımı
public class UserController : ControllerBase
{
    private readonly IUserService _userService;
    
    public UserController(IUserService userService)
    {
        _userService = userService;
    }
}
```

## Best Practices
1. **Geçiş Stratejisi**
   - Incremental migration
   - Feature toggle kullanımı
   - A/B testing
   - Rollback planı

2. **Performans Optimizasyonu**
   - Async/await pattern
   - Memory management
   - Caching stratejileri
   - Connection pooling

3. **Güvenlik**
   - HTTPS enforcement
   - CORS yapılandırması
   - Authentication/Authorization
   - Security headers

## Kaynaklar
- [.NET Core Documentation](https://docs.microsoft.com/tr-tr/dotnet/core/)
- [.NET Framework to .NET Core Migration Guide](https://docs.microsoft.com/tr-tr/dotnet/core/porting/)
- [.NET Core Performance](https://docs.microsoft.com/tr-tr/dotnet/core/performance/) 