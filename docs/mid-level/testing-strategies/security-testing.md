# Security Testing

## Giriş

Security Testing, uygulamanın güvenlik açıklarını tespit eden ve güvenlik önlemlerinin etkinliğini değerlendiren testing yaklaşımıdır. Mid-level geliştiriciler için security testing'i anlamak, vulnerability assessment, penetration testing ve security compliance için kritik öneme sahiptir. Bu dosya, OWASP testing, security scanning, vulnerability assessment ve security automation konularını kapsar.

## OWASP Testing Implementation

### 1. Authentication Testing
Authentication güvenlik testleri implementasyonu.

```csharp
public class AuthenticationSecurityTests
{
    private readonly WebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;
    
    public AuthenticationSecurityTests()
    {
        _factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.ConfigureServices(services =>
                {
                    // Configure test services
                    services.AddScoped<IAuthenticationService, MockAuthenticationService>();
                    services.AddScoped<IPasswordHasher, MockPasswordHasher>();
                });
            });
        
        _client = _factory.CreateClient();
    }
    
    [Fact]
    public async Task Login_WithValidCredentials_ShouldReturnJwtToken()
    {
        // Arrange
        var loginRequest = new LoginRequest
        {
            Username = "testuser",
            Password = "TestPassword123!"
        };
        
        // Act
        var response = await _client.PostAsJsonAsync("/api/auth/login", loginRequest);
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var loginResponse = await response.Content.ReadFromJsonAsync<LoginResponse>();
        loginResponse.Should().NotBeNull();
        loginResponse.Token.Should().NotBeNullOrEmpty();
        loginResponse.RefreshToken.Should().NotBeNullOrEmpty();
    }
    
    [Fact]
    public async Task Login_WithInvalidCredentials_ShouldReturnUnauthorized()
    {
        // Arrange
        var loginRequest = new LoginRequest
        {
            Username = "invaliduser",
            Password = "WrongPassword"
        };
        
        // Act
        var response = await _client.PostAsJsonAsync("/api/auth/login", loginRequest);
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }
    
    [Fact]
    public async Task Login_WithEmptyCredentials_ShouldReturnBadRequest()
    {
        // Arrange
        var loginRequest = new LoginRequest
        {
            Username = "",
            Password = ""
        };
        
        // Act
        var response = await _client.PostAsJsonAsync("/api/auth/login", loginRequest);
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }
    
    [Fact]
    public async Task Login_WithSqlInjectionAttempt_ShouldNotBeVulnerable()
    {
        // Arrange
        var sqlInjectionAttempts = new[]
        {
            "'; DROP TABLE Users; --",
            "' OR '1'='1",
            "' UNION SELECT * FROM Users --",
            "admin'--",
            "'; EXEC xp_cmdshell('dir'); --"
        };
        
        foreach (var attempt in sqlInjectionAttempts)
        {
            var loginRequest = new LoginRequest
            {
                Username = attempt,
                Password = "TestPassword123!"
            };
            
            // Act
            var response = await _client.PostAsJsonAsync("/api/auth/login", loginRequest);
            
            // Assert - Should not crash or expose sensitive information
            response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
            
            var responseContent = await response.Content.ReadAsStringAsync();
            responseContent.Should().NotContain("error in your SQL syntax");
            responseContent.Should().NotContain("System.Data.SqlClient");
            responseContent.Should().NotContain("Exception");
        }
    }
    
    [Fact]
    public async Task Login_WithXssAttempt_ShouldNotBeVulnerable()
    {
        // Arrange
        var xssAttempts = new[]
        {
            "<script>alert('XSS')</script>",
            "javascript:alert('XSS')",
            "<img src=x onerror=alert('XSS')>",
            "';alert('XSS');//",
            "<svg onload=alert('XSS')>"
        };
        
        foreach (var attempt in xssAttempts)
        {
            var loginRequest = new LoginRequest
            {
                Username = attempt,
                Password = "TestPassword123!"
            };
            
            // Act
            var response = await _client.PostAsJsonAsync("/api/auth/login", loginRequest);
            
            // Assert - Should not execute scripts
            response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
            
            var responseContent = await response.Content.ReadAsStringAsync();
            responseContent.Should().NotContain("<script>");
            responseContent.Should().NotContain("javascript:");
            responseContent.Should().NotContain("onerror=");
            responseContent.Should().NotContain("onload=");
        }
    }
    
    [Fact]
    public async Task Login_WithBruteForceAttempt_ShouldImplementRateLimiting()
    {
        // Arrange
        var loginRequest = new LoginRequest
        {
            Username = "testuser",
            Password = "WrongPassword"
        };
        
        var attempts = new List<HttpResponseMessage>();
        
        // Act - Attempt multiple logins
        for (int i = 0; i < 10; i++)
        {
            var response = await _client.PostAsJsonAsync("/api/auth/login", loginRequest);
            attempts.Add(response);
        }
        
        // Assert - Should implement rate limiting after multiple attempts
        var unauthorizedCount = attempts.Count(r => r.StatusCode == HttpStatusCode.Unauthorized);
        var tooManyRequestsCount = attempts.Count(r => r.StatusCode == HttpStatusCode.TooManyRequests);
        
        // Should have some rate limiting response
        (tooManyRequestsCount > 0 || unauthorizedCount < 10).Should().BeTrue();
    }
    
    [Fact]
    public async Task Password_ComplexityRequirements_ShouldBeEnforced()
    {
        // Arrange
        var weakPasswords = new[]
        {
            "123456",
            "password",
            "qwerty",
            "abc123",
            "password123"
        };
        
        var registerRequest = new RegisterRequest
        {
            Username = "testuser",
            Email = "test@example.com",
            Password = "weakpassword"
        };
        
        foreach (var weakPassword in weakPasswords)
        {
            registerRequest.Password = weakPassword;
            
            // Act
            var response = await _client.PostAsJsonAsync("/api/auth/register", registerRequest);
            
            // Assert - Should reject weak passwords
            response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
            
            var errorResponse = await response.Content.ReadFromJsonAsync<ErrorResponse>();
            errorResponse.Should().NotBeNull();
            errorResponse.Message.Should().Contain("password");
        }
    }
    
    [Fact]
    public async Task JwtToken_ShouldBeProperlyValidated()
    {
        // Arrange
        var loginRequest = new LoginRequest
        {
            Username = "testuser",
            Password = "TestPassword123!"
        };
        
        var loginResponse = await _client.PostAsJsonAsync("/api/auth/login", loginRequest);
        var loginResult = await loginResponse.Content.ReadFromJsonAsync<LoginResponse>();
        
        // Act - Try to access protected endpoint with valid token
        _client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", loginResult.Token);
        var protectedResponse = await _client.GetAsync("/api/orders");
        
        // Assert - Should allow access with valid token
        protectedResponse.StatusCode.Should().Be(HttpStatusCode.OK);
        
        // Act - Try to access with invalid token
        _client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", "invalid-token");
        var invalidTokenResponse = await _client.GetAsync("/api/orders");
        
        // Assert - Should reject invalid token
        invalidTokenResponse.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
        
        // Act - Try to access with expired token (if we can generate one)
        var expiredToken = GenerateExpiredToken();
        _client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", expiredToken);
        var expiredTokenResponse = await _client.GetAsync("/api/orders");
        
        // Assert - Should reject expired token
        expiredTokenResponse.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }
    
    private string GenerateExpiredToken()
    {
        // This would generate a JWT token that's already expired
        // Implementation depends on your JWT library
        return "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyLCJleHAiOjE1MTYyMzkwMjJ9.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c";
    }
}

public class LoginRequest
{
    public string Username { get; set; }
    public string Password { get; set; }
}

public class LoginResponse
{
    public string Token { get; set; }
    public string RefreshToken { get; set; }
    public DateTime ExpiresAt { get; set; }
}

public class RegisterRequest
{
    public string Username { get; set; }
    public string Email { get; set; }
    public string Password { get; set; }
}

public class ErrorResponse
{
    public string Message { get; set; }
    public List<string> Details { get; set; } = new();
}
```

