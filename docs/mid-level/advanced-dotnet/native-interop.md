# Native Interop

## Giriş

Native Interop, .NET uygulamalarının native (unmanaged) code ile etkileşim kurmasını sağlayan teknolojidir. Mid-level geliştiriciler için native interop'u anlamak, performance-critical operations, legacy system integration ve platform-specific functionality için kritik öneme sahiptir. Bu dosya, P/Invoke, COM interop, unsafe code ve performance optimization konularını kapsar.

## P/Invoke

### 1. Windows API Integration
Windows API'ları ile P/Invoke kullanarak entegrasyon.

```csharp
public class WindowsApiService
{
    private readonly ILogger<WindowsApiService> _logger;
    
    public WindowsApiService(ILogger<WindowsApiService> logger)
    {
        _logger = logger;
    }
    
    [DllImport("kernel32.dll", SetLastError = true)]
    private static extern IntPtr GetCurrentProcess();
    
    [DllImport("kernel32.dll", SetLastError = true)]
    private static extern bool GetProcessMemoryInfo(IntPtr process, out PROCESS_MEMORY_COUNTERS_EX counters, uint cb);
    
    [DllImport("kernel32.dll", SetLastError = true)]
    private static extern bool SetProcessWorkingSetSize(IntPtr process, IntPtr minimumWorkingSetSize, IntPtr maximumWorkingSetSize);
    
    [DllImport("user32.dll")]
    private static extern bool GetWindowRect(IntPtr hWnd, out RECT lpRect);
    
    [DllImport("user32.dll")]
    private static extern bool SetWindowPos(IntPtr hWnd, IntPtr hWndInsertAfter, int X, int Y, int cx, int cy, uint uFlags);
    
    public ProcessMemoryInfo GetProcessMemoryInfo()
    {
        try
        {
            var process = GetCurrentProcess();
            var result = GetProcessMemoryInfo(process, out PROCESS_MEMORY_COUNTERS_EX counters, (uint)Marshal.SizeOf<PROCESS_MEMORY_COUNTERS_EX>());
            
            if (result)
            {
                return new ProcessMemoryInfo
                {
                    WorkingSetSize = counters.WorkingSetSize,
                    PeakWorkingSetSize = counters.PeakWorkingSetSize,
                    PageFileUsage = counters.PagefileUsage,
                    PeakPageFileUsage = counters.PeakPagefileUsage,
                    PrivateUsage = counters.PrivateUsage
                };
            }
            
            _logger.LogWarning("Failed to get process memory info. Error: {Error}", Marshal.GetLastWin32Error());
            return null;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting process memory info");
            return null;
        }
    }
    
    public bool SetProcessWorkingSetSize(long minimumSize, long maximumSize)
    {
        try
        {
            var process = GetCurrentProcess();
            var result = SetProcessWorkingSetSize(process, (IntPtr)minimumSize, (IntPtr)maximumSize);
            
            if (result)
            {
                _logger.LogInformation("Process working set size set to {Min} - {Max} bytes", minimumSize, maximumSize);
            }
            else
            {
                _logger.LogWarning("Failed to set process working set size. Error: {Error}", Marshal.GetLastWin32Error());
            }
            
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error setting process working set size");
            return false;
        }
    }
    
    public WindowInfo GetWindowInfo(IntPtr windowHandle)
    {
        try
        {
            var result = GetWindowRect(windowHandle, out RECT rect);
            
            if (result)
            {
                return new WindowInfo
                {
                    Left = rect.Left,
                    Top = rect.Top,
                    Right = rect.Right,
                    Bottom = rect.Bottom,
                    Width = rect.Right - rect.Left,
                    Height = rect.Bottom - rect.Top
                };
            }
            
            _logger.LogWarning("Failed to get window info. Error: {Error}", Marshal.GetLastWin32Error());
            return null;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting window info");
            return null;
        }
    }
    
    public bool SetWindowPosition(IntPtr windowHandle, int x, int y, int width, int height)
    {
        try
        {
            const uint SWP_NOZORDER = 0x0004;
            var result = SetWindowPos(windowHandle, IntPtr.Zero, x, y, width, height, SWP_NOZORDER);
            
            if (result)
            {
                _logger.LogInformation("Window position set to ({X}, {Y}) with size {Width}x{Height}", x, y, width, height);
            }
            else
            {
                _logger.LogWarning("Failed to set window position. Error: {Error}", Marshal.GetLastWin32Error());
            }
            
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error setting window position");
            return false;
        }
    }
}

[StructLayout(LayoutKind.Sequential)]
public struct PROCESS_MEMORY_COUNTERS_EX
{
    public uint cb;
    public uint PageFaultCount;
    public ulong PeakWorkingSetSize;
    public ulong WorkingSetSize;
    public ulong QuotaPeakPagedPoolUsage;
    public ulong QuotaPagedPoolUsage;
    public ulong QuotaPeakNonPagedPoolUsage;
    public ulong QuotaNonPagedPoolUsage;
    public ulong PagefileUsage;
    public ulong PeakPagefileUsage;
    public ulong PrivateUsage;
}

[StructLayout(LayoutKind.Sequential)]
public struct RECT
{
    public int Left;
    public int Top;
    public int Right;
    public int Bottom;
}

public class ProcessMemoryInfo
{
    public ulong WorkingSetSize { get; set; }
    public ulong PeakWorkingSetSize { get; set; }
    public ulong PageFileUsage { get; set; }
    public ulong PeakPageFileUsage { get; set; }
    public ulong PrivateUsage { get; set; }
}

public class WindowInfo
{
    public int Left { get; set; }
    public int Top { get; set; }
    public int Right { get; set; }
    public int Bottom { get; set; }
    public int Width { get; set; }
    public int Height { get; set; }
}
```

