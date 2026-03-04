# Technical Debt & Migration

## Genel Bakış

Technical Debt (Teknik Borç), yazılım geliştirme sürecinde hız veya kolaylık uğruna alınan kısa vadeli kararların uzun vadede yarattığı ek maliyet ve karmaşıklıktır. Kötü tasarlanmış mimari, eski teknoloji yığınları, yetersiz test kapsamı ve belgelenmemiş sistemler teknik borcun başlıca kaynakları arasında yer alır. Senior düzeyde bir geliştirici için teknik borcu tespit etmek, ölçmek, yönetmek ve sistematik biçimde azaltmak kritik bir yetkinliktir.

Modern yazılım sistemlerinde migration (göç), legacy (eski) sistemleri modernize etmek, .NET Framework'ten .NET 8'e geçiş yapmak ya da monolitik yapıları mikroservis mimarisine dönüştürmek gibi büyük çaplı değişiklikleri kapsar. Bu dönüşümler dikkatli planlama, doğru pattern seçimi ve aşamalı uygulama stratejileri gerektirir.

## Kapsanan Konular

### 1. Strangler Fig Pattern
Mevcut sistemleri yıkmadan aşamalı olarak yeni sistemlere geçiş stratejisi.

**Öğrenilecekler:**
- Facade ve reverse proxy implementasyonu
- Eski ve yeni sistem arasında trafik yönlendirme
- API Gateway entegrasyonu
- Özellik bazlı (feature-by-feature) migration süreci
- Geri alma (rollback) stratejileri

### 2. Legacy Modernization
.NET Framework'ten .NET 8'e geçiş ve eski sistemlerin modernizasyonu.

**Öğrenilecekler:**
- .NET upgrade araçları ve stratejileri
- Monolitten mikroservise dönüşüm adımları
- Veritabanı migration stratejileri
- Anti-corruption layer (ACL) implementasyonu
- Big Bang vs. aşamalı migrasyon yaklaşımları

### 3. Technical Debt Yönetimi
Teknik borcun ölçülmesi, önceliklendirilmesi ve giderilmesi.

**Öğrenilecekler:**
- Teknik borç tespiti ve ölçüm metrikleri
- Kod kalitesi araçları (SonarQube, NDepend)
- Refactoring stratejileri
- Test coverage artırma yaklaşımları
- Ekip ile teknik borç iletişimi

### 4. Veritabanı Migration Stratejileri
Büyük ölçekli veritabanı değişikliklerinin yönetimi.

**Öğrenilecekler:**
- Zero-downtime migration teknikleri
- Expand-contract pattern
- Blue-green deployment ile veritabanı geçişleri
- Veri doğrulama ve geri alma planları
- Schema versioning

### 5. Anti-Corruption Layer (ACL)
Eski ve yeni sistemler arasında izolasyon katmanı tasarımı.

**Öğrenilecekler:**
- ACL tasarım prensipleri
- Domain model çevirisi
- Integration event kullanımı
- Bounded context sınırlarının korunması
- Performans ve bakım maliyetleri

## Neden Önemli?

### 1. **İş Sürekliliği**
- Production sistemleri çalışmaya devam ederken dönüşüm sağlanır
- Müşteri etkisi minimize edilir
- Risk aşamalı olarak yönetilir
- Geri alma planları her aşamada mevcuttur
- Yatırımın korunması sağlanır

### 2. **Geliştirici Verimliliği**
- Teknik borç yeni özellik geliştirmeyi yavaşlatır
- Modernize edilmiş kod tabanı daha hızlı iteration sağlar
- Yeni geliştiricilerin onboarding süresi kısalır
- Debugging ve sorun giderme kolaylaşır
- Otomatik test kapsamı artar

### 3. **Sistem Güvenilirliği**
- Eski bağımlılıkların güvenlik açıkları kapatılır
- Güncel framework desteği ve yamalar alınır
- Performans iyileştirmeleri sağlanır
- Monitoring ve observability kapasitesi artar
- Kapasite planlaması daha kolay hale gelir

### 4. **Maliyet Optimizasyonu**
- Lisans maliyetleri azaltılır (ör. eski SQL Server sürümleri)
- Cloud-native mimari ile altyapı maliyetleri düşürülür
- Bakım maliyetleri uzun vadede azalır
- Yeni yetenekli geliştiriciler çekilebilir
- Teknik borcun faizi birikimine son verilir

