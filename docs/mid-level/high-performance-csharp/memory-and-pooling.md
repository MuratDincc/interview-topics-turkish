# Bellek ve Pooling (Memory & Pooling)

## Genel Bakış

Yüksek throughput'lu sistemlerde performansın baş düşmanı çoğu zaman CPU değil, **GC baskısıdır** (GC pressure). Çok sayıda kısa ömürlü nesne tahsis etmek, garbage collector'ı sık sık tetikler; bu da duraklamalara (pause) ve gecikme dalgalanmalarına yol açar. Bu sayfa, **tahsisi azaltma** ve **var olan belleği yeniden kullanma** tekniklerini kapsar.

## GC Baskısı (GC Pressure) Nedir?

.NET GC kuşak (generation) tabanlıdır: yeni nesneler **Gen 0**'da doğar. Gen 0 dolduğunda bir collection tetiklenir. Çok allocation = sık Gen 0 collection = sık (kısa) duraklamalar. Allocation'lar bir kuşaktan kurtulursa Gen 1/Gen 2'ye terfi eder; Gen 2 collection'ları **pahalıdır**.

```csharp
// ❌ Döngüde her iterasyonda yeni buffer → Gen 0 dolar, GC sık çalışır
for (int i = 0; i < 1_000_000; i++)
{
    byte[] buffer = new byte[4096];   // her tur 4 KB çöp
    Process(buffer);
}
```

Hedef: Sıcak yolda allocation sayısını azaltmak. İki ana yol vardır — **havuzlama (pooling)** ve **struct kullanımı**.

## ArrayPool&lt;T&gt;: Dizi Yeniden Kullanımı

`ArrayPool<T>.Shared`, dizileri tahsis edip serbest bırakmak yerine bir havuzdan **kiralayıp geri verme** imkânı sunar. En sık kullanılan pooling aracıdır.

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(4096);   // en az 4096 uzunlukta dizi
try
{
    int read = await stream.ReadAsync(buffer.AsMemory(0, 4096));
    Process(buffer.AsSpan(0, read));
}
finally
{
    // KESİNLİKLE geri ver — yoksa havuz boşalır, fayda kaybolur
    ArrayPool<byte>.Shared.Return(buffer, clearArray: true);
}
```

**Kritik kurallar:**

- `Rent(n)`, **en az** n uzunlukta bir dizi döndürür — genelde daha büyük olabilir. Bu yüzden gerçek uzunluğu ayrıca takip et (`AsSpan(0, read)`).
- `Return` **mutlaka** çağrılmalı (`try/finally`). Unutulursa bellek sızmaz ama havuz faydasını yitirir.
- Geri verdikten sonra diziye **dokunma** — başka bir kod onu kiralamış olabilir.
- Hassas veri için `clearArray: true` ile içeriği sıfırla (sonraki kiralayan eski veriyi görmesin).

## MemoryPool&lt;T&gt;

`Memory<T>` döndüren pooling gerektiğinde (async senaryolar) `MemoryPool<T>` kullanılır. `IMemoryOwner<T>` döndürür ve `Dispose` ile geri verilir:

```csharp
using IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(4096);
Memory<byte> memory = owner.Memory;
await stream.ReadAsync(memory);
// using bloğu çıkışında otomatik Return
```

## ObjectPool&lt;T&gt;: Nesne Yeniden Kullanımı

Pahalı kurulumu olan nesneler (örn. `StringBuilder`, parser, büyük buffer'lı nesneler) için `Microsoft.Extensions.ObjectPool`:

```csharp
var provider = new DefaultObjectPoolProvider();
ObjectPool<StringBuilder> pool = provider.CreateStringBuilderPool();

StringBuilder sb = pool.Get();
try
{
    sb.Append("merhaba").Append(' ').Append("dünya");
    return sb.ToString();
}
finally
{
    pool.Return(sb);   // Return, StringBuilder'ı temizleyip havuza koyar
}
```

> ASP.NET Core içte `StringBuilder`, `DbContext` havuzlaması (`AddDbContextPool`) gibi yerlerde tam da bunu kullanır.

## Struct vs Class: Allocation Kararı

`class` her zaman **heap**'te tahsis edilir (GC takip eder). `struct` ise **inline** (stack'te, dizinin/alanın içinde) yaşar — küçük, kısa ömürlü, değer-semantikli veriler için allocation üretmez.

```csharp
// Sıcak yolda milyonlarca kez oluşan küçük bir koordinat
public readonly struct Point(int X, int Y);   // heap allocation YOK

