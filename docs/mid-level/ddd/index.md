# Domain-Driven Design (DDD)

## Genel Bakış

Domain-Driven Design (DDD), karmaşık yazılım sistemlerini iş alanının (domain) kavramları, kuralları ve süreçleri etrafında modelleme felsefesidir. Eric Evans'ın 2003 yılında yayımladığı "Domain-Driven Design: Tackling Complexity in the Heart of Software" kitabıyla sistematik bir şekilde tanımlanan bu yaklaşım, yazılımın gerçek iş problemlerini çözmesini sağlamak için domain uzmanları ile yazılımcılar arasında ortak bir dil (Ubiquitous Language) oluşturmayı hedefler. Mid-level geliştiriciler için DDD, microservices mimarisi, clean architecture ve enterprise uygulamalar geliştirirken kritik öneme sahiptir. Bu bölüm, Aggregate Root, Bounded Context ve Domain Events konularını kapsar.

## Kapsanan Konular

### 1. Aggregate Root
Aggregate, birlikte değişen ve bir bütün olarak ele alınan nesneler kümesidir. Aggregate Root ise bu kümenin dışarıdan erişilebilen tek giriş noktasıdır.

**Öğrenilecekler:**
- Aggregate ve Aggregate Root kavramları
- Entity ve Value Object ayrımı
- Aggregate sınırlarının belirlenmesi
- Consistency boundary yönetimi
- C# ile Aggregate Root implementasyonu
- Repository pattern ile Aggregate erişimi

### 2. Bounded Context
Bounded Context, belirli bir domain modelinin geçerli olduğu açıkça tanımlanmış sınırlardır. Büyük sistemleri daha küçük, bağımsız alt sistemlere ayırmayı sağlar.

**Öğrenilecekler:**
- Bounded Context tanımlama ve sınır belirleme
- Context Mapping stratejileri
- Anti-Corruption Layer (ACL) implementasyonu
- Shared Kernel ve Customer/Supplier ilişkileri
- Bounded Context'ler arası iletişim
- Microservices ile Bounded Context ilişkisi

### 3. Domain Events
Domain Events, domain içinde gerçekleşen önemli iş olaylarını temsil eden nesnelerdir. Loosely coupled sistem tasarımını destekler ve side effect'lerin yönetimini kolaylaştırır.

**Öğrenilecekler:**
- Domain Event kavramı ve önemi
- Event raising ve handling mekanizmaları
- MediatR ile Domain Event implementasyonu
- Event sourcing temelleri
- Eventual consistency
- Domain Events ile Aggregate Root entegrasyonu

## Neden Önemli?

### 1. **İş Karmaşıklığını Yönetme**
- Karmaşık iş kurallarını yazılıma doğrudan yansıtır
- Domain uzmanlarıyla ortak dil (Ubiquitous Language) oluşturur
- İş gereksinimlerini kod yapısına taşır
- Yazılımın iş değişikliklerine uyum sağlamasını kolaylaştırır

### 2. **Mimari Kalite**
- Modüler, sürdürülebilir sistem tasarımı
- Loosely coupled bileşenler
- High cohesion, low coupling prensiplerine uyum
- Microservices mimarisine doğal uyum

### 3. **Takım Verimliliği**
- Domain uzmanları ve yazılımcılar arasında etkin iletişim
- Ortak terminoloji ile yanlış anlamaların azalması
- Takım içi bilgi paylaşımının kolaylaşması
- Paralel geliştirme imkanı

### 4. **Test Edilebilirlik**
- İş mantığının altyapıdan ayrılması
- Unit test yazımının kolaylaşması
- Domain logic'in izole test edilebilmesi
- Regression riskinin azalması

### 5. **Uzun Vadeli Sürdürülebilirlik**
- Teknik borç birikiminin önlenmesi
- Yeni gereksinimlere hızlı adaptasyon
- Kodun self-documenting nitelik kazanması
- Refactoring maliyetinin düşmesi

## DDD Yapı Taşları

### Temel Kavramlar

