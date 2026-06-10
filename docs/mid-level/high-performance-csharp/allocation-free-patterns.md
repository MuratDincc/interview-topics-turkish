# Allocation-Free Desenler

## Genel Bakış

Bu sayfa, [Span ve Memory](span-and-memory.md) ile [Pooling](memory-and-pooling.md) temellerini günlük kodda **çöp (garbage) üretmeden** uygulamaya döken modern .NET API'lerini ve desenlerini toplar: string işlemleri, UTF-8, `SearchValues`, SIMD ve gizli allocation kaynakları.

## String İşlemleri: Allocation Cehennemi

String'ler immutable olduğu için her birleştirme/değişiklik **yeni bir string** (heap allocation) üretir. Sıcak yollarda en büyük allocation kaynağıdır.

### 1. `string.Create` ile tek seferde inşa

Çıktının uzunluğu önceden biliniyorsa, ara string üretmeden doğrudan tampona yazılır:

```csharp
// "USER-00042" gibi bir kimlik üret — ara allocation YOK
public static string FormatId(int id) =>
    string.Create(10, id, static (span, value) =>
    {
        "USER-".CopyTo(span);
        value.TryFormat(span[5..], out _, "D5");
    });
```

Karşılaştırma: `"USER-" + id.ToString("D5")` iki ara string + bir birleştirme üretir; `string.Create` sıfır ara allocation üretir.

### 2. `StringBuilder` (bilinmeyen uzunluk)

Döngüde string biriktiriyorsan `+=` yerine `StringBuilder` — ve mümkünse [havuzdan](memory-and-pooling.md) al:

```csharp
var sb = new StringBuilder(capacity: 256);   // kapasiteyi baştan ver → yeniden tahsisi azalt
foreach (var item in items) sb.Append(item).Append(',');
```

### 3. Span ile allocation'sız ayrıştırma

`Split` bir dizi + N string üretir. `ReadOnlySpan<char>` ile bölme hiç allocation üretmez:

```csharp
ReadOnlySpan<char> line = "12,34,56";
foreach (Range r in line.Split(','))    // .NET 8+ Span.Split, allocation'sız enumerator
{
    int value = int.Parse(line[r]);
}
```

## UTF-8 Doğrudan İşleme

Web/JSON dünyası UTF-8'dir. `string` (UTF-16) ↔ byte dönüşümü hem CPU hem allocation maliyetidir. .NET, UTF-8 ile **doğrudan** çalışmayı destekler:

```csharp
// UTF-8 literal — derleme zamanında byte'lara çevrilir, runtime'da string yok
ReadOnlySpan<byte> prefix = "data: "u8;

// IUtf8SpanFormattable: sayıyı doğrudan UTF-8 byte tamponuna yaz (string atlanır)
Span<byte> buffer = stackalloc byte[32];
int value = 42;
value.TryFormat(buffer, out int written);   // UTF-16 string üretmeden
```

`System.Text.Json`'ın hızının temeli budur: byte → değer, ara string üretmeden.

## SearchValues&lt;T&gt;: Hızlı Çoklu Arama

Bir karakter/byte kümesini tekrar tekrar aramak gerektiğinde (`IndexOfAny` ile), `SearchValues<T>` (.NET 8+) önceden optimize edilmiş, SIMD-hızlandırmalı bir arama yapısı sağlar:

```csharp
private static readonly SearchValues<char> Vowels =
    SearchValues.Create("aeiouAEIOU");

public static int FirstVowel(ReadOnlySpan<char> text) =>
    text.IndexOfAny(Vowels);   // klasik IndexOfAny'den belirgin hızlı, allocation'sız
```

## SIMD: Veri Paralelliği (Vector&lt;T&gt;)

SIMD (Single Instruction, Multiple Data), tek CPU komutuyla birden çok veri öğesini işler. Büyük dizilerde toplama/karşılaştırma gibi işlemleri katlarca hızlandırır:

```csharp
// Klasik: eleman eleman topla
public static int SumScalar(ReadOnlySpan<int> data)
{
    int sum = 0;
    foreach (int x in data) sum += x;
    return sum;
}

// SIMD: aynı anda Vector<int>.Count eleman topla
public static int SumSimd(ReadOnlySpan<int> data)
{
    var acc = Vector<int>.Zero;
    int i = 0;
    int width = Vector<int>.Count;
    for (; i <= data.Length - width; i += width)
        acc += new Vector<int>(data.Slice(i, width));

    int sum = Vector.Dot(acc, Vector<int>.One);   // vektör içini topla
    for (; i < data.Length; i++) sum += data[i];  // kalan elemanlar
    return sum;
}
```

> SIMD güçlüdür ama kodu karmaşıklaştırır. Yalnızca büyük veri üzerinde, **ölçülmüş** sıcak yolda değer. Çoğu zaman `Span` üzerindeki yerleşik metotlar (`IndexOf`, `SequenceEqual`) zaten içeride SIMD kullanır — önce onları dene.

## Gizli Allocation Kaynakları

Sıcak yolda farkında olmadan çöp üreten yaygın noktalar:

