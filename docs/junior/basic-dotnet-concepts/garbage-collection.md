# Garbage Collection

## Genel Bakış
Bu bölümde, .NET'te bellek yönetiminin temel mekanizması olan Garbage Collection'ı, çalışma prensiplerini ve performans optimizasyonlarını inceleyeceğiz.

## Mülakat Soruları ve Cevapları

### 1. Garbage Collection nedir ve nasıl çalışır?
**Cevap:**
Garbage Collection (GC):
- Otomatik bellek yönetimi sağlar
- Kullanılmayan nesneleri temizler
- Heap'teki bellek alanını yönetir
- Generations (0, 1, 2) kullanır
- Mark and Sweep algoritması çalıştırır

**Örnek Kod:**
```csharp
public class GCExample
{
    public void MemoryAllocation()
    {
        // Generation 0'da nesne oluşturulur
        var obj = new LargeObject();
        
        // Nesne kullanılmaz hale gelir
        obj = null;
        
        // Garbage Collection tetiklenir
        GC.Collect();
        
        // Finalization queue kontrolü
        GC.WaitForPendingFinalizers();
    }
}
```

### 2. Generations kavramı nedir ve nasıl çalışır?
**Cevap:**
Generations:
- **Generation 0:**
  - Yeni oluşturulan nesneler
  - Sık GC tetiklenir
  - Küçük nesneler

- **Generation 1:**
  - Generation 0'dan kurtulan nesneler
  - Daha az sıklıkta GC
  - Orta ömürlü nesneler

- **Generation 2:**
  - Generation 1'den kurtulan nesneler
  - Nadiren GC
  - Uzun ömürlü nesneler

**Örnek Kod:**
```csharp
public class GenerationsExample
{
    public void TrackGenerations()
    {
        var obj = new object();
        
        // Nesnenin generation'ını öğrenme
        Console.WriteLine(GC.GetGeneration(obj)); // 0
        
        GC.Collect();
        Console.WriteLine(GC.GetGeneration(obj)); // 1
        
        GC.Collect();
        Console.WriteLine(GC.GetGeneration(obj)); // 2
    }
}
```

### 3. Finalization nedir ve nasıl kullanılır?
**Cevap:**
Finalization:
- Nesne temizlenmeden önce çalışır
- Finalizer metodu kullanır
- Finalization queue yönetir
- Performans maliyeti yüksektir

**Örnek Kod:**
```csharp
public class FinalizationExample : IDisposable
{
    private bool disposed = false;
    
    // Finalizer
    ~FinalizationExample()
    {
        Dispose(false);
    }
    
    // IDisposable implementasyonu
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
                // Managed kaynakları temizle
            }
            
            // Unmanaged kaynakları temizle
            disposed = true;
        }
    }
}
```

### 4. Large Object Heap (LOH) nedir ve nasıl yönetilir?
**Cevap:**
Large Object Heap:
- 85KB'dan büyük nesneler için
- Özel heap bölgesi
- Compaction yapılmaz
- Memory fragmentation riski

**Örnek Kod:**
```csharp
public class LOHExample
{
    public void ManageLargeObjects()
    {
        // LOH'ta yer ayrılır
        var largeArray = new byte[100000];
        
        // LOH temizliği
        largeArray = null;
        GC.Collect();
        
        // LOH durumu kontrolü
        var lohSize = GC.GetTotalMemory(false);
        Console.WriteLine($"LOH Size: {lohSize}");
    }
}
```

### 5. GC Performansını nasıl optimize edebiliriz?
**Cevap:**
GC Optimizasyonu:
- IDisposable pattern kullanımı
- using statement
- Object pooling
- Lazy initialization
- StringBuilder kullanımı
- Collection kapasitesi belirleme

**Örnek Kod:**
```csharp
public class GCOptimization
{
    // Object pooling
    private static readonly ObjectPool<ExpensiveObject> pool = 
        new ObjectPool<ExpensiveObject>(() => new ExpensiveObject());
    
    public void OptimizedMemoryUsage()
    {
        // Object pooling kullanımı
        var obj = pool.Get();
        try
        {
            // İşlemler
        }
        finally
        {
            pool.Return(obj);
        }
        
        // StringBuilder kullanımı
        var sb = new StringBuilder(1000);
        for (int i = 0; i < 1000; i++)
        {
            sb.Append(i);
        }
        
        // Collection kapasitesi
        var list = new List<int>(1000);
    }
}
```

## Best Practices
1. **Memory Yönetimi**
   - IDisposable pattern kullanımı
   - using statement
   - Object pooling
   - Lazy initialization

2. **Performans Optimizasyonu**
   - Finalization'dan kaçınma
   - LOH kullanımını minimize etme
   - Collection kapasitelerini belirleme
   - StringBuilder kullanımı

3. **Debug ve Monitoring**
   - Memory profiler kullanımı
   - GC istatistiklerini izleme
   - Memory leak tespiti
   - Performance counter'lar

## Kaynaklar
- [Garbage Collection in .NET](https://docs.microsoft.com/tr-tr/dotnet/standard/garbage-collection/)
- [Memory Management and Garbage Collection](https://docs.microsoft.com/tr-tr/dotnet/standard/performance/memory-management)
- [GC Class](https://docs.microsoft.com/tr-tr/dotnet/api/system.gc) 