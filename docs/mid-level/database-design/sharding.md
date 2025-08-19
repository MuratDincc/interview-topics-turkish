# Database Sharding

## Giriş

Database Sharding, büyük veritabanlarını daha küçük, yönetilebilir parçalara (shard'lara) bölme tekniğidir. Mid-level geliştiriciler için sharding stratejilerini anlamak, ölçeklenebilir ve yüksek performanslı veritabanı sistemleri tasarlamada kritiktir.

## Sharding Stratejileri

### 1. Horizontal Sharding (Row-based)
Veri satırlarını farklı shard'lara dağıtma stratejisi.

#### Hash-based Sharding
```csharp
public class HashBasedShardingStrategy : IShardingStrategy
{
    private readonly int _numberOfShards;
    
    public HashBasedShardingStrategy(int numberOfShards)
    {
        _numberOfShards = numberOfShards;
    }
    
    public int GetShardId(object shardKey)
    {
        var hash = shardKey.GetHashCode();
        var shardId = Math.Abs(hash) % _numberOfShards;
        return shardId;
    }
    
    public string GetConnectionString(int shardId)
    {
        return $"Server=shard-{shardId};Database=UserDB;Trusted_Connection=true;";
    }
}

// Kullanım örneği
public class UserRepository
{
    private readonly HashBasedShardingStrategy _shardingStrategy;
    private readonly Dictionary<int, IDbConnection> _shardConnections;
    
    public UserRepository(int numberOfShards)
    {
        _shardingStrategy = new HashBasedShardingStrategy(numberOfShards);
        _shardConnections = InitializeShardConnections(numberOfShards);
    }
    
    public async Task<User> GetUserByIdAsync(int userId)
    {
        var shardId = _shardingStrategy.GetShardId(userId);
        var connection = _shardConnections[shardId];
        
        var sql = "SELECT * FROM Users WHERE Id = @UserId";
        return await connection.QueryFirstOrDefaultAsync<User>(sql, new { UserId = userId });
    }
    
    public async Task<int> CreateUserAsync(User user)
    {
        var shardId = _shardingStrategy.GetShardId(user.Id);
        var connection = _shardConnections[shardId];
        
        var sql = @"
            INSERT INTO Users (Id, Username, Email, CreatedAt) 
            VALUES (@Id, @Username, @Email, @CreatedAt);
            SELECT CAST(SCOPE_IDENTITY() as int)
        ";
        
        return await connection.QuerySingleAsync<int>(sql, user);
    }
}
```

#### Range-based Sharding
```csharp
public class RangeBasedShardingStrategy : IShardingStrategy
{
    private readonly List<ShardRange> _shardRanges;
    
    public RangeBasedShardingStrategy()
    {
        _shardRanges = new List<ShardRange>
        {
            new ShardRange { ShardId = 0, MinValue = 1, MaxValue = 1000000, ConnectionString = "shard-0" },
            new ShardRange { ShardId = 1, MinValue = 1000001, MaxValue = 2000000, ConnectionString = "shard-1" },
            new ShardRange { ShardId = 2, MinValue = 2000001, MaxValue = 3000000, ConnectionString = "shard-2" },
            new ShardRange { ShardId = 3, MinValue = 3000001, MaxValue = 4000000, ConnectionString = "shard-3" }
        };
    }
    
    public int GetShardId(object shardKey)
    {
        if (shardKey is int intKey)
        {
            var shard = _shardRanges.FirstOrDefault(s => intKey >= s.MinValue && intKey <= s.MaxValue);
            return shard?.ShardId ?? 0;
        }
        
        throw new ArgumentException("Shard key must be an integer");
    }
    
    public string GetConnectionString(int shardId)
    {
        var shard = _shardRanges.FirstOrDefault(s => s.ShardId == shardId);
        return shard?.ConnectionString ?? _shardRanges[0].ConnectionString;
    }
}

public class ShardRange
{
    public int ShardId { get; set; }
    public long MinValue { get; set; }
    public long MaxValue { get; set; }
    public string ConnectionString { get; set; }
}
```

#### Directory-based Sharding
```csharp
public class DirectoryBasedShardingStrategy : IShardingStrategy
{
    private readonly Dictionary<string, int> _shardDirectory;
    private readonly Dictionary<int, string> _shardConnections;
    
    public DirectoryBasedShardingStrategy()
    {
        _shardDirectory = new Dictionary<string, int>();
        _shardConnections = new Dictionary<int, string>
        {
            { 0, "shard-0" },
            { 1, "shard-1" },
            { 2, "shard-2" },
            { 3, "shard-3" }
        };
        
        InitializeShardDirectory();
    }
    
    private void InitializeShardDirectory()
    {
        // User ID ranges to shard mapping
        for (int i = 1; i <= 1000000; i++)
        {
            _shardDirectory[i.ToString()] = i % 4;
        }
        
        // Username-based sharding
        _shardDirectory["admin"] = 0;
        _shardDirectory["system"] = 0;
        _shardDirectory["guest"] = 1;
        _shardDirectory["test"] = 2;
    }
    
    public int GetShardId(object shardKey)
    {
        var key = shardKey.ToString();
        
        if (_shardDirectory.TryGetValue(key, out int shardId))
        {
            return shardId;
        }
        
        // Fallback to hash-based for unknown keys
        return Math.Abs(key.GetHashCode()) % _shardConnections.Count;
    }
    
    public string GetConnectionString(int shardId)
    {
        return _shardConnections.TryGetValue(shardId, out string connectionString) 
            ? connectionString 
            : _shardConnections[0];
    }
}
```

### 2. Vertical Sharding (Column-based)
Farklı sütunları farklı shard'lara dağıtma stratejisi.

```csharp
public class VerticalShardingStrategy : IShardingStrategy
{
    private readonly Dictionary<string, int> _columnShardMapping;
    private readonly Dictionary<int, string> _shardConnections;
    
    public VerticalShardingStrategy()
    {
        _columnShardMapping = new Dictionary<string, int>
        {
            // Frequently accessed columns in shard 0
            { "Id", 0 },
            { "Username", 0 },
            { "Email", 0 },
            { "IsActive", 0 },
            
            // Less frequently accessed columns in shard 1
            { "FirstName", 1 },
            { "LastName", 1 },
            { "DateOfBirth", 1 },
            { "PhoneNumber", 1 },
            
            // Rarely accessed columns in shard 2
            { "Address", 2 },
            { "Bio", 2 },
            { "ProfilePictureUrl", 2 },
            { "Preferences", 2 }
        };
        
        _shardConnections = new Dictionary<int, string>
        {
            { 0, "shard-core" },
            { 1, "shard-profile" },
            { 2, "shard-extended" }
        };
    }
    
    public int GetShardId(string columnName)
    {
        return _columnShardMapping.TryGetValue(columnName, out int shardId) ? shardId : 0;
    }
    
    public string GetConnectionString(int shardId)
    {
        return _shardConnections.TryGetValue(shardId, out string connectionString) 
            ? connectionString 
            : _shardConnections[0];
    }
}

public class VerticallyShardedUserRepository
{
    private readonly VerticalShardingStrategy _shardingStrategy;
    private readonly Dictionary<int, IDbConnection> _shardConnections;
    
    public VerticallyShardedUserRepository()
    {
        _shardingStrategy = new VerticalShardingStrategy();
        _shardConnections = InitializeShardConnections();
    }
    
    public async Task<User> GetUserByIdAsync(int userId)
    {
        var user = new User();
        
        // Get core data from shard 0
        var coreConnection = _shardConnections[0];
        var coreSql = "SELECT Id, Username, Email, IsActive FROM Users WHERE Id = @UserId";
        var coreData = await coreConnection.QueryFirstOrDefaultAsync(coreSql, new { UserId = userId });
        
        if (coreData != null)
        {
            user.Id = coreData.Id;
            user.Username = coreData.Username;
            user.Email = coreData.Email;
            user.IsActive = coreData.IsActive;
            
            // Get profile data from shard 1
            var profileConnection = _shardConnections[1];
            var profileSql = "SELECT FirstName, LastName, DateOfBirth, PhoneNumber FROM UserProfiles WHERE UserId = @UserId";
            var profileData = await profileConnection.QueryFirstOrDefaultAsync(profileSql, new { UserId = userId });
            
            if (profileData != null)
            {
                user.FirstName = profileData.FirstName;
                user.LastName = profileData.LastName;
                user.DateOfBirth = profileData.DateOfBirth;
                user.PhoneNumber = profileData.PhoneNumber;
            }
            
            // Get extended data from shard 2
            var extendedConnection = _shardConnections[2];
            var extendedSql = "SELECT Address, Bio, ProfilePictureUrl, Preferences FROM UserExtended WHERE UserId = @UserId";
            var extendedData = await extendedConnection.QueryFirstOrDefaultAsync(extendedSql, new { UserId = userId });
            
            if (extendedData != null)
            {
                user.Address = extendedData.Address;
                user.Bio = extendedData.Bio;
                user.ProfilePictureUrl = extendedData.ProfilePictureUrl;
                user.Preferences = extendedData.Preferences;
            }
        }
        
        return user;
    }
}
```

## Sharding Implementation

### 1. Shard Management Service
```csharp
public class ShardManagementService
{
    private readonly IShardingStrategy _shardingStrategy;
    private readonly Dictionary<int, ShardInfo> _shards;
    private readonly ILogger<ShardManagementService> _logger;
    
    public ShardManagementService(IShardingStrategy shardingStrategy, ILogger<ShardManagementService> logger)
    {
        _shardingStrategy = shardingStrategy;
        _logger = logger;
        _shards = new Dictionary<int, ShardInfo>();
    }
    
    public async Task<ShardInfo> GetShardAsync(object shardKey)
    {
        var shardId = _shardingStrategy.GetShardId(shardKey);
        
        if (!_shards.ContainsKey(shardId))
        {
            await InitializeShardAsync(shardId);
        }
        
        return _shards[shardId];
    }
    
    public async Task<bool> IsShardHealthyAsync(int shardId)
    {
        try
        {
            var shard = _shards[shardId];
            using var connection = new SqlConnection(shard.ConnectionString);
            await connection.OpenAsync();
            
            // Simple health check query
            var result = await connection.QueryFirstOrDefaultAsync<int>("SELECT 1");
            return result == 1;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Health check failed for shard {ShardId}", shardId);
            return false;
        }
    }
    
    public async Task RebalanceShardAsync(int shardId, object newShardKey)
    {
        var shard = _shards[shardId];
        var newShardId = _shardingStrategy.GetShardId(newShardKey);
        
        if (shardId != newShardId)
        {
            await MigrateDataAsync(shardId, newShardId);
            await UpdateShardMappingAsync(shardId, newShardId);
        }
    }
    
    private async Task MigrateDataAsync(int fromShardId, int toShardId)
    {
        var fromShard = _shards[fromShardId];
        var toShard = _shards[toShardId];
        
        _logger.LogInformation("Migrating data from shard {FromShardId} to {ToShardId}", fromShardId, toShardId);
        
        // Implementation of data migration logic
        // This would involve copying data between shards
        // and updating any cross-shard references
    }
}

public class ShardInfo
{
    public int ShardId { get; set; }
    public string ConnectionString { get; set; }
    public ShardStatus Status { get; set; }
    public DateTime LastHealthCheck { get; set; }
    public long RecordCount { get; set; }
    public long SizeInBytes { get; set; }
}

public enum ShardStatus
{
    Healthy,
    Degraded,
    Unhealthy,
    Maintenance
}
```

### 2. Cross-Shard Query Service
```csharp
public class CrossShardQueryService
{
    private readonly IShardingStrategy _shardingStrategy;
    private readonly Dictionary<int, IDbConnection> _shardConnections;
    private readonly ILogger<CrossShardQueryService> _logger;
    
    public CrossShardQueryService(IShardingStrategy shardingStrategy, ILogger<CrossShardQueryService> logger)
    {
        _shardingStrategy = shardingStrategy;
        _logger = logger;
        _shardConnections = InitializeShardConnections();
    }
    
    public async Task<List<User>> SearchUsersAcrossShardsAsync(string searchTerm)
    {
        var results = new List<User>();
        var tasks = new List<Task<List<User>>>();
        
        // Query all shards in parallel
        foreach (var shardConnection in _shardConnections.Values)
        {
            var task = SearchUsersInShardAsync(shardConnection, searchTerm);
            tasks.Add(task);
        }
        
        // Wait for all queries to complete
        var shardResults = await Task.WhenAll(tasks);
        
        // Combine results from all shards
        foreach (var shardResult in shardResults)
        {
            results.AddRange(shardResult);
        }
        
        // Sort and limit results
        return results.OrderBy(u => u.Username).Take(100).ToList();
    }
    
    public async Task<Dictionary<string, int>> GetUserCountByShardAsync()
    {
        var results = new Dictionary<string, int>();
        var tasks = new List<Task<KeyValuePair<string, int>>>();
        
        foreach (var kvp in _shardConnections)
        {
            var shardId = kvp.Key;
            var connection = kvp.Value;
            
            var task = GetUserCountForShardAsync(shardId, connection);
            tasks.Add(task);
        }
        
        var shardCounts = await Task.WhenAll(tasks);
        
        foreach (var shardCount in shardCounts)
        {
            results[shardCount.Key] = shardCount.Value;
        }
        
        return results;
    }
    
    private async Task<List<User>> SearchUsersInShardAsync(IDbConnection connection, string searchTerm)
    {
        var sql = @"
            SELECT Id, Username, Email, FirstName, LastName, IsActive
            FROM Users 
            WHERE Username LIKE @SearchTerm 
               OR Email LIKE @SearchTerm 
               OR FirstName LIKE @SearchTerm 
               OR LastName LIKE @SearchTerm
        ";
        
        var searchPattern = $"%{searchTerm}%";
        var users = await connection.QueryAsync<User>(sql, new { SearchTerm = searchPattern });
        
        return users.ToList();
    }
    
    private async Task<KeyValuePair<string, int>> GetUserCountForShardAsync(int shardId, IDbConnection connection)
    {
        var sql = "SELECT COUNT(*) FROM Users";
        var count = await connection.QueryFirstOrDefaultAsync<int>(sql);
        
        return new KeyValuePair<string, int>($"shard-{shardId}", count);
    }
}
```

## Sharding Challenges ve Solutions

### 1. Shard Key Selection
```csharp
public class ShardKeyAnalyzer
{
    public ShardKeyRecommendation AnalyzeShardKey(string tableName, List<string> candidateColumns)
    {
        var recommendations = new List<ShardKeyRecommendation>();
        
        foreach (var column in candidateColumns)
        {
            var recommendation = AnalyzeColumnForSharding(tableName, column);
            recommendations.Add(recommendation);
        }
        
        return recommendations.OrderByDescending(r => r.Score).First();
    }
    
    private ShardKeyRecommendation AnalyzeColumnForSharding(string tableName, string columnName)
    {
        var recommendation = new ShardKeyRecommendation
        {
            ColumnName = columnName,
            Score = 0
        };
        
        // Check cardinality (higher is better)
        var cardinality = GetColumnCardinality(tableName, columnName);
        recommendation.Score += cardinality * 0.3;
        
        // Check distribution (more uniform is better)
        var distribution = GetColumnDistribution(tableName, columnName);
        recommendation.Score += distribution * 0.3;
        
        // Check query patterns (more queries is better)
        var queryFrequency = GetQueryFrequency(tableName, columnName);
        recommendation.Score += queryFrequency * 0.2;
        
        // Check update frequency (lower is better)
        var updateFrequency = GetUpdateFrequency(tableName, columnName);
        recommendation.Score += (1 - updateFrequency) * 0.2;
        
        return recommendation;
    }
}

public class ShardKeyRecommendation
{
    public string ColumnName { get; set; }
    public double Score { get; set; }
    public string Reasoning { get; set; }
}
```

### 2. Shard Rebalancing
```csharp
public class ShardRebalancingService
{
    private readonly IShardingStrategy _shardingStrategy;
    private readonly ILogger<ShardRebalancingService> _logger;
    
    public async Task RebalanceShardsAsync()
    {
        var shardStats = await GetShardStatisticsAsync();
        var targetShardSize = CalculateTargetShardSize(shardStats);
        
        var rebalancingPlan = CreateRebalancingPlan(shardStats, targetShardSize);
        
        foreach (var operation in rebalancingPlan)
        {
            await ExecuteRebalancingOperationAsync(operation);
        }
    }
    
    private async Task ExecuteRebalancingOperationAsync(RebalancingOperation operation)
    {
        try
        {
            _logger.LogInformation("Executing rebalancing operation: {Operation}", operation.Description);
            
            switch (operation.Type)
            {
                case RebalancingOperationType.MoveData:
                    await MoveDataBetweenShardsAsync(operation.FromShardId, operation.ToShardId, operation.RecordCount);
                    break;
                    
                case RebalancingOperationType.SplitShard:
                    await SplitShardAsync(operation.FromShardId, operation.ToShardId);
                    break;
                    
                case RebalancingOperationType.MergeShards:
                    await MergeShardsAsync(operation.FromShardId, operation.ToShardId);
                    break;
            }
            
            _logger.LogInformation("Rebalancing operation completed: {Operation}", operation.Description);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Rebalancing operation failed: {Operation}", operation.Description);
            throw;
        }
    }
}

public class RebalancingOperation
{
    public RebalancingOperationType Type { get; set; }
    public int FromShardId { get; set; }
    public int ToShardId { get; set; }
    public long RecordCount { get; set; }
    public string Description { get; set; }
}

public enum RebalancingOperationType
{
    MoveData,
    SplitShard,
    MergeShards
}
```

## Mülakat Soruları

### Temel Sorular

1. **Database sharding nedir ve neden kullanılır?**
   - **Cevap**: Büyük veritabanlarını küçük parçalara bölme tekniği. Performance, scalability ve availability artışı için kullanılır.

2. **Horizontal vs Vertical sharding arasındaki fark nedir?**
   - **Cevap**: Horizontal satırları böler, vertical sütunları böler. Horizontal daha yaygın, vertical belirli use case'ler için.

3. **Hash-based vs Range-based sharding arasındaki fark nedir?**
   - **Cevap**: Hash-based uniform distribution sağlar, range-based ordered access sağlar. Query patterns'e göre seçim yapılır.

4. **Shard key seçiminde hangi faktörler önemlidir?**
   - **Cevap**: Cardinality, distribution uniformity, query patterns, update frequency, data locality.

5. **Cross-shard query'ler nasıl optimize edilir?**
   - **Cevap**: Parallel execution, result aggregation, caching, query routing optimization.

### Teknik Sorular

1. **Sharding'de data consistency nasıl sağlanır?**
   - **Cevap**: Distributed transactions, eventual consistency, conflict resolution, data versioning.

2. **Shard rebalancing nasıl yapılır?**
   - **Cevap**: Data migration, shard splitting/merging, load balancing, minimal downtime.

3. **Sharding'de failure handling nasıl yapılır?**
   - **Cevap**: Health monitoring, automatic failover, data replication, backup strategies.

4. **Sharding vs partitioning arasındaki fark nedir?**
   - **Cevap**: Sharding farklı database'lerde, partitioning aynı database'de. Sharding daha complex ama daha scalable.

5. **Sharding implementation'da hangi challenges vardır?**
   - **Cevap**: Shard key selection, cross-shard queries, data distribution, rebalancing, monitoring.

## Best Practices

1. **Shard Key Selection**
   - High cardinality seçin
   - Uniform distribution sağlayın
   - Query patterns analiz edin
   - Update frequency düşük olsun

2. **Shard Management**
   - Health monitoring yapın
   - Automatic failover implement edin
   - Rebalancing automate edin
   - Performance metrics izleyin

3. **Cross-Shard Operations**
   - Parallel execution kullanın
   - Result aggregation optimize edin
   - Caching implement edin
   - Query routing optimize edin

4. **Monitoring ve Maintenance**
   - Shard performance izleyin
   - Data distribution balance edin
   - Backup strategies implement edin
   - Disaster recovery planlayın

## Kaynaklar

- [Database Sharding Strategies](https://docs.microsoft.com/en-us/azure/azure-sql/database/sharding-overview)
- [Sharding Best Practices](https://www.mongodb.com/blog/post/building-with-patterns-the-sharding-pattern)
- [Horizontal vs Vertical Sharding](https://www.cockroachlabs.com/blog/horizontal-vs-vertical-scaling/)
- [Sharding Implementation](https://docs.microsoft.com/en-us/azure/architecture/patterns/sharding)
- [Database Scalability Patterns](https://martinfowler.com/articles/patterns-of-distributed-systems/)
