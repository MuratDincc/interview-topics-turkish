# Eventual Consistency (Nihai Tutarlılık)

## Genel Bakış

Eventual Consistency, dağıtık bir sistemde verinin **anlık olarak değil, kısa bir süre sonra** tüm düğümlerde tutarlı hale geleceği garantisidir. "Yeni yazma durmazsa, sistem sonunda (eventually) tutarlı bir duruma yakınsar." Mikroservislerde Outbox, Saga ve asenkron mesajlaşma kullanıldığı an, sistem büyük ölçüde eventual consistent hale gelir.

> **Temel gerçek:** Dağıtık sistemlerde strong consistency genelde ölçeklenmez. Doğru soru "tutarlı mı?" değil, **"ne kadar sürede ve hangi iş kuralları için tutarlı?"** sorusudur.

## CAP Teoremi

Bir dağıtık veri sistemi, ağ bölünmesi (Partition) anında aşağıdaki üçten **en fazla ikisini** garanti edebilir:

- **C — Consistency:** Her okuma en son yazmayı görür.
- **A — Availability:** Her istek (hatasız) cevap alır.
- **P — Partition tolerance:** Ağ kopsa bile sistem çalışır.

```
        Consistency
           /  \
          /    \
         / CAP  \
        /        \
 Availability — Partition Tolerance
```

Gerçek dünyada ağ bölünmesi (P) **kaçınılmazdır** — onu feda edemezsiniz. Dolayısıyla pratikte seçim **C vs A** arasındadır:

- **CP sistem:** Bölünme anında tutarlılığı korur, kullanılabilirlikten ödün verir (istek reddeder/bekletir). Örn: çoğunluk tabanlı sistemler, banka çekirdek işlemleri.
- **AP sistem:** Bölünme anında cevap vermeye devam eder ama veri geçici tutarsız olabilir. Örn: DNS, sosyal medya beğeni sayacı, sepet.

## PACELC: CAP'in Eksiğini Tamamlamak

CAP yalnızca **bölünme (P) anını** ele alır; ya bölünme yokken? PACELC şunu der:

> Partition (P) varsa → A ve C arasında seç (PAC).
> Else (E) → Latency (L) ve Consistency (C) arasında seç (ELC).

Yani bölünme olmasa bile, **güçlü tutarlılık daha yüksek gecikme** demektir (çoğunluğun onayını beklemek). Düşük gecikme istiyorsanız tutarlılıktan ödün verirsiniz. Bu, gerçek tasarım kararlarına CAP'ten daha yakındır.

## Strong vs Eventual Consistency

| | Strong Consistency | Eventual Consistency |
|--|--|--|
| Okuma garantisi | Her zaman en güncel veri | Bir süre eski (stale) veri görülebilir |
| Gecikme | Yüksek (koordinasyon) | Düşük |
| Kullanılabilirlik | Bölünmede düşer | Yüksek |
| Karmaşıklık | Düşük (uygulama açısından) | Yüksek (uygulama tolere etmeli) |
| Uygun senaryo | Bakiye, stok kritik karar | Sayaç, feed, öneri, bildirim |

### İş Kuralına Göre Seçim

Tek bir sistemde **her ikisi de** kullanılır. Örnek e-ticaret:

- **Strong gereken:** Ödeme onayı, stok düşümü (overselling olmamalı), hesap bakiyesi.
- **Eventual yeterli:** Ürün yorum sayısı, "son görülenler", öneri listesi, beğeni sayısı.

> **Mülakat ipucu:** "Her şeyi eventual consistent yaparım" yanlış bir cevaptır. Doğrusu, **iş kuralının tolerans penceresini** sormaktır: "Bu veri 2 saniye eski olsa kullanıcı/işletme zarar görür mü?"

## Tutarlılık Modelleri (Client-Centric Garantiler)

Eventual consistency "her şey serbest" demek değildir. Kullanıcı deneyimini korumak için ek garantiler tanımlanır:

- **Read-Your-Writes:** Kullanıcı kendi yazdığını her zaman okur. (Profilini güncelleyip hemen eski halini görmemeli.)
- **Monotonic Reads:** Bir kez yeni veriyi gören kullanıcı, sonradan eski veriye **geri dönmemeli**.
- **Monotonic Writes:** Aynı kullanıcının yazmaları sırasıyla uygulanır.
- **Causal Consistency:** Sebep-sonuç ilişkili olaylar herkese aynı sırada görünür (yoruma cevap, yorumdan önce görünmemeli).

Pratikte read-your-writes genellikle, kullanıcının kendi yazmasından sonra bir süre **primary (master) düğümden okumasıyla** sağlanır.

