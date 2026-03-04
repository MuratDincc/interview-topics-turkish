# CAP Teoremi

## Genel Bakış

CAP Teoremi (Brewer Teoremi), dağıtık sistemlerin temel kısıtlamalarını tanımlar. Eric Brewer tarafından 2000 yılında öne sürülen ve 2002 yılında Gilbert ile Lynch tarafından kanıtlanan bu teorem, dağıtık bir sistemin aynı anda üç garantiyi birden sağlayamayacağını ifade eder:

- **C (Consistency - Tutarlılık)**: Tüm node'lar aynı anda aynı veriyi görür.
- **A (Availability - Kullanılabilirlik)**: Her istek bir yanıt alır (başarılı ya da başarısız).
- **P (Partition Tolerance - Bölünme Toleransı)**: Sistem, ağ bölünmelerine (network partition) rağmen çalışmaya devam eder.

Gerçek dünya dağıtık sistemlerinde ağ bölünmeleri kaçınılmazdır. Dolayısıyla pratik seçim **CP** (Tutarlılık + Bölünme Toleransı) veya **AP** (Kullanılabilirlik + Bölünme Toleransı) arasındadır.

---

## CP Sistemler

CP sistemler, ağ bölünmesi durumunda tutarlılığı korumak için kullanılabilirliği feda eder. Sistem, tutarsız veri döndürmek yerine hata verir ya da yanıt vermez.

**Örnekler**: SQL Server (distributed modda), HBase, ZooKeeper, etcd

---

## AP Sistemler

AP sistemler, ağ bölünmesi durumunda yanıt vermeye devam eder; ancak veriler node'lar arasında geçici olarak farklı olabilir (eventual consistency).

**Örnekler**: Cassandra, CouchDB, DynamoDB, Riak

---

## Nihai Tutarlılık (Eventual Consistency)

AP sistemlerde en yaygın tutarlılık modeli "eventual consistency"dir. Sistem, tüm güncellemeler sonunda tüm node'lara yayılacağını garanti eder; ancak belirli bir zaman çerçevesi sunmaz. Bu süre içinde farklı node'lar farklı veriler döndürebilir.

---

## Mülakat Soruları

### 1. Soru

**CAP teoremi nedir ve neden önemlidir?**

**Cevap:**

CAP teoremi, bir dağıtık sistemin Consistency (Tutarlılık), Availability (Kullanılabilirlik) ve Partition Tolerance (Bölünme Toleransı) garantilerinin üçünü birden sağlayamayacağını belirtir. Pratik sistemlerde ağ bölünmeleri kaçınılmaz olduğundan, tasarımcılar CP veya AP tercihini yapmak zorundadır. Bu teorem, hangi veritabanı veya mimariyi seçeceğimizi belirleyen temel bir çerçeve sunar.

- **CP tercih edildiğinde**: Bankacılık, envanter yönetimi gibi kesin doğruluğun kritik olduğu sistemler
- **AP tercih edildiğinde**: Sosyal medya akışları, öneri sistemleri gibi geçici tutarsızlığın kabul edilebilir olduğu sistemler

**Örnek Kod:**

```csharp
// CP Sistemi: SQL Server ile dağıtık tutarlılık
// Serialize edilebilir izolasyon seviyesi ile yüksek tutarlılık
public class CpBankingService
{
    private readonly string _connectionString;

    public CpBankingService(string connectionString)
    {
        _connectionString = connectionString;
    }

    public async Task<bool> TransferMoneyAsync(int fromAccountId, int toAccountId, decimal amount)
    {
        using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();

        // Serializable isolation: en yüksek tutarlılık seviyesi
        using var transaction = connection.BeginTransaction(IsolationLevel.Serializable);
        try
        {
            // Kaynak hesap bakiyesini kontrol et ve kilitle
            var fromBalance = await connection.QuerySingleAsync<decimal>(
                "SELECT Balance FROM Accounts WITH (UPDLOCK, ROWLOCK) WHERE Id = @Id",
                new { Id = fromAccountId },
                transaction);

            if (fromBalance < amount)
                throw new InsufficientFundsException("Yetersiz bakiye.");

            // Atomic güncelleme
            await connection.ExecuteAsync(
                "UPDATE Accounts SET Balance = Balance - @Amount WHERE Id = @Id",
                new { Amount = amount, Id = fromAccountId },
                transaction);

            await connection.ExecuteAsync(
                "UPDATE Accounts SET Balance = Balance + @Amount WHERE Id = @Id",
                new { Amount = amount, Id = toAccountId },
                transaction);

            await transaction.CommitAsync();
            return true;
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}
```

---

### 2. Soru

