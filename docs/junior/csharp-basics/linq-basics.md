# LINQ Temelleri

## Genel Bakış

LINQ (Language Integrated Query), C#'ta veri sorgulama için kullanılan güçlü bir özelliktir. LINQ sayesinde, farklı veri kaynaklarından (koleksiyonlar, veritabanları, XML vb.) veri sorgulama işlemleri tutarlı bir şekilde yapılabilir.

## LINQ Sorgu Sözdizimi

1. **Method Syntax**
   ```csharp
   var numbers = new List<int> { 1, 2, 3, 4, 5 };
   var evenNumbers = numbers.Where(n => n % 2 == 0)
                           .OrderBy(n => n)
                           .ToList();
   ```

2. **Query Syntax**
   ```csharp
   var numbers = new List<int> { 1, 2, 3, 4, 5 };
   var evenNumbers = (from n in numbers
                     where n % 2 == 0
                     orderby n
                     select n).ToList();
   ```

## Temel LINQ Operatörleri

1. **Filtreleme**
   ```csharp
   // Where
   var adults = people.Where(p => p.Age >= 18);
   
   // OfType
   var strings = objects.OfType<string>();
   ```

2. **Projeksiyon**
   ```csharp
   // Select
   var names = people.Select(p => p.Name);
   
   // SelectMany
   var allPhones = people.SelectMany(p => p.Phones);
   ```

3. **Sıralama**
   ```csharp
   // OrderBy
   var sortedByName = people.OrderBy(p => p.Name);
   
   // ThenBy
   var sorted = people.OrderBy(p => p.Age)
                     .ThenBy(p => p.Name);
   ```

4. **Gruplama**
   ```csharp
   // GroupBy
   var groups = people.GroupBy(p => p.City);
   
   // Her grup için işlem
   foreach (var group in groups)
   {
       Console.WriteLine($"Şehir: {group.Key}");
       foreach (var person in group)
       {
           Console.WriteLine($"  - {person.Name}");
       }
   }
   ```

5. **Birleştirme**
   ```csharp
   // Join
   var joined = from p in people
               join c in cities on p.CityId equals c.Id
               select new { p.Name, c.CityName };
   ```

## LINQ ve Koleksiyonlar

1. **Listeler**
   ```csharp
   List<int> numbers = new List<int> { 1, 2, 3, 4, 5 };
   var sum = numbers.Sum();
   var avg = numbers.Average();
   var max = numbers.Max();
   ```

2. **Diziler**
   ```csharp
   int[] numbers = { 1, 2, 3, 4, 5 };
   var first = numbers.First();
   var last = numbers.Last();
   var single = numbers.Single(n => n == 3);
   ```

3. **Dictionary**
   ```csharp
   Dictionary<int, string> dict = new Dictionary<int, string>();
   var keys = dict.Keys.Where(k => k > 10);
   var values = dict.Values.Where(v => v.StartsWith("A"));
   ```

## LINQ ve Null Kontrolü

```csharp
// Null kontrolü
var nonNullNames = people.Where(p => p?.Name != null)
                        .Select(p => p.Name);

// Default değer atama
var firstOrDefault = numbers.FirstOrDefault(n => n > 10, -1);
```

## LINQ ve Performans

1. **Deferred Execution**
   ```csharp
   // Sorgu henüz çalıştırılmadı
   var query = numbers.Where(n => n % 2 == 0);
   
   // Sorgu çalıştırıldı
   var result = query.ToList();
   ```

2. **Immediate Execution**
   ```csharp
   // Sorgu hemen çalıştırıldı
   var count = numbers.Count(n => n % 2 == 0);
   var sum = numbers.Sum();
   ```

## LINQ Best Practices

1. **Performans Optimizasyonu**
   ```csharp
   // Kötü
   var result = numbers.Where(n => n % 2 == 0)
                      .OrderBy(n => n)
                      .Where(n => n > 10)
                      .ToList();
   
   // İyi
   var result = numbers.Where(n => n % 2 == 0 && n > 10)
                      .OrderBy(n => n)
                      .ToList();
   ```

2. **Null Kontrolü**
   ```csharp
   // Kötü
   var names = people.Select(p => p.Name.ToUpper());
   
   // İyi
   var names = people.Where(p => p?.Name != null)
                    .Select(p => p.Name.ToUpper());
   ```

