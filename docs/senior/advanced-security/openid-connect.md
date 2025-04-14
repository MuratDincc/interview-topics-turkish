# OpenID Connect

## Genel Bakış
OpenID Connect, OAuth2 protokolünün üzerine inşa edilmiş bir kimlik doğrulama katmanıdır. Kullanıcıların kimliklerini doğrulamak ve temel profil bilgilerine erişmek için standart bir yöntem sağlar.

## Temel Kavramlar

### 1. OpenID Connect Akışları
```csharp
public class OpenIdConnectService
{
    private readonly ILogger<OpenIdConnectService> _logger;
    private readonly IHttpClientFactory _httpClientFactory;

    public OpenIdConnectService(ILogger<OpenIdConnectService> logger, IHttpClientFactory httpClientFactory)
    {
        _logger = logger;
        _httpClientFactory = httpClientFactory;
    }

    // Authorization Request
    public async Task<AuthorizationResponse> GetAuthorizationAsync(AuthorizationRequest request)
    {
        var client = _httpClientFactory.CreateClient();
        var response = await client.PostAsync(request.AuthorizationEndpoint, new FormUrlEncodedContent(new Dictionary<string, string>
        {
            ["client_id"] = request.ClientId,
            ["redirect_uri"] = request.RedirectUri,
            ["response_type"] = "code",
            ["scope"] = "openid profile email",
            ["state"] = request.State,
            ["nonce"] = request.Nonce
        }));

        var content = await response.Content.ReadAsStringAsync();
        return JsonSerializer.Deserialize<AuthorizationResponse>(content);
    }

    // Token Request
    public async Task<TokenResponse> GetTokensAsync(TokenRequest request)
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

### 2. ID Token Doğrulama
```csharp
public class IdTokenValidator
{
    private readonly ILogger<IdTokenValidator> _logger;
    private readonly IJwtValidator _jwtValidator;

    public IdTokenValidator(ILogger<IdTokenValidator> logger, IJwtValidator jwtValidator)
    {
        _logger = logger;
        _jwtValidator = jwtValidator;
    }

    public async Task<IdTokenClaims> ValidateIdTokenAsync(string idToken, string clientId, string issuer)
    {
        // Token imza doğrulama
        var isValid = await _jwtValidator.ValidateSignatureAsync(idToken);
        if (!isValid)
        {
            throw new SecurityException("Invalid token signature");
        }

        // Token claims doğrulama
        var claims = await _jwtValidator.ValidateClaimsAsync(idToken, new TokenValidationParameters
        {
            ValidIssuer = issuer,
            ValidAudience = clientId,
            RequireExpirationTime = true,
            ValidateLifetime = true,
            ClockSkew = TimeSpan.FromMinutes(5)
        });

        return new IdTokenClaims
        {
            Subject = claims.Subject,
            Issuer = claims.Issuer,
            Audience = claims.Audience,
            Expiration = claims.Expiration,
            IssuedAt = claims.IssuedAt,
            Nonce = claims.Nonce
        };
    }
}
```

### 3. UserInfo Endpoint
```csharp
public class UserInfoService
{
    private readonly ILogger<UserInfoService> _logger;
    private readonly IHttpClientFactory _httpClientFactory;

    public UserInfoService(ILogger<UserInfoService> logger, IHttpClientFactory httpClientFactory)
    {
        _logger = logger;
        _httpClientFactory = httpClientFactory;
    }

    public async Task<UserInfo> GetUserInfoAsync(string accessToken, string userInfoEndpoint)
    {
        var client = _httpClientFactory.CreateClient();
        client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", accessToken);

        var response = await client.GetAsync(userInfoEndpoint);
        var content = await response.Content.ReadAsStringAsync();
        
        return JsonSerializer.Deserialize<UserInfo>(content);
    }
}
```

## Best Practices

### 1. Güvenlik Önlemleri
- ID Token doğrulama
- Nonce kontrolü
- Token yenileme
- PKCE kullanımı
- State parametresi

### 2. Kimlik Yönetimi
- Kullanıcı bilgileri
- Profil yönetimi
- Oturum yönetimi
- Çoklu oturum
- Oturum sonlandırma

### 3. Entegrasyon
- Client uygulaması
- Identity Provider
- Token yönetimi
- Hata yönetimi
- Loglama

## Sık Sorulan Sorular

### 1. OpenID Connect ve OAuth2 arasındaki fark nedir?
- Kimlik doğrulama vs yetkilendirme
- ID Token vs Access Token
- Kullanıcı bilgileri
- Profil yönetimi
- Güvenlik önlemleri

### 2. OpenID Connect güvenliği nasıl sağlanır?
- ID Token doğrulama
- Nonce kontrolü
- Token yenileme
- PKCE kullanımı
- State parametresi

### 3. OpenID Connect zorlukları nelerdir?
- Token yönetimi
- Güvenlik riskleri
- Entegrasyon karmaşıklığı
- Performans etkisi
- Hata yönetimi

## Kaynaklar
- [OpenID Connect Specification](https://openid.net/connect/)
- [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)
- [OpenID Connect Best Practices](https://openid.net/developers/certified/)
- [Microsoft Identity Platform](https://docs.microsoft.com/tr-tr/azure/active-directory/develop/v2-protocols-oidc)
- [OpenID Connect Libraries](https://openid.net/developers/libraries/) 