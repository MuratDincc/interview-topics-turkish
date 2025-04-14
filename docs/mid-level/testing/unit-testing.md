# Unit Testing

## Giriş

Unit testing, yazılım geliştirme sürecinde en temel test seviyesidir. Birim testleri, yazılımın en küçük parçalarının (fonksiyonlar, metodlar) bağımsız olarak test edilmesini sağlar. Bu testler, kodun doğru çalıştığını doğrulamak ve hataları erken aşamada tespit etmek için kritik öneme sahiptir.

## Unit Testing'in Önemi

1. **Erken Hata Tespiti**
   - Hataların kaynağında tespit edilmesi
   - Düzeltme maliyetinin azaltılması
   - Daha güvenilir kod üretimi

2. **Kod Kalitesi**
   - Daha temiz ve bakımı kolay kod
   - Daha iyi tasarlanmış API'ler
   - Daha iyi dokümante edilmiş kod

3. **Güven ve Refactoring**
   - Güvenli refactoring imkanı
   - Kod değişikliklerinin etkisinin hızlı görülmesi
   - Regresyon hatalarının önlenmesi

## .NET'te Unit Testing

### Test Framework'leri

1. **xUnit**
   ```csharp
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
   ```

2. **NUnit**
   ```csharp
   [TestFixture]
   public class CalculatorTests
   {
       [Test]
       public void Add_TwoNumbers_ReturnsSum()
       {
           // Arrange
           var calculator = new Calculator();
           
           // Act
           var result = calculator.Add(2, 3);
           
           // Assert
           Assert.That(result, Is.EqualTo(5));
       }
   }
   ```

3. **MSTest**
   ```csharp
   [TestClass]
   public class CalculatorTests
   {
       [TestMethod]
       public void Add_TwoNumbers_ReturnsSum()
       {
           // Arrange
           var calculator = new Calculator();
           
           // Act
           var result = calculator.Add(2, 3);
           
           // Assert
           Assert.AreEqual(5, result);
       }
   }
   ```

## Unit Testing Best Practices

### 1. Test İsimlendirme
- Given-When-Then formatı kullanın
- Test amacını açıkça belirtin
- Test edilen metod ve beklenen sonucu belirtin

Örnek:
```csharp
public void CalculateDiscount_WhenCustomerIsPremium_Returns20PercentDiscount()
public void ValidateEmail_WhenEmailIsInvalid_ThrowsArgumentException()
```

### 2. Test Organizasyonu
- Arrange-Act-Assert (AAA) pattern'ini kullanın
- Her test metodunda tek bir şeyi test edin
- Test sınıflarını mantıklı şekilde organize edin

### 3. Test Bağımsızlığı
- Testler birbirinden bağımsız olmalı
- Test sırası önemli olmamalı
- Her test kendi başlangıç durumunu oluşturmalı

### 4. Test Verileri
- Test verilerini açıkça tanımlayın
- Magic number'lardan kaçının
- Anlamlı test verileri kullanın

## Sık Karşılaşılan Sorunlar ve Çözümleri

### 1. Bağımlılık Yönetimi
- Dependency Injection kullanın
- Mocking framework'lerini öğrenin
- Interface'leri tercih edin

### 2. Test Edilebilirlik
- SOLID prensiplerini uygulayın
- Sıkı bağlılıklardan kaçının
- Single Responsibility Principle'a uyun

### 3. Test Bakımı
- Testleri düzenli olarak güncelleyin
- Test kodunu production kodu gibi yönetin
- Test dokümantasyonunu güncel tutun

## Örnek Senaryo: E-ticaret Sepeti

```csharp
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
}
```

## Mülakat Soruları ve Cevapları

### Temel Sorular

1. **Unit testing nedir ve neden önemlidir?**
   - **Cevap**: Unit testing, yazılımın en küçük parçalarının (fonksiyonlar, metodlar) bağımsız olarak test edilmesidir. Önemlidir çünkü:
     - Hataları erken aşamada tespit eder
     - Kod kalitesini artırır
     - Refactoring sürecini güvenli hale getirir
     - Dokümantasyon görevi görür
     - Regresyon hatalarını önler

2. **Unit test ile integration test arasındaki farklar nelerdir?**
   - **Cevap**: 
     - Unit testler tek bir fonksiyonu/metodu test eder, integration testler sistemin farklı bileşenlerinin birlikte çalışmasını test eder
     - Unit testler bağımlılıklardan izole edilmiştir, integration testler gerçek bağımlılıkları kullanır
     - Unit testler daha hızlı çalışır, integration testler daha yavaştır
     - Unit testler daha küçük kapsama sahiptir, integration testler daha geniş kapsama sahiptir

3. **Test Driven Development (TDD) nedir ve avantajları nelerdir?**
   - **Cevap**: TDD, önce test yazıp sonra kodu geliştirme yaklaşımıdır. Avantajları:
     - Daha temiz ve test edilebilir kod üretir
     - Gereksiz kod yazımını önler
     - Sürekli geri bildirim sağlar
     - Tasarım sürecini iyileştirir
     - Dokümantasyon görevi görür

