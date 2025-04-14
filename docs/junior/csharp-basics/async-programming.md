# Asenkron Programlama

## Genel Bakış

Asenkron programlama, uygulamanın uzun süren işlemleri bloklamadan çalışmasını sağlayan bir programlama tekniğidir. C#'ta async/await anahtar kelimeleri ve Task sınıfı kullanılarak asenkron işlemler yapılır.

## Async/Await Temelleri

1. **Async Metod Tanımlama**
   ```csharp
   public async Task<string> GetDataAsync()
   {
       // Asenkron işlem
       await Task.Delay(1000);
       return "Veri";
   }
   ```

2. **Async Void Kullanımı**
   ```csharp
   public async void HandleButtonClick()
   {
       try
       {
           await ProcessDataAsync();
       }
       catch (Exception ex)
       {
           // Hata yönetimi
       }
   }
   ```

3. **Task Dönüş Tipleri**
   ```csharp
   // Task<T> - Değer döndüren
   public async Task<int> GetNumberAsync()
   {
       return 42;
   }
   
   // Task - Değer döndürmeyen
   public async Task ProcessAsync()
   {
       await Task.Delay(1000);
   }
   ```

## Task İşlemleri

1. **Task Oluşturma**
   ```csharp
   // Task oluşturma
   Task task = Task.Run(() => {
       Console.WriteLine("Task çalışıyor");
   });
   
   // Task<T> oluşturma
   Task<int> taskWithResult = Task.Run(() => {
       return 42;
   });
   ```

2. **Task Bekleme**
   ```csharp
   // Task bekleme
   await task;
   
   // Task sonucu alma
   int result = await taskWithResult;
   
   // Timeout ile bekleme
   if (await Task.WhenAny(task, Task.Delay(5000)) == task)
   {
       // Task tamamlandı
   }
   ```

3. **Task İptali**
   ```csharp
   CancellationTokenSource cts = new CancellationTokenSource();
   
   Task task = Task.Run(() => {
       while (!cts.Token.IsCancellationRequested)
       {
           // İşlem
       }
   }, cts.Token);
   
   // İptal
   cts.Cancel();
   ```

## Paralel İşlemler

1. **Task.WhenAll**
   ```csharp
   Task[] tasks = new Task[3];
   tasks[0] = Task.Delay(1000);
   tasks[1] = Task.Delay(2000);
   tasks[2] = Task.Delay(3000);
   
   await Task.WhenAll(tasks);
   ```

2. **Task.WhenAny**
   ```csharp
   Task[] tasks = new Task[3];
   tasks[0] = Task.Delay(1000);
   tasks[1] = Task.Delay(2000);
   tasks[2] = Task.Delay(3000);
   
   Task completedTask = await Task.WhenAny(tasks);
   ```

3. **Parallel.ForEach**
   ```csharp
   var items = new List<int> { 1, 2, 3, 4, 5 };
   
   await Parallel.ForEachAsync(items, async (item, token) => {
       await ProcessItemAsync(item);
   });
   ```

## Asenkron Stream'ler

1. **IAsyncEnumerable Kullanımı**
   ```csharp
   public async IAsyncEnumerable<int> GetNumbersAsync()
   {
       for (int i = 0; i < 10; i++)
       {
           await Task.Delay(100);
           yield return i;
       }
   }
   
   // Kullanımı
   await foreach (var number in GetNumbersAsync())
   {
       Console.WriteLine(number);
   }
   ```

## Asenkron Exception Handling

1. **Try-Catch Kullanımı**
   ```csharp
   try
   {
       await ProcessAsync();
   }
   catch (Exception ex)
   {
       // Hata yönetimi
   }
   ```

2. **AggregateException**
   ```csharp
   try
   {
       await Task.WhenAll(tasks);
   }
   catch (AggregateException ex)
   {
       foreach (var innerEx in ex.InnerExceptions)
       {
           // Hata yönetimi
       }
   }
   ```

## Asenkron Best Practices

