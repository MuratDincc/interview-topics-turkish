# Nullable Reference Types (NRT)

## Genel Bakış

Nullable Reference Types (NRT), C# 8.0 ile birlikte tanıtılan ve null referans hatalarını derleme zamanında tespit etmeye yarayan bir özelliktir. `NullReferenceException` (NRE), .NET geliştirmede en sık karşılaşılan çalışma zamanı hatalarından biridir. NRT, bu hataları çalışma zamanından derleme zamanına taşıyarak kodun güvenilirliğini artırır.

Bu özellik etkinleştirildiğinde, derleyici hangi değişkenlerin null olabileceğini ve hangilerinin olamayacağını statik analiz yoluyla takip eder. Böylece olası null dereference noktaları derleme aşamasında uyarı olarak bildirilir.

**Temel Kavramlar:**

- **Non-nullable reference type**: `string`, `MyClass` — null atanamaz, null olamaz
- **Nullable reference type**: `string?`, `MyClass?` — null atanabilir, kontrol edilmesi beklenir
- **Nullable context**: Derleyicinin NRT analizini hangi kapsamda yaptığını belirler
- **Nullable attributes**: Derleyiciye ek ipuçları veren özel attribute'lar

---

## Mülakat Soruları ve Cevapları

### 1. Soru

**Nullable Reference Types nedir ve neden kullanılır? Proje seviyesinde nasıl etkinleştirilir?**

**Cevap:**

Nullable Reference Types, C# 8.0 ile gelen ve derleyicinin null güvenliği analizini etkinleştiren bir özelliktir. Bu özellik sayesinde derleyici, bir değişkenin null olup olamayacağını tip sistemi üzerinden takip eder ve potansiyel null dereference hatalarını derleme zamanında uyarı olarak raporlar.

Özelliği etkinleştirmenin iki yolu vardır:

**Dosya seviyesinde** (`#nullable enable` direktifi ile):

```csharp
#nullable enable

public class KullaniciBilgisi
{
    // string null olamaz — derleyici bunu garanti eder
    public string Ad { get; set; } = string.Empty;

    // string? null olabilir — açıkça belirtilmiş
    public string? Soyad { get; set; }

    public KullaniciBilgisi(string ad)
    {
        Ad = ad;
    }
}
```

**Proje seviyesinde** (`.csproj` dosyasında):

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>

    <!-- Tüm proje için NRT'yi etkinleştirir -->
    <Nullable>enable</Nullable>

    <!-- NRT uyarılarını hata olarak işler — CI/CD için önerilir -->
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>

    <!-- Ya da yalnızca nullable uyarılarını hata say -->
    <WarningsAsErrors>nullable</WarningsAsErrors>
  </PropertyGroup>
</Project>
```

**.NET 6+ projelerinde** bu ayar varsayılan olarak `enable` gelir. Eski projelerde ise `disable` veya hiç tanımlı olmayabilir.

Mevcut seçenekler:

| Değer | Açıklama |
|---|---|
| `enable` | NRT tamamen aktif |
| `disable` | NRT devre dışı (eski davranış) |
| `warnings` | Uyarılar gösterilir ama annotation analizi yapılmaz |
| `annotations` | Annotation analizi yapılır ama uyarı verilmez |

---

### 2. Soru

**`string` ile `string?` arasındaki fark nedir? Derleyici bu ayrımı nasıl zorlar?**

**Cevap:**

`string` (non-nullable) ve `string?` (nullable) arasındaki fark, derleyicinin null atamalarına ve kullanımlarına nasıl davrandığıdır.

```csharp
#nullable enable

public class OrnekKullanim
{
    public void NonNullableOrnekleri()
    {
        string ad = "Ahmet";          // Gecerli
        string ad2 = null!;           // Uyarı: CS8600 — null forgiving operator ile susturulur
        // string ad3 = null;         // HATA: CS8600 — non-nullable'a null atanamaz

        // Null kontrolü gerekmez; derleyici null olmadığını bilir
        Console.WriteLine(ad.Length); // Guvenli
    }

    public void NullableOrnekleri()
    {
        string? soyad = null;         // Gecerli
        string? soyad2 = "Yilmaz";   // Gecerli

        // Dogrudan kullanilamaz — CS8602 uyarisi verir
        // Console.WriteLine(soyad.Length); // UYARI: Olasi null dereference

        // Dogru kullanim: once null kontrolu yap
        if (soyad != null)
        {
            Console.WriteLine(soyad.Length); // Guvenli — if blogu icinde null degil
        }

        // Ya da null-conditional operator kullan
        Console.WriteLine(soyad?.Length ?? 0);
    }
}

// Metot imzalari arasindaki fark
public class MetotImzasi
{
    // Garantili non-null donus degeri
    public string AdGetir() => "Ahmet";

    // Null donebilir
    public string? KullaniciAdiBul(int id)
    {
        // Bulamazsa null doner
        return id > 0 ? "kullanici" : null;
    }

