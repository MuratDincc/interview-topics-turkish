# API Security

## Giriş

API Security, modern web uygulamalarında en kritik güvenlik konularından biridir. Mid-level geliştiriciler için API güvenliğini anlamak, güvenli ve güvenilir sistemler tasarlamada esastır. Bu dosya, API güvenlik tehditlerini, korunma yöntemlerini ve best practice'leri kapsar.

## API Security Threats

### 1. Authentication Bypass
API'lerde authentication mekanizmalarının bypass edilmesi.

```csharp
public class AuthenticationBypassProtection
{
    private readonly IUserService _userService;
    private readonly ILogger<AuthenticationBypassProtection> _logger;
    
    public AuthenticationBypassProtection(
        IUserService userService,
        ILogger<AuthenticationBypassProtection> logger)
    {
        _userService = userService;
        _logger = logger;
    }
    
    public async Task<bool> ValidateAuthenticationAsync(HttpContext context)
    {
        try
        {
            // Check if user is authenticated
            if (!context.User.Identity.IsAuthenticated)
            {
                _logger.LogWarning("Unauthenticated access attempt to {Path}", context.Request.Path);
                return false;
            }
            
            // Validate JWT token if present
            var token = ExtractTokenFromContext(context);
            if (!string.IsNullOrEmpty(token))
            {
                var isValid = await ValidateJwtTokenAsync(token);
                if (!isValid)
                {
                    _logger.LogWarning("Invalid JWT token for user {UserId}", context.User.Identity.Name);
                    return false;
                }
            }
            
            // Check if user exists and is active
            var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
            if (!string.IsNullOrEmpty(userId))
            {
                var user = await _userService.GetUserByIdAsync(userId);
                if (user == null || !user.IsActive)
                {
                    _logger.LogWarning("Inactive or non-existent user {UserId} attempting access", userId);
                    return false;
                }
            }
            
            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error validating authentication");
            return false;
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
    
    private async Task<bool> ValidateJwtTokenAsync(string token)
    {
        try
        {
            // Implement JWT validation logic
            // This is a simplified example
            return !string.IsNullOrEmpty(token) && token.Length > 10;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error validating JWT token");
            return false;
        }
    }
}
```

### 2. Authorization Vulnerabilities
Kullanıcıların yetkileri dışındaki kaynaklara erişim sağlaması.

```csharp
public class AuthorizationService
{
    private readonly IUserService _userService;
    private readonly IRoleService _roleService;
    private readonly IPermissionService _permissionService;
    private readonly ILogger<AuthorizationService> _logger;
    
    public AuthorizationService(
        IUserService userService,
        IRoleService roleService,
        IPermissionService permissionService,
        ILogger<AuthorizationService> logger)
    {
        _userService = userService;
        _roleService = roleService;
        _permissionService = permissionService;
        _logger = logger;
    }
    
    public async Task<bool> AuthorizeAsync(string userId, string resource, string action)
    {
        try
        {
            // Get user permissions
            var userPermissions = await GetUserPermissionsAsync(userId);
            
            // Check if user has permission for the specific resource and action
            var hasPermission = userPermissions.Any(p => 
                p.Resource == resource && 
                p.Action == action && 
                p.IsGranted);
            
            if (!hasPermission)
            {
                _logger.LogWarning("User {UserId} denied access to {Resource} with action {Action}", 
                    userId, resource, action);
            }
            
            return hasPermission;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error authorizing user {UserId} for {Resource} {Action}", 
                userId, resource, action);
            return false; // Fail secure
        }
    }
    
    public async Task<bool> AuthorizeResourceAccessAsync(string userId, string resourceId, string resourceType)
    {
        try
        {
            // Get user's resource access permissions
            var resourcePermissions = await GetResourcePermissionsAsync(userId, resourceType);
            
            // Check if user owns the resource or has admin access
            var hasAccess = resourcePermissions.Any(p => 
                (p.ResourceId == resourceId && p.IsOwner) || 
                (p.ResourceType == resourceType && p.Action == "admin"));
            
            if (!hasAccess)
            {
                _logger.LogWarning("User {UserId} denied access to {ResourceType} {ResourceId}", 
                    userId, resourceType, resourceId);
            }
            
            return hasAccess;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error authorizing resource access for user {UserId}", userId);
            return false;
        }
    }
    
    private async Task<IEnumerable<Permission>> GetUserPermissionsAsync(string userId)
    {
        var user = await _userService.GetUserByIdAsync(userId);
        if (user == null) return Enumerable.Empty<Permission>();
        
        var roles = await _roleService.GetUserRolesAsync(userId);
        var permissions = new List<Permission>();
        
        foreach (var role in roles)
        {
            var rolePermissions = await _permissionService.GetRolePermissionsAsync(role.Id);
            permissions.AddRange(rolePermissions);
        }
        
        return permissions;
    }
    
    private async Task<IEnumerable<ResourcePermission>> GetResourcePermissionsAsync(string userId, string resourceType)
    {
        // Implementation for resource-specific permissions
        return await _permissionService.GetResourcePermissionsAsync(userId, resourceType);
    }
}

public class Permission
{
    public string Id { get; set; }
    public string Resource { get; set; }
    public string Action { get; set; }
    public bool IsGranted { get; set; }
}

public class ResourcePermission
{
    public string Id { get; set; }
    public string UserId { get; set; }
    public string ResourceId { get; set; }
    public string ResourceType { get; set; }
    public string Action { get; set; }
    public bool IsOwner { get; set; }
}
```

