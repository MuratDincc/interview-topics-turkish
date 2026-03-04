# Konsensüs Algoritmaları

## Genel Bakış

Konsensüs algoritmaları, dağıtık bir sistemdeki birden fazla node'un (düğüm) belirli bir değer veya eylem üzerinde anlaşmasını sağlayan protokollerdir. Ağ hataları, node çökmeleri ve gecikmelere rağmen sistemin doğru çalışmasını garanti ederler. Dağıtık sistemlerin temel yapı taşları olan konsensüs algoritmaları; lider seçimi, dağıtık kilitleme, log replikasyonu ve yapılandırma yönetimi gibi kritik problemleri çözer.

**Temel konsensüs algoritmaları:**

- **Paxos**: Lamport tarafından tasarlanan klasik konsensüs algoritması. Güçlü teorik temellere sahiptir ancak anlaşılması ve uygulanması zordur.
- **Raft**: Paxos'tan daha anlaşılır olacak şekilde tasarlanmış modern konsensüs algoritması. etcd ve CockroachDB'de kullanılır.
- **ZAB (ZooKeeper Atomic Broadcast)**: Apache ZooKeeper'ın kullandığı konsensüs protokolü.
- **PBFT (Practical Byzantine Fault Tolerance)**: Kötü niyetli node'lara karşı dayanıklı (Byzantine fault tolerant) konsensüs.

---

## Raft Algoritması

Raft, anlaşılabilirlik odaklı tasarlanmış modern bir konsensüs algoritmasıdır. Üç temel alt problemi çözer:

1. **Lider Seçimi (Leader Election)**: Tek bir lider seçilir, tüm istemci istekleri lidere yönlendirilir.
2. **Log Replikasyonu (Log Replication)**: Lider, log girişlerini takipçilere replike eder.
3. **Güvenlik (Safety)**: Taahhüt edilmiş log girişleri asla kaybolmaz.

### Raft Terimleri

- **Term**: Monoton artan sayı. Her lider seçimi yeni bir term başlatır.
- **Leader**: Log yazma işlemlerini yönetir.
- **Follower**: Liderin komutlarını uygular.
- **Candidate**: Lider seçimine katılan node.
- **Heartbeat**: Liderin takipçilere periyodik gönderdiği canlı sinyali.
- **Election Timeout**: Bu sürede heartbeat gelmezse follower candidate'e dönüşür.

---

## Mülakat Soruları

### 1. Soru

**Raft algoritması nasıl çalışır? Lider seçimi sürecini açıklayın.**

**Cevap:**

Raft'ta her node başlangıçta Follower durumundadır. Election timeout dolduğunda node Candidate'e dönüşür, term numarasını artırır ve diğer node'lardan oy ister. Çoğunluk oy alan Candidate Leader olur. Leader, heartbeat mesajları göndererek otoritesini korur. Leader çökerse en güncel log'a sahip Candidate yeni lideri seçilir. Bu sayede log tutarlılığı garanti altına alınır.

**Örnek Kod:**