## Temel Mülakat Soruları

### 1. Technical debt nedir ve nasıl ölçülür?
**Cevap:** Technical debt, doğru tasarım yerine hızlı çözüm tercih edildiğinde biriken ek refactoring maliyetidir. Martin Fowler'ın tanımıyla "bilinçli" ve "bilinçsiz" olmak üzere ikiye ayrılır. Ölçüm için SonarQube teknik borç raporu, code complexity metrikleri (döngüsel karmaşıklık), test coverage yüzdesi, duplicate code oranı ve açık bug/vulnerability sayısı kullanılır.

### 2. Strangler Fig Pattern ile Big Bang migration arasındaki fark nedir?
**Cevap:** Big Bang migration tüm sistemi bir seferde değiştirir — yüksek risk, kısa geçiş süresi. Strangler Fig ise eski sistemi işlevsellik bazında kademeli olarak yeni sistemle değiştirerek riski dağıtır. Production sistemin kesintisiz çalışmasını sağlar ve her aşamada geri alma imkânı sunar.

### 3. .NET Framework'ten .NET 8'e geçişte karşılaşılan en büyük zorluklar nelerdir?
**Cevap:** WCF/Remoting gibi platform-specific API'lerin kaldırılması, üçüncü parti paketlerin .NET 8 desteği, Web Forms ve MVC arasındaki farklar, Global.asax startup yapısından Program.cs minimal API'ye geçiş, bağımlılık enjeksiyonu paradigması değişikliği ve asenkron programlama modelindeki farklılıklar başlıca zorluklardır.

### 4. Expand-Contract pattern veritabanı geçişlerinde nasıl uygulanır?
**Cevap:** Önce yeni kolon/tablo eklenir (expand), eski ve yeni yapı bir süre paralel çalışır, uygulama kodu yeni yapıya geçirilir, eski yapı kaldırılır (contract). Bu yaklaşım zero-downtime sağlar ve geri alma kolaylığı sunar.

### 5. Anti-Corruption Layer (ACL) ne zaman gereklidir?
**Cevap:** Eski bir bounded context ile yeni bir domain modeli arasındaki terminoloji veya veri yapısı farkı büyük olduğunda ACL gereklidir. ACL, eski sistemin modelini yeni sistemin domain diline çevirir, böylece yeni kodun eski sistemin kötü tasarımıyla kirlenmesi önlenir.

## Best Practices

### 1. **Migration Planlaması**
- Küçük, bağımsız adımlara bölün
- Her adım için başarı kriterleri tanımlayın
- Geri alma planı her aşama için hazırlayın
- Production'da feature flag kullanarak kontrollü rollout yapın
- Migration öncesi ve sonrası performans metriklerini karşılaştırın

### 2. **Teknik Borç Yönetimi**
- Ekip içinde görünür kılın (backlog'a alın, sprint'e dahil edin)
- Ürün sahibiyle iş değeri bazında önceliklendirin
- "Boy Scout Kuralı": Kodu bulduğunuzdan daha temiz bırakın
- Teknik borç için sprint kapasitesinin %20'sini ayırın
- Otomatik kod kalitesi kontrollerini CI/CD pipeline'a entegre edin

### 3. **Ekip ve İletişim**
- Teknik borcu iş etkisi diliyle anlatın
- Görsel metrikler ve dashboard kullanın
- Refactoring çalışmalarını küçük PR'larla yapın
- Pair programming ile bilgi yayın
- Post-mortem kültürüyle teknik borcun kökenlerine bakın

## Kaynaklar

- [Martin Fowler - Technical Debt](https://martinfowler.com/bliki/TechnicalDebt.html)
- [Strangler Fig Application](https://martinfowler.com/bliki/StranglerFigApplication.html)
- [Microsoft .NET Upgrade Assistant](https://dotnet.microsoft.com/en-us/platform/upgrade-assistant)
- [Microsoft Migration Guides](https://docs.microsoft.com/en-us/dotnet/core/porting/)
- [Anti-Corruption Layer Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/anti-corruption-layer)
- [Expand-Contract Pattern](https://martinfowler.com/bliki/ParallelChange.html)
