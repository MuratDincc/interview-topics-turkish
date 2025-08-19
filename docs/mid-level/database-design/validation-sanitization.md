# Data Validation & Sanitization

## Giriş

Data Validation & Sanitization, veritabanına gelen verilerin güvenli, doğru ve tutarlı olmasını sağlayan kritik süreçlerdir. Mid-level geliştiriciler için bu konuları anlamak, güvenlik açıklarını önlemek ve veri kalitesini korumak için kritiktir.

## Data Validation Stratejileri

### 1. Input Validation
Gelen verilerin format, tip ve business rule'lara uygunluğunu kontrol etme.

```csharp
public interface IValidator<T>
{
    ValidationResult Validate(T entity);
}

public class UserValidator : IValidator<User>
{
    public ValidationResult Validate(User user)
    {
        var result = new ValidationResult();
        
        // Required field validation
        if (string.IsNullOrWhiteSpace(user.Username))
        {
            result.AddError("Username", "Username is required");
        }
        
        if (string.IsNullOrWhiteSpace(user.Email))
        {
            result.AddError("Email", "Email is required");
        }
        
        // Format validation
        if (!string.IsNullOrEmpty(user.Email) && !IsValidEmail(user.Email))
        {
            result.AddError("Email", "Invalid email format");
        }
        
        if (!string.IsNullOrEmpty(user.Username) && user.Username.Length < 3)
        {
            result.AddError("Username", "Username must be at least 3 characters long");
        }
        
        if (!string.IsNullOrEmpty(user.Username) && user.Username.Length > 50)
        {
            result.AddError("Username", "Username cannot exceed 50 characters");
        }
        
        // Business rule validation
        if (user.DateOfBirth.HasValue && user.DateOfBirth.Value > DateTime.Now)
        {
            result.AddError("DateOfBirth", "Date of birth cannot be in the future");
        }
        
        if (user.DateOfBirth.HasValue && user.DateOfBirth.Value < DateTime.Now.AddYears(-120))
        {
            result.AddError("DateOfBirth", "Date of birth seems invalid");
        }
        
        return result;
    }
    
    private bool IsValidEmail(string email)
    {
        try
        {
            var addr = new System.Net.Mail.MailAddress(email);
            return addr.Address == email;
        }
        catch
        {
            return false;
        }
    }
}

public class ValidationResult
{
    private readonly List<ValidationError> _errors = new();
    
    public bool IsValid => !_errors.Any();
    public IReadOnlyList<ValidationError> Errors => _errors.AsReadOnly();
    
    public void AddError(string field, string message)
    {
        _errors.Add(new ValidationError(field, message));
    }
    
    public void AddErrors(IEnumerable<ValidationError> errors)
    {
        _errors.AddRange(errors);
    }
}

public class ValidationError
{
    public string Field { get; }
    public string Message { get; }
    
    public ValidationError(string field, string message)
    {
        Field = field;
        Message = message;
    }
}
```

### 2. Fluent Validation
Daha okunabilir ve maintainable validation kuralları.

