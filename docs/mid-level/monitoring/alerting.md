# Alerting

## Giriş

Alerting, sistem durumu, performans ve business metrics'lerde anormal durumları tespit eden ve ilgili kişilere bildirim gönderen sistemdir. Mid-level geliştiriciler için alerting'i anlamak, proactive monitoring, incident response ve system reliability için kritik öneme sahiptir. Bu dosya, alert rules, notification systems, escalation procedures ve alert management konularını kapsar.

## Alert Rules Engine

### 1. Alert Rule Engine
Alert kurallarını yöneten ve değerlendiren engine.

```csharp
public class AlertRuleEngine : IAlertRuleEngine
{
    private readonly ILogger<AlertRuleEngine> _logger;
    private readonly IConfiguration _configuration;
    private readonly IAlertRuleRepository _ruleRepository;
    private readonly IAlertNotificationService _notificationService;
    private readonly Dictionary<string, AlertRule> _activeRules;
    private readonly Timer _evaluationTimer;
    
    public AlertRuleEngine(ILogger<AlertRuleEngine> logger, IConfiguration configuration,
        IAlertRuleRepository ruleRepository, IAlertNotificationService notificationService)
    {
        _logger = logger;
        _configuration = configuration;
        _ruleRepository = ruleRepository;
        _notificationService = notificationService;
        _activeRules = new Dictionary<string, AlertRule>();
        
        var evaluationInterval = _configuration.GetValue<int>("Alerting:EvaluationIntervalSeconds", 30);
        _evaluationTimer = new Timer(EvaluateRules, null, TimeSpan.Zero, TimeSpan.FromSeconds(evaluationInterval));
    }
    
    public async Task InitializeAsync()
    {
        try
        {
            var rules = await _ruleRepository.GetActiveRulesAsync();
            
            foreach (var rule in rules)
            {
                _activeRules[rule.Id] = rule;
                _logger.LogInformation("Loaded alert rule: {RuleName}", rule.Name);
            }
            
            _logger.LogInformation("Alert rule engine initialized with {Count} rules", _activeRules.Count);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error initializing alert rule engine");
        }
    }
    
    public async Task AddRuleAsync(AlertRule rule)
    {
        try
        {
            await _ruleRepository.SaveRuleAsync(rule);
            _activeRules[rule.Id] = rule;
            
            _logger.LogInformation("Alert rule added: {RuleName}", rule.Name);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error adding alert rule: {RuleName}", rule.Name);
        }
    }
    
    public async Task UpdateRuleAsync(AlertRule rule)
    {
        try
        {
            await _ruleRepository.SaveRuleAsync(rule);
            _activeRules[rule.Id] = rule;
            
            _logger.LogInformation("Alert rule updated: {RuleName}", rule.Name);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error updating alert rule: {RuleName}", rule.Name);
        }
    }
    
    public async Task RemoveRuleAsync(string ruleId)
    {
        try
        {
            await _ruleRepository.DeleteRuleAsync(ruleId);
            _activeRules.Remove(ruleId);
            
            _logger.LogInformation("Alert rule removed: {RuleId}", ruleId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error removing alert rule: {RuleId}", ruleId);
        }
    }
    
    private async void EvaluateRules(object state)
    {
        try
        {
            foreach (var rule in _activeRules.Values)
            {
                if (rule.IsEnabled && ShouldEvaluateRule(rule))
                {
                    await EvaluateRuleAsync(rule);
                }
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error evaluating alert rules");
        }
    }
    
    private bool ShouldEvaluateRule(AlertRule rule)
    {
        if (rule.LastEvaluation.HasValue)
        {
            var timeSinceLastEvaluation = DateTime.UtcNow - rule.LastEvaluation.Value;
            return timeSinceLastEvaluation >= rule.EvaluationInterval;
        }
        
        return true;
    }
    
    private async Task EvaluateRuleAsync(AlertRule rule)
    {
        try
        {
            var evaluationResult = await EvaluateConditionAsync(rule.Condition);
            
            if (evaluationResult.IsTriggered)
            {
                await HandleAlertTriggeredAsync(rule, evaluationResult);
            }
            else if (rule.IsActive)
            {
                await HandleAlertResolvedAsync(rule, evaluationResult);
            }
            
            // Update last evaluation time
            rule.LastEvaluation = DateTime.UtcNow;
            await _ruleRepository.SaveRuleAsync(rule);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error evaluating rule: {RuleName}", rule.Name);
        }
    }
    
    private async Task<AlertEvaluationResult> EvaluateConditionAsync(AlertCondition condition)
    {
        try
        {
            var result = new AlertEvaluationResult();
            
            switch (condition.Type)
            {
                case AlertConditionType.Threshold:
                    result = await EvaluateThresholdConditionAsync(condition);
                    break;
                case AlertConditionType.Anomaly:
                    result = await EvaluateAnomalyConditionAsync(condition);
                    break;
                case AlertConditionType.Trend:
                    result = await EvaluateTrendConditionAsync(condition);
                    break;
                case AlertConditionType.Custom:
                    result = await EvaluateCustomConditionAsync(condition);
                    break;
            }
            
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error evaluating alert condition");
            return new AlertEvaluationResult { IsTriggered = false, Error = ex.Message };
        }
    }
    
    private async Task<AlertEvaluationResult> EvaluateThresholdConditionAsync(AlertCondition condition)
    {
        var result = new AlertEvaluationResult();
        
        try
        {
            var currentValue = await GetMetricValueAsync(condition.MetricName);
            var threshold = condition.Threshold;
            var operatorType = condition.Operator;
            
            bool isTriggered = operatorType switch
            {
                AlertOperator.GreaterThan => currentValue > threshold,
                AlertOperator.GreaterThanOrEqual => currentValue >= threshold,
                AlertOperator.LessThan => currentValue < threshold,
                AlertOperator.LessThanOrEqual => currentValue <= threshold,
                AlertOperator.Equal => Math.Abs(currentValue - threshold) < 0.001,
                AlertOperator.NotEqual => Math.Abs(currentValue - threshold) >= 0.001,
                _ => false
            };
            
            result.IsTriggered = isTriggered;
            result.CurrentValue = currentValue;
            result.Threshold = threshold;
            result.Operator = operatorType;
            
            if (isTriggered)
            {
                result.Message = $"Metric {condition.MetricName} ({currentValue}) {operatorType} {threshold}";
            }
        }
        catch (Exception ex)
        {
            result.Error = ex.Message;
        }
        
        return result;
    }
    
    private async Task<double> GetMetricValueAsync(string metricName)
    {
        // This would typically integrate with your metrics collection system
        // For demonstration, return a random value
        await Task.Delay(10);
        return new Random().NextDouble() * 100;
    }
    
    private async Task HandleAlertTriggeredAsync(AlertRule rule, AlertEvaluationResult result)
    {
        try
        {
            if (!rule.IsActive)
            {
                rule.IsActive = true;
                rule.TriggeredAt = DateTime.UtcNow;
                rule.TriggerCount++;
                
                await _ruleRepository.SaveRuleAsync(rule);
                
                var alert = new Alert
                {
                    Id = Guid.NewGuid().ToString(),
                    RuleId = rule.Id,
                    RuleName = rule.Name,
                    Severity = rule.Severity,
                    Message = result.Message,
                    CurrentValue = result.CurrentValue,
                    Threshold = result.Threshold,
                    TriggeredAt = DateTime.UtcNow,
                    Status = AlertStatus.Active
                };
                
                await _notificationService.SendAlertAsync(alert);
                
                _logger.LogWarning("Alert triggered: {RuleName}, Message: {Message}", rule.Name, result.Message);
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error handling alert trigger: {RuleName}", rule.Name);
        }
    }
    
    private async Task HandleAlertResolvedAsync(AlertRule rule, AlertEvaluationResult result)
    {
        try
        {
            if (rule.IsActive)
            {
                rule.IsActive = false;
                rule.ResolvedAt = DateTime.UtcNow;
                
                await _ruleRepository.SaveRuleAsync(rule);
                
                var resolution = new AlertResolution
                {
                    AlertId = rule.Id,
                    RuleName = rule.Name,
                    ResolvedAt = DateTime.UtcNow,
                    ResolutionMessage = "Alert condition no longer met"
                };
                
                await _notificationService.SendResolutionAsync(resolution);
                
                _logger.LogInformation("Alert resolved: {RuleName}", rule.Name);
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error handling alert resolution: {RuleName}", rule.Name);
        }
    }
    
    public void Dispose()
    {
        _evaluationTimer?.Dispose();
    }
}

public interface IAlertRuleEngine
{
    Task InitializeAsync();
    Task AddRuleAsync(AlertRule rule);
    Task UpdateRuleAsync(AlertRule rule);
    Task RemoveRuleAsync(string ruleId);
}

public class AlertRule
{
    public string Id { get; set; } = Guid.NewGuid().ToString();
    public string Name { get; set; }
    public string Description { get; set; }
    public AlertSeverity Severity { get; set; }
    public AlertCondition Condition { get; set; }
    public TimeSpan EvaluationInterval { get; set; }
    public bool IsEnabled { get; set; } = true;
    public bool IsActive { get; set; } = false;
    public DateTime? TriggeredAt { get; set; }
    public DateTime? ResolvedAt { get; set; }
    public DateTime? LastEvaluation { get; set; }
    public int TriggerCount { get; set; } = 0;
    public List<string> NotificationChannels { get; set; } = new();
    public List<string> EscalationRules { get; set; } = new();
}

public class AlertCondition
{
    public AlertConditionType Type { get; set; }
    public string MetricName { get; set; }
    public double Threshold { get; set; }
    public AlertOperator Operator { get; set; }
    public TimeSpan? TimeWindow { get; set; }
    public string CustomExpression { get; set; }
}

public class AlertEvaluationResult
{
    public bool IsTriggered { get; set; }
    public double CurrentValue { get; set; }
    public double Threshold { get; set; }
    public AlertOperator Operator { get; set; }
    public string Message { get; set; }
    public string Error { get; set; }
}

public enum AlertConditionType
{
    Threshold,
    Anomaly,
    Trend,
    Custom
}

public enum AlertOperator
{
    GreaterThan,
    GreaterThanOrEqual,
    LessThan,
    LessThanOrEqual,
    Equal,
    NotEqual
}

public enum AlertSeverity
{
    Low,
    Medium,
    High,
    Critical
}
```

