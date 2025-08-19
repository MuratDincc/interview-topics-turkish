# Visual Studio

## Giriş

Visual Studio, Microsoft tarafından geliştirilen kapsamlı bir entegre geliştirme ortamıdır (IDE). .NET geliştiricileri için en yaygın kullanılan geliştirme aracıdır. Bu dosya, Visual Studio'nun temel özelliklerini, kısayollarını ve en iyi uygulamalarını kapsar.

## Temel Kavramlar

### 1. Visual Studio Nedir?
Visual Studio, yazılım geliştirme sürecinin her aşamasını destekleyen kapsamlı bir IDE'dir.

**Özellikler:**
- IntelliSense kod tamamlama
- Gelişmiş debugging
- Git entegrasyonu
- Paket yönetimi (NuGet)
- Proje şablonları

### 2. Solution ve Project Yapısı
```
Solution (.sln)
├── Project1 (.csproj)
│   ├── Controllers/
│   ├── Models/
│   └── Views/
└── Project2 (.csproj)
    ├── Services/
    └── Repositories/
```

### 3. Önemli Menüler ve Pencereler
- **Solution Explorer**: Proje dosyalarını görüntüleme
- **Error List**: Hataları ve uyarıları görüntüleme
- **Output Window**: Build ve debug çıktıları
- **Package Manager Console**: NuGet komutları
- **Server Explorer**: Veritabanı bağlantıları

## Temel İşlemler

### 1. Proje Oluşturma
```csharp
// File -> New -> Project
// Template seçimi:
// - Console Application
// - ASP.NET Core Web Application
// - Class Library
// - Unit Test Project
```

### 2. NuGet Paket Yönetimi
```xml
<!-- Package Manager Console -->
Install-Package EntityFramework
Update-Package EntityFramework
Uninstall-Package EntityFramework

<!-- .csproj dosyasında -->
<PackageReference Include="Microsoft.EntityFrameworkCore" Version="6.0.0" />
```

### 3. Debug İşlemleri
```csharp
public class DebugExample
{
    public void CalculateSum()
    {
        int a = 10;
        int b = 20;
        int sum = a + b; // Breakpoint koyun
        
        Console.WriteLine($"Sum: {sum}");
    }
}
```

## Yararlı Kısayollar

### 1. Kod Düzenleme
```
Ctrl + K, Ctrl + D    : Kodu biçimlendirme
Ctrl + K, Ctrl + C    : Yorum satırı yapma
Ctrl + K, Ctrl + U    : Yorum satırını kaldırma
Ctrl + Space          : IntelliSense tetikleme
Ctrl + .              : Quick Actions ve Refactoring
```

### 2. Navigasyon
```
Ctrl + T              : Go to All (dosya, sınıf, method arama)
F12                   : Go to Definition
Ctrl + F12            : Go to Implementation
Ctrl + -              : Navigate Backward
Ctrl + Shift + -      : Navigate Forward
```

### 3. Debug Kısayolları
```
F5                    : Start Debugging
Ctrl + F5             : Start Without Debugging
F9                    : Toggle Breakpoint
F10                   : Step Over
F11                   : Step Into
Shift + F11           : Step Out
```

### 4. Build ve Run
```
Ctrl + Shift + B      : Build Solution
F6                    : Build Project
Ctrl + F5             : Run Without Debugging
F5                    : Run With Debugging
```

## IntelliSense ve Code Completion

### 1. IntelliSense Özellikleri
```csharp
public class IntelliSenseExample
{
    public void Example()
    {
        var list = new List<string>();
        list. // IntelliSense burada devreye girer
        
        // Parametre ipuçları
        string.Format() // Method imzasını gösterir
    }
}
```

### 2. Code Snippets
```csharp
// "prop" yazıp Tab tuşuna basın
public int MyProperty { get; set; }

// "ctor" yazıp Tab tuşuna basın
public MyClass()
{
    
}

// "for" yazıp Tab tuşuna basın
for (int i = 0; i < length; i++)
{
    
}
```

