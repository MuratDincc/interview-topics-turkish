# Mimari Karar Kayıtları (Architecture Decision Records)

## Genel Bakış

Mimari Karar Kayıtları (Architecture Decision Records - ADR), yazılım sistemlerinde alınan önemli mimari kararların belgelenmesi için kullanılan hafif ağırlıklı bir dokümantasyon tekniğidir. ADR'ler, bir kararın neden alındığını, hangi alternatiflerin değerlendirildiğini ve bu kararın sistemin geleceği üzerindeki etkilerini kayıt altına alır. Senior düzey geliştiriciler için ADR yazma ve kullanma becerisi, teknik liderlik, mimari yönetim ve kurumsal bellek oluşturma açısından kritik öneme sahiptir.

## Kapsanan Konular

### 1. ADR Yazımı (ADR Writing)
ADR şablonu, ADR numaralandırma sistemleri, araçlar ve ne zaman ADR yazılması gerektiği.

**Öğrenilecekler:**
- ADR şablon yapısı (Başlık, Durum, Bağlam, Karar, Sonuçlar)
- ADR numaralandırma ve yönetim stratejileri
- MADR, Nygard ve diğer ADR formatları
- ADR araçları ve otomasyonu (adr-tools, Log4brains, ADR Manager)
- Ne zaman ADR yazılmalı, ne zaman yazılmamalı
- ADR'lerin kod tabanıyla ilişkisi ve C# implementasyon etkileri
- ADR yaşam döngüsü yönetimi

### 2. Ödün Analizi (Trade-off Analysis)
ATAM yöntemi, kalite nitelikleri, maliyet-fayda analizi ve karar matrisleri.

**Öğrenilecekler:**
- ATAM (Architecture Tradeoff Analysis Method) metodolojisi
- Kalite nitelikleri (performans, güvenlik, sürdürülebilirlik, ölçeklenebilirlik)
- Maliyet-fayda analizi teknikleri
- Karar matrisi oluşturma ve ağırlıklandırma
- C# kod örnekleriyle mimari ödün gösterimi
- Risk ve hassasiyet noktalarının belirlenmesi
- Paydaş perspektiflerinin entegrasyonu

## Neden Önemli?

### 1. **Kurumsal Bellek Korunması**
- Ekip değişimlerinde mimari bilginin sürekliliği
- "Neden böyle yaptık?" sorularına hızlı yanıt
- Yeni ekip üyelerinin onboarding sürecinin kısaltılması
- Geçmiş kararlardan ders çıkarma kapasitesi
- Tekrar eden tartışmaların önlenmesi

### 2. **Mimari Yönetim ve Kalite**
- Tutarlı mimari kararların güvence altına alınması
- Standartların ve prensiplerin belgelenmesi
- Mimari borç (architectural debt) takibi
- Gözden geçirme ve iyileştirme süreçlerinin kolaylaştırılması
- Mimari uyumluluk denetimi

### 3. **Ekip İletişimi ve İşbirliği**
- Teknik kararların paydaşlara açıklanması
- Geliştirici ekipleri arasında ortak anlayış
- Tasarım toplantılarında yapısal tartışma zemini
- Uzak ekipler arası asenkron iletişim desteği
- Kod incelemelerinde (code review) referans noktası

### 4. **Risk Yönetimi**
- Alternatiflerin sistematik değerlendirilmesi
- Öngörülemeyen risklerin belgelenmesi
- Geri dönüş stratejilerinin planlanması
- Ödünlerin (trade-off) açıkça görünür kılınması
- Mimari kararların izlenebilirliği

## Mülakat Soruları

### Temel Sorular

1. **ADR nedir ve neden kullanılır?**
   - **Cevap**: ADR (Architecture Decision Record), önemli mimari kararları bağlamı, gerekçesi ve sonuçlarıyla birlikte belgeleyen hafif ağırlıklı dokümantasyon yöntemidir. Kurumsal bellek oluşturur, ekip iletişimini geliştirir ve mimari yönetimi destekler.

2. **ADR hangi bileşenlerden oluşur?**
   - **Cevap**: Temel bileşenler — Başlık (Title), Durum (Status), Bağlam (Context), Karar (Decision) ve Sonuçlar (Consequences). Bazı şablonlar Alternatifler (Alternatives) ve Katılımcılar (Participants) bölümlerini de ekler.

