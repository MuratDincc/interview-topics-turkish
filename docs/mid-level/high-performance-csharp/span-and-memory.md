# Span&lt;T&gt; ve Memory&lt;T&gt;

## Genel Bakış

`Span<T>`, bitişik bir bellek bölgesine (managed array, stack, hatta unmanaged bellek) **kopya yapmadan, allocation üretmeden** erişim sağlayan bir `ref struct`'tır. .NET'in yüksek performanslı parser, serializer ve I/O altyapısının temel yapı taşıdır.

Temel vaadi: Bir dizinin/dizginin bir **parçasıyla** çalışmak için yeni dizi/substring oluşturmak (= heap allocation + kopya) yerine, var olan belleğe bir **pencere** açmak.

```csharp
int[] numbers = { 1, 2, 3, 4, 5 };

// ❌ Allocation + kopya: yeni dizi oluşur
int[] slowSlice = numbers[1..4];

// ✅ Allocation yok: aynı belleğe pencere
Span<int> fastSlice = numbers.AsSpan(1, 3);   // {2, 3, 4}
```

## Span&lt;T&gt; vs ReadOnlySpan&lt;T&gt; vs Memory&lt;T&gt;

| Tip | Yazılabilir mi? | Nerede tutulabilir? | Kullanım |
|-----|-----------------|---------------------|----------|
| `Span<T>` | Evet | Sadece **stack** (yerel değişken, parametre) | Senkron, kısa ömürlü işlemler |
| `ReadOnlySpan<T>` | Hayır | Sadece **stack** | Salt okuma; `string` → `ReadOnlySpan<char>` |
| `Memory<T>` | Evet | **Heap** (field, async state machine) | Async metotlar, sınıf alanı |

### Neden Span heap'te tutulamaz?

`Span<T>` bir **`ref struct`**'tır. Bu, **yalnızca stack'te** yaşayabileceği anlamına gelir. Sebep: Span, stack'teki veriye veya `stackalloc` belleğine işaret edebilir; eğer bir heap nesnesinin alanı (field) olarak saklanabilseydi, işaret ettiği stack belleği metot döndükten sonra geçersiz olurdu (dangling pointer). Bu yüzden derleyici şunları **yasaklar**:

```csharp
class Holder
{
    // ❌ Derleme hatası: ref struct alan olamaz
    private Span<int> _span;
}

// ❌ Span async metotta lokal olamaz (await sınırını geçemez)
async Task Process(Span<byte> data) { }   // hata
```

Async senaryolarda veya alan olarak saklamak gerektiğinde **`Memory<T>`** kullanılır; işleneceği anda `.Span` ile `Span<T>`'e dönüştürülür:

```csharp
async Task ProcessAsync(Memory<byte> buffer)
{
    await SomethingAsync();
    Span<byte> span = buffer.Span;   // await sonrası senkron bölümde Span'e geç
    span[0] = 1;
}
```

## stackalloc: Heap'e Dokunmadan Tampon

Küçük, kısa ömürlü tamponlar için heap yerine **stack**'te yer ayırmak allocation'ı tamamen ortadan kaldırır:

```csharp
// Heap allocation YOK — stack üzerinde 128 byte
Span<byte> buffer = stackalloc byte[128];

// Sayıyı tampona yaz, string allocation üretmeden
int value = 42;
value.TryFormat(buffer, out int written);
```

> **Dikkat:** `stackalloc` boyutu **küçük ve sabit** olmalı (genelde ≤ 1 KB). Büyük veya kullanıcı girdisine bağlı boyutlar **stack overflow** riski yaratır. Değişken boyutlarda `ArrayPool<T>` ile birleştirilir:

```csharp
const int StackLimit = 256;
byte[]? rented = length > StackLimit ? ArrayPool<byte>.Shared.Rent(length) : null;
Span<byte> buffer = rented ?? stackalloc byte[length];
try { /* buffer'ı kullan */ }
finally { if (rented is not null) ArrayPool<byte>.Shared.Return(rented); }
```

## String İşlemlerinde Span

`string`, `ReadOnlySpan<char>`'a **kopyasız** dönüşür. `Substring` her çağrıda yeni string (allocation) üretirken, slicing üretmez:

```csharp
string csv = "Murat;Dinc;1990";

// ❌ Üç ayrı string allocation
string[] parts = csv.Split(';');

// ✅ Allocation'sız ayrıştırma
ReadOnlySpan<char> span = csv;
int sep = span.IndexOf(';');
ReadOnlySpan<char> firstName = span[..sep];        // "Murat" — yeni string yok
ReadOnlySpan<char> rest = span[(sep + 1)..];

// Sayıya çevirmek de allocation'sız:
int year = int.Parse(span[(span.LastIndexOf(';') + 1)..]);
```

