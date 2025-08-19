# OAuth 2.0 Integration

## Giriş

OAuth 2.0, üçüncü taraf uygulamalara kullanıcı verilerine erişim izni vermek için kullanılan açık standarttır. Mid-level geliştiriciler için OAuth 2.0 implementation'ını anlamak, güvenli ve ölçeklenebilir authorization sistemleri tasarlamada kritiktir.

## OAuth 2.0 Flow Types

### 1. Authorization Code Flow
En güvenli OAuth 2.0 flow'u, server-side uygulamalar için.

```csharp
public class OAuth2AuthorizationCodeFlow
{
    private readonly IConfiguration _configuration;
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly ILogger<OAuth2AuthorizationCodeFlow> _logger;
    
    public OAuth2AuthorizationCodeFlow(
        IConfiguration configuration,
        IHttpClientFactory httpClientFactory,
        ILogger<OAuth2AuthorizationCodeFlow> logger)
    {
        _configuration = configuration;
        _httpClientFactory = httpClientFactory;
        _logger = logger;
    }
    
    public string GetAuthorizationUrl(string state, string scope = "read")
    {
        var clientId = _configuration["OAuth2:ClientId"];
        var redirectUri = _configuration["OAuth2:RedirectUri"];
        var authorizationEndpoint = _configuration["OAuth2:AuthorizationEndpoint"];
        
        var queryParams = new Dictionary<string, string>
        {
            ["response_type"] = "code",
            ["client_id"] = clientId,
            ["redirect_uri"] = redirectUri,
            ["scope"] = scope,
            ["state"] = state
        };
        
        var queryString = string.Join("&", queryParams.Select(kvp => $"{kvp.Key}={Uri.EscapeDataString(kvp.Value)}"));
        return $"{authorizationEndpoint}?{queryString}";
    }
    
    public async Task<OAuth2TokenResponse> ExchangeCodeForTokenAsync(string authorizationCode)
    {
        try
        {
            var client = _httpClientFactory.CreateClient("OAuth2");
            
            var tokenRequest = new FormUrlEncodedContent(new[]
            {
                new KeyValuePair<string, string>("grant_type", "authorization_code"),
                new KeyValuePair<string, string>("code", authorizationCode),
                new KeyValuePair<string, string>("client_id", _configuration["OAuth2:ClientId"]),
                new KeyValuePair<string, string>("client_secret", _configuration["OAuth2:ClientSecret"]),
                new KeyValuePair<string, string>("redirect_uri", _configuration["OAuth2:RedirectUri"])
            });
            
            var response = await client.PostAsync(_configuration["OAuth2:TokenEndpoint"], tokenRequest);
            
            if (response.IsSuccessStatusCode)
            {
                var responseContent = await response.Content.ReadAsStringAsync();
                var tokenResponse = JsonSerializer.Deserialize<OAuth2TokenResponse>(responseContent);
                
                _logger.LogInformation("Successfully exchanged authorization code for token");
                return tokenResponse;
            }
            else
            {
                var errorContent = await response.Content.ReadAsStringAsync();
                _logger.LogError("Failed to exchange authorization code. Status: {Status}, Content: {Content}", 
                    response.StatusCode, errorContent);
                throw new OAuth2Exception($"Token exchange failed: {response.StatusCode}");
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error exchanging authorization code for token");
            throw;
        }
    }
    
    public async Task<OAuth2TokenResponse> RefreshTokenAsync(string refreshToken)
    {
        try
        {
            var client = _httpClientFactory.CreateClient("OAuth2");
            
            var tokenRequest = new FormUrlEncodedContent(new[]
            {
                new KeyValuePair<string, string>("grant_type", "refresh_token"),
                new KeyValuePair<string, string>("refresh_token", refreshToken),
                new KeyValuePair<string, string>("client_id", _configuration["OAuth2:ClientId"]),
                new KeyValuePair<string, string>("client_secret", _configuration["OAuth2:ClientSecret"])
            });
            
            var response = await client.PostAsync(_configuration["OAuth2:TokenEndpoint"], tokenRequest);
            
            if (response.IsSuccessStatusCode)
            {
                var responseContent = await response.Content.ReadAsStringAsync();
                var tokenResponse = JsonSerializer.Deserialize<OAuth2TokenResponse>(responseContent);
                
                _logger.LogInformation("Successfully refreshed access token");
                return tokenResponse;
            }
            else
            {
                var errorContent = await response.Content.ReadAsStringAsync();
                _logger.LogError("Failed to refresh token. Status: {Status}, Content: {Content}", 
                    response.StatusCode, errorContent);
                throw new OAuth2Exception($"Token refresh failed: {response.StatusCode}");
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error refreshing token");
            throw;
        }
    }
}

public class OAuth2TokenResponse
{
    [JsonPropertyName("access_token")]
    public string AccessToken { get; set; }
    
    [JsonPropertyName("token_type")]
    public string TokenType { get; set; }
    
    [JsonPropertyName("expires_in")]
    public int ExpiresIn { get; set; }
    
    [JsonPropertyName("refresh_token")]
    public string RefreshToken { get; set; }
    
    [JsonPropertyName("scope")]
    public string Scope { get; set; }
    
    [JsonPropertyName("id_token")]
    public string IdToken { get; set; }
    
    public DateTime ExpiresAt => DateTime.UtcNow.AddSeconds(ExpiresIn);
    public bool IsExpired => DateTime.UtcNow >= ExpiresAt;
}
```