### 2. Authorization Testing
Authorization güvenlik testleri implementasyonu.

```csharp
public class AuthorizationSecurityTests
{
    private readonly WebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;
    
    public AuthorizationSecurityTests()
    {
        _factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.ConfigureServices(services =>
                {
                    // Configure test services
                    services.AddScoped<IAuthorizationService, MockAuthorizationService>();
                    services.AddScoped<IRoleService, MockRoleService>();
                });
            });
        
        _client = _factory.CreateClient();
    }
    
    [Fact]
    public async Task AdminEndpoint_WithUserRole_ShouldReturnForbidden()
    {
        // Arrange
        var userToken = await GetUserTokenAsync("user");
        _client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", userToken);
        
        // Act
        var response = await _client.GetAsync("/api/admin/users");
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Forbidden);
    }
    
    [Fact]
    public async Task AdminEndpoint_WithAdminRole_ShouldReturnOk()
    {
        // Arrange
        var adminToken = await GetAdminTokenAsync("admin");
        _client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", adminToken);
        
        // Act
        var response = await _client.GetAsync("/api/admin/users");
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
    }
    
    [Fact]
    public async Task UserEndpoint_WithDifferentUserId_ShouldReturnForbidden()
    {
        // Arrange
        var userToken = await GetUserTokenAsync("user1");
        _client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", userToken);
        
        // Act - Try to access another user's data
        var response = await _client.GetAsync("/api/users/user2/profile");
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Forbidden);
    }
    
    [Fact]
    public async Task ResourceAccess_WithInsufficientPermissions_ShouldReturnForbidden()
    {
        // Arrange
        var userToken = await GetUserTokenAsync("user");
        _client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", userToken);
        
        // Act - Try to access resource without permission
        var response = await _client.DeleteAsync("/api/orders/123");
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Forbidden);
    }
    
    [Fact]
    public async Task RoleEscalation_ShouldNotBePossible()
    {
        // Arrange
        var userToken = await GetUserTokenAsync("user");
        _client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", userToken);
        
        // Act - Try to escalate role
        var roleUpdateRequest = new RoleUpdateRequest
        {
            UserId = "user",
            NewRole = "Admin"
        };
        
        var response = await _client.PutAsJsonAsync("/api/users/user/role", roleUpdateRequest);
        
        // Assert - Should not allow role escalation
        response.StatusCode.Should().Be(HttpStatusCode.Forbidden);
    }
    
    [Fact]
    public async Task HorizontalPrivilegeEscalation_ShouldBePrevented()
    {
        // Arrange
        var user1Token = await GetUserTokenAsync("user1");
        _client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", user1Token);
        
        // Act - Try to access user2's data
        var response = await _client.GetAsync("/api/users/user2/orders");
        
        // Assert - Should prevent horizontal privilege escalation
        response.StatusCode.Should().Be(HttpStatusCode.Forbidden);
    }
    
    [Fact]
    public async Task VerticalPrivilegeEscalation_ShouldBePrevented()
    {
        // Arrange
        var userToken = await GetUserTokenAsync("user");
        _client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", userToken);
        
        // Act - Try to access admin functionality
        var adminActions = new[]
        {
            "/api/admin/system-status",
            "/api/admin/user-management",
            "/api/admin/audit-logs",
            "/api/admin/configuration"
        };
        
        foreach (var endpoint in adminActions)
        {
            var response = await _client.GetAsync(endpoint);
            response.StatusCode.Should().Be(HttpStatusCode.Forbidden);
        }
    }
    
    private async Task<string> GetUserTokenAsync(string username)
    {
        var loginRequest = new LoginRequest
        {
            Username = username,
            Password = "TestPassword123!"
        };
        
        var response = await _client.PostAsJsonAsync("/api/auth/login", loginRequest);
        var loginResult = await response.Content.ReadFromJsonAsync<LoginResponse>();
        
        return loginResult.Token;
    }
    
    private async Task<string> GetAdminTokenAsync(string username)
    {
        var loginRequest = new LoginRequest
        {
            Username = username,
            Password = "AdminPassword123!"
        };
        
        var response = await _client.PostAsJsonAsync("/api/auth/login", loginRequest);
        var loginResult = await response.Content.ReadFromJsonAsync<LoginResponse>();
        
        return loginResult.Token;
    }
}

public class RoleUpdateRequest
{
    public string UserId { get; set; }
    public string NewRole { get; set; }
}
```

