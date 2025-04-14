# DateTime İşlemleri

## Genel Bakış

DateTime işlemleri, tarih ve saat verilerini manipüle etmek için kullanılan temel operasyonlardır. C#'ta System.DateTime ve System.TimeSpan sınıfları kullanılarak tarih ve saat işlemleri gerçekleştirilir.

## Temel DateTime İşlemleri

1. **DateTime Oluşturma**
   ```csharp
   // Şu anki tarih ve saat
   DateTime now = DateTime.Now;
   
   // UTC tarih ve saat
   DateTime utcNow = DateTime.UtcNow;
   
   // Belirli bir tarih
   DateTime specificDate = new DateTime(2024, 1, 1);
   
   // Belirli bir tarih ve saat
   DateTime specificDateTime = new DateTime(2024, 1, 1, 12, 30, 0);
   ```

2. **DateTime Formatlama**
   ```csharp
   DateTime now = DateTime.Now;
   
   // Kısa tarih
   string shortDate = now.ToShortDateString(); // "01.01.2024"
   
   // Uzun tarih
   string longDate = now.ToLongDateString(); // "1 Ocak 2024 Pazartesi"
   
   // Kısa saat
   string shortTime = now.ToShortTimeString(); // "12:30"
   
   // Uzun saat
   string longTime = now.ToLongTimeString(); // "12:30:45"
   
   // Custom format
   string customFormat = now.ToString("dd.MM.yyyy HH:mm:ss");
   ```

3. **DateTime Karşılaştırma**
   ```csharp
   DateTime date1 = new DateTime(2024, 1, 1);
   DateTime date2 = new DateTime(2024, 1, 2);
   
   // Eşitlik kontrolü
   bool isEqual = date1 == date2;
   
   // Büyük/küçük kontrolü
   bool isGreater = date1 > date2;
   
   // CompareTo
   int result = date1.CompareTo(date2);
   ```

4. **DateTime Manipülasyonu**
   ```csharp
   DateTime date = new DateTime(2024, 1, 1);
   
   // Gün ekleme
   DateTime tomorrow = date.AddDays(1);
   
   // Ay ekleme
   DateTime nextMonth = date.AddMonths(1);
   
   // Yıl ekleme
   DateTime nextYear = date.AddYears(1);
   
   // Saat ekleme
   DateTime nextHour = date.AddHours(1);
   ```

## TimeSpan İşlemleri

1. **TimeSpan Oluşturma**
   ```csharp
   // Belirli bir süre
   TimeSpan duration = new TimeSpan(1, 30, 0); // 1 saat 30 dakika
   
   // İki tarih arasındaki fark
   DateTime start = new DateTime(2024, 1, 1);
   DateTime end = new DateTime(2024, 1, 2);
   TimeSpan difference = end - start;
   ```

2. **TimeSpan Özellikleri**
   ```csharp
   TimeSpan duration = new TimeSpan(1, 30, 0);
   
   int hours = duration.Hours; // 1
   int minutes = duration.Minutes; // 30
   int seconds = duration.Seconds; // 0
   double totalHours = duration.TotalHours; // 1.5
   ```

## DateTime ve TimeZone

1. **TimeZone Dönüşümleri**
   ```csharp
   DateTime utcNow = DateTime.UtcNow;
   
   // UTC'den local'e
   DateTime localTime = utcNow.ToLocalTime();
   
   // Local'den UTC'ye
   DateTime utcTime = localTime.ToUniversalTime();
   ```

2. **TimeZone Bilgisi**
   ```csharp
   // Sistem timezone'u
   TimeZoneInfo localZone = TimeZoneInfo.Local;
   
   // Belirli bir timezone
   TimeZoneInfo istanbulZone = TimeZoneInfo.FindSystemTimeZoneById("Turkey Standard Time");
   
   // Timezone dönüşümü
   DateTime istanbulTime = TimeZoneInfo.ConvertTime(DateTime.Now, istanbulZone);
   ```

## DateTime ve Kültür

1. **Kültüre Özgü Formatlama**
   ```csharp
   DateTime now = DateTime.Now;
   
   // Türkçe format
   string turkishFormat = now.ToString("d", new CultureInfo("tr-TR"));
   
   // İngilizce format
   string englishFormat = now.ToString("d", new CultureInfo("en-US"));
   ```

2. **Kültüre Özgü Tarih Ayrıştırma**
   ```csharp
   string dateString = "01.01.2024";
   DateTime date = DateTime.Parse(dateString, new CultureInfo("tr-TR"));
   ```

## DateTime Validasyonu

