# Identity & Access Management

## Giriş

Identity & Access Management (IAM), kullanıcı kimliklerini yönetme ve sistemlere erişim kontrolü sağlama süreçleridir. Mid-level geliştiriciler için IAM sistemlerini anlamak, güvenli ve ölçeklenebilir authentication/authorization sistemleri tasarlamada kritiktir.

## Identity Management

### 1. User Identity Store
Kullanıcı kimlik bilgilerini güvenli şekilde saklama.

```csharp
public interface IUserStore
{
    Task<User> FindByIdAsync(string userId, CancellationToken cancellationToken = default);
    Task<User> FindByNameAsync(string normalizedUserName, CancellationToken cancellationToken = default);
    Task<User> FindByEmailAsync(string normalizedEmail, CancellationToken cancellationToken = default);
    Task<bool> CreateAsync(User user, CancellationToken cancellationToken = default);
    Task<bool> UpdateAsync(User user, CancellationToken cancellationToken = default);
    Task<bool> DeleteAsync(User user, CancellationToken cancellationToken = default);
}

public class User
{
    public string Id { get; set; }
    public string UserName { get; set; }
    public string NormalizedUserName { get; set; }
    public string Email { get; set; }
    public string NormalizedEmail { get; set; }
    public bool EmailConfirmed { get; set; }
    public string PasswordHash { get; set; }
    public string SecurityStamp { get; set; }
    public string ConcurrencyStamp { get; set; }
    public string PhoneNumber { get; set; }
    public bool PhoneNumberConfirmed { get; set; }
    public bool TwoFactorEnabled { get; set; }
    public DateTimeOffset? LockoutEnd { get; set; }
    public bool LockoutEnabled { get; set; }
    public int AccessFailedCount { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? LastLoginAt { get; set; }
    public bool IsActive { get; set; }
    public List<string> Roles { get; set; } = new();
    public Dictionary<string, string> Claims { get; set; } = new();
}

public class SqlServerUserStore : IUserStore
{
    private readonly IDbConnection _connection;
    private readonly ILogger<SqlServerUserStore> _logger;
    private readonly IPasswordHasher _passwordHasher;
    
    public SqlServerUserStore(
        IDbConnection connection,
        ILogger<SqlServerUserStore> logger,
        IPasswordHasher passwordHasher)
    {
        _connection = connection;
        _logger = logger;
        _passwordHasher = passwordHasher;
    }
    
    public async Task<User> FindByIdAsync(string userId, CancellationToken cancellationToken = default)
    {
        try
        {
            var sql = @"
                SELECT Id, UserName, NormalizedUserName, Email, NormalizedEmail,
                       EmailConfirmed, PasswordHash, SecurityStamp, ConcurrencyStamp,
                       PhoneNumber, PhoneNumberConfirmed, TwoFactorEnabled,
                       LockoutEnd, LockoutEnabled, AccessFailedCount,
                       CreatedAt, LastLoginAt, IsActive
                FROM Users 
                WHERE Id = @UserId AND IsActive = 1
            ";
            
            var user = await _connection.QueryFirstOrDefaultAsync<User>(sql, new { UserId = userId });
            
            if (user != null)
            {
                await LoadUserRolesAsync(user, cancellationToken);
                await LoadUserClaimsAsync(user, cancellationToken);
            }
            
            return user;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error finding user by ID: {UserId}", userId);
            throw;
        }
    }
    
    public async Task<User> FindByNameAsync(string normalizedUserName, CancellationToken cancellationToken = default)
    {
        try
        {
            var sql = @"
                SELECT Id, UserName, NormalizedUserName, Email, NormalizedEmail,
                       EmailConfirmed, PasswordHash, SecurityStamp, ConcurrencyStamp,
                       PhoneNumber, PhoneNumberConfirmed, TwoFactorEnabled,
                       LockoutEnd, LockoutEnabled, AccessFailedCount,
                       CreatedAt, LastLoginAt, IsActive
                FROM Users 
                WHERE NormalizedUserName = @NormalizedUserName AND IsActive = 1
            ";
            
            var user = await _connection.QueryFirstOrDefaultAsync<User>(sql, new { NormalizedUserName = normalizedUserName });
            
            if (user != null)
            {
                await LoadUserRolesAsync(user, cancellationToken);
                await LoadUserClaimsAsync(user, cancellationToken);
            }
            
            return user;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error finding user by name: {NormalizedUserName}", normalizedUserName);
            throw;
        }
    }
    
    public async Task<bool> CreateAsync(User user, CancellationToken cancellationToken = default)
    {
        try
        {
            // Generate security stamp for password change tracking
            user.SecurityStamp = Guid.NewGuid().ToString();
            user.ConcurrencyStamp = Guid.NewGuid().ToString();
            user.CreatedAt = DateTime.UtcNow;
            user.IsActive = true;
            
            var sql = @"
                INSERT INTO Users (Id, UserName, NormalizedUserName, Email, NormalizedEmail,
                                 EmailConfirmed, PasswordHash, SecurityStamp, ConcurrencyStamp,
                                 PhoneNumber, PhoneNumberConfirmed, TwoFactorEnabled,
                                 LockoutEnd, LockoutEnabled, AccessFailedCount,
                                 CreatedAt, LastLoginAt, IsActive)
                VALUES (@Id, @UserName, @NormalizedUserName, @Email, @NormalizedEmail,
                        @EmailConfirmed, @PasswordHash, @SecurityStamp, @ConcurrencyStamp,
                        @PhoneNumber, @PhoneNumberConfirmed, @TwoFactorEnabled,
                        @LockoutEnd, @LockoutEnabled, @AccessFailedCount,
                        @CreatedAt, @LastLoginAt, @IsActive)
            ";
            
            var rowsAffected = await _connection.ExecuteAsync(sql, user);
            
            if (rowsAffected > 0)
            {
                await SaveUserRolesAsync(user, cancellationToken);
                await SaveUserClaimsAsync(user, cancellationToken);
                return true;
            }
            
            return false;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating user: {UserName}", user.UserName);
            throw;
        }
    }
    
    private async Task LoadUserRolesAsync(User user, CancellationToken cancellationToken)
    {
        var sql = "SELECT RoleName FROM UserRoles WHERE UserId = @UserId";
        var roles = await _connection.QueryAsync<string>(sql, new { UserId = user.Id });
        user.Roles = roles.ToList();
    }
    
    private async Task LoadUserClaimsAsync(User user, CancellationToken cancellationToken)
    {
        var sql = "SELECT ClaimType, ClaimValue FROM UserClaims WHERE UserId = @UserId";
        var claims = await _connection.QueryAsync<Claim>(sql, new { UserId = user.Id });
        user.Claims = claims.ToDictionary(c => c.ClaimType, c => c.ClaimValue);
    }
    
    private async Task SaveUserRolesAsync(User user, CancellationToken cancellationToken)
    {
        if (user.Roles?.Any() == true)
        {
            var sql = "INSERT INTO UserRoles (UserId, RoleName) VALUES (@UserId, @RoleName)";
            var parameters = user.Roles.Select(role => new { UserId = user.Id, RoleName = role });
            await _connection.ExecuteAsync(sql, parameters);
        }
    }
    
    private async Task SaveUserClaimsAsync(User user, CancellationToken cancellationToken)
    {
        if (user.Claims?.Any() == true)
        {
            var sql = "INSERT INTO UserClaims (UserId, ClaimType, ClaimValue) VALUES (@UserId, @ClaimType, @ClaimValue)";
            var parameters = user.Claims.Select(kvp => new { UserId = user.Id, ClaimType = kvp.Key, ClaimValue = kvp.Value });
            await _connection.ExecuteAsync(sql, parameters);
        }
    }
}

public class Claim
{
    public string ClaimType { get; set; }
    public string ClaimValue { get; set; }
}
```

