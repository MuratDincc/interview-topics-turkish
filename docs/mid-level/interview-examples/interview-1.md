# Mülakat Örneği 1

## 1. Async/Await kullanırken deadlock'u nasıl önlersiniz?
**Cevap:** `.ConfigureAwait(false)` kullanarak, `Task.Result` veya `Task.Wait()` yerine `await` kullanarak ve async metotları doğru şekilde zincirleyerek.
[Detaylı bilgi için tıklayın](../advanced-csharp/async-await.md)

## 2. Expression Trees kullanarak dinamik sorgular nasıl oluşturulur?
**Cevap:** `Expression<T>` kullanarak, LINQ sorgularını runtime'da oluşturarak ve `IQueryable` ile çalışarak.
[Detaylı bilgi için tıklayın](../advanced-csharp/expression-trees.md)

## 3. Repository Pattern ve Unit of Work Pattern'i birlikte nasıl kullanırsınız?
**Cevap:** Repository'ler veri erişimini soyutlarken, Unit of Work transaction yönetimini ve değişikliklerin toplu kaydedilmesini sağlar.
[Detaylı bilgi için tıklayın](../design-patterns/repository-pattern.md)

## 4. SOLID prensiplerini bir projede nasıl uygularsınız?
**Cevap:** Her prensibi (Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion) projenin farklı katmanlarında uygulayarak.
[Detaylı bilgi için tıklayın](../solid-principles/index.md)

## 5. Clean Architecture'da Domain Layer'ın sorumlulukları nelerdir?
**Cevap:** İş mantığını, entity'leri, value object'leri ve domain event'leri içerir, dış bağımlılıklardan bağımsızdır.
[Detaylı bilgi için tıklayın](../clean-architecture/domain-layer.md)

## 6. Microservice'ler arası iletişimde hangi pattern'leri kullanırsınız?
**Cevap:** REST API, gRPC, Message Queue (RabbitMQ, Kafka) ve Event-Driven Architecture.
[Detaylı bilgi için tıklayın](../microservices/service-communication.md)

## 7. API Gateway'in avantajları ve dezavantajları nelerdir?
**Cevap:** Merkezi yönetim, rate limiting, authentication gibi avantajları varken, single point of failure olma riski gibi dezavantajları vardır.
[Detaylı bilgi için tıklayın](../microservices/api-gateway.md)

## 8. Circuit Breaker pattern'i nasıl implemente edilir?
**Cevap:** Polly gibi kütüphaneler kullanılarak, hata durumlarında servisi izole ederek ve fallback mekanizmaları sağlayarak.
[Detaylı bilgi için tıklayın](../microservices/circuit-breaker.md)

## 9. Event Sourcing pattern'i hangi senaryolarda kullanılır?
**Cevap:** Audit trail gerektiren, state değişikliklerinin izlenmesi gereken ve CQRS ile birlikte kullanılan sistemlerde.
[Detaylı bilgi için tıklayın](../microservices/event-sourcing.md)

## 10. Distributed caching stratejileri nelerdir?
**Cevap:** Redis gibi distributed cache sistemleri, cache invalidation stratejileri ve cache-aside pattern.
[Detaylı bilgi için tıklayın](../performance-optimization/caching.md)

## 11. Entity Framework Core'da performans optimizasyonu nasıl yapılır?
**Cevap:** Eager/Lazy loading stratejileri, AsNoTracking kullanımı, raw SQL sorguları ve index optimizasyonları.
[Detaylı bilgi için tıklayın](../performance-optimization/database-optimization.md)

## 12. Memory leak'leri nasıl tespit ve önlersiniz?
**Cevap:** Profiling araçları kullanarak, IDisposable pattern'i uygulayarak ve weak reference'ları kullanarak.
[Detaylı bilgi için tıklayın](../performance-optimization/memory-management.md)

## 13. Async programming'de best practice'ler nelerdir?
**Cevap:** Async void kullanmaktan kaçınma, cancellation token kullanma ve proper exception handling.
[Detaylı bilgi için tıklayın](../performance-optimization/async-programming.md)

## 14. Application profiling nasıl yapılır?
**Cevap:** Visual Studio Profiler, dotTrace, dotMemory gibi araçlar kullanılarak ve performance counter'lar izlenerek.
[Detaylı bilgi için tıklayın](../performance-optimization/profiling.md)

## 15. Distributed locking mekanizmaları nelerdir?
**Cevap:** Redis RedLock, ZooKeeper ve database-based locking gibi çözümler.
[Detaylı bilgi için tıklayın](../architecture/distributed-locking.md)

## 16. Reflection kullanımında dikkat edilmesi gerekenler nelerdir?
**Cevap:** Performance overhead, type safety ve security riskleri.
[Detaylı bilgi için tıklayın](../advanced-csharp/reflection.md)

## 17. Custom attributes nasıl oluşturulur ve kullanılır?
**Cevap:** Attribute sınıfları tanımlanarak ve reflection ile bu attribute'lar okunarak.
[Detaylı bilgi için tıklayın](../advanced-csharp/attributes.md)

## 18. LINQ'da advanced sorgular nasıl yazılır?
**Cevap:** GroupJoin, GroupBy, SelectMany gibi operatörler ve custom extension method'lar kullanılarak.
[Detaylı bilgi için tıklayın](../advanced-csharp/linq-advanced.md)

## 19. Creational design pattern'ler hangi durumlarda kullanılır?
**Cevap:** Factory Method, Abstract Factory, Builder, Singleton ve Prototype pattern'leri uygun senaryolarda.
[Detaylı bilgi için tıklayın](../design-patterns/creational-patterns.md)

## 20. Behavioral design pattern'ler hangi durumlarda kullanılır?
**Cevap:** Observer, Strategy, Command, State ve Chain of Responsibility pattern'leri uygun senaryolarda.
[Detaylı bilgi için tıklayın](../design-patterns/behavioral-patterns.md) 