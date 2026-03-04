# CQRS & MediatR

## Giriş

CQRS (Command Query Responsibility Segregation) ve MediatR, modern .NET uygulamalarında yaygın olarak kullanılan mimari pattern ve kütüphanelerdir. CQRS, okuma (Query) ve yazma (Command) işlemlerini birbirinden ayırarak uygulamanın karmaşıklığını azaltır ve ölçeklenebilirliği artırır. MediatR ise bu pattern'i .NET ekosisteminde uygulayan, bileşenler arası bağımlılıkları gevşeten (loose coupling) bir kütüphanedir. Mid-level geliştiriciler için bu konuları kavramak; temiz mimari, bakımı kolay kod ve kurumsal uygulamalar geliştirmede kritik öneme sahiptir.

## Kapsanan Konular

### 1. CQRS Pattern
Command ve Query'nin sorumluluk ayrımı, ayrı okuma/yazma modelleri ve nihai tutarlılık (eventual consistency).

**Öğrenilecekler:**
- Command ve Query ayrımının temel mantığı
- Ayrı Read Model ve Write Model tasarımı
- Eventual consistency ve senkronizasyon stratejileri
- C# ile CQRS implementasyonu
- CQRS'in Clean Architecture ile entegrasyonu
- Komut ve sorgu handler'larının bağımsız geliştirilmesi

### 2. MediatR Pipeline
IRequest, IRequestHandler, Pipeline Behavior'lar (doğrulama, loglama), Bildirimler (Notifications) ve MediatR'ın .NET DI ile kullanımı.

**Öğrenilecekler:**
- IRequest ve IRequestHandler arayüzleri
- MediatR ile mediator pattern implementasyonu
- IPipelineBehavior ile cross-cutting concern yönetimi
- Validation pipeline behavior (FluentValidation entegrasyonu)
- Logging pipeline behavior
- INotification ve INotificationHandler ile yayın/abonelik modeli

## Neden Önemli?

### 1. **Separation of Concerns (Endişelerin Ayrılması)**
- Okuma ve yazma işlemleri bağımsız olarak evrilebilir
- Her handler tek bir sorumluluğa odaklanır
- İş mantığı, altyapı kodundan ayrışır
- Kod okunabilirliği ve anlaşılabilirliği artar

### 2. **Ölçeklenebilirlik**
- Okuma tarafı (Query side) bağımsız olarak ölçeklendirilebilir
- Yazma tarafı (Command side) ayrı optimize edilebilir
- Veritabanı okuma/yazma replikaları kolayca entegre edilebilir
- Yüksek trafikli uygulamalarda performans kazanımı sağlanır

### 3. **Test Edilebilirlik**
- Her handler izole birim testlerle test edilebilir
- Mock bağımlılıkları kolayca oluşturulabilir
- Pipeline behavior'lar ayrı ayrı test edilebilir
- Entegrasyon testleri daha sade yazılır

### 4. **Bakım Kolaylığı**
- Yeni özellikler mevcut kodu bozmadan eklenebilir
- Handler'lar küçük ve odaklı kalır
- Cross-cutting concern'ler merkezi olarak yönetilir
- Ekip içinde görev paylaşımı kolaylaşır

## Mülakat Soruları

### Temel Sorular

1. **CQRS nedir ve ne anlama gelir?**
   - **Cevap**: Command Query Responsibility Segregation; okuma (Query) ve yazma (Command) operasyonlarının ayrı model ve handler'larla yönetildiği bir mimari pattern'dir.

2. **CQRS ile geleneksel CRUD arasındaki fark nedir?**
   - **Cevap**: CRUD'da tek bir model hem okuma hem yazmayı yönetir; CQRS'te Command ve Query tarafı birbirinden tamamen ayrılır, farklı modeller ve handler'lar kullanılır.

3. **MediatR nedir ve ne işe yarar?**
   - **Cevap**: MediatR, .NET için bir mediator pattern implementasyonudur; bileşenler arasındaki doğrudan bağımlılıkları ortadan kaldırır, istek ve handler'ları birbirinden soyutlar.

4. **Pipeline Behavior nedir?**
   - **Cevap**: MediatR'da bir isteğin işlenmeden önce ve sonra ara işlem yapılmasını sağlayan katmanlı yapıdır; doğrulama, loglama ve transaction yönetimi için kullanılır.

5. **Eventual consistency nedir?**
   - **Cevap**: Yazma tarafında yapılan değişikliklerin okuma tarafına anlık değil, belirli bir gecikme ile yansıtıldığı tutarlılık modelidir.

### Teknik Sorular

1. **IRequest ve IRequestHandler nasıl kullanılır?**
   - **Cevap**: IRequest, isteği (komut veya sorgu) temsil eden sınıf; IRequestHandler ise bu isteği işleyip yanıt döndüren handler sınıfı için kullanılan generic arayüzlerdir.

2. **MediatR'da Notification ile Request arasındaki fark nedir?**
   - **Cevap**: Request tek bir handler tarafından işlenir ve yanıt döndürür; Notification birden fazla handler tarafından işlenebilir, fire-and-forget senaryolarına uygundur.

3. **CQRS'i Event Sourcing ile birlikte kullanmanın avantajları nelerdir?**
   - **Cevap**: Command tarafındaki her değişiklik event olarak saklanır; bu sayede audit trail, zaman yolculuğu (time travel) ve okuma modeli yeniden oluşturma mümkün olur.

## Best Practices

### 1. **Handler Tasarımı**
- Her handler tek bir işe odaklanmalıdır
- Handler'lar ince (thin) tutulmalı, iş mantığı domain katmanına taşınmalıdır
- Bağımlılıklar constructor injection ile sağlanmalıdır
- Handler'lar asenkron (async/await) yazılmalıdır

### 2. **Pipeline Davranışları**
- Doğrulama (Validation) pipeline en başa yerleştirilmelidir
- Loglama pipeline tüm istekleri kapsamalıdır
- Transaction yönetimi Command pipeline'larına eklenmeli, Query'lerde kullanılmamalıdır
- Exception handling pipeline merkezi hata yönetimini sağlamalıdır

### 3. **Model Tasarımı**
- Command ve Query modelleri birbirinden bağımsız tutulmalıdır
- Read Model'ler denormalize ve sorgu-optimized olmalıdır
- Write Model'ler domain nesneleri ve iş kurallarını korumalıdır
- DTO'lar iç modelden ayırt edilmelidir

### 4. **Hata Yönetimi**
- Handler'lardan exception fırlatmak yerine Result pattern tercih edilmelidir
- Doğrulama hataları ValidationException ile merkezi olarak yönetilmelidir
- Pipeline'da global exception handling kurulmalıdır
- Hata mesajları anlamlı ve izlenebilir olmalıdır

## Kaynaklar

- [MediatR GitHub](https://github.com/jbogard/MediatR)
- [CQRS - Martin Fowler](https://martinfowler.com/bliki/CQRS.html)
- [Microsoft CQRS Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [Clean Architecture with CQRS](https://jasontaylor.dev/clean-architecture-getting-started/)
- [MediatR Pipeline Behaviors](https://github.com/jbogard/MediatR/wiki/Behaviors)
