# Records ve Pattern Matching

## Genel Bakış

C# 9 ile birlikte gelen `record` türleri ve sürekli gelişen pattern matching özellikleri, modern .NET geliştirmesinin temel taşlarından biri haline gelmiştir. Records, değer eşitliği semantiğiyle birlikte değişmez (immutable) veri yapıları oluşturmayı kolaylaştırır. Pattern matching ise karmaşık koşul mantığını daha okunabilir ve güvenli bir şekilde ifade etmeyi sağlar.

Bu bölümde aşağıdaki konular ele alınmaktadır:

- `record`, `record class` ve `record struct` arasındaki farklar
- `init` erişimcisi ile yalnızca başlangıçta atanabilir özellikler
- Switch ifadeleri (switch expressions) ve switch deyimleri (switch statements)
- Property, positional, relational, logical ve list pattern'leri
- Tip pattern'leri ve bildirim pattern'leri
- C# 12 birincil yapıcılar (primary constructors)
- Gerçek dünya kullanım senaryoları

---

## Mülakat Soruları ve Cevapları

---

### 1. Soru

**`record` türü nedir ve normal bir `class`'tan farkı nedir? `record class` ve `record struct` arasındaki fark nedir?**

**Cevap:**

`record`, C# 9 ile tanıtılan ve değer eşitliği semantiğiyle çalışan özel bir referans tipidir. Normal `class`'tan farklı olarak, iki `record` nesnesinin eşitliği referanslarına değil, içerdikleri verilere göre belirlenir. Aynı zamanda records varsayılan olarak değişmezdir (immutable) ve `with` ifadesiyle kopyalama desteği sunar.

| Özellik                 | `class`        | `record` / `record class` | `record struct`   |
|-------------------------|----------------|---------------------------|-------------------|
| Bellek konumu           | Heap           | Heap                      | Stack             |
| Eşitlik semantiği       | Referans       | Değer                     | Değer             |
| Varsayılan değişmezlik  | Hayır          | Evet (positional)         | Evet (positional) |
| `with` ifadesi          | Hayır          | Evet                      | Evet              |
| Kalıtım                 | Evet           | Evet                      | Hayır             |

**Örnek Kod:**

```csharp
// Basit record tanımı (record class ile eşdeğer)
public record Kisi(string Ad, string Soyad, int Yas);

// Açık record class tanımı
public record class Urun(string Adi, decimal Fiyat, int Stok);

// record struct - yığın üzerinde yaşar, kalıtım desteklemez
public record struct Koordinat(double X, double Y);

// Eşitlik karşılaştırması
var kisi1 = new Kisi("Ahmet", "Yılmaz", 30);
var kisi2 = new Kisi("Ahmet", "Yılmaz", 30);
var kisi3 = new Kisi("Mehmet", "Kaya", 25);

Console.WriteLine(kisi1 == kisi2);          // True (değer eşitliği)
Console.WriteLine(ReferenceEquals(kisi1, kisi2)); // False (farklı nesneler)
Console.WriteLine(kisi1 == kisi3);          // False

// with ifadesi ile kopyalama (immutability korunur)
var kisi4 = kisi1 with { Yas = 31 };
Console.WriteLine(kisi4); // Kisi { Ad = Ahmet, Soyad = Yılmaz, Yas = 31 }
Console.WriteLine(kisi1); // Kisi { Ad = Ahmet, Soyad = Yılmaz, Yas = 30 } (değişmedi)

// record struct kullanımı
var nokta1 = new Koordinat(3.0, 4.0);
var nokta2 = nokta1 with { Y = 5.0 };
Console.WriteLine(nokta2); // Koordinat { X = 3, Y = 5 }

// Positional record ile deconstruction
var (ad, soyad, yas) = kisi1;
Console.WriteLine($"Ad: {ad}, Soyad: {soyad}, Yaş: {yas}");

// Kalıtım (yalnızca record class destekler)
public record Calisan(string Ad, string Soyad, int Yas, string Departman)
    : Kisi(Ad, Soyad, Yas);

var calisan = new Calisan("Ayşe", "Demir", 28, "Yazılım");
Console.WriteLine(calisan);
// Calisan { Ad = Ayşe, Soyad = Demir, Yas = 28, Departman = Yazılım }
```

---

### 2. Soru

**`init` erişimcisi nedir ve `set`'ten farkı nedir? Neden kullanılır?**

**Cevap:**

`init` erişimcisi C# 9 ile eklendi ve bir özelliğin yalnızca nesne başlatma (object initialization) sırasında atanabilmesini sağlar. `set`'ten farklı olarak, nesne oluşturulduktan sonra `init` özelliğine atama yapılamaz. Bu, değişmez (immutable) nesneler oluşturmak için kullanılır.

