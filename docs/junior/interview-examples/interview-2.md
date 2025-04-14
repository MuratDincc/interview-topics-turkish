# Mülakat Örneği 2 - ASP.NET Core ve Web Geliştirme

## 1. ASP.NET Core'da Middleware nedir? Nasıl çalışır?
**Cevap:** Middleware, HTTP isteklerini işleyen ve yanıtlayan bir pipeline bileşenidir. Her middleware bir sonraki middleware'i çağırabilir veya pipeline'ı sonlandırabilir. Use, Map ve Run metodları ile kullanılır.
[Detaylı bilgi için tıklayın](../aspnet-core-basics/middleware.md)

## 2. Dependency Injection Container nedir? Nasıl kullanılır?
**Cevap:** Bağımlılıkların yaşam döngüsünü yöneten ve servisleri kaydeden bir mekanizmadır. IServiceCollection üzerinden servisler kaydedilir ve IServiceProvider üzerinden çözümlenir. Singleton, Scoped ve Transient olmak üzere üç yaşam döngüsü vardır.
[Detaylı bilgi için tıklayın](../aspnet-core-basics/dependency-injection.md)

## 3. ASP.NET Core'da Authentication ve Authorization arasındaki fark nedir?
**Cevap:** Authentication kullanıcının kimliğini doğrulama, Authorization ise kullanıcının yetkilerini kontrol etme işlemidir. Authentication kim olduğunu, Authorization ne yapabileceğini belirler.
[Detaylı bilgi için tıklayın](../security-basics/authentication.md)

## 4. JWT (JSON Web Token) nasıl çalışır?
**Cevap:** JWT, kullanıcı bilgilerini içeren, imzalanmış bir token'dır. Header, Payload ve Signature olmak üzere üç bölümden oluşur. Header token tipini ve kullanılan algoritmayı, Payload veriyi, Signature ise token'ın doğruluğunu kontrol etmek için kullanılır.
[Detaylı bilgi için tıklayın](../security-basics/jwt.md)

## 5. ASP.NET Core'da CORS nedir? Nasıl yapılandırılır?
**Cevap:** Cross-Origin Resource Sharing, farklı domainler arası kaynak paylaşımını yöneten bir güvenlik mekanizmasıdır. Startup.cs'de ConfigureServices metodunda yapılandırılır. WithOrigins, WithMethods, WithHeaders gibi metodlar ile detaylı yapılandırma yapılabilir.
[Detaylı bilgi için tıklayın](../aspnet-core-basics/cors.md)

## 6. ASP.NET Core'da Routing nasıl çalışır?
**Cevap:** Gelen HTTP isteklerini ilgili controller action'larına yönlendiren bir mekanizmadır. Attribute routing ve conventional routing olmak üzere iki türü vardır. Route attribute'u ile özel route'lar tanımlanabilir.
[Detaylı bilgi için tıklayın](../aspnet-core-basics/routing.md)

## 7. Model Binding nedir? Nasıl çalışır?
**Cevap:** HTTP isteklerindeki verileri C# nesnelerine dönüştüren bir mekanizmadır. Form verileri, query string, route data gibi kaynaklardan veri bağlama yapabilir. [FromBody], [FromQuery], [FromRoute] gibi attribute'lar ile kaynak belirtilebilir.
[Detaylı bilgi için tıklayın](../aspnet-core-basics/model-binding.md)

## 8. ASP.NET Core'da Session ve Cookie arasındaki farklar nelerdir?
**Cevap:** Session sunucu tarafında, Cookie ise istemci tarafında veri saklar. Session daha güvenli ama daha fazla kaynak kullanır. Cookie daha az kaynak kullanır ama güvenlik açısından daha risklidir.
[Detaylı bilgi için tıklayın](../aspnet-core-basics/session-cookie.md)

## 9. ASP.NET Core'da Logging nasıl yapılır?
**Cevap:** ILogger interface'i üzerinden farklı log seviyelerinde (Information, Warning, Error vb.) loglama yapılabilir. Serilog, NLog gibi üçüncü parti kütüphaneler de kullanılabilir.
[Detaylı bilgi için tıklayın](../aspnet-core-basics/logging.md)

