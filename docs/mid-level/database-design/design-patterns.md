# Database Design Patterns

## Giriş

Database Design Patterns, veritabanı tasarımında karşılaşılan yaygın problemleri çözmek için kullanılan kanıtlanmış çözümlerdir. Mid-level geliştiriciler için bu pattern'leri anlamak, ölçeklenebilir ve maintainable veritabanı sistemleri tasarlamada kritiktir.

## Temel Database Design Patterns

### 1. Normalization Patterns
Veri tekrarını önlemek ve veri bütünlüğünü sağlamak için kullanılan pattern'ler.

#### First Normal Form (1NF)
```sql
-- 1NF Violation - Multiple values in single column
CREATE TABLE Users (
    Id INT PRIMARY KEY,
    Name VARCHAR(100),
    PhoneNumbers VARCHAR(500) -- "555-1234, 555-5678" - VIOLATION
);

-- 1NF Compliant
CREATE TABLE Users (
    Id INT PRIMARY KEY,
    Name VARCHAR(100)
);

CREATE TABLE UserPhoneNumbers (
    Id INT PRIMARY KEY,
    UserId INT,
    PhoneNumber VARCHAR(20),
    FOREIGN KEY (UserId) REFERENCES Users(Id)
);
```

#### Second Normal Form (2NF)
```sql
-- 2NF Violation - Partial dependency
CREATE TABLE Orders (
    OrderId INT PRIMARY KEY,
    CustomerId INT,
    CustomerName VARCHAR(100), -- Depends on CustomerId, not OrderId
    OrderDate DATETIME,
    TotalAmount DECIMAL(10,2)
);

-- 2NF Compliant
CREATE TABLE Customers (
    CustomerId INT PRIMARY KEY,
    CustomerName VARCHAR(100)
);

CREATE TABLE Orders (
    OrderId INT PRIMARY KEY,
    CustomerId INT,
    OrderDate DATETIME,
    TotalAmount DECIMAL(10,2),
    FOREIGN KEY (CustomerId) REFERENCES Customers(CustomerId)
);
```

#### Third Normal Form (3NF)
```sql
-- 3NF Violation - Transitive dependency
CREATE TABLE Orders (
    OrderId INT PRIMARY KEY,
    CustomerId INT,
    CustomerName VARCHAR(100),
    CustomerCity VARCHAR(100), -- Depends on CustomerName, not OrderId
    OrderDate DATETIME
);

-- 3NF Compliant
CREATE TABLE Cities (
    CityId INT PRIMARY KEY,
    CityName VARCHAR(100)
);

CREATE TABLE Customers (
    CustomerId INT PRIMARY KEY,
    CustomerName VARCHAR(100),
    CityId INT,
    FOREIGN KEY (CityId) REFERENCES Cities(CityId)
);

CREATE TABLE Orders (
    OrderId INT PRIMARY KEY,
    CustomerId INT,
    OrderDate DATETIME,
    FOREIGN KEY (CustomerId) REFERENCES Customers(CustomerId)
);
```

### 2. Denormalization Patterns
Performance için bilinçli olarak normalization kurallarını ihlal etme.

#### Read-Optimized Denormalization
```sql
-- Denormalized table for reporting
CREATE TABLE UserActivitySummary (
    UserId INT PRIMARY KEY,
    UserName VARCHAR(100),
    TotalOrders INT,
    TotalSpent DECIMAL(10,2),
    LastOrderDate DATETIME,
    FavoriteCategory VARCHAR(100),
    UpdatedAt DATETIME DEFAULT GETDATE()
);

-- Stored procedure to maintain denormalized data
CREATE PROCEDURE UpdateUserActivitySummary
    @UserId INT
AS
BEGIN
    UPDATE UserActivitySummary
    SET 
        TotalOrders = (
            SELECT COUNT(*) FROM Orders WHERE UserId = @UserId
        ),
        TotalSpent = (
            SELECT SUM(TotalAmount) FROM Orders WHERE UserId = @UserId
        ),
        LastOrderDate = (
            SELECT MAX(OrderDate) FROM Orders WHERE UserId = @UserId
        ),
        FavoriteCategory = (
            SELECT TOP 1 c.CategoryName
            FROM Orders o
            JOIN OrderItems oi ON o.OrderId = oi.OrderId
            JOIN Products p ON oi.ProductId = p.ProductId
            JOIN Categories c ON p.CategoryId = c.CategoryId
            WHERE o.UserId = @UserId
            GROUP BY c.CategoryName
            ORDER BY COUNT(*) DESC
        ),
        UpdatedAt = GETDATE()
    WHERE UserId = @UserId;
END;
```

