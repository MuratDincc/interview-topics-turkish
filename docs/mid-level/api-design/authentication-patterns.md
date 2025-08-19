# API Authentication Patterns

## Giriş

API Authentication Patterns, API'lerin güvenliğini sağlamak için kullanılan kimlik doğrulama yöntemleridir. Mid-level geliştiriciler için bu pattern'leri anlamak ve implement etmek, güvenli API tasarımında kritik öneme sahiptir.

## Authentication Pattern Türleri

### 1. API Key Authentication
En basit authentication yöntemi, genellikle public API'lerde kullanılır.

```csharp
public class ApiKeyAuthenticationHandler : AuthenticationHandler<AuthenticationSchemeOptions>
{
    private const string ApiKeyHeaderName = "X-API-Key";
    private readonly IApiKeyService _apiKeyService;
    
    public ApiKeyAuthenticationHandler(
        IOptionsMonitor<AuthenticationSchemeOptions> options,
        ILoggerFactory logger,
        UrlEncoder encoder,
        ISystemClock clock,
        IApiKeyService apiKeyService)
        : base(options, logger, encoder, clock)
    {
        _apiKeyService = apiKeyService;
    }
    
    protected override async Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue(ApiKeyHeaderName, out var apiKeyHeaderValues))
        {
            return AuthenticateResult.Fail("API Key header not found.");
        }
        
        var providedApiKey = apiKeyHeaderValues.FirstOrDefault();
        if (string.IsNullOrWhiteSpace(providedApiKey))
        {
            return AuthenticateResult.Fail("API Key is empty.");
        }
        
        var apiKey = await _apiKeyService.GetApiKeyAsync(providedApiKey);
        if (apiKey == null)
        {
            return AuthenticateResult.Fail("Invalid API Key.");
        }
        
        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, apiKey.UserId.ToString()),
            new Claim(ClaimTypes.Name, apiKey.UserName),
            new Claim("ApiKeyId", apiKey.Id.ToString()),
            new Claim("Permissions", string.Join(",", apiKey.Permissions))
        };
        
        var identity = new ClaimsIdentity(claims, Scheme.Name);
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, Scheme.Name);
        
        return AuthenticateResult.Success(ticket);
    }
}

// API Key Service
public interface IApiKeyService
{
    Task<ApiKey> GetApiKeyAsync(string key);
    Task<ApiKey> CreateApiKeyAsync(string userId, IEnumerable<string> permissions);
    Task<bool> ValidateApiKeyAsync(string key);
}

public class ApiKeyService : IApiKeyService
{
    private readonly IDbConnection _connection;
    private readonly IMemoryCache _cache;
    
    public async Task<ApiKey> GetApiKeyAsync(string key)
    {
        var cacheKey = $"apikey:{key}";
        
        if (_cache.TryGetValue(cacheKey, out ApiKey cachedKey))
        {
            return cachedKey;
        }
        
        var sql = @"
            SELECT Id, UserId, UserName, KeyHash, Permissions, IsActive, ExpiresAt
            FROM ApiKeys 
            WHERE KeyHash = @KeyHash AND IsActive = 1 AND (ExpiresAt IS NULL OR ExpiresAt > @Now)";
            
        var apiKey = await _connection.QueryFirstOrDefaultAsync<ApiKey>(sql, 
            new { KeyHash = HashKey(key), Now = DateTime.UtcNow });
            
        if (apiKey != null)
        {
            _cache.Set(cacheKey, apiKey, TimeSpan.FromMinutes(5));
        }
        
        return apiKey;
    }
    
    private string HashKey(string key)
    {
        using var sha256 = SHA256.Create();
        var hashBytes = sha256.ComputeHash(Encoding.UTF8.GetBytes(key));
        return Convert.ToBase64String(hashBytes);
    }
}
```

### 2. JWT (JSON Web Token) Authentication
Stateless authentication için kullanılan modern yöntem.

