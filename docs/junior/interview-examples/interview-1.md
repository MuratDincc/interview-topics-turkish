# Mülakat Örneği 1 - Temel .NET ve C# Kavramları

## 1. .NET Framework ve .NET Core arasındaki temel farklar nelerdir?
**Cevap:** .NET Framework Windows'a özgüdür, .NET Core çapraz platform desteği sunar. .NET Core daha modüler, daha hızlı ve açık kaynaklıdır. Ayrıca, .NET Core container desteği ve mikroservis mimarisi için daha uygundur.
[Detaylı bilgi için tıklayın](../basic-dotnet-concepts/framework-vs-core.md)

## 2. Managed ve Unmanaged code arasındaki fark nedir?
**Cevap:** Managed code CLR tarafından yönetilir ve otomatik bellek yönetimi, tip güvenliği gibi özellikler sunar. Unmanaged code ise doğrudan işletim sistemi üzerinde çalışır ve bellek yönetimi manuel olarak yapılmalıdır.
[Detaylı bilgi için tıklayın](../basic-dotnet-concepts/managed-unmanaged.md)

## 3. Value Type ve Reference Type arasındaki farklar nelerdir?
**Cevap:** Value type'lar stack'te saklanır ve değer kopyalanır, reference type'lar heap'te saklanır ve referans kopyalanır.
[Detaylı bilgi için tıklayın](../csharp-basics/value-reference-types.md)

## 4. String immutability nedir? Neden önemlidir?
**Cevap:** String'ler değişmezdir (immutable), bu sayede thread-safe ve güvenli string işlemleri yapılabilir.
[Detaylı bilgi için tıklayın](../csharp-basics/basic-data-types.md)

## 5. Garbage Collection nasıl çalışır?
**Cevap:** GC, kullanılmayan nesneleri otomatik olarak temizler ve bellek yönetimini sağlar. 0, 1 ve 2 olmak üzere üç nesil kullanır ve her nesil farklı sıklıkta temizlenir.
[Detaylı bilgi için tıklayın](../csharp-basics/memory-management.md)

## 6. String ve StringBuilder arasındaki farklar nelerdir?
**Cevap:** String immutable (değişmez) bir tiptir ve her değişiklik yeni bir string nesnesi oluşturur. StringBuilder mutable (değişebilir) bir tiptir ve string birleştirme işlemlerinde daha performanslıdır.
[Detaylı bilgi için tıklayın](../csharp-basics/basic-data-types.md)

## 7. Exception Handling nasıl yapılır?
**Cevap:** try-catch-finally blokları kullanılarak hata yönetimi yapılır. try bloğunda hata oluşabilecek kod, catch bloğunda hata yakalama ve işleme, finally bloğunda ise her durumda çalışması gereken kod yazılır.
[Detaylı bilgi için tıklayın](../csharp-basics/control-structures.md)

## 8. Interface ve Abstract Class arasındaki farklar nelerdir?
**Cevap:** Interface sadece metod imzalarını tanımlar ve çoklu kalıtım destekler. Abstract Class hem metod imzalarını hem de implementasyonları içerebilir ve tekli kalıtım destekler.
[Detaylı bilgi için tıklayın](../csharp-basics/oop.md)

## 9. SOLID prensipleri nelerdir?
**Cevap:** Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation ve Dependency Inversion prensiplerinden oluşur. Bu prensipler, yazılımın daha sürdürülebilir ve esnek olmasını sağlar.
[Detaylı bilgi için tıklayın](../csharp-basics/oop.md)

## 10. Dependency Injection nedir?
**Cevap:** Bağımlılıkların dışarıdan enjekte edilmesi prensibidir. Bu sayede kod daha test edilebilir, esnek ve bakımı kolay hale gelir.
[Detaylı bilgi için tıklayın](../aspnet-core-basics/dependency-injection.md)

## 11. Async/Await nedir?
**Cevap:** Asenkron programlama için kullanılan bir yapıdır. Uzun süren işlemlerin thread'leri bloke etmeden yapılmasını sağlar.
[Detaylı bilgi için tıklayın](../csharp-basics/control-structures.md)

## 12. LINQ nedir? Ne işe yarar?
**Cevap:** Language Integrated Query, veri sorgulama ve manipülasyonu için kullanılan bir özelliktir. SQL benzeri sorgu yazımına olanak sağlar.
[Detaylı bilgi için tıklayın](../csharp-basics/linq.md)

## 13. Entity Framework nedir?
**Cevap:** ORM (Object-Relational Mapping) aracıdır. Veritabanı işlemlerini nesne yönelimli bir şekilde yapmayı sağlar.
[Detaylı bilgi için tıklayın](../database-operations/entity-framework-core.md)

## 14. REST API nedir?
**Cevap:** Representational State Transfer, web servisleri için bir mimari stildir. HTTP metodlarını kullanarak kaynaklara erişim sağlar.
[Detaylı bilgi için tıklayın](../api-development/rest-api.md)

## 15. JWT nedir?
**Cevap:** JSON Web Token, güvenli bilgi alışverişi için kullanılan bir token formatıdır. Authentication ve Authorization için kullanılır.
[Detaylı bilgi için tıklayın](../security-basics/authentication.md)

## 16. Unit Test nedir?
**Cevap:** Yazılımın en küçük birimlerinin (metod, sınıf) test edilmesidir. Genellikle xUnit, NUnit gibi framework'ler kullanılır.
[Detaylı bilgi için tıklayın](../csharp-basics/control-structures.md)

## 17. Boxing ve Unboxing nedir?
**Cevap:** Boxing value type'ı reference type'a, unboxing ise reference type'ı value type'a dönüştürür.
[Detaylı bilgi için tıklayın](../csharp-basics/boxing-unboxing.md)

## 18. Nullable Types nedir? Nasıl kullanılır?
**Cevap:** Value type'lara null değer atanabilmesini sağlar, ? operatörü ile tanımlanır.
[Detaylı bilgi için tıklayın](../csharp-basics/nullable-types.md)

## 19. var, dynamic ve object arasındaki farklar nelerdir?
**Cevap:** var derleme zamanında tip çıkarımı yapar, dynamic çalışma zamanında tip kontrolü yapar, object ise tüm tiplerin temel sınıfıdır.
[Detaylı bilgi için tıklayın](../csharp-basics/basic-data-types.md)

## 20. Extension Methods nedir? Nasıl kullanılır?
**Cevap:** Mevcut tiplere yeni metodlar eklemek için kullanılan bir özelliktir. static bir sınıf içinde static bir metod olarak tanımlanır ve this keyword'ü ile kullanılır.
[Detaylı bilgi için tıklayın](../csharp-basics/oop.md) 