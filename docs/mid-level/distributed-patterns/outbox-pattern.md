# Outbox Pattern (Transactional Outbox)

## Genel Bakış

Outbox Pattern, bir mikroservisin **veritabanına yazma** ile **mesaj/event yayınlama** işlemlerini tek bir atomik birim haline getiren bir tasarım desenidir. Amaç, "dual write problem" olarak bilinen tutarsızlık kaynağını ortadan kaldırmaktır.

## Dual Write Problem (Çift Yazma Sorunu)

Tipik bir senaryo: Sipariş oluştur ve `OrderCreated` event'ini RabbitMQ'ya yayınla.

```csharp
// ❌ HATALI: İki ayrı sistem, atomik değil
public async Task CreateOrderAsync(Order order)
{
    await _dbContext.Orders.AddAsync(order);
    await _dbContext.SaveChangesAsync();        // 1) DB commit oldu

    await _messageBus.PublishAsync(             // 2) Ya burada uygulama çökerse?
        new OrderCreated(order.Id));
}
```

Bu kodda iki bağımsız sistem var: **veritabanı** ve **message broker**. İkisi arasında atomiklik garantisi yoktur:

- **Senaryo A:** DB commit oldu, ama `Publish` öncesi uygulama çöktü → Sipariş var ama event yok. Diğer servisler haberdar olmaz. **Mesaj kaybı.**
- **Senaryo B:** Event yayınlandı, ama DB transaction rollback oldu → Olmayan bir sipariş için event gönderildi. **Hayalet event.**

İki işlemin sırasını değiştirmek de çözmez; problemi yer değiştirir. Distributed transaction (2PC) ise broker'lar genelde desteklemez ve ölçeklenmez.

## Çözüm: Outbox Tablosu

Fikir basit: Event'i broker'a göndermek yerine, **aynı veritabanı transaction'ı içinde** bir `Outbox` tablosuna yaz. Böylece sipariş ve event ya birlikte commit olur ya da birlikte rollback olur — **tek transaction, tek atomiklik**.

Ardından ayrı bir arka plan süreci (Outbox Processor) bu tabloyu okuyup mesajları broker'a yayınlar.

```
┌─────────────────────────────────────────┐
│  Tek DB Transaction (atomik)            │
│   ┌──────────────┐   ┌────────────────┐ │
│   │ Orders tablo │   │  Outbox tablo  │ │
│   └──────────────┘   └────────────────┘ │
└─────────────────────────────────────────┘
                            │
                            ▼ (ayrı süreç, polling/CDC)
                   ┌─────────────────┐
                   │  Message Broker │  (RabbitMQ / Kafka)
                   └─────────────────┘
```

### Outbox Tablosu (Entity)

```csharp
public class OutboxMessage
{
    public Guid Id { get; set; }
    public string Type { get; set; } = default!;        // event tipi (OrderCreated)
    public string Payload { get; set; } = default!;     // JSON serialize edilmiş içerik
    public DateTime OccurredOnUtc { get; set; }
    public DateTime? ProcessedOnUtc { get; set; }        // null ise henüz yayınlanmadı
    public string? Error { get; set; }
    public int RetryCount { get; set; }
}
```

### Atomik Yazma

```csharp
// ✅ DOĞRU: Sipariş ve event aynı transaction'da
public async Task CreateOrderAsync(Order order)
{
    await _dbContext.Orders.AddAsync(order);

    _dbContext.OutboxMessages.Add(new OutboxMessage
    {
        Id = Guid.NewGuid(),
        Type = nameof(OrderCreated),
        Payload = JsonSerializer.Serialize(new OrderCreated(order.Id)),
        OccurredOnUtc = DateTime.UtcNow
    });

    // Tek SaveChanges → tek transaction → atomik
    await _dbContext.SaveChangesAsync();
}
```

EF Core'da bunu otomatikleştirmek için `SaveChangesInterceptor` veya `DbContext.SaveChanges` override kullanarak domain event'leri toplayıp outbox'a yazabilirsiniz.

## Outbox Processor

Tabloya yazılan mesajları broker'a taşıyan arka plan servisi. .NET'te `BackgroundService` ile yazılır.

```csharp
public class OutboxProcessor : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly IMessageBus _bus;
    private readonly ILogger<OutboxProcessor> _logger;

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            using var scope = _scopeFactory.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();

            var messages = await db.OutboxMessages
                .Where(m => m.ProcessedOnUtc == null)
                .OrderBy(m => m.OccurredOnUtc)
                .Take(20)
                .ToListAsync(ct);

            foreach (var message in messages)
            {
                try
                {
                    await _bus.PublishAsync(message.Type, message.Payload, ct);
                    message.ProcessedOnUtc = DateTime.UtcNow;
                }
                catch (Exception ex)
                {
                    message.Error = ex.Message;
                    message.RetryCount++;
                    _logger.LogError(ex, "Outbox mesajı yayınlanamadı: {Id}", message.Id);
                }
            }

            await db.SaveChangesAsync(ct);
            await Task.Delay(TimeSpan.FromSeconds(2), ct);
        }
    }
}
```

## At-Least-Once ve Idempotency Zorunluluğu

> **Kritik nokta:** Outbox Pattern **at-least-once** (en az bir kez) teslimat garantisi sağlar, **exactly-once** değil.

Şu senaryoyu düşünün: Processor mesajı broker'a yayınladı (`PublishAsync` başarılı), ama `ProcessedOnUtc` güncellenmeden önce çöktü. Bir sonraki döngüde aynı mesaj **tekrar** yayınlanır. Bu nedenle:

- **Tüketici (consumer) tarafı mutlaka idempotent olmalıdır.** Aynı event'i iki kez almak yan etki üretmemelidir.
- Bunu sağlamak için **Inbox Pattern** kullanılır (aşağıda).