4. **Mocking nedir ve neden kullanılır?**
   - **Cevap**: Mocking, test sürecinde bağımlılıkları simüle etmek için kullanılan bir tekniktir. Kullanılma nedenleri:
     - Harici servisleri taklit etmek
     - Test ortamını kontrol etmek
     - Test sürecini hızlandırmak
     - Bağımlılıkları izole etmek
     - Test edilemeyen durumları simüle etmek

5. **Test coverage nedir ve neden önemlidir?**
   - **Cevap**: Test coverage, kodun ne kadarının test edildiğini ölçen bir metrikdir. Önemlidir çünkü:
     - Test edilmemiş alanları belirler
     - Test stratejisini yönlendirir
     - Kod kalitesi göstergesi sağlar
     - Risk yönetimine yardımcı olur
     - Test odaklı geliştirmeyi teşvik eder

### Teknik Sorular

1. **xUnit, NUnit ve MSTest arasındaki temel farklar nelerdir?**
   - **Cevap**:
     - **xUnit**: 
       - Daha modern ve esnek
       - [Fact] ve [Theory] attribute'ları
       - Constructor ve Dispose kullanımı
       - Paralel test çalıştırma desteği
     - **NUnit**:
       - [TestFixture] ve [Test] attribute'ları
       - [SetUp] ve [TearDown] kullanımı
       - Kapsamlı assertion kütüphanesi
     - **MSTest**:
       - Microsoft'un resmi framework'ü
       - [TestClass] ve [TestMethod] attribute'ları
       - Visual Studio ile tam entegrasyon
       - Daha az esnek ama daha stabil

2. **Arrange-Act-Assert pattern'i nedir ve neden önemlidir?**
   - **Cevap**: AAA pattern'i, test kodunu üç bölüme ayıran bir yaklaşımdır:
     - **Arrange**: Test için gerekli ortamı hazırlama
     - **Act**: Test edilecek metodu çağırma
     - **Assert**: Sonuçları doğrulama
   - Önemlidir çünkü:
     - Test kodunu daha okunabilir yapar
     - Test yapısını standardize eder
     - Test amacını netleştirir
     - Bakımı kolaylaştırır

3. **Dependency Injection'ın unit testing'deki önemi nedir?**
   - **Cevap**: Dependency Injection (DI) unit testing'de şu nedenlerle önemlidir:
     - Bağımlılıkları kolayca mock'lamayı sağlar
     - Test edilebilirliği artırır
     - Kodun daha modüler olmasını sağlar
     - Test senaryolarını kolaylaştırır
     - Bağımlılık yönetimini merkezileştirir

4. **SOLID prensipleri unit testing'i nasıl etkiler?**
   - **Cevap**: SOLID prensipleri unit testing'i şu şekilde etkiler:
     - **Single Responsibility**: Her sınıfın tek bir sorumluluğu olması, test yazmayı kolaylaştırır
     - **Open/Closed**: Yeni özellikler eklerken mevcut testlerin bozulmamasını sağlar
     - **Liskov Substitution**: Alt sınıfların test edilebilirliğini artırır
     - **Interface Segregation**: Daha küçük ve odaklı interface'ler, daha kolay test yazmayı sağlar
     - **Dependency Inversion**: Bağımlılıkların yönetimini kolaylaştırır

5. **Test bağımsızlığı neden önemlidir?**
   - **Cevap**: Test bağımsızlığı önemlidir çünkü:
     - Testlerin birbirini etkilememesini sağlar
     - Test sırasının önemsiz olmasını sağlar
     - Hata ayıklamayı kolaylaştırır
     - Test sonuçlarının güvenilirliğini artırır
     - Paralel test çalıştırmayı mümkün kılar

### Pratik Sorular

1. **Aşağıdaki metodu test etmek için nasıl bir test yazarsınız?**
```csharp
public class Calculator
{
    public int Divide(int a, int b)
    {
        if (b == 0)
            throw new DivideByZeroException();
        return a / b;
    }
}
```

- **Cevap**:
```csharp
public class CalculatorTests
{
    [Fact]
    public void Divide_WhenDivisorIsNotZero_ReturnsCorrectResult()
    {
        // Arrange
        var calculator = new Calculator();
        
        // Act
        var result = calculator.Divide(10, 2);
        
        // Assert
        Assert.Equal(5, result);
    }
    
    [Fact]
    public void Divide_WhenDivisorIsZero_ThrowsDivideByZeroException()
    {
        // Arrange
        var calculator = new Calculator();
        
        // Act & Assert
        Assert.Throws<DivideByZeroException>(() => calculator.Divide(10, 0));
    }
}
```

2. **Aşağıdaki servisi test etmek için nasıl bir yaklaşım izlersiniz?**
```csharp
public class OrderService
{
    private readonly IOrderRepository _orderRepository;
    private readonly IEmailService _emailService;
    
    public OrderService(IOrderRepository orderRepository, IEmailService emailService)
    {
        _orderRepository = orderRepository;
        _emailService = emailService;
    }
    
    public async Task<Order> CreateOrder(OrderRequest request)
    {
        var order = new Order
        {
            CustomerId = request.CustomerId,
            Items = request.Items,
            TotalAmount = request.Items.Sum(x => x.Price * x.Quantity)
        };
        
        await _orderRepository.SaveAsync(order);
        await _emailService.SendOrderConfirmation(order);
        
        return order;
    }
}
```

