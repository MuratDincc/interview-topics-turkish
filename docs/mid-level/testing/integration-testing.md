# Integration Testing

## Giriş

Integration Testing (Entegrasyon Testi), yazılım bileşenlerinin birbiriyle etkileşimini test etmek için kullanılan bir test türüdür. Bu testler, farklı modüllerin, servislerin veya sistemlerin birlikte doğru çalışıp çalışmadığını kontrol eder.

## Integration Testing'in Önemi

1. **Sistem Bütünlüğü**
   - Bileşenler arası etkileşimleri doğrulama
   - Sistem genelinde veri akışını test etme
   - Bileşen entegrasyonlarını kontrol etme

2. **Hata Tespiti**
   - Bileşenler arası iletişim hatalarını bulma
   - Veri dönüşüm hatalarını tespit etme
   - API uyumsuzluklarını belirleme

3. **Sistem Davranışı**
   - Gerçek dünya senaryolarını test etme
   - Sistem performansını ölçme
   - Ölçeklenebilirliği test etme

## Integration Testing Türleri

1. **Big Bang Testing**
   - Tüm bileşenlerin aynı anda entegre edilmesi
   - Hızlı test süreci
   - Hata tespiti zorluğu

2. **Top-Down Testing**
   - Üst seviyeden başlayarak test etme
   - Stub'lar kullanma
   - Kritik modülleri öncelikli test etme

3. **Bottom-Up Testing**
   - Alt seviyeden başlayarak test etme
   - Driver'lar kullanma
   - Temel fonksiyonları önce test etme

4. **Sandwich Testing**
   - Top-Down ve Bottom-Up yaklaşımların kombinasyonu
   - Paralel test süreci
   - Karmaşık sistemler için uygun

## .NET'te Integration Testing

### TestServer Kullanımı
```csharp
public class IntegrationTests
{
    private readonly TestServer _server;
    private readonly HttpClient _client;

    public IntegrationTests()
    {
        _server = new TestServer(new WebHostBuilder()
            .UseStartup<Startup>());
        _client = _server.CreateClient();
    }

    [Fact]
    public async Task GetUsers_ReturnsSuccessStatusCode()
    {
        // Act
        var response = await _client.GetAsync("/api/users");

        // Assert
        response.EnsureSuccessStatusCode();
        var content = await response.Content.ReadAsStringAsync();
        var users = JsonConvert.DeserializeObject<List<User>>(content);
        Assert.NotEmpty(users);
    }
}
```

### Entity Framework Core ile Test
```csharp
public class UserRepositoryTests
{
    private readonly DbContextOptions<ApplicationDbContext> _options;

    public UserRepositoryTests()
    {
        _options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseInMemoryDatabase(databaseName: "TestDb")
            .Options;
    }

    [Fact]
    public async Task CreateUser_ShouldAddUserToDatabase()
    {
        // Arrange
        using (var context = new ApplicationDbContext(_options))
        {
            var repository = new UserRepository(context);
            var user = new User { Name = "Test User" };

            // Act
            await repository.CreateUser(user);

            // Assert
            var savedUser = await context.Users.FirstOrDefaultAsync();
            Assert.NotNull(savedUser);
            Assert.Equal("Test User", savedUser.Name);
        }
    }
}
```

## Integration Testing Best Practices

1. **Test Ortamı**
   - Gerçeğe yakın test ortamı
   - İzole edilmiş test veritabanı
   - Harici servislerin mock'lanması

2. **Test Verileri**
   - Gerçekçi test verileri
   - Veri temizleme stratejileri
   - Test verilerinin yönetimi

3. **Test Organizasyonu**
   - Mantıksal test grupları
   - Bağımlılık yönetimi
   - Test sırası kontrolü

## Mülakat Soruları ve Cevapları

### Temel Sorular

1. **Integration Testing nedir ve neden önemlidir?**
   - **Cevap**: Integration Testing, yazılım bileşenlerinin birbiriyle etkileşimini test etmek için kullanılan bir test türüdür. Önemi:
     - Bileşenler arası etkileşimleri doğrular
     - Sistem bütünlüğünü kontrol eder
     - Entegrasyon hatalarını erken tespit eder
     - Sistem davranışını doğrular
     - Performans sorunlarını belirler