#### Materialized Views
```sql
-- Materialized view for complex aggregations
CREATE VIEW vw_ProductSalesSummary
WITH SCHEMABINDING
AS
SELECT 
    p.ProductId,
    p.ProductName,
    c.CategoryName,
    COUNT_BIG(*) as TotalOrders,
    SUM(oi.Quantity) as TotalQuantity,
    SUM(oi.Quantity * oi.UnitPrice) as TotalRevenue,
    AVG(oi.UnitPrice) as AveragePrice
FROM dbo.Products p
JOIN dbo.Categories c ON p.CategoryId = c.CategoryId
JOIN dbo.OrderItems oi ON p.ProductId = oi.ProductId
JOIN dbo.Orders o ON oi.OrderId = o.OrderId
GROUP BY p.ProductId, p.ProductName, c.CategoryName;

-- Create indexed view
CREATE UNIQUE CLUSTERED INDEX IX_ProductSalesSummary_ProductId
ON vw_ProductSalesSummary(ProductId);
```

### 3. Partitioning Patterns
Büyük tabloları daha küçük, yönetilebilir parçalara bölme.

#### Horizontal Partitioning (Sharding)
```sql
-- Partition function by date
CREATE PARTITION FUNCTION PF_OrdersByDate (DATETIME)
AS RANGE RIGHT FOR VALUES (
    '2024-01-01', '2024-02-01', '2024-03-01', '2024-04-01',
    '2024-05-01', '2024-06-01', '2024-07-01', '2024-08-01',
    '2024-09-01', '2024-10-01', '2024-11-01', '2024-12-01'
);

-- Partition scheme
CREATE PARTITION SCHEME PS_OrdersByDate
AS PARTITION PF_OrdersByDate
TO (
    [PRIMARY], [PRIMARY], [PRIMARY], [PRIMARY],
    [PRIMARY], [PRIMARY], [PRIMARY], [PRIMARY],
    [PRIMARY], [PRIMARY], [PRIMARY], [PRIMARY]
);

-- Partitioned table
CREATE TABLE Orders (
    OrderId INT,
    UserId INT,
    OrderDate DATETIME,
    TotalAmount DECIMAL(10,2)
) ON PS_OrdersByDate(OrderDate);

-- Create indexes on partitioned table
CREATE CLUSTERED INDEX IX_Orders_OrderDate
ON Orders(OrderDate)
ON PS_OrdersByDate(OrderDate);
```

#### Vertical Partitioning
```sql
-- Main table with frequently accessed columns
CREATE TABLE Users (
    UserId INT PRIMARY KEY,
    Username VARCHAR(50),
    Email VARCHAR(100),
    IsActive BIT,
    CreatedAt DATETIME
);

-- Extended table with less frequently accessed columns
CREATE TABLE UserProfiles (
    UserId INT PRIMARY KEY,
    FirstName VARCHAR(50),
    LastName VARCHAR(50),
    DateOfBirth DATE,
    PhoneNumber VARCHAR(20),
    Address TEXT,
    Bio TEXT,
    ProfilePictureUrl VARCHAR(500),
    FOREIGN KEY (UserId) REFERENCES Users(UserId)
);

-- View to combine both tables
CREATE VIEW vw_CompleteUserProfile
AS
SELECT 
    u.UserId,
    u.Username,
    u.Email,
    u.IsActive,
    u.CreatedAt,
    up.FirstName,
    up.LastName,
    up.DateOfBirth,
    up.PhoneNumber,
    up.Address,
    up.Bio,
    up.ProfilePictureUrl
FROM Users u
LEFT JOIN UserProfiles up ON u.UserId = up.UserId;
```

