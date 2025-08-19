# Assembly Loading

## Giriş

Assembly Loading, .NET runtime'da assembly'lerin yüklenmesi, yönetilmesi ve dinamik olarak çalıştırılması için kullanılan mekanizmalardır. Mid-level geliştiriciler için assembly loading'i anlamak, plugin architecture, dynamic code execution ve runtime extensibility için kritik öneme sahiptir. Bu dosya, assembly loading mechanisms, reflection, dynamic loading ve plugin systems konularını kapsar.

## Assembly Loading Mechanisms

### 1. Assembly Loader Service
Assembly loading'i yöneten servis.

```csharp
public class AssemblyLoaderService
{
    private readonly ILogger<AssemblyLoaderService> _logger;
    private readonly IConfiguration _configuration;
    private readonly Dictionary<string, Assembly> _loadedAssemblies;
    private readonly object _lock = new object();
    
    public AssemblyLoaderService(ILogger<AssemblyLoaderService> logger, IConfiguration configuration)
    {
        _logger = logger;
        _configuration = configuration;
        _loadedAssemblies = new Dictionary<string, Assembly>();
    }
    
    public Assembly LoadAssembly(string assemblyPath)
    {
        try
        {
            lock (_lock)
            {
                if (_loadedAssemblies.ContainsKey(assemblyPath))
                {
                    _logger.LogDebug("Assembly already loaded: {Path}", assemblyPath);
                    return _loadedAssemblies[assemblyPath];
                }
                
                _logger.LogInformation("Loading assembly: {Path}", assemblyPath);
                
                var assembly = Assembly.LoadFrom(assemblyPath);
                _loadedAssemblies[assemblyPath] = assembly;
                
                _logger.LogInformation("Assembly loaded successfully: {Name}, Version: {Version}", 
                    assembly.FullName, assembly.GetName().Version);
                
                return assembly;
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error loading assembly: {Path}", assemblyPath);
            throw;
        }
    }
    
    public Assembly LoadAssemblyByName(string assemblyName)
    {
        try
        {
            _logger.LogInformation("Loading assembly by name: {Name}", assemblyName);
            
            var assembly = Assembly.Load(assemblyName);
            
            _logger.LogInformation("Assembly loaded successfully: {Name}, Version: {Version}", 
                assembly.FullName, assembly.GetName().Version);
            
            return assembly;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error loading assembly by name: {Name}", assemblyName);
            throw;
        }
    }
    
    public Assembly LoadAssemblyFromBytes(byte[] assemblyBytes)
    {
        try
        {
            _logger.LogInformation("Loading assembly from bytes. Size: {Size} bytes", assemblyBytes.Length);
            
            var assembly = Assembly.Load(assemblyBytes);
            
            _logger.LogInformation("Assembly loaded successfully: {Name}, Version: {Version}", 
                assembly.FullName, assembly.GetName().Version);
            
            return assembly;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error loading assembly from bytes");
            throw;
        }
    }
    
    public void UnloadAssembly(string assemblyPath)
    {
        try
        {
            lock (_lock)
            {
                if (_loadedAssemblies.ContainsKey(assemblyPath))
                {
                    _loadedAssemblies.Remove(assemblyPath);
                    _logger.LogInformation("Assembly unloaded: {Path}", assemblyPath);
                }
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error unloading assembly: {Path}", assemblyPath);
        }
    }
    
    public List<AssemblyInfo> GetLoadedAssemblies()
    {
        lock (_lock)
        {
            return _loadedAssemblies.Values.Select(a => new AssemblyInfo
            {
                FullName = a.FullName,
                Location = a.Location,
                Version = a.GetName().Version?.ToString(),
                LoadedAt = DateTime.UtcNow
            }).ToList();
        }
    }
}

public class AssemblyInfo
{
    public string FullName { get; set; }
    public string Location { get; set; }
    public string Version { get; set; }
    public DateTime LoadedAt { get; set; }
}
```

### 2. Plugin System
Dynamic plugin loading implementasyonu.