## Slicing: Kopyasız Alt-Dizi

`Slice` / range operatörü (`..`), aynı belleğe yeni bir pencere döndürür — veri kopyalanmaz, sadece offset ve uzunluk değişir:

```csharp
Span<int> data = stackalloc int[] { 10, 20, 30, 40, 50 };
Span<int> middle = data.Slice(1, 3);   // {20, 30, 40}
middle[0] = 99;                         // data[1] de artık 99 — AYNI bellek!
// data: {10, 99, 30, 40, 50}
```

Bu "aynı bellek" davranışı hem gücün hem de tehlikenin kaynağıdır: pencere üzerinden yazma, asıl veriyi değiştirir.

## Mülakat Soruları

### 1. Soru: "Span&lt;T&gt; neden heap'te (sınıf alanı olarak) tutulamaz?"

**Cevap:** `Span<T>` bir `ref struct`'tır ve yalnızca stack'te yaşayabilir. İçinde stack belleğine veya `stackalloc`'a işaret eden bir referans tutabilir; eğer bir heap nesnesinin alanı olarak saklanabilseydi, ilgili stack frame metot döndükten sonra yok olur ve Span geçersiz belleğe işaret ederdi (dangling reference). Bu güvenlik garantisini korumak için derleyici ref struct'ların field olmasını, async/iterator metotlarda await sınırını geçmesini ve boxing'ini yasaklar. Heap'te taşımak gerekiyorsa `Memory<T>` kullanılır.

### 2. Soru: "Substring ile Span slicing arasındaki fark nedir?"

**Cevap:** `Substring` her çağrıda **yeni bir `string` nesnesi** oluşturur (heap allocation + karakter kopyası). `ReadOnlySpan<char>` slicing ise yeni nesne oluşturmaz; aynı string'in belleğine offset/uzunluk ile bir pencere açar. Sıcak yollarda çok sayıda substring üretmek ciddi GC baskısı yaratır; span slicing bunu sıfırlar. Tek dezavantaj: span'i saklamak/async geçirmek mümkün değildir.

### 3. Soru: "stackalloc ne zaman tehlikelidir?"

**Cevap:** Boyut büyük veya **kullanıcı girdisine bağlı** olduğunda. Stack sınırlıdır (genelde ~1 MB); döngü içinde veya büyük/değişken boyutla `stackalloc` yapmak **stack overflow** ile uygulamayı çökertir (yakalanamayan bir hata). Bu yüzden sabit ve küçük (≤ ~256–1024 byte) boyutlarda kullanılır; değişken boyutta `ArrayPool` ile eşiklenir.

### 4. Soru: "Memory&lt;T&gt; ne zaman Span&lt;T&gt; yerine gerekir?"

**Cevap:** Bellek penceresini bir **sınıf alanında saklamak**, bir **async metotta `await` sınırını geçirmek** veya bir koleksiyonda tutmak gerektiğinde. Span ref struct olduğu için bunların hiçbirini yapamaz. `Memory<T>` heap'te yaşayabilir; gerçek işlem anında (senkron bölümde) `.Span` ile Span'e dönüştürülüp kullanılır.

## Best Practices

- API tasarlarken girdi için `ReadOnlySpan<T>` / çıktı için `Span<T>` parametreleri tercih et — çağıran array, stackalloc veya string geçirebilir.
- Salt okunan veride **`ReadOnlySpan<T>`** kullan; niyeti netleştirir ve yanlışlıkla yazmayı engeller.
- `stackalloc`'u **küçük ve sabit** tut; değişken boyutta `ArrayPool` ile eşikle.
- Async/saklama gereken yerde `Memory<T>`, işlem anında `.Span`.
- Slicing'in **aynı belleği paylaştığını** unutma — pencereye yazmak kaynağı değiştirir.
- Bu teknikleri yalnızca **ölçülmüş sıcak yollarda** uygula; sıradan kodda okunabilirlik önceliklidir.

## İlişkili Konular

- [Bellek ve Pooling](memory-and-pooling.md)
- [Allocation-Free Desenler](allocation-free-patterns.md)
- [Memory Management](../../junior/csharp-basics/memory-management.md)
- [Value Types vs Reference Types](../../junior/csharp-basics/value-reference-types.md)

## Kaynaklar

- Microsoft Docs — Memory&lt;T&gt; and Span&lt;T&gt; usage guidelines
- Stephen Toub — "Span&lt;T&gt;: Performance Improvements in .NET"
- "Pro .NET Memory Management" — Konrad Kokosa
