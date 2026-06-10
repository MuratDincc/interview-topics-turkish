# Yüksek Performans C# (High-Performance C#)

## Giriş

Çoğu uygulamada "yeterince hızlı" kod yazmak iş görür. Ancak **sıcak yollar** (hot paths), yüksek throughput'lu servisler, serializer'lar, parser'lar ve düşük gecikme gerektiren sistemlerde her allocation, her kopya ve her GC duraklaması fark yaratır. Bu bölüm, C# ve .NET runtime'ının **allocation'ı azaltan, GC baskısını düşüren ve CPU'yu verimli kullanan** araçlarını kapsar.

Bu konular, mid-level ve senior mülakatlarda "bu kodu nasıl daha hızlı/az allocation'lı yazarsın?" veya "Span ne işe yarar?" gibi sorularla doğrudan test edilir.

> **Önce ölç, sonra optimize et.** Bu bölümdeki tekniklerin çoğu okunabilirlikten ödün verir. Yalnızca **profiler/benchmark ile kanıtlanmış** darboğazlarda uygulanmalıdır. Erken optimizasyon gerçek bir tuzaktır.

## Kapsanan Konular

### 1. Span&lt;T&gt; ve Memory&lt;T&gt;
Allocation'sız, kopyasız bellek erişimi. Dilimleme (slicing), `stackalloc`, `ref struct` kuralları.

**Öğrenilecekler:**
- `Span<T>`, `ReadOnlySpan<T>`, `Memory<T>` farkları
- Stack vs heap, `stackalloc`
- Slicing ile kopyasız alt-dizi
- `ref struct` neden stack'te kalır?

→ [Span ve Memory](span-and-memory.md)

### 2. Bellek Havuzlama (Pooling)
Tekrar tekrar tahsis yerine yeniden kullanım: `ArrayPool<T>`, `ObjectPool`, struct vs class kararı.

**Öğrenilecekler:**
- `ArrayPool<T>` ve `MemoryPool<T>`
- `ObjectPool<T>` ile nesne yeniden kullanımı
- GC baskısı (GC pressure) nedir?
- Struct vs class ve boxing maliyeti

→ [Bellek ve Pooling](memory-and-pooling.md)

### 3. Benchmarking (BenchmarkDotNet)
Performansı **doğru** ölçmek; mikro-benchmark tuzakları, allocation ölçümü.

**Öğrenilecekler:**
- BenchmarkDotNet kurulumu
- `[MemoryDiagnoser]` ile allocation ölçümü
- JIT warmup, ölçüm tuzakları
- `Stopwatch` neden yetersiz?

→ [Benchmarking](benchmarking.md)

### 4. Allocation-Free Desenler
String işlemleri, SIMD, modern API'lerle çöp üretmeyen kod.

**Öğrenilecekler:**
- `string.Create`, `ValueStringBuilder`
- `SearchValues<T>`, UTF-8 literal'ler
- SIMD (`Vector<T>`) ile veri paralelliği
- Allocation üreten gizli noktalar

→ [Allocation-Free Desenler](allocation-free-patterns.md)

## İlişkili Konular

- [Memory Management (C# Temelleri)](../../junior/csharp-basics/memory-management.md) — Stack/heap, GC temelleri.
- [Boxing ve Unboxing](../../junior/csharp-basics/boxing-unboxing.md)
- [Garbage Collection](../../junior/basic-dotnet-concepts/garbage-collection.md)
- [Performance Optimization](../performance-optimization/index.md) — DB, cache, async seviyesi optimizasyon.
- [Profiling](../performance-optimization/profiling.md)

## Neden Önemli?

Modern .NET'in (özellikle `System.Text.Json`, Kestrel, gRPC, EF Core) yüksek performansının temelinde **tam da bu teknikler** vardır. Bir senior adayından, "neden bu kod çok allocation üretiyor?" sorusunu GC kuşakları (generations) ve heap baskısı üzerinden açıklaması ve `Span`/pooling ile çözmesi beklenir.