2. **Unit Testing ve Integration Testing arasındaki farklar nelerdir?**
   - **Cevap**:
     - **Unit Testing**:
       - Tek bir bileşeni test eder
       - Bağımlılıklar mock'lanır
       - Hızlı çalışır
       - İzole edilmiş testler
     - **Integration Testing**:
       - Birden fazla bileşeni test eder
       - Gerçek bağımlılıklar kullanılır
       - Daha yavaş çalışır
       - Sistem genelinde testler

3. **Integration Testing türleri nelerdir?**
   - **Cevap**:
     - Big Bang Testing
     - Top-Down Testing
     - Bottom-Up Testing
     - Sandwich Testing
     - Continuous Integration Testing

4. **Integration Testing'in avantajları ve dezavantajları nelerdir?**
   - **Cevap**:
     - **Avantajlar**:
       - Sistem bütünlüğünü doğrular
       - Gerçek senaryoları test eder
       - Performans sorunlarını tespit eder
     - **Dezavantajlar**:
       - Test süresi uzundur
       - Bakım maliyeti yüksektir
       - Hata tespiti zordur

5. **Integration Testing'de hangi araçlar kullanılır?**
   - **Cevap**:
     - TestServer (ASP.NET Core)
     - Entity Framework Core
     - xUnit/NUnit
     - Docker
     - Postman
     - Swagger

### Teknik Sorular

1. **TestServer nasıl kullanılır?**
   - **Cevap**:
```csharp
public class ApiTests
{
    private readonly TestServer _server;
    private readonly HttpClient _client;

    public ApiTests()
    {
        _server = new TestServer(new WebHostBuilder()
            .UseStartup<Startup>()
            .ConfigureTestServices(services =>
            {
                services.AddScoped<IService, MockService>();
            }));
        _client = _server.CreateClient();
    }

    [Fact]
    public async Task GetData_ReturnsExpectedResult()
    {
        var response = await _client.GetAsync("/api/data");
        response.EnsureSuccessStatusCode();
    }
}
```

2. **In-Memory Database nasıl kullanılır?**
   - **Cevap**:
```csharp
public class DatabaseTests
{
    private readonly DbContextOptions<AppDbContext> _options;

    public DatabaseTests()
    {
        _options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(databaseName: "TestDb")
            .Options;
    }

    [Fact]
    public async Task SaveData_ShouldWork()
    {
        using (var context = new AppDbContext(_options))
        {
            var repository = new Repository(context);
            await repository.Save(new Data());
            
            Assert.True(await context.Data.AnyAsync());
        }
    }
}
```

3. **Integration Test'lerde mock nasıl kullanılır?**
   - **Cevap**:
```csharp
public class ServiceTests
{
    [Fact]
    public async Task ProcessData_ShouldWork()
    {
        var mockService = new Mock<IExternalService>();
        mockService.Setup(x => x.GetData())
                  .ReturnsAsync(new Data());

        var service = new Service(mockService.Object);
        var result = await service.Process();

        Assert.NotNull(result);
    }
}
```

4. **Test verileri nasıl yönetilir?**
   - **Cevap**:
```csharp
public class TestData
{
    public static List<User> GetTestUsers()
    {
        return new List<User>
        {
            new User { Id = 1, Name = "User1" },
            new User { Id = 2, Name = "User2" }
        };
    }
}

public class UserTests
{
    [Fact]
    public void TestWithData()
    {
        var users = TestData.GetTestUsers();
        // Test logic
    }
}
```

5. **Test sırası nasıl kontrol edilir?**
   - **Cevap**:
```csharp
[Collection("IntegrationTests")]
public class OrderedTests
{
    [Fact]
    [TestOrder(1)]
    public void FirstTest()
    {
        // Test logic
    }

    [Fact]
    [TestOrder(2)]
    public void SecondTest()
    {
        // Test logic
    }
}
```

### Pratik Sorular

