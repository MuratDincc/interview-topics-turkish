# Testing Best Practices

## Giriş

Testing Best Practices (Test En İyi Uygulamaları), yazılım testlerinin etkinliğini, bakımını ve kalitesini artırmak için kullanılan yöntem ve prensiplerdir. Bu uygulamalar, test sürecinin verimliliğini ve güvenilirliğini sağlar.

## Test Yazımı Best Practices

1. **Test İsimlendirme**
   - Açıklayıcı ve anlaşılır isimler
   - Test_Koşul_BeklenenSonuç formatı
   - İngilizce kullanımı
   - Anlamlı kısaltmalar

2. **Test Organizasyonu**
   - Mantıksal gruplandırma
   - Test kategorileri
   - Test sıralaması
   - Bağımlılık yönetimi

3. **Test Verileri**
   - Gerçekçi test verileri
   - Test verisi yönetimi
   - Veri temizleme
   - Veri izolasyonu

## .NET'te Test Best Practices

### Test Sınıfı Örneği
```csharp
public class UserServiceTests
{
    private readonly Mock<IUserRepository> _mockRepository;
    private readonly UserService _service;

    public UserServiceTests()
    {
        _mockRepository = new Mock<IUserRepository>();
        _service = new UserService(_mockRepository.Object);
    }

    [Fact]
    public async Task GetUser_WhenUserExists_ShouldReturnUser()
    {
        // Arrange
        var userId = 1;
        var expectedUser = new User { Id = userId, Name = "Test User" };
        _mockRepository.Setup(x => x.GetById(userId))
                      .ReturnsAsync(expectedUser);

        // Act
        var result = await _service.GetUser(userId);

        // Assert
        Assert.Equal(expectedUser, result);
        _mockRepository.Verify(x => x.GetById(userId), Times.Once);
    }
}
```

### Test Kategorileri
```csharp
[Trait("Category", "Integration")]
public class DatabaseTests
{
    [Fact]
    public async Task SaveUser_ShouldWork()
    {
        // Test logic
    }
}

[Trait("Category", "Performance")]
public class PerformanceTests
{
    [Fact]
    public async Task ProcessLargeData_ShouldCompleteInTime()
    {
        // Test logic
    }
}
```

### Test Verisi Yönetimi
```csharp
public static class TestData
{
    public static User CreateValidUser()
    {
        return new User
        {
            Id = 1,
            Name = "Test User",
            Email = "test@example.com"
        };
    }

    public static List<User> CreateUserList(int count)
    {
        return Enumerable.Range(1, count)
            .Select(i => new User { Id = i, Name = $"User {i}" })
            .ToList();
    }
}
```

## Test Bakımı Best Practices