### 3. Input Validation Attacks
API'ye gönderilen verilerin doğrulanmaması sonucu oluşan güvenlik açıkları.

```csharp
public class InputValidationService
{
    private readonly ILogger<InputValidationService> _logger;
    private readonly IConfiguration _configuration;
    
    public InputValidationService(
        ILogger<InputValidationService> logger,
        IConfiguration configuration)
    {
        _logger = logger;
        _configuration = configuration;
    }
    
    public ValidationResult ValidateUserInput(UserInputModel input)
    {
        var result = new ValidationResult();
        
        try
        {
            // Validate email
            if (!string.IsNullOrEmpty(input.Email))
            {
                if (!IsValidEmail(input.Email))
                {
                    result.AddError("Email", "Invalid email format");
                }
                
                // Check for SQL injection patterns
                if (ContainsSqlInjectionPatterns(input.Email))
                {
                    result.AddError("Email", "Invalid characters detected");
                    _logger.LogWarning("Potential SQL injection attempt in email: {Email}", input.Email);
                }
            }
            
            // Validate username
            if (!string.IsNullOrEmpty(input.Username))
            {
                if (input.Username.Length < 3 || input.Username.Length > 50)
                {
                    result.AddError("Username", "Username must be between 3 and 50 characters");
                }
                
                if (!Regex.IsMatch(input.Username, @"^[a-zA-Z0-9_-]+$"))
                {
                    result.AddError("Username", "Username contains invalid characters");
                }
            }
            
            // Validate password strength
            if (!string.IsNullOrEmpty(input.Password))
            {
                var passwordValidation = ValidatePasswordStrength(input.Password);
                if (!passwordValidation.IsValid)
                {
                    result.AddError("Password", passwordValidation.ErrorMessage);
                }
            }
            
            // Validate file upload
            if (input.ProfileImage != null)
            {
                var fileValidation = ValidateFileUpload(input.ProfileImage);
                if (!fileValidation.IsValid)
                {
                    result.AddError("ProfileImage", fileValidation.ErrorMessage);
                }
            }
            
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error validating user input");
            result.AddError("General", "Validation error occurred");
            return result;
        }
    }
    
    public ValidationResult ValidateApiRequest(ApiRequestModel request)
    {
        var result = new ValidationResult();
        
        try
        {
            // Validate request size
            var requestSize = GetRequestSize(request);
            var maxRequestSize = _configuration.GetValue<int>("Api:MaxRequestSize", 10485760); // 10MB default
            
            if (requestSize > maxRequestSize)
            {
                result.AddError("Request", $"Request size exceeds maximum allowed size of {maxRequestSize} bytes");
                _logger.LogWarning("Large request detected: {Size} bytes", requestSize);
            }
            
            // Validate rate limiting
            if (!ValidateRateLimit(request.ClientId))
            {
                result.AddError("RateLimit", "Rate limit exceeded");
                _logger.LogWarning("Rate limit exceeded for client {ClientId}", request.ClientId);
            }
            
            // Validate content type
            if (!IsValidContentType(request.ContentType))
            {
                result.AddError("ContentType", "Invalid content type");
            }
            
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error validating API request");
            result.AddError("General", "Validation error occurred");
            return result;
        }
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
    
    private bool ContainsSqlInjectionPatterns(string input)
    {
        if (string.IsNullOrEmpty(input)) return false;
        
        var sqlPatterns = new[]
        {
            "SELECT", "INSERT", "UPDATE", "DELETE", "DROP", "CREATE", "ALTER",
            "EXEC", "EXECUTE", "UNION", "OR 1=1", "OR '1'='1", "--", "/*", "*/"
        };
        
        var upperInput = input.ToUpperInvariant();
        return sqlPatterns.Any(pattern => upperInput.Contains(pattern));
    }
    
    private PasswordValidationResult ValidatePasswordStrength(string password)
    {
        var result = new PasswordValidationResult();
        
        if (string.IsNullOrEmpty(password) || password.Length < 8)
        {
            result.IsValid = false;
            result.ErrorMessage = "Password must be at least 8 characters long";
            return result;
        }
        
        if (!Regex.IsMatch(password, @"[A-Z]"))
        {
            result.IsValid = false;
            result.ErrorMessage = "Password must contain at least one uppercase letter";
            return result;
        }
        
        if (!Regex.IsMatch(password, @"[a-z]"))
        {
            result.IsValid = false;
            result.ErrorMessage = "Password must contain at least one lowercase letter";
            return result;
        }
        
        if (!Regex.IsMatch(password, @"[0-9]"))
        {
            result.IsValid = false;
            result.ErrorMessage = "Password must contain at least one number";
            return result;
        }
        
        if (!Regex.IsMatch(password, @"[^A-Za-z0-9]"))
        {
            result.IsValid = false;
            result.ErrorMessage = "Password must contain at least one special character";
            return result;
        }
        
        result.IsValid = true;
        return result;
    }
    
    private FileValidationResult ValidateFileUpload(IFormFile file)
    {
        var result = new FileValidationResult();
        
        // Check file size
        var maxFileSize = _configuration.GetValue<int>("FileUpload:MaxSize", 5242880); // 5MB default
        if (file.Length > maxFileSize)
        {
            result.IsValid = false;
            result.ErrorMessage = $"File size exceeds maximum allowed size of {maxFileSize} bytes";
            return result;
        }
        
        // Check file extension
        var allowedExtensions = _configuration.GetSection("FileUpload:AllowedExtensions").Get<string[]>() 
            ?? new[] { ".jpg", ".jpeg", ".png", ".gif" };
        
        var fileExtension = Path.GetExtension(file.FileName).ToLowerInvariant();
        if (!allowedExtensions.Contains(fileExtension))
        {
            result.IsValid = false;
            result.ErrorMessage = $"File extension {fileExtension} is not allowed";
            return result;
        }
        
        // Check MIME type
        var allowedMimeTypes = _configuration.GetSection("FileUpload:AllowedMimeTypes").Get<string[]>()
            ?? new[] { "image/jpeg", "image/png", "image/gif" };
        
        if (!allowedMimeTypes.Contains(file.ContentType.ToLowerInvariant()))
        {
            result.IsValid = false;
            result.ErrorMessage = $"File type {file.ContentType} is not allowed";
            return result;
        }
        
        result.IsValid = true;
        return result;
    }
    
    private long GetRequestSize(ApiRequestModel request)
    {
        // Calculate request size based on content
        var size = 0L;
        
        if (request.Body != null)
        {
            size += request.Body.Length;
        }
        
        if (request.Headers != null)
        {
            size += request.Headers.Sum(h => h.Key.Length + h.Value.Length);
        }
        
        return size;
    }
    
    private bool ValidateRateLimit(string clientId)
    {
        // Implement rate limiting logic
        // This is a simplified example
        return true;
    }
    
    private bool IsValidContentType(string contentType)
    {
        var validTypes = new[]
        {
            "application/json",
            "application/xml",
            "application/x-www-form-urlencoded",
            "multipart/form-data"
        };
        
        return validTypes.Contains(contentType?.ToLowerInvariant());
    }
}

public class ValidationResult
{
    public List<ValidationError> Errors { get; } = new List<ValidationError>();
    public bool IsValid => !Errors.Any();
    
    public void AddError(string field, string message)
    {
        Errors.Add(new ValidationError { Field = field, Message = message });
    }
}

public class ValidationError
{
    public string Field { get; set; }
    public string Message { get; set; }
}

public class PasswordValidationResult
{
    public bool IsValid { get; set; }
    public string ErrorMessage { get; set; }
}

public class FileValidationResult
{
    public bool IsValid { get; set; }
    public string ErrorMessage { get; set; }
}
```

