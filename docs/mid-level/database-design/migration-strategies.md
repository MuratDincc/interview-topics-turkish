# Database Migration Strategies

## Giriş

Database Migration Strategies, veritabanı şemasındaki değişiklikleri güvenli ve kontrollü bir şekilde yönetmek için kullanılan yaklaşımlardır. Mid-level geliştiriciler için migration stratejilerini anlamak, production ortamlarında veri kaybı olmadan değişiklik yapmak için kritiktir.

## Migration Stratejileri

### 1. Forward-Only Migrations
Her migration'ın sadece ileri yönde çalıştığı, geri alınamayan yaklaşım.

```csharp
// Entity Framework Core migration
public partial class AddUserProfileTable : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "UserProfiles",
            columns: table => new
            {
                Id = table.Column<int>(type: "int", nullable: false)
                    .Annotation("SqlServer:Identity", "1, 1"),
                UserId = table.Column<int>(type: "int", nullable: false),
                FirstName = table.Column<string>(type: "nvarchar(50)", maxLength: 50, nullable: true),
                LastName = table.Column<string>(type: "nvarchar(50)", maxLength: 50, nullable: true),
                DateOfBirth = table.Column<DateTime>(type: "datetime2", nullable: true),
                PhoneNumber = table.Column<string>(type: "nvarchar(20)", maxLength: 20, nullable: true),
                CreatedAt = table.Column<DateTime>(type: "datetime2", nullable: false, defaultValueSql: "GETDATE()"),
                UpdatedAt = table.Column<DateTime>(type: "datetime2", nullable: false, defaultValueSql: "GETDATE()")
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_UserProfiles", x => x.Id);
                table.ForeignKey(
                    name: "FK_UserProfiles_Users_UserId",
                    column: x => x.UserId,
                    principalTable: "Users",
                    principalColumn: "Id",
                    onDelete: ReferentialAction.Cascade);
            });

        migrationBuilder.CreateIndex(
            name: "IX_UserProfiles_UserId",
            table: "UserProfiles",
            column: "UserId",
            unique: true);
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(
            name: "UserProfiles");
    }
}
```

### 2. Reversible Migrations
Her migration'ın geri alınabilir olduğu yaklaşım.

```csharp
public partial class AddUserProfileTable : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // Create table
        migrationBuilder.CreateTable(
            name: "UserProfiles",
            columns: table => new
            {
                Id = table.Column<int>(type: "int", nullable: false)
                    .Annotation("SqlServer:Identity", "1, 1"),
                UserId = table.Column<int>(type: "int", nullable: false),
                FirstName = table.Column<string>(type: "nvarchar(50)", maxLength: 50, nullable: true),
                LastName = table.Column<string>(type: "nvarchar(50)", maxLength: 50, nullable: true),
                DateOfBirth = table.Column<DateTime>(type: "datetime2", nullable: true),
                PhoneNumber = table.Column<string>(type: "nvarchar(20)", maxLength: 20, nullable: true),
                CreatedAt = table.Column<DateTime>(type: "datetime2", nullable: false, defaultValueSql: "GETDATE()"),
                UpdatedAt = table.Column<DateTime>(type: "datetime2", nullable: false, defaultValueSql: "GETDATE()")
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_UserProfiles", x => x.Id);
                table.ForeignKey(
                    name: "FK_UserProfiles_Users_UserId",
                    column: x => x.UserId,
                    principalTable: "Users",
                    principalColumn: "Id",
                    onDelete: ReferentialAction.Cascade);
            });

        // Create index
        migrationBuilder.CreateIndex(
            name: "IX_UserProfiles_UserId",
            table: "UserProfiles",
            column: "UserId",
            unique: true);

        // Data migration
        migrationBuilder.Sql(@"
            INSERT INTO UserProfiles (UserId, FirstName, LastName, CreatedAt, UpdatedAt)
            SELECT Id, 
                   SUBSTRING(Name, 1, CHARINDEX(' ', Name + ' ') - 1) as FirstName,
                   SUBSTRING(Name, CHARINDEX(' ', Name + ' ') + 1, LEN(Name)) as LastName,
                   CreatedAt, UpdatedAt
            FROM Users
            WHERE Name IS NOT NULL AND Name <> ''
        ");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        // Drop index
        migrationBuilder.DropIndex(
            name: "IX_UserProfiles_UserId",
            table: "UserProfiles");

        // Drop table
        migrationBuilder.DropTable(
            name: "UserProfiles");
    }
}
```

## Migration Deployment Stratejileri

### 1. Blue-Green Deployment
Sıfır downtime ile migration yapma stratejisi.