1. **ConfigureAwait Kullanımı**
   ```csharp
   public async Task ProcessAsync()
   {
       await Task.Delay(1000).ConfigureAwait(false);
   }
   ```

2. **ValueTask Kullanımı**
   ```csharp
   public async ValueTask<int> GetNumberAsync()
   {
       if (_cache.TryGetValue("number", out int value))
           return value;
           
       return await FetchNumberAsync();
   }
   ```

## Mülakat Soruları

1. **Async/Await Temelleri**
   - Async/await nedir ve nasıl çalışır?
   - Task ve ValueTask arasındaki farklar nelerdir?
   - Async void ne zaman kullanılmalıdır?

2. **Task İşlemleri**
   - Task.Run() ne zaman kullanılmalıdır?
   - Task.WhenAll() ve Task.WhenAny() arasındaki farklar nelerdir?
   - Task iptali nasıl yapılır?

3. **Paralel İşlemler**
   - Paralel işlemler ne zaman kullanılmalıdır?
   - Task.WhenAll() ve Parallel.ForEach() arasındaki farklar nelerdir?
   - Paralel işlemlerde exception handling nasıl yapılır?

4. **Asenkron Stream'ler**
   - IAsyncEnumerable nedir ve ne işe yarar?
   - Asenkron stream'ler ne zaman kullanılmalıdır?
   - Asenkron stream'lerde performans optimizasyonu nasıl yapılır?

5. **Exception Handling**
   - Asenkron metodlarda exception handling nasıl yapılır?
   - AggregateException nedir ve nasıl yönetilir?
   - Asenkron işlemlerde unhandled exception'lar nasıl yakalanır?

6. **Performans**
   - Asenkron işlemlerde performans optimizasyonu nasıl yapılır?
   - ConfigureAwait(false) ne işe yarar?
   - Asenkron işlemlerde memory kullanımı nasıl optimize edilir?

7. **Deadlock**
   - Asenkron işlemlerde deadlock nasıl oluşur?
   - Deadlock nasıl önlenir?
   - ConfigureAwait(false) deadlock'u nasıl önler?

8. **Resource Yönetimi**
   - Asenkron işlemlerde resource leak nasıl önlenir?
   - IDisposable ve asenkron işlemler nasıl kullanılır?
   - Asenkron işlemlerde connection pooling nasıl yapılır?

9. **Testing**
   - Asenkron kodlar nasıl test edilir?
   - Asenkron testlerde best practices nelerdir?
   - Asenkron testlerde mock'lar nasıl kullanılır?

10. **Best Practices**
    - Asenkron programlamada best practices nelerdir?
    - Asenkron metod isimlendirmesi nasıl yapılmalıdır?
    - Asenkron işlemlerde logging nasıl yapılmalıdır?

## Örnek Kod Soruları

1. **Asenkron Dosya Okuma**
   ```csharp
   public async Task<string> ReadFileAsync(string path)
   {
       // Implementasyon
   }
   ```

2. **Asenkron HTTP İsteği**
   ```csharp
   public async Task<string> GetDataFromApiAsync(string url)
   {
       // Implementasyon
   }
   ```

3. **Asenkron Veritabanı İşlemi**
   ```csharp
   public async Task<List<Customer>> GetCustomersAsync()
   {
       // Implementasyon
   }
   ```

4. **Asenkron Stream İşlemi**
   ```csharp
   public async IAsyncEnumerable<string> ReadLinesAsync(string path)
   {
       // Implementasyon
   }
   ```

5. **Asenkron Cache İşlemi**
   ```csharp
   public async ValueTask<T> GetOrAddAsync<T>(string key, Func<Task<T>> factory)
   {
       // Implementasyon
   }
   ```

## Kaynaklar

- [Microsoft Docs - Async Programming](https://docs.microsoft.com/en-us/dotnet/csharp/async)
- [Task Class](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.task)
- [Async/Await Best Practices](https://docs.microsoft.com/en-us/archive/msdn-magazine/2013/march/async-await-best-practices-in-asynchronous-programming) 