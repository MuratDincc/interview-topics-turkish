# Entity Framework - Value Objects

## Giriş

Entity Framework'te Value Objects (Değer Nesneleri), kimlikleri olmayan ve değişmez (immutable) olan nesnelerdir. Mid-level geliştiriciler için bu kavramın anlaşılması ve etkin kullanımı önemlidir.

## Value Objects'ın Önemi

1. **Domain Model**
   - Zengin domain model
   - Daha iyi encapsulation
   - Daha iyi validasyon
   - Daha iyi business logic

2. **Veri Bütünlüğü**
   - Immutability
   - Validation
   - Consistency
   - Integrity

3. **Bakım**
   - Daha az kod tekrarı
   - Daha kolay test edilebilirlik
   - Daha iyi okunabilirlik
   - Daha kolay bakım

## Value Objects Özellikleri

1. **Temel Value Object**
```csharp
public class Address : ValueObject
{
    public string Street { get; }
    public string City { get; }
    public string State { get; }
    public string ZipCode { get; }

    public Address(string street, string city, string state, string zipCode)
    {
        Street = street;
        City = city;
        State = state;
        ZipCode = zipCode;
    }

    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return Street;
        yield return City;
        yield return State;
        yield return ZipCode;
    }
}
```

2. **Money Value Object**
```csharp
public class Money : ValueObject
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        if (amount < 0)
            throw new ArgumentException("Amount cannot be negative");
        
        if (string.IsNullOrEmpty(currency))
            throw new ArgumentException("Currency cannot be empty");

        Amount = amount;
        Currency = currency;
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot add money with different currencies");

        return new Money(Amount + other.Amount, Currency);
    }

    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return Amount;
        yield return Currency;
    }
}
```

3. **Complex Value Object**
```csharp
public class FullName : ValueObject
{
    public string FirstName { get; }
    public string LastName { get; }

    public FullName(string firstName, string lastName)
    {
        if (string.IsNullOrEmpty(firstName))
            throw new ArgumentException("First name cannot be empty");
        
        if (string.IsNullOrEmpty(lastName))
            throw new ArgumentException("Last name cannot be empty");

        FirstName = firstName;
        LastName = lastName;
    }

    public string GetFullName() => $"{FirstName} {LastName}";

    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return FirstName;
        yield return LastName;
    }
}
```

## Value Objects Kullanımı

1. **Entity İçinde Kullanım**
```csharp
public class Customer : Entity
{
    public FullName Name { get; private set; }
    public Address Address { get; private set; }
    public Money Balance { get; private set; }

    public Customer(FullName name, Address address, Money balance)
    {
        Name = name;
        Address = address;
        Balance = balance;
    }

    public void UpdateAddress(Address newAddress)
    {
        Address = newAddress;
    }

    public void AddMoney(Money amount)
    {
        Balance = Balance.Add(amount);
    }
}
```

2. **DbContext Konfigürasyonu**
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Customer>(entity =>
    {
        entity.OwnsOne(c => c.Name, name =>
        {
            name.Property(n => n.FirstName).HasColumnName("FirstName");
            name.Property(n => n.LastName).HasColumnName("LastName");
        });

        entity.OwnsOne(c => c.Address, address =>
        {
            address.Property(a => a.Street).HasColumnName("Street");
            address.Property(a => a.City).HasColumnName("City");
            address.Property(a => a.State).HasColumnName("State");
            address.Property(a => a.ZipCode).HasColumnName("ZipCode");
        });

        entity.OwnsOne(c => c.Balance, money =>
        {
            money.Property(m => m.Amount).HasColumnName("BalanceAmount");
            money.Property(m => m.Currency).HasColumnName("BalanceCurrency");
        });
    });
}
```

3. **Value Object Dönüşümleri**
```csharp
// Value Object to String
public static class AddressExtensions
{
    public static string ToString(this Address address)
    {
        return $"{address.Street}, {address.City}, {address.State} {address.ZipCode}";
    }
}

