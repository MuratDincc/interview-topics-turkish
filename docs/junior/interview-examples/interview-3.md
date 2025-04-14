# Mülakat Örneği 3 - Veritabanı ve Entity Framework Core

## 1. Entity Framework Core nedir? Avantajları nelerdir?
**Cevap:** Entity Framework Core, ORM (Object-Relational Mapping) aracıdır. Veritabanı işlemlerini nesne yönelimli bir şekilde yapmayı sağlar. Kod tekrarını azaltır, geliştirme sürecini hızlandırır ve veritabanı bağımsızlığı sağlar.
[Detaylı bilgi için tıklayın](../database-operations/entity-framework-core.md)

## 2. Code First ve Database First yaklaşımları arasındaki farklar nelerdir?
**Cevap:** Code First'te önce C# sınıfları oluşturulur, Database First'te ise önce veritabanı şeması oluşturulur. Code First daha esnek ve bakımı kolaydır, Database First ise mevcut veritabanıyla çalışmak için uygundur.
[Detaylı bilgi için tıklayın](../database-operations/code-first-db-first.md)

## 3. Entity Framework Core'da Migration nedir? Nasıl kullanılır?
**Cevap:** Migration, veritabanı şemasındaki değişiklikleri yönetmek için kullanılan bir mekanizmadır. Add-Migration ve Update-Database komutları ile kullanılır. Her migration bir snapshot oluşturur ve bu sayede veritabanı versiyonları yönetilebilir.
[Detaylı bilgi için tıklayın](../database-operations/migrations.md)

## 4. Lazy Loading ve Eager Loading arasındaki farklar nelerdir?
**Cevap:** Lazy Loading ilişkili verileri ihtiyaç duyulduğunda yükler, Eager Loading ise tüm ilişkili verileri tek seferde yükler. Lazy Loading daha az bellek kullanır ama N+1 problemi oluşturabilir, Eager Loading ise daha fazla bellek kullanır ama performanslıdır.
[Detaylı bilgi için tıklayın](../database-operations/loading-strategies.md)

## 5. Entity Framework Core'da Change Tracking nedir?
**Cevap:** Change Tracking, entity'lerdeki değişiklikleri takip eden bir mekanizmadır. SaveChanges() çağrıldığında bu değişiklikler veritabanına yansıtılır. AsNoTracking() ile devre dışı bırakılabilir.
[Detaylı bilgi için tıklayın](../database-operations/change-tracking.md)

## 6. Entity Framework Core'da Raw SQL sorguları nasıl çalıştırılır?
**Cevap:** FromSqlRaw ve ExecuteSqlRaw metodları kullanılarak raw SQL sorguları çalıştırılabilir. Parametreler kullanılarak SQL injection saldırılarına karşı koruma sağlanabilir.
[Detaylı bilgi için tıklayın](../database-operations/raw-sql.md)

## 7. Entity Framework Core'da Stored Procedure nasıl kullanılır?
**Cevap:** FromSqlRaw metodu ile stored procedure'lar çağrılabilir. Parametreler DbParameter sınıfı kullanılarak güvenli bir şekilde geçilebilir.
[Detaylı bilgi için tıklayın](../database-operations/stored-procedures.md)

## 8. Entity Framework Core'da Transaction nedir? Nasıl kullanılır?
**Cevap:** Transaction, birden fazla veritabanı işlemini atomik hale getiren bir mekanizmadır. BeginTransaction() metodu ile kullanılır. Commit() ile işlemler onaylanır, Rollback() ile geri alınır.
[Detaylı bilgi için tıklayın](../database-operations/transactions.md)

## 9. Entity Framework Core'da Concurrency nedir? Nasıl yönetilir?
**Cevap:** Concurrency, aynı veri üzerinde eşzamanlı değişikliklerin yönetilmesidir. Concurrency token'lar ile yönetilir. RowVersion veya Timestamp kullanılarak implemente edilebilir.
[Detaylı bilgi için tıklayın](../database-operations/concurrency.md)