## Advanced Database Patterns

### 1. Audit Trail Pattern
Veri değişikliklerini takip etmek için kullanılan pattern.

```sql
-- Audit table
CREATE TABLE AuditLogs (
    AuditId BIGINT IDENTITY(1,1) PRIMARY KEY,
    TableName VARCHAR(100),
    PrimaryKeyValue VARCHAR(100),
    ColumnName VARCHAR(100),
    OldValue NVARCHAR(MAX),
    NewValue NVARCHAR(MAX),
    ChangeType VARCHAR(10), -- INSERT, UPDATE, DELETE
    ChangedBy VARCHAR(100),
    ChangedAt DATETIME DEFAULT GETDATE()
);

-- Trigger for audit logging
CREATE TRIGGER TR_Users_Audit
ON Users
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Handle INSERT
    IF EXISTS(SELECT * FROM INSERTED) AND NOT EXISTS(SELECT * FROM DELETED)
    BEGIN
        INSERT INTO AuditLogs (TableName, PrimaryKeyValue, ColumnName, NewValue, ChangeType, ChangedBy)
        SELECT 
            'Users',
            CAST(i.UserId AS VARCHAR(100)),
            'ALL',
            'INSERTED',
            'INSERT',
            SYSTEM_USER
        FROM INSERTED i;
    END
    
    -- Handle UPDATE
    IF EXISTS(SELECT * FROM INSERTED) AND EXISTS(SELECT * FROM DELETED)
    BEGIN
        INSERT INTO AuditLogs (TableName, PrimaryKeyValue, ColumnName, OldValue, NewValue, ChangeType, ChangedBy)
        SELECT 
            'Users',
            CAST(i.UserId AS VARCHAR(100)),
            'Username',
            d.Username,
            i.Username,
            'UPDATE',
            SYSTEM_USER
        FROM INSERTED i
        JOIN DELETED d ON i.UserId = d.UserId
        WHERE i.Username <> d.Username;
        
        -- Add more columns as needed
    END
    
    -- Handle DELETE
    IF EXISTS(SELECT * FROM DELETED) AND NOT EXISTS(SELECT * FROM INSERTED)
    BEGIN
        INSERT INTO AuditLogs (TableName, PrimaryKeyValue, ColumnName, OldValue, ChangeType, ChangedBy)
        SELECT 
            'Users',
            CAST(d.UserId AS VARCHAR(100)),
            'ALL',
            'DELETED',
            'DELETE',
            SYSTEM_USER
        FROM DELETED d;
    END
END;
```

### 2. Soft Delete Pattern
Veriyi fiziksel olarak silmek yerine işaretleme.

```sql
-- Soft delete implementation
CREATE TABLE Users (
    UserId INT PRIMARY KEY,
    Username VARCHAR(50),
    Email VARCHAR(100),
    IsActive BIT DEFAULT 1,
    IsDeleted BIT DEFAULT 0,
    DeletedAt DATETIME NULL,
    DeletedBy VARCHAR(100) NULL,
    CreatedAt DATETIME DEFAULT GETDATE(),
    UpdatedAt DATETIME DEFAULT GETDATE()
);

-- Soft delete procedure
CREATE PROCEDURE SoftDeleteUser
    @UserId INT,
    @DeletedBy VARCHAR(100)
AS
BEGIN
    UPDATE Users
    SET 
        IsDeleted = 1,
        DeletedAt = GETDATE(),
        DeletedBy = @DeletedBy,
        UpdatedAt = GETDATE()
    WHERE UserId = @UserId;
END;

-- View to exclude deleted records
CREATE VIEW vw_ActiveUsers
AS
SELECT UserId, Username, Email, IsActive, CreatedAt, UpdatedAt
FROM Users
WHERE IsDeleted = 0;

-- Index for soft delete queries
CREATE INDEX IX_Users_IsDeleted_IsActive
ON Users(IsDeleted, IsActive)
WHERE IsDeleted = 0;
```

