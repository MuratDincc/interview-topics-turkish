# SQL Temelleri

## Genel Bakış
SQL (Structured Query Language), ilişkisel veritabanlarında veri tanımlama, sorgulama ve yönetim işlemleri için kullanılan standart bir dildir. .NET backend geliştiricileri için SQL bilgisi kritik öneme sahiptir; hem doğrudan ADO.NET ile hem de Dapper gibi micro-ORM araçlarıyla SQL sorguları yazılır. Bu bölümde normalizasyon, JOIN türleri, gruplama, Stored Procedure'ler ve indeksleme gibi temel kavramlar mülakat soruları ekseninde ele alınmaktadır.

## Mülakat Soruları ve Cevapları

### 1. Veritabanı Normalizasyonu nedir? 1NF, 2NF ve 3NF'i açıklayınız.

**Cevap:**
Normalizasyon, veritabanı tasarımında veri tekrarını azaltmak ve veri bütünlüğünü korumak amacıyla tabloları belirli kurallara göre organize etme sürecidir.

**1NF (Birinci Normal Form):**
- Her sütun atomik (bölünemez) değerler içermelidir.
- Her satır benzersiz olmalıdır (birincil anahtar bulunmalıdır).
- Bir sütunda birden fazla değer (örneğin virgülle ayrılmış liste) bulunmamalıdır.

**2NF (İkinci Normal Form):**
- 1NF koşullarını sağlamalıdır.
- Bileşik birincil anahtar varsa, anahtar olmayan her sütun birincil anahtarın tamamına bağımlı olmalıdır (kısmi bağımlılık olmamalıdır).

**3NF (Üçüncü Normal Form):**
- 2NF koşullarını sağlamalıdır.
- Anahtar olmayan sütunlar arasında geçişli bağımlılık (transitive dependency) bulunmamalıdır; yani anahtar olmayan bir sütun, başka bir anahtar olmayan sütuna bağımlı olmamalıdır.

**Örnek Kod:**
```csharp
// 1NF İhlali: Telefon numaraları tek sütunda, virgülle ayrılmış
// Yanlış tablo yapısı (kavramsal):
// MusteriId | Ad    | Telefonlar
// 1         | Ali   | 555-1111, 555-2222

// 1NF Uyumlu C# modeli — her değer ayrı satır/tabloda
public class Musteri
{
    public int MusteriId { get; set; }
    public string Ad { get; set; } = string.Empty;
}

public class MusteriTelefon
{
    public int TelefonId { get; set; }
    public int MusteriId { get; set; }
    public string TelefonNumarasi { get; set; } = string.Empty;
    public Musteri Musteri { get; set; } = null!;
}

// 3NF İhlali: SiparisToplami, BirimFiyat ve Miktar'dan türetildiği için
// geçişli bağımlılık oluşturur.
// Yanlış yapı: SiparisId | UrunId | Miktar | BirimFiyat | SiparisToplami
// Doğru yapı: SiparisToplami hesaplanmalı, saklanmamalı.
public class SiparisDetay
{
    public int SiparisDetayId { get; set; }
    public int SiparisId { get; set; }
    public int UrunId { get; set; }
    public int Miktar { get; set; }
    public decimal BirimFiyat { get; set; }

    // Hesaplanan alan — veritabanında saklanmaz
    public decimal Toplam => Miktar * BirimFiyat;
}
```

---

### 2. JOIN türlerini açıklayınız ve aralarındaki farkları belirtiniz.

**Cevap:**
JOIN, iki veya daha fazla tabloyu ortak bir sütun üzerinden birleştirmek için kullanılır.

- **INNER JOIN**: Her iki tabloda da eşleşen satırları döndürür.
- **LEFT JOIN (LEFT OUTER JOIN)**: Sol tablodaki tüm satırları, sağ tabloda eşleşme yoksa NULL ile döndürür.
- **RIGHT JOIN (RIGHT OUTER JOIN)**: Sağ tablodaki tüm satırları, sol tabloda eşleşme yoksa NULL ile döndürür.
- **FULL JOIN (FULL OUTER JOIN)**: Her iki tablodaki tüm satırları döndürür; eşleşme yoksa NULL ile doldurur.
- **CROSS JOIN**: Sol tablonun her satırını sağ tablonun her satırıyla eşleştirir (Kartezyen çarpım).