```csharp
public class UserFluentValidator : AbstractValidator<User>
{
    public UserFluentValidator()
    {
        RuleFor(x => x.Username)
            .NotEmpty().WithMessage("Username is required")
            .Length(3, 50).WithMessage("Username must be between 3 and 50 characters")
            .Matches(@"^[a-zA-Z0-9_]+$").WithMessage("Username can only contain letters, numbers and underscores")
            .MustAsync(BeUniqueUsername).WithMessage("Username already exists");
        
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email is required")
            .EmailAddress().WithMessage("Invalid email format")
            .MustAsync(BeUniqueEmail).WithMessage("Email already exists");
        
        RuleFor(x => x.FirstName)
            .NotEmpty().WithMessage("First name is required")
            .Length(1, 50).WithMessage("First name cannot exceed 50 characters")
            .Matches(@"^[a-zA-Z\s]+$").WithMessage("First name can only contain letters and spaces");
        
        RuleFor(x => x.LastName)
            .NotEmpty().WithMessage("Last name is required")
            .Length(1, 50).WithMessage("Last name cannot exceed 50 characters")
            .Matches(@"^[a-zA-Z\s]+$").WithMessage("Last name can only contain letters and spaces");
        
        RuleFor(x => x.DateOfBirth)
            .Must(BeValidDate).WithMessage("Invalid date of birth")
            .Must(BeNotInFuture).WithMessage("Date of birth cannot be in the future")
            .Must(BeReasonableAge).WithMessage("Date of birth seems invalid");
        
        RuleFor(x => x.PhoneNumber)
            .Matches(@"^\+?[1-9]\d{1,14}$").WithMessage("Invalid phone number format")
            .When(x => !string.IsNullOrEmpty(x.PhoneNumber));
        
        RuleFor(x => x.Password)
            .NotEmpty().WithMessage("Password is required")
            .MinimumLength(8).WithMessage("Password must be at least 8 characters long")
            .Matches(@"[A-Z]").WithMessage("Password must contain at least one uppercase letter")
            .Matches(@"[a-z]").WithMessage("Password must contain at least one lowercase letter")
            .Matches(@"[0-9]").WithMessage("Password must contain at least one number")
            .Matches(@"[^a-zA-Z0-9]").WithMessage("Password must contain at least one special character");
        
        RuleFor(x => x.ConfirmPassword)
            .Equal(x => x.Password).WithMessage("Passwords do not match");
    }
    
    private async Task<bool> BeUniqueUsername(string username, CancellationToken cancellationToken)
    {
        // Implementation to check if username is unique in database
        return await Task.FromResult(true); // Placeholder
    }
    
    private async Task<bool> BeUniqueEmail(string email, CancellationToken cancellationToken)
    {
        // Implementation to check if email is unique in database
        return await Task.FromResult(true); // Placeholder
    }
    
    private bool BeValidDate(DateTime? date)
    {
        return !date.HasValue || date.Value != DateTime.MinValue;
    }
    
    private bool BeNotInFuture(DateTime? date)
    {
        return !date.HasValue || date.Value <= DateTime.Now;
    }
    
    private bool BeReasonableAge(DateTime? date)
    {
        if (!date.HasValue) return true;
        var age = DateTime.Now.Year - date.Value.Year;
        return age >= 0 && age <= 120;
    }
}
```

### 3. Custom Validation Attributes
ASP.NET Core için custom validation attribute'ları.

```csharp
[AttributeUsage(AttributeTargets.Property)]
public class StrongPasswordAttribute : ValidationAttribute
{
    protected override ValidationResult IsValid(object value, ValidationContext validationContext)
    {
        if (value == null)
        {
            return ValidationResult.Success;
        }
        
        var password = value.ToString();
        
        if (string.IsNullOrEmpty(password))
        {
            return ValidationResult.Success;
        }
        
        var errors = new List<string>();
        
        if (password.Length < 8)
        {
            errors.Add("Password must be at least 8 characters long");
        }
        
        if (!password.Any(char.IsUpper))
        {
            errors.Add("Password must contain at least one uppercase letter");
        }
        
        if (!password.Any(char.IsLower))
        {
            errors.Add("Password must contain at least one lowercase letter");
        }
        
        if (!password.Any(char.IsDigit))
        {
            errors.Add("Password must contain at least one number");
        }
        
        if (!password.Any(c => !char.IsLetterOrDigit(c)))
        {
            errors.Add("Password must contain at least one special character");
        }
        
        if (errors.Any())
        {
            return new ValidationResult(string.Join("; ", errors));
        }
        
        return ValidationResult.Success;
    }
}

[AttributeUsage(AttributeTargets.Property)]
public class NoSpecialCharactersAttribute : ValidationAttribute
{
    private readonly string _pattern;
    
    public NoSpecialCharactersAttribute(string pattern = @"^[a-zA-Z0-9\s]+$")
    {
        _pattern = pattern;
    }
    
    protected override ValidationResult IsValid(object value, ValidationContext validationContext)
    {
        if (value == null)
        {
            return ValidationResult.Success;
        }
        
        var stringValue = value.ToString();
        
        if (string.IsNullOrEmpty(stringValue))
        {
            return ValidationResult.Success;
        }
        
        if (!Regex.IsMatch(stringValue, _pattern))
        {
            return new ValidationResult($"Field contains invalid characters. Only letters, numbers and spaces are allowed.");
        }
        
        return ValidationResult.Success;
    }
}

[AttributeUsage(AttributeTargets.Property)]
public class FutureDateAttribute : ValidationAttribute
{
    protected override ValidationResult IsValid(object value, ValidationContext validationContext)
    {
        if (value == null)
        {
            return ValidationResult.Success;
        }
        
        if (value is DateTime date)
        {
            if (date <= DateTime.Now)
            {
                return new ValidationResult("Date must be in the future");
            }
        }
        
        return ValidationResult.Success;
    }
}
```

