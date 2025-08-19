# Deployment Strategies

## Giriş

Deployment Strategies, uygulamaların production environment'a güvenli ve etkili bir şekilde deploy edilmesi için kullanılan yaklaşımlardır. Mid-level geliştiriciler için deployment stratejilerini anlamak, zero-downtime deployment, rollback capability ve risk minimization için kritik öneme sahiptir. Bu dosya, blue-green deployment, rolling deployment, canary deployment ve deployment automation konularını kapsar.

## Blue-Green Deployment

### 1. Blue-Green Deployment Service
Blue-green deployment pattern'ini implement eden servis.

```csharp
public interface IBlueGreenDeploymentService
{
    Task<DeploymentResult> DeployAsync(DeploymentRequest request);
    Task<bool> SwitchTrafficAsync(string deploymentId, TrafficSwitchRequest request);
    Task<bool> RollbackAsync(string deploymentId);
    Task<DeploymentStatus> GetDeploymentStatusAsync(string deploymentId);
}

public class BlueGreenDeploymentService : IBlueGreenDeploymentService
{
    private readonly ILogger<BlueGreenDeploymentService> _logger;
    private readonly IConfiguration _configuration;
    private readonly ILoadBalancerService _loadBalancerService;
    private readonly IHealthCheckService _healthCheckService;
    private readonly IDeploymentRepository _deploymentRepository;
    
    public BlueGreenDeploymentService(
        ILogger<BlueGreenDeploymentService> logger,
        IConfiguration configuration,
        ILoadBalancerService loadBalancerService,
        IHealthCheckService healthCheckService,
        IDeploymentRepository deploymentRepository)
    {
        _logger = logger;
        _configuration = configuration;
        _loadBalancerService = loadBalancerService;
        _healthCheckService = healthCheckService;
        _deploymentRepository = deploymentRepository;
    }
    
    public async Task<DeploymentResult> DeployAsync(DeploymentRequest request)
    {
        try
        {
            _logger.LogInformation("Starting blue-green deployment for application {ApplicationName}", 
                request.ApplicationName);
            
            // Create deployment record
            var deployment = new Deployment
            {
                Id = Guid.NewGuid().ToString(),
                ApplicationName = request.ApplicationName,
                Version = request.Version,
                Environment = request.Environment,
                Strategy = "BlueGreen",
                Status = DeploymentStatus.InProgress,
                CreatedAt = DateTime.UtcNow
            };
            
            await _deploymentRepository.CreateAsync(deployment);
            
            // Deploy to inactive environment (Green)
            var deployResult = await DeployToEnvironmentAsync(request, deployment.Id);
            if (!deployResult.Success)
            {
                deployment.Status = DeploymentStatus.Failed;
                await _deploymentRepository.UpdateAsync(deployment);
                return new DeploymentResult { Success = false, Error = deployResult.Error };
            }
            
            // Run health checks
            var healthCheckResult = await RunHealthChecksAsync(request.HealthCheckUrl);
            if (!healthCheckResult.Success)
            {
                _logger.LogWarning("Health checks failed for deployment {DeploymentId}", deployment.Id);
                deployment.Status = DeploymentStatus.HealthCheckFailed;
                await _deploymentRepository.UpdateAsync(deployment);
                return new DeploymentResult { Success = false, Error = "Health checks failed" };
            }
            
            // Run smoke tests
            var smokeTestResult = await RunSmokeTestsAsync(request.SmokeTestUrl);
            if (!smokeTestResult.Success)
            {
                _logger.LogWarning("Smoke tests failed for deployment {DeploymentId}", deployment.Id);
                deployment.Status = DeploymentStatus.SmokeTestFailed;
                await _deploymentRepository.UpdateAsync(deployment);
                return new DeploymentResult { Success = false, Error = "Smoke tests failed" };
            }
            
            deployment.Status = DeploymentStatus.Ready;
            await _deploymentRepository.UpdateAsync(deployment);
            
            _logger.LogInformation("Blue-green deployment completed successfully for {DeploymentId}", deployment.Id);
            
            return new DeploymentResult 
            { 
                Success = true, 
                DeploymentId = deployment.Id,
                Status = deployment.Status
            };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error during blue-green deployment for {ApplicationName}", request.ApplicationName);
            return new DeploymentResult { Success = false, Error = ex.Message };
        }
    }
    
    public async Task<bool> SwitchTrafficAsync(string deploymentId, TrafficSwitchRequest request)
    {
        try
        {
            var deployment = await _deploymentRepository.GetByIdAsync(deploymentId);
            if (deployment == null)
            {
                _logger.LogError("Deployment {DeploymentId} not found", deploymentId);
                return false;
            }
            
            if (deployment.Status != DeploymentStatus.Ready)
            {
                _logger.LogWarning("Cannot switch traffic for deployment {DeploymentId} with status {Status}", 
                    deploymentId, deployment.Status);
                return false;
            }
            
            _logger.LogInformation("Switching traffic for deployment {DeploymentId}", deploymentId);
            
            // Switch traffic to new environment
            var switchResult = await _loadBalancerService.SwitchTrafficAsync(
                request.CurrentEnvironment, 
                request.NewEnvironment, 
                request.Percentage);
            
            if (switchResult)
            {
                deployment.Status = DeploymentStatus.Active;
                deployment.TrafficSwitchAt = DateTime.UtcNow;
                await _deploymentRepository.UpdateAsync(deployment);
                
                _logger.LogInformation("Traffic switched successfully for deployment {DeploymentId}", deploymentId);
                return true;
            }
            
            return false;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error switching traffic for deployment {DeploymentId}", deploymentId);
            return false;
        }
    }
    
    public async Task<bool> RollbackAsync(string deploymentId)
    {
        try
        {
            var deployment = await _deploymentRepository.GetByIdAsync(deploymentId);
            if (deployment == null)
            {
                _logger.LogError("Deployment {DeploymentId} not found", deploymentId);
                return false;
            }
            
            _logger.LogInformation("Rolling back deployment {DeploymentId}", deploymentId);
            
            // Switch traffic back to previous environment
            var rollbackResult = await _loadBalancerService.RollbackTrafficAsync(
                deployment.Environment);
            
            if (rollbackResult)
            {
                deployment.Status = DeploymentStatus.RolledBack;
                deployment.RollbackAt = DateTime.UtcNow;
                await _deploymentRepository.UpdateAsync(deployment);
                
                _logger.LogInformation("Rollback completed successfully for deployment {DeploymentId}", deploymentId);
                return true;
            }
            
            return false;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error rolling back deployment {DeploymentId}", deploymentId);
            return false;
        }
    }
    
    private async Task<DeploymentResult> DeployToEnvironmentAsync(DeploymentRequest request, string deploymentId)
    {
        // Implementation for deploying to target environment
        await Task.Delay(1000); // Simulate deployment
        return new DeploymentResult { Success = true };
    }
    
    private async Task<HealthCheckResult> RunHealthChecksAsync(string healthCheckUrl)
    {
        // Implementation for running health checks
        await Task.Delay(500); // Simulate health checks
        return new HealthCheckResult { Success = true };
    }
    
    private async Task<SmokeTestResult> RunSmokeTestsAsync(string smokeTestUrl)
    {
        // Implementation for running smoke tests
        await Task.Delay(500); // Simulate smoke tests
        return new SmokeTestResult { Success = true };
    }
}

public class DeploymentRequest
{
    public string ApplicationName { get; set; }
    public string Version { get; set; }
    public string Environment { get; set; }
    public string HealthCheckUrl { get; set; }
    public string SmokeTestUrl { get; set; }
}

public class TrafficSwitchRequest
{
    public string CurrentEnvironment { get; set; }
    public string NewEnvironment { get; set; }
    public int Percentage { get; set; }
}

public class DeploymentResult
{
    public bool Success { get; set; }
    public string DeploymentId { get; set; }
    public DeploymentStatus Status { get; set; }
    public string Error { get; set; }
}

public class Deployment
{
    public string Id { get; set; }
    public string ApplicationName { get; set; }
    public string Version { get; set; }
    public string Environment { get; set; }
    public string Strategy { get; set; }
    public DeploymentStatus Status { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? TrafficSwitchAt { get; set; }
    public DateTime? RollbackAt { get; set; }
}

public enum DeploymentStatus
{
    InProgress,
    Ready,
    Active,
    Failed,
    HealthCheckFailed,
    SmokeTestFailed,
    RolledBack
}
```