## API Security Middleware

### 1. Security Headers Middleware
Güvenlik başlıklarını otomatik olarak ekleyen middleware.

```csharp
public class SecurityHeadersMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<SecurityHeadersMiddleware> _logger;
    private readonly SecurityHeadersOptions _options;
    
    public SecurityHeadersMiddleware(
        RequestDelegate next,
        ILogger<SecurityHeadersMiddleware> logger,
        IOptions<SecurityHeadersOptions> options)
    {
        _next = next;
        _logger = logger;
        _options = options.Value;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            // Add security headers
            AddSecurityHeaders(context.Response);
            
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error in security headers middleware");
            await _next(context);
        }
    }
    
    private void AddSecurityHeaders(HttpResponse response)
    {
        // Content Security Policy
        if (_options.EnableContentSecurityPolicy)
        {
            response.Headers["Content-Security-Policy"] = _options.ContentSecurityPolicy;
        }
        
        // X-Frame-Options
        if (_options.EnableXFrameOptions)
        {
            response.Headers["X-Frame-Options"] = _options.XFrameOptions;
        }
        
        // X-Content-Type-Options
        if (_options.EnableXContentTypeOptions)
        {
            response.Headers["X-Content-Type-Options"] = "nosniff";
        }
        
        // X-XSS-Protection
        if (_options.EnableXXssProtection)
        {
            response.Headers["X-XSS-Protection"] = "1; mode=block";
        }
        
        // Referrer Policy
        if (_options.EnableReferrerPolicy)
        {
            response.Headers["Referrer-Policy"] = _options.ReferrerPolicy;
        }
        
        // Strict Transport Security
        if (_options.EnableStrictTransportSecurity)
        {
            response.Headers["Strict-Transport-Security"] = _options.StrictTransportSecurity;
        }
        
        // Permissions Policy
        if (_options.EnablePermissionsPolicy)
        {
            response.Headers["Permissions-Policy"] = _options.PermissionsPolicy;
        }
        
        // Remove server header
        if (_options.RemoveServerHeader)
        {
            response.Headers.Remove("Server");
        }
        
        // Remove X-Powered-By header
        if (_options.RemoveXPoweredByHeader)
        {
            response.Headers.Remove("X-Powered-By");
        }
    }
}

public class SecurityHeadersOptions
{
    public bool EnableContentSecurityPolicy { get; set; } = true;
    public string ContentSecurityPolicy { get; set; } = "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';";
    
    public bool EnableXFrameOptions { get; set; } = true;
    public string XFrameOptions { get; set; } = "DENY";
    
    public bool EnableXContentTypeOptions { get; set; } = true;
    public bool EnableXXssProtection { get; set; } = true;
    
    public bool EnableReferrerPolicy { get; set; } = true;
    public string ReferrerPolicy { get; set; } = "strict-origin-when-cross-origin";
    
    public bool EnableStrictTransportSecurity { get; set; } = true;
    public string StrictTransportSecurity { get; set; } = "max-age=31536000; includeSubDomains";
    
    public bool EnablePermissionsPolicy { get; set; } = true;
    public string PermissionsPolicy { get; set; } = "geolocation=(), microphone=(), camera=()";
    
    public bool RemoveServerHeader { get; set; } = true;
    public bool RemoveXPoweredByHeader { get; set; } = true;
}

public static class SecurityHeadersMiddlewareExtensions
{
    public static IApplicationBuilder UseSecurityHeaders(this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<SecurityHeadersMiddleware>();
    }
}
```

