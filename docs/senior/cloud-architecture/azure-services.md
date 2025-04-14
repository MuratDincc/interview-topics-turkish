# Azure Services

## Genel Bakış
Azure, Microsoft'un bulut bilişim platformudur ve geniş bir yelpazede hizmet sunar. Bu hizmetler, uygulamaların geliştirilmesi, dağıtılması ve yönetilmesi için gerekli tüm altyapıyı ve araçları sağlar.

## Temel Hizmetler

### 1. Compute Services
```csharp
public class AzureComputeService
{
    private readonly ILogger<AzureComputeService> _logger;
    private readonly IConfiguration _configuration;

    public AzureComputeService(
        ILogger<AzureComputeService> logger,
        IConfiguration configuration)
    {
        _logger = logger;
        _configuration = configuration;
    }

    public async Task DeployVirtualMachineAsync(VirtualMachineConfig config)
    {
        // Azure VM oluşturma ve yapılandırma
        var credentials = new DefaultAzureCredential();
        var computeClient = new ComputeManagementClient(credentials)
        {
            SubscriptionId = _configuration["Azure:SubscriptionId"]
        };

        var vmParameters = new VirtualMachine
        {
            Location = config.Location,
            HardwareProfile = new HardwareProfile
            {
                VmSize = config.VmSize
            },
            StorageProfile = new StorageProfile
            {
                ImageReference = new ImageReference
                {
                    Publisher = config.ImagePublisher,
                    Offer = config.ImageOffer,
                    Sku = config.ImageSku,
                    Version = "latest"
                }
            },
            NetworkProfile = new NetworkProfile
            {
                NetworkInterfaces = new List<NetworkInterfaceReference>
                {
                    new NetworkInterfaceReference
                    {
                        Id = config.NetworkInterfaceId
                    }
                }
            }
        };

        await computeClient.VirtualMachines
            .CreateOrUpdateAsync(config.ResourceGroupName, config.VmName, vmParameters);
    }
}
```

### 2. Storage Services
```csharp
public class AzureStorageService
{
    private readonly ILogger<AzureStorageService> _logger;
    private readonly BlobServiceClient _blobServiceClient;

    public AzureStorageService(
        ILogger<AzureStorageService> logger,
        string connectionString)
    {
        _logger = logger;
        _blobServiceClient = new BlobServiceClient(connectionString);
    }

    public async Task UploadFileAsync(string containerName, string blobName, Stream content)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
        await containerClient.CreateIfNotExistsAsync();

        var blobClient = containerClient.GetBlobClient(blobName);
        await blobClient.UploadAsync(content, true);
    }

    public async Task<Stream> DownloadFileAsync(string containerName, string blobName)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
        var blobClient = containerClient.GetBlobClient(blobName);

        var response = await blobClient.DownloadAsync();
        return response.Value.Content;
    }
}
```

### 3. Networking Services
```csharp
public class AzureNetworkingService
{
    private readonly ILogger<AzureNetworkingService> _logger;
    private readonly NetworkManagementClient _networkClient;

    public AzureNetworkingService(
        ILogger<AzureNetworkingService> logger,
        IConfiguration configuration)
    {
        _logger = logger;
        var credentials = new DefaultAzureCredential();
        _networkClient = new NetworkManagementClient(credentials)
        {
            SubscriptionId = configuration["Azure:SubscriptionId"]
        };
    }

    public async Task CreateVirtualNetworkAsync(string resourceGroupName, string vnetName, string location)
    {
        var vnet = new VirtualNetwork
        {
            Location = location,
            AddressSpace = new AddressSpace
            {
                AddressPrefixes = new List<string> { "10.0.0.0/16" }
            },
            Subnets = new List<Subnet>
            {
                new Subnet
                {
                    Name = "default",
                    AddressPrefix = "10.0.0.0/24"
                }
            }
        };

        await _networkClient.VirtualNetworks
            .CreateOrUpdateAsync(resourceGroupName, vnetName, vnet);
    }
}
```