| Kaynak | Sorun | Çözüm |
|--------|-------|-------|
| **Closure / lambda yakalama** | Yakalanan değişken için heap nesnesi | `static` lambda, yakalamadan kaçın |
| **Boxing** | Değer tipi → object/interface | Generic, `Span` |
| **`params object[]`** | Her çağrıda dizi | Overload veya `params ReadOnlySpan` (.NET 9) |
| **LINQ zinciri** | Her operatör enumerator + delegate allocation | Sıcak yolda elle `for` döngüsü |
| **`IEnumerator` (interface)** | Struct enumerator yerine boxing | `foreach`'i somut tip üzerinde yap |
| **String interpolation** | Karmaşık formatta ara string | `string.Create`, `IFormattable` |
| **`async` state machine** | Her `await` için (gerekirse) heap | Sıcak yolda `ValueTask`, senkron tamamlanan yol |

```csharp
// ❌ Lambda her çağrıda 'threshold'u yakalar → closure allocation
items.Where(x => x.Value > threshold);

// ✅ Sıcak yolda elle döngü — allocation yok
foreach (var x in items) if (x.Value > threshold) yield return x;
```

> **Not:** LINQ ve string interpolation sıradan kodda gayet iyidir. Bunlardan kaçınmak **yalnızca ölçülmüş, çok sık çalışan sıcak yollar** için geçerlidir. Okunabilirlik genelde önceliklidir.

## Mülakat Soruları

### 1. Soru: "`+` ile string birleştirme neden yavaştır, alternatifi nedir?"

**Cevap:** String immutable olduğu için her `+` yeni bir string (heap allocation + karakter kopyası) üretir; döngüde bu O(n²) kopya ve çok sayıda çöp demektir. Alternatifler: uzunluk biliniyorsa `string.Create` (sıfır ara allocation), bilinmiyorsa `StringBuilder` (tercihen kapasiteyle veya havuzdan), ayrıştırmada `ReadOnlySpan<char>` slicing, sabit birleştirmede `string.Join`/interpolation.

### 2. Soru: "UTF-8 ile doğrudan çalışmak neden allocation azaltır?"

**Cevap:** `string` UTF-16'dır; ağdan/diskten gelen veri ise genelde UTF-8 byte'larıdır. Geleneksel yol byte → string (UTF-16, allocation) → işlem yapar. `u8` literal'ler, `IUtf8SpanFormattable` ve `Utf8JsonReader` gibi API'ler byte'lar üzerinde **doğrudan** çalışır; ara `string` hiç oluşmaz. `System.Text.Json`'ın yüksek performansının temeli budur.

### 3. Soru: "Closure allocation nedir, nasıl önlersin?"

**Cevap:** Bir lambda, kendi dışındaki bir değişkeni **yakaladığında** (capture), derleyici bu değişkeni tutmak için bir heap nesnesi (display class) oluşturur. Sıcak yolda her çağrıda allocation üretir. Önlemek için: değişken yakalamayan `static` lambda kullanmak, gerekli veriyi parametre/state olarak geçirmek (örn. `string.Create`'in state parametresi), veya sıcak yolda lambda yerine elle döngü.

### 4. Soru: "SIMD ne zaman kullanılır, riski nedir?"

**Cevap:** Büyük diziler üzerinde aynı işlemi tekrarlayan (toplama, karşılaştırma, filtreleme) sayısal/veri-yoğun sıcak yollarda. Riski: kod ciddi karmaşıklaşır, bakımı zorlaşır ve yanlış yazılırsa (kalan elemanlar, hizalama) hata üretir. Üstelik `Span`'in `IndexOf`/`SequenceEqual` gibi yerleşik metotları zaten içeride SIMD kullanır; çoğu durumda elle SIMD yazmaya gerek kalmadan onlardan faydalanılır. Her zaman benchmark ile doğrulanmalıdır.

## Best Practices

- Sabit uzunluklu string üretiminde `string.Create`; değişkende kapasiteli `StringBuilder`.
- Ayrıştırmada `Split` yerine `ReadOnlySpan<char>` slicing / `Span.Split`.
- UTF-8 verisinde `u8` literal ve UTF-8 API'leriyle doğrudan çalış.
- Tekrarlı çoklu-karakter aramada `SearchValues<T>`.
- SIMD'den önce `Span`'in yerleşik (zaten SIMD'li) metotlarını dene.
- Sıcak yolda closure/boxing/LINQ allocation'larını `[MemoryDiagnoser]` ile avla.
- **Bu tekniklerin hiçbirini sıradan koda uygulama** — yalnızca ölçülmüş, çok sık çalışan sıcak yollar için.

## İlişkili Konular

- [Span ve Memory](span-and-memory.md)
- [Bellek ve Pooling](memory-and-pooling.md)
- [Benchmarking](benchmarking.md)
- [String İşlemleri](../../junior/csharp-basics/string-operations.md)
- [Boxing ve Unboxing](../../junior/csharp-basics/boxing-unboxing.md)

## Kaynaklar

- Stephen Toub — "Performance Improvements in .NET 8/9" (UTF-8, SearchValues)
- Microsoft Docs — SearchValues, string.Create, System.Numerics.Vector
- "Pro .NET Memory Management" — Konrad Kokosa
