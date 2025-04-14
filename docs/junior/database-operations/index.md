# Veritabanı İşlemleri

## Genel Bakış
Bu bölümde, .NET uygulamalarında veritabanı işlemlerinin nasıl gerçekleştirileceğini, Entity Framework Core kullanımını ve veritabanı yönetiminin temel prensiplerini inceleyeceğiz.

## İçindekiler
1. [Entity Framework Core](entity-framework-core.md)
2. [LINQ](linq.md)
3. [Migrations](migrations.md)
4. [Transactions](transactions.md)
5. [Performance](performance.md)

## Öğrenme Hedefleri
Bu bölümü tamamladığınızda:
- Entity Framework Core'un temel kavramlarını anlayacaksınız
- LINQ sorguları yazabileceksiniz
- Repository Pattern'i uygulayabileceksiniz
- Unit of Work pattern'ini kullanabileceksiniz
- Veritabanı migration'larını yönetebileceksiniz

## Ön Gereksinimler
Bu bölümü takip etmek için:
- C# programlama dili bilgisi
- Temel SQL bilgisi
- .NET Core temel kavramları
- Visual Studio veya VS Code kurulumu
- .NET SDK kurulumu

## Best Practices
1. **Veritabanı Tasarımı**
   - Normalizasyon kuralları
   - İndeksleme stratejileri
   - İlişki tipleri
   - Performans optimizasyonu

2. **Kod Organizasyonu**
   - Repository pattern
   - Unit of work
   - Dependency injection
   - SOLID prensipleri

3. **Performans**
   - Query optimizasyonu
   - Connection pooling
   - Caching stratejileri
   - Batch işlemler

## Örnek Proje Yapısı
```plaintext
DatabaseOperations/
├── Data/
│   ├── Context/
│   │   └── ApplicationDbContext.cs
│   ├── Entities/
│   │   ├── Product.cs
│   │   └── Category.cs
│   └── Repositories/
│       ├── IRepository.cs
│       └── Repository.cs
├── Services/
│   ├── IProductService.cs
│   └── ProductService.cs
└── Migrations/
    └── InitialCreate.cs
```

## Sık Sorulan Sorular
1. **Entity Framework Core nedir ve neden kullanılır?**
   - ORM (Object-Relational Mapping) aracı
   - Veritabanı işlemlerini kolaylaştırır
   - LINQ desteği
   - Cross-platform çalışabilme

2. **Repository Pattern neden önemlidir?**
   - Veritabanı erişimini soyutlar
   - Test edilebilirliği artırır
   - Kod tekrarını önler
   - Bakımı kolaylaştırır

3. **LINQ nedir ve nasıl kullanılır?**
   - Language Integrated Query
   - Veri sorgulama ve manipülasyonu
   - Lambda expressions
   - Extension methods

## Kaynaklar
- [Entity Framework Core Dokümantasyonu](https://docs.microsoft.com/tr-tr/ef/core/)
- [LINQ Dokümantasyonu](https://docs.microsoft.com/tr-tr/dotnet/csharp/programming-guide/concepts/linq/)
- [Repository Pattern](https://docs.microsoft.com/tr-tr/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-design) 