3. **ADR ile RFC arasındaki fark nedir?**
   - **Cevap**: RFC (Request for Comments) bir karar alınmadan önce önerileri toplamak için kullanılır; ADR ise bir karar alındıktan sonra o kararı belgelemek için kullanılır. ADR daha kalıcı ve arşivleme odaklıdır.

4. **Trade-off analizi nedir?**
   - **Cevap**: Bir mimari kararın farklı kalite nitelikleri (performans, güvenlik, sürdürülebilirlik, maliyet) arasındaki dengeyi sistematik olarak değerlendirme sürecidir. Hiçbir mimari karar tüm nitelikleri aynı anda optimize edemez; ödünler açıkça belirlenmeli ve belgelenmelidir.

5. **ATAM nedir?**
   - **Cevap**: Architecture Tradeoff Analysis Method, bir mimarinin kalite nitelikleri üzerindeki etkisini değerlendirmek için kullanılan sistematik bir yöntemdir. Risk noktalarını, hassasiyet noktalarını ve ödünleri tespit eder.

6. **Ne tür kararlar için ADR yazılmalıdır?**
   - **Cevap**: Geri döndürülmesi zor veya maliyetli olan, birden fazla bileşeni etkileyen, önemli ödünler içeren veya ekip içinde tartışma yaratan kararlar için ADR yazılmalıdır. Sıradan tasarım kararları için ADR gerekmez.

### Teknik Sorular

1. **ADR numaralandırma sistemi nasıl tasarlanır?**
   - **Cevap**: Genellikle sıralı sayısal ID (0001, 0002) kullanılır. Monorepo'larda proje öneki (api-0001, db-0001) eklenebilir. Numaralar hiçbir zaman değiştirilmemeli; iptal edilen ADR'ler için yeni bir ADR yazılmalıdır.

2. **ADR'lerin durumu (status) nasıl yönetilir?**
   - **Cevap**: Tipik durumlar: Proposed (önerildi), Accepted (kabul edildi), Deprecated (eskidi), Superseded (yenisiyle değiştirildi). Superseded durumundaki ADR hangi ADR tarafından değiştirildiğine referans vermelidir.

3. **Karar matrisi nasıl oluşturulur?**
   - **Cevap**: Alternatifler satırlara, değerlendirme kriterleri sütunlara yerleştirilir. Her kriter için ağırlık belirlenir, her alternatif her kritere göre puanlanır ve ağırlıklı toplam hesaplanır. Sonuç hem nicel hem de nitel değerlendirmeyle desteklenir.

## Best Practices

### 1. **ADR Yazım Prensipleri**
- ADR'leri olabildiğince kısa ve odaklı tutun
- Kararı değil, kararın gerekçesini açıklayın
- Değerlendirilen alternatifleri mutlaka belgeleyin
- Kararın ileride nasıl gözden geçirileceğini belirtin
- Teknik olmayan paydaşların da anlayabileceği bir dil kullanın

### 2. **Trade-off Analizi Prensipleri**
- Kalite niteliklerini önceden tanımlayın ve önceliklendirin
- Ödünleri açık ve dürüst biçimde belgeleyin
- Nicel verilerle desteklenen kararlar verin
- Kısa vadeli ve uzun vadeli etkileri birlikte değerlendirin
- Paydaşların perspektiflerini analiz sürecine dahil edin

### 3. **Araç ve Süreç**
- ADR'leri kod tabanıyla birlikte versiyonlayın (docs/adr/ klasörü)
- Ekip standartları için tek bir şablon belirleyin
- ADR incelemelerini kod inceleme süreciyle entegre edin
- Periyodik ADR gözden geçirme toplantıları planlayın
- Eskiyen ADR'leri güncel tutun veya Deprecated olarak işaretleyin

## Kaynaklar

- [ADR GitHub Organization](https://adr.github.io/)
- [Michael Nygard - ADR Template](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions)
- [MADR - Markdown Architectural Decision Records](https://adr.github.io/madr/)
- [ATAM - SEI Carnegie Mellon](https://resources.sei.cmu.edu/library/asset-view.cfm?assetid=513908)
- [Documenting Software Architectures - Bass et al.](https://www.informit.com/store/documenting-software-architectures-views-and-beyond-9780321552686)
- [Microsoft Architecture Decision Records Guidance](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/)
