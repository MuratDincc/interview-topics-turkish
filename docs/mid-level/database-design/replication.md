# Database Replication

## Giriş

Database Replication, veritabanı verilerini birden fazla sunucuda kopyalama ve senkronize etme sürecidir. Mid-level geliştiriciler için replication stratejilerini anlamak, yüksek availability, disaster recovery ve read scalability sağlamada kritiktir.

## Replication Türleri

### 1. Master-Slave Replication (Primary-Secondary)
Tek bir master database ve birden fazla slave database'den oluşan yapı.

```csharp
public class MasterSlaveReplicationService
{
    private readonly IDbConnection _masterConnection;
    private readonly List<IDbConnection> _slaveConnections;
    private readonly ILogger<MasterSlaveReplicationService> _logger;
    private readonly Random _random;
    
    public MasterSlaveReplicationService(
        string masterConnectionString, 
        List<string> slaveConnectionStrings,
        ILogger<MasterSlaveReplicationService> logger)
    {
        _masterConnection = new SqlConnection(masterConnectionString);
        _slaveConnections = slaveConnectionStrings.Select(cs => new SqlConnection(cs)).ToList();
        _logger = logger;
        _random = new Random();
    }
    
    public async Task<int> WriteToMasterAsync(string sql, object parameters)
    {
        try
        {
            _logger.LogInformation("Writing to master database");
            return await _masterConnection.ExecuteAsync(sql, parameters);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error writing to master database");
            throw;
        }
    }
    
    public async Task<T> ReadFromSlaveAsync<T>(string sql, object parameters)
    {
        // Round-robin load balancing among slaves
        var slaveConnection = GetNextSlaveConnection();
        
        try
        {
            _logger.LogInformation("Reading from slave database");
            return await slaveConnection.QueryFirstOrDefaultAsync<T>(sql, parameters);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error reading from slave database, trying master");
            // Fallback to master if slave fails
            return await _masterConnection.QueryFirstOrDefaultAsync<T>(sql, parameters);
        }
    }
    
    public async Task<List<T>> ReadFromAllSlavesAsync<T>(string sql, object parameters)
    {
        var tasks = _slaveConnections.Select(conn => 
            conn.QueryAsync<T>(sql, parameters));
        
        var results = await Task.WhenAll(tasks);
        return results.SelectMany(r => r).ToList();
    }
    
    private IDbConnection GetNextSlaveConnection()
    {
        var index = _random.Next(_slaveConnections.Count);
        return _slaveConnections[index];
    }
    
    public async Task<bool> IsReplicationLagAcceptableAsync()
    {
        try
        {
            // Check replication lag between master and slaves
            var masterTime = await GetMasterTimeAsync();
            var slaveTimes = await GetSlaveTimesAsync();
            
            var maxLag = slaveTimes.Max(st => Math.Abs((masterTime - st).TotalSeconds));
            
            // Acceptable lag is less than 5 seconds
            return maxLag < 5;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error checking replication lag");
            return false;
        }
    }
    
    private async Task<DateTime> GetMasterTimeAsync()
    {
        var result = await _masterConnection.QueryFirstOrDefaultAsync<DateTime>("SELECT GETUTCDATE()");
        return result;
    }
    
    private async Task<List<DateTime>> GetSlaveTimesAsync()
    {
        var tasks = _slaveConnections.Select(conn => 
            conn.QueryFirstOrDefaultAsync<DateTime>("SELECT GETUTCDATE()"));
        
        var results = await Task.WhenAll(tasks);
        return results.ToList();
    }
}
```

### 2. Multi-Master Replication
Birden fazla master database'in aynı anda yazma yapabildiği yapı.

