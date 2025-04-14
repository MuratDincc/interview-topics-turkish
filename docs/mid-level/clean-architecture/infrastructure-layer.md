# Infrastructure Layer

## Genel Bakış
Infrastructure Layer, Clean Architecture'ın dış katmanlarından biridir ve uygulamanın teknik detaylarını içerir. Bu katman, veritabanı işlemleri, harici servis entegrasyonları ve framework kullanımı gibi teknik konuları yönetir.

## Temel Bileşenler

### 1. Persistence
Veritabanı işlemleri ve repository implementasyonları.

**Örnek Kod:**
```csharp
public class OrderRepository : IOrderRepository
{
    private readonly ApplicationDbContext _context;

    public OrderRepository(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<Order> GetById(Guid id)
    {
        return await _context.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id);
    }

    public async Task Add(Order order)
    {
        await _context.Orders.AddAsync(order);
    }

    public async Task Update(Order order)
    {
        _context.Orders.Update(order);
    }

    public async Task Delete(Order order)
    {
        _context.Orders.Remove(order);
    }
}

public class UnitOfWork : IUnitOfWork
{
    private readonly ApplicationDbContext _context;
    private IDbContextTransaction _transaction;

    public UnitOfWork(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<int> SaveChanges()
    {
        return await _context.SaveChangesAsync();
    }

    public async Task BeginTransaction()
    {
        _transaction = await _context.Database.BeginTransactionAsync();
    }

    public async Task CommitTransaction()
    {
        await _transaction.CommitAsync();
    }

    public async Task RollbackTransaction()
    {
        await _transaction.RollbackAsync();
    }

    public void Dispose()
    {
        _transaction?.Dispose();
        _context.Dispose();
    }
}
```

### 2. External Services
Harici servis entegrasyonları.

**Örnek Kod:**
```csharp
public interface IEmailService
{
    Task SendEmailAsync(string to, string subject, string body);
}

public class SmtpEmailService : IEmailService
{
    private readonly SmtpClient _smtpClient;
    private readonly ILogger<SmtpEmailService> _logger;

    public SmtpEmailService(SmtpClient smtpClient, ILogger<SmtpEmailService> logger)
    {
        _smtpClient = smtpClient;
        _logger = logger;
    }

    public async Task SendEmailAsync(string to, string subject, string body)
    {
        try
        {
            var message = new MailMessage
            {
                To = { to },
                Subject = subject,
                Body = body
            };

            await _smtpClient.SendMailAsync(message);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Email gönderimi başarısız oldu");
            throw;
        }
    }
}
```

### 3. Logging
Loglama işlemleri.

**Örnek Kod:**
```csharp
public interface ILoggerService
{
    void LogInformation(string message);
    void LogWarning(string message);
    void LogError(string message, Exception exception = null);
}

public class SerilogLoggerService : ILoggerService
{
    private readonly ILogger _logger;

    public SerilogLoggerService(ILogger logger)
    {
        _logger = logger;
    }

    public void LogInformation(string message)
    {
        _logger.LogInformation(message);
    }

    public void LogWarning(string message)
    {
        _logger.LogWarning(message);
    }

    public void LogError(string message, Exception exception = null)
    {
        _logger.LogError(exception, message);
    }
}
```

### 4. Caching
Önbellekleme işlemleri.

**Örnek Kod:**
```csharp
public interface ICacheService
{
    Task<T> GetAsync<T>(string key);
    Task SetAsync<T>(string key, T value, TimeSpan? expiration = null);
    Task RemoveAsync(string key);
}

public class RedisCacheService : ICacheService
{
    private readonly IConnectionMultiplexer _redis;
    private readonly IDatabase _database;

    public RedisCacheService(IConnectionMultiplexer redis)
    {
        _redis = redis;
        _database = redis.GetDatabase();
    }

    public async Task<T> GetAsync<T>(string key)
    {
        var value = await _database.StringGetAsync(key);
        return value.HasValue ? JsonSerializer.Deserialize<T>(value) : default;
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan? expiration = null)
    {
        var serializedValue = JsonSerializer.Serialize(value);
        await _database.StringSetAsync(key, serializedValue, expiration);
    }

    public async Task RemoveAsync(string key)
    {
        await _database.KeyDeleteAsync(key);
    }
}
```

### 5. File Storage
Dosya depolama işlemleri.

**Örnek Kod:**
```csharp
public interface IFileStorageService
{
    Task<string> UploadFileAsync(Stream fileStream, string fileName);
    Task<Stream> DownloadFileAsync(string filePath);
    Task DeleteFileAsync(string filePath);
}

public class AzureBlobStorageService : IFileStorageService
{
    private readonly BlobServiceClient _blobServiceClient;
    private readonly string _containerName;

    public AzureBlobStorageService(BlobServiceClient blobServiceClient, string containerName)
    {
        _blobServiceClient = blobServiceClient;
        _containerName = containerName;
    }

    public async Task<string> UploadFileAsync(Stream fileStream, string fileName)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(_containerName);
        var blobClient = containerClient.GetBlobClient(fileName);

        await blobClient.UploadAsync(fileStream, true);
        return blobClient.Uri.ToString();
    }

    public async Task<Stream> DownloadFileAsync(string filePath)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(_containerName);
        var blobClient = containerClient.GetBlobClient(filePath);

        var response = await blobClient.DownloadAsync();
        return response.Value.Content;
    }

    public async Task DeleteFileAsync(string filePath)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(_containerName);
        var blobClient = containerClient.GetBlobClient(filePath);

        await blobClient.DeleteAsync();
    }
}
```

## Best Practices

### 1. Repository Tasarımı
- Repository'ler interface'ler üzerinden tanımlanmalıdır
- Repository'ler unit of work pattern ile kullanılmalıdır
- Repository'ler generic olarak tasarlanmalıdır
- Repository'ler test edilebilir olmalıdır

### 2. External Service Tasarımı
- Harici servisler interface'ler üzerinden tanımlanmalıdır
- Harici servisler retry pattern ile kullanılmalıdır
- Harici servisler circuit breaker pattern ile korunmalıdır
- Harici servisler test edilebilir olmalıdır

### 3. Logging Stratejisi
- Logging seviyeleri doğru kullanılmalıdır
- Log mesajları anlamlı olmalıdır
- Log formatı standart olmalıdır
- Log rotasyonu yapılandırılmalıdır

### 4. Caching Stratejisi
- Cache key'leri anlamlı olmalıdır
- Cache süreleri optimize edilmelidir
- Cache invalidation stratejisi belirlenmelidir
- Distributed cache kullanılmalıdır

## Sık Sorulan Sorular

### 1. Infrastructure Layer neden önemlidir?
- Teknik detayları izole eder
- Test edilebilirliği artırır
- Bağımlılıkları yönetir
- Framework bağımsızlığı sağlar

### 2. Repository pattern ne zaman kullanılmalıdır?
- Veritabanı işlemleri için
- Data access logic'i izole etmek için
- Unit of work pattern ile birlikte
- Test edilebilirliği artırmak için

### 3. External service entegrasyonları nasıl yönetilmelidir?
- Interface'ler üzerinden
- Retry pattern ile
- Circuit breaker pattern ile
- Timeout yönetimi ile

## Kaynaklar
- [Microsoft Infrastructure Layer](https://docs.microsoft.com/tr-tr/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/ddd-oriented-microservice#the-infrastructure-layer)
- [Clean Architecture - Infrastructure Layer](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Infrastructure Layer Best Practices](https://docs.microsoft.com/tr-tr/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/ddd-oriented-microservice#the-infrastructure-layer) 