    public void KullanımOrnegi()
    {
        string ad = AdGetir();        // Null kontrolu gerekmez
        Console.WriteLine(ad.Length); // Guvenli

        string? bulunanAd = KullaniciAdiBul(5);

        // CS8602 uyarisi alirmak istemiyorsak null kontrolu sart
        if (bulunanAd is not null)
        {
            Console.WriteLine(bulunanAd.Length); // Guvenli
        }

        // Ya da:
        Console.WriteLine(bulunanAd?.Length ?? -1);
    }
}
```

Derleyici, `string` tipini "null olamaz" olarak işaretler ve şu durumlarda uyarı verir:

- Null atandığında (CS8600)
- Null kontrol edilmeden kullanıldığında (CS8602)
- Null döndürülmesi beklenmeyen metodun null döndürdüğünde (CS8603)

---

### 3. Soru

**Null-forgiving operator (`!`) nedir, ne zaman kullanılmalı ve riskleri nelerdir?**

**Cevap:**

Null-forgiving operator (`!`), derleyiciye "ben bu değerin null olmadığını garanti ediyorum, beni bu konuda uyarma" demek için kullanılır. Derleyicinin null analizini geçici olarak susturur; ancak çalışma zamanı davranışını değiştirmez.

```csharp
#nullable enable

public class NullForgivingOrnek
{
    private string? _dahiliDeger;

    // DOGRU KULLANIM 1: Test kodlarinda — test framework null olmayan deger saglar
    [SetUp]
    public void Setup()
    {
        // NUnit veya xUnit, [SetUp] calistiginda bu garantilenebilir
        _dahiliDeger = "test degeri";
    }

    // DOGRU KULLANIM 2: Null olamayacagini biz biliyoruz ama derleyici bilmiyor
    public void DogrunuKullanim()
    {
        string? metin = AramaYap("anahtar");

        // Eger arama her zaman sonuc donuyorsa ve biz bunu biliyorsak:
        if (metin != null)
        {
            string kesinlikleVar = metin!; // Aslinda burada ! gerekmez, if yeterli
        }

        // Daha gercekci senaryo: Dictionary lookup
        var sozluk = new Dictionary<string, string>
        {
            { "tr", "Merhaba" }
        };

        // TryGetValue kontrol ettikten sonra degeri kullanmak
        if (sozluk.TryGetValue("tr", out string? deger))
        {
            // Burada deger null olamaz, ama derleyici bunu her zaman anlamayabilir
            string kesin = deger!;
            Console.WriteLine(kesin.ToUpper());
        }
    }

    // YANLIS KULLANIM — null olabilecek yere ! koymak calisma zamaninda patlar
    public void YanlisKullanim()
    {
        string? nullable = null;

        // Derleyici susar, ama calisma zamaninda NullReferenceException firlatir!
        string yanlis = nullable!;
        Console.WriteLine(yanlis.Length); // NullReferenceException!
    }

    // TERCIH EDILMESI GEREKEN YAKLASIMLAR
    public void TercihEdilenYaklasimlar()
    {
        string? deger = null;

        // 1. Null kontrolu ile
        if (deger is not null)
        {
            Console.WriteLine(deger.Length);
        }

        // 2. Null-conditional ile
        int uzunluk = deger?.Length ?? 0;

        // 3. ArgumentNullException ile (constructor veya public API icin)
        string kesin = deger ?? throw new ArgumentNullException(nameof(deger));
        Console.WriteLine(kesin.Length);

        // 4. Pattern matching ile
        if (deger is string gecerliDeger)
        {
            Console.WriteLine(gecerliDeger.Length);
        }
    }

    private string? AramaYap(string anahtar) => null;
}
```

**Null-forgiving operator ne zaman KULLANILMALI:**
- Test setup metodlarında, framework tarafından doldurulacak alanlar için
- Deserialization sonrasında (JSON, XML) — framework null olmayan değer sağlar
- Derleyicinin pattern'i anlayamadığı karmaşık null kontrolü senaryolarında
- Legacy kod entegrasyonunda geçici olarak

**Null-forgiving operator ne zaman KULLANILMAMALI:**
- "Muhtemelen null değildir" varsayımıyla
- Uyarıyı hızlıca kapatmak için
- Null kontrolü yazmak yerine kısayol olarak

---

### 4. Soru

**Null-conditional (`?.`, `?[]`) ve null-coalescing (`??`, `??=`) operatörleri nasıl çalışır? NRT ile birlikte nasıl kullanılır?**

**Cevap:**

```csharp
#nullable enable

public class Siparis
{
    public string SiparisNo { get; set; } = string.Empty;
    public Musteri? Musteri { get; set; }
    public List<SiparisKalemi>? Kalemler { get; set; }
}

public class Musteri
{
    public string Ad { get; set; } = string.Empty;
    public Adres? Adres { get; set; }
}

public class Adres
{
    public string Sehir { get; set; } = string.Empty;
    public string? PostaKodu { get; set; }
}

public class SiparisKalemi
{
    public string UrunAdi { get; set; } = string.Empty;
    public decimal Fiyat { get; set; }
}

