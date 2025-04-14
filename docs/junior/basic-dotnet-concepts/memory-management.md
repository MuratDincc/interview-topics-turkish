# Memory Management

## Genel Bakış
Memory Management (Bellek Yönetimi), .NET uygulamalarında bellek kullanımının verimli ve güvenli bir şekilde yönetilmesini sağlayan mekanizmalar bütünüdür. Bu konu, managed ve unmanaged bellek yönetimini, bellek tahsisi ve serbest bırakma işlemlerini kapsar.

## Bellek Yönetimi Temelleri

### Stack ve Heap
1. **Stack**
   - Değer tipleri (value types) saklar
   - Hızlı erişim
   - Otomatik temizleme
   - Sınırlı boyut

2. **Heap**
   - Referans tipleri (reference types) saklar
   - Dinamik boyut
   - Garbage Collection tarafından yönetilir
   - Daha yavaş erişim

### Değer ve Referans Tipleri
```csharp
// Değer tipi (Stack'te saklanır)
int number = 42;
DateTime date = DateTime.Now;

// Referans tipi (Heap'te saklanır)
string text = "Hello";
object obj = new object();
```

## Managed Bellek Yönetimi

### 1. Nesne Yaşam Döngüsü
- Oluşturma (Allocation)
- Kullanım
- Referans kaybı
- Garbage Collection
- Temizleme

### 2. Bellek Tahsisi
```csharp
// Yeni nesne oluşturma
var person = new Person();

// Dizi oluşturma
var numbers = new int[1000];

// Koleksiyon oluşturma
var list = new List<string>();
```

### 3. Bellek Serbest Bırakma
```csharp
// IDisposable kullanımı
using (var resource = new Resource())
{
    // Kaynak kullanımı
}

// Manuel temizleme
resource = null;
GC.Collect();
```

## Unmanaged Bellek Yönetimi

### 1. P/Invoke Kullanımı
```csharp
[DllImport("kernel32.dll")]
static extern IntPtr HeapAlloc(IntPtr hHeap, uint dwFlags, UIntPtr dwBytes);

[DllImport("kernel32.dll")]
static extern bool HeapFree(IntPtr hHeap, uint dwFlags, IntPtr lpMem);
```

### 2. Unsafe Kod
```csharp
unsafe
{
    int* pointer = stackalloc int[100];
    // pointer kullanımı
}
```

### 3. Marshal Sınıfı
```csharp
// Unmanaged bellek tahsisi
IntPtr ptr = Marshal.AllocHGlobal(1000);

// Bellek serbest bırakma
Marshal.FreeHGlobal(ptr);
```

## Bellek Optimizasyonu

### 1. Büyük Nesneler
- LOH (Large Object Heap) kullanımı
- Büyük nesnelerden kaçınma
- Pooling pattern kullanımı

### 2. String İşlemleri
```csharp
// StringBuilder kullanımı
var sb = new StringBuilder();
sb.Append("Hello");
sb.Append(" World");

// String interpolation
string message = $"Hello {name}";
```

### 3. Array Pool
```csharp
// ArrayPool kullanımı
var pool = ArrayPool<int>.Shared;
var array = pool.Rent(1000);
try
{
    // array kullanımı
}
finally
{
    pool.Return(array);
}
```

## Memory Leak Önleme

### 1. Event Handler'lar
```csharp
public class EventPublisher
{
    public event EventHandler SomethingHappened;

    public void Cleanup()
    {
        SomethingHappened = null;
    }
}

public class EventSubscriber
{
    private EventPublisher publisher;

    public void Subscribe()
    {
        publisher.SomethingHappened += OnSomethingHappened;
    }

    public void Unsubscribe()
    {
        publisher.SomethingHappened -= OnSomethingHappened;
    }
}
```

### 2. Static Referanslar
```csharp
public class Cache
{
    private static readonly Dictionary<string, object> _cache = 
        new Dictionary<string, object>();

    public void Add(string key, object value)
    {
        _cache[key] = value;
    }

    public void Remove(string key)
    {
        _cache.Remove(key);
    }
}
```

### 3. Timer Kullanımı
```csharp
public class TimerService : IDisposable
{
    private Timer _timer;

    public TimerService()
    {
        _timer = new Timer(TimerCallback, null, 1000, 1000);
    }

    public void Dispose()
    {
        _timer?.Dispose();
    }
}
```

## Best Practices

### 1. Bellek Yönetimi
- IDisposable pattern kullanın
- using statement kullanın
- Finalizer'ları dikkatli kullanın
- Weak references kullanın

### 2. Performans
- Büyük nesnelerden kaçının
- String concatenation yerine StringBuilder kullanın
- ArrayPool kullanın
- Boxing/Unboxing'den kaçının

### 3. Güvenlik
- Unsafe kod kullanımını minimize edin
- P/Invoke çağrılarını dikkatli yapın
- Bellek sızıntılarını önleyin
- Exception handling kullanın

## Sık Sorulan Sorular

### 1. Stack ve Heap arasındaki fark nedir?
- Stack değer tipleri, Heap referans tipleri saklar
- Stack otomatik temizlenir, Heap GC tarafından yönetilir
- Stack sınırlı boyutlu, Heap dinamik boyutlu
- Stack hızlı, Heap daha yavaş

### 2. Memory leak nasıl tespit edilir?
- Memory profiler araçları kullanın
- GC.Collect() çağrılarından sonra bellek kullanımını kontrol edin
- Event handler'ları ve static referansları kontrol edin
- Timer ve unmanaged kaynakları kontrol edin

### 3. Bellek optimizasyonu için neler yapılabilir?
- Büyük nesnelerden kaçının
- String işlemlerinde StringBuilder kullanın
- ArrayPool kullanın
- Boxing/Unboxing'den kaçının

## Kaynaklar
- [Memory Management](https://docs.microsoft.com/en-us/dotnet/standard/memory-and-spans/)
- [Unmanaged Memory](https://docs.microsoft.com/en-us/dotnet/standard/native-interop/)
- [Memory Profiling](https://docs.microsoft.com/en-us/visualstudio/profiling/memory-usage) 