**SQL Server, Cassandra ve MongoDB CAP teoremi açısından nasıl konumlanır?**

**Cevap:**

- **SQL Server**: Varsayılan olarak CA sistemdir; ancak Always On veya distributed deployment'ta CP eğilimlidir. Tutarlılığı her zaman ön planda tutar.
- **Cassandra**: AP sistemidir. Tüm node'lar eşit önemdedir (leaderless), yazma işlemleri birden fazla node'a yapılır, ağ bölünmesinde yazma/okuma devam eder ancak veriler geçici olarak farklı olabilir.
- **MongoDB**: Varsayılan ayarlarla CP sistemidir. Primary node yazmaları kabul eder; Primary kaybolursa yeni seçim yapılana kadar sistem yazmalara kapalıdır.

**Örnek Kod:**

```csharp
// Cassandra ile AP: Tunable Consistency
public class CassandraApService
{
    private readonly ISession _session;

    public CassandraApService(ISession session)
    {
        _session = session;
    }

    // ConsistencyLevel.ONE: En az 1 node yanıt versin (AP tarafı - yüksek erişilebilirlik)
    public async Task<UserProfile?> GetUserProfileWithHighAvailabilityAsync(Guid userId)
    {
        var statement = new SimpleStatement(
            "SELECT * FROM user_profiles WHERE user_id = ?", userId)
            .SetConsistencyLevel(ConsistencyLevel.One);

        var row = await _session.ExecuteAsync(statement);
        var result = row.FirstOrDefault();

        if (result == null) return null;

        return new UserProfile
        {
            Id = result.GetValue<Guid>("user_id"),
            Name = result.GetValue<string>("name"),
            Email = result.GetValue<string>("email")
        };
    }

    // ConsistencyLevel.QUORUM: Çoğunluk node'dan yanıt al (CP tarafı - yüksek tutarlılık)
    public async Task WriteUserProfileWithConsistencyAsync(UserProfile profile)
    {
        var statement = new SimpleStatement(
            "INSERT INTO user_profiles (user_id, name, email, updated_at) VALUES (?, ?, ?, ?)",
            profile.Id, profile.Name, profile.Email, DateTimeOffset.UtcNow)
            .SetConsistencyLevel(ConsistencyLevel.Quorum);

        await _session.ExecuteAsync(statement);
    }
}

// MongoDB ile CP: Primary Node Yazma Garantisi
public class MongoDbCpService
{
    private readonly IMongoCollection<Order> _orders;

    public MongoDbCpService(IMongoDatabase database)
    {
        // WriteConcern.WMajority: Çoğunluk node'a yazılana kadar bekle
        _orders = database
            .WithWriteConcern(WriteConcern.WMajority)
            .WithReadConcern(ReadConcern.Majority)
            .GetCollection<Order>("orders");
    }

    public async Task<string> CreateOrderAsync(Order order)
    {
        // Primary node'a yazılır, majority node'a replike edilene kadar bloke olur
        await _orders.InsertOneAsync(order);
        return order.Id;
    }

    public async Task<Order?> GetOrderAsync(string orderId)
    {
        // ReadConcern.Majority: Çoğunluk node tarafından onaylanan veriyi oku
        return await _orders
            .Find(o => o.Id == orderId)
            .FirstOrDefaultAsync();
    }
}
```

---

### 3. Soru

**Eventual Consistency nedir ve C# ile nasıl yönetilir?**

**Cevap:**

Eventual Consistency (Nihai Tutarlılık), dağıtık sistemde bir güncelleme yapıldıktan sonra tüm node'ların eninde sonunda aynı değere ulaşacağını garanti eden bir modeldir. Bu süre zarfında farklı node'lar farklı değerler döndürebilir. Uygulama katmanında bu durumu yönetmek için read-your-writes, monotonic reads gibi session garantileri veya versiyon çakışmalarını çözmek için conflict resolution stratejileri kullanılır.

**Örnek Kod:**

