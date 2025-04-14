# OAuth2

## Genel Bakış
OAuth2, üçüncü taraf uygulamaların kullanıcı adına sınırlı erişim sağlamasına olanak tanıyan bir yetkilendirme protokolüdür. Kullanıcıların kimlik bilgilerini paylaşmadan, belirli kaynaklara erişim izni vermesini sağlar.

## Temel Kavramlar

### 1. OAuth2 Akışları
```csharp
public class OAuth2Service
{
    private readonly ILogger<OAuth2Service> _logger;
    private readonly IHttpClientFactory _httpClientFactory;

    public OAuth2Service(ILogger<OAuth2Service> logger, IHttpClientFactory httpClientFactory)
    {
        _logger = logger;
        _httpClientFactory = httpClientFactory;
    }

    // Authorization Code Flow
    public async Task<AuthorizationCodeResponse> GetAuthorizationCodeAsync(AuthorizationRequest request)
    {
        var client = _httpClientFactory.CreateClient();
        var response = await client.PostAsync(request.AuthorizationEndpoint, new FormUrlEncodedContent(new Dictionary<string, string>
        {
            ["client_id"] = request.ClientId,
            ["redirect_uri"] = request.RedirectUri,
            ["response_type"] = "code",
            ["scope"] = string.Join(" ", request.Scopes),
            ["state"] = request.State
        }));

        var content = await response.Content.ReadAsStringAsync();
        return JsonSerializer.Deserialize<AuthorizationCodeResponse>(content);
    }

    // Token Exchange
    public async Task<TokenResponse> ExchangeCodeForTokenAsync(TokenRequest request)
    {
        var client = _httpClientFactory.CreateClient();
        var response = await client.PostAsync(request.TokenEndpoint, new FormUrlEncodedContent(new Dictionary<string, string>
        {
            ["grant_type"] = "authorization_code",
            ["code"] = request.Code,
            ["redirect_uri"] = request.RedirectUri,
            ["client_id"] = request.ClientId,
            ["client_secret"] = request.ClientSecret
        }));

        var content = await response.Content.ReadAsStringAsync();
        return JsonSerializer.Deserialize<TokenResponse>(content);
    }
}
```

### 2. Token Yönetimi
```csharp
public class TokenManagementService
{
    private readonly ILogger<TokenManagementService> _logger;
    private readonly ITokenStorage _tokenStorage;

    public TokenManagementService(ILogger<TokenManagementService> logger, ITokenStorage tokenStorage)
    {
        _logger = logger;
        _tokenStorage = tokenStorage;
    }

    public async Task<TokenInfo> StoreTokenAsync(TokenResponse tokenResponse)
    {
        var tokenInfo = new TokenInfo
        {
            AccessToken = tokenResponse.AccessToken,
            RefreshToken = tokenResponse.RefreshToken,
            ExpiresIn = tokenResponse.ExpiresIn,
            TokenType = tokenResponse.TokenType,
            Scope = tokenResponse.Scope,
            CreatedAt = DateTime.UtcNow
        };

        await _tokenStorage.StoreTokenAsync(tokenInfo);
        _logger.LogInformation("Token stored successfully");
        return tokenInfo;
    }

    public async Task<TokenInfo> RefreshTokenAsync(string refreshToken)
    {
        var tokenInfo = await _tokenStorage.GetTokenByRefreshTokenAsync(refreshToken);
        if (tokenInfo == null)
        {
            throw new InvalidOperationException("Refresh token not found");
        }

        // Token yenileme işlemi
        var newToken = await RefreshTokenInternalAsync(tokenInfo);
        await _tokenStorage.UpdateTokenAsync(newToken);
        return newToken;
    }
}
```

### 3. Güvenlik Uygulamaları
```csharp
public class OAuth2SecurityService
{
    private readonly ILogger<OAuth2SecurityService> _logger;
    private readonly ISecurityValidator _securityValidator;

    public OAuth2SecurityService(ILogger<OAuth2SecurityService> logger, ISecurityValidator securityValidator)
    {
        _logger = logger;
        _securityValidator = securityValidator;
    }

    public async Task ValidateRequestAsync(OAuth2Request request)
    {
        // State parametresi kontrolü
        if (!await _securityValidator.ValidateStateAsync(request.State))
        {
            throw new SecurityException("Invalid state parameter");
        }

        // Redirect URI kontrolü
        if (!await _securityValidator.ValidateRedirectUriAsync(request.RedirectUri))
        {
            throw new SecurityException("Invalid redirect URI");
        }

        // Scope kontrolü
        if (!await _securityValidator.ValidateScopesAsync(request.Scopes))
        {
            throw new SecurityException("Invalid scopes");
        }

        // PKCE kontrolü
        if (request.CodeChallenge != null)
        {
            if (!await _securityValidator.ValidateCodeChallengeAsync(request.CodeChallenge, request.CodeChallengeMethod))
            {
                throw new SecurityException("Invalid code challenge");
            }
        }
    }
}
```

## Best Practices

### 1. Güvenlik Önlemleri
- PKCE (Proof Key for Code Exchange) kullanımı
- State parametresi kontrolü
- Redirect URI doğrulama
- Scope sınırlaması
- Token güvenliği

### 2. Token Yönetimi
- Güvenli token saklama
- Token yenileme stratejisi
- Token iptal mekanizması
- Token süre yönetimi
- Token şifreleme

### 3. Hata Yönetimi
- Güvenli hata mesajları
- Loglama stratejisi
- Hata izleme
- İyileştirme önerileri
- Kullanıcı bildirimleri

## Sık Sorulan Sorular

### 1. OAuth2 akışları nelerdir?
- Authorization Code Flow
- Implicit Flow
- Client Credentials Flow
- Resource Owner Password Credentials Flow
- Device Flow

### 2. OAuth2 güvenliği nasıl sağlanır?
- PKCE kullanımı
- State parametresi
- Redirect URI doğrulama
- Scope sınırlaması
- Token güvenliği

### 3. OAuth2 zorlukları nelerdir?
- Token yönetimi
- Güvenlik riskleri
- Performans etkisi
- Entegrasyon karmaşıklığı
- Hata yönetimi

## Kaynaklar
- [OAuth2 Specification](https://tools.ietf.org/html/rfc6749)
- [OAuth2 Best Practices](https://oauth.net/2/)
- [OAuth2 Security Considerations](https://tools.ietf.org/html/rfc6819)
- [OAuth2 Implementation Guide](https://oauth.net/2/grant-types/)
- [OAuth2 Libraries](https://oauth.net/code/) 