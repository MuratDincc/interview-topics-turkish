# JWT Implementation

## Giriş

JSON Web Token (JWT), taraflar arasında güvenli bilgi aktarımı için kullanılan açık standarttır. Mid-level geliştiriciler için JWT implementation'ını anlamak, stateless authentication ve authorization sistemleri tasarlamada kritiktir.

## JWT Structure

### 1. JWT Components
JWT'nin üç ana bileşeni: Header, Payload ve Signature.

```csharp
public class JwtToken
{
    public string Header { get; set; }
    public string Payload { get; set; }
    public string Signature { get; set; }
    
    public string ToTokenString()
    {
        return $"{Header}.{Payload}.{Signature}";
    }
    
    public static JwtToken FromTokenString(string token)
    {
        var parts = token.Split('.');
        if (parts.Length != 3)
        {
            throw new ArgumentException("Invalid JWT token format");
        }
        
        return new JwtToken
        {
            Header = parts[0],
            Payload = parts[1],
            Signature = parts[2]
        };
    }
}

public class JwtHeader
{
    [JsonPropertyName("alg")]
    public string Algorithm { get; set; } = "HS256";
    
    [JsonPropertyName("typ")]
    public string Type { get; set; } = "JWT";
    
    [JsonPropertyName("kid")]
    public string KeyId { get; set; }
}

public class JwtPayload
{
    [JsonPropertyName("iss")]
    public string Issuer { get; set; }
    
    [JsonPropertyName("sub")]
    public string Subject { get; set; }
    
    [JsonPropertyName("aud")]
    public string Audience { get; set; }
    
    [JsonPropertyName("exp")]
    public long ExpirationTime { get; set; }
    
    [JsonPropertyName("nbf")]
    public long NotBefore { get; set; }
    
    [JsonPropertyName("iat")]
    public long IssuedAt { get; set; }
    
    [JsonPropertyName("jti")]
    public string JwtId { get; set; }
    
    // Custom claims
    [JsonPropertyName("user_id")]
    public string UserId { get; set; }
    
    [JsonPropertyName("username")]
    public string Username { get; set; }
    
    [JsonPropertyName("email")]
    public string Email { get; set; }
    
    [JsonPropertyName("roles")]
    public List<string> Roles { get; set; } = new();
    
    [JsonPropertyName("permissions")]
    public List<string> Permissions { get; set; } = new();
    
    public DateTime ExpirationTimeUtc => DateTimeOffset.FromUnixTimeSeconds(ExpirationTime).UtcDateTime;
    public DateTime NotBeforeUtc => DateTimeOffset.FromUnixTimeSeconds(NotBefore).UtcDateTime;
    public DateTime IssuedAtUtc => DateTimeOffset.FromUnixTimeSeconds(IssuedAt).UtcDateTime;
}
```

### 2. JWT Token Builder
JWT token oluşturma ve yönetme.

