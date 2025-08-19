# Dashboard Creation

## Giriş

Dashboard Creation, monitoring ve observability sistemlerinde verileri görsel olarak sunan ve analiz eden arayüzler oluşturma sürecidir. Mid-level geliştiriciler için dashboard creation'ı anlamak, data visualization, real-time monitoring ve business intelligence için kritik öneme sahiptir. Bu dosya, dashboard design, widget implementation, data binding ve real-time updates konularını kapsar.

## Dashboard Framework

### 1. Dashboard Engine
Dashboard'ları yöneten ve render eden engine.

```csharp
public class DashboardEngine : IDashboardEngine
{
    private readonly ILogger<DashboardEngine> _logger;
    private readonly IConfiguration _configuration;
    private readonly IDashboardRepository _dashboardRepository;
    private readonly IWidgetFactory _widgetFactory;
    private readonly IDataProvider _dataProvider;
    private readonly Dictionary<string, Dashboard> _activeDashboards;
    
    public DashboardEngine(ILogger<DashboardEngine> logger, IConfiguration configuration,
        IDashboardRepository dashboardRepository, IWidgetFactory widgetFactory, IDataProvider dataProvider)
    {
        _logger = logger;
        _configuration = configuration;
        _dashboardRepository = dashboardRepository;
        _widgetFactory = widgetFactory;
        _dataProvider = dataProvider;
        _activeDashboards = new Dictionary<string, Dashboard>();
    }
    
    public async Task InitializeAsync()
    {
        try
        {
            var dashboards = await _dashboardRepository.GetAllDashboardsAsync();
            
            foreach (var dashboard in dashboards)
            {
                await LoadDashboardAsync(dashboard);
            }
            
            _logger.LogInformation("Dashboard engine initialized with {Count} dashboards", _activeDashboards.Count);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error initializing dashboard engine");
        }
    }
    
    public async Task<Dashboard> CreateDashboardAsync(DashboardDefinition definition)
    {
        try
        {
            var dashboard = new Dashboard
            {
                Id = Guid.NewGuid().ToString(),
                Name = definition.Name,
                Description = definition.Description,
                Layout = definition.Layout,
                RefreshInterval = definition.RefreshInterval,
                IsPublic = definition.IsPublic,
                CreatedAt = DateTime.UtcNow,
                CreatedBy = definition.CreatedBy
            };
            
            // Create widgets based on definition
            foreach (var widgetDef in definition.Widgets)
            {
                var widget = await _widgetFactory.CreateWidgetAsync(widgetDef);
                dashboard.Widgets.Add(widget);
            }
            
            await _dashboardRepository.SaveDashboardAsync(dashboard);
            await LoadDashboardAsync(dashboard);
            
            _logger.LogInformation("Dashboard created: {Name}", dashboard.Name);
            return dashboard;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating dashboard: {Name}", definition.Name);
            throw;
        }
    }
    
    public async Task<Dashboard> GetDashboardAsync(string dashboardId)
    {
        if (_activeDashboards.TryGetValue(dashboardId, out var dashboard))
        {
            return dashboard;
        }
        
        var dbDashboard = await _dashboardRepository.GetDashboardAsync(dashboardId);
        if (dbDashboard != null)
        {
            await LoadDashboardAsync(dbDashboard);
            return dbDashboard;
        }
        
        return null;
    }
    
    public async Task<List<Dashboard>> GetAllDashboardsAsync()
    {
        return _activeDashboards.Values.ToList();
    }
    
    public async Task UpdateDashboardAsync(Dashboard dashboard)
    {
        try
        {
            await _dashboardRepository.SaveDashboardAsync(dashboard);
            
            if (_activeDashboards.ContainsKey(dashboard.Id))
            {
                _activeDashboards[dashboard.Id] = dashboard;
            }
            
            _logger.LogInformation("Dashboard updated: {Name}", dashboard.Name);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error updating dashboard: {Name}", dashboard.Name);
            throw;
        }
    }
    
    public async Task DeleteDashboardAsync(string dashboardId)
    {
        try
        {
            await _dashboardRepository.DeleteDashboardAsync(dashboardId);
            _activeDashboards.Remove(dashboardId);
            
            _logger.LogInformation("Dashboard deleted: {DashboardId}", dashboardId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error deleting dashboard: {DashboardId}", dashboardId);
            throw;
        }
    }
    
    public async Task<DashboardData> GetDashboardDataAsync(string dashboardId)
    {
        try
        {
            var dashboard = await GetDashboardAsync(dashboardId);
            if (dashboard == null)
            {
                return null;
            }
            
            var dashboardData = new DashboardData
            {
                DashboardId = dashboardId,
                DashboardName = dashboard.Name,
                LastUpdated = DateTime.UtcNow,
                Widgets = new List<WidgetData>()
            };
            
            // Get data for each widget
            foreach (var widget in dashboard.Widgets)
            {
                var widgetData = await GetWidgetDataAsync(widget);
                dashboardData.Widgets.Add(widgetData);
            }
            
            return dashboardData;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting dashboard data: {DashboardId}", dashboardId);
            throw;
        }
    }
    
    private async Task LoadDashboardAsync(Dashboard dashboard)
    {
        try
        {
            // Initialize widgets
            foreach (var widget in dashboard.Widgets)
            {
                await widget.InitializeAsync();
            }
            
            _activeDashboards[dashboard.Id] = dashboard;
            _logger.LogDebug("Dashboard loaded: {Name}", dashboard.Name);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error loading dashboard: {Name}", dashboard.Name);
        }
    }
    
    private async Task<WidgetData> GetWidgetDataAsync(Widget widget)
    {
        try
        {
            var data = await widget.GetDataAsync();
            
            return new WidgetData
            {
                WidgetId = widget.Id,
                WidgetType = widget.Type,
                Data = data,
                LastUpdated = DateTime.UtcNow,
                Status = widget.Status
            };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting widget data: {WidgetId}", widget.Id);
            
            return new WidgetData
            {
                WidgetId = widget.Id,
                WidgetType = widget.Type,
                Data = null,
                LastUpdated = DateTime.UtcNow,
                Status = WidgetStatus.Error,
                Error = ex.Message
            };
        }
    }
}

public interface IDashboardEngine
{
    Task InitializeAsync();
    Task<Dashboard> CreateDashboardAsync(DashboardDefinition definition);
    Task<Dashboard> GetDashboardAsync(string dashboardId);
    Task<List<Dashboard>> GetAllDashboardsAsync();
    Task UpdateDashboardAsync(Dashboard dashboard);
    Task DeleteDashboardAsync(string dashboardId);
    Task<DashboardData> GetDashboardDataAsync(string dashboardId);
}

public class Dashboard
{
    public string Id { get; set; }
    public string Name { get; set; }
    public string Description { get; set; }
    public DashboardLayout Layout { get; set; }
    public TimeSpan RefreshInterval { get; set; }
    public bool IsPublic { get; set; }
    public DateTime CreatedAt { get; set; }
    public string CreatedBy { get; set; }
    public List<Widget> Widgets { get; set; } = new();
}

public class DashboardDefinition
{
    public string Name { get; set; }
    public string Description { get; set; }
    public DashboardLayout Layout { get; set; }
    public TimeSpan RefreshInterval { get; set; }
    public bool IsPublic { get; set; }
    public string CreatedBy { get; set; }
    public List<WidgetDefinition> Widgets { get; set; } = new();
}

public class DashboardLayout
{
    public int Columns { get; set; } = 12;
    public int Rows { get; set; } = 8;
    public List<WidgetPosition> WidgetPositions { get; set; } = new();
}

public class WidgetPosition
{
    public string WidgetId { get; set; }
    public int Column { get; set; }
    public int Row { get; set; }
    public int Width { get; set; }
    public int Height { get; set; }
}

public class DashboardData
{
    public string DashboardId { get; set; }
    public string DashboardName { get; set; }
    public DateTime LastUpdated { get; set; }
    public List<WidgetData> Widgets { get; set; } = new();
}
```