### 2. Custom Native Library Integration
Custom native library'ler ile entegrasyon.

```csharp
public class NativeLibraryService
{
    private readonly ILogger<NativeLibraryService> _logger;
    private readonly string _libraryPath;
    
    public NativeLibraryService(ILogger<NativeLibraryService> logger, IConfiguration configuration)
    {
        _logger = logger;
        _libraryPath = configuration["NativeLibrary:Path"] ?? "native_lib.dll";
    }
    
    [DllImport("native_lib.dll", CallingConvention = CallingConvention.Cdecl)]
    private static extern int Add(int a, int b);
    
    [DllImport("native_lib.dll", CallingConvention = CallingConvention.Cdecl)]
    private static extern double Multiply(double a, double b);
    
    [DllImport("native_lib.dll", CallingConvention = CallingConvention.Cdecl)]
    private static extern IntPtr CreateString([MarshalAs(UnmanagedType.LPStr)] string text);
    
    [DllImport("native_lib.dll", CallingConvention = CallingConvention.Cdecl)]
    private static extern void FreeString(IntPtr ptr);
    
    [DllImport("native_lib.dll", CallingConvention = CallingConvention.Cdecl)]
    private static extern int ProcessArray([In, Out] int[] array, int length);
    
    [DllImport("native_lib.dll", CallingConvention = CallingConvention.Cdecl)]
    private static extern bool ProcessStruct([In, Out] ref CustomStruct data);
    
    public int AddNumbers(int a, int b)
    {
        try
        {
            var result = Add(a, b);
            _logger.LogDebug("Native Add({A}, {B}) = {Result}", a, b, result);
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error calling native Add function");
            throw;
        }
    }
    
    public double MultiplyNumbers(double a, double b)
    {
        try
        {
            var result = Multiply(a, b);
            _logger.LogDebug("Native Multiply({A}, {B}) = {Result}", a, b, result);
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error calling native Multiply function");
            throw;
        }
    }
    
    public string ProcessString(string text)
    {
        IntPtr nativeString = IntPtr.Zero;
        
        try
        {
            nativeString = CreateString(text);
            var result = Marshal.PtrToStringAnsi(nativeString);
            
            _logger.LogDebug("Native string processing: {Input} -> {Output}", text, result);
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing string with native library");
            throw;
        }
        finally
        {
            if (nativeString != IntPtr.Zero)
            {
                FreeString(nativeString);
            }
        }
    }
    
    public int[] ProcessArray(int[] array)
    {
        try
        {
            var result = ProcessArray(array, array.Length);
            
            if (result == 0)
            {
                _logger.LogDebug("Native array processing completed successfully");
                return array;
            }
            else
            {
                _logger.LogWarning("Native array processing failed with code: {Code}", result);
                return null;
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing array with native library");
            throw;
        }
    }
    
    public CustomStruct ProcessStruct(CustomStruct data)
    {
        try
        {
            var result = ProcessStruct(ref data);
            
            if (result)
            {
                _logger.LogDebug("Native struct processing completed successfully");
                return data;
            }
            else
            {
                _logger.LogWarning("Native struct processing failed");
                return data;
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing struct with native library");
            throw;
        }
    }
}

[StructLayout(LayoutKind.Sequential)]
public struct CustomStruct
{
    public int Id;
    [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 256)]
    public string Name;
    public double Value;
    public bool Flag;
}
```

## Unsafe Code

### 1. Pointer Operations
Unsafe code ile pointer operations.

