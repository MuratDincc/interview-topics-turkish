# Temel .NET Kavramları

## Genel Bakış
Bu bölüm, .NET geliştiricilerinin bilmesi gereken temel kavramları kapsar. CLR, managed/unmanaged code, assembly ve namespace gibi temel yapıları öğreneceksiniz.

## İçindekiler
1. [.NET Framework vs .NET Core](framework-vs-core.md)
2. [CLR (Common Language Runtime)](clr.md)
3. [Managed ve Unmanaged Code](managed-unmanaged.md)
4. [Assembly ve Namespace](assembly-namespace.md)
5. [Garbage Collection](garbage-collection.md)

## Mülakat Soruları ve Cevapları

### 1. .NET Framework ve .NET Core arasındaki temel farklar nelerdir?
**Cevap:**
- .NET Framework Windows'a özgüdür, .NET Core çapraz platformdur
- .NET Core daha modüler ve hafiftir
- .NET Core daha hızlı performans sunar
- .NET Core container desteği vardır
- .NET Core açık kaynak kodludur

**Örnek Kod:**
```csharp
// .NET Framework'te Windows Forms uygulaması
public class WindowsFormApp : Form
{
    public WindowsFormApp()
    {
        this.Text = "Windows Forms App";
    }
}

// .NET Core'da Console uygulaması
public class Program
{
    public static void Main(string[] args)
    {
        Console.WriteLine("Cross-platform .NET Core App");
    }
}
```

### 2. CLR nedir ve nasıl çalışır?
**Cevap:**
CLR (Common Language Runtime), .NET uygulamalarının çalışma zamanı ortamıdır. Temel görevleri:
- Memory management
- Type safety
- Exception handling
- Security
- Thread management

**Örnek Kod:**
```csharp
// CLR tarafından yönetilen memory allocation
public class MemoryExample
{
    public void AllocateMemory()
    {
        // CLR heap'te yer ayırır
        var list = new List<int>();
        
        // CLR garbage collection yapar
        list = null;
    }
}
```

### 3. Managed ve Unmanaged Code arasındaki farklar nelerdir?
**Cevap:**
Managed Code:
- CLR tarafından yönetilir
- Otomatik memory management
- Type safety
- Exception handling

Unmanaged Code:
- İşletim sistemi tarafından yönetilir
- Manuel memory management
- Daha hızlı performans
- Daha fazla kontrol

**Örnek Kod:**
```csharp
// Managed Code
public class ManagedExample
{
    public void ManagedMethod()
    {
        var managedResource = new Resource();
        // CLR otomatik olarak memory'yi yönetir
    }
}

// Unmanaged Code
public class UnmanagedExample
{
    [DllImport("user32.dll")]
    public static extern int MessageBox(IntPtr hWnd, String text, String caption, uint type);
    
    public void UnmanagedMethod()
    {
        // Windows API çağrısı - unmanaged code
        MessageBox(IntPtr.Zero, "Hello", "Message", 0);
    }
}
```

### 4. Assembly ve Namespace kavramlarını açıklayınız.
**Cevap:**
Assembly:
- .NET'te dağıtım birimi
- DLL veya EXE dosyası
- Metadata içerir
- Version bilgisi taşır

Namespace:
- Kod organizasyonu sağlar
- İsim çakışmalarını önler
- Hiyerarşik yapı sunar
- Kod okunabilirliğini artırır

**Örnek Kod:**
```csharp
// Assembly: MyApp.dll
namespace MyApp.Data
{
    public class Database
    {
        // Database işlemleri
    }
}

namespace MyApp.Services
{
    public class UserService
    {
        // Servis işlemleri
    }
}
```

### 5. Garbage Collection nasıl çalışır?
**Cevap:**
Garbage Collection:
- Otomatik memory yönetimi
- Heap'teki kullanılmayan nesneleri temizler
- Generations (0, 1, 2) kullanır
- Finalization queue yönetir

**Örnek Kod:**
```csharp
public class GarbageCollectionExample
{
    public void CreateAndCollect()
    {
        // Generation 0'da nesne oluşturulur
        var obj = new LargeObject();
        
        // Nesne kullanılmaz hale gelir
        obj = null;
        
        // Garbage Collection tetiklenir
        GC.Collect();
        
        // Finalization queue kontrolü
        GC.WaitForPendingFinalizers();
    }
}
```

## Best Practices
1. **Memory Yönetimi**
   - IDisposable pattern kullanımı
   - using statement kullanımı
   - Large object heap yönetimi
   - Finalization'dan kaçınma

2. **Performans**
   - Boxing/unboxing'den kaçınma
   - String concatenation yerine StringBuilder
   - Collection initializers kullanımı
   - Async/await pattern kullanımı

3. **Güvenlik**
   - Code access security
   - Role-based security
   - Cryptography kullanımı
   - Secure string handling

## Kaynaklar
- [.NET Documentation](https://docs.microsoft.com/tr-tr/dotnet/)
- [CLR via C#](https://www.amazon.com/CLR-via-4th-Developer-Reference/dp/0735667454)
- [Pro .NET Memory Management](https://www.apress.com/gp/book/9781484240267) 