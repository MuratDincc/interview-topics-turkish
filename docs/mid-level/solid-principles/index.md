# SOLID Prensipleri

## Genel Bakış
SOLID, nesne yönelimli programlama ve tasarımda kullanılan beş temel prensibin baş harflerinden oluşan bir kısaltmadır. Bu prensipler, yazılımın daha anlaşılır, esnek, bakımı kolay ve test edilebilir olmasını sağlar.

## İçindekiler
1. **Single Responsibility Principle (SRP)**
   - Tanım ve Açıklama
   - Kullanım Alanları
   - Örnek Kodlar
   - Avantajları ve Dezavantajları
   - Mülakat Soruları ve Cevapları

2. **Open/Closed Principle (OCP)**
   - Tanım ve Açıklama
   - Kullanım Alanları
   - Örnek Kodlar
   - Avantajları ve Dezavantajları
   - Mülakat Soruları ve Cevapları

3. **Liskov Substitution Principle (LSP)**
   - Tanım ve Açıklama
   - Kullanım Alanları
   - Örnek Kodlar
   - Avantajları ve Dezavantajları
   - Mülakat Soruları ve Cevapları

4. **Interface Segregation Principle (ISP)**
   - Tanım ve Açıklama
   - Kullanım Alanları
   - Örnek Kodlar
   - Avantajları ve Dezavantajları
   - Mülakat Soruları ve Cevapları

5. **Dependency Inversion Principle (DIP)**
   - Tanım ve Açıklama
   - Kullanım Alanları
   - Örnek Kodlar
   - Avantajları ve Dezavantajları
   - Mülakat Soruları ve Cevapları

## Öğrenme Hedefleri
Bu bölümü tamamladıktan sonra:
- SOLID prensiplerinin her birini detaylı olarak anlayabileceksiniz
- Her prensibin ne zaman ve nasıl uygulanacağını bileceksiniz
- SOLID prensiplerini kullanarak daha temiz ve bakımı kolay kod yazabileceksiniz
- SOLID prensiplerinin avantajlarını ve dezavantajlarını değerlendirebileceksiniz
- Gerçek dünya senaryolarında SOLID prensiplerini uygulayabileceksiniz

## Ön Koşullar
Bu bölümü anlamak için:
- Nesne yönelimli programlama (OOP) temellerini bilmeniz
- C# programlama diline hakim olmanız
- Tasarım desenleri hakkında temel bilgiye sahip olmanız
- Dependency Injection kavramını anlamanız
- Interface ve abstract class kullanımını bilmeniz

## Best Practices
1. **Kod Organizasyonu**
   - Her sınıfın tek bir sorumluluğu olmalı
   - Interface'leri küçük ve öz tutun
   - Soyutlamaları doğru seviyede yapın
   - Bağımlılıkları minimize edin

2. **Test Edilebilirlik**
   - Unit testler yazın
   - Mock nesneler kullanın
   - Test coverage'ı takip edin
   - Test edilebilir kod yazın

3. **Bakım ve Geliştirme**
   - Kod tekrarından kaçının
   - Yorum satırları ekleyin
   - Düzenli refactoring yapın
   - Kod kalitesini sürekli iyileştirin

## Örnek Proje Yapısı
```
SOLID.Examples/
├── SingleResponsibility/
│   ├── Before/
│   └── After/
├── OpenClosed/
│   ├── Before/
│   └── After/
├── LiskovSubstitution/
│   ├── Before/
│   └── After/
├── InterfaceSegregation/
│   ├── Before/
│   └── After/
└── DependencyInversion/
    ├── Before/
    └── After/
```

## Sık Sorulan Sorular
1. **SOLID prensipleri her zaman uygulanmalı mı?**
   - Hayır, her durumda SOLID prensiplerini uygulamak gerekmez. Projenin büyüklüğü, karmaşıklığı ve gereksinimleri dikkate alınmalıdır.

2. **SOLID prensipleri performansı etkiler mi?**
   - Evet, bazı durumlarda performansı etkileyebilir. Ancak genellikle bu etki minimaldir ve kodun bakımı ve genişletilebilirliği açısından faydaları daha önemlidir.

3. **SOLID prensipleri ile diğer tasarım desenleri arasındaki ilişki nedir?**
   - SOLID prensipleri, tasarım desenlerinin temelini oluşturur. Birçok tasarım deseni, SOLID prensiplerini uygulayarak geliştirilmiştir.

## Kaynaklar
- [Microsoft SOLID Principles](https://docs.microsoft.com/tr-tr/dotnet/architecture/modern-web-apps-azure/architectural-principles)
- [SOLID Principles in C#](https://docs.microsoft.com/tr-tr/dotnet/architecture/modern-web-apps-azure/architectural-principles#solid-principles)
- [SOLID Principles: Explanation and Examples](https://docs.microsoft.com/tr-tr/dotnet/architecture/modern-web-apps-azure/architectural-principles#single-responsibility-principle) 