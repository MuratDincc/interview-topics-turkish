# Routing

## Genel Bakış
Routing, ASP.NET Core uygulamalarında gelen HTTP isteklerini ilgili controller action'larına yönlendiren mekanizmadır. İki temel routing yaklaşımı vardır: convention-based routing ve attribute routing.

## Mülakat Soruları ve Cevapları

### 1. Routing nedir ve nasıl çalışır?
**Cevap:**
Routing, URL'leri controller action'larına eşleştiren bir mekanizmadır. Çalışma prensibi:
- URL'yi parse eder
- Route template'leri kontrol eder
- Route değerlerini çıkarır
- Controller ve action'ı belirler

**Örnek Kod:**
```csharp
// Convention-based routing
app.UseEndpoints(endpoints =>
{
    endpoints.MapControllerRoute(
        name: "default",
        pattern: "{controller=Home}/{action=Index}/{id?}");
});

// Attribute routing
[Route("api/[controller]")]
public class ProductsController : Controller
{
    [HttpGet("{id}")]
    public IActionResult GetProduct(int id)
    {
        // ...
    }
}
```

### 2. Convention-based routing ve attribute routing arasındaki farklar nelerdir?
**Cevap:**
Convention-based routing:
- Startup.cs'te tanımlanır
- Global route kuralları
- Daha az esnek
- Legacy uygulamalar için uygun

Attribute routing:
- Controller/action seviyesinde tanımlanır
- Daha esnek ve okunabilir
- RESTful API'ler için ideal
- Modern yaklaşım

**Örnek Kod:**
```csharp
// Convention-based
app.UseEndpoints(endpoints =>
{
    endpoints.MapControllerRoute(
        name: "blog",
        pattern: "blog/{*article}",
        defaults: new { controller = "Blog", action = "Article" });
});

// Attribute
[Route("api/[controller]")]
public class BlogController : Controller
{
    [HttpGet("{article}")]
    public IActionResult Article(string article)
    {
        // ...
    }
}
```

### 3. Route constraints nedir ve nasıl kullanılır?
**Cevap:**
Route constraints, route parametrelerine kısıtlama getirmek için kullanılır:
- Veri tipi kontrolü
- Regex pattern kontrolü
- Min/max değer kontrolü
- Custom constraint'ler

**Örnek Kod:**
```csharp
// Convention-based constraints
app.UseEndpoints(endpoints =>
{
    endpoints.MapControllerRoute(
        name: "default",
        pattern: "{controller=Home}/{action=Index}/{id:int}");
});

// Attribute constraints
[Route("api/[controller]")]
public class ProductsController : Controller
{
    [HttpGet("{id:int:min(1)}")]
    public IActionResult GetProduct(int id)
    {
        // ...
    }

    [HttpGet("{name:alpha}")]
    public IActionResult GetProductByName(string name)
    {
        // ...
    }
}
```

### 4. Route parameters nasıl kullanılır?
**Cevap:**
Route parametreleri şu şekilde kullanılabilir:
- Query string parametreleri
- Route segment parametreleri
- Optional parametreler
- Default değerler

**Örnek Kod:**
```csharp
[Route("api/[controller]")]
public class ProductsController : Controller
{
    // Query string: /api/products?category=electronics
    [HttpGet]
    public IActionResult GetByCategory(string category)
    {
        // ...
    }

    // Route parameter: /api/products/1
    [HttpGet("{id}")]
    public IActionResult GetById(int id)
    {
        // ...
    }

    // Optional parameter: /api/products/search/electronics
    [HttpGet("search/{category?}")]
    public IActionResult Search(string category = "all")
    {
        // ...
    }
}
```

### 5. Custom route handler nasıl oluşturulur?
**Cevap:**
Custom route handler oluşturmak için:
- IRouteConstraint interface'i implemente edilir
- Match metodu override edilir
- Startup.cs'te kaydedilir
- Route'larda kullanılır

**Örnek Kod:**
```csharp
public class CustomRouteConstraint : IRouteConstraint
{
    public bool Match(
        HttpContext httpContext,
        IRouter route,
        string routeKey,
        RouteValueDictionary values,
        RouteDirection routeDirection)
    {
        // Custom logic
        return true;
    }
}

// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.Configure<RouteOptions>(options =>
    {
        options.ConstraintMap.Add("custom", typeof(CustomRouteConstraint));
    });
}

// Usage
[Route("api/[controller]")]
public class ProductsController : Controller
{
    [HttpGet("{id:custom}")]
    public IActionResult GetProduct(string id)
    {
        // ...
    }
}
```

## Best Practices
1. **Route Tasarımı**
   - RESTful prensipleri takip edin
   - Anlamlı URL'ler kullanın
   - Versioning için prefix kullanın
   - Route'ları modüler tutun

2. **Performans**
   - Route sayısını minimize edin
   - Complex constraint'lerden kaçının
   - Route cache'ini kullanın
   - Route order'ı optimize edin

3. **Güvenlik**
   - Route injection'dan kaçının
   - Sensitive data'ları route'ta kullanmayın
   - Authorization kontrolü yapın
   - Rate limiting uygulayın

## Kaynaklar
- [ASP.NET Core Routing](https://docs.microsoft.com/tr-tr/aspnet/core/fundamentals/routing)
- [Route Constraints](https://docs.microsoft.com/tr-tr/aspnet/core/fundamentals/routing#route-constraints)
- [Attribute Routing](https://docs.microsoft.com/tr-tr/aspnet/core/mvc/controllers/routing#attribute-routing) 