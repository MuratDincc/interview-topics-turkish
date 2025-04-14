# Infrastructure as Code (Altyapı Kod Olarak)

## Genel Bakış
Infrastructure as Code (IaC), altyapı kaynaklarının kod olarak tanımlanması ve yönetilmesi yaklaşımıdır. Bu sayede altyapı kaynakları versiyon kontrolü altında tutulabilir, otomatik olarak sağlanabilir ve tekrar tekrar oluşturulabilir.

## Temel Kavramlar

### 1. Terraform ile Altyapı Tanımlama
```csharp
public class TerraformService
{
    private readonly ILogger<TerraformService> _logger;

    public TerraformService(ILogger<TerraformService> logger)
    {
        _logger = logger;
    }

    public async Task CreateInfrastructureAsync(InfrastructureConfig config)
    {
        // Terraform konfigürasyonu oluştur
        var terraformConfig = new TerraformConfiguration
        {
            Provider = new Provider
            {
                Name = "azurerm",
                Version = "~> 2.0",
                Features = new ProviderFeatures()
            },
            ResourceGroups = config.ResourceGroups.Select(rg => new ResourceGroup
            {
                Name = rg.Name,
                Location = rg.Location
            }).ToList(),
            VirtualNetworks = config.VirtualNetworks.Select(vnet => new VirtualNetwork
            {
                Name = vnet.Name,
                AddressSpace = vnet.AddressSpace,
                Subnets = vnet.Subnets.Select(s => new Subnet
                {
                    Name = s.Name,
                    AddressPrefix = s.AddressPrefix
                }).ToList()
            }).ToList()
        };

        await SaveTerraformConfigurationAsync(terraformConfig);
        _logger.LogInformation("Terraform configuration created");
    }
}
```

### 2. ARM Template ile Azure Kaynakları
```csharp
public class ARMTemplateService
{
    private readonly ILogger<ARMTemplateService> _logger;

    public ARMTemplateService(ILogger<ARMTemplateService> logger)
    {
        _logger = logger;
    }

    public async Task CreateARMTemplateAsync(ARMConfig config)
    {
        var template = new ARMTemplate
        {
            Schema = "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
            ContentVersion = "1.0.0.0",
            Parameters = config.Parameters.ToDictionary(
                p => p.Name,
                p => new Parameter
                {
                    Type = p.Type,
                    DefaultValue = p.DefaultValue,
                    AllowedValues = p.AllowedValues
                }),
            Variables = config.Variables.ToDictionary(
                v => v.Name,
                v => v.Value),
            Resources = config.Resources.Select(r => new Resource
            {
                Type = r.Type,
                Name = r.Name,
                ApiVersion = r.ApiVersion,
                Location = r.Location,
                Properties = r.Properties
            }).ToList()
        };

        await SaveARMTemplateAsync(template);
        _logger.LogInformation("ARM template created");
    }
}
```

### 3. Ansible ile Konfigürasyon Yönetimi
```csharp
public class AnsibleService
{
    private readonly ILogger<AnsibleService> _logger;

    public AnsibleService(ILogger<AnsibleService> logger)
    {
        _logger = logger;
    }

    public async Task CreatePlaybookAsync(PlaybookConfig config)
    {
        var playbook = new Playbook
        {
            Name = config.Name,
            Hosts = config.Hosts,
            Tasks = config.Tasks.Select(t => new Task
            {
                Name = t.Name,
                Module = t.Module,
                Args = t.Args,
                When = t.When
            }).ToList(),
            Handlers = config.Handlers.Select(h => new Handler
            {
                Name = h.Name,
                Module = h.Module,
                Args = h.Args
            }).ToList()
        };

        await SavePlaybookAsync(playbook);
        _logger.LogInformation("Ansible playbook created");
    }
}
```

### 4. Pulumi ile Programatik Altyapı
```csharp
public class PulumiService
{
    private readonly ILogger<PulumiService> _logger;

    public PulumiService(ILogger<PulumiService> logger)
    {
        _logger = logger;
    }

    public async Task CreateStackAsync(StackConfig config)
    {
        var stack = new Stack
        {
            Name = config.Name,
            Description = config.Description,
            Resources = config.Resources.Select(r => new Resource
            {
                Type = r.Type,
                Name = r.Name,
                Properties = r.Properties,
                Options = r.Options
            }).ToList(),
            Outputs = config.Outputs.ToDictionary(
                o => o.Name,
                o => o.Value)
        };

        await SaveStackAsync(stack);
        _logger.LogInformation("Pulumi stack created");
    }
}
```

## Best Practices

### 1. Kod Organizasyonu
- Modüler yapı
- Tekrar kullanılabilir modüller
- Versiyon kontrolü
- Dokümantasyon
- Kod kalitesi

### 2. Güvenlik
- Hassas veri yönetimi
- Erişim kontrolü
- Güvenlik politikaları
- Şifreleme
- Denetim

### 3. Test ve Doğrulama
- Altyapı testleri
- Konfigürasyon doğrulama
- Güvenlik taramaları
- Performans testleri
- Uyumluluk kontrolleri

## Sık Sorulan Sorular

### 1. IaC neden önemlidir?
- Tutarlılık
- Hız
- Güvenilirlik
- Versiyon kontrolü
- Otomasyon

### 2. IaC nasıl uygulanır?
- Araç seçimi
- Kod organizasyonu
- Test stratejisi
- Güvenlik
- Monitoring

### 3. IaC zorlukları nelerdir?
- Öğrenme eğrisi
- Araç seçimi
- Karmaşık yapılar
- Güvenlik
- Performans

## Kaynaklar
- [Terraform Documentation](https://www.terraform.io/docs/)
- [Azure Resource Manager Templates](https://docs.microsoft.com/tr-tr/azure/azure-resource-manager/templates/)
- [Ansible Documentation](https://docs.ansible.com/)
- [Pulumi Documentation](https://www.pulumi.com/docs/) 