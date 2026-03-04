# Idempotency (Kıdem)

## Genel Bakış

Idempotency (Kıdem), bir operasyonun birden fazla kez uygulanmasının tek bir kez uygulanmasıyla aynı sonucu vermesi özelliğidir. Dağıtık sistemlerde ağ hataları, timeout'lar ve yeniden denemeler (retry) kaçınılmazdır. Bu koşullarda aynı isteğin birden fazla kez işlenmesini önlemek için idempotency kritik öneme sahiptir.

**Temel kavramlar:**

- **Idempotent Operasyon**: Tekrar çalıştırıldığında yan etkisi olmayan işlem. `GET`, `PUT`, `DELETE` HTTP metotları idempotent; `POST` idempotent değildir.
- **Idempotency Key**: İstemcinin ürettiği, isteği benzersiz şekilde tanımlayan bir token. Sunucu bu token'ı kullanarak tekrar eden istekleri tespit eder.
- **At-Least-Once Delivery**: Mesaj en az bir kez iletilir; tekrar iletim mümkündür. Tüketici idempotent olmalıdır.
- **Exactly-Once Delivery**: Mesaj tam olarak bir kez işlenir. Broker ve tüketici tarafında ek garantiler gerektirir.
- **At-Most-Once Delivery**: Mesaj en fazla bir kez iletilir; kayıp mümkündür.

---

## HTTP Metotlarının Idempotency Durumu

| Metot | Idempotent | Güvenli (Safe) |
|---|---|---|
| GET | Evet | Evet |
| HEAD | Evet | Evet |
| PUT | Evet | Hayır |
| DELETE | Evet | Hayır |
| POST | Hayır | Hayır |
| PATCH | Hayır | Hayır |

---

## Mülakat Soruları

### 1. Soru

**Idempotency nedir ve dağıtık sistemlerde neden kritiktir?**

**Cevap:**

Idempotency, bir işlemin aynı girdilerle kaç kez çalıştırılırsa çalıştırılsın aynı sonucu üretmesi ve sistem durumunu tek çalıştırmayla eşdeğer bırakması özelliğidir. Dağıtık sistemlerde ağ hataları sonucunda istemci timeout alabilir ve isteği yeniden gönderebilir. Sunucu isteği zaten işlemiş olsa bile ikinci isteği idempotency olmadan işlerse çift ödeme, çift sipariş gibi kritik hatalar oluşur. Idempotency key mekanizması ile sunucu önceden işlenmiş istekleri tanımlayabilir ve aynı sonucu döndürebilir.

**Örnek Kod:**

```csharp
// Idempotency Key ile ödeme servisi
public class PaymentService
{
    private readonly IPaymentRepository _paymentRepository;
    private readonly IIdempotencyStore _idempotencyStore;
    private readonly ILogger<PaymentService> _logger;

    public PaymentService(
        IPaymentRepository paymentRepository,
        IIdempotencyStore idempotencyStore,
        ILogger<PaymentService> logger)
    {
        _paymentRepository = paymentRepository;
        _idempotencyStore = idempotencyStore;
        _logger = logger;
    }

    public async Task<PaymentResult> ProcessPaymentAsync(
        PaymentRequest request,
        string idempotencyKey)
    {
        // 1. Daha önce işlenmiş mi kontrol et
        var cached = await _idempotencyStore.GetAsync<PaymentResult>(idempotencyKey);
        if (cached != null)
        {
            _logger.LogInformation(
                "Idempotency key {Key} daha önce işlenmiş, cache'den döndürülüyor.",
                idempotencyKey);
            return cached;
        }

        // 2. Kritik bölge: sadece bir kez çalışmalı
        // Distributed lock ile eş zamanlı tekrar eden istekleri engelle
        var lockKey = $"payment:idempotency:{idempotencyKey}";
        await using var @lock = await _idempotencyStore.AcquireLockAsync(lockKey);

        if (!@lock.IsAcquired)
            throw new ConcurrentRequestException("Aynı idempotency key ile eş zamanlı istek.");

        // Lock alındıktan sonra tekrar kontrol et (double-check locking)
        cached = await _idempotencyStore.GetAsync<PaymentResult>(idempotencyKey);
        if (cached != null) return cached;

        // 3. Ödemeyi işle
        var result = await _paymentRepository.CreatePaymentAsync(new Payment
        {
            Id = Guid.NewGuid().ToString(),
            Amount = request.Amount,
            Currency = request.Currency,
            CustomerId = request.CustomerId,
            IdempotencyKey = idempotencyKey,
            Status = PaymentStatus.Completed,
            CreatedAt = DateTimeOffset.UtcNow
        });

        var paymentResult = new PaymentResult
        {
            PaymentId = result.Id,
            Status = result.Status,
            Amount = result.Amount,
            ProcessedAt = result.CreatedAt
        };

        // 4. Sonucu idempotency store'a kaydet (TTL: 24 saat)
        await _idempotencyStore.SetAsync(
            idempotencyKey,
            paymentResult,
            TimeSpan.FromHours(24));

        return paymentResult;
    }
}

public interface IIdempotencyStore
{
    Task<T?> GetAsync<T>(string key);
    Task SetAsync<T>(string key, T value, TimeSpan ttl);
    Task<IAsyncDisposable> AcquireLockAsync(string key);
}

// Redis tabanlı Idempotency Store
public class RedisIdempotencyStore : IIdempotencyStore
{
    private readonly IDatabase _db;

    public RedisIdempotencyStore(IDatabase db)
    {
        _db = db;
    }

    public async Task<T?> GetAsync<T>(string key)
    {
        var value = await _db.StringGetAsync($"idempotency:{key}");
        if (!value.HasValue) return default;
        return JsonSerializer.Deserialize<T>(value!);
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan ttl)
    {
        var json = JsonSerializer.Serialize(value);
        await _db.StringSetAsync($"idempotency:{key}", json, ttl);
    }

    public async Task<IAsyncDisposable> AcquireLockAsync(string key)
    {
        return await RedisDistributedLock.AcquireAsync(
            _db,
            $"lock:{key}",
            TimeSpan.FromSeconds(30));
    }
}
```

