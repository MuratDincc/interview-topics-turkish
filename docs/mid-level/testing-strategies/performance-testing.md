# Performance Testing

## Giriş

Performance Testing, uygulamanın performans karakteristiklerini ölçen ve analiz eden testing yaklaşımıdır. Mid-level geliştiriciler için performance testing'i anlamak, application optimization, capacity planning ve user experience için kritik öneme sahiptir. Bu dosya, load testing, stress testing, benchmark testing ve performance monitoring konularını kapsar.

## NBomber Implementation

### 1. Load Testing
Yük testi implementasyonu.

```csharp
public class OrderServiceLoadTests
{
    [Fact]
    public async Task CreateOrder_LoadTest()
    {
        var scenario = ScenarioBuilder.CreateScenario("Create Order Load Test", async context =>
        {
            var orderData = new CreateOrderRequest
            {
                CustomerId = Guid.NewGuid(),
                Items = new List<OrderItem>
                {
                    new OrderItem { ProductId = Guid.NewGuid(), Quantity = Random.Shared.Next(1, 10), Price = Random.Shared.Next(10, 100) }
                },
                ShippingAddress = new Address
                {
                    Street = "Test Street",
                    City = "Test City",
                    Country = "Test Country",
                    PostalCode = "12345"
                }
            };
            
            var httpClient = context.GetHttpClient();
            var request = HttpRequestMessage.Create(HttpMethod.Post, "/api/orders", 
                JsonContent.Create(orderData));
            
            var response = await httpClient.SendAsync(request);
            
            return response.IsSuccessStatusCode ? Response.Ok() : Response.Fail();
        })
        .WithLoadSimulations(
            Simulation.Inject(rate: 100, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromMinutes(5))
        );
        
        NBomberRunner
            .RegisterScenarios(scenario)
            .Run();
    }
    
    [Fact]
    public async Task GetOrder_LoadTest()
    {
        var scenario = ScenarioBuilder.CreateScenario("Get Order Load Test", async context =>
        {
            var orderId = Guid.NewGuid();
            var httpClient = context.GetHttpClient();
            var request = HttpRequestMessage.Create(HttpMethod.Get, $"/api/orders/{orderId}");
            
            var response = await httpClient.SendAsync(request);
            
            return response.IsSuccessStatusCode ? Response.Ok() : Response.Fail();
        })
        .WithLoadSimulations(
            Simulation.Inject(rate: 200, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromMinutes(5))
        );
        
        NBomberRunner
            .RegisterScenarios(scenario)
            .Run();
    }
    
    [Fact]
    public async Task SearchOrders_LoadTest()
    {
        var scenario = ScenarioBuilder.CreateScenario("Search Orders Load Test", async context =>
        {
            var searchTerm = $"customer_{Random.Shared.Next(1, 1000)}";
            var httpClient = context.GetHttpClient();
            var request = HttpRequestMessage.Create(HttpMethod.Get, $"/api/orders/search?q={searchTerm}");
            
            var response = await httpClient.SendAsync(request);
            
            return response.IsSuccessStatusCode ? Response.Ok() : Response.Fail();
        })
        .WithLoadSimulations(
            Simulation.Inject(rate: 150, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromMinutes(5))
        );
        
        NBomberRunner
            .RegisterScenarios(scenario)
            .Run();
    }
    
    [Fact]
    public async Task UpdateOrder_LoadTest()
    {
        var scenario = ScenarioBuilder.CreateScenario("Update Order Load Test", async context =>
        {
            var orderId = Guid.NewGuid();
            var updateData = new UpdateOrderRequest
            {
                Status = "Processing",
                Notes = $"Updated at {DateTime.UtcNow}"
            };
            
            var httpClient = context.GetHttpClient();
            var request = HttpRequestMessage.Create(HttpMethod.Put, $"/api/orders/{orderId}", 
                JsonContent.Create(updateData));
            
            var response = await httpClient.SendAsync(request);
            
            return response.IsSuccessStatusCode ? Response.Ok() : Response.Fail();
        })
        .WithLoadSimulations(
            Simulation.Inject(rate: 80, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromMinutes(5))
        );
        
        NBomberRunner
            .RegisterScenarios(scenario)
            .Run();
    }
}

public class CreateOrderRequest
{
    public Guid CustomerId { get; set; }
    public List<OrderItem> Items { get; set; } = new();
    public Address ShippingAddress { get; set; }
}

public class OrderItem
{
    public Guid ProductId { get; set; }
    public int Quantity { get; set; }
    public decimal Price { get; set; }
}

public class Address
{
    public string Street { get; set; }
    public string City { get; set; }
    public string Country { get; set; }
    public string PostalCode { get; set; }
}

public class UpdateOrderRequest
{
    public string Status { get; set; }
    public string Notes { get; set; }
}
```

