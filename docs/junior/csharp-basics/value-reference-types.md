# Value ve Reference Types

## Genel Bakış

C#'ta veri tipleri iki ana kategoriye ayrılır: Value Types (Değer Tipleri) ve Reference Types (Referans Tipleri). Bu ayrım, verilerin bellekte nasıl saklandığını ve işlendiğini belirler.

## Value Types (Değer Tipleri)

1. **Temel Value Types**
   ```csharp
   // Sayısal tipler
   int number = 42;
   double pi = 3.14;
   decimal price = 99.99m;
   
   // Karakter ve boolean
   char letter = 'A';
   bool isTrue = true;
   
   // Struct'lar
   DateTime date = DateTime.Now;
   TimeSpan duration = TimeSpan.FromHours(1);
   ```

2. **Struct Tanımlama**
   ```csharp
   public struct Point
   {
       public int X { get; set; }
       public int Y { get; set; }
       
       public Point(int x, int y)
       {
           X = x;
           Y = y;
       }
   }
   
   // Kullanımı
   Point p1 = new Point(10, 20);
   Point p2 = p1; // Değer kopyalanır
   p2.X = 30; // p1 etkilenmez
   ```

3. **Enum Tanımlama**
   ```csharp
   public enum DayOfWeek
   {
       Monday,
       Tuesday,
       Wednesday,
       Thursday,
       Friday,
       Saturday,
       Sunday
   }
   
   // Kullanımı
   DayOfWeek today = DayOfWeek.Monday;
   ```

## Reference Types (Referans Tipleri)

1. **Temel Reference Types**
   ```csharp
   // String
   string name = "Ahmet";
   
   // Array
   int[] numbers = new int[] { 1, 2, 3 };
   
   // Class
   Person person = new Person { Name = "Mehmet" };
   ```

2. **Class Tanımlama**
   ```csharp
   public class Person
   {
       public string Name { get; set; }
       public int Age { get; set; }
       
       public Person(string name, int age)
       {
           Name = name;
           Age = age;
       }
   }
   
   // Kullanımı
   Person p1 = new Person("Ahmet", 30);
   Person p2 = p1; // Referans kopyalanır
   p2.Name = "Mehmet"; // p1 de etkilenir
   ```

## Value vs Reference Types Karşılaştırması

1. **Bellek Yönetimi**
   ```csharp
   // Value Type - Stack'te saklanır
   int x = 10;
   int y = x; // Değer kopyalanır
   
   // Reference Type - Heap'te saklanır
   Person p1 = new Person();
   Person p2 = p1; // Referans kopyalanır
   ```

2. **Null Değer**
   ```csharp
   // Value Type - Varsayılan değer
   int number = default; // 0
   
   // Reference Type - null
   Person person = default; // null
   ```

3. **Boxing/Unboxing**
   ```csharp
   // Value Type -> Object (Boxing)
   int number = 42;
   object boxed = number;
   
   // Object -> Value Type (Unboxing)
   int unboxed = (int)boxed;
   ```

## Value ve Reference Types Kullanım Örnekleri

1. **Method Parametreleri**
   ```csharp
   // Value Type - Değer kopyalanır
   void ModifyValue(int x)
   {
       x = 10; // Orijinal değer etkilenmez
   }
   
   // Reference Type - Referans kopyalanır
   void ModifyReference(Person p)
   {
       p.Name = "Yeni İsim"; // Orijinal nesne etkilenir
   }
   ```

2. **Return Değerleri**
   ```csharp
   // Value Type - Yeni kopya döner
   int GetValue()
   {
       int x = 42;
       return x; // Yeni kopya
   }
   
   // Reference Type - Aynı referans döner
   Person GetPerson()
   {
       Person p = new Person();
       return p; // Aynı referans
   }
   ```

## Value ve Reference Types Best Practices

