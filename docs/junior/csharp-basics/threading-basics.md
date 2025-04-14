# Threading Temelleri

## Genel Bakış

Threading, bir uygulamanın aynı anda birden fazla işi paralel olarak yürütmesini sağlayan bir programlama tekniğidir. C#'ta System.Threading namespace'i altında bulunan sınıflar kullanılarak thread işlemleri yapılır.

## Thread Oluşturma ve Yönetimi

1. **Thread Oluşturma**
   ```csharp
   // Thread oluşturma ve başlatma
   Thread thread = new Thread(() => {
       Console.WriteLine("Thread çalışıyor");
   });
   thread.Start();
   
   // Thread parametreli metod
   Thread thread2 = new Thread((object param) => {
       Console.WriteLine($"Parametre: {param}");
   });
   thread2.Start("Merhaba");
   ```

2. **Thread Özellikleri**
   ```csharp
   Thread thread = new Thread(() => { });
   
   // Thread önceliği
   thread.Priority = ThreadPriority.Highest;
   
   // Thread durumu
   bool isAlive = thread.IsAlive;
   
   // Thread adı
   thread.Name = "WorkerThread";
   ```

3. **Thread Bekletme**
   ```csharp
   // Thread'i belirli süre bekletme
   Thread.Sleep(1000); // 1 saniye
   
   // Thread'i sonsuza kadar bekletme
   thread.Join();
   ```

## Thread Senkronizasyonu

1. **lock Kullanımı**
   ```csharp
   private object _lock = new object();
   private int _counter = 0;
   
   public void IncrementCounter()
   {
       lock (_lock)
       {
           _counter++;
       }
   }
   ```

2. **Monitor Kullanımı**
   ```csharp
   private object _monitor = new object();
   
   public void CriticalSection()
   {
       Monitor.Enter(_monitor);
       try
       {
           // Kritik kod
       }
       finally
       {
           Monitor.Exit(_monitor);
       }
   }
   ```

3. **Mutex Kullanımı**
   ```csharp
   private Mutex _mutex = new Mutex();
   
   public void ProtectedMethod()
   {
       _mutex.WaitOne();
       try
       {
           // Korunan kod
       }
       finally
       {
           _mutex.ReleaseMutex();
       }
   }
   ```

## Thread Pool Kullanımı

1. **ThreadPool ile İş Yürütme**
   ```csharp
   ThreadPool.QueueUserWorkItem((state) => {
       Console.WriteLine("ThreadPool thread'i çalışıyor");
   });
   ```

2. **ThreadPool Özellikleri**
   ```csharp
   // Minimum thread sayısı
   ThreadPool.SetMinThreads(4, 4);
   
   // Maximum thread sayısı
   ThreadPool.SetMaxThreads(16, 16);
   
   // Mevcut thread sayısı
   ThreadPool.GetAvailableThreads(out int workerThreads, out int completionPortThreads);
   ```

## Thread Güvenliği

1. **Thread-Safe Koleksiyonlar**
   ```csharp
   // Concurrent koleksiyonlar
   ConcurrentQueue<int> queue = new ConcurrentQueue<int>();
   ConcurrentStack<int> stack = new ConcurrentStack<int>();
   ConcurrentDictionary<string, int> dictionary = new ConcurrentDictionary<string, int>();
   ```

2. **Volatile Değişkenler**
   ```csharp
   private volatile bool _isRunning = true;
   
   public void Stop()
   {
       _isRunning = false;
   }
   ```

3. **ThreadLocal Değişkenler**
   ```csharp
   private ThreadLocal<int> _threadLocal = new ThreadLocal<int>(() => 0);
   
   public void UseThreadLocal()
   {
       _threadLocal.Value = 42;
   }
   ```

## Thread İletişimi

1. **AutoResetEvent Kullanımı**
   ```csharp
   private AutoResetEvent _event = new AutoResetEvent(false);
   
   public void SignalThread()
   {
       _event.Set();
   }
   
   public void WaitForSignal()
   {
       _event.WaitOne();
   }
   ```

2. **ManualResetEvent Kullanımı**
   ```csharp
   private ManualResetEvent _event = new ManualResetEvent(false);
   
   public void SignalAllThreads()
   {
       _event.Set();
   }
   
   public void ResetSignal()
   {
       _event.Reset();
   }
   ```