```csharp
public enum RaftNodeState { Follower, Candidate, Leader }

public class RaftNode
{
    public string NodeId { get; }
    public RaftNodeState State { get; private set; } = RaftNodeState.Follower;
    public int CurrentTerm { get; private set; } = 0;
    public string? VotedFor { get; private set; }
    public string? CurrentLeaderId { get; private set; }

    private readonly List<string> _peers;
    private readonly ILogger<RaftNode> _logger;
    private readonly Random _random = new();
    private Timer? _electionTimer;
    private Timer? _heartbeatTimer;

    // Log: her giriş (term, komut) ikilisinden oluşur
    private readonly List<LogEntry> _log = new();
    private int _commitIndex = -1;
    private int _lastApplied = -1;

    // Leader state
    private readonly Dictionary<string, int> _nextIndex = new();
    private readonly Dictionary<string, int> _matchIndex = new();

    public RaftNode(string nodeId, List<string> peers, ILogger<RaftNode> logger)
    {
        NodeId = nodeId;
        _peers = peers;
        _logger = logger;
        ResetElectionTimer();
    }

    private void ResetElectionTimer()
    {
        _electionTimer?.Dispose();
        // Randomized timeout: split-vote'ları önler (150-300ms)
        var timeout = _random.Next(150, 300);
        _electionTimer = new Timer(_ => StartElection(), null, timeout, Timeout.Infinite);
    }

    private async void StartElection()
    {
        State = RaftNodeState.Candidate;
        CurrentTerm++;
        VotedFor = NodeId; // Kendine oy ver
        int votesReceived = 1;

        _logger.LogInformation(
            "[{NodeId}] Seçim başlatıldı. Term: {Term}", NodeId, CurrentTerm);

        var lastLogIndex = _log.Count - 1;
        var lastLogTerm = lastLogIndex >= 0 ? _log[lastLogIndex].Term : 0;

        var voteRequest = new VoteRequest
        {
            Term = CurrentTerm,
            CandidateId = NodeId,
            LastLogIndex = lastLogIndex,
            LastLogTerm = lastLogTerm
        };

        var voteTasks = _peers.Select(peer => RequestVoteAsync(peer, voteRequest));
        var results = await Task.WhenAll(voteTasks);

        foreach (var result in results.Where(r => r != null))
        {
            if (result!.Term > CurrentTerm)
            {
                // Daha yüksek term → Follower'a dön
                BecomeFollower(result.Term);
                return;
            }

            if (result.VoteGranted) votesReceived++;
        }

        // Çoğunluk oyu aldıysak lider ol
        if (votesReceived > (_peers.Count + 1) / 2)
        {
            BecomeLeader();
        }
        else
        {
            // Seçim başarısız, tekrar follower
            BecomeFollower(CurrentTerm);
        }
    }

    private void BecomeLeader()
    {
        State = RaftNodeState.Leader;
        CurrentLeaderId = NodeId;

        // Leader state'i başlat
        foreach (var peer in _peers)
        {
            _nextIndex[peer] = _log.Count;
            _matchIndex[peer] = -1;
        }

        _logger.LogInformation(
            "[{NodeId}] Lider seçildim! Term: {Term}", NodeId, CurrentTerm);

        // Heartbeat göndermeye başla
        _heartbeatTimer = new Timer(_ => SendHeartbeats(), null, 0, 50);
    }

    private void BecomeFollower(int term)
    {
        State = RaftNodeState.Follower;
        CurrentTerm = term;
        VotedFor = null;
        _heartbeatTimer?.Dispose();
        ResetElectionTimer();
    }

    private async void SendHeartbeats()
    {
        if (State != RaftNodeState.Leader) return;

        var tasks = _peers.Select(peer => SendAppendEntriesAsync(peer, new List<LogEntry>()));
        await Task.WhenAll(tasks);
    }

    // Oy isteği değerlendirme (RequestVote RPC alındığında)
    public VoteResponse HandleVoteRequest(VoteRequest request)
    {
        if (request.Term < CurrentTerm)
            return new VoteResponse { Term = CurrentTerm, VoteGranted = false };

        if (request.Term > CurrentTerm)
            BecomeFollower(request.Term);

        bool logIsUpToDate = IsLogUpToDate(request.LastLogIndex, request.LastLogTerm);
        bool canVote = VotedFor == null || VotedFor == request.CandidateId;

        if (canVote && logIsUpToDate)
        {
            VotedFor = request.CandidateId;
            ResetElectionTimer();
            return new VoteResponse { Term = CurrentTerm, VoteGranted = true };
        }

        return new VoteResponse { Term = CurrentTerm, VoteGranted = false };
    }

    private bool IsLogUpToDate(int lastLogIndex, int lastLogTerm)
    {
        var myLastIndex = _log.Count - 1;
        var myLastTerm = myLastIndex >= 0 ? _log[myLastIndex].Term : 0;

        if (lastLogTerm != myLastTerm)
            return lastLogTerm > myLastTerm;

        return lastLogIndex >= myLastIndex;
    }

    private Task<VoteResponse?> RequestVoteAsync(string peer, VoteRequest request)
        => Task.FromResult<VoteResponse?>(null); // Gerçek implementasyonda RPC çağrısı

    private Task SendAppendEntriesAsync(string peer, List<LogEntry> entries)
        => Task.CompletedTask; // Gerçek implementasyonda RPC çağrısı
}

public record LogEntry(int Term, string Command);
public record VoteRequest(int Term, string CandidateId, int LastLogIndex, int LastLogTerm);
public record VoteResponse(int Term, bool VoteGranted);
```

---

