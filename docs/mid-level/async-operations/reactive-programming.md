# Reactive Programming

## Giriş

Reactive Programming, veri akışları ve değişiklik yayılımına dayalı bir programlama paradigmasıdır. .NET dünyasında Reactive Extensions (Rx.NET) ile uygulanır ve asenkron veri akışlarını yönetmek için güçlü bir araç seti sunar.

## Reactive Programming'in Önemi

1. **Asenkron Veri Yönetimi**
   - Veri akışı kontrolü
   - Olay tabanlı programlama
   - Zaman uyumsuz işlemler
   - Gerçek zamanlı veri işleme

2. **Kod Kalitesi**
   - Daha temiz kod yapısı
   - Daha az boilerplate kod
   - Daha iyi hata yönetimi
   - Daha kolay bakım

3. **Ölçeklenebilirlik**
   - Yüksek performans
   - Kaynak optimizasyonu
   - Paralel işleme
   - Dağıtık sistemler

## Reactive Programming Özellikleri

1. **Observable Pattern**
   - Veri kaynağı
   - Olay yayınlama
   - Abonelik yönetimi
   - Veri dönüşümü

2. **Operatörler**
   - Filtreleme
   - Dönüştürme
   - Birleştirme
   - Zamanlama

3. **Schedulers**
   - Thread yönetimi
   - Zamanlama
   - Eşzamanlılık kontrolü
   - Kaynak yönetimi

## Reactive Programming Kullanımı

1. **Temel Observable**
```csharp
public class DataProducer
{
    public IObservable<int> CreateObservable()
    {
        return Observable.Create<int>(observer =>
        {
            // Veri üretme
            observer.OnNext(1);
            observer.OnNext(2);
            observer.OnNext(3);
            observer.OnCompleted();

            return Disposable.Empty;
        });
    }

    public void SubscribeToObservable()
    {
        var observable = CreateObservable();
        
        var subscription = observable.Subscribe(
            value => Console.WriteLine($"Value: {value}"),
            error => Console.WriteLine($"Error: {error}"),
            () => Console.WriteLine("Completed")
        );
    }
}
```

2. **Event Stream**
```csharp
public class EventProcessor
{
    public IObservable<MouseEvent> CreateMouseEventStream()
    {
        return Observable.FromEventPattern<MouseEventHandler, MouseEventArgs>(
            h => MouseMove += h,
            h => MouseMove -= h
        ).Select(e => new MouseEvent(e.EventArgs.X, e.EventArgs.Y));
    }

    public void ProcessMouseEvents()
    {
        var mouseStream = CreateMouseEventStream();
        
        var subscription = mouseStream
            .Throttle(TimeSpan.FromMilliseconds(250))
            .Subscribe(e => ProcessEvent(e));
    }

    private void ProcessEvent(MouseEvent e)
    {
        // Olay işleme
    }
}
```

3. **Veri Dönüşümü**
```csharp
public class DataTransformer
{
    public IObservable<string> TransformData(IObservable<int> source)
    {
        return source
            .Where(x => x > 0)
            .Select(x => x * 2)
            .Select(x => $"Transformed: {x}");
    }

    public void ProcessTransformedData()
    {
        var source = Observable.Range(1, 5);
        var transformed = TransformData(source);
        
        transformed.Subscribe(Console.WriteLine);
    }
}
```

4. **Zamanlama İşlemleri**
```csharp
public class TimeBasedProcessor
{
    public IObservable<int> CreateTimedSequence()
    {
        return Observable.Interval(TimeSpan.FromSeconds(1))
            .Select(x => (int)x)
            .Take(5);
    }

    public void ProcessTimedSequence()
    {
        var sequence = CreateTimedSequence();
        
        sequence.Subscribe(
            x => Console.WriteLine($"Tick: {x}"),
            () => Console.WriteLine("Sequence completed")
        );
    }
}
```

5. **Hata Yönetimi**
```csharp
public class ErrorHandler
{
    public IObservable<int> CreateErrorProneStream()
    {
        return Observable.Create<int>(observer =>
        {
            try
            {
                observer.OnNext(1);
                throw new Exception("Simulated error");
                observer.OnNext(2);
            }
            catch (Exception ex)
            {
                observer.OnError(ex);
            }
            return Disposable.Empty;
        });
    }

    public void HandleErrors()
    {
        var stream = CreateErrorProneStream();
        
        stream
            .Catch<int, Exception>(ex =>
            {
                Console.WriteLine($"Error caught: {ex.Message}");
                return Observable.Return(-1);
            })
            .Subscribe(
                x => Console.WriteLine($"Value: {x}"),
                ex => Console.WriteLine($"Error: {ex}"),
                () => Console.WriteLine("Completed")
            );
    }
}
```

## Reactive Programming Best Practices

1. **Stream Tasarımı**
   - Tek sorumluluk prensibi
   - İsimlendirme kuralları
   - Hata yönetimi
   - Kaynak temizleme

2. **Performans**
   - Backpressure yönetimi
   - Buffer optimizasyonu
   - Memory kullanımı
   - CPU kullanımı

