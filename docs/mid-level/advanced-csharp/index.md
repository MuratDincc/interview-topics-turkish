# Advanced C#

## Genel Bakış
Advanced C# konuları, C# programlama dilinin daha ileri seviye özelliklerini ve kavramlarını içerir. Bu bölüm, orta ve ileri seviye C# geliştiricilerinin bilmesi gereken konuları kapsar.

## İçindekiler
1. **Async/Await**
   - Asenkron programlama temelleri
   - Task ve async/await kullanımı
   - Asenkron metodların yönetimi
   - Hata yönetimi

2. **LINQ Advanced**
   - İleri seviye LINQ sorguları
   - Performans optimizasyonu
   - Custom extension metodlar
   - Query syntax vs Method syntax

3. **Reflection**
   - Type bilgisi ve metadata
   - Runtime type discovery
   - Dynamic code execution
   - Assembly yönetimi

4. **Attributes**
   - Custom attribute oluşturma
   - Attribute kullanımı
   - Reflection ile attribute okuma
   - Attribute best practices

5. **Expression Trees**
   - Expression tree yapısı
   - Dynamic query oluşturma
   - Code generation
   - Performance considerations

6. **Dependency Injection**
   - DI container'lar
   - Service lifetime
   - Constructor injection
   - Property injection

7. **Memory Management**
   - Garbage collection
   - IDisposable pattern
   - Memory leaks
   - Performance optimization

8. **Design Patterns**
   - Creational patterns
   - Structural patterns
   - Behavioral patterns
   - SOLID principles

9. **Testing**
   - Unit testing
   - Integration testing
   - Mocking
   - Test driven development

## Öğrenme Hedefleri
Bu bölümü tamamladıktan sonra şunları yapabileceksiniz:
- Asenkron programlama kavramlarını anlama ve uygulama
- LINQ sorgularını etkin şekilde kullanma
- Dependency Injection prensiplerini uygulama
- Reflection ve dynamic özelliklerini kullanma
- Memory yönetimi ve performans optimizasyonu yapma
- Design pattern'leri uygun şekilde uygulama
- Test driven development yaklaşımını benimseme

## Ön Koşullar
Bu bölümü takip etmek için:
- Temel C# bilgisi
- Nesne yönelimli programlama kavramları
- .NET framework temelleri
- Visual Studio veya VS Code kullanımı

## Best Practices
1. **Kod Kalitesi**
   - Clean code prensipleri
   - SOLID prensipleri
   - Code review süreçleri
   - Documentation

2. **Performans**
   - Memory optimizasyonu
   - Asenkron operasyonlar
   - Caching stratejileri
   - Profiling

3. **Testability**
   - Unit test coverage
   - Mocking stratejileri
   - Integration testing
   - Continuous integration

## Örnek Proje Yapısı
```
AdvancedCSharp/
├── src/
│   ├── Core/
│   │   ├── Entities/
│   │   ├── Interfaces/
│   │   └── Services/
│   ├── Infrastructure/
│   │   ├── Data/
│   │   ├── Logging/
│   │   └── Security/
│   └── Web/
│       ├── Controllers/
│       ├── Models/
│       └── Views/
├── tests/
│   ├── UnitTests/
│   ├── IntegrationTests/
│   └── TestHelpers/
└── docs/
    ├── architecture/
    ├── api/
    └── deployment/
```

## Sık Sorulan Sorular
1. **Asenkron programlama ne zaman kullanılmalı?**
   - I/O operasyonlarında
   - Uzun süren işlemlerde
   - Paralel işlem gerektiren durumlarda

2. **LINQ performansı nasıl optimize edilir?**
   - Lazy evaluation kullanımı
   - İndeksleme
   - Caching
   - Query optimization

3. **Dependency Injection ne zaman kullanılmalı?**
   - Test edilebilirlik gerektiğinde
   - Loose coupling istendiğinde
   - Service lifetime yönetimi gerektiğinde

4. **Memory leak'ler nasıl önlenir?**
   - IDisposable pattern kullanımı
   - Event handler'ların düzgün temizlenmesi
   - Weak references kullanımı
   - Memory profiling

## Kaynaklar
- [Microsoft C# Documentation](https://docs.microsoft.com/tr-tr/dotnet/csharp/)
- [C# Design Patterns](https://refactoring.guru/design-patterns/csharp)
- [Async/Await Best Practices](https://docs.microsoft.com/tr-tr/dotnet/csharp/programming-guide/concepts/async/)
- [SOLID Principles](https://www.digitalocean.com/community/conceptual-articles/s-o-l-i-d-the-first-five-principles-of-object-oriented-design) 