## Data Sanitization Stratejileri

### 1. HTML Sanitization
HTML içeriğini güvenli hale getirme.

```csharp
public class HtmlSanitizer
{
    private readonly HtmlSanitizer _sanitizer;
    
    public HtmlSanitizer()
    {
        _sanitizer = new HtmlSanitizer();
        
        // Allow safe HTML tags
        _sanitizer.AllowedTags.Add("div");
        _sanitizer.AllowedTags.Add("span");
        _sanitizer.AllowedTags.Add("p");
        _sanitizer.AllowedTags.Add("br");
        _sanitizer.AllowedTags.Add("strong");
        _sanitizer.AllowedTags.Add("em");
        _sanitizer.AllowedTags.Add("ul");
        _sanitizer.AllowedTags.Add("ol");
        _sanitizer.AllowedTags.Add("li");
        
        // Allow safe CSS properties
        _sanitizer.AllowedCssProperties.Add("color");
        _sanitizer.AllowedCssProperties.Add("background-color");
        _sanitizer.AllowedCssProperties.Add("font-size");
        _sanitizer.AllowedCssProperties.Add("text-align");
        
        // Allow safe attributes
        _sanitizer.AllowedAttributes.Add("class");
        _sanitizer.AllowedAttributes.Add("id");
        _sanitizer.AllowedAttributes.Add("style");
    }
    
    public string Sanitize(string html)
    {
        if (string.IsNullOrEmpty(html))
        {
            return html;
        }
        
        return _sanitizer.Sanitize(html);
    }
    
    public string SanitizeWithCustomRules(string html, IEnumerable<string> allowedTags, IEnumerable<string> allowedAttributes)
    {
        var customSanitizer = new HtmlSanitizer();
        
        foreach (var tag in allowedTags)
        {
            customSanitizer.AllowedTags.Add(tag);
        }
        
        foreach (var attribute in allowedAttributes)
        {
            customSanitizer.AllowedAttributes.Add(attribute);
        }
        
        return customSanitizer.Sanitize(html);
    }
}

public class ContentSanitizationService
{
    private readonly HtmlSanitizer _htmlSanitizer;
    private readonly ILogger<ContentSanitizationService> _logger;
    
    public ContentSanitizationService(ILogger<ContentSanitizationService> logger)
    {
        _htmlSanitizer = new HtmlSanitizer();
        _logger = logger;
    }
    
    public async Task<string> SanitizeUserContentAsync(string content, ContentType contentType)
    {
        try
        {
            switch (contentType)
            {
                case ContentType.Html:
                    return _htmlSanitizer.Sanitize(content);
                
                case ContentType.PlainText:
                    return SanitizePlainText(content);
                
                case ContentType.Markdown:
                    return await SanitizeMarkdownAsync(content);
                
                default:
                    return content;
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error sanitizing content of type {ContentType}", contentType);
            return string.Empty; // Return empty string on error
        }
    }
    
    private string SanitizePlainText(string text)
    {
        if (string.IsNullOrEmpty(text))
        {
            return text;
        }
        
        // Remove control characters except newlines and tabs
        var sanitized = Regex.Replace(text, @"[\x00-\x08\x0B\x0C\x0E-\x1F\x7F]", "");
        
        // Limit length
        if (sanitized.Length > 10000)
        {
            sanitized = sanitized.Substring(0, 10000);
        }
        
        return sanitized;
    }
    
    private async Task<string> SanitizeMarkdownAsync(string markdown)
    {
        if (string.IsNullOrEmpty(markdown))
        {
            return markdown;
        }
        
        // Convert markdown to HTML first
        var html = await ConvertMarkdownToHtmlAsync(markdown);
        
        // Then sanitize the HTML
        return _htmlSanitizer.Sanitize(html);
    }
    
    private async Task<string> ConvertMarkdownToHtmlAsync(string markdown)
    {
        // Implementation using a markdown library like Markdig
        return await Task.FromResult(markdown); // Placeholder
    }
}

public enum ContentType
{
    PlainText,
    Html,
    Markdown,
    Json,
    Xml
}
```

