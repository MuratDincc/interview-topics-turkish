# Felaket Kurtarma (Disaster Recovery)

## Genel Bakış

Felaket Kurtarma (Disaster Recovery - DR), bir sistemin beklenmedik arızalar, doğal afetler, siber saldırılar veya insan hataları sonrasında hizmetini yeniden işler hale getirme sürecidir. Senior backend geliştiriciler için DR stratejilerini tasarlamak, uygulamak ve test etmek temel yetkinlikler arasındadır. İş sürekliliğini (Business Continuity) garanti altına alan DR planları; veri kaybını minimize etmek, kesinti sürelerini kısaltmak ve sistemlerin belirlenen hedefler dahilinde kurtarılmasını sağlamak üzere tasarlanır.

Modern bulut tabanlı ve dağıtık sistemlerde DR, yalnızca yedekleme yapmaktan ibaret değildir. Aktif-pasif ve aktif-aktif mimariler, otomatik failover mekanizmaları, sağlık kontrolleri ve kaos mühendisliği gibi konular da DR kapsamında değerlendirilmektedir.

## Kapsanan Konular

### 1. RPO ve RTO
Recovery Point Objective (Kurtarma Noktası Hedefi) ve Recovery Time Objective (Kurtarma Süresi Hedefi) kavramları; DR planlamasının temelini oluşturur. RPO, kabul edilebilir maksimum veri kaybı süresini; RTO ise sistemin kesinti sonrasında yeniden işler hale gelmesi için gereken maksimum süreyi tanımlar.

**Öğrenilecekler:**
- RPO ve RTO tanımları ve hesaplama yöntemleri
- İş gereksinimlerine göre hedef belirleme
- Yedekleme stratejileri (tam, artımlı, diferansiyel)
- Otomatik yedekleme ve geri yükleme pipeline'ları
- Yedek doğrulama ve test prosedürleri
- C# ile health check ve yedekleme otomasyon örnekleri

### 2. Failover Stratejileri
Failover, birincil sistemin devre dışı kalması durumunda trafiğin otomatik ya da manuel olarak yedek sisteme yönlendirilmesi işlemidir. Farklı hazırlık seviyeleri (Hot, Warm, Cold Standby) ve mimariler (Active-Passive, Active-Active) farklı RPO/RTO hedefleri için kullanılır.

**Öğrenilecekler:**
- Active-Passive ve Active-Active mimariler
- Hot, Warm ve Cold Standby yaklaşımları
- Veritabanı failover stratejileri
- Circuit Breaker pattern ve Polly entegrasyonu
- DNS tabanlı ve load balancer tabanlı failover
- C# ile failover implementasyon örnekleri

## Neden Senior Geliştiriciler İçin Önemlidir?

### 1. **İş Sürekliliği Sorumluluğu**
- Kritik sistemlerin kesintisiz çalışması iş geliri ve itibar açısından doğrudan etkilidir.
- SLA (Service Level Agreement) ihlalleri cezai yaptırım ve müşteri kaybına yol açar.
- Senior geliştiriciler DR planı tasarımında ve teknik kararların alınmasında liderlik üstlenir.

### 2. **Mimari Tasarım Kararları**
- Doğru failover stratejisi; maliyet, karmaşıklık ve güvenilirlik dengesini doğrudan etkiler.
- Yanlış RPO/RTO hedefi, yatırım israfına ya da yetersiz korumaya neden olabilir.
- Multi-region ve multi-cloud mimariler DR gereksinimlerini karşılamak üzere tasarlanır.

### 3. **Regülasyon ve Uyumluluk**
- Finans, sağlık ve kamu sektöründe DR planları yasal zorunluluktur.
- GDPR, HIPAA, ISO 27001 gibi standartlar veri kurtarma gereksinimlerini açıkça tanımlar.
- DR testleri belirli aralıklarla yapılmalı ve kayıt altına alınmalıdır.

### 4. **Kaos Mühendisliği ve Test**
- Netflix'in Chaos Monkey gibi araçlarla hataların kasıtlı olarak tetiklenmesi DR olgunluğunu artırır.
- Game Day tatbikatları ekiplerin gerçek kesinti anında doğru davranmasını sağlar.
- Otomatik failover testleri insan hatası riskini azaltır.

### 5. **Maliyet Optimizasyonu**
- Hot standby yüksek maliyet gerektirirken cold standby daha ekonomiktir; doğru seçim kritiktir.
- Bulut sağlayıcıların DR servisleri (AWS Route 53, Azure Site Recovery) doğru kullanıldığında maliyeti optimize eder.
- Aşırı yedekleme yatırımı da kaçınılması gereken bir anti-pattern'dır.

## Temel Kavramlar

| Kavram | Açıklama |
|---|---|
| **RPO** | Kabul edilebilir maksimum veri kaybı süresi |
| **RTO** | Sistemin yeniden işler hale gelmesi için gereken maksimum süre |
| **Failover** | Birincil sistemden yedek sisteme otomatik geçiş |
| **Failback** | Yedek sistemden birincil sisteme dönüş |
| **Hot Standby** | Sürekli senkronize, anlık devreye girebilecek yedek |
| **Warm Standby** | Kısmen aktif, kısa sürede devreye girebilecek yedek |
| **Cold Standby** | Pasif, devreye girmesi daha uzun süren yedek |
| **BCP** | Business Continuity Plan - İş Sürekliliği Planı |
| **DRP** | Disaster Recovery Plan - Felaket Kurtarma Planı |
| **MTTR** | Mean Time To Recovery - Ortalama Kurtarma Süresi |
| **MTBF** | Mean Time Between Failures - Ortalama Arıza Aralığı |

## Temel Mülakat Soruları