### 2. Password Management
Güvenli şifre yönetimi ve hash'leme.

```csharp
public interface IPasswordHasher
{
    string HashPassword(string password);
    PasswordVerificationResult VerifyHashedPassword(string hashedPassword, string providedPassword);
}

public class PasswordHasher : IPasswordHasher
{
    private readonly ILogger<PasswordHasher> _logger;
    
    public PasswordHasher(ILogger<PasswordHasher> logger)
    {
        _logger = logger;
    }
    
    public string HashPassword(string password)
    {
        if (string.IsNullOrEmpty(password))
        {
            throw new ArgumentException("Password cannot be null or empty", nameof(password));
        }
        
        try
        {
            // Use BCrypt for password hashing
            return BCrypt.Net.BCrypt.HashPassword(password, BCrypt.Net.BCrypt.GenerateSalt(12));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error hashing password");
            throw;
        }
    }
    
    public PasswordVerificationResult VerifyHashedPassword(string hashedPassword, string providedPassword)
    {
        if (string.IsNullOrEmpty(hashedPassword))
        {
            return PasswordVerificationResult.Failed;
        }
        
        if (string.IsNullOrEmpty(providedPassword))
        {
            return PasswordVerificationResult.Failed;
        }
        
        try
        {
            var isValid = BCrypt.Net.BCrypt.Verify(providedPassword, hashedPassword);
            return isValid ? PasswordVerificationResult.Success : PasswordVerificationResult.Failed;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error verifying password hash");
            return PasswordVerificationResult.Failed;
        }
    }
}

public enum PasswordVerificationResult
{
    Failed,
    Success,
    SuccessRehashNeeded
}

public class PasswordPolicy
{
    public int MinimumLength { get; set; } = 8;
    public bool RequireUppercase { get; set; } = true;
    public bool RequireLowercase { get; set; } = true;
    public bool RequireDigit { get; set; } = true;
    public bool RequireSpecialCharacter { get; set; } = true;
    public int MaximumAgeDays { get; set; } = 90;
    public int PasswordHistoryCount { get; set; } = 5;
    public bool PreventCommonPasswords { get; set; } = true;
}

public class PasswordValidator
{
    private readonly PasswordPolicy _policy;
    private readonly ILogger<PasswordValidator> _logger;
    
    public PasswordValidator(PasswordPolicy policy, ILogger<PasswordValidator> logger)
    {
        _policy = policy;
        _logger = logger;
    }
    
    public ValidationResult ValidatePassword(string password)
    {
        var result = new ValidationResult();
        
        if (string.IsNullOrEmpty(password))
        {
            result.AddError("Password", "Password is required");
            return result;
        }
        
        // Length validation
        if (password.Length < _policy.MinimumLength)
        {
            result.AddError("Password", $"Password must be at least {_policy.MinimumLength} characters long");
        }
        
        // Character type validation
        if (_policy.RequireUppercase && !password.Any(char.IsUpper))
        {
            result.AddError("Password", "Password must contain at least one uppercase letter");
        }
        
        if (_policy.RequireLowercase && !password.Any(char.IsLower))
        {
            result.AddError("Password", "Password must contain at least one lowercase letter");
        }
        
        if (_policy.RequireDigit && !password.Any(char.IsDigit))
        {
            result.AddError("Password", "Password must contain at least one number");
        }
        
        if (_policy.RequireSpecialCharacter && !password.Any(c => !char.IsLetterOrDigit(c)))
        {
            result.AddError("Password", "Password must contain at least one special character");
        }
        
        // Common password check
        if (_policy.PreventCommonPasswords && IsCommonPassword(password))
        {
            result.AddError("Password", "Password is too common, please choose a stronger password");
        }
        
        return result;
    }
    
    private bool IsCommonPassword(string password)
    {
        var commonPasswords = new[]
        {
            "password", "123456", "123456789", "qwerty", "abc123",
            "password123", "admin", "letmein", "welcome", "monkey"
        };
        
        return commonPasswords.Contains(password.ToLower());
    }
}
```

