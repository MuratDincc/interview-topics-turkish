# CLR (Common Language Runtime)

## Genel Bakış
CLR, .NET uygulamalarının çalışma zamanı ortamıdır. Bu bölümde CLR'ın temel bileşenlerini, çalışma prensiplerini ve önemli özelliklerini inceleyeceğiz.

## Mülakat Soruları ve Cevapları

### 1. CLR nedir ve temel görevleri nelerdir?
**Cevap:**
CLR (Common Language Runtime), .NET uygulamalarının çalışma zamanı ortamıdır. Temel görevleri:
- Memory management (Bellek yönetimi)
- Type safety (Tip güvenliği)
- Exception handling (Hata yönetimi)
- Security (Güvenlik)
- Thread management (İş parçacığı yönetimi)
- Code execution (Kod çalıştırma)

**Örnek Kod:**
```csharp
public class CLRExample
{
    // Memory management örneği
    public void MemoryExample()
    {
        // CLR heap'te yer ayırır
        var list = new List<int>();
        
        // CLR garbage collection yapar
        list = null;
    }

    // Type safety örneği
    public void TypeSafetyExample()
    {
        object obj = "string";
        // CLR runtime'da tip kontrolü yapar
        int number = (int)obj; // InvalidCastException
    }
}
```

### 2. CLR'ın bileşenleri nelerdir ve nasıl çalışır?
**Cevap:**
1. **Class Loader:**
   - Assembly'leri yükler
   - Metadata'yı okur
   - Tip bilgilerini oluşturur

2. **JIT Compiler:**
   - IL kodunu native koda dönüştürür
   - Optimizasyonlar yapar
   - Code caching kullanır

3. **Garbage Collector:**
   - Memory yönetimi yapar
   - Generations kullanır (0, 1, 2)
   - Finalization queue yönetir

4. **Security Engine:**
   - Code access security
   - Role-based security
   - Permission checking

**Örnek Kod:**
```csharp
public class CLRComponents
{
    // JIT Compilation örneği
    public void JITExample()
    {
        // İlk çalıştırmada JIT compilation
        var result = Calculate(5, 10);
        
        // Sonraki çalıştırmalarda cached native code
        result = Calculate(5, 10);
    }

    private int Calculate(int a, int b)
    {
        return a + b;
    }

    // Garbage Collection örneği
    public void GCExample()
    {
        // Generation 0'da nesne oluşturulur
        var obj = new LargeObject();
        
        // Nesne kullanılmaz hale gelir
        obj = null;
        
        // Garbage Collection tetiklenir
        GC.Collect();
    }
}
```

### 3. CLR'da memory management nasıl çalışır?
**Cevap:**
1. **Heap Yapısı:**
   - Small Object Heap (SOH)
   - Large Object Heap (LOH)
   - Generations (0, 1, 2)

2. **Allocation:**
   - Stack allocation (value types)
   - Heap allocation (reference types)
   - Pinned objects

3. **Collection:**
   - Mark and sweep algoritması
   - Compaction
   - Finalization

**Örnek Kod:**
```csharp
public class MemoryManagement
{
    // Stack allocation
    public void StackExample()
    {
        int number = 42; // Stack'te
        Point point = new Point(1, 2); // Stack'te (struct)
    }

    // Heap allocation
    public void HeapExample()
    {
        var list = new List<int>(); // Heap'te
        var obj = new object(); // Heap'te
    }

    // Pinned objects
    public unsafe void PinnedExample()
    {
        byte[] buffer = new byte[1000];
        fixed (byte* ptr = buffer)
        {
            // Pointer işlemleri
        }
    }
}
```

### 4. CLR'da exception handling nasıl çalışır?
**Cevap:**
1. **Exception Types:**
   - System.Exception
   - Custom exceptions
   - Checked/unchecked exceptions

2. **Exception Flow:**
   - Try-catch-finally blokları
   - Exception propagation
   - Stack unwinding

3. **Best Practices:**
   - Specific exception handling
   - Exception logging
   - Resource cleanup

**Örnek Kod:**
```csharp
public class ExceptionHandling
{
    public void HandleException()
    {
        try
        {
            // Riskli kod
            ProcessData();
        }
        catch (FileNotFoundException ex)
        {
            // Spesifik hata yönetimi
            LogError(ex);
            HandleFileNotFound();
        }
        catch (Exception ex)
        {
            // Genel hata yönetimi
            LogError(ex);
            throw;
        }
        finally
        {
            // Kaynak temizleme
            CleanupResources();
        }
    }

    private void LogError(Exception ex)
    {
        // Hata loglama
    }
}
```

### 5. CLR'da thread management nasıl çalışır?
**Cevap:**
1. **Thread Pool:**
   - Worker threads
   - I/O completion threads
   - Thread scheduling

2. **Synchronization:**
   - Lock
   - Monitor
   - Semaphore
   - Mutex

3. **Async/Await:**
   - Task-based asynchrony
   - State machine
   - Context switching

**Örnek Kod:**
```csharp
public class ThreadManagement
{
    // Thread Pool kullanımı
    public void ThreadPoolExample()
    {
        ThreadPool.QueueUserWorkItem(state =>
        {
            // Arka plan işi
            ProcessData();
        });
    }

    // Async/Await kullanımı
    public async Task AsyncExample()
    {
        await ProcessDataAsync();
        // Context switch
        await ProcessMoreDataAsync();
    }

    // Synchronization
    private readonly object _lock = new object();
    public void SynchronizedMethod()
    {
        lock (_lock)
        {
            // Thread-safe kod
            UpdateSharedResource();
        }
    }
}
```

## Best Practices
1. **Memory Management**
   - IDisposable pattern kullanımı
   - using statement
   - Large object heap yönetimi
   - Finalization'dan kaçınma

2. **Exception Handling**
   - Specific exception handling
   - Exception logging
   - Resource cleanup
   - Exception propagation

3. **Thread Safety**
   - Immutable objects
   - Thread synchronization
   - Async/await pattern
   - Concurrent collections

## Kaynaklar
- [CLR via C#](https://www.amazon.com/CLR-via-4th-Developer-Reference/dp/0735667454)
- [.NET Memory Management](https://docs.microsoft.com/tr-tr/dotnet/standard/garbage-collection/)
- [Threading in .NET](https://docs.microsoft.com/tr-tr/dotnet/standard/threading/) 