### 1. Soru
**RPO ve RTO nedir? Birbirinden farkı nedir?**

**Cevap:** RPO (Recovery Point Objective), bir kesinti sonrasında kabul edilebilir maksimum veri kaybı miktarını zaman cinsinden ifade eder. Örneğin RPO = 1 saat ise sistemin en fazla 1 saatlik verisi kaybedilebilir anlamına gelir; bu da en az saatlik yedekleme yapılması gerektiğine işaret eder. RTO (Recovery Time Objective) ise sistemin kesinti sonrasında yeniden işler hale gelmesi için geçmesi gereken maksimum süreyi tanımlar. Örneğin RTO = 4 saat ise sistem en geç 4 saat içinde ayağa kaldırılmalıdır. RPO veri kaybıyla, RTO ise hizmet kesintisiyle ilgilidir.

### 2. Soru
**Active-Passive ile Active-Active mimariler arasındaki fark nedir?**

**Cevap:** Active-Passive mimaride birincil sistem trafiği karşılarken yedek sistem hazır bekler; birincil sistem devre dışı kalınca failover tetiklenir. Bu yaklaşım daha düşük maliyetlidir ancak failover süresince kısa kesinti yaşanabilir. Active-Active mimaride ise her iki (ya da daha fazla) sistem aynı anda trafik karşılar; bir sistem devre dışı kalınca diğerleri tüm yükü devralır. Bu yaklaşım sıfıra yakın kesinti süresi sağlar ancak veri senkronizasyonu ve maliyet açısından daha karmaşıktır.

### 3. Soru
**Bir DR planı oluştururken hangi adımlar izlenmelidir?**

**Cevap:** DR planı oluştururken izlenmesi gereken adımlar şunlardır: (1) İş etki analizi (BIA) yaparak kritik sistemlerin belirlenmesi, (2) RPO ve RTO hedeflerinin iş gereksinimleriyle uyumlu olarak tanımlanması, (3) Olası felaket senaryolarının (doğal afet, siber saldırı, insan hatası, vb.) listelenmesi, (4) Her senaryo için kurtarma prosedürlerinin detaylandırılması, (5) Rol ve sorumlulukların atanması, (6) İletişim planının oluşturulması, (7) DR planının düzenli aralıklarla test edilmesi ve (8) Test sonuçlarına göre planın güncellenmesi.

### 4. Soru
**Hot, Warm ve Cold Standby arasındaki farklar nelerdir?**

**Cevap:** Hot Standby, birincil sistemle sürekli senkronize çalışan ve anlık failover yapılabilen yedek sistemdir; RTO neredeyse sıfırdır fakat maliyet en yüksektir. Warm Standby, belirli aralıklarla güncellenen ve kısa bir hazırlık süresinden (dakikalar) sonra devreye girebilen yedektir; maliyet ve RTO orta düzeydedir. Cold Standby ise yalnızca yedek verinin saklandığı, doğrudan çalışır durumda olmayan sistemdir; RTO saatler hatta günler düzeyinde olabilir, maliyet ise en düşüktür.

### 5. Soru
**Circuit Breaker pattern DR ile nasıl ilişkilidir?**

**Cevap:** Circuit Breaker pattern, bağımlı bir servis hata vermeye başladığında isteklerin o servise iletilmesini belirli bir süre durdurur ve sistemin çökmesini engeller. Bu sayede bağımlı servis kurtarılırken ana sistem çalışmaya devam edebilir (fallback mekanizması ile). DR bağlamında Circuit Breaker; bir alt sistemin arızasının tüm sistemi etkilemesini önleyerek izolasyonu sağlar ve kısmi DR senaryolarında (bir mikroservisin devre dışı kalması gibi) sistemin genel sağlığını korur.

## Best Practices

### 1. **DR Planını Düzenli Test Edin**
- Yılda en az iki kez tam DR tatbikatı yapın.
- Otomatik failover testlerini CI/CD pipeline'ına entegre edin.
- Test sonuçlarını belgeleyin ve aksiyonları takip edin.

### 2. **İzleme ve Uyarı Sistemleri Kurun**
- Otomatik failover tetikleyicileri için sağlık kontrolleri tanımlayın.
- Kritik metrikleri (uptime, latency, error rate) sürekli izleyin.
- Anormallik tespit edildiğinde ilgili ekipleri otomatik uyarın.

### 3. **Veri Tutarlılığını Garanti Altına Alın**
- Yedeklerinizin düzenli olarak geri yüklenebildiğini doğrulayın.
- Replikasyon gecikmesini (replication lag) izleyin.
- Veri bütünlüğü kontrolleri uygulayın.

### 4. **Belgeleme ve İletişim**
- DR prosedürlerini güncel ve erişilebilir tutun.
- Kriz anında iletişim zincirini önceden tanımlayın.
- Runbook'ları otomatik adımlarla zenginleştirin.

### 5. **Maliyeti Optimum Düzeyde Tutun**
- RPO/RTO hedeflerine göre doğru standby tipini seçin.
- Bulut sağlayıcıların managed DR servislerini değerlendirin.
- Kullanılmayan DR kaynaklarını düzenli olarak gözden geçirin.

## Kaynaklar

- [AWS Disaster Recovery Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-workloads-on-aws.html)
- [Azure Site Recovery](https://docs.microsoft.com/en-us/azure/site-recovery/site-recovery-overview)
- [Microsoft Well-Architected Framework - Reliability](https://docs.microsoft.com/en-us/azure/architecture/framework/resiliency/)
- [Polly - .NET Resilience Library](https://github.com/App-vNext/Polly)
- [Google SRE Book - Disaster Recovery](https://sre.google/sre-book/data-processing-pipelines/)
