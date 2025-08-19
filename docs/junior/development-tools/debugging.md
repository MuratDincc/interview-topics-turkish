# Debugging

## Giriş

Debugging, yazılım geliştirmede hataları tespit etme ve düzeltme sürecidir. Etkili debugging becerileri, .NET geliştiricileri için kritik öneme sahiptir. Bu dosya, debugging tekniklerini, araçlarını ve en iyi uygulamalarını kapsar.

## Temel Kavramlar

### 1. Debugging Nedir?
Debugging, uygulamalardaki hataları (bug) tespit etme, analiz etme ve düzeltme sürecidir.

**Debugging Türleri:**
- Runtime debugging
- Static code analysis
- Unit test debugging
- Remote debugging
- Performance debugging

### 2. Common Bug Types
```csharp
// 1. Null Reference Exception
public void NullReferenceExample()
{
    string text = null;
    int length = text.Length; // NullReferenceException
}

// 2. Index Out of Range
public void IndexExample()
{
    int[] numbers = {1, 2, 3};
    int value = numbers[5]; // IndexOutOfRangeException
}

// 3. Logic Error
public void LogicError()
{
    for (int i = 0; i <= 10; i++) // Should be i < 10
    {
        Console.WriteLine(i);
    }
}
```

## Visual Studio Debugging Araçları

### 1. Breakpoints
```csharp
public class BreakpointExample
{
    public void Calculate()
    {
        int a = 10;
        int b = 20;
        int sum = a + b; // F9: Breakpoint koy
        
        Console.WriteLine($"Sum: {sum}");
    }
}
```

**Breakpoint Türleri:**
- **Standard Breakpoint**: F9
- **Conditional Breakpoint**: Sağ tık → Conditions
- **Tracepoint**: Sağ tık → Actions
- **Function Breakpoint**: Debug → New Breakpoint

### 2. Debug Windows
```csharp
public void DebugWindowsExample()
{
    var list = new List<int> {1, 2, 3, 4, 5};
    var evenNumbers = new List<int>();
    
    foreach (var number in list) // Debug sırasında
    {
        if (number % 2 == 0)     // Watch: number, list.Count
        {
            evenNumbers.Add(number); // Locals window'da değişkenleri gör
        }
    }
    
    Console.WriteLine($"Even count: {evenNumbers.Count}"); // Immediate: evenNumbers.Count
}
```

**Debug Windows:**
- **Locals**: Mevcut scope'daki değişkenler
- **Watch**: Belirli değişkenleri izleme
- **Call Stack**: Method çağrı zinciri
- **Immediate**: Debug sırasında kod çalıştırma
- **Output**: Debug çıktıları

### 3. Debug Navigation
```csharp
public class NavigationExample
{
    public void MainMethod()
    {
        int result = CalculateSum(10, 20); // F10: Step Over
        ProcessResult(result);             // F11: Step Into
    }
    
    private int CalculateSum(int a, int b)
    {
        return a + b; // Shift+F11: Step Out
    }
    
    private void ProcessResult(int result)
    {
        Console.WriteLine($"Result: {result}");
    }
}
```

**Navigation Komutları:**
- **F5**: Continue
- **F10**: Step Over
- **F11**: Step Into
- **Shift+F11**: Step Out
- **Ctrl+F10**: Run to Cursor

## Debugging Teknikleri

### 1. Console Debugging
```csharp
public class ConsoleDebugging
{
    public void ProcessData(List<string> data)
    {
        Console.WriteLine($"Processing {data.Count} items"); // Debug bilgisi
        
        for (int i = 0; i < data.Count; i++)
        {
            Console.WriteLine($"Processing item {i}: {data[i]}"); // İterasyon bilgisi
            
            try
            {
                var processed = ProcessItem(data[i]);
                Console.WriteLine($"Processed: {processed}"); // Sonuç bilgisi
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error processing item {i}: {ex.Message}"); // Hata bilgisi
            }
        }
        
        Console.WriteLine("Processing completed"); // Tamamlanma bilgisi
    }
    
    private string ProcessItem(string item)
    {
        return item.ToUpper();
    }
}
```