`init` kullanımının avantajları:
- Nesne başlatma sözdizimini (`{ }`) destekler, bu da okunabilirliği artırır.
- Sınıf değişmezliğini garanti eder.
- Zorunlu alanlar için `required` anahtar kelimesiyle birlikte kullanılabilir (C# 11).

**Örnek Kod:**

```csharp
// init erişimcisi kullanımı
public class SiparisKalemi
{
    public required string UrunAdi { get; init; }
    public required int Miktar { get; init; }
    public required decimal BirimFiyat { get; init; }

    // Hesaplanan özellik - sadece okunabilir
    public decimal ToplamFiyat => Miktar * BirimFiyat;
}

// Nesne başlatma sözdizimi ile kullanım (geçerli)
var kalem = new SiparisKalemi
{
    UrunAdi = "Laptop",
    Miktar = 2,
    BirimFiyat = 15000m
};

Console.WriteLine(kalem.ToplamFiyat); // 30000

// Sonradan atama denemesi (derleme hatası!)
// kalem.UrunAdi = "Tablet"; // CS8852: init-only özelliğe atama yapılamaz

// required ile birlikte kullanım
// Aşağıdaki kod derleme hatası verir (required özellikler atlanmış)
// var eksikKalem = new SiparisKalemi { UrunAdi = "Mouse" }; // CS9035

// Records zaten init kullanır, ancak normal class'larda da faydalıdır
public class ImmutableAdres
{
    public string Sokak { get; init; } = string.Empty;
    public string Sehir { get; init; } = string.Empty;
    public string PostaKodu { get; init; } = string.Empty;
    public string Ulke { get; init; } = "Türkiye";
}

var adres = new ImmutableAdres
{
    Sokak = "Atatürk Caddesi No:1",
    Sehir = "İstanbul",
    PostaKodu = "34000"
};

// with ifadesi records gibi kullanılamaz ama değiştirilemez güvencesi vardır
Console.WriteLine($"{adres.Sehir}, {adres.Ulke}");
```

---

### 3. Soru

**Switch expression nedir? Switch statement'tan ne gibi avantajları vardır? Temel pattern matching örnekleri gösteriniz.**

**Cevap:**

Switch expression, C# 8 ile gelen ve `switch` deyiminin daha kısa ve işlevsel bir versiyonudur. Her `case` için `return` veya `break` yazmak yerine, doğrudan bir değer döndürür. Kapsamlı kontrol (exhaustiveness check) yaparak tüm durumların ele alınmasını derleyici düzeyinde garanti eder.

Avantajları:
- Daha kısa ve okunabilir sözdizimi
- Her dal bir değer döndürmek zorundadır (ifade semantiği)
- Derleyici, ele alınmamış durumları uyarı olarak gösterir
- `when` korumaları (guards) ile birleştirilebilir

**Örnek Kod:**

```csharp
// Switch statement (eski yöntem)
public string SipariseDurumAciklamasiEski(SiparisDurumu durum)
{
    switch (durum)
    {
        case SiparisDurumu.Beklemede:
            return "Sipariş işleme alınmayı bekliyor";
        case SiparisDurumu.Onaylandi:
            return "Sipariş onaylandı, hazırlanıyor";
        case SiparisDurumu.Kargolandi:
            return "Sipariş kargoya verildi";
        case SiparisDurumu.Teslim:
            return "Sipariş teslim edildi";
        case SiparisDurumu.Iptal:
            return "Sipariş iptal edildi";
        default:
            return "Bilinmeyen durum";
    }
}

// Switch expression (modern yöntem)
public string SiparisDurumAciklamasi(SiparisDurumu durum) =>
    durum switch
    {
        SiparisDurumu.Beklemede => "Sipariş işleme alınmayı bekliyor",
        SiparisDurumu.Onaylandi => "Sipariş onaylandı, hazırlanıyor",
        SiparisDurumu.Kargolandi => "Sipariş kargoya verildi",
        SiparisDurumu.Teslim     => "Sipariş teslim edildi",
        SiparisDurumu.Iptal      => "Sipariş iptal edildi",
        _                        => throw new ArgumentOutOfRangeException(nameof(durum))
    };

public enum SiparisDurumu
{
    Beklemede, Onaylandi, Kargolandi, Teslim, Iptal
}

// when koruması (guard) ile birlikte kullanım
public string IndirimiHesapla(decimal fiyat, int adet) =>
    (fiyat, adet) switch
    {
        (>= 1000, >= 10) => "Yüzde yirmi indirim",
        (>= 1000, _)     => "Yüzde on indirim",
        (_, >= 10)       => "Yüzde beş indirim",
        _                => "İndirim yok"
    };

// Tip kontrolü ile birlikte
public double AlanHesapla(object sekil) =>
    sekil switch
    {
        Daire d              => Math.PI * d.Yaricap * d.Yaricap,
        Dikdortgen r         => r.Genislik * r.Yukseklik,
        Ucgen t              => t.Taban * t.Yukseklik / 2,
        null                 => throw new ArgumentNullException(nameof(sekil)),
        _                    => throw new ArgumentException("Bilinmeyen şekil")
    };

public record Daire(double Yaricap);
public record Dikdortgen(double Genislik, double Yukseklik);
public record Ucgen(double Taban, double Yukseklik);

// Kullanım örnekleri
Console.WriteLine(SiparisDurumAciklamasi(SiparisDurumu.Kargolandi));
// Çıktı: Sipariş kargoya verildi

Console.WriteLine(IndirimiHesapla(1500, 15));
// Çıktı: Yüzde yirmi indirim

Console.WriteLine(AlanHesapla(new Daire(5)));
// Çıktı: 78.53981633974483
```

---

### 4. Soru

**Property pattern ve positional pattern nedir? Aralarındaki farkı kod örnekleriyle açıklayınız.**

**Cevap:**

**Property pattern**, bir nesnenin özelliklerine (properties) göre eşleşme yapar. `{ OzellikAdi: deger }` sözdizimini kullanır. Herhangi bir sınıf, struct veya record ile çalışır.

**Positional pattern**, `Deconstruct` metodunu veya positional record'ların bileşenlerini kullanarak eşleşme yapar. Nesneyi bileşenlerine ayrıştırır ve her bileşen için ayrı pattern uygulanır.

**Örnek Kod:**

```csharp
public record Adres(string Sehir, string Ulke, string PostaKodu);

public record Musteri(
    string Ad,
    string Soyad,
    int Yas,
    Adres Adres,
    bool PremiumUye);

// Property pattern kullanımı
public string MusteriKategorisi(Musteri musteri) =>
    musteri switch
    {
        // İç içe (nested) property pattern
        { PremiumUye: true, Yas: >= 60 }
            => "Kıdemli Premium Üye",

        { PremiumUye: true, Adres: { Ulke: "Türkiye" } }
            => "Yurt İçi Premium Üye",

        { PremiumUye: true }
            => "Premium Üye",

        { Yas: < 18 }
            => "Genç Üye",

        { Adres.Sehir: "İstanbul" }   // C# 10 - nokta sözdizimi
            => "İstanbul Üyesi",

        _
            => "Standart Üye"
    };

// Positional pattern kullanımı
public record Nokta(int X, int Y);

public string KoordinatAciklamasi(Nokta nokta) =>
    nokta switch
    {
        (0, 0)     => "Orijin noktası",
        (0, _)     => "Y ekseninde",
        (_, 0)     => "X ekseninde",
        (> 0, > 0) => "Birinci bölge",
        (< 0, > 0) => "İkinci bölge",
        (< 0, < 0) => "Üçüncü bölge",
        (> 0, < 0) => "Dördüncü bölge",
        _          => "Belirsiz konum"
    };

// Tuple pattern (positional pattern ile aynı mekanizma)
public decimal KDVOrani(string ulke, string urunTipi) =>
    (ulke, urunTipi) switch
    {
        ("Türkiye", "Gıda")        => 0.01m,
        ("Türkiye", "Elektronik")  => 0.20m,
        ("Türkiye", _)             => 0.18m,
        ("AB", _)                  => 0.21m,
        _                          => 0.00m
    };

// Positional pattern ile deconstruction (record olmayan sınıf)
public class Dikdortgen2
{
    public double Genislik { get; }
    public double Yukseklik { get; }

    public Dikdortgen2(double genislik, double yukseklik)
        => (Genislik, Yukseklik) = (genislik, yukseklik);

    // Deconstruct metodu tanımlanmış olması gerekir
    public void Deconstruct(out double genislik, out double yukseklik)
        => (genislik, yukseklik) = (Genislik, Yukseklik);
}

public string SekilSinifi(Dikdortgen2 d) =>
    d switch
    {
        (var g, var y) when g == y => "Kare",
        (var g, var y) when g > y  => "Yatay dikdörtgen",
        _                          => "Dikey dikdörtgen"
    };

// Kullanım örnekleri
var musteri = new Musteri("Ali", "Veli", 65, new Adres("Ankara", "Türkiye", "06000"), true);
Console.WriteLine(MusteriKategorisi(musteri)); // Kıdemli Premium Üye

var nokta = new Nokta(3, -2);
Console.WriteLine(KoordinatAciklamasi(nokta)); // Dördüncü bölge
```

---

### 5. Soru

**Relational pattern ve logical pattern nedir? `and`, `or`, `not` kullanımını örneklerle açıklayınız.**

**Cevap:**

C# 9 ile gelen relational ve logical pattern'ler, sayısal karşılaştırmalar ve mantıksal kombinasyonlar için kullanılır.

- **Relational pattern**: `<`, `>`, `<=`, `>=` operatörleriyle karşılaştırma yapar.
- **Logical pattern**: `and`, `or`, `not` anahtar kelimeleriyle birden fazla pattern'i birleştirir.

Bu pattern'ler yalnızca switch ifadelerinde değil, `is` ifadesiyle de kullanılabilir.

**Örnek Kod:**

```csharp
// Relational pattern - is ifadesiyle
int sicaklik = 25;
bool rahat = sicaklik is >= 18 and <= 28;
Console.WriteLine(rahat); // True

// Logical patterns
public string HavaKosuluAciklamasi(int derece) =>
    derece switch
    {
        < -10          => "Dondurucu soğuk",
        >= -10 and < 0 => "Çok soğuk",
        >= 0 and < 10  => "Soğuk",
        >= 10 and < 20 => "Serin",
        >= 20 and < 30 => "Ilık",
        >= 30 and < 40 => "Sıcak",
        >= 40          => "Kavurucu sıcak",
        _              => "Bilinmiyor"
    };

// not pattern
public bool GecerliYas(int yas) => yas is not (< 0 or > 150);

// or pattern - birden fazla değer eşleşmesi
public bool HaftatatimiMi(DayOfWeek gun) =>
    gun is DayOfWeek.Saturday or DayOfWeek.Sunday;

// Karmaşık kombinasyonlar
public record UrunBilgisi(string Kategori, decimal Fiyat, int StokAdedi, bool AktifMi);

public string UrunOnceligi(UrunBilgisi urun) =>
    urun switch
    {
        // Aktif değil ve stok bitmiş
        { AktifMi: false, StokAdedi: 0 }
            => "Pasif - Stok yok",

        // Aktif, stok kritik ve pahalı ürün
        { AktifMi: true, StokAdedi: > 0 and <= 5, Fiyat: > 1000 }
            => "Acil sipariş gerekli - Yüksek değerli ürün",

        // Aktif, stok kritik
        { AktifMi: true, StokAdedi: > 0 and <= 10 }
            => "Stok kritik seviyede",

        // Kategori bazlı kontrol
        { Kategori: "Elektronik" or "Beyaz Eşya", AktifMi: true }
            => "Öncelikli kategori",

        { AktifMi: true }
            => "Normal",

        _
            => "Değerlendirme gerekli"
    };

// is ifadesiyle tip ve relational birleştirme
public void DegerKontrol(object deger)
{
    if (deger is int sayi and > 0 and < 100)
        Console.WriteLine($"{sayi} pozitif iki haneli sayı");
    else if (deger is string metin and { Length: > 0 })
        Console.WriteLine($"Dolu metin: {metin}");
    else if (deger is not null)
        Console.WriteLine($"Başka tip: {deger.GetType().Name}");
    else
        Console.WriteLine("Null değer");
}

// Kullanım örnekleri
Console.WriteLine(HavaKosuluAciklamasi(25));  // Ilık
Console.WriteLine(HavaKosuluAciklamasi(-5));  // Çok soğuk
Console.WriteLine(HaftatatimiMi(DayOfWeek.Saturday)); // True
Console.WriteLine(GecerliYas(-1));  // False
Console.WriteLine(GecerliYas(25));  // True

var urun = new UrunBilgisi("Elektronik", 2500m, 3, true);
Console.WriteLine(UrunOnceligi(urun)); // Acil sipariş gerekli - Yüksek değerli ürün
```

---

### 6. Soru

**C# 11 ile gelen list pattern nedir? `[first, .., last]` sözdizimini ve kullanım senaryolarını açıklayınız.**

**Cevap:**

List pattern, C# 11 ile eklenen ve diziler ile listeler üzerinde desen eşleşmesi yapmayı sağlayan bir özelliktir. `[eleman1, eleman2, ...]` sözdizimini kullanır. `..` (dilim operatörü) ise sıfır veya daha fazla elemanı temsil eder.

Temel bileşenler:
- `[a, b, c]` - Tam olarak üç elemanlı dizi
- `[a, ..]` - En az bir eleman, geri kalanı önemsiz
- `[.., z]` - Son elemanı z olan dizi
- `[a, .. var orta, z]` - İlk, son ve ortadaki elemanlar
- `[]` - Boş dizi

**Örnek Kod:**

```csharp
// Temel list pattern kullanımı
public string DiziAciklamasi(int[] sayilar) =>
    sayilar switch
    {
        []              => "Boş dizi",
        [var tek]       => $"Tek elemanlı: {tek}",
        [var ilk, var son] => $"İki elemanlı: {ilk}, {son}",
        [1, 2, 3]       => "Tam olarak 1, 2, 3",
        [1, ..]         => "1 ile başlıyor",
        [.., 0]         => "0 ile bitiyor",
        [var bas, .. , var son2] => $"İlk: {bas}, Son: {son2}, {sayilar.Length} eleman"
    };

// Gerçek kullanım senaryosu: Komut satırı argüman işleme
public string KomutIsle(string[] args) =>
    args switch
    {
        []                         => "Yardım bilgisi göster",
        ["--help"] or ["-h"]       => "Detaylı yardım",
        ["--version"] or ["-v"]    => "Versiyon: 1.0.0",
        ["create", var ad]         => $"'{ad}' oluşturuluyor",
        ["create", var ad, "--force"] => $"'{ad}' zorla oluşturuluyor",
        ["delete", var ad]         => $"'{ad}' siliniyor",
        ["list", ..]               => "Tümü listeleniyor",
        [var bilinmez, ..]         => $"Bilinmeyen komut: {bilinmez}"
    };

// Karmaşık liste eşleştirmesi
public bool FibonacciMu(IList<int> seri) =>
    seri switch
    {
        [.. var bas, var a, var b, var c] when c == a + b
            => FibonacciMu(bas.Append(a).Append(b).ToList()),
        [_, _] => true,  // En az iki eleman varsa geçerli başlangıç
        [_]    => true,  // Tek eleman
        []     => false, // Boş dizi Fibonacci değil
        _      => false
    };

// İç içe list pattern
public string MatrisAciklamasi(int[][] matris) =>
    matris switch
    {
        []                 => "Boş matris",
        [[var tek]]        => $"1x1 matris: {tek}",
        [[var a, var b], [var c, var d]]
            => $"2x2 matris: [[{a},{b}],[{c},{d}]]",
        [var satir1, ..]   => $"{matris.Length}x{satir1.Length} matris"
    };

// Koleksiyon başlığı analizi
public record HttpIstek(string Metod, string[] YolSegmentleri, Dictionary<string, string> Basliklar);

public string YolIsle(string[] segmentler) =>
    segmentler switch
    {
        ["api", "v1", var kaynak]
            => $"API v1 - Kaynak: {kaynak}",
        ["api", "v1", var kaynak, var id]
            => $"API v1 - Kaynak: {kaynak}, ID: {id}",
        ["api", "v2", ..]
            => "API v2 endpoint",
        ["admin", ..]
            => "Admin paneli",
        [var ilk, ..]
            => $"Bilinmeyen rota: {ilk}",
        []
            => "Kök dizin"
    };

// Kullanım örnekleri
Console.WriteLine(DiziAciklamasi(new[] { 1, 5, 9, 3 }));
// Çıktı: İlk: 1, Son: 3, 4 eleman

Console.WriteLine(KomutIsle(new[] { "create", "proje-adi" }));
// Çıktı: 'proje-adi' oluşturuluyor

Console.WriteLine(YolIsle(new[] { "api", "v1", "urunler", "42" }));
// Çıktı: API v1 - Kaynak: urunler, ID: 42
```

---

### 7. Soru

**Type pattern ve declaration pattern nedir? `is` ifadesiyle nasıl kullanılır?**

**Cevap:**

**Type pattern**, bir nesnenin belirli bir türde olup olmadığını kontrol eder. `x is Tip` sözdizimini kullanır.

**Declaration pattern**, tip kontrolü yaparken aynı zamanda sonucu bir değişkene atar. `x is Tip degisken` sözdizimini kullanır. Bu, C# 7 ile gelen `is` desen eşleşmesinin temelidir.

**Örnek Kod:**

```csharp
// Temel declaration pattern
public void NesneBilgisi(object nesne)
{
    if (nesne is string metin)
    {
        // metin burada string türündedir
        Console.WriteLine($"Metin uzunluğu: {metin.Length}");
    }
    else if (nesne is int sayi)
    {
        Console.WriteLine($"Sayının karesi: {sayi * sayi}");
    }
    else if (nesne is DateTime tarih)
    {
        Console.WriteLine($"Yıl: {tarih.Year}");
    }
}

// Polimorfik işleme - switch expression ile
public abstract record Sekil;
public record Cember(double Yaricap) : Sekil;
public record Kare(double Kenar) : Sekil;
public record Dikdortgen3(double En, double Boy) : Sekil;
public record Ucgen2(double TabanUzunluk, double Yukseklik) : Sekil;

public double Alan(Sekil sekil) =>
    sekil switch
    {
        Cember c               => Math.PI * c.Yaricap * c.Yaricap,
        Kare k                 => k.Kenar * k.Kenar,
        Dikdortgen3 { En: var e, Boy: var b }
                               => e * b,
        Ucgen2 u               => u.TabanUzunluk * u.Yukseklik / 2.0,
        _                      => throw new ArgumentException("Bilinmeyen şekil")
    };

public double Cevre(Sekil sekil) =>
    sekil switch
    {
        Cember c     => 2 * Math.PI * c.Yaricap,
        Kare k       => 4 * k.Kenar,
        Dikdortgen3 d => 2 * (d.En + d.Boy),
        Ucgen2       => throw new NotSupportedException("Üçgen çevresi için ek veri gerekli"),
        _            => throw new ArgumentException("Bilinmeyen şekil")
    };

// Null kontrolü ile birlikte
public string MusteriAdi(object? nesne) =>
    nesne switch
    {
        null                 => "(boş)",
        string s             => s,
        Musteri2 { Ad: var ad, Soyad: var soyad }
                             => $"{ad} {soyad}",
        IEnumerable<string> liste
                             => string.Join(", ", liste),
        _                    => nesne.ToString() ?? "(bilinmiyor)"
    };

public record Musteri2(string Ad, string Soyad);

// var pattern - her zaman eşleşir, tipi çıkarsamasız kullanılır
public void VarPatternOrnegi(object? deger)
{
    // var pattern null dahil her değerle eşleşir
    if (deger is var x)
    {
        Console.WriteLine($"Değer: {x ?? "(null)"}");
    }

    // switch içinde var
    var aciklama = deger switch
    {
        null      => "null",
        var obj   => $"Tip: {obj.GetType().Name}"
    };
}

// İç içe tip pattern
public string AnalizEt(IEnumerable<object> koleksiyon)
{
    var sonuc = new List<string>();
    foreach (var eleman in koleksiyon)
    {
        var bilgi = eleman switch
        {
            int i when i < 0    => $"Negatif sayı: {i}",
            int i               => $"Pozitif sayı: {i}",
            string { Length: 0 } => "Boş metin",
            string s            => $"Metin: {s}",
            bool b              => $"Boolean: {b}",
            null                => "Null",
            _                   => $"Diğer: {eleman.GetType().Name}"
        };
        sonuc.Add(bilgi);
    }
    return string.Join(" | ", sonuc);
}

// Kullanım örnekleri
var sekiller = new Sekil[] { new Cember(5), new Kare(4), new Dikdortgen3(3, 7) };
foreach (var s in sekiller)
    Console.WriteLine($"{s.GetType().Name}: Alan={Alan(s):F2}, Çevre={Cevre(s):F2}");

Console.WriteLine(AnalizEt(new object[] { 42, -3, "merhaba", "", true, null! }));
// Çıktı: Pozitif sayı: 42 | Negatif sayı: -3 | Metin: merhaba | Boş metin | Boolean: True | Null
```

---

### 8. Soru

**C# 12 primary constructor nedir? Records'tan farkı nedir ve ne zaman tercih edilmelidir?**

**Cevap:**

C# 12 ile birlikte normal `class` ve `struct` tanımlarına **primary constructor** (birincil yapıcı) desteği eklendi. Bu, yapıcı parametrelerini doğrudan sınıf tanımı satırına yazabilmeyi sağlar. Records'ta bu özellik zaten vardı ancak semantik olarak bazı önemli farklar mevcuttur.

**Records ile primary constructor farkları:**
- Records'ta primary constructor parametreleri otomatik olarak `init` özelliklerine dönüşür.
- Class'larda primary constructor parametreleri yalnızca parametredir; otomatik özellik oluşturulmaz.
- Records değer eşitliği sağlar; normal class'lar referans eşitliği kullanır.
- Records `with` ifadesini destekler; normal class'lar desteklemez.

**Örnek Kod:**

```csharp
// C# 12 - Normal class ile primary constructor
public class Logger(string kategori, bool ayrıntılı = false)
{
    // Parametreler doğrudan kullanılabilir
    private readonly string _prefiks = $"[{kategori}]";

    public void Bilgi(string mesaj) =>
        Console.WriteLine($"{_prefiks} INFO: {mesaj}");

    public void Hata(string mesaj) =>
        Console.WriteLine($"{_prefiks} ERROR: {mesaj}");

    public void Ayrintili(string mesaj)
    {
        if (ayrıntılı) // primary constructor parametresi
            Console.WriteLine($"{_prefiks} DEBUG: {mesaj}");
    }
}

// Bağımlılık enjeksiyonu ile primary constructor
public interface IRepository<T>
{
    Task<T?> GetirAsync(int id);
    Task<IEnumerable<T>> TumunuGetirAsync();
    Task EkleAsync(T entity);
}

public class UrunService(
    IRepository<Urun2> urunRepo,
    ILogger<UrunService> logger,
    IMemoryCache onbellek)
{
    public async Task<Urun2?> UrunGetirAsync(int id)
    {
        var anahtar = $"urun_{id}";

        if (onbellek.TryGetValue(anahtar, out Urun2? onbellekteUrun))
        {
            logger.LogInformation("Önbellekten alındı: {Id}", id);
            return onbellekteUrun;
        }

        var urun = await urunRepo.GetirAsync(id);
        if (urun is not null)
            onbellek.Set(anahtar, urun, TimeSpan.FromMinutes(5));

        return urun;
    }
}

public record Urun2(int Id, string Ad, decimal Fiyat);

// Record vs Primary Constructor karşılaştırması
public record KisiRecord(string Ad, string Soyad, int Yas);

public class KisiClass(string ad, string soyad, int yas)
{
    // Primary constructor parametrelerinden özellik oluşturmak için açıkça tanımlamak gerekir
    public string Ad { get; } = ad;
    public string Soyad { get; } = soyad;
    public int Yas { get; } = yas;

    // Ek iş mantığı eklenebilir
    public string TamAd => $"{Ad} {Soyad}";
    public bool YetiskinMi => Yas >= 18;
}

// Primary constructor validasyonu
public class SicaklikSensoru(string sensorId, double minDerece, double maxDerece)
{
    // Constructor'da validasyon
    private readonly string _sensorId = string.IsNullOrWhiteSpace(sensorId)
        ? throw new ArgumentException("Sensör ID boş olamaz")
        : sensorId;

    public SicaklikOkuma OkumaYap(double olcum) =>
        olcum switch
        {
            < -273.15               => throw new ArgumentException("Mutlak sıfırın altında değer"),
            var d when d < minDerece => new SicaklikOkuma(_sensorId, d, SicaklikDurum.DusukUyari),
            var d when d > maxDerece => new SicaklikOkuma(_sensorId, d, SicaklikDurum.YuksekUyari),
            var d                   => new SicaklikOkuma(_sensorId, d, SicaklikDurum.Normal)
        };
}

public record SicaklikOkuma(string SensorId, double Derece, SicaklikDurum Durum);
public enum SicaklikDurum { Normal, DusukUyari, YuksekUyari }

// Kullanım örneği
var logger = new Logger("Uygulama", ayrıntılı: true);
logger.Bilgi("Uygulama başlatıldı");
logger.Ayrintili("Detaylı bilgi mesajı");

var kisiRecord = new KisiRecord("Ali", "Veli", 30);
var kisiClass = new KisiClass("Ali", "Veli", 30);

// Record'ta değer eşitliği
var kisiRecord2 = new KisiRecord("Ali", "Veli", 30);
Console.WriteLine(kisiRecord == kisiRecord2); // True

// Class'ta referans eşitliği
var kisiClass2 = new KisiClass("Ali", "Veli", 30);
Console.WriteLine(kisiClass == kisiClass2); // False

// Record'ta with ifadesi
var guncellenmis = kisiRecord with { Yas = 31 };
Console.WriteLine(guncellenmis); // KisiRecord { Ad = Ali, Soyad = Veli, Yas = 31 }
```

---

### 9. Soru

**Records ve pattern matching'in gerçek dünya kullanım senaryolarını gösteriniz. DTO'lar, domain event'ler ve API yanıtları için nasıl kullanılır?**

**Cevap:**

Records ve pattern matching, özellikle DDD (Domain Driven Design), CQRS ve functional programming yaklaşımlarında güçlü kombinasyonlar oluşturur.

**Örnek Kod:**

```csharp
// ===== 1. DTO'lar için Records =====

// API istek/yanıt DTO'ları
public record UrunOlusturIstek(
    string Ad,
    string Aciklama,
    decimal Fiyat,
    int StokAdedi,
    string Kategori);

public record UrunYanit(
    int Id,
    string Ad,
    decimal Fiyat,
    bool StokVarMi,
    string Durum);

// Sayfalama için generic record
public record SayfaliYanit<T>(
    IReadOnlyList<T> Veriler,
    int ToplamAdet,
    int SayfaNo,
    int SayfaBoyutu)
{
    public int ToplamSayfa => (int)Math.Ceiling(ToplamAdet / (double)SayfaBoyutu);
    public bool SonrakiSayfaVarMi => SayfaNo < ToplamSayfa;
    public bool OncekiSayfaVarMi => SayfaNo > 1;
}

// ===== 2. Domain Events için Records =====

// Temel domain event
public abstract record DomainEvent(Guid EventId, DateTime OlusturulmaZamani)
{
    protected DomainEvent() : this(Guid.NewGuid(), DateTime.UtcNow) { }
}

public record UrunOlusturuldu(
    int UrunId,
    string UrunAdi,
    decimal Fiyat) : DomainEvent;

public record UrunFiyatiGuncellendi(
    int UrunId,
    decimal EskiFiyat,
    decimal YeniFiyat) : DomainEvent;

public record SiparisVerildi(
    int SiparisId,
    int MusteriId,
    IReadOnlyList<SiparisKalemi2> Kalemler,
    decimal ToplamTutar) : DomainEvent;

public record SiparisKalemi2(int UrunId, int Miktar, decimal BirimFiyat);

// Domain event işleme - pattern matching ile
public class DomainEventIsleyici
{
    public string EventAciklamasi(DomainEvent evt) =>
        evt switch
        {
            UrunOlusturuldu { UrunAdi: var ad, Fiyat: var fiyat }
                => $"Yeni ürün eklendi: {ad} ({fiyat:C})",

            UrunFiyatiGuncellendi { EskiFiyat: var eski, YeniFiyat: var yeni }
                when yeni > eski
                => $"Fiyat artışı: {eski:C} -> {yeni:C} (%{(yeni - eski) / eski * 100:F1})",

            UrunFiyatiGuncellendi { EskiFiyat: var eski, YeniFiyat: var yeni }
                => $"Fiyat indirimi: {eski:C} -> {yeni:C} (%{(eski - yeni) / eski * 100:F1})",

            SiparisVerildi { MusteriId: var mId, ToplamTutar: >= 1000 }
                => $"Büyük sipariş! Müşteri #{mId}, Tutar: {evt.ToplamTutar:C}",

            SiparisVerildi { ToplamTutar: var tutar }
                => $"Yeni sipariş, Tutar: {tutar:C}",

            _
                => $"Bilinmeyen event: {evt.GetType().Name}"
        };
}

// ===== 3. Result pattern ile API yanıtları =====

// Discriminated union benzeri Result tipi
public abstract record Sonuc<T>
{
    public record Basarili(T Deger) : Sonuc<T>;
    public record Basarisiz(string HataKodu, string HataMesaji) : Sonuc<T>;
    public record BulunamadiSonuc() : Sonuc<T>;
}

// Kullanım
public class UrunServisi
{
    private readonly List<UrunYanit> _urunler = new()
    {
        new(1, "Laptop", 15000m, true, "Aktif"),
        new(2, "Mouse", 250m, false, "Stok Yok"),
        new(3, "Klavye", 500m, true, "Aktif")
    };

    public Sonuc<UrunYanit> UrunGetir(int id)
    {
        var urun = _urunler.FirstOrDefault(u => u.Id == id);
        return urun switch
        {
            null                        => new Sonuc<UrunYanit>.BulunamadiSonuc(),
            { Durum: "Pasif" }          => new Sonuc<UrunYanit>.Basarisiz("PASIF_URUN", "Ürün aktif değil"),
            var u                       => new Sonuc<UrunYanit>.Basarili(u)
        };
    }

    public IActionResult SonucaDonustur<T>(Sonuc<T> sonuc) =>
        sonuc switch
        {
            Sonuc<T>.Basarili { Deger: var d }
                => new OkObjectResult(d),

            Sonuc<T>.BulunamadiSonuc
                => new NotFoundResult(),

            Sonuc<T>.Basarisiz { HataKodu: var kod, HataMesaji: var mesaj }
                => new BadRequestObjectResult(new { kod, mesaj }),

            _
                => new StatusCodeResult(500)
        };
}

// Kullanım örnekleri
var isleyici = new DomainEventIsleyici();

var evt1 = new UrunOlusturuldu(1, "Laptop", 15000m);
Console.WriteLine(isleyici.EventAciklamasi(evt1));
// Çıktı: Yeni ürün eklendi: Laptop (₺15.000,00)

var evt2 = new UrunFiyatiGuncellendi(1, 15000m, 12000m);
Console.WriteLine(isleyici.EventAciklamasi(evt2));
// Çıktı: Fiyat indirimi: ₺15.000,00 -> ₺12.000,00 (%20.0)

var evt3 = new SiparisVerildi(101, 55, new List<SiparisKalemi2>
{
    new(1, 2, 15000m)
}, 30000m);
Console.WriteLine(isleyici.EventAciklamasi(evt3));
// Çıktı: Büyük sipariş! Müşteri #55, Tutar: ₺30.000,00

var servis = new UrunServisi();
var sonuc = servis.UrunGetir(999);
Console.WriteLine(sonuc switch
{
    Sonuc<UrunYanit>.Basarili { Deger: var u } => $"Bulundu: {u.Ad}",
    Sonuc<UrunYanit>.BulunamadiSonuc           => "Ürün bulunamadı",
    Sonuc<UrunYanit>.Basarisiz { HataMesaji: var m } => $"Hata: {m}",
    _                                          => "Bilinmeyen durum"
});
// Çıktı: Ürün bulunamadı
```

---

## Özet

| Özellik | C# Versiyonu | Açıklama |
|---|---|---|
| `record` | C# 9 | Değer eşitliği, immutability, `with` ifadesi |
| `record struct` | C# 10 | Stack üzerinde yaşayan record |
| `init` erişimcisi | C# 9 | Yalnızca başlatma sırasında atanabilir özellik |
| Switch expression | C# 8 | Değer döndüren switch |
| Property pattern | C# 8 | `{ OzellikAdi: deger }` eşleşmesi |
| Positional pattern | C# 8 | Deconstruct tabanlı eşleşme |
| Relational pattern | C# 9 | `<`, `>`, `<=`, `>=` karşılaştırmaları |
| Logical pattern | C# 9 | `and`, `or`, `not` kombinasyonları |
| List pattern | C# 11 | `[ilk, .., son]` dizi eşleşmesi |
| Primary constructor | C# 12 | Class/struct için birincil yapıcı |

Records ve pattern matching kullanırken dikkat edilmesi gereken noktalar:

- `record class` referans tipidir, `record struct` değer tipidir
- Records'ta kalıtım yalnızca `record class` ile mümkündür
- Pattern matching'de `_` joker karakteri her zaman eşleşir ve son sıraya konulmalıdır
- Switch expression'da tüm durumlar ele alınmazsa derleme uyarısı alınır
- `required` anahtar kelimesi (C# 11) ile init özelliklerinin zorunlu olması sağlanabilir
- List pattern'ler `IList<T>` ve dizilerle çalışır, `IEnumerable<T>` ile çalışmaz