```csharp
public class MultiMasterReplicationService
{
    private readonly List<MasterNode> _masterNodes;
    private readonly ILogger<MultiMasterReplicationService> _logger;
    private readonly IConflictResolutionStrategy _conflictResolver;
    
    public MultiMasterReplicationService(
        List<string> masterConnectionStrings,
        IConflictResolutionStrategy conflictResolver,
        ILogger<MultiMasterReplicationService> logger)
    {
        _masterNodes = masterConnectionStrings.Select((cs, index) => 
            new MasterNode { Id = index, Connection = new SqlConnection(cs) }).ToList();
        _conflictResolver = conflictResolver;
        _logger = logger;
    }
    
    public async Task<bool> WriteToAllMastersAsync(string sql, object parameters)
    {
        var tasks = _masterNodes.Select(node => WriteToMasterAsync(node, sql, parameters));
        var results = await Task.WhenAll(tasks);
        
        // Check if all writes were successful
        return results.All(r => r);
    }
    
    public async Task<bool> WriteToNearestMasterAsync(string sql, object parameters)
    {
        var nearestNode = await GetNearestMasterNodeAsync();
        return await WriteToMasterAsync(nearestNode, sql, parameters);
    }
    
    public async Task<T> ReadFromNearestMasterAsync<T>(string sql, object parameters)
    {
        var nearestNode = await GetNearestMasterNodeAsync();
        
        try
        {
            return await nearestNode.Connection.QueryFirstOrDefaultAsync<T>(sql, parameters);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error reading from nearest master, trying others");
            
            // Try other nodes if nearest fails
            foreach (var node in _masterNodes.Where(n => n.Id != nearestNode.Id))
            {
                try
                {
                    return await node.Connection.QueryFirstOrDefaultAsync<T>(sql, parameters);
                }
                catch
                {
                    continue;
                }
            }
            
            throw;
        }
    }
    
    private async Task<MasterNode> GetNearestMasterNodeAsync()
    {
        // Simple round-robin for now, could be enhanced with latency-based selection
        var index = Environment.TickCount % _masterNodes.Count;
        return _masterNodes[index];
    }
    
    private async Task<bool> WriteToMasterAsync(MasterNode node, string sql, object parameters)
    {
        try
        {
            await node.Connection.ExecuteAsync(sql, parameters);
            _logger.LogInformation("Successfully wrote to master node {NodeId}", node.Id);
            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error writing to master node {NodeId}", node.Id);
            return false;
        }
    }
    
    public async Task ResolveConflictsAsync()
    {
        var conflicts = await DetectConflictsAsync();
        
        foreach (var conflict in conflicts)
        {
            var resolution = await _conflictResolver.ResolveAsync(conflict);
            await ApplyResolutionAsync(resolution);
        }
    }
    
    private async Task<List<DataConflict>> DetectConflictsAsync()
    {
        var conflicts = new List<DataConflict>();
        
        // Implementation to detect conflicts between master nodes
        // This would involve comparing data across nodes and identifying inconsistencies
        
        return conflicts;
    }
    
    private async Task ApplyResolutionAsync(ConflictResolution resolution)
    {
        // Apply the resolved conflict to all master nodes
        var tasks = _masterNodes.Select(node => 
            ApplyResolutionToNodeAsync(node, resolution));
        
        await Task.WhenAll(tasks);
    }
}

public class MasterNode
{
    public int Id { get; set; }
    public IDbConnection Connection { get; set; }
    public DateTime LastSync { get; set; }
    public bool IsHealthy { get; set; }
}

public class DataConflict
{
    public string TableName { get; set; }
    public string PrimaryKey { get; set; }
    public Dictionary<string, object> ConflictingValues { get; set; }
    public List<int> ConflictingNodeIds { get; set; }
}

public class ConflictResolution
{
    public string TableName { get; set; }
    public string PrimaryKey { get; set; }
    public Dictionary<string, object> ResolvedValues { get; set; }
    public string ResolutionStrategy { get; set; }
}
```

### 3. Chain Replication
Verilerin sıralı olarak bir node'dan diğerine aktarıldığı yapı.