---

### 2. Soru

**ASP.NET Core'da Idempotency Middleware nasıl yazılır?**

**Cevap:**

Idempotency Middleware, her HTTP isteği geldiğinde `Idempotency-Key` başlığını kontrol eder. Eğer bu key daha önce işlenmişse cache'deki yanıtı döner, işlenmemişse downstream'e devam eder ve yanıtı önbelleğe alır. Bu yaklaşım, idempotency mantığını tüm endpoint'lere uygulamayı kolaylaştırır.

**Örnek Kod:**

```csharp
// Idempotency Middleware
public class IdempotencyMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IIdempotencyStore _store;
    private readonly ILogger<IdempotencyMiddleware> _logger;

    // Idempotency kontrolü uygulanacak HTTP metotları
    private static readonly HashSet<string> IdempotentMethods =
        new(StringComparer.OrdinalIgnoreCase) { "POST", "PATCH" };

    public IdempotencyMiddleware(
        RequestDelegate next,
        IIdempotencyStore store,
        ILogger<IdempotencyMiddleware> logger)
    {
        _next = next;
        _store = store;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Yalnızca non-idempotent metotları işle
        if (!IdempotentMethods.Contains(context.Request.Method))
        {
            await _next(context);
            return;
        }

        // Idempotency-Key başlığını oku
        if (!context.Request.Headers.TryGetValue("Idempotency-Key", out var keyValues)
            || string.IsNullOrWhiteSpace(keyValues.FirstOrDefault()))
        {
            // Key yoksa devam et (bazı endpoint'ler zorunlu tutmayabilir)
            await _next(context);
            return;
        }

        var idempotencyKey = keyValues.First()!.Trim();

        // Key formatını doğrula (UUID formatı bekleniyor)
        if (!Guid.TryParse(idempotencyKey, out _))
        {
            context.Response.StatusCode = StatusCodes.Status400BadRequest;
            await context.Response.WriteAsJsonAsync(new
            {
                error = "Geçersiz Idempotency-Key formatı. UUID bekleniyor."
            });
            return;
        }

        // Daha önce işlenmiş mi?
        var cached = await _store.GetAsync<CachedResponse>(
            $"{context.Request.Path}:{idempotencyKey}");

        if (cached != null)
        {
            _logger.LogDebug("Idempotent yanıt döndürülüyor. Key: {Key}", idempotencyKey);

            context.Response.StatusCode = cached.StatusCode;
            context.Response.ContentType = cached.ContentType;

            // Orijinal yanıtın tekrar edildiğini belirt
            context.Response.Headers["X-Idempotency-Replayed"] = "true";

            await context.Response.WriteAsync(cached.Body);
            return;
        }

        // Yanıtı yakala
        var originalBody = context.Response.Body;
        using var memoryStream = new MemoryStream();
        context.Response.Body = memoryStream;

        try
        {
            await _next(context);

            // Yalnızca başarılı yanıtları önbelleğe al (2xx)
            if (context.Response.StatusCode >= 200 && context.Response.StatusCode < 300)
            {
                memoryStream.Seek(0, SeekOrigin.Begin);
                var responseBody = await new StreamReader(memoryStream).ReadToEndAsync();

                var responseToCache = new CachedResponse
                {
                    StatusCode = context.Response.StatusCode,
                    ContentType = context.Response.ContentType ?? "application/json",
                    Body = responseBody
                };

                await _store.SetAsync(
                    $"{context.Request.Path}:{idempotencyKey}",
                    responseToCache,
                    TimeSpan.FromHours(24));
            }
        }
        finally
        {
            memoryStream.Seek(0, SeekOrigin.Begin);
            await memoryStream.CopyToAsync(originalBody);
            context.Response.Body = originalBody;
        }
    }
}

public record CachedResponse
{
    public int StatusCode { get; set; }
    public string ContentType { get; set; } = string.Empty;
    public string Body { get; set; } = string.Empty;
}

// Program.cs'e kayıt
// app.UseMiddleware<IdempotencyMiddleware>();
```

