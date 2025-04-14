# Message Queue Genel Bakış

## Giriş

Message Queue (Mesaj Kuyruğu), uygulamalar arasında asenkron iletişim sağlayan bir sistemdir. Bu sistem, uygulamaların birbirleriyle gevşek bağlı (loosely coupled) bir şekilde iletişim kurmasını sağlar.

## Message Queue Nedir?

Message Queue, mesajların gönderildiği ve alındığı bir ara katmandır. Bu sistem:
- Asenkron iletişim sağlar
- Uygulamalar arası bağımlılığı azaltır
- Ölçeklenebilirlik sağlar
- Hata toleransı sunar
- Yük dengeleme yapar

## Message Queue Kullanım Senaryoları

1. **Asenkron İşlemler**
   - Uzun süren işlemler
   - Arka plan görevleri
   - Bildirim gönderimi

2. **Mikroservis İletişimi**
   - Servisler arası iletişim
   - Event-driven mimari
   - Servis entegrasyonu

3. **Yük Dengeleme**
   - İş yükü dağıtımı
   - Trafik yönetimi
   - Kaynak optimizasyonu

4. **Hata Toleransı**
   - Sistem kesintilerinde veri kaybını önleme
   - Yeniden deneme mekanizmaları
   - Veri tutarlılığı

## Popüler Message Queue Sistemleri

1. **RabbitMQ**
   - Açık kaynak
   - AMQP protokolü
   - Kolay kurulum ve yönetim
   - .NET entegrasyonu

2. **Apache Kafka**
   - Yüksek performans
   - Dağıtık sistem
   - Event streaming
   - Büyük veri işleme

## Message Queue Seçim Kriterleri

1. **Performans**
   - Mesaj işleme hızı
   - Gecikme süresi
   - Ölçeklenebilirlik

2. **Güvenilirlik**
   - Veri kaybı riski
   - Hata toleransı
   - Yeniden deneme mekanizmaları

3. **Özellikler**
   - Protokol desteği
   - Yönetim araçları
   - Monitoring
   - Güvenlik

4. **Entegrasyon**
   - .NET desteği
   - API kalitesi
   - Dokümantasyon
   - Topluluk desteği

## Message Queue Best Practices

1. **Mesaj Tasarımı**
   - Anlamlı mesaj yapısı
   - Versiyonlama
   - Serileştirme formatı

2. **Hata Yönetimi**
   - Retry mekanizmaları
   - Dead letter queue
   - Hata loglama

3. **Performans Optimizasyonu**
   - Batch işlemler
   - Mesaj boyutu
   - Queue yapılandırması

4. **Monitoring**
   - Queue durumu
   - Performans metrikleri
   - Hata takibi

## Mülakat Soruları

### Temel Sorular

1. **Message Queue nedir ve neden kullanılır?**
   - **Cevap**: Message Queue, uygulamalar arasında asenkron iletişim sağlayan bir sistemdir. Kullanım nedenleri:
     - Asenkron işlemler
     - Uygulama bağımsızlığı
     - Ölçeklenebilirlik
     - Hata toleransı
     - Yük dengeleme

2. **Message Queue sistemlerinin temel bileşenleri nelerdir?**
   - **Cevap**:
     - Producer (Üretici)
     - Consumer (Tüketici)
     - Queue (Kuyruk)
     - Exchange (Değişim)
     - Routing Key (Yönlendirme Anahtarı)
     - Message (Mesaj)

3. **Message Queue sistemlerinde hangi protokoller kullanılır?**
   - **Cevap**:
     - AMQP (Advanced Message Queuing Protocol)
     - MQTT (Message Queuing Telemetry Transport)
     - STOMP (Streaming Text Oriented Messaging Protocol)
     - HTTP/HTTPS
     - WebSocket

4. **Message Queue sistemlerinde mesaj garantisi nasıl sağlanır?**
   - **Cevap**:
     - ACK (Acknowledgment) mekanizması
     - Persistence (Kalıcılık)
     - Transaction desteği
     - Retry mekanizması
     - Dead letter queue

5. **Message Queue sistemlerinde yük dengeleme nasıl yapılır?**
   - **Cevap**:
     - Round-robin dağıtım
     - Work queue pattern
     - Consumer grupları
     - Queue partitioning
     - Load balancing algoritmaları

### Teknik Sorular

