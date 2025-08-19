# Unit Testing Basics

## Giriş

Unit Testing, yazılım geliştirmede kod kalitesini artırmak ve hataları erken tespit etmek için kullanılan temel testing yaklaşımıdır. Bu dosya, .NET'te unit testing'in temellerini, framework'leri ve en iyi uygulamalarını kapsar.

## Temel Kavramlar

### 1. Unit Test Nedir?
Unit test, bir software unit'inin (method, class) izole bir şekilde test edilmesidir.

**Özellikleri:**
- Fast (Hızlı)
- Independent (Bağımsız)
- Repeatable (Tekrarlanabilir)
- Self-validating (Kendi kendini doğrulayan)
- Timely (Zamanında yazılan)

### 2. Test Anatomisi
```csharp
[Test]
public void CalculateSum_WithValidNumbers_ReturnsCorrectSum()
{
    // Arrange (Hazırlık)
    var calculator = new Calculator();
    int a = 5;
    int b = 3;
    int expected = 8;
    
    // Act (Eylem)
    int actual = calculator.Add(a, b);
    
    // Assert (Doğrulama)
    Assert.AreEqual(expected, actual);
}
```

### 3. Test Framework'leri
- **xUnit**: Modern, .NET Core için önerilen
- **NUnit**: Popüler, feature-rich
- **MSTest**: Microsoft'un framework'ü

## xUnit ile Unit Testing

### 1. Temel xUnit Kurulumu
```xml
<!-- Test project .csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net6.0</TargetFramework>
    <IsPackable>false</IsPackable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.0.0" />
    <PackageReference Include="xunit" Version="2.4.1" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.4.3" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\MyApp\MyApp.csproj" />
  </ItemGroup>
</Project>
```

### 2. Basit Test Örnekleri
```csharp
using Xunit;

public class CalculatorTests
{
    [Fact]
    public void Add_WithPositiveNumbers_ReturnsSum()
    {
        // Arrange
        var calculator = new Calculator();
        
        // Act
        var result = calculator.Add(2, 3);
        
        // Assert
        Assert.Equal(5, result);
    }
    
    [Theory]
    [InlineData(1, 2, 3)]
    [InlineData(-1, 1, 0)]
    [InlineData(0, 0, 0)]
    public void Add_WithVariousInputs_ReturnsCorrectSum(int a, int b, int expected)
    {
        // Arrange
        var calculator = new Calculator();
        
        // Act
        var result = calculator.Add(a, b);
        
        // Assert
        Assert.Equal(expected, result);
    }
}

public class Calculator
{
    public int Add(int a, int b) => a + b;
    
    public int Subtract(int a, int b) => a - b;
    
    public int Multiply(int a, int b) => a * b;
    
    public double Divide(int a, int b)
    {
        if (b == 0)
            throw new DivideByZeroException("Cannot divide by zero");
        return (double)a / b;
    }
}
```

### 3. Exception Testing
```csharp
public class CalculatorExceptionTests
{
    [Fact]
    public void Divide_ByZero_ThrowsDivideByZeroException()
    {
        // Arrange
        var calculator = new Calculator();
        
        // Act & Assert
        Assert.Throws<DivideByZeroException>(() => calculator.Divide(10, 0));
    }
    
    [Fact]
    public void Divide_ByZero_ThrowsExceptionWithCorrectMessage()
    {
        // Arrange
        var calculator = new Calculator();
        
        // Act & Assert
        var exception = Assert.Throws<DivideByZeroException>(() => calculator.Divide(10, 0));
        Assert.Equal("Cannot divide by zero", exception.Message);
    }
}
```

## Test Organization

### 1. Test Class Organization
```csharp
// Class under test: UserService
public class UserServiceTests
{
    private readonly UserService _userService;
    private readonly List<User> _testUsers;
    
    public UserServiceTests()
    {
        // Test setup
        _userService = new UserService();
        _testUsers = new List<User>
        {
            new User { Id = 1, Name = "John", Age = 25 },
            new User { Id = 2, Name = "Jane", Age = 30 }
        };
    }
    
    public class GetUserById
    {
        [Fact]
        public void ValidId_ReturnsUser()
        {
            // Test implementation
        }
        
        [Fact]
        public void InvalidId_ReturnsNull()
        {
            // Test implementation
        }
    }
    
    public class CreateUser
    {
        [Fact]
        public void ValidUser_ReturnsCreatedUser()
        {
            // Test implementation
        }
        
        [Fact]
        public void NullUser_ThrowsArgumentNullException()
        {
            // Test implementation
        }
    }
}
```