---

### 3. Soru

**Idempotency Filter (Action Filter) ile endpoint bazlı idempotency nasıl uygulanır?**

**Cevap:**

Action Filter yaklaşımı, middleware'den daha granüler bir kontrol sağlar. Yalnızca `[Idempotent]` attribute'u ile işaretlenen endpoint'lere idempotency mantığı uygulanır. Bu yaklaşım, her endpoint'in kendi idempotency süresini ve davranışını yapılandırabilmesine olanak tanır.

**Örnek Kod:**

```csharp
// Idempotent Attribute
[AttributeUsage(AttributeTargets.Method | AttributeTargets.Class)]
public class IdempotentAttribute : Attribute
{
    public int TtlHours { get; set; } = 24;
    public bool Required { get; set; } = true;
}

// Action Filter
public class IdempotencyFilter : IAsyncActionFilter
{
    private readonly IIdempotencyStore _store;
    private readonly ILogger<IdempotencyFilter> _logger;

    public IdempotencyFilter(IIdempotencyStore store, ILogger<IdempotencyFilter> logger)
    {
        _store = store;
        _logger = logger;
    }

    public async Task OnActionExecutionAsync(
        ActionExecutingContext context,
        ActionExecutionDelegate next)
    {
        var attribute = context.ActionDescriptor
            .EndpointMetadata
            .OfType<IdempotentAttribute>()
            .FirstOrDefault();

        if (attribute == null)
        {
            await next();
            return;
        }

        var request = context.HttpContext.Request;

        if (!request.Headers.TryGetValue("Idempotency-Key", out var keyValues)
            || string.IsNullOrWhiteSpace(keyValues.FirstOrDefault()))
        {
            if (attribute.Required)
            {
                context.Result = new BadRequestObjectResult(new ProblemDetails
                {
                    Title = "Idempotency-Key Eksik",
                    Detail = "Bu endpoint için Idempotency-Key başlığı zorunludur.",
                    Status = 400
                });
                return;
            }

            await next();
            return;
        }

        var idempotencyKey = $"{request.Path}:{keyValues.First()!.Trim()}";

        // Cache kontrolü
        var cachedResult = await _store.GetAsync<IdempotentActionResult>(idempotencyKey);
        if (cachedResult != null)
        {
            _logger.LogInformation(
                "Idempotent yanıt: {Key}", idempotencyKey);

            context.HttpContext.Response.Headers["X-Idempotency-Replayed"] = "true";
            context.Result = new ObjectResult(cachedResult.Value)
            {
                StatusCode = cachedResult.StatusCode
            };
            return;
        }

        // Eylemi çalıştır
        var executedContext = await next();

        // Başarılı sonucu önbelleğe al
        if (executedContext.Exception == null
            && executedContext.Result is ObjectResult objectResult
            && objectResult.StatusCode is >= 200 and < 300)
        {
            var toCache = new IdempotentActionResult
            {
                Value = objectResult.Value,
                StatusCode = objectResult.StatusCode ?? 200
            };

            await _store.SetAsync(
                idempotencyKey,
                toCache,
                TimeSpan.FromHours(attribute.TtlHours));
        }
    }
}

public record IdempotentActionResult
{
    public object? Value { get; set; }
    public int StatusCode { get; set; }
}

// Controller'da kullanım
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;

    public OrdersController(IOrderService orderService)
    {
        _orderService = orderService;
    }

    [HttpPost]
    [Idempotent(TtlHours = 48)]
    public async Task<ActionResult<OrderResponse>> CreateOrderAsync(
        [FromBody] CreateOrderRequest request)
    {
        var order = await _orderService.CreateOrderAsync(request);
        return CreatedAtAction(
            nameof(GetOrderAsync),
            new { id = order.Id },
            order);
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<OrderResponse>> GetOrderAsync(string id)
    {
        var order = await _orderService.GetOrderAsync(id);
        return order == null ? NotFound() : Ok(order);
    }
}

// Program.cs
// builder.Services.AddScoped<IdempotencyFilter>();
// builder.Services.AddControllers(options =>
//     options.Filters.AddService<IdempotencyFilter>());
```

