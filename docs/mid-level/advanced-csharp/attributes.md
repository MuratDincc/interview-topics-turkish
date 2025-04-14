# Attributes

## Genel Bakış
Attributes (Öznitelikler), C#'ta kod elemanlarına (sınıflar, metodlar, özellikler vb.) metadata eklemek için kullanılan yapılardır. Bu metadata'lar, çalışma zamanında reflection ile okunabilir ve kodun davranışını etkileyebilir.

## Mülakat Soruları ve Cevapları

### 1. Attributes nedir ve ne için kullanılır?
**Cevap:**
Attributes kullanım senaryoları:
- Metadata ekleme
- Kod davranışını değiştirme
- Validation kuralları
- Serialization kontrolü
- Dependency injection

**Örnek Kod:**
```csharp
// Temel attribute kullanımı
[Serializable]
public class Student
{
    [Required(ErrorMessage = "İsim alanı zorunludur")]
    [StringLength(50, MinimumLength = 3)]
    public string Name { get; set; }

    [Range(18, 100, ErrorMessage = "Yaş 18-100 arasında olmalıdır")]
    public int Age { get; set; }

    [Obsolete("Bu metod yerine yeni metod kullanılmalıdır", false)]
    public void OldMethod() { }
}

// Custom attribute örneği
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Property)]
public class DisplayNameAttribute : Attribute
{
    public string Name { get; }
    public DisplayNameAttribute(string name) => Name = name;
}
```

### 2. Custom Attributes nasıl oluşturulur?
**Cevap:**
Custom attribute oluşturma adımları:
- Attribute sınıfı tanımlama
- AttributeUsage belirtme
- Constructor tanımlama
- Özellikler ekleme

**Örnek Kod:**
```csharp
// Custom attribute örnekleri
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method, AllowMultiple = true)]
public class AuthorAttribute : Attribute
{
    public string Name { get; }
    public string Version { get; set; }

    public AuthorAttribute(string name)
    {
        Name = name;
        Version = "1.0";
    }
}

[AttributeUsage(AttributeTargets.Property)]
public class ValidationAttribute : Attribute
{
    public string ErrorMessage { get; set; }
    public bool IsRequired { get; set; }

    public ValidationAttribute(bool isRequired)
    {
        IsRequired = isRequired;
    }
}

// Kullanım örnekleri
[Author("John Doe", Version = "2.0")]
public class Document
{
    [Validation(true, ErrorMessage = "Bu alan zorunludur")]
    public string Title { get; set; }
}
```

### 3. Attributes nasıl okunur ve kullanılır?
**Cevap:**
Attribute okuma yöntemleri:
- Reflection kullanımı
- GetCustomAttributes
- Attribute discovery
- Metadata processing

**Örnek Kod:**
```csharp
// Attribute okuma örnekleri
public class AttributeReader
{
    public void ReadAttributes(Type type)
    {
        // Class attribute'ları
        var classAttributes = type.GetCustomAttributes(typeof(AuthorAttribute), false);
        foreach (AuthorAttribute attr in classAttributes)
        {
            Console.WriteLine($"Yazar: {attr.Name}, Versiyon: {attr.Version}");
        }

        // Method attribute'ları
        foreach (var method in type.GetMethods())
        {
            var methodAttributes = method.GetCustomAttributes(typeof(AuthorAttribute), false);
            foreach (AuthorAttribute attr in methodAttributes)
            {
                Console.WriteLine($"Metod: {method.Name}, Yazar: {attr.Name}");
            }
        }

        // Property attribute'ları
        foreach (var prop in type.GetProperties())
        {
            var propAttributes = prop.GetCustomAttributes(typeof(ValidationAttribute), false);
            foreach (ValidationAttribute attr in propAttributes)
            {
                Console.WriteLine($"Özellik: {prop.Name}, Zorunlu: {attr.IsRequired}");
            }
        }
    }
}
```

