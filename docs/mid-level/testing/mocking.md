# Mocking

## Giriş

Mocking, yazılım testlerinde bağımlılıkları simüle etmek için kullanılan bir tekniktir. Bu teknik sayesinde, test edilen kodun bağımlı olduğu dış servisler, veritabanları veya diğer bileşenler gerçekte çalıştırılmadan test edilebilir.

## Mocking'in Önemi

1. **Test İzolasyonu**
   - Bağımlılıklardan izole edilmiş testler
   - Daha hızlı test çalıştırma
   - Daha güvenilir test sonuçları

2. **Kontrol ve Tahmin Edilebilirlik**
   - Test senaryolarını kontrol etme
   - Beklenen davranışları simüle etme
   - Edge case'leri test etme

3. **Test Edilebilirlik**
   - Karmaşık bağımlılıkları test etme
   - Harici servisleri test etme
   - Zaman bağımlı işlemleri test etme

## Mocking Türleri

1. **Mock**
   - Davranışı kontrol edilen nesneler
   - Beklenen çağrıları doğrulama
   - Gerçek nesnelerin yerine geçme

2. **Stub**
   - Sabit yanıtlar döndüren nesneler
   - Test senaryolarını basitleştirme
   - Bağımlılıkları basitleştirme

3. **Fake**
   - Basitleştirilmiş gerçek nesneler
   - Hafif implementasyonlar
   - Test amaçlı kullanım

## .NET'te Mocking

### Moq Örneği
```csharp
public interface IUserRepository
{
    User GetUser(int id);
    void SaveUser(User user);
}

public class UserService
{
    private readonly IUserRepository _userRepository;
    
    public UserService(IUserRepository userRepository)
    {
        _userRepository = userRepository;
    }
    
    public User GetUser(int id)
    {
        return _userRepository.GetUser(id);
    }
}

// Test
public class UserServiceTests
{
    [Fact]
    public void GetUser_WhenUserExists_ReturnsUser()
    {
        // Arrange
        var mockRepository = new Mock<IUserRepository>();
        var expectedUser = new User { Id = 1, Name = "Test User" };
        
        mockRepository.Setup(x => x.GetUser(1))
                     .Returns(expectedUser);
                     
        var service = new UserService(mockRepository.Object);
        
        // Act
        var result = service.GetUser(1);
        
        // Assert
        Assert.Equal(expectedUser, result);
        mockRepository.Verify(x => x.GetUser(1), Times.Once);
    }
}
```

### NSubstitute Örneği
```csharp
public class OrderServiceTests
{
    [Fact]
    public void CreateOrder_WhenValid_ShouldSaveOrder()
    {
        // Arrange
        var orderRepository = Substitute.For<IOrderRepository>();
        var service = new OrderService(orderRepository);
        
        var order = new Order { Id = 1, Total = 100 };
        
        // Act
        service.CreateOrder(order);
        
        // Assert
        orderRepository.Received(1).Save(order);
    }
}
```

## Mocking Best Practices

1. **Mock Sadece Gerektiğinde**
   - Gereksiz mocking'den kaçının
   - Gerçek nesneleri tercih edin
   - Mock'ları sadece gerekli bağımlılıklar için kullanın

2. **Doğru Mock Türünü Seçin**
   - Mock: Davranış doğrulama için
   - Stub: Sabit yanıtlar için
   - Fake: Basitleştirilmiş implementasyonlar için

3. **Mock'ları Doğru Yapılandırın**
   - Açık ve anlaşılır setup'lar
   - Doğru verify çağrıları
   - Uygun exception handling

## Mülakat Soruları ve Cevapları

### Temel Sorular

1. **Mocking nedir ve neden kullanılır?**
   - **Cevap**: Mocking, test sürecinde bağımlılıkları simüle etmek için kullanılan bir tekniktir. Kullanılma nedenleri:
     - Test izolasyonu sağlar
     - Test sürecini hızlandırır
     - Bağımlılıkları kontrol eder
     - Edge case'leri test etmeyi kolaylaştırır
     - Harici servisleri test etmeyi mümkün kılar

