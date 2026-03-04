# Cost Optimization & FinOps

## Genel Bakış

Cost Optimization ve FinOps (Financial Operations), bulut tabanlı sistemlerin finansal yönetimini ve maliyet verimliliğini ele alan kritik bir disiplindir. Senior seviye .NET geliştiricileri için bulut maliyetlerini anlamak, optimize etmek ve finansal hesap verebilirlik kültürü oluşturmak vazgeçilmez bir yetkinliktir. FinOps, mühendislik, finans ve iş birimleri arasında köprü kurarak bulut harcamalarının iş değerine dönüştürülmesini sağlar.

Modern bulut mimarilerinde kaynak kullanımı dinamik olduğundan, geleneksel sabit bütçe yönetimi yetersiz kalmaktadır. FinOps çerçevesi; görünürlük, optimizasyon ve işbirliği sütunları üzerine kurularak ekiplerin veri odaklı kararlar almasını mümkün kılar.

## Kapsanan Konular

### 1. Cloud Cost Management

Bulut maliyetlerinin izlenmesi, analiz edilmesi ve optimize edilmesi.

**Öğrenilecekler:**
- Azure Cost Management ve AWS Cost Explorer kullanımı
- Right-sizing stratejileri ile kaynak boyutlandırma
- Reserved Instances ve Savings Plans ile uzun vadeli tasarruf
- Spot/Preemptible instance kullanımı
- Auto-scaling ile dinamik maliyet kontrolü
- Azure SDK ve AWS SDK ile programatik maliyet izleme

### 2. FinOps Pratikleri

FinOps çerçevesi ve organizasyonel maliyet yönetimi.

**Öğrenilecekler:**
- FinOps Foundation çerçevesi (Inform, Optimize, Operate)
- Cost allocation ve chargeback/showback modelleri
- Kaynak etiketleme (tagging) stratejileri
- Bütçe tanımlama ve uyarı mekanizmaları
- Unit economics ve maliyet metrikleri
- C# ile maliyet takip sistemleri geliştirme

## Neden Önemli?

### 1. **İş Değeri ve Rekabet Avantajı**
- Bulut maliyetleri, yazılım geliştirme bütçelerinin önemli bir bölümünü oluşturmaktadır
- Maliyet optimizasyonu, ürün fiyatlandırmasını ve kar marjlarını doğrudan etkiler
- Kaynakların verimli kullanımı, daha hızlı inovasyon kapasitesi sağlar
- Finansal disiplin, kurumsal güvenilirlik ve büyüme sürdürülebilirliği açısından kritiktir

### 2. **Mühendislik Sorumluluğu**
- Bulut harcamaları artık sadece finans departmanının değil, mühendislerin de sorumluluğundadır
- Mimari kararlar (servis seçimi, ölçeklendirme stratejisi) doğrudan maliyet üretir
- Senior geliştiriciler maliyet etkin çözümler tasarlamalıdır
- Infrastructure as Code pratikleri ile maliyet kontrolü mümkün hale gelir

### 3. **Organizasyonel Kültür**
- FinOps, mühendislik ve finans ekipleri arasında ortak bir dil oluşturur
- Şeffaflık ve hesap verebilirlik kültürü geliştirir
- Takımlar harcama kararlarında daha özerk ve bilinçli olur
- Otomatik uyarılar ve dashboardlar ile proaktif yönetim mümkün olur

### 4. **Teknik Mükemmellik**
- Kaynakların doğru boyutlandırılması performans ve maliyet dengesini optimize eder
- Kullanılmayan kaynakların tespit edilmesi teknik borç azaltır
- Maliyet odaklı mimari kararlar sistemin sürdürülebilirliğini artırır
- Otomasyon ile insan hatası kaynaklı israf önlenir

## Mülakat Soruları

### Temel Sorular

1. **FinOps nedir ve neden önemlidir?**
   - **Cevap**: FinOps (Financial Operations), bulut finansal yönetimini bir disiplin olarak ele alan bir çerçevedir. Mühendislik, finans ve iş birimlerini bir araya getirerek bulut harcamalarının iş değerine dönüştürülmesini sağlar. Üç temel sütun üzerine kurulur: Inform (görünürlük), Optimize (iyileştirme) ve Operate (sürekli yönetim).

2. **Right-sizing nedir?**
   - **Cevap**: Right-sizing, bulut kaynaklarının (VM, container, veritabanı) gerçek kullanım ihtiyacına göre yeniden boyutlandırılması sürecidir. Fazla kapasite israfı ve yetersiz kapasite kaynaklı performans sorunlarını önler. CPU, bellek ve ağ metrikleri analiz edilerek optimum boyut belirlenir.

3. **Reserved Instances ile Spot Instances arasındaki fark nedir?**
   - **Cevap**: Reserved Instances, 1 veya 3 yıllık taahhüt karşılığında %30-72 indirim sağlayan sabit kapasiteli ayrımlardır. Spot Instances ise kullanılmayan bulut kapasitesinin açık artırma ile kiralanmasıdır; %60-90 daha ucuz olabilir ancak kapasite geri alınabilir. Kritik iş yükleri için Reserved, hata toleranslı ve kesintili iş yükleri için Spot önerilir.