```csharp
public interface IPlugin
{
    string Name { get; }
    string Version { get; }
    void Initialize();
    void Execute();
    void Cleanup();
}

public class PluginManager
{
    private readonly ILogger<PluginManager> _logger;
    private readonly IConfiguration _configuration;
    private readonly Dictionary<string, IPlugin> _plugins;
    private readonly AssemblyLoaderService _assemblyLoader;
    
    public PluginManager(ILogger<PluginManager> logger, IConfiguration configuration, AssemblyLoaderService assemblyLoader)
    {
        _logger = logger;
        _configuration = configuration;
        _assemblyLoader = assemblyLoader;
        _plugins = new Dictionary<string, IPlugin>();
    }
    
    public async Task LoadPluginsAsync(string pluginDirectory)
    {
        try
        {
            if (!Directory.Exists(pluginDirectory))
            {
                _logger.LogWarning("Plugin directory does not exist: {Directory}", pluginDirectory);
                return;
            }
            
            var pluginFiles = Directory.GetFiles(pluginDirectory, "*.dll");
            _logger.LogInformation("Found {Count} plugin files in {Directory}", pluginFiles.Length, pluginDirectory);
            
            foreach (var pluginFile in pluginFiles)
            {
                await LoadPluginAsync(pluginFile);
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error loading plugins from directory: {Directory}", pluginDirectory);
        }
    }
    
    private async Task LoadPluginAsync(string pluginPath)
    {
        try
        {
            _logger.LogInformation("Loading plugin: {Path}", pluginPath);
            
            var assembly = _assemblyLoader.LoadAssembly(pluginPath);
            var pluginTypes = assembly.GetTypes()
                .Where(t => typeof(IPlugin).IsAssignableFrom(t) && !t.IsInterface && !t.IsAbstract)
                .ToList();
            
            foreach (var pluginType in pluginTypes)
            {
                var plugin = (IPlugin)Activator.CreateInstance(pluginType);
                _plugins[plugin.Name] = plugin;
                
                plugin.Initialize();
                
                _logger.LogInformation("Plugin loaded successfully: {Name}, Version: {Version}", 
                    plugin.Name, plugin.Version);
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error loading plugin: {Path}", pluginPath);
        }
    }
    
    public IPlugin GetPlugin(string name)
    {
        return _plugins.TryGetValue(name, out var plugin) ? plugin : null;
    }
    
    public List<IPlugin> GetAllPlugins()
    {
        return _plugins.Values.ToList();
    }
    
    public void ExecutePlugin(string name)
    {
        try
        {
            var plugin = GetPlugin(name);
            if (plugin != null)
            {
                _logger.LogInformation("Executing plugin: {Name}", name);
                plugin.Execute();
            }
            else
            {
                _logger.LogWarning("Plugin not found: {Name}", name);
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error executing plugin: {Name}", name);
        }
    }
    
    public void UnloadAllPlugins()
    {
        try
        {
            foreach (var plugin in _plugins.Values)
            {
                plugin.Cleanup();
            }
            
            _plugins.Clear();
            _logger.LogInformation("All plugins unloaded");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error unloading plugins");
        }
    }
}
```

## Reflection and Dynamic Loading

### 1. Dynamic Type Loader
Runtime'da type'ları dinamik olarak yükleyen servis.

```csharp
public class DynamicTypeLoader
{
    private readonly ILogger<DynamicTypeLoader> _logger;
    
    public DynamicTypeLoader(ILogger<DynamicTypeLoader> logger)
    {
        _logger = logger;
    }
    
    public object CreateInstance(string typeName, params object[] constructorArgs)
    {
        try
        {
            _logger.LogDebug("Creating instance of type: {TypeName}", typeName);
            
            var type = Type.GetType(typeName);
            if (type == null)
            {
                throw new TypeLoadException($"Type not found: {typeName}");
            }
            
            var instance = Activator.CreateInstance(type, constructorArgs);
            
            _logger.LogDebug("Instance created successfully: {TypeName}", typeName);
            return instance;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating instance of type: {TypeName}", typeName);
            throw;
        }
    }
    
    public object InvokeMethod(object instance, string methodName, params object[] parameters)
    {
        try
        {
            _logger.LogDebug("Invoking method: {MethodName} on type: {TypeName}", 
                methodName, instance.GetType().Name);
            
            var method = instance.GetType().GetMethod(methodName);
            if (method == null)
            {
                throw new MissingMethodException($"Method not found: {methodName}");
            }
            
            var result = method.Invoke(instance, parameters);
            
            _logger.LogDebug("Method invoked successfully: {MethodName}", methodName);
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error invoking method: {MethodName}", methodName);
            throw;
        }
    }
    
    public object GetPropertyValue(object instance, string propertyName)
    {
        try
        {
            _logger.LogDebug("Getting property value: {PropertyName} from type: {TypeName}", 
                propertyName, instance.GetType().Name);
            
            var property = instance.GetType().GetProperty(propertyName);
            if (property == null)
            {
                throw new MissingMemberException($"Property not found: {propertyName}");
            }
            
            var value = property.GetValue(instance);
            
            _logger.LogDebug("Property value retrieved successfully: {PropertyName} = {Value}", 
                propertyName, value);
            
            return value;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting property value: {PropertyName}", propertyName);
            throw;
        }
    }
    
    public void SetPropertyValue(object instance, string propertyName, object value)
    {
        try
        {
            _logger.LogDebug("Setting property value: {PropertyName} = {Value} on type: {TypeName}", 
                propertyName, value, instance.GetType().Name);
            
            var property = instance.GetType().GetProperty(propertyName);
            if (property == null)
            {
                throw new MissingMemberException($"Property not found: {propertyName}");
            }
            
            property.SetValue(instance, value);
            
            _logger.LogDebug("Property value set successfully: {PropertyName}", propertyName);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error setting property value: {PropertyName}", propertyName);
            throw;
        }
    }
    
    public List<TypeInfo> GetTypesWithAttribute<TAttribute>() where TAttribute : Attribute
    {
        try
        {
            var types = new List<TypeInfo>();
            var assemblies = AppDomain.CurrentDomain.GetAssemblies();
            
            foreach (var assembly in assemblies)
            {
                try
                {
                    var assemblyTypes = assembly.GetTypes()
                        .Where(t => t.GetCustomAttribute<TAttribute>() != null)
                        .Select(t => new TypeInfo
                        {
                            FullName = t.FullName,
                            AssemblyName = assembly.FullName,
                            Attributes = t.GetCustomAttributes<TAttribute>().ToList()
                        });
                    
                    types.AddRange(assemblyTypes);
                }
                catch (Exception ex)
                {
                    _logger.LogWarning(ex, "Error processing assembly: {Assembly}", assembly.FullName);
                }
            }
            
            _logger.LogDebug("Found {Count} types with attribute {AttributeName}", 
                types.Count, typeof(TAttribute).Name);
            
            return types;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting types with attribute: {AttributeName}", typeof(TAttribute).Name);
            throw;
        }
    }
}

public class TypeInfo
{
    public string FullName { get; set; }
    public string AssemblyName { get; set; }
    public List<Attribute> Attributes { get; set; } = new();
}
```

