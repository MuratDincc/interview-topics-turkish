# Asenkron Programlama

## Genel Bakış
Asenkron programlama, uygulamanın uzun süren işlemleri (I/O, ağ istekleri, veritabanı sorguları vb.) sırasında thread'leri bloke etmeden çalışmasını sağlayan bir programlama modelidir. Bu sayede uygulama daha responsive ve ölçeklenebilir hale gelir.

## Temel Kavramlar

### 1. Async/Await Pattern
```csharp
public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly ILogger<OrderService> _logger;

    public OrderService(
        IOrderRepository repository,
        ILogger<OrderService> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    public async Task<Order> GetOrderAsync(int id)
    {
        try
        {
            _logger.LogInformation("Getting order {Id}", id);
            
            // Asenkron veritabanı sorgusu
            var order = await _repository.GetByIdAsync(id);
            
            if (order == null)
            {
                _logger.LogWarning("Order {Id} not found", id);
                return null;
            }

            // Asenkron hesaplama
            await CalculateOrderTotalAsync(order);
            
            _logger.LogInformation("Order {Id} retrieved successfully", id);
            return order;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting order {Id}", id);
            throw;
        }
    }

    private async Task CalculateOrderTotalAsync(Order order)
    {
        // Uzun süren hesaplama işlemi
        await Task.Run(() =>
        {
            order.Total = order.Items.Sum(item => item.Price * item.Quantity);
        });
    }
}
```

### 2. Task Parallel Library (TPL)
```csharp
public class DataProcessor
{
    private readonly ILogger<DataProcessor> _logger;

    public DataProcessor(ILogger<DataProcessor> logger)
    {
        _logger = logger;
    }

    public async Task ProcessDataAsync(List<DataItem> items)
    {
        // Paralel işleme
        var tasks = items.Select(async item =>
        {
            try
            {
                await ProcessItemAsync(item);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing item {Id}", item.Id);
            }
        });

        await Task.WhenAll(tasks);
    }

    private async Task ProcessItemAsync(DataItem item)
    {
        // CPU-bound işlem
        await Task.Run(() =>
        {
            // Yoğun hesaplama işlemi
            item.Process();
        });

        // I/O-bound işlem
        await SaveItemAsync(item);
    }

    private async Task SaveItemAsync(DataItem item)
    {
        // Dosya I/O işlemi
        await File.WriteAllTextAsync(
            $"item_{item.Id}.json",
            JsonSerializer.Serialize(item));
    }
}
```

### 3. Cancellation Token
```csharp
public class LongRunningOperation
{
    private readonly ILogger<LongRunningOperation> _logger;

    public LongRunningOperation(ILogger<LongRunningOperation> logger)
    {
        _logger = logger;
    }

    public async Task ProcessAsync(CancellationToken cancellationToken)
    {
        try
        {
            // İşlem başladı
            _logger.LogInformation("Operation started");

            // Her adımda iptal kontrolü
            for (int i = 0; i < 10; i++)
            {
                cancellationToken.ThrowIfCancellationRequested();
                
                await Task.Delay(1000, cancellationToken);
                _logger.LogInformation("Processing step {Step}", i);
            }

            _logger.LogInformation("Operation completed");
        }
        catch (OperationCanceledException)
        {
            _logger.LogInformation("Operation cancelled");
            throw;
        }
    }
}
```

## Best Practices

### 1. Async/Await Kullanımı
- Async void'den kaçının
- ConfigureAwait(false) kullanın
- Task.WhenAll/WhenAny kullanın
- Exception handling yapın
- Deadlock'lardan kaçının

### 2. Performans Optimizasyonu
- CPU-bound işlemleri Task.Run ile yapın
- I/O-bound işlemleri async/await ile yapın
- Task pooling kullanın
- Memory allocation'ı minimize edin
- Thread pool'u yönetin

### 3. Hata Yönetimi
- Try-catch bloklarını doğru kullanın
- AggregateException'ları handle edin
- Cancellation token'ları kullanın
- Timeout mekanizmaları ekleyin
- Retry pattern uygulayın

## Sık Sorulan Sorular

### 1. Async/Await ne zaman kullanılmalıdır?
- I/O işlemlerinde
- Ağ isteklerinde
- Veritabanı sorgularında
- Dosya işlemlerinde
- Uzun süren hesaplamalarda

### 2. Task.Run ne zaman kullanılmalıdır?
- CPU-bound işlemlerde
- UI thread'i bloke etmemek için
- Paralel işleme gerektiğinde
- Background task'lar için
- Legacy kod entegrasyonunda

### 3. Asenkron programlamada hangi hatalar yapılır?
- Async void kullanımı
- Deadlock oluşturma
- Gereksiz Task.Run kullanımı
- Exception handling eksikliği
- Cancellation token kullanmama

## Kaynaklar
- [Async/Await Best Practices](https://docs.microsoft.com/tr-tr/dotnet/csharp/async)
- [Task Parallel Library](https://docs.microsoft.com/tr-tr/dotnet/standard/parallel-programming/task-parallel-library-tpl)
- [Cancellation in Managed Threads](https://docs.microsoft.com/tr-tr/dotnet/standard/threading/cancellation-in-managed-threads)
- [Async Programming Patterns](https://docs.microsoft.com/tr-tr/dotnet/standard/asynchronous-programming-patterns/) 