### 2. Stress Testing
Stres testi implementasyonu.

```csharp
public class OrderServiceStressTests
{
    [Fact]
    public async Task CreateOrder_StressTest()
    {
        var scenario = ScenarioBuilder.CreateScenario("Create Order Stress Test", async context =>
        {
            var orderData = new CreateOrderRequest
            {
                CustomerId = Guid.NewGuid(),
                Items = Enumerable.Range(1, Random.Shared.Next(1, 20))
                    .Select(i => new OrderItem
                    {
                        ProductId = Guid.NewGuid(),
                        Quantity = Random.Shared.Next(1, 50),
                        Price = Random.Shared.Next(1, 1000)
                    })
                    .ToList(),
                ShippingAddress = new Address
                {
                    Street = "Test Street",
                    City = "Test City",
                    Country = "Test Country",
                    PostalCode = "12345"
                }
            };
            
            var httpClient = context.GetHttpClient();
            var request = HttpRequestMessage.Create(HttpMethod.Post, "/api/orders", 
                JsonContent.Create(orderData));
            
            var response = await httpClient.SendAsync(request);
            
            return response.IsSuccessStatusCode ? Response.Ok() : Response.Fail();
        })
        .WithLoadSimulations(
            Simulation.Stress(
                rate: 50,
                interval: TimeSpan.FromSeconds(1),
                during: TimeSpan.FromMinutes(10),
                copies: 10,
                rampUp: TimeSpan.FromMinutes(2)
            )
        );
        
        NBomberRunner
            .RegisterScenarios(scenario)
            .Run();
    }
    
    [Fact]
    public async Task DatabaseConnection_StressTest()
    {
        var scenario = ScenarioBuilder.CreateScenario("Database Connection Stress Test", async context =>
        {
            try
            {
                using var scope = context.GetServiceProvider().CreateScope();
                var dbContext = scope.ServiceProvider.GetRequiredService<OrderDbContext>();
                
                // Perform multiple database operations
                var orders = await dbContext.Orders
                    .Where(o => o.CreatedAt > DateTime.UtcNow.AddDays(-30))
                    .Take(100)
                    .ToListAsync();
                
                var customers = await dbContext.Customers
                    .Where(c => c.IsActive)
                    .Take(50)
                    .ToListAsync();
                
                var products = await dbContext.Products
                    .Where(p => p.Price > 0)
                    .Take(200)
                    .ToListAsync();
                
                return Response.Ok();
            }
            catch (Exception)
            {
                return Response.Fail();
            }
        })
        .WithLoadSimulations(
            Simulation.Stress(
                rate: 100,
                interval: TimeSpan.FromSeconds(1),
                during: TimeSpan.FromMinutes(15),
                copies: 20,
                rampUp: TimeSpan.FromMinutes(3)
            )
        );
        
        NBomberRunner
            .RegisterScenarios(scenario)
            .Run();
    }
    
    [Fact]
    public async Task MemoryUsage_StressTest()
    {
        var scenario = ScenarioBuilder.CreateScenario("Memory Usage Stress Test", async context =>
        {
            try
            {
                // Simulate memory-intensive operations
                var largeData = new List<byte[]>();
                
                for (int i = 0; i < 1000; i++)
                {
                    largeData.Add(new byte[1024 * 1024]); // 1MB chunks
                }
                
                // Perform some processing
                var processedData = largeData
                    .SelectMany(bytes => bytes)
                    .Where(b => b > 0)
                    .ToArray();
                
                // Clear memory
                largeData.Clear();
                largeData = null;
                
                GC.Collect();
                GC.WaitForPendingFinalizers();
                
                return Response.Ok();
            }
            catch (Exception)
            {
                return Response.Fail();
            }
        })
        .WithLoadSimulations(
            Simulation.Stress(
                rate: 30,
                interval: TimeSpan.FromSeconds(2),
                during: TimeSpan.FromMinutes(20),
                copies: 5,
                rampUp: TimeSpan.FromMinutes(5)
            )
        );
        
        NBomberRunner
            .RegisterScenarios(scenario)
            .Run();
    }
}
```

### 3. Benchmark Testing
Benchmark testi implementasyonu.

