# SOLID Prensipleri

## Giriş

SOLID prensipleri, object-oriented programming'de clean, maintainable ve scalable code yazmak için temel olan beş tasarım prensibidir. Mid-level geliştiriciler için SOLID prensiplerini anlamak, software architecture, code quality ve maintainability konularında uzmanlaşmak için gereklidir. Bu bölüm, Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation ve Dependency Inversion prensiplerini kapsar.

## Kapsanan Konular

### 1. Single Responsibility Principle (SRP)
Bir sınıfın sadece bir sorumluluğa sahip olması, tek bir değişiklik nedeni olması.

**Öğrenilecekler:**
- Responsibility identification
- Class cohesion
- Change impact analysis
- Refactoring strategies
- SRP violations

### 2. Open/Closed Principle (OCP)
Yazılım varlıkları (sınıflar, modüller, fonksiyonlar) genişletmeye açık, değiştirmeye kapalı olmalı.

**Öğrenilecekler:**
- Extension mechanisms
- Abstraction usage
- Polymorphism
- Strategy pattern
- Template method pattern

### 3. Liskov Substitution Principle (LSP)
Alt sınıflar, üst sınıfların yerine geçebilmeli ve program davranışı değişmemeli.

**Öğrenilecekler:**
- Subtype relationships
- Contract compliance
- Behavioral compatibility
- Inheritance design
- LSP violations

### 4. Interface Segregation Principle (ISP)
Client'lar kullanmadıkları interface'lere bağımlı olmamalı, büyük interface'ler küçük parçalara bölünmeli.

**Öğrenilecekler:**
- Interface design
- Client-specific interfaces
- Interface cohesion
- Fat interface problems
- ISP violations

### 5. Dependency Inversion Principle (DIP)
Yüksek seviye modüller düşük seviye modüllere bağımlı olmamalı, her ikisi de abstraction'lara bağımlı olmalı.

**Öğrenilecekler:**
- Dependency direction
- Abstraction usage
- Inversion of control
- Dependency injection
- DIP violations

## Neden Önemli?

### 1. **Code Quality**
- Clean, readable code
- Reduced complexity
- Better maintainability
- Improved testability

### 2. **Software Design**
- Better architecture
- Reduced coupling
- Increased cohesion
- Flexible design

### 3. **Maintainability**
- Easier modifications
- Reduced side effects
- Better extensibility
- Faster development

### 4. **Team Collaboration**
- Shared understanding
- Consistent approach
- Better code reviews
- Knowledge transfer

## Mülakat Soruları

### Temel Sorular

1. **SOLID prensipleri nelerdir?**
   - **Cevap**: SRP, OCP, LSP, ISP, DIP - object-oriented design principles.

2. **Single Responsibility Principle nedir?**
   - **Cevap**: One class, one responsibility, one change reason.

3. **Open/Closed Principle nedir?**
   - **Cevap**: Open for extension, closed for modification.

4. **Liskov Substitution Principle nedir?**
   - **Cevap**: Subtypes replaceable, behavior preserved.

5. **Interface Segregation Principle nedir?**
   - **Cevap**: Client-specific interfaces, no unused dependencies.

### Teknik Sorular

1. **SRP violation nasıl tespit edilir?**
   - **Cevap**: Multiple responsibilities, multiple change reasons, low cohesion.

2. **OCP nasıl implement edilir?**
   - **Cevap**: Abstraction, polymorphism, strategy pattern, extension points.

3. **LSP violation nasıl önlenir?**
   - **Cevap**: Contract compliance, behavioral compatibility, proper inheritance.

4. **ISP nasıl uygulanır?**
   - **Cevap**: Small interfaces, client-specific contracts, interface segregation.

5. **DIP nasıl implement edilir?**
   - **Cevap**: Dependency injection, abstraction usage, inversion of control.

## Best Practices

### 1. **SRP Implementation**
- Identify single responsibility
- Maintain high cohesion
- Avoid multiple change reasons
- Refactor when needed
- Monitor class size

### 2. **OCP Implementation**
- Use abstraction
- Implement extension points
- Avoid modification
- Use design patterns
- Plan for evolution

### 3. **LSP Implementation**
- Maintain contracts
- Ensure compatibility
- Test substitution
- Monitor behavior
- Document expectations

### 4. **ISP Implementation**
- Design small interfaces
- Avoid fat interfaces
- Client-specific contracts
- Monitor interface size
- Refactor when needed

### 5. **DIP Implementation**
- Use dependency injection
- Rely on abstractions
- Invert dependencies
- Use IoC containers
- Monitor coupling

## Kaynaklar

- [SOLID Principles](https://docs.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/architectural-principles#solid-principles)
- [Single Responsibility Principle](https://docs.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/architectural-principles#single-responsibility)
- [Open/Closed Principle](https://docs.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/architectural-principles#openclosed-principle)
- [Liskov Substitution Principle](https://docs.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/architectural-principles#liskov-substitution-principle)
- [Interface Segregation Principle](https://docs.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/architectural-principles#interface-segregation-principle)
- [Dependency Inversion Principle](https://docs.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/architectural-principles#dependency-inversion-principle) 