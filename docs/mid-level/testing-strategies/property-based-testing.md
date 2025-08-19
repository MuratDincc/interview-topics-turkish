# Property-Based Testing

## Giriş

Property-Based Testing, test input'larını otomatik olarak generate eden ve belirli property'lerin her zaman doğru olduğunu kanıtlayan testing yaklaşımıdır. Mid-level geliştiriciler için property-based testing'i anlamak, comprehensive testing, edge case discovery ve test automation için kritik öneme sahiptir. Bu dosya, FsCheck, property definition, data generation ve shrinking konularını kapsar.

## FsCheck Implementation

### 1. Basic Property Testing
Temel property testing implementasyonu.

```csharp
public class CalculatorPropertyTests
{
    [Property]
    public Property Addition_IsCommutative(int a, int b)
    {
        var result1 = Calculator.Add(a, b);
        var result2 = Calculator.Add(b, a);
        
        return (result1 == result2).ToProperty()
            .Label($"Addition is commutative: {a} + {b} = {b} + {a}");
    }
    
    [Property]
    public Property Addition_IsAssociative(int a, int b, int c)
    {
        var result1 = Calculator.Add(Calculator.Add(a, b), c);
        var result2 = Calculator.Add(a, Calculator.Add(b, c));
        
        return (result1 == result2).ToProperty()
            .Label($"Addition is associative: ({a} + {b}) + {c} = {a} + ({b} + {c})");
    }
    
    [Property]
    public Property Addition_WithZero_ReturnsSameNumber(int a)
    {
        var result = Calculator.Add(a, 0);
        
        return (result == a).ToProperty()
            .Label($"Adding zero returns the same number: {a} + 0 = {a}");
    }
    
    [Property]
    public Property Addition_IsDistributiveOverMultiplication(int a, int b, int c)
    {
        var result1 = Calculator.Multiply(a, Calculator.Add(b, c));
        var result2 = Calculator.Add(Calculator.Multiply(a, b), Calculator.Multiply(a, c));
        
        return (result1 == result2).ToProperty()
            .Label($"Multiplication distributes over addition: {a} * ({b} + {c}) = {a} * {b} + {a} * {c}");
    }
    
    [Property]
    public Property Division_ByNonZero_IsInverseOfMultiplication(int a, int b)
    {
        // Ensure b is not zero
        var nonZeroB = b == 0 ? 1 : b;
        
        var result = Calculator.Divide(Calculator.Multiply(a, nonZeroB), nonZeroB);
        
        return (result == a).ToProperty()
            .Label($"Division by non-zero is inverse of multiplication: ({a} * {nonZeroB}) / {nonZeroB} = {a}");
    }
    
    [Property]
    public Property Square_IsAlwaysPositive(int a)
    {
        var result = Calculator.Square(a);
        
        return (result >= 0).ToProperty()
            .Label($"Square is always non-negative: {a}² = {result} >= 0");
    }
    
    [Property]
    public Property Square_IsEven_WhenInputIsEven(int a)
    {
        // Generate only even numbers
        var evenA = a * 2;
        var result = Calculator.Square(evenA);
        
        return (result % 2 == 0).ToProperty()
            .Label($"Square of even number is even: {evenA}² = {result} is even");
    }
}

public static class Calculator
{
    public static int Add(int a, int b) => a + b;
    public static int Multiply(int a, int b) => a * b;
    public static int Divide(int a, int b) => b != 0 ? a / b : throw new DivideByZeroException();
    public static int Square(int a) => a * a;
}
```

### 2. Custom Data Generators
Özel veri generator'ları implementasyonu.

