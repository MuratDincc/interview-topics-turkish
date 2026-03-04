# Feature Flags & A/B Testing

## Genel Bakış

Feature Flags (özellik bayrakları) ve A/B Testing, modern yazılım geliştirme süreçlerinin vazgeçilmez parçalarıdır. Feature flags, kodun belirli bölümlerini çalışma zamanında etkinleştirip devre dışı bırakmanıza olanak tanır. A/B Testing ise farklı kullanıcı gruplarına farklı deneyimler sunarak veri odaklı kararlar almanızı sağlar. Bu bölüm, .NET ekosisteminde bu tekniklerin nasıl uygulanacağını kapsamlı bir şekilde ele alır.

## Kapsanan Konular

### 1. Feature Management

Microsoft'un resmi `Microsoft.FeatureManagement` kütüphanesi ile feature flag yönetimi.

**Öğrenilecekler:**
- `IFeatureManager` arayüzü ve kullanımı
- `IFeatureDefinitionProvider` ile özel tanımlar
- Feature filter kavramı ve yerleşik filtreler
- ASP.NET Core ile Dependency Injection entegrasyonu
- Feature flag'lerin appsettings.json ile konfigürasyonu
- Controller ve Action seviyesinde feature gate kullanımı
- Razor view'larında feature flag kontrolü

### 2. Gradual Rollout & A/B Testing

Kademeli yayılım stratejileri ve A/B test implementasyonları.

**Öğrenilecekler:**
- Yüzde tabanlı kademeli açılım (percentage-based rollout)
- Kullanıcı hedefleme ve segmentasyon
- Canary release deseni
- A/B test sonuçlarının toplanması ve analizi
- Feature flag'lerin izlenmesi ve metrik entegrasyonu
- Azure App Configuration ile bulut tabanlı flag yönetimi
- LaunchDarkly gibi üçüncü taraf entegrasyonları

## Neden Önemli?

### 1. **Güvenli Dağıtım**
- Yeni özellikleri üretim ortamında küçük kullanıcı gruplarıyla test etme
- Sorunlu özellikleri kod dağıtımı yapmadan anında devre dışı bırakma
- Trunk-based development ile sürekli entegrasyon desteği
- Deployment ile release süreçlerini birbirinden ayırma

### 2. **Veri Odaklı Karar Alma**
- A/B testleri ile hangi tasarım veya davranışın daha iyi çalıştığını ölçme
- Kullanıcı davranışı verilerine dayalı ürün geliştirme
- Hipotez tabanlı geliştirme süreçleri
- Dönüşüm oranı ve kullanıcı memnuniyeti metrikleri

### 3. **Operasyonel Esneklik**
- Bakım pencereleri için özellikleri geçici olarak devre dışı bırakma
- Farklı müşteri segmentlerine özel özellik sunma
- Abonelik veya lisansa göre özellik erişim kontrolü
- Bölgesel veya ortam bazlı özellik yönetimi

### 4. **Azaltılmış Risk**
- Büyük özellikleri küçük parçalar halinde yayınlama
- Hızlı geri alma (rollback) kapasitesi
- Üretim ortamında gerçek kullanıcı verileriyle doğrulama
- Teknik borcu kontrollü bir şekilde yönetme

## Temel Kavramlar

### Feature Flag Türleri

| Tür | Açıklama | Kullanım Alanı |
|-----|----------|----------------|
| Release Flags | Tamamlanmamış özellikleri gizler | Trunk-based geliştirme |
| Experiment Flags | A/B testleri için kullanılır | Ürün optimizasyonu |
| Ops Flags | Operasyonel kontrol sağlar | Kill switch, bakım modu |
| Permission Flags | Erişim kontrolü | Premium özellikler |

### Temel Bileşenler

```
+------------------+       +-------------------+       +------------------+
|   Feature Store  | ----> |  Feature Manager  | ----> |  Feature Gates   |
| (Config/DB/API)  |       | (IFeatureManager) |       | (Controller/View)|
+------------------+       +-------------------+       +------------------+
                                    |
                           +--------+--------+
                           |                 |
                  +--------+------+  +-------+-------+
                  | Feature       |  | Feature       |
                  | Filters       |  | Providers     |
                  | (Percentage,  |  | (Custom,      |
                  |  TimeWindow,  |  |  AppConfig)   |
                  |  Targeting)   |  |               |
                  +---------------+  +---------------+
```

## Mülakat Soruları

### 1. Soru

**Feature flag nedir ve neden kullanılır?**

**Cevap:** Feature flag (özellik bayrağı), belirli bir kod parçasını veya uygulama özelliğini çalışma zamanında açıp kapatmayı sağlayan bir mekanizmadır. Temel avantajları şunlardır: yeni özellikleri kademeli olarak kullanıcılara açma, sorunlu bir özelliği kod dağıtımına gerek kalmadan anında devre dışı bırakma, A/B testleri ile veri odaklı karar alma ve deployment ile feature release süreçlerini birbirinden ayırma. Bu yaklaşım, Continuous Delivery (CD) pratiklerini güçlendirir ve ekiplerin daha güvenle kod dağıtmasını sağlar.