public class OperatorOrnekleri
{
    public void NullConditionalOrnekleri(Siparis? siparis)
    {
        // ?. — Null-conditional member access
        // Sol taraf null ise ifade null doner, null degilse devam eder
        string? musteriAdi = siparis?.Musteri?.Ad;
        string? sehir = siparis?.Musteri?.Adres?.Sehir;

        Console.WriteLine($"Musteri: {musteriAdi ?? "Bilinmiyor"}");

        // ?[] — Null-conditional index access
        // Koleksiyon null ise null doner
        SiparisKalemi? ilkKalem = siparis?.Kalemler?[0];
        Console.WriteLine($"Ilk kalem: {ilkKalem?.UrunAdi ?? "Yok"}");

        // Metot cagrimasi ile
        int? kalemSayisi = siparis?.Kalemler?.Count;

        // Null-conditional ile event tetikleme (klasik pattern)
        EventHandler? olay = null;
        olay?.Invoke(this, EventArgs.Empty); // Null ise hicbir sey yapmaz
    }

    public void NullCoalescingOrnekleri(string? ad, List<string>? liste)
    {
        // ?? — Null-coalescing operator
        // Sol taraf null ise sag tarafi doner, degil ise sol tarafi
        string kesinAd = ad ?? "Anonim";
        Console.WriteLine(kesinAd.Length); // Guvenli — kesinAd null olamaz

        // Zincirleme kullanim
        string? oncelik1 = null;
        string? oncelik2 = null;
        string? oncelik3 = "Bulundu";

        string sonuc = oncelik1 ?? oncelik2 ?? oncelik3 ?? "Hic biri yok";

        // ?? ile exception firlatma (guard clause pattern)
        public void IslemYap(string? veri)
        {
            string guvenliVeri = veri ?? throw new ArgumentNullException(nameof(veri));
            // Bu noktadan sonra guvenliVeri null olamaz
            Console.WriteLine(guvenliVeri.ToUpper());
        }

        // ??= — Null-coalescing assignment
        // Sol taraf null ise sag tarafi atar, degilse hicbir sey yapmaz
        string? gecikmeliDeger = null;
        gecikmeliDeger ??= "Varsayilan"; // Sadece null ise atar
        gecikmeliDeger ??= "Bu atanmaz"; // Artik null degil, atama yapilmaz

        // Liste ornegi
        List<string> guvenliListe = liste ?? new List<string>();
        // ya da:
        liste ??= new List<string>(); // liste null ise yeni liste olustur
        liste.Add("Eleman"); // Guvenli — liste artik null olamaz
    }

    // Gercek dunya senaryosu: Derinlemesine nested null kontrolu
    public string MusteriSehriGetir(Siparis? siparis)
    {
        // Uzun yol
        if (siparis == null) return "Bilinmiyor";
        if (siparis.Musteri == null) return "Bilinmiyor";
        if (siparis.Musteri.Adres == null) return "Bilinmiyor";
        return siparis.Musteri.Adres.Sehir;

        // Kisa yol — ayni sonucu verir
        return siparis?.Musteri?.Adres?.Sehir ?? "Bilinmiyor";
    }
}
```

---

### 5. Soru

**Nullable attributes nelerdir ve nasıl kullanılır? `[NotNull]`, `[MaybeNull]`, `[NotNullWhen]`, `[MemberNotNull]` örnekleri verin.**

**Cevap:**

Nullable attributes, `System.Diagnostics.CodeAnalysis` namespace'inden gelen ve derleyiciye ek null analiz ipuçları veren attribute'lardır. Basit null kontrolünün yetmediği durumlarda devreye girer.

```csharp
#nullable enable
using System.Diagnostics.CodeAnalysis;

public class NullableAttributeOrnekleri
{
    // [NotNull] — Parametre null gelebilir ama metot cikmadan once null olmayacak
    // veya donus degeri hic null olmaz
    public void Normalize([NotNull] ref string? deger)
    {
        deger ??= string.Empty;
        // Bu noktadan sonra deger null olamaz
    }

    // Kullanim:
    public void NormalizeKullanim()
    {
        string? ad = null;
        Normalize(ref ad);
        Console.WriteLine(ad.Length); // Uyari yok — [NotNull] garanti verdi
    }

    // [MaybeNull] — Non-nullable tipte donus degerinin null olabileceğini belirtir
    // Generic metodlar icin kullanisli
    [return: MaybeNull]
    public T Bul<T>(IEnumerable<T> koleksiyon, Func<T, bool> kosul)
    {
        foreach (var eleman in koleksiyon)
        {
            if (kosul(eleman)) return eleman;
        }
        return default!; // T reference type ise null, value type ise default
    }

    // [NotNullWhen] — Boolean donus degeri true/false oldugunda null durumunu belirtir
    public bool DogrulamaYap([NotNullWhen(true)] string? deger)
    {
        return !string.IsNullOrWhiteSpace(deger);
    }

