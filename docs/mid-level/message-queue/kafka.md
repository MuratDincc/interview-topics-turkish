# Apache Kafka

## Giriş

Apache Kafka, yüksek performanslı, dağıtık bir event streaming platformudur. Büyük veri işleme, gerçek zamanlı veri akışı ve event-driven mimariler için ideal bir çözümdür.

## Kafka Temel Kavramlar

1. **Topic**
   - Mesajların kategorize edildiği yapı
   - Partition'lara bölünür
   - Replication ile çoğaltılır

2. **Partition**
   - Topic'in parçalara bölünmesi
   - Paralel işleme sağlar
   - Sıralı mesaj garantisi

3. **Producer**
   - Mesaj üreten uygulama
   - Partition seçimi
   - Mesaj gönderim stratejileri

4. **Consumer**
   - Mesaj tüketen uygulama
   - Consumer grupları
   - Offset yönetimi

## Kafka Mimarisi

1. **Broker**
   - Kafka sunucusu
   - Topic ve partition yönetimi
   - Replication koordinasyonu

2. **Zookeeper**
   - Cluster koordinasyonu
   - Broker yönetimi
   - Topic yapılandırması

3. **Cluster**
   - Birden fazla broker
   - Yüksek erişilebilirlik
   - Veri replikasyonu

## .NET'te Kafka Kullanımı

### Temel Kurulum
```csharp
// NuGet paketleri
Install-Package Confluent.Kafka
```

### Producer Örneği
```csharp
public class KafkaProducer
{
    private readonly IProducer<Null, string> _producer;

    public KafkaProducer(string bootstrapServers)
    {
        var config = new ProducerConfig
        {
            BootstrapServers = bootstrapServers
        };

        _producer = new ProducerBuilder<Null, string>(config).Build();
    }

    public async Task ProduceAsync(string topic, string message)
    {
        try
        {
            var result = await _producer.ProduceAsync(topic, new Message<Null, string> { Value = message });
            Console.WriteLine($"Delivered to {result.TopicPartitionOffset}");
        }
        catch (ProduceException<Null, string> e)
        {
            Console.WriteLine($"Delivery failed: {e.Error.Reason}");
        }
    }

    public void Dispose()
    {
        _producer?.Dispose();
    }
}
```

### Consumer Örneği
```csharp
public class KafkaConsumer
{
    private readonly IConsumer<Ignore, string> _consumer;

    public KafkaConsumer(string bootstrapServers, string groupId)
    {
        var config = new ConsumerConfig
        {
            BootstrapServers = bootstrapServers,
            GroupId = groupId,
            AutoOffsetReset = AutoOffsetReset.Earliest
        };

        _consumer = new ConsumerBuilder<Ignore, string>(config).Build();
    }

    public void StartConsuming(string topic, Action<string> messageHandler)
    {
        _consumer.Subscribe(topic);

        try
        {
            while (true)
            {
                var result = _consumer.Consume();
                messageHandler(result.Message.Value);
            }
        }
        catch (OperationCanceledException)
        {
            _consumer.Close();
        }
    }

    public void Dispose()
    {
        _consumer?.Dispose();
    }
}
```

## Kafka Best Practices

1. **Topic Tasarımı**
   - Anlamlı isimlendirme
   - Partition sayısı
   - Replication faktörü
   - Retention politikası

2. **Producer Yapılandırması**
   - Batch size
   - Compression
   - Retry politikası
   - Timeout ayarları

3. **Consumer Yapılandırması**
   - Consumer grupları
   - Offset yönetimi
   - Commit stratejisi
   - Poll interval

4. **Cluster Yapılandırması**
   - Broker sayısı
   - Replication faktörü
   - Network ayarları
   - Disk yapılandırması

## Kafka Monitoring

1. **JMX Metrikleri**
   - Broker metrikleri
   - Topic metrikleri
   - Consumer metrikleri
   - Producer metrikleri

2. **Prometheus Integration**
   - Metrik toplama
   - Alerting
   - Grafana dashboards

3. **Logging**
   - Broker logs
   - Producer logs
   - Consumer logs
   - Error logs

## Mülakat Soruları

### Temel Sorular

1. **Kafka nedir ve nasıl çalışır?**
   - **Cevap**: Kafka, yüksek performanslı, dağıtık bir event streaming platformudur. Topic'ler üzerinden mesajların partition'lara bölünerek işlenmesini sağlar.

2. **Kafka'da partition ve replication nedir?**
   - **Cevap**:
     - Partition: Topic'in parçalara bölünmesi
     - Replication: Partition'ların kopyalanması
     - Leader/Follower yapısı
     - ISR (In-Sync Replicas)

3. **Kafka'da consumer grupları nedir?**
   - **Cevap**:
     - Paralel işleme
     - Yük dengeleme
     - Offset yönetimi
     - Consumer rebalancing

4. **Kafka'da mesaj garantisi nasıl sağlanır?**
   - **Cevap**:
     - ACK mekanizması
     - ISR yapılandırması
     - Producer retry
     - Consumer commit

5. **Kafka'da yük dengeleme nasıl yapılır?**
   - **Cevap**:
     - Partition dağılımı
     - Consumer grupları
     - Replication
     - Load balancing

### Teknik Sorular