```csharp
public class ChainReplicationService
{
    private readonly List<ReplicationNode> _nodes;
    private readonly ILogger<ChainReplicationService> _logger;
    
    public ChainReplicationService(
        List<string> connectionStrings,
        ILogger<ChainReplicationService> logger)
    {
        _nodes = connectionStrings.Select((cs, index) => new ReplicationNode
        {
            Id = index,
            Connection = new SqlConnection(cs),
            IsHead = index == 0,
            IsTail = index == connectionStrings.Count - 1
        }).ToList();
        
        _logger = logger;
    }
    
    public async Task<bool> WriteToHeadAsync(string sql, object parameters)
    {
        var headNode = _nodes.First(n => n.IsHead);
        
        try
        {
            // Write to head node
            await headNode.Connection.ExecuteAsync(sql, parameters);
            
            // Propagate to next node in chain
            await PropagateToNextNodeAsync(headNode, sql, parameters);
            
            _logger.LogInformation("Successfully wrote to head node and propagated");
            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error writing to head node");
            return false;
        }
    }
    
    private async Task PropagateToNextNodeAsync(ReplicationNode currentNode, string sql, object parameters)
    {
        var nextNode = _nodes.FirstOrDefault(n => n.Id == currentNode.Id + 1);
        
        if (nextNode != null)
        {
            try
            {
                await nextNode.Connection.ExecuteAsync(sql, parameters);
                _logger.LogInformation("Propagated to node {NodeId}", nextNode.Id);
                
                // Continue propagation
                await PropagateToNextNodeAsync(nextNode, sql, parameters);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error propagating to node {NodeId}", nextNode.Id);
                throw;
            }
        }
    }
    
    public async Task<T> ReadFromTailAsync<T>(string sql, object parameters)
    {
        var tailNode = _nodes.First(n => n.IsTail);
        
        try
        {
            return await tailNode.Connection.QueryFirstOrDefaultAsync<T>(sql, parameters);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error reading from tail node");
            throw;
        }
    }
    
    public async Task<bool> IsChainHealthyAsync()
    {
        foreach (var node in _nodes)
        {
            try
            {
                var result = await node.Connection.QueryFirstOrDefaultAsync<int>("SELECT 1");
                if (result != 1)
                {
                    _logger.LogWarning("Node {NodeId} is unhealthy", node.Id);
                    return false;
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Node {NodeId} health check failed", node.Id);
                return false;
            }
        }
        
        return true;
    }
}

public class ReplicationNode
{
    public int Id { get; set; }
    public IDbConnection Connection { get; set; }
    public bool IsHead { get; set; }
    public bool IsTail { get; set; }
    public DateTime LastSync { get; set; }
}
```

## Replication Implementation

### 1. Replication Monitor
```csharp
public class ReplicationMonitor
{
    private readonly IReplicationService _replicationService;
    private readonly ILogger<ReplicationMonitor> _logger;
    private readonly Timer _healthCheckTimer;
    
    public ReplicationMonitor(
        IReplicationService replicationService,
        ILogger<ReplicationMonitor> logger)
    {
        _replicationService = replicationService;
        _logger = logger;
        _healthCheckTimer = new Timer(PerformHealthCheck, null, TimeSpan.Zero, TimeSpan.FromMinutes(1));
    }
    
    private async void PerformHealthCheck(object state)
    {
        try
        {
            var isHealthy = await _replicationService.IsHealthyAsync();
            
            if (!isHealthy)
            {
                _logger.LogWarning("Replication health check failed");
                await NotifyAdministratorsAsync("Replication health check failed");
            }
            
            var lag = await _replicationService.GetReplicationLagAsync();
            if (lag > TimeSpan.FromMinutes(5))
            {
                _logger.LogWarning("Replication lag is high: {Lag}", lag);
                await NotifyAdministratorsAsync($"High replication lag: {lag}");
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error during health check");
        }
    }
    
    private async Task NotifyAdministratorsAsync(string message)
    {
        // Implementation to notify administrators via email, Slack, etc.
        _logger.LogInformation("Notifying administrators: {Message}", message);
    }
    
    public async Task<ReplicationStatus> GetReplicationStatusAsync()
    {
        return new ReplicationStatus
        {
            IsHealthy = await _replicationService.IsHealthyAsync(),
            ReplicationLag = await _replicationService.GetReplicationLagAsync(),
            LastSyncTime = await _replicationService.GetLastSyncTimeAsync(),
            NodeCount = await _replicationService.GetNodeCountAsync(),
            FailedNodes = await _replicationService.GetFailedNodesAsync()
        };
    }
}

public class ReplicationStatus
{
    public bool IsHealthy { get; set; }
    public TimeSpan ReplicationLag { get; set; }
    public DateTime LastSyncTime { get; set; }
    public int NodeCount { get; set; }
    public List<int> FailedNodes { get; set; }
}
```