3. **Güvenlik**
   - Thread safety
   - Race condition önleme
   - Deadlock önleme
   - Kaynak yönetimi

4. **Monitoring**
   - Stream durumu
   - Performans metrikleri
   - Hata izleme
   - Kaynak kullanımı

## Mülakat Soruları

### Temel Sorular

1. **Reactive Programming nedir ve neden kullanılır?**
   - **Cevap**: Reactive Programming, veri akışları ve değişiklik yayılımına dayalı bir programlama paradigmasıdır. Asenkron veri akışlarını yönetmek, olay tabanlı programlama yapmak ve gerçek zamanlı veri işlemek için kullanılır.

2. **Reactive Programming'in temel özellikleri nelerdir?**
   - **Cevap**:
     - Observable pattern
     - Operatörler
     - Schedulers
     - Asenkron veri akışı
     - Olay yönetimi

3. **Reactive Programming'in avantajları nelerdir?**
   - **Cevap**:
     - Asenkron veri yönetimi
     - Kod kalitesi
     - Ölçeklenebilirlik
     - Performans
     - Esneklik

4. **Observable ve Observer nedir?**
   - **Cevap**: Observable, veri kaynağını temsil eden ve veri yayınlayan bir yapıdır. Observer ise bu verileri alan ve işleyen yapıdır. Observer pattern'in bir uygulamasıdır.

5. **Reactive Programming'de operatörler nelerdir?**
   - **Cevap**:
     - Filtreleme operatörleri (Where, Filter)
     - Dönüştürme operatörleri (Select, Map)
     - Birleştirme operatörleri (Merge, Concat)
     - Zamanlama operatörleri (Delay, Throttle)

### Teknik Sorular

1. **Observable nasıl oluşturulur?**
   - **Cevap**:
```csharp
public class ObservableCreator
{
    public IObservable<int> CreateObservable()
    {
        return Observable.Create<int>(observer =>
        {
            observer.OnNext(1);
            observer.OnNext(2);
            observer.OnCompleted();
            return Disposable.Empty;
        });
    }
}
```

2. **Event stream nasıl oluşturulur?**
   - **Cevap**:
```csharp
public class EventStreamCreator
{
    public IObservable<MouseEvent> CreateMouseEventStream()
    {
        return Observable.FromEventPattern<MouseEventHandler, MouseEventArgs>(
            h => MouseMove += h,
            h => MouseMove -= h
        ).Select(e => new MouseEvent(e.EventArgs.X, e.EventArgs.Y));
    }
}
```

3. **Veri dönüşümü nasıl yapılır?**
   - **Cevap**:
```csharp
public class DataTransformer
{
    public IObservable<string> TransformData(IObservable<int> source)
    {
        return source
            .Where(x => x > 0)
            .Select(x => x * 2)
            .Select(x => $"Transformed: {x}");
    }
}
```

4. **Hata yönetimi nasıl yapılır?**
   - **Cevap**:
```csharp
public class ErrorHandler
{
    public void HandleErrors()
    {
        var stream = CreateErrorProneStream();
        
        stream
            .Catch<int, Exception>(ex =>
            {
                Console.WriteLine($"Error caught: {ex.Message}");
                return Observable.Return(-1);
            })
            .Subscribe(
                x => Console.WriteLine($"Value: {x}"),
                ex => Console.WriteLine($"Error: {ex}"),
                () => Console.WriteLine("Completed")
            );
    }
}
```

5. **Zamanlama işlemleri nasıl yapılır?**
   - **Cevap**:
```csharp
public class TimeBasedProcessor
{
    public IObservable<int> CreateTimedSequence()
    {
        return Observable.Interval(TimeSpan.FromSeconds(1))
            .Select(x => (int)x)
            .Take(5);
    }
}
```

### İleri Seviye Sorular

1. **Reactive Programming'de performans optimizasyonu nasıl yapılır?**
   - **Cevap**:
     - Backpressure yönetimi
     - Buffer optimizasyonu
     - Memory kullanımı
     - CPU kullanımı
     - Thread pool yönetimi

2. **Reactive Programming'de memory leak nasıl önlenir?**
   - **Cevap**:
     - Subscription yönetimi
     - Dispose pattern
     - Weak references
     - Memory profiling
     - Resource cleanup

3. **Reactive Programming ile distributed sistemler nasıl yönetilir?**
   - **Cevap**:
     - Stream dağıtımı
     - Load balancing
     - Failover stratejileri
     - Consistency yönetimi
     - Monitoring

4. **Reactive Programming'de monitoring nasıl yapılır?**
   - **Cevap**:
     - Stream durumu takibi
     - Performans metrikleri
     - Hata izleme
     - Kaynak kullanımı
     - Profiling

5. **Reactive Programming'de scaling nasıl yapılır?**
   - **Cevap**:
     - Stream parçalama
     - Paralellik derecesi ayarı
     - Kaynak yönetimi
     - Load balancing
     - Performance tuning 