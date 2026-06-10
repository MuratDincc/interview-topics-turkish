# Benchmarking (BenchmarkDotNet)

## Genel Bakış

"Bu kod daha hızlı" iddiası, **ölçülmediği sürece** bir varsayımdır. Mikro-benchmark yazmak göründüğünden çok daha zordur: JIT optimizasyonları, warmup, dead code elimination, CPU frekans dalgalanması ve GC, elle yazılmış `Stopwatch` ölçümlerini güvenilmez kılar. **BenchmarkDotNet**, bu tuzakların hepsini yöneten endüstri standardı .NET benchmark kütüphanesidir.

## Stopwatch Neden Yetersiz?

```csharp
// ❌ Güvenilmez mikro-benchmark
var sw = Stopwatch.StartNew();
for (int i = 0; i < 1000; i++) DoWork();
sw.Stop();
Console.WriteLine(sw.ElapsedMilliseconds);
```

Bu yaklaşımın sorunları:

- **JIT warmup yok:** İlk çağrılar henüz optimize edilmemiş (Tier 0) koddur; ölçüme dahil olur.
- **Dead code elimination:** JIT, sonucu kullanılmayan `DoWork()`'ü tamamen silebilir → 0 ms ölçersin.
- **Tek çalıştırma:** İstatistiksel anlamlılık yok; gürültü (GC, OS scheduling) sonucu bozar.
- **Allocation görünmez:** Sadece süreyi ölçer, bellek tahsisini değil.

## BenchmarkDotNet Kurulumu

```bash
dotnet add package BenchmarkDotNet
```

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

[MemoryDiagnoser]   // allocation'ı da ölç
public class StringBenchmarks
{
    private readonly string[] _parts = { "Murat", "Dinc", "1990" };

    [Benchmark(Baseline = true)]
    public string StringConcat()
    {
        string result = "";
        foreach (var p in _parts) result += p + ";";
        return result;
    }

    [Benchmark]
    public string StringBuilder_()
    {
        var sb = new StringBuilder();
        foreach (var p in _parts) sb.Append(p).Append(';');
        return sb.ToString();
    }

    [Benchmark]
    public string StringJoin() => string.Join(';', _parts);
}

// Program.cs
BenchmarkRunner.Run<StringBenchmarks>();
```

`dotnet run -c Release` ile çalıştırılır (**Release şart** — Debug ölçümleri anlamsızdır).

## Örnek Çıktı ve Yorumu

```
| Method        | Mean      | Ratio | Allocated |
|-------------- |----------:|------:|----------:|
| StringConcat  | 142.3 ns  |  1.00 |     280 B |
| StringBuilder_|  88.1 ns  |  0.62 |     152 B |
| StringJoin    |  61.4 ns  |  0.43 |      96 B |
```

- **Mean:** Ortalama süre. Düşük = hızlı.
- **Ratio:** Baseline'a göre oran (`[Benchmark(Baseline = true)]`). 0.43 = baseline'ın %43'ü kadar süre.
- **Allocated:** Çağrı başına heap tahsisi — `[MemoryDiagnoser]` sayesinde. Genelde süreden **daha önemli** bir gösterge, çünkü GC baskısını doğrudan yansıtır.

## Önemli Attribute'lar

```csharp
[MemoryDiagnoser]                          // allocation ölçer
[SimpleJob(RuntimeMoniker.Net80)]          // belirli runtime
[Params(10, 100, 1000)]                    // farklı girdi boyutlarıyla çalıştır
public int Size;

[GlobalSetup]                              // ölçüm dışı hazırlık (bir kez)
public void Setup() { /* veri hazırla */ }

[Benchmark]
public void Method() { /* sadece bu ölçülür */ }
```

`[Params]` ile aynı benchmark farklı girdi boyutlarında çalışır — algoritmanın **ölçeklenmesini** (O(n) davranışını) görmek için çok değerlidir.

## Mikro-Benchmark Tuzakları

### 1. Sonucu döndür (dead code elimination)

```csharp
// ❌ JIT bunu silebilir — sonuç kullanılmıyor
[Benchmark]
public void Bad() { var x = Compute(); }