---

### 4. Soru

**At-Least-Once, At-Most-Once ve Exactly-Once teslimat farkları nelerdir?**

**Cevap:**

- **At-Most-Once**: Mesaj gönderilir, alındı onayı beklenmez. Mesaj kaybolabilir. En basit ama güvenilirliği en düşük yaklaşım. Log/metrics gibi kayıp tolere edilebilen durumlar için.
- **At-Least-Once**: Mesajın iletildiği garanti edilir; ancak tekrar iletilebilir. Tüketici idempotent olmalıdır. Kafka'nın varsayılan davranışı.
- **Exactly-Once**: Mesaj tam olarak bir kez işlenir. Broker ve tüketici tarafında idempotency + transaction gerektirir. Kafka ile Transactional Producer/Consumer kullanılır.

**Örnek Kod:**

```csharp
// At-Least-Once: Idempotent tüketici ile
public class OrderCreatedConsumer : IConsumer<OrderCreatedEvent>
{
    private readonly IOrderRepository _orderRepository;
    private readonly IProcessedMessageRepository _processedMessages;
    private readonly ILogger<OrderCreatedConsumer> _logger;

    public OrderCreatedConsumer(
        IOrderRepository orderRepository,
        IProcessedMessageRepository processedMessages,
        ILogger<OrderCreatedConsumer> logger)
    {
        _orderRepository = orderRepository;
        _processedMessages = processedMessages;
        _logger = logger;
    }

    public async Task ConsumeAsync(OrderCreatedEvent message, string messageId)
    {
        // Mesaj daha önce işlendi mi? (At-Least-Once için idempotency)
        if (await _processedMessages.ExistsAsync(messageId))
        {
            _logger.LogInformation(
                "Mesaj {MessageId} zaten işlenmiş, atlanıyor.", messageId);
            return;
        }

        try
        {
            // İşlemi ve mesaj takibini tek transaction'da yap
            await using var tx = await _orderRepository.BeginTransactionAsync();

            // Siparişi işle
            await _orderRepository.CreateAsync(new Order
            {
                Id = message.OrderId,
                CustomerId = message.CustomerId,
                Items = message.Items,
                TotalAmount = message.TotalAmount,
                Status = OrderStatus.Created
            }, tx);

            // Mesajı işlenmiş olarak işaretle
            await _processedMessages.MarkAsync(messageId, tx);

            await tx.CommitAsync();

            _logger.LogInformation(
                "Sipariş {OrderId} başarıyla oluşturuldu.", message.OrderId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Sipariş işleme hatası: {OrderId}", message.OrderId);
            throw; // Retry için yeniden fırlat
        }
    }
}

// Outbox Pattern ile Exactly-Once benzeri güvence
public class OrderService
{
    private readonly IDbContext _dbContext;
    private readonly ILogger<OrderService> _logger;

    public OrderService(IDbContext dbContext, ILogger<OrderService> logger)
    {
        _dbContext = dbContext;
        _logger = logger;
    }

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        await using var transaction = await _dbContext.Database.BeginTransactionAsync();

        try
        {
            // Sipariş oluştur
            var order = new Order
            {
                Id = Guid.NewGuid().ToString(),
                CustomerId = request.CustomerId,
                Items = request.Items,
                TotalAmount = request.TotalAmount,
                Status = OrderStatus.Created,
                CreatedAt = DateTimeOffset.UtcNow
            };
            _dbContext.Orders.Add(order);

            // Outbox mesajı ekle (aynı transaction içinde)
            var outboxMessage = new OutboxMessage
            {
                Id = Guid.NewGuid().ToString(),
                EventType = nameof(OrderCreatedEvent),
                Payload = JsonSerializer.Serialize(new OrderCreatedEvent
                {
                    OrderId = order.Id,
                    CustomerId = order.CustomerId,
                    TotalAmount = order.TotalAmount
                }),
                CreatedAt = DateTimeOffset.UtcNow,
                ProcessedAt = null
            };
            _dbContext.OutboxMessages.Add(outboxMessage);

            // Tek atomik işlem: sipariş + outbox mesajı birlikte kaydedilir
            await _dbContext.SaveChangesAsync();
            await transaction.CommitAsync();

            return order;
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}

// Outbox Processor: Mesajları broker'a iletir
public class OutboxProcessor : BackgroundService
{
    private readonly IDbContext _dbContext;
    private readonly IMessageBroker _messageBroker;
    private readonly ILogger<OutboxProcessor> _logger;

    public OutboxProcessor(
        IDbContext dbContext,
        IMessageBroker messageBroker,
        ILogger<OutboxProcessor> logger)
    {
        _dbContext = dbContext;
        _messageBroker = messageBroker;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await ProcessOutboxMessagesAsync(stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Outbox işleme hatası.");
            }

            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }

    private async Task ProcessOutboxMessagesAsync(CancellationToken cancellationToken)
    {
        var pending = await _dbContext.OutboxMessages
            .Where(m => m.ProcessedAt == null)
            .OrderBy(m => m.CreatedAt)
            .Take(100)
            .ToListAsync(cancellationToken);

        foreach (var message in pending)
        {
            try
            {
                await _messageBroker.PublishAsync(
                    message.EventType,
                    message.Payload,
                    cancellationToken);

                message.ProcessedAt = DateTimeOffset.UtcNow;
                await _dbContext.SaveChangesAsync(cancellationToken);

                _logger.LogDebug(
                    "Outbox mesajı gönderildi: {MessageId}", message.Id);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex,
                    "Outbox mesajı gönderilemedi: {MessageId}", message.Id);
            }
        }
    }
}
```

