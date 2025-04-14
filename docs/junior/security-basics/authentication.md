# Authentication

## Genel Bakış
Authentication (Kimlik Doğrulama), bir kullanıcının veya sistemin kimliğinin doğrulanması sürecidir. Bu süreç, kullanıcının iddia ettiği kimliğin doğru olup olmadığını kontrol eder.

## Mülakat Soruları ve Cevapları

### 1. Temel kimlik doğrulama yöntemleri nelerdir?
**Cevap:**
Temel kimlik doğrulama yöntemleri:
- Kullanıcı adı/şifre
- Token tabanlı (JWT)
- OAuth/OpenID Connect
- Biyometrik
- Çok faktörlü kimlik doğrulama (MFA)

**Örnek Kod:**
```csharp
// Kullanıcı adı/şifre doğrulama
public class AccountController : ControllerBase
{
    [HttpPost("login")]
    public async Task<IActionResult> Login(LoginModel model)
    {
        var user = await _userManager.FindByNameAsync(model.Username);
        if (user != null && await _userManager.CheckPasswordAsync(user, model.Password))
        {
            var token = await GenerateJwtToken(user);
            return Ok(new { token });
        }
        return Unauthorized();
    }
}

// JWT token oluşturma
private async Task<string> GenerateJwtToken(ApplicationUser user)
{
    var claims = new List<Claim>
    {
        new Claim(ClaimTypes.NameIdentifier, user.Id),
        new Claim(ClaimTypes.Name, user.UserName)
    };

    var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_configuration["Jwt:Key"]));
    var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

    var token = new JwtSecurityToken(
        issuer: _configuration["Jwt:Issuer"],
        audience: _configuration["Jwt:Audience"],
        claims: claims,
        expires: DateTime.Now.AddHours(1),
        signingCredentials: creds
    );

    return new JwtSecurityTokenHandler().WriteToken(token);
}
```

### 2. JWT (JSON Web Token) nedir ve nasıl çalışır?
**Cevap:**
JWT, güvenli bilgi alışverişi için kullanılan bir token formatıdır. Üç bölümden oluşur:
- Header: Token tipi ve kullanılan algoritma
- Payload: Kullanıcı bilgileri ve claims
- Signature: Token'ın doğruluğunu kontrol etmek için

**Örnek Kod:**
```csharp
// JWT yapılandırması
public void ConfigureServices(IServiceCollection services)
{
    services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
        .AddJwtBearer(options =>
        {
            options.TokenValidationParameters = new TokenValidationParameters
            {
                ValidateIssuer = true,
                ValidateAudience = true,
                ValidateLifetime = true,
                ValidateIssuerSigningKey = true,
                ValidIssuer = Configuration["Jwt:Issuer"],
                ValidAudience = Configuration["Jwt:Audience"],
                IssuerSigningKey = new SymmetricSecurityKey(
                    Encoding.UTF8.GetBytes(Configuration["Jwt:Key"]))
            };
        });
}

// JWT kullanımı
[Authorize]
[ApiController]
[Route("api/[controller]")]
public class SecureController : ControllerBase
{
    [HttpGet]
    public IActionResult GetSecureData()
    {
        var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        return Ok($"Secure data for user {userId}");
    }
}
```

### 3. OAuth 2.0 ve OpenID Connect nedir?
**Cevap:**
OAuth 2.0, yetkilendirme protokolüdür. OpenID Connect ise OAuth 2.0 üzerine inşa edilmiş bir kimlik doğrulama katmanıdır.

**Örnek Kod:**
```csharp
// OAuth yapılandırması
public void ConfigureServices(IServiceCollection services)
{
    services.AddAuthentication()
        .AddGoogle(options =>
        {
            options.ClientId = Configuration["Authentication:Google:ClientId"];
            options.ClientSecret = Configuration["Authentication:Google:ClientSecret"];
        })
        .AddMicrosoftAccount(options =>
        {
            options.ClientId = Configuration["Authentication:Microsoft:ClientId"];
            options.ClientSecret = Configuration["Authentication:Microsoft:ClientSecret"];
        });
}

// OAuth kullanımı
[Authorize]
[ApiController]
[Route("api/[controller]")]
public class OAuthController : ControllerBase
{
    [HttpGet("userinfo")]
    public async Task<IActionResult> GetUserInfo()
    {
        var email = User.FindFirst(ClaimTypes.Email)?.Value;
        var name = User.FindFirst(ClaimTypes.Name)?.Value;
        return Ok(new { email, name });
    }
}
```