### 2. Client Credentials Flow
Machine-to-machine authentication için kullanılan flow.

```csharp
public class OAuth2ClientCredentialsFlow
{
    private readonly IConfiguration _configuration;
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly IMemoryCache _cache;
    private readonly ILogger<OAuth2ClientCredentialsFlow> _logger;
    
    public OAuth2ClientCredentialsFlow(
        IConfiguration configuration,
        IHttpClientFactory httpClientFactory,
        IMemoryCache cache,
        ILogger<OAuth2ClientCredentialsFlow> logger)
    {
        _configuration = configuration;
        _httpClientFactory = httpClientFactory;
        _cache = cache;
        _logger = logger;
    }
    
    public async Task<string> GetAccessTokenAsync(string scope = null)
    {
        var cacheKey = $"oauth2_token:{scope ?? "default"}";
        
        if (_cache.TryGetValue(cacheKey, out string cachedToken))
        {
            return cachedToken;
        }
        
        try
        {
            var client = _httpClientFactory.CreateClient("OAuth2");
            
            var tokenRequest = new FormUrlEncodedContent(new[]
            {
                new KeyValuePair<string, string>("grant_type", "client_credentials"),
                new KeyValuePair<string, string>("client_id", _configuration["OAuth2:ClientId"]),
                new KeyValuePair<string, string>("client_secret", _configuration["OAuth2:ClientSecret"]),
                new KeyValuePair<string, string>("scope", scope ?? _configuration["OAuth2:DefaultScope"])
            });
            
            var response = await client.PostAsync(_configuration["OAuth2:TokenEndpoint"], tokenRequest);
            
            if (response.IsSuccessStatusCode)
            {
                var responseContent = await response.Content.ReadAsStringAsync();
                var tokenResponse = JsonSerializer.Deserialize<OAuth2TokenResponse>(responseContent);
                
                // Cache the token with expiration
                var cacheOptions = new MemoryCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromSeconds(tokenResponse.ExpiresIn - 60) // Cache 1 minute less
                };
                
                _cache.Set(cacheKey, tokenResponse.AccessToken, cacheOptions);
                
                _logger.LogInformation("Successfully obtained client credentials token");
                return tokenResponse.AccessToken;
            }
            else
            {
                var errorContent = await response.Content.ReadAsStringAsync();
                _logger.LogError("Failed to obtain client credentials token. Status: {Status}, Content: {Content}", 
                    response.StatusCode, errorContent);
                throw new OAuth2Exception($"Client credentials flow failed: {response.StatusCode}");
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error obtaining client credentials token");
            throw;
        }
    }
    
    public async Task<bool> ValidateTokenAsync(string accessToken)
    {
        try
        {
            var client = _httpClientFactory.CreateClient("OAuth2");
            client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", accessToken);
            
            var response = await client.GetAsync(_configuration["OAuth2:UserInfoEndpoint"]);
            return response.IsSuccessStatusCode;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error validating token");
            return false;
        }
    }
}
```

