# Memory Management

## Genel Bakış

C#'ta bellek yönetimi, .NET CLR (Common Language Runtime) tarafından otomatik olarak yapılır. Bu sistem, geliştiricilerin bellek yönetimiyle ilgili detaylarla uğraşmasını engelleyerek, daha güvenli ve verimli kod yazılmasını sağlar.

## Stack ve Heap

1. **Stack (Yığın)**
   ```csharp
   // Stack'te saklanan değerler
   int number = 42;        // Value type
   double pi = 3.14;       // Value type
   bool isTrue = true;     // Value type
   
   // Method çağrıları
   void Method()
   {
       int localVar = 10;  // Stack'te saklanır
   }
   ```

2. **Heap (Yığın)**
   ```csharp
   // Heap'te saklanan nesneler
   string name = "Ahmet";  // Reference type
   Person person = new Person(); // Reference type
   int[] numbers = new int[10]; // Reference type
   ```

## Garbage Collection (Çöp Toplama)

1. **GC Temelleri**
   ```csharp
   // Nesne oluşturma
   Person person = new Person();
   
   // Referansı kaldırma
   person = null; // GC tarafından toplanabilir
   ```

2. **GC Generations**
   ```csharp
   // Generation 0 - Yeni nesneler
   var obj1 = new object();
   
   // Generation 1 - Hayatta kalan nesneler
   GC.Collect(0); // Generation 0 temizlenir
   
   // Generation 2 - Uzun süreli nesneler
   GC.Collect(1); // Generation 1 temizlenir
   ```

## IDisposable ve Dispose Pattern

1. **Temel Dispose Pattern**
   ```csharp
   public class Resource : IDisposable
   {
       private bool disposed = false;
       
       public void Dispose()
       {
           Dispose(true);
           GC.SuppressFinalize(this);
       }
       
       protected virtual void Dispose(bool disposing)
       {
           if (!disposed)
           {
               if (disposing)
               {
                   // Yönetilen kaynakları serbest bırak
               }
               // Yönetilmeyen kaynakları serbest bırak
               disposed = true;
           }
       }
       
       ~Resource()
       {
           Dispose(false);
       }
   }
   ```

2. **using Statement**
   ```csharp
   using (var resource = new Resource())
   {
       // Kaynak kullanımı
   } // Otomatik olarak Dispose çağrılır
   ```

## Memory Leaks (Bellek Sızıntıları)

1. **Event Handlers**
   ```csharp
   // Bellek sızıntısı örneği
   public class Publisher
   {
       public event EventHandler SomethingHappened;
   }
   
   public class Subscriber
   {
       public void Subscribe(Publisher publisher)
       {
           publisher.SomethingHappened += OnSomethingHappened;
       }
       
       private void OnSomethingHappened(object sender, EventArgs e)
       {
           // Event handler
       }
   }
   ```

2. **Static References**
   ```csharp
   // Statik referans örneği
   public static class Cache
   {
       private static List<object> items = new List<object>();
       
       public static void Add(object item)
       {
           items.Add(item);
       }
   }
   ```

## Memory Management Best Practices

1. **Value vs Reference Types**
   ```csharp
   // Value type kullanımı (küçük, değişmez veriler için)
   public struct Point
   {
       public int X { get; }
       public int Y { get; }
   }
   
   // Reference type kullanımı (karmaşık, değişebilir veriler için)
   public class Person
   {
       public string Name { get; set; }
       public List<string> Addresses { get; set; }
   }
   ```

2. **String Handling**
   ```csharp
   // String birleştirme
   string result = string.Concat("Hello", " ", "World");
   
   // StringBuilder kullanımı
   var builder = new StringBuilder();
   builder.Append("Hello");
   builder.Append(" ");
   builder.Append("World");
   string result = builder.ToString();
   ```

## Memory Profiling

1. **Memory Profiler Kullanımı**
   ```csharp
   // Memory snapshot alma
   var snapshot1 = GC.GetTotalMemory(false);
   
   // İşlemler
   var list = new List<int>();
   for (int i = 0; i < 1000000; i++)
   {
       list.Add(i);
   }
   
   var snapshot2 = GC.GetTotalMemory(false);
   var memoryUsed = snapshot2 - snapshot1;
   ```

2. **Weak References**
   ```csharp
   // Weak reference kullanımı
   var weakRef = new WeakReference(new object());
   
   if (weakRef.IsAlive)
   {
       var target = weakRef.Target;
   }
   ```

## Mülakat Soruları

1. **Bellek Yönetimi Temelleri**
   - Stack ve Heap arasındaki farklar nelerdir?
   - Value type ve Reference type'lar bellekte nasıl saklanır?
   - Garbage Collection nasıl çalışır?

2. **Garbage Collection**
   - GC Generations nedir ve nasıl çalışır?
   - GC ne zaman tetiklenir?
   - GC'nin performans etkileri nelerdir?

3. **IDisposable Pattern**
   - Dispose pattern nedir ve neden kullanılır?
   - using statement nasıl çalışır?
   - Finalizer ne zaman kullanılmalıdır?

4. **Memory Leaks**
   - Bellek sızıntıları nasıl oluşur?
   - Event handler'lar nasıl bellek sızıntısına neden olur?
   - Statik referansların riskleri nelerdir?

5. **Value vs Reference Types**
   - Value type ve Reference type ne zaman kullanılmalıdır?
   - Struct ve Class arasındaki bellek yönetimi farkları nelerdir?
   - Boxing/Unboxing'in bellek etkileri nelerdir?

6. **String Handling**
   - String immutability nedir?
   - StringBuilder ne zaman kullanılmalıdır?
   - String interning nedir?

7. **Memory Profiling**
   - Memory profiler nasıl kullanılır?
   - Weak references ne işe yarar?
   - Memory snapshot'lar nasıl analiz edilir?

8. **Best Practices**
   - Bellek yönetimi için en iyi uygulamalar nelerdir?
   - Büyük nesneler nasıl yönetilmelidir?
   - Bellek fragmentasyonu nasıl önlenir?

## Örnek Kod Soruları

1. **Memory Efficient Cache**
   ```csharp
   public class MemoryEfficientCache<TKey, TValue>
   {
       // Implementasyon
   }
   ```

2. **Resource Manager**
   ```csharp
   public class ResourceManager : IDisposable
   {
       // Implementasyon
   }
   ```

3. **Memory Monitor**
   ```csharp
   public class MemoryMonitor
   {
       // Implementasyon
   }
   ```

4. **Weak Reference Cache**
   ```csharp
   public class WeakReferenceCache<T>
   {
       // Implementasyon
   }
   ```

5. **Memory Optimized Collection**
   ```csharp
   public class MemoryOptimizedCollection<T>
   {
       // Implementasyon
   }
   ```

## Kaynaklar

- [Microsoft Docs - Memory Management](https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/)
- [Microsoft Docs - IDisposable Pattern](https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-dispose)
- [Microsoft Docs - Memory Profiling](https://docs.microsoft.com/en-us/visualstudio/profiling/memory-usage) 