### 4. Database Services
```csharp
public class AzureDatabaseService
{
    private readonly ILogger<AzureDatabaseService> _logger;
    private readonly SqlManagementClient _sqlClient;

    public AzureDatabaseService(
        ILogger<AzureDatabaseService> logger,
        IConfiguration configuration)
    {
        _logger = logger;
        var credentials = new DefaultAzureCredential();
        _sqlClient = new SqlManagementClient(credentials)
        {
            SubscriptionId = configuration["Azure:SubscriptionId"]
        };
    }

    public async Task CreateSqlDatabaseAsync(
        string resourceGroupName,
        string serverName,
        string databaseName)
    {
        var database = new Database
        {
            Location = "West Europe",
            Sku = new Sku
            {
                Name = "S0",
                Tier = "Standard"
            }
        };

        await _sqlClient.Databases
            .CreateOrUpdateAsync(resourceGroupName, serverName, databaseName, database);
    }
}
```

### 5. AI & Machine Learning Services
```csharp
public class AzureAIService
{
    private readonly ILogger<AzureAIService> _logger;
    private readonly TextAnalyticsClient _textAnalyticsClient;

    public AzureAIService(
        ILogger<AzureAIService> logger,
        string endpoint,
        string key)
    {
        _logger = logger;
        _textAnalyticsClient = new TextAnalyticsClient(
            new Uri(endpoint),
            new AzureKeyCredential(key));
    }

    public async Task<SentimentAnalysisResult> AnalyzeSentimentAsync(string text)
    {
        var response = await _textAnalyticsClient.AnalyzeSentimentAsync(text);
        return new SentimentAnalysisResult
        {
            Sentiment = response.Value.Sentiment,
            ConfidenceScores = response.Value.ConfidenceScores
        };
    }
}
```

## Best Practices

### 1. Servis Seçimi ve Yapılandırma
- İş gereksinimlerine uygun servis seçimi
- Doğru servis katmanı seçimi
- Bölge seçimi ve replikasyon
- Ölçeklendirme stratejisi
- Maliyet optimizasyonu

### 2. Güvenlik ve Uyumluluk
- Azure Active Directory entegrasyonu
- Rol tabanlı erişim kontrolü (RBAC)
- Ağ güvenliği ve güvenlik duvarı kuralları
- Veri şifreleme
- Uyumluluk sertifikaları

### 3. Monitoring ve Yönetim
- Azure Monitor kullanımı
- Log Analytics yapılandırması
- Alert kuralları
- Otomatik ölçeklendirme
- Yedekleme ve felaket kurtarma

## Sık Sorulan Sorular

### 1. Azure servisleri arasında nasıl seçim yapılır?
- İş gereksinimleri analizi
- Maliyet analizi
- Ölçeklenebilirlik gereksinimleri
- Güvenlik gereksinimleri
- Entegrasyon kolaylığı

### 2. Azure maliyetlerini nasıl optimize edebiliriz?
- Rezerve edilmiş örnekler
- Spot örnekleri
- Otomatik ölçeklendirme
- Kaynak kullanımını izleme
- Kullanılmayan kaynakları kapatma

### 3. Azure güvenliği nasıl sağlanır?
- Azure AD kimlik doğrulama
- Ağ güvenlik grupları
- Azure Key Vault
- Azure Security Center
- Güvenlik izleme ve raporlama

## Kaynaklar
- [Azure Documentation](https://docs.microsoft.com/tr-tr/azure/)
- [Azure Architecture Center](https://docs.microsoft.com/tr-tr/azure/architecture/)
- [Azure Best Practices](https://docs.microsoft.com/tr-tr/azure/architecture/best-practices/)
- [Azure Pricing Calculator](https://azure.microsoft.com/tr-tr/pricing/calculator/) 