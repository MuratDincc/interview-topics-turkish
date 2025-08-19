# API Testing

## Giriş

API Testing, API'lerin doğru çalıştığını, beklenen davranışları sergilediğini ve güvenlik gereksinimlerini karşıladığını doğrulamak için yapılan test süreçleridir. Mid-level geliştiriciler için kapsamlı API testing stratejileri önemlidir.

## API Testing Türleri

### 1. Unit Testing
API controller'ların ve service'lerin bağımsız test edilmesi.

```csharp
public class UserControllerTests
{
    private readonly UserController _controller;
    private readonly Mock<IUserService> _mockUserService;
    private readonly Mock<ILogger<UserController>> _mockLogger;
    
    public UserControllerTests()
    {
        _mockUserService = new Mock<IUserService>();
        _mockLogger = new Mock<ILogger<UserController>>();
        _controller = new UserController(_mockUserService.Object, _mockLogger.Object);
    }
    
    [Fact]
    public async Task GetUser_WithValidId_ReturnsOkResult()
    {
        // Arrange
        var userId = 1;
        var expectedUser = new UserDto { Id = userId, Name = "Ahmet", Email = "ahmet@example.com" };
        _mockUserService.Setup(x => x.GetByIdAsync(userId)).ReturnsAsync(expectedUser);
        
        // Act
        var result = await _controller.GetUser(userId);
        
        // Assert
        var okResult = Assert.IsType<OkObjectResult>(result);
        var returnedUser = Assert.IsType<UserDto>(okResult.Value);
        Assert.Equal(expectedUser.Id, returnedUser.Id);
        Assert.Equal(expectedUser.Name, returnedUser.Name);
    }
    
    [Fact]
    public async Task GetUser_WithInvalidId_ReturnsNotFound()
    {
        // Arrange
        var userId = 999;
        _mockUserService.Setup(x => x.GetByIdAsync(userId)).ReturnsAsync((UserDto)null);
        
        // Act
        var result = await _controller.GetUser(userId);
        
        // Assert
        Assert.IsType<NotFoundResult>(result);
    }
    
    [Fact]
    public async Task CreateUser_WithValidData_ReturnsCreatedResult()
    {
        // Arrange
        var createUserRequest = new CreateUserRequest { Name = "Ahmet", Email = "ahmet@example.com" };
        var createdUser = new UserDto { Id = 1, Name = "Ahmet", Email = "ahmet@example.com" };
        
        _mockUserService.Setup(x => x.CreateAsync(It.IsAny<User>())).ReturnsAsync(createdUser);
        
        // Act
        var result = await _controller.CreateUser(createUserRequest);
        
        // Assert
        var createdResult = Assert.IsType<CreatedAtActionResult>(result);
        var returnedUser = Assert.IsType<UserDto>(createdResult.Value);
        Assert.Equal(createdUser.Id, returnedUser.Id);
        Assert.Equal("GetUser", createdResult.ActionName);
    }
    
    [Fact]
    public async Task CreateUser_WithInvalidData_ReturnsBadRequest()
    {
        // Arrange
        var createUserRequest = new CreateUserRequest { Name = "", Email = "invalid-email" };
        _controller.ModelState.AddModelError("Name", "Name is required");
        _controller.ModelState.AddModelError("Email", "Invalid email format");
        
        // Act
        var result = await _controller.CreateUser(createUserRequest);
        
        // Assert
        Assert.IsType<BadRequestObjectResult>(result);
    }
}
```

### 2. Integration Testing
API'lerin gerçek veritabanı ve external service'ler ile entegrasyonunun test edilmesi.