```csharp
public class CustomGenerators
{
    public static Arbitrary<EmailAddress> EmailAddressGenerator()
    {
        var localPart = Gen.Choose(1, 10).SelectMany(length => 
            Gen.Choose(0, 25).SelectMany(seed => 
                Gen.Elements("abcdefghijklmnopqrstuvwxyz0123456789._%+-")
                    .ArrayOf(length)
                    .Select(chars => new string(chars))));
        
        var domain = Gen.Choose(1, 20).SelectMany(length => 
            Gen.Choose(0, 25).SelectMany(seed => 
                Gen.Elements("abcdefghijklmnopqrstuvwxyz0123456789.-")
                    .ArrayOf(length)
                    .Select(chars => new string(chars))));
        
        var tld = Gen.Elements("com", "org", "net", "edu", "gov");
        
        return Gen.Three(localPart, domain, tld)
            .Select(tuple => new EmailAddress($"{tuple.Item1}@{tuple.Item2}.{tuple.Item3}"))
            .ToArbitrary();
    }
    
    public static Arbitrary<PhoneNumber> PhoneNumberGenerator()
    {
        var countryCode = Gen.Elements("+1", "+44", "+49", "+33", "+81");
        var areaCode = Gen.Choose(100, 999);
        var prefix = Gen.Choose(100, 999);
        var lineNumber = Gen.Choose(1000, 9999);
        
        return Gen.Three(countryCode, areaCode, Gen.Two(prefix, lineNumber))
            .Select(tuple => new PhoneNumber($"{tuple.Item1} {tuple.Item2} {tuple.Item3.Item1}-{tuple.Item3.Item2}"))
            .ToArbitrary();
    }
    
    public static Arbitrary<CreditCardNumber> CreditCardNumberGenerator()
    {
        var cardTypes = new[] { "Visa", "MasterCard", "American Express", "Discover" };
        var cardType = Gen.Elements(cardTypes);
        
        var cardNumber = Gen.Choose(0, 9).ArrayOf(16)
            .Select(digits => string.Join("", digits));
        
        return Gen.Two(cardType, cardNumber)
            .Select(tuple => new CreditCardNumber(tuple.Item1, tuple.Item2))
            .ToArbitrary();
    }
    
    public static Arbitrary<DateRange> DateRangeGenerator()
    {
        var startDate = Gen.Choose(DateTime.MinValue.Ticks, DateTime.MaxValue.Ticks)
            .Select(ticks => new DateTime(ticks));
        
        var endDate = startDate.SelectMany(start => 
            Gen.Choose(start.Ticks, start.Ticks + TimeSpan.FromDays(365).Ticks)
                .Select(ticks => new DateTime(ticks)));
        
        return Gen.Two(startDate, endDate)
            .Select(tuple => new DateRange(tuple.Item1, tuple.Item2))
            .Where(range => range.StartDate < range.EndDate)
            .ToArbitrary();
    }
    
    public static Arbitrary<ComplexNumber> ComplexNumberGenerator()
    {
        var realPart = Gen.Choose(-1000.0, 1000.0);
        var imaginaryPart = Gen.Choose(-1000.0, 1000.0);
        
        return Gen.Two(realPart, imaginaryPart)
            .Select(tuple => new ComplexNumber(tuple.Item1, tuple.Item2))
            .ToArbitrary();
    }
}

public class EmailAddress
{
    public string Value { get; }
    
    public EmailAddress(string value)
    {
        Value = value;
    }
    
    public override string ToString() => Value;
}

public class PhoneNumber
{
    public string Value { get; }
    
    public PhoneNumber(string value)
    {
        Value = value;
    }
    
    public override string ToString() => Value;
}

public class CreditCardNumber
{
    public string Type { get; }
    public string Number { get; }
    
    public CreditCardNumber(string type, string number)
    {
        Type = type;
        Number = number;
    }
    
    public override string ToString() => $"{Type}: {Number}";
}

public class DateRange
{
    public DateTime StartDate { get; }
    public DateTime EndDate { get; }
    
    public DateRange(DateTime startDate, DateTime endDate)
    {
        StartDate = startDate;
        EndDate = endDate;
    }
    
    public TimeSpan Duration => EndDate - StartDate;
    
    public override string ToString() => $"{StartDate:yyyy-MM-dd} to {EndDate:yyyy-MM-dd}";
}

public class ComplexNumber
{
    public double Real { get; }
    public double Imaginary { get; }
    
    public ComplexNumber(double real, double imaginary)
    {
        Real = real;
        Imaginary = imaginary;
    }
    
    public double Magnitude => Math.Sqrt(Real * Real + Imaginary * Imaginary);
    
    public override string ToString() => $"{Real} + {Imaginary}i";
}
```

### 3. Advanced Property Testing
Gelişmiş property testing implementasyonu.