```csharp
public class BlueGreenMigrationService
{
    private readonly IConfiguration _configuration;
    private readonly ILogger<BlueGreenMigrationService> _logger;
    
    public async Task DeployMigrationAsync(string connectionString, string migrationScript)
    {
        try
        {
            // Step 1: Create new database (Green)
            var greenConnectionString = await CreateGreenDatabaseAsync(connectionString);
            
            // Step 2: Apply migration to Green database
            await ApplyMigrationToDatabaseAsync(greenConnectionString, migrationScript);
            
            // Step 3: Verify Green database
            await VerifyDatabaseAsync(greenConnectionString);
            
            // Step 4: Switch traffic to Green database
            await SwitchTrafficToGreenAsync(greenConnectionString);
            
            // Step 5: Clean up old database (Blue)
            await CleanupBlueDatabaseAsync(connectionString);
            
            _logger.LogInformation("Blue-Green migration completed successfully");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Blue-Green migration failed");
            await RollbackToBlueAsync(connectionString);
            throw;
        }
    }
    
    private async Task<string> CreateGreenDatabaseAsync(string blueConnectionString)
    {
        // Create new database with timestamp suffix
        var timestamp = DateTime.UtcNow.ToString("yyyyMMddHHmmss");
        var greenDbName = $"Database_{timestamp}";
        
        // Implementation details...
        return greenConnectionString.Replace("Database", greenDbName);
    }
    
    private async Task SwitchTrafficToGreenAsync(string greenConnectionString)
    {
        // Update connection string in configuration
        // Update load balancer settings
        // Update application configuration
        // Implementation details...
    }
}
```

### 2. Rolling Migration
Aşamalı olarak migration yapma stratejisi.

```csharp
public class RollingMigrationService
{
    private readonly IConfiguration _configuration;
    private readonly ILogger<RollingMigrationService> _logger;
    
    public async Task DeployRollingMigrationAsync(string migrationScript)
    {
        try
        {
            // Step 1: Deploy to first instance
            await DeployToInstanceAsync("instance-1", migrationScript);
            
            // Step 2: Verify first instance
            await VerifyInstanceAsync("instance-1");
            
            // Step 3: Deploy to second instance
            await DeployToInstanceAsync("instance-2", migrationScript);
            
            // Step 4: Verify second instance
            await VerifyInstanceAsync("instance-2");
            
            // Step 5: Continue with remaining instances
            var instances = await GetRemainingInstancesAsync();
            foreach (var instance in instances)
            {
                await DeployToInstanceAsync(instance, migrationScript);
                await VerifyInstanceAsync(instance);
            }
            
            _logger.LogInformation("Rolling migration completed successfully");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Rolling migration failed");
            await RollbackMigrationAsync();
            throw;
        }
    }
    
    private async Task DeployToInstanceAsync(string instanceId, string migrationScript)
    {
        _logger.LogInformation("Deploying migration to instance: {InstanceId}", instanceId);
        
        // Take instance out of load balancer
        await RemoveFromLoadBalancerAsync(instanceId);
        
        // Apply migration
        await ApplyMigrationAsync(instanceId, migrationScript);
        
        // Verify migration
        await VerifyMigrationAsync(instanceId);
        
        // Put instance back in load balancer
        await AddToLoadBalancerAsync(instanceId);
    }
}
```

## Data Migration Stratejileri

### 1. Schema-First Migration
Önce şema değişikliği, sonra veri migration'ı.

```csharp
public class SchemaFirstMigrationService
{
    public async Task MigrateSchemaFirstAsync()
    {
        // Step 1: Add new nullable columns
        await AddNewColumnsAsync();
        
        // Step 2: Migrate data
        await MigrateDataAsync();
        
        // Step 3: Make columns non-nullable
        await MakeColumnsRequiredAsync();
        
        // Step 4: Remove old columns
        await RemoveOldColumnsAsync();
    }
    
    private async Task AddNewColumnsAsync()
    {
        var sql = @"
            ALTER TABLE Users 
            ADD FirstName NVARCHAR(50) NULL,
                LastName NVARCHAR(50) NULL
        ";
        
        await ExecuteSqlAsync(sql);
    }
    
    private async Task MigrateDataAsync()
    {
        var sql = @"
            UPDATE Users 
            SET FirstName = SUBSTRING(Name, 1, CHARINDEX(' ', Name + ' ') - 1),
                LastName = SUBSTRING(Name, CHARINDEX(' ', Name + ' ') + 1, LEN(Name))
            WHERE Name IS NOT NULL AND Name <> ''
        ";
        
        await ExecuteSqlAsync(sql);
    }
    
    private async Task MakeColumnsRequiredAsync()
    {
        var sql = @"
            ALTER TABLE Users 
            ALTER COLUMN FirstName NVARCHAR(50) NOT NULL;
            
            ALTER TABLE Users 
            ALTER COLUMN LastName NVARCHAR(50) NOT NULL;
        ";
        
        await ExecuteSqlAsync(sql);
    }
    
    private async Task RemoveOldColumnsAsync()
    {
        var sql = "ALTER TABLE Users DROP COLUMN Name";
        await ExecuteSqlAsync(sql);
    }
}
```