## .NET'te Eventual Consistency ile Çalışmak

### 1. Stale Veriyi UI'da Yönetmek

Saga/async akışlarda kullanıcıya "kesin" değil "geçici" durum gösterin:

```csharp
public enum OrderStatus
{
    Pending,      // saga devam ediyor — henüz kesinleşmedi
    Confirmed,
    Failed
}
// UI: "Siparişiniz işleniyor..." → sonra "Onaylandı"
```

### 2. CQRS ve Read Model Gecikmesi

CQRS'te write model güncellenir, read model (projeksiyon) event ile **asenkron** beslenir. Aradaki süre **consistency window**'dur.

```csharp
// Write tarafı
await _bus.PublishAsync(new ProductPriceChanged(productId, newPrice));

// Read tarafı (projeksiyon) — milisaniyeler/saniyeler sonra güncellenir
public async Task Handle(ProductPriceChanged evt)
{
    await _readDb.UpdatePriceAsync(evt.ProductId, evt.NewPrice);
}
```

Kullanıcı fiyatı değiştirip listede eski fiyatı kısa süre görebilir. Bu kabul edilebilirse CQRS uygundur. → [CQRS & MediatR](../cqrs-mediatr/index.md)

### 3. Çakışma Çözümü (Conflict Resolution)

İki düğüm aynı veriyi farklı güncellerse: **Last-Write-Wins (LWW)** (zaman damgası), **version vector**, veya **CRDT** (çakışmasız veri tipleri — örn. sayaç) kullanılır. Banka bakiyesi gibi kritik veride LWW tehlikelidir (yazma kaybı); orada strong consistency gerekir.

## Mülakat Soruları

### 1. Soru: "Eventual consistency ne demek? Sistemin asla tutarlı olmayacağı mı?"

**Cevap:** Hayır. Yeni yazmalar durduğunda sistemin **sonunda** tüm replikalarda tutarlı hale geleceği garantisidir. Geçici bir tutarsızlık penceresi vardır ama bu pencere kapanır. "Asla tutarlı olmaz" değil, "anlık tutarlı olmayabilir, ama yakınsar" demektir.

### 2. Soru: "CAP teoreminde neden genelde P'yi feda edemeyiz?"

**Cevap:** Ağ bölünmesi bir tasarım tercihi değil, gerçeğin bir parçasıdır — kablo kopar, paket düşer, düğüm erişilemez olur. Birden fazla makineye yayılmış her sistem partition'a maruz kalır. Dolayısıyla P zorunludur ve gerçek seçim C ile A arasındadır.

### 3. Soru: "Bir kullanıcı profilini güncelledi ama eski halini görüyor. Nasıl çözersin?"

**Cevap:** Bu read-your-writes ihlalidir. Çözüm: kullanıcının kendi yazmasından sonra bir süre **primary düğümden okuması**, ya da yazma sonrası read model'i senkron güncellemek, ya da istemcide iyimser (optimistic) güncelleme yapıp arka planda doğrulamak.

### 4. Soru: "Stok düşümünü eventual consistent yapar mısın?"

**Cevap:** Genelde hayır. Overselling (olmayan stoğu satma) iş açısından kabul edilemezse, stok rezervasyonu **strong consistency** veya en azından atomik bir rezervasyon (distributed lock / koşullu güncelleme) gerektirir. Buna karşılık "kaç adet kaldı" gösterimi eventual olabilir.

## Best Practices

- Her veri için **tutarlılık gereksinimini ayrı ayrı** belirle; tek tip uygulama.
- Strong gereken kritik yolları (ödeme, stok, bakiye) **net şekilde işaretle**.
- Eventual akışlarda **read-your-writes** ve **monotonic reads**'i kullanıcı deneyimi için sağla.
- Consistency window'u **ölç ve izle** (event lag, projeksiyon gecikmesi).
- UI/sözleşmeyi geçici "Pending/İşleniyor" durumlarını gösterecek şekilde tasarla.
- Çakışma çözümünü (LWW/version) veriye uygun seç; kritik veride LWW'den kaçın.

## İlişkili Konular

- [Saga Pattern](saga-pattern.md) — Eventual consistency üreten ana desen.
- [Outbox Pattern](outbox-pattern.md)
- [CQRS & MediatR](../cqrs-mediatr/index.md) — Read model gecikmesi.
- [Dağıtık Transaction'lar](distributed-transactions.md)

## Kaynaklar

- Eric Brewer — CAP Theorem
- Daniel Abadi — PACELC
- "Designing Data-Intensive Applications" — Martin Kleppmann (Bölüm 5 & 9)