// ✅ Sonucu döndür; BenchmarkDotNet "tüketir"
[Benchmark]
public int Good() => Compute();
```

### 2. Setup'ı ölçüme katma

Veri hazırlama, dosya okuma gibi işleri `[GlobalSetup]`'a koy; yoksa benchmark'ın kendisini değil hazırlığı ölçersin.

### 3. Çok küçük işler yanıltır

Nanosaniyelik işlerde ölçüm gürültüsü baskın olur. Gerçek senaryoyu yansıtan **anlamlı bir iş** ölç.

### 4. Release + gerçekçi runtime

Her zaman `-c Release`. Hedef ortam neyse (örn. .NET 8) onunla ölç.

## Profiler ile Tamamlamak

BenchmarkDotNet **"hangisi daha hızlı?"** sorusunu yanıtlar (mikro). **"Uygulamamın neresi yavaş?"** sorusu (makro) için **profiler** gerekir: Visual Studio Profiler, dotTrace, PerfView, `dotnet-trace`/`dotnet-counters`. Önce profiler ile darboğazı bul, sonra BenchmarkDotNet ile alternatifleri karşılaştır.

→ [Profiling](../performance-optimization/profiling.md)

## Mülakat Soruları

### 1. Soru: "Stopwatch ile mikro-benchmark neden güvenilmezdir?"

**Cevap:** JIT warmup ölçüme dahil olur (ilk çağrılar optimize değildir), dead code elimination sonucu kullanılmayan kodu silebilir (sahte 0 ms), tek çalıştırma istatistiksel anlam taşımaz ve GC/OS gürültüsü sonucu bozar. Ayrıca allocation ölçülmez. BenchmarkDotNet warmup, çoklu iterasyon, istatistik ve allocation ölçümünü otomatik yönetir.

### 2. Soru: "Benchmark'ı neden Release modda çalıştırmak gerekir?"

**Cevap:** Debug modda JIT optimizasyonları kapalıdır (inlining yok, ek kontroller var). Debug ölçümleri gerçek üretim performansını yansıtmaz ve yanıltıcı sonuçlar üretir. Üretimde Release çalıştığı için benchmark da Release olmalıdır.

### 3. Soru: "Süre mi yoksa allocation mı daha önemli bir metrik?"

**Cevap:** Duruma göre, ama yüksek throughput'lu servislerde **allocation çoğu zaman daha kritiktir**. Düşük allocation, GC baskısını ve dolayısıyla p99 gecikme dalgalanmalarını azaltır. İki yöntem benzer süreye sahipse, daha az allocation üreteni seçmek genelde doğrudur. `[MemoryDiagnoser]` bu yüzden neredeyse her benchmark'ta açılır.

### 4. Soru: "[Params] ne işe yarar?"

**Cevap:** Aynı benchmark'ı farklı girdi boyutlarıyla (örn. 10, 100, 1000) otomatik çalıştırır. Bu, bir yöntemin **nasıl ölçeklendiğini** (O(n), O(n²)) ortaya çıkarır — küçük girdide hızlı görünen bir yaklaşım büyük girdide çökebilir. Algoritma karşılaştırmalarında kritik.

## Best Practices

- Her zaman **`-c Release`** ile çalıştır.
- `[MemoryDiagnoser]` ekle — allocation'ı her zaman gör.
- Benchmark metotları **sonuç döndürsün** (dead code elimination'a karşı).
- Hazırlık işini `[GlobalSetup]`'a taşı.
- Farklı boyutları `[Params]` ile test et.
- Önce **profiler** ile gerçek darboğazı bul, sonra benchmark ile alternatifleri kıyasla — rastgele kod benchmark'lama zaman kaybıdır.
- Baseline belirle (`Baseline = true`) ki "ne kadar iyileştirdim" net olsun.

## İlişkili Konular

- [Span ve Memory](span-and-memory.md)
- [Bellek ve Pooling](memory-and-pooling.md)
- [Profiling](../performance-optimization/profiling.md)

## Kaynaklar

- BenchmarkDotNet Documentation — benchmarkdotnet.org
- Andrey Akinshin — "Pro .NET Benchmarking"
- Microsoft Docs — .NET diagnostic tools (dotnet-trace, dotnet-counters)
