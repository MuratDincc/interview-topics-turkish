# Web Development Basics

## Giriş

Web geliştirme temelleri, .NET backend geliştiricileri için önemli bir konudur. Backend API'ler geliştirirken, frontend teknolojileri ve web standartları hakkında temel bilgiye sahip olmak gerekir.

## Web Development Temel Konuları

### 1. HTML & CSS Basics
- HTML yapısı ve semantik
- CSS styling ve layout
- Responsive design temelleri
- CSS framework'leri

### 2. JavaScript Basics
- JavaScript syntax ve temel kavramlar
- DOM manipulation
- Event handling
- AJAX ve fetch API

### 3. HTTP Fundamentals
- HTTP protokolü
- Request/Response yapısı
- HTTP methods
- Status codes
- Headers

### 4. Web Security Basics
- XSS (Cross-Site Scripting)
- CSRF (Cross-Site Request Forgery)
- SQL Injection
- Input validation
- HTTPS

## Web Development'ın Backend Geliştiriciler İçin Önemi

1. **API Tasarımı**
   - RESTful API prensipleri
   - HTTP status codes
   - Request/Response formatları

2. **Frontend Entegrasyonu**
   - CORS yapılandırması
   - Authentication/Authorization
   - Data serialization

3. **Web Güvenliği**
   - Security headers
   - Input validation
   - Output encoding

4. **Performance**
   - Caching stratejileri
   - Compression
   - Minification

## Mülakat Soruları

### Temel Sorular

1. **HTML ve CSS'in backend geliştiriciler için önemi nedir?**
   - **Cevap**: Backend API'ler frontend ile entegre olur, bu yüzden HTML/CSS bilgisi API tasarımında ve dokümantasyonda önemlidir.

2. **JavaScript'in backend geliştiriciler için önemi nedir?**
   - **Cevap**: Modern web uygulamalarında JavaScript backend ile sık etkileşim kurar, API kullanımı ve hata yönetimi için önemlidir.

3. **HTTP protokolünün temel bileşenleri nelerdir?**
   - **Cevap**: Request line, headers, body ve response line, status code, headers, body.

4. **Web güvenliği için hangi temel önlemler alınmalıdır?**
   - **Cevap**: Input validation, output encoding, HTTPS kullanımı, security headers.

5. **Responsive design nedir ve neden önemlidir?**
   - **Cevap**: Farklı cihazlarda uyumlu görünüm sağlayan tasarım yaklaşımı, kullanıcı deneyimi için kritiktir.

### Teknik Sorular

1. **CORS nedir ve nasıl yapılandırılır?**
   - **Cevap**: Cross-Origin Resource Sharing, farklı domain'lerden gelen isteklere izin vermek için kullanılır.

2. **HTTP status codes'ların kategorileri nelerdir?**
   - **Cevap**: 1xx (Informational), 2xx (Success), 3xx (Redirection), 4xx (Client Error), 5xx (Server Error).

3. **XSS saldırısı nedir ve nasıl önlenir?**
   - **Cevap**: Cross-Site Scripting, kullanıcı input'larını validate ederek ve output'ları encode ederek önlenir.

4. **AJAX nedir ve nasıl çalışır?**
   - **Cevap**: Asynchronous JavaScript and XML, sayfa yenilenmeden veri alışverişi yapmayı sağlar.

5. **Web API'lerde authentication nasıl sağlanır?**
   - **Cevap**: JWT tokens, API keys, OAuth, session-based authentication gibi yöntemlerle.

## Best Practices

1. **API Tasarımı**
   - RESTful prensipleri uygulayın
   - Tutarlı naming convention kullanın
   - Proper HTTP status codes kullanın

2. **Güvenlik**
   - Input validation yapın
   - Output encoding uygulayın
   - HTTPS kullanın
   - Security headers ekleyin

3. **Performance**
   - Caching stratejileri uygulayın
   - Compression kullanın
   - CDN kullanmayı değerlendirin

4. **Dokümantasyon**
   - API dokümantasyonu hazırlayın
   - Örnek request/response'lar ekleyin
   - Error handling açıklayın

## Kaynaklar

- [MDN Web Docs](https://developer.mozilla.org/)
- [W3Schools](https://www.w3schools.com/)
- [HTTP Status Codes](https://httpstatuses.com/)
- [Web Security Guidelines](https://owasp.org/www-project-top-ten/)
- [REST API Design](https://restfulapi.net/) 