### 2. Conflict Resolution Strategies
```csharp
public interface IConflictResolutionStrategy
{
    Task<ConflictResolution> ResolveAsync(DataConflict conflict);
}

public class LastWriteWinsStrategy : IConflictResolutionStrategy
{
    public async Task<ConflictResolution> ResolveAsync(DataConflict conflict)
    {
        // Simple strategy: use the most recent timestamp
        var mostRecentValue = conflict.ConflictingValues
            .OrderByDescending(kv => kv.Value)
            .First();
        
        return new ConflictResolution
        {
            TableName = conflict.TableName,
            PrimaryKey = conflict.PrimaryKey,
            ResolvedValues = new Dictionary<string, object> { mostRecentValue },
            ResolutionStrategy = "LastWriteWins"
        };
    }
}

public class CustomBusinessLogicStrategy : IConflictResolutionStrategy
{
    private readonly IBusinessRuleEngine _businessRuleEngine;
    
    public CustomBusinessLogicStrategy(IBusinessRuleEngine businessRuleEngine)
    {
        _businessRuleEngine = businessRuleEngine;
    }
    
    public async Task<ConflictResolution> ResolveAsync(DataConflict conflict)
    {
        // Apply business rules to resolve conflicts
        var resolvedValues = await _businessRuleEngine.ResolveConflictAsync(conflict);
        
        return new ConflictResolution
        {
            TableName = conflict.TableName,
            PrimaryKey = conflict.PrimaryKey,
            ResolvedValues = resolvedValues,
            ResolutionStrategy = "BusinessLogic"
        };
    }
}

public class ManualResolutionStrategy : IConflictResolutionStrategy
{
    private readonly IConflictNotificationService _notificationService;
    
    public ManualResolutionStrategy(IConflictNotificationService notificationService)
    {
        _notificationService = notificationService;
    }
    
    public async Task<ConflictResolution> ResolveAsync(DataConflict conflict)
    {
        // Notify administrators for manual resolution
        await _notificationService.NotifyConflictAsync(conflict);
        
        // Return a placeholder resolution that will be updated manually
        return new ConflictResolution
        {
            TableName = conflict.TableName,
            PrimaryKey = conflict.PrimaryKey,
            ResolvedValues = new Dictionary<string, object>(),
            ResolutionStrategy = "ManualResolution"
        };
    }
}
```

## Replication Best Practices

### 1. Monitoring ve Alerting
```csharp
public class ReplicationMetricsCollector
{
    private readonly IReplicationService _replicationService;
    private readonly IMetricsService _metricsService;
    
    public async Task CollectMetricsAsync()
    {
        var lag = await _replicationService.GetReplicationLagAsync();
        var nodeCount = await _replicationService.GetNodeCountAsync();
        var failedNodes = await _replicationService.GetFailedNodesAsync();
        
        // Record metrics
        _metricsService.RecordGauge("replication.lag.seconds", lag.TotalSeconds);
        _metricsService.RecordGauge("replication.nodes.total", nodeCount);
        _metricsService.RecordGauge("replication.nodes.failed", failedNodes.Count);
        
        // Set alerts
        if (lag > TimeSpan.FromMinutes(5))
        {
            _metricsService.RecordAlert("replication.lag.high", lag.TotalSeconds);
        }
        
        if (failedNodes.Count > 0)
        {
            _metricsService.RecordAlert("replication.nodes.failed", failedNodes.Count);
        }
    }
}
```

### 2. Disaster Recovery
```csharp
public class DisasterRecoveryService
{
    private readonly IReplicationService _replicationService;
    private readonly IBackupService _backupService;
    
    public async Task<RecoveryPlan> CreateRecoveryPlanAsync()
    {
        var healthyNodes = await _replicationService.GetHealthyNodesAsync();
        var failedNodes = await _replicationService.GetFailedNodesAsync();
        
        var plan = new RecoveryPlan
        {
            FailedNodes = failedNodes,
            RecoverySteps = new List<RecoveryStep>()
        };
        
        foreach (var failedNode in failedNodes)
        {
            var recoveryStep = await CreateRecoveryStepAsync(failedNode, healthyNodes);
            plan.RecoverySteps.Add(recoveryStep);
        }
        
        return plan;
    }
    
    private async Task<RecoveryStep> CreateRecoveryStepAsync(
        int failedNodeId, 
        List<int> healthyNodes)
    {
        var sourceNode = healthyNodes.First(); // Use first healthy node as source
        
        return new RecoveryStep
        {
            FailedNodeId = failedNodeId,
            SourceNodeId = sourceNode,
            Action = "RestoreFromReplica",
            EstimatedDuration = TimeSpan.FromMinutes(30),
            Dependencies = new List<string>()
        };
    }
    
    public async Task ExecuteRecoveryPlanAsync(RecoveryPlan plan)
    {
        foreach (var step in plan.RecoverySteps)
        {
            await ExecuteRecoveryStepAsync(step);
        }
    }
    
    private async Task ExecuteRecoveryStepAsync(RecoveryStep step)
    {
        // Implementation of recovery step execution
        // This would involve restoring data from healthy nodes
        // and bringing failed nodes back online
    }
}

public class RecoveryPlan
{
    public List<int> FailedNodes { get; set; }
    public List<RecoveryStep> RecoverySteps { get; set; }
}

public class RecoveryStep
{
    public int FailedNodeId { get; set; }
    public int SourceNodeId { get; set; }
    public string Action { get; set; }
    public TimeSpan EstimatedDuration { get; set; }
    public List<string> Dependencies { get; set; }
}
```

