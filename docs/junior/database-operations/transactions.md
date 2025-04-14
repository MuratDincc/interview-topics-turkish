# Transactions

## Genel Bakış
Transactions (İşlemler), veritabanı işlemlerinin atomik, tutarlı, izole ve kalıcı (ACID) olmasını sağlayan bir mekanizmadır. Entity Framework Core'da transaction yönetimi, veri bütünlüğünü korumak için kullanılır.

## Mülakat Soruları ve Cevapları

### 1. Transaction nedir ve neden kullanılır?
**Cevap:**
Transaction, bir grup veritabanı işleminin tek bir birim olarak çalışmasını sağlayan bir mekanizmadır. Kullanım nedenleri:
- Veri bütünlüğünü korumak
- ACID özelliklerini sağlamak
- Hata durumunda geri alma imkanı
- Eşzamanlı işlemleri yönetmek

**Örnek Kod:**
```csharp
using (var transaction = await _context.Database.BeginTransactionAsync())
{
    try
    {
        // İşlem 1
        await _context.Products.AddAsync(product1);
        await _context.SaveChangesAsync();

        // İşlem 2
        await _context.Orders.AddAsync(order1);
        await _context.SaveChangesAsync();

        // İşlem başarılı, commit
        await transaction.CommitAsync();
    }
    catch (Exception)
    {
        // Hata durumunda geri alma
        await transaction.RollbackAsync();
        throw;
    }
}
```

### 2. ACID özellikleri nelerdir?
**Cevap:**
ACID özellikleri:
- Atomicity (Atomiklik): İşlemlerin ya tamamı ya da hiçbiri
- Consistency (Tutarlılık): Veritabanı kurallarına uygunluk
- Isolation (İzolasyon): İşlemlerin birbirinden bağımsız çalışması
- Durability (Kalıcılık): İşlem sonuçlarının kalıcı olması

**Örnek Kod:**
```csharp
// Isolation level örneği
using (var transaction = await _context.Database.BeginTransactionAsync(
    System.Data.IsolationLevel.Serializable))
{
    try
    {
        var product = await _context.Products
            .FirstOrDefaultAsync(p => p.Id == productId);

        if (product != null)
        {
            product.Stock -= quantity;
            await _context.SaveChangesAsync();
        }

        await transaction.CommitAsync();
    }
    catch (Exception)
    {
        await transaction.RollbackAsync();
        throw;
    }
}
```

### 3. Transaction isolation level'ları nelerdir?
**Cevap:**
Transaction isolation level'ları:
- Read Uncommitted: En düşük izolasyon
- Read Committed: Varsayılan seviye
- Repeatable Read: Tekrarlanabilir okuma
- Serializable: En yüksek izolasyon

**Örnek Kod:**
```csharp
// Farklı isolation level'ları
using (var transaction = await _context.Database.BeginTransactionAsync(
    System.Data.IsolationLevel.ReadCommitted))
{
    // İşlemler
}

using (var transaction = await _context.Database.BeginTransactionAsync(
    System.Data.IsolationLevel.Serializable))
{
    // Kritik işlemler
}
```

### 4. Transaction scope nedir ve nasıl kullanılır?
**Cevap:**
TransactionScope:
- Dağıtık işlemleri yönetir
- Otomatik transaction yönetimi sağlar
- Nested transaction'ları destekler
- Timeout yönetimi sunar

**Örnek Kod:**
```csharp
using (var scope = new TransactionScope(
    TransactionScopeOption.Required,
    new TransactionOptions
    {
        IsolationLevel = IsolationLevel.ReadCommitted,
        Timeout = TimeSpan.FromSeconds(30)
    },
    TransactionScopeAsyncFlowOption.Enabled))
{
    try
    {
        // İşlem 1
        await _context.Products.AddAsync(product1);
        await _context.SaveChangesAsync();

        // İşlem 2
        await _context.Orders.AddAsync(order1);
        await _context.SaveChangesAsync();

        scope.Complete();
    }
    catch (Exception)
    {
        // Otomatik rollback
        throw;
    }
}
```

### 5. Transaction'ları nasıl optimize edersiniz?
**Cevap:**
Transaction optimizasyonu için:
- Uygun isolation level seçin
- Transaction süresini kısaltın
- Deadlock'lardan kaçının
- Batch işlemleri kullanın

**Örnek Kod:**
```csharp
// Batch işlem örneği
using (var transaction = await _context.Database.BeginTransactionAsync())
{
    try
    {
        // Toplu ekleme
        await _context.Products.AddRangeAsync(products);
        await _context.SaveChangesAsync();

        // Toplu güncelleme
        foreach (var product in products)
        {
            product.Price *= 1.1m;
        }
        await _context.SaveChangesAsync();

        await transaction.CommitAsync();
    }
    catch (Exception)
    {
        await transaction.RollbackAsync();
        throw;
    }
}
```

## Best Practices
1. **Transaction Yönetimi**
   - Transaction süresini kısaltın
   - Uygun isolation level seçin
   - Hata yönetimini unutmayın
   - Timeout değerlerini ayarlayın

2. **Performans Optimizasyonu**
   - Batch işlemleri kullanın
   - Deadlock'lardan kaçının
   - Index'leri optimize edin
   - Connection pooling kullanın

3. **Güvenlik**
   - Yetkilendirme kontrolleri yapın
   - Input validasyonu yapın
   - Loglama yapın
   - Audit trail tutun

## Kaynaklar
- [Entity Framework Core Transactions](https://docs.microsoft.com/tr-tr/ef/core/saving/transactions)
- [Transaction Isolation Levels](https://docs.microsoft.com/tr-tr/dotnet/api/system.transactions.isolationlevel)
- [TransactionScope](https://docs.microsoft.com/tr-tr/dotnet/api/system.transactions.transactionscope) 