**Örnek Kod:**
```csharp
using System.Data;
using Microsoft.Data.SqlClient;

public class SiparisRepository
{
    private readonly string _connectionString;

    public SiparisRepository(string connectionString)
    {
        _connectionString = connectionString;
    }

    // INNER JOIN — sadece siparişi olan müşterileri getir
    public async Task<List<MusteriSiparisBilgisi>> GetMusteriSiparisleriAsync()
    {
        const string sql = @"
            SELECT m.MusteriId, m.Ad, m.Soyad, s.SiparisId, s.SiparisTarihi
            FROM Musteriler m
            INNER JOIN Siparisler s ON m.MusteriId = s.MusteriId";

        var liste = new List<MusteriSiparisBilgisi>();

        await using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();

        await using var command = new SqlCommand(sql, connection);
        await using var reader = await command.ExecuteReaderAsync();

        while (await reader.ReadAsync())
        {
            liste.Add(new MusteriSiparisBilgisi
            {
                MusteriId = reader.GetInt32(0),
                Ad = reader.GetString(1),
                Soyad = reader.GetString(2),
                SiparisId = reader.GetInt32(3),
                SiparisTarihi = reader.GetDateTime(4)
            });
        }

        return liste;
    }

    // LEFT JOIN — siparişi olmayan müşterileri de getir
    public async Task<List<MusteriSiparisBilgisi>> GetTumMusterilerSiparislerleAsync()
    {
        const string sql = @"
            SELECT m.MusteriId, m.Ad, m.Soyad,
                   s.SiparisId, s.SiparisTarihi
            FROM Musteriler m
            LEFT JOIN Siparisler s ON m.MusteriId = s.MusteriId";

        var liste = new List<MusteriSiparisBilgisi>();

        await using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();

        await using var command = new SqlCommand(sql, connection);
        await using var reader = await command.ExecuteReaderAsync();

        while (await reader.ReadAsync())
        {
            liste.Add(new MusteriSiparisBilgisi
            {
                MusteriId = reader.GetInt32(0),
                Ad = reader.GetString(1),
                Soyad = reader.GetString(2),
                SiparisId = reader.IsDBNull(3) ? null : reader.GetInt32(3),
                SiparisTarihi = reader.IsDBNull(4) ? null : reader.GetDateTime(4)
            });
        }

        return liste;
    }

    // CROSS JOIN — tüm ürün-renk kombinasyonlarını oluştur
    public async Task<List<UrunRenkKombinasyon>> GetUrunRenkKombinasyonlariAsync()
    {
        const string sql = @"
            SELECT u.UrunAdi, r.RenkAdi
            FROM Urunler u
            CROSS JOIN Renkler r";

        var liste = new List<UrunRenkKombinasyon>();

        await using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();

        await using var command = new SqlCommand(sql, connection);
        await using var reader = await command.ExecuteReaderAsync();

        while (await reader.ReadAsync())
        {
            liste.Add(new UrunRenkKombinasyon
            {
                UrunAdi = reader.GetString(0),
                RenkAdi = reader.GetString(1)
            });
        }

        return liste;
    }
}

public record MusteriSiparisBilgisi
{
    public int MusteriId { get; init; }
    public string Ad { get; init; } = string.Empty;
    public string Soyad { get; init; } = string.Empty;
    public int? SiparisId { get; init; }
    public DateTime? SiparisTarihi { get; init; }
}

public record UrunRenkKombinasyon(string UrunAdi = "", string RenkAdi = "");
```

---

### 3. GROUP BY ve HAVING arasındaki fark nedir?

**Cevap:**
- **GROUP BY**: Satırları belirtilen sütun değerlerine göre gruplara ayırır. Genellikle `COUNT`, `SUM`, `AVG`, `MIN`, `MAX` gibi agregasyon fonksiyonlarıyla birlikte kullanılır.
- **HAVING**: Gruplandırma yapıldıktan sonra oluşan gruplara koşul uygulamak için kullanılır. `WHERE` satır düzeyinde filtreleme yaparken `HAVING` grup düzeyinde filtreleme yapar.

Önemli kural: `WHERE` agregasyon fonksiyonlarıyla birlikte kullanılamaz; bunun yerine `HAVING` kullanılmalıdır.

**Örnek Kod:**
```csharp
using Dapper;
using Microsoft.Data.SqlClient;

public class RaporRepository
{
    private readonly string _connectionString;

    public RaporRepository(string connectionString)
    {
        _connectionString = connectionString;
    }

    // GROUP BY — kategoriye göre ürün sayısı ve ortalama fiyat
    public async Task<List<KategoriRapor>> GetKategoriRaporuAsync()
    {
        const string sql = @"
            SELECT
                k.KategoriAdi,
                COUNT(u.UrunId)      AS UrunSayisi,
                AVG(u.Fiyat)         AS OrtalamaFiyat,
                MIN(u.Fiyat)         AS MinFiyat,
                MAX(u.Fiyat)         AS MaxFiyat,
                SUM(u.StokMiktari)   AS ToplamStok
            FROM Kategoriler k
            INNER JOIN Urunler u ON k.KategoriId = u.KategoriId
            GROUP BY k.KategoriAdi
            ORDER BY UrunSayisi DESC";

        await using var connection = new SqlConnection(_connectionString);
        var sonuclar = await connection.QueryAsync<KategoriRapor>(sql);
        return sonuclar.AsList();
    }

    // HAVING — yalnızca 5'ten fazla siparişi olan müşterileri getir
    public async Task<List<AktifMusteri>> GetAktifMusterilerAsync(int minSiparisSayisi = 5)
    {
        const string sql = @"
            SELECT
                m.MusteriId,
                m.Ad + ' ' + m.Soyad  AS TamAd,
                COUNT(s.SiparisId)     AS SiparisSayisi,
                SUM(s.ToplamTutar)     AS ToplamHarcama
            FROM Musteriler m
            INNER JOIN Siparisler s ON m.MusteriId = s.MusteriId
            WHERE s.Durum = 'Tamamlandi'
            GROUP BY m.MusteriId, m.Ad, m.Soyad
            HAVING COUNT(s.SiparisId) >= @MinSiparisSayisi
            ORDER BY ToplamHarcama DESC";

        await using var connection = new SqlConnection(_connectionString);
        var sonuclar = await connection.QueryAsync<AktifMusteri>(sql,
            new { MinSiparisSayisi = minSiparisSayisi });
        return sonuclar.AsList();
    }

    // Karmaşık GROUP BY — aylık satış özeti
    public async Task<List<AylikSatisOzeti>> GetAylikSatisOzetiAsync(int yil)
    {
        const string sql = @"
            SELECT
                YEAR(SiparisTarihi)  AS Yil,
                MONTH(SiparisTarihi) AS Ay,
                COUNT(*)             AS SiparisSayisi,
                SUM(ToplamTutar)     AS ToplamCiro
            FROM Siparisler
            WHERE YEAR(SiparisTarihi) = @Yil
              AND Durum = 'Tamamlandi'
            GROUP BY YEAR(SiparisTarihi), MONTH(SiparisTarihi)
            HAVING SUM(ToplamTutar) > 10000
            ORDER BY Yil, Ay";

        await using var connection = new SqlConnection(_connectionString);
        var sonuclar = await connection.QueryAsync<AylikSatisOzeti>(sql, new { Yil = yil });
        return sonuclar.AsList();
    }
}

public record KategoriRapor
{
    public string KategoriAdi { get; init; } = string.Empty;
    public int UrunSayisi { get; init; }
    public decimal OrtalamaFiyat { get; init; }
    public decimal MinFiyat { get; init; }
    public decimal MaxFiyat { get; init; }
    public int ToplamStok { get; init; }
}

public record AktifMusteri
{
    public int MusteriId { get; init; }
    public string TamAd { get; init; } = string.Empty;
    public int SiparisSayisi { get; init; }
    public decimal ToplamHarcama { get; init; }
}

public record AylikSatisOzeti(int Yil, int Ay, int SiparisSayisi, decimal ToplamCiro);
```

