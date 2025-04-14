# Nullable Types

## Genel Bakış

Nullable Types (Null Olabilen Tipler), C#'ta değer tiplerinin (value types) null değer alabilmesini sağlayan bir özelliktir. Bu özellik sayesinde, değer tipleri için null durumunu temsil edebilir ve bu durumu kontrol edebiliriz.

## Nullable Types Tanımlama

1. **Nullable<T> Sözdizimi**
   ```csharp
   Nullable<int> nullableInt = null;
   Nullable<double> nullableDouble = 3.14;
   ```

2. **? Operatörü ile Kısa Sözdizimi**
   ```csharp
   int? nullableInt = null;
   double? nullableDouble = 3.14;
   ```

## Nullable Types Özellikleri

1. **HasValue**
   ```csharp
   int? number = 42;
   if (number.HasValue)
   {
       Console.WriteLine($"Değer: {number.Value}");
   }
   else
   {
       Console.WriteLine("Değer null");
   }
   ```

2. **Value**
   ```csharp
   int? number = 42;
   int actualNumber = number.Value; // 42
   
   int? nullNumber = null;
   // nullNumber.Value kullanımı InvalidOperationException fırlatır
   ```

3. **GetValueOrDefault**
   ```csharp
   int? number = null;
   int result = number.GetValueOrDefault(); // 0
   int resultWithDefault = number.GetValueOrDefault(42); // 42
   ```

## Nullable Types Operatörleri

1. **Null-Coalescing Operatörü (??)**
   ```csharp
   int? number = null;
   int result = number ?? 42; // 42
   ```

2. **Null-Conditional Operatörü (?.)**
   ```csharp
   Person? person = null;
   string? name = person?.Name; // null
   
   // Zincirleme kullanım
   string? city = person?.Address?.City;
   ```

3. **Null-Forgiving Operatörü (!)**
   ```csharp
   int? number = null;
   // Derleyiciye null olmadığını garanti ediyoruz
   int result = number!.Value;
   ```

## Nullable Types Dönüşümleri

1. **Implicit Conversion**
   ```csharp
   int regularInt = 42;
   int? nullableInt = regularInt; // Otomatik dönüşüm
   ```

2. **Explicit Conversion**
   ```csharp
   int? nullableInt = 42;
   int regularInt = (int)nullableInt; // Explicit dönüşüm
   
   int? nullValue = null;
   // (int)nullValue kullanımı InvalidOperationException fırlatır
   ```

## Nullable Types ve LINQ

```csharp
List<int?> numbers = new List<int?> { 1, null, 3, null, 5 };

// Null olmayan değerleri filtreleme
var nonNullNumbers = numbers.Where(n => n.HasValue);

// Null değerleri varsayılan değerle değiştirme
var defaultNumbers = numbers.Select(n => n ?? 0);
```

## Nullable Types ve Pattern Matching

```csharp
int? number = 42;

// Pattern matching ile kontrol
if (number is int value)
{
    Console.WriteLine($"Değer: {value}");
}

// Switch expression ile kullanım
string result = number switch
{
    null => "Değer null",
    int value => $"Değer: {value}"
};
```

## Nullable Types Best Practices

1. **Null Kontrolü**
   ```csharp
   // Kötü
   int? number = GetNumber();
   int result = number.Value; // Potansiyel exception
   
   // İyi
   int? number = GetNumber();
   if (number.HasValue)
   {
       int result = number.Value;
   }
   ```

2. **Default Değer Kullanımı**
   ```csharp
   // Kötü
   int? number = GetNumber();
   int result = number ?? 0;
   
   // İyi
   int? number = GetNumber();
   int result = number.GetValueOrDefault(0);
   ```

3. **Null-Conditional Operatörü**
   ```csharp
   // Kötü
   if (person != null && person.Address != null)
   {
       string city = person.Address.City;
   }
   
   // İyi
   string? city = person?.Address?.City;
   ```

## Mülakat Soruları

1. **Nullable Types Temelleri**
   - Nullable Types nedir ve ne işe yarar?
   - Nullable<T> ve ? operatörü arasındaki fark nedir?
   - Hangi tipler nullable olabilir?

2. **Nullable Types Özellikleri**
   - HasValue ve Value özellikleri ne işe yarar?
   - GetValueOrDefault metodu nasıl kullanılır?
   - Nullable Types'ın varsayılan değeri nedir?

3. **Nullable Types Operatörleri**
   - Null-Coalescing operatörü (??) nasıl çalışır?
   - Null-Conditional operatörü (?.) nasıl kullanılır?
   - Null-Forgiving operatörü (!) ne işe yarar?

4. **Nullable Types ve Tip Dönüşümleri**
   - Nullable Types'tan normal tipe dönüşüm nasıl yapılır?
   - Dönüşümlerde dikkat edilmesi gerekenler nelerdir?
   - Implicit ve explicit dönüşümler arasındaki farklar nelerdir?

5. **Nullable Types ve Exception Handling**
   - Nullable Types kullanırken hangi exception'lar oluşabilir?
   - Bu exception'lar nasıl önlenebilir?
   - Nullable Types ve try-catch blokları nasıl kullanılır?

6. **Nullable Types ve Performans**
   - Nullable Types'ın performans etkisi nedir?
   - Boxing/unboxing durumları nasıl oluşur?
   - Performans optimizasyonu nasıl yapılır?

7. **Nullable Types ve LINQ**
   - LINQ sorgularında Nullable Types nasıl kullanılır?
   - Null değerler LINQ sorgularında nasıl işlenir?
   - LINQ ve Nullable Types performans etkileşimi nasıldır?

8. **Nullable Types ve Pattern Matching**
   - Pattern matching ile Nullable Types nasıl kontrol edilir?
   - Switch expression'larda Nullable Types nasıl kullanılır?
   - Pattern matching performans etkisi nedir?

9. **Nullable Types Best Practices**
   - Nullable Types kullanırken dikkat edilmesi gerekenler nelerdir?
   - Kod okunabilirliği nasıl sağlanır?
   - Test edilebilirlik nasıl artırılır?

10. **Nullable Types ve Modern C#**
    - C# 8.0 ve sonrasında Nullable Types nasıl gelişti?
    - Nullable reference types ile ilişkisi nedir?
    - Modern C# özellikleri Nullable Types'ı nasıl etkiler?

## Örnek Kod Soruları

1. **Nullable Int Toplama**
   ```csharp
   public static int? AddNullableInts(int? a, int? b)
   {
       // Implementasyon
   }
   ```

2. **Nullable String Kontrolü**
   ```csharp
   public static bool IsNullOrEmpty(string? text)
   {
       // Implementasyon
   }
   ```

3. **Nullable Object Dönüşümü**
   ```csharp
   public static T? ConvertToNullable<T>(object value) where T : struct
   {
       // Implementasyon
   }
   ```

4. **Nullable List Filtreleme**
   ```csharp
   public static IEnumerable<T> FilterNulls<T>(IEnumerable<T?> source)
   {
       // Implementasyon
   }
   ```

5. **Nullable Dictionary İşlemleri**
   ```csharp
   public static TValue? GetValueOrDefault<TKey, TValue>(
       IDictionary<TKey, TValue> dictionary,
       TKey key)
   {
       // Implementasyon
   }
   ```

## Kaynaklar

- [Microsoft Docs - Nullable Types](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/nullable-value-types)
- [Nullable Reference Types](https://docs.microsoft.com/en-us/dotnet/csharp/nullable-references)
- [Nullable Types Best Practices](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/nullable-value-types#nullable-types-and-the-null-coalescing-operator) 