2. **Mock, Stub ve Fake arasındaki farklar nelerdir?**
   - **Cevap**:
     - **Mock**: Davranışı kontrol edilen ve doğrulanan nesneler
     - **Stub**: Sabit yanıtlar döndüren basit nesneler
     - **Fake**: Gerçek nesnelerin basitleştirilmiş versiyonları
     - Mock'lar davranış doğrulama yapar, Stub'lar sabit yanıt verir, Fake'ler gerçek implementasyon sağlar

3. **Hangi durumlarda mocking kullanılmalıdır?**
   - **Cevap**:
     - Harici servislerle iletişimde
     - Veritabanı işlemlerinde
     - Zaman bağımlı işlemlerde
     - Karmaşık bağımlılıklarda
     - Test edilemeyen durumlarda

4. **Mocking'in dezavantajları nelerdir?**
   - **Cevap**:
     - Test bakım maliyeti
     - Gerçek davranışı tam yansıtmayabilir
     - Test kodunun karmaşıklığı
     - Öğrenme eğrisi
     - Aşırı kullanım riski

5. **Moq ve NSubstitute arasındaki farklar nelerdir?**
   - **Cevap**:
     - **Moq**:
       - Daha yaygın kullanım
       - Daha fazla özellik
       - Daha karmaşık syntax
     - **NSubstitute**:
       - Daha basit syntax
       - Daha az özellik
       - Daha modern yaklaşım

### Teknik Sorular

1. **Mocking framework'lerinde setup nasıl yapılır?**
   - **Cevap**:
     - **Moq**:
       ```csharp
       mock.Setup(x => x.Method()).Returns(value);
       mock.Setup(x => x.Method(It.IsAny<Type>())).Returns(value);
       mock.Setup(x => x.Method()).Throws<Exception>();
       ```
     - **NSubstitute**:
       ```csharp
       substitute.Method().Returns(value);
       substitute.Method(Arg.Any<Type>()).Returns(value);
       substitute.Method().Throws<Exception>();
       ```

2. **Mocking'de verify nasıl yapılır?**
   - **Cevap**:
     - **Moq**:
       ```csharp
       mock.Verify(x => x.Method(), Times.Once);
       mock.Verify(x => x.Method(It.IsAny<Type>()), Times.Exactly(2));
       mock.VerifyNoOtherCalls();
       ```
     - **NSubstitute**:
       ```csharp
       substitute.Received(1).Method();
       substitute.Received(2).Method(Arg.Any<Type>());
       substitute.DidNotReceive().Method();
       ```

3. **Mocking'de exception handling nasıl yapılır?**
   - **Cevap**:
     - **Moq**:
       ```csharp
       mock.Setup(x => x.Method()).Throws<Exception>();
       Assert.Throws<Exception>(() => service.Method());
       ```
     - **NSubstitute**:
       ```csharp
       substitute.Method().Throws<Exception>();
       Assert.Throws<Exception>(() => service.Method());
       ```

4. **Mocking'de async metodlar nasıl test edilir?**
   - **Cevap**:
     - **Moq**:
       ```csharp
       mock.Setup(x => x.MethodAsync())
           .ReturnsAsync(value);
       mock.Setup(x => x.MethodAsync())
           .ThrowsAsync<Exception>();
       ```
     - **NSubstitute**:
       ```csharp
       substitute.MethodAsync().Returns(Task.FromResult(value));
       substitute.MethodAsync().Throws<Exception>();
       ```

5. **Mocking'de property'ler nasıl test edilir?**
   - **Cevap**:
     - **Moq**:
       ```csharp
       mock.SetupProperty(x => x.Property, initialValue);
       mock.SetupGet(x => x.Property).Returns(value);
       mock.SetupSet(x => x.Property = value);
       ```
     - **NSubstitute**:
       ```csharp
       substitute.Property.Returns(value);
       substitute.Property = value;
       ```

### Pratik Sorular