### 2. SQL Injection Prevention
SQL injection saldırılarını önleme.

```csharp
public class SafeSqlBuilder
{
    private readonly Dictionary<string, object> _parameters = new();
    private int _parameterCounter = 0;
    
    public SafeSqlBuilder AddParameter(object value)
    {
        var parameterName = $"@p{_parameterCounter++}";
        _parameters[parameterName] = value;
        return this;
    }
    
    public SafeSqlBuilder AddParameter(string name, object value)
    {
        _parameters[name] = value;
        return this;
    }
    
    public (string sql, Dictionary<string, object> parameters) Build(string sqlTemplate)
    {
        // Replace placeholders with parameter names
        var sql = sqlTemplate;
        var parameters = new Dictionary<string, object>(_parameters);
        
        return (sql, parameters);
    }
}

public class SecureUserRepository
{
    private readonly IDbConnection _connection;
    private readonly ILogger<SecureUserRepository> _logger;
    
    public SecureUserRepository(IDbConnection connection, ILogger<SecureUserRepository> logger)
    {
        _connection = connection;
        _logger = logger;
    }
    
    public async Task<User> GetUserByUsernameAsync(string username)
    {
        try
        {
            // Use parameterized query to prevent SQL injection
            var sql = "SELECT * FROM Users WHERE Username = @Username";
            var parameters = new { Username = username };
            
            return await _connection.QueryFirstOrDefaultAsync<User>(sql, parameters);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error retrieving user by username: {Username}", username);
            throw;
        }
    }
    
    public async Task<List<User>> SearchUsersAsync(string searchTerm, int page, int pageSize)
    {
        try
        {
            // Use parameterized query with LIKE for search
            var sql = @"
                SELECT * FROM Users 
                WHERE Username LIKE @SearchTerm 
                   OR Email LIKE @SearchTerm 
                   OR FirstName LIKE @SearchTerm 
                   OR LastName LIKE @SearchTerm
                ORDER BY Username
                OFFSET @Offset ROWS
                FETCH NEXT @PageSize ROWS ONLY
            ";
            
            var parameters = new
            {
                SearchTerm = $"%{searchTerm}%",
                Offset = (page - 1) * pageSize,
                PageSize = pageSize
            };
            
            return (await _connection.QueryAsync<User>(sql, parameters)).ToList();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error searching users with term: {SearchTerm}", searchTerm);
            throw;
        }
    }
    
    public async Task<bool> UpdateUserAsync(User user)
    {
        try
        {
            // Use parameterized query for update
            var sql = @"
                UPDATE Users 
                SET Username = @Username,
                    Email = @Email,
                    FirstName = @FirstName,
                    LastName = @LastName,
                    UpdatedAt = @UpdatedAt
                WHERE Id = @Id
            ";
            
            var parameters = new
            {
                user.Id,
                user.Username,
                user.Email,
                user.FirstName,
                user.LastName,
                UpdatedAt = DateTime.UtcNow
            };
            
            var rowsAffected = await _connection.ExecuteAsync(sql, parameters);
            return rowsAffected > 0;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error updating user: {UserId}", user.Id);
            throw;
        }
    }
}
```

### 3. XSS Prevention
Cross-site scripting saldırılarını önleme.