## Code Analysis ve Refactoring

### 1. Code Analysis
```csharp
public class CodeAnalysisExample
{
    // Unused variable warning
    public void UnusedVariable()
    {
        int unusedVariable = 10; // CS0219 warning
    }
    
    // Null reference warning
    public void NullReference(string text)
    {
        var length = text.Length; // Possible null reference
    }
}
```

### 2. Refactoring İşlemleri
```csharp
// Extract Method
public void LongMethod()
{
    // Kodu seçin ve Ctrl + R, Ctrl + M
    int a = 10;
    int b = 20;
    int sum = a + b;
    Console.WriteLine($"Sum: {sum}");
}

// Rename
public void OldMethodName() // F2 ile yeniden adlandırın
{
    
}
```

## Extensions ve Customization

### 1. Önerilen Extensions
- **ReSharper**: Gelişmiş kod analizi
- **CodeMaid**: Kod temizleme
- **Git Extensions**: Git işlemleri
- **Postman**: API testi
- **SQLite Toolbox**: SQLite veritabanı yönetimi

### 2. Theme ve Görünüm
```
Tools -> Options -> Environment -> General
- Color theme: Dark, Light, Blue
- Font and Colors ayarları
- Editor ayarları
```

## Mülakat Soruları

### Temel Sorular

1. **Visual Studio nedir ve neden kullanılır?**
   - **Cevap**: Microsoft'un IDE'si, .NET geliştirme için kapsamlı araçlar sağlar.

2. **Solution ve Project arasındaki fark nedir?**
   - **Cevap**: Solution birden fazla project'i gruplar, Project tek bir uygulama veya kütüphanedir.

3. **IntelliSense nedir?**
   - **Cevap**: Kod yazarken otomatik tamamlama ve öneri sistemi.

4. **Breakpoint nedir ve nasıl kullanılır?**
   - **Cevap**: Debug sırasında kodun durmasını sağlayan nokta, F9 ile eklenir.

5. **NuGet nedir?**
   - **Cevap**: .NET için paket yönetim sistemi, kütüphaneleri yönetir.

### Teknik Sorular

1. **Debug ile Release build arasındaki fark nedir?**
   - **Cevap**: Debug geliştirme için optimize edilmiş, Release üretim için optimize edilmiş.

2. **Code-first approach nedir?**
   - **Cevap**: Veritabanını kod üzerinden model sınıfları ile oluşturma yaklaşımı.

3. **Refactoring nedir ve örnekleri?**
   - **Cevap**: Kodu yeniden yapılandırma, Extract Method, Rename, Move Class gibi.

## Best Practices

### 1. **Proje Organizasyonu**
- Solution'da mantıklı proje grupları oluşturun
- Naming conventions kullanın
- Folder structure'ı düzenli tutun

### 2. **Kod Yazma**
- IntelliSense'i etkin kullanın
- Code snippets kullanın
- Regular refactoring yapın

### 3. **Debug İşlemleri**
- Meaningful breakpoint'ler koyun
- Watch window'u kullanın
- Exception handling implement edin

### 4. **Performance**
- Large solution'larda lightweight load kullanın
- Unnecessary extensions'ları kaldırın
- Regular cleanup yapın

## Kaynaklar

- [Visual Studio Documentation](https://docs.microsoft.com/en-us/visualstudio/)
- [Visual Studio Keyboard Shortcuts](https://docs.microsoft.com/en-us/visualstudio/ide/default-keyboard-shortcuts-in-visual-studio)
- [Visual Studio Tips and Tricks](https://docs.microsoft.com/en-us/visualstudio/ide/productivity-features)
- [Visual Studio Extensions](https://marketplace.visualstudio.com/)
- [IntelliSense Guide](https://docs.microsoft.com/en-us/visualstudio/ide/using-intellisense)