### 2. Data-First Migration
Önce veri migration'ı, sonra şema değişikliği.

```csharp
public class DataFirstMigrationService
{
    public async Task MigrateDataFirstAsync()
    {
        // Step 1: Create new table structure
        await CreateNewTableStructureAsync();
        
        // Step 2: Migrate data to new structure
        await MigrateDataToNewStructureAsync();
        
        // Step 3: Verify data integrity
        await VerifyDataIntegrityAsync();
        
        // Step 4: Switch to new table
        await SwitchToNewTableAsync();
        
        // Step 5: Clean up old table
        await CleanupOldTableAsync();
    }
    
    private async Task CreateNewTableStructureAsync()
    {
        var sql = @"
            CREATE TABLE Users_New (
                Id INT PRIMARY KEY,
                Username NVARCHAR(50) NOT NULL,
                Email NVARCHAR(100) NOT NULL,
                FirstName NVARCHAR(50) NOT NULL,
                LastName NVARCHAR(50) NOT NULL,
                IsActive BIT NOT NULL DEFAULT 1,
                CreatedAt DATETIME2 NOT NULL DEFAULT GETDATE(),
                UpdatedAt DATETIME2 NOT NULL DEFAULT GETDATE()
            )
        ";
        
        await ExecuteSqlAsync(sql);
    }
    
    private async Task MigrateDataToNewStructureAsync()
    {
        var sql = @"
            INSERT INTO Users_New (Id, Username, Email, FirstName, LastName, IsActive, CreatedAt, UpdatedAt)
            SELECT 
                Id,
                Username,
                Email,
                SUBSTRING(Name, 1, CHARINDEX(' ', Name + ' ') - 1) as FirstName,
                SUBSTRING(Name, CHARINDEX(' ', Name + ' ') + 1, LEN(Name)) as LastName,
                IsActive,
                CreatedAt,
                UpdatedAt
            FROM Users
        ";
        
        await ExecuteSqlAsync(sql);
    }
    
    private async Task SwitchToNewTableAsync()
    {
        // Rename tables
        var sql = @"
            EXEC sp_rename 'Users', 'Users_Old';
            EXEC sp_rename 'Users_New', 'Users';
        ";
        
        await ExecuteSqlAsync(sql);
    }
}
```

## Migration Testing Stratejileri

### 1. Migration Testing
```csharp
public class MigrationTests
{
    [Fact]
    public async Task Migration_ShouldPreserveDataIntegrity()
    {
        // Arrange
        var testData = CreateTestData();
        var migrationService = new MigrationService();
        
        // Act
        await migrationService.MigrateAsync();
        
        // Assert
        var migratedData = await GetMigratedDataAsync();
        Assert.Equal(testData.Count, migratedData.Count);
        
        foreach (var original in testData)
        {
            var migrated = migratedData.FirstOrDefault(x => x.Id == original.Id);
            Assert.NotNull(migrated);
            Assert.Equal(original.Username, migrated.Username);
            Assert.Equal(original.Email, migrated.Email);
        }
    }
    
    [Fact]
    public async Task Migration_ShouldHandleLargeDatasets()
    {
        // Arrange
        var largeDataset = CreateLargeDataset(100000); // 100K records
        var migrationService = new MigrationService();
        
        // Act & Assert
        var stopwatch = Stopwatch.StartNew();
        await migrationService.MigrateAsync();
        stopwatch.Stop();
        
        // Migration should complete within reasonable time
        Assert.True(stopwatch.ElapsedMilliseconds < 30000); // 30 seconds
    }
    
    [Fact]
    public async Task Migration_ShouldBeReversible()
    {
        // Arrange
        var originalData = await GetCurrentDataAsync();
        var migrationService = new MigrationService();
        
        // Act
        await migrationService.MigrateAsync();
        await migrationService.RollbackAsync();
        
        // Assert
        var rolledBackData = await GetCurrentDataAsync();
        Assert.Equal(originalData.Count, rolledBackData.Count);
        
        // Verify data is identical
        foreach (var original in originalData)
        {
            var rolledBack = rolledBackData.FirstOrDefault(x => x.Id == original.Id);
            Assert.NotNull(rolledBack);
            Assert.Equal(original, rolledBack);
        }
    }
}
```

