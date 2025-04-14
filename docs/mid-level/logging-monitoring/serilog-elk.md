# Serilog/ELK Stack

## Giriş

Serilog, .NET uygulamaları için yapılandırılabilir ve genişletilebilir bir logging framework'üdür. ELK Stack (Elasticsearch, Logstash, Kibana) ise log yönetimi ve analizi için kullanılan açık kaynaklı bir çözüm paketidir. Bu iki teknoloji birlikte kullanıldığında, uygulama loglarının toplanması, işlenmesi, depolanması ve görselleştirilmesi için güçlü bir altyapı sağlar.

## Serilog/ELK Stack'in Önemi

1. **Merkezi Log Yönetimi**
   - Tüm logların tek bir yerde toplanması
   - Kolay erişim ve analiz
   - Gerçek zamanlı izleme

2. **Gelişmiş Analiz**
   - Yapılandırılmış log verileri
   - Güçlü arama yetenekleri
   - Özelleştirilebilir dashboardlar

3. **Ölçeklenebilirlik**
   - Yüksek hacimli log yönetimi
   - Dağıtık mimari
   - Yük dengeleme

## Serilog Kullanımı

1. **Temel Kurulum**
```csharp
// NuGet paketleri:
// Serilog
// Serilog.Sinks.Console
// Serilog.Sinks.File
// Serilog.Sinks.Elasticsearch

public static class SerilogConfiguration
{
    public static IHostBuilder UseSerilog(this IHostBuilder hostBuilder)
    {
        return hostBuilder.UseSerilog((context, services, configuration) => configuration
            .ReadFrom.Configuration(context.Configuration)
            .ReadFrom.Services(services)
            .Enrich.FromLogContext()
            .WriteTo.Console()
            .WriteTo.File("logs/log-.txt", rollingInterval: RollingInterval.Day)
            .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(new Uri("http://localhost:9200"))
            {
                AutoRegisterTemplate = true,
                IndexFormat = "logstash-{0:yyyy.MM.dd}",
                NumberOfShards = 2,
                NumberOfReplicas = 1
            }));
    }
}
```

2. **Yapılandırılmış Logging**
```csharp
public class LoggingService
{
    private readonly ILogger<LoggingService> _logger;

    public LoggingService(ILogger<LoggingService> logger)
    {
        _logger = logger;
    }

    public void LogUserAction(string userId, string action, object details)
    {
        _logger.LogInformation("User {UserId} performed {Action} with details {@Details}", 
            userId, action, details);
    }

    public void LogError(Exception ex, string context)
    {
        _logger.LogError(ex, "Error occurred in {Context}", context);
    }

    public void LogPerformance(string operation, TimeSpan duration)
    {
        _logger.LogInformation("Operation {Operation} completed in {Duration}ms", 
            operation, duration.TotalMilliseconds);
    }
}
```

3. **Log Enrichment**
```csharp
public static class LogEnricher
{
    public static ILogger EnrichWithContext(this ILogger logger, string correlationId)
    {
        return logger.ForContext("CorrelationId", correlationId);
    }

    public static ILogger EnrichWithEnvironment(this ILogger logger)
    {
        return logger
            .ForContext("Environment", Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT"))
            .ForContext("MachineName", Environment.MachineName);
    }
}
```

## ELK Stack Kurulumu

1. **Elasticsearch Yapılandırması**
```yaml
# elasticsearch.yml
cluster.name: logging-cluster
node.name: node-1
network.host: 0.0.0.0
http.port: 9200
discovery.type: single-node
```

2. **Logstash Yapılandırması**
```conf
# logstash.conf
input {
  tcp {
    port => 5000
    codec => json
  }
}

filter {
  json {
    source => "message"
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "logstash-%{+YYYY.MM.dd}"
  }
}
```

3. **Kibana Yapılandırması**
```yaml
# kibana.yml
server.port: 5601
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://localhost:9200"]
```

## Best Practices

1. **Log Tasarımı**
   - Yapılandırılmış log formatı
   - Anlamlı log seviyeleri
   - Context bilgileri
   - Performans metrikleri

2. **Performans**
   - Asenkron logging
   - Batch işlemler
   - Buffer yönetimi
   - Rate limiting

3. **Güvenlik**
   - Hassas veri filtreleme
   - Log rotasyonu
   - Erişim kontrolü
   - Audit logging

4. **Monitoring**
   - Log kalitesi
   - Sistem performansı
   - Hata oranları
   - Alerting

## Mülakat Soruları

### Temel Sorular

1. **Serilog nedir ve ne için kullanılır?**
   - **Cevap**: Serilog, .NET uygulamaları için yapılandırılabilir ve genişletilebilir bir logging framework'üdür. Yapılandırılmış loglama, çoklu output desteği ve zengin enrichment özellikleri sunar.