### 2. Logging ile Debugging
```csharp
using Microsoft.Extensions.Logging;

public class LoggingDebugging
{
    private readonly ILogger<LoggingDebugging> _logger;
    
    public LoggingDebugging(ILogger<LoggingDebugging> logger)
    {
        _logger = logger;
    }
    
    public async Task<List<User>> GetUsersAsync()
    {
        _logger.LogInformation("Starting GetUsersAsync method");
        
        try
        {
            var users = await _userRepository.GetAllAsync();
            _logger.LogInformation("Retrieved {UserCount} users", users.Count);
            
            var filteredUsers = users.Where(u => u.IsActive).ToList();
            _logger.LogDebug("Filtered to {ActiveUserCount} active users", filteredUsers.Count);
            
            return filteredUsers;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error occurred while getting users");
            throw;
        }
    }
}
```

### 3. Debug Attributes
```csharp
using System.Diagnostics;

public class DebugAttributeExample
{
    [Conditional("DEBUG")]
    public void DebugMethod()
    {
        Console.WriteLine("This only runs in Debug mode");
    }
    
    [DebuggerStepThrough]
    public string SimpleProperty => "This method will be skipped during debugging";
    
    [DebuggerDisplay("User: {Name}, Age: {Age}")]
    public class User
    {
        public string Name { get; set; }
        public int Age { get; set; }
    }
    
    [DebuggerBrowsable(DebuggerBrowsableState.Never)]
    private string _internalField;
}
```

## Exception Handling ve Debugging

### 1. Try-Catch Debugging
```csharp
public class ExceptionDebugging
{
    public void ProcessFile(string fileName)
    {
        try
        {
            var content = File.ReadAllText(fileName);
            var lines = content.Split('\n');
            
            foreach (var line in lines)
            {
                ProcessLine(line); // Hata burada olabilir
            }
        }
        catch (FileNotFoundException ex)
        {
            Console.WriteLine($"File not found: {ex.FileName}");
            // Breakpoint koy ve ex.StackTrace'i incele
        }
        catch (UnauthorizedAccessException ex)
        {
            Console.WriteLine($"Access denied: {ex.Message}");
            // Permission sorununu debug et
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Unexpected error: {ex.Message}");
            Console.WriteLine($"Stack trace: {ex.StackTrace}");
            // Genel hata debugging
        }
    }
    
    private void ProcessLine(string line)
    {
        if (string.IsNullOrEmpty(line))
            return;
            
        // Line processing logic
        var parts = line.Split(',');
        
        if (parts.Length < 3)
            throw new InvalidOperationException($"Invalid line format: {line}");
    }
}
```

### 2. Exception Settings
```csharp
// Debug → Windows → Exception Settings
// Common Language Runtime Exceptions
// - System.NullReferenceException ✓ (Break when thrown)
// - System.ArgumentException ✓
// - System.InvalidOperationException ✓
```

## Performance Debugging

### 1. Diagnostic Tools
```csharp
using System.Diagnostics;

public class PerformanceDebugging
{
    public void MeasurePerformance()
    {
        var stopwatch = Stopwatch.StartNew();
        
        // CPU-intensive operation
        var result = CalculateComplexOperation();
        
        stopwatch.Stop();
        Console.WriteLine($"Operation took: {stopwatch.ElapsedMilliseconds}ms");
        
        // Memory usage
        var memoryBefore = GC.GetTotalMemory(false);
        CreateLargeObject();
        var memoryAfter = GC.GetTotalMemory(false);
        
        Console.WriteLine($"Memory used: {memoryAfter - memoryBefore} bytes");
    }
    
    private int CalculateComplexOperation()
    {
        // Simulate complex calculation
        int result = 0;
        for (int i = 0; i < 1000000; i++)
        {
            result += i * 2;
        }
        return result;
    }
    
    private void CreateLargeObject()
    {
        var largeArray = new int[100000];
        // Use array...
    }
}
```

### 2. PerfView ve dotTrace
```csharp
// Performance profiling için external tools
// - PerfView (Microsoft)
// - dotTrace (JetBrains)
// - Application Insights
// - MiniProfiler
```

## Remote Debugging

### 1. Remote Debugging Setup
```xml
<!-- appsettings.json -->
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft": "Warning"
    }
  }
}
```