### 2. API Rate Limiting Middleware
API rate limiting implementasyonu.

```csharp
public class ApiRateLimitingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IMemoryCache _cache;
    private readonly ILogger<ApiRateLimitingMiddleware> _logger;
    private readonly RateLimitingOptions _options;
    
    public ApiRateLimitingMiddleware(
        RequestDelegate next,
        IMemoryCache cache,
        ILogger<ApiRateLimitingMiddleware> logger,
        IOptions<RateLimitingOptions> options)
    {
        _next = next;
        _cache = cache;
        _logger = logger;
        _options = options.Value;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            var clientIdentifier = GetClientIdentifier(context);
            
            if (await IsRateLimitExceededAsync(clientIdentifier))
            {
                _logger.LogWarning("Rate limit exceeded for client {ClientId}", clientIdentifier);
                
                context.Response.StatusCode = 429; // Too Many Requests
                context.Response.Headers["Retry-After"] = _options.RetryAfterSeconds.ToString();
                
                var errorResponse = new
                {
                    Error = "Rate limit exceeded",
                    RetryAfter = _options.RetryAfterSeconds,
                    Message = "Too many requests. Please try again later."
                };
                
                await context.Response.WriteAsJsonAsync(errorResponse);
                return;
            }
            
            // Add rate limit headers
            AddRateLimitHeaders(context.Response, clientIdentifier);
            
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error in rate limiting middleware");
            await _next(context);
        }
    }
    
    private string GetClientIdentifier(HttpContext context)
    {
        // Try to get client ID from various sources
        var clientId = context.User?.FindFirst("client_id")?.Value;
        
        if (!string.IsNullOrEmpty(clientId))
        {
            return clientId;
        }
        
        // Fall back to IP address
        var ipAddress = context.Connection.RemoteIpAddress?.ToString();
        if (!string.IsNullOrEmpty(ipAddress))
        {
            return $"ip_{ipAddress}";
        }
        
        // Fall back to user agent
        var userAgent = context.Request.Headers["User-Agent"].FirstOrDefault();
        if (!string.IsNullOrEmpty(userAgent))
        {
            return $"ua_{userAgent.GetHashCode()}";
        }
        
        return "unknown";
    }
    
    private async Task<bool> IsRateLimitExceededAsync(string clientIdentifier)
    {
        var cacheKey = $"rate_limit:{clientIdentifier}";
        
        if (_cache.TryGetValue(cacheKey, out RateLimitInfo rateLimitInfo))
        {
            // Check if window has expired
            if (DateTime.UtcNow >= rateLimitInfo.WindowEnd)
            {
                // Reset for new window
                rateLimitInfo = new RateLimitInfo
                {
                    RequestCount = 1,
                    WindowStart = DateTime.UtcNow,
                    WindowEnd = DateTime.UtcNow.AddSeconds(_options.WindowSizeSeconds)
                };
            }
            else
            {
                // Increment request count
                rateLimitInfo.RequestCount++;
            }
        }
        else
        {
            // First request in new window
            rateLimitInfo = new RateLimitInfo
            {
                RequestCount = 1,
                WindowStart = DateTime.UtcNow,
                WindowEnd = DateTime.UtcNow.AddSeconds(_options.WindowSizeSeconds)
            };
        }
        
        // Update cache
        var cacheOptions = new MemoryCacheEntryOptions
        {
            AbsoluteExpiration = rateLimitInfo.WindowEnd
        };
        
        _cache.Set(cacheKey, rateLimitInfo, cacheOptions);
        
        return rateLimitInfo.RequestCount > _options.MaxRequestsPerWindow;
    }
    
    private void AddRateLimitHeaders(HttpResponse response, string clientIdentifier)
    {
        var cacheKey = $"rate_limit:{clientIdentifier}";
        
        if (_cache.TryGetValue(cacheKey, out RateLimitInfo rateLimitInfo))
        {
            response.Headers["X-RateLimit-Limit"] = _options.MaxRequestsPerWindow.ToString();
            response.Headers["X-RateLimit-Remaining"] = Math.Max(0, _options.MaxRequestsPerWindow - rateLimitInfo.RequestCount).ToString();
            response.Headers["X-RateLimit-Reset"] = ((DateTimeOffset)rateLimitInfo.WindowEnd).ToUnixTimeSeconds().ToString();
        }
    }
}

public class RateLimitInfo
{
    public int RequestCount { get; set; }
    public DateTime WindowStart { get; set; }
    public DateTime WindowEnd { get; set; }
}

public class RateLimitingOptions
{
    public int MaxRequestsPerWindow { get; set; } = 100;
    public int WindowSizeSeconds { get; set; } = 60;
    public int RetryAfterSeconds { get; set; } = 60;
}

public static class ApiRateLimitingMiddlewareExtensions
{
    public static IApplicationBuilder UseApiRateLimiting(this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<ApiRateLimitingMiddleware>();
    }
}
```