1. **Struct vs Class Seçimi**
   ```csharp
   // Struct kullanımı (küçük, değişmez veriler için)
   public struct Point
   {
       public int X { get; }
       public int Y { get; }
   }
   
   // Class kullanımı (karmaşık, değişebilir veriler için)
   public class Person
   {
       public string Name { get; set; }
       public List<string> Addresses { get; set; }
   }
   ```

2. **Immutable Types**
   ```csharp
   // Immutable struct
   public readonly struct ImmutablePoint
   {
       public int X { get; }
       public int Y { get; }
       
       public ImmutablePoint(int x, int y)
       {
           X = x;
           Y = y;
       }
   }
   ```

## Mülakat Soruları

1. **Value Types Temelleri**
   - Value Types nedir ve nasıl çalışır?
   - Hangi veri tipleri value type'dır?
   - Value Types'ın avantajları ve dezavantajları nelerdir?

2. **Reference Types Temelleri**
   - Reference Types nedir ve nasıl çalışır?
   - Hangi veri tipleri reference type'dır?
   - Reference Types'ın avantajları ve dezavantajları nelerdir?

3. **Bellek Yönetimi**
   - Value ve Reference Types bellekte nasıl saklanır?
   - Stack ve Heap arasındaki farklar nelerdir?
   - Garbage Collection nasıl çalışır?

4. **Struct vs Class**
   - Struct ve Class arasındaki farklar nelerdir?
   - Hangi durumlarda struct kullanılmalıdır?
   - Struct kullanırken dikkat edilmesi gerekenler nelerdir?

5. **Boxing/Unboxing**
   - Boxing ve Unboxing nedir?
   - Boxing/Unboxing'in performans etkisi nedir?
   - Boxing/Unboxing nasıl önlenebilir?

6. **Method Parametreleri**
   - Value ve Reference Types method parametrelerinde nasıl davranır?
   - ref ve out parametreleri ne işe yarar?
   - Parametre geçişlerinde dikkat edilmesi gerekenler nelerdir?

7. **Null Handling**
   - Value ve Reference Types'ta null değer nasıl işlenir?
   - Nullable value types nasıl kullanılır?
   - Null reference exception nasıl önlenir?

8. **Performans Konuları**
   - Value ve Reference Types'ın performans etkileri nelerdir?
   - Struct kullanımında performans optimizasyonu nasıl yapılır?
   - Garbage Collection performansı nasıl optimize edilir?

9. **Thread Safety**
   - Value ve Reference Types thread safety açısından nasıl davranır?
   - Immutable types ne işe yarar?
   - Thread-safe kod yazarken dikkat edilmesi gerekenler nelerdir?

10. **Modern C# Özellikleri**
    - C# 7.0 ve sonrasında value types nasıl gelişti?
    - ref struct ve readonly struct nedir?
    - Modern C# özellikleri value ve reference types'ı nasıl etkiler?

## Örnek Kod Soruları

1. **Value Type Swap**
   ```csharp
   public static void Swap<T>(ref T a, ref T b) where T : struct
   {
       // Implementasyon
   }
   ```

2. **Reference Type Clone**
   ```csharp
   public static T DeepClone<T>(T source) where T : class
   {
       // Implementasyon
   }
   ```

3. **Immutable Builder**
   ```csharp
   public class ImmutablePersonBuilder
   {
       // Implementasyon
   }
   ```

4. **Value Type Validator**
   ```csharp
   public static bool ValidateValueType<T>(T value) where T : struct
   {
       // Implementasyon
   }
   ```

5. **Reference Type Cache**
   ```csharp
   public class ReferenceTypeCache<T> where T : class
   {
       // Implementasyon
   }
   ```

## Kaynaklar

- [Microsoft Docs - Value Types](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/value-types)
- [Microsoft Docs - Reference Types](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/reference-types)
- [Struct vs Class](https://docs.microsoft.com/en-us/dotnet/standard/design-guidelines/choosing-between-class-and-struct) 