| Kavram | Açıklama |
|--------|----------|
| **Entity** | Kimliği (ID) ile tanımlanan, yaşam döngüsü olan nesne |
| **Value Object** | Kimliği olmayan, değerleri ile tanımlanan değişmez nesne |
| **Aggregate** | Birlikte değişen, tutarlı nesneler kümesi |
| **Aggregate Root** | Aggregate'in dış dünyaya açılan tek giriş noktası |
| **Domain Event** | Domain'de gerçekleşen önemli iş olayı |
| **Repository** | Aggregate'lerin saklanması ve geri alınması için soyutlama |
| **Domain Service** | Birden fazla Aggregate'i kapsayan iş mantığı |
| **Factory** | Karmaşık Aggregate ve Entity oluşturma mantığı |

### Stratejik DDD

| Kavram | Açıklama |
|--------|----------|
| **Bounded Context** | Domain modelinin geçerli olduğu açık sınır |
| **Ubiquitous Language** | Domain uzmanları ve geliştiriciler arasında ortak dil |
| **Context Map** | Bounded Context'ler arası ilişkilerin haritası |
| **Core Domain** | İşin rekabet avantajını sağlayan kritik alan |
| **Supporting Domain** | Core Domain'i destekleyen yan alanlar |
| **Generic Domain** | Off-the-shelf çözümlerle karşılanabilecek genel alanlar |

## Mülakat Soruları

### Temel Sorular

1. **DDD nedir ve ne zaman kullanılmalıdır?**
   - **Cevap**: DDD, karmaşık iş domainlerini modelleme yaklaşımıdır. Karmaşık iş kuralları olan, domain uzmanlarıyla yakın çalışma gerektiren, uzun vadeli bakımı düşünülen projelerde kullanılmalıdır. Basit CRUD uygulamalarında overkill olabilir.

2. **Ubiquitous Language nedir?**
   - **Cevap**: Domain uzmanları ve yazılımcıların kullandığı ortak dildir. Kod, belgeler ve konuşmalarda aynı terimlerin kullanılmasını sağlar. Yanlış anlamaları önler ve iletişimi güçlendirir.

3. **Entity ile Value Object arasındaki fark nedir?**
   - **Cevap**: Entity, benzersiz bir kimliğe (ID) sahip ve zamanla değişebilen nesnedir. Value Object ise kimliği olmayan, değerleriyle tanımlanan ve değişmez (immutable) nesnedir. Örneğin `Order` bir Entity, `Money` bir Value Object'tir.

4. **Aggregate nedir?**
   - **Cevap**: Aggregate, birlikte değişen ve bir bütün olarak ele alınan nesneler kümesidir. Veri tutarlılığı (consistency) sınırını tanımlar. Tüm değişiklikler Aggregate Root üzerinden yapılır.

5. **Bounded Context neden gereklidir?**
   - **Cevap**: Büyük sistemlerde aynı kavram farklı alt sistemlerde farklı anlam taşıyabilir (örn. "Müşteri" satış departmanında farklı, destek departmanında farklı tanımlanır). Bounded Context, her alt sistemin kendi tutarlı modelini korumasını sağlar.

### Teknik Sorular

1. **Aggregate Root nasıl tasarlanır?**
   - **Cevap**: Aggregate Root, dış dünyaya açılan tek giriş noktasıdır. Child entity'lere doğrudan erişim kısıtlanır, tüm işlemler root üzerinden yapılır. Aggregate sınırları consistency gereksinimleri gözetilerek belirlenir.

2. **Domain Event ne zaman kullanılır?**
   - **Cevap**: Bir Aggregate'deki önemli iş olaylarını diğer Aggregate veya servislere bildirmek gerektiğinde kullanılır. Loosely coupled tasarım ve eventual consistency sağlar.

3. **Anti-Corruption Layer nedir?**
   - **Cevap**: İki Bounded Context arasına konulan, farklı domain modellerini birbirine dönüştüren çeviri katmanıdır. Dış sistemlerin iç modeli bozmasını engeller.

4. **Repository pattern DDD'de nasıl kullanılır?**
   - **Cevap**: Repository, yalnızca Aggregate Root için tanımlanır. Child entity'lere ayrı repository oluşturulmaz. Aggregate'i bir bütün olarak yükler ve kaydeder.

5. **DDD ile microservices arasındaki ilişki nedir?**
   - **Cevap**: Her Bounded Context, bir microservice'e iyi bir aday olabilir. Ancak birebir eşleme zorunlu değildir. Bounded Context, servis sınırlarını belirlemek için güçlü bir rehberdir.
