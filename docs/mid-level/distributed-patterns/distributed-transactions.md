# Dağıtık Transaction'lar (2PC ve Alternatifleri)

## Genel Bakış

Tek bir veritabanında ACID transaction işimizi görür: `BEGIN ... COMMIT/ROLLBACK` ile ya hep ya hiç garantisi alırız. Ancak işlem **birden fazla servis veya veritabanına** yayıldığında, bu garantiyi sağlamak zorlaşır. Bu sayfa, dağıtık transaction yaklaşımlarını — özellikle **Two-Phase Commit (2PC)** ve neden mikroservislerde kaçınıldığını, yerine ne kullanıldığını — ele alır.

## Two-Phase Commit (2PC)

2PC, bir **coordinator** (transaction manager) ve birden fazla **participant** arasında atomik commit sağlayan klasik protokoldür. İki aşamadan oluşur:

```
Aşama 1 — PREPARE (Voting)
  Coordinator → tüm participant'lara: "Commit'e hazır mısın?"
  Her participant: işi yapar, kilitler, log'lar → "Yes" / "No" oyu

Aşama 2 — COMMIT / ABORT
  Hepsi "Yes" dediyse  → Coordinator: "Commit!"  → herkes commit eder
  Biri "No" dediyse    → Coordinator: "Abort!"   → herkes rollback eder
```

### 2PC'nin Sorunları

2PC teoride atomiklik sağlar ama mikroservislerde pratikte **kaçınılır**:

1. **Bloklama (Blocking):** Participant'lar "prepare"den sonra "commit/abort" emrini beklerken **kaynakları kilitli tutar**. Coordinator çökerse, participant'lar belirsizlikte (in-doubt) takılır ve kilitler açılmaz → kullanılabilirlik çöker.
2. **Coordinator tek hata noktası:** Coordinator çökerse tüm transaction askıda kalır.
3. **Ölçeklenmezlik:** Senkron kilitler ve çok turlu mesajlaşma, yüksek gecikme ve düşük throughput demektir.
4. **Heterojenlik:** Modern broker'lar (Kafka, RabbitMQ) ve birçok NoSQL veritabanı XA/2PC desteklemez.

> **Mülakat ipucu:** "Mikroservislerde iki DB'yi nasıl tek transaction'da güncellersin?" sorusuna "distributed transaction / 2PC kullanırım" demek genelde **zayıf** bir cevaptır. Güçlü cevap: "2PC'nin bloklama ve ölçeklenme sorunları nedeniyle kaçınırım; bunun yerine Saga + Outbox + Idempotency ile eventual consistency sağlarım."

## Alternatif 1: Saga Pattern

En yaygın modern alternatif. Tek bir global transaction yerine, her servis **kendi lokal transaction'ını** yapar; hata olursa **compensating transaction** ile geri alınır. Kilit tutmaz, ölçeklenir, ama **eventual consistency** sağlar.

```
2PC:   tek atomik transaction, kilitli, strong consistency, ölçeklenmez
Saga:  lokal transaction'lar + telafi, kilitsiz, eventual consistency, ölçeklenir
```

→ Detay: [Saga Pattern](saga-pattern.md)

## Alternatif 2: TCC (Try-Confirm/Cancel)

2PC'nin uygulama seviyesindeki, kilitsiz bir varyantı. Her servis üç operasyon sunar:

- **Try:** Kaynağı **rezerve et** (ör. bakiyeyi düş değil, "blokaj" koy), ama kesinleştirme.
- **Confirm:** Rezervasyonu kesinleştir (idempotent).
- **Cancel:** Rezervasyonu serbest bırak (idempotent).

```csharp
public interface IAccountService
{
    Task<bool> TryReserve(Guid txId, decimal amount);  // bakiyeyi "blokaj"a al
    Task Confirm(Guid txId);                            // blokajı düş
    Task Cancel(Guid txId);                             // blokajı geri ver
}
```

Akış: Tüm servislerde `Try` başarılıysa → hepsine `Confirm`; biri başarısızsa → hepsine `Cancel`. 2PC'den farkı: veritabanı kilidi yerine **iş seviyesinde rezervasyon** kullanır, bu yüzden uzun süre kilit tutmaz.

## Alternatif 3: Outbox + Idempotency

Çoğu durumda "atomik dağıtık transaction" ihtiyacı aslında **"veriyi yaz ve event'i güvenilir yayınla"** ihtiyacıdır. Bunun için tam bir distributed transaction gerekmez:

- **Outbox Pattern** ile DB yazma + event yayınlama atomik olur. → [Outbox Pattern](outbox-pattern.md)
- **Idempotency** ile tekrarlanan mesajlar güvenle işlenir. → [Idempotency](../../senior/advanced-system-design/idempotency.md)

## Karşılaştırma Tablosu