```csharp
// Eventual Consistency: Versiyon tabanlı çakışma çözümü (Last-Write-Wins)
public class EventuallyConsistentCacheService
{
    private readonly IDistributedCache _distributedCache;
    private readonly ILogger<EventuallyConsistentCacheService> _logger;

    public EventuallyConsistentCacheService(
        IDistributedCache distributedCache,
        ILogger<EventuallyConsistentCacheService> logger)
    {
        _distributedCache = distributedCache;
        _logger = logger;
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan? expiry = null)
    {
        var versioned = new VersionedValue<T>
        {
            Value = value,
            Version = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds(),
            NodeId = Environment.MachineName
        };

        var json = JsonSerializer.Serialize(versioned);
        var options = new DistributedCacheEntryOptions();
        if (expiry.HasValue) options.AbsoluteExpirationRelativeToNow = expiry;

        await _distributedCache.SetStringAsync(key, json, options);
    }

    public async Task<T?> GetAsync<T>(string key)
    {
        var json = await _distributedCache.GetStringAsync(key);
        if (json == null) return default;

        var versioned = JsonSerializer.Deserialize<VersionedValue<T>>(json);
        return versioned != null ? versioned.Value : default;
    }

    // Read-your-writes: Yazılan değerin hemen okunabilmesi için local cache
    public async Task<T?> GetWithReadYourWritesAsync<T>(
        string key,
        Dictionary<string, string> localWriteCache)
    {
        // Önce local write cache'e bak (session garantisi)
        if (localWriteCache.TryGetValue(key, out var localJson))
        {
            var local = JsonSerializer.Deserialize<VersionedValue<T>>(localJson);
            return local != null ? local.Value : default;
        }

        return await GetAsync<T>(key);
    }
}

public class VersionedValue<T>
{
    public T Value { get; set; } = default!;
    public long Version { get; set; }
    public string NodeId { get; set; } = string.Empty;
}

// Conflict Resolution: Vector Clock ile çakışma tespiti
public class VectorClock
{
    private readonly Dictionary<string, long> _clock = new();

    public void Increment(string nodeId)
    {
        _clock[nodeId] = _clock.GetValueOrDefault(nodeId) + 1;
    }

    public bool HappensBefore(VectorClock other)
    {
        return _clock.All(kv =>
            other._clock.GetValueOrDefault(kv.Key) >= kv.Value)
            && _clock.Any(kv =>
            other._clock.GetValueOrDefault(kv.Key) > kv.Value);
    }

    public bool IsConcurrentWith(VectorClock other)
    {
        return !HappensBefore(other) && !other.HappensBefore(this);
    }

    public static VectorClock Merge(VectorClock a, VectorClock b)
    {
        var merged = new VectorClock();
        var allKeys = a._clock.Keys.Union(b._clock.Keys);
        foreach (var key in allKeys)
        {
            merged._clock[key] = Math.Max(
                a._clock.GetValueOrDefault(key),
                b._clock.GetValueOrDefault(key));
        }
        return merged;
    }
}
```

---

### 4. Soru

**PACELC teoremi nedir? CAP teoreminden nasıl farklıdır?**

**Cevap:**

PACELC teoremi, CAP teoremini genişletir. Ağ bölünmesi (Partition) olmasa bile, dağıtık sistemlerin Latency (Gecikme) ve Consistency (Tutarlılık) arasında denge kurması gerektiğini söyler:

- **PAC**: Partition varsa → Availability veya Consistency seç
- **ELC**: Else (Partition yoksa) → Latency veya Consistency seç

Örneğin DynamoDB'de ReadConsistency seçeneği ile düşük gecikme (eventual consistency) veya güçlü tutarlılık (strong consistency) arasında tercih yapılabilir.

**Örnek Kod:**

```csharp
// PACELC: Latency vs Consistency seçimi
public class PacelcDemoService
{
    private readonly IAmazonDynamoDB _dynamoDb;

    public PacelcDemoService(IAmazonDynamoDB dynamoDb)
    {
        _dynamoDb = dynamoDb;
    }

    // EL tarafı: Düşük gecikme, eventual consistency
    // Hızlı okuma ama stale veri olabilir
    public async Task<string?> GetItemEventuallyConsistentAsync(string tableName, string key)
    {
        var request = new GetItemRequest
        {
            TableName = tableName,
            Key = new Dictionary<string, AttributeValue>
            {
                ["pk"] = new AttributeValue { S = key }
            },
            ConsistentRead = false  // Eventual consistency (daha hızlı, daha ucuz)
        };

        var response = await _dynamoDb.GetItemAsync(request);
        return response.Item.TryGetValue("value", out var val) ? val.S : null;
    }

    // EC tarafı: Yüksek gecikme, strong consistency
    // Daha yavaş ama her zaman güncel veri
    public async Task<string?> GetItemStronglyConsistentAsync(string tableName, string key)
    {
        var request = new GetItemRequest
        {
            TableName = tableName,
            Key = new Dictionary<string, AttributeValue>
            {
                ["pk"] = new AttributeValue { S = key }
            },
            ConsistentRead = true  // Strong consistency (daha yavaş, iki kat RCU)
        };

        var response = await _dynamoDb.GetItemAsync(request);
        return response.Item.TryGetValue("value", out var val) ? val.S : null;
    }
}
```

---

### 5. Soru