```csharp
public class UserApiIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;
    
    public UserApiIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Test database kullan
                var descriptor = services.SingleOrDefault(
                    d => d.ServiceType == typeof(DbContextOptions<ApplicationDbContext>));
                
                if (descriptor != null)
                {
                    services.Remove(descriptor);
                }
                
                services.AddDbContext<ApplicationDbContext>(options =>
                {
                    options.UseInMemoryDatabase("TestDb");
                });
            });
        });
        
        _client = _factory.CreateClient();
    }
    
    [Fact]
    public async Task CreateUser_Integration_ShouldCreateUserInDatabase()
    {
        // Arrange
        var createUserRequest = new CreateUserRequest
        {
            Name = "Integration Test User",
            Email = "integration@test.com"
        };
        
        var json = JsonSerializer.Serialize(createUserRequest);
        var content = new StringContent(json, Encoding.UTF8, "application/json");
        
        // Act
        var response = await _client.PostAsync("/api/users", content);
        
        // Assert
        response.EnsureSuccessStatusCode();
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        
        var responseContent = await response.Content.ReadAsStringAsync();
        var createdUser = JsonSerializer.Deserialize<UserDto>(responseContent, new JsonSerializerOptions
        {
            PropertyNameCaseInsensitive = true
        });
        
        Assert.NotNull(createdUser);
        Assert.Equal(createUserRequest.Name, createdUser.Name);
        Assert.Equal(createUserRequest.Email, createdUser.Email);
        
        // Verify user exists in database
        var getResponse = await _client.GetAsync($"/api/users/{createdUser.Id}");
        getResponse.EnsureSuccessStatusCode();
    }
    
    [Fact]
    public async Task GetUsers_Integration_ShouldReturnUsersFromDatabase()
    {
        // Arrange - Test data ekle
        var testUsers = new[]
        {
            new CreateUserRequest { Name = "User 1", Email = "user1@test.com" },
            new CreateUserRequest { Name = "User 2", Email = "user2@test.com" }
        };
        
        foreach (var user in testUsers)
        {
            var json = JsonSerializer.Serialize(user);
            var content = new StringContent(json, Encoding.UTF8, "application/json");
            await _client.PostAsync("/api/users", content);
        }
        
        // Act
        var response = await _client.GetAsync("/api/users");
        
        // Assert
        response.EnsureSuccessStatusCode();
        var responseContent = await response.Content.ReadAsStringAsync();
        var users = JsonSerializer.Deserialize<List<UserDto>>(responseContent, new JsonSerializerOptions
        {
            PropertyNameCaseInsensitive = true
        });
        
        Assert.NotNull(users);
        Assert.True(users.Count >= testUsers.Length);
    }
}
```

### 3. Contract Testing
API consumer'lar ile provider'lar arasındaki contract'ların test edilmesi.

```csharp
// Pact.NET kullanarak contract testing
public class UserApiContractTests
{
    private readonly IPactBuilderV3 _pactBuilder;
    
    public UserApiContractTests()
    {
        var config = new PactConfig
        {
            PactDir = Path.Join("..", "..", "..", "pacts")
        };
        
        _pactBuilder = Pact.V3("UserApi", "UserClient", config);
    }
    
    [Fact]
    public async Task GetUser_Contract_ShouldMatchExpectedResponse()
    {
        // Arrange
        var expectedUser = new UserDto
        {
            Id = 1,
            Name = "Contract Test User",
            Email = "contract@test.com"
        };
        
        _pactBuilder
            .UponReceiving("A request for a user")
            .Given("A user exists")
            .WithRequest(HttpMethod.Get, "/api/users/1")
            .WillRespond()
            .WithStatus(HttpStatusCode.OK)
            .WithHeader("Content-Type", "application/json; charset=utf-8")
            .WithJsonBody(expectedUser);
        
        await _pactBuilder.VerifyAsync(async ctx =>
        {
            var client = new HttpClient { BaseAddress = new Uri(ctx.MockServerUri) };
            var response = await client.GetAsync("/api/users/1");
            
            Assert.Equal(HttpStatusCode.OK, response.StatusCode);
            
            var content = await response.Content.ReadAsStringAsync();
            var actualUser = JsonSerializer.Deserialize<UserDto>(content);
            
            Assert.Equal(expectedUser.Id, actualUser.Id);
            Assert.Equal(expectedUser.Name, actualUser.Name);
            Assert.Equal(expectedUser.Email, actualUser.Email);
        });
    }
}
```

## Postman ve Newman ile API Testing