### 2. Test Data Management
```csharp
public class TestDataBuilder
{
    public static User CreateValidUser(string name = "Test User", int age = 25)
    {
        return new User
        {
            Id = Random.Shared.Next(1, 1000),
            Name = name,
            Age = age,
            Email = $"{name.Replace(" ", "").ToLower()}@test.com",
            CreatedDate = DateTime.Now
        };
    }
    
    public static List<User> CreateUserList(int count = 3)
    {
        var users = new List<User>();
        for (int i = 0; i < count; i++)
        {
            users.Add(CreateValidUser($"User {i + 1}", 20 + i));
        }
        return users;
    }
}

public class UserServiceTestsWithBuilder
{
    [Fact]
    public void CreateUser_ValidUser_ReturnsUserWithId()
    {
        // Arrange
        var service = new UserService();
        var user = TestDataBuilder.CreateValidUser("John Doe", 30);
        
        // Act
        var result = service.CreateUser(user);
        
        // Assert
        Assert.NotNull(result);
        Assert.True(result.Id > 0);
        Assert.Equal("John Doe", result.Name);
    }
}
```

## Mocking ile Testing

### 1. Moq Framework
```xml
<PackageReference Include="Moq" Version="4.18.0" />
```

```csharp
using Moq;

public interface IUserRepository
{
    User GetById(int id);
    void Save(User user);
    List<User> GetAll();
}

public class UserService
{
    private readonly IUserRepository _repository;
    
    public UserService(IUserRepository repository)
    {
        _repository = repository;
    }
    
    public User GetUser(int id)
    {
        if (id <= 0)
            throw new ArgumentException("Id must be positive", nameof(id));
            
        return _repository.GetById(id);
    }
    
    public void CreateUser(User user)
    {
        if (user == null)
            throw new ArgumentNullException(nameof(user));
            
        if (string.IsNullOrEmpty(user.Name))
            throw new ArgumentException("Name is required", nameof(user));
            
        _repository.Save(user);
    }
}
```

### 2. Mock Kullanımı
```csharp
public class UserServiceWithMockTests
{
    private readonly Mock<IUserRepository> _mockRepository;
    private readonly UserService _userService;
    
    public UserServiceWithMockTests()
    {
        _mockRepository = new Mock<IUserRepository>();
        _userService = new UserService(_mockRepository.Object);
    }
    
    [Fact]
    public void GetUser_ValidId_ReturnsUser()
    {
        // Arrange
        var expectedUser = new User { Id = 1, Name = "John" };
        _mockRepository.Setup(r => r.GetById(1)).Returns(expectedUser);
        
        // Act
        var result = _userService.GetUser(1);
        
        // Assert
        Assert.Equal(expectedUser, result);
        _mockRepository.Verify(r => r.GetById(1), Times.Once);
    }
    
    [Fact]
    public void CreateUser_ValidUser_CallsRepositorySave()
    {
        // Arrange
        var user = new User { Name = "John" };
        
        // Act
        _userService.CreateUser(user);
        
        // Assert
        _mockRepository.Verify(r => r.Save(user), Times.Once);
    }
    
    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    public void GetUser_InvalidId_ThrowsArgumentException(int invalidId)
    {
        // Act & Assert
        Assert.Throws<ArgumentException>(() => _userService.GetUser(invalidId));
    }
}
```

## Test Patterns

### 1. Test Fixtures
```csharp
public class DatabaseTestFixture : IDisposable
{
    public string ConnectionString { get; }
    
    public DatabaseTestFixture()
    {
        // Setup test database
        ConnectionString = "Data Source=:memory:";
        InitializeDatabase();
    }
    
    private void InitializeDatabase()
    {
        // Create tables, seed data
    }
    
    public void Dispose()
    {
        // Cleanup test database
    }
}

public class UserRepositoryTests : IClassFixture<DatabaseTestFixture>
{
    private readonly DatabaseTestFixture _fixture;
    
    public UserRepositoryTests(DatabaseTestFixture fixture)
    {
        _fixture = fixture;
    }
    
    [Fact]
    public void GetById_ExistingUser_ReturnsUser()
    {
        // Use _fixture.ConnectionString for test
    }
}
```

