# Contract Testing

## Giriş

Contract Testing, microservices architecture'da servisler arası API contract'larının tutarlılığını sağlayan testing yaklaşımıdır. Mid-level geliştiriciler için contract testing'i anlamak, service integration, API compatibility ve deployment safety için kritik öneme sahiptir. Bu dosya, Pact.NET, contract definition, consumer-driven testing ve provider verification konularını kapsar.

## Pact.NET Implementation

### 1. Consumer Contract Testing
API consumer'ları için contract testing implementasyonu.

```csharp
public class OrderServiceContractTests
{
    private readonly IPactBuilderV3 _pactBuilder;
    private readonly List<object> _pacts;
    
    public OrderServiceContractTests()
    {
        var config = new PactConfig
        {
            PactDir = Path.Join("..", "..", "..", "pacts"),
            DefaultJsonSettings = new JsonSerializerSettings
            {
                ContractResolver = new CamelCasePropertyNamesContractResolver()
            }
        };
        
        _pactBuilder = Pact.V3("OrderService", "PaymentService", config);
        _pacts = new List<object>();
    }
    
    [Fact]
    public async Task ProcessPayment_WithValidOrder_ShouldReturnSuccess()
    {
        // Arrange
        var orderId = Guid.NewGuid();
        var amount = 150.00m;
        
        var expectedRequest = new PaymentRequest
        {
            OrderId = orderId,
            Amount = amount,
            Currency = "USD",
            PaymentMethod = "CreditCard"
        };
        
        var expectedResponse = new PaymentResponse
        {
            PaymentId = Guid.NewGuid(),
            Status = "Success",
            TransactionId = "TXN123456",
            ProcessedAt = DateTime.UtcNow
        };
        
        _pactBuilder
            .UponReceiving("A valid payment request")
            .Given("Payment service is available")
            .WithRequest(HttpMethod.Post, "/api/payments")
            .WithJsonBody(expectedRequest)
            .WillRespond()
            .WithStatus(HttpStatusCode.OK)
            .WithJsonBody(expectedResponse);
        
        var pact = _pactBuilder.Build();
        _pacts.Add(pact);
        
        // Act
        using var client = new HttpClient { BaseAddress = new Uri(pact.MockServerUri) };
        var response = await client.PostAsJsonAsync("/api/payments", expectedRequest);
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var responseContent = await response.Content.ReadFromJsonAsync<PaymentResponse>();
        responseContent.Should().NotBeNull();
        responseContent.Status.Should().Be("Success");
        
        pact.Verify();
    }
    
    [Fact]
    public async Task ProcessPayment_WithInvalidOrder_ShouldReturnBadRequest()
    {
        // Arrange
        var invalidRequest = new PaymentRequest
        {
            OrderId = Guid.Empty,
            Amount = -100.00m,
            Currency = "",
            PaymentMethod = ""
        };
        
        var expectedError = new ErrorResponse
        {
            Error = "Validation failed",
            Details = new List<string> { "OrderId cannot be empty", "Amount must be positive", "Currency is required" }
        };
        
        _pactBuilder
            .UponReceiving("An invalid payment request")
            .Given("Payment service validates input")
            .WithRequest(HttpMethod.Post, "/api/payments")
            .WithJsonBody(invalidRequest)
            .WillRespond()
            .WithStatus(HttpStatusCode.BadRequest)
            .WithJsonBody(expectedError);
        
        var pact = _pactBuilder.Build();
        _pacts.Add(pact);
        
        // Act
        using var client = new HttpClient { BaseAddress = new Uri(pact.MockServerUri) };
        var response = await client.PostAsJsonAsync("/api/payments", invalidRequest);
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
        var errorContent = await response.Content.ReadFromJsonAsync<ErrorResponse>();
        errorContent.Should().NotBeNull();
        errorContent.Error.Should().Be("Validation failed");
        
        pact.Verify();
    }
    
    [Fact]
    public async Task GetPaymentStatus_WithValidPaymentId_ShouldReturnStatus()
    {
        // Arrange
        var paymentId = Guid.NewGuid();
        var expectedResponse = new PaymentStatusResponse
        {
            PaymentId = paymentId,
            Status = "Completed",
            LastUpdated = DateTime.UtcNow,
            ProcessingTime = TimeSpan.FromSeconds(2.5)
        };
        
        _pactBuilder
            .UponReceiving("A request for payment status")
            .Given($"Payment {paymentId} exists")
            .WithRequest(HttpMethod.Get, $"/api/payments/{paymentId}/status")
            .WillRespond()
            .WithStatus(HttpStatusCode.OK)
            .WithJsonBody(expectedResponse);
        
        var pact = _pactBuilder.Build();
        _pacts.Add(pact);
        
        // Act
        using var client = new HttpClient { BaseAddress = new Uri(pact.MockServerUri) };
        var response = await client.GetAsync($"/api/payments/{paymentId}/status");
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var statusResponse = await response.Content.ReadFromJsonAsync<PaymentStatusResponse>();
        statusResponse.Should().NotBeNull();
        statusResponse.PaymentId.Should().Be(paymentId);
        
        pact.Verify();
    }
    
    public void Dispose()
    {
        foreach (var pact in _pacts)
        {
            pact.Dispose();
        }
    }
}

public class PaymentRequest
{
    public Guid OrderId { get; set; }
    public decimal Amount { get; set; }
    public string Currency { get; set; }
    public string PaymentMethod { get; set; }
}

public class PaymentResponse
{
    public Guid PaymentId { get; set; }
    public string Status { get; set; }
    public string TransactionId { get; set; }
    public DateTime ProcessedAt { get; set; }
}

public class PaymentStatusResponse
{
    public Guid PaymentId { get; set; }
    public string Status { get; set; }
    public DateTime LastUpdated { get; set; }
    public TimeSpan ProcessingTime { get; set; }
}

public class ErrorResponse
{
    public string Error { get; set; }
    public List<string> Details { get; set; }
}
```

