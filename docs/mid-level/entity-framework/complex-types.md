# Entity Framework - Complex Types

## Giriş

Entity Framework'te Complex Types (Karmaşık Tipler), bir entity'nin içinde yer alan ve kendi başına bir entity olmayan, ancak birden fazla özellik içeren yapılardır. Mid-level geliştiriciler için bu kavramın anlaşılması ve etkin kullanımı önemlidir.

## Complex Types'ın Önemi

1. **Domain Model**
   - Daha iyi organizasyon
   - Daha iyi encapsulation
   - Daha iyi okunabilirlik
   - Daha iyi bakım

2. **Veri Yapısı**
   - Daha iyi veri organizasyonu
   - Daha iyi veri bütünlüğü
   - Daha iyi veri erişimi
   - Daha iyi veri yönetimi

3. **Bakım**
   - Daha az kod tekrarı
   - Daha kolay test edilebilirlik
   - Daha iyi modülerlik
   - Daha kolay genişletilebilirlik

## Complex Types Özellikleri

1. **Temel Complex Type**
```csharp
public class Address
{
    public string Street { get; set; }
    public string City { get; set; }
    public string State { get; set; }
    public string ZipCode { get; set; }
}

public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
    public Address BillingAddress { get; set; }
    public Address ShippingAddress { get; set; }
}
```

2. **Complex Type Konfigürasyonu**
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Customer>(entity =>
    {
        entity.OwnsOne(c => c.BillingAddress, address =>
        {
            address.Property(a => a.Street).HasColumnName("BillingStreet");
            address.Property(a => a.City).HasColumnName("BillingCity");
            address.Property(a => a.State).HasColumnName("BillingState");
            address.Property(a => a.ZipCode).HasColumnName("BillingZipCode");
        });

        entity.OwnsOne(c => c.ShippingAddress, address =>
        {
            address.Property(a => a.Street).HasColumnName("ShippingStreet");
            address.Property(a => a.City).HasColumnName("ShippingCity");
            address.Property(a => a.State).HasColumnName("ShippingState");
            address.Property(a => a.ZipCode).HasColumnName("ShippingZipCode");
        });
    });
}
```

3. **Complex Type Validasyonu**
```csharp
public class Address
{
    private string _street;
    private string _city;
    private string _state;
    private string _zipCode;

    public string Street
    {
        get => _street;
        set
        {
            if (string.IsNullOrEmpty(value))
                throw new ArgumentException("Street cannot be empty");
            _street = value;
        }
    }

    public string City
    {
        get => _city;
        set
        {
            if (string.IsNullOrEmpty(value))
                throw new ArgumentException("City cannot be empty");
            _city = value;
        }
    }

    public string State
    {
        get => _state;
        set
        {
            if (string.IsNullOrEmpty(value))
                throw new ArgumentException("State cannot be empty");
            _state = value;
        }
    }

    public string ZipCode
    {
        get => _zipCode;
        set
        {
            if (string.IsNullOrEmpty(value))
                throw new ArgumentException("ZipCode cannot be empty");
            _zipCode = value;
        }
    }
}
```

## Complex Types Kullanımı

1. **Entity İçinde Kullanım**
```csharp
public class Order
{
    public int Id { get; set; }
    public DateTime OrderDate { get; set; }
    public Address ShippingAddress { get; set; }
    public Address BillingAddress { get; set; }
    public List<OrderItem> Items { get; set; }
}

public class OrderItem
{
    public int Id { get; set; }
    public int OrderId { get; set; }
    public string ProductName { get; set; }
    public decimal Price { get; set; }
    public int Quantity { get; set; }
}
```

2. **DbContext Konfigürasyonu**
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Order>(entity =>
    {
        entity.OwnsOne(o => o.ShippingAddress, address =>
        {
            address.Property(a => a.Street).HasColumnName("ShippingStreet");
            address.Property(a => a.City).HasColumnName("ShippingCity");
            address.Property(a => a.State).HasColumnName("ShippingState");
            address.Property(a => a.ZipCode).HasColumnName("ShippingZipCode");
        });

        entity.OwnsOne(o => o.BillingAddress, address =>
        {
            address.Property(a => a.Street).HasColumnName("BillingStreet");
            address.Property(a => a.City).HasColumnName("BillingCity");
            address.Property(a => a.State).HasColumnName("BillingState");
            address.Property(a => a.ZipCode).HasColumnName("BillingZipCode");
        });

        entity.HasMany(o => o.Items)
            .WithOne()
            .HasForeignKey(i => i.OrderId);
    });
}
```

3. **Complex Type Dönüşümleri**
```csharp
// Complex Type to String
public static class AddressExtensions
{
    public static string ToString(this Address address)
    {
        return $"{address.Street}, {address.City}, {address.State} {address.ZipCode}";
    }
}

// String to Complex Type
public static class AddressParser
{
    public static Address Parse(string addressString)
    {
        var parts = addressString.Split(',');
        if (parts.Length != 3)
            throw new ArgumentException("Invalid address format");

        var cityStateZip = parts[2].Trim().Split(' ');
        if (cityStateZip.Length != 3)
            throw new ArgumentException("Invalid address format");

        return new Address
        {
            Street = parts[0].Trim(),
            City = parts[1].Trim(),
            State = cityStateZip[0],
            ZipCode = cityStateZip[2]
        };
    }
}
```

