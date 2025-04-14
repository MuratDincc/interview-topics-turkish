# API Development

## Genel Bakış
API (Application Programming Interface) geliştirme, modern yazılım mimarilerinin temel taşlarından biridir. RESTful API'ler, mikroservis mimarileri ve gRPC gibi teknolojiler, farklı sistemlerin birbiriyle iletişim kurmasını sağlar.

## İçindekiler
1. **REST API**
   - REST prensipleri
   - Endpoint tasarımı
   - Resource naming
   - HATEOAS
   - RESTful best practices

2. **HTTP Methods**
   - GET
   - POST
   - PUT
   - PATCH
   - DELETE
   - HEAD
   - OPTIONS

3. **Status Codes**
   - 1xx: Bilgilendirme
   - 2xx: Başarılı
   - 3xx: Yönlendirme
   - 4xx: İstemci Hatası
   - 5xx: Sunucu Hatası

4. **API Versioning**
   - URL versioning
   - Header versioning
   - Media type versioning
   - Versioning stratejileri
   - Breaking changes yönetimi

5. **API Documentation**
   - Swagger/OpenAPI
   - API dokümantasyonu oluşturma
   - Örnek istekler
   - Hata kodları
   - API test araçları

6. **API Güvenliği**
   - Authentication
   - Authorization
   - JWT
   - OAuth
   - API Keys

7. **API Testi**
   - Unit testler
   - Integration testler
   - Postman
   - Load testler
   - Security testler

8. **API Performansı**
   - Caching
   - Rate limiting
   - Compression
   - Pagination
   - Monitoring

## Öğrenme Hedefleri
Bu bölümü tamamladıktan sonra:
- RESTful API tasarım prensiplerini anlayabilecek
- API güvenliği konularında bilgi sahibi olacak
- API dokümantasyonu oluşturabilecek
- API testlerini yazabilecek
- API performans optimizasyonu yapabileceksiniz

## Ön Koşullar
Bu bölümü takip etmek için:
- Temel C# bilgisi
- HTTP protokolü hakkında bilgi
- Visual Studio veya VS Code kullanımı
- .NET Core SDK kurulumu
- Postman veya benzeri API test araçları

## Best Practices
1. **API Tasarımı**
   - REST prensiplerine uyun
   - Anlamlı endpoint isimleri kullanın
   - Versioning stratejisi belirleyin
   - Hata yönetimini standartlaştırın

2. **Güvenlik**
   - HTTPS kullanın
   - Input validasyonu yapın
   - Rate limiting uygulayın
   - Loglama yapın

3. **Performans**
   - Caching stratejisi belirleyin
   - Pagination kullanın
   - Response compression uygulayın
   - Monitoring yapın

## Örnek Proje Yapısı
```
API.Project/
├── Controllers/
│   ├── ProductsController.cs
│   ├── OrdersController.cs
│   └── AuthController.cs
├── Models/
│   ├── DTOs/
│   └── Entities/
├── Services/
│   ├── ProductService.cs
│   └── OrderService.cs
├── Middleware/
│   ├── ExceptionMiddleware.cs
│   └── LoggingMiddleware.cs
└── Tests/
    ├── UnitTests/
    └── IntegrationTests/
```

## Sık Sorulan Sorular
1. REST ve SOAP arasındaki farklar nelerdir?
2. API versiyonlama stratejileri nelerdir?
3. JWT ve OAuth arasındaki farklar nelerdir?
4. API rate limiting nasıl uygulanır?
5. API monitoring nasıl yapılır?

## Kaynaklar
- [REST API Best Practices](https://docs.microsoft.com/tr-tr/azure/architecture/best-practices/api-design)
- [ASP.NET Core Web API](https://docs.microsoft.com/tr-tr/aspnet/core/web-api/)
- [API Security Best Practices](https://docs.microsoft.com/tr-tr/azure/architecture/best-practices/api-security)
- [API Testing](https://docs.microsoft.com/tr-tr/aspnet/core/test/integration-tests) 