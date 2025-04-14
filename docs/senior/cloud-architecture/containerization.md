# Containerization

## Genel Bakış
Containerization, uygulamaları ve bağımlılıklarını izole edilmiş, taşınabilir ve tutarlı bir ortamda paketleme sürecidir. Bu sayede uygulamalar farklı ortamlarda sorunsuz çalışabilir ve dağıtım süreçleri kolaylaşır.

## Temel Kavramlar

### 1. Docker Container
```csharp
public class DockerService
{
    private readonly ILogger<DockerService> _logger;
    private readonly DockerClient _dockerClient;

    public DockerService(
        ILogger<DockerService> logger,
        DockerClient dockerClient)
    {
        _logger = logger;
        _dockerClient = dockerClient;
    }

    public async Task CreateContainerAsync(ContainerConfig config)
    {
        var createContainerResponse = await _dockerClient.Containers.CreateContainerAsync(
            new CreateContainerParameters
            {
                Image = config.ImageName,
                Name = config.ContainerName,
                Env = config.EnvironmentVariables,
                ExposedPorts = config.ExposedPorts,
                HostConfig = new HostConfig
                {
                    PortBindings = config.PortBindings,
                    Binds = config.VolumeBindings
                }
            });

        _logger.LogInformation($"Container created with ID: {createContainerResponse.ID}");
    }

    public async Task StartContainerAsync(string containerId)
    {
        await _dockerClient.Containers.StartContainerAsync(
            containerId,
            new ContainerStartParameters());
        
        _logger.LogInformation($"Container started: {containerId}");
    }
}
```

### 2. Container Orchestration
```csharp
public class KubernetesService
{
    private readonly ILogger<KubernetesService> _logger;
    private readonly IKubernetes _kubernetesClient;

    public KubernetesService(
        ILogger<KubernetesService> logger,
        IKubernetes kubernetesClient)
    {
        _logger = logger;
        _kubernetesClient = kubernetesClient;
    }

    public async Task DeployPodAsync(PodConfig config)
    {
        var pod = new V1Pod
        {
            Metadata = new V1ObjectMeta
            {
                Name = config.PodName,
                Namespace = config.Namespace
            },
            Spec = new V1PodSpec
            {
                Containers = new List<V1Container>
                {
                    new V1Container
                    {
                        Name = config.ContainerName,
                        Image = config.Image,
                        Ports = config.Ports.Select(p => new V1ContainerPort
                        {
                            ContainerPort = p
                        }).ToList(),
                        Env = config.EnvironmentVariables.Select(e => new V1EnvVar
                        {
                            Name = e.Key,
                            Value = e.Value
                        }).ToList()
                    }
                }
            }
        };

        await _kubernetesClient.CreateNamespacedPodAsync(pod, config.Namespace);
        _logger.LogInformation($"Pod deployed: {config.PodName}");
    }

    public async Task ScaleDeploymentAsync(string deploymentName, string @namespace, int replicas)
    {
        var scale = new V1Scale
        {
            Spec = new V1ScaleSpec
            {
                Replicas = replicas
            }
        };

        await _kubernetesClient.ReplaceNamespacedDeploymentScaleAsync(
            scale,
            deploymentName,
            @namespace);
        
        _logger.LogInformation($"Deployment scaled to {replicas} replicas");
    }
}
```

### 3. Container Registry
```csharp
public class ContainerRegistryService
{
    private readonly ILogger<ContainerRegistryService> _logger;
    private readonly DockerClient _dockerClient;

    public ContainerRegistryService(
        ILogger<ContainerRegistryService> logger,
        DockerClient dockerClient)
    {
        _logger = logger;
        _dockerClient = dockerClient;
    }

    public async Task PushImageAsync(string imageName, string tag)
    {
        var authConfig = new AuthConfig
        {
            Username = "username",
            Password = "password",
            ServerAddress = "registry.example.com"
        };

        await _dockerClient.Images.PushImageAsync(
            imageName,
            new ImagePushParameters
            {
                Tag = tag
            },
            new AuthConfigParameters
            {
                RegistryAuth = authConfig
            });

        _logger.LogInformation($"Image pushed: {imageName}:{tag}");
    }

    public async Task PullImageAsync(string imageName, string tag)
    {
        await _dockerClient.Images.PullImageAsync(
            new ImagesPullParameters
            {
                FromImage = imageName,
                Tag = tag
            });

        _logger.LogInformation($"Image pulled: {imageName}:{tag}");
    }
}
```

### 4. Container Security
```csharp
public class ContainerSecurityService
{
    private readonly ILogger<ContainerSecurityService> _logger;
    private readonly DockerClient _dockerClient;

    public ContainerSecurityService(
        ILogger<ContainerSecurityService> logger,
        DockerClient dockerClient)
    {
        _logger = logger;
        _dockerClient = dockerClient;
    }

    public async Task ScanContainerAsync(string containerId)
    {
        var container = await _dockerClient.Containers.InspectContainerAsync(containerId);
        
        // Güvenlik kontrolleri
        if (container.Config.User == "root")
        {
            _logger.LogWarning($"Container {containerId} running as root");
        }

        if (container.HostConfig.Privileged)
        {
            _logger.LogWarning($"Container {containerId} running in privileged mode");
        }

        // Port taraması
        foreach (var port in container.NetworkSettings.Ports)
        {
            if (port.Value.Any(p => p.HostPort == "0.0.0.0"))
            {
                _logger.LogWarning($"Container {containerId} exposing port {port.Key} to all interfaces");
            }
        }
    }
}
```

## Best Practices

### 1. Container Yapılandırması
- Minimal base image kullanımı
- Multi-stage builds
- Layer optimizasyonu
- Environment variable yönetimi
- Volume kullanımı

### 2. Güvenlik
- Non-root user kullanımı
- Image signing
- Vulnerability scanning
- Network isolation
- Resource limits

### 3. Monitoring ve Logging
- Container metrics
- Log aggregation
- Health checks
- Alerting
- Tracing

## Sık Sorulan Sorular

### 1. Container ve VM arasındaki farklar nelerdir?
- İzolasyon seviyesi
- Kaynak kullanımı
- Başlatma süresi
- Taşınabilirlik
- Yönetim kolaylığı

### 2. Container orchestration neden önemlidir?
- Ölçeklenebilirlik
- Yüksek erişilebilirlik
- Servis keşfi
- Load balancing
- Otomatik kurtarma

### 3. Container güvenliği nasıl sağlanır?
- Image güvenliği
- Runtime güvenliği
- Network güvenliği
- Access control
- Monitoring

## Kaynaklar
- [Docker Documentation](https://docs.docker.com/)
- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [Container Security Best Practices](https://docs.docker.com/engine/security/)
- [Container Orchestration Patterns](https://kubernetes.io/docs/concepts/workloads/controllers/) 