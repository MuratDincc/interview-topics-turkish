# HTTP Methods

## Genel Bakış
HTTP metodları, RESTful API'lerde kaynaklar üzerinde gerçekleştirilecek işlemleri belirten standart HTTP istek tipleridir. Her metodun belirli bir amacı ve kullanım alanı vardır.

## Mülakat Soruları ve Cevapları

### 1. Temel HTTP metodları nelerdir ve ne için kullanılır?
**Cevap:**
Temel HTTP metodları:
- GET: Kaynakları okumak için
- POST: Yeni kaynak oluşturmak için
- PUT: Kaynağı tamamen güncellemek için
- PATCH: Kaynağın bir kısmını güncellemek için
- DELETE: Kaynağı silmek için
- HEAD: Kaynak hakkında meta bilgi almak için
- OPTIONS: Sunucunun desteklediği metodları öğrenmek için

**Örnek Kod:**
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // GET: api/products
    [HttpGet]
    public async Task<ActionResult<IEnumerable<Product>>> GetProducts()
    {
        return await _productService.GetAllAsync();
    }

    // POST: api/products
    [HttpPost]
    public async Task<ActionResult<Product>> CreateProduct(ProductCreateDto productDto)
    {
        var product = await _productService.CreateAsync(productDto);
        return CreatedAtAction(nameof(GetProduct), new { id = product.Id }, product);
    }

    // PUT: api/products/5
    [HttpPut("{id}")]
    public async Task<IActionResult> UpdateProduct(int id, ProductUpdateDto productDto)
    {
        await _productService.UpdateAsync(id, productDto);
        return NoContent();
    }

    // DELETE: api/products/5
    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteProduct(int id)
    {
        await _productService.DeleteAsync(id);
        return NoContent();
    }
}
```

### 2. GET ve POST metodları arasındaki farklar nelerdir?
**Cevap:**
GET ve POST farkları:
- GET: Verileri URL'de taşır, POST: Request body'de taşır
- GET: Önbelleğe alınabilir, POST: Önbelleğe alınamaz
- GET: Güvenli metod, POST: Güvenli değil
- GET: Idempotent, POST: Idempotent değil
- GET: Veri boyutu sınırlı, POST: Sınırsız

**Örnek Kod:**
```csharp
// GET örneği
[HttpGet("search")]
public async Task<ActionResult<IEnumerable<Product>>> SearchProducts(
    [FromQuery] string name,
    [FromQuery] decimal? minPrice,
    [FromQuery] decimal? maxPrice)
{
    return await _productService.SearchAsync(name, minPrice, maxPrice);
}

// POST örneği
[HttpPost("complex-search")]
public async Task<ActionResult<IEnumerable<Product>>> ComplexSearch(
    [FromBody] ProductSearchDto searchDto)
{
    return await _productService.ComplexSearchAsync(searchDto);
}
```

### 3. PUT ve PATCH metodları arasındaki farklar nelerdir?
**Cevap:**
PUT ve PATCH farkları:
- PUT: Tüm kaynağı günceller
- PATCH: Kaynağın belirli alanlarını günceller
- PUT: Idempotent
- PATCH: Idempotent olmayabilir
- PUT: Tüm alanları göndermek zorunlu
- PATCH: Sadece değişen alanları gönderir

**Örnek Kod:**
```csharp
// PUT örneği
[HttpPut("{id}")]
public async Task<IActionResult> UpdateProduct(int id, ProductUpdateDto productDto)
{
    // Tüm alanlar güncellenir
    await _productService.UpdateAsync(id, productDto);
    return NoContent();
}

// PATCH örneği
[HttpPatch("{id}")]
public async Task<IActionResult> PatchProduct(int id, JsonPatchDocument<ProductUpdateDto> patchDoc)
{
    // Sadece belirtilen alanlar güncellenir
    var product = await _productService.GetByIdAsync(id);
    if (product == null)
    {
        return NotFound();
    }

    var productDto = new ProductUpdateDto();
    patchDoc.ApplyTo(productDto);
    await _productService.PatchAsync(id, productDto);
    return NoContent();
}
```

### 4. HTTP metodlarının güvenlik özellikleri nelerdir?
**Cevap:**
HTTP metodlarının güvenlik özellikleri:
- Safe Methods: GET, HEAD, OPTIONS
- Idempotent Methods: GET, HEAD, PUT, DELETE
- Cacheable Methods: GET, HEAD
- Body Allowed: POST, PUT, PATCH

**Örnek Kod:**
```csharp
// Güvenli metod örneği
[HttpGet]
public async Task<ActionResult<IEnumerable<Product>>> GetProducts()
{
    return await _productService.GetAllAsync();
}

// Güvenli olmayan metod örneği
[HttpPost]
public async Task<ActionResult<Product>> CreateProduct(ProductCreateDto productDto)
{
    // CSRF token kontrolü
    if (!await _antiForgeryService.ValidateRequestAsync())
    {
        return BadRequest("Invalid request");
    }

    var product = await _productService.CreateAsync(productDto);
    return CreatedAtAction(nameof(GetProduct), new { id = product.Id }, product);
}
```

### 5. HTTP metodlarının performans etkileri nelerdir?
**Cevap:**
Performans etkileri:
- GET: Önbelleklenebilir, hızlı
- POST: Her seferinde yeni kaynak oluşturur
- PUT: Tüm kaynağı günceller
- PATCH: Sadece değişen kısmı günceller
- DELETE: Kaynak silinir

**Örnek Kod:**
```csharp
// Performans optimizasyonu örneği
[HttpGet]
[ResponseCache(Duration = 60)] // 60 saniye önbellekleme
public async Task<ActionResult<IEnumerable<Product>>> GetProducts()
{
    return await _productService.GetAllAsync();
}

// Batch işlem örneği
[HttpPost("batch")]
public async Task<ActionResult> CreateProducts([FromBody] List<ProductCreateDto> products)
{
    await _productService.CreateBatchAsync(products);
    return Ok();
}
```

## Best Practices
1. **Metod Seçimi**
   - Amaca uygun metod seçin
   - REST prensiplerine uyun
   - Idempotent metodları tercih edin
   - Güvenli metodları dikkatli kullanın

2. **Güvenlik**
   - CSRF koruması ekleyin
   - Input validasyonu yapın
   - Yetkilendirme kontrolleri yapın
   - Rate limiting uygulayın

3. **Performans**
   - Önbellekleme kullanın
   - Batch işlemleri tercih edin
   - Gereksiz veri transferinden kaçının
   - Compression kullanın

## Kaynaklar
- [HTTP Methods](https://developer.mozilla.org/tr/docs/Web/HTTP/Methods)
- [REST API Best Practices](https://docs.microsoft.com/tr-tr/azure/architecture/best-practices/api-design)
- [HTTP/1.1 Specification](https://tools.ietf.org/html/rfc2616) 