# Environment Management

## Giriş

Environment Management, software development lifecycle'ında farklı ortamların (development, staging, production) yapılandırılması, yönetimi ve izlenmesi için kritik bir konudur. Mid-level geliştiriciler için environment management'ı anlamak, güvenli ve tutarlı deployment süreçleri tasarlamada esastır. Bu dosya, environment configuration, secrets management, infrastructure as code ve environment monitoring konularını kapsar.

## Environment Configuration

### 1. Environment-Specific Configuration
Farklı ortamlar için configuration management.

```csharp
public class EnvironmentConfigurationService
{
    private readonly ILogger<EnvironmentConfigurationService> _logger;
    private readonly IConfiguration _configuration;
    private readonly IEnvironmentProvider _environmentProvider;
    
    public EnvironmentConfigurationService(
        ILogger<EnvironmentConfigurationService> logger,
        IConfiguration configuration,
        IEnvironmentProvider environmentProvider)
    {
        _logger = logger;
        _configuration = configuration;
        _environmentProvider = environmentProvider;
    }
    
    public async Task<EnvironmentConfig> GetEnvironmentConfigurationAsync(string environmentName = null)
    {
        try
        {
            var environment = environmentName ?? _environmentProvider.GetCurrentEnvironment();
            _logger.LogInformation("Loading configuration for environment: {Environment}", environment);
            
            var config = new EnvironmentConfig
            {
                EnvironmentName = environment,
                LoadedAt = DateTime.UtcNow
            };
            
            // Load base configuration
            config.BaseSettings = await LoadBaseConfigurationAsync();
            
            // Load environment-specific configuration
            config.EnvironmentSettings = await LoadEnvironmentSpecificConfigurationAsync(environment);
            
            // Load secrets
            config.Secrets = await LoadSecretsAsync(environment);
            
            // Validate configuration
            var validationResult = await ValidateConfigurationAsync(config);
            if (!validationResult.IsValid)
            {
                _logger.LogError("Configuration validation failed: {Errors}", 
                    string.Join(", ", validationResult.Errors));
                throw new ConfigurationValidationException("Configuration validation failed", validationResult.Errors);
            }
            
            _logger.LogInformation("Configuration loaded successfully for environment: {Environment}", environment);
            return config;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error loading configuration for environment: {Environment}", environmentName);
            throw;
        }
    }
    
    private async Task<Dictionary<string, object>> LoadBaseConfigurationAsync()
    {
        var baseSettings = new Dictionary<string, object>();
        
        // Load common settings that apply to all environments
        baseSettings["ApplicationName"] = _configuration["Application:Name"];
        baseSettings["Version"] = _configuration["Application:Version"];
        baseSettings["LogLevel"] = _configuration["Logging:LogLevel:Default"];
        baseSettings["AllowedHosts"] = _configuration["AllowedHosts"];
        
        // Load database connection templates
        baseSettings["DatabaseTemplate"] = _configuration["Database:ConnectionTemplate"];
        baseSettings["RedisTemplate"] = _configuration["Cache:Redis:ConnectionTemplate"];
        
        return baseSettings;
    }
    
    private async Task<Dictionary<string, object>> LoadEnvironmentSpecificConfigurationAsync(string environment)
    {
        var envSettings = new Dictionary<string, object>();
        
        // Load environment-specific configuration
        var envConfig = _configuration.GetSection($"Environments:{environment}");
        
        if (envConfig.Exists())
        {
            envSettings["DatabaseConnection"] = envConfig["Database:ConnectionString"];
            envSettings["RedisConnection"] = envConfig["Cache:Redis:ConnectionString"];
            envSettings["ApiBaseUrl"] = envConfig["Api:BaseUrl"];
            envSettings["CorsOrigins"] = envConfig.GetSection("Cors:AllowedOrigins").Get<string[]>();
            envSettings["FeatureFlags"] = envConfig.GetSection("Features").Get<Dictionary<string, bool>>();
            envSettings["LogLevel"] = envConfig["Logging:LogLevel:Default"] ?? "Information";
            envSettings["EnableSwagger"] = envConfig.GetValue<bool>("EnableSwagger", false);
            envSettings["EnableDetailedErrors"] = envConfig.GetValue<bool>("EnableDetailedErrors", false);
        }
        else
        {
            _logger.LogWarning("No specific configuration found for environment: {Environment}", environment);
        }
        
        return envSettings;
    }
    
    private async Task<Dictionary<string, string>> LoadSecretsAsync(string environment)
    {
        var secrets = new Dictionary<string, string>();
        
        try
        {
            // Load secrets from Azure Key Vault or other secret management service
            var secretProvider = _configuration.GetSection("Secrets:Provider").Value;
            
            switch (secretProvider?.ToLowerInvariant())
            {
                case "azurekeyvault":
                    secrets = await LoadAzureKeyVaultSecretsAsync(environment);
                    break;
                case "awssecretsmanager":
                    secrets = await LoadAwsSecretsAsync(environment);
                    break;
                case "hashicorpvault":
                    secrets = await LoadHashiCorpVaultSecretsAsync(environment);
                    break;
                default:
                    _logger.LogWarning("No secret provider configured, using configuration secrets");
                    secrets = await LoadConfigurationSecretsAsync(environment);
                    break;
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error loading secrets for environment: {Environment}", environment);
            // Fall back to configuration secrets
            secrets = await LoadConfigurationSecretsAsync(environment);
        }
        
        return secrets;
    }
    
    private async Task<Dictionary<string, string>> LoadAzureKeyVaultSecretsAsync(string environment)
    {
        var secrets = new Dictionary<string, string>();
        
        try
        {
            var keyVaultUrl = _configuration["Azure:KeyVault:Url"];
            var credential = new DefaultAzureCredential();
            var client = new SecretClient(new Uri(keyVaultUrl), credential);
            
            // Load environment-specific secrets
            var secretNames = new[]
            {
                $"{environment}-db-password",
                $"{environment}-jwt-secret",
                $"{environment}-api-key",
                $"{environment}-redis-password"
            };
            
            foreach (var secretName in secretNames)
            {
                try
                {
                    var secret = await client.GetSecretAsync(secretName);
                    secrets[secretName] = secret.Value.Value;
                }
                catch (RequestFailedException ex) when (ex.Status == 404)
                {
                    _logger.LogDebug("Secret {SecretName} not found in Key Vault", secretName);
                }
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error loading Azure Key Vault secrets");
            throw;
        }
        
        return secrets;
    }
    
    private async Task<Dictionary<string, string>> LoadConfigurationSecretsAsync(string environment)
    {
        var secrets = new Dictionary<string, string>();
        
        // Load secrets from configuration (not recommended for production)
        var secretsSection = _configuration.GetSection($"Secrets:{environment}");
        
        if (secretsSection.Exists())
        {
            foreach (var secret in secretsSection.GetChildren())
            {
                secrets[secret.Key] = secret.Value;
            }
        }
        
        return secrets;
    }
    
    private async Task<ConfigurationValidationResult> ValidateConfigurationAsync(EnvironmentConfig config)
    {
        var result = new ConfigurationValidationResult();
        
        try
        {
            // Validate required settings
            var requiredSettings = new[]
            {
                "DatabaseConnection",
                "RedisConnection",
                "ApiBaseUrl"
            };
            
            foreach (var setting in requiredSettings)
            {
                if (!config.EnvironmentSettings.ContainsKey(setting) || 
                    string.IsNullOrEmpty(config.EnvironmentSettings[setting]?.ToString()))
                {
                    result.AddError(setting, $"Required setting '{setting}' is missing or empty");
                }
            }
            
            // Validate database connection
            if (config.EnvironmentSettings.ContainsKey("DatabaseConnection"))
            {
                var connectionString = config.EnvironmentSettings["DatabaseConnection"].ToString();
                if (!IsValidConnectionString(connectionString))
                {
                    result.AddError("DatabaseConnection", "Invalid database connection string format");
                }
            }
            
            // Validate CORS origins
            if (config.EnvironmentSettings.ContainsKey("CorsOrigins"))
            {
                var corsOrigins = config.EnvironmentSettings["CorsOrigins"] as string[];
                if (corsOrigins == null || corsOrigins.Length == 0)
                {
                    result.AddError("CorsOrigins", "CORS origins must be specified");
                }
            }
            
            // Validate feature flags
            if (config.EnvironmentSettings.ContainsKey("FeatureFlags"))
            {
                var featureFlags = config.EnvironmentSettings["FeatureFlags"] as Dictionary<string, bool>;
                if (featureFlags == null)
                {
                    result.AddError("FeatureFlags", "Feature flags must be a valid dictionary");
                }
            }
            
            result.IsValid = !result.Errors.Any();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error validating configuration");
            result.AddError("Validation", $"Configuration validation error: {ex.Message}");
            result.IsValid = false;
        }
        
        return result;
    }
    
    private bool IsValidConnectionString(string connectionString)
    {
        if (string.IsNullOrEmpty(connectionString))
            return false;
        
        // Basic validation for SQL Server connection string
        return connectionString.Contains("Server=") && 
               connectionString.Contains("Database=") && 
               connectionString.Contains("User Id=") && 
               connectionString.Contains("Password=");
    }
}

public class EnvironmentConfig
{
    public string EnvironmentName { get; set; }
    public DateTime LoadedAt { get; set; }
    public Dictionary<string, object> BaseSettings { get; set; } = new Dictionary<string, object>();
    public Dictionary<string, object> EnvironmentSettings { get; set; } = new Dictionary<string, object>();
    public Dictionary<string, string> Secrets { get; set; } = new Dictionary<string, string>();
    
    public T GetSetting<T>(string key, T defaultValue = default(T))
    {
        // Try environment-specific setting first
        if (EnvironmentSettings.ContainsKey(key))
        {
            var value = EnvironmentSettings[key];
            if (value is T typedValue)
                return typedValue;
            
            // Try to convert
            try
            {
                return (T)Convert.ChangeType(value, typeof(T));
            }
            catch
            {
                // Conversion failed, return default
            }
        }
        
        // Try base setting
        if (BaseSettings.ContainsKey(key))
        {
            var value = BaseSettings[key];
            if (value is T typedValue)
                return typedValue;
            
            // Try to convert
            try
            {
                return (T)Convert.ChangeType(value, typeof(T));
            }
            catch
            {
                // Conversion failed, return default
            }
        }
        
        return defaultValue;
    }
    
    public string GetSecret(string key, string defaultValue = null)
    {
        return Secrets.ContainsKey(key) ? Secrets[key] : defaultValue;
    }
}

public class ConfigurationValidationResult
{
    public bool IsValid { get; set; }
    public List<ConfigurationError> Errors { get; } = new List<ConfigurationError>();
    
    public void AddError(string field, string message)
    {
        Errors.Add(new ConfigurationError { Field = field, Message = message });
    }
}

public class ConfigurationError
{
    public string Field { get; set; }
    public string Message { get; set; }
}

public class ConfigurationValidationException : Exception
{
    public List<ConfigurationError> Errors { get; }
    
    public ConfigurationValidationException(string message, List<ConfigurationError> errors) 
        : base(message)
    {
        Errors = errors;
    }
}
```