## Best Practices

1. **Complex Type Tasarımı**
   - Single Responsibility
   - Immutability
   - Validation
   - Business logic

2. **Güvenlik**
   - Input validation
   - Data integrity
   - Access control
   - Audit logging

3. **Performans**
   - Memory usage
   - Query optimization
   - Lazy loading
   - Caching

4. **Bakım**
   - Code organization
   - Documentation
   - Testing
   - Versioning

## Mülakat Soruları

### Temel Sorular

1. **Entity Framework'te Complex Type nedir?**
   - **Cevap**: Complex Type, bir entity'nin içinde yer alan ve kendi başına bir entity olmayan, ancak birden fazla özellik içeren yapılardır.

2. **Entity Framework'te Complex Type ve Entity arasındaki fark nedir?**
   - **Cevap**: Entity'lerin kendi tabloları vardır ve kimlikleri vardır, Complex Type'ların kendi tabloları yoktur ve entity'lerin içinde yer alırlar.

3. **Entity Framework'te Complex Type nasıl konfigüre edilir?**
   - **Cevap**: DbContext.OnModelCreating metodunda OwnsOne metodu kullanılarak konfigüre edilir.

4. **Entity Framework'te Complex Type ne zaman kullanılır?**
   - **Cevap**: Bir entity'nin içinde yer alan ve kendi başına bir entity olmayan, ancak birden fazla özellik içeren yapılar için kullanılır.

5. **Entity Framework'te Complex Type performansı nasıl etkiler?**
   - **Cevap**: Memory kullanımını artırabilir ancak veri organizasyonunu ve kod kalitesini artırır.

### Teknik Sorular

1. **Temel Complex Type nasıl oluşturulur?**
   - **Cevap**:
```csharp
public class Address
{
    public string Street { get; set; }
    public string City { get; set; }
    public string State { get; set; }
    public string ZipCode { get; set; }
}
```

2. **Complex Type DbContext'te nasıl konfigüre edilir?**
   - **Cevap**:
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Customer>(entity =>
    {
        entity.OwnsOne(c => c.Address, address =>
        {
            address.Property(a => a.Street).HasColumnName("Street");
            address.Property(a => a.City).HasColumnName("City");
            address.Property(a => a.State).HasColumnName("State");
            address.Property(a => a.ZipCode).HasColumnName("ZipCode");
        });
    });
}
```

3. **Complex Type validasyonu nasıl yapılır?**
   - **Cevap**:
```csharp
public class Address
{
    private string _street;
    private string _city;
    private string _state;
    private string _zipCode;

    public string Street
    {
        get => _street;
        set
        {
            if (string.IsNullOrEmpty(value))
                throw new ArgumentException("Street cannot be empty");
            _street = value;
        }
    }

    // Diğer property'ler için benzer validasyonlar
}
```

4. **Complex Type equality nasıl sağlanır?**
   - **Cevap**:
```csharp
public class Address
{
    public string Street { get; set; }
    public string City { get; set; }
    public string State { get; set; }
    public string ZipCode { get; set; }

    public override bool Equals(object obj)
    {
        if (obj is Address other)
        {
            return Street == other.Street &&
                   City == other.City &&
                   State == other.State &&
                   ZipCode == other.ZipCode;
        }
        return false;
    }

    public override int GetHashCode()
    {
        return HashCode.Combine(Street, City, State, ZipCode);
    }
}
```

5. **Complex Type dönüşümleri nasıl yapılır?**
   - **Cevap**:
```csharp
public static class AddressExtensions
{
    public static string ToString(this Address address)
    {
        return $"{address.Street}, {address.City}, {address.State} {address.ZipCode}";
    }
}

public static class AddressParser
{
    public static Address Parse(string addressString)
    {
        var parts = addressString.Split(',');
        if (parts.Length != 3)
            throw new ArgumentException("Invalid address format");

        var cityStateZip = parts[2].Trim().Split(' ');
        if (cityStateZip.Length != 3)
            throw new ArgumentException("Invalid address format");

        return new Address
        {
            Street = parts[0].Trim(),
            City = parts[1].Trim(),
            State = cityStateZip[0],
            ZipCode = cityStateZip[2]
        };
    }
}
```

### İleri Seviye Sorular

1. **Entity Framework'te Complex Type performansı nasıl optimize edilir?**
   - **Cevap**:
     - Memory kullanımı optimizasyonu
     - Query optimizasyonu
     - Lazy loading
     - Caching stratejileri
     - Index kullanımı

2. **Entity Framework'te distributed sistemlerde Complex Type nasıl yönetilir?**
   - **Cevap**:
     - Serialization
     - Versioning
     - Migration
     - Consistency
     - Conflict resolution

3. **Entity Framework'te high concurrency senaryolarında Complex Type nasıl yönetilir?**
   - **Cevap**:
     - Immutability
     - Thread safety
     - Atomic operations
     - Versioning
     - Conflict resolution

4. **Entity Framework'te Complex Type monitoring ve profiling nasıl yapılır?**
   - **Cevap**:
     - Memory profiling
     - Performance metrics
     - Resource monitoring
     - Profiling tools
     - Health checks

5. **Entity Framework'te custom Complex Type stratejileri nasıl geliştirilir?**
   - **Cevap**:
     - Custom equality
     - Custom validation
     - Custom serialization
     - Custom conversion
     - Custom optimization 