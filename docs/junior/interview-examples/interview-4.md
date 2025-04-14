# Mülakat Örneği 4 - Güvenlik ve API Geliştirme

## 1. REST API nedir? Temel prensipleri nelerdir?
**Cevap:** REST (Representational State Transfer), web servisleri için bir mimari stildir. Temel prensipleri: İstemci-Sunucu ayrımı, Durumsuzluk (Stateless), Önbelleğe alınabilirlik, Katmanlı sistem, Tekdüze arayüz ve İsteğe bağlı kod.

## 2. JWT (JSON Web Token) nedir? Nasıl çalışır?
**Cevap:** JWT, güvenli bilgi alışverişi için kullanılan bir token formatıdır. Header, Payload ve Signature olmak üzere üç bölümden oluşur. Header token tipini ve kullanılan algoritmayı, Payload veriyi, Signature ise token'ın doğruluğunu kontrol etmek için kullanılır.

## 3. Authentication ve Authorization arasındaki fark nedir?
**Cevap:** Authentication (Kimlik Doğrulama), kullanıcının kim olduğunu doğrulama işlemidir. Authorization (Yetkilendirme) ise, kullanıcının hangi kaynaklara erişebileceğini belirleme işlemidir.

## 4. API güvenliği için hangi önlemler alınmalıdır?
**Cevap:** HTTPS kullanımı, JWT token kullanımı, Rate limiting, Input validation, CORS yapılandırması, SQL injection koruması, XSS koruması, CSRF koruması, güvenli şifre politikaları ve düzenli güvenlik güncellemeleri.

## 5. Rate Limiting nedir? Neden önemlidir?
**Cevap:** Rate Limiting, API'ye yapılan isteklerin sayısını sınırlandıran bir güvenlik mekanizmasıdır. DDoS saldırılarını önlemek, sunucu kaynaklarını korumak ve adil kullanımı sağlamak için önemlidir.

## 6. API versiyonlama nedir? Nasıl yapılır?
**Cevap:** API versiyonlama, API'de yapılan değişiklikleri yönetmek için kullanılan bir yöntemdir. URL'de versiyon belirtme, header'da versiyon belirtme veya media type'da versiyon belirtme gibi yöntemlerle yapılabilir.

## 7. API dokümantasyonu neden önemlidir? Hangi araçlar kullanılabilir?
**Cevap:** API dokümantasyonu, API'nin nasıl kullanılacağını açıklayan bir rehberdir. Swagger/OpenAPI, Postman, Apiary gibi araçlar kullanılabilir. İyi bir dokümantasyon, geliştiricilerin API'yi daha hızlı anlamasını ve kullanmasını sağlar.

## 8. API testi için hangi yöntemler kullanılabilir?
**Cevap:** Unit testler, Integration testler, Postman testleri, Swagger testleri, API test otomasyon araçları (RestSharp, HttpClient vb.) ve API monitoring araçları kullanılabilir.

## 9. API'de hata yönetimi nasıl yapılmalıdır?
**Cevap:** Standart HTTP durum kodları kullanılmalı, anlamlı hata mesajları dönülmeli, hata detayları loglanmalı ve hata yanıtları tutarlı bir formatta olmalıdır. Try-catch blokları ile hatalar yakalanmalı ve uygun şekilde işlenmelidir.

## 10. API'de performans optimizasyonu için neler yapılabilir?
**Cevap:** Caching kullanımı, pagination, lazy loading, compression, CDN kullanımı, veritabanı optimizasyonu, asenkron işlemler ve microservice mimarisi gibi yöntemler kullanılabilir.

## 11. API'de input validation neden önemlidir?
**Cevap:** Input validation, gelen verilerin doğruluğunu ve güvenliğini kontrol eder. SQL injection, XSS gibi güvenlik açıklarını önler ve veri bütünlüğünü sağlar.

## 12. API'de caching nasıl yapılır?
**Cevap:** Response caching, distributed caching, memory caching gibi yöntemler kullanılabilir. Cache-Control header'ları ile önbelleğe alma stratejileri belirlenebilir.

## 13. API'de logging nasıl yapılmalıdır?
**Cevap:** Structured logging kullanılmalı, log seviyeleri (Information, Warning, Error) doğru kullanılmalı, hassas bilgiler loglanmamalı ve log rotasyonu yapılmalıdır.

## 14. API'de monitoring nasıl yapılır?
**Cevap:** Application Insights, New Relic, Prometheus gibi monitoring araçları kullanılabilir. Response time, error rate, request rate gibi metrikler takip edilmelidir.

## 15. API'de versioning stratejileri nelerdir?
**Cevap:** URL versioning (/api/v1/resource), Header versioning (Accept: application/vnd.company.api.v1+json), Media type versioning ve Query parameter versioning gibi stratejiler kullanılabilir.

## 16. API'de pagination nasıl yapılır?
**Cevap:** Offset-based pagination (skip/take) veya cursor-based pagination kullanılabilir. Sayfa numarası, sayfa boyutu ve toplam kayıt sayısı gibi bilgiler yanıtta dönülmelidir.

## 17. API'de filtering, sorting ve searching nasıl yapılır?
**Cevap:** Query string parametreleri kullanılarak (filter, sort, search) yapılabilir. Örneğin: /api/products?filter=category=electronics&sort=price&search=phone

## 18. API'de authentication için hangi yöntemler kullanılabilir?
**Cevap:** JWT, OAuth 2.0, API Key, Basic Auth, OIDC gibi yöntemler kullanılabilir. Her yöntemin kendine özgü avantaj ve dezavantajları vardır.

## 19. API'de rate limiting nasıl yapılır?
**Cevap:** Token bucket, Leaky bucket, Fixed window counter gibi algoritmalar kullanılabilir. X-RateLimit-Limit, X-RateLimit-Remaining gibi header'lar ile limit bilgileri dönülmelidir.

## 20. API'de security headers nelerdir? Neden önemlidir?
**Cevap:** Content-Security-Policy, X-Frame-Options, X-Content-Type-Options, X-XSS-Protection, Strict-Transport-Security gibi header'lar kullanılmalıdır. Bu header'lar, çeşitli güvenlik açıklarını önlemek için önemlidir. 