```csharp
public interface IJwtTokenBuilder
{
    string BuildToken(JwtPayload payload);
    JwtPayload ValidateToken(string token);
    string RefreshToken(string token);
}

public class JwtTokenBuilder : IJwtTokenBuilder
{
    private readonly JwtSettings _settings;
    private readonly ILogger<JwtTokenBuilder> _logger;
    private readonly byte[] _secretKey;
    
    public JwtTokenBuilder(JwtSettings settings, ILogger<JwtTokenBuilder> logger)
    {
        _settings = settings;
        _logger = logger;
        _secretKey = Encoding.UTF8.GetBytes(settings.SecretKey);
    }
    
    public string BuildToken(JwtPayload payload)
    {
        try
        {
            // Set standard claims if not provided
            var now = DateTime.UtcNow;
            if (payload.IssuedAt == 0)
            {
                payload.IssuedAt = new DateTimeOffset(now).ToUnixTimeSeconds();
            }
            
            if (payload.NotBefore == 0)
            {
                payload.NotBefore = new DateTimeOffset(now).ToUnixTimeSeconds();
            }
            
            if (payload.ExpirationTime == 0)
            {
                payload.ExpirationTime = new DateTimeOffset(now.AddMinutes(_settings.ExpirationMinutes)).ToUnixTimeSeconds();
            }
            
            if (string.IsNullOrEmpty(payload.Issuer))
            {
                payload.Issuer = _settings.Issuer;
            }
            
            if (string.IsNullOrEmpty(payload.Audience))
            {
                payload.Audience = _settings.Audience;
            }
            
            if (string.IsNullOrEmpty(payload.JwtId))
            {
                payload.JwtId = Guid.NewGuid().ToString();
            }
            
            // Create header
            var header = new JwtHeader
            {
                Algorithm = _settings.Algorithm,
                Type = "JWT",
                KeyId = _settings.KeyId
            };
            
            // Serialize header and payload
            var headerJson = JsonSerializer.Serialize(header);
            var payloadJson = JsonSerializer.Serialize(payload);
            
            var headerBase64 = Base64UrlEncode(headerJson);
            var payloadBase64 = Base64UrlEncode(payloadJson);
            
            // Create signature
            var signatureInput = $"{headerBase64}.{payloadBase64}";
            var signature = CreateSignature(signatureInput);
            
            // Build token
            var token = new JwtToken
            {
                Header = headerBase64,
                Payload = payloadBase64,
                Signature = signature
            };
            
            _logger.LogInformation("JWT token created for user {UserId}", payload.UserId);
            return token.ToTokenString();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error building JWT token");
            throw;
        }
    }
    
    public JwtPayload ValidateToken(string token)
    {
        try
        {
            var jwtToken = JwtToken.FromTokenString(token);
            
            // Verify signature
            var signatureInput = $"{jwtToken.Header}.{jwtToken.Payload}";
            var expectedSignature = CreateSignature(signatureInput);
            
            if (jwtToken.Signature != expectedSignature)
            {
                throw new SecurityTokenInvalidSignatureException("Invalid JWT signature");
            }
            
            // Decode payload
            var payloadJson = Base64UrlDecode(jwtToken.Payload);
            var payload = JsonSerializer.Deserialize<JwtPayload>(payloadJson);
            
            // Validate claims
            ValidateClaims(payload);
            
            _logger.LogInformation("JWT token validated for user {UserId}", payload.UserId);
            return payload;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error validating JWT token");
            throw;
        }
    }
    
    public string RefreshToken(string token)
    {
        try
        {
            var payload = ValidateToken(token);
            
            // Create new payload with extended expiration
            var newPayload = new JwtPayload
            {
                Issuer = payload.Issuer,
                Subject = payload.Subject,
                Audience = payload.Audience,
                UserId = payload.UserId,
                Username = payload.Username,
                Email = payload.Email,
                Roles = payload.Roles,
                Permissions = payload.Permissions
            };
            
            return BuildToken(newPayload);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error refreshing JWT token");
            throw;
        }
    }
    
    private string CreateSignature(string input)
    {
        using var hmac = new HMACSHA256(_secretKey);
        var hash = hmac.ComputeHash(Encoding.UTF8.GetBytes(input));
        return Base64UrlEncode(hash);
    }
    
    private void ValidateClaims(JwtPayload payload)
    {
        var now = DateTimeOffset.UtcNow.ToUnixTimeSeconds();
        
        if (payload.ExpirationTime <= now)
        {
            throw new SecurityTokenExpiredException("JWT token has expired");
        }
        
        if (payload.NotBefore > now)
        {
            throw new SecurityTokenNotYetValidException("JWT token is not yet valid");
        }
        
        if (!string.IsNullOrEmpty(_settings.Issuer) && payload.Issuer != _settings.Issuer)
        {
            throw new SecurityTokenInvalidIssuerException("Invalid JWT issuer");
        }
        
        if (!string.IsNullOrEmpty(_settings.Audience) && payload.Audience != _settings.Audience)
        {
            throw new SecurityTokenInvalidAudienceException("Invalid JWT audience");
        }
    }
    
    private static string Base64UrlEncode(string input)
    {
        var bytes = Encoding.UTF8.GetBytes(input);
        return Base64UrlEncode(bytes);
    }
    
    private static string Base64UrlEncode(byte[] input)
    {
        var base64 = Convert.ToBase64String(input);
        return base64.Replace('+', '-').Replace('/', '_').TrimEnd('=');
    }
    
    private static string Base64UrlDecode(string input)
    {
        var base64 = input.Replace('-', '+').Replace('_', '/');
        var padding = 4 - (base64.Length % 4);
        if (padding != 4)
        {
            base64 += new string('=', padding);
        }
        
        var bytes = Convert.FromBase64String(base64);
        return Encoding.UTF8.GetString(bytes);
    }
}

public class JwtSettings
{
    public string SecretKey { get; set; }
    public string Issuer { get; set; }
    public string Audience { get; set; }
    public int ExpirationMinutes { get; set; } = 60;
    public string Algorithm { get; set; } = "HS256";
    public string KeyId { get; set; }
}
```