```csharp
public class OrderServiceBenchmarkTests
{
    [Fact]
    public async Task CreateOrder_Benchmark()
    {
        var scenario = ScenarioBuilder.CreateScenario("Create Order Benchmark", async context =>
        {
            var orderData = new CreateOrderRequest
            {
                CustomerId = Guid.NewGuid(),
                Items = new List<OrderItem>
                {
                    new OrderItem { ProductId = Guid.NewGuid(), Quantity = 1, Price = 100 }
                },
                ShippingAddress = new Address
                {
                    Street = "Benchmark Street",
                    City = "Benchmark City",
                    Country = "Benchmark Country",
                    PostalCode = "12345"
                }
            };
            
            var httpClient = context.GetHttpClient();
            var request = HttpRequestMessage.Create(HttpMethod.Post, "/api/orders", 
                JsonContent.Create(orderData));
            
            var stopwatch = Stopwatch.StartNew();
            var response = await httpClient.SendAsync(request);
            stopwatch.Stop();
            
            context.Logger.Information($"Create Order took: {stopwatch.ElapsedMilliseconds}ms");
            
            return response.IsSuccessStatusCode ? Response.Ok() : Response.Fail();
        })
        .WithLoadSimulations(
            Simulation.Inject(rate: 1000, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromMinutes(1))
        );
        
        NBomberRunner
            .RegisterScenarios(scenario)
            .Run();
    }
    
    [Fact]
    public async Task DatabaseQuery_Benchmark()
    {
        var scenario = ScenarioBuilder.CreateScenario("Database Query Benchmark", async context =>
        {
            try
            {
                using var scope = context.GetServiceProvider().CreateScope();
                var dbContext = scope.ServiceProvider.GetRequiredService<OrderDbContext>();
                
                var stopwatch = Stopwatch.StartNew();
                
                // Benchmark different query types
                var simpleQuery = await dbContext.Orders
                    .Where(o => o.Status == "Active")
                    .CountAsync();
                
                var complexQuery = await dbContext.Orders
                    .Include(o => o.Customer)
                    .Include(o => o.Items)
                    .ThenInclude(i => i.Product)
                    .Where(o => o.CreatedAt > DateTime.UtcNow.AddDays(-7))
                    .OrderByDescending(o => o.CreatedAt)
                    .Take(100)
                    .ToListAsync();
                
                var aggregationQuery = await dbContext.Orders
                    .Where(o => o.CreatedAt > DateTime.UtcNow.AddDays(-30))
                    .GroupBy(o => o.Status)
                    .Select(g => new { Status = g.Key, Count = g.Count(), TotalAmount = g.Sum(o => o.TotalAmount) })
                    .ToListAsync();
                
                stopwatch.Stop();
                
                context.Logger.Information($"Database queries took: {stopwatch.ElapsedMilliseconds}ms");
                context.Logger.Information($"Simple query result: {simpleQuery}");
                context.Logger.Information($"Complex query result: {complexQuery.Count} orders");
                context.Logger.Information($"Aggregation query result: {aggregationQuery.Count} groups");
                
                return Response.Ok();
            }
            catch (Exception ex)
            {
                context.Logger.Error(ex, "Database benchmark failed");
                return Response.Fail();
            }
        })
        .WithLoadSimulations(
            Simulation.Inject(rate: 500, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromMinutes(2))
        );
        
        NBomberRunner
            .RegisterScenarios(scenario)
            .Run();
    }
    
    [Fact]
    public async Task Caching_Benchmark()
    {
        var scenario = ScenarioBuilder.CreateScenario("Caching Benchmark", async context =>
        {
            try
            {
                var cache = context.GetServiceProvider().GetRequiredService<IDistributedCache>();
                var cacheKey = $"benchmark_{Random.Shared.Next(1, 1000)}";
                var testData = new byte[1024]; // 1KB test data
                
                var stopwatch = Stopwatch.StartNew();
                
                // Benchmark cache operations
                await cache.SetAsync(cacheKey, testData, new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
                });
                
                var retrievedData = await cache.GetAsync(cacheKey);
                var cacheHit = retrievedData != null;
                
                if (cacheHit)
                {
                    await cache.RemoveAsync(cacheKey);
                }
                
                stopwatch.Stop();
                
                context.Logger.Information($"Cache operations took: {stopwatch.ElapsedMilliseconds}ms, Cache hit: {cacheHit}");
                
                return Response.Ok();
            }
            catch (Exception ex)
            {
                context.Logger.Error(ex, "Cache benchmark failed");
                return Response.Fail();
            }
        })
        .WithLoadSimulations(
            Simulation.Inject(rate: 2000, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromMinutes(1))
        );
        
        NBomberRunner
            .RegisterScenarios(scenario)
            .Run();
    }
}

public class PerformanceMetrics
{
    public string Operation { get; set; }
    public long MinResponseTime { get; set; }
    public long MaxResponseTime { get; set; }
    public long AverageResponseTime { get; set; }
    public long P95ResponseTime { get; set; }
    public long P99ResponseTime { get; set; }
    public int TotalRequests { get; set; }
    public int SuccessfulRequests { get; set; }
    public int FailedRequests { get; set; }
    public double SuccessRate => TotalRequests > 0 ? (double)SuccessfulRequests / TotalRequests * 100 : 0;
    public double RequestsPerSecond => TotalRequests > 0 ? (double)TotalRequests / (MaxResponseTime - MinResponseTime) * 1000 : 0;
}

public class PerformanceTestConfiguration
{
    public int ConcurrentUsers { get; set; } = 100;
    public TimeSpan TestDuration { get; set; } = TimeSpan.FromMinutes(10);
    public TimeSpan RampUpTime { get; set; } = TimeSpan.FromMinutes(2);
    public int RequestRate { get; set; } = 1000;
    public TimeSpan RequestInterval { get; set; } = TimeSpan.FromSeconds(1);
    public bool EnableMetrics { get; set; } = true;
    public bool EnableLogging { get; set; } = true;
    public string MetricsEndpoint { get; set; } = "http://localhost:9090";
}
```