```csharp
public class XssPreventionService
{
    private readonly ILogger<XssPreventionService> _logger;
    
    public XssPreventionService(ILogger<XssPreventionService> logger)
    {
        _logger = logger;
    }
    
    public string EncodeHtml(string input)
    {
        if (string.IsNullOrEmpty(input))
        {
            return input;
        }
        
        return HttpUtility.HtmlEncode(input);
    }
    
    public string EncodeJavaScript(string input)
    {
        if (string.IsNullOrEmpty(input))
        {
            return input;
        }
        
        return HttpUtility.JavaScriptStringEncode(input);
    }
    
    public string EncodeUrl(string input)
    {
        if (string.IsNullOrEmpty(input))
        {
            return input;
        }
        
        return HttpUtility.UrlEncode(input);
    }
    
    public string EncodeAttribute(string input)
    {
        if (string.IsNullOrEmpty(input))
        {
            return input;
        }
        
        // Encode for HTML attributes
        return input.Replace("\"", "&quot;")
                   .Replace("'", "&#39;")
                   .Replace("<", "&lt;")
                   .Replace(">", "&gt;")
                   .Replace("&", "&amp;");
    }
    
    public bool ContainsXssPatterns(string input)
    {
        if (string.IsNullOrEmpty(input))
        {
            return false;
        }
        
        var xssPatterns = new[]
        {
            @"<script[^>]*>.*?</script>",
            @"javascript:",
            @"vbscript:",
            @"onload\s*=",
            @"onerror\s*=",
            @"onclick\s*=",
            @"onmouseover\s*=",
            @"<iframe[^>]*>",
            @"<object[^>]*>",
            @"<embed[^>]*>"
        };
        
        return xssPatterns.Any(pattern => Regex.IsMatch(input, pattern, RegexOptions.IgnoreCase));
    }
    
    public string SanitizeForDisplay(string input, bool allowHtml = false)
    {
        if (string.IsNullOrEmpty(input))
        {
            return input;
        }
        
        if (ContainsXssPatterns(input))
        {
            _logger.LogWarning("XSS pattern detected in input: {Input}", input);
            
            if (allowHtml)
            {
                // Use HTML sanitizer for allowed HTML
                var sanitizer = new HtmlSanitizer();
                return sanitizer.Sanitize(input);
            }
            else
            {
                // Encode everything
                return EncodeHtml(input);
            }
        }
        
        return allowHtml ? input : EncodeHtml(input);
    }
}
```

## Validation Pipeline

### 1. Validation Pipeline Implementation
```csharp
public class ValidationPipeline<T>
{
    private readonly List<IValidator<T>> _validators;
    private readonly ILogger<ValidationPipeline<T>> _logger;
    
    public ValidationPipeline(ILogger<ValidationPipeline<T>> logger)
    {
        _validators = new List<IValidator<T>>();
        _logger = logger;
    }
    
    public ValidationPipeline<T> AddValidator(IValidator<T> validator)
    {
        _validators.Add(validator);
        return this;
    }
    
    public async Task<ValidationResult> ValidateAsync(T entity)
    {
        var result = new ValidationResult();
        
        foreach (var validator in _validators)
        {
            try
            {
                var validationResult = validator.Validate(entity);
                result.AddErrors(validationResult.Errors);
                
                if (!validationResult.IsValid)
                {
                    _logger.LogWarning("Validation failed for entity {EntityType}: {Errors}", 
                        typeof(T).Name, string.Join(", ", validationResult.Errors.Select(e => $"{e.Field}: {e.Message}")));
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error during validation with validator {ValidatorType}", validator.GetType().Name);
                result.AddError("Validation", "Validation error occurred");
            }
        }
        
        return result;
    }
}

public class UserValidationPipeline
{
    private readonly ValidationPipeline<User> _pipeline;
    
    public UserValidationPipeline(ILogger<UserValidationPipeline> logger)
    {
        _pipeline = new ValidationPipeline<User>(logger)
            .AddValidator(new UserValidator())
            .AddValidator(new UserBusinessRuleValidator())
            .AddValidator(new UserSecurityValidator());
    }
    
    public async Task<ValidationResult> ValidateUserAsync(User user)
    {
        return await _pipeline.ValidateAsync(user);
    }
}

public class UserBusinessRuleValidator : IValidator<User>
{
    public ValidationResult Validate(User user)
    {
        var result = new ValidationResult();
        
        // Business rule: Users under 13 cannot register
        if (user.DateOfBirth.HasValue && user.DateOfBirth.Value > DateTime.Now.AddYears(-13))
        {
            result.AddError("DateOfBirth", "Users must be at least 13 years old to register");
        }
        
        // Business rule: Premium users must have valid payment method
        if (user.IsPremium && string.IsNullOrEmpty(user.PaymentMethodId))
        {
            result.AddError("PaymentMethodId", "Premium users must have a valid payment method");
        }
        
        return result;
    }
}

public class UserSecurityValidator : IValidator<User>
{
    public ValidationResult Validate(User user)
    {
        var result = new ValidationResult();
        
        // Security: Check for suspicious patterns
        if (ContainsSuspiciousPatterns(user.Username))
        {
            result.AddError("Username", "Username contains suspicious patterns");
        }
        
        if (ContainsSuspiciousPatterns(user.Email))
        {
            result.AddError("Email", "Email contains suspicious patterns");
        }
        
        return result;
    }
    
    private bool ContainsSuspiciousPatterns(string input)
    {
        if (string.IsNullOrEmpty(input))
        {
            return false;
        }
        
        var suspiciousPatterns = new[]
        {
            @"admin",
            @"root",
            @"system",
            @"test",
            @"guest",
            @"anonymous"
        };
        
        return suspiciousPatterns.Any(pattern => 
            Regex.IsMatch(input, pattern, RegexOptions.IgnoreCase));
    }
}
```