## 10. ASP.NET Core'da Configuration nasıl yönetilir?
**Cevap:** appsettings.json, environment variables, command line arguments gibi farklı kaynaklardan yapılandırma yapılabilir. IConfiguration interface'i üzerinden erişilebilir.
[Detaylı bilgi için tıklayın](../aspnet-core-basics/configuration.md)

## 11. ASP.NET Core'da Response Caching nedir?
**Cevap:** HTTP yanıtlarını önbelleğe alarak performansı artıran bir mekanizmadır. ResponseCache attribute'u ile kullanılır. Duration, Location, VaryByHeader gibi özellikler ile yapılandırılabilir.
[Detaylı bilgi için tıklayın](../aspnet-core-basics/caching.md)

## 12. ASP.NET Core'da Health Checks nedir?
**Cevap:** Uygulamanın sağlık durumunu kontrol eden ve raporlayan bir mekanizmadır. UseHealthChecks middleware'i ile kullanılır. Veritabanı, dış servisler gibi bağımlılıkların durumunu kontrol edebilir.
[Detaylı bilgi için tıklayın](../aspnet-core-basics/health-checks.md)

## 13. ASP.NET Core'da Background Services nedir?
**Cevap:** Arka planda çalışan uzun süreli görevleri yönetmek için kullanılan servislerdir. IHostedService interface'ini implemente eder. HostedService sınıfından kalıtım alarak kullanılabilir.
[Detaylı bilgi için tıklayın](../aspnet-core-basics/background-services.md)

## 14. ASP.NET Core'da SignalR nedir?
**Cevap:** Gerçek zamanlı iki yönlü iletişim sağlayan bir kütüphanedir. WebSocket, Server-Sent Events gibi teknolojileri kullanır. Hub sınıfı üzerinden istemci-sunucu iletişimi sağlar.
[Detaylı bilgi için tıklayın](../aspnet-core-basics/signalr.md)

## 15. ASP.NET Core'da gRPC nedir?
**Cevap:** Yüksek performanslı RPC (Remote Procedure Call) framework'üdür. Protocol Buffers kullanarak veri serileştirme yapar. HTTP/2 üzerinde çalışır ve binary format kullanır.
[Detaylı bilgi için tıklayın](../aspnet-core-basics/grpc.md)

## 16. ASP.NET Core'da Rate Limiting nedir?
**Cevap:** API isteklerini sınırlandıran bir güvenlik mekanizmasıdır. Belirli bir zaman aralığında maksimum istek sayısını kontrol eder. AspNetCoreRateLimit gibi kütüphaneler kullanılabilir.
[Detaylı bilgi için tıklayın](../aspnet-core-basics/rate-limiting.md)

## 17. ASP.NET Core'da Output Caching nedir?
**Cevap:** Action metodlarının çıktılarını önbelleğe alan bir mekanizmadır. ResponseCache attribute'u ile kullanılır. VaryByQueryKeys özelliği ile farklı parametreler için farklı önbellekler oluşturulabilir.
[Detaylı bilgi için tıklayın](../aspnet-core-basics/output-caching.md)

## 18. ASP.NET Core'da Request Validation nedir?
**Cevap:** Gelen isteklerin doğruluğunu kontrol eden bir mekanizmadır. ModelState.IsValid ile kontrol edilir. Data Annotations ile validasyon kuralları tanımlanabilir.
[Detaylı bilgi için tıklayın](../aspnet-core-basics/validation.md)

## 19. ASP.NET Core'da Tag Helpers nedir?
**Cevap:** Razor view'larda HTML elementlerini C# koduna dönüştüren yardımcı sınıflardır. @addTagHelper direktifi ile kullanılır. Form, Input, Label gibi hazır tag helper'lar vardır.
[Detaylı bilgi için tıklayın](../aspnet-core-basics/tag-helpers.md)

## 20. ASP.NET Core'da View Components nedir?
**Cevap:** Yeniden kullanılabilir UI bileşenleri oluşturmak için kullanılan bir yapıdır. Partial view'lara benzer ama daha güçlüdür. ViewComponent sınıfından kalıtım alarak kullanılır.
[Detaylı bilgi için tıklayın](../aspnet-core-basics/view-components.md) 