### 2. Provider Contract Testing
API provider'ları için contract testing implementasyonu.

```csharp
public class PaymentServiceProviderTests
{
    private readonly WebApplicationFactory<Program> _factory;
    private readonly PactVerifier _pactVerifier;
    
    public PaymentServiceProviderTests()
    {
        _factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.ConfigureServices(services =>
                {
                    // Configure test database
                    services.RemoveAll<DbContextOptions<PaymentDbContext>>();
                    services.AddDbContext<PaymentDbContext>(options =>
                    {
                        options.UseInMemoryDatabase("TestDb");
                    });
                    
                    // Configure test services
                    services.AddScoped<IPaymentProcessor, MockPaymentProcessor>();
                    services.AddScoped<IPaymentValidator, PaymentValidator>();
                });
            });
        
        _pactVerifier = new PactVerifier("PaymentService");
    }
    
    [Fact]
    public async Task VerifyConsumerContracts()
    {
        // Arrange
        var pactFiles = Directory.GetFiles("../../../pacts", "*.json");
        
        foreach (var pactFile in pactFiles)
        {
            var pactFileInfo = new FileInfo(pactFile);
            var consumerName = pactFileInfo.Name.Split('-')[0];
            
            // Act & Assert
            _pactVerifier
                .ServiceProvider("PaymentService", _factory.CreateClient())
                .HonoursPactWith(consumerName)
                .PactUri(pactFile)
                .Verify();
        }
    }
    
    [Fact]
    public async Task VerifySpecificConsumerContract()
    {
        // Arrange
        var consumerName = "OrderService";
        var pactFile = $"../../../pacts/{consumerName}-PaymentService.json";
        
        if (!File.Exists(pactFile))
        {
            throw new FileNotFoundException($"Pact file not found: {pactFile}");
        }
        
        // Act & Assert
        _pactVerifier
            .ServiceProvider("PaymentService", _factory.CreateClient())
            .HonoursPactWith(consumerName)
            .PactUri(pactFile)
            .Verify();
    }
    
    [Fact]
    public async Task VerifyContractWithCustomHeaders()
    {
        // Arrange
        var consumerName = "OrderService";
        var pactFile = $"../../../pacts/{consumerName}-PaymentService.json";
        
        // Act & Assert
        _pactVerifier
            .ServiceProvider("PaymentService", _factory.CreateClient())
            .HonoursPactWith(consumerName)
            .PactUri(pactFile)
            .WithRequestCustomisation(request =>
            {
                request.Headers.Add("X-Test-Header", "TestValue");
                return request;
            })
            .Verify();
    }
    
    [Fact]
    public async Task VerifyContractWithStateHandling()
    {
        // Arrange
        var consumerName = "OrderService";
        var pactFile = $"../../../pacts/{consumerName}-PaymentService.json";
        
        // Act & Assert
        _pactVerifier
            .ServiceProvider("PaymentService", _factory.CreateClient())
            .HonoursPactWith(consumerName)
            .PactUri(pactFile)
            .WithState("Payment service is available", async () =>
            {
                // Setup test data or service state
                using var scope = _factory.Services.CreateScope();
                var context = scope.ServiceProvider.GetRequiredService<PaymentDbContext>();
                
                // Ensure database is clean and ready
                context.Database.EnsureCreated();
                
                return Task.CompletedTask;
            })
            .Verify();
    }
}

public class MockPaymentProcessor : IPaymentProcessor
{
    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        await Task.Delay(100); // Simulate processing time
        
        if (request.Amount <= 0)
        {
            return new PaymentResult
            {
                Success = false,
                ErrorMessage = "Amount must be positive"
            };
        }
        
        return new PaymentResult
        {
            Success = true,
            PaymentId = Guid.NewGuid(),
            TransactionId = $"TXN{DateTime.UtcNow.Ticks}",
            ProcessedAt = DateTime.UtcNow
        };
    }
}

public class PaymentValidator : IPaymentValidator
{
    public ValidationResult ValidatePayment(PaymentRequest request)
    {
        var errors = new List<string>();
        
        if (request.OrderId == Guid.Empty)
            errors.Add("OrderId cannot be empty");
        
        if (request.Amount <= 0)
            errors.Add("Amount must be positive");
        
        if (string.IsNullOrEmpty(request.Currency))
            errors.Add("Currency is required");
        
        if (string.IsNullOrEmpty(request.PaymentMethod))
            errors.Add("Payment method is required");
        
        return new ValidationResult
        {
            IsValid = !errors.Any(),
            Errors = errors
        };
    }
}

public interface IPaymentProcessor
{
    Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request);
}

public interface IPaymentValidator
{
    ValidationResult ValidatePayment(PaymentRequest request);
}

public class PaymentResult
{
    public bool Success { get; set; }
    public Guid PaymentId { get; set; }
    public string TransactionId { get; set; }
    public DateTime ProcessedAt { get; set; }
    public string ErrorMessage { get; set; }
}

public class ValidationResult
{
    public bool IsValid { get; set; }
    public List<string> Errors { get; set; } = new();
}
```

