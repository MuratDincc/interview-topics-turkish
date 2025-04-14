# Async/Await

## Genel Bakış
Async/Await, C#'ta asenkron programlama için kullanılan temel yapıdır. Bu yapı, uzun süren işlemleri bloklamadan yönetmeyi ve uygulama performansını artırmayı sağlar.

## Mülakat Soruları ve Cevapları

### 1. Async/Await nedir ve nasıl çalışır?
**Cevap:**
Async/Await, asenkron operasyonları yönetmek için kullanılan bir C# özelliğidir. Çalışma prensibi:
1. `async` keyword'ü ile metod işaretlenir
2. `await` ile asenkron operasyon beklenir
3. State machine oluşturulur
4. Callback mekanizması kullanılır

**Örnek Kod:**
```csharp
// Basit async metod
public async Task<string> GetDataAsync()
{
    // Simüle edilmiş uzun süren işlem
    await Task.Delay(1000);
    return "Veri alındı";
}

// Async metod kullanımı
public async Task ProcessDataAsync()
{
    try
    {
        string data = await GetDataAsync();
        Console.WriteLine(data);
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Hata: {ex.Message}");
    }
}
```

### 2. Task ve ValueTask arasındaki farklar nelerdir?
**Cevap:**
Task ve ValueTask farkları:
- Memory allocation
- Performance
- Kullanım senaryoları
- Boxing/Unboxing

**Örnek Kod:**
```csharp
// Task kullanımı
public async Task<int> GetNumberAsync()
{
    await Task.Delay(100);
    return 42;
}

// ValueTask kullanımı
public async ValueTask<int> GetNumberValueTaskAsync()
{
    if (_cache.TryGetValue("number", out int value))
    {
        return value;
    }
    
    value = await GetNumberFromServiceAsync();
    _cache["number"] = value;
    return value;
}
```

### 3. Async metodlarda hata yönetimi nasıl yapılır?
**Cevap:**
Async metodlarda hata yönetimi için:
- try-catch blokları
- Task.Exception
- AggregateException
- CancellationToken

**Örnek Kod:**
```csharp
// Hata yönetimi örneği
public async Task ProcessWithErrorHandlingAsync()
{
    try
    {
        await ProcessDataAsync();
    }
    catch (HttpRequestException ex)
    {
        Console.WriteLine($"HTTP Hatası: {ex.Message}");
    }
    catch (TimeoutException ex)
    {
        Console.WriteLine($"Zaman Aşımı: {ex.Message}");
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Genel Hata: {ex.Message}");
    }
}

// CancellationToken kullanımı
public async Task ProcessWithCancellationAsync(CancellationToken cancellationToken)
{
    try
    {
        await Task.Delay(1000, cancellationToken);
        // İşlem devam eder
    }
    catch (OperationCanceledException)
    {
        Console.WriteLine("İşlem iptal edildi");
    }
}
```

### 4. Async void ne zaman kullanılmalıdır?
**Cevap:**
Async void kullanımı:
- Event handler'larda
- Dikkat edilmesi gereken noktalar
- Exception handling
- Best practices

**Örnek Kod:**
```csharp
// Event handler örneği
private async void Button_Click(object sender, EventArgs e)
{
    try
    {
        await ProcessDataAsync();
    }
    catch (Exception ex)
    {
        // Hata loglama
        Logger.LogError(ex);
    }
}

// Async void kullanımından kaçınma
public async Task ProcessDataAsync()
{
    // Task döndüren async metod
    await Task.Delay(1000);
}
```

### 5. Async metodlarda deadlock nasıl önlenir?
**Cevap:**
Deadlock önleme yöntemleri:
- ConfigureAwait(false)
- Task.Run kullanımı
- SynchronizationContext
- Best practices

**Örnek Kod:**
```csharp
// Deadlock önleme
public async Task<string> GetDataWithoutDeadlockAsync()
{
    // ConfigureAwait(false) kullanımı
    var result = await GetDataFromServiceAsync().ConfigureAwait(false);
    return result;
}

// Task.Run kullanımı
public async Task ProcessInBackgroundAsync()
{
    await Task.Run(async () =>
    {
        // Uzun süren işlem
        await Task.Delay(1000);
    });
}

// SynchronizationContext örneği
public async Task ProcessWithContextAsync()
{
    var context = SynchronizationContext.Current;
    await Task.Run(() =>
    {
        // Arka plan işlemi
    });
    context?.Post(_ =>
    {
        // UI güncelleme
    }, null);
}
```

## Best Practices
1. **Performans**
   - ConfigureAwait(false) kullanın
   - Gereksiz async/await kullanımından kaçının
   - ValueTask kullanımını değerlendirin
   - Task.Run'ı dikkatli kullanın

2. **Güvenlik**
   - Exception handling yapın
   - CancellationToken kullanın
   - Timeout mekanizması ekleyin
   - Resource cleanup yapın

3. **Kod Kalitesi**
   - Async suffix kullanın
   - Task döndüren metodları tercih edin
   - Async void kullanımından kaçının
   - Documentation ekleyin

## Kaynaklar
- [Microsoft Async Programming](https://docs.microsoft.com/tr-tr/dotnet/csharp/programming-guide/concepts/async/)
- [Async/Await Best Practices](https://docs.microsoft.com/tr-tr/archive/msdn-magazine/2013/march/async-await-best-practices-in-asynchronous-programming)
- [Task vs ValueTask](https://devblogs.microsoft.com/dotnet/understanding-the-whys-whats-and-whens-of-valuetask/)
- [Async/Await Pitfalls](https://blog.stephencleary.com/2012/02/async-and-await.html) 