### 4. Attributes ile validation nasıl yapılır?
**Cevap:**
Validation attribute'ları:
- Data annotations
- Custom validation
- Error messages
- Validation context

**Örnek Kod:**
```csharp
// Validation attribute'ları
public class ValidationAttributes
{
    [Required(ErrorMessage = "Email adresi zorunludur")]
    [EmailAddress(ErrorMessage = "Geçerli bir email adresi giriniz")]
    public string Email { get; set; }

    [StringLength(100, MinimumLength = 6, ErrorMessage = "Şifre 6-100 karakter arasında olmalıdır")]
    [RegularExpression(@"^(?=.*[a-z])(?=.*[A-Z])(?=.*\d).{6,}$", 
        ErrorMessage = "Şifre en az bir büyük harf, bir küçük harf ve bir rakam içermelidir")]
    public string Password { get; set; }

    [Range(0, 120, ErrorMessage = "Yaş 0-120 arasında olmalıdır")]
    public int Age { get; set; }
}

// Validation işlemi
public class Validator
{
    public bool Validate(object obj)
    {
        var validationContext = new ValidationContext(obj);
        var validationResults = new List<ValidationResult>();
        
        bool isValid = Validator.TryValidateObject(
            obj, 
            validationContext, 
            validationResults, 
            true);

        if (!isValid)
        {
            foreach (var result in validationResults)
            {
                Console.WriteLine(result.ErrorMessage);
            }
        }

        return isValid;
    }
}
```

### 5. Attributes performans optimizasyonu nasıl yapılır?
**Cevap:**
Performans optimizasyonu için:
- Attribute caching
- Reflection optimizasyonu
- Lazy loading
- Metadata management

**Örnek Kod:**
```csharp
// Attribute optimizasyonu
public class AttributeCache
{
    private readonly Dictionary<Type, List<Attribute>> _attributeCache = new();
    private readonly Dictionary<string, List<Attribute>> _propertyCache = new();

    public List<Attribute> GetCachedAttributes(Type type)
    {
        if (!_attributeCache.TryGetValue(type, out var attributes))
        {
            attributes = type.GetCustomAttributes().ToList();
            _attributeCache[type] = attributes;
        }
        return attributes;
    }

    public List<Attribute> GetCachedPropertyAttributes(Type type, string propertyName)
    {
        string key = $"{type.FullName}.{propertyName}";
        if (!_propertyCache.TryGetValue(key, out var attributes))
        {
            var property = type.GetProperty(propertyName);
            attributes = property?.GetCustomAttributes().ToList() ?? new List<Attribute>();
            _propertyCache[key] = attributes;
        }
        return attributes;
    }
}

// Lazy loading örneği
public class LazyAttributeLoader
{
    private readonly Lazy<List<Attribute>> _attributes;

    public LazyAttributeLoader(Type type)
    {
        _attributes = new Lazy<List<Attribute>>(() => 
            type.GetCustomAttributes().ToList());
    }

    public List<Attribute> GetAttributes() => _attributes.Value;
}
```

## Best Practices
1. **Kullanım**
   - Uygun attribute'ları seçin
   - Attribute'ları doğru yerde kullanın
   - Documentation ekleyin
   - Error handling yapın

2. **Performans**
   - Attribute caching kullanın
   - Reflection optimizasyonu yapın
   - Lazy loading uygulayın
   - Memory management yapın

3. **Bakım**
   - Kod okunabilirliğini koruyun
   - Test coverage sağlayın
   - Version control yapın
   - Documentation güncelleyin

## Kaynaklar
- [Microsoft Attributes](https://docs.microsoft.com/tr-tr/dotnet/csharp/programming-guide/concepts/attributes/)
- [Custom Attributes](https://docs.microsoft.com/tr-tr/dotnet/standard/attributes/writing-custom-attributes)
- [Data Annotations](https://docs.microsoft.com/tr-tr/dotnet/api/system.componentmodel.dataannotations)
- [Attribute Usage](https://docs.microsoft.com/tr-tr/dotnet/api/system.attributeusageattribute) 