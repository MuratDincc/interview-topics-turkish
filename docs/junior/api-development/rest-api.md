# REST API

## Genel Bakış
REST (Representational State Transfer), web servisleri için bir mimari stildir. RESTful API'ler, HTTP protokolünü kullanarak kaynaklara erişim sağlar ve istemci-sunucu arasındaki iletişimi standartlaştırır.

## Mülakat Soruları ve Cevapları

### 1. REST nedir ve temel prensipleri nelerdir?
**Cevap:**
REST'in temel prensipleri:
- Client-Server: İstemci ve sunucu birbirinden bağımsız
- Stateless: Her istek kendi içinde bağımsız
- Cacheable: İstekler önbelleğe alınabilir
- Uniform Interface: Standart arayüz
- Layered System: Katmanlı mimari
- Code on Demand (opsiyonel): İstemciye kod gönderilebilir

**Örnek Kod:**
```csharp
// RESTful controller örneği
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;

    public ProductsController(IProductService productService)
    {
        _productService = productService;
    }

    [HttpGet]
    public async Task<ActionResult<IEnumerable<Product>>> GetProducts()
    {
        var products = await _productService.GetAllAsync();
        return Ok(products);
    }

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

### 2. Resource naming kuralları nelerdir?
**Cevap:**
Resource naming kuralları:
- İsimler çoğul olmalı
- Küçük harf kullanılmalı
- Tire (-) ile kelimeler ayrılmalı
- İç içe kaynaklar için hiyerarşik yapı
- Anlamlı isimler seçilmeli

**Örnek Kod:**
```csharp
// Doğru resource naming örnekleri
[Route("api/products")]
[Route("api/products/{productId}/reviews")]
[Route("api/users/{userId}/orders")]
[Route("api/categories/{categoryId}/products")]

// Yanlış resource naming örnekleri
[Route("api/getProducts")]
[Route("api/Product")]
[Route("api/product_list")]
```

### 3. HATEOAS nedir ve nasıl kullanılır?
**Cevap:**
HATEOAS (Hypermedia as the Engine of Application State):
- Kaynaklar arası ilişkileri linklerle gösterir
- İstemciye sonraki adımları gösterir
- API'nin keşfedilebilirliğini artırır
- Bağımlılıkları azaltır

**Örnek Kod:**
```csharp
// HATEOAS örneği
public class ProductResource
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public List<Link> Links { get; set; }
}

public class Link
{
    public string Href { get; set; }
    public string Rel { get; set; }
    public string Method { get; set; }
}

// Controller'da kullanımı
[HttpGet("{id}")]
public async Task<ActionResult<ProductResource>> GetProduct(int id)
{
    var product = await _productService.GetByIdAsync(id);
    if (product == null)
    {
        return NotFound();
    }

    var resource = new ProductResource
    {
        Id = product.Id,
        Name = product.Name,
        Price = product.Price,
        Links = new List<Link>
        {
            new Link { Href = $"/api/products/{id}", Rel = "self", Method = "GET" },
            new Link { Href = $"/api/products/{id}", Rel = "update", Method = "PUT" },
            new Link { Href = $"/api/products/{id}", Rel = "delete", Method = "DELETE" }
        }
    };

    return Ok(resource);
}
```

### 4. RESTful API'lerde hata yönetimi nasıl yapılır?
**Cevap:**
Hata yönetimi için:
- Standart HTTP status code'ları kullanın
- Hata mesajlarını detaylı verin
- Hata formatını standartlaştırın
- Validation hatalarını ayrı ele alın

**Örnek Kod:**
```csharp
// Hata yönetimi örneği
public class ApiResponse<T>
{
    public bool Success { get; set; }
    public T Data { get; set; }
    public string Message { get; set; }
    public List<string> Errors { get; set; }
}

[HttpPost]
public async Task<ActionResult<ApiResponse<Product>>> CreateProduct(ProductCreateDto productDto)
{
    if (!ModelState.IsValid)
    {
        return BadRequest(new ApiResponse<Product>
        {
            Success = false,
            Message = "Validation error",
            Errors = ModelState.Values
                .SelectMany(v => v.Errors)
                .Select(e => e.ErrorMessage)
                .ToList()
        });
    }

    try
    {
        var product = await _productService.CreateAsync(productDto);
        return Ok(new ApiResponse<Product>
        {
            Success = true,
            Data = product,
            Message = "Product created successfully"
        });
    }
    catch (Exception ex)
    {
        return StatusCode(500, new ApiResponse<Product>
        {
            Success = false,
            Message = "An error occurred",
            Errors = new List<string> { ex.Message }
        });
    }
}
```

### 5. RESTful API'lerde pagination nasıl uygulanır?
**Cevap:**
Pagination için:
- Query parametreleri kullanın
- Metadata bilgisi ekleyin
- Link header'ları kullanın
- Sayfa boyutunu sınırlayın

**Örnek Kod:**
```csharp
// Pagination örneği
public class PagedResponse<T>
{
    public IEnumerable<T> Items { get; set; }
    public int PageNumber { get; set; }
    public int PageSize { get; set; }
    public int TotalPages { get; set; }
    public int TotalItems { get; set; }
    public bool HasPrevious => PageNumber > 1;
    public bool HasNext => PageNumber < TotalPages;
}

[HttpGet]
public async Task<ActionResult<PagedResponse<Product>>> GetProducts(
    [FromQuery] int pageNumber = 1,
    [FromQuery] int pageSize = 10)
{
    var (products, totalItems) = await _productService.GetPagedAsync(pageNumber, pageSize);
    
    var response = new PagedResponse<Product>
    {
        Items = products,
        PageNumber = pageNumber,
        PageSize = pageSize,
        TotalItems = totalItems,
        TotalPages = (int)Math.Ceiling(totalItems / (double)pageSize)
    };

    return Ok(response);
}
```

## Best Practices
1. **API Tasarımı**
   - REST prensiplerine uyun
   - Anlamlı resource isimleri kullanın
   - HTTP metodlarını doğru kullanın
   - HATEOAS uygulayın

2. **Hata Yönetimi**
   - Standart HTTP status code'ları kullanın
   - Detaylı hata mesajları verin
   - Validation hatalarını ayrı ele alın
   - Loglama yapın

3. **Performans**
   - Pagination kullanın
   - Caching uygulayın
   - Response compression kullanın
   - Gereksiz veri göndermeyin

## Kaynaklar
- [REST API Best Practices](https://docs.microsoft.com/tr-tr/azure/architecture/best-practices/api-design)
- [REST Architectural Style](https://docs.microsoft.com/tr-tr/azure/architecture/best-practices/api-design#rest-architectural-style)
- [HATEOAS](https://docs.microsoft.com/tr-tr/azure/architecture/best-practices/api-design#hateoas) 