## JWT Authentication

### 1. JWT Authentication Handler
ASP.NET Core için JWT authentication handler.

```csharp
public class JwtAuthenticationHandler : AuthenticationHandler<AuthenticationSchemeOptions>
{
    private readonly IJwtTokenBuilder _jwtTokenBuilder;
    private readonly IUserService _userService;
    
    public JwtAuthenticationHandler(
        IOptionsMonitor<AuthenticationSchemeOptions> options,
        ILoggerFactory logger,
        UrlEncoder encoder,
        ISystemClock clock,
        IJwtTokenBuilder jwtTokenBuilder,
        IUserService userService)
        : base(options, logger, encoder, clock)
    {
        _jwtTokenBuilder = jwtTokenBuilder;
        _userService = userService;
    }
    
    protected override async Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        try
        {
            // Extract token from Authorization header
            var token = ExtractTokenFromHeader();
            if (string.IsNullOrEmpty(token))
            {
                return AuthenticateResult.Fail("No JWT token provided");
            }
            
            // Validate token
            var payload = _jwtTokenBuilder.ValidateToken(token);
            
            // Get user information
            var user = await _userService.GetUserByIdAsync(payload.UserId);
            if (user == null || !user.IsActive)
            {
                return AuthenticateResult.Fail("User not found or inactive");
            }
            
            // Create claims principal
            var claims = CreateClaims(payload, user);
            var identity = new ClaimsIdentity(claims, Scheme.Name);
            var principal = new ClaimsPrincipal(identity);
            
            var ticket = new AuthenticationTicket(principal, Scheme.Name);
            return AuthenticateResult.Success(ticket);
        }
        catch (SecurityTokenExpiredException)
        {
            return AuthenticateResult.Fail("JWT token has expired");
        }
        catch (SecurityTokenInvalidSignatureException)
        {
            return AuthenticateResult.Fail("Invalid JWT signature");
        }
        catch (Exception ex)
        {
            Logger.LogError(ex, "Error during JWT authentication");
            return AuthenticateResult.Fail("Authentication failed");
        }
    }
    
    protected override Task HandleChallengeAsync(AuthenticationProperties properties)
    {
        Response.Headers["WWW-Authenticate"] = "Bearer";
        return Task.CompletedTask;
    }
    
    private string ExtractTokenFromHeader()
    {
        var authorizationHeader = Request.Headers["Authorization"].FirstOrDefault();
        
        if (string.IsNullOrEmpty(authorizationHeader) || !authorizationHeader.StartsWith("Bearer "))
        {
            return null;
        }
        
        return authorizationHeader.Substring("Bearer ".Length).Trim();
    }
    
    private IEnumerable<Claim> CreateClaims(JwtPayload payload, User user)
    {
        var claims = new List<Claim>
        {
            new Claim(ClaimTypes.NameIdentifier, payload.UserId),
            new Claim(ClaimTypes.Name, payload.Username),
            new Claim(ClaimTypes.Email, payload.Email),
            new Claim("jti", payload.JwtId),
            new Claim("iat", payload.IssuedAt.ToString()),
            new Claim("exp", payload.ExpirationTime.ToString())
        };
        
        // Add role claims
        foreach (var role in payload.Roles)
        {
            claims.Add(new Claim(ClaimTypes.Role, role));
        }
        
        // Add permission claims
        foreach (var permission in payload.Permissions)
        {
            claims.Add(new Claim("permission", permission));
        }
        
        return claims;
    }
}

public class JwtAuthenticationSchemeOptions : AuthenticationSchemeOptions
{
    public string TokenHeaderName { get; set; } = "Authorization";
    public string TokenPrefix { get; set; } = "Bearer";
    public bool RequireHttps { get; set; } = true;
}
```