1. **RabbitMQ'da Exchange tipleri nelerdir?**
   - **Cevap**:
     - Direct Exchange
     - Fanout Exchange
     - Topic Exchange
     - Headers Exchange

2. **Kafka'da Partition ve Replication kavramları nedir?**
   - **Cevap**:
     - Partition: Topic'in parçalara bölünmesi
     - Replication: Partition'ların kopyalanması
     - Leader/Follower yapısı
     - ISR (In-Sync Replicas)

3. **Message Queue sistemlerinde mesaj serileştirme nasıl yapılır?**
   - **Cevap**:
     - JSON
     - XML
     - Protocol Buffers
     - Avro
     - MessagePack

4. **Message Queue sistemlerinde monitoring nasıl yapılır?**
   - **Cevap**:
     - Queue metrikleri
     - Consumer metrikleri
     - Producer metrikleri
     - Hata oranları
     - Gecikme süreleri

5. **Message Queue sistemlerinde güvenlik nasıl sağlanır?**
   - **Cevap**:
     - SSL/TLS
     - Authentication
     - Authorization
     - ACL (Access Control List)
     - Network izolasyonu

### Pratik Sorular

1. **RabbitMQ'da basit bir producer ve consumer nasıl oluşturulur?**
   - **Cevap**:
```csharp
// Producer
using var connection = factory.CreateConnection();
using var channel = connection.CreateModel();

channel.QueueDeclare(queue: "hello",
                     durable: false,
                     exclusive: false,
                     autoDelete: false,
                     arguments: null);

var message = "Hello World!";
var body = Encoding.UTF8.GetBytes(message);

channel.BasicPublish(exchange: "",
                     routingKey: "hello",
                     basicProperties: null,
                     body: body);

// Consumer
var factory = new ConnectionFactory() { HostName = "localhost" };
using var connection = factory.CreateConnection();
using var channel = connection.CreateModel();

channel.QueueDeclare(queue: "hello",
                     durable: false,
                     exclusive: false,
                     autoDelete: false,
                     arguments: null);

var consumer = new EventingBasicConsumer(channel);
consumer.Received += (model, ea) =>
{
    var body = ea.Body.ToArray();
    var message = Encoding.UTF8.GetString(body);
    Console.WriteLine($"Received: {message}");
};

channel.BasicConsume(queue: "hello",
                     autoAck: true,
                     consumer: consumer);
```

2. **Kafka'da basit bir producer ve consumer nasıl oluşturulur?**
   - **Cevap**:
```csharp
// Producer
var config = new ProducerConfig
{
    BootstrapServers = "localhost:9092"
};

using var producer = new ProducerBuilder<Null, string>(config).Build();

var message = new Message<Null, string>
{
    Value = "Hello World!"
};

await producer.ProduceAsync("test-topic", message);

// Consumer
var config = new ConsumerConfig
{
    BootstrapServers = "localhost:9092",
    GroupId = "test-group",
    AutoOffsetReset = AutoOffsetReset.Earliest
};

using var consumer = new ConsumerBuilder<Ignore, string>(config).Build();
consumer.Subscribe("test-topic");

while (true)
{
    var result = consumer.Consume();
    Console.WriteLine($"Received: {result.Message.Value}");
}
```

### İleri Seviye Sorular

1. **Message Queue sistemlerinde CAP teoremi nasıl uygulanır?**
   - **Cevap**:
     - Consistency (Tutarlılık)
     - Availability (Erişilebilirlik)
     - Partition Tolerance (Bölünme Toleransı)
     - Trade-off'lar
     - Sistem seçimi

2. **Message Queue sistemlerinde ölçeklendirme stratejileri nelerdir?**
   - **Cevap**:
     - Horizontal scaling
     - Vertical scaling
     - Sharding
     - Partitioning
     - Load balancing

3. **Message Queue sistemlerinde veri kaybı nasıl önlenir?**
   - **Cevap**:
     - Persistence
     - Replication
     - ACK mekanizması
     - Transaction
     - Backup stratejileri

4. **Message Queue sistemlerinde performans optimizasyonu nasıl yapılır?**
   - **Cevap**:
     - Batch processing
     - Compression
     - Caching
     - Queue tuning
     - Network optimizasyonu

5. **Message Queue sistemlerinde monitoring ve alerting nasıl yapılır?**
   - **Cevap**:
     - Metrik toplama
     - Logging
     - Alerting kuralları
     - Dashboard
     - Trend analizi 