### 3. Resource Owner Password Credentials Flow
Direct username/password authentication için (legacy uygulamalar).

```csharp
public class OAuth2PasswordFlow
{
    private readonly IConfiguration _configuration;
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly ILogger<OAuth2PasswordFlow> _logger;
    
    public OAuth2PasswordFlow(
        IConfiguration configuration,
        IHttpClientFactory httpClientFactory,
        ILogger<OAuth2PasswordFlow> logger)
    {
        _configuration = configuration;
        _httpClientFactory = httpClientFactory;
        _logger = logger;
    }
    
    public async Task<OAuth2TokenResponse> AuthenticateAsync(string username, string password, string scope = null)
    {
        try
        {
            var client = _httpClientFactory.CreateClient("OAuth2");
            
            var tokenRequest = new FormUrlEncodedContent(new[]
            {
                new KeyValuePair<string, string>("grant_type", "password"),
                new KeyValuePair<string, string>("username", username),
                new KeyValuePair<string, string>("password", password),
                new KeyValuePair<string, string>("client_id", _configuration["OAuth2:ClientId"]),
                new KeyValuePair<string, string>("client_secret", _configuration["OAuth2:ClientSecret"]),
                new KeyValuePair<string, string>("scope", scope ?? _configuration["OAuth2:DefaultScope"])
            });
            
            var response = await client.PostAsync(_configuration["OAuth2:TokenEndpoint"], tokenRequest);
            
            if (response.IsSuccessStatusCode)
            {
                var responseContent = await response.Content.ReadAsStringAsync();
                var tokenResponse = JsonSerializer.Deserialize<OAuth2TokenResponse>(responseContent);
                
                _logger.LogInformation("Successfully authenticated user {Username} with password flow", username);
                return tokenResponse;
            }
            else
            {
                var errorContent = await response.Content.ReadAsStringAsync();
                _logger.LogError("Failed to authenticate user {Username}. Status: {Status}, Content: {Content}", 
                    username, response.StatusCode, errorContent);
                throw new OAuth2Exception($"Password flow authentication failed: {response.StatusCode}");
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error authenticating user {Username} with password flow", username);
            throw;
        }
    }
}
```

## OAuth 2.0 Server Implementation

### 1. Authorization Server
OAuth 2.0 authorization server implementation.