## Mülakat Soruları

### Temel Sorular

1. **Assembly loading nedir?**
   - **Cevap**: .NET runtime'da assembly'lerin yüklenmesi ve yönetilmesi.

2. **Plugin system nedir?**
   - **Cevap**: Runtime'da dinamik olarak yüklenen ve çalıştırılan modüller.

3. **Reflection nedir?**
   - **Cevap**: Runtime'da type'lar hakkında bilgi alma ve dinamik olarak çalıştırma.

4. **Assembly.Load vs Assembly.LoadFrom farkı nedir?**
   - **Cevap**: Load: GAC'dan, LoadFrom: belirtilen path'den yükler.

5. **Dynamic loading ne zaman kullanılır?**
   - **Cevap**: Plugin architecture, runtime extensibility, configuration-based loading.

### Teknik Sorular

1. **Assembly loading nasıl implement edilir?**
   - **Cevap**: Assembly.Load, Assembly.LoadFrom, Assembly.LoadFile methods.

2. **Plugin system nasıl tasarlanır?**
   - **Cevap**: Interface-based design, assembly loading, dependency injection.

3. **Reflection performance nasıl optimize edilir?**
   - **Cevap**: Caching, compiled expressions, code generation.

4. **Assembly unloading nasıl yapılır?**
   - **Cevap**: AppDomain isolation, weak references, manual cleanup.

5. **Security assembly loading'de nasıl sağlanır?**
   - **Cevap**: Code signing, strong names, security policies.

## Best Practices

1. **Assembly Loading**
   - Appropriate loading method seçin
   - Error handling implement edin
   - Resource cleanup yapın
   - Security considerations göz önünde bulundurun

2. **Plugin Architecture**
   - Interface-based design kullanın
   - Dependency injection implement edin
   - Versioning support ekleyin
   - Error isolation sağlayın

3. **Reflection Usage**
   - Performance impact minimize edin
   - Caching strategies uygulayın
   - Error handling ekleyin
   - Security validation yapın

4. **Resource Management**
   - Proper disposal implement edin
   - Memory leaks prevent edin
   - Assembly unloading consider edin
   - Monitoring ekleyin

5. **Security**
   - Code signing validate edin
   - Strong names kullanın
   - Security policies implement edin
   - Input validation yapın

## Kaynaklar

- [Assembly Loading](https://docs.microsoft.com/en-us/dotnet/framework/deployment/best-practices-for-assembly-loading)
- [Reflection](https://docs.microsoft.com/en-us/dotnet/framework/reflection-and-codedom/reflection)
- [Plugin Architecture](https://docs.microsoft.com/en-us/dotnet/framework/add-ins/)
- [Dynamic Loading](https://docs.microsoft.com/en-us/dotnet/framework/reflection-and-codedom/how-to-load-assemblies-into-an-application-domain)
- [Assembly Security](https://docs.microsoft.com/en-us/dotnet/framework/misc/assembly-security-considerations)