    // Kullanim — derleyici [NotNullWhen(true)] sayesinde bunu anlar:
    public void DogrulamaKullanim(string? kullaniciAdi)
    {
        if (DogrulamaYap(kullaniciAdi))
        {
            // Bu blokta kullaniciAdi null olamaz!
            Console.WriteLine(kullaniciAdi.ToUpper()); // Uyari yok
        }
    }

    // string.IsNullOrEmpty de bu attribute'u kullanir:
    // public static bool IsNullOrEmpty([NotNullWhen(false)] string? value)
    public void IsNullOrEmptyOrnegi(string? metin)
    {
        if (!string.IsNullOrEmpty(metin))
        {
            Console.WriteLine(metin.Length); // Guvenli — IsNullOrEmpty false dondu
        }
    }

    // [MemberNotNull] — Metot cagirildiktan sonra belirtilen uyelerin null olmadigi garantisi
    private string? _ad;
    private string? _email;

    [MemberNotNull(nameof(_ad), nameof(_email))]
    private void Baslat()
    {
        _ad = "Varsayilan";
        _email = "varsayilan@ornek.com";
    }

    public void MemberNotNullKullanim()
    {
        Baslat();
        // Derleyici Baslat() cagirildiktan sonra _ad ve _email'in null olmadığini bilir
        Console.WriteLine(_ad.Length);   // Uyari yok
        Console.WriteLine(_email.Length); // Uyari yok
    }

    // [AllowNull] — Non-nullable parametreye null atanabilmesini saglar
    // Ozelliklerin set'i icin kullanisli
    private string _baslik = string.Empty;

    [AllowNull]
    public string Baslik
    {
        get => _baslik;
        set => _baslik = value ?? string.Empty; // null gelirse bos string yap
    }

    // [DisallowNull] — Nullable parametreye null atanamayacagini belirtir
    // Bir nullable property'ye null set edilmesini engeller
    private string? _aciklama;

    [DisallowNull]
    public string? Aciklama
    {
        get => _aciklama;
        set => _aciklama = value ?? throw new ArgumentNullException(nameof(value));
    }

    // [MaybeNullWhen] — NotNullWhen'in tersi: false dondiginde null olabilir
    public bool Cevir(string? kaynak, [MaybeNullWhen(false)] out string hedef)
    {
        if (string.IsNullOrEmpty(kaynak))
        {
            hedef = null;
            return false;
        }
        hedef = kaynak.ToUpper();
        return true;
    }

    public void CevirKullanim(string? veri)
    {
        if (Cevir(veri, out string? sonuc))
        {
            Console.WriteLine(sonuc.Length); // Guvenli — true dondu, MaybeNullWhen(false) yok
        }
    }
}
```

---

### 6. Soru

**Nullable context direktifleri (`#nullable enable/disable/restore`) nasıl çalışır ve granüler kontrolde nasıl kullanılır?**

**Cevap:**

Nullable context, derleyicinin hangi kapsamda NRT analizi yapacağını belirler. Büyük projelerde veya migration süreçlerinde parçalı kontrol önemlidir.

```csharp
// Proje genelinde disable olabilir (.csproj'da Nullable=disable)
// Ama bu dosyada enable edebiliriz:

#nullable enable

public class YeniKod
{
    public string Ad { get; set; } = string.Empty; // NRT kurallari gecerli
    public string? Soyad { get; set; }
}

#nullable disable

public class EskiKod
{
    // Burada NRT kurallari gecerli degil
    // string ve string? arasinda fark yok
    public string Ad { get; set; } // Uyari yok
    public string Soyad { get; set; } // Uyari yok, null atanabilir
}

#nullable restore
// Proje seviyesindeki ayara geri doner
// Proje genelinde enable ise enable, disable ise disable

// Sadece annotations veya sadece warnings da acilabilir
#nullable enable annotations  // Sadece annotation analizi, uyari yok
#nullable enable warnings      // Sadece uyarilar, annotation analizi yok

// KULLANIM SENARYOLARI:

// Senaryo 1: Belirli bir metot icin susturma
#nullable enable

public class KarisikKod
{
    public string? KullaniciyiGetir(int id)
    {
        // Null donebilir — NRT ile uyumlu
        return id > 0 ? "kullanici" : null;
    }

#nullable disable
    // Eski, refactor edilmemis kod — gecici olarak disable
    public string EskiMetot(string deger)
    {
        if (deger == null) return ""; // Eski stil null kontrolu
        return deger.ToUpper();
    }
#nullable restore

    // Bu noktan sonra proje ayari gecerli
    public string YeniMetot(string deger)
    {
        // NRT kurallari yeniden gecerli
        ArgumentNullException.ThrowIfNull(deger);
        return deger.ToUpper();
    }
}
```

**Nullable context seviyeleri:**

| Direktif | Annotations | Warnings |
|---|---|---|
| `#nullable enable` | Aktif | Aktif |
| `#nullable disable` | Devre disi | Devre disi |
| `#nullable restore` | Proje ayarına göre | Proje ayarına göre |
| `#nullable enable annotations` | Aktif | Devre disi |
| `#nullable enable warnings` | Devre disi | Aktif |

---

### 7. Soru