```csharp
public class OAuth2AuthorizationServer
{
    private readonly IUserService _userService;
    private readonly IClientService _clientService;
    private readonly ITokenService _tokenService;
    private readonly ILogger<OAuth2AuthorizationServer> _logger;
    
    public OAuth2AuthorizationServer(
        IUserService userService,
        IClientService clientService,
        ITokenService tokenService,
        ILogger<OAuth2AuthorizationServer> logger)
    {
        _userService = userService;
        _clientService = clientService;
        _tokenService = tokenService;
        _logger = logger;
    }
    
    public async Task<AuthorizationRequest> ValidateAuthorizationRequestAsync(
        string clientId, 
        string redirectUri, 
        string responseType, 
        string scope, 
        string state)
    {
        try
        {
            // Validate client
            var client = await _clientService.GetClientByIdAsync(clientId);
            if (client == null || !client.IsActive)
            {
                throw new OAuth2Exception("Invalid client");
            }
            
            // Validate redirect URI
            if (!client.RedirectUris.Contains(redirectUri))
            {
                throw new OAuth2Exception("Invalid redirect URI");
            }
            
            // Validate response type
            if (responseType != "code")
            {
                throw new OAuth2Exception("Unsupported response type");
            }
            
            // Validate scope
            var requestedScopes = scope?.Split(' ', StringSplitOptions.RemoveEmptyEntries) ?? Array.Empty<string>();
            var validScopes = await ValidateScopesAsync(requestedScopes);
            
            var authorizationRequest = new AuthorizationRequest
            {
                ClientId = clientId,
                RedirectUri = redirectUri,
                ResponseType = responseType,
                Scope = string.Join(" ", validScopes),
                State = state,
                CreatedAt = DateTime.UtcNow,
                ExpiresAt = DateTime.UtcNow.AddMinutes(10)
            };
            
            _logger.LogInformation("Authorization request validated for client {ClientId}", clientId);
            return authorizationRequest;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error validating authorization request for client {ClientId}", clientId);
            throw;
        }
    }
    
    public async Task<AuthorizationCode> CreateAuthorizationCodeAsync(
        string clientId, 
        string userId, 
        string scope, 
        string redirectUri)
    {
        try
        {
            var authorizationCode = new AuthorizationCode
            {
                Code = GenerateRandomCode(),
                ClientId = clientId,
                UserId = userId,
                Scope = scope,
                RedirectUri = redirectUri,
                CreatedAt = DateTime.UtcNow,
                ExpiresAt = DateTime.UtcNow.AddMinutes(10)
            };
            
            await _tokenService.StoreAuthorizationCodeAsync(authorizationCode);
            
            _logger.LogInformation("Authorization code created for client {ClientId} and user {UserId}", clientId, userId);
            return authorizationCode;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating authorization code for client {ClientId} and user {UserId}", clientId, userId);
            throw;
        }
    }
    
    public async Task<OAuth2TokenResponse> ExchangeAuthorizationCodeAsync(
        string authorizationCode, 
        string clientId, 
        string clientSecret, 
        string redirectUri)
    {
        try
        {
            // Validate authorization code
            var code = await _tokenService.GetAuthorizationCodeAsync(authorizationCode);
            if (code == null || code.IsExpired || code.IsUsed)
            {
                throw new OAuth2Exception("Invalid or expired authorization code");
            }
            
            // Validate client
            var client = await _clientService.GetClientByIdAsync(clientId);
            if (client == null || !client.IsActive || client.ClientSecret != clientSecret)
            {
                throw new OAuth2Exception("Invalid client");
            }
            
            // Validate redirect URI
            if (code.RedirectUri != redirectUri)
            {
                throw new OAuth2Exception("Redirect URI mismatch");
            }
            
            // Mark code as used
            await _tokenService.MarkAuthorizationCodeAsUsedAsync(authorizationCode);
            
            // Create access token
            var accessToken = await _tokenService.CreateAccessTokenAsync(code.UserId, code.ClientId, code.Scope);
            
            // Create refresh token
            var refreshToken = await _tokenService.CreateRefreshTokenAsync(code.UserId, code.ClientId, code.Scope);
            
            var tokenResponse = new OAuth2TokenResponse
            {
                AccessToken = accessToken.Token,
                TokenType = "Bearer",
                ExpiresIn = (int)(accessToken.ExpiresAt - DateTime.UtcNow).TotalSeconds,
                RefreshToken = refreshToken.Token,
                Scope = code.Scope
            };
            
            _logger.LogInformation("Authorization code exchanged for tokens for client {ClientId} and user {UserId}", clientId, code.UserId);
            return tokenResponse;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error exchanging authorization code {Code}", authorizationCode);
            throw;
        }
    }
    
    private async Task<string[]> ValidateScopesAsync(string[] requestedScopes)
    {
        var validScopes = new List<string>();
        var availableScopes = await GetAvailableScopesAsync();
        
        foreach (var scope in requestedScopes)
        {
            if (availableScopes.Contains(scope))
            {
                validScopes.Add(scope);
            }
            else
            {
                _logger.LogWarning("Invalid scope requested: {Scope}", scope);
            }
        }
        
        return validScopes.ToArray();
    }
    
    private async Task<string[]> GetAvailableScopesAsync()
    {
        return new[]
        {
            "read",
            "write",
            "delete",
            "admin"
        };
    }
    
    private string GenerateRandomCode()
    {
        var randomBytes = new byte[32];
        using var rng = new RNGCryptoServiceProvider();
        rng.GetBytes(randomBytes);
        return Convert.ToBase64String(randomBytes).Replace("+", "-").Replace("/", "_").TrimEnd('=');
    }
}

public class AuthorizationRequest
{
    public string ClientId { get; set; }
    public string RedirectUri { get; set; }
    public string ResponseType { get; set; }
    public string Scope { get; set; }
    public string State { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime ExpiresAt { get; set; }
    public bool IsExpired => DateTime.UtcNow >= ExpiresAt;
}

public class AuthorizationCode
{
    public string Code { get; set; }
    public string ClientId { get; set; }
    public string UserId { get; set; }
    public string Scope { get; set; }
    public string RedirectUri { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime ExpiresAt { get; set; }
    public bool IsUsed { get; set; }
    public bool IsExpired => DateTime.UtcNow >= ExpiresAt;
}
```

