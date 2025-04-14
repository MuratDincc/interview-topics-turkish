# Exception Handling

## Genel Bakış

Exception handling, program çalışırken oluşabilecek beklenmedik durumları yönetmek için kullanılan bir mekanizmadır. C#'ta try-catch-finally blokları kullanılarak exception'lar yakalanır ve uygun şekilde işlenir.

## Temel Kavramlar

1. **Exception Türleri**
   - System.Exception (Tüm exception'ların temel sınıfı)
   - System.ArgumentException
   - System.ArgumentNullException
   - System.IndexOutOfRangeException
   - System.NullReferenceException
   - System.DivideByZeroException
   - System.IO.IOException
   - System.FormatException

2. **Try-Catch-Finally Blokları**
   ```csharp
   try
   {
       // Riskli kod
   }
   catch (SpecificException ex)
   {
       // Spesifik exception'ı yakala
   }
   catch (Exception ex)
   {
       // Genel exception'ı yakala
   }
   finally
   {
       // Her durumda çalışacak kod
   }
   ```

3. **Exception Properties**
   - Message: Hata mesajı
   - StackTrace: Hata oluştuğu yığın izi
   - InnerException: İç exception
   - Source: Hatanın kaynağı
   - HelpLink: Yardım linki

## Best Practices

1. **Spesifik Exception'ları Yakalama**
   ```csharp
   try
   {
       int.Parse("abc");
   }
   catch (FormatException ex)
   {
       Console.WriteLine("Geçersiz sayı formatı");
   }
   ```

2. **Exception Filtreleme**
   ```csharp
   try
   {
       // Kod
   }
   catch (Exception ex) when (ex.Message.Contains("specific"))
   {
       // Filtrelenmiş exception
   }
   ```

3. **Custom Exception Oluşturma**
   ```csharp
   public class CustomException : Exception
   {
       public CustomException(string message) : base(message)
       {
       }
       
       public CustomException(string message, Exception inner) 
           : base(message, inner)
       {
       }
   }
   ```

4. **Exception Loglama**
   ```csharp
   try
   {
       // Kod
   }
   catch (Exception ex)
   {
       LogError(ex);
       throw; // Exception'ı yeniden fırlat
   }
   ```

## Örnek Senaryolar

1. **Dosya İşlemleri**
   ```csharp
   try
   {
       using (StreamReader reader = new StreamReader("file.txt"))
       {
           string content = reader.ReadToEnd();
       }
   }
   catch (FileNotFoundException ex)
   {
       Console.WriteLine("Dosya bulunamadı");
   }
   catch (IOException ex)
   {
       Console.WriteLine("Dosya okuma hatası");
   }
   ```

2. **Veritabanı İşlemleri**
   ```csharp
   try
   {
       using (SqlConnection connection = new SqlConnection(connectionString))
       {
           connection.Open();
           // Veritabanı işlemleri
       }
   }
   catch (SqlException ex)
   {
       Console.WriteLine("Veritabanı hatası: " + ex.Message);
   }
   ```

3. **Web İstekleri**
   ```csharp
   try
   {
       using (HttpClient client = new HttpClient())
       {
           HttpResponseMessage response = await client.GetAsync(url);
           response.EnsureSuccessStatusCode();
       }
   }
   catch (HttpRequestException ex)
   {
       Console.WriteLine("HTTP isteği başarısız: " + ex.Message);
   }
   ```

## Exception Handling Stratejileri

1. **Fail-Fast**
   - Hataları erken yakala
   - Uygulama durumunu koru
   - Detaylı hata mesajları sağla

2. **Graceful Degradation**
   - Alternatif yollar sun
   - Kullanıcıya bilgi ver
   - Uygulamayı çalışır durumda tut

3. **Retry Pattern**
   ```csharp
   public async Task<T> RetryOperation<T>(Func<Task<T>> operation, int maxRetries = 3)
   {
       for (int i = 0; i < maxRetries; i++)
       {
           try
           {
               return await operation();
           }
           catch (Exception ex) when (i < maxRetries - 1)
           {
               await Task.Delay(1000 * (i + 1));
           }
       }
       throw new Exception("Maksimum deneme sayısına ulaşıldı");
   }
   ```

## Hata Ayıklama İpuçları

1. **Debug Modunda Exception'ları Yakalama**
   ```csharp
   #if DEBUG
   try
   {
       // Kod
   }
   catch (Exception ex)
   {
       Debug.WriteLine($"Hata: {ex.Message}");
       throw;
   }
   #endif
   ```

2. **Exception Breakpoints**
   - Visual Studio'da belirli exception'lar için breakpoint ayarla
   - Exception oluştuğunda debugger'ı durdur

3. **Exception Details**
   ```csharp
   catch (Exception ex)
   {
       Console.WriteLine($"Message: {ex.Message}");
       Console.WriteLine($"StackTrace: {ex.StackTrace}");
       Console.WriteLine($"Source: {ex.Source}");
       if (ex.InnerException != null)
       {
           Console.WriteLine($"InnerException: {ex.InnerException.Message}");
       }
   }
   ```

## Performans Konuları

1. **Exception Overhead**
   - Exception'lar pahalıdır
   - Sık kullanılan yollarda exception kullanmaktan kaçın
   - TryParse gibi alternatifleri tercih et

2. **Exception Pooling**
   ```csharp
   private static readonly ExceptionPool<CustomException> _exceptionPool = 
       new ExceptionPool<CustomException>();

   public static CustomException GetException(string message)
   {
       return _exceptionPool.GetOrCreate(() => new CustomException(message));
   }
   ```

## Güvenlik Konuları

1. **Exception Information Exposure**
   - Production'da detaylı hata mesajları gösterme
   - Hassas bilgileri loglama
   - Stack trace'i kullanıcıya gösterme

2. **Exception Handling in Web Applications**
   ```csharp
   public class GlobalExceptionHandler : IExceptionHandler
   {
       public async Task HandleExceptionAsync(ExceptionContext context)
       {
           var response = new ErrorResponse
           {
               Message = "Bir hata oluştu",
               ErrorId = Guid.NewGuid().ToString()
           };

           context.Result = new ObjectResult(response)
           {
               StatusCode = 500
           };

           // Log the actual exception
           LogError(context.Exception, response.ErrorId);
       }
   }
   ```

## Mülakat Soruları

1. **Exception Temelleri**
   - Exception nedir ve ne zaman kullanılır?
   - Checked ve unchecked exception'lar arasındaki fark nedir?
   - Exception handling'in avantajları ve dezavantajları nelerdir?

2. **Exception Hiyerarşisi**
   - System.Exception sınıfının özellikleri nelerdir?
   - Custom exception nasıl oluşturulur?
   - Exception inheritance hiyerarşisi nasıl tasarlanmalıdır?

3. **Try-Catch-Finally**
   - Try-catch-finally bloklarının çalışma sırası nasıldır?
   - Multiple catch blokları nasıl sıralanmalıdır?
   - Finally bloğu ne zaman kullanılmalıdır?

4. **Exception Filtreleme**
   - Exception filtreleme nedir ve nasıl kullanılır?
   - When anahtar kelimesi ne işe yarar?
   - Exception filtrelemenin performans etkisi nedir?

5. **Exception Best Practices**
   - Exception'lar ne zaman yakalanmalıdır?
   - Exception'lar ne zaman yeniden fırlatılmalıdır?
   - Exception mesajları nasıl yazılmalıdır?

6. **Exception Logging**
   - Exception'lar nasıl loglanmalıdır?
   - Loglama stratejileri nelerdir?
   - Sensitive bilgiler exception'larda nasıl korunur?

7. **Exception ve Performans**
   - Exception handling'in performans maliyeti nedir?
   - Exception'lar ne zaman kullanılmamalıdır?
   - Exception pooling nedir ve nasıl kullanılır?

8. **Exception ve Güvenlik**
   - Exception'lar güvenlik açıklarına nasıl yol açabilir?
   - Production ortamında exception detayları nasıl yönetilmelidir?
   - Exception'lar SQL injection'a nasıl yol açabilir?

9. **Global Exception Handling**
   - Global exception handler nasıl implemente edilir?
   - Web uygulamalarında exception handling nasıl yapılır?
   - API'lerde exception'lar nasıl yönetilmelidir?

10. **Exception ve Asenkron Programlama**
    - Async/await ile exception handling nasıl yapılır?
    - Task exception'ları nasıl yönetilir?
    - AggregateException nedir ve nasıl kullanılır?

## Örnek Kod Soruları

1. **Custom Exception Oluşturma**
   ```csharp
   public class ValidationException : Exception
   {
       // Implementasyon
   }
   ```

2. **Exception Filtreleme**
   ```csharp
   try
   {
       // Kod
   }
   catch (Exception ex) when (ex.Message.Contains("specific"))
   {
       // Implementasyon
   }
   ```

3. **Global Exception Handler**
   ```csharp
   public class GlobalExceptionHandler : IExceptionHandler
   {
       // Implementasyon
   }
   ```

4. **Retry Pattern**
   ```csharp
   public async Task<T> RetryOperation<T>(Func<Task<T>> operation, int maxRetries)
   {
       // Implementasyon
   }
   ```

5. **Exception Logging**
   ```csharp
   public void LogException(Exception ex, string context)
   {
       // Implementasyon
   }
   ```

## Kaynaklar

- [Microsoft Docs - Exception Handling](https://docs.microsoft.com/en-us/dotnet/csharp/fundamentals/exceptions/)
- [Exception Handling Best Practices](https://docs.microsoft.com/en-us/dotnet/standard/exceptions/best-practices-for-exceptions)
- [C# Exception Handling Patterns](https://docs.microsoft.com/en-us/dotnet/standard/exceptions/exception-handling-patterns) 