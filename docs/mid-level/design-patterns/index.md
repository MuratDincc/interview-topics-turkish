# Design Patterns

> 💡 **Ücretsiz Eğitim**: Tüm design pattern'leri detaylı olarak öğrenmek için [Design Patterns Eğitim Serisi](https://www.youtube.com/watch?v=qnywUzPq93w&list=PL8RMb0y6lf9s2YVmfmwRUcHZMB6Uuk3_V)'ni takip edebilirsiniz.

## Genel Bakış
Design Patterns (Tasarım Kalıpları), yazılım geliştirmede sık karşılaşılan sorunlara önerilen, test edilmiş ve kanıtlanmış çözümlerdir. Bu kalıplar, kodun daha okunabilir, bakımı kolay ve yeniden kullanılabilir olmasını sağlar.

## İçindekiler
1. **Creational Patterns (Oluşturucu Kalıplar)**
   - Singleton
   - Factory Method
   - Abstract Factory
   - Builder
   - Prototype

2. **Structural Patterns (Yapısal Kalıplar)**
   - Adapter
   - Bridge
   - Composite
   - Decorator
   - Facade
   - Flyweight
   - Proxy

3. **Behavioral Patterns (Davranışsal Kalıplar)**
   - Chain of Responsibility
   - Command
   - Iterator
   - Mediator
   - Memento
   - Observer
   - State
   - Strategy
   - Template Method
   - Visitor

4. **Repository Pattern**
   - Repository Pattern'in tanımı ve kullanımı
   - Generic Repository implementasyonu
   - Repository Pattern avantajları ve dezavantajları

5. **Unit of Work**
   - Unit of Work Pattern'in tanımı ve kullanımı
   - Transaction yönetimi
   - Repository Pattern ile birlikte kullanımı

## Öğrenme Hedefleri
Bu bölümü tamamladıktan sonra:
- Her bir design pattern'in ne zaman ve nasıl kullanılacağını anlayabileceksiniz
- Gerçek dünya senaryolarında design pattern'leri uygulayabileceksiniz
- Kodunuzu daha modüler ve bakımı kolay hale getirebileceksiniz
- Design pattern'lerin avantaj ve dezavantajlarını değerlendirebileceksiniz

## Ön Koşullar
Bu bölümü takip etmek için:
- Temel C# programlama bilgisi
- Nesne yönelimli programlama (OOP) kavramlarına hakimiyet
- Temel yazılım tasarım prensipleri bilgisi

## Best Practices
1. **Pattern Seçimi**
   - Sorunu doğru analiz edin
   - En uygun pattern'i seçin
   - Pattern'i gereksiz yere kullanmayın
   - Pattern'leri birleştirmekten çekinmeyin

2. **Uygulama**
   - SOLID prensiplerini takip edin
   - Kod okunabilirliğini koruyun
   - Test edilebilirliği göz önünde bulundurun
   - Documentation ekleyin

3. **Bakım**
   - Kod tekrarından kaçının
   - Pattern'leri gerektiğinde güncelleyin
   - Performans etkilerini değerlendirin
   - Team review yapın

## Örnek Proje Yapısı
```plaintext
DesignPatterns/
├── Creational/
│   ├── Singleton/
│   ├── FactoryMethod/
│   ├── AbstractFactory/
│   ├── Builder/
│   └── Prototype/
├── Structural/
│   ├── Adapter/
│   ├── Bridge/
│   ├── Composite/
│   ├── Decorator/
│   ├── Facade/
│   ├── Flyweight/
│   └── Proxy/
└── Behavioral/
    ├── ChainOfResponsibility/
    ├── Command/
    ├── Iterator/
    ├── Mediator/
    ├── Memento/
    ├── Observer/
    ├── State/
    ├── Strategy/
    ├── TemplateMethod/
    └── Visitor/
```

## Sık Sorulan Sorular
1. **Design Pattern'ler ne zaman kullanılmalıdır?**
   - Tekrar eden sorunlarla karşılaşıldığında
   - Kodun bakımı zorlaştığında
   - Yeni özellikler eklenmesi gerektiğinde
   - Test edilebilirlik gerektiğinde

2. **Hangi pattern'i seçmeliyim?**
   - Sorunun doğasına göre
   - Projenin gereksinimlerine göre
   - Takımın tecrübesine göre
   - Performans gereksinimlerine göre

3. **Pattern'ler performansı etkiler mi?**
   - Bazı pattern'ler ekstra katman ekler
   - Memory kullanımını artırabilir
   - Doğru kullanıldığında performansı iyileştirebilir
   - Trade-off'ları değerlendirin

4. **Pattern'ler nasıl test edilir?**
   - Unit testler yazın
   - Mock'lar kullanın
   - Integration testler yapın
   - Code coverage'ı takip edin

## Kaynaklar
- [Design Patterns: Elements of Reusable Object-Oriented Software](https://www.amazon.com/Design-Patterns-Elements-Reusable-Object-Oriented/dp/0201633612)
- [Head First Design Patterns](https://www.amazon.com/Head-First-Design-Patterns-Brain-Friendly/dp/149207800X)
- [Microsoft Design Patterns](https://docs.microsoft.com/tr-tr/archive/msdn-magazine/2009/brownfield/patterns-in-practice)
- [Refactoring Guru](https://refactoring.guru/design-patterns) 