### 4. Çok faktörlü kimlik doğrulama (MFA) nasıl uygulanır?
**Cevap:**
MFA, birden fazla doğrulama yönteminin kullanılmasıdır. Örneğin:
- SMS/Email kodları
- Authenticator uygulamaları
- Biyometrik doğrulama

**Örnek Kod:**
```csharp
// MFA yapılandırması
public class MfaController : ControllerBase
{
    [HttpPost("enable-mfa")]
    public async Task<IActionResult> EnableMfa()
    {
        var user = await _userManager.GetUserAsync(User);
        var authenticatorKey = await _userManager.GetAuthenticatorKeyAsync(user);
        if (string.IsNullOrEmpty(authenticatorKey))
        {
            await _userManager.ResetAuthenticatorKeyAsync(user);
            authenticatorKey = await _userManager.GetAuthenticatorKeyAsync(user);
        }

        return Ok(new { authenticatorKey });
    }

    [HttpPost("verify-mfa")]
    public async Task<IActionResult> VerifyMfa(string code)
    {
        var user = await _userManager.GetUserAsync(User);
        var isValid = await _userManager.VerifyTwoFactorTokenAsync(
            user, _userManager.Options.Tokens.AuthenticatorTokenProvider, code);

        if (isValid)
        {
            await _userManager.SetTwoFactorEnabledAsync(user, true);
            return Ok();
        }

        return BadRequest("Invalid verification code");
    }
}
```

### 5. Güvenli oturum yönetimi nasıl sağlanır?
**Cevap:**
Güvenli oturum yönetimi için:
- Güvenli token saklama
- Token yenileme
- Oturum zaman aşımı
- Güvenli çıkış

**Örnek Kod:**
```csharp
// Oturum yönetimi
public class SessionController : ControllerBase
{
    [HttpPost("refresh-token")]
    public async Task<IActionResult> RefreshToken(string refreshToken)
    {
        var principal = GetPrincipalFromExpiredToken(refreshToken);
        var username = principal.Identity.Name;
        var user = await _userManager.FindByNameAsync(username);

        if (user == null || user.RefreshToken != refreshToken)
            return BadRequest("Invalid refresh token");

        var newJwtToken = await GenerateJwtToken(user);
        var newRefreshToken = GenerateRefreshToken();
        
        user.RefreshToken = newRefreshToken;
        await _userManager.UpdateAsync(user);

        return Ok(new
        {
            token = newJwtToken,
            refreshToken = newRefreshToken
        });
    }

    [HttpPost("revoke-token")]
    public async Task<IActionResult> RevokeToken(string refreshToken)
    {
        var user = await _userManager.Users
            .SingleOrDefaultAsync(u => u.RefreshToken == refreshToken);

        if (user == null) return BadRequest();

        user.RefreshToken = null;
        await _userManager.UpdateAsync(user);

        return NoContent();
    }
}
```

## Best Practices
1. **Token Güvenliği**
   - Güvenli token saklama
   - Kısa token ömrü
   - Token yenileme mekanizması
   - Güvenli token iletimi

2. **Kimlik Doğrulama**
   - Güçlü şifre politikaları
   - Brute force koruması
   - Hesap kilitleme
   - Şifre sıfırlama

3. **Oturum Yönetimi**
   - Güvenli oturum başlatma
   - Oturum izleme
   - Zaman aşımı kontrolü
   - Güvenli oturum sonlandırma

## Kaynaklar
- [ASP.NET Core Authentication](https://docs.microsoft.com/tr-tr/aspnet/core/security/authentication/)
- [JWT Best Practices](https://auth0.com/blog/jwt-security-best-practices/)
- [OAuth 2.0 Documentation](https://oauth.net/2/) 