## Mülakat Soruları

### Temel Sorular

1. **Data validation nedir ve neden önemlidir?**
   - **Cevap**: Gelen verilerin doğruluğunu ve uygunluğunu kontrol etme süreci. Data integrity, security ve business rule compliance için kritik.

2. **Data sanitization nedir ve hangi türlerde yapılır?**
   - **Cevap**: Veriyi güvenli hale getirme süreci. HTML, SQL, JavaScript, URL encoding gibi türlerde yapılır.

3. **SQL injection nedir ve nasıl önlenir?**
   - **Cevap**: SQL sorgularına zararlı kod enjekte etme saldırısı. Parameterized queries, input validation, stored procedures ile önlenir.

4. **XSS nedir ve nasıl önlenir?**
   - **Cevap**: Cross-site scripting saldırısı. HTML encoding, input validation, content security policy ile önlenir.

5. **Validation pipeline nedir ve nasıl implement edilir?**
   - **Cevap**: Birden fazla validation rule'ı sırayla çalıştırma sistemi. Chain of responsibility pattern ile implement edilir.

### Teknik Sorular

1. **Fluent validation'da custom rule'lar nasıl yazılır?**
   - **Cevap**: Must() method'u ile custom validation logic, async validation için MustAsync(), cross-property validation için When().

2. **HTML sanitization'da hangi HTML tag'ları güvenli kabul edilir?**
   - **Cevap**: div, span, p, br, strong, em gibi formatting tag'ları. script, iframe, object gibi tag'lar güvenli değil.

3. **Validation error'ları nasıl handle edilir?**
   - **Cevap**: ValidationResult object'i ile error collection, field-specific error mapping, user-friendly error messages.

4. **Business rule validation nasıl implement edilir?**
   - **Cevap**: Custom validator'lar, business logic encapsulation, rule engine pattern, configuration-based rules.

5. **Performance için validation nasıl optimize edilir?**
   - **Cevap**: Early validation, caching, parallel validation, lazy validation, validation result caching.

## Best Practices

1. **Input Validation**
   - Server-side validation yapın
   - Client-side validation'ı güvenlik için kullanmayın
   - Whitelist approach kullanın
   - Business rules'ları validate edin

2. **Data Sanitization**
   - Context-aware sanitization yapın
   - HTML encoding kullanın
   - SQL injection prevention implement edin
   - XSS prevention implement edin

3. **Validation Pipeline**
   - Modular validation yapın
   - Error handling implement edin
   - Performance optimize edin
   - Logging ve monitoring ekleyin

4. **Security**
   - Defense in depth approach kullanın
   - Regular security testing yapın
   - Input validation'ı her seviyede yapın
   - Security headers kullanın

## Kaynaklar

- [Data Validation in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/mvc/models/validation)
- [FluentValidation](https://docs.fluentvalidation.net/)
- [HTML Sanitization](https://github.com/mganss/HtmlSanitizer)
- [SQL Injection Prevention](https://docs.microsoft.com/en-us/sql/relational-databases/security/sql-injection)
- [XSS Prevention](https://owasp.org/www-project-cheat-sheets/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
