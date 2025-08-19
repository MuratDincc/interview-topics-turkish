# Web Security Basics

## Giriş

Web güvenliği, modern web uygulamalarının en kritik konularından biridir. Backend geliştiriciler olarak güvenlik açıklarını anlamak ve önlemek, uygulamanızı korumak için gereklidir.

## Temel Güvenlik Tehditleri

### 1. XSS (Cross-Site Scripting)
XSS, saldırganın web sayfasına zararlı JavaScript kodu enjekte etmesiyle oluşan güvenlik açığıdır.

#### XSS Türleri
- **Stored XSS**: Zararlı kod veritabanında saklanır
- **Reflected XSS**: Zararlı kod URL'de yansıtılır
- **DOM-based XSS**: Client-side JavaScript'te oluşur

#### XSS Örneği
```html
<!-- Güvensiz kod -->
<div id="user-input">
    <script>alert('XSS Attack!')</script>
</div>

<!-- Güvenli kod -->
<div id="user-input">
    <!-- HTML encoding yapılmış -->
    &lt;script&gt;alert('XSS Attack!')&lt;/script&gt;
</div>
```

#### XSS Önleme
```csharp
// ASP.NET Core'da HTML encoding
@Html.Encode(userInput)

// JavaScript'te
function escapeHtml(text) {
    const div = document.createElement('div');
    div.textContent = text;
    return div.innerHTML;
}
```

### 2. CSRF (Cross-Site Request Forgery)
CSRF, saldırganın kullanıcının kimlik bilgileriyle istenmeyen işlemler yapmasını sağlayan saldırıdır.

#### CSRF Örneği
```html
<!-- Saldırganın sayfası -->
<img src="http://bank.com/transfer?amount=1000&to=attacker" style="display:none">
```

#### CSRF Önleme
```csharp
// ASP.NET Core'da AntiForgeryToken
<form asp-action="Transfer" method="post">
    @Html.AntiForgeryToken()
    <input type="number" name="amount" />
    <input type="text" name="to" />
    <button type="submit">Transfer</button>
</form>

// Controller'da validation
[ValidateAntiForgeryToken]
public IActionResult Transfer(TransferModel model)
{
    // Transfer işlemi
}
```

### 3. SQL Injection
SQL Injection, saldırganın veritabanı sorgularını manipüle etmesini sağlayan saldırıdır.

#### SQL Injection Örneği
```sql
-- Güvensiz sorgu
SELECT * FROM Users WHERE Username = 'admin' OR '1'='1' --' AND Password = 'password'

-- Güvenli sorgu (parametre kullanımı)
SELECT * FROM Users WHERE Username = @Username AND Password = @Password
```

#### SQL Injection Önleme
```csharp
// Entity Framework kullanımı (güvenli)
var user = await _context.Users
    .FirstOrDefaultAsync(u => u.Username == username && u.Password == password);

// Raw SQL kullanımında parametre
var sql = "SELECT * FROM Users WHERE Username = @Username AND Password = @Password";
var user = await _context.Users
    .FromSqlRaw(sql, new SqlParameter("@Username", username), new SqlParameter("@Password", password))
    .FirstOrDefaultAsync();
```

### 4. Input Validation
Kullanıcı girdilerinin doğrulanması güvenlik için kritiktir.

#### Input Validation Örnekleri
```csharp
// Model validation
public class UserModel
{
    [Required]
    [StringLength(50, MinimumLength = 3)]
    [RegularExpression(@"^[a-zA-Z0-9_]+$", ErrorMessage = "Sadece harf, rakam ve alt çizgi kullanılabilir")]
    public string Username { get; set; }

    [Required]
    [EmailAddress]
    public string Email { get; set; }

    [Required]
    [StringLength(100, MinimumLength = 8)]
    [RegularExpression(@"^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]", 
        ErrorMessage = "Şifre en az bir büyük harf, küçük harf, rakam ve özel karakter içermelidir")]
    public string Password { get; set; }
}

// Controller'da validation
[HttpPost]
public async Task<IActionResult> CreateUser(UserModel model)
{
    if (!ModelState.IsValid)
    {
        return BadRequest(ModelState);
    }
    
    // Güvenli işlem
}
```

## HTTPS ve SSL/TLS

### HTTPS Nedir?
HTTPS, HTTP protokolünün SSL/TLS şifrelemesi ile güvenli hale getirilmiş versiyonudur.

### SSL/TLS Avantajları
- **Encryption**: Veri şifreleme
- **Authentication**: Sunucu kimlik doğrulama
- **Integrity**: Veri bütünlüğü

### HTTPS Yapılandırması
```csharp
// Program.cs veya Startup.cs
var builder = WebApplication.CreateBuilder(args);

// HTTPS yönlendirme
builder.Services.AddHttpsRedirection(options =>
{
    options.RedirectStatusCode = StatusCodes.Status307TemporaryRedirect;
    options.HttpsPort = 443;
});

var app = builder.Build();

// HTTPS middleware
app.UseHttpsRedirection();
```

