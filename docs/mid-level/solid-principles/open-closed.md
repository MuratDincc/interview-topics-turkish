# Open/Closed Principle (OCP)

## Genel Bakış
Open/Closed Principle (OCP), yazılım varlıklarının (sınıflar, modüller, fonksiyonlar vb.) genişletmeye açık, değişikliğe kapalı olması gerektiğini belirtir. Bu prensip, mevcut kodu değiştirmeden yeni özellikler eklememizi sağlar.

## Mülakat Soruları ve Cevapları

### 1. Open/Closed Principle nedir ve neden önemlidir?
**Cevap:**
Open/Closed Principle, yazılım varlıklarının genişletmeye açık, değişikliğe kapalı olması gerektiğini belirtir. Önemlidir çünkü:
- Mevcut kodu değiştirmeden yeni özellikler eklememizi sağlar
- Kodun bakımını kolaylaştırır
- Hata riskini azaltır
- Test edilebilirliği artırır

**Örnek Kod:**
```csharp
// OCP'ye uymayan kod
public class AreaCalculator
{
    public double CalculateArea(object shape)
    {
        if (shape is Rectangle)
        {
            var rectangle = (Rectangle)shape;
            return rectangle.Width * rectangle.Height;
        }
        else if (shape is Circle)
        {
            var circle = (Circle)shape;
            return Math.PI * circle.Radius * circle.Radius;
        }
        throw new ArgumentException("Bilinmeyen şekil");
    }
}

// OCP'ye uyan kod
public abstract class Shape
{
    public abstract double CalculateArea();
}

public class Rectangle : Shape
{
    public double Width { get; set; }
    public double Height { get; set; }

    public override double CalculateArea()
    {
        return Width * Height;
    }
}

public class Circle : Shape
{
    public double Radius { get; set; }

    public override double CalculateArea()
    {
        return Math.PI * Radius * Radius;
    }
}
```

### 2. OCP'yi ihlal eden durumları nasıl tespit edebiliriz?
**Cevap:**
OCP ihlallerini tespit etmek için:
- Switch-case veya if-else bloklarını kontrol edin
- Yeni özellik eklemek için mevcut kodu değiştirmeniz gerekiyorsa
- Sınıfın birden fazla sorumluluğu varsa
- Kod tekrarı varsa

**Örnek Kod:**
```csharp
// OCP ihlali
public class PaymentProcessor
{
    public void ProcessPayment(string paymentType, decimal amount)
    {
        if (paymentType == "CreditCard")
        {
            // Kredi kartı işlemi
        }
        else if (paymentType == "PayPal")
        {
            // PayPal işlemi
        }
        else if (paymentType == "BankTransfer")
        {
            // Banka transferi işlemi
        }
    }
}

// OCP'ye uygun
public interface IPaymentProcessor
{
    void ProcessPayment(decimal amount);
}

public class CreditCardPaymentProcessor : IPaymentProcessor
{
    public void ProcessPayment(decimal amount)
    {
        // Kredi kartı işlemi
    }
}

public class PayPalPaymentProcessor : IPaymentProcessor
{
    public void ProcessPayment(decimal amount)
    {
        // PayPal işlemi
    }
}
```

### 3. OCP'yi uygularken dikkat edilmesi gereken noktalar nelerdir?
**Cevap:**
OCP uygularken dikkat edilmesi gerekenler:
- Soyutlamaları doğru seviyede yapın
- Interface'leri küçük ve öz tutun
- Dependency injection kullanın
- Strategy pattern gibi tasarım desenlerini kullanın

**Örnek Kod:**
```csharp
public interface ILogger
{
    void Log(string message);
}

public class FileLogger : ILogger
{
    public void Log(string message)
    {
        // Dosyaya loglama
    }
}

public class DatabaseLogger : ILogger
{
    public void Log(string message)
    {
        // Veritabanına loglama
    }
}

public class Application
{
    private readonly ILogger _logger;

    public Application(ILogger logger)
    {
        _logger = logger;
    }

    public void DoSomething()
    {
        _logger.Log("İşlem başladı");
        // İşlemler
        _logger.Log("İşlem tamamlandı");
    }
}
```

### 4. OCP ile ilgili yaygın hatalar nelerdir?
**Cevap:**
Yaygın hatalar:
- Aşırı soyutlama
- Gereksiz interface'ler
- Yanlış seviyede soyutlama
- Performans kaybı

**Örnek Kod:**
```csharp
// Yaygın hata: Aşırı soyutlama
public interface IDataAccess
{
    void Connect();
    void Disconnect();
    void ExecuteQuery(string query);
    void BeginTransaction();
    void CommitTransaction();
    void RollbackTransaction();
}

// Daha uygun
public interface IRepository<T>
{
    void Add(T entity);
    void Update(T entity);
    void Delete(T entity);
    T GetById(int id);
}
```

### 5. OCP'yi gerçek dünya senaryolarında nasıl uygularız?
**Cevap:**
Gerçek dünya senaryolarında:
- Plugin mimarisi kullanın
- Strategy pattern uygulayın
- Factory pattern kullanın
- Dependency injection kullanın

**Örnek Kod:**
```csharp
public interface IExportStrategy
{
    void Export(Report report);
}

public class PdfExportStrategy : IExportStrategy
{
    public void Export(Report report)
    {
        // PDF'e dışa aktarma
    }
}

public class ExcelExportStrategy : IExportStrategy
{
    public void Export(Report report)
    {
        // Excel'e dışa aktarma
    }
}

public class ReportExporter
{
    private readonly IExportStrategy _exportStrategy;

    public ReportExporter(IExportStrategy exportStrategy)
    {
        _exportStrategy = exportStrategy;
    }

    public void Export(Report report)
    {
        _exportStrategy.Export(report);
    }
}
```

## Best Practices
1. **Soyutlama Seviyesi**
   - Soyutlamaları doğru seviyede yapın
   - Interface'leri küçük tutun
   - Bağımlılıkları minimize edin
   - Dependency injection kullanın

2. **Kod Organizasyonu**
   - Plugin mimarisi kullanın
   - Strategy pattern uygulayın
   - Factory pattern kullanın
   - Interface segregation uygulayın

3. **Test Edilebilirlik**
   - Unit testler yazın
   - Mock nesneler kullanın
   - Test coverage'ı takip edin
   - Test edilebilir kod yazın

## Kaynaklar
- [Microsoft Open/Closed Principle](https://docs.microsoft.com/tr-tr/dotnet/architecture/modern-web-apps-azure/architectural-principles#openclosed-principle)
- [SOLID Principles in C#](https://docs.microsoft.com/tr-tr/dotnet/architecture/modern-web-apps-azure/architectural-principles#solid-principles)
- [Open/Closed Principle: Explanation and Examples](https://docs.microsoft.com/tr-tr/dotnet/architecture/modern-web-apps-azure/architectural-principles#openclosed-principle) 