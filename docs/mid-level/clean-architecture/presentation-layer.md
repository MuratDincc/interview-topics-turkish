# Presentation Layer

## Genel Bakış
Presentation Layer, Clean Architecture'ın en dış katmanıdır ve kullanıcı arayüzü ile etkileşimi yönetir. Bu katman, API endpoints, web sayfaları ve kullanıcı arayüzü bileşenlerini içerir.

## Temel Bileşenler

### 1. Controllers
API endpoints ve HTTP isteklerini yöneten bileşenler.

**Örnek Kod:**
```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly ICreateOrderUseCase _createOrderUseCase;
    private readonly IGetOrderUseCase _getOrderUseCase;
    private readonly ILogger<OrdersController> _logger;

    public OrdersController(
        ICreateOrderUseCase createOrderUseCase,
        IGetOrderUseCase getOrderUseCase,
        ILogger<OrdersController> logger)
    {
        _createOrderUseCase = createOrderUseCase;
        _getOrderUseCase = getOrderUseCase;
        _logger = logger;
    }

    [HttpPost]
    public async Task<IActionResult> CreateOrder([FromBody] CreateOrderRequest request)
    {
        try
        {
            var order = await _createOrderUseCase.Execute(request);
            return CreatedAtAction(nameof(GetOrder), new { id = order.Id }, order);
        }
        catch (DomainException ex)
        {
            _logger.LogWarning(ex, "Domain exception occurred");
            return BadRequest(new { message = ex.Message });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating order");
            return StatusCode(500, new { message = "Internal server error" });
        }
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetOrder(Guid id)
    {
        try
        {
            var order = await _getOrderUseCase.Execute(id);
            return Ok(order);
        }
        catch (NotFoundException)
        {
            return NotFound();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting order");
            return StatusCode(500, new { message = "Internal server error" });
        }
    }
}
```

### 2. ViewModels
Kullanıcı arayüzü için veri modelleri.

**Örnek Kod:**
```csharp
public class OrderViewModel
{
    public Guid Id { get; set; }
    public string OrderNumber { get; set; }
    public DateTime OrderDate { get; set; }
    public string Status { get; set; }
    public List<OrderItemViewModel> Items { get; set; }
    public decimal TotalAmount { get; set; }

    public static OrderViewModel FromOrder(OrderDto order)
    {
        return new OrderViewModel
        {
            Id = order.Id,
            OrderNumber = order.OrderNumber,
            OrderDate = order.OrderDate,
            Status = order.Status.ToString(),
            Items = order.Items.Select(OrderItemViewModel.FromOrderItem).ToList(),
            TotalAmount = order.Items.Sum(i => i.Total)
        };
    }
}

public class CreateOrderViewModel
{
    [Required]
    public string OrderNumber { get; set; }

    [Required]
    [MinLength(1)]
    public List<OrderItemViewModel> Items { get; set; }
}
```

### 3. Middleware
HTTP pipeline'da çalışan ara yazılımlar.

**Örnek Kod:**
```csharp
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(
        RequestDelegate next,
        ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (DomainException ex)
        {
            _logger.LogWarning(ex, "Domain exception occurred");
            await HandleExceptionAsync(context, ex, StatusCodes.Status400BadRequest);
        }
        catch (NotFoundException ex)
        {
            _logger.LogWarning(ex, "Resource not found");
            await HandleExceptionAsync(context, ex, StatusCodes.Status404NotFound);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception occurred");
            await HandleExceptionAsync(context, ex, StatusCodes.Status500InternalServerError);
        }
    }

    private static async Task HandleExceptionAsync(
        HttpContext context,
        Exception exception,
        int statusCode)
    {
        context.Response.ContentType = "application/json";
        context.Response.StatusCode = statusCode;

        var response = new
        {
            message = exception.Message,
            statusCode = statusCode
        };

        await context.Response.WriteAsJsonAsync(response);
    }
}
```

### 4. Filters
Controller action'ları için filtreler.