```csharp
public unsafe class UnsafeOperations
{
    private readonly ILogger<UnsafeOperations> _logger;
    
    public UnsafeOperations(ILogger<UnsafeOperations> logger)
    {
        _logger = logger;
    }
    
    public unsafe void ProcessArrayWithPointers(int[] array)
    {
        try
        {
            fixed (int* ptr = array)
            {
                var length = array.Length;
                
                // Process array using pointers
                for (int i = 0; i < length; i++)
                {
                    *(ptr + i) = *(ptr + i) * 2;
                }
                
                _logger.LogDebug("Array processed using unsafe pointers. Length: {Length}", length);
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing array with unsafe pointers");
            throw;
        }
    }
    
    public unsafe void CopyMemory(byte[] source, byte[] destination)
    {
        try
        {
            if (source.Length > destination.Length)
                throw new ArgumentException("Destination array is too small");
            
            fixed (byte* srcPtr = source, dstPtr = destination)
            {
                Buffer.MemoryCopy(srcPtr, dstPtr, destination.Length, source.Length);
            }
            
            _logger.LogDebug("Memory copied using unsafe operations. Size: {Size} bytes", source.Length);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error copying memory with unsafe operations");
            throw;
        }
    }
    
    public unsafe void ProcessStructWithPointer(ref CustomStruct data)
    {
        try
        {
            fixed (CustomStruct* ptr = &data)
            {
                // Access struct fields using pointers
                ptr->Id = ptr->Id + 1;
                ptr->Value = ptr->Value * 1.5;
                ptr->Flag = !ptr->Flag;
                
                _logger.LogDebug("Struct processed using unsafe pointer. ID: {Id}, Value: {Value}", 
                    ptr->Id, ptr->Value);
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing struct with unsafe pointer");
            throw;
        }
    }
    
    public unsafe int FindPattern(byte[] data, byte[] pattern)
    {
        try
        {
            fixed (byte* dataPtr = data, patternPtr = pattern)
            {
                var dataLength = data.Length;
                var patternLength = pattern.Length;
                
                for (int i = 0; i <= dataLength - patternLength; i++)
                {
                    var found = true;
                    
                    for (int j = 0; j < patternLength; j++)
                    {
                        if (*(dataPtr + i + j) != *(patternPtr + j))
                        {
                            found = false;
                            break;
                        }
                    }
                    
                    if (found)
                    {
                        _logger.LogDebug("Pattern found at index {Index}", i);
                        return i;
                    }
                }
                
                _logger.LogDebug("Pattern not found in data");
                return -1;
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error finding pattern with unsafe operations");
            throw;
        }
    }
}
```

## Mülakat Soruları

### Temel Sorular

1. **P/Invoke nedir ve ne için kullanılır?**
   - **Cevap**: Platform Invoke, native libraries ile .NET entegrasyonu.

2. **Unsafe code nedir?**
   - **Cevap**: Pointer operations, direct memory access, performance optimization.

3. **COM Interop nedir?**
   - **Cevap**: Component Object Model, legacy COM components ile entegrasyon.

4. **Native interop ne zaman kullanılır?**
   - **Cevap**: Performance-critical operations, platform-specific features, legacy integration.

5. **Marshaling nedir?**
   - **Cevap**: Data conversion between managed and unmanaged code.

### Teknik Sorular

1. **P/Invoke ile native function nasıl çağrılır?**
   - **Cevap**: DllImport attribute, function signature, marshaling.

2. **Unsafe code ile pointer operations nasıl yapılır?**
   - **Cevap**: Fixed statement, pointer arithmetic, memory manipulation.

3. **Memory leaks native interop'ta nasıl önlenir?**
   - **Cevap**: Proper cleanup, using statements, resource management.

4. **Performance optimization native interop'ta nasıl yapılır?**
   - **Cevap**: Minimize marshaling, use value types, avoid allocations.

5. **Error handling native interop'ta nasıl yapılır?**
   - **Cevap**: SetLastError, exception handling, return value checking.

## Best Practices

1. **P/Invoke Usage**
   - Correct function signatures kullanın
   - Proper marshaling implement edin
   - Error handling ekleyin
   - Resource cleanup yapın

2. **Unsafe Code**
   - Minimize unsafe blocks
   - Proper bounds checking
   - Memory safety ensure edin
   - Performance benefits validate edin

3. **Error Handling**
   - SetLastError check edin
   - Exception handling implement edin
   - Resource cleanup guarantee edin
   - Logging ekleyin

4. **Performance Optimization**
   - Marshaling minimize edin
   - Value types kullanın
   - Allocations avoid edin
   - Benchmarking yapın

5. **Security**
   - Input validation yapın
   - Buffer overflow prevent edin
   - Trusted libraries kullanın
   - Security review yapın

## Kaynaklar

- [P/Invoke](https://docs.microsoft.com/en-us/dotnet/standard/native-interop/pinvoke)
- [Unsafe Code](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/unsafe-code)
- [COM Interop](https://docs.microsoft.com/en-us/dotnet/standard/native-interop/com-interop)
- [Native Interop](https://docs.microsoft.com/en-us/dotnet/standard/native-interop/)
- [Performance Considerations](https://docs.microsoft.com/en-us/dotnet/standard/native-interop/pinvoke#performance-considerations)