### 2. Soru

**Paxos algoritması nedir? Raft'tan farkları nelerdir?**

**Cevap:**

Paxos, Lamport tarafından tasarlanmış ve dağıtık konsensüs alanında teorik temel oluşturan algoritmadır. Proposer, Acceptor ve Learner rolleri vardır. İki aşamalıdır: Prepare ve Accept.

Raft ile temel farkları:
- **Anlaşılabilirlik**: Raft, Paxos'tan çok daha anlaşılır ve öğrenmesi kolaydır.
- **Lider merkezlilik**: Raft güçlü lider kullanırken, Paxos'ta birden fazla proposer olabilir.
- **Log yönetimi**: Raft log sırasını garanti ederken, Paxos log açıklarına izin verebilir.
- **Üye değişikliği**: Raft'ta joint consensus mekanizması vardır.

**Örnek Kod:**

```csharp
// Paxos: Basit Single-Decree Paxos implementasyonu
public class PaxosProposer
{
    private readonly int _proposerId;
    private readonly List<PaxosAcceptor> _acceptors;
    private int _proposalNumber;

    public PaxosProposer(int proposerId, List<PaxosAcceptor> acceptors)
    {
        _proposerId = proposerId;
        _acceptors = acceptors;
    }

    public async Task<string?> ProposeAsync(string value)
    {
        _proposalNumber = GenerateProposalNumber();
        int quorum = (_acceptors.Count / 2) + 1;

        // Aşama 1: Prepare
        var prepareResponses = new List<PrepareResponse>();
        foreach (var acceptor in _acceptors)
        {
            var response = await acceptor.PrepareAsync(_proposalNumber);
            if (response.Promised) prepareResponses.Add(response);
        }

        if (prepareResponses.Count < quorum)
            return null; // Quorum sağlanamadı

        // En yüksek önceki proposal'dan değer al (varsa)
        string? proposalValue = value;
        var highestPrevious = prepareResponses
            .Where(r => r.AcceptedValue != null)
            .OrderByDescending(r => r.AcceptedProposalNumber)
            .FirstOrDefault();

        if (highestPrevious != null)
            proposalValue = highestPrevious.AcceptedValue; // Önceki değeri koru

        // Aşama 2: Accept
        int acceptCount = 0;
        foreach (var acceptor in _acceptors)
        {
            var accepted = await acceptor.AcceptAsync(_proposalNumber, proposalValue!);
            if (accepted) acceptCount++;
        }

        return acceptCount >= quorum ? proposalValue : null;
    }

    private int GenerateProposalNumber()
        => (_proposerId * 1000) + (int)(DateTimeOffset.UtcNow.ToUnixTimeMilliseconds() % 1000);
}

public class PaxosAcceptor
{
    private int _promisedNumber = -1;
    private int _acceptedNumber = -1;
    private string? _acceptedValue;
    private readonly SemaphoreSlim _lock = new(1, 1);

    public async Task<PrepareResponse> PrepareAsync(int proposalNumber)
    {
        await _lock.WaitAsync();
        try
        {
            if (proposalNumber > _promisedNumber)
            {
                _promisedNumber = proposalNumber;
                return new PrepareResponse(
                    Promised: true,
                    AcceptedProposalNumber: _acceptedNumber,
                    AcceptedValue: _acceptedValue);
            }

            return new PrepareResponse(Promised: false, -1, null);
        }
        finally { _lock.Release(); }
    }

    public async Task<bool> AcceptAsync(int proposalNumber, string value)
    {
        await _lock.WaitAsync();
        try
        {
            if (proposalNumber >= _promisedNumber)
            {
                _promisedNumber = proposalNumber;
                _acceptedNumber = proposalNumber;
                _acceptedValue = value;
                return true;
            }
            return false;
        }
        finally { _lock.Release(); }
    }
}

public record PrepareResponse(bool Promised, int AcceptedProposalNumber, string? AcceptedValue);
```

---

### 3. Soru

**Dağıtık kilitleme (Distributed Locking) nedir? C# ile nasıl implement edilir?**

**Cevap:**

Dağıtık kilitleme, birden fazla node'un aynı anda kritik bir kaynağa erişmesini engellemek için kullanılır. Redis tabanlı Redlock algoritması, ZooKeeper tabanlı kilitleme veya veritabanı tabanlı kilitler yaygın yöntemlerdir.