**Distributed sistemlerde Read-Your-Writes ve Monotonic Reads garantileri nedir?**

**Cevap:**

- **Read-Your-Writes**: Bir kullanıcı yazdıktan sonra kendi yazdığı değeri okuyabilmesini garanti eder. Örneğin profil güncelledikten sonra sayfayı yenilediğinde güncel profili görmelidir.
- **Monotonic Reads**: Kullanıcı bir değeri bir kez okuduktan sonra daha eski bir değer görmez. Zaman içinde geri gitmez.
- **Monotonic Writes**: Yazma işlemleri, gönderildiği sırayla işlenir.
- **Writes-Follow-Reads**: Okunan bir değere göre yapılan yazma, okuma ile kausal ilişki içindedir.

**Örnek Kod:**

```csharp
// Read-Your-Writes: Sticky session veya session token ile
public class ReadYourWritesService
{
    private readonly IDistributedCache _cache;
    private readonly IUserRepository _userRepository;

    public ReadYourWritesService(
        IDistributedCache cache,
        IUserRepository userRepository)
    {
        _cache = cache;
        _userRepository = userRepository;
    }

    public async Task UpdateProfileAsync(string userId, UserProfile profile)
    {
        await _userRepository.UpdateAsync(profile);

        // Kullanıcının kendi oturumu için yazılanı önbelleğe al
        var cacheKey = $"user:profile:{userId}:session";
        var json = JsonSerializer.Serialize(profile);
        await _cache.SetStringAsync(cacheKey, json,
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
            });
    }

    public async Task<UserProfile?> GetProfileAsync(string userId, string sessionId)
    {
        // Önce oturum bazlı cache'e bak (Read-Your-Writes garantisi)
        var sessionKey = $"user:profile:{userId}:session";
        var cached = await _cache.GetStringAsync(sessionKey);

        if (cached != null)
        {
            return JsonSerializer.Deserialize<UserProfile>(cached);
        }

        // Yoksa replikadan oku
        return await _userRepository.GetAsync(userId);
    }
}

// Monotonic Reads: Token tabanlı versiyon takibi
public class MonotonicReadService
{
    private readonly IUserRepository _repository;

    public MonotonicReadService(IUserRepository repository)
    {
        _repository = repository;
    }

    public async Task<(UserProfile? Profile, long Version)> GetProfileWithVersionAsync(
        string userId,
        long minimumVersion = 0)
    {
        var profile = await _repository.GetAsync(userId);
        if (profile == null) return (null, 0);

        // Eğer dönen versiyon istenilen minimum versiyondan küçükse
        // primary node'dan oku (monotonic read garantisi)
        if (profile.Version < minimumVersion)
        {
            profile = await _repository.GetFromPrimaryAsync(userId);
        }

        return (profile, profile?.Version ?? 0);
    }
}
```

---

### 6. Soru

**Consistency Level kavramı nedir? Cassandra'da nasıl yapılandırılır?**

**Cevap:**

Consistency Level, bir okuma veya yazma işleminin başarılı sayılabilmesi için kaç node'dan yanıt alınması gerektiğini belirler. Cassandra'da bu level ayarlanabilir (tunable consistency):

- **ONE**: 1 node yanıt versin (en hızlı, en az tutarlı)
- **QUORUM**: (N/2 + 1) node yanıt versin (denge)
- **ALL**: Tüm node'lar yanıt versin (en yavaş, en tutarlı)
- **LOCAL_QUORUM**: Yerel datacenter'da quorum (multi-datacenter için)

Okuma + Yazma quorum kombinasyonu **strong consistency** sağlar: `W + R > N`

**Örnek Kod:**