### 3. Input Validation Testing
Input validation güvenlik testleri implementasyonu.

```csharp
public class InputValidationSecurityTests
{
    private readonly WebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;
    
    public InputValidationSecurityTests()
    {
        _factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.ConfigureServices(services =>
                {
                    // Configure test services
                    services.AddScoped<IValidationService, ValidationService>();
                });
            });
        
        _client = _factory.CreateClient();
    }
    
    [Fact]
    public async Task CreateOrder_WithMaliciousInput_ShouldBeSanitized()
    {
        // Arrange
        var maliciousInputs = new[]
        {
            "<script>alert('XSS')</script>",
            "'; DROP TABLE Orders; --",
            "../../../etc/passwd",
            "javascript:alert('XSS')",
            "<img src=x onerror=alert('XSS')>"
        };
        
        foreach (var maliciousInput in maliciousInputs)
        {
            var orderRequest = new CreateOrderRequest
            {
                CustomerId = Guid.NewGuid(),
                Notes = maliciousInput,
                Items = new List<OrderItem>
                {
                    new OrderItem
                    {
                        ProductId = Guid.NewGuid(),
                        Quantity = 1,
                        Price = 100,
                        Notes = maliciousInput
                    }
                }
            };
            
            // Act
            var response = await _client.PostAsJsonAsync("/api/orders", orderRequest);
            
            // Assert - Should accept the request but sanitize the input
            response.StatusCode.Should().Be(HttpStatusCode.Created);
            
            var orderResponse = await response.Content.ReadFromJsonAsync<OrderResponse>();
            orderResponse.Should().NotBeNull();
            
            // Check that malicious content is sanitized
            orderResponse.Notes.Should().NotContain("<script>");
            orderResponse.Notes.Should().NotContain("javascript:");
            orderResponse.Notes.Should().NotContain("onerror=");
            orderResponse.Notes.Should().NotContain("DROP TABLE");
        }
    }
    
    [Fact]
    public async Task FileUpload_WithMaliciousFiles_ShouldBeRejected()
    {
        // Arrange
        var maliciousFiles = new[]
        {
            ("malicious.exe", "application/x-msdownload", new byte[] { 0x4D, 0x5A, 0x90, 0x00 }),
            ("script.php", "application/x-php", Encoding.UTF8.GetBytes("<?php system($_GET['cmd']); ?>")),
            ("shell.sh", "application/x-sh", Encoding.UTF8.GetBytes("#!/bin/bash\nrm -rf /")),
            ("test.asp", "application/x-asp", Encoding.UTF8.GetBytes("<% Response.Write(Request.QueryString(\"cmd\")) %>"))
        };
        
        foreach (var (filename, contentType, content) in maliciousFiles)
        {
            var formData = new MultipartFormDataContent();
            var fileContent = new ByteArrayContent(content);
            fileContent.Headers.ContentType = MediaTypeHeaderValue.Parse(contentType);
            formData.Add(fileContent, "file", filename);
            
            // Act
            var response = await _client.PostAsync("/api/files/upload", formData);
            
            // Assert - Should reject malicious files
            response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
            
            var errorResponse = await response.Content.ReadFromJsonAsync<ErrorResponse>();
            errorResponse.Should().NotBeNull();
            errorResponse.Message.Should().Contain("file type");
        }
    }
    
    [Fact]
    public async Task SearchEndpoint_WithInjectionAttempts_ShouldBeProtected()
    {
        // Arrange
        var injectionAttempts = new[]
        {
            "'; DROP TABLE Users; --",
            "' OR '1'='1",
            "' UNION SELECT * FROM Users --",
            "'; EXEC xp_cmdshell('dir'); --",
            "<script>alert('XSS')</script>",
            "javascript:alert('XSS')"
        };
        
        foreach (var attempt in injectionAttempts)
        {
            // Act
            var response = await _client.GetAsync($"/api/orders/search?q={Uri.EscapeDataString(attempt)}");
            
            // Assert - Should not crash or expose sensitive information
            response.StatusCode.Should().BeOneOf(HttpStatusCode.OK, HttpStatusCode.BadRequest, HttpStatusCode.NotFound);
            
            if (response.IsSuccessStatusCode)
            {
                var responseContent = await response.Content.ReadAsStringAsync();
                responseContent.Should().NotContain("error in your SQL syntax");
                responseContent.Should().NotContain("System.Data.SqlClient");
                responseContent.Should().NotContain("Exception");
                responseContent.Should().NotContain("<script>");
                responseContent.Should().NotContain("javascript:");
            }
        }
    }
    
    [Fact]
    public async Task JsonPayload_WithOversizedData_ShouldBeRejected()
    {
        // Arrange
        var oversizedData = new string('A', 1024 * 1024); // 1MB string
        
        var orderRequest = new CreateOrderRequest
        {
            CustomerId = Guid.NewGuid(),
            Notes = oversizedData,
            Items = new List<OrderItem>
            {
                new OrderItem
                {
                    ProductId = Guid.NewGuid(),
                    Quantity = 1,
                    Price = 100,
                    Notes = oversizedData
                }
            }
        };
        
        // Act
        var response = await _client.PostAsJsonAsync("/api/orders", orderRequest);
        
        // Assert - Should reject oversized payloads
        response.StatusCode.Should().Be(HttpStatusCode.RequestEntityTooLarge);
    }
    
    [Fact]
    public async Task Headers_WithMaliciousValues_ShouldBeSanitized()
    {
        // Arrange
        var maliciousHeaders = new[]
        {
            ("X-Forwarded-For", "127.0.0.1, 192.168.1.1, 10.0.0.1"),
            ("User-Agent", "<script>alert('XSS')</script>"),
            ("Referer", "javascript:alert('XSS')"),
            ("X-Custom-Header", "'; DROP TABLE Users; --")
        };
        
        foreach (var (headerName, headerValue) in maliciousHeaders)
        {
            _client.DefaultRequestHeaders.Add(headerName, headerValue);
            
            // Act
            var response = await _client.GetAsync("/api/orders");
            
            // Assert - Should handle malicious headers gracefully
            response.StatusCode.Should().BeOneOf(HttpStatusCode.OK, HttpStatusCode.Unauthorized, HttpStatusCode.BadRequest);
            
            // Clean up headers for next iteration
            _client.DefaultRequestHeaders.Remove(headerName);
        }
    }
}

public class OrderResponse
{
    public Guid Id { get; set; }
    public string Notes { get; set; }
    public List<OrderItemResponse> Items { get; set; } = new();
    public DateTime CreatedAt { get; set; }
}

public class OrderItemResponse
{
    public Guid ProductId { get; set; }
    public int Quantity { get; set; }
    public decimal Price { get; set; }
    public string Notes { get; set; }
}
```