### 2. JWT Authentication Service
JWT authentication işlemlerini yöneten service.

```csharp
public interface IJwtAuthenticationService
{
    Task<AuthenticationResult> AuthenticateAsync(string username, string password);
    Task<AuthenticationResult> AuthenticateWithRefreshTokenAsync(string refreshToken);
    Task<bool> RevokeTokenAsync(string token);
    Task<bool> IsTokenValidAsync(string token);
}

public class JwtAuthenticationService : IJwtAuthenticationService
{
    private readonly IUserService _userService;
    private readonly IPasswordHasher _passwordHasher;
    private readonly IJwtTokenBuilder _jwtTokenBuilder;
    private readonly IRefreshTokenService _refreshTokenService;
    private readonly ILogger<JwtAuthenticationService> _logger;
    
    public JwtAuthenticationService(
        IUserService userService,
        IPasswordHasher passwordHasher,
        IJwtTokenBuilder jwtTokenBuilder,
        IRefreshTokenService refreshTokenService,
        ILogger<JwtAuthenticationService> logger)
    {
        _userService = userService;
        _passwordHasher = passwordHasher;
        _jwtTokenBuilder = jwtTokenBuilder;
        _refreshTokenService = refreshTokenService;
        _logger = logger;
    }
    
    public async Task<AuthenticationResult> AuthenticateAsync(string username, string password)
    {
        try
        {
            // Validate input
            if (string.IsNullOrEmpty(username) || string.IsNullOrEmpty(password))
            {
                return AuthenticationResult.Failure("Username and password are required");
            }
            
            // Get user
            var user = await _userService.GetUserByUsernameAsync(username);
            if (user == null)
            {
                _logger.LogWarning("Authentication failed for username: {Username}", username);
                return AuthenticationResult.Failure("Invalid credentials");
            }
            
            // Check if user is active
            if (!user.IsActive)
            {
                _logger.LogWarning("Authentication failed for inactive user: {Username}", username);
                return AuthenticationResult.Failure("User account is inactive");
            }
            
            // Check if user is locked out
            if (user.LockoutEnd.HasValue && user.LockoutEnd.Value > DateTime.UtcNow)
            {
                _logger.LogWarning("Authentication failed for locked user: {Username}", username);
                return AuthenticationResult.Failure("User account is locked");
            }
            
            // Verify password
            var passwordResult = _passwordHasher.VerifyHashedPassword(user.PasswordHash, password);
            if (passwordResult == PasswordVerificationResult.Failed)
            {
                await HandleFailedLoginAsync(user);
                return AuthenticationResult.Failure("Invalid credentials");
            }
            
            // Reset failed login count on successful authentication
            if (user.AccessFailedCount > 0)
            {
                await _userService.ResetAccessFailedCountAsync(user.Id);
            }
            
            // Update last login time
            await _userService.UpdateLastLoginAsync(user.Id);
            
            // Create JWT token
            var payload = CreateJwtPayload(user);
            var accessToken = _jwtTokenBuilder.BuildToken(payload);
            
            // Create refresh token
            var refreshToken = await _refreshTokenService.CreateRefreshTokenAsync(user.Id);
            
            _logger.LogInformation("User {Username} authenticated successfully", username);
            
            return AuthenticationResult.Success(accessToken, refreshToken, user);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error during authentication for username: {Username}", username);
            return AuthenticationResult.Failure("Authentication failed");
        }
    }
    
    public async Task<AuthenticationResult> AuthenticateWithRefreshTokenAsync(string refreshToken)
    {
        try
        {
            // Validate refresh token
            var refreshTokenInfo = await _refreshTokenService.ValidateRefreshTokenAsync(refreshToken);
            if (refreshTokenInfo == null)
            {
                return AuthenticationResult.Failure("Invalid refresh token");
            }
            
            // Get user
            var user = await _userService.GetUserByIdAsync(refreshTokenInfo.UserId);
            if (user == null || !user.IsActive)
            {
                return AuthenticationResult.Failure("User not found or inactive");
            }
            
            // Create new JWT token
            var payload = CreateJwtPayload(user);
            var accessToken = _jwtTokenBuilder.BuildToken(payload);
            
            // Create new refresh token
            var newRefreshToken = await _refreshTokenService.CreateRefreshTokenAsync(user.Id);
            
            // Revoke old refresh token
            await _refreshTokenService.RevokeRefreshTokenAsync(refreshToken);
            
            _logger.LogInformation("User {Username} authenticated with refresh token", user.Username);
            
            return AuthenticationResult.Success(accessToken, newRefreshToken, user);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error during refresh token authentication");
            return AuthenticationResult.Failure("Authentication failed");
        }
    }
    
    public async Task<bool> RevokeTokenAsync(string token)
    {
        try
        {
            // Add token to blacklist or revoke it
            // Implementation depends on your requirements
            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error revoking token");
            return false;
        }
    }
    
    public async Task<bool> IsTokenValidAsync(string token)
    {
        try
        {
            var payload = _jwtTokenBuilder.ValidateToken(token);
            return true;
        }
        catch
        {
            return false;
        }
    }
    
    private JwtPayload CreateJwtPayload(User user)
    {
        return new JwtPayload
        {
            Subject = user.Id,
            UserId = user.Id,
            Username = user.Username,
            Email = user.Email,
            Roles = user.Roles,
            Permissions = user.Permissions,
            IssuedAt = new DateTimeOffset(DateTime.UtcNow).ToUnixTimeSeconds(),
            NotBefore = new DateTimeOffset(DateTime.UtcNow).ToUnixTimeSeconds(),
            ExpirationTime = new DateTimeOffset(DateTime.UtcNow.AddMinutes(60)).ToUnixTimeSeconds(),
            JwtId = Guid.NewGuid().ToString()
        };
    }
    
    private async Task HandleFailedLoginAsync(User user)
    {
        user.AccessFailedCount++;
        
        // Lock user account if too many failed attempts
        if (user.AccessFailedCount >= 5)
        {
            user.LockoutEnd = DateTime.UtcNow.AddMinutes(30);
            user.LockoutEnabled = true;
        }
        
        await _userService.UpdateAsync(user);
        
        _logger.LogWarning("Failed login attempt for user {Username}. Attempt {AttemptCount}", 
            user.Username, user.AccessFailedCount);
    }
}

public class AuthenticationResult
{
    public bool Succeeded { get; private set; }
    public string AccessToken { get; private set; }
    public string RefreshToken { get; private set; }
    public User User { get; private set; }
    public string ErrorMessage { get; private set; }
    
    public static AuthenticationResult Success(string accessToken, string refreshToken, User user)
    {
        return new AuthenticationResult
        {
            Succeeded = true,
            AccessToken = accessToken,
            RefreshToken = refreshToken,
            User = user
        };
    }
    
    public static AuthenticationResult Failure(string errorMessage)
    {
        return new AuthenticationResult
        {
            Succeeded = false,
            ErrorMessage = errorMessage
        };
    }
}
```

