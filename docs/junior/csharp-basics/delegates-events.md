# Delegates ve Events

## Genel Bakış
Bu bölümde, C#'ta fonksiyon işaretçileri olan delegate'leri ve event mekanizmasını inceleyeceğiz.

## Mülakat Soruları ve Cevapları

### 1. Delegate nedir ve nasıl kullanılır?
**Cevap:**
Delegate:
- Fonksiyon işaretçisi
- Tip güvenli
- Multicast desteği
- Lambda expression desteği

**Örnek Kod:**
```csharp
// Delegate tanımlama
public delegate void LogDelegate(string message);

// Delegate kullanımı
public class Logger
{
    public void LogToConsole(string message)
    {
        Console.WriteLine(message);
    }
    
    public void LogToFile(string message)
    {
        File.WriteAllText("log.txt", message);
    }
}

// Kullanım
var logger = new Logger();
LogDelegate logDelegate = logger.LogToConsole;
logDelegate += logger.LogToFile;  // Multicast
logDelegate("Test mesajı");

// Lambda expression
LogDelegate lambdaDelegate = msg => Console.WriteLine($"Lambda: {msg}");
lambdaDelegate("Test");
```

### 2. Event nedir ve nasıl kullanılır?
**Cevap:**
Event:
- Delegate wrapper
- Encapsulation sağlar
- Publisher-Subscriber pattern
- Thread-safe

**Örnek Kod:**
```csharp
// Event tanımlama
public class Button
{
    public event EventHandler Clicked;
    
    public void Click()
    {
        Clicked?.Invoke(this, EventArgs.Empty);
    }
}

// Event kullanımı
public class Form
{
    private Button _button;
    
    public Form()
    {
        _button = new Button();
        _button.Clicked += OnButtonClicked;
    }
    
    private void OnButtonClicked(object sender, EventArgs e)
    {
        Console.WriteLine("Buton tıklandı!");
    }
}
```

### 3. Action ve Func delegate'leri nedir?
**Cevap:**
Generic Delegate'ler:
- **Action:**
  - Void dönüş tipi
  - 0-16 parametre
  - Genel amaçlı

- **Func:**
  - Generic dönüş tipi
  - 0-16 parametre
  - Son parametre dönüş tipi

**Örnek Kod:**
```csharp
// Action kullanımı
Action<string> logAction = message => Console.WriteLine(message);
Action<int, int> sumAction = (a, b) => Console.WriteLine(a + b);

// Func kullanımı
Func<int, int, int> sumFunc = (a, b) => a + b;
Func<string, bool> isValidFunc = s => !string.IsNullOrEmpty(s);

// Kullanım
logAction("Test");
int result = sumFunc(5, 3);
bool valid = isValidFunc("test");
```

### 4. Event ve Delegate arasındaki farklar nelerdir?
**Cevap:**
Event vs Delegate:
- **Event:**
  - Sadece sınıf içinden tetiklenebilir
  - += ve -= operatörleri
  - Thread-safe
  - Encapsulation

- **Delegate:**
  - Her yerden tetiklenebilir
  - = operatörü
  - Thread-safe değil
  - Direkt erişim

**Örnek Kod:**
```csharp
public class Publisher
{
    // Delegate
    public LogDelegate LogDelegate { get; set; }
    
    // Event
    public event EventHandler SomethingHappened;
    
    public void DoSomething()
    {
        // Delegate kullanımı
        LogDelegate?.Invoke("Log mesajı");
        
        // Event kullanımı
        SomethingHappened?.Invoke(this, EventArgs.Empty);
    }
}

public class Subscriber
{
    public void Subscribe(Publisher publisher)
    {
        // Delegate atama
        publisher.LogDelegate = LogMessage;
        
        // Event subscription
        publisher.SomethingHappened += OnSomethingHappened;
    }
    
    private void LogMessage(string message)
    {
        Console.WriteLine(message);
    }
    
    private void OnSomethingHappened(object sender, EventArgs e)
    {
        Console.WriteLine("Bir şey oldu!");
    }
}
```

### 5. Lambda expression nedir ve nasıl kullanılır?
**Cevap:**
Lambda Expression:
- Anonim fonksiyon
- Delegate kısayolu
- Expression/Statement body
- Closure desteği

**Örnek Kod:**
```csharp
// Expression lambda
Func<int, int> square = x => x * x;

// Statement lambda
Action<string> print = message =>
{
    Console.WriteLine("Başlık");
    Console.WriteLine(message);
};

// Closure örneği
int factor = 2;
Func<int, int> multiplier = n => n * factor;
Console.WriteLine(multiplier(5));  // 10

// LINQ ile kullanım
var numbers = new List<int> { 1, 2, 3, 4, 5 };
var evenNumbers = numbers.Where(n => n % 2 == 0);
```

## Best Practices
1. **Event Kullanımı**
   - EventHandler<T> kullanımı
   - Null kontrolü
   - Thread safety
   - Event naming

2. **Delegate Kullanımı**
   - Generic delegate'ler
   - Lambda expression
   - Multicast dikkatli kullanım
   - Memory leak önleme

3. **Performans**
   - Delegate caching
   - Event invocation optimizasyonu
   - Lambda expression reuse
   - Closure dikkatli kullanımı

## Kaynaklar
- [C# Delegates](https://docs.microsoft.com/tr-tr/dotnet/csharp/programming-guide/delegates/)
- [C# Events](https://docs.microsoft.com/tr-tr/dotnet/csharp/programming-guide/events/)
- [Lambda Expressions](https://docs.microsoft.com/tr-tr/dotnet/csharp/language-reference/operators/lambda-expressions) 