**CS8600, CS8602, CS8603 uyarıları ne anlama gelir ve nasıl düzeltilir?**

**Cevap:**

```csharp
#nullable enable

public class CompilerWarningOrnekleri
{
    // ===== CS8600 =====
    // "Converting null literal or possible null value to non-nullable type"
    // Non-nullable'a null veya nullable deger atanmayi deniyorsunuz

    public void CS8600Ornekleri()
    {
        string? nullableDeger = null;

        // CS8600 uyarisi: nullable'dan non-nullable'a dogrudan atama
        // string kesin = nullableDeger; // UYARI CS8600

        // COZUM 1: Null kontrolu ile
        if (nullableDeger is not null)
        {
            string kesin1 = nullableDeger; // Guvenli
        }

        // COZUM 2: Null-coalescing ile
        string kesin2 = nullableDeger ?? "Varsayilan";

        // COZUM 3: Guard clause ile
        string kesin3 = nullableDeger ?? throw new InvalidOperationException("Deger null olamaz");

        // COZUM 4: ArgumentNullException (public API icin)
        static string Islem(string? deger)
        {
            ArgumentNullException.ThrowIfNull(deger); // .NET 6+
            return deger.ToUpper(); // Guvenli
        }
    }

    // ===== CS8602 =====
    // "Dereference of a possibly null reference"
    // Nullable bir referansi null kontrolu yapmadan kullaniyorsunuz

    public void CS8602Ornekleri()
    {
        string? ad = OkuVeriTabanindan();

        // CS8602 uyarisi: null kontrolu olmadan kullanim
        // Console.WriteLine(ad.Length); // UYARI CS8602

        // COZUM 1: if ile null kontrolu
        if (ad != null)
        {
            Console.WriteLine(ad.Length); // Guvenli
        }

        // COZUM 2: Null-conditional operator
        Console.WriteLine(ad?.Length ?? 0); // Guvenli

        // COZUM 3: Pattern matching
        if (ad is string gecerliAd)
        {
            Console.WriteLine(gecerliAd.Length); // Guvenli
        }

        // COZUM 4: Debug.Assert (test/debug ortami icin)
        System.Diagnostics.Debug.Assert(ad != null, "ad null olmamali");
        Console.WriteLine(ad!.Length); // Debug.Assert sonrasi ! guvenli sayilabilir
    }

    // ===== CS8603 =====
    // "Possible null reference return"
    // Non-nullable donus tipli metodun null donebilecegi ifade var

    // CS8603 uyarisi veren metot:
    // public string AdGetir(int id)
    // {
    //     return _veritabani.Bul(id)?.Ad; // UYARI CS8603 — nullable donuyor
    // }

    // COZUM 1: Donus tipini nullable yap
    public string? AdGetirNullable(int id)
    {
        return _veritabani.Bul(id)?.Ad; // Guvenli — string? donuyor
    }

    // COZUM 2: Null-coalescing ile default deger ver
    public string AdGetirDefault(int id)
    {
        return _veritabani.Bul(id)?.Ad ?? "Bilinmiyor"; // Guvenli — string donuyor
    }

    // COZUM 3: Guard clause
    public string AdGetirGuard(int id)
    {
        var kullanici = _veritabani.Bul(id)
            ?? throw new InvalidOperationException($"ID {id} icin kullanici bulunamadi");
        return kullanici.Ad; // Guvenli
    }

    // Diger yaygin uyarilar:
    // CS8604: Possible null reference argument — nullable'i non-nullable parametreye gecirme
    // CS8618: Non-nullable field must contain a non-null value when exiting constructor
    // CS8625: Cannot convert null literal to non-nullable reference type

    public void CS8604Ornegi()
    {
        string? nullable = null;

        // CS8604: nullable deger non-nullable parametre bekleyen metoda geciliyor
        // KullanNonNullable(nullable); // UYARI CS8604

        // COZUM:
        if (nullable is not null)
            KullanNonNullable(nullable);

        // veya:
        KullanNonNullable(nullable ?? "Varsayilan");
    }

    private void KullanNonNullable(string deger) => Console.WriteLine(deger);
    private string? OkuVeriTabanindan() => null;

    private readonly FakeVeriTabani _veritabani = new();
}

// CS8618 ornegi — constructor'da null olabilecek field
public class CS8618Ornegi
{
    // CS8618: Non-nullable property 'Ad' must contain a non-null value
    // public string Ad { get; set; } // UYARI — constructor'da set edilmemis

    // COZUM 1: Constructor'da set et
    public string Ad { get; set; }
    public CS8618Ornegi(string ad) { Ad = ad; }
}

public class CS8618Cozum2
{
    // COZUM 2: Field initializer ile
    public string Ad { get; set; } = string.Empty;
    public List<string> Liste { get; set; } = new();
}

// Yardimci siniflar
public class FakeVeriTabani
{
    public FakeKullanici? Bul(int id) => id > 0 ? new FakeKullanici { Ad = "Test" } : null;
}
public class FakeKullanici { public string Ad { get; set; } = string.Empty; }
```

---

### 8. Soru