Redlock algoritması (Redis):
1. N Redis instance'ına (tek sayı, örn. 5) bağlan.
2. Millisaniye hassasiyetiyle geçerli zamanı al.
3. Her instance'a SET NX PX komutu ile kilit almaya çalış.
4. (N/2 + 1) instance'dan kilit alındıysa ve geçen süre kilit süresinden kısaysa başarılı.
5. Başarısız olunursa tüm instance'lardaki kilitleri serbest bırak.

**Örnek Kod:**

```csharp
// Redis ile Distributed Lock (Redlock benzeri)
public class RedisDistributedLock : IAsyncDisposable
{
    private readonly IDatabase _db;
    private readonly string _key;
    private readonly string _lockValue;
    private bool _isAcquired;

    private RedisDistributedLock(IDatabase db, string key, string lockValue, bool isAcquired)
    {
        _db = db;
        _key = key;
        _lockValue = lockValue;
        _isAcquired = isAcquired;
    }

    public bool IsAcquired => _isAcquired;

    public static async Task<RedisDistributedLock> AcquireAsync(
        IDatabase db,
        string key,
        TimeSpan expiry,
        int retryCount = 3,
        TimeSpan? retryDelay = null)
    {
        var lockValue = Guid.NewGuid().ToString("N");
        var delay = retryDelay ?? TimeSpan.FromMilliseconds(100);

        for (int i = 0; i < retryCount; i++)
        {
            // SET key value NX PX milliseconds
            var acquired = await db.StringSetAsync(
                key,
                lockValue,
                expiry,
                When.NotExists);

            if (acquired)
                return new RedisDistributedLock(db, key, lockValue, true);

            if (i < retryCount - 1)
                await Task.Delay(delay);
        }

        return new RedisDistributedLock(db, key, lockValue, false);
    }

    public async Task<bool> ExtendAsync(TimeSpan extension)
    {
        // Yalnızca bizim kilidimizsek uzat (Lua script ile atomik)
        var script = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('pexpire', KEYS[1], ARGV[2])
            else
                return 0
            end
            """;

        var result = await _db.ScriptEvaluateAsync(
            script,
            new RedisKey[] { _key },
            new RedisValue[] { _lockValue, (long)extension.TotalMilliseconds });

        return (long)result == 1;
    }

    public async ValueTask DisposeAsync()
    {
        if (!_isAcquired) return;

        // Yalnızca bizim kilidimizsek serbest bırak (Lua script ile atomik)
        var script = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
            """;

        await _db.ScriptEvaluateAsync(
            script,
            new RedisKey[] { _key },
            new RedisValue[] { _lockValue });

        _isAcquired = false;
    }
}

// Kullanım örneği
public class InventoryService
{
    private readonly IDatabase _redisDb;
    private readonly IProductRepository _productRepository;

    public InventoryService(IDatabase redisDb, IProductRepository productRepository)
    {
        _redisDb = redisDb;
        _productRepository = productRepository;
    }

    public async Task<bool> ReserveStockAsync(string productId, int quantity)
    {
        var lockKey = $"inventory:lock:{productId}";

        await using var distributedLock = await RedisDistributedLock.AcquireAsync(
            _redisDb,
            lockKey,
            expiry: TimeSpan.FromSeconds(30),
            retryCount: 5,
            retryDelay: TimeSpan.FromMilliseconds(200));

        if (!distributedLock.IsAcquired)
            throw new LockAcquisitionException($"Ürün {productId} için kilit alınamadı.");

        // Kritik bölge: yalnızca bir node aynı anda burada
        var product = await _productRepository.GetAsync(productId);
        if (product == null || product.StockQuantity < quantity)
            return false;

        product.StockQuantity -= quantity;
        await _productRepository.UpdateAsync(product);
        return true;
    } // DisposeAsync otomatik kilidi serbest bırakır
}

public class LockAcquisitionException : Exception
{
    public LockAcquisitionException(string message) : base(message) { }
}
```

---

### 4. Soru

**Leader Election nedir? etcd ve ZooKeeper nasıl kullanılır?**

**Cevap:**