```csharp
public class AdvancedPropertyTests
{
    [Property]
    public Property List_Reverse_IsIdempotent(List<int> list)
    {
        var reversedOnce = list.Reverse().ToList();
        var reversedTwice = reversedOnce.Reverse().ToList();
        
        return list.SequenceEqual(reversedTwice).ToProperty()
            .Label($"Reverse is idempotent: Reverse(Reverse({list})) = {list}");
    }
    
    [Property]
    public Property List_Sort_IsIdempotent(List<int> list)
    {
        var sortedOnce = list.OrderBy(x => x).ToList();
        var sortedTwice = sortedOnce.OrderBy(x => x).ToList();
        
        return sortedOnce.SequenceEqual(sortedTwice).ToProperty()
            .Label($"Sort is idempotent: Sort(Sort({list})) = Sort({list})");
    }
    
    [Property]
    public Property List_Concat_IsAssociative(List<int> list1, List<int> list2, List<int> list3)
    {
        var result1 = list1.Concat(list2).Concat(list3).ToList();
        var result2 = list1.Concat(list2.Concat(list3)).ToList();
        
        return result1.SequenceEqual(result2).ToProperty()
            .Label($"List concatenation is associative");
    }
    
    [Property]
    public Property String_Length_IsAlwaysNonNegative(string input)
    {
        return (input.Length >= 0).ToProperty()
            .Label($"String length is always non-negative: '{input}'.Length = {input.Length}");
    }
    
    [Property]
    public Property String_Substring_Length_IsValid(string input, int startIndex, int length)
    {
        // Ensure valid parameters
        if (startIndex < 0 || startIndex >= input.Length || length < 0 || startIndex + length > input.Length)
        {
            return true.ToProperty().Label("Invalid parameters, skipping test");
        }
        
        var substring = input.Substring(startIndex, length);
        
        return (substring.Length == length).ToProperty()
            .Label($"Substring length is correct: '{input}'.Substring({startIndex}, {length}) = '{substring}'");
    }
    
    [Property]
    public Property Dictionary_Add_Then_Get_ReturnsSameValue(int key, string value)
    {
        var dict = new Dictionary<int, string>();
        dict.Add(key, value);
        
        var retrievedValue = dict[key];
        
        return (retrievedValue == value).ToProperty()
            .Label($"Dictionary add then get returns same value: [{key}] = '{value}'");
    }
    
    [Property]
    public Property Dictionary_Remove_Then_ContainsKey_ReturnsFalse(int key, string value)
    {
        var dict = new Dictionary<int, string>();
        dict.Add(key, value);
        dict.Remove(key);
        
        return (!dict.ContainsKey(key)).ToProperty()
            .Label($"Dictionary remove then contains key returns false: [{key}] removed");
    }
    
    [Property]
    public Property Queue_Enqueue_Then_Dequeue_ReturnsSameValue(int value)
    {
        var queue = new Queue<int>();
        queue.Enqueue(value);
        
        var dequeuedValue = queue.Dequeue();
        
        return (dequeuedValue == value).ToProperty()
            .Label($"Queue enqueue then dequeue returns same value: {value}");
    }
    
    [Property]
    public Property Stack_Push_Then_Pop_ReturnsSameValue(int value)
    {
        var stack = new Stack<int>();
        stack.Push(value);
        
        var poppedValue = stack.Pop();
        
        return (poppedValue == value).ToProperty()
            .Label($"Stack push then pop returns same value: {value}");
    }
    
    [Property]
    public Property HashSet_Add_Then_Contains_ReturnsTrue(int value)
    {
        var hashSet = new HashSet<int>();
        hashSet.Add(value);
        
        return hashSet.Contains(value).ToProperty()
            .Label($"HashSet add then contains returns true: {value}");
    }
    
    [Property]
    public Property HashSet_Remove_Then_Contains_ReturnsFalse(int value)
    {
        var hashSet = new HashSet<int>();
        hashSet.Add(value);
        hashSet.Remove(value);
        
        return (!hashSet.Contains(value)).ToProperty()
            .Label($"HashSet remove then contains returns false: {value}");
    }
}

public class BusinessLogicPropertyTests
{
    [Property]
    public Property UserValidation_ValidUser_AlwaysPasses()
    {
        var validUser = new User
        {
            Username = "validuser",
            Email = "user@example.com",
            Age = 25,
            Password = "SecurePass123!"
        };
        
        var validator = new UserValidator();
        var result = validator.Validate(validUser);
        
        return result.IsValid.ToProperty()
            .Label($"Valid user always passes validation: {validUser.Username}");
    }
    
    [Property]
    public Property UserValidation_InvalidAge_AlwaysFails()
    {
        var invalidUser = new User
        {
            Username = "testuser",
            Email = "test@example.com",
            Age = -5, // Invalid age
            Password = "Password123!"
        };
        
        var validator = new UserValidator();
        var result = validator.Validate(invalidUser);
        
        return (!result.IsValid).ToProperty()
            .Label($"User with invalid age always fails validation: Age = {invalidUser.Age}");
    }
    
    [Property]
    public Property UserValidation_EmptyUsername_AlwaysFails()
    {
        var invalidUser = new User
        {
            Username = "", // Empty username
            Email = "test@example.com",
            Age = 25,
            Password = "Password123!"
        };
        
        var validator = new UserValidator();
        var result = validator.Validate(invalidUser);
        
        return (!result.IsValid).ToProperty()
            .Label($"User with empty username always fails validation");
    }
    
    [Property]
    public Property UserValidation_InvalidEmail_AlwaysFails()
    {
        var invalidUser = new User
        {
            Username = "testuser",
            Email = "invalid-email", // Invalid email
            Age = 25,
            Password = "Password123!"
        };
        
        var validator = new UserValidator();
        var result = validator.Validate(invalidUser);
        
        return (!result.IsValid).ToProperty()
            .Label($"User with invalid email always fails validation: {invalidUser.Email}");
    }
}

public class User
{
    public string Username { get; set; }
    public string Email { get; set; }
    public int Age { get; set; }
    public string Password { get; set; }
}

public class UserValidator
{
    public ValidationResult Validate(User user)
    {
        var errors = new List<string>();
        
        if (string.IsNullOrWhiteSpace(user.Username))
            errors.Add("Username is required");
        
        if (string.IsNullOrWhiteSpace(user.Email) || !IsValidEmail(user.Email))
            errors.Add("Valid email is required");
        
        if (user.Age < 0 || user.Age > 150)
            errors.Add("Age must be between 0 and 150");
        
        if (string.IsNullOrWhiteSpace(user.Password) || user.Password.Length < 8)
            errors.Add("Password must be at least 8 characters long");
        
        return new ValidationResult
        {
            IsValid = !errors.Any(),
            Errors = errors
        };
    }
    
    private bool IsValidEmail(string email)
    {
        try
        {
            var addr = new System.Net.Mail.MailAddress(email);
            return addr.Address == email;
        }
        catch
        {
            return false;
        }
    }
}

public class ValidationResult
{
    public bool IsValid { get; set; }
    public List<string> Errors { get; set; } = new();
}
```