## Security Headers

### Güvenlik Başlıkları
```csharp
// Program.cs'de middleware ekleme
app.Use(async (context, next) =>
{
    // Security headers
    context.Response.Headers.Add("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Add("X-Frame-Options", "DENY");
    context.Response.Headers.Add("X-XSS-Protection", "1; mode=block");
    context.Response.Headers.Add("Referrer-Policy", "strict-origin-when-cross-origin");
    context.Response.Headers.Add("Content-Security-Policy", "default-src 'self'");
    
    await next();
});
```

### Security Headers Açıklamaları
- **X-Content-Type-Options**: MIME type sniffing'i önler
- **X-Frame-Options**: Clickjacking saldırılarını önler
- **X-XSS-Protection**: XSS koruması (eski tarayıcılar için)
- **Content-Security-Policy**: Kaynak yükleme politikası

## Authentication ve Authorization

### Authentication
```csharp
// JWT Token oluşturma
public class JwtService
{
    private readonly IConfiguration _configuration;
    
    public JwtService(IConfiguration configuration)
    {
        _configuration = configuration;
    }
    
    public string GenerateToken(User user)
    {
        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
            new Claim(ClaimTypes.Name, user.Username),
            new Claim(ClaimTypes.Email, user.Email),
            new Claim(ClaimTypes.Role, user.Role)
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
}
```

### Authorization
```csharp
// Role-based authorization
[Authorize(Roles = "Admin")]
public IActionResult AdminPanel()
{
    return View();
}

// Policy-based authorization
[Authorize(Policy = "MinimumAge")]
public IActionResult AgeRestrictedContent()
{
    return View();
}

// Custom policy
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("MinimumAge", policy =>
        policy.RequireAssertion(context =>
            context.User.HasClaim(c => c.Type == "Age" && 
                int.TryParse(c.Value, out int age) && age >= 18)));
});
```

## Mülakat Soruları

### Temel Sorular

1. **XSS nedir ve nasıl önlenir?**
   - **Cevap**: Cross-Site Scripting, zararlı JavaScript kodu enjekte etme. HTML encoding, input validation ve CSP ile önlenir.

2. **CSRF nedir ve nasıl önlenir?**
   - **Cevap**: Cross-Site Request Forgery, kullanıcının kimlik bilgileriyle istenmeyen işlemler. AntiForgeryToken ile önlenir.

3. **SQL Injection nedir ve nasıl önlenir?**
   - **Cevap**: Veritabanı sorgularını manipüle etme. Parametre kullanımı ve ORM kullanarak önlenir.

4. **HTTPS neden önemlidir?**
   - **Cevap**: Veri şifreleme, kimlik doğrulama ve bütünlük sağlar. Man-in-the-middle saldırılarını önler.

5. **Input validation neden önemlidir?**
   - **Cevap**: Zararlı veri girişini önler, güvenlik açıklarını kapatır ve veri kalitesini artırır.

### Teknik Sorular

1. **Content Security Policy nedir?**
   - **Cevap**: Hangi kaynakların yüklenebileceğini belirleyen güvenlik politikası. XSS ve injection saldırılarını önler.

2. **JWT token'ların güvenlik riskleri nelerdir?**
   - **Cevap**: Token çalınması, XSS ile token erişimi, token expiration. Secure storage ve HTTPS kullanımı gerekli.

3. **Rate limiting nedir ve nasıl uygulanır?**
   - **Cevap**: API çağrı sayısını sınırlama. Brute force saldırıları önler. Middleware ile uygulanır.

4. **Session hijacking nedir?**
   - **Cevap**: Kullanıcının session bilgilerinin çalınması. HTTPS, secure cookies ve session timeout ile önlenir.

5. **OWASP Top 10 nedir?**
   - **Cevap**: Web uygulamalarındaki en kritik 10 güvenlik açığı. Injection, XSS, broken authentication gibi.

## Best Practices

1. **Input Validation**
   - Tüm kullanıcı girdilerini validate edin
   - Whitelist yaklaşımı kullanın
   - Server-side validation yapın

2. **Output Encoding**
   - HTML, JavaScript ve CSS encoding yapın
   - Context-aware encoding kullanın
   - Framework'ün built-in encoding'ini kullanın

3. **Authentication**
   - Güçlü şifre politikaları uygulayın
   - Multi-factor authentication kullanın
   - Session timeout ayarlayın

4. **Authorization**
   - Principle of least privilege uygulayın
   - Role-based access control kullanın
   - Resource-level authorization yapın

5. **Monitoring ve Logging**
   - Güvenlik olaylarını loglayın
   - Anormal aktiviteleri izleyin
   - Regular security audits yapın

## Kaynaklar

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [Microsoft Security Documentation](https://docs.microsoft.com/en-us/aspnet/core/security/)
- [Security Headers](https://securityheaders.com/)
- [Mozilla Security Guidelines](https://developer.mozilla.org/en-US/docs/Web/Security) 