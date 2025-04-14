# Input Validation

## Genel Bakış
Input Validation (Girdi Doğrulama), uygulamaya gelen tüm verilerin güvenli ve beklenen formatta olduğundan emin olmak için yapılan kontrollerdir. Bu işlem, güvenlik açıklarını önlemek ve veri bütünlüğünü korumak için kritik öneme sahiptir.

## Mülakat Soruları ve Cevapları

### 1. Input Validation neden önemlidir?
**Cevap:**
Input Validation'ın önemi:
- Güvenlik: SQL Injection, XSS gibi saldırıları önler
- Veri Bütünlüğü: Beklenen veri formatını sağlar
- Kullanıcı Deneyimi: Hatalı girişleri erken tespit eder
- Sistem Performansı: Geçersiz verilerin işlenmesini engeller

**Örnek Kod:**
```csharp
// Model doğrulama
public class UserModel
{
    [Required(ErrorMessage = "Kullanıcı adı zorunludur")]
    [StringLength(50, MinimumLength = 3, ErrorMessage = "Kullanıcı adı 3-50 karakter arasında olmalıdır")]
    [RegularExpression(@"^[a-zA-Z0-9_]+$", ErrorMessage = "Kullanıcı adı sadece harf, rakam ve alt çizgi içerebilir")]
    public string Username { get; set; }

    [Required(ErrorMessage = "E-posta zorunludur")]
    [EmailAddress(ErrorMessage = "Geçerli bir e-posta adresi giriniz")]
    public string Email { get; set; }
}

// Controller'da doğrulama
[HttpPost]
public IActionResult CreateUser(UserModel model)
{
    if (!ModelState.IsValid)
    {
        return BadRequest(ModelState);
    }
    // İşlem devam eder...
}
```

### 2. Input Validation yöntemleri nelerdir?
**Cevap:**
Input Validation yöntemleri:
- Client-side validation
- Server-side validation
- Model validation
- Custom validation
- Sanitization

**Örnek Kod:**
```csharp
// Custom validation attribute
public class AgeValidationAttribute : ValidationAttribute
{
    protected override ValidationResult IsValid(object value, ValidationContext validationContext)
    {
        if (value is int age)
        {
            if (age < 18)
            {
                return new ValidationResult("Yaş 18'den küçük olamaz");
            }
            if (age > 120)
            {
                return new ValidationResult("Geçerli bir yaş giriniz");
            }
        }
        return ValidationResult.Success;
    }
}

// Sanitization örneği
public class InputSanitizer
{
    public string SanitizeInput(string input)
    {
        if (string.IsNullOrEmpty(input))
            return input;

        // HTML encoding
        input = WebUtility.HtmlEncode(input);
        
        // SQL injection koruması
        input = input.Replace("'", "''");
        
        // XSS koruması
        input = Regex.Replace(input, "<script.*?>.*?</script>", "", RegexOptions.IgnoreCase);
        
        return input;
    }
}
```

### 3. SQL Injection nasıl önlenir?
**Cevap:**
SQL Injection önleme yöntemleri:
- Parametreli sorgular
- Stored procedures
- ORM kullanımı
- Input sanitization
- Minimum yetki prensibi

**Örnek Kod:**
```csharp
// Güvenli olmayan kod
public IActionResult GetUser(string username)
{
    var query = $"SELECT * FROM Users WHERE Username = '{username}'";
    // SQL Injection riski!
}

// Güvenli kod (Entity Framework)
public IActionResult GetUser(string username)
{
    var user = _context.Users
        .Where(u => u.Username == username)
        .FirstOrDefault();
    return Ok(user);
}

// Güvenli kod (Dapper)
public IActionResult GetUser(string username)
{
    var query = "SELECT * FROM Users WHERE Username = @Username";
    var parameters = new { Username = username };
    var user = _connection.QueryFirstOrDefault<User>(query, parameters);
    return Ok(user);
}
```

### 4. XSS (Cross-Site Scripting) nasıl önlenir?
**Cevap:**
XSS önleme yöntemleri:
- Output encoding
- Content Security Policy
- Input validation
- Sanitization
- HttpOnly cookies

**Örnek Kod:**
```csharp
// XSS koruması
public class XssProtectionMiddleware
{
    public async Task InvokeAsync(HttpContext context)
    {
        context.Response.Headers.Add("X-XSS-Protection", "1; mode=block");
        context.Response.Headers.Add("Content-Security-Policy", 
            "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval';");
        
        await _next(context);
    }
}

// Output encoding
public class HtmlEncoder
{
    public string EncodeOutput(string input)
    {
        if (string.IsNullOrEmpty(input))
            return input;

        return WebUtility.HtmlEncode(input);
    }
}

// View'da encoding
@Html.Raw(Model.Content) // Güvenli değil
@Html.Encode(Model.Content) // Güvenli
```

### 5. File Upload güvenliği nasıl sağlanır?
**Cevap:**
File Upload güvenliği için:
- Dosya tipi kontrolü
- Dosya boyutu sınırlaması
- Güvenli dosya isimlendirme
- Virüs taraması
- Güvenli depolama

**Örnek Kod:**
```csharp
// Güvenli dosya yükleme
public class FileUploadService
{
    private readonly string[] _allowedExtensions = { ".jpg", ".png", ".pdf" };
    private readonly long _maxFileSize = 5 * 1024 * 1024; // 5MB

    public async Task<string> UploadFile(IFormFile file)
    {
        // Dosya tipi kontrolü
        var extension = Path.GetExtension(file.FileName).ToLowerInvariant();
        if (!_allowedExtensions.Contains(extension))
        {
            throw new InvalidOperationException("Geçersiz dosya tipi");
        }

        // Dosya boyutu kontrolü
        if (file.Length > _maxFileSize)
        {
            throw new InvalidOperationException("Dosya boyutu çok büyük");
        }

        // Güvenli dosya ismi oluşturma
        var fileName = $"{Guid.NewGuid()}{extension}";
        var filePath = Path.Combine("uploads", fileName);

        // Dosyayı güvenli şekilde kaydetme
        using (var stream = new FileStream(filePath, FileMode.Create))
        {
            await file.CopyToAsync(stream);
        }

        return fileName;
    }
}
```

## Best Practices
1. **Güvenlik**
   - Tüm girdileri doğrulayın
   - Whitelist yaklaşımı kullanın
   - Minimum yetki prensibini uygulayın
   - Hata mesajlarını güvenli hale getirin

2. **Performans**
   - Doğrulamayı erken yapın
   - Gereksiz kontrollerden kaçının
   - Önbellek kullanın
   - Batch işlemleri optimize edin

3. **Kullanılabilirlik**
   - Anlamlı hata mesajları verin
   - Client-side validation kullanın
   - Kullanıcı dostu geri bildirimler sağlayın
   - Form tasarımını iyileştirin

## Kaynaklar
- [OWASP Input Validation](https://owasp.org/www-community/Input_Validation_Cheat_Sheet)
- [ASP.NET Core Validation](https://docs.microsoft.com/tr-tr/aspnet/core/mvc/models/validation)
- [XSS Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html) 