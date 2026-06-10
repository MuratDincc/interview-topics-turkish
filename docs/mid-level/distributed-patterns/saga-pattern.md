# Saga Pattern

## Genel Bakış

Saga Pattern, birden fazla servise (ve birden fazla veritabanına) yayılan **uzun süreli iş akışlarını** (long-running transaction) yönetmek için kullanılan bir desendir. Klasik ACID transaction'ın çalışmadığı dağıtık ortamda, her adım kendi lokal transaction'ını yapar; bir adım başarısız olursa önceki adımlar **compensating transaction** (telafi işlemi) ile geri alınır.

> **Temel fikir:** Dağıtık bir "rollback" yoktur. Bunun yerine her başarılı adımın tersini yapan bir telafi adımı tanımlanır.

## Neden Saga?

Klasik örnek: Bir sipariş süreci üç servise dokunur.

1. **Order Service** — siparişi oluştur
2. **Payment Service** — ödemeyi çek
3. **Inventory Service** — stoğu düş

Bunlar farklı veritabanlarında olduğu için tek bir `BEGIN TRANSACTION ... COMMIT` mümkün değildir. Ödeme alındıktan sonra stok yetersizse, ödemeyi **iade etmemiz** (compensate) gerekir.

```
Order oluştur  →  Ödeme al  →  Stok düş
                                   │ (başarısız!)
                                   ▼
            Ödeme iade  ←  Sipariş iptal     (compensation, ters sırada)
```

## İki Yaklaşım: Choreography vs Orchestration

### 1. Choreography (Koreografi)

Merkezi bir yönetici yoktur. Her servis bir event yayınlar, diğer servisler bu event'leri dinleyerek tepki verir. Akış, event zincirinden **ortaya çıkar**.

```csharp
// Order Service
public async Task CreateOrder(Order order)
{
    await _repo.SaveAsync(order);
    await _bus.PublishAsync(new OrderCreated(order.Id, order.Total));
}

// Payment Service — OrderCreated'ı dinler
public async Task Handle(OrderCreated evt)
{
    var result = await _paymentGateway.ChargeAsync(evt.OrderId, evt.Total);
    if (result.Success)
        await _bus.PublishAsync(new PaymentCompleted(evt.OrderId));
    else
        await _bus.PublishAsync(new PaymentFailed(evt.OrderId)); // tetikler: sipariş iptali
}

// Inventory Service — PaymentCompleted'ı dinler
public async Task Handle(PaymentCompleted evt)
{
    if (!await _stock.TryReserveAsync(evt.OrderId))
        await _bus.PublishAsync(new StockReservationFailed(evt.OrderId)); // tetikler: ödeme iadesi
}
```