### 2. Environment Provider
Environment detection ve management.

```csharp
public interface IEnvironmentProvider
{
    string GetCurrentEnvironment();
    bool IsDevelopment();
    bool IsStaging();
    bool IsProduction();
    bool IsEnvironment(string environmentName);
    EnvironmentInfo GetEnvironmentInfo();
}

public class EnvironmentProvider : IEnvironmentProvider
{
    private readonly IConfiguration _configuration;
    private readonly IWebHostEnvironment _webHostEnvironment;
    private readonly ILogger<EnvironmentProvider> _logger;
    
    public EnvironmentProvider(
        IConfiguration configuration,
        IWebHostEnvironment webHostEnvironment,
        ILogger<EnvironmentProvider> logger)
    {
        _configuration = configuration;
        _webHostEnvironment = webHostEnvironment;
        _logger = logger;
    }
    
    public string GetCurrentEnvironment()
    {
        // Priority order: Environment Variable > Configuration > Default
        var environment = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") ??
                         _configuration["Environment"] ??
                         _webHostEnvironment.EnvironmentName ??
                         "Development";
        
        _logger.LogDebug("Current environment detected: {Environment}", environment);
        return environment;
    }
    
    public bool IsDevelopment()
    {
        return IsEnvironment("Development");
    }
    
    public bool IsStaging()
    {
        return IsEnvironment("Staging");
    }
    
    public bool IsProduction()
    {
        return IsEnvironment("Production");
    }
    
    public bool IsEnvironment(string environmentName)
    {
        var currentEnv = GetCurrentEnvironment();
        return string.Equals(currentEnv, environmentName, StringComparison.OrdinalIgnoreCase);
    }
    
    public EnvironmentInfo GetEnvironmentInfo()
    {
        var currentEnv = GetCurrentEnvironment();
        
        return new EnvironmentInfo
        {
            Name = currentEnv,
            IsDevelopment = IsDevelopment(),
            IsStaging = IsStaging(),
            IsProduction = IsProduction(),
            MachineName = Environment.MachineName,
            ProcessId = Environment.ProcessId,
            OSVersion = Environment.OSVersion.ToString(),
            ProcessorCount = Environment.ProcessorCount,
            WorkingSet = Environment.WorkingSet,
            Timestamp = DateTime.UtcNow
        };
    }
}

public class EnvironmentInfo
{
    public string Name { get; set; }
    public bool IsDevelopment { get; set; }
    public bool IsStaging { get; set; }
    public bool IsProduction { get; set; }
    public string MachineName { get; set; }
    public int ProcessId { get; set; }
    public string OSVersion { get; set; }
    public int ProcessorCount { get; set; }
    public long WorkingSet { get; set; }
    public DateTime Timestamp { get; set; }
}
```

