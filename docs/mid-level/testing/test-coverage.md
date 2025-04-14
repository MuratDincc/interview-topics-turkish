# Test Coverage

## Giriş

Test Coverage (Test Kapsamı), yazılım kodunun ne kadarının test edildiğini ölçen bir metriktir. Bu metrik, test sürecinin etkinliğini değerlendirmek ve test edilmemiş kod bölümlerini belirlemek için kullanılır.

## Test Coverage'ın Önemi

1. **Kalite Güvence**
   - Kod kalitesini ölçme
   - Test edilmemiş kodları belirleme
   - Potansiyel hataları tespit etme

2. **Risk Yönetimi**
   - Riskli kod bölümlerini belirleme
   - Test önceliklerini belirleme
   - Hata olasılığını azaltma

3. **Süreç İyileştirme**
   - Test sürecini optimize etme
   - Test stratejisini geliştirme
   - Kaynak kullanımını iyileştirme

## Test Coverage Türleri

1. **Statement Coverage**
   - Kod satırlarının test edilme oranı
   - En temel kapsam metriği
   - Her ifadenin en az bir kez çalıştırılması

2. **Branch Coverage**
   - Koşullu ifadelerin test edilme oranı
   - Her dalın en az bir kez çalıştırılması
   - If-else, switch-case gibi yapıların kontrolü

3. **Function Coverage**
   - Fonksiyonların test edilme oranı
   - Her fonksiyonun en az bir kez çağrılması
   - Metod düzeyinde kapsam

4. **Path Coverage**
   - Kod yollarının test edilme oranı
   - En kapsamlı metrik
   - Tüm olası yolların test edilmesi

## .NET'te Test Coverage

### Coverlet Kullanımı
```csharp
// .NET CLI ile
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=opencover

// MSBuild ile
<PropertyGroup>
  <CollectCoverage>true</CollectCoverage>
  <CoverletOutputFormat>opencover</CoverletOutputFormat>
</PropertyGroup>
```

### ReportGenerator ile Raporlama
```csharp
// Rapor oluşturma
dotnet tool install -g dotnet-reportgenerator-globaltool
reportgenerator -reports:coverage.cobertura.xml -targetdir:coveragereport
```

### Örnek Test Projesi
```csharp
public class Calculator
{
    public int Add(int a, int b)
    {
        return a + b;
    }

    public int Subtract(int a, int b)
    {
        return a - b;
    }
}

public class CalculatorTests
{
    [Fact]
    public void Add_ShouldReturnCorrectSum()
    {
        // Arrange
        var calculator = new Calculator();

        // Act
        var result = calculator.Add(2, 3);

        // Assert
        Assert.Equal(5, result);
    }

    [Fact]
    public void Subtract_ShouldReturnCorrectDifference()
    {
        // Arrange
        var calculator = new Calculator();

        // Act
        var result = calculator.Subtract(5, 3);

        // Assert
        Assert.Equal(2, result);
    }
}
```

## Test Coverage Best Practices

1. **Hedef Belirleme**
   - Gerçekçi hedefler koyma
   - Kritik kodlar için yüksek hedefler
   - Legacy kod için makul hedefler

2. **Ölçüm Stratejisi**
   - Düzenli ölçüm yapma
   - Farklı coverage türlerini kullanma
   - Trend analizi yapma

3. **İyileştirme Stratejisi**
   - Düşük coverage'ı önceliklendirme
   - Test edilebilir kod yazma
   - Test otomasyonunu artırma

## Mülakat Soruları ve Cevapları

### Temel Sorular

1. **Test Coverage nedir ve neden önemlidir?**
   - **Cevap**: Test Coverage, yazılım kodunun ne kadarının test edildiğini ölçen bir metriktir. Önemi:
     - Kod kalitesini ölçer
     - Test edilmemiş kodları belirler
     - Risk yönetimini sağlar
     - Test sürecini iyileştirir
     - Kalite güvencesi sağlar

2. **Farklı Test Coverage türleri nelerdir?**
   - **Cevap**:
     - **Statement Coverage**: Kod satırlarının test edilme oranı
     - **Branch Coverage**: Koşullu ifadelerin test edilme oranı
     - **Function Coverage**: Fonksiyonların test edilme oranı
     - **Path Coverage**: Kod yollarının test edilme oranı