- **Cevap**:
```csharp
public class OrderServiceTests
{
    [Fact]
    public async Task CreateOrder_WhenRequestIsValid_CreatesAndSavesOrder()
    {
        // Arrange
        var mockOrderRepository = new Mock<IOrderRepository>();
        var mockEmailService = new Mock<IEmailService>();
        var service = new OrderService(mockOrderRepository.Object, mockEmailService.Object);
        
        var request = new OrderRequest
        {
            CustomerId = 1,
            Items = new List<OrderItem>
            {
                new OrderItem { Price = 100, Quantity = 2 }
            }
        };
        
        // Act
        var result = await service.CreateOrder(request);
        
        // Assert
        Assert.NotNull(result);
        Assert.Equal(1, result.CustomerId);
        Assert.Equal(200, result.TotalAmount);
        mockOrderRepository.Verify(x => x.SaveAsync(It.IsAny<Order>()), Times.Once);
        mockEmailService.Verify(x => x.SendOrderConfirmation(It.IsAny<Order>()), Times.Once);
    }
}
```

3. **Aşağıdaki test sınıfındaki sorunları belirleyin ve düzeltin:**
```csharp
public class UserServiceTests
{
    private UserService _userService;
    private static int _testCounter = 0;
    
    [SetUp]
    public void Setup()
    {
        _userService = new UserService();
        _testCounter++;
    }
    
    [Test]
    public void Test1()
    {
        var result = _userService.CreateUser("test@email.com");
        Assert.NotNull(result);
    }
    
    [Test]
    public void Test2()
    {
        var result = _userService.CreateUser("test@email.com");
        Assert.NotNull(result);
    }
}
```

- **Cevap**: Sorunlar ve düzeltilmiş hali:
```csharp
public class UserServiceTests
{
    private UserService _userService;
    
    [SetUp]
    public void Setup()
    {
        _userService = new UserService();
    }
    
    [Test]
    public void CreateUser_WhenEmailIsValid_ReturnsUser()
    {
        // Arrange
        var email = "test@email.com";
        
        // Act
        var result = _userService.CreateUser(email);
        
        // Assert
        Assert.NotNull(result);
        Assert.Equal(email, result.Email);
    }
    
    [Test]
    public void CreateUser_WhenEmailIsInvalid_ThrowsArgumentException()
    {
        // Arrange
        var invalidEmail = "invalid-email";
        
        // Act & Assert
        Assert.Throws<ArgumentException>(() => _userService.CreateUser(invalidEmail));
    }
}
```

Sorunlar:
1. Statik `_testCounter` değişkeni test bağımsızlığını bozar
2. Test isimleri anlamsız
3. Test metodları aynı şeyi test ediyor
4. Given-When-Then formatı kullanılmamış
5. Eksik test senaryoları

### İleri Seviye Sorular

1. **Unit testlerde bağımlılık yönetimi için hangi pattern'leri kullanırsınız?**
   - **Cevap**:
     - **Dependency Injection**: Constructor injection, property injection, method injection
     - **Factory Pattern**: Test için özel factory'ler
     - **Strategy Pattern**: Farklı davranışları test etmek için
     - **Adapter Pattern**: Legacy sistemleri test etmek için
     - **Proxy Pattern**: Test sırasında davranışı değiştirmek için

2. **Test edilebilir kod yazmak için hangi prensipleri takip edersiniz?**
   - **Cevap**:
     - SOLID prensiplerini uygulama
     - Dependency Injection kullanma
     - Interface'leri tercih etme
     - Sıkı bağlılıklardan kaçınma
     - Single Responsibility Principle'a uyma
     - Pure function'lar yazma
     - Side effect'leri minimize etme

3. **Legacy kodda unit test yazarken karşılaştığınız zorluklar nelerdir?**
   - **Cevap**:
     - Sıkı bağlılıklar
     - Global state kullanımı
     - Statik metodlar
     - Test edilemeyen tasarım
     - Eksik interface'ler
     - Karmaşık bağımlılıklar
     - Dokümantasyon eksikliği

4. **Unit testlerde performans optimizasyonu için neler yapabilirsiniz?**
   - **Cevap**:
     - Paralel test çalıştırma
     - Test verilerini önbelleğe alma
     - Gereksiz setup/teardown işlemlerinden kaçınma
     - Test süitlerini optimize etme
     - Mock'ları etkin kullanma
     - Test veritabanını optimize etme

5. **Unit testlerde exception handling nasıl yapılmalıdır?**
   - **Cevap**:
     - Beklenen exception'ları test etme
     - Exception mesajlarını doğrulama
     - Exception tiplerini kontrol etme
     - Exception stack trace'ini inceleme
     - Custom exception'ları test etme
     - Exception handling stratejilerini test etme 