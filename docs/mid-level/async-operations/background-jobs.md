# Background Jobs

## Giriş

Background Jobs (Arka Plan İşleri), uygulamanın ana iş akışından bağımsız olarak çalışan, genellikle zaman uyumsuz (asynchronous) ve uzun süren işlemlerdir. .NET uygulamalarında e-posta gönderimi, rapor oluşturma, veri işleme gibi işlemler için kullanılır.

## Background Jobs'ın Önemi

1. **Performans**
   - Ana iş akışını bloklamama
   - Kaynak kullanımı optimizasyonu
   - Yük dengeleme
   - Ölçeklenebilirlik

2. **Güvenilirlik**
   - Hata yönetimi
   - Yeniden deneme mekanizmaları
   - İş durumu takibi
   - İşlem izlenebilirliği

3. **Kullanıcı Deneyimi**
   - Hızlı yanıt süreleri
   - Asenkron işlem bildirimleri
   - İşlem durumu takibi
   - Kullanıcı geri bildirimi

## Background Jobs Araçları

1. **Hangfire**
   - Kolay kurulum
   - Zengin özellik seti
   - Dashboard
   - Persistence desteği

2. **Quartz.NET**
   - Güçlü zamanlama
   - Cluster desteği
   - İşlem yönetimi
   - Plugin mimarisi

3. **Azure WebJobs**
   - Azure entegrasyonu
   - Otomatik ölçeklendirme
   - Monitoring
   - Logging

## Background Jobs Kullanımı

1. **Hangfire Kurulumu**
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

2. **Background Job Oluşturma**
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

3. **Zamanlanmış Job**
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

4. **Job İzleme**
```csharp
public class JobMonitor
{
    private readonly IMonitoringApi _monitoringApi;

    public JobMonitor(IMonitoringApi monitoringApi)
    {
        _monitoringApi = monitoringApi;
    }

    public async Task<JobStatus> GetJobStatus(string jobId)
    {
        var job = await _monitoringApi.JobDetails(jobId);
        return new JobStatus
        {
            State = job.History[0].StateName,
            CreatedAt = job.CreatedAt,
            StartedAt = job.StartedAt,
            CompletedAt = job.CompletedAt
        };
    }
}
```

## Background Jobs Best Practices

1. **Job Tasarımı**
   - Idempotent işlemler
   - Küçük ve odaklı joblar
   - Hata yönetimi
   - İşlem durumu takibi

2. **Performans**
   - Kaynak kullanımı optimizasyonu
   - Batch işlemler
   - Ölçeklendirme stratejileri
   - Queue yönetimi

3. **Güvenlik**
   - Erişim kontrolü
   - Veri güvenliği
   - Audit logging
   - Job doğrulama

4. **Monitoring**
   - Job durumu takibi
   - Hata izleme
   - Performans metrikleri
   - Alerting

## Mülakat Soruları

### Temel Sorular

1. **Background Jobs nedir ve neden kullanılır?**
   - **Cevap**: Background Jobs, uygulamanın ana iş akışından bağımsız olarak çalışan, genellikle zaman uyumsuz ve uzun süren işlemlerdir. E-posta gönderimi, rapor oluşturma, veri işleme gibi işlemler için kullanılır.

2. **Popüler Background Jobs araçları nelerdir?**
   - **Cevap**:
     - Hangfire
     - Quartz.NET
     - Azure WebJobs
     - MassTransit
     - NServiceBus

3. **Background Jobs'ın avantajları nelerdir?**
   - **Cevap**:
     - Performans optimizasyonu
     - Güvenilirlik
     - Kullanıcı deneyimi
     - Ölçeklenebilirlik
     - Kaynak yönetimi

4. **Idempotent işlem nedir?**
   - **Cevap**: Idempotent işlem, aynı işlemin birden fazla kez yapılmasının sonucu değiştirmemesidir. Background Jobs'da önemli bir kavramdır.

5. **Job durumu nedir?**
   - **Cevap**: Job durumu, bir background job'ın yaşam döngüsündeki mevcut durumunu temsil eder (örn. Scheduled, Processing, Succeeded, Failed).

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

2. **Background job nasıl oluşturulur?**
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

3. **Zamanlanmış job nasıl oluşturulur?**
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

4. **Job izleme nasıl yapılır?**
   - **Cevap**:
```csharp
public class JobMonitor
{
    private readonly IMonitoringApi _monitoringApi;

    public JobMonitor(IMonitoringApi monitoringApi)
    {
        _monitoringApi = monitoringApi;
    }

    public async Task<JobStatus> GetJobStatus(string jobId)
    {
        var job = await _monitoringApi.JobDetails(jobId);
        return new JobStatus
        {
            State = job.History[0].StateName,
            CreatedAt = job.CreatedAt,
            StartedAt = job.StartedAt,
            CompletedAt = job.CompletedAt
        };
    }
}
```

5. **Job hata yönetimi nasıl yapılır?**
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
        
        // Yeniden deneme stratejisi
        if (ex is TransientException)
        {
            _backgroundJobClient.Schedule(() => RetryJob(jobId), TimeSpan.FromMinutes(5));
        }
    }
}
```

### İleri Seviye Sorular

1. **Background Jobs performans optimizasyonu nasıl yapılır?**
   - **Cevap**:
     - Job parçalama
     - Batch işlemler
     - Queue yönetimi
     - Kaynak optimizasyonu
     - Ölçeklendirme stratejileri

2. **Background Jobs güvenliği nasıl sağlanır?**
   - **Cevap**:
     - Erişim kontrolü
     - Veri şifreleme
     - Job doğrulama
     - Audit logging
     - Güvenlik izleme

3. **Background Jobs ile distributed sistemler nasıl yönetilir?**
   - **Cevap**:
     - Cluster yapılandırması
     - Job dağıtımı
     - Load balancing
     - Failover stratejileri
     - Consistency yönetimi

4. **Background Jobs ile monitoring nasıl yapılır?**
   - **Cevap**:
     - Job durumu takibi
     - Performans metrikleri
     - Hata izleme
     - Alerting
     - Dashboard tasarımı

5. **Background Jobs ile scaling nasıl yapılır?**
   - **Cevap**:
     - Horizontal scaling
     - Vertical scaling
     - Auto-scaling
     - Load balancing
     - Resource allocation 