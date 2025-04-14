# AWS Services

## Genel Bakış
Amazon Web Services (AWS), Amazon'un bulut bilişim platformudur ve geniş bir yelpazede hizmet sunar. Bu hizmetler, uygulamaların geliştirilmesi, dağıtılması ve yönetilmesi için gerekli tüm altyapıyı ve araçları sağlar.

## Temel Hizmetler

### 1. Compute Services
```csharp
public class AWSComputeService
{
    private readonly ILogger<AWSComputeService> _logger;
    private readonly IAmazonEC2 _ec2Client;

    public AWSComputeService(
        ILogger<AWSComputeService> logger,
        IAmazonEC2 ec2Client)
    {
        _logger = logger;
        _ec2Client = ec2Client;
    }

    public async Task LaunchEC2InstanceAsync(EC2Config config)
    {
        var request = new RunInstancesRequest
        {
            ImageId = config.ImageId,
            InstanceType = config.InstanceType,
            MinCount = 1,
            MaxCount = 1,
            SecurityGroupIds = config.SecurityGroupIds,
            SubnetId = config.SubnetId,
            KeyName = config.KeyName
        };

        var response = await _ec2Client.RunInstancesAsync(request);
        _logger.LogInformation($"EC2 instance launched with ID: {response.Reservation.Instances[0].InstanceId}");
    }
}
```

### 2. Storage Services
```csharp
public class AWSStorageService
{
    private readonly ILogger<AWSStorageService> _logger;
    private readonly IAmazonS3 _s3Client;

    public AWSStorageService(
        ILogger<AWSStorageService> logger,
        IAmazonS3 s3Client)
    {
        _logger = logger;
        _s3Client = s3Client;
    }

    public async Task UploadFileAsync(string bucketName, string key, Stream content)
    {
        var request = new PutObjectRequest
        {
            BucketName = bucketName,
            Key = key,
            InputStream = content
        };

        await _s3Client.PutObjectAsync(request);
        _logger.LogInformation($"File uploaded to S3: {bucketName}/{key}");
    }

    public async Task<Stream> DownloadFileAsync(string bucketName, string key)
    {
        var request = new GetObjectRequest
        {
            BucketName = bucketName,
            Key = key
        };

        var response = await _s3Client.GetObjectAsync(request);
        return response.ResponseStream;
    }
}
```

### 3. Networking Services
```csharp
public class AWSNetworkingService
{
    private readonly ILogger<AWSNetworkingService> _logger;
    private readonly IAmazonEC2 _ec2Client;

    public AWSNetworkingService(
        ILogger<AWSNetworkingService> logger,
        IAmazonEC2 ec2Client)
    {
        _logger = logger;
        _ec2Client = ec2Client;
    }

    public async Task CreateVPCAsync(string cidrBlock)
    {
        var request = new CreateVpcRequest
        {
            CidrBlock = cidrBlock
        };

        var response = await _ec2Client.CreateVpcAsync(request);
        _logger.LogInformation($"VPC created with ID: {response.Vpc.VpcId}");
    }

    public async Task CreateSubnetAsync(string vpcId, string cidrBlock, string availabilityZone)
    {
        var request = new CreateSubnetRequest
        {
            VpcId = vpcId,
            CidrBlock = cidrBlock,
            AvailabilityZone = availabilityZone
        };

        var response = await _ec2Client.CreateSubnetAsync(request);
        _logger.LogInformation($"Subnet created with ID: {response.Subnet.SubnetId}");
    }
}
```

### 4. Database Services
```csharp
public class AWSDatabaseService
{
    private readonly ILogger<AWSDatabaseService> _logger;
    private readonly IAmazonRDS _rdsClient;

    public AWSDatabaseService(
        ILogger<AWSDatabaseService> logger,
        IAmazonRDS rdsClient)
    {
        _logger = logger;
        _rdsClient = rdsClient;
    }

    public async Task CreateRDSInstanceAsync(RDSConfig config)
    {
        var request = new CreateDBInstanceRequest
        {
            DBInstanceIdentifier = config.InstanceIdentifier,
            DBInstanceClass = config.InstanceClass,
            Engine = config.Engine,
            MasterUsername = config.MasterUsername,
            MasterUserPassword = config.MasterUserPassword,
            AllocatedStorage = config.AllocatedStorage,
            VpcSecurityGroupIds = config.SecurityGroupIds,
            DBSubnetGroupName = config.SubnetGroupName
        };

        var response = await _rdsClient.CreateDBInstanceAsync(request);
        _logger.LogInformation($"RDS instance created with ID: {response.DBInstance.DBInstanceIdentifier}");
    }
}
```

### 5. AI & Machine Learning Services
```csharp
public class AWSMachineLearningService
{
    private readonly ILogger<AWSMachineLearningService> _logger;
    private readonly IAmazonComprehend _comprehendClient;

    public AWSMachineLearningService(
        ILogger<AWSMachineLearningService> logger,
        IAmazonComprehend comprehendClient)
    {
        _logger = logger;
        _comprehendClient = comprehendClient;
    }

    public async Task<SentimentAnalysisResult> AnalyzeSentimentAsync(string text)
    {
        var request = new DetectSentimentRequest
        {
            Text = text,
            LanguageCode = "tr"
        };

        var response = await _comprehendClient.DetectSentimentAsync(request);
        return new SentimentAnalysisResult
        {
            Sentiment = response.Sentiment,
            SentimentScore = response.SentimentScore
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
- AWS IAM yapılandırması
- Güvenlik grupları ve NACL'ler
- AWS KMS kullanımı
- AWS WAF yapılandırması
- Uyumluluk sertifikaları

### 3. Monitoring ve Yönetim
- CloudWatch kullanımı
- CloudTrail yapılandırması
- Alert kuralları
- Otomatik ölçeklendirme
- Yedekleme ve felaket kurtarma

## Sık Sorulan Sorular

### 1. AWS servisleri arasında nasıl seçim yapılır?
- İş gereksinimleri analizi
- Maliyet analizi
- Ölçeklenebilirlik gereksinimleri
- Güvenlik gereksinimleri
- Entegrasyon kolaylığı

### 2. AWS maliyetlerini nasıl optimize edebiliriz?
- Rezerve edilmiş örnekler
- Spot örnekleri
- Otomatik ölçeklendirme
- Kaynak kullanımını izleme
- Kullanılmayan kaynakları kapatma

### 3. AWS güvenliği nasıl sağlanır?
- IAM kimlik doğrulama
- Güvenlik grupları
- AWS KMS
- AWS WAF
- Güvenlik izleme ve raporlama

## Kaynaklar
- [AWS Documentation](https://docs.aws.amazon.com/)
- [AWS Architecture Center](https://aws.amazon.com/architecture/)
- [AWS Best Practices](https://aws.amazon.com/architecture/well-architected/)
- [AWS Pricing Calculator](https://calculator.aws/) 