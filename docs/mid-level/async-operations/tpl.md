# Task Parallel Library (TPL)

## Giriş

Task Parallel Library (TPL), .NET uygulamalarında paralel programlama ve eşzamanlılık için güçlü bir framework'tür. Paralel işlemleri kolaylaştırmak, performansı artırmak ve kod karmaşıklığını azaltmak için tasarlanmıştır.

## TPL'in Önemi

1. **Performans**
   - Çok çekirdekli işlemcilerden yararlanma
   - Paralel işlem optimizasyonu
   - Kaynak kullanımı optimizasyonu
   - Ölçeklenebilirlik

2. **Kod Kalitesi**
   - Daha temiz kod yapısı
   - Daha az boilerplate kod
   - Daha iyi hata yönetimi
   - Daha kolay bakım

3. **Üretkenlik**
   - Kolay kullanım
   - Zengin API
   - Entegre hata yönetimi
   - İleri seviye özellikler

## TPL Özellikleri

1. **Task Sınıfı**
   - Asenkron işlemler
   - İptal desteği
   - Hata yönetimi
   - Continuation desteği

2. **Parallel Sınıfı**
   - Parallel.For
   - Parallel.ForEach
   - Parallel.Invoke
   - Paralel LINQ (PLINQ)

3. **Dataflow Kütüphanesi**
   - Pipeline işlemleri
   - Mesaj geçişi
   - Veri dönüşümü
   - Akış kontrolü

## TPL Kullanımı

1. **Task Oluşturma**
```csharp
public class DataProcessor
{
    public async Task ProcessDataAsync()
    {
        // Basit task oluşturma
        var task = Task.Run(() => HeavyComputation());

        // Task ile çalışma
        await task;

        // Task sonucu alma
        var result = await Task.Run(() => ComputeResult());
    }

    private void HeavyComputation()
    {
        // Ağır hesaplama işlemi
        Thread.Sleep(1000);
    }

    private int ComputeResult()
    {
        // Sonuç hesaplama
        return 42;
    }
}
```

2. **Task Continuation**
```csharp
public class DataProcessor
{
    public async Task ProcessWithContinuationAsync()
    {
        var task = Task.Run(() => GetData())
            .ContinueWith(t => ProcessData(t.Result))
            .ContinueWith(t => SaveData(t.Result));

        await task;
    }

    private string GetData()
    {
        // Veri alma
        return "data";
    }

    private string ProcessData(string data)
    {
        // Veri işleme
        return data.ToUpper();
    }

    private void SaveData(string data)
    {
        // Veri kaydetme
    }
}
```

3. **Parallel.For Kullanımı**
```csharp
public class ParallelProcessor
{
    public void ProcessInParallel()
    {
        var data = new int[1000];
        
        Parallel.For(0, data.Length, i =>
        {
            data[i] = ComputeValue(i);
        });
    }

    private int ComputeValue(int index)
    {
        // Değer hesaplama
        return index * 2;
    }
}
```

4. **Parallel.ForEach Kullanımı**
```csharp
public class ParallelProcessor
{
    public void ProcessCollectionInParallel()
    {
        var items = Enumerable.Range(0, 1000).ToList();
        var results = new ConcurrentBag<int>();

        Parallel.ForEach(items, item =>
        {
            var result = ProcessItem(item);
            results.Add(result);
        });
    }

    private int ProcessItem(int item)
    {
        // Öğe işleme
        return item * 2;
    }
}
```

5. **Dataflow Kullanımı**
```csharp
public class DataflowProcessor
{
    public async Task ProcessWithDataflowAsync()
    {
        var transformBlock = new TransformBlock<int, string>(n =>
        {
            return n.ToString();
        });

        var actionBlock = new ActionBlock<string>(s =>
        {
            Console.WriteLine(s);
        });

        transformBlock.LinkTo(actionBlock, new DataflowLinkOptions { PropagateCompletion = true });

        for (int i = 0; i < 10; i++)
        {
            transformBlock.Post(i);
        }

        transformBlock.Complete();
        await actionBlock.Completion;
    }
}
```

## TPL Best Practices

1. **Task Tasarımı**
   - Küçük ve odaklı tasklar
   - İptal desteği
   - Hata yönetimi
   - Kaynak temizleme

2. **Performans**
   - Task boyutu optimizasyonu
   - Paralellik derecesi ayarı
   - Kaynak kullanımı
   - Ölçeklenebilirlik

3. **Güvenlik**
   - Thread-safe kod
   - Veri senkronizasyonu
   - Kaynak yönetimi
   - Hata izolasyonu

4. **Monitoring**
   - Task durumu takibi
   - Performans metrikleri
   - Hata izleme
   - Kaynak kullanımı