## Infrastructure as Code

### 1. Terraform Configuration
Environment infrastructure'ını Terraform ile yönetme.

```hcl
# main.tf - Main Terraform configuration
terraform {
  required_version = ">= 1.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
  
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "tfstate12345"
    container_name       = "tfstate"
    key                  = "environments.terraform.tfstate"
  }
}

# Variables
variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}

variable "location" {
  description = "Azure region"
  type        = string
  default     = "East US"
}

variable "resource_group_name" {
  description = "Resource group name"
  type        = string
}

variable "app_service_plan_sku" {
  description = "App Service Plan SKU"
  type        = string
  default     = "B1"
}

variable "database_sku" {
  description = "Database SKU"
  type        = string
  default     = "Basic"
}

# Data sources
data "azurerm_resource_group" "main" {
  name = var.resource_group_name
}

data "azurerm_client_config" "current" {}

# Resource group
resource "azurerm_resource_group" "main" {
  count    = var.resource_group_name == null ? 1 : 0
  name     = "rg-${var.environment}-${random_string.suffix.result}"
  location = var.location
  
  tags = local.common_tags
}

# Random suffix for unique names
resource "random_string" "suffix" {
  length  = 8
  special = false
  upper   = false
}

# App Service Plan
resource "azurerm_service_plan" "main" {
  name                = "plan-${var.environment}-${random_string.suffix.result}"
  resource_group_name = data.azurerm_resource_group.main.name
  location           = data.azurerm_resource_group.main.location
  os_type            = "Windows"
  sku_name           = var.app_service_plan_sku
  
  tags = local.common_tags
}

# App Service
resource "azurerm_windows_web_app" "main" {
  name                = "app-${var.environment}-${random_string.suffix.result}"
  resource_group_name = data.azurerm_resource_group.main.name
  location           = data.azurerm_resource_group.main.location
  service_plan_id    = azurerm_service_plan.main.id
  
  site_config {
    application_stack {
      dotnet_version = "v8.0"
    }
    
    application_settings = {
      "ASPNETCORE_ENVIRONMENT" = title(var.environment)
      "WEBSITE_RUN_FROM_PACKAGE" = "1"
    }
    
    health_check_path = "/health"
    health_check_eviction_time_in_minutes = 5
  }
  
  app_settings = local.app_settings
  
  tags = local.common_tags
}

# SQL Server
resource "azurerm_mssql_server" "main" {
  name                         = "sql-${var.environment}-${random_string.suffix.result}"
  resource_group_name          = data.azurerm_resource_group.main.name
  location                    = data.azurerm_resource_group.main.location
  version                     = "12.0"
  administrator_login          = "sqladmin"
  administrator_login_password = random_password.sql_password.result
  
  tags = local.common_tags
}

# SQL Database
resource "azurerm_mssql_database" "main" {
  name           = "db-${var.environment}"
  server_id      = azurerm_mssql_server.main.id
  sku_name       = var.database_sku
  collation      = "SQL_Latin1_General_CP1_CI_AS"
  
  tags = local.common_tags
}

# Redis Cache
resource "azurerm_redis_cache" "main" {
  name                = "redis-${var.environment}-${random_string.suffix.result}"
  resource_group_name = data.azurerm_resource_group.main.name
  location           = data.azurerm_resource_group.main.location
  capacity           = 0
  family             = "C"
  sku_name           = "Basic"
  
  tags = local.common_tags
}

# Key Vault
resource "azurerm_key_vault" "main" {
  name                        = "kv-${var.environment}-${random_string.suffix.result}"
  resource_group_name         = data.azurerm_resource_group.main.name
  location                    = data.azurerm_resource_group.main.location
  enabled_for_disk_encryption = true
  tenant_id                   = data.azurerm_client_config.current.tenant_id
  soft_delete_retention_days  = 7
  purge_protection_enabled    = false
  sku_name                   = "standard"
  
  access_policy {
    tenant_id = data.azurerm_client_config.current.tenant_id
    object_id = data.azurerm_client_config.current.object_id
    
    secret_permissions = [
      "Get", "List", "Set", "Delete", "Recover", "Backup", "Restore"
    ]
    
    key_permissions = [
      "Get", "List", "Create", "Delete", "Recover", "Backup", "Restore"
    ]
  }
  
  tags = local.common_tags
}

# Random password for SQL Server
resource "random_password" "sql_password" {
  length           = 16
  special          = true
  override_special = "!#$%&*()-_=+[]{}<>:?"
}

# Local values
locals {
  common_tags = {
    Environment = var.environment
    Project     = "MyApp"
    ManagedBy   = "Terraform"
    CreatedAt   = timestamp()
  }
  
  app_settings = {
    "ConnectionStrings__DefaultConnection" = "Server=${azurerm_mssql_server.main.fully_qualified_domain_name};Database=${azurerm_mssql_database.main.name};User Id=sqladmin;Password=${random_password.sql_password.result};TrustServerCertificate=true"
    "ConnectionStrings__Redis"            = "${azurerm_redis_cache.main.hostname}:${azurerm_redis_cache.main.ssl_port},password=${azurerm_redis_cache.main.primary_access_key},ssl=True,abortConnect=False"
    "KeyVault__Url"                       = azurerm_key_vault.main.vault_uri
    "Logging__LogLevel__Default"          = var.environment == "prod" ? "Warning" : "Information"
    "Logging__LogLevel__Microsoft"        = var.environment == "prod" ? "Warning" : "Information"
    "Cors__AllowedOrigins"                = var.environment == "prod" ? "https://myapp.com" : "*"
  }
}

# Outputs
output "app_service_url" {
  description = "App Service URL"
  value       = azurerm_windows_web_app.main.default_hostname
}

output "database_connection_string" {
  description = "Database connection string"
  value       = "Server=${azurerm_mssql_server.main.fully_qualified_domain_name};Database=${azurerm_mssql_database.main.name};User Id=sqladmin;Password=${random_password.sql_password.result};TrustServerCertificate=true"
  sensitive   = true
}

output "redis_connection_string" {
  description = "Redis connection string"
  value       = "${azurerm_redis_cache.main.hostname}:${azurerm_redis_cache.main.ssl_port},password=${azurerm_redis_cache.main.primary_access_key},ssl=True,abortConnect=False"
  sensitive   = true
}

output "key_vault_url" {
  description = "Key Vault URL"
  value       = azurerm_key_vault.main.vault_uri
}
```

