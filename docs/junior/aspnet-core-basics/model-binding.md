# Model Binding

## Genel Bakış
Model Binding, ASP.NET Core'da HTTP isteklerinden gelen verileri C# nesnelerine otomatik olarak dönüştüren mekanizmadır. Form verileri, route parametreleri, query string ve request body gibi farklı kaynaklardan veri bağlama işlemi yapar.

## Mülakat Soruları ve Cevapları

### 1. Model Binding nedir ve nasıl çalışır?
**Cevap:**
Model Binding, HTTP isteğinden gelen verileri C# nesnelerine dönüştürme işlemidir. Çalışma prensibi:
- Veri kaynaklarını tarar
- Uygun property'leri bulur
- Tip dönüşümü yapar
- Validation kurallarını uygular

**Örnek Kod:**
```csharp
public class ProductController : Controller
{
    // Form verisi binding
    [HttpPost]
    public IActionResult Create(Product product)
    {
        // ...
    }

    // Query string binding
    [HttpGet]
    public IActionResult Search(string category, decimal? minPrice)
    {
        // ...
    }

    // Route parameter binding
    [HttpGet("{id}")]
    public IActionResult GetById(int id)
    {
        // ...
    }
}
```

### 2. Model Binding kaynakları nelerdir?
**Cevap:**
Model Binding şu kaynaklardan veri alabilir:
- Form data ([FromForm])
- Route data ([FromRoute])
- Query string ([FromQuery])
- Request body ([FromBody])
- Header ([FromHeader])

**Örnek Kod:**
```csharp
public class OrderController : Controller
{
    [HttpPost]
    public IActionResult Create(
        [FromBody] Order order,
        [FromQuery] string promoCode,
        [FromHeader(Name = "X-User-Id")] string userId)
    {
        // ...
    }
}
```

### 3. Complex type binding nasıl yapılır?
**Cevap:**
Complex type binding için:
- Nested property'ler desteklenir
- Collection binding yapılabilir
- Custom model binder kullanılabilir
- Validation attribute'ları eklenebilir

**Örnek Kod:**
```csharp
public class Order
{
    public int Id { get; set; }
    public Customer Customer { get; set; }
    public List<OrderItem> Items { get; set; }
}

public class Customer
{
    public string Name { get; set; }
    public Address ShippingAddress { get; set; }
}

[HttpPost]
public IActionResult CreateOrder(Order order)
{
    // ...
}
```

### 4. Custom model binder nasıl oluşturulur?
**Cevap:**
Custom model binder oluşturmak için:
- IModelBinder interface'i implemente edilir
- BindModelAsync metodu override edilir
- Startup.cs'te kaydedilir
- Attribute ile kullanılır

**Örnek Kod:**
```csharp
public class CustomModelBinder : IModelBinder
{
    public Task BindModelAsync(ModelBindingContext bindingContext)
    {
        var value = bindingContext.ValueProvider.GetValue("custom").FirstValue;
        
        if (string.IsNullOrEmpty(value))
        {
            bindingContext.Result = ModelBindingResult.Failed();
            return Task.CompletedTask;
        }

        bindingContext.Result = ModelBindingResult.Success(value);
        return Task.CompletedTask;
    }
}

[ModelBinder(BinderType = typeof(CustomModelBinder))]
public class CustomModel
{
    public string Value { get; set; }
}
```

### 5. Model binding validation nasıl yapılır?
**Cevap:**
Model binding validation için:
- Data annotations kullanılır
- Custom validation attribute'ları yazılabilir
- IValidatableObject interface'i implemente edilebilir
- ModelState kontrolü yapılır

**Örnek Kod:**
```csharp
public class Product
{
    [Required]
    [StringLength(100)]
    public string Name { get; set; }

    [Range(0, 1000)]
    public decimal Price { get; set; }

    [CustomValidation]
    public string Category { get; set; }
}

[HttpPost]
public IActionResult Create(Product product)
{
    if (!ModelState.IsValid)
    {
        return BadRequest(ModelState);
    }
    // ...
}
```

## Best Practices
1. **Model Tasarımı**
   - Property isimlerini anlamlı seçin
   - Validation kurallarını ekleyin
   - Complex type'ları modüler tutun
   - Interface'leri kullanın

2. **Performans**
   - Gereksiz binding'den kaçının
   - Büyük modelleri parçalayın
   - Async binding kullanın
   - Cache mekanizmalarını değerlendirin

3. **Güvenlik**
   - Input validation yapın
   - Over-posting'e karşı koruyun
   - Sensitive data'ları kontrol edin
   - XSS ve CSRF koruması ekleyin

## Kaynaklar
- [ASP.NET Core Model Binding](https://docs.microsoft.com/tr-tr/aspnet/core/mvc/models/model-binding)
- [Custom Model Binding](https://docs.microsoft.com/tr-tr/aspnet/core/mvc/advanced/custom-model-binding)
- [Model Validation](https://docs.microsoft.com/tr-tr/aspnet/core/mvc/models/validation) 