→ Detay için: [Idempotency](../../senior/advanced-system-design/idempotency.md)

## Inbox Pattern (Tüketici Tarafı)

Outbox üreten servisin karşılığı; tüketici tarafında **deduplication** sağlar. Gelen her mesajın `MessageId`'sini bir `Inbox` tablosuna kaydeder ve daha önce işlenmiş mesajları atlar.

```csharp
public async Task HandleAsync(OrderCreated evt, Guid messageId)
{
    // Bu mesaj daha önce işlendi mi?
    bool alreadyProcessed = await _db.InboxMessages
        .AnyAsync(x => x.MessageId == messageId);

    if (alreadyProcessed)
        return; // tekrarı sessizce atla

    using var tx = await _db.Database.BeginTransactionAsync();

    // İş mantığı + inbox kaydı aynı transaction'da
    await ProcessOrderAsync(evt);
    _db.InboxMessages.Add(new InboxMessage { MessageId = messageId, ProcessedOnUtc = DateTime.UtcNow });

    await _db.SaveChangesAsync();
    await tx.CommitAsync();
}
```

## Polling vs CDC ile Yayınlama

Outbox'tan mesaj okumanın iki yolu vardır:

| Yöntem | Açıklama | Artı / Eksi |
|--------|----------|-------------|
| **Polling Publisher** | Processor tabloyu periyodik sorgular (yukarıdaki örnek) | ✅ Basit, her DB ile çalışır. ❌ Sürekli sorgu yükü, küçük gecikme. |
| **CDC (Change Data Capture)** | Debezium gibi araçlar DB transaction log'unu okuyup Kafka'ya basar | ✅ Düşük gecikme, DB yükü yok. ❌ Altyapı karmaşıklığı (Debezium/Kafka Connect). |

Yüksek hacimli sistemlerde CDC + Debezium tercih edilir; orta ölçekte polling fazlasıyla yeterlidir.

## Hazır Kütüphaneler

Her şeyi elle yazmak yerine .NET ekosisteminde hazır çözümler vardır:

- **MassTransit** — `UseInMemoryOutbox` veya EF Core tabanlı `AddEntityFrameworkOutbox`. Outbox + inbox + deduplication'ı kutudan çıkar.
- **NServiceBus** — Outbox desteği yerleşik.
- **Brighter** — Outbox (Outbox Sweeper) desteği.

```csharp
// MassTransit ile EF Core Outbox
services.AddMassTransit(x =>
{
    x.AddEntityFrameworkOutbox<AppDbContext>(o =>
    {
        o.QueryDelay = TimeSpan.FromSeconds(1);
        o.UsePostgres();
        o.UseBusOutbox();
    });
});
```

## Mülakat Soruları

### 1. Soru: "Dual write problem nedir ve Outbox bunu nasıl çözer?"

**Cevap:** Dual write, bir işlemde iki farklı sisteme (DB + message broker) yazmaya çalışırken aralarında atomiklik olmamasıdır. Biri başarılı olup diğeri çökerse sistem tutarsız kalır. Outbox, event'i broker yerine **aynı DB transaction'ı içindeki** bir tabloya yazarak iki yazmayı tek atomik yazmaya indirger. Mesajı broker'a taşıma işi, ayrı ve yeniden denenebilir bir sürece (processor) bırakılır.

### 2. Soru: "Outbox exactly-once garantisi verir mi?"

**Cevap:** Hayır. Outbox **at-least-once** sağlar — processor mesajı yayınlayıp `ProcessedOnUtc`'yi güncelleyemeden çökerse mesaj tekrar gönderilir. Exactly-once etkisi, tüketici tarafında **idempotency / Inbox pattern** ile elde edilir. Yani exactly-once "delivery" değil, exactly-once "processing" hedeflenir.

### 3. Soru: "Outbox tablosu sonsuza kadar büyür mü?"

**Cevap:** İşlenmiş (`ProcessedOnUtc != null`) kayıtlar bir temizleme (cleanup/archival) işiyle periyodik silinmelidir. Genellikle ayrı bir `BackgroundService` belirli bir yaştan eski işlenmiş kayıtları temizler. `ProcessedOnUtc IS NULL` üzerinde **partial index** kullanmak, processor sorgusunu hızlı tutar.

### 4. Soru: "Mesaj sırası (ordering) nasıl korunur?"

**Cevap:** Outbox kayıtları `OccurredOnUtc` veya artan bir sequence ile sıralı okunur. Ancak çoklu processor instance'ı paralel çalışırsa sıra bozulabilir. Sıra kritikse: tek processor, aggregate bazında partition'lama, veya Kafka'da aynı partition key kullanılır.

## Best Practices

- Event payload'unu **JSON** olarak sakla; şema versiyonlamayı (`Version` alanı) baştan düşün.
- Processor'da **batch** oku (örn. 20'şer), tek tek değil — verimlilik için.
- `ProcessedOnUtc IS NULL` için **partial/filtered index** ekle.
- Tüketiciyi **mutlaka idempotent** yap; Outbox tek başına yeterli değildir.
- Kalıcı hata veren mesajlar için **dead-letter** / max retry mekanizması koy.
- Mümkünse elle yazmak yerine **MassTransit/NServiceBus** outbox'unu kullan.

## İlişkili Konular

- [Idempotency](../../senior/advanced-system-design/idempotency.md)
- [Saga Pattern](saga-pattern.md)
- [Message Queue (RabbitMQ/Kafka)](../message-queue/index.md)
- [Eventual Consistency](eventual-consistency.md)

## Kaynaklar

- Microservices.io — Transactional Outbox Pattern (Chris Richardson)
- MassTransit Documentation — Transactional Outbox
- Debezium Documentation — Outbox Event Router
