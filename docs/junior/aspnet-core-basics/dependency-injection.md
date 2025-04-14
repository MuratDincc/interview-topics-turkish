# Dependency Injection

## Genel Bakış
Dependency Injection (DI), nesnelerin bağımlılıklarını dışarıdan almasını sağlayan bir tasarım desenidir. ASP.NET Core'da built-in DI container bulunur ve bu sayede loose coupling, test edilebilirlik ve modülerlik sağlanır.

## Mülakat Soruları ve Cevapları

### 1. Dependency Injection nedir ve neden kullanılır?
**Cevap:**
Dependency Injection, bir nesnenin bağımlılıklarının dışarıdan verilmesi prensibidir. Kullanım nedenleri:
- Loose coupling sağlar
- Test edilebilirliği artırır
- Kod bakımını kolaylaştırır
- Modüler yapıyı destekler

**Örnek Kod:**
```csharp
// DI kullanmadan
public class UserService
{
    private readonly UserRepository _repository = new UserRepository();
}

// DI ile
public class UserService
{
    private readonly IUserRepository _repository;

    public UserService(IUserRepository repository)
    {
        _repository = repository;
    }
}
```

### 2. Service lifetimes nelerdir?
**Cevap:**
ASP.NET Core'da üç farklı service lifetime vardır:
- **Singleton**: Uygulama yaşam döngüsü boyunca tek instance
- **Scoped**: Her HTTP isteği için yeni instance
- **Transient**: Her servis talebinde yeni instance

**Örnek Kod:**
```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddSingleton<ICacheService, CacheService>();
    services.AddScoped<IUserService, UserService>();
    services.AddTransient<IEmailService, EmailService>();
}
```

### 3. Constructor injection ve method injection arasındaki farklar nelerdir?
**Cevap:**
Constructor injection:
- Sınıf seviyesinde bağımlılık
- Tüm metodlar için kullanılabilir
- Daha yaygın kullanım
- Test edilebilirlik açısından avantajlı

Method injection:
- Metod seviyesinde bağımlılık
- Sadece ilgili metod için kullanılır
- Daha az yaygın
- Geçici bağımlılıklar için uygun

**Örnek Kod:**
```csharp
// Constructor injection
public class UserService
{
    private readonly IUserRepository _repository;

    public UserService(IUserRepository repository)
    {
        _repository = repository;
    }
}

// Method injection
public class UserService
{
    public void SendEmail(IEmailService emailService, string message)
    {
        emailService.Send(message);
    }
}
```

### 4. Service collection'a servis nasıl kaydedilir?
**Cevap:**
Servisler ConfigureServices metodunda kaydedilir:
- AddSingleton: Tek instance
- AddScoped: Request başına instance
- AddTransient: Her talepte instance
- AddScoped vs AddTransient kullanım senaryoları

**Örnek Kod:**
```csharp
public void ConfigureServices(IServiceCollection services)
{
    // Interface-implementation mapping
    services.AddScoped<IUserService, UserService>();
    
    // Concrete type
    services.AddSingleton<CacheService>();
    
    // Factory method
    services.AddTransient<IService>(sp => 
        new Service(sp.GetRequiredService<ILogger>()));
}
```

### 5. DI container'dan servis nasıl resolve edilir?
**Cevap:**
Servisler şu yollarla resolve edilebilir:
- Constructor injection (önerilen)
- IServiceProvider kullanımı
- [FromServices] attribute'u
- ActivatorUtilities

**Örnek Kod:**
```csharp
// Constructor injection
public class HomeController : Controller
{
    private readonly IUserService _userService;

    public HomeController(IUserService userService)
    {
        _userService = userService;
    }
}

// IServiceProvider
public class CustomMiddleware
{
    public async Task InvokeAsync(HttpContext context, IServiceProvider serviceProvider)
    {
        var service = serviceProvider.GetRequiredService<IService>();
    }
}
```

## Best Practices
1. **Servis Kaydı**
   - Interface kullanın
   - Doğru lifetime seçin
   - Servisleri modüler tutun
   - Circular dependency'den kaçının

2. **Performans**
   - Gereksiz servis kaydı yapmayın
   - Singleton servisleri thread-safe yapın
   - Memory leak'leri önleyin
   - Dispose pattern'i uygulayın

3. **Test Edilebilirlik**
   - Mock'lanabilir servisler tasarlayın
   - Interface'leri küçük tutun
   - Bağımlılıkları minimize edin
   - Unit test yazın

## Kaynaklar
- [ASP.NET Core Dependency Injection](https://docs.microsoft.com/tr-tr/aspnet/core/fundamentals/dependency-injection)
- [Dependency Injection Guidelines](https://docs.microsoft.com/tr-tr/dotnet/core/extensions/dependency-injection-guidelines)
- [Service Lifetimes](https://docs.microsoft.com/tr-tr/dotnet/core/extensions/dependency-injection#service-lifetimes) 