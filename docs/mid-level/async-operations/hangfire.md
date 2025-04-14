# Hangfire

## Giriş

Hangfire, .NET uygulamaları için açık kaynaklı bir background job processing framework'üdür. Zamanlanmış işler, tekrarlayan görevler ve arka plan işlemleri için güvenilir ve ölçeklenebilir bir çözüm sunar.

## Hangfire'ın Önemi

1. **Güvenilirlik**
   - İşlem kalıcılığı
   - Otomatik yeniden deneme
   - Hata yönetimi
   - İşlem izlenebilirliği

2. **Ölçeklenebilirlik**
   - Çoklu sunucu desteği
   - Yük dengeleme
   - Queue yönetimi
   - Cluster desteği

3. **Kullanım Kolaylığı**
   - Kolay kurulum
   - Zengin API
   - Dashboard
   - Detaylı dokümantasyon

## Hangfire Özellikleri

1. **Job Tipleri**
   - Fire-and-forget jobs
   - Delayed jobs
   - Recurring jobs
   - Continuations
   - Batches

2. **Storage Seçenekleri**
   - SQL Server
   - Redis
   - MongoDB
   - PostgreSQL
   - MSMQ

3. **Monitoring**
   - Web Dashboard
   - Job durumu takibi
   - Performans metrikleri
   - Hata izleme

## Hangfire Kullanımı

1. **Temel Kurulum**
```csharp
// NuGet paketleri:
// Hangfire
// Hangfire.AspNetCore
// Hangfire.SqlServer

public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddHangfire(config => config
            .SetDataCompatibilityLevel(CompatibilityLevel.Version_170)
            .UseSimpleAssemblyNameTypeSerializer()
            .UseRecommendedSerializerSettings()
            .UseSqlServerStorage(Configuration.GetConnectionString("HangfireConnection")));

        services.AddHangfireServer();
    }

    public void Configure(IApplicationBuilder app)
    {
        app.UseHangfireDashboard();
    }
}
```

2. **Fire-and-Forget Job**
```csharp
public class EmailService
{
    private readonly IBackgroundJobClient _backgroundJobClient;

    public EmailService(IBackgroundJobClient backgroundJobClient)
    {
        _backgroundJobClient = backgroundJobClient;
    }

    public void SendEmail(string to, string subject, string body)
    {
        _backgroundJobClient.Enqueue(() => SendEmailAsync(to, subject, body));
    }

    public async Task SendEmailAsync(string to, string subject, string body)
    {
        // E-posta gönderme işlemi
        await Task.Delay(1000); // Simülasyon
    }
}
```

3. **Delayed Job**
```csharp
public class NotificationService
{
    private readonly IBackgroundJobClient _backgroundJobClient;

    public NotificationService(IBackgroundJobClient backgroundJobClient)
    {
        _backgroundJobClient = backgroundJobClient;
    }

    public void ScheduleNotification(string userId, string message, TimeSpan delay)
    {
        _backgroundJobClient.Schedule(
            () => SendNotificationAsync(userId, message),
            delay);
    }

    public async Task SendNotificationAsync(string userId, string message)
    {
        // Bildirim gönderme işlemi
        await Task.Delay(1000); // Simülasyon
    }
}
```

4. **Recurring Job**
```csharp
public class ReportService
{
    private readonly IRecurringJobManager _recurringJobManager;

    public ReportService(IRecurringJobManager recurringJobManager)
    {
        _recurringJobManager = recurringJobManager;
    }

    public void ScheduleDailyReport()
    {
        _recurringJobManager.AddOrUpdate(
            "daily-report",
            () => GenerateDailyReportAsync(),
            Cron.Daily);
    }

    public async Task GenerateDailyReportAsync()
    {
        // Günlük rapor oluşturma işlemi
        await Task.Delay(5000); // Simülasyon
    }
}
```

5. **Job Continuations**
```csharp
public class OrderService
{
    private readonly IBackgroundJobClient _backgroundJobClient;

    public OrderService(IBackgroundJobClient backgroundJobClient)
    {
        _backgroundJobClient = backgroundJobClient;
    }

    public void ProcessOrder(string orderId)
    {
        var jobId = _backgroundJobClient.Enqueue(() => ValidateOrderAsync(orderId));
        _backgroundJobClient.ContinueWith(jobId, () => ProcessPaymentAsync(orderId));
        _backgroundJobClient.ContinueWith(jobId, () => SendConfirmationAsync(orderId));
    }

    public async Task ValidateOrderAsync(string orderId)
    {
        // Sipariş doğrulama
        await Task.Delay(1000);
    }

    public async Task ProcessPaymentAsync(string orderId)
    {
        // Ödeme işlemi
        await Task.Delay(1000);
    }

    public async Task SendConfirmationAsync(string orderId)
    {
        // Onay gönderme
        await Task.Delay(1000);
    }
}
```

## Hangfire Best Practices

1. **Job Tasarımı**
   - Idempotent işlemler
   - Küçük ve odaklı joblar
   - Hata yönetimi
   - İşlem durumu takibi

2. **Performans**
   - Queue yönetimi
   - Worker sayısı optimizasyonu
   - Batch işlemler
   - Kaynak kullanımı

3. **Güvenlik**
   - Dashboard erişim kontrolü
   - Job doğrulama
   - Veri güvenliği
   - Audit logging

4. **Monitoring**
   - Job durumu takibi
   - Performans metrikleri
   - Hata izleme
   - Alerting

## Mülakat Soruları

### Temel Sorular

