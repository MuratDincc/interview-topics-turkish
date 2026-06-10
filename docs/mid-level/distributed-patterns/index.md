# Dağıtık Sistem Desenleri (Distributed Patterns)

## Giriş

Mikroservisler ve dağıtık sistemler, monolitik mimarinin sağladığı **tek veritabanı / tek transaction** garantisini kaybeder. Birden fazla servis, birden fazla veritabanı ve ağ üzerinden haberleşme devreye girince; "ya hep ya hiç" (atomicity) ve "anlık tutarlılık" (strong consistency) artık ücretsiz değildir.

Bu bölüm, dağıtık sistemlerde **veri tutarlılığını** ve **mesaj güvenilirliğini** sağlamak için kullanılan en sık sorulan desenleri kapsar. Mid-level ve senior mülakatlarda bu konular, "iki servis arasında parayı nasıl tutarlı transfer edersin?" gibi sorularla doğrudan test edilir.

## Kapsanan Konular

### 1. Outbox Pattern (Transactional Outbox)
Veritabanı değişikliği ile mesaj yayınlamayı **atomik** hale getirme. "Dual write problem"i çözer.

**Öğrenilecekler:**
- Dual write problem nedir?
- Transactional Outbox tablosu
- Outbox processor (polling / CDC)
- Inbox pattern ve deduplication
- At-least-once teslimat

→ [Outbox Pattern](outbox-pattern.md)

### 2. Saga Pattern
Birden fazla servise yayılan iş akışlarını (long-running transaction) yönetme ve hata durumunda geri alma (compensation).

**Öğrenilecekler:**
- Orchestration vs Choreography
- Compensating transactions
- Saga state yönetimi
- MassTransit ile saga

→ [Saga Pattern](saga-pattern.md)

### 3. Eventual Consistency
CAP teoremi, güçlü tutarlılık (strong) vs nihai tutarlılık (eventual) ve iş gereksinimlerine göre seçim.

**Öğrenilecekler:**
- CAP ve PACELC teoremi
- Strong vs eventual consistency
- Read-your-writes, monotonic reads
- Tutarlılık penceresi (consistency window)

→ [Eventual Consistency](eventual-consistency.md)

### 4. Dağıtık Transaction'lar (2PC ve Alternatifleri)
Two-Phase Commit neden kaçınılır, compensating transaction ile nasıl değiştirilir.

**Öğrenilecekler:**
- Two-Phase Commit (2PC) ve dezavantajları
- Compensating transaction
- TCC (Try-Confirm/Cancel)
- Saga ile karşılaştırma

→ [Dağıtık Transaction'lar](distributed-transactions.md)

## İlişkili Konular

Bu desenler tek başına çalışmaz; aşağıdaki konularla birlikte değerlendirilmelidir:

- [Idempotency](../../senior/advanced-system-design/idempotency.md) — Mesajların tekrar işlenmesine karşı koruma (at-least-once teslimatın olmazsa olmazı).
- [Distributed Locking](../architecture/distributed-locking.md) — Dağıtık ortamda kilitleme.
- [Circuit Breaker](../microservices/circuit-breaker.md) — Hata toleransı ve dayanıklılık.
- [Event Sourcing](../microservices/event-sourcing.md) — Olay tabanlı durum yönetimi.
- [Message Queue](../message-queue/index.md) — RabbitMQ / Kafka ile asenkron iletişim.

## Neden Önemli?

> **Mülakat ipucu:** "İki mikroservis arasında veriyi nasıl tutarlı tutarsın?" sorusunun doğru cevabı genellikle **"dağıtık transaction kullanmam"** ile başlar. Bunun yerine: Outbox ile mesajı güvenilir yayınlarım, Saga ile iş akışını yönetirim, Idempotency ile tekrarları tolere ederim ve sistem **eventual consistent** olur.

Modern .NET dağıtık sistemlerinde (RabbitMQ/Kafka + EF Core) bu desenler standart araç kutusudur. Bir senior adayından, ağ kopması veya servis çökmesi anında verinin **kaybolmayacağını veya çift işlenmeyeceğini** garanti eden bir tasarım çizmesi beklenir.