---

### 5. Soru

**Idempotency Key nasıl üretilmeli ve istemci tarafında nasıl yönetilmeli?**

**Cevap:**

Idempotency key, istemci tarafından üretilmeli ve her benzersiz işlem için farklı olmalıdır. UUID v4 en yaygın kullanılan formattır. İstemci, başarılı yanıt alana kadar aynı key ile yeniden deneme yapabilir. Farklı işlemler için farklı key kullanılmalıdır. Key, kullanıcı oturumu ve işlem tipiyle ilişkilendirilerek üretilebilir.

**Örnek Kod:**

```csharp
// İstemci tarafı: Idempotency Key yönetimi
public class IdempotentHttpClient
{
    private readonly HttpClient _httpClient;
    private readonly ILocalStorage _localStorage;
    private readonly ILogger<IdempotentHttpClient> _logger;

    public IdempotentHttpClient(
        HttpClient httpClient,
        ILocalStorage localStorage,
        ILogger<IdempotentHttpClient> logger)
    {
        _httpClient = httpClient;
        _localStorage = localStorage;
        _logger = logger;
    }

    public async Task<T?> PostWithIdempotencyAsync<T>(
        string endpoint,
        object payload,
        string operationKey,
        int maxRetries = 3,
        CancellationToken cancellationToken = default)
    {
        // Yerel depolamadan mevcut idempotency key'i al veya yeni üret
        var storageKey = $"idempotency:{endpoint}:{operationKey}";
        var idempotencyKey = await _localStorage.GetAsync<string>(storageKey)
                             ?? Guid.NewGuid().ToString("N");

        // Key'i sakla (yeniden denemeler için)
        await _localStorage.SetAsync(storageKey, idempotencyKey,
            TimeSpan.FromHours(24));

        var json = JsonSerializer.Serialize(payload);
        var lastException = default(Exception);

        for (int attempt = 1; attempt <= maxRetries; attempt++)
        {
            try
            {
                using var request = new HttpRequestMessage(HttpMethod.Post, endpoint)
                {
                    Content = new StringContent(json, Encoding.UTF8, "application/json")
                };
                request.Headers.Add("Idempotency-Key", idempotencyKey);

                var response = await _httpClient.SendAsync(request, cancellationToken);

                // Başarılı veya istemci hatası (4xx) → retry etme
                if (response.IsSuccessStatusCode || (int)response.StatusCode < 500)
                {
                    // Başarılı olunca key'i temizle
                    if (response.IsSuccessStatusCode)
                        await _localStorage.RemoveAsync(storageKey);

                    return await response.Content.ReadFromJsonAsync<T>(
                        cancellationToken: cancellationToken);
                }

                // Sunucu hatası (5xx) → exponential backoff ile retry
                var delay = TimeSpan.FromMilliseconds(Math.Pow(2, attempt) * 100);
                _logger.LogWarning(
                    "İstek başarısız (deneme {Attempt}/{Max}). {Delay}ms sonra tekrar.",
                    attempt, maxRetries, delay.TotalMilliseconds);

                await Task.Delay(delay, cancellationToken);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                lastException = ex;
                _logger.LogWarning(ex,
                    "İstek hatası (deneme {Attempt}/{Max}).", attempt, maxRetries);

                if (attempt < maxRetries)
                {
                    var delay = TimeSpan.FromMilliseconds(Math.Pow(2, attempt) * 100);
                    await Task.Delay(delay, cancellationToken);
                }
            }
        }

        throw new HttpRequestException(
            $"{maxRetries} denemeden sonra istek başarısız oldu.",
            lastException);
    }
}

// Sunucu tarafı: İşlem bazlı benzersiz key üretimi
public class IdempotencyKeyGenerator
{
    // Kullanıcı + işlem tipi + içerik hash'i ile deterministik key
    public static string GenerateFromContent(
        string userId,
        string operationType,
        object content)
    {
        var json = JsonSerializer.Serialize(content);
        var input = $"{userId}:{operationType}:{json}";

        using var sha256 = SHA256.Create();
        var hash = sha256.ComputeHash(Encoding.UTF8.GetBytes(input));
        return Convert.ToHexString(hash).ToLowerInvariant()[..32];
    }

    // UUID v4 tabanlı rastgele key
    public static string GenerateRandom() => Guid.NewGuid().ToString("N");

    // Zaman tabanlı key (aynı dakikadaki işlemleri gruplar)
    public static string GenerateTimeBased(string userId, string operationType)
    {
        var minute = DateTimeOffset.UtcNow.ToString("yyyyMMddHHmm");
        return $"{userId}:{operationType}:{minute}";
    }
}
```