1. **Kafka'da producer yapılandırması nasıl yapılır?**
   - **Cevap**:
```csharp
public class ConfiguredProducer
{
    private readonly IProducer<Null, string> _producer;

    public ConfiguredProducer(string bootstrapServers)
    {
        var config = new ProducerConfig
        {
            BootstrapServers = bootstrapServers,
            Acks = Acks.All,
            MessageSendMaxRetries = 3,
            RetryBackoffMs = 1000,
            CompressionType = CompressionType.Snappy,
            BatchSize = 16384,
            LingerMs = 5
        };

        _producer = new ProducerBuilder<Null, string>(config).Build();
    }
}
```

2. **Kafka'da consumer yapılandırması nasıl yapılır?**
   - **Cevap**:
```csharp
public class ConfiguredConsumer
{
    private readonly IConsumer<Ignore, string> _consumer;

    public ConfiguredConsumer(string bootstrapServers, string groupId)
    {
        var config = new ConsumerConfig
        {
            BootstrapServers = bootstrapServers,
            GroupId = groupId,
            AutoOffsetReset = AutoOffsetReset.Earliest,
            EnableAutoCommit = false,
            MaxPollIntervalMs = 300000,
            SessionTimeoutMs = 10000,
            HeartbeatIntervalMs = 3000
        };

        _consumer = new ConsumerBuilder<Ignore, string>(config).Build();
    }
}
```

3. **Kafka'da transaction nasıl kullanılır?**
   - **Cevap**:
```csharp
public class TransactionalProducer
{
    private readonly IProducer<string, string> _producer;

    public TransactionalProducer(string bootstrapServers)
    {
        var config = new ProducerConfig
        {
            BootstrapServers = bootstrapServers,
            TransactionalId = "my-transactional-id"
        };

        _producer = new ProducerBuilder<string, string>(config).Build();
        _producer.InitTransactions(TimeSpan.FromSeconds(10));
    }

    public async Task ProduceTransactionally(string topic, string key, string value)
    {
        _producer.BeginTransaction();
        try
        {
            await _producer.ProduceAsync(topic, new Message<string, string>
            {
                Key = key,
                Value = value
            });
            _producer.CommitTransaction();
        }
        catch
        {
            _producer.AbortTransaction();
            throw;
        }
    }
}
```

4. **Kafka'da consumer offset yönetimi nasıl yapılır?**
   - **Cevap**:
```csharp
public class OffsetManagedConsumer
{
    private readonly IConsumer<Ignore, string> _consumer;

    public OffsetManagedConsumer(string bootstrapServers, string groupId)
    {
        var config = new ConsumerConfig
        {
            BootstrapServers = bootstrapServers,
            GroupId = groupId,
            EnableAutoCommit = false
        };

        _consumer = new ConsumerBuilder<Ignore, string>(config).Build();
    }

    public void ProcessMessages(string topic)
    {
        _consumer.Subscribe(topic);

        try
        {
            while (true)
            {
                var result = _consumer.Consume();
                try
                {
                    // Process message
                    _consumer.Commit(result);
                }
                catch
                {
                    // Handle error without committing offset
                }
            }
        }
        finally
        {
            _consumer.Close();
        }
    }
}
```

5. **Kafka'da topic yapılandırması nasıl yapılır?**
   - **Cevap**:
```csharp
public class TopicConfigurator
{
    private readonly AdminClientConfig _config;

    public TopicConfigurator(string bootstrapServers)
    {
        _config = new AdminClientConfig
        {
            BootstrapServers = bootstrapServers
        };
    }

    public async Task CreateTopic(string topicName, int numPartitions, short replicationFactor)
    {
        using var adminClient = new AdminClientBuilder(_config).Build();

        await adminClient.CreateTopicsAsync(new[]
        {
            new TopicSpecification
            {
                Name = topicName,
                NumPartitions = numPartitions,
                ReplicationFactor = replicationFactor,
                Configs = new Dictionary<string, string>
                {
                    { "retention.ms", "604800000" },
                    { "cleanup.policy", "delete" }
                }
            }
        });
    }
}
```

### İleri Seviye Sorular

1. **Kafka'da cluster yapılandırması nasıl yapılır?**
   - **Cevap**:
     - Broker yapılandırması
     - Zookeeper yapılandırması
     - Network ayarları
     - Disk yapılandırması
     - Replication stratejisi

2. **Kafka'da high availability nasıl sağlanır?**
   - **Cevap**:
     - Replication
     - ISR yapılandırması
     - Broker dağılımı
     - Network redundancy
     - Failover stratejisi

3. **Kafka'da performans optimizasyonu nasıl yapılır?**
   - **Cevap**:
     - Partition sayısı
     - Replication faktörü
     - Producer batch size
     - Consumer poll interval
     - Disk I/O optimizasyonu

4. **Kafka'da monitoring ve alerting nasıl yapılır?**
   - **Cevap**:
     - JMX metrikleri
     - Prometheus integration
     - Custom metrics
     - Alert rules
     - Dashboard creation

5. **Kafka'da güvenlik nasıl sağlanır?**
   - **Cevap**:
     - SSL/TLS
     - SASL authentication
     - ACL yapılandırması
     - Network security
     - Encryption 