### 2. Notification Service
Alert bildirimlerini gönderen servis.

```csharp
public class AlertNotificationService : IAlertNotificationService
{
    private readonly ILogger<AlertNotificationService> _logger;
    private readonly IConfiguration _configuration;
    private readonly IEmailService _emailService;
    private readonly ISlackService _slackService;
    private readonly ISmsService _smsService;
    private readonly IWebhookService _webhookService;
    
    public AlertNotificationService(ILogger<AlertNotificationService> logger, IConfiguration configuration,
        IEmailService emailService, ISlackService slackService, ISmsService smsService, IWebhookService webhookService)
    {
        _logger = logger;
        _configuration = configuration;
        _emailService = emailService;
        _slackService = slackService;
        _smsService = smsService;
        _webhookService = webhookService;
    }
    
    public async Task SendAlertAsync(Alert alert)
    {
        try
        {
            _logger.LogInformation("Sending alert notification: {RuleName}, Severity: {Severity}", 
                alert.RuleName, alert.Severity);
            
            var notificationTasks = new List<Task>();
            
            // Send to all configured channels
            if (ShouldSendEmail(alert))
            {
                notificationTasks.Add(SendEmailAlertAsync(alert));
            }
            
            if (ShouldSendSlack(alert))
            {
                notificationTasks.Add(SendSlackAlertAsync(alert));
            }
            
            if (ShouldSendSms(alert))
            {
                notificationTasks.Add(SendSmsAlertAsync(alert));
            }
            
            if (ShouldSendWebhook(alert))
            {
                notificationTasks.Add(SendWebhookAlertAsync(alert));
            }
            
            await Task.WhenAll(notificationTasks);
            
            _logger.LogInformation("Alert notification sent successfully: {RuleName}", alert.RuleName);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error sending alert notification: {RuleName}", alert.RuleName);
        }
    }
    
    public async Task SendResolutionAsync(AlertResolution resolution)
    {
        try
        {
            _logger.LogInformation("Sending resolution notification: {RuleName}", resolution.RuleName);
            
            var notificationTasks = new List<Task>();
            
            if (ShouldSendEmail(null))
            {
                notificationTasks.Add(SendEmailResolutionAsync(resolution));
            }
            
            if (ShouldSendSlack(null))
            {
                notificationTasks.Add(SendSlackResolutionAsync(resolution));
            }
            
            await Task.WhenAll(notificationTasks);
            
            _logger.LogInformation("Resolution notification sent successfully: {RuleName}", resolution.RuleName);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error sending resolution notification: {RuleName}", resolution.RuleName);
        }
    }
    
    private bool ShouldSendEmail(Alert alert)
    {
        var emailEnabled = _configuration.GetValue<bool>("Notifications:Email:Enabled", true);
        var severityThreshold = _configuration.GetValue<AlertSeverity>("Notifications:Email:MinSeverity", AlertSeverity.Low);
        
        return emailEnabled && (alert?.Severity >= severityThreshold || alert == null);
    }
    
    private bool ShouldSendSlack(Alert alert)
    {
        var slackEnabled = _configuration.GetValue<bool>("Notifications:Slack:Enabled", true);
        var severityThreshold = _configuration.GetValue<AlertSeverity>("Notifications:Slack:MinSeverity", AlertSeverity.Medium);
        
        return slackEnabled && (alert?.Severity >= severityThreshold || alert == null);
    }
    
    private bool ShouldSendSms(Alert alert)
    {
        var smsEnabled = _configuration.GetValue<bool>("Notifications:Sms:Enabled", false);
        var severityThreshold = _configuration.GetValue<AlertSeverity>("Notifications:Sms:MinSeverity", AlertSeverity.High);
        
        return smsEnabled && alert?.Severity >= severityThreshold;
    }
    
    private bool ShouldSendWebhook(Alert alert)
    {
        var webhookEnabled = _configuration.GetValue<bool>("Notifications:Webhook:Enabled", false);
        var severityThreshold = _configuration.GetValue<AlertSeverity>("Notifications:Webhook:MinSeverity", AlertSeverity.Medium);
        
        return webhookEnabled && (alert?.Severity >= severityThreshold || alert == null);
    }
    
    private async Task SendEmailAlertAsync(Alert alert)
    {
        try
        {
            var subject = $"[{alert.Severity}] Alert: {alert.RuleName}";
            var body = GenerateEmailBody(alert);
            var recipients = GetEmailRecipients(alert.Severity);
            
            await _emailService.SendEmailAsync(recipients, subject, body);
            
            _logger.LogDebug("Email alert sent: {RuleName}", alert.RuleName);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error sending email alert: {RuleName}", alert.RuleName);
        }
    }
    
    private async Task SendSlackAlertAsync(Alert alert)
    {
        try
        {
            var message = GenerateSlackMessage(alert);
            var channel = GetSlackChannel(alert.Severity);
            
            await _slackService.SendMessageAsync(channel, message);
            
            _logger.LogDebug("Slack alert sent: {RuleName}", alert.RuleName);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error sending Slack alert: {RuleName}", alert.RuleName);
        }
    }
    
    private async Task SendSmsAlertAsync(Alert alert)
    {
        try
        {
            var message = GenerateSmsMessage(alert);
            var recipients = GetSmsRecipients(alert.Severity);
            
            foreach (var recipient in recipients)
            {
                await _smsService.SendSmsAsync(recipient, message);
            }
            
            _logger.LogDebug("SMS alert sent: {RuleName}", alert.RuleName);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error sending SMS alert: {RuleName}", alert.RuleName);
        }
    }
    
    private async Task SendWebhookAlertAsync(Alert alert)
    {
        try
        {
            var payload = GenerateWebhookPayload(alert);
            var webhookUrl = _configuration["Notifications:Webhook:Url"];
            
            await _webhookService.SendWebhookAsync(webhookUrl, payload);
            
            _logger.LogDebug("Webhook alert sent: {RuleName}", alert.RuleName);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error sending webhook alert: {RuleName}", alert.RuleName);
        }
    }
    
    private string GenerateEmailBody(Alert alert)
    {
        return $@"
Alert Details:
==============
Rule Name: {alert.RuleName}
Severity: {alert.Severity}
Message: {alert.Message}
Current Value: {alert.CurrentValue}
Threshold: {alert.Threshold}
Triggered At: {alert.TriggeredAt:yyyy-MM-dd HH:mm:ss UTC}

Please investigate this issue immediately.
";
    }
    
    private string GenerateSlackMessage(Alert alert)
    {
        var emoji = alert.Severity switch
        {
            AlertSeverity.Low => ":information_source:",
            AlertSeverity.Medium => ":warning:",
            AlertSeverity.High => ":rotating_light:",
            AlertSeverity.Critical => ":fire:",
            _ => ":warning:"
        };
        
        return $"{emoji} *[{alert.Severity}] Alert: {alert.RuleName}*\n{alert.Message}\nTriggered at {alert.TriggeredAt:HH:mm:ss UTC}";
    }
    
    private string GenerateSmsMessage(Alert alert)
    {
        return $"ALERT: {alert.RuleName} - {alert.Message}";
    }
    
    private object GenerateWebhookPayload(Alert alert)
    {
        return new
        {
            alert_type = "alert",
            rule_name = alert.RuleName,
            severity = alert.Severity.ToString(),
            message = alert.Message,
            current_value = alert.CurrentValue,
            threshold = alert.Threshold,
            triggered_at = alert.TriggeredAt,
            status = alert.Status.ToString()
        };
    }
    
    private List<string> GetEmailRecipients(AlertSeverity severity)
    {
        var baseRecipients = _configuration.GetSection("Notifications:Email:Recipients").Get<string[]>() ?? new string[0];
        var severityRecipients = _configuration.GetSection($"Notifications:Email:SeverityRecipients:{severity}").Get<string[]>() ?? new string[0];
        
        return baseRecipients.Concat(severityRecipients).Distinct().ToList();
    }
    
    private string GetSlackChannel(AlertSeverity severity)
    {
        return _configuration[$"Notifications:Slack:Channels:{severity}"] ?? 
               _configuration["Notifications:Slack:DefaultChannel"] ?? 
               "#alerts";
    }
    
    private List<string> GetSmsRecipients(AlertSeverity severity)
    {
        return _configuration.GetSection($"Notifications:Sms:Recipients:{severity}").Get<string[]>()?.ToList() ?? new List<string>();
    }
    
    // Resolution notification methods
    private async Task SendEmailResolutionAsync(AlertResolution resolution)
    {
        var subject = $"Resolved: {resolution.RuleName}";
        var body = $"Alert {resolution.RuleName} has been resolved at {resolution.ResolvedAt:yyyy-MM-dd HH:mm:ss UTC}";
        var recipients = GetEmailRecipients(AlertSeverity.Low);
        
        await _emailService.SendEmailAsync(recipients, subject, body);
    }
    
    private async Task SendSlackResolutionAsync(AlertResolution resolution)
    {
        var message = $":white_check_mark: *Resolved: {resolution.RuleName}*\nAlert has been resolved at {resolution.ResolvedAt:HH:mm:ss UTC}";
        var channel = GetSlackChannel(AlertSeverity.Low);
        
        await _slackService.SendMessageAsync(channel, message);
    }
}

public interface IAlertNotificationService
{
    Task SendAlertAsync(Alert alert);
    Task SendResolutionAsync(AlertResolution resolution);
}

public class Alert
{
    public string Id { get; set; }
    public string RuleId { get; set; }
    public string RuleName { get; set; }
    public AlertSeverity Severity { get; set; }
    public string Message { get; set; }
    public double CurrentValue { get; set; }
    public double Threshold { get; set; }
    public DateTime TriggeredAt { get; set; }
    public AlertStatus Status { get; set; }
}

public class AlertResolution
{
    public string AlertId { get; set; }
    public string RuleName { get; set; }
    public DateTime ResolvedAt { get; set; }
    public string ResolutionMessage { get; set; }
}

public enum AlertStatus
{
    Active,
    Acknowledged,
    Resolved
}
```