### 2. Environment-Specific Terraform Files
```hcl
# environments/dev.tfvars
environment = "dev"
location = "East US"
resource_group_name = "rg-myapp-dev"
app_service_plan_sku = "B1"
database_sku = "Basic"

# environments/staging.tfvars
environment = "staging"
location = "East US"
resource_group_name = "rg-myapp-staging"
app_service_plan_sku = "S1"
database_sku = "Standard"

# environments/prod.tfvars
environment = "prod"
location = "East US"
resource_group_name = "rg-myapp-prod"
app_service_plan_sku = "P1v2"
database_sku = "Premium"
```

## Environment Monitoring

### 1. Environment Health Monitoring
Environment sağlık durumunu izleyen servis.

```csharp
public class EnvironmentHealthMonitor : BackgroundService
{
    private readonly ILogger<EnvironmentHealthMonitor> _logger;
    private readonly IConfiguration _configuration;
    private readonly IEnvironmentProvider _environmentProvider;
    private readonly IHealthCheckService _healthCheckService;
    private readonly IMetricsService _metricsService;
    
    public EnvironmentHealthMonitor(
        ILogger<EnvironmentHealthMonitor> logger,
        IConfiguration configuration,
        IEnvironmentProvider environmentProvider,
        IHealthCheckService healthCheckService,
        IMetricsService metricsService)
    {
        _logger = logger;
        _configuration = configuration;
        _environmentProvider = environmentProvider;
        _healthCheckService = healthCheckService;
        _metricsService = metricsService;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await MonitorEnvironmentHealthAsync();
                
                // Wait for next monitoring cycle
                var interval = _configuration.GetValue<int>("EnvironmentMonitoring:IntervalSeconds", 60);
                await Task.Delay(TimeSpan.FromSeconds(interval), stoppingToken);
            }
            catch (OperationCanceledException)
            {
                // Service is stopping
                break;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error in environment health monitoring");
                await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
            }
        }
    }
    
    private async Task MonitorEnvironmentHealthAsync()
    {
        var environment = _environmentProvider.GetCurrentEnvironment();
        var startTime = DateTime.UtcNow;
        
        _logger.LogDebug("Starting environment health monitoring for {Environment}", environment);
        
        try
        {
            // Run health checks
            var healthReport = await _healthCheckService.CheckHealthAsync();
            
            // Record metrics
            await RecordHealthMetricsAsync(environment, healthReport, startTime);
            
            // Check for critical issues
            await CheckForCriticalIssuesAsync(environment, healthReport);
            
            // Update environment status
            await UpdateEnvironmentStatusAsync(environment, healthReport);
            
            _logger.LogDebug("Environment health monitoring completed for {Environment}", environment);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error monitoring environment health for {Environment}", environment);
            
            // Record failure metric
            await _metricsService.RecordCounterAsync("environment_health_check_failures", 1, 
                new Dictionary<string, object> { ["environment"] = environment });
        }
    }
    
    private async Task RecordHealthMetricsAsync(string environment, HealthReport healthReport, DateTime startTime)
    {
        var duration = DateTime.UtcNow - startTime;
        
        // Record health check duration
        await _metricsService.RecordHistogramAsync("environment_health_check_duration", duration.TotalMilliseconds,
            new Dictionary<string, object> { ["environment"] = environment });
        
        // Record health status
        var healthyChecks = healthReport.Entries.Count(e => e.Value.Status == HealthStatus.Healthy);
        var unhealthyChecks = healthReport.Entries.Count(e => e.Value.Status == HealthStatus.Unhealthy);
        var degradedChecks = healthReport.Entries.Count(e => e.Value.Status == HealthStatus.Degraded);
        
        await _metricsService.RecordGaugeAsync("environment_health_checks_total", healthReport.Entries.Count,
            new Dictionary<string, object> { ["environment"] = environment });
        await _metricsService.RecordGaugeAsync("environment_health_checks_healthy", healthyChecks,
            new Dictionary<string, object> { ["environment"] = environment });
        await _metricsService.RecordGaugeAsync("environment_health_checks_unhealthy", unhealthyChecks,
            new Dictionary<string, object> { ["environment"] = environment });
        await _metricsService.RecordGaugeAsync("environment_health_checks_degraded", degradedChecks,
            new Dictionary<string, object> { ["environment"] = environment });
        
        // Record overall health status
        var overallHealth = healthReport.Status == HealthStatus.Healthy ? 1 : 0;
        await _metricsService.RecordGaugeAsync("environment_overall_health", overallHealth,
            new Dictionary<string, object> { ["environment"] = environment });
    }
    
    private async Task CheckForCriticalIssuesAsync(string environment, HealthReport healthReport)
    {
        var criticalIssues = healthReport.Entries
            .Where(e => e.Value.Status == HealthStatus.Unhealthy && 
                       IsCriticalHealthCheck(e.Key))
            .ToList();
        
        if (criticalIssues.Any())
        {
            _logger.LogWarning("Critical health issues detected in environment {Environment}: {Issues}",
                environment, string.Join(", ", criticalIssues.Select(i => i.Key)));
            
            // Send alert
            await SendCriticalIssueAlertAsync(environment, criticalIssues);
            
            // Record critical issue metric
            await _metricsService.RecordCounterAsync("environment_critical_issues", criticalIssues.Count,
                new Dictionary<string, object> { ["environment"] = environment });
        }
    }
    
    private bool IsCriticalHealthCheck(string healthCheckName)
    {
        var criticalChecks = new[]
        {
            "database",
            "redis",
            "external_api",
            "disk_space"
        };
        
        return criticalChecks.Any(check => healthCheckName.Contains(check, StringComparison.OrdinalIgnoreCase));
    }
    
    private async Task SendCriticalIssueAlertAsync(string environment, List<KeyValuePair<string, HealthReportEntry>> criticalIssues)
    {
        try
        {
            var alertMessage = new
            {
                Environment = environment,
                Timestamp = DateTime.UtcNow,
                CriticalIssues = criticalIssues.Select(i => new
                {
                    HealthCheck = i.Key,
                    Status = i.Value.Status.ToString(),
                    Description = i.Value.Description,
                    Duration = i.Value.Duration
                }).ToArray()
            };
            
            // Send to configured alerting service (Slack, Teams, etc.)
            var webhookUrl = _configuration["Alerting:WebhookUrl"];
            if (!string.IsNullOrEmpty(webhookUrl))
            {
                using var client = new HttpClient();
                var json = JsonSerializer.Serialize(alertMessage);
                var content = new StringContent(json, Encoding.UTF8, "application/json");
                
                var response = await client.PostAsync(webhookUrl, content);
                if (!response.IsSuccessStatusCode)
                {
                    _logger.LogWarning("Failed to send critical issue alert. Status: {Status}", response.StatusCode);
                }
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error sending critical issue alert");
        }
    }
    
    private async Task UpdateEnvironmentStatusAsync(string environment, HealthReport healthReport)
    {
        try
        {
            var status = new EnvironmentStatus
            {
                EnvironmentName = environment,
                LastChecked = DateTime.UtcNow,
                OverallStatus = healthReport.Status.ToString(),
                HealthyChecks = healthReport.Entries.Count(e => e.Value.Status == HealthStatus.Healthy),
                UnhealthyChecks = healthReport.Entries.Count(e => e.Value.Status == HealthStatus.Unhealthy),
                DegradedChecks = healthReport.Entries.Count(e => e.Value.Status == HealthStatus.Degraded),
                TotalChecks = healthReport.Entries.Count,
                ResponseTime = healthReport.TotalDuration.TotalMilliseconds
            };
            
            // Store status in cache or database
            await StoreEnvironmentStatusAsync(status);
            
            _logger.LogDebug("Environment status updated for {Environment}: {Status}", 
                environment, status.OverallStatus);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error updating environment status");
        }
    }
    
    private async Task StoreEnvironmentStatusAsync(EnvironmentStatus status)
    {
        // Implementation to store status in cache or database
        await Task.CompletedTask;
    }
}

public class EnvironmentStatus
{
    public string EnvironmentName { get; set; }
    public DateTime LastChecked { get; set; }
    public string OverallStatus { get; set; }
    public int HealthyChecks { get; set; }
    public int UnhealthyChecks { get; set; }
    public int DegradedChecks { get; set; }
    public int TotalChecks { get; set; }
    public double ResponseTime { get; set; }
}
```

