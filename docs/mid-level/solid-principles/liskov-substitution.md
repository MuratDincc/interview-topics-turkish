# Liskov Substitution Principle (LSP)

## Genel Bakış
Liskov Substitution Principle (LSP), alt sınıfların, üst sınıfların yerine geçebilmesi gerektiğini belirtir. Yani, bir programda üst sınıf tipinde bir nesne kullanılıyorsa, bu nesnenin yerine alt sınıf tipinde bir nesne kullanıldığında programın davranışı bozulmamalıdır.

## Mülakat Soruları ve Cevapları

### 1. Liskov Substitution Principle nedir ve neden önemlidir?
**Cevap:**
Liskov Substitution Principle, alt sınıfların üst sınıfların yerine geçebilmesi gerektiğini belirtir. Önemlidir çünkü:
- Kodun tutarlılığını sağlar
- Beklenmeyen davranışları önler
- Kalıtım hiyerarşisinin doğru kullanılmasını sağlar
- Test edilebilirliği artırır

**Örnek Kod:**
```csharp
// LSP'ye uymayan kod
public class Rectangle
{
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }

    public int CalculateArea()
    {
        return Width * Height;
    }
}

public class Square : Rectangle
{
    public override int Width
    {
        set { base.Width = value; base.Height = value; }
    }

    public override int Height
    {
        set { base.Height = value; base.Width = value; }
    }
}

// LSP'ye uyan kod
public abstract class Shape
{
    public abstract int CalculateArea();
}

public class Rectangle : Shape
{
    public int Width { get; set; }
    public int Height { get; set; }

    public override int CalculateArea()
    {
        return Width * Height;
    }
}

public class Square : Shape
{
    public int Side { get; set; }

    public override int CalculateArea()
    {
        return Side * Side;
    }
}
```

### 2. LSP'yi ihlal eden durumları nasıl tespit edebiliriz?
**Cevap:**
LSP ihlallerini tespit etmek için:
- Alt sınıfın üst sınıfın davranışını değiştirmesi
- Alt sınıfın üst sınıfın ön koşullarını güçlendirmesi
- Alt sınıfın üst sınıfın son koşullarını zayıflatması
- Alt sınıfın üst sınıfın değişmezlerini ihlal etmesi

**Örnek Kod:**
```csharp
// LSP ihlali
public class Bird
{
    public virtual void Fly()
    {
        // Uçma işlemi
    }
}

public class Penguin : Bird
{
    public override void Fly()
    {
        throw new NotSupportedException("Penguenler uçamaz!");
    }
}

// LSP'ye uygun
public abstract class Bird
{
    public abstract void Move();
}

public class FlyingBird : Bird
{
    public override void Move()
    {
        // Uçma işlemi
    }
}

public class Penguin : Bird
{
    public override void Move()
    {
        // Yürüme işlemi
    }
}
```

### 3. LSP'yi uygularken dikkat edilmesi gereken noktalar nelerdir?
**Cevap:**
LSP uygularken dikkat edilmesi gerekenler:
- Kalıtım hiyerarşisini doğru tasarlayın
- Alt sınıfların davranışlarını kontrol edin
- Ön koşulları ve son koşulları dikkate alın
- Değişmezleri koruyun

**Örnek Kod:**
```csharp
public abstract class Account
{
    public decimal Balance { get; protected set; }

    public virtual void Deposit(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentException("Miktar pozitif olmalıdır");
        
        Balance += amount;
    }

    public virtual void Withdraw(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentException("Miktar pozitif olmalıdır");
        
        if (amount > Balance)
            throw new InvalidOperationException("Yetersiz bakiye");
        
        Balance -= amount;
    }
}

public class SavingsAccount : Account
{
    public decimal InterestRate { get; set; }

    public override void Withdraw(decimal amount)
    {
        if (amount > Balance * 0.9m)
            throw new InvalidOperationException("Maksimum çekim limiti aşıldı");
        
        base.Withdraw(amount);
    }
}
```

### 4. LSP ile ilgili yaygın hatalar nelerdir?
**Cevap:**
Yaygın hatalar:
- Yanlış kalıtım hiyerarşisi
- Davranış değişiklikleri
- Ön koşul ihlalleri
- Son koşul ihlalleri

**Örnek Kod:**
```csharp
// Yaygın hata: Yanlış kalıtım hiyerarşisi
public class Employee
{
    public decimal CalculateSalary()
    {
        // Maaş hesaplama
        return 0;
    }
}

public class Manager : Employee
{
    public decimal CalculateBonus()
    {
        // Bonus hesaplama
        return 0;
    }
}

// Daha uygun
public interface IEmployee
{
    decimal CalculateSalary();
}

public interface IManager : IEmployee
{
    decimal CalculateBonus();
}
```

### 5. LSP'yi gerçek dünya senaryolarında nasıl uygularız?
**Cevap:**
Gerçek dünya senaryolarında:
- Domain-driven design yaklaşımını kullanın
- Interface segregation uygulayın
- Composition over inheritance kullanın
- Design by contract yaklaşımını uygulayın

**Örnek Kod:**
```csharp
public interface IOrderProcessor
{
    void ProcessOrder(Order order);
}

public interface IOrderValidator
{
    bool Validate(Order order);
}

public class OrderProcessor : IOrderProcessor
{
    private readonly IOrderValidator _validator;

    public OrderProcessor(IOrderValidator validator)
    {
        _validator = validator;
    }

    public void ProcessOrder(Order order)
    {
        if (!_validator.Validate(order))
            throw new InvalidOperationException("Sipariş geçersiz");

        // Sipariş işleme mantığı
    }
}

public class ExpressOrderProcessor : IOrderProcessor
{
    private readonly IOrderValidator _validator;

    public ExpressOrderProcessor(IOrderValidator validator)
    {
        _validator = validator;
    }

    public void ProcessOrder(Order order)
    {
        if (!_validator.Validate(order))
            throw new InvalidOperationException("Sipariş geçersiz");

        // Express sipariş işleme mantığı
    }
}
```

## Best Practices
1. **Kalıtım Hiyerarşisi**
   - Doğru kalıtım hiyerarşisi tasarlayın
   - Interface'leri tercih edin
   - Composition over inheritance kullanın
   - Design by contract uygulayın

2. **Davranış Kontrolü**
   - Alt sınıfların davranışlarını kontrol edin
   - Ön koşulları ve son koşulları dikkate alın
   - Değişmezleri koruyun
   - Exception handling uygulayın

3. **Test Edilebilirlik**
   - Unit testler yazın
   - Mock nesneler kullanın
   - Test coverage'ı takip edin
   - Test edilebilir kod yazın

## Kaynaklar
- [Microsoft Liskov Substitution Principle](https://docs.microsoft.com/tr-tr/dotnet/architecture/modern-web-apps-azure/architectural-principles#liskov-substitution-principle)
- [SOLID Principles in C#](https://docs.microsoft.com/tr-tr/dotnet/architecture/modern-web-apps-azure/architectural-principles#solid-principles)
- [Liskov Substitution Principle: Explanation and Examples](https://docs.microsoft.com/tr-tr/dotnet/architecture/modern-web-apps-azure/architectural-principles#liskov-substitution-principle) 