### 2. Parameterized Tests
```csharp
public class StringUtilsTests
{
    [Theory]
    [InlineData("hello", "Hello")]
    [InlineData("WORLD", "World")]
    [InlineData("", "")]
    [InlineData("a", "A")]
    public void Capitalize_VariousInputs_ReturnsCapitalizedString(string input, string expected)
    {
        // Arrange
        var utils = new StringUtils();
        
        // Act
        var result = utils.Capitalize(input);
        
        // Assert
        Assert.Equal(expected, result);
    }
    
    [Theory]
    [MemberData(nameof(GetTestData))]
    public void IsValidEmail_VariousInputs_ReturnsExpectedResult(string email, bool expected)
    {
        // Arrange
        var utils = new StringUtils();
        
        // Act
        var result = utils.IsValidEmail(email);
        
        // Assert
        Assert.Equal(expected, result);
    }
    
    public static IEnumerable<object[]> GetTestData()
    {
        yield return new object[] { "test@example.com", true };
        yield return new object[] { "invalid-email", false };
        yield return new object[] { "", false };
        yield return new object[] { "user@domain.co.uk", true };
    }
}
```

## Test Best Practices

### 1. Naming Conventions
```csharp
public class NamingConventionExamples
{
    // Pattern: MethodName_StateUnderTest_ExpectedBehavior
    
    [Fact]
    public void Add_TwoPositiveNumbers_ReturnsSum() { }
    
    [Fact]
    public void GetUser_NonExistentId_ReturnsNull() { }
    
    [Fact]
    public void ProcessPayment_InsufficientFunds_ThrowsInsufficientFundsException() { }
    
    [Fact]
    public void ValidateEmail_EmptyString_ReturnsFalse() { }
}
```

### 2. Test Independence
```csharp
public class TestIndependenceExample
{
    // ❌ Bad: Tests depend on each other
    private static int _counter = 0;
    
    [Fact]
    public void Test1_IncrementCounter_CounterIsOne()
    {
        _counter++; // Modifies shared state
        Assert.Equal(1, _counter);
    }
    
    [Fact]
    public void Test2_IncrementCounter_CounterIsTwo() // This will fail if Test1 doesn't run first
    {
        _counter++;
        Assert.Equal(2, _counter);
    }
    
    // ✅ Good: Each test is independent
    [Fact]
    public void Add_TwoNumbers_ReturnsSum()
    {
        var calculator = new Calculator(); // Fresh instance
        var result = calculator.Add(2, 3);
        Assert.Equal(5, result);
    }
}
```

## Mülakat Soruları

### Temel Sorular

1. **Unit test nedir ve neden önemlidir?**
   - **Cevap**: Kod birimlerinin izole testleri, erken hata tespiti ve kod kalitesi için kritik.

2. **Arrange, Act, Assert pattern'i nedir?**
   - **Cevap**: Test organize etme pattern'i: hazırlık, eylem, doğrulama.

3. **xUnit'te [Fact] ve [Theory] arasındaki fark nedir?**
   - **Cevap**: [Fact] parametresiz test, [Theory] parametreli test.

4. **Mock nedir ve neden kullanılır?**
   - **Cevap**: Dependency'leri taklit etme, izole test için.

5. **Test-driven development (TDD) nedir?**
   - **Cevap**: Önce test yazma, sonra kod yazma yaklaşımı.

### Teknik Sorular

1. **Exception testing nasıl yapılır?**
   - **Cevap**: Assert.Throws kullanarak expected exception'ları test etme.

2. **Parameterized test nasıl yazılır?**
   - **Cevap**: [Theory] ve [InlineData] veya [MemberData] kullanarak.

3. **Test fixture nedir ve ne zaman kullanılır?**
   - **Cevap**: Test setup/teardown için, IClassFixture ile.

4. **Code coverage nedir?**
   - **Cevap**: Test'lerin ne kadar kod kapsadığının ölçümü.

## Best Practices

### 1. **Test Writing**
- Clear test names kullanın
- AAA pattern uygulayın
- Test'leri independent tutun
- Bir test bir assertion

### 2. **Test Organization**
- Test'leri logical olarak gruplandırın
- Meaningful test data kullanın
- Test helper method'ları oluşturun
- Test documentation yazın

### 3. **Mock Usage**
- Only dependencies'i mock edin
- Verify interactions gerektiğinde
- Setup realistic return values
- Don't over-mock

### 4. **Maintenance**
- Test'leri düzenli güncelleyin
- Flaky test'leri düzeltin
- Code coverage takip edin
- Performance test'leri optimize edin

## Kaynaklar

- [xUnit Documentation](https://xunit.net/)
- [Moq Documentation](https://github.com/moq/moq4)
- [.NET Testing Guide](https://docs.microsoft.com/en-us/dotnet/core/testing/)
- [Unit Testing Best Practices](https://docs.microsoft.com/en-us/dotnet/core/testing/unit-testing-best-practices)
- [Test-Driven Development](https://martinfowler.com/bliki/TestDrivenDevelopment.html)