Leader Election (Lider Seçimi), dağıtık bir sistemde koordinatör rolünü üstlenecek tek bir node belirleme sürecidir. Aynı anda yalnızca bir node lider olabilir; lider çöktüğünde yeni bir lider seçilir. etcd, Raft tabanlı olup güçlü tutarlılık sunar. ZooKeeper ise ZAB protokolünü kullanır. Her ikisi de dağıtık sistemlerde lider seçimi ve servis koordinasyonu için yaygın kullanılır.

**Örnek Kod:**

```csharp
// etcd ile Leader Election (dotnet-etcd kütüphanesi)
public class EtcdLeaderElection : IAsyncDisposable
{
    private readonly EtcdClient _client;
    private readonly string _electionName;
    private readonly string _nodeId;
    private readonly ILogger<EtcdLeaderElection> _logger;
    private string? _leaderKey;
    private long _leaseId;
    private CancellationTokenSource? _cts;

    public bool IsLeader { get; private set; }
    public event EventHandler? BecameLeader;
    public event EventHandler? LostLeadership;

    public EtcdLeaderElection(
        EtcdClient client,
        string electionName,
        string nodeId,
        ILogger<EtcdLeaderElection> logger)
    {
        _client = client;
        _electionName = electionName;
        _nodeId = nodeId;
        _logger = logger;
    }

    public async Task StartAsync(CancellationToken cancellationToken = default)
    {
        _cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);

        // TTL'li kira (lease) oluştur
        var leaseResponse = await _client.LeaseGrantAsync(
            new LeaseGrantRequest { TTL = 15 });
        _leaseId = leaseResponse.ID;

        // Lease yenileme görevi
        _ = Task.Run(() => KeepLeaseAliveAsync(_cts.Token), _cts.Token);

        // Seçime katıl
        await CampaignAsync(_cts.Token);
    }

    private async Task CampaignAsync(CancellationToken cancellationToken)
    {
        var electionKey = $"/elections/{_electionName}/{_leaseId:x}";

        // Lease'e bağlı anahtar yaz (lease sona erince anahtar silinir)
        var txn = new TxnRequest();
        txn.Compare.Add(new Compare
        {
            Key = ByteString.CopyFromUtf8(electionKey),
            Target = Compare.Types.CompareTarget.Version,
            Result = Compare.Types.CompareResult.Equal,
            Version = 0 // Anahtar yoksa
        });
        txn.Success.Add(new RequestOp
        {
            RequestPut = new PutRequest
            {
                Key = ByteString.CopyFromUtf8(electionKey),
                Value = ByteString.CopyFromUtf8(_nodeId),
                Lease = _leaseId
            }
        });

        var txnResponse = await _client.TransactionAsync(txn);
        if (!txnResponse.Succeeded) return;

        _leaderKey = electionKey;

        // En küçük revision'ı olan anahtar liderdir
        await WatchForLeadershipAsync(cancellationToken);
    }

    private async Task WatchForLeadershipAsync(CancellationToken cancellationToken)
    {
        // Prefix altındaki tüm anahtarları listele ve en küçük revision'lıyı bul
        var range = await _client.GetRangeAsync($"/elections/{_electionName}/");
        var sorted = range.Kvs.OrderBy(kv => kv.CreateRevision).ToList();

        if (sorted.Count > 0 && sorted[0].Key.ToStringUtf8() == _leaderKey)
        {
            IsLeader = true;
            _logger.LogInformation("[{NodeId}] Lider seçildim.", _nodeId);
            BecameLeader?.Invoke(this, EventArgs.Empty);
        }
        else if (sorted.Count > 1)
        {
            // Bir önceki anahtarı izle; silinince tekrar dene
            var myIndex = sorted.FindIndex(kv => kv.Key.ToStringUtf8() == _leaderKey);
            if (myIndex > 0)
            {
                var predecessor = sorted[myIndex - 1].Key.ToStringUtf8();
                _logger.LogInformation(
                    "[{NodeId}] Lider değilim, {Predecessor} izliyorum.", _nodeId, predecessor);

                // predecessor silinince yeniden dene
                await WatchKeyDeletionAsync(predecessor, cancellationToken);
                await WatchForLeadershipAsync(cancellationToken);
            }
        }
    }

    private Task WatchKeyDeletionAsync(string key, CancellationToken cancellationToken)
    {
        var tcs = new TaskCompletionSource();
        // etcd watch API ile silme eventi bekle
        // Gerçek implementasyon için dotnet-etcd watch stream kullanılır
        return tcs.Task.WaitAsync(cancellationToken);
    }

    private async Task KeepLeaseAliveAsync(CancellationToken cancellationToken)
    {
        using var stream = _client.LeaseKeepAlive(cancellationToken);

        while (!cancellationToken.IsCancellationRequested)
        {
            await stream.RequestStream.WriteAsync(
                new LeaseKeepAliveRequest { ID = _leaseId });
            await Task.Delay(TimeSpan.FromSeconds(5), cancellationToken);
        }
    }

    public async ValueTask DisposeAsync()
    {
        _cts?.Cancel();
        IsLeader = false;

        try
        {
            await _client.LeaseRevokeAsync(new LeaseRevokeRequest { ID = _leaseId });
        }
        catch { /* intentional */ }
    }
}
```