## Rolling Deployment

### 1. Rolling Deployment Service
Rolling deployment pattern'ini implement eden servis.

```csharp
public class RollingDeploymentService : IDeploymentService
{
    private readonly ILogger<RollingDeploymentService> _logger;
    private readonly IConfiguration _configuration;
    private readonly ILoadBalancerService _loadBalancerService;
    private readonly IHealthCheckService _healthCheckService;
    
    public async Task<DeploymentResult> DeployAsync(DeploymentRequest request)
    {
        try
        {
            _logger.LogInformation("Starting rolling deployment for {ApplicationName}", request.ApplicationName);
            
            var instances = await GetApplicationInstancesAsync(request.ApplicationName);
            var batchSize = request.BatchSize ?? 1;
            var healthCheckDelay = request.HealthCheckDelay ?? TimeSpan.FromSeconds(30);
            
            var deployedInstances = new List<string>();
            var failedInstances = new List<string>();
            
            // Deploy in batches
            for (int i = 0; i < instances.Count; i += batchSize)
            {
                var batch = instances.Skip(i).Take(batchSize).ToList();
                
                _logger.LogInformation("Deploying batch {BatchNumber} with {InstanceCount} instances", 
                    (i / batchSize) + 1, batch.Count);
                
                foreach (var instance in batch)
                {
                    try
                    {
                        // Remove from load balancer
                        await _loadBalancerService.RemoveInstanceAsync(instance);
                        
                        // Deploy new version
                        var deployResult = await DeployToInstanceAsync(instance, request.Version);
                        if (deployResult.Success)
                        {
                            // Run health checks
                            var healthResult = await RunHealthChecksAsync(instance, healthCheckDelay);
                            if (healthResult.Success)
                            {
                                // Add back to load balancer
                                await _loadBalancerService.AddInstanceAsync(instance);
                                deployedInstances.Add(instance);
                                
                                _logger.LogInformation("Instance {Instance} deployed successfully", instance);
                            }
                            else
                            {
                                failedInstances.Add(instance);
                                _logger.LogError("Health checks failed for instance {Instance}", instance);
                            }
                        }
                        else
                        {
                            failedInstances.Add(instance);
                            _logger.LogError("Deployment failed for instance {Instance}", instance);
                        }
                    }
                    catch (Exception ex)
                    {
                        failedInstances.Add(instance);
                        _logger.LogError(ex, "Error deploying to instance {Instance}", instance);
                    }
                }
                
                // Wait between batches
                if (i + batchSize < instances.Count)
                {
                    await Task.Delay(request.BatchDelay ?? TimeSpan.FromMinutes(2));
                }
            }
            
            var success = failedInstances.Count == 0;
            var status = success ? DeploymentStatus.Completed : DeploymentStatus.PartiallyCompleted;
            
            _logger.LogInformation("Rolling deployment completed. Success: {Success}, " +
                "Deployed: {DeployedCount}, Failed: {FailedCount}", 
                success, deployedInstances.Count, failedInstances.Count);
            
            return new DeploymentResult
            {
                Success = success,
                Status = status,
                DeployedInstances = deployedInstances,
                FailedInstances = failedInstances
            };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error during rolling deployment");
            return new DeploymentResult { Success = false, Error = ex.Message };
        }
    }
    
    private async Task<List<string>> GetApplicationInstancesAsync(string applicationName)
    {
        // Implementation to get application instances
        return new List<string> { "instance1", "instance2", "instance3" };
    }
    
    private async Task<DeploymentResult> DeployToInstanceAsync(string instance, string version)
    {
        // Implementation to deploy to specific instance
        await Task.Delay(1000);
        return new DeploymentResult { Success = true };
    }
    
    private async Task<HealthCheckResult> RunHealthChecksAsync(string instance, TimeSpan delay)
    {
        // Wait for application to start
        await Task.Delay(delay);
        
        // Implementation to run health checks
        return new HealthCheckResult { Success = true };
    }
}
```