## API Security Best Practices

### 1. Secure Configuration
```csharp
public class SecurityConfiguration
{
    public static void ConfigureSecurityServices(IServiceCollection services, IConfiguration configuration)
    {
        // Configure authentication
        services.AddAuthentication(options =>
        {
            options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
            options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
        })
        .AddJwtBearer(options =>
        {
            options.TokenValidationParameters = new TokenValidationParameters
            {
                ValidateIssuer = true,
                ValidateAudience = true,
                ValidateLifetime = true,
                ValidateIssuerSigningKey = true,
                ValidIssuer = configuration["Jwt:Issuer"],
                ValidAudience = configuration["Jwt:Audience"],
                IssuerSigningKey = new SymmetricSecurityKey(
                    Encoding.UTF8.GetBytes(configuration["Jwt:Key"]))
            };
        });
        
        // Configure authorization
        services.AddAuthorization(options =>
        {
            options.AddPolicy("RequireAdminRole", policy =>
                policy.RequireRole("Admin"));
            
            options.AddPolicy("RequireUserRole", policy =>
                policy.RequireRole("User", "Admin"));
            
            options.AddPolicy("RequireApiKey", policy =>
                policy.RequireClaim("api_key"));
        });
        
        // Configure CORS
        services.AddCors(options =>
        {
            options.AddPolicy("SecureCorsPolicy", policy =>
            {
                policy.WithOrigins(configuration.GetSection("AllowedOrigins").Get<string[]>())
                      .AllowAnyMethod()
                      .AllowAnyHeader()
                      .AllowCredentials();
            });
        });
        
        // Configure security headers
        services.Configure<SecurityHeadersOptions>(configuration.GetSection("SecurityHeaders"));
        
        // Configure rate limiting
        services.Configure<RateLimitingOptions>(configuration.GetSection("RateLimiting"));
    }
    
    public static void ConfigureSecurityApp(IApplicationBuilder app, IWebHostEnvironment env)
    {
        // Use HTTPS redirection in production
        if (env.IsProduction())
        {
            app.UseHttpsRedirection();
        }
        
        // Use security headers
        app.UseSecurityHeaders();
        
        // Use CORS
        app.UseCors("SecureCorsPolicy");
        
        // Use rate limiting
        app.UseApiRateLimiting();
        
        // Use authentication and authorization
        app.UseAuthentication();
        app.UseAuthorization();
    }
}
```