3. **Test Coverage hedefi ne olmalıdır?**
   - **Cevap**:
     - Kritik kodlar için %80-90
     - Genel kod için %70-80
     - Legacy kod için %50-60
     - Hedefler projeye özel belirlenmeli
     - Kalite güvencesi için yeterli olmalı

4. **Test Coverage'ın avantajları ve dezavantajları nelerdir?**
   - **Cevap**:
     - **Avantajlar**:
       - Kod kalitesini ölçer
       - Riskleri belirler
       - Test sürecini iyileştirir
     - **Dezavantajlar**:
       - Yüksek coverage kalite garantisi değil
       - Test maliyetini artırabilir
       - Yanlış güven hissi verebilir

5. **Test Coverage nasıl ölçülür?**
   - **Cevap**:
     - Coverlet gibi araçlar kullanılır
     - CI/CD pipeline'da entegre edilir
     - Raporlar oluşturulur
     - Trend analizi yapılır
     - Düzenli ölçüm yapılır

### Teknik Sorular

1. **Coverlet nasıl kullanılır?**
   - **Cevap**:
```xml
<ItemGroup>
  <PackageReference Include="coverlet.msbuild" Version="3.1.2">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
  </PackageReference>
</ItemGroup>

<PropertyGroup>
  <CollectCoverage>true</CollectCoverage>
  <CoverletOutputFormat>opencover</CoverletOutputFormat>
  <CoverletOutput>./coverage/</CoverletOutput>
</PropertyGroup>
```

2. **Test Coverage raporu nasıl oluşturulur?**
   - **Cevap**:
```bash
# Rapor oluşturma
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=opencover
reportgenerator -reports:coverage.cobertura.xml -targetdir:coveragereport
```

3. **Belirli bir sınıfın coverage'ı nasıl ölçülür?**
   - **Cevap**:
```xml
<PropertyGroup>
  <CollectCoverage>true</CollectCoverage>
  <CoverletOutputFormat>opencover</CoverletOutputFormat>
  <Include>[Namespace]*</Include>
  <Exclude>[Namespace].Tests*</Exclude>
</PropertyGroup>
```

4. **Test Coverage nasıl artırılır?**
   - **Cevap**:
```csharp
// Örnek: Eksik test senaryolarını ekleme
public class CalculatorTests
{
    [Theory]
    [InlineData(2, 3, 5)]
    [InlineData(-1, 1, 0)]
    [InlineData(0, 0, 0)]
    public void Add_ShouldReturnCorrectSum(int a, int b, int expected)
    {
        var calculator = new Calculator();
        var result = calculator.Add(a, b);
        Assert.Equal(expected, result);
    }
}
```

5. **Test Coverage nasıl izlenir?**
   - **Cevap**:
```yaml
# GitHub Actions örneği
name: Test Coverage

on: [push, pull_request]

jobs:
  coverage:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Setup .NET
      uses: actions/setup-dotnet@v1
    - name: Test with coverage
      run: dotnet test /p:CollectCoverage=true
    - name: Generate report
      run: reportgenerator -reports:coverage.cobertura.xml -targetdir:coveragereport
```

### Pratik Sorular

1. **Aşağıdaki sınıfın test coverage'ını artırın:**
```csharp
public class UserValidator
{
    public bool Validate(User user)
    {
        if (user == null)
            return false;

        if (string.IsNullOrEmpty(user.Name))
            return false;

        if (user.Age < 18)
            return false;

        return true;
    }
}
```

- **Cevap**:
```csharp
public class UserValidatorTests
{
    [Theory]
    [InlineData(null, false)]
    [InlineData("", false)]
    [InlineData("John", 17, false)]
    [InlineData("John", 18, true)]
    [InlineData("John", 25, true)]
    public void Validate_ShouldReturnCorrectResult(string name, int age, bool expected)
    {
        // Arrange
        var validator = new UserValidator();
        var user = name != null ? new User { Name = name, Age = age } : null;

        // Act
        var result = validator.Validate(user);

        // Assert
        Assert.Equal(expected, result);
    }
}
```