### 2. Widget System
Dashboard widget'larını yöneten sistem.

```csharp
public abstract class Widget : IWidget
{
    public string Id { get; set; } = Guid.NewGuid().ToString();
    public string Name { get; set; }
    public string Type { get; set; }
    public WidgetConfiguration Configuration { get; set; }
    public WidgetStatus Status { get; set; } = WidgetStatus.Inactive;
    public DateTime LastDataUpdate { get; set; }
    
    public abstract Task InitializeAsync();
    public abstract Task<object> GetDataAsync();
    public abstract Task<bool> ValidateConfigurationAsync();
    
    protected virtual async Task<object> GetDataFromProviderAsync(string dataSource, Dictionary<string, object> parameters)
    {
        // This would integrate with your data provider
        await Task.Delay(100); // Simulate async operation
        
        // Return mock data based on widget type
        return Type switch
        {
            "line-chart" => GenerateLineChartData(),
            "bar-chart" => GenerateBarChartData(),
            "gauge" => GenerateGaugeData(),
            "table" => GenerateTableData(),
            "metric" => GenerateMetricData(),
            _ => new { message = "Unknown widget type" }
        };
    }
    
    private object GenerateLineChartData()
    {
        var random = new Random();
        var dataPoints = new List<object>();
        
        for (int i = 0; i < 24; i++)
        {
            dataPoints.Add(new
            {
                time = DateTime.UtcNow.AddHours(-23 + i).ToString("HH:mm"),
                value = random.Next(50, 150)
            });
        }
        
        return new
        {
            type = "line-chart",
            data = dataPoints,
            xAxis = "Time",
            yAxis = "Value"
        };
    }
    
    private object GenerateBarChartData()
    {
        var random = new Random();
        var categories = new[] { "Category A", "Category B", "Category C", "Category D", "Category E" };
        var data = categories.Select(c => new { category = c, value = random.Next(10, 100) }).ToList();
        
        return new
        {
            type = "bar-chart",
            data = data,
            xAxis = "Category",
            yAxis = "Value"
        };
    }
    
    private object GenerateGaugeData()
    {
        var random = new Random();
        var value = random.Next(0, 100);
        
        return new
        {
            type = "gauge",
            value = value,
            min = 0,
            max = 100,
            thresholds = new[] { 25, 50, 75 }
        };
    }
    
    private object GenerateTableData()
    {
        var random = new Random();
        var rows = new List<object>();
        
        for (int i = 0; i < 10; i++)
        {
            rows.Add(new
            {
                id = i + 1,
                name = $"Item {i + 1}",
                status = random.Next(0, 2) == 0 ? "Active" : "Inactive",
                value = random.Next(100, 1000)
            });
        }
        
        return new
        {
            type = "table",
            columns = new[] { "ID", "Name", "Status", "Value" },
            data = rows
        };
    }
    
    private object GenerateMetricData()
    {
        var random = new Random();
        var value = random.Next(1000, 10000);
        var previousValue = value + random.Next(-500, 500);
        var change = value - previousValue;
        var changePercent = (double)change / previousValue * 100;
        
        return new
        {
            type = "metric",
            value = value,
            previousValue = previousValue,
            change = change,
            changePercent = Math.Round(changePercent, 2),
            trend = change >= 0 ? "up" : "down"
        };
    }
}

public interface IWidget
{
    string Id { get; set; }
    string Name { get; set; }
    string Type { get; set; }
    WidgetConfiguration Configuration { get; set; }
    WidgetStatus Status { get; set; }
    DateTime LastDataUpdate { get; set; }
    
    Task InitializeAsync();
    Task<object> GetDataAsync();
    Task<bool> ValidateConfigurationAsync();
}

public class WidgetConfiguration
{
    public Dictionary<string, object> Settings { get; set; } = new();
    public string DataSource { get; set; }
    public Dictionary<string, object> Parameters { get; set; } = new();
    public TimeSpan? RefreshInterval { get; set; }
    public bool AutoRefresh { get; set; } = true;
}

public class WidgetData
{
    public string WidgetId { get; set; }
    public string WidgetType { get; set; }
    public object Data { get; set; }
    public DateTime LastUpdated { get; set; }
    public WidgetStatus Status { get; set; }
    public string Error { get; set; }
}

public enum WidgetStatus
{
    Inactive,
    Active,
    Loading,
    Error,
    Disabled
}

// Specific widget implementations
public class LineChartWidget : Widget
{
    public LineChartWidget()
    {
        Type = "line-chart";
    }
    
    public override async Task InitializeAsync()
    {
        Status = WidgetStatus.Active;
        await Task.CompletedTask;
    }
    
    public override async Task<object> GetDataAsync()
    {
        Status = WidgetStatus.Loading;
        
        try
        {
            var data = await GetDataFromProviderAsync(Configuration.DataSource, Configuration.Parameters);
            Status = WidgetStatus.Active;
            LastDataUpdate = DateTime.UtcNow;
            return data;
        }
        catch (Exception)
        {
            Status = WidgetStatus.Error;
            throw;
        }
    }
    
    public override async Task<bool> ValidateConfigurationAsync()
    {
        return await Task.FromResult(!string.IsNullOrEmpty(Configuration.DataSource));
    }
}

public class MetricWidget : Widget
{
    public MetricWidget()
    {
        Type = "metric";
    }
    
    public override async Task InitializeAsync()
    {
        Status = WidgetStatus.Active;
        await Task.CompletedTask;
    }
    
    public override async Task<object> GetDataAsync()
    {
        Status = WidgetStatus.Loading;
        
        try
        {
            var data = await GetDataFromProviderAsync(Configuration.DataSource, Configuration.Parameters);
            Status = WidgetStatus.Active;
            LastDataUpdate = DateTime.UtcNow;
            return data;
        }
        catch (Exception)
        {
            Status = WidgetStatus.Error;
            throw;
        }
    }
    
    public override async Task<bool> ValidateConfigurationAsync()
    {
        return await Task.FromResult(!string.IsNullOrEmpty(Configuration.DataSource));
    }
}
```