---

### 6. Soru

**Kafka'da Exactly-Once semantics nasıl sağlanır?**

**Cevap:**

Kafka, Exactly-Once Semantics (EOS) için Transactional Producer ve Consumer kullanır. Producer her başladığında benzersiz bir `transactional.id` ile tanımlanır. Kafka broker'ı bu ID aracılığıyla aynı producer'dan gelen tekrar eden mesajları (idempotent producer) ve birden fazla partition'a yayılan atomik yazmaları (transactional write) yönetir. Consumer tarafında `isolation.level=read_committed` ayarı ile yalnızca taahhüt edilmiş mesajlar okunur.

**Örnek Kod:**

```csharp
// Confluent.Kafka ile Exactly-Once Producer
public class ExactlyOnceKafkaProducer : IAsyncDisposable
{
    private readonly IProducer<string, string> _producer;
    private readonly ILogger<ExactlyOnceKafkaProducer> _logger;

    public ExactlyOnceKafkaProducer(
        string bootstrapServers,
        string transactionalId,
        ILogger<ExactlyOnceKafkaProducer> logger)
    {
        var config = new ProducerConfig
        {
            BootstrapServers = bootstrapServers,
            TransactionalId = transactionalId,       // Benzersiz producer kimliği
            EnableIdempotence = true,                 // Broker tarafı duplicate engelleme
            Acks = Acks.All,                          // Tüm in-sync replikalar onaylasın
            MaxInFlight = 5                           // Idempotency için max 5 uçuşta mesaj
        };

        _producer = new ProducerBuilder<string, string>(config).Build();
        _producer.InitTransactions(TimeSpan.FromSeconds(30));

        _logger = logger;
    }

    public async Task SendExactlyOnceAsync(
        IEnumerable<(string Topic, string Key, string Value)> messages,
        CancellationToken cancellationToken = default)
    {
        _producer.BeginTransaction();

        try
        {
            foreach (var (topic, key, value) in messages)
            {
                await _producer.ProduceAsync(topic, new Message<string, string>
                {
                    Key = key,
                    Value = value
                }, cancellationToken);
            }

            _producer.CommitTransaction();
            _logger.LogDebug("Transaction başarıyla commit edildi.");
        }
        catch (Exception ex)
        {
            _producer.AbortTransaction();
            _logger.LogError(ex, "Transaction abort edildi.");
            throw;
        }
    }

    public async ValueTask DisposeAsync()
    {
        _producer.Flush(TimeSpan.FromSeconds(10));
        _producer.Dispose();
        await ValueTask.CompletedTask;
    }
}

// Exactly-Once Consumer: Read-Process-Write döngüsü
public class ExactlyOnceKafkaConsumer : BackgroundService
{
    private readonly IConsumer<string, string> _consumer;
    private readonly IOrderRepository _orderRepository;
    private readonly ILogger<ExactlyOnceKafkaConsumer> _logger;

    public ExactlyOnceKafkaConsumer(
        string bootstrapServers,
        string groupId,
        IOrderRepository orderRepository,
        ILogger<ExactlyOnceKafkaConsumer> logger)
    {
        var config = new ConsumerConfig
        {
            BootstrapServers = bootstrapServers,
            GroupId = groupId,
            IsolationLevel = IsolationLevel.ReadCommitted, // Sadece committed mesajları oku
            EnableAutoCommit = false,                       // Manuel offset commit
            AutoOffsetReset = AutoOffsetReset.Earliest
        };

        _consumer = new ConsumerBuilder<string, string>(config).Build();
        _orderRepository = orderRepository;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _consumer.Subscribe("order-events");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                var consumeResult = _consumer.Consume(stoppingToken);
                if (consumeResult == null) continue;

                await ProcessMessageAsync(consumeResult);

                // İşlem başarılı olunca offset commit et
                _consumer.Commit(consumeResult);
            }
            catch (OperationCanceledException) { break; }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Mesaj işleme hatası.");
                // Offset commit etme → mesaj tekrar tüketilecek (at-least-once fallback)
            }
        }
    }

    private async Task ProcessMessageAsync(ConsumeResult<string, string> result)
    {
        var orderEvent = JsonSerializer.Deserialize<OrderCreatedEvent>(result.Message.Value);
        if (orderEvent == null) return;

        // Idempotent işlem: mevcut kontrolü
        var exists = await _orderRepository.ExistsAsync(orderEvent.OrderId);
        if (!exists)
        {
            await _orderRepository.CreateAsync(new Order
            {
                Id = orderEvent.OrderId,
                CustomerId = orderEvent.CustomerId,
                TotalAmount = orderEvent.TotalAmount,
                Status = OrderStatus.Created
            });
        }
    }

    public override void Dispose()
    {
        _consumer.Close();
        _consumer.Dispose();
        base.Dispose();
    }
}
```