1. **Hangfire nedir ve neden kullanılır?**
   - **Cevap**: Hangfire, .NET uygulamaları için açık kaynaklı bir background job processing framework'üdür. Zamanlanmış işler, tekrarlayan görevler ve arka plan işlemleri için güvenilir ve ölçeklenebilir bir çözüm sunar.

2. **Hangfire'ın temel özellikleri nelerdir?**
   - **Cevap**:
     - Fire-and-forget jobs
     - Delayed jobs
     - Recurring jobs
     - Continuations
     - Batches
     - Web Dashboard
     - Çoklu storage desteği

3. **Hangfire'ın avantajları nelerdir?**
   - **Cevap**:
     - Güvenilirlik
     - Ölçeklenebilirlik
     - Kullanım kolaylığı
     - Zengin özellik seti
     - Detaylı dokümantasyon

4. **Hangfire'da job tipleri nelerdir?**
   - **Cevap**:
     - Fire-and-forget jobs: Hemen çalıştırılan işler
     - Delayed jobs: Belirli bir süre sonra çalıştırılan işler
     - Recurring jobs: Periyodik olarak çalıştırılan işler
     - Continuations: Başka bir job'ın tamamlanmasına bağlı işler
     - Batches: Toplu işlemler

5. **Hangfire'da storage seçenekleri nelerdir?**
   - **Cevap**:
     - SQL Server
     - Redis
     - MongoDB
     - PostgreSQL
     - MSMQ

### Teknik Sorular

1. **Hangfire nasıl yapılandırılır?**
   - **Cevap**:
```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddHangfire(config => config
            .SetDataCompatibilityLevel(CompatibilityLevel.Version_170)
            .UseSimpleAssemblyNameTypeSerializer()
            .UseRecommendedSerializerSettings()
            .UseSqlServerStorage(Configuration.GetConnectionString("HangfireConnection")));

        services.AddHangfireServer();
    }

    public void Configure(IApplicationBuilder app)
    {
        app.UseHangfireDashboard();
    }
}
```

2. **Fire-and-forget job nasıl oluşturulur?**
   - **Cevap**:
```csharp
public class EmailService
{
    private readonly IBackgroundJobClient _backgroundJobClient;

    public EmailService(IBackgroundJobClient backgroundJobClient)
    {
        _backgroundJobClient = backgroundJobClient;
    }

    public void SendEmail(string to, string subject, string body)
    {
        _backgroundJobClient.Enqueue(() => SendEmailAsync(to, subject, body));
    }
}
```

3. **Recurring job nasıl oluşturulur?**
   - **Cevap**:
```csharp
public class ReportService
{
    private readonly IRecurringJobManager _recurringJobManager;

    public ReportService(IRecurringJobManager recurringJobManager)
    {
        _recurringJobManager = recurringJobManager;
    }

    public void ScheduleDailyReport()
    {
        _recurringJobManager.AddOrUpdate(
            "daily-report",
            () => GenerateDailyReportAsync(),
            Cron.Daily);
    }
}
```

4. **Job continuations nasıl kullanılır?**
   - **Cevap**:
```csharp
public class OrderService
{
    private readonly IBackgroundJobClient _backgroundJobClient;

    public OrderService(IBackgroundJobClient backgroundJobClient)
    {
        _backgroundJobClient = backgroundJobClient;
    }

    public void ProcessOrder(string orderId)
    {
        var jobId = _backgroundJobClient.Enqueue(() => ValidateOrderAsync(orderId));
        _backgroundJobClient.ContinueWith(jobId, () => ProcessPaymentAsync(orderId));
        _backgroundJobClient.ContinueWith(jobId, () => SendConfirmationAsync(orderId));
    }
}
```

5. **Hangfire'da hata yönetimi nasıl yapılır?**
   - **Cevap**:
```csharp
public class JobErrorHandler
{
    private readonly ILogger _logger;
    private readonly IBackgroundJobClient _backgroundJobClient;

    public JobErrorHandler(ILogger<JobErrorHandler> logger, IBackgroundJobClient backgroundJobClient)
    {
        _logger = logger;
        _backgroundJobClient = backgroundJobClient;
    }

    public void HandleJobError(string jobId, Exception ex)
    {
        _logger.LogError(ex, "Job {JobId} failed", jobId);
        
        if (ex is TransientException)
        {
            _backgroundJobClient.Schedule(() => RetryJob(jobId), TimeSpan.FromMinutes(5));
        }
    }
}
```

### İleri Seviye Sorular

1. **Hangfire'da performans optimizasyonu nasıl yapılır?**
   - **Cevap**:
     - Queue yönetimi
     - Worker sayısı optimizasyonu
     - Batch işlemler
     - Kaynak kullanımı
     - Storage optimizasyonu

2. **Hangfire'da güvenlik nasıl sağlanır?**
   - **Cevap**:
     - Dashboard erişim kontrolü
     - Job doğrulama
     - Veri güvenliği
     - Audit logging
     - Güvenlik izleme

3. **Hangfire ile distributed sistemler nasıl yönetilir?**
   - **Cevap**:
     - Çoklu sunucu yapılandırması
     - Queue yönetimi
     - Load balancing
     - Failover stratejileri
     - Consistency yönetimi

4. **Hangfire'da monitoring nasıl yapılır?**
   - **Cevap**:
     - Web Dashboard kullanımı
     - Job durumu takibi
     - Performans metrikleri
     - Hata izleme
     - Alerting

5. **Hangfire'da scaling nasıl yapılır?**
   - **Cevap**:
     - Worker sayısı artırma
     - Queue sayısı artırma
     - Storage optimizasyonu
     - Load balancing
     - Resource allocation 