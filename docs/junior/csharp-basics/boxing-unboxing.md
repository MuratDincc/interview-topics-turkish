# Boxing ve Unboxing

## Genel Bakış

Boxing ve Unboxing, C#'ta value type'ların reference type'lara dönüştürülmesi ve tekrar value type'lara dönüştürülmesi işlemleridir. Bu işlemler performans açısından önemli etkilere sahiptir.

## Boxing (Kutulama)

1. **Temel Boxing**
   ```csharp
   // Value type -> object (boxing)
   int number = 42;
   object boxed = number; // Boxing işlemi
   ```

2. **Boxing Örnekleri**
   ```csharp
   // Sayısal tipler
   double pi = 3.14;
   object boxedPi = pi;
   
   // Struct'lar
   DateTime now = DateTime.Now;
   object boxedNow = now;
   
   // Enum'lar
   DayOfWeek day = DayOfWeek.Monday;
   object boxedDay = day;
   ```

3. **Boxing ve Interface**
   ```csharp
   // Value type -> interface (boxing)
   IComparable comparable = 42; // Boxing
   ```

## Unboxing (Kutudan Çıkarma)

1. **Temel Unboxing**
   ```csharp
   // object -> value type (unboxing)
   object boxed = 42;
   int number = (int)boxed; // Unboxing işlemi
   ```

2. **Unboxing Örnekleri**
   ```csharp
   // Sayısal tipler
   object boxedNumber = 42;
   int number = (int)boxedNumber;
   
   // Struct'lar
   object boxedDate = DateTime.Now;
   DateTime date = (DateTime)boxedDate;
   
   // Enum'lar
   object boxedEnum = DayOfWeek.Monday;
   DayOfWeek day = (DayOfWeek)boxedEnum;
   ```

3. **Unboxing ve Interface**
   ```csharp
   // Interface -> value type (unboxing)
   IComparable comparable = 42;
   int number = (int)comparable;
   ```

## Boxing/Unboxing Performans Etkileri

1. **Bellek Kullanımı**
   ```csharp
   // Boxing öncesi
   int number = 42; // 4 byte
   
   // Boxing sonrası
   object boxed = number; // 4 byte (değer) + 8 byte (object header) + 4 byte (type pointer)
   ```

2. **Performans Ölçümü**
   ```csharp
   // Boxing olmadan
   int sum = 0;
   for (int i = 0; i < 1000000; i++)
   {
       sum += i;
   }
   
   // Boxing ile
   object sum = 0;
   for (int i = 0; i < 1000000; i++)
   {
       sum = (int)sum + i; // Boxing ve unboxing
   }
   ```

## Boxing/Unboxing Önleme Yöntemleri

1. **Generic Kullanımı**
   ```csharp
   // Boxing olmadan
   public class GenericList<T>
   {
       private T[] items;
       // ...
   }
   
   // Boxing ile
   public class ObjectList
   {
       private object[] items;
       // ...
   }
   ```

2. **Interface Kullanımı**
   ```csharp
   // Boxing olmadan
   public interface IValueContainer<T>
   {
       T Value { get; }
   }
   
   // Boxing ile
   public interface IValueContainer
   {
       object Value { get; }
   }
   ```

3. **Struct Kullanımı**
   ```csharp
   // Boxing olmadan
   public struct Point
   {
       public int X { get; }
       public int Y { get; }
   }
   
   // Boxing ile
   public class Point
   {
       public int X { get; }
       public int Y { get; }
   }
   ```

## Boxing/Unboxing ve Collections

1. **Generic Collections**
   ```csharp
   // Boxing olmadan
   List<int> numbers = new List<int>();
   numbers.Add(42);
   
   // Boxing ile
   ArrayList numbers = new ArrayList();
   numbers.Add(42); // Boxing
   ```

2. **Dictionary Kullanımı**
   ```csharp
   // Boxing olmadan
   Dictionary<int, string> dict = new Dictionary<int, string>();
   dict[42] = "Answer";
   
   // Boxing ile
   Hashtable dict = new Hashtable();
   dict[42] = "Answer"; // Boxing
   ```