### 3. Contract Testing Service
Contract testing'i yöneten servis.

```csharp
public class ContractTestingService : IContractTestingService
{
    private readonly ILogger<ContractTestingService> _logger;
    private readonly IConfiguration _configuration;
    private readonly string _pactBrokerUrl;
    private readonly string _pactBrokerToken;
    
    public ContractTestingService(ILogger<ContractTestingService> logger, IConfiguration configuration)
    {
        _logger = logger;
        _configuration = configuration;
        _pactBrokerUrl = _configuration["Pact:BrokerUrl"] ?? "http://localhost:9292";
        _pactBrokerToken = _configuration["Pact:BrokerToken"];
    }
    
    public async Task<bool> PublishConsumerContractsAsync(string consumerName, string version)
    {
        try
        {
            _logger.LogInformation("Publishing consumer contracts for {ConsumerName} version {Version}", 
                consumerName, version);
            
            var pactFiles = Directory.GetFiles("pacts", $"{consumerName}-*.json");
            
            foreach (var pactFile in pactFiles)
            {
                await PublishPactFileAsync(pactFile, consumerName, version);
            }
            
            _logger.LogInformation("Successfully published {Count} contract(s) for {ConsumerName}", 
                pactFiles.Length, consumerName);
            
            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error publishing consumer contracts for {ConsumerName}", consumerName);
            return false;
        }
    }
    
    public async Task<bool> VerifyProviderContractsAsync(string providerName, string version)
    {
        try
        {
            _logger.LogInformation("Verifying provider contracts for {ProviderName} version {Version}", 
                providerName, version);
            
            var verificationResult = await RunProviderVerificationAsync(providerName, version);
            
            if (verificationResult.Success)
            {
                _logger.LogInformation("Provider verification completed successfully for {ProviderName}", providerName);
            }
            else
            {
                _logger.LogWarning("Provider verification failed for {ProviderName}: {Error}", 
                    providerName, verificationResult.Error);
            }
            
            return verificationResult.Success;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error verifying provider contracts for {ProviderName}", providerName);
            return false;
        }
    }
    
    public async Task<List<ContractInfo>> GetContractInfoAsync()
    {
        try
        {
            var contracts = new List<ContractInfo>();
            
            // Get consumer contracts
            var consumerContracts = await GetConsumerContractsAsync();
            contracts.AddRange(consumerContracts);
            
            // Get provider contracts
            var providerContracts = await GetProviderContractsAsync();
            contracts.AddRange(providerContracts);
            
            return contracts;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting contract information");
            return new List<ContractInfo>();
        }
    }
    
    public async Task<bool> ValidateContractCompatibilityAsync(string consumerName, string providerName)
    {
        try
        {
            _logger.LogInformation("Validating contract compatibility between {ConsumerName} and {ProviderName}", 
                consumerName, providerName);
            
            var consumerPacts = await GetConsumerPactsAsync(consumerName);
            var providerPacts = await GetProviderPactsAsync(providerName);
            
            var compatibilityIssues = new List<string>();
            
            foreach (var consumerPact in consumerPacts)
            {
                var providerPact = providerPacts.FirstOrDefault(p => p.Endpoint == consumerPact.Endpoint);
                
                if (providerPact == null)
                {
                    compatibilityIssues.Add($"Provider {providerName} does not support endpoint {consumerPact.Endpoint}");
                    continue;
                }
                
                var compatibilityResult = await ValidatePactCompatibilityAsync(consumerPact, providerPact);
                if (!compatibilityResult.IsCompatible)
                {
                    compatibilityIssues.AddRange(compatibilityResult.Issues);
                }
            }
            
            if (compatibilityIssues.Any())
            {
                _logger.LogWarning("Contract compatibility issues found: {Issues}", 
                    string.Join(", ", compatibilityIssues));
                return false;
            }
            
            _logger.LogInformation("Contract compatibility validation passed for {ConsumerName} and {ProviderName}", 
                consumerName, providerName);
            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error validating contract compatibility");
            return false;
        }
    }
    
    private async Task PublishPactFileAsync(string pactFile, string consumerName, string version)
    {
        try
        {
            var pactContent = await File.ReadAllTextAsync(pactFile);
            var pactData = JsonSerializer.Deserialize<JsonElement>(pactContent);
            
            var publishUrl = $"{_pactBrokerUrl}/pacts/provider/{pactData.GetProperty("provider").GetProperty("name").GetString()}/consumer/{consumerName}/version/{version}";
            
            using var client = CreateHttpClient();
            var content = new StringContent(pactContent, Encoding.UTF8, "application/json");
            
            var response = await client.PutAsync(publishUrl, content);
            
            if (!response.IsSuccessStatusCode)
            {
                var errorContent = await response.Content.ReadAsStringAsync();
                throw new Exception($"Failed to publish pact: {response.StatusCode}, Error: {errorContent}");
            }
            
            _logger.LogDebug("Published pact file: {PactFile}", pactFile);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error publishing pact file: {PactFile}", pactFile);
            throw;
        }
    }
    
    private async Task<VerificationResult> RunProviderVerificationAsync(string providerName, string version)
    {
        try
        {
            var verificationUrl = $"{_pactBrokerUrl}/pacts/provider/{providerName}/latest";
            
            using var client = CreateHttpClient();
            var response = await client.GetAsync(verificationUrl);
            
            if (!response.IsSuccessStatusCode)
            {
                return new VerificationResult
                {
                    Success = false,
                    Error = $"Failed to get verification URL: {response.StatusCode}"
                };
            }
            
            // In a real implementation, this would run the actual provider verification
            // For now, return success
            return new VerificationResult { Success = true };
        }
        catch (Exception ex)
        {
            return new VerificationResult
            {
                Success = false,
                Error = ex.Message
            };
        }
    }
    
    private async Task<List<ContractInfo>> GetConsumerContractsAsync()
    {
        // Implementation to get consumer contract information
        await Task.Delay(100);
        return new List<ContractInfo>();
    }
    
    private async Task<List<ContractInfo>> GetProviderContractsAsync()
    {
        // Implementation to get provider contract information
        await Task.Delay(100);
        return new List<ContractInfo>();
    }
    
    private async Task<List<PactInfo>> GetConsumerPactsAsync(string consumerName)
    {
        // Implementation to get consumer pact information
        await Task.Delay(100);
        return new List<PactInfo>();
    }
    
    private async Task<List<PactInfo>> GetProviderPactsAsync(string providerName)
    {
        // Implementation to get provider pact information
        await Task.Delay(100);
        return new List<PactInfo>();
    }
    
    private async Task<CompatibilityResult> ValidatePactCompatibilityAsync(PactInfo consumerPact, PactInfo providerPact)
    {
        // Implementation to validate pact compatibility
        await Task.Delay(100);
        return new CompatibilityResult { IsCompatible = true };
    }
    
    private HttpClient CreateHttpClient()
    {
        var client = new HttpClient();
        
        if (!string.IsNullOrEmpty(_pactBrokerToken))
        {
            client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", _pactBrokerToken);
        }
        
        return client;
    }
}

public interface IContractTestingService
{
    Task<bool> PublishConsumerContractsAsync(string consumerName, string version);
    Task<bool> VerifyProviderContractsAsync(string providerName, string version);
    Task<List<ContractInfo>> GetContractInfoAsync();
    Task<bool> ValidateContractCompatibilityAsync(string consumerName, string providerName);
}

public class ContractInfo
{
    public string Name { get; set; }
    public string Type { get; set; }
    public string Version { get; set; }
    public DateTime LastUpdated { get; set; }
    public string Status { get; set; }
}

public class PactInfo
{
    public string Endpoint { get; set; }
    public string Method { get; set; }
    public object Request { get; set; }
    public object Response { get; set; }
}

public class VerificationResult
{
    public bool Success { get; set; }
    public string Error { get; set; }
}

public class CompatibilityResult
{
    public bool IsCompatible { get; set; }
    public List<string> Issues { get; set; } = new();
}
```