1. **Aşağıdaki servisi test etmek için nasıl mocking yaparsınız?**
```csharp
public class PaymentService
{
    private readonly IPaymentGateway _paymentGateway;
    private readonly IEmailService _emailService;
    
    public PaymentService(IPaymentGateway paymentGateway, IEmailService emailService)
    {
        _paymentGateway = paymentGateway;
        _emailService = emailService;
    }
    
    public async Task ProcessPayment(PaymentRequest request)
    {
        var result = await _paymentGateway.ProcessPayment(request);
        
        if (result.Success)
        {
            await _emailService.SendReceipt(result.Receipt);
        }
    }
}
```

- **Cevap**:
```csharp
public class PaymentServiceTests
{
    [Fact]
    public async Task ProcessPayment_WhenSuccessful_ShouldSendReceipt()
    {
        // Arrange
        var mockPaymentGateway = new Mock<IPaymentGateway>();
        var mockEmailService = new Mock<IEmailService>();
        
        var service = new PaymentService(
            mockPaymentGateway.Object,
            mockEmailService.Object);
            
        var request = new PaymentRequest { Amount = 100 };
        var receipt = new Receipt { Id = 1 };
        
        mockPaymentGateway.Setup(x => x.ProcessPayment(request))
                         .ReturnsAsync(new PaymentResult 
                         { 
                             Success = true,
                             Receipt = receipt 
                         });
                         
        // Act
        await service.ProcessPayment(request);
        
        // Assert
        mockEmailService.Verify(x => x.SendReceipt(receipt), Times.Once);
    }
}
```

2. **Aşağıdaki servisi test etmek için nasıl mocking yaparsınız?**
```csharp
public class UserService
{
    private readonly IUserRepository _userRepository;
    private readonly ILogger _logger;
    
    public UserService(IUserRepository userRepository, ILogger logger)
    {
        _userRepository = userRepository;
        _logger = logger;
    }
    
    public User GetUser(int id)
    {
        try
        {
            return _userRepository.GetUser(id);
        }
        catch (Exception ex)
        {
            _logger.Error($"Error getting user {id}", ex);
            throw;
        }
    }
}
```

- **Cevap**:
```csharp
public class UserServiceTests
{
    [Fact]
    public void GetUser_WhenRepositoryThrowsException_ShouldLogError()
    {
        // Arrange
        var mockRepository = new Mock<IUserRepository>();
        var mockLogger = new Mock<ILogger>();
        
        var service = new UserService(
            mockRepository.Object,
            mockLogger.Object);
            
        var userId = 1;
        var exception = new Exception("Test exception");
        
        mockRepository.Setup(x => x.GetUser(userId))
                     .Throws(exception);
                     
        // Act & Assert
        Assert.Throws<Exception>(() => service.GetUser(userId));
        mockLogger.Verify(x => x.Error(
            $"Error getting user {userId}",
            exception), Times.Once);
    }
}
```

### İleri Seviye Sorular

1. **Mocking'de circular dependency nasıl yönetilir?**
   - **Cevap**:
     - Dependency Injection kullanın
     - Interface'leri ayırın
     - Mediator pattern kullanın
     - Event-driven mimari kullanın
     - Service locator kullanın

2. **Mocking'de distributed transaction nasıl test edilir?**
   - **Cevap**:
     - Transaction scope mock'layın
     - Unit of work pattern kullanın
     - Saga pattern kullanın
     - Event sourcing kullanın
     - Compensation transaction'ları test edin

3. **Mocking'de cache nasıl test edilir?**
   - **Cevap**:
     - Cache provider'ı mock'layın
     - Cache hit/miss senaryolarını test edin
     - Cache invalidation'ı test edin
     - Cache expiration'ı test edin
     - Distributed cache'i test edin

4. **Mocking'de message queue nasıl test edilir?**
   - **Cevap**:
     - Queue client'ı mock'layın
     - Message publish/subscribe'ı test edin
     - Retry mekanizmalarını test edin
     - Dead letter queue'yu test edin
     - Message ordering'i test edin

5. **Mocking'de circuit breaker nasıl test edilir?**
   - **Cevap**:
     - Circuit breaker state'ini mock'layın
     - Failure threshold'u test edin
     - Half-open state'i test edin
     - Reset mekanizmasını test edin
     - Fallback mekanizmasını test edin 