// String to Value Object
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

        return new Address(
            parts[0].Trim(),
            parts[1].Trim(),
            cityStateZip[0],
            cityStateZip[2]);
    }
}
```

## Best Practices

1. **Value Object Tasarımı**
   - Immutability
   - Validation
   - Equality
   - Business logic

2. **Güvenlik**
   - Input validation
   - Data integrity
   - Access control
   - Audit logging

3. **Performans**
   - Memory usage
   - Equality comparison
   - Serialization
   - Caching

4. **Bakım**
   - Code organization
   - Documentation
   - Testing
   - Versioning

## Mülakat Soruları

### Temel Sorular

1. **Entity Framework'te Value Object nedir?**
   - **Cevap**: Value Object, kimliği olmayan ve değişmez (immutable) olan bir nesnedir.

2. **Entity Framework'te Value Object ve Entity arasındaki fark nedir?**
   - **Cevap**: Entity'lerin kimlikleri vardır ve değişebilir, Value Object'lerin kimlikleri yoktur ve değişmezdir.

3. **Entity Framework'te Value Object nasıl konfigüre edilir?**
   - **Cevap**: DbContext.OnModelCreating metodunda OwnsOne metodu kullanılarak konfigüre edilir.

4. **Entity Framework'te Value Object ne zaman kullanılır?**
   - **Cevap**: Değerlerin bir bütün olarak ele alınması gerektiğinde ve değişmezlik gerektiğinde kullanılır.

5. **Entity Framework'te Value Object performansı nasıl etkiler?**
   - **Cevap**: Memory kullanımını artırabilir ancak domain modeli zenginleştirir ve kod kalitesini artırır.

### Teknik Sorular

1. **Temel Value Object nasıl oluşturulur?**
   - **Cevap**:
```csharp
public class Address : ValueObject
{
    public string Street { get; }
    public string City { get; }
    public string State { get; }
    public string ZipCode { get; }

    public Address(string street, string city, string state, string zipCode)
    {
        Street = street;
        City = city;
        State = state;
        ZipCode = zipCode;
    }

    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return Street;
        yield return City;
        yield return State;
        yield return ZipCode;
    }
}
```

2. **Money Value Object nasıl oluşturulur?**
   - **Cevap**:
```csharp
public class Money : ValueObject
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        if (amount < 0)
            throw new ArgumentException("Amount cannot be negative");
        
        if (string.IsNullOrEmpty(currency))
            throw new ArgumentException("Currency cannot be empty");

        Amount = amount;
        Currency = currency;
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot add money with different currencies");

        return new Money(Amount + other.Amount, Currency);
    }

    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return Amount;
        yield return Currency;
    }
}
```

3. **Value Object DbContext'te nasıl konfigüre edilir?**
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

4. **Value Object equality nasıl sağlanır?**
   - **Cevap**:
```csharp
protected override IEnumerable<object> GetEqualityComponents()
{
    yield return Street;
    yield return City;
    yield return State;
    yield return ZipCode;
}
```

5. **Value Object validation nasıl yapılır?**
   - **Cevap**:
```csharp
public Address(string street, string city, string state, string zipCode)
{
    if (string.IsNullOrEmpty(street))
        throw new ArgumentException("Street cannot be empty");
    
    if (string.IsNullOrEmpty(city))
        throw new ArgumentException("City cannot be empty");
    
    if (string.IsNullOrEmpty(state))
        throw new ArgumentException("State cannot be empty");
    
    if (string.IsNullOrEmpty(zipCode))
        throw new ArgumentException("ZipCode cannot be empty");

    Street = street;
    City = city;
    State = state;
    ZipCode = zipCode;
}
```

### İleri Seviye Sorular

1. **Entity Framework'te Value Object performansı nasıl optimize edilir?**
   - **Cevap**:
     - Memory kullanımı optimizasyonu
     - Equality comparison optimizasyonu
     - Serialization optimizasyonu
     - Caching stratejileri
     - Lazy loading

2. **Entity Framework'te distributed sistemlerde Value Object nasıl yönetilir?**
   - **Cevap**:
     - Serialization
     - Versioning
     - Migration
     - Consistency
     - Conflict resolution

3. **Entity Framework'te high concurrency senaryolarında Value Object nasıl yönetilir?**
   - **Cevap**:
     - Immutability
     - Thread safety
     - Atomic operations
     - Versioning
     - Conflict resolution

4. **Entity Framework'te Value Object monitoring ve profiling nasıl yapılır?**
   - **Cevap**:
     - Memory profiling
     - Performance metrics
     - Resource monitoring
     - Profiling tools
     - Health checks

5. **Entity Framework'te custom Value Object stratejileri nasıl geliştirilir?**
   - **Cevap**:
     - Custom equality
     - Custom validation
     - Custom serialization
     - Custom conversion
     - Custom optimization 