## Mülakat Soruları

### Temel Sorular

1. **Security Testing nedir?**
   - **Cevap**: Güvenlik açıklarını tespit eden ve güvenlik önlemlerini değerlendiren testing.

2. **OWASP nedir?**
   - **Cevap**: Web application security için open source community ve standard.

3. **Penetration Testing nedir?**
   - **Cevap**: Güvenlik açıklarını aktif olarak test etme ve exploit etme.

4. **Vulnerability Assessment nedir?**
   - **Cevap**: Güvenlik açıklarını sistematik olarak tespit etme ve değerlendirme.

5. **Security testing ne zaman kullanılır?**
   - **Cevap**: Security compliance, vulnerability detection, risk assessment.

### Teknik Sorular

1. **SQL Injection nasıl test edilir?**
   - **Cevap**: Malicious SQL queries, parameter manipulation, error analysis.

2. **XSS nasıl test edilir?**
   - **Cevap**: Script injection, HTML encoding, Content Security Policy.

3. **Authentication bypass nasıl test edilir?**
   - **Cevap**: Weak credentials, session management, token validation.

4. **Authorization testing nasıl yapılır?**
   - **Cevap**: Role-based access, privilege escalation, resource isolation.

5. **Security testing CI/CD'de nasıl kullanılır?**
   - **Cevap**: Automated scanning, security gates, vulnerability management.

## Best Practices

1. **Test Strategy**
   - Comprehensive security coverage sağlayın
   - OWASP guidelines follow edin
   - Regular security assessments yapın
   - Risk-based testing implement edin

2. **Test Implementation**
   - Realistic attack scenarios tasarlayın
   - Automated security tools kullanın
   - Manual testing combine edin
   - Continuous monitoring implement edin

3. **Vulnerability Management**
   - Security findings track edin
   - Risk assessment yapın
   - Remediation planning implement edin
   - Security metrics collect edin

4. **Compliance & Reporting**
   - Security standards follow edin
   - Detailed security reports oluşturun
   - Compliance documentation maintain edin
   - Stakeholder communication sağlayın

5. **Continuous Improvement**
   - Security testing evolve edin
   - New threats monitor edin
   - Security tools update edin
   - Team training provide edin

## Kaynaklar

- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [Security Testing](https://docs.microsoft.com/en-us/azure/security/develop/secure-code-overview)
- [Penetration Testing](https://docs.microsoft.com/en-us/azure/security/fundamentals/pen-testing)
- [Security Best Practices](https://docs.microsoft.com/en-us/azure/security/fundamentals/)
- [.NET Security](https://docs.microsoft.com/en-us/dotnet/standard/security/)