| Yaklaşım | Tutarlılık | Kilitleme | Ölçeklenme | Karmaşıklık | Modern kullanım |
|----------|------------|-----------|------------|-------------|-----------------|
| **2PC / XA** | Strong | Yüksek (bloklar) | Zayıf | Orta | Nadir (legacy, tek vendor) |
| **Saga** | Eventual | Yok | İyi | Yüksek | Yaygın |
| **TCC** | Eventual (kısa pencere) | İş seviyesi rezervasyon | İyi | Yüksek | Orta (finans) |
| **Outbox + Idempotency** | Eventual | Yok | İyi | Orta | Çok yaygın |

## Ne Zaman 2PC Hâlâ Uygundur?

2PC tamamen ölü değildir. **Tek bir güvenilir altyapı içinde**, düşük katılımcı sayısı ve düşük gecikme toleransı olan senaryolarda kullanılır:

- Aynı veritabanı motorunun birden çok şeması/sunucusu arasında (XA).
- Bir DB + bir mesaj broker'ın aynı transaction manager altında olduğu kapalı kurumsal sistemler (örn. eski Java EE / MSDTC).
- Servis sayısının az, güvenilirliğin yüksek ve trafiğin düşük olduğu durumlar.

Mikroservis mimarisinde, bağımsız ölçeklenebilirlik ve dayanıklılık öncelikliyse 2PC genellikle yanlış araçtır.

## Mülakat Soruları

### 1. Soru: "2PC'nin en büyük problemi nedir?"

**Cevap:** Bloklama. Participant'lar "prepare" aşamasından sonra kaynakları kilitler ve coordinator'ın commit/abort kararını bekler. Coordinator bu arada çökerse, participant'lar "in-doubt" durumda kilitleri açamadan takılır; bu da kullanılabilirliği ve throughput'u ciddi düşürür. Ayrıca coordinator tek hata noktasıdır.

### 2. Soru: "Saga, 2PC'nin sağladığı atomikliği sağlar mı?"

**Cevap:** Tam anlamıyla hayır. 2PC tüm sistem için anlık (atomic + isolated) bir commit verir. Saga ise her adımı ayrı commit eder ve hata durumunda telafi (compensation) uygular — yani sistem geçici olarak tutarsız (eventual consistent) olabilir ve klasik izolasyon yoktur. Karşılığında kilit tutmaz ve ölçeklenir.

### 3. Soru: "TCC ile 2PC arasındaki fark nedir?"

**Cevap:** İkisi de iki aşamalıdır, ama 2PC veritabanı seviyesinde kilit tutarken TCC **iş/uygulama seviyesinde rezervasyon** yapar (Try). TCC'de "prepare" yerine kaynağı blokaja alan bir Try, "commit" yerine Confirm, "abort" yerine Cancel vardır. Bu sayede uzun süreli DB kilidi olmaz; ama her servisin üç idempotent operasyonu sunması gerekir.

### 4. Soru: "İki mikroservis arasında para transferini nasıl tasarlarsın?"

**Cevap:** 2PC yerine Saga kullanırım: (1) Kaynak hesaptan düş (lokal transaction), (2) Hedef hesaba ekle (lokal transaction). İkinci adım başarısızsa birinci adımı telafi ederim (parayı geri ekle). Event'leri **Outbox** ile güvenilir yayınlar, tüketicileri **idempotent** yaparım. Sonuç eventual consistent olur; kullanıcıya "transfer işleniyor" durumu gösteririm. Kritik invariant (negatif bakiye olmaması) her lokal transaction içinde koşullu güncellemeyle korunur.

## Best Practices

- Mikroservislerde **varsayılan olarak 2PC'den kaçın**; Saga / Outbox / TCC değerlendir.
- "Atomik dağıtık transaction" ihtiyacının çoğu aslında **güvenilir event yayını**dır → Outbox yeterli olabilir.
- 2PC kullanacaksan **kapalı, az katılımcılı, tek vendor** ortamla sınırla.
- Tüm alternatiflerde **idempotency** olmazsa olmazdır.
- İş kuralının gerçekten strong consistency mi yoksa eventual mı gerektirdiğini **önce netleştir**.

## İlişkili Konular

- [Saga Pattern](saga-pattern.md)
- [Outbox Pattern](outbox-pattern.md)
- [Eventual Consistency](eventual-consistency.md)
- [Idempotency](../../senior/advanced-system-design/idempotency.md)
- [Distributed Locking](../architecture/distributed-locking.md)
- [Distributed Transactions (EF Core)](../entity-framework/distributed-transactions.md)

## Kaynaklar

- "Designing Data-Intensive Applications" — Martin Kleppmann (Bölüm 9: Consistency & Consensus)
- Microservices.io — Saga Pattern
- Pat Helland — "Life Beyond Distributed Transactions"
