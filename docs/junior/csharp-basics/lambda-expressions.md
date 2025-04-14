# Lambda Expressions

## Genel Bakış

Lambda Expressions (Lambda İfadeleri), C#'ta anonim fonksiyonlar oluşturmak için kullanılan kısa ve öz bir sözdizimidir. Lambda ifadeleri, özellikle LINQ sorgularında ve fonksiyonel programlama yaklaşımlarında yaygın olarak kullanılır.

## Temel Lambda İfadeleri

1. **Basit Lambda İfadesi**
   ```csharp
   // Tek parametreli lambda
   Func<int, int> square = x => x * x;
   int result = square(5); // 25
   
   // Çoklu parametreli lambda
   Func<int, int, int> add = (x, y) => x + y;
   int sum = add(3, 4); // 7
   ```

2. **Statement Lambda**
   ```csharp
   Action<string> printMessage = message =>
   {
       Console.WriteLine($"Mesaj: {message}");
       Console.WriteLine($"Uzunluk: {message.Length}");
   };
   
   printMessage("Merhaba Dünya");
   ```

3. **Expression Lambda**
   ```csharp
   Func<int, bool> isEven = x => x % 2 == 0;
   bool result = isEven(4); // true
   ```

## Lambda ve Delegeler

1. **Func Delegate**
   ```csharp
   // Son parametre dönüş tipini belirtir
   Func<int, int, int> multiply = (x, y) => x * y;
   int product = multiply(5, 3); // 15
   ```

2. **Action Delegate**
   ```csharp
   // Dönüş değeri olmayan işlemler için
   Action<string> log = message => Console.WriteLine(message);
   log("Hata oluştu!");
   ```

3. **Predicate Delegate**
   ```csharp
   // Boolean dönen işlemler için
   Predicate<int> isPositive = x => x > 0;
   bool result = isPositive(-5); // false
   ```

## Lambda ve LINQ

1. **Where Kullanımı**
   ```csharp
   var numbers = new List<int> { 1, 2, 3, 4, 5 };
   var evenNumbers = numbers.Where(x => x % 2 == 0);
   ```

2. **Select Kullanımı**
   ```csharp
   var names = new List<string> { "Ahmet", "Mehmet", "Ayşe" };
   var nameLengths = names.Select(name => name.Length);
   ```

3. **OrderBy Kullanımı**
   ```csharp
   var products = new List<Product>();
   var sortedProducts = products.OrderBy(p => p.Price);
   ```

## Lambda ve Event Handler'lar

```csharp
button.Click += (sender, e) =>
{
    MessageBox.Show("Butona tıklandı!");
};
```

## Lambda ve Asenkron Programlama

```csharp
async Task ProcessData()
{
    await Task.Run(() =>
    {
        // Uzun süren işlem
        Thread.Sleep(1000);
        Console.WriteLine("İşlem tamamlandı");
    });
}
```

## Lambda ve Closure'lar

```csharp
int multiplier = 2;
Func<int, int> multiply = x => x * multiplier;
int result = multiply(5); // 10

// multiplier değiştiğinde
multiplier = 3;
result = multiply(5); // 15
```

## Lambda Best Practices

1. **İsimlendirme**
   ```csharp
   // İyi
   Func<int, int> square = x => x * x;
   
   // Kötü
   Func<int, int> f = x => x * x;
   ```

2. **Karmaşıklık**
   ```csharp
   // İyi
   Func<int, bool> isEven = x => x % 2 == 0;
   
   // Kötü (çok karmaşık)
   Func<int, bool> complexCheck = x => 
   {
       var y = x * 2;
       var z = y + 5;
       return z % 3 == 0 && x > 10;
   };
   ```

3. **Okunabilirlik**
   ```csharp
   // İyi
   Func<string, bool> isValid = name => 
       !string.IsNullOrEmpty(name) && 
       name.Length >= 3;
   
   // Kötü
   Func<string, bool> isValid = name => !string.IsNullOrEmpty(name) && name.Length >= 3;
   ```

## Mülakat Soruları