### 2. Security Monitoring
```csharp
public class SecurityMonitoringService
{
    private readonly ILogger<SecurityMonitoringService> _logger;
    private readonly IConfiguration _configuration;
    private readonly IMemoryCache _cache;
    
    public SecurityMonitoringService(
        ILogger<SecurityMonitoringService> logger,
        IConfiguration configuration,
        IMemoryCache cache)
    {
        _logger = logger;
        _configuration = configuration;
        _cache = cache;
    }
    
    public void LogSecurityEvent(SecurityEvent securityEvent)
    {
        try
        {
            // Log security event
            _logger.LogWarning("Security Event: {EventType} - {Description} - User: {UserId} - IP: {IpAddress} - Timestamp: {Timestamp}",
                securityEvent.EventType,
                securityEvent.Description,
                securityEvent.UserId ?? "Anonymous",
                securityEvent.IpAddress,
                securityEvent.Timestamp);
            
            // Store in cache for monitoring
            var cacheKey = $"security_event_{DateTime.UtcNow:yyyyMMdd}";
            var events = _cache.Get<List<SecurityEvent>>(cacheKey) ?? new List<SecurityEvent>();
            events.Add(securityEvent);
            
            var cacheOptions = new MemoryCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromDays(1)
            };
            
            _cache.Set(cacheKey, events, cacheOptions);
            
            // Check for security alerts
            CheckSecurityAlerts(events);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error logging security event");
        }
    }
    
    private void CheckSecurityAlerts(List<SecurityEvent> events)
    {
        var recentEvents = events.Where(e => 
            e.Timestamp >= DateTime.UtcNow.AddMinutes(-5)).ToList();
        
        // Check for multiple failed login attempts
        var failedLogins = recentEvents.Count(e => e.EventType == SecurityEventType.FailedLogin);
        if (failedLogins >= 5)
        {
            _logger.LogError("SECURITY ALERT: Multiple failed login attempts detected. Count: {Count}", failedLogins);
            // Send alert notification
            SendSecurityAlert("Multiple failed login attempts detected", failedLogins);
        }
        
        // Check for suspicious IP addresses
        var suspiciousIps = recentEvents
            .GroupBy(e => e.IpAddress)
            .Where(g => g.Count() >= 10)
            .Select(g => g.Key);
        
        foreach (var ip in suspiciousIps)
        {
            _logger.LogError("SECURITY ALERT: Suspicious activity from IP {IpAddress}", ip);
            SendSecurityAlert($"Suspicious activity from IP {ip}", 0);
        }
        
        // Check for unauthorized access attempts
        var unauthorizedAccess = recentEvents.Count(e => e.EventType == SecurityEventType.UnauthorizedAccess);
        if (unauthorizedAccess >= 3)
        {
            _logger.LogError("SECURITY ALERT: Multiple unauthorized access attempts detected. Count: {Count}", unauthorizedAccess);
            SendSecurityAlert("Multiple unauthorized access attempts detected", unauthorizedAccess);
        }
    }
    
    private void SendSecurityAlert(string message, int count)
    {
        // Implement alert notification logic
        // This could be email, SMS, Slack, etc.
        _logger.LogWarning("SECURITY ALERT: {Message} - Count: {Count}", message, count);
    }
}

public class SecurityEvent
{
    public SecurityEventType EventType { get; set; }
    public string Description { get; set; }
    public string UserId { get; set; }
    public string IpAddress { get; set; }
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
    public Dictionary<string, object> AdditionalData { get; set; } = new Dictionary<string, object>();
}

public enum SecurityEventType
{
    FailedLogin,
    SuccessfulLogin,
    UnauthorizedAccess,
    SuspiciousActivity,
    RateLimitExceeded,
    InvalidToken,
    SecurityHeaderViolation
}
```