2. **ELK Stack'in bileşenleri nelerdir?**
   - **Cevap**:
     - Elasticsearch: Log depolama ve arama
     - Logstash: Log toplama ve işleme
     - Kibana: Log görselleştirme ve analiz

3. **Serilog'un avantajları nelerdir?**
   - **Cevap**:
     - Yapılandırılabilir yapı
     - Zengin sink desteği
     - Performans optimizasyonu
     - Kolay entegrasyon

4. **Log enrichment nedir?**
   - **Cevap**: Log enrichment, log kayıtlarına ek bilgiler ekleme işlemidir. Context bilgileri, ortam değişkenleri, performans metrikleri gibi ek bilgiler log kayıtlarına zenginlik katar.

5. **Log rotasyonu nedir?**
   - **Cevap**: Log rotasyonu, log dosyalarının belirli bir boyuta veya süreye ulaştığında yeni dosyalara aktarılması işlemidir. Disk alanı yönetimi ve performans için önemlidir.

### Teknik Sorular

1. **Serilog nasıl yapılandırılır?**
   - **Cevap**:
```csharp
public static class SerilogConfiguration
{
    public static IHostBuilder UseSerilog(this IHostBuilder hostBuilder)
    {
        return hostBuilder.UseSerilog((context, services, configuration) => configuration
            .ReadFrom.Configuration(context.Configuration)
            .ReadFrom.Services(services)
            .Enrich.FromLogContext()
            .WriteTo.Console()
            .WriteTo.File("logs/log-.txt", rollingInterval: RollingInterval.Day)
            .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(new Uri("http://localhost:9200"))
            {
                AutoRegisterTemplate = true,
                IndexFormat = "logstash-{0:yyyy.MM.dd}",
                NumberOfShards = 2,
                NumberOfReplicas = 1
            }));
    }
}
```

2. **Yapılandırılmış loglama nasıl yapılır?**
   - **Cevap**:
```csharp
public class LoggingService
{
    private readonly ILogger<LoggingService> _logger;

    public LoggingService(ILogger<LoggingService> logger)
    {
        _logger = logger;
    }

    public void LogUserAction(string userId, string action, object details)
    {
        _logger.LogInformation("User {UserId} performed {Action} with details {@Details}", 
            userId, action, details);
    }

    public void LogError(Exception ex, string context)
    {
        _logger.LogError(ex, "Error occurred in {Context}", context);
    }
}
```

3. **Log enrichment nasıl yapılır?**
   - **Cevap**:
```csharp
public static class LogEnricher
{
    public static ILogger EnrichWithContext(this ILogger logger, string correlationId)
    {
        return logger.ForContext("CorrelationId", correlationId);
    }

    public static ILogger EnrichWithEnvironment(this ILogger logger)
    {
        return logger
            .ForContext("Environment", Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT"))
            .ForContext("MachineName", Environment.MachineName);
    }
}
```

4. **Logstash pipeline nasıl yapılandırılır?**
   - **Cevap**:
```conf
input {
  tcp {
    port => 5000
    codec => json
  }
}

filter {
  json {
    source => "message"
  }
  mutate {
    add_field => { "application" => "myapp" }
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "logstash-%{+YYYY.MM.dd}"
  }
}
```

5. **Kibana dashboard nasıl oluşturulur?**
   - **Cevap**:
     - Index pattern tanımlama
     - Visualization oluşturma
     - Dashboard tasarlama
     - Alert kuralları belirleme

### İleri Seviye Sorular

1. **Serilog performans optimizasyonu nasıl yapılır?**
   - **Cevap**:
     - Asenkron logging
     - Batch işlemler
     - Buffer yönetimi
     - Rate limiting
     - Sink optimizasyonu

2. **ELK Stack ölçeklendirme stratejileri nelerdir?**
   - **Cevap**:
     - Cluster yapılandırması
     - Shard yönetimi
     - Replica dağıtımı
     - Load balancing
     - Cache stratejileri

3. **Log güvenliği nasıl sağlanır?**
   - **Cevap**:
     - Hassas veri filtreleme
     - Erişim kontrolü
     - SSL/TLS
     - Audit logging
     - Compliance

4. **Log analizi ve alerting nasıl yapılır?**
   - **Cevap**:
     - Anomali tespiti
     - Pattern matching
     - Threshold monitoring
     - Alert kuralları
     - Notification kanalları

5. **Log yönetimi ve arşivleme stratejileri nelerdir?**
   - **Cevap**:
     - Retention politikaları
     - Cold storage
     - Index lifecycle
     - Backup stratejileri
     - Compliance gereksinimleri 