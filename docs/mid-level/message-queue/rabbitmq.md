# RabbitMQ

## Giriş

RabbitMQ, açık kaynaklı bir message broker'dır. AMQP (Advanced Message Queuing Protocol) protokolünü kullanır ve .NET uygulamalarıyla kolay entegrasyon sağlar.

## RabbitMQ Temel Kavramlar

1. **Exchange**
   - Mesajların gönderildiği ilk nokta
   - Farklı tipleri vardır
   - Routing kurallarını belirler

2. **Queue**
   - Mesajların depolandığı yer
   - Consumer'ların bağlandığı nokta
   - Özellikleri yapılandırılabilir

3. **Binding**
   - Exchange ve Queue arasındaki bağlantı
   - Routing key ile yönlendirme
   - Farklı binding tipleri

## Exchange Tipleri

1. **Direct Exchange**
   - Routing key tam eşleşme
   - Tek bir queue'ya yönlendirme
   - Basit routing senaryoları

2. **Fanout Exchange**
   - Tüm bağlı queue'lara gönderim
   - Routing key kullanılmaz
   - Broadcast senaryoları

3. **Topic Exchange**
   - Pattern matching ile routing
   - Wildcard kullanımı
   - Kompleks routing senaryoları

4. **Headers Exchange**
   - Header değerlerine göre routing
   - Routing key kullanılmaz
   - Özel routing senaryoları

## .NET'te RabbitMQ Kullanımı

### Temel Kurulum
```csharp
// NuGet paketleri
Install-Package RabbitMQ.Client
```

### Connection Factory
```csharp
var factory = new ConnectionFactory
{
    HostName = "localhost",
    UserName = "guest",
    Password = "guest",
    Port = 5672
};
```

### Producer Örneği
```csharp
public class RabbitMQProducer
{
    private readonly IConnection _connection;
    private readonly IModel _channel;

    public RabbitMQProducer(string hostName)
    {
        var factory = new ConnectionFactory { HostName = hostName };
        _connection = factory.CreateConnection();
        _channel = _connection.CreateModel();
    }

    public void PublishMessage(string exchange, string routingKey, string message)
    {
        var body = Encoding.UTF8.GetBytes(message);
        _channel.BasicPublish(exchange: exchange,
                            routingKey: routingKey,
                            basicProperties: null,
                            body: body);
    }

    public void Dispose()
    {
        _channel?.Dispose();
        _connection?.Dispose();
    }
}
```

### Consumer Örneği
```csharp
public class RabbitMQConsumer
{
    private readonly IConnection _connection;
    private readonly IModel _channel;

    public RabbitMQConsumer(string hostName)
    {
        var factory = new ConnectionFactory { HostName = hostName };
        _connection = factory.CreateConnection();
        _channel = _connection.CreateModel();
    }

    public void StartConsuming(string queueName, Action<string> messageHandler)
    {
        var consumer = new EventingBasicConsumer(_channel);
        consumer.Received += (model, ea) =>
        {
            var body = ea.Body.ToArray();
            var message = Encoding.UTF8.GetString(body);
            messageHandler(message);
            _channel.BasicAck(ea.DeliveryTag, false);
        };

        _channel.BasicConsume(queue: queueName,
                            autoAck: false,
                            consumer: consumer);
    }

    public void Dispose()
    {
        _channel?.Dispose();
        _connection?.Dispose();
    }
}
```

## RabbitMQ Best Practices

1. **Connection Yönetimi**
   - Connection pooling
   - Connection recovery
   - Heartbeat ayarları
   - Timeout yönetimi

2. **Channel Yönetimi**
   - Channel pooling
   - Channel recovery
   - Channel limitleri
   - Channel multiplexing

3. **Queue Yapılandırması**
   - Durable queues
   - Exclusive queues
   - Auto-delete queues
   - Queue limits

4. **Mesaj Yönetimi**
   - Message persistence
   - Message TTL
   - Message priority
   - Message batching

## RabbitMQ Monitoring

1. **Management UI**
   - Queue durumu
   - Connection durumu
   - Channel durumu
   - Message rates

2. **Prometheus Integration**
   - Metrik toplama
   - Alerting
   - Grafana dashboards

3. **Logging**
   - Connection logs
   - Channel logs
   - Message logs
   - Error logs

## Mülakat Soruları

### Temel Sorular

1. **RabbitMQ nedir ve nasıl çalışır?**
   - **Cevap**: RabbitMQ, açık kaynaklı bir message broker'dır. AMQP protokolünü kullanır ve mesajların exchange'ler üzerinden queue'lara yönlendirilmesini sağlar.

2. **RabbitMQ'da exchange tipleri nelerdir?**
   - **Cevap**:
     - Direct Exchange
     - Fanout Exchange
     - Topic Exchange
     - Headers Exchange

3. **RabbitMQ'da mesaj garantisi nasıl sağlanır?**
   - **Cevap**:
     - Publisher confirms
     - Consumer acknowledgments
     - Message persistence
     - Dead letter exchanges

4. **RabbitMQ'da yük dengeleme nasıl yapılır?**
   - **Cevap**:
     - Work queue pattern
     - Consumer prefetch
     - Queue length limits
     - Message priority