```csharp
// Program.cs
public class Program
{
    public static void Main(string[] args)
    {
        #if DEBUG
        // Wait for debugger attachment
        if (args.Contains("--wait-for-debugger"))
        {
            while (!Debugger.IsAttached)
            {
                Thread.Sleep(100);
            }
        }
        #endif
        
        CreateHostBuilder(args).Build().Run();
    }
}
```

## Debugging Best Practices

### 1. Systematic Approach
```csharp
public class DebuggingBestPractices
{
    // 1. Reproduce the bug consistently
    public void ReproduceBug()
    {
        // Create minimal test case
        // Document steps to reproduce
        // Use consistent test data
    }
    
    // 2. Isolate the problem
    public void IsolateProblem()
    {
        // Use binary search approach
        // Comment out code sections
        // Add debug output at key points
    }
    
    // 3. Use appropriate debugging tools
    public void UseRightTools()
    {
        // Breakpoints for logic errors
        // Logging for production issues
        // Profiling for performance issues
        // Static analysis for code quality
    }
}
```

### 2. Defensive Programming
```csharp
public class DefensiveProgramming
{
    public void ProcessUser(User user)
    {
        // Input validation
        if (user == null)
            throw new ArgumentNullException(nameof(user));
            
        if (string.IsNullOrEmpty(user.Name))
            throw new ArgumentException("User name cannot be empty", nameof(user));
        
        // Guard clauses
        Debug.Assert(user.Age >= 0, "User age should be non-negative");
        
        // Logging
        Console.WriteLine($"Processing user: {user.Name}");
        
        try
        {
            // Main logic
            ProcessUserInternal(user);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error processing user {user.Name}: {ex.Message}");
            throw; // Re-throw for caller to handle
        }
    }
    
    private void ProcessUserInternal(User user)
    {
        // Implementation...
    }
}
```

## Mülakat Soruları

### Temel Sorular

1. **Debugging nedir ve neden önemlidir?**
   - **Cevap**: Hataları tespit etme ve düzeltme süreci, kod kalitesi için kritik.

2. **Breakpoint nedir ve nasıl kullanılır?**
   - **Cevap**: Kodun durmasını sağlayan nokta, F9 ile eklenir/kaldırılır.

3. **Step Into, Step Over ve Step Out arasındaki fark nedir?**
   - **Cevap**: Step Into method'a girer, Step Over atlar, Step Out çıkar.

4. **Watch window ne işe yarar?**
   - **Cevap**: Belirli değişkenlerin değerlerini debug sırasında izleme.

5. **Exception handling debugging'de nasıl kullanılır?**
   - **Cevap**: Try-catch blokları ile hataları yakalama ve analiz etme.

### Teknik Sorular

1. **Conditional breakpoint nasıl kurulur?**
   - **Cevap**: Breakpoint'e sağ tık → Conditions → koşul ekleme.

2. **Remote debugging nasıl yapılır?**
   - **Cevap**: Remote debugger tool ile uzak makinedeki uygulamaya bağlanma.

3. **Performance debugging araçları nelerdir?**
   - **Cevap**: Diagnostic Tools, PerfView, dotTrace, memory profilers.

4. **Debug vs Release build farkları nelerdir?**
   - **Cevap**: Debug optimizasyon yok, Release optimize edilmiş.

## Best Practices

### 1. **Systematic Debugging**
- Problemi reproduce edin
- Minimal test case oluşturun
- Binary search approach kullanın
- Değişiklikleri document edin

### 2. **Tool Usage**
- Doğru debugging tool'u seçin
- Breakpoint'leri strategic yerlere koyun
- Watch expressions etkili kullanın
- Logging ile production debugging

### 3. **Prevention**
- Defensive programming yapın
- Unit test yazın
- Code review yapın
- Static analysis kullanın

### 4. **Documentation**
- Bug report'ları detaylı yazın
- Çözüm sürecini document edin
- Knowledge sharing yapın
- Pattern'leri paylaşın

## Kaynaklar

- [Visual Studio Debugging](https://docs.microsoft.com/en-us/visualstudio/debugger/)
- [.NET Debugging Guide](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/)
- [Debugging Best Practices](https://docs.microsoft.com/en-us/visualstudio/debugger/debugging-techniques-and-tools)
- [Exception Handling](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/exceptions/)
- [Performance Debugging](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/debug-highcpu)