```csharp
public class JwtAuthenticationHandler : AuthenticationHandler<AuthenticationSchemeOptions>
{
    private readonly IJwtService _jwtService;
    
    public JwtAuthenticationHandler(
        IOptionsMonitor<AuthenticationSchemeOptions> options,
        ILoggerFactory logger,
        UrlEncoder encoder,
        ISystemClock clock,
        IJwtService jwtService)
        : base(options, logger, encoder, clock)
    {
        _jwtService = jwtService;
    }
    
    protected override async Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        var token = GetTokenFromRequest();
        if (string.IsNullOrEmpty(token))
        {
            return AuthenticateResult.Fail("JWT token not found.");
        }
        
        try
        {
            var principal = await _jwtService.ValidateTokenAsync(token);
            var ticket = new AuthenticationTicket(principal, Scheme.Name);
            return AuthenticateResult.Success(ticket);
        }
        catch (Exception ex)
        {
            return AuthenticateResult.Fail($"JWT validation failed: {ex.Message}");
        }
    }
    
    private string GetTokenFromRequest()
    {
        // Authorization header'dan Bearer token al
        if (Request.Headers.TryGetValue("Authorization", out var authHeader))
        {
            var authValue = authHeader.FirstOrDefault();
            if (authValue?.StartsWith("Bearer ") == true)
            {
                return authValue.Substring("Bearer ".Length);
            }
        }
        
        return null;
    }
}

// JWT Service
public interface IJwtService
{
    Task<string> GenerateTokenAsync(User user);
    Task<ClaimsPrincipal> ValidateTokenAsync(string token);
    Task<string> RefreshTokenAsync(string refreshToken);
}

public class JwtService : IJwtService
{
    private readonly IConfiguration _configuration;
    private readonly IUserService _userService;
    
    public async Task<string> GenerateTokenAsync(User user)
    {
        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
            new Claim(ClaimTypes.Name, user.UserName),
            new Claim(ClaimTypes.Email, user.Email),
            new Claim(ClaimTypes.Role, user.Role),
            new Claim("UserId", user.Id.ToString()),
            new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new Claim(JwtRegisteredClaimNames.Iat, DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString())
        };
        
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_configuration["Jwt:Key"]));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        
        var token = new JwtSecurityToken(
            issuer: _configuration["Jwt:Issuer"],
            audience: _configuration["Jwt:Audience"],
            claims: claims,
            expires: DateTime.UtcNow.AddHours(1),
            signingCredentials: creds
        );
        
        return new JwtSecurityTokenHandler().WriteToken(token);
    }
    
    public async Task<ClaimsPrincipal> ValidateTokenAsync(string token)
    {
        var tokenHandler = new JwtSecurityTokenHandler();
        var key = Encoding.UTF8.GetBytes(_configuration["Jwt:Key"]);
        
        var validationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(key),
            ValidateIssuer = true,
            ValidIssuer = _configuration["Jwt:Issuer"],
            ValidateAudience = true,
            ValidAudience = _configuration["Jwt:Audience"],
            ValidateLifetime = true,
            ClockSkew = TimeSpan.Zero
        };
        
        var principal = tokenHandler.ValidateToken(token, validationParameters, out var validatedToken);
        
        // Token blacklist kontrolü
        if (await IsTokenBlacklistedAsync(token))
        {
            throw new SecurityTokenException("Token is blacklisted.");
        }
        
        return principal;
    }
    
    private async Task<bool> IsTokenBlacklistedAsync(string token)
    {
        // Redis veya database'de blacklist kontrolü
        return false; // Implementation
    }
}
```

### 3. OAuth 2.0 Authentication
Third-party authentication için kullanılan standart protokol.