### 3. Polymorphic Association Pattern
Farklı entity türleri ile ilişki kurma.

```sql
-- Polymorphic association table
CREATE TABLE Comments (
    CommentId INT PRIMARY KEY,
    Content TEXT,
    CreatedBy INT,
    CreatedAt DATETIME DEFAULT GETDATE()
);

CREATE TABLE CommentableEntities (
    CommentId INT,
    EntityType VARCHAR(50), -- 'Post', 'Product', 'Review'
    EntityId INT,
    PRIMARY KEY (CommentId, EntityType, EntityId),
    FOREIGN KEY (CommentId) REFERENCES Comments(CommentId)
);

-- Alternative approach with separate tables
CREATE TABLE PostComments (
    CommentId INT PRIMARY KEY,
    PostId INT,
    Content TEXT,
    CreatedBy INT,
    CreatedAt DATETIME DEFAULT GETDATE(),
    FOREIGN KEY (PostId) REFERENCES Posts(PostId)
);

CREATE TABLE ProductComments (
    CommentId INT PRIMARY KEY,
    ProductId INT,
    Content TEXT,
    CreatedBy INT,
    CreatedAt DATETIME DEFAULT GETDATE(),
    FOREIGN KEY (ProductId) REFERENCES Products(ProductId)
);

-- View to combine all comments
CREATE VIEW vw_AllComments
AS
SELECT 
    'Post' as EntityType,
    pc.PostId as EntityId,
    pc.CommentId,
    pc.Content,
    pc.CreatedBy,
    pc.CreatedAt
FROM PostComments pc

UNION ALL

SELECT 
    'Product' as EntityType,
    pc.ProductId as EntityId,
    pc.CommentId,
    pc.Content,
    pc.CreatedBy,
    pc.CreatedAt
FROM ProductComments pc;
```

## Performance Optimization Patterns

### 1. Indexing Strategies
```sql
-- Composite index for common queries
CREATE INDEX IX_Orders_UserId_OrderDate
ON Orders(UserId, OrderDate)
INCLUDE (TotalAmount);

-- Covering index
CREATE INDEX IX_Users_Email_Covering
ON Users(Email)
INCLUDE (UserId, Username, IsActive);

-- Filtered index for active users only
CREATE INDEX IX_Users_Username_Active
ON Users(Username)
WHERE IsActive = 1 AND IsDeleted = 0;

-- Partial index for recent orders
CREATE INDEX IX_Orders_Recent_UserId
ON Orders(UserId)
WHERE OrderDate >= DATEADD(YEAR, -1, GETDATE());
```

### 2. Query Optimization Patterns
```sql
-- Common Table Expression (CTE) for complex queries
WITH UserOrderStats AS (
    SELECT 
        UserId,
        COUNT(*) as OrderCount,
        SUM(TotalAmount) as TotalSpent,
        AVG(TotalAmount) as AverageOrderValue
    FROM Orders
    WHERE OrderDate >= DATEADD(YEAR, -1, GETDATE())
    GROUP BY UserId
),
UserCategoryPreferences AS (
    SELECT 
        o.UserId,
        c.CategoryName,
        COUNT(*) as PurchaseCount
    FROM Orders o
    JOIN OrderItems oi ON o.OrderId = oi.OrderId
    JOIN Products p ON oi.ProductId = p.ProductId
    JOIN Categories c ON p.CategoryId = c.CategoryId
    GROUP BY o.UserId, c.CategoryName
)
SELECT 
    u.Username,
    uos.OrderCount,
    uos.TotalSpent,
    uos.AverageOrderValue,
    ucp.CategoryName as FavoriteCategory
FROM Users u
JOIN UserOrderStats uos ON u.UserId = uos.UserId
JOIN UserCategoryPreferences ucp ON u.UserId = ucp.UserId
WHERE ucp.PurchaseCount = (
    SELECT MAX(PurchaseCount)
    FROM UserCategoryPreferences ucp2
    WHERE ucp2.UserId = u.UserId
);
```