### 2. Token Service
OAuth 2.0 token yönetimi.

```csharp
public interface ITokenService
{
    Task<AccessToken> CreateAccessTokenAsync(string userId, string clientId, string scope);
    Task<RefreshToken> CreateRefreshTokenAsync(string userId, string clientId, string scope);
    Task<AccessToken> GetAccessTokenAsync(string token);
    Task<RefreshToken> GetRefreshTokenAsync(string token);
    Task<bool> RevokeAccessTokenAsync(string token);
    Task<bool> RevokeRefreshTokenAsync(string token);
    Task StoreAuthorizationCodeAsync(AuthorizationCode code);
    Task<AuthorizationCode> GetAuthorizationCodeAsync(string code);
    Task MarkAuthorizationCodeAsUsedAsync(string code);
}

public class TokenService : ITokenService
{
    private readonly IDbConnection _connection;
    private readonly ILogger<TokenService> _logger;
    private readonly IJwtTokenBuilder _jwtTokenBuilder;
    
    public TokenService(
        IDbConnection connection,
        ILogger<TokenService> logger,
        IJwtTokenBuilder jwtTokenBuilder)
    {
        _connection = connection;
        _logger = logger;
        _jwtTokenBuilder = jwtTokenBuilder;
    }
    
    public async Task<AccessToken> CreateAccessTokenAsync(string userId, string clientId, string scope)
    {
        try
        {
            var accessToken = new AccessToken
            {
                Token = GenerateRandomToken(),
                UserId = userId,
                ClientId = clientId,
                Scope = scope,
                CreatedAt = DateTime.UtcNow,
                ExpiresAt = DateTime.UtcNow.AddHours(1)
            };
            
            var sql = @"
                INSERT INTO AccessTokens (Token, UserId, ClientId, Scope, CreatedAt, ExpiresAt)
                VALUES (@Token, @UserId, @ClientId, @Scope, @CreatedAt, @ExpiresAt)
            ";
            
            await _connection.ExecuteAsync(sql, accessToken);
            
            _logger.LogInformation("Access token created for user {UserId} and client {ClientId}", userId, clientId);
            return accessToken;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating access token for user {UserId} and client {ClientId}", userId, clientId);
            throw;
        }
    }
    
    public async Task<RefreshToken> CreateRefreshTokenAsync(string userId, string clientId, string scope)
    {
        try
        {
            var refreshToken = new RefreshToken
            {
                Token = GenerateRandomToken(),
                UserId = userId,
                ClientId = clientId,
                Scope = scope,
                CreatedAt = DateTime.UtcNow,
                ExpiresAt = DateTime.UtcNow.AddDays(30)
            };
            
            var sql = @"
                INSERT INTO RefreshTokens (Token, UserId, ClientId, Scope, CreatedAt, ExpiresAt)
                VALUES (@Token, @UserId, @ClientId, @Scope, @CreatedAt, @ExpiresAt)
            ";
            
            await _connection.ExecuteAsync(sql, refreshToken);
            
            _logger.LogInformation("Refresh token created for user {UserId} and client {ClientId}", userId, clientId);
            return refreshToken;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating refresh token for user {UserId} and client {ClientId}", userId, clientId);
            throw;
        }
    }
    
    public async Task<AccessToken> GetAccessTokenAsync(string token)
    {
        try
        {
            var sql = @"
                SELECT Token, UserId, ClientId, Scope, CreatedAt, ExpiresAt
                FROM AccessTokens 
                WHERE Token = @Token AND ExpiresAt > @Now
            ";
            
            var accessToken = await _connection.QueryFirstOrDefaultAsync<AccessToken>(sql, new { Token = token, Now = DateTime.UtcNow });
            
            if (accessToken != null && accessToken.IsExpired)
            {
                await RevokeAccessTokenAsync(token);
                return null;
            }
            
            return accessToken;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting access token {Token}", token);
            throw;
        }
    }
    
    public async Task<bool> RevokeAccessTokenAsync(string token)
    {
        try
        {
            var sql = "DELETE FROM AccessTokens WHERE Token = @Token";
            var rowsAffected = await _connection.ExecuteAsync(sql, new { Token = token });
            
            _logger.LogInformation("Access token {Token} revoked", token);
            return rowsAffected > 0;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error revoking access token {Token}", token);
            throw;
        }
    }
    
    public async Task StoreAuthorizationCodeAsync(AuthorizationCode code)
    {
        try
        {
            var sql = @"
                INSERT INTO AuthorizationCodes (Code, ClientId, UserId, Scope, RedirectUri, CreatedAt, ExpiresAt)
                VALUES (@Code, @ClientId, @UserId, @Scope, @RedirectUri, @CreatedAt, @ExpiresAt)
            ";
            
            await _connection.ExecuteAsync(sql, code);
            
            _logger.LogInformation("Authorization code {Code} stored", code.Code);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error storing authorization code {Code}", code.Code);
            throw;
        }
    }
    
    public async Task<AuthorizationCode> GetAuthorizationCodeAsync(string code)
    {
        try
        {
            var sql = @"
                SELECT Code, ClientId, UserId, Scope, RedirectUri, CreatedAt, ExpiresAt, IsUsed
                FROM AuthorizationCodes 
                WHERE Code = @Code
            ";
            
            return await _connection.QueryFirstOrDefaultAsync<AuthorizationCode>(sql, new { Code = code });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting authorization code {Code}", code);
            throw;
        }
    }
    
    public async Task MarkAuthorizationCodeAsUsedAsync(string code)
    {
        try
        {
            var sql = "UPDATE AuthorizationCodes SET IsUsed = 1 WHERE Code = @Code";
            await _connection.ExecuteAsync(sql, new { Code = code });
            
            _logger.LogInformation("Authorization code {Code} marked as used", code);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error marking authorization code {Code} as used", code);
            throw;
        }
    }
    
    private string GenerateRandomToken()
    {
        var randomBytes = new byte[32];
        using var rng = new RNGCryptoServiceProvider();
        rng.GetBytes(randomBytes);
        return Convert.ToBase64String(randomBytes).Replace("+", "-").Replace("/", "_").TrimEnd('=');
    }
}

public class AccessToken
{
    public string Token { get; set; }
    public string UserId { get; set; }
    public string ClientId { get; set; }
    public string Scope { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime ExpiresAt { get; set; }
    public bool IsExpired => DateTime.UtcNow >= ExpiresAt;
}

public class RefreshToken
{
    public string Token { get; set; }
    public string UserId { get; set; }
    public string ClientId { get; set; }
    public string Scope { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime ExpiresAt { get; set; }
    public bool IsExpired => DateTime.UtcNow >= ExpiresAt;
}
```