### Postman Collection
```json
{
  "info": {
    "name": "User API Tests",
    "description": "Comprehensive tests for User API endpoints"
  },
  "item": [
    {
      "name": "Get Users",
      "request": {
        "method": "GET",
        "header": [
          {
            "key": "Authorization",
            "value": "Bearer {{access_token}}"
          }
        ],
        "url": {
          "raw": "{{base_url}}/api/users",
          "host": ["{{base_url}}"],
          "path": ["api", "users"]
        }
      },
      "response": [
        {
          "name": "Success Response",
          "originalRequest": {
            "method": "GET",
            "header": [],
            "url": {
              "raw": "{{base_url}}/api/users",
              "host": ["{{base_url}}"],
              "path": ["api", "users"]
            }
          },
          "status": "OK",
          "code": 200,
          "_postman_previewlanguage": "json",
          "header": [
            {
              "key": "Content-Type",
              "value": "application/json"
            }
          ],
          "cookie": [],
          "body": "[\n  {\n    \"id\": 1,\n    \"name\": \"Ahmet\",\n    \"email\": \"ahmet@example.com\"\n  }\n]"
        }
      ]
    },
    {
      "name": "Create User",
      "request": {
        "method": "POST",
        "header": [
          {
            "key": "Content-Type",
            "value": "application/json"
          },
          {
            "key": "Authorization",
            "value": "Bearer {{access_token}}"
          }
        ],
        "body": {
          "mode": "raw",
          "raw": "{\n  \"name\": \"Test User\",\n  \"email\": \"test@example.com\"\n}"
        },
        "url": {
          "raw": "{{base_url}}/api/users",
          "host": ["{{base_url}}"],
          "path": ["api", "users"]
        }
      },
      "response": [
        {
          "name": "Created Response",
          "originalRequest": {
            "method": "POST",
            "header": [],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"name\": \"Test User\",\n  \"email\": \"test@example.com\"\n}"
            },
            "url": {
              "raw": "{{base_url}}/api/users",
              "host": ["{{base_url}}"],
              "path": ["api", "users"]
            }
          },
          "status": "Created",
          "code": 201,
          "_postman_previewlanguage": "json",
          "header": [
            {
              "key": "Content-Type",
              "value": "application/json"
            }
          ],
          "cookie": [],
          "body": "{\n  \"id\": 2,\n  \"name\": \"Test User\",\n  \"email\": \"test@example.com\"\n}"
        }
      ]
    }
  ],
  "variable": [
    {
      "key": "base_url",
      "value": "https://api.example.com"
    },
    {
      "key": "access_token",
      "value": "your_jwt_token_here"
    }
  ]
}
```

### Newman CLI ile Automation
```bash
# Newman installation
npm install -g newman

# Collection'ı çalıştır
newman run UserApiTests.postman_collection.json

# Environment variables ile
newman run UserApiTests.postman_collection.json -e environment.json

# CI/CD pipeline'da
newman run UserApiTests.postman_collection.json \
  --reporters cli,json \
  --reporter-json-export results.json \
  --exit-code
```

## Performance Testing

### Load Testing with NBomber
```csharp
public class UserApiLoadTests
{
    [Fact]
    public void GetUsers_LoadTest()
    {
        var httpClient = new HttpClient();
        httpClient.BaseAddress = new Uri("https://api.example.com");
        
        var scenario = ScenarioBuilder.CreateScenario("Get Users Load Test", async context =>
        {
            try
            {
                var response = await httpClient.GetAsync("/api/users");
                
                if (response.IsSuccessStatusCode)
                {
                    context.Ok();
                }
                else
                {
                    context.Fail($"HTTP {response.StatusCode}");
                }
            }
            catch (Exception ex)
            {
                context.Fail(ex.Message);
            }
        })
        .WithLoadSimulations(
            Simulation.Inject(rate: 100, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromMinutes(5))
        );
        
        NBomberRunner
            .RegisterScenarios(scenario)
            .Run();
    }
    
    [Fact]
    public void CreateUser_StressTest()
    {
        var httpClient = new HttpClient();
        httpClient.BaseAddress = new Uri("https://api.example.com");
        
        var scenario = ScenarioBuilder.CreateScenario("Create User Stress Test", async context =>
        {
            try
            {
                var userData = new { name = $"User_{Guid.NewGuid()}", email = $"user_{Guid.NewGuid()}@test.com" };
                var json = JsonSerializer.Serialize(userData);
                var content = new StringContent(json, Encoding.UTF8, "application/json");
                
                var response = await httpClient.PostAsync("/api/users", content);
                
                if (response.IsSuccessStatusCode)
                {
                    context.Ok();
                }
                else
                {
                    context.Fail($"HTTP {response.StatusCode}");
                }
            }
            catch (Exception ex)
            {
                context.Fail(ex.Message);
            }
        })
        .WithLoadSimulations(
            Simulation.Stress(rate: 50, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromMinutes(10))
        );
        
        NBomberRunner
            .RegisterScenarios(scenario)
            .Run();
    }
}
```

## Security Testing