## Access Control

### 1. Role-Based Access Control (RBAC)
Rol tabanlı erişim kontrolü.

```csharp
public interface IRoleStore
{
    Task<Role> FindByIdAsync(string roleId, CancellationToken cancellationToken = default);
    Task<Role> FindByNameAsync(string normalizedRoleName, CancellationToken cancellationToken = default);
    Task<bool> CreateAsync(Role role, CancellationToken cancellationToken = default);
    Task<bool> UpdateAsync(Role role, CancellationToken cancellationToken = default);
    Task<bool> DeleteAsync(Role role, CancellationToken cancellationToken = default);
}

public class Role
{
    public string Id { get; set; }
    public string Name { get; set; }
    public string NormalizedName { get; set; }
    public string Description { get; set; }
    public List<string> Permissions { get; set; } = new();
    public DateTime CreatedAt { get; set; }
    public bool IsActive { get; set; }
}

public class Permission
{
    public string Id { get; set; }
    public string Name { get; set; }
    public string Resource { get; set; }
    public string Action { get; set; }
    public string Description { get; set; }
}

public class RoleBasedAuthorizationHandler : AuthorizationHandler<RoleRequirement>
{
    private readonly IUserStore _userStore;
    private readonly IRoleStore _roleStore;
    
    public RoleBasedAuthorizationHandler(IUserStore userStore, IRoleStore roleStore)
    {
        _userStore = userStore;
        _roleStore = roleStore;
    }
    
    protected override async Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        RoleRequirement requirement)
    {
        var user = context.User;
        
        if (!user.Identity.IsAuthenticated)
        {
            context.Fail();
            return;
        }
        
        var userId = user.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        if (string.IsNullOrEmpty(userId))
        {
            context.Fail();
            return;
        }
        
        var userEntity = await _userStore.FindByIdAsync(userId);
        if (userEntity == null)
        {
            context.Fail();
            return;
        }
        
        // Check if user has any of the required roles
        var hasRequiredRole = requirement.Roles.Any(role => 
            userEntity.Roles.Contains(role, StringComparer.OrdinalIgnoreCase));
        
        if (hasRequiredRole)
        {
            context.Succeed(requirement);
        }
        else
        {
            context.Fail();
        }
    }
}

public class RoleRequirement : IAuthorizationRequirement
{
    public IEnumerable<string> Roles { get; }
    
    public RoleRequirement(params string[] roles)
    {
        Roles = roles ?? Array.Empty<string>();
    }
}

public class PermissionBasedAuthorizationHandler : AuthorizationHandler<PermissionRequirement>
{
    private readonly IUserStore _userStore;
    private readonly IRoleStore _roleStore;
    
    public PermissionBasedAuthorizationHandler(IUserStore userStore, IRoleStore roleStore)
    {
        _userStore = userStore;
        _roleStore = roleStore;
    }
    
    protected override async Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        PermissionRequirement requirement)
    {
        var user = context.User;
        
        if (!user.Identity.IsAuthenticated)
        {
            context.Fail();
            return;
        }
        
        var userId = user.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        if (string.IsNullOrEmpty(userId))
        {
            context.Fail();
            return;
        }
        
        var userEntity = await _userStore.FindByIdAsync(userId);
        if (userEntity == null)
        {
            context.Fail();
            return;
        }
        
        // Get all permissions from user's roles
        var userPermissions = new HashSet<string>(StringComparer.OrdinalIgnoreCase);
        
        foreach (var roleName in userEntity.Roles)
        {
            var role = await _roleStore.FindByNameAsync(roleName);
            if (role?.Permissions != null)
            {
                foreach (var permission in role.Permissions)
                {
                    userPermissions.Add(permission);
                }
            }
        }
        
        // Check if user has the required permission
        var hasPermission = userPermissions.Contains(requirement.Permission);
        
        if (hasPermission)
        {
            context.Succeed(requirement);
        }
        else
        {
            context.Fail();
        }
    }
}

public class PermissionRequirement : IAuthorizationRequirement
{
    public string Permission { get; }
    
    public PermissionRequirement(string permission)
    {
        Permission = permission ?? throw new ArgumentNullException(nameof(permission));
    }
}
```