## OAuth 2.0 Middleware

### 1. OAuth 2.0 Authentication Middleware
```csharp
public class OAuth2AuthenticationMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ITokenService _tokenService;
    private readonly ILogger<OAuth2AuthenticationMiddleware> _logger;
    
    public OAuth2AuthenticationMiddleware(
        RequestDelegate next,
        ITokenService tokenService,
        ILogger<OAuth2AuthenticationMiddleware> logger)
    {
        _next = next;
        _tokenService = tokenService;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            var token = ExtractTokenFromContext(context);
            
            if (!string.IsNullOrEmpty(token))
            {
                var accessToken = await _tokenService.GetAccessTokenAsync(token);
                
                if (accessToken != null)
                {
                    // Set user identity in context
                    var claims = new List<Claim>
                    {
                        new Claim(ClaimTypes.NameIdentifier, accessToken.UserId),
                        new Claim("client_id", accessToken.ClientId),
                        new Claim("scope", accessToken.Scope)
                    };
                    
                    var identity = new ClaimsIdentity(claims, "OAuth2");
                    context.User = new ClaimsPrincipal(identity);
                    
                    _logger.LogDebug("OAuth2 authentication successful for user {UserId}", accessToken.UserId);
                }
                else
                {
                    _logger.LogWarning("Invalid or expired OAuth2 token: {Token}", token);
                }
            }
            
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error in OAuth2 authentication middleware");
            await _next(context);
        }
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
}

public static class OAuth2AuthenticationMiddlewareExtensions
{
    public static IApplicationBuilder UseOAuth2Authentication(this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<OAuth2AuthenticationMiddleware>();
    }
}
```