## 10. Entity Framework Core'da Index nedir? Nasıl oluşturulur?
**Cevap:** Index, veritabanı sorgularının performansını artırmak için kullanılan yapılardır. HasIndex() metodu ile oluşturulur. Unique, clustered gibi özellikler tanımlanabilir.
[Detaylı bilgi için tıklayın](../database-operations/indexes.md)

## 11. Entity Framework Core'da Seed Data nedir? Nasıl kullanılır?
**Cevap:** Seed Data, veritabanı oluşturulduğunda otomatik olarak eklenen başlangıç verileridir. OnModelCreating metodunda tanımlanır. HasData() metodu ile kullanılır.
[Detaylı bilgi için tıklayın](../database-operations/seed-data.md)

## 12. Entity Framework Core'da DbContext nedir? Nasıl kullanılır?
**Cevap:** DbContext, veritabanı bağlantısını ve entity'leri yöneten bir sınıftır. DbSet'ler üzerinden entity'lere erişim sağlar. OnConfiguring ve OnModelCreating metodları ile yapılandırılır.
[Detaylı bilgi için tıklayın](../database-operations/dbcontext.md)

## 13. Entity Framework Core'da AsNoTracking nedir? Ne zaman kullanılır?
**Cevap:** AsNoTracking, change tracking'i devre dışı bırakan bir metoddur. Sadece okuma işlemleri yapılacaksa kullanılır. Performans artışı sağlar.
[Detaylı bilgi için tıklayın](../database-operations/change-tracking.md)

## 14. Entity Framework Core'da Include nedir? Nasıl kullanılır?
**Cevap:** Include, ilişkili verileri yüklemek için kullanılan bir metoddur. Eager loading için kullanılır. ThenInclude ile iç içe ilişkiler de yüklenebilir.
[Detaylı bilgi için tıklayın](../database-operations/loading-strategies.md)

## 15. Entity Framework Core'da Global Query Filters nedir?
**Cevap:** Global Query Filters, tüm sorgulara otomatik olarak uygulanan filtrelerdir. OnModelCreating metodunda tanımlanır. Soft delete gibi senaryolarda kullanılır.
[Detaylı bilgi için tıklayın](../database-operations/query-filters.md)

## 16. Entity Framework Core'da Soft Delete nedir? Nasıl uygulanır?
**Cevap:** Soft Delete, verileri fiziksel olarak silmek yerine bir flag ile işaretleme yöntemidir. Global query filter ile uygulanır. IsDeleted gibi bir property kullanılır.
[Detaylı bilgi için tıklayın](../database-operations/soft-delete.md)

## 17. Entity Framework Core'da Value Conversions nedir?
**Cevap:** Value Conversions, entity property'lerinin veritabanındaki karşılıklarını dönüştürmek için kullanılan bir mekanizmadır. HasConversion() metodu ile kullanılır.
[Detaylı bilgi için tıklayın](../database-operations/value-conversions.md)

## 18. Entity Framework Core'da Owned Entity Types nedir?
**Cevap:** Owned Entity Types, başka bir entity'nin parçası olan ve kendi başına bir tablo oluşturmayan entity'lerdir. OwnsOne veya OwnsMany metodları ile tanımlanır.
[Detaylı bilgi için tıklayın](../database-operations/owned-entities.md)

## 19. Entity Framework Core'da Shadow Properties nedir?
**Cevap:** Shadow Properties, entity sınıfında tanımlanmayan ama veritabanında bulunan property'lerdir. Property() metodu ile tanımlanır. Audit trail gibi senaryolarda kullanılır.
[Detaylı bilgi için tıklayın](../database-operations/shadow-properties.md)

## 20. Entity Framework Core'da Database Provider nedir? Hangi provider'lar mevcuttur?
**Cevap:** Database Provider, farklı veritabanı sistemleriyle çalışmak için kullanılan adaptörlerdir. SQL Server, PostgreSQL, MySQL, SQLite gibi provider'lar mevcuttur. UseSqlServer(), UseNpgsql() gibi metodlarla kullanılır.
[Detaylı bilgi için tıklayın](../database-operations/database-providers.md) 