## Mülakat Soruları

### Temel Sorular

1. **Contract Testing nedir?**
   - **Cevap**: API contract'larının tutarlılığını sağlayan testing yaklaşımı.

2. **Consumer-Driven Testing nedir?**
   - **Cevap**: Consumer'ların beklediği API behavior'larını test eden yaklaşım.

3. **Pact nedir?**
   - **Cevap**: Consumer ve provider arasındaki contract'ı tanımlayan format.

4. **Contract testing ne zaman kullanılır?**
   - **Cevap**: Microservices, API integration, deployment safety.

5. **Pact Broker nedir?**
   - **Cevap**: Pact contract'larını saklayan ve yöneten merkezi sistem.

### Teknik Sorular

1. **Pact.NET nasıl implement edilir?**
   - **Cevap**: Consumer tests, provider verification, pact publishing.

2. **Contract validation nasıl yapılır?**
   - **Cevap**: Request/response matching, schema validation, state handling.

3. **Provider verification nasıl çalışır?**
   - **Cevap**: Pact broker integration, automated testing, compatibility checking.

4. **Contract testing CI/CD'de nasıl kullanılır?**
   - **Cevap**: Pre-deployment validation, contract compatibility checks.

5. **Contract testing performance nasıl optimize edilir?**
   - **Cevap**: Parallel execution, caching, selective testing.

