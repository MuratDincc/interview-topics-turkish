# Assembly ve Namespace

## Genel Bakış
Bu bölümde, .NET'te assembly ve namespace kavramlarını, bunların yapısını ve kullanımını inceleyeceğiz.

## Mülakat Soruları ve Cevapları

### 1. Assembly nedir ve temel özellikleri nelerdir?
**Cevap:**
Assembly, .NET'te dağıtım birimidir. Temel özellikleri:
- DLL veya EXE dosyası olarak dağıtılır
- Metadata içerir (IL kodu, tip bilgileri)
- Version bilgisi taşır
- Güvenlik sınırı oluşturur
- Tip yükleme birimidir

**Örnek Kod:**
```csharp
// Assembly bilgilerini okuma
public class AssemblyExample
{
    public void GetAssemblyInfo()
    {
        // Mevcut assembly'yi alma
        var assembly = Assembly.GetExecutingAssembly();
        
        // Assembly bilgilerini yazdırma
        Console.WriteLine($"Assembly Name: {assembly.GetName().Name}");
        Console.WriteLine($"Version: {assembly.GetName().Version}");
        Console.WriteLine($"Location: {assembly.Location}");
    }
}
```

### 2. Strong Name nedir ve nasıl kullanılır?
**Cevap:**
Strong Name:
- Assembly'ye benzersiz kimlik verir
- Public/private key çifti kullanır
- Version kontrolü sağlar
- Güvenlik sağlar

**Örnek Kod:**
```csharp
// Strong name ile assembly oluşturma
[assembly: AssemblyKeyFile("MyKey.snk")]
[assembly: AssemblyVersion("1.0.0.0")]

public class StrongNameExample
{
    public void VerifyStrongName()
    {
        var assembly = Assembly.GetExecutingAssembly();
        var name = assembly.GetName();
        
        // Strong name kontrolü
        if (name.GetPublicKey().Length > 0)
        {
            Console.WriteLine("Assembly strong name ile imzalanmış");
        }
    }
}
```

### 3. Namespace nedir ve nasıl kullanılır?
**Cevap:**
Namespace:
- Kod organizasyonu sağlar
- İsim çakışmalarını önler
- Hiyerarşik yapı sunar
- Kod okunabilirliğini artırır

**Örnek Kod:**
```csharp
// Namespace kullanımı
namespace MyApp.Data
{
    public class Database
    {
        public void Connect()
        {
            // Veritabanı bağlantısı
        }
    }
}

namespace MyApp.Services
{
    public class UserService
    {
        private readonly Data.Database _db;
        
        public UserService()
        {
            _db = new Data.Database();
        }
    }
}
```

### 4. Assembly yükleme süreci nasıl çalışır?
**Cevap:**
Assembly yükleme süreci:
1. Fusion (Assembly resolver) devreye girer
2. GAC kontrolü yapılır
3. Probing yapılır (bin, private path)
4. Assembly manifest kontrolü
5. Tip yükleme

**Örnek Kod:**
```csharp
public class AssemblyLoading
{
    public void LoadAssembly()
    {
        // Assembly yükleme
        var assembly = Assembly.Load("MyLibrary");
        
        // Tip yükleme
        var type = assembly.GetType("MyLibrary.MyClass");
        
        // Instance oluşturma
        var instance = Activator.CreateInstance(type);
    }
    
    // Custom assembly resolver
    static AssemblyLoading()
    {
        AppDomain.CurrentDomain.AssemblyResolve += (sender, args) =>
        {
            // Custom assembly çözümleme
            return Assembly.LoadFrom("CustomPath\\" + args.Name + ".dll");
        };
    }
}
```

### 5. Assembly ve Namespace arasındaki ilişki nedir?
**Cevap:**
Assembly ve Namespace ilişkisi:
- Assembly fiziksel birimdir
- Namespace mantıksal birimdir
- Bir assembly birden fazla namespace içerebilir
- Bir namespace birden fazla assembly'de olabilir
- Namespace'ler assembly sınırlarını aşabilir

**Örnek Kod:**
```csharp
// Farklı assembly'lerde aynı namespace
// Assembly1.dll
namespace MyApp.Common
{
    public class Logger { }
}

// Assembly2.dll
namespace MyApp.Common
{
    public class Configuration { }
}

// Kullanım
public class Example
{
    public void UseNamespaces()
    {
        // Farklı assembly'lerden aynı namespace
        var logger = new MyApp.Common.Logger();
        var config = new MyApp.Common.Configuration();
    }
}
```

## Best Practices
1. **Assembly Tasarımı**
   - Single responsibility
   - Version yönetimi
   - Dependency minimizasyonu
   - Strong name kullanımı

2. **Namespace Organizasyonu**
   - Anlamlı isimlendirme
   - Hiyerarşik yapı
   - İsim çakışmalarından kaçınma
   - Kod organizasyonu

3. **Assembly Yönetimi**
   - Lazy loading
   - Assembly binding
   - Version kontrolü
   - Security considerations

## Kaynaklar
- [Assemblies in .NET](https://docs.microsoft.com/tr-tr/dotnet/standard/assembly/)
- [Namespaces in C#](https://docs.microsoft.com/tr-tr/dotnet/csharp/programming-guide/namespaces/)
- [Strong-Named Assemblies](https://docs.microsoft.com/tr-tr/dotnet/standard/assembly/strong-named) 