## Mülakat Soruları

### Temel Sorular

1. **Property-Based Testing nedir?**
   - **Cevap**: Test input'larını otomatik generate eden ve property'leri doğrulayan testing yaklaşımı.

2. **FsCheck nedir?**
   - **Cevap**: .NET için property-based testing framework'ü.

3. **Property nedir?**
   - **Cevap**: Test edilen kodun her zaman doğru olması gereken özellik.

4. **Shrinking nedir?**
   - **Cevap**: Test failure'ları için minimal counterexample bulma süreci.

5. **Property-based testing ne zaman kullanılır?**
   - **Cevap**: Mathematical properties, data structure invariants, business rules.

### Teknik Sorular

1. **Custom generator nasıl implement edilir?**
   - **Cevap**: Arbitrary<T> class, Gen<T> combinators, ToArbitrary() extension.

2. **Property validation nasıl yapılır?**
   - **Cevap**: ToProperty() extension, Label() method, conditional properties.

3. **Data shrinking nasıl çalışır?**
   - **Cevap**: Automatic shrinking, custom shrinkers, minimal counterexamples.

4. **Property-based testing performance nasıl optimize edilir?**
   - **Cevap**: Efficient generators, property filtering, test count optimization.

5. **Property-based testing CI/CD'de nasıl kullanılır?**
   - **Cevap**: Automated testing, regression detection, continuous validation.

## Best Practices

1. **Property Design**
   - Mathematical properties tanımlayın
   - Invariants identify edin
   - Edge cases cover edin
   - Clear property names kullanın

2. **Generator Implementation**
   - Efficient generators yazın
   - Realistic data generate edin
   - Edge case data include edin
   - Custom types support edin

3. **Property Validation**
   - Comprehensive validation implement edin
   - Error messages ekleyin
   - Property labels kullanın
   - Conditional properties yazın

4. **Performance & Scalability**
   - Generator performance optimize edin
   - Test count balance edin
   - Shrinking efficiency sağlayın
   - Memory usage monitor edin

5. **Integration & Maintenance**
   - CI/CD pipeline entegre edin
   - Test results analyze edin
   - Property coverage measure edin
   - Continuous improvement sağlayın

## Kaynaklar

- [FsCheck](https://fscheck.github.io/FsCheck/)
- [Property-Based Testing](https://hypothesis.works/articles/what-is-property-based-testing/)
- [QuickCheck](https://hackage.haskell.org/package/QuickCheck)
- [Property Testing Best Practices](https://blog.jessitron.com/2013/04/25/property-based-testing-what-is-it/)
- [.NET Testing Strategies](https://docs.microsoft.com/en-us/dotnet/core/testing/)