## Mülakat Soruları

### Temel Sorular

1. **Database replication nedir ve neden kullanılır?**
   - **Cevap**: Veritabanı verilerini birden fazla sunucuda kopyalama. High availability, disaster recovery, read scalability için.

2. **Master-Slave vs Multi-Master replication arasındaki fark nedir?**
   - **Cevap**: Master-Slave tek yazma noktası, Multi-Master çoklu yazma noktası. Complexity ve consistency trade-off'ları var.

3. **Replication lag nedir ve nasıl ölçülür?**
   - **Cevap**: Master ve slave arasındaki veri senkronizasyon gecikmesi. Timestamp karşılaştırması ile ölçülür.

4. **Conflict resolution nedir ve hangi stratejiler vardır?**
   - **Cevap**: Çakışan verileri çözme süreci. Last-write-wins, business logic, manual resolution stratejileri.

5. **Chain replication nedir ve ne zaman kullanılır?**
   - **Cevap**: Verilerin sıralı olarak aktarıldığı yapı. Strong consistency gerektiren durumlarda kullanılır.

### Teknik Sorular

1. **Replication'da data consistency nasıl sağlanır?**
   - **Cevap**: Synchronous/asynchronous replication, conflict resolution, eventual consistency, monitoring.

2. **Replication failure durumunda ne yapılır?**
   - **Cevap**: Health monitoring, automatic failover, manual intervention, disaster recovery procedures.

3. **Replication performance nasıl optimize edilir?**
   - **Cevap**: Batch operations, compression, network optimization, monitoring, tuning.

4. **Replication vs sharding arasındaki fark nedir?**
   - **Cevap**: Replication aynı veriyi kopyalar, sharding farklı veriyi böler. Farklı scalability patterns.

5. **Replication monitoring'de hangi metrics izlenir?**
   - **Cevap**: Replication lag, node health, sync status, performance metrics, error rates.

## Best Practices

1. **Replication Strategy Selection**
   - Business requirements analiz edin
   - Consistency vs performance trade-off'ları değerlendirin
   - Network infrastructure'ı göz önünde bulundurun
   - Monitoring capabilities planlayın

2. **Conflict Resolution**
   - Business rules tanımlayın
   - Automated resolution implement edin
   - Manual intervention için processes oluşturun
   - Conflict history maintain edin

3. **Monitoring ve Alerting**
   - Real-time monitoring implement edin
   - Automated alerting kurun
   - Performance metrics izleyin
   - Health checks automate edin

4. **Disaster Recovery**
   - Recovery procedures document edin
   - Regular testing yapın
   - Backup strategies implement edin
   - Failover automation kurun

## Kaynaklar

- [SQL Server Replication](https://docs.microsoft.com/en-us/sql/relational-databases/replication/)
- [Database Replication Strategies](https://docs.microsoft.com/en-us/azure/architecture/patterns/replication)
- [Replication Best Practices](https://docs.microsoft.com/en-us/sql/relational-databases/replication/administration/best-practices-for-replication-administration)
- [Conflict Resolution](https://docs.microsoft.com/en-us/sql/relational-databases/replication/merge/advanced-merge-replication-conflict-resolution)
- [High Availability](https://docs.microsoft.com/en-us/sql/database-engine/availability-groups/)