2. **Aşağıdaki servisin test coverage'ını ölçün:**
```csharp
public class PaymentService
{
    private readonly IPaymentGateway _gateway;
    private readonly ILogger _logger;

    public PaymentService(IPaymentGateway gateway, ILogger logger)
    {
        _gateway = gateway;
        _logger = logger;
    }

    public async Task<PaymentResult> ProcessPayment(PaymentRequest request)
    {
        try
        {
            if (request.Amount <= 0)
                return new PaymentResult { Success = false, Message = "Invalid amount" };

            var result = await _gateway.Process(request);
            
            if (result.Success)
                _logger.LogInformation($"Payment processed: {request.Amount}");
            else
                _logger.LogError($"Payment failed: {result.Message}");

            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Payment processing failed");
            return new PaymentResult { Success = false, Message = "Internal error" };
        }
    }
}
```

- **Cevap**:
```csharp
public class PaymentServiceTests
{
    [Theory]
    [InlineData(0, false, "Invalid amount")]
    [InlineData(-100, false, "Invalid amount")]
    [InlineData(100, true, "Success")]
    public async Task ProcessPayment_ShouldHandleAmountValidation(decimal amount, bool expectedSuccess, string expectedMessage)
    {
        // Arrange
        var mockGateway = new Mock<IPaymentGateway>();
        var mockLogger = new Mock<ILogger>();
        var service = new PaymentService(mockGateway.Object, mockLogger.Object);
        var request = new PaymentRequest { Amount = amount };

        // Act
        var result = await service.ProcessPayment(request);

        // Assert
        Assert.Equal(expectedSuccess, result.Success);
        Assert.Equal(expectedMessage, result.Message);
    }

    [Fact]
    public async Task ProcessPayment_ShouldLogSuccess()
    {
        // Arrange
        var mockGateway = new Mock<IPaymentGateway>();
        var mockLogger = new Mock<ILogger>();
        var service = new PaymentService(mockGateway.Object, mockLogger.Object);
        var request = new PaymentRequest { Amount = 100 };

        mockGateway.Setup(x => x.Process(request))
                   .ReturnsAsync(new PaymentResult { Success = true });

        // Act
        await service.ProcessPayment(request);

        // Assert
        mockLogger.Verify(x => x.LogInformation(It.Is<string>(s => s.Contains("processed"))), Times.Once);
    }

    [Fact]
    public async Task ProcessPayment_ShouldHandleExceptions()
    {
        // Arrange
        var mockGateway = new Mock<IPaymentGateway>();
        var mockLogger = new Mock<ILogger>();
        var service = new PaymentService(mockGateway.Object, mockLogger.Object);
        var request = new PaymentRequest { Amount = 100 };

        mockGateway.Setup(x => x.Process(request))
                   .ThrowsAsync(new Exception("Test exception"));

        // Act
        var result = await service.ProcessPayment(request);

        // Assert
        Assert.False(result.Success);
        Assert.Equal("Internal error", result.Message);
        mockLogger.Verify(x => x.LogError(It.IsAny<Exception>(), It.Is<string>(s => s.Contains("failed"))), Times.Once);
    }
}
```

### İleri Seviye Sorular

1. **Test Coverage ve kod kalitesi arasındaki ilişki nedir?**
   - **Cevap**:
     - Yüksek coverage kalite garantisi değil
     - Test kalitesi önemli
     - Edge case'lerin testi önemli
     - Test maintainability önemli
     - Test readability önemli

2. **Test Coverage'ı nasıl optimize edersiniz?**
   - **Cevap**:
     - Kritik kodları önceliklendirin
     - Test edilebilir kod yazın
     - Test stratejisi geliştirin
     - Coverage araçlarını etkin kullanın
     - Düzenli ölçüm ve analiz yapın

3. **Test Coverage'ı CI/CD'ye nasıl entegre edersiniz?**
   - **Cevap**:
     - Pipeline'da coverage ölçümü
     - Coverage raporları oluşturma
     - Coverage eşikleri belirleme
     - Trend analizi yapma
     - Otomatik uyarılar oluşturma

4. **Test Coverage ve performans arasındaki denge nasıl sağlanır?**
   - **Cevap**:
     - Kritik kodları önceliklendirin
     - Test süresini optimize edin
     - Paralel test çalıştırın
     - Test verilerini yönetin
     - Test ortamını optimize edin

5. **Test Coverage'ı nasıl raporlarsınız?**
   - **Cevap**:
     - HTML raporları oluşturun
     - Trend grafikleri çizin
     - Ekip raporları hazırlayın
     - Dashboard'lar oluşturun
     - Otomatik uyarılar ayarlayın 