```csharp
public class CassandraConsistencyLevelService
{
    private readonly ISession _session;

    public CassandraConsistencyLevelService(ISession session)
    {
        _session = session;
    }

    // Strong Consistency: QUORUM okuma + QUORUM yazma
    // W + R > N koşulunu sağlar (3 node için: 2 + 2 > 3)
    public async Task WriteStrongAsync(string key, string value)
    {
        var ps = await _session.PrepareAsync(
            "INSERT INTO kv_store (key, value, updated_at) VALUES (?, ?, ?)");

        var bound = ps.Bind(key, value, DateTimeOffset.UtcNow)
            .SetConsistencyLevel(ConsistencyLevel.Quorum);

        await _session.ExecuteAsync(bound);
    }

    public async Task<string?> ReadStrongAsync(string key)
    {
        var ps = await _session.PrepareAsync(
            "SELECT value FROM kv_store WHERE key = ?");

        var bound = ps.Bind(key)
            .SetConsistencyLevel(ConsistencyLevel.Quorum);

        var result = await _session.ExecuteAsync(bound);
        return result.FirstOrDefault()?.GetValue<string>("value");
    }

    // High Availability: ONE okuma + ANY yazma (AP tarafı)
    public async Task WriteHighAvailabilityAsync(string key, string value)
    {
        var ps = await _session.PrepareAsync(
            "INSERT INTO kv_store (key, value, updated_at) VALUES (?, ?, ?)");

        var bound = ps.Bind(key, value, DateTimeOffset.UtcNow)
            .SetConsistencyLevel(ConsistencyLevel.Any); // Herhangi bir node yazabilir

        await _session.ExecuteAsync(bound);
    }

    public async Task<string?> ReadHighAvailabilityAsync(string key)
    {
        var ps = await _session.PrepareAsync(
            "SELECT value FROM kv_store WHERE key = ?");

        var bound = ps.Bind(key)
            .SetConsistencyLevel(ConsistencyLevel.One); // İlk yanıt veren node'dan al

        var result = await _session.ExecuteAsync(bound);
        return result.FirstOrDefault()?.GetValue<string>("value");
    }
}
```

---

### 7. Soru

**Split-Brain problemi nedir ve nasıl önlenir?**

**Cevap:**

Split-Brain, bir ağ bölünmesi sırasında birden fazla node'un kendini "master" veya "primary" sanması durumudur. Her iki taraf da yazma kabul ettiğinde çelişen veriler oluşur. Önleme yöntemleri:

- **Quorum tabanlı liderlik**: Çoğunluğu olmayan taraf yazmaları reddeder.
- **Fencing token**: Eski lider token'ı sona erdiğinde sistem tarafından işlemleri bloke edilir.
- **STONITH (Shoot The Other Node In The Head)**: Eski lideri sisteme zorla kapatmak.
- **Lease tabanlı liderlik**: Lider periyodik olarak yenilenen bir kira süresi ile çalışır.

**Örnek Kod:**

```csharp
// Fencing Token ile Split-Brain koruması
public class LeaderElectionService
{
    private readonly IDistributedLockProvider _lockProvider;
    private readonly ILogger<LeaderElectionService> _logger;
    private long _fencingToken;

    public LeaderElectionService(
        IDistributedLockProvider lockProvider,
        ILogger<LeaderElectionService> logger)
    {
        _lockProvider = lockProvider;
        _logger = logger;
    }

    public async Task<bool> TryBecomeLeaderAsync(string resource, TimeSpan leaseDuration)
    {
        var result = await _lockProvider.TryAcquireAsync(resource, leaseDuration);
        if (result.Acquired)
        {
            // Monotonik artan token ile eski liderin işlemlerini reddet
            _fencingToken = result.FencingToken;
            _logger.LogInformation(
                "Lider seçildim. Fencing token: {Token}", _fencingToken);
            return true;
        }

        return false;
    }

    // Yazma işleminde fencing token kontrolü
    public async Task WriteAsLeaderAsync(string key, string value, long expectedToken)
    {
        if (_fencingToken != expectedToken)
        {
            throw new StaleLeaderException(
                $"Eski lider token'ı ({expectedToken}). Geçerli token: {_fencingToken}");
        }

        // Yazma işlemi burada gerçekleşir
        _logger.LogInformation("Lider olarak yazıyorum: {Key} = {Value}", key, value);
    }
}

public class StaleLeaderException : Exception
{
    public StaleLeaderException(string message) : base(message) { }
}
```

---

## Özet Karşılaştırma Tablosu

| Özellik | SQL Server | Cassandra | MongoDB |
|---|---|---|---|
| CAP Tipi | CP (distributed) | AP | CP |
| Tutarlılık | Güçlü (ACID) | Ayarlanabilir | Ayarlanabilir |
| Kullanılabilirlik | Orta (Primary gerekli) | Yüksek (leaderless) | Orta (Primary gerekli) |
| Bölünme Toleransı | Evet | Evet | Evet |
| Kullanım Alanı | Finansal işlemler | IoT, sosyal medya | Genel amaçlı |

## Kaynaklar

- [CAP Theorem - Wikipedia](https://en.wikipedia.org/wiki/CAP_theorem)
- [PACELC Theorem](https://en.wikipedia.org/wiki/PACELC_theorem)
- [Eventual Consistency - Werner Vogels](https://www.allthingsdistributed.com/2008/12/eventually_consistent.html)
- [Cassandra Consistency Levels](https://cassandra.apache.org/doc/latest/cassandra/architecture/dynamo.html)
- [Designing Data-Intensive Applications - Martin Kleppmann](https://dataintensive.net/)