---

### 4. Stored Procedure nedir ve .NET'te nasıl kullanılır?

**Cevap:**
Stored Procedure (Saklı Yordam), veritabanı sunucusunda önceden derlenmiş ve saklanmış SQL kod bloklarıdır. Avantajları:
- **Performans**: Derleme planı önbelleğe alınır, tekrar derlenmez.
- **Güvenlik**: Tablolara doğrudan erişim yerine SP üzerinden erişim sağlanabilir; SQL injection riski azalır.
- **Bakım kolaylığı**: İş mantığı tek bir yerde toplanır.
- **Ağ trafiği**: Büyük SQL sorguları yerine sadece SP adı ve parametreler gönderilir.

**Örnek Kod:**
```csharp
// Stored Procedure SQL tanımı (referans için):
// CREATE PROCEDURE usp_UrunAra
//     @AramaMetni NVARCHAR(100),
//     @MinFiyat   DECIMAL(18,2) = 0,
//     @MaxFiyat   DECIMAL(18,2) = NULL,
//     @KategoriId INT = NULL
// AS
// BEGIN
//     SET NOCOUNT ON;
//     SELECT u.UrunId, u.UrunAdi, u.Fiyat, k.KategoriAdi
//     FROM Urunler u
//     INNER JOIN Kategoriler k ON u.KategoriId = k.KategoriId
//     WHERE u.UrunAdi LIKE '%' + @AramaMetni + '%'
//       AND u.Fiyat >= @MinFiyat
//       AND (@MaxFiyat IS NULL OR u.Fiyat <= @MaxFiyat)
//       AND (@KategoriId IS NULL OR u.KategoriId = @KategoriId)
// END

// --- ADO.NET ile SP çağrısı ---
public class UrunAdoRepository
{
    private readonly string _connectionString;

    public UrunAdoRepository(string connectionString)
    {
        _connectionString = connectionString;
    }

    public async Task<List<UrunDto>> UrunAraAsync(
        string aramaMetni,
        decimal minFiyat = 0,
        decimal? maxFiyat = null,
        int? kategoriId = null)
    {
        var liste = new List<UrunDto>();

        await using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();

        await using var command = new SqlCommand("usp_UrunAra", connection)
        {
            CommandType = CommandType.StoredProcedure
        };

        command.Parameters.AddWithValue("@AramaMetni", aramaMetni);
        command.Parameters.AddWithValue("@MinFiyat", minFiyat);
        command.Parameters.AddWithValue("@MaxFiyat",
            maxFiyat.HasValue ? (object)maxFiyat.Value : DBNull.Value);
        command.Parameters.AddWithValue("@KategoriId",
            kategoriId.HasValue ? (object)kategoriId.Value : DBNull.Value);

        await using var reader = await command.ExecuteReaderAsync();

        while (await reader.ReadAsync())
        {
            liste.Add(new UrunDto
            {
                UrunId = reader.GetInt32(reader.GetOrdinal("UrunId")),
                UrunAdi = reader.GetString(reader.GetOrdinal("UrunAdi")),
                Fiyat = reader.GetDecimal(reader.GetOrdinal("Fiyat")),
                KategoriAdi = reader.GetString(reader.GetOrdinal("KategoriAdi"))
            });
        }

        return liste;
    }

    // OUTPUT parametresi kullanan SP örneği
    public async Task<int> UrunEkleAsync(UrunEkleDto dto)
    {
        await using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();

        await using var command = new SqlCommand("usp_UrunEkle", connection)
        {
            CommandType = CommandType.StoredProcedure
        };

        command.Parameters.AddWithValue("@UrunAdi", dto.UrunAdi);
        command.Parameters.AddWithValue("@Fiyat", dto.Fiyat);
        command.Parameters.AddWithValue("@KategoriId", dto.KategoriId);

        var outputParam = new SqlParameter("@YeniUrunId", SqlDbType.Int)
        {
            Direction = ParameterDirection.Output
        };
        command.Parameters.Add(outputParam);

        await command.ExecuteNonQueryAsync();

        return (int)outputParam.Value;
    }
}

// --- Dapper ile SP çağrısı ---
public class UrunDapperRepository
{
    private readonly string _connectionString;

    public UrunDapperRepository(string connectionString)
    {
        _connectionString = connectionString;
    }

    public async Task<List<UrunDto>> UrunAraAsync(
        string aramaMetni,
        decimal minFiyat = 0,
        decimal? maxFiyat = null,
        int? kategoriId = null)
    {
        await using var connection = new SqlConnection(_connectionString);

        var parametreler = new DynamicParameters();
        parametreler.Add("@AramaMetni", aramaMetni);
        parametreler.Add("@MinFiyat", minFiyat);
        parametreler.Add("@MaxFiyat", maxFiyat);
        parametreler.Add("@KategoriId", kategoriId);

        var sonuclar = await connection.QueryAsync<UrunDto>(
            "usp_UrunAra",
            parametreler,
            commandType: CommandType.StoredProcedure);

        return sonuclar.AsList();
    }
}

public class UrunDto
{
    public int UrunId { get; set; }
    public string UrunAdi { get; set; } = string.Empty;
    public decimal Fiyat { get; set; }
    public string KategoriAdi { get; set; } = string.Empty;
}

public record UrunEkleDto(string UrunAdi, decimal Fiyat, int KategoriId);
```

