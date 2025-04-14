# Single Responsibility Principle (SRP)

## Genel Bakış
Single Responsibility Principle (SRP), bir sınıfın sadece bir sorumluluğu olması gerektiğini belirtir. Bu prensip, kodun daha anlaşılır, bakımı kolay ve test edilebilir olmasını sağlar.

## Mülakat Soruları ve Cevapları

### 1. Single Responsibility Principle nedir ve neden önemlidir?
**Cevap:**
Single Responsibility Principle, bir sınıfın sadece bir değişiklik nedeni olması gerektiğini belirtir. Önemlidir çünkü:
- Kodun bakımını kolaylaştırır
- Test edilebilirliği artırır
- Değişikliklerin etkisini sınırlar
- Kodun anlaşılabilirliğini artırır

**Örnek Kod:**
```csharp
// SRP'ye uymayan kod
public class User
{
    public string Name { get; set; }
    public string Email { get; set; }
    
    public void SaveToDatabase()
    {
        // Veritabanına kaydetme işlemi
    }
    
    public void SendEmail()
    {
        // Email gönderme işlemi
    }
}

// SRP'ye uyan kod
public class User
{
    public string Name { get; set; }
    public string Email { get; set; }
}

public class UserRepository
{
    public void Save(User user)
    {
        // Veritabanına kaydetme işlemi
    }
}

public class EmailService
{
    public void SendEmail(User user)
    {
        // Email gönderme işlemi
    }
}
```

### 2. SRP'yi ihlal eden durumları nasıl tespit edebiliriz?
**Cevap:**
SRP ihlallerini tespit etmek için:
- Sınıfın birden fazla sorumluluğu olup olmadığını kontrol edin
- "Ve" kelimesi kullanıyorsanız, muhtemelen birden fazla sorumluluk var
- Sınıfın değişiklik nedenlerini analiz edin
- Metotların birbiriyle ilişkisini inceleyin

**Örnek Kod:**
```csharp
// SRP ihlali
public class ReportGenerator
{
    public void GenerateReport()
    {
        // Rapor oluşturma
    }
    
    public void SaveToFile()
    {
        // Dosyaya kaydetme
    }
    
    public void SendToPrinter()
    {
        // Yazıcıya gönderme
    }
}

// SRP'ye uygun
public class ReportGenerator
{
    public Report GenerateReport()
    {
        // Sadece rapor oluşturma
        return new Report();
    }
}

public class ReportSaver
{
    public void SaveToFile(Report report)
    {
        // Sadece dosyaya kaydetme
    }
}

public class ReportPrinter
{
    public void Print(Report report)
    {
        // Sadece yazdırma
    }
}
```

### 3. SRP'yi uygularken dikkat edilmesi gereken noktalar nelerdir?
**Cevap:**
SRP uygularken dikkat edilmesi gerekenler:
- Sorumlulukları doğru belirleyin
- Aşırı parçalama yapmayın
- İlgili sorumlulukları bir arada tutun
- Bağımlılıkları minimize edin

**Örnek Kod:**
```csharp
// Aşırı parçalama
public class UserName
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
}

public class UserEmail
{
    public string Email { get; set; }
}

// Daha uygun
public class User
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Email { get; set; }
}
```

### 4. SRP ile ilgili yaygın hatalar nelerdir?
**Cevap:**
Yaygın hatalar:
- Sorumlulukları yanlış belirleme
- Aşırı parçalama
- İlgili sorumlulukları ayırma
- Bağımlılıkları artırma

**Örnek Kod:**
```csharp
// Yaygın hata: Aşırı parçalama
public class UserFirstName
{
    public string Value { get; set; }
}

public class UserLastName
{
    public string Value { get; set; }
}

// Daha uygun
public class UserName
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
}
```

### 5. SRP'yi gerçek dünya senaryolarında nasıl uygularız?
**Cevap:**
Gerçek dünya senaryolarında:
- Domain-driven design yaklaşımını kullanın
- Bounded context'leri belirleyin
- Aggregate root'ları doğru tanımlayın
- Value object'leri kullanın

**Örnek Kod:**
```csharp
public class Order
{
    public int Id { get; private set; }
    public OrderStatus Status { get; private set; }
    private readonly List<OrderItem> _items;
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();

    public void AddItem(Product product, int quantity)
    {
        // Sipariş öğesi ekleme mantığı
    }

    public void ProcessPayment(Payment payment)
    {
        // Ödeme işleme mantığı
    }
}

public class OrderRepository
{
    public void Save(Order order)
    {
        // Siparişi veritabanına kaydetme
    }
}

public class PaymentProcessor
{
    public void ProcessPayment(Order order, Payment payment)
    {
        // Ödeme işleme mantığı
    }
}
```

## Best Practices
1. **Sorumluluk Belirleme**
   - Sorumlulukları net tanımlayın
   - İlgili sorumlulukları gruplayın
   - Aşırı parçalamaktan kaçının
   - Domain-driven design yaklaşımını kullanın

2. **Kod Organizasyonu**
   - İlgili sınıfları aynı namespace'te tutun
   - Bağımlılıkları minimize edin
   - Interface'leri küçük tutun
   - Dependency injection kullanın

3. **Test Edilebilirlik**
   - Unit testler yazın
   - Mock nesneler kullanın
   - Test coverage'ı takip edin
   - Test edilebilir kod yazın

## Kaynaklar
- [Microsoft Single Responsibility Principle](https://docs.microsoft.com/tr-tr/dotnet/architecture/modern-web-apps-azure/architectural-principles#single-responsibility-principle)
- [SOLID Principles in C#](https://docs.microsoft.com/tr-tr/dotnet/architecture/modern-web-apps-azure/architectural-principles#solid-principles)
- [Single Responsibility Principle: Explanation and Examples](https://docs.microsoft.com/tr-tr/dotnet/architecture/modern-web-apps-azure/architectural-principles#single-responsibility-principle) 