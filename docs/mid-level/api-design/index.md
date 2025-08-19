# API Design & Development

## Giriş

API Design & Development, modern backend geliştiricileri için kritik bir konudur. İyi tasarlanmış API'ler, frontend-backend entegrasyonunu kolaylaştırır, developer experience'i artırır ve sistem ölçeklenebilirliğini sağlar.

## API Design & Development Temel Konuları

### 1. GraphQL
- GraphQL vs REST karşılaştırması
- Schema design ve type system
- Resolvers ve data fetching
- Performance optimization

### 2. API Rate Limiting
- Rate limiting stratejileri
- Token bucket algorithm
- Sliding window counter
- Distributed rate limiting

### 3. API Authentication Patterns
- JWT implementation
- OAuth 2.0 flows
- API key management
- Multi-factor authentication

### 4. API Testing
- Postman ve Newman
- API automation
- Contract testing
- Performance testing

## API Design Prensipleri

### 1. RESTful Design
- Resource-based URL yapısı
- Proper HTTP methods kullanımı
- Consistent response formatları
- Error handling standartları

### 2. API Versioning
- URL versioning
- Header versioning
- Content negotiation
- Backward compatibility

### 3. Documentation
- OpenAPI/Swagger specification
- Interactive documentation
- Code examples
- Error code reference

### 4. Security
- Authentication ve authorization
- Input validation
- Rate limiting
- Security headers

## API Development Best Practices

### 1. Design Patterns
- Repository pattern
- Service layer pattern
- DTO pattern
- Response wrapper pattern

### 2. Error Handling
- Consistent error formatları
- Proper HTTP status codes
- Error logging ve monitoring
- User-friendly error messages

### 3. Performance
- Caching stratejileri
- Pagination
- Compression
- Async processing

### 4. Monitoring
- API metrics
- Response time tracking
- Error rate monitoring
- Usage analytics

## Mülakat Soruları

### Temel Sorular

1. **REST API nedir ve temel prensipleri nelerdir?**
   - **Cevap**: Representational State Transfer, stateless, client-server, cacheable, uniform interface, layered system.

2. **GraphQL'in REST'e göre avantajları nelerdir?**
   - **Cevap**: Over-fetching ve under-fetching'i önler, single endpoint, strong typing, real-time updates.

3. **API versioning neden önemlidir?**
   - **Cevap**: Backward compatibility, breaking changes yönetimi, client migration süreci için gerekli.

4. **Rate limiting nedir ve nasıl uygulanır?**
   - **Cevap**: API kullanımını sınırlama, abuse prevention, fair usage için. Token bucket, sliding window algoritmaları.

5. **API authentication yöntemleri nelerdir?**
   - **Cevap**: JWT, OAuth 2.0, API keys, Basic auth, certificate-based authentication.

### Teknik Sorular

1. **JWT token'ların güvenlik riskleri nelerdir?**
   - **Cevap**: Token hijacking, XSS attacks, token expiration, secure storage requirements.

2. **GraphQL'de N+1 problem nedir?**
   - **Cevap**: Multiple database queries, DataLoader pattern, batching ve caching ile çözülür.

3. **API caching stratejileri nelerdir?**
   - **Cevap**: HTTP caching, application-level caching, distributed caching, cache invalidation.

4. **API testing'de contract testing nedir?**
   - **Cevap**: Consumer-driven contracts, API schema validation, breaking changes detection.

5. **Microservices'de API gateway pattern nedir?**
   - **Cevap**: Centralized routing, authentication, rate limiting, monitoring ve logging.

## Best Practices

1. **Design**
   - RESTful prensipleri uygulayın
   - Consistent naming convention kullanın
   - Proper HTTP status codes kullanın
   - Meaningful error messages verin

2. **Security**
   - HTTPS kullanın
   - Input validation yapın
   - Rate limiting uygulayın
   - Security headers ekleyin

3. **Performance**
   - Caching stratejileri uygulayın
   - Pagination kullanın
   - Compression uygulayın
   - Async processing yapın

4. **Documentation**
   - OpenAPI specification yazın
   - Interactive documentation sağlayın
   - Code examples ekleyin
   - Regular updates yapın

## Kaynaklar

- [REST API Design](https://restfulapi.net/)
- [GraphQL Documentation](https://graphql.org/)
- [OpenAPI Specification](https://swagger.io/specification/)
- [API Design Patterns](https://docs.microsoft.com/en-us/azure/architecture/patterns/)
- [OWASP API Security](https://owasp.org/www-project-api-security/) 