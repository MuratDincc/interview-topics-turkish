# Extension Methods

## Genel Bakış

Extension Methods (Genişletme Metodları), mevcut tiplere yeni metodlar eklememizi sağlayan bir C# özelliğidir. Bu metodlar, orijinal tipin kaynak koduna erişmeden veya türetme yapmadan, tipin işlevselliğini genişletmemize olanak tanır.

## Extension Method Tanımlama

1. **Temel Extension Method**
   ```csharp
   public static class StringExtensions
   {
       public static bool IsNullOrEmpty(this string str)
       {
           return string.IsNullOrEmpty(str);
       }
   }
   
   // Kullanımı
   string text = "Merhaba";
   bool isEmpty = text.IsNullOrEmpty();
   ```

2. **Generic Extension Method**
   ```csharp
   public static class CollectionExtensions
   {
       public static bool IsNullOrEmpty<T>(this IEnumerable<T> collection)
       {
           return collection == null || !collection.Any();
       }
   }
   
   // Kullanımı
   List<int> numbers = new List<int>();
   bool isEmpty = numbers.IsNullOrEmpty();
   ```

3. **Parametreli Extension Method**
   ```csharp
   public static class StringExtensions
   {
       public static string Repeat(this string str, int count)
       {
           return string.Concat(Enumerable.Repeat(str, count));
       }
   }
   
   // Kullanımı
   string result = "Merhaba".Repeat(3); // "MerhabaMerhabaMerhaba"
   ```

## Extension Method Kullanım Alanları

1. **String İşlemleri**
   ```csharp
   public static class StringExtensions
   {
       public static string ToTitleCase(this string str)
       {
           return CultureInfo.CurrentCulture.TextInfo.ToTitleCase(str.ToLower());
       }
       
       public static bool IsValidEmail(this string email)
       {
           return Regex.IsMatch(email, @"^[^@\s]+@[^@\s]+\.[^@\s]+$");
       }
   }
   ```

2. **Koleksiyon İşlemleri**
   ```csharp
   public static class CollectionExtensions
   {
       public static IEnumerable<T> Shuffle<T>(this IEnumerable<T> source)
       {
           return source.OrderBy(x => Guid.NewGuid());
       }
       
       public static IEnumerable<T> DistinctBy<T, TKey>(this IEnumerable<T> source, Func<T, TKey> keySelector)
       {
           return source.GroupBy(keySelector).Select(x => x.First());
       }
   }
   ```

3. **DateTime İşlemleri**
   ```csharp
   public static class DateTimeExtensions
   {
       public static bool IsWeekend(this DateTime date)
       {
           return date.DayOfWeek == DayOfWeek.Saturday || date.DayOfWeek == DayOfWeek.Sunday;
       }
       
       public static int GetAge(this DateTime birthDate)
       {
           var today = DateTime.Today;
           var age = today.Year - birthDate.Year;
           if (birthDate.Date > today.AddYears(-age)) age--;
           return age;
       }
   }
   ```

## Extension Method Best Practices

1. **Namespace Kullanımı**
   ```csharp
   namespace MyApp.Extensions
   {
       public static class StringExtensions
       {
           // Extension methods
       }
   }
   ```

2. **Statik Sınıf Kullanımı**
   ```csharp
   public static class Extensions
   {
       // Tüm extension methods burada
   }
   ```

3. **Method İsimlendirme**
   ```csharp
   public static class StringExtensions
   {
       // Açıklayıcı isimler kullan
       public static string RemoveWhitespace(this string str)
       {
           return new string(str.Where(c => !char.IsWhiteSpace(c)).ToArray());
       }
   }
   ```

## Extension Method ve Interface'ler

1. **Interface Extension**
   ```csharp
   public interface ILogger
   {
       void Log(string message);
   }
   
   public static class LoggerExtensions
   {
       public static void LogError(this ILogger logger, Exception ex)
       {
           logger.Log($"Hata: {ex.Message}");
       }
   }
   ```