### 2. Soru

**A/B Testing ile Feature Flag arasındaki fark nedir?**

**Cevap:** Feature flag genel bir mekanizma iken A/B testing bu mekanizmanın özel bir kullanım şeklidir. Feature flag; bir özelliği açık/kapalı tutmak, belirli kullanıcılara açmak veya kademeli yayılım yapmak gibi çok sayıda amaçla kullanılabilir. A/B testing ise kullanıcı tabanını rastgele iki veya daha fazla gruba böler, her gruba farklı bir deneyim sunar ve hangi versiyonun daha iyi performans gösterdiğini istatistiksel yöntemlerle ölçer. Kısacası, her A/B testi bir feature flag kullanır; ancak her feature flag bir A/B testi değildir.

### 3. Soru

**Microsoft.FeatureManagement kütüphanesinin temel bileşenleri nelerdir?**

**Cevap:** Kütüphanenin temel bileşenleri şunlardır:
- **IFeatureManager**: Feature flag durumunu sorgulayan merkezi arayüz
- **IFeatureDefinitionProvider**: Flag tanımlarının nereden okunacağını belirler (appsettings, veritabanı vb.)
- **IFeatureFilter**: Bir flag'in etkin olup olmayacağını belirleyen filtre mantığı (yüzde, zaman penceresi, kullanıcı hedefleme)
- **FeatureGate Attribute**: Controller action'larını flag durumuna göre kısıtlayan attribute
- **IVariantFeatureManager**: A/B testing için farklı varyantlar sunan genişletilmiş arayüz

### 4. Soru

**Canary release ile blue-green deployment arasındaki fark nedir?**

**Cevap:** Blue-green deployment'ta iki özdeş üretim ortamı bulunur; trafik anlık olarak bir ortamdan diğerine yönlendirilir. Sorun durumunda eski ortama anında geri dönülür. Canary release ise yeni sürümü tüm kullanıcılara birden sunmak yerine trafiğin küçük bir yüzdesini (örneğin %5) yeni sürüme yönlendirir, izler ve sorun yoksa yüzdeyi kademeli olarak artırır. Feature flag'ler canary release'i altyapı değişikliğine gerek kalmadan uygulama katmanında gerçekleştirmeye olanak tanır.

### 5. Soru

**Feature flag'lerin sakıncaları veya riskleri nelerdir?**

**Cevap:** Feature flag'lerin yanlış yönetilmesi çeşitli sorunlara yol açabilir: Teknik borç birikimi (eski, kullanılmayan flag'ler kodda kalır ve karmaşıklık yaratır), test kombinasyonlarının artması (N flag için 2^N test senaryosu oluşabilir), flag'lerin birbirine bağımlı hale gelmesi, yapılandırma karmaşıklığı ve yanlış yapılandırma durumunda üretim hatası riski. Bu nedenle flag'lerin düzenli olarak temizlenmesi (flag hygiene), yaşam döngüsü yönetimi ve kapsamlı test stratejileri kritik öneme sahiptir.

## Best Practices

### 1. **Flag Yaşam Döngüsü Yönetimi**
- Her flag için bir sahip (owner) belirleyin
- Flag'lere son kullanma tarihi tanımlayın
- Kullanılmayan flag'leri düzenli olarak kaldırın
- Flag'lerin amacını ve durumunu dokümante edin

### 2. **Isimlendirme Konvansiyonları**
- Açıklayıcı ve tutarlı isimler kullanın: `NewCheckoutFlow`, `EnableDarkMode`
- Tür bazlı prefix kullanmayı düşünün: `Experiment_`, `Release_`, `Ops_`
- Boolean flag adlarında "Enable" veya "Use" prefix'i tercih edin

### 3. **Test Stratejisi**
- Her flag durumu için ayrı test senaryoları yazın
- Integration test'lerde flag'leri kontrol edin
- Flag kombinasyonlarını test edin

### 4. **Güvenlik ve Erişim**
- Flag konfigürasyonuna erişimi kısıtlayın
- Hassas flag değişikliklerini audit log'a kaydedin
- Üretim ortamındaki flag değişikliklerini onay sürecine bağlayın

## Kaynaklar

- [Microsoft.FeatureManagement Dokümantasyonu](https://docs.microsoft.com/en-us/azure/azure-app-configuration/feature-management-overview)
- [Azure App Configuration Feature Flags](https://docs.microsoft.com/en-us/azure/azure-app-configuration/concept-feature-management)
- [Martin Fowler - Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
- [LaunchDarkly Feature Flag Guide](https://launchdarkly.com/blog/what-are-feature-flags/)
