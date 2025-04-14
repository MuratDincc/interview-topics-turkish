# Clean Architecture

## Genel Bakış
Clean Architecture, Robert C. Martin tarafından önerilen bir yazılım mimarisi yaklaşımıdır. Bu mimari, uygulamanın farklı katmanlarını birbirinden bağımsız hale getirerek, bağımlılıkları minimize etmeyi ve kodun bakımını kolaylaştırmayı amaçlar.

## Temel Prensipler
1. **Bağımlılık Kuralı**
   - İç katmanlar dış katmanlara bağımlı olmamalıdır
   - Bağımlılıklar her zaman içe doğru olmalıdır
   - Dış katmanlar iç katmanların detaylarını bilmemelidir

2. **Soyutlama Kuralı**
   - İç katmanlar soyutlamaları tanımlar
   - Dış katmanlar bu soyutlamaları uygular
   - Katmanlar arası iletişim interface'ler üzerinden yapılır

3. **Bağımsızlık Kuralı**
   - Her katman bağımsız olarak geliştirilebilir
   - Her katman bağımsız olarak test edilebilir
   - Her katman bağımsız olarak deploy edilebilir

## Katmanlar
1. **Domain Layer (İç Katman)**
   - İş mantığının merkezi
   - Entity'ler ve business rules
   - Dış dünyadan tamamen bağımsız

2. **Application Layer**
   - Use case'lerin implementasyonu
   - Domain layer ile dış katmanlar arası köprü
   - Business logic koordinasyonu

3. **Interface Layer**
   - Kullanıcı arayüzü
   - API endpoints
   - Dış dünya ile iletişim

4. **Infrastructure Layer**
   - Veritabanı işlemleri
   - Harici servis entegrasyonları
   - Framework ve kütüphane kullanımı

## Avantajları
- Bağımlılıkların minimize edilmesi
- Test edilebilirliğin artması
- Kodun bakımının kolaylaştırılması
- Esnek ve ölçeklenebilir yapı
- Framework bağımsızlığı

## Dezavantajları
- Başlangıçta daha fazla kod yazma gerekliliği
- Karmaşık projelerde öğrenme eğrisi
- Performans optimizasyonu gerekliliği
- Over-engineering riski

## Best Practices
1. **Katmanlar Arası İletişim**
   - Interface'ler kullanın
   - DTO'lar ile veri transferi yapın
   - Dependency injection kullanın
   - Event-driven mimariyi tercih edin

2. **Kod Organizasyonu**
   - Her katmanı ayrı projede tutun
   - Feature-based organizasyon yapın
   - SOLID prensiplerini uygulayın
   - DRY prensibine uyun

3. **Test Stratejisi**
   - Unit testler yazın
   - Integration testler yazın
   - Test coverage'ı takip edin
   - Mock ve stub kullanın

## Örnek Proje Yapısı
```
src/
├── Domain/
│   ├── Entities/
│   ├── ValueObjects/
│   ├── Interfaces/
│   └── Exceptions/
├── Application/
│   ├── Services/
│   ├── DTOs/
│   ├── Mappings/
│   └── Validators/
├── Infrastructure/
│   ├── Persistence/
│   ├── ExternalServices/
│   └── Logging/
└── Presentation/
    ├── Controllers/
    ├── Views/
    └── Middleware/
```

## Sık Sorulan Sorular
1. **Clean Architecture ne zaman kullanılmalıdır?**
   - Büyük ve karmaşık projelerde
   - Uzun ömürlü projelerde
   - Sık değişim gerektiren projelerde
   - Test edilebilirlik önemli olduğunda

2. **Clean Architecture ile Microservices arasındaki fark nedir?**
   - Clean Architecture bir mimari yaklaşımıdır
   - Microservices bir deployment stratejisidir
   - İkisi birlikte kullanılabilir
   - Her microservice Clean Architecture ile tasarlanabilir

3. **Clean Architecture performansı nasıl etkiler?**
   - Başlangıçta performans maliyeti olabilir
   - Doğru implementasyon ile minimize edilebilir
   - Caching stratejileri ile optimize edilebilir
   - Asenkron işlemler ile iyileştirilebilir

## Kaynaklar
- [Clean Architecture - Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Microsoft Clean Architecture](https://docs.microsoft.com/tr-tr/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures#clean-architecture)
- [Clean Architecture with ASP.NET Core](https://docs.microsoft.com/tr-tr/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures#clean-architecture) 