## Mülakat Soruları

### Temel Sorular

1. **OAuth 2.0 nedir ve neden kullanılır?**
   - **Cevap**: Üçüncü taraf uygulamalara kullanıcı verilerine erişim izni vermek için kullanılan standart. Security ve user experience için kritik.

2. **OAuth 2.0 flow türleri nelerdir?**
   - **Cevap**: Authorization Code, Client Credentials, Password, Implicit. Her biri farklı use case'ler için tasarlanmış.

3. **Authorization Code flow nasıl çalışır?**
   - **Cevap**: User authorization → Authorization code → Token exchange → Access token. En güvenli flow.

4. **Refresh token nedir ve nasıl kullanılır?**
   - **Cevap**: Access token'ı yenilemek için kullanılan token. Long-lived ama revokable.

5. **OAuth 2.0 vs OpenID Connect arasındaki fark nedir?**
   - **Cevap**: OAuth 2.0 authorization, OpenID Connect authentication. OpenID Connect OAuth 2.0 üzerine kurulu.

### Teknik Sorular

1. **OAuth 2.0 server nasıl implement edilir?**
   - **Cevap**: Authorization endpoint, token endpoint, client validation, scope management, token storage.

2. **OAuth 2.0 security vulnerabilities nasıl önlenir?**
   - **Cevap**: HTTPS, state parameter, PKCE, scope validation, token expiration, secure storage.

3. **OAuth 2.0 client registration nasıl yapılır?**
   - **Cevap**: Client ID/Secret, redirect URIs, scope definition, client validation.

4. **OAuth 2.0 token validation nasıl yapılır?**
   - **Cevap**: Signature verification, expiration check, scope validation, client validation.

5. **OAuth 2.0 rate limiting nasıl implement edilir?**
   - **Cevap**: Client-based limiting, user-based limiting, endpoint-based limiting, token-based limiting.

## Best Practices

1. **Security**
   - HTTPS zorunlu yapın
   - State parameter kullanın
   - PKCE implement edin
   - Scope validation yapın

2. **Performance**
   - Token caching implement edin
   - Database optimization yapın
   - Async operations kullanın
   - Connection pooling kullanın

3. **Monitoring**
   - Token usage izleyin
   - Failed authentications log edin
   - Rate limiting metrics toplayın
   - Security alerts kurun

4. **Maintenance**
   - Regular security audits yapın
   - Client management automate edin
   - Token cleanup implement edin
   - Documentation güncelleyin

## Kaynaklar

- [OAuth 2.0 Specification](https://tools.ietf.org/html/rfc6749)
- [OAuth 2.0 Security Best Practices](https://tools.ietf.org/html/rfc6819)
- [ASP.NET Core OAuth 2.0](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/oauth)
- [OAuth 2.0 Implementation Guide](https://auth0.com/docs/protocols/oauth2)
- [OAuth 2.0 Security Considerations](https://tools.ietf.org/html/rfc6819)