// Aynısı class olsaydı her oluşturma bir Gen 0 allocation olurdu
```

**Struct ne zaman?** Küçük (≤ ~16-24 byte), immutable, değer-semantiği doğal, kısa ömürlü. **Class ne zaman?** Büyük, mutable, kimlik (identity) önemli, kalıtım gerekli.

> **Dikkat:** Büyük struct'ları kopyalamak (metoda değer olarak geçmek) pahalıdır. Büyük struct'ı `in` parametresiyle (readonly referans) geçir veya class kullan.

## Boxing: Gizli Allocation

Bir değer tipini (`int`, `struct`) `object` veya bir interface'e atamak **boxing** üretir — heap'te bir kutu nesnesi tahsis edilir. Sıcak yollarda en sinsi allocation kaynağıdır:

```csharp
// ❌ Boxing: her int bir heap nesnesine kutulanır
object o = 42;
ArrayList list = new();  list.Add(42);     // boxing

// ❌ Gizli boxing: değer tipinde struct enumerator yerine interface üzerinden
IEnumerable<int> seq = ...;

// ✅ Generic ile boxing yok
List<int> typed = new();  typed.Add(42);   // boxing yok
```

→ Detay: [Boxing ve Unboxing](../../junior/csharp-basics/boxing-unboxing.md)

## Mülakat Soruları

### 1. Soru: "ArrayPool kullanırken Return'ü unutursak ne olur?"

**Cevap:** Bellek **sızmaz** — kiralanan dizi sıradan bir GC nesnesidir, referans kalmazsa toplanır. Ancak havuz o diziyi geri alamadığı için bir sonraki `Rent` çağrısı havuzda boş yer bulamaz ve **yeni allocation** yapar. Yani Return unutulursa pooling'in tüm faydası kaybolur ve kod sıradan `new[]` gibi davranır. Bu yüzden `try/finally` zorunludur.

### 2. Soru: "GC baskısı nedir, neden önemlidir?"

**Cevap:** Kısa ömürlü çok sayıda allocation, Gen 0'ı hızla doldurup sık GC collection'larını tetikler. Her collection (özellikle Gen 2) uygulamayı kısa süre durdurur (pause) ve gecikme dalgalanmaları (latency spikes) yaratır. Yüksek throughput'lu servislerde bu, p99 gecikmesini ciddi bozar. Pooling ve struct kullanımı allocation'ı azaltarak GC sıklığını düşürür.

### 3. Soru: "Her şeyi struct yapsak performans hep artar mı?"

**Cevap:** Hayır. Struct yalnızca **küçük ve kısa ömürlü** veride kazandırır. Büyük struct'lar metoda değer olarak geçerken **kopyalanır** — bu kopya maliyeti allocation'dan daha pahalı olabilir. Ayrıca struct'ı `object`/interface'e atamak boxing üretir (allocation geri gelir). Büyük veya mutable veride class daha iyidir. Karar boyut, ömür ve kullanım şekline bağlıdır; benchmark ile doğrulanmalıdır.

### 4. Soru: "Boxing'i nasıl fark eder ve önlersin?"

**Cevap:** Değer tipini `object`, non-generic koleksiyon (`ArrayList`), veya generic olmayan interface'e atadığında oluşur. Önlemek için: generic koleksiyonlar (`List<int>`), generic kısıtlar, `Span<T>`. Fark etmek için: BenchmarkDotNet `[MemoryDiagnoser]` ile allocation ölçmek veya bir allocation profiler (dotMemory, PerfView) kullanmak.

## Best Practices

- Sıcak yolda tekrar tahsis edilen büyük buffer'lar için **`ArrayPool<T>`** kullan; `try/finally` ile mutlaka `Return`.
- `Rent` döndürülen dizinin istenenden **büyük olabileceğini** unutma; gerçek uzunluğu ayrı takip et.
- Küçük, immutable, kısa ömürlü veriler için `struct`; büyük/mutable için `class`.
- Büyük struct'ları `in` ile geç (kopya maliyetini önle).
- Boxing'i `[MemoryDiagnoser]` ile avla; generic API'lerle önle.
- Pooling'i **ölçülmüş** darboğazlarda uygula; gereksiz pooling kodu karmaşıklaştırır ve bug (geri verilmiş buffer'a yazma) riski yaratır.

## İlişkili Konular

- [Span ve Memory](span-and-memory.md)
- [Allocation-Free Desenler](allocation-free-patterns.md)
- [Boxing ve Unboxing](../../junior/csharp-basics/boxing-unboxing.md)
- [Garbage Collection](../../junior/basic-dotnet-concepts/garbage-collection.md)
- [Memory Management (Performance)](../performance-optimization/memory-management.md)

## Kaynaklar

- Microsoft Docs — ArrayPool&lt;T&gt; ve Microsoft.Extensions.ObjectPool
- "Pro .NET Memory Management" — Konrad Kokosa
- Stephen Toub — "Performance Improvements in .NET" serisi
