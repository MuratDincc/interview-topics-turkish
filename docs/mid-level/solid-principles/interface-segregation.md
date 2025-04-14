# Interface Segregation Principle (ISP)

## Genel Bakış
Interface Segregation Principle (ISP), bir sınıfın kullanmadığı interface'leri implemente etmeye zorlanmaması gerektiğini belirtir. Yani, interface'ler müşterilerinin ihtiyaçlarına göre ayrılmalı ve her müşteri sadece kendisi için gerekli olan metodlara sahip olmalıdır.

## Mülakat Soruları ve Cevapları

### 1. Interface Segregation Principle nedir ve neden önemlidir?
**Cevap:**
Interface Segregation Principle, interface'lerin müşterilerinin ihtiyaçlarına göre ayrılması gerektiğini belirtir. Önemlidir çünkü:
- Kodun bakımını kolaylaştırır
- Bağımlılıkları azaltır
- Test edilebilirliği artırır
- Kodun anlaşılabilirliğini artırır

**Örnek Kod:**
```csharp
// ISP'ye uymayan kod
public interface IWorker
{
    void Work();
    void Eat();
    void Sleep();
}

public class HumanWorker : IWorker
{
    public void Work() { /* Çalışma işlemi */ }
    public void Eat() { /* Yemek yeme işlemi */ }
    public void Sleep() { /* Uyuma işlemi */ }
}

public class RobotWorker : IWorker
{
    public void Work() { /* Çalışma işlemi */ }
    public void Eat() { throw new NotSupportedException(); }
    public void Sleep() { throw new NotSupportedException(); }
}

// ISP'ye uyan kod
public interface IWorkable
{
    void Work();
}

public interface IEatable
{
    void Eat();
}

public interface ISleepable
{
    void Sleep();
}

public class HumanWorker : IWorkable, IEatable, ISleepable
{
    public void Work() { /* Çalışma işlemi */ }
    public void Eat() { /* Yemek yeme işlemi */ }
    public void Sleep() { /* Uyuma işlemi */ }
}

public class RobotWorker : IWorkable
{
    public void Work() { /* Çalışma işlemi */ }
}
```

### 2. ISP'yi ihlal eden durumları nasıl tespit edebiliriz?
**Cevap:**
ISP ihlallerini tespit etmek için:
- Interface'lerin büyük ve karmaşık olması
- Sınıfların kullanmadığı metodları implemente etmesi
- NotSupportedException fırlatılması
- Interface'lerin çok fazla sorumluluğu olması

**Örnek Kod:**
```csharp
// ISP ihlali
public interface IOrderProcessor
{
    void ProcessOrder(Order order);
    void CancelOrder(Order order);
    void RefundOrder(Order order);
    void GenerateInvoice(Order order);
    void SendEmail(Order order);
}

// ISP'ye uygun
public interface IOrderProcessor
{
    void ProcessOrder(Order order);
}

public interface IOrderCancellation
{
    void CancelOrder(Order order);
}

public interface IOrderRefund
{
    void RefundOrder(Order order);
}

public interface IInvoiceGenerator
{
    void GenerateInvoice(Order order);
}

public interface IEmailSender
{
    void SendEmail(Order order);
}
```

### 3. ISP'yi uygularken dikkat edilmesi gereken noktalar nelerdir?
**Cevap:**
ISP uygularken dikkat edilmesi gerekenler:
- Interface'leri küçük ve öz tutun
- İlgili metodları gruplayın
- Bağımlılıkları minimize edin
- Single Responsibility Principle'ı dikkate alın

**Örnek Kod:**
```csharp
public interface IRepository<T>
{
    T GetById(int id);
    IEnumerable<T> GetAll();
    void Add(T entity);
    void Update(T entity);
    void Delete(T entity);
}

public interface IReadOnlyRepository<T>
{
    T GetById(int id);
    IEnumerable<T> GetAll();
}

public interface IWriteOnlyRepository<T>
{
    void Add(T entity);
    void Update(T entity);
    void Delete(T entity);
}
```

### 4. ISP ile ilgili yaygın hatalar nelerdir?
**Cevap:**
Yaygın hatalar:
- Interface'lerin çok büyük olması
- Gereksiz metodların eklenmesi
- İlgili metodların ayrılması
- Aşırı parçalama

**Örnek Kod:**
```csharp
// Yaygın hata: Aşırı parçalama
public interface IId
{
    int Id { get; set; }
}

public interface IName
{
    string Name { get; set; }
}

// Daha uygun
public interface IEntity
{
    int Id { get; set; }
    string Name { get; set; }
}
```

### 5. ISP'yi gerçek dünya senaryolarında nasıl uygularız?
**Cevap:**
Gerçek dünya senaryolarında:
- Domain-driven design yaklaşımını kullanın
- CQRS pattern uygulayın
- Repository pattern kullanın
- Service layer pattern uygulayın

**Örnek Kod:**
```csharp
public interface IOrderQueryService
{
    Order GetOrderById(int id);
    IEnumerable<Order> GetOrdersByCustomer(int customerId);
    IEnumerable<Order> GetOrdersByDateRange(DateTime startDate, DateTime endDate);
}

public interface IOrderCommandService
{
    void CreateOrder(Order order);
    void UpdateOrder(Order order);
    void CancelOrder(int orderId);
}

public interface IOrderReportService
{
    OrderReport GenerateDailyReport(DateTime date);
    OrderReport GenerateMonthlyReport(int year, int month);
}

public class OrderService : IOrderQueryService, IOrderCommandService, IOrderReportService
{
    private readonly IOrderRepository _orderRepository;
    private readonly IReportGenerator _reportGenerator;

    public OrderService(IOrderRepository orderRepository, IReportGenerator reportGenerator)
    {
        _orderRepository = orderRepository;
        _reportGenerator = reportGenerator;
    }

    // IOrderQueryService implementasyonu
    public Order GetOrderById(int id)
    {
        return _orderRepository.GetById(id);
    }

    // IOrderCommandService implementasyonu
    public void CreateOrder(Order order)
    {
        _orderRepository.Add(order);
    }

    // IOrderReportService implementasyonu
    public OrderReport GenerateDailyReport(DateTime date)
    {
        return _reportGenerator.GenerateDailyReport(date);
    }
}
```

## Best Practices
1. **Interface Tasarımı**
   - Interface'leri küçük ve öz tutun
   - İlgili metodları gruplayın
   - Bağımlılıkları minimize edin
   - Single Responsibility Principle'ı dikkate alın

2. **Kod Organizasyonu**
   - Domain-driven design yaklaşımını kullanın
   - CQRS pattern uygulayın
   - Repository pattern kullanın
   - Service layer pattern uygulayın

3. **Test Edilebilirlik**
   - Unit testler yazın
   - Mock nesneler kullanın
   - Test coverage'ı takip edin
   - Test edilebilir kod yazın

## Kaynaklar
- [Microsoft Interface Segregation Principle](https://docs.microsoft.com/tr-tr/dotnet/architecture/modern-web-apps-azure/architectural-principles#interface-segregation-principle)
- [SOLID Principles in C#](https://docs.microsoft.com/tr-tr/dotnet/architecture/modern-web-apps-azure/architectural-principles#solid-principles)
- [Interface Segregation Principle: Explanation and Examples](https://docs.microsoft.com/tr-tr/dotnet/architecture/modern-web-apps-azure/architectural-principles#interface-segregation-principle) 