### 2. Claims-Based Authorization
Claim tabanlı yetkilendirme.

```csharp
public class ClaimsBasedAuthorizationHandler : AuthorizationHandler<ClaimsRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        ClaimsRequirement requirement)
    {
        var user = context.User;
        
        if (!user.Identity.IsAuthenticated)
        {
            context.Fail();
            return Task.CompletedTask;
        }
        
        // Check if user has all required claims
        var hasAllClaims = requirement.RequiredClaims.All(requiredClaim =>
            user.HasClaim(requiredClaim.Type, requiredClaim.Value));
        
        if (hasAllClaims)
        {
            context.Succeed(requirement);
        }
        else
        {
            context.Fail();
        }
        
        return Task.CompletedTask;
    }
}

public class ClaimsRequirement : IAuthorizationRequirement
{
    public IEnumerable<Claim> RequiredClaims { get; }
    
    public ClaimsRequirement(params Claim[] claims)
    {
        RequiredClaims = claims ?? Array.Empty<Claim>();
    }
}

public class CustomClaimsPrincipalFactory : IClaimsPrincipalFactory<User>
{
    private readonly IUserStore _userStore;
    private readonly IRoleStore _roleStore;
    
    public CustomClaimsPrincipalFactory(IUserStore userStore, IRoleStore roleStore)
    {
        _userStore = userStore;
        _roleStore = roleStore;
    }
    
    public async Task<ClaimsPrincipal> CreateAsync(User user)
    {
        var claims = new List<Claim>
        {
            new Claim(ClaimTypes.NameIdentifier, user.Id),
            new Claim(ClaimTypes.Name, user.UserName),
            new Claim(ClaimTypes.Email, user.Email),
            new Claim("created_at", user.CreatedAt.ToString("O")),
            new Claim("is_active", user.IsActive.ToString())
        };
        
        // Add role claims
        foreach (var role in user.Roles)
        {
            claims.Add(new Claim(ClaimTypes.Role, role));
        }
        
        // Add custom claims
        foreach (var kvp in user.Claims)
        {
            claims.Add(new Claim(kvp.Key, kvp.Value));
        }
        
        var identity = new ClaimsIdentity(claims, "Custom", ClaimTypes.Name, ClaimTypes.Role);
        return new ClaimsPrincipal(identity);
    }
}
```