## Canary Deployment

### 1. Canary Deployment Service
Canary deployment pattern'ini implement eden servis.

```csharp
public class CanaryDeploymentService : IDeploymentService
{
    private readonly ILogger<CanaryDeploymentService> _logger;
    private readonly IConfiguration _configuration;
    private readonly ILoadBalancerService _loadBalancerService;
    private readonly IMetricsService _metricsService;
    
    public async Task<DeploymentResult> DeployAsync(DeploymentRequest request)
    {
        try
        {
            _logger.LogInformation("Starting canary deployment for {ApplicationName}", request.ApplicationName);
            
            // Deploy to canary environment
            var canaryResult = await DeployToCanaryAsync(request);
            if (!canaryResult.Success)
            {
                return canaryResult;
            }
            
            // Route small percentage of traffic
            var initialTrafficPercentage = request.InitialTrafficPercentage ?? 5;
            await _loadBalancerService.SetTrafficPercentageAsync(
                request.CanaryEnvironment, initialTrafficPercentage);
            
            // Monitor canary performance
            var monitoringResult = await MonitorCanaryAsync(request, initialTrafficPercentage);
            if (!monitoringResult.Success)
            {
                // Rollback canary
                await RollbackCanaryAsync(request);
                return new DeploymentResult { Success = false, Error = "Canary monitoring failed" };
            }
            
            // Gradually increase traffic
            var trafficSteps = new[] { 10, 25, 50, 75, 100 };
            foreach (var percentage in trafficSteps)
            {
                if (percentage <= initialTrafficPercentage) continue;
                
                _logger.LogInformation("Increasing canary traffic to {Percentage}%", percentage);
                
                await _loadBalancerService.SetTrafficPercentageAsync(
                    request.CanaryEnvironment, percentage);
                
                // Monitor for specified duration
                await Task.Delay(request.TrafficStepDelay ?? TimeSpan.FromMinutes(5));
                
                var stepResult = await MonitorCanaryAsync(request, percentage);
                if (!stepResult.Success)
                {
                    // Rollback to previous percentage
                    var previousPercentage = trafficSteps.TakeWhile(p => p < percentage).LastOrDefault();
                    await _loadBalancerService.SetTrafficPercentageAsync(
                        request.CanaryEnvironment, previousPercentage);
                    
                    _logger.LogWarning("Canary monitoring failed at {Percentage}% traffic", percentage);
                    break;
                }
            }
            
            _logger.LogInformation("Canary deployment completed successfully");
            return new DeploymentResult { Success = true, Status = DeploymentStatus.Completed };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error during canary deployment");
            return new DeploymentResult { Success = false, Error = ex.Message };
        }
    }
    
    private async Task<DeploymentResult> DeployToCanaryAsync(DeploymentRequest request)
    {
        // Implementation to deploy to canary environment
        await Task.Delay(2000);
        return new DeploymentResult { Success = true };
    }
    
    private async Task<MonitoringResult> MonitorCanaryAsync(DeploymentRequest request, int trafficPercentage)
    {
        // Implementation to monitor canary performance
        await Task.Delay(1000);
        return new MonitoringResult { Success = true };
    }
    
    private async Task RollbackCanaryAsync(DeploymentRequest request)
    {
        // Implementation to rollback canary deployment
        await Task.Delay(1000);
    }
}

public class MonitoringResult
{
    public bool Success { get; set; }
    public Dictionary<string, double> Metrics { get; set; } = new Dictionary<string, double>();
}
```

