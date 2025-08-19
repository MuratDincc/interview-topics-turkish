# HTTP Fundamentals

## Giriş

HTTP (HyperText Transfer Protocol), web üzerinde veri alışverişi yapmak için kullanılan temel protokoldür. Backend geliştiriciler olarak HTTP'i anlamak, API tasarımı ve web güvenliği için kritiktir.

## HTTP Protokolü

### HTTP Nedir?
HTTP, client (tarayıcı) ve server arasında veri alışverişi yapmayı sağlayan application layer protokolüdür. Stateless bir protokoldür, yani her request bağımsızdır.

### HTTP Versiyonları
- **HTTP/1.0**: Basit request-response modeli
- **HTTP/1.1**: Persistent connections, chunked transfer
- **HTTP/2**: Multiplexing, server push, header compression
- **HTTP/3**: QUIC protokolü üzerine kurulu, UDP tabanlı

## HTTP Request/Response Yapısı

### HTTP Request
```http
GET /api/users HTTP/1.1
Host: api.example.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
Accept: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
Content-Type: application/json

{
  "name": "Ahmet",
  "email": "ahmet@example.com"
}
```

### HTTP Response
```http
HTTP/1.1 200 OK
Date: Mon, 23 May 2024 22:38:34 GMT
Content-Type: application/json
Content-Length: 135
Server: nginx/1.18.0
Cache-Control: no-cache

{
  "id": 123,
  "name": "Ahmet",
  "email": "ahmet@example.com",
  "createdAt": "2024-05-23T22:38:34Z"
}
```

## HTTP Methods

### GET
```http
GET /api/users HTTP/1.1
Host: api.example.com
Accept: application/json
```

**Özellikler:**
- Veri okuma için kullanılır
- Idempotent (birden fazla çağrı aynı sonucu verir)
- Cacheable
- Request body genellikle kullanılmaz

### POST
```http
POST /api/users HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "name": "Ahmet",
  "email": "ahmet@example.com"
}
```

**Özellikler:**
- Yeni kaynak oluşturma
- Non-idempotent
- Request body gerekli
- Cacheable değil

### PUT
```http
PUT /api/users/123 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "name": "Ahmet Yılmaz",
  "email": "ahmet@example.com"
}
```

**Özellikler:**
- Kaynak güncelleme (tam güncelleme)
- Idempotent
- Request body gerekli

### PATCH
```http
PATCH /api/users/123 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "name": "Ahmet Yılmaz"
}
```

**Özellikler:**
- Kısmi kaynak güncelleme
- Idempotent
- Request body gerekli

### DELETE
```http
DELETE /api/users/123 HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
```

**Özellikler:**
- Kaynak silme
- Idempotent
- Request body genellikle kullanılmaz

## HTTP Status Codes

### 1xx - Informational
- **100 Continue**: Request devam edebilir
- **101 Switching Protocols**: Protokol değişimi

### 2xx - Success
- **200 OK**: Başarılı
- **201 Created**: Kaynak oluşturuldu
- **204 No Content**: Başarılı ama içerik yok

### 3xx - Redirection
- **301 Moved Permanently**: Kalıcı yönlendirme
- **302 Found**: Geçici yönlendirme
- **304 Not Modified**: Değişmemiş (cache)

### 4xx - Client Error
- **400 Bad Request**: Hatalı istek
- **401 Unauthorized**: Kimlik doğrulama gerekli
- **403 Forbidden**: Erişim yasak
- **404 Not Found**: Bulunamadı
- **422 Unprocessable Entity**: İşlenemeyen içerik

### 5xx - Server Error
- **500 Internal Server Error**: Sunucu hatası
- **502 Bad Gateway**: Gateway hatası
- **503 Service Unavailable**: Servis kullanılamıyor

## HTTP Headers

### Request Headers
```http
Accept: application/json, text/html
Accept-Language: tr-TR, tr;q=0.9, en;q=0.8
Accept-Encoding: gzip, deflate, br
Authorization: Bearer <token>
Content-Type: application/json
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
```

### Response Headers
```http
Content-Type: application/json
Content-Length: 135
Cache-Control: no-cache, no-store, must-revalidate
ETag: "33a64df551"
Last-Modified: Mon, 23 May 2024 22:38:34 GMT
Set-Cookie: sessionId=abc123; HttpOnly; Secure
```