---

### 7. Soru

**Idempotency ve transaction arasındaki fark nedir? Her ikisini birlikte nasıl kullanırsınız?**

**Cevap:**

- **Transaction**: Birden fazla işlemin tümünün veya hiçbirinin uygulanmamasını garanti eder (atomicity). Hata durumunda geri alınır.
- **Idempotency**: Aynı işlemin birden fazla kez çalıştırılmasının aynı sonucu üretmesini garanti eder.

İkisi birbirini tamamlar. Transaction, tek bir deneme içinde atomicity sağlarken; idempotency, yeniden denemeler boyunca tutarlılık sağlar. En güçlü güvence için her ikisi birlikte kullanılmalıdır: işlem idempotency key ile kontrol edilir, işlemin kendisi transaction içinde gerçekleştirilir.

**Örnek Kod:**

```csharp
// Transactional Idempotency: En güçlü güvence
public class TransactionalIdempotencyService
{
    private readonly IDbContext _dbContext;
    private readonly ILogger<TransactionalIdempotencyService> _logger;

    public TransactionalIdempotencyService(
        IDbContext dbContext,
        ILogger<TransactionalIdempotencyService> logger)
    {
        _dbContext = dbContext;
        _logger = logger;
    }

    public async Task<TResult> ExecuteIdempotentlyAsync<TResult>(
        string idempotencyKey,
        Func<Task<TResult>> operation,
        TimeSpan? ttl = null)
    {
        // Transaction içinde idempotency kontrolü + işlem
        var strategy = _dbContext.Database.CreateExecutionStrategy();

        return await strategy.ExecuteAsync(async () =>
        {
            await using var transaction = await _dbContext.Database.BeginTransactionAsync(
                IsolationLevel.RepeatableRead);

            try
            {
                // SELECT FOR UPDATE: eş zamanlı istekleri serialize et
                var existing = await _dbContext.IdempotencyRecords
                    .FromSqlRaw(
                        "SELECT * FROM idempotency_records WHERE key = {0} FOR UPDATE",
                        idempotencyKey)
                    .FirstOrDefaultAsync();

                if (existing != null)
                {
                    _logger.LogInformation(
                        "Idempotency key {Key} zaten mevcut.", idempotencyKey);
                    await transaction.RollbackAsync();
                    return JsonSerializer.Deserialize<TResult>(existing.ResultJson)!;
                }

                // İşlemi çalıştır
                var result = await operation();

                // Sonucu kaydet
                _dbContext.IdempotencyRecords.Add(new IdempotencyRecord
                {
                    Key = idempotencyKey,
                    ResultJson = JsonSerializer.Serialize(result),
                    CreatedAt = DateTimeOffset.UtcNow,
                    ExpiresAt = DateTimeOffset.UtcNow.Add(ttl ?? TimeSpan.FromHours(24))
                });

                await _dbContext.SaveChangesAsync();
                await transaction.CommitAsync();

                return result;
            }
            catch
            {
                await transaction.RollbackAsync();
                throw;
            }
        });
    }
}

public class IdempotencyRecord
{
    public string Key { get; set; } = string.Empty;
    public string ResultJson { get; set; } = string.Empty;
    public DateTimeOffset CreatedAt { get; set; }
    public DateTimeOffset ExpiresAt { get; set; }
}

// Kullanım örneği
public class CheckoutService
{
    private readonly TransactionalIdempotencyService _idempotencyService;
    private readonly IPaymentGateway _paymentGateway;

    public CheckoutService(
        TransactionalIdempotencyService idempotencyService,
        IPaymentGateway paymentGateway)
    {
        _idempotencyService = idempotencyService;
        _paymentGateway = paymentGateway;
    }

    public async Task<CheckoutResult> CheckoutAsync(
        CheckoutRequest request,
        string idempotencyKey)
    {
        return await _idempotencyService.ExecuteIdempotentlyAsync(
            idempotencyKey,
            async () =>
            {
                var paymentResult = await _paymentGateway.ChargeAsync(
                    request.CustomerId,
                    request.Amount,
                    request.Currency);

                return new CheckoutResult
                {
                    OrderId = Guid.NewGuid().ToString(),
                    PaymentId = paymentResult.PaymentId,
                    Status = CheckoutStatus.Completed,
                    TotalAmount = request.Amount
                };
            },
            TimeSpan.FromHours(48));
    }
}
```

---

## Best Practices

### Idempotency Key Tasarımı
- UUID v4 formatını tercih edin (`Guid.NewGuid().ToString("N")`)
- İstemci tarafında üretin, sunucu tarafında kabul edin
- 24-72 saat TTL uygulayın
- İş mantığına göre belirleyici (deterministic) key üretilebilir

### Depolama Stratejisi
- **Redis**: Hızlı, TTL destekli, distributed cache için idealdir
- **Veritabanı**: Daha güçlü garantiler, transaction desteği
- **Hybrid**: Redis önce kontrol, fallback veritabanı

### Monitoring
- Idempotency cache hit rate'ini izleyin
- Replay edilen istek oranını takip edin
- TTL süresi dolmuş key erişimlerini loglayın

## Kaynaklar

- [Idempotent REST APIs - Stripe Engineering](https://stripe.com/blog/idempotency)
- [Exactly-Once Semantics in Kafka](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/)
- [The Outbox Pattern - Chris Richardson](https://microservices.io/patterns/data/transactional-outbox.html)
- [Designing Data-Intensive Applications - Martin Kleppmann](https://dataintensive.net/)
- [Idempotency Keys - IETF Draft](https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-idempotency-key-header)