**NRT ile API tasarımı nasıl yapılmalı? Method signature ve constructor pattern best practices nelerdir?**

**Cevap:**

```csharp
#nullable enable

// KOTU TASARIM — NRT'den once tipik pattern
public class KotuAPITasarimi
{
    // Null donebilir mi? Belirsiz
    public string KullaniciBul(int id) => ""; // Aslinda null donebiliyor olabilir

    // Parametre null olabilir mi? Belirsiz
    public void KaydetYeni(string? ad, string? email) { } // Hem nullable hem non-nullable belirsiz
}

// IYI TASARIM — NRT ile net sozlesme
public class IyiAPITasarimi
{
    // Net: null donebilir, cagiran null kontrolu yapmali
    public Kullanici? KullaniciGetir(int id) => null;

    // Net: null donmez, bulamazsa exception firlatir
    public Kullanici KullaniciGetirYaDaHata(int id)
        => KullaniciGetir(id) ?? throw new KeyNotFoundException($"Kullanici {id} bulunamadi");

    // Net: ad zorunlu, aciklama optional
    public void KaydetYeni(string ad, string? aciklama = null) { }
}

// CONSTRUCTOR PATTERNS

// Pattern 1: Zorunlu parametreler constructor'da
public class UrunZorunluCtr
{
    public string Ad { get; }
    public decimal Fiyat { get; }
    public string? Aciklama { get; set; } // Optional

    public UrunZorunluCtr(string ad, decimal fiyat)
    {
        // Guard clauses — public API icin
        ArgumentNullException.ThrowIfNull(ad);
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(fiyat);

        Ad = ad;
        Fiyat = fiyat;
    }
}

// Pattern 2: Primary constructor (C# 12)
public class UrunPrimary(string ad, decimal fiyat)
{
    public string Ad { get; } = ad ?? throw new ArgumentNullException(nameof(ad));
    public decimal Fiyat { get; } = fiyat;
    public string? Aciklama { get; set; }
}

// Pattern 3: Builder pattern ile null guvenli nesne olusturma
public class SiparisBuilder
{
    private string? _musteriAdi;
    private string? _adres;
    private readonly List<string> _urunler = new();

    public SiparisBuilder MusteriAdiKoy(string musteriAdi)
    {
        _musteriAdi = musteriAdi ?? throw new ArgumentNullException(nameof(musteriAdi));
        return this;
    }

    public SiparisBuilder AdresKoy(string adres)
    {
        _adres = adres ?? throw new ArgumentNullException(nameof(adres));
        return this;
    }

    public SiparisBuilder UrunEkle(string urun)
    {
        _urunler.Add(urun ?? throw new ArgumentNullException(nameof(urun)));
        return this;
    }

    public Siparis Olustur()
    {
        if (_musteriAdi is null) throw new InvalidOperationException("Musteri adi zorunludur");
        if (_adres is null) throw new InvalidOperationException("Adres zorunludur");

        return new Siparis
        {
            MusteriAdi = _musteriAdi,
            Adres = _adres,
            Urunler = _urunler.ToList()
        };
    }
}

public class Siparis
{
    public string MusteriAdi { get; init; } = string.Empty;
    public string Adres { get; init; } = string.Empty;
    public List<string> Urunler { get; init; } = new();
}

// IYI PRACTICE: Interface tasariminda null netligi
public interface IKullaniciRepository
{
    // Null donebilir — bulamazsa null
    Task<Kullanici?> GetirAsync(int id);

    // Null donmez — bulamazsa exception
    Task<IReadOnlyList<Kullanici>> TumunuGetirAsync();

    // Parametre null olamaz
    Task<Kullanici> EkleAsync(Kullanici kullanici);

    // Optional filter
    Task<IReadOnlyList<Kullanici>> AraAsync(string? aramaMetni = null);
}

public class Kullanici
{
    public int Id { get; set; }
    public string Ad { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string? Telefon { get; set; }
}
```

---

### 9. Soru

**Entity Framework Core ile NRT nasıl kullanılır? Required ve optional navigation property farkı nedir?**

**Cevap:**