### 2. Migration Validation
```csharp
public class MigrationValidator
{
    public async Task<ValidationResult> ValidateMigrationAsync(string migrationScript)
    {
        var result = new ValidationResult();
        
        try
        {
            // Syntax validation
            await ValidateSyntaxAsync(migrationScript, result);
            
            // Data integrity validation
            await ValidateDataIntegrityAsync(migrationScript, result);
            
            // Performance validation
            await ValidatePerformanceAsync(migrationScript, result);
            
            // Rollback validation
            await ValidateRollbackAsync(migrationScript, result);
        }
        catch (Exception ex)
        {
            result.AddError("Validation failed", ex.Message);
        }
        
        return result;
    }
    
    private async Task ValidateSyntaxAsync(string migrationScript, ValidationResult result)
    {
        try
        {
            // Parse SQL syntax
            var parsed = await ParseSqlAsync(migrationScript);
            if (!parsed.IsValid)
            {
                result.AddError("Syntax Error", parsed.ErrorMessage);
            }
        }
        catch (Exception ex)
        {
            result.AddError("Syntax Validation Failed", ex.Message);
        }
    }
    
    private async Task ValidateDataIntegrityAsync(string migrationScript, ValidationResult result)
    {
        // Check for potential data loss
        if (migrationScript.Contains("DROP TABLE") || migrationScript.Contains("TRUNCATE TABLE"))
        {
            result.AddWarning("Data Loss Risk", "Migration contains operations that may cause data loss");
        }
        
        // Check for foreign key constraints
        if (migrationScript.Contains("ALTER TABLE") && !migrationScript.Contains("FOREIGN KEY"))
        {
            result.AddWarning("Constraint Risk", "Table alteration without foreign key consideration");
        }
    }
}
```

## Mülakat Soruları

### Temel Sorular

1. **Database migration nedir ve neden önemlidir?**
   - **Cevap**: Veritabanı şemasındaki değişiklikleri yönetme süreci. Version control, rollback, team collaboration için kritik.

2. **Forward-only vs reversible migrations arasındaki fark nedir?**
   - **Cevap**: Forward-only sadece ileri, reversible geri alınabilir. Production'da forward-only daha güvenli.

3. **Blue-green deployment nedir?**
   - **Cevap**: Sıfır downtime ile migration yapma stratejisi. Yeni database oluşturulur, traffic switch edilir.

4. **Rolling migration nedir?**
   - **Cevap**: Aşamalı olarak migration yapma. Her instance sırayla güncellenir, availability korunur.

5. **Schema-first vs data-first migration arasındaki fark nedir?**
   - **Cevap**: Schema-first önce şema sonra veri, data-first önce veri sonra şema. Risk ve complexity farklı.

### Teknik Sorular

1. **Migration'da data loss nasıl önlenir?**
   - **Cevap**: Backup alınır, test environment'da çalıştırılır, validation yapılır, rollback plan hazırlanır.

2. **Large dataset'lerde migration performance nasıl optimize edilir?**
   - **Cevap**: Batch processing, parallel execution, index management, maintenance window kullanılır.

3. **Migration testing'de hangi testler yapılır?**
   - **Cevap**: Data integrity, performance, rollback, large dataset, edge cases testing.

4. **Production'da migration rollback nasıl yapılır?**
   - **Cevap**: Automated rollback scripts, backup restoration, blue-green switchback, rolling rollback.

5. **Migration'da team collaboration nasıl sağlanır?**
   - **Cevap**: Version control, code review, migration scripts, documentation, communication.

## Best Practices

1. **Planning**
   - Migration strategy belirleyin
   - Risk assessment yapın
   - Rollback plan hazırlayın
   - Testing plan oluşturun

2. **Execution**
   - Maintenance window kullanın
   - Backup alın
   - Monitoring yapın
   - Communication sağlayın

3. **Validation**
   - Data integrity kontrol edin
   - Performance test edin
   - Rollback test edin
   - User acceptance test yapın

4. **Documentation**
   - Migration scripts document edin
   - Rollback procedures yazın
   - Lessons learned kaydedin
   - Runbook oluşturun

## Kaynaklar

- [Entity Framework Core Migrations](https://docs.microsoft.com/en-us/ef/core/managing-schemas/migrations/)
- [Database Migration Best Practices](https://martinfowler.com/articles/evodb.html)
- [Blue-Green Deployment](https://martinfowler.com/bliki/BlueGreenDeployment.html)
- [Database Migration Strategies](https://www.atlassian.com/continuous-delivery/continuous-delivery-principles/database-migration)
- [Migration Testing](https://docs.microsoft.com/en-us/ef/core/testing/)