**Örnek Kod:**
```csharp
public class ValidateModelAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(ActionExecutingContext context)
    {
        if (!context.ModelState.IsValid)
        {
            var errors = context.ModelState
                .Where(x => x.Value.Errors.Count > 0)
                .ToDictionary(
                    kvp => kvp.Key,
                    kvp => kvp.Value.Errors.Select(e => e.ErrorMessage).ToArray()
                );

            context.Result = new BadRequestObjectResult(new
            {
                message = "Validation failed",
                errors = errors
            });
        }
    }
}

public class LogActionAttribute : ActionFilterAttribute
{
    private readonly ILogger _logger;

    public LogActionAttribute(ILogger logger)
    {
        _logger = logger;
    }

    public override void OnActionExecuting(ActionExecutingContext context)
    {
        _logger.LogInformation(
            "Executing {Action} on {Controller}",
            context.ActionDescriptor.RouteValues["action"],
            context.ActionDescriptor.RouteValues["controller"]
        );
    }
}
```

### 5. Authentication/Authorization
Kimlik doğrulama ve yetkilendirme.

**Örnek Kod:**
```csharp
[Authorize]
[ApiController]
[Route("api/[controller]")]
public class SecureController : ControllerBase
{
    [Authorize(Roles = "Admin")]
    [HttpGet("admin-only")]
    public IActionResult AdminOnly()
    {
        return Ok(new { message = "Admin access granted" });
    }

    [Authorize(Policy = "RequireManager")]
    [HttpGet("manager-only")]
    public IActionResult ManagerOnly()
    {
        return Ok(new { message = "Manager access granted" });
    }
}

public class JwtAuthenticationMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IConfiguration _configuration;

    public JwtAuthenticationMiddleware(
        RequestDelegate next,
        IConfiguration configuration)
    {
        _next = next;
        _configuration = configuration;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var token = context.Request.Headers["Authorization"].ToString().Replace("Bearer ", "");

        if (!string.IsNullOrEmpty(token))
        {
            try
            {
                var tokenHandler = new JwtSecurityTokenHandler();
                var key = Encoding.ASCII.GetBytes(_configuration["Jwt:Secret"]);
                
                tokenHandler.ValidateToken(token, new TokenValidationParameters
                {
                    ValidateIssuerSigningKey = true,
                    IssuerSigningKey = new SymmetricSecurityKey(key),
                    ValidateIssuer = false,
                    ValidateAudience = false
                }, out SecurityToken validatedToken);

                var jwtToken = (JwtSecurityToken)validatedToken;
                var userId = jwtToken.Claims.First(x => x.Type == "id").Value;

                context.Items["UserId"] = userId;
            }
            catch
            {
                // Token validation failed
            }
        }

        await _next(context);
    }
}
```

## Best Practices

### 1. Controller Tasarımı
- Controller'lar ince olmalıdır
- Business logic controller'larda olmamalıdır
- Exception handling merkezi olmalıdır
- Action'lar tek sorumluluğa sahip olmalıdır

### 2. ViewModel Tasarımı
- ViewModel'ler sadece gerekli verileri içermelidir
- Validation logic ViewModel'lerde olmalıdır
- Mapping logic ayrı tutulmalıdır
- ViewModel'ler immutable olmalıdır

### 3. Middleware Tasarımı
- Middleware'ler tek sorumluluğa sahip olmalıdır
- Exception handling merkezi olmalıdır
- Logging stratejisi belirlenmelidir
- Performance optimizasyonu yapılmalıdır

### 4. Security Stratejisi
- Authentication merkezi olmalıdır
- Authorization granular olmalıdır
- Input validation yapılmalıdır
- CORS politikası belirlenmelidir

## Sık Sorulan Sorular

### 1. Presentation Layer neden önemlidir?
- Kullanıcı etkileşimini yönetir
- API kontratlarını tanımlar
- Güvenlik katmanı sağlar
- Hata yönetimini merkezileştirir

### 2. Controller'lar nasıl tasarlanmalıdır?
- İnce olmalıdır
- Business logic içermemelidir
- Exception handling yapmalıdır
- Tek sorumluluğa sahip olmalıdır

### 3. Security nasıl sağlanmalıdır?
- Authentication merkezi olmalıdır
- Authorization granular olmalıdır
- Input validation yapılmalıdır
- CORS politikası belirlenmelidir

## Kaynaklar
- [Microsoft Presentation Layer](https://docs.microsoft.com/tr-tr/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/ddd-oriented-microservice#the-presentation-layer)
- [Clean Architecture - Presentation Layer](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Presentation Layer Best Practices](https://docs.microsoft.com/tr-tr/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/ddd-oriented-microservice#the-presentation-layer) 