# ASP.NET Core Temelleri

## Genel Bakış
Bu bölümde, modern web uygulamaları geliştirmek için kullanılan ASP.NET Core framework'ünün temel kavramlarını ve bileşenlerini inceleyeceğiz.

## İçindekiler
1. [Middleware](middleware.md)
2. [Dependency Injection](dependency-injection.md)
3. [Routing](routing.md)
4. [Model Binding](model-binding.md)
5. [Validation](validation.md)

6. [Authentication](authentication.md)
   - Cookie authentication
   - JWT authentication
   - Identity framework
   - Authorization

## Öğrenme Hedefleri
Bu bölümü tamamladığınızda:
- ASP.NET Core'un temel mimarisini anlayacaksınız
- Middleware pipeline'ını yapılandırabileceksiniz
- Dependency Injection'ı etkin şekilde kullanabileceksiniz
- Routing mekanizmasını yönetebileceksiniz
- Model binding ve validation işlemlerini yapabileceksiniz
- Authentication ve authorization mekanizmalarını uygulayabileceksiniz

## Ön Gereksinimler
Bu bölümü takip etmek için:
- C# programlama dili bilgisi
- .NET Core temel kavramları
- Visual Studio veya VS Code kurulumu
- .NET SDK kurulumu

## Best Practices
1. **Mimari**
   - Clean Architecture
   - SOLID prensipleri
   - Dependency Injection
   - Middleware pipeline

2. **Performans**
   - Response compression
   - Caching
   - Async/await
   - Connection pooling

3. **Güvenlik**
   - Input validation
   - Authentication
   - Authorization
   - HTTPS

## Örnek Proje Yapısı
```plaintext
ASPNetCoreBasics/
├── Controllers/
│   ├── HomeController.cs
│   └── AccountController.cs
├── Models/
│   ├── User.cs
│   └── Product.cs
├── Services/
│   ├── IUserService.cs
│   └── UserService.cs
├── Middleware/
│   └── CustomMiddleware.cs
└── Program.cs
```

## Sık Sorulan Sorular
1. **ASP.NET Core ve ASP.NET Framework arasındaki farklar nelerdir?**
   - Cross-platform desteği
   - Performans iyileştirmeleri
   - Modüler yapı
   - Dependency Injection

2. **Middleware nedir ve ne işe yarar?**
   - Request pipeline bileşeni
   - Sıralı işlem
   - Request/Response manipülasyonu
   - Authentication/Authorization

3. **Dependency Injection neden önemlidir?**
   - Loose coupling
   - Test edilebilirlik
   - Kod organizasyonu
   - Service lifetime yönetimi

## Kaynaklar
- [ASP.NET Core Dokümantasyonu](https://docs.microsoft.com/tr-tr/aspnet/core/)
- [ASP.NET Core Fundamentals](https://docs.microsoft.com/tr-tr/aspnet/core/fundamentals/)
- [ASP.NET Core Best Practices](https://docs.microsoft.com/tr-tr/dotnet/architecture/modern-web-apps-azure/) 