```csharp
public class OAuth2Service : IOAuth2Service
{
    private readonly IConfiguration _configuration;
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly IUserService _userService;
    
    public async Task<OAuth2TokenResponse> ExchangeCodeForTokenAsync(string authorizationCode, string redirectUri)
    {
        var client = _httpClientFactory.CreateClient("OAuth2");
        
        var tokenRequest = new FormUrlEncodedContent(new[]
        {
            new KeyValuePair<string, string>("grant_type", "authorization_code"),
            new KeyValuePair<string, string>("code", authorizationCode),
            new KeyValuePair<string, string>("redirect_uri", redirectUri),
            new KeyValuePair<string, string>("client_id", _configuration["OAuth2:ClientId"]),
            new KeyValuePair<string, string>("client_secret", _configuration["OAuth2:ClientSecret"])
        });
        
        var response = await client.PostAsync("/oauth/token", tokenRequest);
        var responseContent = await response.Content.ReadAsStringAsync();
        
        if (!response.IsSuccessStatusCode)
        {
            throw new OAuth2Exception($"Token exchange failed: {responseContent}");
        }
        
        return JsonSerializer.Deserialize<OAuth2TokenResponse>(responseContent);
    }
    
    public async Task<UserInfo> GetUserInfoAsync(string accessToken)
    {
        var client = _httpClientFactory.CreateClient("OAuth2");
        client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", accessToken);
        
        var response = await client.GetAsync("/oauth/userinfo");
        var responseContent = await response.Content.ReadAsStringAsync();
        
        if (!response.IsSuccessStatusCode)
        {
            throw new OAuth2Exception($"User info retrieval failed: {responseContent}");
        }
        
        return JsonSerializer.Deserialize<UserInfo>(responseContent);
    }
}

// OAuth2 Controller
[ApiController]
[Route("api/[controller]")]
public class OAuth2Controller : ControllerBase
{
    private readonly IOAuth2Service _oauth2Service;
    private readonly IJwtService _jwtService;
    private readonly IUserService _userService;
    
    [HttpGet("authorize")]
    public IActionResult Authorize(string responseType, string clientId, string redirectUri, string scope, string state)
    {
        // OAuth2 authorization endpoint
        var authorizationUrl = $"{_configuration["OAuth2:AuthorizationUrl"]}?" +
            $"response_type={responseType}&" +
            $"client_id={clientId}&" +
            $"redirect_uri={Uri.EscapeDataString(redirectUri)}&" +
            $"scope={scope}&" +
            $"state={state}";
            
        return Redirect(authorizationUrl);
    }
    
    [HttpPost("callback")]
    public async Task<IActionResult> Callback(string code, string state)
    {
        try
        {
            // Authorization code'u access token ile değiştir
            var tokenResponse = await _oauth2Service.ExchangeCodeForTokenAsync(code, _configuration["OAuth2:RedirectUri"]);
            
            // User bilgilerini al
            var userInfo = await _oauth2Service.GetUserInfoAsync(tokenResponse.AccessToken);
            
            // User'ı sistemde bul veya oluştur
            var user = await _userService.GetOrCreateUserAsync(userInfo);
            
            // JWT token oluştur
            var jwtToken = await _jwtService.GenerateTokenAsync(user);
            
            return Ok(new
            {
                access_token = jwtToken,
                token_type = "Bearer",
                expires_in = 3600,
                user = user
            });
        }
        catch (Exception ex)
        {
            return BadRequest(new { error = "OAuth2 callback failed", message = ex.Message });
        }
    }
}
```

### 4. Multi-Factor Authentication (MFA)
Ek güvenlik katmanı için kullanılan authentication yöntemi.

```csharp
public class MfaService : IMfaService
{
    private readonly IUserService _userService;
    private readonly IEmailService _emailService;
    private readonly ISmsService _smsService;
    private readonly IMemoryCache _cache;
    
    public async Task<bool> SendMfaCodeAsync(string userId, string method)
    {
        var user = await _userService.GetByIdAsync(userId);
        if (user == null)
            return false;
        
        var mfaCode = GenerateMfaCode();
        var cacheKey = $"mfa:{userId}:{method}";
        
        // MFA code'u cache'e kaydet (5 dakika geçerli)
        _cache.Set(cacheKey, mfaCode, TimeSpan.FromMinutes(5));
        
        switch (method.ToLower())
        {
            case "email":
                await _emailService.SendMfaCodeAsync(user.Email, mfaCode);
                break;
            case "sms":
                await _smsService.SendMfaCodeAsync(user.PhoneNumber, mfaCode);
                break;
            default:
                return false;
        }
        
        return true;
    }
    
    public async Task<bool> ValidateMfaCodeAsync(string userId, string method, string code)
    {
        var cacheKey = $"mfa:{userId}:{method}";
        
        if (!_cache.TryGetValue(cacheKey, out string cachedCode))
        {
            return false; // Code expired or not found
        }
        
        if (cachedCode == code)
        {
            _cache.Remove(cacheKey); // Code used, remove from cache
            return true;
        }
        
        return false;
    }
    
    private string GenerateMfaCode()
    {
        var random = new Random();
        return random.Next(100000, 999999).ToString();
    }
}

// MFA Controller
[ApiController]
[Route("api/[controller]")]
public class MfaController : ControllerBase
{
    private readonly IMfaService _mfaService;
    
    [HttpPost("send")]
    public async Task<IActionResult> SendMfaCode([FromBody] SendMfaRequest request)
    {
        var success = await _mfaService.SendMfaCodeAsync(request.UserId, request.Method);
        
        if (success)
        {
            return Ok(new { message = "MFA code sent successfully" });
        }
        
        return BadRequest(new { error = "Failed to send MFA code" });
    }
    
    [HttpPost("validate")]
    public async Task<IActionResult> ValidateMfaCode([FromBody] ValidateMfaRequest request)
    {
        var isValid = await _mfaService.ValidateMfaCodeAsync(request.UserId, request.Method, request.Code);
        
        if (isValid)
        {
            return Ok(new { message = "MFA code validated successfully" });
        }
        
        return BadRequest(new { error = "Invalid MFA code" });
    }
}
```