## JWT Security

### 1. JWT Security Best Practices
JWT güvenliği için best practices.

```csharp
public class JwtSecurityService
{
    private readonly IJwtTokenBuilder _jwtTokenBuilder;
    private readonly ILogger<JwtSecurityService> _logger;
    private readonly IMemoryCache _cache;
    
    public JwtSecurityService(
        IJwtTokenBuilder jwtTokenBuilder,
        ILogger<JwtSecurityService> logger,
        IMemoryCache cache)
    {
        _jwtTokenBuilder = jwtTokenBuilder;
        _logger = logger;
        _cache = cache;
    }
    
    public async Task<bool> IsTokenBlacklistedAsync(string token)
    {
        var tokenHash = ComputeTokenHash(token);
        return _cache.TryGetValue($"blacklist:{tokenHash}", out _);
    }
    
    public async Task BlacklistTokenAsync(string token, TimeSpan duration)
    {
        var tokenHash = ComputeTokenHash(token);
        var expiration = DateTime.UtcNow.Add(duration);
        
        _cache.Set($"blacklist:{tokenHash}", true, expiration);
        
        _logger.LogInformation("Token blacklisted for {Duration} minutes", duration.TotalMinutes);
    }
    
    public async Task<bool> ValidateTokenSecurityAsync(string token)
    {
        try
        {
            // Check if token is blacklisted
            if (await IsTokenBlacklistedAsync(token))
            {
                _logger.LogWarning("Attempted to use blacklisted token");
                return false;
            }
            
            // Validate token structure and signature
            var payload = _jwtTokenBuilder.ValidateToken(token);
            
            // Check token age
            var tokenAge = DateTime.UtcNow - payload.IssuedAtUtc;
            if (tokenAge > TimeSpan.FromDays(1))
            {
                _logger.LogWarning("Token is too old: {TokenAge}", tokenAge);
                return false;
            }
            
            // Check for suspicious patterns
            if (HasSuspiciousPatterns(payload))
            {
                _logger.LogWarning("Suspicious patterns detected in token for user {UserId}", payload.UserId);
                return false;
            }
            
            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error validating token security");
            return false;
        }
    }
    
    private string ComputeTokenHash(string token)
    {
        using var sha256 = SHA256.Create();
        var hash = sha256.ComputeHash(Encoding.UTF8.GetBytes(token));
        return Convert.ToBase64String(hash);
    }
    
    private bool HasSuspiciousPatterns(JwtPayload payload)
    {
        // Check for unusual claim values
        if (payload.Roles.Contains("admin") && payload.IssuedAtUtc < DateTime.UtcNow.AddHours(-1))
        {
            return true;
        }
        
        // Check for multiple tokens with same user in short time
        var cacheKey = $"user_tokens:{payload.UserId}";
        var tokenCount = _cache.GetOrCreate(cacheKey, entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
            return 0;
        });
        
        if (tokenCount > 10)
        {
            return true;
        }
        
        _cache.Set(cacheKey, tokenCount + 1, TimeSpan.FromMinutes(5));
        
        return false;
    }
}

public class JwtRateLimitingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IMemoryCache _cache;
    private readonly ILogger<JwtRateLimitingMiddleware> _logger;
    
    public JwtRateLimitingMiddleware(
        RequestDelegate next,
        IMemoryCache cache,
        ILogger<JwtRateLimitingMiddleware> logger)
    {
        _next = next;
        _cache = cache;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var token = ExtractTokenFromContext(context);
        
        if (!string.IsNullOrEmpty(token))
        {
            var userId = ExtractUserIdFromToken(token);
            
            if (!string.IsNullOrEmpty(userId))
            {
                var cacheKey = $"rate_limit:{userId}";
                var requestCount = _cache.GetOrCreate(cacheKey, entry =>
                {
                    entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(1);
                    return 0;
                });
                
                if (requestCount > 100) // 100 requests per minute
                {
                    _logger.LogWarning("Rate limit exceeded for user {UserId}", userId);
                    context.Response.StatusCode = 429; // Too Many Requests
                    await context.Response.WriteAsync("Rate limit exceeded");
                    return;
                }
                
                _cache.Set(cacheKey, requestCount + 1, TimeSpan.FromMinutes(1));
            }
        }
        
        await _next(context);
    }
    
    private string ExtractTokenFromContext(HttpContext context)
    {
        var authorizationHeader = context.Request.Headers["Authorization"].FirstOrDefault();
        
        if (string.IsNullOrEmpty(authorizationHeader) || !authorizationHeader.StartsWith("Bearer "))
        {
            return null;
        }
        
        return authorizationHeader.Substring("Bearer ".Length).Trim();
    }
    
    private string ExtractUserIdFromToken(string token)
    {
        try
        {
            var jwtToken = JwtToken.FromTokenString(token);
            var payloadJson = Base64UrlDecode(jwtToken.Payload);
            var payload = JsonSerializer.Deserialize<JwtPayload>(payloadJson);
            return payload.UserId;
        }
        catch
        {
            return null;
        }
    }
    
    private static string Base64UrlDecode(string input)
    {
        var base64 = input.Replace('-', '+').Replace('_', '/');
        var padding = 4 - (base64.Length % 4);
        if (padding != 4)
        {
            base64 += new string('=', padding);
        }
        
        var bytes = Convert.FromBase64String(base64);
        return Encoding.UTF8.GetString(bytes);
    }
}
```