---

### 5. Soru

**Two-Phase Commit (2PC) nedir ve konsensüs algoritmalarıyla ilişkisi nedir?**

**Cevap:**

2PC, birden fazla node'a yayılan bir işlemin atomik olarak tamamlanmasını sağlayan protokoldür. Prepare ve Commit olmak üzere iki aşamadan oluşur. Koordinatör (coordinator) tüm katılımcılara Prepare gönderir; tümü hazır derse Commit, biri hayır derse Abort gönderir. 2PC, konsensüsün özel bir biçimidir ancak koordinatör çökerse sistem bloke kalabilir (blocking protocol). Bu sorunu çözmek için 3PC veya Raft tabanlı yaklaşımlar kullanılır.

**Örnek Kod:**

```csharp
public enum TwoPhaseCommitStatus { Prepared, Committed, Aborted }

public class TwoPhaseCommitCoordinator
{
    private readonly List<ITwoPhaseParticipant> _participants;
    private readonly ILogger<TwoPhaseCommitCoordinator> _logger;

    public TwoPhaseCommitCoordinator(
        List<ITwoPhaseParticipant> participants,
        ILogger<TwoPhaseCommitCoordinator> logger)
    {
        _participants = participants;
        _logger = logger;
    }

    public async Task<bool> ExecuteAsync(string transactionId, object payload)
    {
        _logger.LogInformation("[2PC] {TransactionId} başlatıldı.", transactionId);

        // Aşama 1: Prepare
        var prepareResults = await Task.WhenAll(
            _participants.Select(p => p.PrepareAsync(transactionId, payload)));

        bool allReady = prepareResults.All(r => r);

        // Aşama 2: Commit veya Abort
        if (allReady)
        {
            _logger.LogInformation("[2PC] {TransactionId} commit ediliyor.", transactionId);
            await Task.WhenAll(_participants.Select(p => p.CommitAsync(transactionId)));
            return true;
        }
        else
        {
            _logger.LogWarning("[2PC] {TransactionId} abort ediliyor.", transactionId);
            // Tüm katılımcılara abort gönder
            await Task.WhenAll(_participants.Select(p =>
                p.AbortAsync(transactionId)));
            return false;
        }
    }
}

public interface ITwoPhaseParticipant
{
    Task<bool> PrepareAsync(string transactionId, object payload);
    Task CommitAsync(string transactionId);
    Task AbortAsync(string transactionId);
}

public class DatabaseParticipant : ITwoPhaseParticipant
{
    private readonly string _connectionString;
    private readonly Dictionary<string, SqlTransaction> _pendingTransactions = new();
    private readonly ILogger<DatabaseParticipant> _logger;

    public DatabaseParticipant(string connectionString, ILogger<DatabaseParticipant> logger)
    {
        _connectionString = connectionString;
        _logger = logger;
    }

    public async Task<bool> PrepareAsync(string transactionId, object payload)
    {
        try
        {
            var connection = new SqlConnection(_connectionString);
            await connection.OpenAsync();

            var transaction = connection.BeginTransaction();
            _pendingTransactions[transactionId] = transaction;

            // İşlemi yap ama commit etme
            await connection.ExecuteAsync(
                "INSERT INTO distributed_log (tx_id, status) VALUES (@TxId, 'PREPARED')",
                new { TxId = transactionId },
                transaction);

            _logger.LogInformation(
                "[Participant] {TransactionId} PREPARED.", transactionId);
            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex,
                "[Participant] {TransactionId} PREPARE başarısız.", transactionId);
            return false;
        }
    }

    public async Task CommitAsync(string transactionId)
    {
        if (_pendingTransactions.TryGetValue(transactionId, out var transaction))
        {
            await transaction.CommitAsync();
            _pendingTransactions.Remove(transactionId);
            _logger.LogInformation("[Participant] {TransactionId} COMMITTED.", transactionId);
        }
    }

    public async Task AbortAsync(string transactionId)
    {
        if (_pendingTransactions.TryGetValue(transactionId, out var transaction))
        {
            await transaction.RollbackAsync();
            _pendingTransactions.Remove(transactionId);
            _logger.LogInformation("[Participant] {TransactionId} ABORTED.", transactionId);
        }
    }
}
```