1. **Geçerlilik Kontrolü**
   ```csharp
   string dateString = "31.02.2024";
   DateTime date;
   bool isValid = DateTime.TryParse(dateString, out date);
   ```

2. **Tarih Aralığı Kontrolü**
   ```csharp
   DateTime date = new DateTime(2024, 1, 1);
   DateTime minDate = new DateTime(2023, 1, 1);
   DateTime maxDate = new DateTime(2025, 1, 1);
   
   bool isInRange = date >= minDate && date <= maxDate;
   ```

## Mülakat Soruları

1. **DateTime Temelleri**
   - DateTime ve DateTimeOffset arasındaki farklar nelerdir?
   - DateTime.Now ve DateTime.UtcNow arasındaki fark nedir?
   - DateTime'ın precision (hassasiyet) değeri nedir?

2. **TimeZone İşlemleri**
   - TimeZoneInfo sınıfı ne işe yarar?
   - Daylight Saving Time (Yaz Saati) nedir ve nasıl yönetilir?
   - TimeZone dönüşümlerinde dikkat edilmesi gerekenler nelerdir?

3. **DateTime Formatlama**
   - Custom DateTime formatları nasıl oluşturulur?
   - Kültüre özgü DateTime formatlaması nasıl yapılır?
   - DateTime formatlamada performans optimizasyonu nasıl yapılır?

4. **DateTime Manipülasyonu**
   - DateTime.AddXXX() metodları ne zaman kullanılmalıdır?
   - DateTime aritmetik işlemlerinde dikkat edilmesi gerekenler nelerdir?
   - DateTime değerlerinin karşılaştırılmasında best practices nelerdir?

5. **TimeSpan İşlemleri**
   - TimeSpan ne zaman kullanılmalıdır?
   - TimeSpan ve DateTime arasındaki ilişki nedir?
   - TimeSpan formatlaması nasıl yapılır?

6. **DateTime Validasyonu**
   - DateTime.TryParse() ne zaman kullanılmalıdır?
   - DateTime validasyonunda dikkat edilmesi gerekenler nelerdir?
   - DateTime aralık kontrolleri nasıl yapılır?

7. **DateTime ve Veritabanı**
   - DateTime değerleri veritabanında nasıl saklanmalıdır?
   - DateTime ve SQL arasındaki dönüşümler nasıl yapılır?
   - DateTime değerlerinin veritabanında indexlenmesi nasıl yapılır?

8. **DateTime ve API'ler**
   - API'lerde DateTime değerleri nasıl formatlanmalıdır?
   - API'lerde TimeZone bilgisi nasıl yönetilmelidir?
   - API'lerde DateTime validasyonu nasıl yapılmalıdır?

9. **DateTime ve Performans**
   - DateTime işlemlerinde performans optimizasyonu nasıl yapılır?
   - DateTime.Parse() vs DateTime.TryParse() performans karşılaştırması nasıldır?
   - DateTime formatlamada string pooling nasıl kullanılır?

10. **DateTime ve Güvenlik**
    - DateTime değerlerinde güvenlik açıkları nasıl önlenir?
    - DateTime injection nedir ve nasıl önlenir?
    - DateTime değerlerinin loglanmasında dikkat edilmesi gerekenler nelerdir?

## Örnek Kod Soruları

1. **Tarih Farkı Hesaplama**
   ```csharp
   public int CalculateDaysBetween(DateTime startDate, DateTime endDate)
   {
       // Implementasyon
   }
   ```

2. **Yaş Hesaplama**
   ```csharp
   public int CalculateAge(DateTime birthDate)
   {
       // Implementasyon
   }
   ```

3. **İş Günü Hesaplama**
   ```csharp
   public int CalculateBusinessDays(DateTime startDate, DateTime endDate)
   {
       // Implementasyon (hafta sonları ve resmi tatiller hariç)
   }
   ```

4. **Tarih Formatlama**
   ```csharp
   public string FormatDate(DateTime date, string culture)
   {
       // Implementasyon
   }
   ```

5. **TimeZone Dönüşümü**
   ```csharp
   public DateTime ConvertTimeZone(DateTime date, string sourceZone, string targetZone)
   {
       // Implementasyon
   }
   ```

## Kaynaklar

- [Microsoft Docs - DateTime](https://docs.microsoft.com/en-us/dotnet/api/system.datetime)
- [DateTime Formatting](https://docs.microsoft.com/en-us/dotnet/standard/base-types/custom-date-and-time-format-strings)
- [TimeZoneInfo Class](https://docs.microsoft.com/en-us/dotnet/api/system.timezoneinfo) 