## Mülakat Soruları

### Temel Sorular

1. **Environment Management nedir ve neden önemlidir?**
   - **Cevap**: Farklı ortamların yapılandırılması, tutarlılık, güvenlik, deployment reliability için kritik.

2. **Development, Staging, Production ortamları arasındaki farklar nelerdir?**
   - **Cevap**: Development (geliştirme), Staging (test), Production (canlı). Her biri farklı configuration ve security level.

3. **Configuration management nasıl yapılır?**
   - **Cevap**: Environment-specific config files, secrets management, configuration validation, centralized configuration.

4. **Infrastructure as Code nedir?**
   - **Cevap**: Infrastructure'ı kod ile tanımlama, version control, automation, consistency, repeatability.

5. **Environment monitoring neden önemlidir?**
   - **Cevap**: Health tracking, issue detection, performance monitoring, alerting, proactive problem solving.

### Teknik Sorular

1. **Environment-specific configuration nasıl implement edilir?**
   - **Cevap**: Configuration providers, environment variables, appsettings files, secrets management.

2. **Terraform ile environment management nasıl yapılır?**
   - **Cevap**: Variable files, modules, state management, environment separation, resource tagging.

3. **Secrets management nasıl implement edilir?**
   - **Cevap**: Key Vault, environment variables, configuration providers, encryption, access control.