## Mülakat Soruları

### Temel Sorular

1. **Database normalization nedir ve neden önemlidir?**
   - **Cevap**: Veri tekrarını önleme ve veri bütünlüğünü sağlama süreci. 1NF, 2NF, 3NF kuralları ile veri organize edilir.

2. **Denormalization nedir ve ne zaman kullanılır?**
   - **Cevap**: Performance için bilinçli olarak normalization kurallarını ihlal etme. Read-heavy workloads ve reporting için kullanılır.

3. **Horizontal vs Vertical partitioning arasındaki fark nedir?**
   - **Cevap**: Horizontal partitioning satırları böler (sharding), vertical partitioning sütunları böler. Farklı performance ihtiyaçları için kullanılır.

4. **Audit trail pattern nedir?**
   - **Cevap**: Veri değişikliklerini takip etme pattern'i. Triggers ile otomatik logging, compliance ve debugging için kullanılır.

5. **Soft delete pattern nedir?**
   - **Cevap**: Veriyi fiziksel olarak silmek yerine işaretleme. Data recovery ve audit trail için önemli.

### Teknik Sorular

1. **Partitioning'de partition function ve scheme nasıl çalışır?**
   - **Cevap**: Partition function veriyi nasıl böleceğini belirler, partition scheme hangi filegroup'lara gideceğini belirler.

2. **Materialized view nedir ve ne zaman kullanılır?**
   - **Cevap**: Pre-computed sonuçları saklayan view. Complex aggregations ve reporting için performance artışı sağlar.

3. **Polymorphic association'da hangi yaklaşımlar vardır?**
   - **Cevap**: Single table with type column, separate tables per entity, junction table with entity type. Trade-offs vardır.

4. **Indexing strategies'de covering index nedir?**
   - **Cevap**: Query'deki tüm sütunları içeren index. Table lookup'ı önler, performance artırır.

5. **Database design'da performance vs normalization trade-off nasıl yapılır?**
   - **Cevap**: Business requirements, query patterns, data volume analiz edilir. Balanced approach gerekli.

## Best Practices

1. **Design Principles**
   - Normalization kurallarını takip edin
   - Performance için denormalize edin
   - Indexing strategy belirleyin
   - Partitioning planlayın

2. **Performance Optimization**
   - Query patterns analiz edin
   - Appropriate indexes oluşturun
   - Partitioning uygulayın
   - Monitoring yapın

3. **Maintainability**
   - Consistent naming conventions
   - Documentation yazın
   - Change management planlayın
   - Testing yapın

4. **Scalability**
   - Future growth planlayın
   - Partitioning strategy belirleyin
   - Sharding architecture tasarlayın
   - Performance metrics izleyin

## Kaynaklar

- [Database Design Patterns](https://martinfowler.com/eaaCatalog/)
- [SQL Server Partitioning](https://docs.microsoft.com/en-us/sql/relational-databases/partitions/partitioned-tables-and-indexes)
- [Database Normalization](https://en.wikipedia.org/wiki/Database_normalization)
- [Performance Tuning](https://docs.microsoft.com/en-us/sql/relational-databases/performance/performance-tuning)
- [Database Design Best Practices](https://www.red-gate.com/simple-talk/sql/database-administration/database-design-best-practices/)