## Mülakat Soruları

### Temel Sorular

1. **JWT nedir ve nasıl çalışır?**
   - **Cevap**: JSON Web Token, stateless authentication için kullanılan standart. Header, Payload ve Signature'dan oluşur.

2. **JWT'nin avantajları ve dezavantajları nelerdir?**
   - **Cevap**: Avantajlar: Stateless, scalable, cross-domain. Dezavantajlar: Size, revocation difficulty, security concerns.

3. **JWT signature nasıl çalışır?**
   - **Cevap**: HMAC veya RSA ile header ve payload hash'lenir. Veri bütünlüğü ve authenticity sağlar.

4. **JWT token'ları nasıl güvenli hale getirilir?**
   - **Cevap**: Strong secret keys, HTTPS, short expiration, token blacklisting, rate limiting.

5. **JWT vs Session-based authentication arasındaki fark nedir?**
   - **Cevap**: JWT stateless, session stateful. JWT daha scalable ama daha büyük, session daha güvenli ama server memory kullanır.

### Teknik Sorular

1. **JWT token'ı nasıl revoke edilir?**
   - **Cevap**: Token blacklisting, refresh token rotation, short expiration times, server-side validation.

2. **JWT payload'da hangi bilgiler saklanmalı?**
   - **Cevap**: User ID, roles, permissions, expiration, issued at. Sensitive data saklanmamalı.