## Mülakat Soruları

### Temel Sorular

1. **Blue-Green Deployment nedir ve avantajları nelerdir?**
   - **Cevap**: İki environment kullanarak zero-downtime deployment. Avantajları: instant rollback, risk minimization, easy testing.

2. **Rolling Deployment nasıl çalışır?**
   - **Cevap**: Instance'ları batch'ler halinde deploy eder. Her batch deploy edildikten sonra health check yapılır.

3. **Canary Deployment nedir?**
   - **Cevap**: Küçük trafik yüzdesi ile yeni versiyonu test eder. Gradual traffic increase ile risk minimize edilir.

4. **Deployment automation neden önemlidir?**
   - **Cevap**: Consistency, repeatability, error reduction, faster deployment, rollback capability.

5. **Health checks deployment'ta neden kritiktir?**
   - **Cevap**: Deployment success validation, issue detection, automatic rollback triggers.

### Teknik Sorular

1. **Blue-green deployment'ta traffic switching nasıl yapılır?**
   - **Cevap**: Load balancer configuration, health check validation, gradual traffic shift.

2. **Rolling deployment'ta batch size nasıl belirlenir?**
   - **Cevap**: Application capacity, risk tolerance, health check duration, rollback time.

3. **Canary deployment'ta monitoring metrics nelerdir?**
   - **Cevap**: Error rate, response time, throughput, business metrics, custom KPIs.

4. **Deployment rollback nasıl implement edilir?**
   - **Cevap**: Automatic triggers, manual triggers, health check failures, metrics thresholds.

5. **Deployment strategies arasında nasıl seçim yapılır?**
   - **Cevap**: Application type, downtime tolerance, risk tolerance, infrastructure complexity.

## Best Practices

1. **Deployment Strategy Selection**
   - Application requirements analiz edin
   - Risk tolerance değerlendirin
   - Infrastructure capabilities göz önünde bulundurun
   - Team expertise değerlendirin

2. **Health Check Implementation**
   - Comprehensive health checks implement edin
   - Timeout management yapın
   - Critical path validation sağlayın
   - Performance impact minimize edin

3. **Monitoring & Alerting**
   - Real-time monitoring implement edin
   - Automated alerting kurun
   - Business metrics track edin
   - Performance baselines oluşturun

4. **Rollback Strategy**
   - Automated rollback triggers implement edin
   - Manual rollback capability sağlayın
   - Rollback testing yapın
   - Data consistency validate edin

5. **Documentation & Training**
   - Deployment procedures document edin
   - Team training sağlayın
   - Runbook'lar oluşturun
   - Post-deployment reviews yapın

## Kaynaklar

- [Blue-Green Deployment](https://martinfowler.com/bliki/BlueGreenDeployment.html)
- [Rolling Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment)
- [Canary Deployment](https://martinfowler.com/bliki/CanaryRelease.html)
- [Deployment Strategies](https://docs.microsoft.com/en-us/azure/devops/pipelines/release/deployment-patterns/)
- [Zero-Downtime Deployment](https://cloud.google.com/architecture/zero-downtime-deployment)