## Boxing/Unboxing ve LINQ

1. **LINQ Sorguları**
   ```csharp
   // Boxing olmadan
   var numbers = new List<int> { 1, 2, 3 };
   var sum = numbers.Sum(); // No boxing
   
   // Boxing ile
   var objects = new ArrayList { 1, 2, 3 };
   var sum = objects.Cast<int>().Sum(); // Boxing
   ```

2. **LINQ ve Value Types**
   ```csharp
   // Boxing olmadan
   var points = new List<Point>();
   var xSum = points.Sum(p => p.X);
   
   // Boxing ile
   var points = new ArrayList();
   var xSum = points.Cast<Point>().Sum(p => p.X); // Boxing
   ```

## Mülakat Soruları

1. **Boxing/Unboxing Temelleri**
   - Boxing ve Unboxing nedir?
   - Hangi durumlarda boxing/unboxing gerçekleşir?
   - Boxing/unboxing'in performans etkileri nelerdir?

2. **Bellek Yönetimi**
   - Boxing işlemi bellekte nasıl gerçekleşir?
   - Boxing sonrası oluşan nesne nerede saklanır?
   - Boxing/unboxing'in garbage collection üzerindeki etkisi nedir?

3. **Performans Optimizasyonu**
   - Boxing/unboxing nasıl önlenebilir?
   - Generic tipler boxing/unboxing'i nasıl önler?
   - Value type'ların performans optimizasyonu nasıl yapılır?

4. **Collections ve Boxing**
   - Generic collections boxing/unboxing'i nasıl önler?
   - ArrayList vs List<T> performans farkları nelerdir?
   - Dictionary vs Hashtable boxing/unboxing açısından farkları nelerdir?

5. **LINQ ve Boxing**
   - LINQ sorgularında boxing/unboxing nasıl önlenir?
   - LINQ ve value type'ların kullanımında dikkat edilmesi gerekenler nelerdir?
   - LINQ performans optimizasyonu nasıl yapılır?

6. **Interface ve Boxing**
   - Interface'ler boxing/unboxing'i nasıl etkiler?
   - Generic interface'ler boxing/unboxing'i nasıl önler?
   - Interface kullanımında performans optimizasyonu nasıl yapılır?

7. **Struct ve Boxing**
   - Struct'lar boxing/unboxing'i nasıl etkiler?
   - Immutable struct'lar boxing/unboxing'i nasıl etkiler?
   - Struct kullanımında performans optimizasyonu nasıl yapılır?

8. **Modern C# Özellikleri**
   - C# 7.0 ve sonrasında boxing/unboxing nasıl gelişti?
   - ref struct ve readonly struct boxing/unboxing'i nasıl etkiler?
   - Modern C# özellikleri boxing/unboxing'i nasıl optimize eder?

## Örnek Kod Soruları

1. **Boxing Önleyici Generic Sınıf**
   ```csharp
   public class ValueContainer<T> where T : struct
   {
       // Implementasyon
   }
   ```

2. **Boxing Önleyici Collection**
   ```csharp
   public class ValueCollection<T> where T : struct
   {
       // Implementasyon
   }
   ```

3. **Boxing Önleyici Cache**
   ```csharp
   public class ValueCache<TKey, TValue> where TValue : struct
   {
       // Implementasyon
   }
   ```

4. **Boxing Önleyici LINQ Extension**
   ```csharp
   public static class ValueTypeExtensions
   {
       // Implementasyon
   }
   ```

5. **Boxing Önleyici Serializer**
   ```csharp
   public class ValueTypeSerializer<T> where T : struct
   {
       // Implementasyon
   }
   ```

## Kaynaklar

- [Microsoft Docs - Boxing and Unboxing](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/types/boxing-and-unboxing)
- [Microsoft Docs - Performance Tips](https://docs.microsoft.com/en-us/dotnet/framework/performance/performance-tips)
- [Microsoft Docs - Generic Collections](https://docs.microsoft.com/en-us/dotnet/standard/collections/generic) 