---

### 6. Soru

**Byzantine Fault Tolerance (BFT) nedir? PBFT nasıl çalışır?**

**Cevap:**

Byzantine Fault, bir node'un sadece çökmek yerine hatalı veya kötü niyetli davranışlar sergilemesidir (yanlış değer göndermek, farklı node'lara farklı mesaj göndermek gibi). PBFT (Practical Byzantine Fault Tolerance) algoritması, 3f+1 node ile f adet Byzantine hataya dayanabilir.

PBFT üç aşamadan oluşur:
1. **Pre-Prepare**: Lider isteği yayınlar.
2. **Prepare**: Node'lar isteği doğrular ve hazır olduklarını bildirir.
3. **Commit**: 2f+1 node'dan prepare mesajı alındıktan sonra commit yayınlanır.

Blockchain ağlarında kullanılan Tendermint ve HotStuff bu temelden türetilmiştir.

**Örnek Kod:**

```csharp
// Basit BFT simülasyonu: Threshold tabanlı karar alma
public class ByzantineFaultTolerantService
{
    private readonly int _totalNodes;
    private readonly int _byzantineNodes;
    private readonly ILogger<ByzantineFaultTolerantService> _logger;

    public ByzantineFaultTolerantService(
        int totalNodes,
        ILogger<ByzantineFaultTolerantService> logger)
    {
        _totalNodes = totalNodes;
        // BFT için: n >= 3f + 1 → f = (n - 1) / 3
        _byzantineNodes = (totalNodes - 1) / 3;
        _logger = logger;
    }

    // Çoğunluk oylaması ile Byzantine node'ları etkisizleştir
    public T? ReachConsensus<T>(IEnumerable<(string NodeId, T Value)> votes)
        where T : IEquatable<T>
    {
        int quorum = 2 * _byzantineNodes + 1; // 2f + 1

        var grouped = votes
            .GroupBy(v => v.Value)
            .Select(g => new { Value = g.Key, Count = g.Count() })
            .OrderByDescending(g => g.Count)
            .FirstOrDefault();

        if (grouped == null || grouped.Count < quorum)
        {
            _logger.LogWarning("Konsensüs sağlanamadı. Quorum: {Quorum}", quorum);
            return default;
        }

        _logger.LogInformation(
            "Konsensüs sağlandı. Değer: {Value}, Oy: {Count}/{Total}",
            grouped.Value, grouped.Count, _totalNodes);

        return grouped.Value;
    }
}
```

---

## Karşılaştırma Tablosu

| Özellik | Paxos | Raft | ZAB | PBFT |
|---|---|---|---|---|
| Anlaşılabilirlik | Düşük | Yüksek | Orta | Orta |
| Hata Toleransı | f < n/2 | f < n/2 | f < n/2 | f < n/3 |
| Byzantine Tolerans | Hayır | Hayır | Hayır | Evet |
| Kullanım | Chubby | etcd | ZooKeeper | Blockchain |
| Lider | Multi-Paxos | Tek lider | Tek lider | Tek lider |

## Kaynaklar

- [Raft Consensus Algorithm](https://raft.github.io/)
- [In Search of an Understandable Consensus Algorithm (Raft Paper)](https://raft.github.io/raft.pdf)
- [Paxos Made Simple - Lamport](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)
- [ZooKeeper: Wait-free coordination for Internet-scale systems](https://www.usenix.org/legacy/event/atc10/tech/full_papers/Hunt.pdf)
- [Practical Byzantine Fault Tolerance - Castro & Liskov](https://pmg.csail.mit.edu/papers/osdi99.pdf)