### Authentication ve Authorization Testing
```csharp
public class UserApiSecurityTests
{
    private readonly WebApplicationFactory<Program> _factory;
    
    [Fact]
    public async Task GetUsers_WithoutAuthentication_ReturnsUnauthorized()
    {
        // Arrange
        var client = _factory.CreateClient();
        
        // Act
        var response = await client.GetAsync("/api/users");
        
        // Assert
        Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
    }
    
    [Fact]
    public async Task GetUsers_WithInvalidToken_ReturnsUnauthorized()
    {
        // Arrange
        var client = _factory.CreateClient();
        client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", "invalid_token");
        
        // Act
        var response = await client.GetAsync("/api/users");
        
        // Assert
        Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
    }
    
    [Fact]
    public async Task CreateUser_WithoutAdminRole_ReturnsForbidden()
    {
        // Arrange
        var client = _factory.CreateClient();
        var token = await GetUserToken("regular_user");
        client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", token);
        
        var userData = new { name = "Test User", email = "test@example.com" };
        var json = JsonSerializer.Serialize(userData);
        var content = new StringContent(json, Encoding.UTF8, "application/json");
        
        // Act
        var response = await client.PostAsync("/api/users", content);
        
        // Assert
        Assert.Equal(HttpStatusCode.Forbidden, response.StatusCode);
    }
    
    [Fact]
    public async Task CreateUser_WithXssPayload_ShouldSanitizeInput()
    {
        // Arrange
        var client = _factory.CreateClient();
        var token = await GetAdminToken();
        client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", token);
        
        var maliciousData = new { name = "<script>alert('xss')</script>", email = "test@example.com" };
        var json = JsonSerializer.Serialize(maliciousData);
        var content = new StringContent(json, Encoding.UTF8, "application/json");
        
        // Act
        var response = await client.PostAsync("/api/users", content);
        
        // Assert
        response.EnsureSuccessStatusCode();
        
        var responseContent = await response.Content.ReadAsStringAsync();
        var createdUser = JsonSerializer.Deserialize<UserDto>(responseContent);
        
        // XSS payload'ın sanitize edildiğini kontrol et
        Assert.DoesNotContain("<script>", createdUser.Name);
        Assert.Contains("&lt;script&gt;", createdUser.Name);
    }
}
```

## Mülakat Soruları

### Temel Sorular

1. **API testing türleri nelerdir?**
   - **Cevap**: Unit testing, integration testing, contract testing, performance testing, security testing. Her biri farklı seviyede test coverage sağlar.

2. **Integration testing nedir ve neden önemlidir?**
   - **Cevap**: API'lerin gerçek dependencies ile çalışmasının test edilmesi. Database, external services ve middleware entegrasyonunu doğrular.

3. **Contract testing nedir?**
   - **Cevap**: API consumer ve provider arasındaki contract'ların test edilmesi. Breaking changes'ları önler ve API compatibility sağlar.

4. **Postman vs Newman arasındaki fark nedir?**
   - **Cevap**: Postman GUI tool, Newman CLI tool. Newman CI/CD pipeline'da automation için kullanılır.

5. **Performance testing'de hangi metrikler önemlidir?**
   - **Cevap**: Response time, throughput, error rate, resource utilization, scalability metrics.

### Teknik Sorular

1. **API testing'de mocking nasıl yapılır?**
   - **Cevap**: Moq, NSubstitute gibi library'ler kullanarak. External dependencies mock edilir, test isolation sağlanır.

2. **Integration testing'de test database nasıl yönetilir?**
   - **Cevap**: In-memory database, test database, database seeding, test data cleanup. Her test için isolated environment.

3. **Contract testing'de Pact nasıl kullanılır?**
   - **Cevap**: Consumer-driven contracts, provider verification, contract validation. Breaking changes detection.

4. **Security testing'de hangi testler yapılır?**
   - **Cevap**: Authentication bypass, authorization testing, input validation, XSS protection, SQL injection prevention.

5. **Performance testing'de load vs stress testing arasındaki fark nedir?**
   - **Cevap**: Load testing normal yük, stress testing sistem limitlerini test eder. Performance degradation ve failure points bulunur.

## Best Practices

1. **Test Organization**
   - Test pyramid'i takip edin
   - Test isolation sağlayın
   - Test data management yapın
   - CI/CD integration kurun

2. **Test Coverage**
   - Happy path testing
   - Error scenarios testing
   - Edge cases testing
   - Security testing

3. **Performance Testing**
   - Realistic load patterns
   - Monitoring ve alerting
   - Baseline metrics
   - Continuous performance testing

4. **Security Testing**
   - OWASP Top 10 testing
   - Authentication testing
   - Authorization testing
   - Input validation testing

## Kaynaklar

- [ASP.NET Core Testing](https://docs.microsoft.com/en-us/aspnet/core/test/)
- [Postman Documentation](https://learning.postman.com/)
- [Newman CLI](https://github.com/postmanlabs/newman)
- [NBomber](https://nbomber.com/)
- [Pact.NET](https://docs.pact.io/implementation_guides/dotnet/)