## Best Practices

1. **Contract Design**
   - Clear API specifications yazın
   - Versioning strategy belirleyin
   - Backward compatibility sağlayın
   - Documentation ekleyin

2. **Testing Strategy**
   - Consumer-driven approach kullanın
   - Comprehensive scenarios test edin
   - Edge cases cover edin
   - Performance testing ekleyin

3. **Integration & Deployment**
   - CI/CD pipeline entegre edin
   - Pre-deployment validation yapın
   - Contract compatibility check edin
   - Rollback procedures tanımlayın

4. **Monitoring & Maintenance**
   - Contract changes track edin
   - Breaking changes monitor edin
   - Performance metrics collect edin
   - Continuous improvement sağlayın

5. **Team Collaboration**
   - Consumer-provider communication sağlayın
   - Contract review process implement edin
   - Change management procedures tanımlayın
   - Knowledge sharing yapın

## Kaynaklar

- [Pact.NET](https://docs.pact.io/implementation_guides/dotnet/)
- [Contract Testing](https://docs.pact.io/contract_testing/)
- [Consumer-Driven Contracts](https://martinfowler.com/articles/consumerDrivenContracts.html)
- [Pact Broker](https://docs.pact.io/pact_broker/)
- [API Testing Strategies](https://docs.microsoft.com/en-us/azure/architecture/patterns/anti-corruption-layer)