## Mülakat Soruları

### Temel Sorular

1. **Performance Testing nedir?**
   - **Cevap**: Uygulamanın performans karakteristiklerini ölçen testing yaklaşımı.

2. **Load Testing nedir?**
   - **Cevap**: Normal yük altında sistem performansını test etme.

3. **Stress Testing nedir?**
   - **Cevap**: Sistem limitlerini aşan yük altında davranışı test etme.

4. **Benchmark Testing nedir?**
   - **Cevap**: Belirli operasyonların performansını ölçme ve karşılaştırma.

5. **Performance testing ne zaman kullanılır?**
   - **Cevap**: Performance optimization, capacity planning, SLA validation.

### Teknik Sorular

1. **NBomber nasıl implement edilir?**
   - **Cevap**: ScenarioBuilder, Simulation, NBomberRunner, Response handling.

2. **Load simulation nasıl yapılır?**
   - **Cevap**: Inject, Stress, RampUp, Concurrent users, Request rate.

3. **Performance metrics nasıl toplanır?**
   - **Cevap**: Response time, Throughput, Error rate, Resource usage.

4. **Performance testing CI/CD'de nasıl kullanılır?**
   - **Cevap**: Automated testing, Performance regression detection, SLA monitoring.

5. **Performance testing performance nasıl optimize edilir?**
   - **Cevap**: Efficient scenarios, Resource management, Metrics collection, Result analysis.

## Best Practices

1. **Test Design**
   - Realistic scenarios tasarlayın
   - Appropriate load levels belirleyin
   - Clear success criteria tanımlayın
   - Baseline measurements alın

2. **Test Execution**
   - Controlled environment kullanın
   - Consistent test conditions sağlayın
   - Resource monitoring yapın
   - Error handling implement edin

3. **Metrics Collection**
   - Comprehensive metrics toplayın
   - Performance baselines oluşturun
   - Trend analysis yapın
   - Alerting implement edin

4. **Result Analysis**
   - Performance bottlenecks identify edin
   - Root cause analysis yapın
   - Optimization recommendations geliştirin
   - Continuous improvement sağlayın

5. **Integration & Maintenance**
   - CI/CD pipeline entegre edin
   - Automated performance testing yapın
   - Performance regression detection implement edin
   - Regular performance reviews yapın

## Kaynaklar

- [NBomber](https://nbomber.com/)
- [Performance Testing](https://docs.microsoft.com/en-us/azure/devops/test/load-test/get-started-simple-cloud-load-test)
- [Load Testing](https://docs.microsoft.com/en-us/azure/architecture/patterns/queue-based-load-leveling)
- [Performance Monitoring](https://docs.microsoft.com/en-us/azure/azure-monitor/app/performance-counters)
- [.NET Performance](https://docs.microsoft.com/en-us/dotnet/standard/performance/)