1. **Lambda Temelleri**
   - Lambda ifadesi nedir?
   - Lambda ifadelerinin avantajları nelerdir?
   - Lambda ifadeleri ve anonim metodlar arasındaki farklar nelerdir?

2. **Lambda Sözdizimi**
   - Lambda ifadelerinin temel sözdizimi nasıldır?
   - Statement lambda ve expression lambda arasındaki farklar nelerdir?
   - Lambda ifadelerinde parametre tipleri nasıl belirtilir?

3. **Lambda ve Delegeler**
   - Lambda ifadeleri hangi delegate tipleriyle kullanılabilir?
   - Func, Action ve Predicate delegate'leri arasındaki farklar nelerdir?
   - Lambda ifadelerinde delegate dönüşümü nasıl çalışır?

4. **Lambda ve LINQ**
   - Lambda ifadeleri LINQ sorgularında nasıl kullanılır?
   - LINQ metodlarında lambda ifadelerinin kullanım örnekleri nelerdir?
   - Lambda ifadeleri ve extension metodlar arasındaki ilişki nedir?

5. **Lambda ve Closure'lar**
   - Closure nedir ve nasıl çalışır?
   - Lambda ifadelerinde closure kullanımının avantajları ve dezavantajları nelerdir?
   - Closure'larda memory leak riski nasıl önlenir?

6. **Lambda ve Performans**
   - Lambda ifadelerinin performans etkisi nedir?
   - Lambda ifadelerinde boxing/unboxing nasıl önlenir?
   - Lambda ifadelerinin derleme sürecindeki davranışı nasıldır?

7. **Lambda ve Asenkron Programlama**
   - Lambda ifadeleri asenkron metodlarda nasıl kullanılır?
   - Task.Run ile lambda kullanımı nasıl yapılır?
   - Asenkron lambda ifadelerinde exception handling nasıl yapılır?

8. **Lambda ve Event Handling**
   - Lambda ifadeleri event handler'larda nasıl kullanılır?
   - Event handler'larda lambda kullanmanın avantajları ve dezavantajları nelerdir?
   - Event handler'larda memory leak riski nasıl önlenir?

9. **Lambda Best Practices**
   - Lambda ifadeleri yazarken dikkat edilmesi gerekenler nelerdir?
   - Lambda ifadelerinde okunabilirlik nasıl sağlanır?
   - Lambda ifadelerinde test edilebilirlik nasıl sağlanır?

10. **Lambda ve Fonksiyonel Programlama**
    - Lambda ifadeleri fonksiyonel programlama yaklaşımında nasıl kullanılır?
    - Higher-order fonksiyonlarda lambda kullanımı nasıl yapılır?
    - Lambda ifadeleri ve immutability arasındaki ilişki nedir?

## Örnek Kod Soruları

1. **Lambda ile Filtreleme**
   ```csharp
   public static IEnumerable<T> Filter<T>(IEnumerable<T> source, Func<T, bool> predicate)
   {
       // Implementasyon
   }
   ```

2. **Lambda ile Dönüşüm**
   ```csharp
   public static IEnumerable<TResult> Transform<T, TResult>(
       IEnumerable<T> source, 
       Func<T, TResult> transform)
   {
       // Implementasyon
   }
   ```

3. **Lambda ile Toplama**
   ```csharp
   public static T Aggregate<T>(
       IEnumerable<T> source, 
       Func<T, T, T> accumulator)
   {
       // Implementasyon
   }
   ```

4. **Lambda ile Sıralama**
   ```csharp
   public static IEnumerable<T> SortBy<T, TKey>(
       IEnumerable<T> source, 
       Func<T, TKey> keySelector)
   {
       // Implementasyon
   }
   ```

5. **Lambda ile Gruplama**
   ```csharp
   public static IEnumerable<IGrouping<TKey, T>> GroupBy<T, TKey>(
       IEnumerable<T> source, 
       Func<T, TKey> keySelector)
   {
       // Implementasyon
   }
   ```

## Kaynaklar

- [Microsoft Docs - Lambda Expressions](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/lambda-expressions)
- [Lambda Expressions Best Practices](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/statements-expressions-operators/lambda-expressions)
- [LINQ and Lambda Expressions](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/linq/) 