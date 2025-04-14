# Mülakat Örneği 3 - Veritabanı ve Entity Framework Core

## 1. Entity Framework Core nedir? Avantajları nelerdir?
**Cevap:** Entity Framework Core, ORM (Object-Relational Mapping) aracıdır. Veritabanı işlemlerini nesne yönelimli bir şekilde yapmayı sağlar. Kod tekrarını azaltır, geliştirme sürecini hızlandırır ve veritabanı bağımsızlığı sağlar.

## 2. Code First ve Database First yaklaşımları arasındaki farklar nelerdir?
**Cevap:** Code First'te önce C# sınıfları oluşturulur, Database First'te ise önce veritabanı şeması oluşturulur. Code First daha esnek ve bakımı kolaydır, Database First ise mevcut veritabanıyla çalışmak için uygundur.

## 3. Entity Framework Core'da Migration nedir? Nasıl kullanılır?
**Cevap:** Migration, veritabanı şemasındaki değişiklikleri yönetmek için kullanılan bir mekanizmadır. Add-Migration ve Update-Database komutları ile kullanılır. Her migration bir snapshot oluşturur ve bu sayede veritabanı versiyonları yönetilebilir.

## 4. Lazy Loading ve Eager Loading arasındaki farklar nelerdir?
**Cevap:** Lazy Loading ilişkili verileri ihtiyaç duyulduğunda yükler, Eager Loading ise tüm ilişkili verileri tek seferde yükler. Lazy Loading daha az bellek kullanır ama N+1 problemi oluşturabilir, Eager Loading ise daha fazla bellek kullanır ama performanslıdır.

## 5. Entity Framework Core'da Change Tracking nedir?
**Cevap:** Change Tracking, entity'lerdeki değişiklikleri takip eden bir mekanizmadır. SaveChanges() çağrıldığında bu değişiklikler veritabanına yansıtılır. AsNoTracking() ile devre dışı bırakılabilir.

## 6. Entity Framework Core'da Raw SQL sorguları nasıl çalıştırılır?
**Cevap:** FromSqlRaw ve ExecuteSqlRaw metodları kullanılarak raw SQL sorguları çalıştırılabilir. Parametreler kullanılarak SQL injection saldırılarına karşı koruma sağlanabilir.

## 7. Entity Framework Core'da Stored Procedure nasıl kullanılır?
**Cevap:** FromSqlRaw metodu ile stored procedure'lar çağrılabilir. Parametreler DbParameter sınıfı kullanılarak güvenli bir şekilde geçilebilir.

## 8. Entity Framework Core'da Transaction nedir? Nasıl kullanılır?
**Cevap:** Transaction, birden fazla veritabanı işlemini atomik hale getiren bir mekanizmadır. BeginTransaction() metodu ile kullanılır. Commit() ile işlemler onaylanır, Rollback() ile geri alınır.

## 9. Entity Framework Core'da Concurrency nedir? Nasıl yönetilir?
**Cevap:** Concurrency, aynı veri üzerinde eşzamanlı değişikliklerin yönetilmesidir. Concurrency token'lar ile yönetilir. RowVersion veya Timestamp kullanılarak implemente edilebilir.

## 10. Entity Framework Core'da Index nedir? Nasıl oluşturulur?
**Cevap:** Index, veritabanı sorgularının performansını artırmak için kullanılan yapılardır. HasIndex() metodu ile oluşturulur. Unique, clustered gibi özellikler tanımlanabilir.

## 11. Entity Framework Core'da Seed Data nedir? Nasıl kullanılır?
**Cevap:** Seed Data, veritabanı oluşturulduğunda otomatik olarak eklenen başlangıç verileridir. OnModelCreating metodunda tanımlanır. HasData() metodu ile kullanılır.

## 12. Entity Framework Core'da DbContext nedir? Nasıl kullanılır?
**Cevap:** DbContext, veritabanı bağlantısını ve entity'leri yöneten bir sınıftır. DbSet'ler üzerinden entity'lere erişim sağlar. OnConfiguring ve OnModelCreating metodları ile yapılandırılır.

## 13. Entity Framework Core'da AsNoTracking nedir? Ne zaman kullanılır?
**Cevap:** AsNoTracking, change tracking'i devre dışı bırakan bir metoddur. Sadece okuma işlemleri yapılacaksa kullanılır. Performans artışı sağlar.

## 14. Entity Framework Core'da Include nedir? Nasıl kullanılır?
**Cevap:** Include, ilişkili verileri yüklemek için kullanılan bir metoddur. Eager loading için kullanılır. ThenInclude ile iç içe ilişkiler de yüklenebilir.

## 15. Entity Framework Core'da Global Query Filters nedir?
**Cevap:** Global Query Filters, tüm sorgulara otomatik olarak uygulanan filtrelerdir. OnModelCreating metodunda tanımlanır. Soft delete gibi senaryolarda kullanılır.

## 16. Entity Framework Core'da Soft Delete nedir? Nasıl uygulanır?
**Cevap:** Soft Delete, verileri fiziksel olarak silmek yerine bir flag ile işaretleme yöntemidir. Global query filter ile uygulanır. IsDeleted gibi bir property kullanılır.

## 17. Entity Framework Core'da Value Conversions nedir?
**Cevap:** Value Conversions, entity property'lerinin veritabanındaki karşılıklarını dönüştürmek için kullanılan bir mekanizmadır. HasConversion() metodu ile kullanılır.

## 18. Entity Framework Core'da Owned Entity Types nedir?
**Cevap:** Owned Entity Types, başka bir entity'nin parçası olan ve kendi başına bir tablo oluşturmayan entity'lerdir. OwnsOne veya OwnsMany metodları ile tanımlanır.

## 19. Entity Framework Core'da Shadow Properties nedir?
**Cevap:** Shadow Properties, entity sınıfında tanımlanmayan ama veritabanında bulunan property'lerdir. Property() metodu ile tanımlanır. Audit trail gibi senaryolarda kullanılır.

## 20. Entity Framework Core'da Database Provider nedir? Hangi provider'lar mevcuttur?
**Cevap:** Database Provider, farklı veritabanı sistemleriyle çalışmak için kullanılan adaptörlerdir. SQL Server, PostgreSQL, MySQL, SQLite gibi provider'lar mevcuttur. UseSqlServer(), UseNpgsql() gibi metodlarla kullanılır. 