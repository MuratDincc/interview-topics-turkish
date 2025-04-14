# C# Temelleri

## Genel Bakış
Bu bölümde, C# programlama dilinin temel kavramlarını, veri tiplerini, operatörleri ve kontrol yapılarını inceleyeceğiz. Bu konular, C# ile geliştirme yaparken sıkça karşılaşacağınız temel yapı taşlarıdır.

## İçindekiler
1. [Temel Veri Tipleri](basic-data-types.md)
   - Value Types
   - Reference Types
   - Nullable Types
   - Tip Dönüşümleri

2. [Kontrol Yapıları](control-structures.md)
   - If-Else
   - Switch-Case
   - Döngüler (for, while, do-while)
   - Break ve Continue

3. [Nesne Yönelimli Programlama](oop.md)
   - Sınıflar ve Nesneler
   - Kalıtım
   - Polimorfizm
   - Kapsülleme
   - Soyutlama

4. [Koleksiyonlar](collections.md)
   - List
   - Dictionary
   - Array
   - LINQ

5. [Delegates ve Events](delegates-events.md)
   - Delegate Tanımlama
   - Event Tanımlama
   - Lambda Expressions
   - Event Handling

## Öğrenme Hedefleri
Bu bölümü tamamladığınızda:
- C#'ın temel veri tiplerini ve değişken kavramını anlayacaksınız
- Operatörleri ve kontrol yapılarını etkin şekilde kullanabileceksiniz
- Nesne yönelimli programlama prensiplerini uygulayabileceksiniz
- Metotları ve parametreleri doğru şekilde kullanabileceksiniz
- Koleksiyonları ve LINQ sorgularını yazabileceksiniz

## Ön Gereksinimler
Bu bölümü takip etmek için:
- Temel programlama bilgisi
- Visual Studio veya VS Code kurulumu
- .NET SDK kurulumu

## Best Practices
1. **Kod Organizasyonu**
   - Anlamlı değişken isimleri kullanın
   - Kodunuzu mantıksal bölümlere ayırın
   - Yorum satırları ekleyin
   - Kod tekrarından kaçının

2. **Performans**
   - Uygun veri tiplerini seçin
   - Gereksiz dönüşümlerden kaçının
   - Koleksiyonları doğru kullanın
   - Memory leak'lerden kaçının

3. **Güvenlik**
   - Input validasyonu yapın
   - Exception handling kullanın
   - Güvenli tip dönüşümleri yapın
   - Null kontrollerini unutmayın

## Örnek Proje Yapısı
```plaintext
CSharpBasics/
├── DataTypes/
│   ├── PrimitiveTypes.cs
│   ├── ReferenceTypes.cs
│   └── ValueTypes.cs
├── Operators/
│   ├── Arithmetic.cs
│   ├── Comparison.cs
│   └── Logical.cs
├── OOP/
│   ├── Classes.cs
│   ├── Inheritance.cs
│   └── Polymorphism.cs
└── Collections/
    ├── Lists.cs
    ├── Dictionary.cs
    └── LINQ.cs
```

## Sık Sorulan Sorular
1. **Value Type ve Reference Type arasındaki fark nedir?**
   - Value Type'lar stack'te, Reference Type'lar heap'te saklanır
   - Value Type'lar değer kopyalama, Reference Type'lar referans kopyalama yapar
   - Value Type'lar null olamaz (nullable olmadıkça)

2. **Boxing ve Unboxing nedir?**
   - Boxing: Value Type'ı Reference Type'a dönüştürme
   - Unboxing: Reference Type'ı Value Type'a dönüştürme
   - Performans maliyeti yüksektir

3. **LINQ ne işe yarar?**
   - Veri sorgulama ve manipülasyonu için kullanılır
   - SQL benzeri sorgu yazımı sağlar
   - Koleksiyonlar üzerinde işlem yapmayı kolaylaştırır

## Kaynaklar
- [C# Programlama Kılavuzu](https://docs.microsoft.com/tr-tr/dotnet/csharp/programming-guide/)
- [C# Dil Referansı](https://docs.microsoft.com/tr-tr/dotnet/csharp/language-reference/)
- [C# Best Practices](https://docs.microsoft.com/tr-tr/dotnet/csharp/fundamentals/coding-style/coding-conventions) 