2. **Generic Interface Extension**
   ```csharp
   public interface IRepository<T>
   {
       T GetById(int id);
   }
   
   public static class RepositoryExtensions
   {
       public static IEnumerable<T> GetByIds<T>(this IRepository<T> repository, IEnumerable<int> ids)
       {
           return ids.Select(id => repository.GetById(id));
       }
   }
   ```

## Mülakat Soruları

1. **Extension Method Temelleri**
   - Extension method nedir ve ne işe yarar?
   - Extension method'lar nasıl tanımlanır?
   - Extension method'ların avantajları ve dezavantajları nelerdir?

2. **Extension Method Kullanımı**
   - Extension method'lar ne zaman kullanılmalıdır?
   - Extension method'lar ve normal method'lar arasındaki farklar nelerdir?
   - Extension method'ların performans etkisi nedir?

3. **Extension Method ve OOP**
   - Extension method'lar OOP prensiplerini nasıl etkiler?
   - Extension method'lar inheritance'a alternatif midir?
   - Extension method'lar encapsulation'ı nasıl etkiler?

4. **Extension Method ve Interface'ler**
   - Interface'lere extension method eklemek ne zaman uygundur?
   - Interface extension'ların kullanım alanları nelerdir?
   - Interface extension'ların sınırlamaları nelerdir?

5. **Extension Method ve Generic'ler**
   - Generic extension method'lar nasıl tanımlanır?
   - Generic extension method'ların kullanım alanları nelerdir?
   - Generic extension method'larda type constraint'ler nasıl kullanılır?

6. **Extension Method ve LINQ**
   - LINQ extension method'ları nasıl çalışır?
   - Custom LINQ extension method'ları nasıl yazılır?
   - LINQ extension method'larında deferred execution nasıl sağlanır?

7. **Extension Method ve Testing**
   - Extension method'lar nasıl test edilir?
   - Extension method testlerinde dikkat edilmesi gerekenler nelerdir?
   - Extension method'lar unit test'leri nasıl etkiler?

8. **Extension Method ve Performans**
   - Extension method'ların performans etkisi nedir?
   - Extension method'larda performans optimizasyonu nasıl yapılır?
   - Extension method'lar memory kullanımını nasıl etkiler?

9. **Extension Method ve Best Practices**
   - Extension method yazarken dikkat edilmesi gerekenler nelerdir?
   - Extension method isimlendirme kuralları nelerdir?
   - Extension method'larda exception handling nasıl yapılır?

10. **Extension Method ve Framework Entegrasyonu**
    - Extension method'lar framework'lerle nasıl entegre edilir?
    - Extension method'ların versioning'i nasıl yapılır?
    - Extension method'ların backward compatibility'si nasıl sağlanır?

## Örnek Kod Soruları

1. **String Extension**
   ```csharp
   public static class StringExtensions
   {
       public static string Reverse(this string str)
       {
           // Implementasyon
       }
   }
   ```

2. **Collection Extension**
   ```csharp
   public static class CollectionExtensions
   {
       public static T GetRandomItem<T>(this IEnumerable<T> source)
       {
           // Implementasyon
       }
   }
   ```

3. **DateTime Extension**
   ```csharp
   public static class DateTimeExtensions
   {
       public static bool IsBusinessDay(this DateTime date)
       {
           // Implementasyon (hafta sonları ve resmi tatiller hariç)
       }
   }
   ```

4. **LINQ Extension**
   ```csharp
   public static class LinqExtensions
   {
       public static IEnumerable<T> WhereNotNull<T>(this IEnumerable<T> source)
       {
           // Implementasyon
       }
   }
   ```

5. **Validation Extension**
   ```csharp
   public static class ValidationExtensions
   {
       public static bool IsValidTurkishIdNumber(this string idNumber)
       {
           // Implementasyon
       }
   }
   ```

## Kaynaklar

- [Microsoft Docs - Extension Methods](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/extension-methods)
- [Extension Methods Best Practices](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/extension-methods#general-guidelines)
- [LINQ Extension Methods](https://docs.microsoft.com/en-us/dotnet/api/system.linq.enumerable) 