1. **Kod Organizasyonu**
   - DRY (Don't Repeat Yourself) prensibi
   - Test yardımcı metodları
   - Test base sınıfları
   - Test utilities

2. **Test Temizliği**
   - Test verilerini temizleme
   - Kaynakları serbest bırakma
   - Test izolasyonu
   - Test bağımsızlığı

3. **Test Dokümantasyonu**
   - Test amaçlarını belgeleme
   - Test senaryolarını açıklama
   - Test verilerini belgeleme
   - Test sonuçlarını raporlama

## Mülakat Soruları ve Cevapları

### Temel Sorular

1. **Test yazarken dikkat edilmesi gereken en önemli prensipler nelerdir?**
   - **Cevap**:
     - Test bağımsızlığı
     - Açıklayıcı test isimleri
     - Tek bir şeyi test etme
     - Arrange-Act-Assert pattern
     - Test edilebilir kod yazma

2. **Test isimlendirme kuralları nelerdir?**
   - **Cevap**:
     - Test_Koşul_BeklenenSonuç formatı
     - Açıklayıcı ve anlaşılır isimler
     - İngilizce kullanımı
     - Anlamlı kısaltmalar
     - Tutarlı isimlendirme

3. **Test verileri nasıl yönetilmelidir?**
   - **Cevap**:
     - Gerçekçi test verileri
     - Test verisi fabrikaları
     - Veri temizleme stratejileri
     - Veri izolasyonu
     - Test verisi yönetimi

4. **Test bakımı nasıl yapılmalıdır?**
   - **Cevap**:
     - DRY prensibi
     - Test yardımcı metodları
     - Test base sınıfları
     - Test utilities
     - Düzenli refactoring

5. **Test dokümantasyonu neden önemlidir?**
   - **Cevap**:
     - Test amaçlarını açıklar
     - Test senaryolarını belgeler
     - Test verilerini açıklar
     - Test sonuçlarını raporlar
     - Test sürecini iyileştirir

### Teknik Sorular

1. **Test base sınıfı nasıl oluşturulur?**
   - **Cevap**:
```csharp
public abstract class TestBase : IDisposable
{
    protected readonly Mock<IUserRepository> MockRepository;
    protected readonly UserService Service;
    protected readonly TestData TestData;

    protected TestBase()
    {
        MockRepository = new Mock<IUserRepository>();
        Service = new UserService(MockRepository.Object);
        TestData = new TestData();
    }

    public void Dispose()
    {
        // Cleanup logic
    }
}

public class UserServiceTests : TestBase
{
    [Fact]
    public async Task GetUser_ShouldWork()
    {
        // Test logic using base class members
    }
}
```

2. **Test verisi fabrikası nasıl oluşturulur?**
   - **Cevap**:
```csharp
public static class TestDataFactory
{
    public static User CreateUser(int? id = null, string name = null)
    {
        return new User
        {
            Id = id ?? 1,
            Name = name ?? "Test User",
            Email = "test@example.com"
        };
    }

    public static List<User> CreateUserList(int count)
    {
        return Enumerable.Range(1, count)
            .Select(i => CreateUser(i, $"User {i}"))
            .ToList();
    }
}
```

3. **Test kategorileri nasıl kullanılır?**
   - **Cevap**:
```csharp
[Trait("Category", "Integration")]
public class IntegrationTests
{
    [Fact]
    public async Task DatabaseTest()
    {
        // Test logic
    }
}

[Trait("Category", "Performance")]
public class PerformanceTests
{
    [Fact]
    public async Task PerformanceTest()
    {
        // Test logic
    }
}
```

4. **Test yardımcı metodları nasıl oluşturulur?**
   - **Cevap**:
```csharp
public static class TestHelper
{
    public static void SetupMock<T>(Mock<T> mock, Expression<Action<T>> expression)
        where T : class
    {
        mock.Setup(expression);
    }

    public static void VerifyMock<T>(Mock<T> mock, Expression<Action<T>> expression, Times times)
        where T : class
    {
        mock.Verify(expression, times);
    }

    public static async Task<T> ExecuteWithTimeout<T>(Func<Task<T>> action, int timeoutSeconds = 5)
    {
        var task = action();
        if (await Task.WhenAny(task, Task.Delay(timeoutSeconds * 1000)) == task)
        {
            return await task;
        }
        throw new TimeoutException();
    }
}
```

5. **Test izolasyonu nasıl sağlanır?**
   - **Cevap**:
```csharp
public class IsolatedTests : IDisposable
{
    private readonly string _testDatabase;
    private readonly DbContext _context;

    public IsolatedTests()
    {
        _testDatabase = $"TestDb_{Guid.NewGuid()}";
        _context = new DbContext($"Server=.;Database={_testDatabase};Trusted_Connection=True;");
    }

    [Fact]
    public async Task TestWithIsolation()
    {
        // Test logic with isolated database
    }

    public void Dispose()
    {
        _context.Database.EnsureDeleted();
        _context.Dispose();
    }
}
```

### Pratik Sorular

1. **Aşağıdaki test sınıfını best practices'e göre düzenleyin:**
```csharp
public class ProductServiceTests
{
    [Fact]
    public void Test1()
    {
        var mock = new Mock<IRepository>();
        var service = new ProductService(mock.Object);
        var result = service.GetProduct(1);
        Assert.NotNull(result);
    }

    [Fact]
    public void Test2()
    {
        var mock = new Mock<IRepository>();
        var service = new ProductService(mock.Object);
        var result = service.GetProduct(2);
        Assert.Null(result);
    }
}
```

- **Cevap**:
```csharp
public class ProductServiceTests : IDisposable
{
    private readonly Mock<IRepository> _mockRepository;
    private readonly ProductService _service;

    public ProductServiceTests()
    {
        _mockRepository = new Mock<IRepository>();
        _service = new ProductService(_mockRepository.Object);
    }

    [Fact]
    public async Task GetProduct_WhenProductExists_ShouldReturnProduct()
    {
        // Arrange
        var productId = 1;
        var expectedProduct = new Product { Id = productId, Name = "Test Product" };
        _mockRepository.Setup(x => x.GetById(productId))
                      .ReturnsAsync(expectedProduct);

        // Act
        var result = await _service.GetProduct(productId);

        // Assert
        Assert.Equal(expectedProduct, result);
        _mockRepository.Verify(x => x.GetById(productId), Times.Once);
    }

    [Fact]
    public async Task GetProduct_WhenProductNotExists_ShouldReturnNull()
    {
        // Arrange
        var productId = 2;
        _mockRepository.Setup(x => x.GetById(productId))
                      .ReturnsAsync((Product)null);

        // Act
        var result = await _service.GetProduct(productId);

        // Assert
        Assert.Null(result);
        _mockRepository.Verify(x => x.GetById(productId), Times.Once);
    }

    public void Dispose()
    {
        // Cleanup if needed
    }
}
```

2. **Aşağıdaki test verisi yönetimini best practices'e göre düzenleyin:**
```csharp
public class OrderServiceTests
{
    [Fact]
    public void TestOrder()
    {
        var order = new Order
        {
            Id = 1,
            CustomerId = 1,
            Items = new List<OrderItem>
            {
                new OrderItem { ProductId = 1, Quantity = 2 },
                new OrderItem { ProductId = 2, Quantity = 1 }
            }
        };
        // Test logic
    }
}
```

- **Cevap**:
```csharp
public static class OrderTestData
{
    public static Order CreateValidOrder(int? id = null, int? customerId = null)
    {
        return new Order
        {
            Id = id ?? 1,
            CustomerId = customerId ?? 1,
            Items = new List<OrderItem>
            {
                CreateOrderItem(1, 2),
                CreateOrderItem(2, 1)
            }
        };
    }

    public static OrderItem CreateOrderItem(int productId, int quantity)
    {
        return new OrderItem
        {
            ProductId = productId,
            Quantity = quantity
        };
    }
}

public class OrderServiceTests
{
    [Fact]
    public async Task ProcessOrder_WhenValid_ShouldSucceed()
    {
        // Arrange
        var order = OrderTestData.CreateValidOrder();
        // Test logic
    }
}
```

### İleri Seviye Sorular

1. **Test edilebilir kod nasıl yazılır?**
   - **Cevap**:
     - Dependency Injection kullanın
     - Interface'leri tercih edin
     - Single Responsibility prensibini uygulayın
     - Bağımlılıkları minimize edin
     - Test edilebilir tasarım desenleri kullanın

2. **Test performansı nasıl optimize edilir?**
   - **Cevap**:
     - Paralel test çalıştırın
     - Test verilerini optimize edin
     - Test ortamını yapılandırın
     - Gereksiz testleri kaldırın
     - Test süresini ölçün

3. **Test bakım maliyeti nasıl azaltılır?**
   - **Cevap**:
     - Test kodunu modülerleştirin
     - Test yardımcı metodları kullanın
     - Test verilerini merkezileştirin
     - Test dokümantasyonunu güncel tutun
     - Düzenli refactoring yapın

4. **Test kalitesi nasıl ölçülür?**
   - **Cevap**:
     - Test coverage'ı ölçün
     - Test başarı oranını takip edin
     - Test süresini ölçün
     - Test bakım maliyetini hesaplayın
     - Test dokümantasyonunu değerlendirin

5. **Test stratejisi nasıl geliştirilir?**
   - **Cevap**:
     - Test hedeflerini belirleyin
     - Test tiplerini seçin
     - Test araçlarını belirleyin
     - Test sürecini tanımlayın
     - Test metriklerini belirleyin 