## Mülakat Soruları

### Temel Sorular

1. **Dashboard nedir?**
   - **Cevap**: Monitoring verilerini görsel olarak sunan arayüz.

2. **Widget nedir?**
   - **Cevap**: Dashboard'da belirli veri türünü gösteren bileşen.

3. **Real-time dashboard nedir?**
   - **Cevap**: Canlı veri güncellemeleri ile çalışan dashboard.

4. **Dashboard layout nedir?**
   - **Cevap**: Widget'ların dashboard'da nasıl yerleştirileceğini tanımlayan yapı.

5. **Data binding nedir?**
   - **Cevap**: Widget'ların veri kaynaklarına bağlanması.

### Teknik Sorular

1. **Dashboard engine nasıl implement edilir?**
   - **Cevap**: Dashboard management, widget lifecycle, data aggregation.

2. **Widget system nasıl tasarlanır?**
   - **Cevap**: Abstract base class, configuration management, data providers.

3. **Real-time updates nasıl implement edilir?**
   - **Cevap**: SignalR, WebSockets, polling, event-driven updates.

4. **Dashboard performance nasıl optimize edilir?**
   - **Cevap**: Data caching, lazy loading, efficient rendering, background updates.

5. **Responsive dashboard nasıl tasarlanır?**
   - **Cevap**: CSS Grid, Flexbox, mobile-first design, adaptive layouts.

