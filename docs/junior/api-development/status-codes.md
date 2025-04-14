# Status Codes

## Genel Bakış
HTTP durum kodları, bir HTTP isteğinin sonucunu belirten üç basamaklı sayılardır. RESTful API'lerde, isteğin başarılı olup olmadığını ve sonucun ne olduğunu belirtmek için kullanılır.

## Mülakat Soruları ve Cevapları

### 1. Temel HTTP durum kodları nelerdir ve ne için kullanılır?
**Cevap:**
Temel HTTP durum kodları:
- 1xx: Bilgilendirme
- 2xx: Başarılı
- 3xx: Yönlendirme
- 4xx: İstemci Hatası
- 5xx: Sunucu Hatası

**Örnek Kod:**
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // 200 OK
    [HttpGet]
    public async Task<ActionResult<IEnumerable<Product>>> GetProducts()
    {
        var products = await _productService.GetAllAsync();
        return Ok(products);
    }

    // 201 Created
    [HttpPost]
    public async Task<ActionResult<Product>> CreateProduct(ProductCreateDto productDto)
    {
        var product = await _productService.CreateAsync(productDto);
        return CreatedAtAction(nameof(GetProduct), new { id = product.Id }, product);
    }

    // 204 No Content
    [HttpPut("{id}")]
    public async Task<IActionResult> UpdateProduct(int id, ProductUpdateDto productDto)
    {
        await _productService.UpdateAsync(id, productDto);
        return NoContent();
    }

    // 404 Not Found
    [HttpGet("{id}")]
    public async Task<ActionResult<Product>> GetProduct(int id)
    {
        var product = await _productService.GetByIdAsync(id);
        if (product == null)
        {
            return NotFound();
        }
        return Ok(product);
    }
}
```

### 2. 2xx başarılı durum kodları nelerdir?
**Cevap:**
2xx başarılı durum kodları:
- 200 OK: İstek başarılı
- 201 Created: Kaynak oluşturuldu
- 202 Accepted: İstek kabul edildi
- 204 No Content: İçerik yok
- 206 Partial Content: Kısmi içerik

**Örnek Kod:**
```csharp
// 200 OK
[HttpGet]
public async Task<ActionResult<IEnumerable<Product>>> GetProducts()
{
    return Ok(await _productService.GetAllAsync());
}

// 201 Created
[HttpPost]
public async Task<ActionResult<Product>> CreateProduct(ProductCreateDto productDto)
{
    var product = await _productService.CreateAsync(productDto);
    return CreatedAtAction(nameof(GetProduct), new { id = product.Id }, product);
}

// 202 Accepted
[HttpPost("process")]
public async Task<ActionResult> ProcessOrder(int orderId)
{
    _backgroundJobService.Enqueue(() => _orderService.ProcessAsync(orderId));
    return Accepted();
}

// 204 No Content
[HttpPut("{id}")]
public async Task<IActionResult> UpdateProduct(int id, ProductUpdateDto productDto)
{
    await _productService.UpdateAsync(id, productDto);
    return NoContent();
}
```

### 3. 4xx istemci hata kodları nelerdir?
**Cevap:**
4xx istemci hata kodları:
- 400 Bad Request: Geçersiz istek
- 401 Unauthorized: Yetkisiz erişim
- 403 Forbidden: Erişim engellendi
- 404 Not Found: Kaynak bulunamadı
- 409 Conflict: Çakışma

**Örnek Kod:**
```csharp
// 400 Bad Request
[HttpPost]
public async Task<ActionResult<Product>> CreateProduct(ProductCreateDto productDto)
{
    if (!ModelState.IsValid)
    {
        return BadRequest(ModelState);
    }
    // ...
}

// 401 Unauthorized
[HttpGet("secure")]
[Authorize]
public async Task<ActionResult> GetSecureData()
{
    // ...
}

// 403 Forbidden
[HttpDelete("{id}")]
[Authorize(Roles = "Admin")]
public async Task<IActionResult> DeleteProduct(int id)
{
    // ...
}

// 404 Not Found
[HttpGet("{id}")]
public async Task<ActionResult<Product>> GetProduct(int id)
{
    var product = await _productService.GetByIdAsync(id);
    if (product == null)
    {
        return NotFound();
    }
    return Ok(product);
}

// 409 Conflict
[HttpPost]
public async Task<ActionResult<Product>> CreateProduct(ProductCreateDto productDto)
{
    if (await _productService.ExistsAsync(productDto.Name))
    {
        return Conflict("Product already exists");
    }
    // ...
}
```

### 4. 5xx sunucu hata kodları nelerdir?
**Cevap:**
5xx sunucu hata kodları:
- 500 Internal Server Error: Sunucu hatası
- 501 Not Implemented: Uygulanmamış
- 502 Bad Gateway: Geçersiz ağ geçidi
- 503 Service Unavailable: Servis kullanılamıyor
- 504 Gateway Timeout: Ağ geçidi zaman aşımı

**Örnek Kod:**
```csharp
// Global hata yönetimi
public class GlobalExceptionHandler
{
    public async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        context.Response.ContentType = "application/json";
        
        switch (exception)
        {
            case ValidationException:
                context.Response.StatusCode = 400;
                break;
            case NotFoundException:
                context.Response.StatusCode = 404;
                break;
            case UnauthorizedException:
                context.Response.StatusCode = 401;
                break;
            default:
                context.Response.StatusCode = 500;
                break;
        }

        await context.Response.WriteAsync(new ErrorDetails
        {
            StatusCode = context.Response.StatusCode,
            Message = exception.Message
        }.ToString());
    }
}
```

### 5. Özel durum kodları nasıl oluşturulur?
**Cevap:**
Özel durum kodları için:
- ActionResult dönüş tipi kullanın
- StatusCode metodunu kullanın
- ProblemDetails sınıfını kullanın
- Özel hata sınıfları oluşturun

**Örnek Kod:**
```csharp
// Özel durum kodu örneği
[HttpPost("custom")]
public async Task<ActionResult> CustomAction()
{
    // Özel durum kodu
    return StatusCode(418, new { Message = "I'm a teapot" });

    // ProblemDetails kullanımı
    return Problem(
        title: "Custom Error",
        detail: "Something went wrong",
        statusCode: 418
    );

    // Özel hata sınıfı
    return StatusCode(418, new CustomError
    {
        Code = "TEAPOT_ERROR",
        Message = "I'm a teapot",
        Details = "Cannot brew coffee"
    });
}
```

## Best Practices
1. **Durum Kodu Seçimi**
   - Doğru durum kodunu kullanın
   - Tutarlı olun
   - Açıklayıcı hata mesajları verin
   - ProblemDetails kullanın

2. **Hata Yönetimi**
   - Global hata yakalama kullanın
   - Özel hata sınıfları oluşturun
   - Loglama yapın
   - Hata detaylarını güvenli şekilde iletin

3. **Dokümantasyon**
   - Durum kodlarını dokümante edin
   - Örnek yanıtlar ekleyin
   - Hata senaryolarını açıklayın
   - Çözüm önerileri sunun

## Kaynaklar
- [HTTP Status Codes](https://developer.mozilla.org/tr/docs/Web/HTTP/Status)
- [Problem Details for HTTP APIs](https://tools.ietf.org/html/rfc7807)
- [ASP.NET Core Web API Error Handling](https://docs.microsoft.com/tr-tr/aspnet/core/web-api/handle-errors) 