## Mülakat Soruları

### Temel Sorular

1. **Alerting nedir?**
   - **Cevap**: Anormal durumları tespit eden ve bildirim gönderen sistem.

2. **Alert rule nedir?**
   - **Cevap**: Alert koşullarını tanımlayan kurallar.

3. **Alert severity nedir?**
   - **Cevap**: Alert'in önem derecesi (Low, Medium, High, Critical).

4. **Escalation nedir?**
   - **Cevap**: Alert'e yanıt verilmediğinde üst seviyeye bildirim gönderme.

5. **Alert fatigue nedir?**
   - **Cevap**: Çok fazla alert'ten dolayı önemli olanları gözden kaçırma.

### Teknik Sorular

1. **Alert rules nasıl implement edilir?**
   - **Cevap**: Rule engine, condition evaluation, threshold checking.

2. **Notification channels nasıl yönetilir?**
   - **Cevap**: Email, Slack, SMS, Webhook, multiple providers.

3. **Alert correlation nasıl yapılır?**
   - **Cevap**: Related alerts grouping, root cause analysis.

4. **Alert suppression nasıl implement edilir?**
   - **Cevap**: Duplicate detection, time-based suppression.

5. **Alert dashboard nasıl tasarlanır?**
   - **Cevap**: Real-time monitoring, status overview, trend analysis.

## Best Practices

1. **Alert Design**
   - Meaningful thresholds belirleyin
   - False positive'leri minimize edin
   - Clear messages yazın
   - Appropriate severity levels kullanın

2. **Notification Management**
   - Multiple channels implement edin
   - Escalation procedures tanımlayın
   - Recipient groups organize edin
   - Quiet hours configure edin

3. **Performance Optimization**
   - Efficient evaluation implement edin
   - Rate limiting uygulayın
   - Caching strategies kullanın
   - Background processing yapın

4. **Alert Lifecycle**
   - Acknowledgment workflow implement edin
   - Resolution tracking yapın
   - Post-incident analysis ekleyin
   - Continuous improvement sağlayın

5. **Integration & Monitoring**
   - Metrics collection ekleyin
   - Performance monitoring yapın
   - Error handling implement edin
   - Testing strategies ekleyin

## Kaynaklar

- [Alerting Best Practices](https://docs.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-overview)
- [Alert Rules](https://docs.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-activity-log)
- [Notification Channels](https://docs.microsoft.com/en-us/azure/azure-monitor/alerts/action-groups)
- [Alert Management](https://docs.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-manage-alerts)
- [Escalation Procedures](https://docs.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-action-rules)
