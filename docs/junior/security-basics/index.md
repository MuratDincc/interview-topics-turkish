# Güvenlik Temelleri

## Genel Bakış
Güvenlik temelleri, yazılım geliştirme sürecinde uygulanması gereken temel güvenlik prensiplerini ve uygulamalarını kapsar. Bu bölüm, güvenli yazılım geliştirme için gerekli olan temel kavramları ve uygulamaları içerir.

## İçindekiler
1. **Authentication**
   - Kimlik doğrulama yöntemleri
   - JWT ve OAuth
   - Token yönetimi
   - Güvenli oturum yönetimi

2. **Authorization**
   - Yetkilendirme mekanizmaları
   - Role-based access control
   - Policy-based authorization
   - Claims-based authorization

3. **HTTPS**
   - SSL/TLS protokolleri
   - Sertifika yönetimi
   - Güvenli iletişim
   - HTTPS yapılandırması

4. **CORS**
   - Cross-Origin Resource Sharing
   - CORS politikaları
   - Güvenli kaynak paylaşımı
   - CORS yapılandırması

5. **Input Validation**
   - Veri doğrulama teknikleri
   - XSS koruması
   - SQL injection koruması
   - Güvenli input handling

6. **Veri Güvenliği**
   - Veri şifreleme
   - Hassas veri yönetimi
   - Veri maskeleme
   - Güvenli veri depolama

7. **Güvenli Kod Yazımı**
   - Output encoding
   - Error handling
   - Secure coding practices

8. **Güvenlik Testleri**
   - Penetrasyon testleri
   - Güvenlik açığı taraması
   - Code review
   - Security testing tools

## Öğrenme Hedefleri
Bu bölümü tamamladıktan sonra:
- Temel güvenlik kavramlarını anlayabileceksiniz
- Authentication ve authorization mekanizmalarını uygulayabileceksiniz
- Veri güvenliği prensiplerini öğreneceksiniz
- Güvenli kod yazımı tekniklerini kullanabileceksiniz
- Güvenlik testlerini gerçekleştirebileceksiniz

## Ön Koşullar
Bu bölümü takip etmek için:
- Temel programlama bilgisi
- Web uygulama geliştirme deneyimi
- HTTP protokolü hakkında bilgi
- Veritabanı temelleri

## Best Practices
1. **Güvenlik Prensipleri**
   - Defense in depth
   - Least privilege
   - Fail securely
   - Keep it simple

2. **Kod Güvenliği**
   - Output encoding
   - Error handling
   - Secure defaults

3. **Veri Güvenliği**
   - Encryption at rest
   - Encryption in transit
   - Secure key management
   - Data classification

## Örnek Proje Yapısı
```
SecurityBasics/
├── Authentication/
│   ├── JwtAuthentication
│   ├── OAuthAuthentication
│   └── RoleBasedAccess
├── DataSecurity/
│   ├── Encryption
│   ├── DataMasking
│   └── SecureStorage
├── SecureCoding/
│   ├── InputValidation
│   ├── OutputEncoding
│   └── ErrorHandling
└── SecurityTesting/
    ├── PenetrationTesting
    ├── VulnerabilityScanning
    └── CodeReview
```

## Sık Sorulan Sorular
1. Authentication ve Authorization arasındaki fark nedir?
2. JWT token'ları nasıl güvenli bir şekilde saklanır?
3. Veri şifreleme için hangi algoritmalar kullanılmalıdır?
4. Güvenli kod yazımı için temel prensipler nelerdir?
5. Güvenlik testleri nasıl planlanmalıdır?

## Kaynaklar
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Microsoft Security Documentation](https://docs.microsoft.com/tr-tr/security/)
- [NIST Security Guidelines](https://www.nist.gov/topics/cybersecurity) 