3. **JWT security vulnerabilities nasıl önlenir?**
   - **Cevap**: Strong algorithms, secure storage, HTTPS, input validation, rate limiting.

4. **JWT performance nasıl optimize edilir?**
   - **Cevap**: Minimal payload size, caching, async validation, efficient algorithms.

5. **JWT refresh token pattern nasıl implement edilir?**
   - **Cevap**: Access token + refresh token, rotation, secure storage, validation.

## Best Practices

1. **Security**
   - Strong secret keys kullanın
   - HTTPS zorunlu yapın
   - Short expiration times belirleyin
   - Token blacklisting implement edin

2. **Performance**
   - Minimal payload size sağlayın
   - Efficient algorithms kullanın
   - Caching implement edin
   - Async operations kullanın

3. **Monitoring**
   - Token usage izleyin
   - Failed validations log edin
   - Rate limiting metrics toplayın
   - Security alerts kurun

4. **Maintenance**
   - Regular key rotation yapın
   - Security updates takip edin
   - Performance monitoring yapın
   - Documentation güncelleyin

## Kaynaklar

- [JWT.io](https://jwt.io/)
- [ASP.NET Core JWT Authentication](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/jwt-authn)
- [JWT Security Best Practices](https://auth0.com/blog/a-look-at-the-latest-draft-for-jwt-bcp/)
- [JWT Implementation Guide](https://docs.microsoft.com/en-us/azure/active-directory/develop/access-tokens)
- [JWT Security Considerations](https://tools.ietf.org/html/rfc7519#section-10)
