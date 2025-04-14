# Dependency Inversion Principle (DIP)

## Genel Bakış
Dependency Inversion Principle (DIP), üst seviye modüllerin alt seviye modüllere bağımlı olmaması gerektiğini belirtir. Her ikisi de soyutlamalara bağımlı olmalıdır. Ayrıca, soyutlamalar detaylara bağımlı olmamalı, detaylar soyutlamalara bağımlı olmalıdır.

## Mülakat Soruları ve Cevapları

### 1. Dependency Inversion Principle nedir ve neden önemlidir?
**Cevap:**
Dependency Inversion Principle, üst seviye modüllerin alt seviye modüllere bağımlı olmaması gerektiğini belirtir. Önemlidir çünkü:
- Bağımlılıkları azaltır
- Kodun bakımını kolaylaştırır
- Test edilebilirliği artırır
- Esnekliği artırır

**Örnek Kod:**
```csharp
// DIP'ye uymayan kod
public class OrderService
{
    private readonly SqlServerDatabase _database;

    public OrderService()
    {
        _database = new SqlServerDatabase();
    }

    public void SaveOrder(Order order)
    {
        _database.Save(order);
    }
}

// DIP'ye uyan kod
public interface IDatabase
{
    void Save(Order order);
}

public class OrderService
{
    private readonly IDatabase _database;

    public OrderService(IDatabase database)
    {
        _database = database;
    }

    public void SaveOrder(Order order)
    {
        _database.Save(order);
    }
}

public class SqlServerDatabase : IDatabase
{
    public void Save(Order order)
    {
        // SQL Server'a kaydetme işlemi
    }
}

public class MongoDbDatabase : IDatabase
{
    public void Save(Order order)
    {
        // MongoDB'ye kaydetme işlemi
    }
}
```

### 2. DIP'yi ihlal eden durumları nasıl tespit edebiliriz?
**Cevap:**
DIP ihlallerini tespit etmek için:
- Somut sınıflara doğrudan bağımlılık
- new operatörü ile nesne oluşturma
- Statik metod çağrıları
- Bağımlılıkların constructor'da oluşturulması

**Örnek Kod:**
```csharp
// DIP ihlali
public class EmailService
{
    private readonly SmtpClient _smtpClient;

    public EmailService()
    {
        _smtpClient = new SmtpClient();
    }

    public void SendEmail(string to, string subject, string body)
    {
        _smtpClient.Send(to, subject, body);
    }
}

// DIP'ye uygun
public interface IEmailSender
{
    void SendEmail(string to, string subject, string body);
}

public class EmailService
{
    private readonly IEmailSender _emailSender;

    public EmailService(IEmailSender emailSender)
    {
        _emailSender = emailSender;
    }

    public void SendEmail(string to, string subject, string body)
    {
        _emailSender.SendEmail(to, subject, body);
    }
}
```

### 3. DIP'yi uygularken dikkat edilmesi gereken noktalar nelerdir?
**Cevap:**
DIP uygularken dikkat edilmesi gerekenler:
- Dependency injection kullanın
- Interface'leri doğru tasarlayın
- Bağımlılıkları dışarıdan alın
- Constructor injection tercih edin

**Örnek Kod:**
```csharp
public interface ILogger
{
    void Log(string message);
}

public interface IEmailSender
{
    void SendEmail(string to, string subject, string body);
}

public class OrderProcessor
{
    private readonly ILogger _logger;
    private readonly IEmailSender _emailSender;

    public OrderProcessor(ILogger logger, IEmailSender emailSender)
    {
        _logger = logger;
        _emailSender = emailSender;
    }

    public void ProcessOrder(Order order)
    {
        try
        {
            // Sipariş işleme mantığı
            _logger.Log("Sipariş işlendi");
            _emailSender.SendEmail(order.CustomerEmail, "Sipariş Onayı", "Siparişiniz alındı");
        }
        catch (Exception ex)
        {
            _logger.Log($"Hata: {ex.Message}");
            throw;
        }
    }
}
```

### 4. DIP ile ilgili yaygın hatalar nelerdir?
**Cevap:**
Yaygın hatalar:
- Service locator pattern kullanımı
- Static sınıflara bağımlılık
- Constructor'da nesne oluşturma
- Interface'lerin yanlış tasarlanması

**Örnek Kod:**
```csharp
// Yaygın hata: Service locator pattern
public class OrderService
{
    public void ProcessOrder(Order order)
    {
        var logger = ServiceLocator.GetService<ILogger>();
        var emailSender = ServiceLocator.GetService<IEmailSender>();
        
        logger.Log("Sipariş işleniyor");
        emailSender.SendEmail(order.CustomerEmail, "Sipariş", "İşlem başarılı");
    }
}

// Daha uygun
public class OrderService
{
    private readonly ILogger _logger;
    private readonly IEmailSender _emailSender;

    public OrderService(ILogger logger, IEmailSender emailSender)
    {
        _logger = logger;
        _emailSender = emailSender;
    }

    public void ProcessOrder(Order order)
    {
        _logger.Log("Sipariş işleniyor");
        _emailSender.SendEmail(order.CustomerEmail, "Sipariş", "İşlem başarılı");
    }
}
```

### 5. DIP'yi gerçek dünya senaryolarında nasıl uygularız?
**Cevap:**
Gerçek dünya senaryolarında:
- Dependency injection container kullanın
- Interface'leri doğru tasarlayın
- Unit of work pattern uygulayın
- Repository pattern kullanın

**Örnek Kod:**
```csharp
public interface IUnitOfWork : IDisposable
{
    IRepository<Order> Orders { get; }
    IRepository<Customer> Customers { get; }
    int SaveChanges();
}

public class OrderService
{
    private readonly IUnitOfWork _unitOfWork;
    private readonly IEmailSender _emailSender;

    public OrderService(IUnitOfWork unitOfWork, IEmailSender emailSender)
    {
        _unitOfWork = unitOfWork;
        _emailSender = emailSender;
    }

    public void ProcessOrder(Order order)
    {
        try
        {
            _unitOfWork.Orders.Add(order);
            _unitOfWork.SaveChanges();

            var customer = _unitOfWork.Customers.GetById(order.CustomerId);
            _emailSender.SendEmail(customer.Email, "Sipariş Onayı", "Siparişiniz alındı");
        }
        catch (Exception)
        {
            _unitOfWork.Dispose();
            throw;
        }
    }
}
```

## Best Practices
1. **Dependency Injection**
   - Constructor injection kullanın
   - Property injection'dan kaçının
   - Method injection'ı dikkatli kullanın
   - Dependency injection container kullanın

2. **Interface Tasarımı**
   - Interface'leri küçük ve öz tutun
   - Single Responsibility Principle'ı uygulayın
   - Interface segregation uygulayın
   - Bağımlılıkları minimize edin

3. **Test Edilebilirlik**
   - Unit testler yazın
   - Mock nesneler kullanın
   - Test coverage'ı takip edin
   - Test edilebilir kod yazın

## Kaynaklar
- [Microsoft Dependency Inversion Principle](https://docs.microsoft.com/tr-tr/dotnet/architecture/modern-web-apps-azure/architectural-principles#dependency-inversion-principle)
- [SOLID Principles in C#](https://docs.microsoft.com/tr-tr/dotnet/architecture/modern-web-apps-azure/architectural-principles#solid-principles)
- [Dependency Inversion Principle: Explanation and Examples](https://docs.microsoft.com/tr-tr/dotnet/architecture/modern-web-apps-azure/architectural-principles#dependency-inversion-principle) 