## Authentication Middleware Configuration

### Program.cs'de Authentication Setup
```csharp
var builder = WebApplication.CreateBuilder(args);

// Authentication services
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidIssuer = builder.Configuration["Jwt:Issuer"],
        ValidateAudience = true,
        ValidAudience = builder.Configuration["Jwt:Audience"],
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]))
    };
    
    options.Events = new JwtBearerEvents
    {
        OnAuthenticationFailed = context =>
        {
            context.Response.StatusCode = 401;
            context.Response.ContentType = "application/json";
            var result = JsonSerializer.Serialize(new { error = "Authentication failed" });
            return context.Response.WriteAsync(result);
        },
        OnChallenge = context =>
        {
            context.HandleResponse();
            context.Response.StatusCode = 401;
            context.Response.ContentType = "application/json";
            var result = JsonSerializer.Serialize(new { error = "Authentication required" });
            return context.Response.WriteAsync(result);
        }
    };
})
.AddApiKey(options => { }) // Custom API Key authentication
.AddOAuth2(options => { }); // OAuth2 authentication

// Authorization
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("RequireAdminRole", policy =>
        policy.RequireRole("Admin"));
    
    options.AddPolicy("RequirePremiumUser", policy =>
        policy.RequireClaim("UserType", "Premium"));
    
    options.AddPolicy("RequireMfa", policy =>
        policy.RequireClaim("MfaVerified", "true"));
});

var app = builder.Build();

// Authentication middleware
app.UseAuthentication();
app.UseAuthorization();

app.Run();
```

## Mülakat Soruları

### Temel Sorular

1. **API Key authentication nedir ve ne zaman kullanılır?**
   - **Cevap**: Basit authentication yöntemi, public API'lerde kullanılır. Client'ın API key ile kimlik doğrulaması yapması.

2. **JWT nedir ve avantajları nelerdir?**
   - **Cevap**: JSON Web Token, stateless authentication sağlar. Stateless, scalable, self-contained özellikleri vardır.

3. **OAuth 2.0 nedir ve flow'ları nelerdir?**
   - **Cevap**: Authorization framework, Authorization Code, Client Credentials, Resource Owner Password, Implicit flow'ları vardır.

4. **Multi-Factor Authentication nedir?**
   - **Cevap**: Birden fazla authentication yöntemi kullanarak güvenliği artırma. SMS, email, TOTP gibi yöntemler.

5. **Stateless vs Stateful authentication arasındaki fark nedir?**
   - **Cevap**: Stateless server'da session bilgisi saklanmaz (JWT), stateful server'da session bilgisi saklanır.

### Teknik Sorular

1. **JWT token'ların güvenlik riskleri nelerdir?**
   - **Cevap**: Token hijacking, XSS attacks, token expiration, secure storage requirements. HTTPS ve proper validation gerekli.

2. **OAuth 2.0'da refresh token nasıl kullanılır?**
   - **Cevap**: Access token expire olduğunda refresh token ile yeni access token alınır. Refresh token daha uzun süreli.

3. **API authentication'da rate limiting nasıl implement edilir?**
   - **Cevap**: Client identification (API key, JWT), request counting, window-based limits, distributed rate limiting.

4. **JWT token blacklisting nasıl yapılır?**
   - **Cevap**: Redis veya database'de blacklisted token'lar saklanır, token validation sırasında kontrol edilir.

5. **Multi-tenant sistemde authentication nasıl yapılır?**
   - **Cevap**: Tenant-specific claims, tenant isolation, separate authentication services, tenant-aware token validation.

## Best Practices

1. **Security**
   - HTTPS kullanın
   - Token expiration ayarlayın
   - Secure storage implement edin
   - Input validation yapın

2. **Performance**
   - Token caching kullanın
   - Efficient validation implement edin
   - Background token cleanup yapın
   - Monitoring ekleyin

3. **User Experience**
   - Clear error messages verin
   - Graceful degradation sağlayın
   - MFA options sunun
   - Remember me functionality ekleyin

4. **Monitoring**
   - Authentication metrics toplayın
   - Failed attempts izleyin
   - Token usage analiz edin
   - Security alerts kurun

## Kaynaklar

- [ASP.NET Core Authentication](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/)
- [JWT.io](https://jwt.io/)
- [OAuth 2.0 Specification](https://tools.ietf.org/html/rfc6749)
- [OpenID Connect](https://openid.net/connect/)
- [Multi-Factor Authentication](https://docs.microsoft.com/en-us/azure/active-directory/authentication/concept-mfa-howitworks)