1. **Aşağıdaki API endpoint'ini test edin:**
```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserRepository _repository;
    private readonly IEmailService _emailService;

    public UsersController(IUserRepository repository, IEmailService emailService)
    {
        _repository = repository;
        _emailService = emailService;
    }

    [HttpPost]
    public async Task<IActionResult> CreateUser([FromBody] UserDto userDto)
    {
        var user = new User
        {
            Name = userDto.Name,
            Email = userDto.Email
        };

        await _repository.AddAsync(user);
        await _emailService.SendWelcomeEmail(user.Email);

        return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
    }
}
```

- **Cevap**:
```csharp
public class UsersControllerTests
{
    private readonly TestServer _server;
    private readonly HttpClient _client;
    private readonly Mock<IEmailService> _mockEmailService;

    public UsersControllerTests()
    {
        _mockEmailService = new Mock<IEmailService>();
        
        _server = new TestServer(new WebHostBuilder()
            .UseStartup<Startup>()
            .ConfigureTestServices(services =>
            {
                services.AddScoped(_ => _mockEmailService.Object);
            }));
            
        _client = _server.CreateClient();
    }

    [Fact]
    public async Task CreateUser_ShouldCreateUserAndSendEmail()
    {
        // Arrange
        var userDto = new UserDto
        {
            Name = "Test User",
            Email = "test@example.com"
        };

        // Act
        var response = await _client.PostAsJsonAsync("/api/users", userDto);

        // Assert
        response.EnsureSuccessStatusCode();
        _mockEmailService.Verify(x => x.SendWelcomeEmail("test@example.com"), Times.Once);
    }
}
```

2. **Aşağıdaki repository'yi test edin:**
```csharp
public class UserRepository : IUserRepository
{
    private readonly ApplicationDbContext _context;

    public UserRepository(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<User> GetUserWithOrders(int userId)
    {
        return await _context.Users
            .Include(u => u.Orders)
            .FirstOrDefaultAsync(u => u.Id == userId);
    }
}
```

- **Cevap**:
```csharp
public class UserRepositoryTests
{
    private readonly DbContextOptions<ApplicationDbContext> _options;

    public UserRepositoryTests()
    {
        _options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseInMemoryDatabase(databaseName: "TestDb")
            .Options;
    }

    [Fact]
    public async Task GetUserWithOrders_ShouldReturnUserWithOrders()
    {
        // Arrange
        using (var context = new ApplicationDbContext(_options))
        {
            var user = new User { Id = 1, Name = "Test User" };
            user.Orders.Add(new Order { Id = 1, Amount = 100 });
            
            context.Users.Add(user);
            await context.SaveChangesAsync();
        }

        // Act
        using (var context = new ApplicationDbContext(_options))
        {
            var repository = new UserRepository(context);
            var result = await repository.GetUserWithOrders(1);

            // Assert
            Assert.NotNull(result);
            Assert.Single(result.Orders);
            Assert.Equal(100, result.Orders.First().Amount);
        }
    }
}
```

### İleri Seviye Sorular

1. **Microservices mimarisinde Integration Testing nasıl yapılır?**
   - **Cevap**:
     - Service mesh kullanımı
     - API Gateway testleri
     - Event-driven testler
     - Contract testing
     - Chaos engineering

2. **Distributed sistemlerde Integration Testing nasıl yapılır?**
   - **Cevap**:
     - Message queue testleri
     - Distributed transaction testleri
     - Event sourcing testleri
     - Saga pattern testleri
     - Circuit breaker testleri

3. **CI/CD pipeline'da Integration Testing nasıl entegre edilir?**
   - **Cevap**:
     - Test ortamı yönetimi
     - Test verisi yönetimi
     - Paralel test çalıştırma
     - Test sonuçları raporlama
     - Test otomasyonu

4. **Performance testing ile Integration Testing nasıl birleştirilir?**
   - **Cevap**:
     - Load testing entegrasyonu
     - Stress testing entegrasyonu
     - Endurance testing
     - Spike testing
     - Scalability testing

5. **Security testing ile Integration Testing nasıl birleştirilir?**
   - **Cevap**:
     - Authentication testleri
     - Authorization testleri
     - Input validation testleri
     - SQL injection testleri
     - XSS testleri 