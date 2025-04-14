# Managed ve Unmanaged Code

## Genel Bakış
Bu bölümde, .NET'te managed ve unmanaged code kavramlarını, aralarındaki farkları ve kullanım senaryolarını inceleyeceğiz.

## Mülakat Soruları ve Cevapları

### 1. Managed ve Unmanaged Code arasındaki temel farklar nelerdir?
**Cevap:**
**Managed Code:**
- CLR tarafından yönetilir
- Otomatik memory management
- Type safety
- Exception handling
- Garbage collection
- Platform bağımsızlık

**Unmanaged Code:**
- İşletim sistemi tarafından yönetilir
- Manuel memory management
- Daha hızlı performans
- Daha fazla kontrol
- Native kod
- Platform bağımlılık

**Örnek Kod:**
```csharp
// Managed Code örneği
public class ManagedExample
{
    public void ManagedMethod()
    {
        // CLR tarafından yönetilen kaynak
        var list = new List<int>();
        // Otomatik memory yönetimi
        list.Add(1);
    }
}

// Unmanaged Code örneği
public class UnmanagedExample
{
    [DllImport("kernel32.dll")]
    public static extern IntPtr HeapAlloc(IntPtr hHeap, uint dwFlags, UIntPtr dwBytes);

    public void UnmanagedMethod()
    {
        // Manuel memory yönetimi
        IntPtr memory = HeapAlloc(GetProcessHeap(), 0, (UIntPtr)1000);
        // Belleği serbest bırakma sorumluluğu geliştiricide
    }
}
```

### 2. P/Invoke nedir ve nasıl kullanılır?
**Cevap:**
P/Invoke (Platform Invocation Services):
- Unmanaged DLL'leri çağırmak için kullanılır
- DllImport attribute'u ile tanımlanır
- Marshaling işlemleri yapar
- Memory yönetimi gerektirir

**Örnek Kod:**
```csharp
public class PInvokeExample
{
    // Windows API çağrısı
    [DllImport("user32.dll", CharSet = CharSet.Auto)]
    public static extern int MessageBox(IntPtr hWnd, string text, string caption, uint type);

    // Custom DLL çağrısı
    [DllImport("MyNativeLib.dll")]
    public static extern int ProcessData(byte[] data, int length);

    public void ShowMessage()
    {
        // Unmanaged kod çağrısı
        MessageBox(IntPtr.Zero, "Merhaba", "Mesaj", 0);
    }
}
```

### 3. COM Interop nedir ve nasıl kullanılır?
**Cevap:**
COM Interop:
- COM bileşenlerini .NET'te kullanmayı sağlar
- Type library import
- Runtime Callable Wrapper (RCW)
- COM Callable Wrapper (CCW)

**Örnek Kod:**
```csharp
// COM bileşeni kullanımı
public class ComInteropExample
{
    public void UseComComponent()
    {
        // COM bileşeni oluşturma
        var excel = new Microsoft.Office.Interop.Excel.Application();
        
        try
        {
            // COM bileşeni kullanımı
            excel.Visible = true;
            var workbook = excel.Workbooks.Add();
            // ...
        }
        finally
        {
            // COM nesnelerini temizleme
            System.Runtime.InteropServices.Marshal.ReleaseComObject(excel);
        }
    }
}
```

### 4. Safe ve Unsafe Code nedir?
**Cevap:**
**Safe Code:**
- CLR tarafından tam kontrol
- Type safety
- Memory safety
- Exception handling

**Unsafe Code:**
- Pointer kullanımı
- Memory adreslerine direkt erişim
- unsafe keyword'ü gerektirir
- Performans optimizasyonu

**Örnek Kod:**
```csharp
public class SafeUnsafeExample
{
    // Safe code
    public void SafeMethod()
    {
        var array = new int[10];
        array[0] = 42; // CLR bounds checking
    }

    // Unsafe code
    public unsafe void UnsafeMethod()
    {
        int[] array = new int[10];
        fixed (int* ptr = array)
        {
            // Pointer aritmetiği
            *(ptr + 1) = 42;
        }
    }
}
```

### 5. Memory Management farklılıkları nelerdir?
**Cevap:**
**Managed Memory:**
- CLR tarafından yönetilir
- Garbage collection
- Heap allocation
- Reference counting

**Unmanaged Memory:**
- Manuel yönetim
- malloc/free
- Heap/stack allocation
- Memory leaks riski

**Örnek Kod:**
```csharp
public class MemoryManagementExample
{
    // Managed memory
    public void ManagedMemory()
    {
        var list = new List<int>();
        // CLR memory'yi yönetir
        list.Add(1);
        // Garbage collection otomatik temizler
    }

    // Unmanaged memory
    public unsafe void UnmanagedMemory()
    {
        // Manuel memory allocation
        int* ptr = (int*)Marshal.AllocHGlobal(sizeof(int));
        try
        {
            *ptr = 42;
            // Memory kullanımı
        }
        finally
        {
            // Manuel temizleme
            Marshal.FreeHGlobal((IntPtr)ptr);
        }
    }
}
```

## Best Practices
1. **Managed Code Kullanımı**
   - IDisposable pattern
   - using statement
   - Exception handling
   - Resource cleanup

2. **Unmanaged Code Kullanımı**
   - Memory leaks önleme
   - Error handling
   - Resource cleanup
   - Thread safety

3. **Interop Kullanımı**
   - Marshaling optimizasyonu
   - COM nesnelerinin temizlenmesi
   - Exception handling
   - Performance monitoring

## Kaynaklar
- [P/Invoke Documentation](https://docs.microsoft.com/tr-tr/dotnet/standard/native-interop/pinvoke)
- [COM Interop](https://docs.microsoft.com/tr-tr/dotnet/standard/native-interop/com-interop)
- [Unsafe Code](https://docs.microsoft.com/tr-tr/dotnet/csharp/language-reference/unsafe-code) 