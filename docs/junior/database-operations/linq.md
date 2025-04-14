# LINQ (Language Integrated Query)

## Genel Bakış
LINQ, .NET platformunda veri sorgulama işlemlerini gerçekleştirmek için kullanılan bir teknolojidir. Farklı veri kaynaklarından (koleksiyonlar, veritabanları, XML vb.) veri sorgulamayı sağlar.

## Mülakat Soruları ve Cevapları

### 1. LINQ nedir ve neden kullanılır?
**Cevap:**
LINQ, veri sorgulama işlemlerini C# dilinin bir parçası haline getiren bir teknolojidir. Kullanım nedenleri:
- Tip güvenli sorgulama
- Okunabilir ve anlaşılır kod
- Farklı veri kaynakları için tutarlı sorgulama
- Derleme zamanı hata kontrolü

**Örnek Kod:**
```csharp
// Koleksiyon üzerinde LINQ sorgusu
var numbers = new List<int> { 1, 2, 3, 4, 5 };
var evenNumbers = numbers.Where(n => n % 2 == 0);

// Veritabanı üzerinde LINQ sorgusu
var products = await _context.Products
    .Where(p => p.Price > 100)
    .OrderBy(p => p.Name)
    .ToListAsync();
```

### 2. LINQ metodları nelerdir ve nasıl kullanılır?
**Cevap:**
Temel LINQ metodları:
- Where: Filtreleme
- Select: Projeksiyon
- OrderBy: Sıralama
- GroupBy: Gruplama
- Join: Birleştirme

**Örnek Kod:**
```csharp
// Where ve Select
var expensiveProducts = products
    .Where(p => p.Price > 1000)
    .Select(p => new { p.Name, p.Price });

// OrderBy
var sortedProducts = products
    .OrderBy(p => p.Price)
    .ThenBy(p => p.Name);

// GroupBy
var productsByCategory = products
    .GroupBy(p => p.Category.Name)
    .Select(g => new { Category = g.Key, Count = g.Count() });

// Join
var productDetails = products
    .Join(categories,
        p => p.CategoryId,
        c => c.Id,
        (p, c) => new { Product = p.Name, Category = c.Name });
```

### 3. Deferred Execution (Ertelenmiş Çalıştırma) nedir?
**Cevap:**
Deferred Execution:
- Sorgu tanımlandığında çalıştırılmaz
- Sonuçlar ihtiyaç duyulduğunda çalıştırılır
- Performans optimizasyonu sağlar
- ToList, ToArray gibi metodlar ile hemen çalıştırılabilir

**Örnek Kod:**
```csharp
// Sorgu tanımlanır ama çalıştırılmaz
var query = products.Where(p => p.Price > 100);

// Sorgu çalıştırılır
var result = query.ToList();

// Her çağrıda yeniden çalıştırılır
foreach (var product in query)
{
    Console.WriteLine(product.Name);
}
```

### 4. LINQ to SQL ve LINQ to Entities arasındaki farklar nelerdir?
**Cevap:**
LINQ to SQL:
- Sadece SQL Server ile çalışır
- Daha basit mapping
- Entity Framework'ten önceki teknoloji
- Sınırlı özellikler

LINQ to Entities:
- Farklı veritabanları ile çalışır
- Daha gelişmiş mapping
- Entity Framework Core ile kullanılır
- Daha fazla özellik ve esneklik

**Örnek Kod:**
```csharp
// LINQ to Entities
var products = await _context.Products
    .Where(p => p.Price > 100)
    .Include(p => p.Category)
    .ToListAsync();

// Stored Procedure çağrısı
var result = await _context.Products
    .FromSqlRaw("EXEC GetExpensiveProducts @minPrice", 
        new SqlParameter("@minPrice", 100))
    .ToListAsync();
```

### 5. LINQ sorgularında performans optimizasyonu nasıl yapılır?
**Cevap:**
Performans optimizasyonu için:
- Gereksiz sorgulardan kaçının
- Select ile sadece ihtiyaç duyulan alanları çekin
- Index'leri doğru kullanın
- Asenkron metodları tercih edin

**Örnek Kod:**
```csharp
// Performanslı sorgu
var products = await _context.Products
    .Where(p => p.Price > 100)
    .Select(p => new { p.Id, p.Name, p.Price })
    .ToListAsync();

// Index kullanımı
var products = await _context.Products
    .Where(p => p.CategoryId == categoryId && p.Price > minPrice)
    .ToListAsync();
```

## Best Practices
1. **Sorgu Optimizasyonu**
   - Gereksiz Include'lardan kaçının
   - Select ile projeksiyon yapın
   - N+1 probleminden kaçının
   - Raw SQL sorgularını dikkatli kullanın

2. **Kod Organizasyonu**
   - Sorguları repository pattern ile organize edin
   - Extension metodları kullanın
   - Sorguları tekrar kullanılabilir hale getirin
   - Asenkron metodları tercih edin

3. **Hata Yönetimi**
   - Try-catch bloklarını kullanın
   - Loglama yapın
   - Validation kontrollerini ekleyin
   - Timeout değerlerini ayarlayın

## Kaynaklar
- [LINQ Dokümantasyonu](https://docs.microsoft.com/tr-tr/dotnet/csharp/programming-guide/concepts/linq/)
- [Entity Framework Core ve LINQ](https://docs.microsoft.com/tr-tr/ef/core/querying/)
- [LINQ Best Practices](https://docs.microsoft.com/tr-tr/dotnet/csharp/programming-guide/concepts/linq/best-practices) 