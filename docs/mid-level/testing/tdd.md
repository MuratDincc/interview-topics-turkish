# Test Driven Development (TDD)

## Giriş

Test Driven Development (TDD), yazılım geliştirme sürecinde önce test yazıp sonra kodu geliştirme yaklaşımıdır. Bu yaklaşım, daha temiz, test edilebilir ve bakımı kolay kod üretmeyi hedefler.

## TDD Süreci

TDD süreci üç ana adımdan oluşur:

1. **Red**: Test yaz ve çalıştır (başarısız olacak)
2. **Green**: Kodu yaz ve testi geç
3. **Refactor**: Kodu iyileştir, testlerin hala geçtiğinden emin ol

### Örnek: Hesap Makinesi

```csharp
// 1. Adım: Test yaz (Red)
public class CalculatorTests
{
    [Fact]
    public void Add_TwoNumbers_ReturnsSum()
    {
        // Arrange
        var calculator = new Calculator();
        
        // Act
        var result = calculator.Add(2, 3);
        
        // Assert
        Assert.Equal(5, result);
    }
}

// 2. Adım: Kodu yaz (Green)
public class Calculator
{
    public int Add(int a, int b)
    {
        return a + b;
    }
}

// 3. Adım: Refactor (gerekirse)
```

## TDD'nin Avantajları

1. **Daha İyi Tasarım**
   - Test edilebilir kod
   - Daha küçük ve odaklı metodlar
   - Daha iyi arayüz tasarımı

2. **Daha Az Hata**
   - Erken hata tespiti
   - Daha az regresyon hatası
   - Daha güvenilir kod

3. **Daha İyi Dokümantasyon**
   - Canlı dokümantasyon
   - Kullanım örnekleri
   - Beklenen davranışın açıklaması

## TDD Best Practices

1. **Test İsimlendirme**
   - Given-When-Then formatı
   - Açıklayıcı isimler
   - Test amacını belirten isimler

2. **Test Organizasyonu**
   - Tek bir şeyi test et
   - Arrange-Act-Assert pattern'i
   - Test bağımsızlığı

3. **Kod Organizasyonu**
   - SOLID prensipleri
   - Dependency Injection
   - Interface'ler

## TDD Örnek Senaryo: E-ticaret Sepeti

```csharp
// 1. Adım: Test yaz (Red)
public class ShoppingCartTests
{
    [Fact]
    public void AddItem_WhenItemIsValid_AddsItemToCart()
    {
        // Arrange
        var cart = new ShoppingCart();
        var item = new CartItem { ProductId = 1, Quantity = 2, Price = 100 };
        
        // Act
        cart.AddItem(item);
        
        // Assert
        Assert.Single(cart.Items);
        Assert.Equal(2, cart.Items[0].Quantity);
    }
}

// 2. Adım: Kodu yaz (Green)
public class ShoppingCart
{
    private List<CartItem> _items = new List<CartItem>();
    
    public IReadOnlyList<CartItem> Items => _items.AsReadOnly();
    
    public void AddItem(CartItem item)
    {
        _items.Add(item);
    }
}

// 3. Adım: Yeni test ekle (Red)
[Fact]
public void CalculateTotal_WhenCartHasItems_ReturnsCorrectTotal()
{
    // Arrange
    var cart = new ShoppingCart();
    cart.AddItem(new CartItem { ProductId = 1, Quantity = 2, Price = 100 });
    cart.AddItem(new CartItem { ProductId = 2, Quantity = 1, Price = 50 });
    
    // Act
    var total = cart.CalculateTotal();
    
    // Assert
    Assert.Equal(250, total);
}

// 4. Adım: Kodu güncelle (Green)
public class ShoppingCart
{
    // ... existing code ...
    
    public decimal CalculateTotal()
    {
        return _items.Sum(x => x.Price * x.Quantity);
    }
}
```

## Mülakat Soruları ve Cevapları

### Temel Sorular

1. **TDD nedir ve nasıl çalışır?**
   - **Cevap**: TDD, önce test yazıp sonra kodu geliştirme yaklaşımıdır. Çalışma süreci:
     - Test yaz (Red)
     - Kodu yaz (Green)
     - Kodu iyileştir (Refactor)
   - Bu döngü sürekli tekrarlanır

2. **TDD'nin avantajları nelerdir?**
   - **Cevap**:
     - Daha temiz ve test edilebilir kod
     - Daha az hata
     - Daha iyi tasarım
     - Canlı dokümantasyon
     - Daha güvenli refactoring

3. **TDD'nin dezavantajları nelerdir?**
   - **Cevap**:
     - Başlangıçta daha yavaş geliştirme
     - Öğrenme eğrisi
     - Test bakım maliyeti
     - Legacy kodda uygulama zorluğu

4. **TDD ile BDD arasındaki farklar nelerdir?**
   - **Cevap**:
     - TDD teknik odaklıdır, BDD iş odaklıdır
     - TDD developer'lar için, BDD tüm paydaşlar için
     - TDD'de test isimleri teknik, BDD'de iş dili kullanılır
     - TDD'de unit testler, BDD'de senaryo testleri

5. **TDD'de test coverage önemli midir?**
   - **Cevap**: Evet, önemlidir çünkü:
     - Test edilmemiş kodları gösterir
     - Test stratejisini yönlendirir
     - Kod kalitesi göstergesidir
     - Risk yönetimine yardımcı olur