5. **RabbitMQ'da hata yönetimi nasıl yapılır?**
   - **Cevap**:
     - Dead letter exchanges
     - Retry mechanisms
     - Error queues
     - Error handling strategies

### Teknik Sorular

1. **RabbitMQ'da connection ve channel yönetimi nasıl yapılır?**
   - **Cevap**:
```csharp
public class RabbitMQConnectionManager
{
    private readonly ConnectionFactory _factory;
    private IConnection _connection;
    private readonly ConcurrentDictionary<string, IModel> _channels;

    public RabbitMQConnectionManager(string hostName)
    {
        _factory = new ConnectionFactory { HostName = hostName };
        _channels = new ConcurrentDictionary<string, IModel>();
    }

    public IModel GetChannel(string channelName)
    {
        return _channels.GetOrAdd(channelName, _ =>
        {
            if (_connection == null || !_connection.IsOpen)
            {
                _connection = _factory.CreateConnection();
            }
            return _connection.CreateModel();
        });
    }
}
```

2. **RabbitMQ'da publisher confirms nasıl implemente edilir?**
   - **Cevap**:
```csharp
public class ConfirmedPublisher
{
    private readonly IModel _channel;

    public ConfirmedPublisher(IModel channel)
    {
        _channel = channel;
        _channel.ConfirmSelect();
    }

    public async Task PublishWithConfirmation(string exchange, string routingKey, string message)
    {
        var body = Encoding.UTF8.GetBytes(message);
        _channel.BasicPublish(exchange, routingKey, null, body);
        
        if (!_channel.WaitForConfirms(TimeSpan.FromSeconds(5)))
        {
            throw new Exception("Message was not confirmed");
        }
    }
}
```

3. **RabbitMQ'da consumer prefetch nasıl ayarlanır?**
   - **Cevap**:
```csharp
public class PrefetchConsumer
{
    private readonly IModel _channel;

    public PrefetchConsumer(IModel channel, ushort prefetchCount)
    {
        _channel = channel;
        _channel.BasicQos(0, prefetchCount, false);
    }

    public void StartConsuming(string queueName)
    {
        var consumer = new EventingBasicConsumer(_channel);
        consumer.Received += (model, ea) =>
        {
            // Process message
            _channel.BasicAck(ea.DeliveryTag, false);
        };

        _channel.BasicConsume(queueName, false, consumer);
    }
}
```

4. **RabbitMQ'da dead letter exchange nasıl yapılandırılır?**
   - **Cevap**:
```csharp
public class DeadLetterSetup
{
    private readonly IModel _channel;

    public DeadLetterSetup(IModel channel)
    {
        _channel = channel;
    }

    public void SetupDeadLetter(string queueName)
    {
        var args = new Dictionary<string, object>
        {
            { "x-dead-letter-exchange", "dlx" },
            { "x-dead-letter-routing-key", queueName }
        };

        _channel.QueueDeclare(queueName, true, false, false, args);
        _channel.ExchangeDeclare("dlx", ExchangeType.Direct);
        _channel.QueueDeclare($"dlq.{queueName}", true, false, false);
        _channel.QueueBind($"dlq.{queueName}", "dlx", queueName);
    }
}
```

5. **RabbitMQ'da message TTL nasıl ayarlanır?**
   - **Cevap**:
```csharp
public class MessageTTLSetup
{
    private readonly IModel _channel;

    public MessageTTLSetup(IModel channel)
    {
        _channel = channel;
    }

    public void SetupQueueWithTTL(string queueName, int ttlMilliseconds)
    {
        var args = new Dictionary<string, object>
        {
            { "x-message-ttl", ttlMilliseconds }
        };

        _channel.QueueDeclare(queueName, true, false, false, args);
    }

    public void PublishMessageWithTTL(string exchange, string routingKey, string message, int ttlMilliseconds)
    {
        var properties = _channel.CreateBasicProperties();
        properties.Expiration = ttlMilliseconds.ToString();

        var body = Encoding.UTF8.GetBytes(message);
        _channel.BasicPublish(exchange, routingKey, properties, body);
    }
}
```

### İleri Seviye Sorular

1. **RabbitMQ'da cluster yapılandırması nasıl yapılır?**
   - **Cevap**:
     - Node discovery
     - Cluster formation
     - Queue mirroring
     - Network partition handling
     - Cluster monitoring

2. **RabbitMQ'da high availability nasıl sağlanır?**
   - **Cevap**:
     - Queue mirroring
     - Publisher confirms
     - Consumer acknowledgments
     - Connection recovery
     - Network redundancy

3. **RabbitMQ'da performans optimizasyonu nasıl yapılır?**
   - **Cevap**:
     - Connection pooling
     - Channel multiplexing
     - Message batching
     - Queue optimization
     - Resource management

4. **RabbitMQ'da monitoring ve alerting nasıl yapılır?**
   - **Cevap**:
     - Management API
     - Prometheus integration
     - Custom metrics
     - Alert rules
     - Dashboard creation

5. **RabbitMQ'da güvenlik nasıl sağlanır?**
   - **Cevap**:
     - SSL/TLS
     - Authentication
     - Authorization
     - Network security
     - Access control 