4. **Cost allocation nedir?**
   - **Cevap**: Cost allocation, bulut maliyetlerinin takımlar, projeler, ürünler veya müşteriler gibi iş birimlerine atanması sürecidir. Kaynak etiketleme (tagging), maliyet merkezi tanımlama ve subscription/account ayrımı gibi yöntemlerle gerçekleştirilir.

5. **Showback ve Chargeback arasındaki fark nedir?**
   - **Cevap**: Showback, maliyetleri iş birimlerine görünür kılar ama fiili ödeme talep etmez; farkındalık yaratmaya yöneliktir. Chargeback ise maliyetleri gerçek anlamda ilgili departmanlara fatura eder. Showback kültür oluşturma aşamasında, chargeback ise maliyetlerin gerçek sahipliğini zorunlu kılmak için kullanılır.

### Teknik Sorular

1. **Azure Cost Management API'si nasıl kullanılır?**
   - **Cevap**: Azure SDK'nın `CostManagementClient` sınıfı ile maliyet verileri sorgulanabilir, bütçe tanımlanabilir ve uyarılar oluşturulabilir. Kimlik doğrulama için `DefaultAzureCredential` kullanılır ve REST API üzerinden `usageDetails`, `budgets` gibi endpoint'lere erişilir.

2. **Auto-scaling maliyet optimizasyonunu nasıl etkiler?**
   - **Cevap**: Auto-scaling, trafik yüküne göre kaynakları dinamik olarak artırıp azaltarak boşta bekleyen kapasite maliyetini ortadan kaldırır. Kubernetes HPA/KEDA, Azure VMSS veya AWS Auto Scaling Group ile saatlik/dakikalık granülaritede ölçeklendirme yapılabilir.

3. **Tagging stratejisi nasıl oluşturulur?**
   - **Cevap**: Etkili bir tagging stratejisi şu etiketleri içermelidir: `environment` (prod/staging/dev), `team`, `project`, `cost-center`, `application`, `owner`. Azure Policy veya AWS Service Control Policies ile etiket zorunluluğu uygulanır. IaC araçları (Terraform, Bicep) ile etiketler otomatik uygulanır.

4. **Kullanılmayan kaynaklar nasıl tespit edilir?**
   - **Cevap**: Azure Advisor veya AWS Trusted Advisor önerileri, düşük CPU/bellek kullanımı olan VM'lerin tespiti, bağlantısız diskler/IP'ler, boş depolama hesapları ve düşük trafik alan load balancer'lar programatik olarak sorgulanabilir. Azure Resource Graph veya AWS Config ile otomatik tarama yapılır.

5. **Unit economics bulut maliyetlerinde nasıl hesaplanır?**
   - **Cevap**: Unit economics, maliyet başına iş değerini ölçer. Örneğin; API isteği başına maliyet, müşteri başına maliyet veya işlem başına maliyet gibi metrikler hesaplanır. Bu metrikler ürün kararlarını yönlendirmek ve ROI hesaplamak için kritiktir.

## Best Practices

### 1. **Görünürlük ve Şeffaflık**
- Tüm kaynaklar için tutarlı etiketleme politikası uygulayın
- Gerçek zamanlı maliyet dashboardları oluşturun
- Ekiplere kendi harcamalarını görebilecekleri erişim verin
- Anomali tespiti için uyarılar tanımlayın

### 2. **Optimizasyon**
- Düzenli right-sizing analizi yapın (aylık önerilir)
- Uzun vadeli iş yükleri için Reserved Instance satın alın
- Dev/Test ortamlarını mesai saatleri dışında kapatın
- Lifecycle policy ile depolama maliyetlerini azaltın

### 3. **Organizasyonel Kültür**
- Mühendisleri maliyet konusunda eğitin ve sahiplik verin
- Sprint review'lere maliyet metriklerini dahil edin
- Maliyet hedeflerini teknik OKR'larla ilişkilendirin
- Başarılı optimizasyonları takım içinde kutlayın

### 4. **Otomasyon**
- IaC ile kaynak oluşturma ve silme işlemlerini otomatize edin
- Otomatik kapatma/açma politikaları uygulayın
- Bütçe aşımlarında otomatik bildirim ve aksiyon tetikleyin
- Maliyet verilerini CI/CD pipeline'larına entegre edin

## Kaynaklar

- [FinOps Foundation](https://www.finops.org/)
- [Azure Cost Management](https://docs.microsoft.com/tr-tr/azure/cost-management-billing/)
- [AWS Cost Optimization](https://aws.amazon.com/aws-cost-management/)
- [Google Cloud Cost Management](https://cloud.google.com/cost-management)
- [FinOps Certified Practitioner](https://www.finops.org/certification/)
- [Azure Pricing Calculator](https://azure.microsoft.com/tr-tr/pricing/calculator/)
- [Well-Architected Framework - Cost Optimization](https://docs.microsoft.com/tr-tr/azure/architecture/framework/cost/)