---

### 5. Index (İndeks) nedir? Clustered ve Non-Clustered Index arasındaki fark nedir?

**Cevap:**
İndeks, veritabanında sorguların daha hızlı çalışması için oluşturulan bir veri yapısıdır (genellikle B-Tree). Kitabın sonundaki dizine benzer şekilde, veritabanı motorunun tüm tabloyu taramak yerine doğrudan ilgili satırlara ulaşmasını sağlar.

**Clustered Index:**
- Tablodaki satırları fiziksel olarak belirtilen sütuna göre sıralar ve saklar.
- Tablo başına yalnızca bir adet olabilir (PRIMARY KEY varsayılan olarak Clustered Index'tir).
- Aralık sorgularında (BETWEEN, >, <) çok etkilidir.

**Non-Clustered Index:**
- Verinin fiziksel sıralamasını değiştirmez; ayrı bir yapıda tutulur ve Clustered Index'e pointer içerir.
- Tablo başına birden fazla olabilir (SQL Server'da 999'a kadar).
- Belirli sütunlara göre hızlı nokta araması yapmak için kullanılır.

**Örnek Kod:**
```csharp
// İndeks oluşturma SQL'leri (referans):
// -- Clustered Index (PRIMARY KEY ile otomatik oluşur)
// CREATE CLUSTERED INDEX IX_Siparisler_SiparisId ON Siparisler(SiparisId);
//
// -- Non-Clustered Index
// CREATE NONCLUSTERED INDEX IX_Siparisler_MusteriId ON Siparisler(MusteriId);
//
// -- Composite (Bileşik) Index
// CREATE NONCLUSTERED INDEX IX_Siparisler_Durum_Tarih
//     ON Siparisler(Durum, SiparisTarihi DESC);
//
// -- Covering Index (Include ile ek sütunlar dahil et)
// CREATE NONCLUSTERED INDEX IX_Siparisler_MusteriId_Covering
//     ON Siparisler(MusteriId)
//     INCLUDE (ToplamTutar, SiparisTarihi);

// .NET'te indeks kullanımını gösteren Dapper sorgusu
public class IndeksliSiparisRepository
{
    private readonly string _connectionString;

    public IndeksliSiparisRepository(string connectionString)
    {
        _connectionString = connectionString;
    }

    // Bu sorgu IX_Siparisler_MusteriId_Covering indeksini kullanır
    // Index Seek yapılır, tam tablo taraması (Table Scan) yapılmaz
    public async Task<List<SiparisOzet>> GetMusteriSiparisleriAsync(int musteriId)
    {
        const string sql = @"
            SELECT SiparisId, ToplamTutar, SiparisTarihi
            FROM Siparisler
            WHERE MusteriId = @MusteriId
            ORDER BY SiparisTarihi DESC";

        await using var connection = new SqlConnection(_connectionString);
        var sonuclar = await connection.QueryAsync<SiparisOzet>(sql,
            new { MusteriId = musteriId });
        return sonuclar.AsList();
    }

    // Execution plan kontrolü için sorgu ipucu (hint) örneği
    public async Task<List<SiparisOzet>> GetSiparislerWithHintAsync(int musteriId)
    {
        // WITH (INDEX = IX_Siparisler_MusteriId) ipucu ile belirli bir index zorlama
        const string sql = @"
            SELECT SiparisId, ToplamTutar, SiparisTarihi
            FROM Siparisler WITH (INDEX = IX_Siparisler_MusteriId)
            WHERE MusteriId = @MusteriId";

        await using var connection = new SqlConnection(_connectionString);
        var sonuclar = await connection.QueryAsync<SiparisOzet>(sql,
            new { MusteriId = musteriId });
        return sonuclar.AsList();
    }
}

public record SiparisOzet(int SiparisId, decimal ToplamTutar, DateTime SiparisTarihi);
```

---

### 6. ADO.NET ile parametreli sorgu nasıl yazılır? SQL Injection nasıl önlenir?

**Cevap:**
SQL Injection, kullanıcı girdisinin doğrudan SQL sorgusuna eklenmesiyle oluşan kritik bir güvenlik açığıdır. Parametreli sorgular (parameterized queries) kullanmak, bu açığı tamamen engelleyen en temel yöntemdir. Parametreli sorgularda kullanıcı girdisi asla SQL kodu olarak yorumlanmaz; yalnızca değer olarak ele alınır.

**Örnek Kod:**
```csharp
using System.Data;
using Microsoft.Data.SqlClient;

public class GuvenliKullaniciRepository
{
    private readonly string _connectionString;

    public GuvenliKullaniciRepository(string connectionString)
    {
        _connectionString = connectionString;
    }

    // YANLIS — SQL Injection açığı var (asla kullanmayın)
    public async Task<KullaniciDto?> GetKullaniciYanlis_TEHLIKELI(string email)
    {
        // Saldırgan email olarak "' OR '1'='1" girebilir
        string sql = $"SELECT * FROM Kullanicilar WHERE Email = '{email}'";

        await using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();
        await using var command = new SqlCommand(sql, connection);
        // ... tehlikeli kod
        return null;
    }

    // DOGRU — Parametreli sorgu (SQL Injection'a karşı güvenli)
    public async Task<KullaniciDto?> GetKullaniciByEmailAsync(string email)
    {
        const string sql = @"
            SELECT KullaniciId, Ad, Soyad, Email, OlusturmaTarihi
            FROM Kullanicilar
            WHERE Email = @Email AND AktifMi = 1";

        await using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();

        await using var command = new SqlCommand(sql, connection);
        // Parametre tipi ve boyutunu açıkça belirtmek en iyi pratiktir
        command.Parameters.Add(new SqlParameter("@Email", SqlDbType.NVarChar, 256)
        {
            Value = email
        });

        await using var reader = await command.ExecuteReaderAsync();

        if (!await reader.ReadAsync())
            return null;

        return new KullaniciDto
        {
            KullaniciId = reader.GetInt32(0),
            Ad = reader.GetString(1),
            Soyad = reader.GetString(2),
            Email = reader.GetString(3),
            OlusturmaTarihi = reader.GetDateTime(4)
        };
    }

    // Toplu INSERT — SqlParameter koleksiyonu ile
    public async Task<int> KullaniciEkleAsync(KullaniciEkleDto dto)
    {
        const string sql = @"
            INSERT INTO Kullanicilar (Ad, Soyad, Email, SifreHash, OlusturmaTarihi)
            OUTPUT INSERTED.KullaniciId
            VALUES (@Ad, @Soyad, @Email, @SifreHash, @OlusturmaTarihi)";

        await using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();

        await using var command = new SqlCommand(sql, connection);

        command.Parameters.AddRange(new SqlParameter[]
        {
            new("@Ad",             SqlDbType.NVarChar, 50)  { Value = dto.Ad },
            new("@Soyad",          SqlDbType.NVarChar, 50)  { Value = dto.Soyad },
            new("@Email",          SqlDbType.NVarChar, 256) { Value = dto.Email },
            new("@SifreHash",      SqlDbType.NVarChar, 256) { Value = dto.SifreHash },
            new("@OlusturmaTarihi",SqlDbType.DateTime2)     { Value = DateTime.UtcNow }
        });

        var yeniId = await command.ExecuteScalarAsync();
        return Convert.ToInt32(yeniId);
    }
}

public class KullaniciDto
{
    public int KullaniciId { get; set; }
    public string Ad { get; set; } = string.Empty;
    public string Soyad { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public DateTime OlusturmaTarihi { get; set; }
}

public record KullaniciEkleDto(
    string Ad,
    string Soyad,
    string Email,
    string SifreHash);
```

---

### 7. Dapper nedir ve Entity Framework Core'dan ne zaman tercih edilir?

**Cevap:**
Dapper, Stack Overflow ekibi tarafından geliştirilen hafif bir micro-ORM kütüphanesidir. `IDbConnection` arayüzünü genişleten extension metotlar aracılığıyla SQL sorgularını doğrudan C# nesnelerine eşler.

**Dapper'ın avantajları:**
- Entity Framework Core'a kıyasla çok daha hızlıdır (özellikle büyük veri setlerinde).
- SQL üzerinde tam kontrol sağlar.
- Mevcut Stored Procedure ve karmaşık SQL sorgularıyla kolayca çalışır.
- Öğrenmesi ve kullanması basittir.

**EF Core'un avantajları:**
- Change tracking ile nesne değişikliklerini otomatik takip eder.
- Migration desteği sunar.
- LINQ ile tip güvenli sorgulama sağlar.
- CRUD işlemlerinde daha az kod yazılır.

**Genel öneri:** Raporlama, analitik sorgular ve yüksek performans gereken okuma işlemlerinde Dapper; karmaşık nesne grafiği yönetimi ve CRUD ağırlıklı işlemlerde EF Core tercih edilebilir. İkisi aynı projede birlikte kullanılabilir.

**Örnek Kod:**
```csharp
using Dapper;
using Microsoft.Data.SqlClient;

public class UrunDapperService
{
    private readonly string _connectionString;

    public UrunDapperService(string connectionString)
    {
        _connectionString = connectionString;
    }

    // Tekil nesne sorgulama
    public async Task<UrunDetay?> GetUrunByIdAsync(int urunId)
    {
        const string sql = @"
            SELECT u.UrunId, u.UrunAdi, u.Fiyat, u.StokMiktari,
                   k.KategoriId, k.KategoriAdi
            FROM Urunler u
            INNER JOIN Kategoriler k ON u.KategoriId = k.KategoriId
            WHERE u.UrunId = @UrunId";

        await using var connection = new SqlConnection(_connectionString);

        // Multi-mapping: tek sorgudan iki nesne oluştur
        var sonuclar = await connection.QueryAsync<UrunDetay, KategoriDetay, UrunDetay>(
            sql,
            (urun, kategori) =>
            {
                urun.Kategori = kategori;
                return urun;
            },
            new { UrunId = urunId },
            splitOn: "KategoriId");

        return sonuclar.FirstOrDefault();
    }

    // Liste sorgulama
    public async Task<IEnumerable<UrunOzet>> GetAktifUrunlerAsync(decimal minFiyat = 0)
    {
        const string sql = @"
            SELECT UrunId, UrunAdi, Fiyat, StokMiktari
            FROM Urunler
            WHERE AktifMi = 1 AND Fiyat >= @MinFiyat
            ORDER BY Fiyat ASC";

        await using var connection = new SqlConnection(_connectionString);
        return await connection.QueryAsync<UrunOzet>(sql, new { MinFiyat = minFiyat });
    }

    // INSERT / UPDATE / DELETE
    public async Task<int> StokGuncelleAsync(int urunId, int yeniMiktar)
    {
        const string sql = @"
            UPDATE Urunler
            SET StokMiktari = @YeniMiktar,
                GuncellenmeTarihi = GETUTCDATE()
            WHERE UrunId = @UrunId";

        await using var connection = new SqlConnection(_connectionString);
        return await connection.ExecuteAsync(sql,
            new { UrunId = urunId, YeniMiktar = yeniMiktar });
    }

    // QueryMultiple — tek roundtrip'te birden fazla sonuç seti
    public async Task<(List<UrunOzet> Urunler, int ToplamSayfa)> GetSayfalanmisUrunlerAsync(
        int sayfa, int sayfaBoyutu)
    {
        const string sql = @"
            SELECT UrunId, UrunAdi, Fiyat, StokMiktari
            FROM Urunler
            WHERE AktifMi = 1
            ORDER BY UrunAdi
            OFFSET @Offset ROWS FETCH NEXT @SayfaBoyutu ROWS ONLY;

            SELECT CEILING(CAST(COUNT(*) AS FLOAT) / @SayfaBoyutu) AS ToplamSayfa
            FROM Urunler
            WHERE AktifMi = 1;";

        await using var connection = new SqlConnection(_connectionString);

        using var multi = await connection.QueryMultipleAsync(sql, new
        {
            Offset = (sayfa - 1) * sayfaBoyutu,
            SayfaBoyutu = sayfaBoyutu
        });

        var urunler = (await multi.ReadAsync<UrunOzet>()).AsList();
        var toplamSayfa = await multi.ReadSingleAsync<int>();

        return (urunler, toplamSayfa);
    }
}

public class UrunDetay
{
    public int UrunId { get; set; }
    public string UrunAdi { get; set; } = string.Empty;
    public decimal Fiyat { get; set; }
    public int StokMiktari { get; set; }
    public KategoriDetay? Kategori { get; set; }
}

public record KategoriDetay(int KategoriId, string KategoriAdi);
public record UrunOzet(int UrunId, string UrunAdi, decimal Fiyat, int StokMiktari);
```

---

### 8. Transaction (İşlem) yönetimi ADO.NET ile nasıl yapılır?

**Cevap:**
Transaction, birden fazla veritabanı işleminin atomik bir şekilde gerçekleştirilmesini sağlar. ACID özelliklerinden (Atomicity, Consistency, Isolation, Durability) birini destekler. Tüm işlemler başarılı olursa COMMIT, herhangi biri başarısız olursa ROLLBACK yapılır.

**Örnek Kod:**
```csharp
using System.Data;
using Microsoft.Data.SqlClient;
using Dapper;

public class SiparisService
{
    private readonly string _connectionString;

    public SiparisService(string connectionString)
    {
        _connectionString = connectionString;
    }

    // ADO.NET ile manuel transaction yönetimi
    public async Task<int> SiparisOlusturAsync(SiparisOlusturDto dto)
    {
        await using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();

        await using var transaction = await connection.BeginTransactionAsync(
            IsolationLevel.ReadCommitted);

        try
        {
            // 1. Sipariş başlığını ekle
            const string siparisSql = @"
                INSERT INTO Siparisler (MusteriId, SiparisTarihi, Durum, ToplamTutar)
                OUTPUT INSERTED.SiparisId
                VALUES (@MusteriId, @SiparisTarihi, 'Beklemede', @ToplamTutar)";

            await using var siparisCmd = new SqlCommand(siparisSql, connection,
                (SqlTransaction)transaction);

            siparisCmd.Parameters.AddWithValue("@MusteriId", dto.MusteriId);
            siparisCmd.Parameters.AddWithValue("@SiparisTarihi", DateTime.UtcNow);
            siparisCmd.Parameters.AddWithValue("@ToplamTutar", dto.ToplamTutar);

            var siparisId = Convert.ToInt32(await siparisCmd.ExecuteScalarAsync());

            // 2. Sipariş kalemlerini ekle ve stok düş
            foreach (var kalem in dto.Kalemler)
            {
                const string kalemSql = @"
                    INSERT INTO SiparisKalemleri (SiparisId, UrunId, Miktar, BirimFiyat)
                    VALUES (@SiparisId, @UrunId, @Miktar, @BirimFiyat)";

                await using var kalemCmd = new SqlCommand(kalemSql, connection,
                    (SqlTransaction)transaction);

                kalemCmd.Parameters.AddWithValue("@SiparisId", siparisId);
                kalemCmd.Parameters.AddWithValue("@UrunId", kalem.UrunId);
                kalemCmd.Parameters.AddWithValue("@Miktar", kalem.Miktar);
                kalemCmd.Parameters.AddWithValue("@BirimFiyat", kalem.BirimFiyat);

                await kalemCmd.ExecuteNonQueryAsync();

                // Stok güncelle
                const string stokSql = @"
                    UPDATE Urunler
                    SET StokMiktari = StokMiktari - @Miktar
                    WHERE UrunId = @UrunId AND StokMiktari >= @Miktar";

                await using var stokCmd = new SqlCommand(stokSql, connection,
                    (SqlTransaction)transaction);

                stokCmd.Parameters.AddWithValue("@Miktar", kalem.Miktar);
                stokCmd.Parameters.AddWithValue("@UrunId", kalem.UrunId);

                var etkilenenSatir = await stokCmd.ExecuteNonQueryAsync();

                if (etkilenenSatir == 0)
                    throw new InvalidOperationException(
                        $"Ürün {kalem.UrunId} için yeterli stok yok.");
            }

            await transaction.CommitAsync();
            return siparisId;
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }

    // Dapper ile transaction yönetimi
    public async Task<bool> ParaTransferiAsync(
        int kaynakHesapId, int hedefHesapId, decimal tutar)
    {
        await using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();
        await using var transaction = connection.BeginTransaction();

        try
        {
            // Kaynak hesaptan düş
            var etkilenen1 = await connection.ExecuteAsync(@"
                UPDATE Hesaplar SET Bakiye = Bakiye - @Tutar
                WHERE HesapId = @HesapId AND Bakiye >= @Tutar",
                new { HesapId = kaynakHesapId, Tutar = tutar },
                transaction);

            if (etkilenen1 == 0)
                throw new InvalidOperationException("Yetersiz bakiye.");

            // Hedef hesaba ekle
            await connection.ExecuteAsync(@"
                UPDATE Hesaplar SET Bakiye = Bakiye + @Tutar
                WHERE HesapId = @HesapId",
                new { HesapId = hedefHesapId, Tutar = tutar },
                transaction);

            // Transfer kaydını oluştur
            await connection.ExecuteAsync(@"
                INSERT INTO TransferGecmisi
                    (KaynakHesapId, HedefHesapId, Tutar, TransferTarihi)
                VALUES (@KaynakHesapId, @HedefHesapId, @Tutar, @TransferTarihi)",
                new
                {
                    KaynakHesapId = kaynakHesapId,
                    HedefHesapId = hedefHesapId,
                    Tutar = tutar,
                    TransferTarihi = DateTime.UtcNow
                },
                transaction);

            transaction.Commit();
            return true;
        }
        catch
        {
            transaction.Rollback();
            throw;
        }
    }
}

public record SiparisOlusturDto(
    int MusteriId,
    decimal ToplamTutar,
    List<SiparisKalemDto> Kalemler);

public record SiparisKalemDto(int UrunId, int Miktar, decimal BirimFiyat);
```

---

### 9. Subquery (Alt Sorgu) ve CTE (Common Table Expression) nedir?

**Cevap:**
**Subquery (Alt Sorgu):** Bir SQL sorgusunun içine yerleştirilen başka bir sorgudur. `WHERE`, `FROM` veya `SELECT` kısımlarında kullanılabilir.

**CTE (Common Table Expression):** `WITH` anahtar kelimesiyle tanımlanan, geçici ve isimlendirilmiş sonuç kümesidir. Karmaşık sorguları daha okunabilir hale getirir ve recursive (özyinelemeli) sorgulara olanak tanır.

**Örnek Kod:**
```csharp
public class GelismisRaporRepository
{
    private readonly string _connectionString;

    public GelismisRaporRepository(string connectionString)
    {
        _connectionString = connectionString;
    }

    // Subquery örneği — ortalamanın üzerindeki ürünleri bul
    public async Task<List<UrunOzet>> GetOrtalamaUstundeUrunlerAsync()
    {
        const string sql = @"
            SELECT UrunId, UrunAdi, Fiyat, StokMiktari
            FROM Urunler
            WHERE Fiyat > (
                SELECT AVG(Fiyat)
                FROM Urunler
                WHERE AktifMi = 1
            )
            AND AktifMi = 1
            ORDER BY Fiyat DESC";

        await using var connection = new SqlConnection(_connectionString);
        var sonuclar = await connection.QueryAsync<UrunOzet>(sql);
        return sonuclar.AsList();
    }

    // CTE örneği — kategorideki en pahalı ürünü bul
    public async Task<List<KategoriEnPahaliUrun>> GetKategoriEnPahaliUrunAsync()
    {
        const string sql = @"
            WITH KategoriMaxFiyat AS (
                SELECT KategoriId, MAX(Fiyat) AS MaxFiyat
                FROM Urunler
                WHERE AktifMi = 1
                GROUP BY KategoriId
            )
            SELECT k.KategoriAdi, u.UrunAdi, u.Fiyat
            FROM KategoriMaxFiyat kmf
            INNER JOIN Urunler u
                ON kmf.KategoriId = u.KategoriId AND kmf.MaxFiyat = u.Fiyat
            INNER JOIN Kategoriler k
                ON u.KategoriId = k.KategoriId
            ORDER BY u.Fiyat DESC";

        await using var connection = new SqlConnection(_connectionString);
        var sonuclar = await connection.QueryAsync<KategoriEnPahaliUrun>(sql);
        return sonuclar.AsList();
    }

    // Recursive CTE — organizasyon hiyerarşisi
    public async Task<List<CalisanHiyerarsi>> GetOrganizasyonAgaciAsync(int kok)
    {
        const string sql = @"
            WITH OrgAgaci AS (
                -- Başlangıç (anchor) üye
                SELECT CalisanId, Ad, Soyad, YoneticisiId, 0 AS Seviye
                FROM Calisanlar
                WHERE CalisanId = @Kok

                UNION ALL

                -- Recursive üye
                SELECT c.CalisanId, c.Ad, c.Soyad, c.YoneticisiId, oa.Seviye + 1
                FROM Calisanlar c
                INNER JOIN OrgAgaci oa ON c.YoneticisiId = oa.CalisanId
            )
            SELECT CalisanId, Ad, Soyad, YoneticisiId, Seviye
            FROM OrgAgaci
            ORDER BY Seviye, CalisanId";

        await using var connection = new SqlConnection(_connectionString);
        var sonuclar = await connection.QueryAsync<CalisanHiyerarsi>(sql, new { Kok = kok });
        return sonuclar.AsList();
    }
}

public record KategoriEnPahaliUrun(string KategoriAdi, string UrunAdi, decimal Fiyat);
public record CalisanHiyerarsi(
    int CalisanId, string Ad, string Soyad, int? YoneticisiId, int Seviye);
```

---

### 10. Connection Pool nedir ve ADO.NET'te nasıl yönetilir?

**Cevap:**
Connection Pool (Bağlantı Havuzu), veritabanı bağlantılarının yeniden kullanılması için tutulan bir önbellektir. Veritabanı bağlantısı açmak maliyetli bir işlemdir; Connection Pool sayesinde bağlantılar kapatılmak yerine havuza iade edilir ve bir sonraki istek tarafından yeniden kullanılır.

ADO.NET, SQL Server ile çalışırken bağlantı havuzunu varsayılan olarak etkinleştirir. Bağlantı dizesindeki `Max Pool Size`, `Min Pool Size` ve `Connection Timeout` gibi parametrelerle havuz davranışı yapılandırılabilir.

**Örnek Kod:**
```csharp
using Microsoft.Data.SqlClient;
using Microsoft.Extensions.DependencyInjection;

// appsettings.json bağlantı dizesi örneği:
// "Server=.;Database=SatisDB;Integrated Security=true;
//  Min Pool Size=5;Max Pool Size=100;Connection Timeout=30;
//  TrustServerCertificate=true"

// Dependency Injection ile bağlantı yönetimi
public static class DatabaseServiceExtensions
{
    public static IServiceCollection AddDatabaseServices(
        this IServiceCollection services,
        string connectionString)
    {
        // Factory pattern ile bağlantı oluşturma
        services.AddTransient<IDbConnectionFactory>(_ =>
            new SqlConnectionFactory(connectionString));

        return services;
    }
}

public interface IDbConnectionFactory
{
    Task<SqlConnection> CreateOpenConnectionAsync();
}

public class SqlConnectionFactory : IDbConnectionFactory
{
    private readonly string _connectionString;

    public SqlConnectionFactory(string connectionString)
    {
        _connectionString = connectionString;
    }

    public async Task<SqlConnection> CreateOpenConnectionAsync()
    {
        var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();
        return connection;
    }
}

// Doğru kullanım: using ile bağlantıyı mutlaka kapat (havuza iade edilir)
public class OrnekRepository
{
    private readonly IDbConnectionFactory _dbFactory;

    public OrnekRepository(IDbConnectionFactory dbFactory)
    {
        _dbFactory = dbFactory;
    }

    public async Task<List<string>> GetUrunAdlariAsync()
    {
        // using/await using kullanımı bağlantının havuza iadesini garantiler
        await using var connection = await _dbFactory.CreateOpenConnectionAsync();

        var sonuclar = await connection.QueryAsync<string>(
            "SELECT UrunAdi FROM Urunler WHERE AktifMi = 1");

        return sonuclar.AsList();
        // connection Dispose edilirken, bağlantı fiziksel olarak kapanmaz,
        // havuza iade edilir ve bir sonraki çağrıda yeniden kullanılır.
    }
}
```

---

## Özet Tablosu

| Kavram           | Açıklama                                              | Kullanım Amacı                             |
|------------------|-------------------------------------------------------|--------------------------------------------|
| 1NF              | Atomik değerler, benzersiz satırlar                  | Tekrar eden grupları ortadan kaldırma      |
| 2NF              | Kısmi bağımlılık yok                                 | Bileşik anahtar tablolarında veri tutarlılığı |
| 3NF              | Geçişli bağımlılık yok                               | Güncelleme anomalilerini önleme            |
| INNER JOIN       | Eşleşen satırlar                                     | İlişkili kayıtları birleştirme             |
| LEFT JOIN        | Sol tablo + eşleşen sağ                              | Eksik ilişkileri dahil etme                |
| FULL JOIN        | Her iki tablonun tamamı                              | Veri karşılaştırma ve eksik kayıt tespiti  |
| CROSS JOIN       | Kartezyen çarpım                                     | Kombinasyon oluşturma                      |
| GROUP BY         | Satırları gruplama                                   | Agregasyon işlemleri                       |
| HAVING           | Grup koşulu                                          | Agregasyon sonucu filtreleme               |
| Stored Procedure | Sunucuda derlenmiş SQL bloğu                         | Performans ve güvenlik                     |
| Clustered Index  | Fiziksel sıralama ile indeks                        | Aralık sorguları                           |
| Non-Clustered    | Ayrı yapıda indeks                                   | Nokta araması ve covering index            |
| CTE              | Geçici isimli sonuç kümesi                           | Karmaşık sorgular, recursive işlemler      |
| Connection Pool  | Yeniden kullanılabilir bağlantı havuzu               | Performans ve ölçeklenebilirlik            |

## Kaynaklar
- [SQL Server Dokümantasyonu](https://docs.microsoft.com/tr-tr/sql/sql-server/)
- [Dapper GitHub](https://github.com/DapperLib/Dapper)
- [ADO.NET Genel Bakış](https://docs.microsoft.com/tr-tr/dotnet/framework/data/adonet/ado-net-overview)
- [SQL Server Index Tasarımı](https://docs.microsoft.com/tr-tr/sql/relational-databases/indexes/indexes)
