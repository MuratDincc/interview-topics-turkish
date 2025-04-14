# JWT (JSON Web Token)

## Genel Bakış
JWT, iki taraf arasında güvenli bilgi alışverişi için kullanılan açık bir standarttır. Token'lar JSON formatında yapılandırılmış ve dijital olarak imzalanmıştır. JWT'ler kimlik doğrulama ve yetkilendirme için yaygın olarak kullanılır.

## Temel Kavramlar

### 1. JWT Oluşturma ve İmzalama
```csharp
public class JwtService
{
    private readonly ILogger<JwtService> _logger;
    private readonly IConfiguration _configuration;

    public JwtService(ILogger<JwtService> logger, IConfiguration configuration)
    {
        _logger = logger;
        _configuration = configuration;
    }

    public string GenerateJwtToken(UserClaims claims)
    {
        var securityKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_configuration["Jwt:Key"]));
        var credentials = new SigningCredentials(securityKey, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: _configuration["Jwt:Issuer"],
            audience: _configuration["Jwt:Audience"],
            claims: new List<Claim>
            {
                new Claim(JwtRegisteredClaimNames.Sub, claims.Subject),
                new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
                new Claim(JwtRegisteredClaimNames.Iat, DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString()),
                new Claim(ClaimTypes.Name, claims.Username),
                new Claim(ClaimTypes.Role, claims.Role)
            },
            expires: DateTime.UtcNow.AddMinutes(30),
            signingCredentials: credentials
        );

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

### 2. JWT Doğrulama
```csharp
public class JwtValidator
{
    private readonly ILogger<JwtValidator> _logger;
    private readonly IConfiguration _configuration;

    public JwtValidator(ILogger<JwtValidator> logger, IConfiguration configuration)
    {
        _logger = logger;
        _configuration = configuration;
    }

    public async Task<ClaimsPrincipal> ValidateTokenAsync(string token)
    {
        var tokenHandler = new JwtSecurityTokenHandler();
        var key = Encoding.UTF8.GetBytes(_configuration["Jwt:Key"]);

        try
        {
            var tokenValidationParameters = new TokenValidationParameters
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

            var principal = tokenHandler.ValidateToken(token, tokenValidationParameters, out var validatedToken);
            return principal;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Token validation failed");
            throw new SecurityException("Invalid token");
        }
    }
}
```

### 3. JWT Yenileme
```csharp
public class JwtRefreshService
{
    private readonly ILogger<JwtRefreshService> _logger;
    private readonly IJwtService _jwtService;
    private readonly ITokenStorage _tokenStorage;

    public JwtRefreshService(ILogger<JwtRefreshService> logger, IJwtService jwtService, ITokenStorage tokenStorage)
    {
        _logger = logger;
        _jwtService = jwtService;
        _tokenStorage = tokenStorage;
    }

    public async Task<TokenResponse> RefreshTokenAsync(string refreshToken)
    {
        // Refresh token doğrulama
        var storedToken = await _tokenStorage.GetTokenByRefreshTokenAsync(refreshToken);
        if (storedToken == null || storedToken.IsRevoked)
        {
            throw new SecurityException("Invalid refresh token");
        }

        // Yeni access token oluştur
        var newAccessToken = _jwtService.GenerateJwtToken(new UserClaims
        {
            Subject = storedToken.Subject,
            Username = storedToken.Username,
            Role = storedToken.Role
        });

        // Yeni refresh token oluştur
        var newRefreshToken = Guid.NewGuid().ToString();

        // Token'ları güncelle
        await _tokenStorage.UpdateTokensAsync(storedToken.Id, newAccessToken, newRefreshToken);

        return new TokenResponse
        {
            AccessToken = newAccessToken,
            RefreshToken = newRefreshToken,
            ExpiresIn = 1800 // 30 dakika
        };
    }
}
```

## Best Practices

### 1. Token Güvenliği
- Güçlü imzalama algoritmaları
- Uygun token süresi
- Token şifreleme
- Token iptali
- Token yenileme

### 2. Token Yönetimi
- Güvenli token saklama
- Token yenileme stratejisi
- Token iptal mekanizması
- Token süre yönetimi
- Token izleme

### 3. Güvenlik Önlemleri
- HTTPS kullanımı
- Token boyutu
- Claim doğrulama
- Token sızıntısı
- Brute force koruması

## Sık Sorulan Sorular

### 1. JWT neden kullanılır?
- Kimlik doğrulama
- Yetkilendirme
- Bilgi alışverişi
- Oturum yönetimi
- API güvenliği

### 2. JWT güvenliği nasıl sağlanır?
- Güçlü imzalama
- Uygun süre
- Güvenli saklama
- Token yenileme
- Token iptali

### 3. JWT zorlukları nelerdir?
- Token boyutu
- Token iptali
- Güvenlik riskleri
- Performans etkisi
- Karmaşık yapı

## Kaynaklar
- [JWT Specification](https://tools.ietf.org/html/rfc7519)
- [JWT Best Practices](https://auth0.com/blog/brute-forcing-hs256-is-possible-the-importance-of-using-strong-keys-to-sign-jwts/)
- [JWT Security Considerations](https://tools.ietf.org/html/rfc8725)
- [Microsoft JWT Documentation](https://docs.microsoft.com/tr-tr/azure/active-directory/develop/access-tokens)
- [JWT Libraries](https://jwt.io/libraries) 