## Thread İptali

1. **CancellationToken Kullanımı**
   ```csharp
   private CancellationTokenSource _cts = new CancellationTokenSource();
   
   public void StartOperation()
   {
       Task.Run(() => {
           while (!_cts.Token.IsCancellationRequested)
           {
               // İşlem
           }
       }, _cts.Token);
   }
   
   public void CancelOperation()
   {
       _cts.Cancel();
   }
   ```

## Mülakat Soruları

1. **Thread Temelleri**
   - Thread nedir ve ne işe yarar?
   - Process ve Thread arasındaki farklar nelerdir?
   - Thread oluşturmanın maliyeti nedir?

2. **Thread Senkronizasyonu**
   - Race condition nedir ve nasıl önlenir?
   - Deadlock nedir ve nasıl önlenir?
   - lock, Monitor ve Mutex arasındaki farklar nelerdir?

3. **Thread Pool**
   - ThreadPool nedir ve ne zaman kullanılır?
   - ThreadPool'un avantajları ve dezavantajları nelerdir?
   - ThreadPool thread'leri nasıl yönetilir?

4. **Thread Güvenliği**
   - Thread-safe kod nedir?
   - Volatile anahtar kelimesi ne işe yarar?
   - ThreadLocal nedir ve ne zaman kullanılır?

5. **Thread İletişimi**
   - Thread'ler arası iletişim nasıl sağlanır?
   - AutoResetEvent ve ManualResetEvent arasındaki farklar nelerdir?
   - Thread'ler arası veri paylaşımı nasıl yapılır?

6. **Thread İptali**
   - Thread iptali nasıl yapılır?
   - CancellationToken nasıl kullanılır?
   - Thread iptalinde dikkat edilmesi gerekenler nelerdir?

7. **Performans**
   - Thread sayısı nasıl belirlenir?
   - Thread'lerde performans optimizasyonu nasıl yapılır?
   - Thread context switching nedir?

8. **Hata Yönetimi**
   - Thread'lerde exception handling nasıl yapılır?
   - Unhandled exception'lar nasıl yakalanır?
   - Thread'lerde hata raporlama nasıl yapılır?

9. **Resource Yönetimi**
   - Thread'lerde resource leak nasıl önlenir?
   - Thread'lerde memory kullanımı nasıl yönetilir?
   - Thread'lerde file handle'ları nasıl yönetilir?

10. **Best Practices**
    - Thread kullanımında best practices nelerdir?
    - Thread senkronizasyonunda dikkat edilmesi gerekenler nelerdir?
    - Thread güvenliği nasıl sağlanır?

## Örnek Kod Soruları

1. **Thread-Safe Counter**
   ```csharp
   public class ThreadSafeCounter
   {
       private int _count = 0;
       
       public void Increment()
       {
           // Implementasyon
       }
       
       public int GetCount()
       {
           // Implementasyon
       }
   }
   ```

2. **Producer-Consumer Pattern**
   ```csharp
   public class ProducerConsumer
   {
       private Queue<int> _queue = new Queue<int>();
       
       public void Start()
       {
           // Implementasyon
       }
   }
   ```

3. **Thread Pool Manager**
   ```csharp
   public class ThreadPoolManager
   {
       public void ExecuteTask(Action task)
       {
           // Implementasyon
       }
   }
   ```

4. **Thread İptal Mekanizması**
   ```csharp
   public class CancellableOperation
   {
       public void Start(CancellationToken token)
       {
           // Implementasyon
       }
   }
   ```

5. **Thread Senkronizasyonu**
   ```csharp
   public class SynchronizedResource
   {
       private object _resource;
       
       public void AccessResource()
       {
           // Implementasyon
       }
   }
   ```

## Kaynaklar

- [Microsoft Docs - Threading](https://docs.microsoft.com/en-us/dotnet/standard/threading/)
- [Thread Class](https://docs.microsoft.com/en-us/dotnet/api/system.threading.thread)
- [ThreadPool Class](https://docs.microsoft.com/en-us/dotnet/api/system.threading.threadpool) 