## Session Management

### 1. Session Store
Kullanıcı oturumlarını yönetme.

```csharp
public interface ISessionStore
{
    Task<Session> CreateAsync(string userId, TimeSpan duration, CancellationToken cancellationToken = default);
    Task<Session> GetAsync(string sessionId, CancellationToken cancellationToken = default);
    Task<bool> UpdateAsync(Session session, CancellationToken cancellationToken = default);
    Task<bool> DeleteAsync(string sessionId, CancellationToken cancellationToken = default);
    Task<bool> ExtendAsync(string sessionId, TimeSpan additionalDuration, CancellationToken cancellationToken = default);
    Task<IEnumerable<Session>> GetUserSessionsAsync(string userId, CancellationToken cancellationToken = default);
}

public class Session
{
    public string Id { get; set; }
    public string UserId { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime ExpiresAt { get; set; }
    public DateTime LastAccessedAt { get; set; }
    public string IpAddress { get; set; }
    public string UserAgent { get; set; }
    public bool IsActive { get; set; }
    public Dictionary<string, string> Metadata { get; set; } = new();
}

public class RedisSessionStore : ISessionStore
{
    private readonly IDatabase _redis;
    private readonly ILogger<RedisSessionStore> _logger;
    private readonly string _keyPrefix = "session:";
    
    public RedisSessionStore(IConnectionMultiplexer redis, ILogger<RedisSessionStore> logger)
    {
        _redis = redis.GetDatabase();
        _logger = logger;
    }
    
    public async Task<Session> CreateAsync(string userId, TimeSpan duration, CancellationToken cancellationToken = default)
    {
        try
        {
            var session = new Session
            {
                Id = Guid.NewGuid().ToString(),
                UserId = userId,
                CreatedAt = DateTime.UtcNow,
                ExpiresAt = DateTime.UtcNow.Add(duration),
                LastAccessedAt = DateTime.UtcNow,
                IsActive = true
            };
            
            var key = $"{_keyPrefix}{session.Id}";
            var json = JsonSerializer.Serialize(session);
            
            await _redis.StringSetAsync(key, json, duration);
            
            // Store session ID in user's session list
            var userSessionsKey = $"user_sessions:{userId}";
            await _redis.SetAddAsync(userSessionsKey, session.Id);
            await _redis.KeyExpireAsync(userSessionsKey, duration);
            
            _logger.LogInformation("Created session {SessionId} for user {UserId}", session.Id, userId);
            return session;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating session for user {UserId}", userId);
            throw;
        }
    }
    
    public async Task<Session> GetAsync(string sessionId, CancellationToken cancellationToken = default)
    {
        try
        {
            var key = $"{_keyPrefix}{sessionId}";
            var json = await _redis.StringGetAsync(key);
            
            if (json.IsNull)
            {
                return null;
            }
            
            var session = JsonSerializer.Deserialize<Session>(json);
            
            // Check if session is expired
            if (session.ExpiresAt <= DateTime.UtcNow)
            {
                await DeleteAsync(sessionId, cancellationToken);
                return null;
            }
            
            // Update last accessed time
            session.LastAccessedAt = DateTime.UtcNow;
            await UpdateAsync(session, cancellationToken);
            
            return session;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting session {SessionId}", sessionId);
            throw;
        }
    }
    
    public async Task<bool> UpdateAsync(Session session, CancellationToken cancellationToken = default)
    {
        try
        {
            var key = $"{_keyPrefix}{session.Id}";
            var json = JsonSerializer.Serialize(session);
            
            var remainingTime = session.ExpiresAt - DateTime.UtcNow;
            if (remainingTime <= TimeSpan.Zero)
            {
                return false;
            }
            
            return await _redis.StringSetAsync(key, json, remainingTime);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error updating session {SessionId}", session.Id);
            throw;
        }
    }
    
    public async Task<bool> DeleteAsync(string sessionId, CancellationToken cancellationToken = default)
    {
        try
        {
            var key = $"{_keyPrefix}{sessionId}";
            var session = await GetAsync(sessionId, cancellationToken);
            
            if (session != null)
            {
                // Remove from user's session list
                var userSessionsKey = $"user_sessions:{session.UserId}";
                await _redis.SetRemoveAsync(userSessionsKey, sessionId);
            }
            
            return await _redis.KeyDeleteAsync(key);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error deleting session {SessionId}", sessionId);
            throw;
        }
    }
    
    public async Task<bool> ExtendAsync(string sessionId, TimeSpan additionalDuration, CancellationToken cancellationToken = default)
    {
        try
        {
            var session = await GetAsync(sessionId, cancellationToken);
            if (session == null)
            {
                return false;
            }
            
            session.ExpiresAt = session.ExpiresAt.Add(additionalDuration);
            return await UpdateAsync(session, cancellationToken);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error extending session {SessionId}", sessionId);
            throw;
        }
    }
    
    public async Task<IEnumerable<Session>> GetUserSessionsAsync(string userId, CancellationToken cancellationToken = default)
    {
        try
        {
            var userSessionsKey = $"user_sessions:{userId}";
            var sessionIds = await _redis.SetMembersAsync(userSessionsKey);
            
            var sessions = new List<Session>();
            foreach (var sessionId in sessionIds)
            {
                var session = await GetAsync(sessionId.ToString(), cancellationToken);
                if (session != null)
                {
                    sessions.Add(session);
                }
            }
            
            return sessions;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting sessions for user {UserId}", userId);
            throw;
        }
    }
}
```