**Artıları:** Basit, gevşek bağlı (loose coupling), tek hata noktası yok.
**Eksileri:** Akışı izlemek zor (event'ler her yere dağılmış), döngüsel bağımlılık riski, "süreç şu an nerede?" sorusunu yanıtlamak zor.

### 2. Orchestration (Orkestrasyon)

Merkezi bir **orchestrator** (saga state machine) tüm adımları sırayla çağırır ve durumu yönetir. Servisler birbirini tanımaz; sadece orchestrator'a cevap verir.

```csharp
public class OrderSaga
{
    public async Task ExecuteAsync(Guid orderId)
    {
        try
        {
            await _orderService.CreateAsync(orderId);
            await _paymentService.ChargeAsync(orderId);
            await _inventoryService.ReserveAsync(orderId);
            await _orderService.ConfirmAsync(orderId);
        }
        catch (PaymentFailedException)
        {
            await _orderService.CancelAsync(orderId);          // compensation
        }
        catch (StockUnavailableException)
        {
            await _paymentService.RefundAsync(orderId);        // compensation (ters sırada)
            await _orderService.CancelAsync(orderId);
        }
    }
}
```

**Artıları:** Akış tek yerde görünür, izlenebilir, karmaşık mantık (koşul/zaman aşımı) yönetilebilir.
**Eksileri:** Orchestrator merkezi bir bağımlılık ve olası darboğaz; daha fazla kod.

### Karşılaştırma

| Kriter | Choreography | Orchestration |
|--------|--------------|---------------|
| Kontrol | Dağıtık (event-driven) | Merkezi (orchestrator) |
| Bağlılık | Gevşek | Orchestrator'a bağımlı |
| İzlenebilirlik | Zor | Kolay |
| Uygun senaryo | Az adımlı, basit akışlar | Çok adımlı, karmaşık akışlar |
| Hata ayıklama | Zor (dağınık) | Kolay (tek yer) |

> **Mülakat ipucu:** 2-3 adımlı basit akışlarda choreography; 4+ adımlı, koşullu, zaman aşımlı karmaşık akışlarda orchestration önerilir. Cevabı "duruma göre" diye verip bu kriteri açıklamak güçlü bir sinyaldir.

## Compensating Transactions (Telafi İşlemleri)

Saga'nın kalbidir. Her adımın bir tersi olmalıdır:

| İleri İşlem | Telafi İşlemi |
|-------------|---------------|
| Ödeme çek | Ödemeyi iade et |
| Stok rezerve et | Rezervasyonu serbest bırak |
| E-posta gönder | "İptal edildi" e-postası gönder (geri alınamaz, dengeleme) |

**Önemli incelikler:**

- **Semantik geri alma:** Gerçek bir "undo" olmayabilir. Gönderilen e-posta geri alınamaz; bunun yerine telafi edici bir aksiyon (iptal bildirimi) yapılır.
- **Compensation da başarısız olabilir:** Telafi işlemleri **idempotent ve tekrar denenebilir** olmalıdır. Gerekirse manuel müdahale için dead-letter'a düşürülür.
- **Pivot transaction:** Bir noktadan sonra geri dönüş yoktur (örn. ürün kargoya verildi). Bu noktadan önce telafi, sonra sadece ileri (retry) yapılır.

## MassTransit ile Saga State Machine

.NET'te orchestration-based saga için en yaygın araç **MassTransit Saga State Machine**'dir (Automatonymous).

```csharp
public class OrderState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }   // saga örneğini olaylarla eşleştirir
    public string CurrentState { get; set; } = default!;
    public Guid OrderId { get; set; }
}

public class OrderStateMachine : MassTransitStateMachine<OrderState>
{
    public State AwaitingPayment { get; private set; } = default!;
    public State AwaitingStock { get; private set; } = default!;

    public Event<OrderCreated> OrderCreated { get; private set; } = default!;
    public Event<PaymentCompleted> PaymentCompleted { get; private set; } = default!;
    public Event<PaymentFailed> PaymentFailed { get; private set; } = default!;

    public OrderStateMachine()
    {
        InstanceState(x => x.CurrentState);

        Initially(
            When(OrderCreated)
                .Then(ctx => ctx.Saga.OrderId = ctx.Message.OrderId)
                .Publish(ctx => new ChargePayment(ctx.Saga.OrderId))
                .TransitionTo(AwaitingPayment));

        During(AwaitingPayment,
            When(PaymentCompleted)
                .Publish(ctx => new ReserveStock(ctx.Saga.OrderId))
                .TransitionTo(AwaitingStock),
            When(PaymentFailed)
                .Publish(ctx => new CancelOrder(ctx.Saga.OrderId))  // compensation
                .Finalize());
    }
}
```

State, EF Core / MongoDB / Redis ile kalıcı (persistent) saklanır; böylece orchestrator çökse bile saga kaldığı yerden devam eder.

## Saga'nın Riskleri

- **Eventual consistency:** Saga sırasında sistem geçici olarak tutarsızdır (ödeme alındı ama henüz onaylanmadı). UI bunu "İşleniyor" gibi yansıtmalıdır. → [Eventual Consistency](eventual-consistency.md)
- **Isolation eksikliği:** ACID'in "I"si yoktur. Yarı tamamlanmış saga'nın ara durumunu başka bir işlem okuyabilir (dirty read benzeri). Semantik kilitler veya durum bayrakları ile yönetilir.
- **Karmaşıklık:** Her başarı yolu için bir hata yolu tasarlanmalıdır; test yükü yüksektir.

## Mülakat Soruları

### 1. Soru: "Saga ile Two-Phase Commit (2PC) arasındaki fark nedir?"

**Cevap:** 2PC **eş zamanlı (synchronous)** ve **kilitlemeli** bir protokoldür; tüm katılımcılar "prepare" aşamasında kilitlenir ve coordinator çökerse kaynaklar bloke kalır — ölçeklenmez. Saga ise her adımı **ayrı lokal transaction** olarak yürütür, kilit tutmaz ve hata durumunda **compensation** ile geri alır. Saga eventual consistency sağlar; 2PC strong consistency hedefler ama mikroservislerde pratik değildir. → [Dağıtık Transaction'lar](distributed-transactions.md)

### 2. Soru: "Compensation işlemi de başarısız olursa ne yaparsın?"

**Cevap:** Telafi işlemleri idempotent yazılır ve **retry** edilir. Kalıcı başarısızlıkta mesaj **dead-letter queue**'ya düşürülür, alarm üretilir ve gerekirse manuel/operasyonel müdahale devreye girer. Asla sessizce yutulmamalıdır; saga'nın "stuck" durumu izlenebilir olmalıdır.

### 3. Soru: "Choreography'de döngüsel bağımlılığı nasıl önlersin?"

**Cevap:** Event akışını net tanımlamak (her servis hangi event'i dinler/yayınlar), bir event haritası çıkarmak ve adım sayısı arttıkça orchestration'a geçmek gerekir. Choreography'nin sınırı tam da budur: akış görünmez hale gelir, bu yüzden karmaşıklık eşiğinde orchestration tercih edilir.

### 4. Soru: "Saga state'i nerede saklarsın?"

**Cevap:** Orchestration saga'da state kalıcı bir depoda (EF Core/SQL, MongoDB, Redis) `CorrelationId` ile saklanır. Bu sayede orchestrator yeniden başlatılsa veya çökse bile saga kaldığı durumdan devam eder. State'i bellekte tutmak, çökmede tüm akışı kaybetmek demektir.

## Best Practices

- Her ileri adım için **net bir compensation** tanımla; tanımlayamıyorsan adımı saga'nın sonuna (pivot sonrası) taşı.
- Tüm saga adımlarını ve compensation'ları **idempotent** yap (mesajlar tekrar gelebilir).
- Saga state'ini **kalıcı** sakla; bellekte tutma.
- Karmaşık akışlarda **orchestration + state machine** (MassTransit) kullan.
- Saga'nın takıldığı durumlar için **timeout** ve **monitoring** ekle.
- UI/istemciyi eventual consistency'ye göre tasarla ("Siparişiniz işleniyor").

## İlişkili Konular

- [Outbox Pattern](outbox-pattern.md) — Saga event'lerini güvenilir yayınlamak için.
- [Dağıtık Transaction'lar](distributed-transactions.md) — Saga'nın 2PC'ye alternatifi olduğu yer.
- [Eventual Consistency](eventual-consistency.md)
- [Idempotency](../../senior/advanced-system-design/idempotency.md)
- [Event Sourcing](../microservices/event-sourcing.md)

## Kaynaklar

- Microservices.io — Saga Pattern (Chris Richardson)
- MassTransit Documentation — Saga State Machines
- "Microservices Patterns" — Chris Richardson (Manning)