## Best Practices

1. **Dashboard Design**
   - User-centric design kullanın
   - Consistent layout implement edin
   - Responsive design sağlayın
   - Accessibility standards uygulayın

2. **Widget Implementation**
   - Reusable components tasarlayın
   - Configuration-driven approach kullanın
   - Error handling implement edin
   - Performance optimization yapın

3. **Data Management**
   - Efficient data binding implement edin
   - Caching strategies uygulayın
   - Real-time updates sağlayın
   - Data validation ekleyin

4. **Performance & Scalability**
   - Lazy loading implement edin
   - Background processing kullanın
   - Resource optimization yapın
   - Monitoring ekleyin

5. **User Experience**
   - Intuitive navigation sağlayın
   - Customization options ekleyin
   - Interactive features implement edin
   - Mobile optimization yapın

## Kaynaklar

- [Dashboard Design](https://docs.microsoft.com/en-us/azure/azure-monitor/visualize/dashboards)
- [Data Visualization](https://docs.microsoft.com/en-us/azure/azure-monitor/visualize/charts)
- [Real-time Monitoring](https://docs.microsoft.com/en-us/azure/azure-monitor/app/live-stream)
- [Widget Development](https://docs.microsoft.com/en-us/azure/azure-monitor/visualize/partners)
- [Dashboard Best Practices](https://docs.microsoft.com/en-us/azure/azure-monitor/visualize/dashboard-design)