### Common Headers
- **Content-Type**: Veri tipi (MIME type)
- **Authorization**: Kimlik doğrulama bilgisi
- **Cache-Control**: Cache davranışı
- **ETag**: Entity tag (cache validation)
- **Location**: Yönlendirme URL'i

## HTTP Request Lifecycle

### 1. DNS Resolution
```
Client -> DNS Server: example.com IP'si nedir?
DNS Server -> Client: 93.184.216.34
```

### 2. TCP Connection
```
Client -> Server: SYN
Server -> Client: SYN-ACK
Client -> Server: ACK
```

### 3. HTTP Request
```
Client -> Server: GET /api/users HTTP/1.1
```

### 4. Server Processing
```
Server: Request'i işle, database'den veri al
```

### 5. HTTP Response
```
Server -> Client: HTTP/1.1 200 OK + JSON data
```

### 6. Connection Close (HTTP/1.0) veya Keep-Alive (HTTP/1.1)

## HTTP Security

### HTTPS (HTTP over SSL/TLS)
```http
HTTPS://api.example.com/api/users
```

**Avantajlar:**
- Encryption (şifreleme)
- Authentication (kimlik doğrulama)
- Integrity (bütünlük)

### Security Headers
```http
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Content-Security-Policy: default-src 'self'
```

## Mülakat Soruları

### Temel Sorular

1. **HTTP nedir ve nasıl çalışır?**
   - **Cevap**: Web üzerinde veri alışverişi yapmak için kullanılan application layer protokolü. Client-server modeli ile çalışır.

2. **HTTP stateless protokol nedir?**
   - **Cevap**: Her request bağımsızdır, önceki request'ler hatırlanmaz. Session yönetimi için ek mekanizmalar gerekir.

3. **GET ve POST arasındaki fark nedir?**
   - **Cevap**: GET veri okuma, POST veri oluşturma için kullanılır. GET idempotent, POST değildir.

4. **HTTP status codes kategorileri nelerdir?**
   - **Cevap**: 1xx (Informational), 2xx (Success), 3xx (Redirection), 4xx (Client Error), 5xx (Server Error).

5. **HTTPS nedir ve neden önemlidir?**
   - **Cevap**: HTTP over SSL/TLS, encryption, authentication ve integrity sağlar. Güvenli veri iletişimi için kritiktir.

### Teknik Sorular

1. **HTTP/1.1 vs HTTP/2 farkları nelerdir?**
   - **Cevap**: HTTP/2 multiplexing, server push, header compression gibi özellikler sunar. Performance artışı sağlar.

2. **HTTP caching nasıl çalışır?**
   - **Cevap**: Cache-Control headers, ETag, Last-Modified ile cache validation yapılır. Conditional requests ile bandwidth tasarrufu.

3. **CORS nedir ve nasıl çalışır?**
   - **Cevap**: Cross-Origin Resource Sharing, farklı domain'lerden gelen isteklere izin vermek için kullanılır.

4. **HTTP authentication yöntemleri nelerdir?**
   - **Cevap**: Basic Auth, Digest Auth, Bearer Token, API Key, OAuth 2.0 gibi yöntemler.

5. **HTTP request smuggling nedir?**
   - **Cevap**: HTTP headers'da yanlış yapılandırma ile saldırganın request'leri manipüle etmesi.

## Best Practices

1. **API Tasarımı**
   - RESTful prensipleri uygulayın
   - Proper HTTP methods kullanın
   - Consistent status codes kullanın
   - Meaningful error messages verin

2. **Güvenlik**
   - HTTPS kullanın
   - Security headers ekleyin
   - Input validation yapın
   - Rate limiting uygulayın

3. **Performance**
   - Caching stratejileri uygulayın
   - Compression kullanın
   - CDN kullanmayı değerlendirin
   - HTTP/2 kullanın

4. **Monitoring**
   - Status codes'ları takip edin
   - Response time'ları ölçün
   - Error rate'leri izleyin
   - Logging yapın

## Kaynaklar

- [HTTP Status Codes](https://httpstatuses.com/)
- [MDN HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP)
- [HTTP/2 Specification](https://tools.ietf.org/html/rfc7540)
- [OWASP Security Headers](https://owasp.org/www-project-secure-headers/)
- [HTTP Security Best Practices](https://tools.ietf.org/html/rfc9116) 