4. **Environment health monitoring nasıl yapılır?**
   - **Cevap**: Health checks, metrics collection, alerting, status tracking, response time monitoring.

5. **Environment deployment automation nasıl yapılır?**
   - **Cevap**: CI/CD pipelines, infrastructure provisioning, configuration management, health validation.

## Best Practices

1. **Configuration Management**
   - Environment-specific config files kullanın
   - Secrets management implement edin
   - Configuration validation yapın
   - Centralized configuration sağlayın

2. **Infrastructure as Code**
   - Version control kullanın
   - Environment separation yapın
   - Resource tagging implement edin
   - State management optimize edin

3. **Security & Compliance**
   - Least privilege principle uygulayın
   - Secrets encryption yapın
   - Access control implement edin
   - Audit logging sağlayın

4. **Monitoring & Alerting**
   - Health checks implement edin
   - Metrics collection yapın
   - Proactive alerting kurun
   - Performance monitoring sağlayın

5. **Automation & Deployment**
   - CI/CD pipelines kullanın
   - Infrastructure automation yapın
   - Environment provisioning automate edin
   - Rollback capability sağlayın

## Kaynaklar

- [Microsoft Environment Configuration](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/)
- [Terraform Best Practices](https://www.terraform.io/docs/cloud/guides/recommended-practices/)
- [Infrastructure as Code](https://www.hashicorp.com/resources/infrastructure-as-code)
- [Environment Management](https://cloud.google.com/architecture/devops-environment-management)
- [Configuration Management](https://12factor.net/config)
