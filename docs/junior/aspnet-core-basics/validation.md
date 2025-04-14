# Validation

## Genel Bakış
Validation, ASP.NET Core uygulamalarında gelen verilerin doğruluğunu ve geçerliliğini kontrol eden mekanizmadır. Data annotations, custom validation ve model state kullanılarak gerçekleştirilir.

## Mülakat Soruları ve Cevapları

### 1. Validation nedir ve neden önemlidir?
**Cevap:**
Validation, veri doğrulama işlemidir. Önemi:
- Veri bütünlüğünü sağlar
- Güvenlik açıklarını önler
- Kullanıcı deneyimini iyileştirir
- Hatalı veri girişini engeller

**Örnek Kod:**
```csharp
public class Product
{
    [Required(ErrorMessage = "Ürün adı zorunludur")]
    [StringLength(100, MinimumLength = 3)]
    public string Name { get; set; }

    [Range(0, 1000, ErrorMessage = "Fiyat 0-1000 arasında olmalıdır")]
    public decimal Price { get; set; }

    [RegularExpression(@"^[A-Z]{2}\d{4}$", ErrorMessage = "Geçersiz ürün kodu")]
    public string ProductCode { get; set; }
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

### 2. Data annotations nedir ve nasıl kullanılır?
**Cevap:**
Data annotations, property'lere eklenen attribute'lar ile validation kuralları tanımlamayı sağlar. Kullanımı:
- Required: Zorunlu alan
- StringLength: String uzunluğu
- Range: Sayısal aralık
- RegularExpression: Regex pattern

**Örnek Kod:**
```csharp
public class User
{
    [Required]
    [EmailAddress]
    public string Email { get; set; }

    [Required]
    [StringLength(100, MinimumLength = 6)]
    [RegularExpression(@"^(?=.*[a-z])(?=.*[A-Z])(?=.*\d).{6,}$")]
    public string Password { get; set; }

    [Compare("Password")]
    public string ConfirmPassword { get; set; }

    [Phone]
    public string PhoneNumber { get; set; }
}
```

### 3. Custom validation nasıl oluşturulur?
**Cevap:**
Custom validation için:
- ValidationAttribute sınıfından türetilir
- IsValid metodu override edilir
- ValidationContext kullanılır
- Error message tanımlanır

**Örnek Kod:**
```csharp
public class CustomValidationAttribute : ValidationAttribute
{
    protected override ValidationResult IsValid(
        object value,
        ValidationContext validationContext)
    {
        if (value == null)
        {
            return ValidationResult.Success;
        }

        var stringValue = value.ToString();
        if (stringValue.Contains("test"))
        {
            return new ValidationResult(
                "Değer 'test' içeremez",
                new[] { validationContext.MemberName });
        }

        return ValidationResult.Success;
    }
}

public class Product
{
    [CustomValidation]
    public string Name { get; set; }
}
```

### 4. IValidatableObject interface'i nasıl kullanılır?
**Cevap:**
IValidatableObject, kompleks validation kuralları için kullanılır:
- Validate metodu implemente edilir
- Cross-property validation yapılabilir
- Business logic validation eklenebilir
- Custom error messages döndürülebilir

**Örnek Kod:**
```csharp
public class Order : IValidatableObject
{
    public DateTime OrderDate { get; set; }
    public DateTime DeliveryDate { get; set; }
    public List<OrderItem> Items { get; set; }

    public IEnumerable<ValidationResult> Validate(
        ValidationContext validationContext)
    {
        if (DeliveryDate < OrderDate)
        {
            yield return new ValidationResult(
                "Teslimat tarihi sipariş tarihinden önce olamaz",
                new[] { nameof(DeliveryDate) });
        }

        if (Items == null || !Items.Any())
        {
            yield return new ValidationResult(
                "Sipariş en az bir ürün içermelidir",
                new[] { nameof(Items) });
        }
    }
}
```

### 5. Client-side validation nasıl yapılır?
**Cevap:**
Client-side validation için:
- jQuery Validation kullanılır
- Unobtrusive validation eklenir
- Data annotations client-side'a taşınır
- Custom validation script'leri yazılabilir

**Örnek Kod:**
```html
<!-- _Layout.cshtml -->
<script src="~/lib/jquery-validation/dist/jquery.validate.min.js"></script>
<script src="~/lib/jquery-validation-unobtrusive/jquery.validate.unobtrusive.min.js"></script>

<!-- View -->
<form asp-action="Create" method="post">
    <div class="form-group">
        <label asp-for="Name"></label>
        <input asp-for="Name" class="form-control" />
        <span asp-validation-for="Name" class="text-danger"></span>
    </div>
    
    <div class="form-group">
        <label asp-for="Price"></label>
        <input asp-for="Price" class="form-control" />
        <span asp-validation-for="Price" class="text-danger"></span>
    </div>
    
    <button type="submit" class="btn btn-primary">Kaydet</button>
</form>
```

## Best Practices
1. **Validation Stratejisi**
   - Hem client hem server validation yapın
   - Anlamlı hata mesajları kullanın
   - Validation kurallarını merkezi tutun
   - Business logic validation'ı ayrı tutun

2. **Performans**
   - Gereksiz validation'dan kaçının
   - Complex validation'ları optimize edin
   - Validation cache kullanın
   - Async validation'ı değerlendirin

3. **Güvenlik**
   - Input sanitization yapın
   - XSS koruması ekleyin
   - SQL injection'a karşı koruyun
   - Rate limiting uygulayın

## Kaynaklar
- [ASP.NET Core Validation](https://docs.microsoft.com/tr-tr/aspnet/core/mvc/models/validation)
- [Custom Validation Attributes](https://docs.microsoft.com/tr-tr/aspnet/core/mvc/models/validation#custom-attributes)
- [Client-side Validation](https://docs.microsoft.com/tr-tr/aspnet/core/mvc/models/validation#client-side-validation) 