## Mülakat Soruları

### Temel Sorular

1. **API Security'de en önemli tehditler nelerdir?**
   - **Cevap**: Authentication bypass, Authorization vulnerabilities, Input validation attacks, SQL injection, XSS, CSRF, Rate limiting bypass.

2. **API'lerde authentication nasıl implement edilir?**
   - **Cevap**: JWT tokens, OAuth 2.0, API keys, Multi-factor authentication, Session management.

3. **Authorization vs Authentication arasındaki fark nedir?**
   - **Cevap**: Authentication kim olduğunu doğrular, Authorization ne yapabileceğini belirler.

4. **API rate limiting neden önemlidir?**
   - **Cevap**: DDoS attacks'i önler, resource abuse'u engeller, fair usage sağlar, API stability korur.

5. **Security headers nelerdir ve neden kullanılır?**
   - **Cevap**: CSP, X-Frame-Options, X-Content-Type-Options, HSTS. Browser-based attacks'i önler.

### Teknik Sorular

1. **API'de SQL injection nasıl önlenir?**
   - **Cevap**: Parameterized queries, Input validation, ORM kullanımı, Least privilege principle.

2. **JWT token security nasıl sağlanır?**
   - **Cevap**: Secure signing, Expiration times, Token validation, Secure storage, HTTPS.

3. **API versioning security implications nelerdir?**
   - **Cevap**: Backward compatibility, Deprecation policies, Security updates, Access control.

4. **API monitoring ve alerting nasıl implement edilir?**
   - **Cevap**: Log aggregation, Metrics collection, Anomaly detection, Real-time alerts.

5. **API security testing nasıl yapılır?**
   - **Cevap**: Penetration testing, Vulnerability scanning, Security code review, Automated security testing.

## Best Practices

1. **Authentication & Authorization**
   - Multi-factor authentication kullanın
   - Role-based access control implement edin
   - JWT token expiration sürelerini optimize edin
   - Secure password policies uygulayın

2. **Input Validation**
   - Tüm input'ları validate edin
   - Whitelist approach kullanın
   - SQL injection ve XSS prevention implement edin
   - File upload security sağlayın

3. **Security Headers**
   - Security headers middleware kullanın
   - Content Security Policy implement edin
   - HTTPS zorunlu yapın
   - CORS policy'leri sıkılaştırın

4. **Monitoring & Alerting**
   - Security events log edin
   - Real-time monitoring implement edin
   - Automated alerting kurun
   - Security metrics toplayın

5. **Rate Limiting**
   - Client-based rate limiting implement edin
   - IP-based rate limiting kullanın
   - Rate limit headers ekleyin
   - Graceful degradation sağlayın

## Kaynaklar

- [OWASP API Security Top 10](https://owasp.org/www-project-api-security/)
- [Microsoft Security Documentation](https://docs.microsoft.com/en-us/aspnet/core/security/)
- [API Security Best Practices](https://cloud.google.com/apis/design/security)
- [REST API Security](https://restfulapi.net/security-essentials/)
- [API Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
