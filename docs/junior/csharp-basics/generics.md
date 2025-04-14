# Generics

## Genel Bakış

Generics (Jenerikler), C#'ta tip güvenliğini sağlarken kod tekrarını önleyen ve yeniden kullanılabilirliği artıran bir özelliktir. Jenerikler sayesinde, aynı kodu farklı veri tipleriyle kullanabiliriz.

## Temel Jenerik Kullanımı

1. **Jenerik Sınıf**
   ```csharp
   public class GenericList<T>
   {
       private List<T> items = new List<T>();
       
       public void Add(T item)
       {
           items.Add(item);
       }
       
       public T Get(int index)
       {
           return items[index];
       }
   }
   
   // Kullanımı
   var stringList = new GenericList<string>();
   stringList.Add("Merhaba");
   
   var intList = new GenericList<int>();
   intList.Add(42);
   ```

2. **Jenerik Metod**
   ```csharp
   public static class GenericMethods
   {
       public static T Max<T>(T a, T b) where T : IComparable<T>
       {
           return a.CompareTo(b) > 0 ? a : b;
       }
   }
   
   // Kullanımı
   int max = GenericMethods.Max(5, 10);
   string longer = GenericMethods.Max("abc", "abcd");
   ```

3. **Jenerik Interface**
   ```csharp
   public interface IRepository<T>
   {
       T GetById(int id);
       void Add(T entity);
       void Update(T entity);
       void Delete(int id);
   }
   
   public class UserRepository : IRepository<User>
   {
       // Implementasyon
   }
   ```

## Jenerik Kısıtlamalar (Constraints)

1. **where T : class**
   ```csharp
   public class ReferenceTypeContainer<T> where T : class
   {
       public T Value { get; set; }
   }
   ```

2. **where T : struct**
   ```csharp
   public class ValueTypeContainer<T> where T : struct
   {
       public T Value { get; set; }
   }
   ```

3. **where T : new()**
   ```csharp
   public class DefaultConstructorContainer<T> where T : new()
   {
       public T CreateInstance()
       {
           return new T();
       }
   }
   ```

4. **where T : BaseClass**
   ```csharp
   public class BaseClassContainer<T> where T : BaseClass
   {
       public void DoSomething(T item)
       {
           item.BaseMethod();
       }
   }
   ```

5. **where T : Interface**
   ```csharp
   public class InterfaceContainer<T> where T : IComparable
   {
       public int Compare(T a, T b)
       {
           return a.CompareTo(b);
       }
   }
   ```

## Jenerik Koleksiyonlar

1. **List<T>**
   ```csharp
   List<string> names = new List<string>();
   names.Add("Ahmet");
   names.Add("Mehmet");
   ```

2. **Dictionary<TKey, TValue>**
   ```csharp
   Dictionary<int, string> users = new Dictionary<int, string>();
   users.Add(1, "Ahmet");
   users.Add(2, "Mehmet");
   ```

3. **Queue<T>**
   ```csharp
   Queue<string> messages = new Queue<string>();
   messages.Enqueue("İlk mesaj");
   messages.Enqueue("İkinci mesaj");
   ```

4. **Stack<T>**
   ```csharp
   Stack<int> numbers = new Stack<int>();
   numbers.Push(1);
   numbers.Push(2);
   ```

## Jenerik Delegeler

1. **Func<T>**
   ```csharp
   Func<int, int> square = x => x * x;
   int result = square(5); // 25
   ```

2. **Action<T>**
   ```csharp
   Action<string> print = message => Console.WriteLine(message);
   print("Merhaba Dünya");
   ```

3. **Predicate<T>**
   ```csharp
   Predicate<int> isEven = x => x % 2 == 0;
   bool result = isEven(4); // true
   ```

## Jenerik ve Reflection

```csharp
public class GenericReflection
{
    public static Type GetGenericType<T>()
    {
        return typeof(T);
    }
    
    public static object CreateGenericInstance(Type genericType, Type typeArgument)
    {
        Type constructedType = genericType.MakeGenericType(typeArgument);
        return Activator.CreateInstance(constructedType);
    }
}
```

## Mülakat Soruları

1. **Jenerik Temelleri**
   - Jenerik nedir ve ne işe yarar?
   - Jeneriklerin avantajları nelerdir?
   - Jenerikler ve boxing/unboxing arasındaki ilişki nedir?

2. **Jenerik Kısıtlamalar**
   - Jenerik kısıtlamalar nelerdir?
   - Hangi durumlarda hangi kısıtlamaları kullanmalıyız?
   - Jenerik kısıtlamaların performans etkisi nedir?

3. **Jenerik ve Tip Güvenliği**
   - Jenerikler tip güvenliğini nasıl sağlar?
   - Jenerikler ve runtime tip kontrolü nasıl çalışır?
   - Jeneriklerde tip dönüşümleri nasıl yapılır?

4. **Jenerik Koleksiyonlar**
   - Jenerik koleksiyonlar nelerdir?
   - Jenerik koleksiyonların avantajları nelerdir?
   - Hangi durumda hangi jenerik koleksiyonu kullanmalıyız?

5. **Jenerik Metodlar**
   - Jenerik metodlar nasıl tanımlanır?
   - Jenerik metodların kullanım alanları nelerdir?
   - Jenerik metodlarda tip çıkarımı nasıl çalışır?

6. **Jenerik ve Interface'ler**
   - Jenerik interface'ler nasıl tanımlanır?
   - Jenerik interface'lerin kullanım alanları nelerdir?
   - Jenerik interface'lerde covariance ve contravariance nedir?

7. **Jenerik ve Delegeler**
   - Jenerik delegeler nasıl tanımlanır?
   - Func, Action ve Predicate arasındaki farklar nelerdir?
   - Jenerik delegelerin kullanım alanları nelerdir?

8. **Jenerik ve Performans**
   - Jeneriklerin performans etkisi nedir?
   - Jeneriklerde memory kullanımı nasıldır?
   - Jeneriklerde boxing/unboxing nasıl önlenir?

9. **Jenerik ve Reflection**
   - Jenerik tiplerle reflection nasıl kullanılır?
   - Jenerik tiplerin runtime'da oluşturulması nasıl yapılır?
   - Jenerik tiplerin tip bilgisi nasıl alınır?

10. **Jenerik Best Practices**
    - Jenerik kullanırken dikkat edilmesi gerekenler nelerdir?
    - Jenerik isimlendirme kuralları nelerdir?
    - Jeneriklerde exception handling nasıl yapılır?

## Örnek Kod Soruları

1. **Jenerik Stack**
   ```csharp
   public class GenericStack<T>
   {
       // Implementasyon
   }
   ```

2. **Jenerik Sıralama**
   ```csharp
   public static class GenericSorter
   {
       public static void Sort<T>(T[] array) where T : IComparable<T>
       {
           // Implementasyon
       }
   }
   ```

3. **Jenerik Cache**
   ```csharp
   public class GenericCache<TKey, TValue>
   {
       // Implementasyon
   }
   ```

4. **Jenerik Repository**
   ```csharp
   public interface IGenericRepository<T> where T : class
   {
       // Interface tanımı
   }
   ```

5. **Jenerik Validator**
   ```csharp
   public class GenericValidator<T>
   {
       // Implementasyon
   }
   ```

## Kaynaklar

- [Microsoft Docs - Generics](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/generics/)
- [Generic Constraints](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/generics/constraints-on-type-parameters)
- [Generic Collections](https://docs.microsoft.com/en-us/dotnet/standard/generics/collections) 