```csharp
#nullable enable
using Microsoft.EntityFrameworkCore;

// ENTITY TASARIMI

// NRT ile entity ozellikleri — EF Core NRT'yi anlayarak migration uretir
public class Blog
{
    public int Id { get; set; }

    // Non-nullable string — EF Core bu kolonu NOT NULL olarak olusturur
    public string Baslik { get; set; } = string.Empty;

    // Nullable string — EF Core bu kolonu NULL yapabilir olarak olusturur
    public string? Aciklama { get; set; }

    // Required navigation property (non-nullable) — Blog her zaman bir sahibe sahip
    public Kullanici Sahip { get; set; } = null!; // EF Core yukler, null! gerekli
    public int SahipId { get; set; }

    // Optional navigation property (nullable) — Blog kategorisiz olabilir
    public Kategori? Kategori { get; set; }
    public int? KategoriId { get; set; }

    // Collection navigation — hic null olmaz, EF Core listeler
    public ICollection<Yazi> Yazilar { get; set; } = new List<Yazi>();
}

public class Yazi
{
    public int Id { get; set; }
    public string Baslik { get; set; } = string.Empty;
    public string Icerik { get; set; } = string.Empty;
    public DateTime YayinTarihi { get; set; }

    // Required — her yazi bir bloga ait olmali
    public Blog Blog { get; set; } = null!;
    public int BlogId { get; set; }

    // Optional — yazar olmayabilir (anonim)
    public Kullanici? Yazar { get; set; }
    public int? YazarId { get; set; }
}

public class Kullanici
{
    public int Id { get; set; }
    public string Ad { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;

    public ICollection<Blog> Bloglar { get; set; } = new List<Blog>();
}

public class Kategori
{
    public int Id { get; set; }
    public string Ad { get; set; } = string.Empty;
    public ICollection<Blog> Bloglar { get; set; } = new List<Blog>();
}

// DbContext konfigurasyonu
public class BlogDbContext : DbContext
{
    public DbSet<Blog> Bloglar => Set<Blog>();
    public DbSet<Yazi> Yazilar => Set<Yazi>();
    public DbSet<Kullanici> Kullanicilar => Set<Kullanici>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // NRT ile EF Core genellikle convention'larla dogru cikarsama yapar
        // Ancak karmasik iliskiler icin explicit konfigürasyon gerekebilir

        modelBuilder.Entity<Blog>(entity =>
        {
            entity.HasOne(b => b.Sahip)
                  .WithMany(u => u.Bloglar)
                  .HasForeignKey(b => b.SahipId)
                  .IsRequired(); // Non-nullable navigation = required

            entity.HasOne(b => b.Kategori)
                  .WithMany(k => k.Bloglar)
                  .HasForeignKey(b => b.KategoriId)
                  .IsRequired(false); // Nullable navigation = optional

            entity.Property(b => b.Baslik)
                  .IsRequired()
                  .HasMaxLength(200);

            entity.Property(b => b.Aciklama)
                  .HasMaxLength(1000); // Nullable — IsRequired() cagirilmaz
        });
    }
}

// REPOSITORY KULLANIMI — NRT ile

public class BlogRepository
{
    private readonly BlogDbContext _context;

    public BlogRepository(BlogDbContext context)
    {
        _context = context ?? throw new ArgumentNullException(nameof(context));
    }

    // Nullable donus — bulamazsa null
    public async Task<Blog?> GetirAsync(int id)
    {
        return await _context.Bloglar
            .Include(b => b.Sahip)
            .Include(b => b.Kategori) // Nullable — null olabilir
            .FirstOrDefaultAsync(b => b.Id == id);
    }

    // Non-nullable donus — bulamazsa exception
    public async Task<Blog> GetirYaDaHataAsync(int id)
    {
        return await _context.Bloglar
            .Include(b => b.Sahip)
            .FirstOrDefaultAsync(b => b.Id == id)
            ?? throw new KeyNotFoundException($"Blog {id} bulunamadi");
    }

    public async Task KullanAsync(int blogId)
    {
        var blog = await GetirAsync(blogId);

        // blog nullable oldugu icin null kontrolu zorunlu
        if (blog is null)
        {
            Console.WriteLine("Blog bulunamadi");
            return;
        }

        // Blog.Sahip non-nullable — null kontrolu gerekmez
        Console.WriteLine($"Sahip: {blog.Sahip.Ad}");

        // Blog.Kategori nullable — null kontrolu gerekir
        if (blog.Kategori is not null)
        {
            Console.WriteLine($"Kategori: {blog.Kategori.Ad}");
        }

        // Ya da null-conditional ile:
        Console.WriteLine($"Kategori: {blog.Kategori?.Ad ?? "Kategorisiz"}");
    }
}
```

**EF Core ve NRT ozet kurallar:**

| Durum | EF Core Davranisi | Ornek |
|---|---|---|
| `string` property | NOT NULL kolonu | `public string Ad { get; set; }` |
| `string?` property | NULL kolonu | `public string? Aciklama { get; set; }` |
| Non-nullable nav | Required FK | `public Blog Blog { get; set; } = null!;` |
| Nullable nav | Optional FK | `public Blog? Blog { get; set; }` |
| `int` FK | NOT NULL FK | `public int BlogId { get; set; }` |
| `int?` FK | NULL FK | `public int? BlogId { get; set; }` |

---

### 10. Soru

**Mevcut bir projeye NRT nasıl eklenir? Migration stratejisi nedir?**

**Cevap:**

Büyük bir projeye NRT eklemek, dikkatli bir migration stratejisi gerektirir. Tüm projeyi bir anda enable etmek çok sayıda uyarıya neden olabilir.