## Mülakat Soruları

### Temel Sorular

1. **Identity & Access Management nedir ve neden önemlidir?**
   - **Cevap**: Kullanıcı kimliklerini yönetme ve sistemlere erişim kontrolü sağlama. Security, compliance ve user experience için kritik.

2. **Role-Based Access Control (RBAC) nedir?**
   - **Cevap**: Kullanıcılara rol atayarak yetki verme sistemi. Permission management ve access control için yaygın kullanılan pattern.

3. **Claims-based authorization nedir?**
   - **Cevap**: Kullanıcı hakkında bilgi içeren claim'ler ile yetkilendirme. Flexible ve granular access control sağlar.

4. **Session management nedir ve nasıl yapılır?**
   - **Cevap**: Kullanıcı oturumlarını yönetme. Redis, database veya in-memory store kullanılarak implement edilir.

5. **Password hashing nedir ve hangi algoritmalar kullanılır?**
   - **Cevap**: Şifreleri güvenli şekilde saklama. BCrypt, Argon2, PBKDF2 gibi algoritmalar kullanılır.

### Teknik Sorular

1. **User store'da password security nasıl sağlanır?**
   - **Cevap**: Strong hashing algorithms, salt generation, secure storage, password policies, regular updates.

2. **RBAC vs Claims-based authorization arasındaki fark nedir?**
   - **Cevap**: RBAC rol odaklı, Claims-based daha granular. RBAC daha basit, Claims-based daha flexible.

3. **Session hijacking nasıl önlenir?**
   - **Cevap**: Secure session IDs, HTTPS, secure cookies, session timeout, IP validation, user agent validation.

4. **Multi-factor authentication nasıl implement edilir?**
   - **Cevap**: TOTP, SMS, email verification, hardware tokens, biometric authentication.

5. **Identity federation nedir ve nasıl çalışır?**
   - **Cevap**: Farklı sistemler arası kimlik paylaşımı. SAML, OpenID Connect, OAuth2 protokolleri kullanılır.

## Best Practices

1. **Security**
   - Strong password policies implement edin
   - Multi-factor authentication kullanın
   - Session security sağlayın
   - Regular security audits yapın

2. **Performance**
   - Caching implement edin
   - Database optimization yapın
   - Async operations kullanın
   - Connection pooling kullanın

3. **Scalability**
   - Distributed session store kullanın
   - Horizontal scaling planlayın
   - Load balancing implement edin
   - Database sharding düşünün

4. **Monitoring**
   - Authentication logs tutun
   - Failed login attempts izleyin
   - Session analytics toplayın
   - Security alerts kurun

## Kaynaklar

- [ASP.NET Core Identity](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/identity)
- [Role-Based Access Control](https://docs.microsoft.com/en-us/azure/role-based-access-control/)
- [Claims-Based Authorization](https://docs.microsoft.com/en-us/dotnet/framework/wcf/feature-details/claims-based-authorization)
- [Session Management](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/session)
- [Identity Federation](https://docs.microsoft.com/en-us/azure/active-directory/develop/)
