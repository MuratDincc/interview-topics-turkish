# Authorization

## Genel Bakış
Authorization (Yetkilendirme), kimliği doğrulanmış bir kullanıcının veya sistemin belirli kaynaklara erişim izninin kontrol edilmesi sürecidir. Bu süreç, kullanıcının hangi işlemleri yapabileceğini belirler.

## Mülakat Soruları ve Cevapları

### 1. Role-based Authorization nedir ve nasıl uygulanır?
**Cevap:**
Role-based Authorization, kullanıcıların rollerine göre yetkilendirme yapılmasıdır. ASP.NET Core'da `[Authorize(Roles = "RoleName")]` attribute'u ile uygulanır.

**Örnek Kod:**
```csharp
// Role tanımlama
public class ApplicationUser : IdentityUser
{
    public string FullName { get; set; }
}

// Role-based authorization
[Authorize(Roles = "Admin")]
[ApiController]
[Route("api/[controller]")]
public class AdminController : ControllerBase
{
    [HttpGet]
    public IActionResult GetAdminData()
    {
        return Ok("Admin data");
    }
}

// Çoklu role kontrolü
[Authorize(Roles = "Admin,Manager")]
[ApiController]
[Route("api/[controller]")]
public class ManagementController : ControllerBase
{
    [HttpGet]
    public IActionResult GetManagementData()
    {
        return Ok("Management data");
    }
}
```

### 2. Policy-based Authorization nedir ve nasıl uygulanır?
**Cevap:**
Policy-based Authorization, daha esnek ve karmaşık yetkilendirme kuralları tanımlamaya olanak sağlar. Policy'ler gereksinimlere göre özelleştirilebilir.

**Örnek Kod:**
```csharp
// Policy tanımlama
public void ConfigureServices(IServiceCollection services)
{
    services.AddAuthorization(options =>
    {
        options.AddPolicy("RequireAdminRole", policy =>
            policy.RequireRole("Admin"));

        options.AddPolicy("MinimumAge", policy =>
            policy.RequireAssertion(context =>
                context.User.HasClaim(c =>
                    (c.Type == "Age" && int.Parse(c.Value) >= 18))));

        options.AddPolicy("CanEditProduct", policy =>
            policy.RequireAssertion(context =>
                context.User.IsInRole("Admin") ||
                (context.User.IsInRole("Editor") && 
                 context.User.HasClaim("Permission", "Edit"))));
    });
}

// Policy kullanımı
[Authorize(Policy = "MinimumAge")]
[ApiController]
[Route("api/[controller]")]
public class AdultController : ControllerBase
{
    [HttpGet]
    public IActionResult GetAdultContent()
    {
        return Ok("Adult content");
    }
}
```

### 3. Claims-based Authorization nedir ve nasıl uygulanır?
**Cevap:**
Claims-based Authorization, kullanıcının sahip olduğu claims'lere göre yetkilendirme yapılmasıdır. Claims, kullanıcı hakkında bilgi içeren name-value çiftleridir.

**Örnek Kod:**
```csharp
// Claims tanımlama
public class ClaimsController : ControllerBase
{
    [HttpPost("add-claim")]
    public async Task<IActionResult> AddClaim(string userId, string claimType, string claimValue)
    {
        var user = await _userManager.FindByIdAsync(userId);
        if (user == null) return NotFound();

        var claim = new Claim(claimType, claimValue);
        var result = await _userManager.AddClaimAsync(user, claim);

        if (result.Succeeded)
            return Ok();
        return BadRequest(result.Errors);
    }
}

// Claims-based authorization
[Authorize]
[ApiController]
[Route("api/[controller]")]
public class ClaimsBasedController : ControllerBase
{
    [HttpGet]
    public IActionResult GetData()
    {
        var hasPermission = User.HasClaim("Permission", "Read");
        if (!hasPermission)
            return Forbid();

        return Ok("Protected data");
    }
}
```

### 4. Resource-based Authorization nedir ve nasıl uygulanır?
**Cevap:**
Resource-based Authorization, belirli bir kaynağa erişim yetkisinin kontrol edilmesidir. Bu yaklaşımda, kullanıcının kaynağa erişim hakkı olup olmadığı kontrol edilir.

**Örnek Kod:**
```csharp
// Resource authorization handler
public class DocumentAuthorizationHandler : AuthorizationHandler<DocumentRequirement, Document>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        DocumentRequirement requirement,
        Document resource)
    {
        if (context.User.IsInRole("Admin"))
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }

        if (context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value == resource.OwnerId)
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}

// Resource authorization kullanımı
[Authorize]
[ApiController]
[Route("api/[controller]")]
public class DocumentsController : ControllerBase
{
    private readonly IAuthorizationService _authorizationService;

    [HttpGet("{id}")]
    public async Task<IActionResult> GetDocument(int id)
    {
        var document = await _documentRepository.GetByIdAsync(id);
        if (document == null) return NotFound();

        var authResult = await _authorizationService.AuthorizeAsync(
            User, document, "DocumentAccess");

        if (!authResult.Succeeded)
            return Forbid();

        return Ok(document);
    }
}
```

### 5. Custom Authorization nasıl uygulanır?
**Cevap:**
Custom Authorization, özel yetkilendirme gereksinimleri için kullanılır. Bu yaklaşımda, `IAuthorizationRequirement` ve `AuthorizationHandler` sınıfları kullanılır.

**Örnek Kod:**
```csharp
// Custom requirement
public class MinimumAgeRequirement : IAuthorizationRequirement
{
    public int MinimumAge { get; }

    public MinimumAgeRequirement(int minimumAge)
    {
        MinimumAge = minimumAge;
    }
}

// Custom handler
public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        var dateOfBirthClaim = context.User.FindFirst(c => c.Type == "DateOfBirth");
        if (dateOfBirthClaim == null) return Task.CompletedTask;

        var dateOfBirth = Convert.ToDateTime(dateOfBirthClaim.Value);
        var age = DateTime.Today.Year - dateOfBirth.Year;
        if (dateOfBirth.Date > DateTime.Today.AddYears(-age)) age--;

        if (age >= requirement.MinimumAge)
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}

// Custom authorization kullanımı
[Authorize]
[ApiController]
[Route("api/[controller]")]
public class CustomAuthController : ControllerBase
{
    [MinimumAgeAuthorize(18)]
    [HttpGet]
    public IActionResult GetAdultContent()
    {
        return Ok("Adult content");
    }
}
```

## Best Practices
1. **Yetkilendirme Stratejisi**
   - En az ayrıcalık prensibi
   - Role-based ve policy-based kombinasyonu
   - Claims tabanlı yetkilendirme
   - Resource-based kontrol

2. **Güvenlik**
   - Yetki kontrolü her seviyede
   - Default-deny yaklaşımı
   - Yetki değişikliklerinin loglanması
   - Düzenli yetki gözden geçirme

3. **Performans**
   - Yetki kontrollerinin önbelleğe alınması
   - Gereksiz yetki kontrollerinden kaçınma
   - Verimli yetki sorguları
   - Ölçeklenebilir yetkilendirme

## Kaynaklar
- [ASP.NET Core Authorization](https://docs.microsoft.com/tr-tr/aspnet/core/security/authorization/)
- [Role-based Authorization](https://docs.microsoft.com/tr-tr/aspnet/core/security/authorization/roles)
- [Policy-based Authorization](https://docs.microsoft.com/tr-tr/aspnet/core/security/authorization/policies) 