```csharp
// ADIM 1: .csproj'u guncelle — once sadece annotations, warnings degil
// <Nullable>annotations</Nullable>
// Bu sayede mevcut kod uyari vermez ama yeni kod annotation'larla yazilabilir

// ADIM 2: En kritik, en cok kullanilan siniflardan baslayarak dosya bazli enable et

// Eski dosya — henuz migrate edilmemis
#nullable disable  // Proje annotations modunda, bunu disable eder
public class EskiSinif
{
    public string Ad { get; set; }  // Uyari yok
    public string Soyad { get; set; } // Uyari yok
}

// Yeni dosya — migrate edilmis
#nullable enable
public class YeniSinif
{
    public string Ad { get; set; } = string.Empty;
    public string? Soyad { get; set; }

    public YeniSinif(string ad)
    {
        Ad = ad;
    }
}

// ADIM 3: Migration yardimci pattern'leri

// PATTERN A: Suppress-then-fix
// Once ! ile sustur, zamanla dogru kodu yaz
public class GecisSinifi
{
    private string _veri = null!; // CS8618'i susturur, sonra duzeltilecek

    // TODO: Bu field lazy init veya constructor'da baslatilmali
}

// PATTERN B: Partial dosyalar icin ayri nullable context
public partial class BuyukSinif
{
    // Bu partial dosya migrate edildi
    #pragma warning disable CS8618 // Diger partial'lar ele alacak
    public string Ad { get; set; }
    #pragma warning restore CS8618
}

// ADIM 4: Global usings ve suppress (gecici)
// Projenin kökünde GlobalSuppressions.cs olustur
// [assembly: System.Diagnostics.CodeAnalysis.SuppressMessage(
//     "Nullable", "CS8618", Justification = "Migration sureci")]

// ADIM 5: Tum dosyalar migrate edildikten sonra .csproj'u tam enable'a al
// <Nullable>enable</Nullable>

// MIGRATION KONTROL LISTESI:

// 1. string property'ler
public class MigrationOrnek_Oncesi
{
    // Onceki hali:
    // public string Ad { get; set; }
    // public string Email { get; set; }
    // public string Aciklama { get; set; }
}

#nullable enable
public class MigrationOrnek_Sonrasi
{
    // Zorunlu alanlar
    public string Ad { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;

    // Optional alanlar
    public string? Aciklama { get; set; }

    // Constructor pattern
    public MigrationOrnek_Sonrasi(string ad, string email)
    {
        Ad = ad ?? throw new ArgumentNullException(nameof(ad));
        Email = email ?? throw new ArgumentNullException(nameof(email));
    }
}

// 2. Method signatures guncelleme
public class MetotMigration
{
    // Onceki:
    // public string KullaniciBul(int id) { return null; }
    // public void Kaydet(string ad, string email) { }

    // Sonraki:
    public string? KullaniciBul(int id) { return null; }  // Null donebilir

    public void Kaydet(string ad, string? email = null)   // email optional
    {
        ArgumentNullException.ThrowIfNull(ad);
        // ...
    }
}

// 3. Null check pattern modernlestirme
public class NullCheckMigration
{
    // Eski stil:
    // if (deger != null) { ... }
    // if (deger == null) return;
    // if (Object.ReferenceEquals(deger, null)) { ... }

    // Yeni stil (ayni semantik, daha okunakli):
    public void ModernNullCheck(string? deger)
    {
        // Hicbiri degil kontrolu
        if (deger is not null) { Console.WriteLine(deger); }

        // Guard:
        ArgumentNullException.ThrowIfNull(deger); // .NET 6+

        // Pattern matching:
        if (deger is { Length: > 0 }) { Console.WriteLine("Dolu"); }

        // Throw expression:
        string kesin = deger ?? throw new ArgumentNullException(nameof(deger));
    }
}
```

**Migration adimlar ozeti:**

1. `.csproj`'a `<Nullable>annotations</Nullable>` ekle (uyari vermeden annotation baslat)
2. En cok kullanilan/kritik siniflardan baslayarak `#nullable enable` ekle
3. Her dosyada: string property'leri `string` vs `string?` olarak siniflandir
4. Method signature'lari guncelle
5. Constructor'larda `= string.Empty`, `= new()` veya parameterized init ekle
6. Null kontrol stilini modernlestir (`is not null`, `??`, `?.`)
7. Tum dosyalar tamamlandiktan sonra `.csproj`'u `<Nullable>enable</Nullable>` yap
8. CI/CD'de nullable uyarilarini hata say: `<WarningsAsErrors>nullable</WarningsAsErrors>`

---

## Ozet

Nullable Reference Types, .NET projelerinde null guvenligini derleme zamanina tasiyarak `NullReferenceException` hatalarini onlemenin en etkili yoludur.

**Hatirlama noktalari:**

- `string` = null olamaz; `string?` = null olabilir
- `!` operatoru derleyiciyi susturur, runtime'i degistirmez — dikkatli kullanin
- `?.` ve `??` operatorleri null-safe kod yazmada temel araclardir
- Nullable attributes (`[NotNull]`, `[NotNullWhen]` vb.) karmasik senaryolarda derleyiciye ipucu verir
- EF Core, NRT annotation'larini okuyarak migration uretir
- Migration icin once `annotations` mode, sonra `enable` mode stratejisi uygulayın
- `CS8618` cozmek icin: field initializer, constructor init veya `= null!` (gecici)