## Mülakat Soruları

1. **LINQ Temelleri**
   - LINQ nedir ve ne işe yarar?
   - LINQ sorgu sözdizimleri nelerdir?
   - LINQ'in avantajları nelerdir?

2. **LINQ Operatörleri**
   - Temel LINQ operatörleri nelerdir?
   - Where ve Select operatörleri nasıl kullanılır?
   - GroupBy ve Join operatörleri ne işe yarar?

3. **LINQ ve Performans**
   - Deferred Execution nedir?
   - LINQ sorgularında performans nasıl optimize edilir?
   - LINQ sorgularında memory kullanımı nasıl kontrol edilir?

4. **LINQ ve Null Handling**
   - LINQ sorgularında null kontrolü nasıl yapılır?
   - FirstOrDefault ve SingleOrDefault arasındaki farklar nelerdir?
   - Null-conditional operatörler LINQ'de nasıl kullanılır?

5. **LINQ ve Koleksiyonlar**
   - LINQ hangi koleksiyon tipleriyle kullanılabilir?
   - IEnumerable ve IQueryable arasındaki farklar nelerdir?
   - LINQ sorguları koleksiyonları nasıl etkiler?

6. **LINQ ve Veritabanı**
   - LINQ to SQL nedir?
   - Entity Framework'te LINQ nasıl kullanılır?
   - LINQ sorguları veritabanında nasıl çalıştırılır?

7. **LINQ ve Lambda**
   - LINQ ve Lambda ifadeleri arasındaki ilişki nedir?
   - Lambda ifadeleri LINQ sorgularında nasıl kullanılır?
   - Expression ve Func delegate'leri LINQ'de nasıl kullanılır?

8. **LINQ ve Asenkron Programlama**
   - Asenkron LINQ sorguları nasıl yazılır?
   - Task ve LINQ nasıl birlikte kullanılır?
   - Asenkron LINQ sorgularında exception handling nasıl yapılır?

9. **LINQ Best Practices**
   - LINQ sorguları yazarken dikkat edilmesi gerekenler nelerdir?
   - LINQ sorgularında okunabilirlik nasıl sağlanır?
   - LINQ sorgularında test edilebilirlik nasıl sağlanır?

10. **LINQ ve Fonksiyonel Programlama**
    - LINQ fonksiyonel programlama prensiplerini nasıl kullanır?
    - LINQ'de immutability nasıl sağlanır?
    - LINQ ve higher-order fonksiyonlar arasındaki ilişki nedir?

## Örnek Kod Soruları

1. **LINQ ile Filtreleme**
   ```csharp
   public static IEnumerable<T> Filter<T>(IEnumerable<T> source, Func<T, bool> predicate)
   {
       // Implementasyon
   }
   ```

2. **LINQ ile Gruplama**
   ```csharp
   public static IEnumerable<IGrouping<TKey, T>> GroupBy<T, TKey>(
       IEnumerable<T> source, 
       Func<T, TKey> keySelector)
   {
       // Implementasyon
   }
   ```

3. **LINQ ile Birleştirme**
   ```csharp
   public static IEnumerable<TResult> Join<T, TInner, TKey, TResult>(
       IEnumerable<T> outer,
       IEnumerable<TInner> inner,
       Func<T, TKey> outerKeySelector,
       Func<TInner, TKey> innerKeySelector,
       Func<T, TInner, TResult> resultSelector)
   {
       // Implementasyon
   }
   ```

4. **LINQ ile Sıralama**
   ```csharp
   public static IOrderedEnumerable<T> OrderBy<T, TKey>(
       IEnumerable<T> source,
       Func<T, TKey> keySelector)
   {
       // Implementasyon
   }
   ```

5. **LINQ ile Agregasyon**
   ```csharp
   public static T Aggregate<T>(
       IEnumerable<T> source,
       Func<T, T, T> func)
   {
       // Implementasyon
   }
   ```

## Kaynaklar

- [Microsoft Docs - LINQ](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/linq/)
- [LINQ Query Syntax](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/linq/query-syntax-and-method-syntax-in-linq)
- [LINQ Standard Query Operators](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/linq/standard-query-operators-overview) 