## Mülakat Soruları

### Temel Sorular

1. **TPL nedir ve neden kullanılır?**
   - **Cevap**: Task Parallel Library (TPL), .NET uygulamalarında paralel programlama ve eşzamanlılık için güçlü bir framework'tür. Paralel işlemleri kolaylaştırmak, performansı artırmak ve kod karmaşıklığını azaltmak için tasarlanmıştır.

2. **TPL'in temel özellikleri nelerdir?**
   - **Cevap**:
     - Task sınıfı
     - Parallel sınıfı
     - Dataflow kütüphanesi
     - Asenkron programlama desteği
     - Paralel LINQ

3. **TPL'in avantajları nelerdir?**
   - **Cevap**:
     - Performans artışı
     - Kod kalitesi
     - Üretkenlik
     - Ölçeklenebilirlik
     - Kolay kullanım

4. **Task ve Thread arasındaki fark nedir?**
   - **Cevap**: Task, daha yüksek seviyeli bir soyutlamadır ve thread havuzunu kullanır. Thread ise daha düşük seviyeli bir işletim sistemi kaynağıdır. Task'lar daha az kaynak kullanır ve daha kolay yönetilir.

5. **TPL'de task tipleri nelerdir?**
   - **Cevap**:
     - Task: Genel amaçlı task
     - Task<T>: Sonuç döndüren task
     - ValueTask: Hafif task
     - TaskCompletionSource: Manuel task kontrolü

### Teknik Sorular

1. **Task nasıl oluşturulur?**
   - **Cevap**:
```csharp
public class TaskCreator
{
    public Task CreateTask()
    {
        return Task.Run(() =>
        {
            // İşlem
        });
    }

    public Task<int> CreateTaskWithResult()
    {
        return Task.Run(() =>
        {
            return 42;
        });
    }
}
```

2. **Task continuation nasıl kullanılır?**
   - **Cevap**:
```csharp
public class TaskContinuation
{
    public async Task ProcessWithContinuation()
    {
        await Task.Run(() => GetData())
            .ContinueWith(t => ProcessData(t.Result))
            .ContinueWith(t => SaveData(t.Result));
    }
}
```

3. **Parallel.For nasıl kullanılır?**
   - **Cevap**:
```csharp
public class ParallelProcessor
{
    public void ProcessInParallel()
    {
        var data = new int[1000];
        
        Parallel.For(0, data.Length, i =>
        {
            data[i] = ComputeValue(i);
        });
    }
}
```

4. **Dataflow nasıl kullanılır?**
   - **Cevap**:
```csharp
public class DataflowProcessor
{
    public async Task ProcessWithDataflow()
    {
        var transformBlock = new TransformBlock<int, string>(n => n.ToString());
        var actionBlock = new ActionBlock<string>(s => Console.WriteLine(s));

        transformBlock.LinkTo(actionBlock);

        for (int i = 0; i < 10; i++)
        {
            transformBlock.Post(i);
        }

        transformBlock.Complete();
        await actionBlock.Completion;
    }
}
```

5. **TPL'de hata yönetimi nasıl yapılır?**
   - **Cevap**:
```csharp
public class TaskErrorHandler
{
    public async Task HandleErrors()
    {
        try
        {
            await Task.Run(() => RiskyOperation());
        }
        catch (Exception ex)
        {
            // Hata yönetimi
        }
    }

    private void RiskyOperation()
    {
        // Riskli işlem
    }
}
```

### İleri Seviye Sorular

1. **TPL'de performans optimizasyonu nasıl yapılır?**
   - **Cevap**:
     - Task boyutu optimizasyonu
     - Paralellik derecesi ayarı
     - Kaynak kullanımı
     - Ölçeklenebilirlik
     - Thread pool yönetimi

2. **TPL'de deadlock nasıl önlenir?**
   - **Cevap**:
     - Async/await kullanımı
     - Task.WhenAll kullanımı
     - Timeout mekanizmaları
     - Kaynak sıralaması
     - Deadlock tespiti

3. **TPL ile distributed sistemler nasıl yönetilir?**
   - **Cevap**:
     - Task dağıtımı
     - Load balancing
     - Failover stratejileri
     - Consistency yönetimi
     - Monitoring

4. **TPL'de monitoring nasıl yapılır?**
   - **Cevap**:
     - Task durumu takibi
     - Performans metrikleri
     - Hata izleme
     - Kaynak kullanımı
     - Profiling

5. **TPL'de scaling nasıl yapılır?**
   - **Cevap**:
     - Task parçalama
     - Paralellik derecesi ayarı
     - Kaynak yönetimi
     - Load balancing
     - Performance tuning 