### Teknik Sorular

1. **TDD'de hangi test framework'lerini kullanırsınız?**
   - **Cevap**:
     - xUnit (modern ve esnek)
     - NUnit (kapsamlı özellikler)
     - MSTest (Visual Studio entegrasyonu)
     - Moq (mocking için)
     - NSubstitute (mocking için)

2. **TDD'de bağımlılık yönetimi nasıl yapılır?**
   - **Cevap**:
     - Dependency Injection kullanılır
     - Interface'ler tercih edilir
     - Mocking framework'leri kullanılır
     - SOLID prensipleri uygulanır

3. **TDD'de test isimlendirme nasıl yapılmalıdır?**
   - **Cevap**:
     - Given-When-Then formatı kullanılır
     - Test edilen metod belirtilir
     - Beklenen sonuç belirtilir
     - Koşullar belirtilir

4. **TDD'de refactoring nasıl yapılır?**
   - **Cevap**:
     - Testlerin geçtiğinden emin olunur
     - Küçük adımlarla ilerlenir
     - Kod kalitesi artırılır
     - Testler korunur

5. **TDD'de edge case'ler nasıl ele alınır?**
   - **Cevap**:
     - Her edge case için ayrı test yazılır
     - Exception handling test edilir
     - Sınır değerler test edilir
     - Negatif senaryolar test edilir

### Pratik Sorular

1. **Aşağıdaki senaryo için TDD yaklaşımıyla nasıl ilerlersiniz?**
```csharp
// Bir e-posta doğrulama servisi geliştirin
public interface IEmailValidator
{
    bool IsValid(string email);
}
```

- **Cevap**:
```csharp
// 1. Adım: Test yaz (Red)
public class EmailValidatorTests
{
    [Fact]
    public void IsValid_WhenEmailIsValid_ReturnsTrue()
    {
        // Arrange
        var validator = new EmailValidator();
        
        // Act
        var result = validator.IsValid("test@example.com");
        
        // Assert
        Assert.True(result);
    }
    
    [Fact]
    public void IsValid_WhenEmailIsInvalid_ReturnsFalse()
    {
        // Arrange
        var validator = new EmailValidator();
        
        // Act
        var result = validator.IsValid("invalid-email");
        
        // Assert
        Assert.False(result);
    }
}

// 2. Adım: Kodu yaz (Green)
public class EmailValidator : IEmailValidator
{
    public bool IsValid(string email)
    {
        try
        {
            var addr = new System.Net.Mail.MailAddress(email);
            return addr.Address == email;
        }
        catch
        {
            return false;
        }
    }
}
```

2. **Aşağıdaki senaryo için TDD yaklaşımıyla nasıl ilerlersiniz?**
```csharp
// Bir indirim hesaplama servisi geliştirin
public interface IDiscountCalculator
{
    decimal CalculateDiscount(Order order);
}
```

- **Cevap**:
```csharp
// 1. Adım: Test yaz (Red)
public class DiscountCalculatorTests
{
    [Fact]
    public void CalculateDiscount_WhenOrderTotalIsLessThan100_ReturnsNoDiscount()
    {
        // Arrange
        var calculator = new DiscountCalculator();
        var order = new Order { TotalAmount = 50 };
        
        // Act
        var discount = calculator.CalculateDiscount(order);
        
        // Assert
        Assert.Equal(0, discount);
    }
    
    [Fact]
    public void CalculateDiscount_WhenOrderTotalIsGreaterThan100_Returns10PercentDiscount()
    {
        // Arrange
        var calculator = new DiscountCalculator();
        var order = new Order { TotalAmount = 200 };
        
        // Act
        var discount = calculator.CalculateDiscount(order);
        
        // Assert
        Assert.Equal(20, discount);
    }
}

// 2. Adım: Kodu yaz (Green)
public class DiscountCalculator : IDiscountCalculator
{
    public decimal CalculateDiscount(Order order)
    {
        if (order.TotalAmount < 100)
            return 0;
            
        return order.TotalAmount * 0.1m;
    }
}
```

### İleri Seviye Sorular

1. **TDD'de legacy kodla nasıl çalışırsınız?**
   - **Cevap**:
     - Önce test edilebilir parçaları ayırın
     - Strangler Pattern kullanın
     - Adım adım ilerleyin
     - Test edilebilir arayüzler oluşturun
     - Refactoring yapın

2. **TDD'de performans testleri nasıl yapılır?**
   - **Cevap**:
     - Benchmark testleri yazın
     - Profiling yapın
     - Memory kullanımını test edin
     - CPU kullanımını test edin
     - I/O operasyonlarını test edin

3. **TDD'de distributed sistemler nasıl test edilir?**
   - **Cevap**:
     - Mock'lar kullanın
     - Integration testleri yazın
     - Contract testleri yazın
     - Circuit breaker testleri yazın
     - Retry mekanizmalarını test edin

4. **TDD'de microservices nasıl test edilir?**
   - **Cevap**:
     - Her servis için ayrı testler yazın
     - Contract testleri kullanın
     - Integration testleri yazın
     - Service mesh testleri yazın
     - API testleri yazın

5. **TDD'de security testleri nasıl yapılır?**
   - **Cevap**:
     - Input validation testleri yazın
     - Authentication testleri yazın
     - Authorization testleri yazın
     - XSS testleri yazın
     - SQL injection testleri yazın 