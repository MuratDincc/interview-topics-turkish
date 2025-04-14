# Behavioral Patterns (Davranışsal Kalıplar)

## Genel Bakış
Behavioral Patterns, nesneler arasındaki iletişimi ve sorumluluk dağılımını düzenleyen tasarım kalıplarıdır. Bu kalıplar, nesnelerin nasıl etkileşime gireceğini ve görevlerini nasıl paylaşacağını tanımlar.

## Mülakat Soruları ve Cevapları

### 1. Observer Pattern nedir ve ne zaman kullanılır?
**Cevap:**
Observer Pattern, bir nesnenin durumu değiştiğinde, ona bağlı olan diğer nesnelerin otomatik olarak bilgilendirilmesini ve güncellenmesini sağlar.

**Örnek Kod:**
```csharp
// Subject (Gözlemlenen)
public class WeatherStation
{
    private List<IObserver> _observers = new List<IObserver>();
    private float _temperature;

    public float Temperature
    {
        get => _temperature;
        set
        {
            _temperature = value;
            Notify();
        }
    }

    public void Attach(IObserver observer)
    {
        _observers.Add(observer);
    }

    public void Detach(IObserver observer)
    {
        _observers.Remove(observer);
    }

    private void Notify()
    {
        foreach (var observer in _observers)
        {
            observer.Update(this);
        }
    }
}

// Observer (Gözlemci)
public interface IObserver
{
    void Update(WeatherStation subject);
}

public class TemperatureDisplay : IObserver
{
    public void Update(WeatherStation subject)
    {
        Console.WriteLine($"Temperature Display: {subject.Temperature}°C");
    }
}

// Kullanım
var weatherStation = new WeatherStation();
var display = new TemperatureDisplay();
weatherStation.Attach(display);
weatherStation.Temperature = 25.5f;
```

### 2. Strategy Pattern nedir ve ne zaman kullanılır?
**Cevap:**
Strategy Pattern, bir algoritmanın farklı varyasyonlarını tanımlar ve bu algoritmaları kullanan nesnelerin, çalışma zamanında algoritmayı değiştirebilmesini sağlar.

**Örnek Kod:**
```csharp
// Strategy interface
public interface ISortStrategy
{
    void Sort(List<int> list);
}

// Concrete strategies
public class BubbleSort : ISortStrategy
{
    public void Sort(List<int> list)
    {
        Console.WriteLine("Sorting using Bubble Sort");
        // Bubble sort implementation
    }
}

public class QuickSort : ISortStrategy
{
    public void Sort(List<int> list)
    {
        Console.WriteLine("Sorting using Quick Sort");
        // Quick sort implementation
    }
}

// Context
public class Sorter
{
    private ISortStrategy _strategy;

    public Sorter(ISortStrategy strategy)
    {
        _strategy = strategy;
    }

    public void SetStrategy(ISortStrategy strategy)
    {
        _strategy = strategy;
    }

    public void Sort(List<int> list)
    {
        _strategy.Sort(list);
    }
}

// Kullanım
var numbers = new List<int> { 5, 2, 8, 1, 9 };
var sorter = new Sorter(new BubbleSort());
sorter.Sort(numbers);

sorter.SetStrategy(new QuickSort());
sorter.Sort(numbers);
```

### 3. Command Pattern nedir ve ne zaman kullanılır?
**Cevap:**
Command Pattern, bir isteği bir nesne olarak kapsüller, böylece farklı istekleri parametreleştirebilir, sıraya alabilir veya geri alabilirsiniz.

**Örnek Kod:**
```csharp
// Command interface
public interface ICommand
{
    void Execute();
    void Undo();
}

// Concrete command
public class LightOnCommand : ICommand
{
    private Light _light;
    private bool _previousState;

    public LightOnCommand(Light light)
    {
        _light = light;
    }

    public void Execute()
    {
        _previousState = _light.IsOn;
        _light.TurnOn();
    }

    public void Undo()
    {
        if (!_previousState)
            _light.TurnOff();
    }
}

// Receiver
public class Light
{
    public bool IsOn { get; private set; }

    public void TurnOn()
    {
        IsOn = true;
        Console.WriteLine("Light is on");
    }

    public void TurnOff()
    {
        IsOn = false;
        Console.WriteLine("Light is off");
    }
}

// Invoker
public class RemoteControl
{
    private ICommand _command;

    public void SetCommand(ICommand command)
    {
        _command = command;
    }

    public void PressButton()
    {
        _command.Execute();
    }

    public void PressUndo()
    {
        _command.Undo();
    }
}

// Kullanım
var light = new Light();
var lightOn = new LightOnCommand(light);
var remote = new RemoteControl();

remote.SetCommand(lightOn);
remote.PressButton();
remote.PressUndo();
```

### 4. State Pattern nedir ve ne zaman kullanılır?
**Cevap:**
State Pattern, bir nesnenin iç durumu değiştiğinde davranışını değiştirmesini sağlar. Nesne, sınıfını değiştirmeden farklı durumlarda farklı davranabilir.

**Örnek Kod:**
```csharp
// State interface
public interface IState
{
    void Handle(Context context);
}

// Concrete states
public class ConcreteStateA : IState
{
    public void Handle(Context context)
    {
        Console.WriteLine("Handling in State A");
        context.State = new ConcreteStateB();
    }
}

public class ConcreteStateB : IState
{
    public void Handle(Context context)
    {
        Console.WriteLine("Handling in State B");
        context.State = new ConcreteStateA();
    }
}

// Context
public class Context
{
    private IState _state;

    public Context(IState state)
    {
        _state = state;
    }

    public IState State
    {
        get => _state;
        set
        {
            _state = value;
            Console.WriteLine($"State changed to {_state.GetType().Name}");
        }
    }

    public void Request()
    {
        _state.Handle(this);
    }
}

// Kullanım
var context = new Context(new ConcreteStateA());
context.Request();
context.Request();
```

### 5. Chain of Responsibility Pattern nedir ve ne zaman kullanılır?
**Cevap:**
Chain of Responsibility Pattern, bir isteği işleyebilecek nesnelerin zincirini oluşturur. İstek, zincir boyunca ilerler ve uygun işleyici bulunana kadar devam eder.

**Örnek Kod:**
```csharp
// Handler interface
public interface IHandler
{
    IHandler SetNext(IHandler handler);
    object Handle(object request);
}

// Abstract handler
public abstract class AbstractHandler : IHandler
{
    private IHandler _nextHandler;

    public IHandler SetNext(IHandler handler)
    {
        _nextHandler = handler;
        return handler;
    }

    public virtual object Handle(object request)
    {
        if (_nextHandler != null)
        {
            return _nextHandler.Handle(request);
        }
        return null;
    }
}

// Concrete handlers
public class MonkeyHandler : AbstractHandler
{
    public override object Handle(object request)
    {
        if (request.ToString() == "Banana")
        {
            return $"Monkey: I'll eat the {request}.";
        }
        return base.Handle(request);
    }
}

public class SquirrelHandler : AbstractHandler
{
    public override object Handle(object request)
    {
        if (request.ToString() == "Nut")
        {
            return $"Squirrel: I'll eat the {request}.";
        }
        return base.Handle(request);
    }
}

// Kullanım
var monkey = new MonkeyHandler();
var squirrel = new SquirrelHandler();

monkey.SetNext(squirrel);

Console.WriteLine(monkey.Handle("Banana"));
Console.WriteLine(monkey.Handle("Nut"));
```

## Best Practices
1. **Pattern Seçimi**
   - Observer: Durum değişikliklerini takip etmek için
   - Strategy: Algoritma varyasyonları için
   - Command: İstekleri nesneleştirmek için
   - State: Nesne durumlarını yönetmek için
   - Chain of Responsibility: İstek işleme zinciri için

2. **Uygulama**
   - Interface'leri doğru tanımlayın
   - SOLID prensiplerini takip edin
   - Dependency Injection kullanın
   - Documentation ekleyin

3. **Bakım**
   - Kod tekrarından kaçının
   - Test edilebilirliği göz önünde bulundurun
   - Performans etkilerini değerlendirin
   - Team review yapın

## Kaynaklar
- [Microsoft Behavioral Patterns](https://docs.microsoft.com/tr-tr/dotnet/standard/modern-web-apps-azure-architecture/architectural-principles#behavioral-patterns)
- [Refactoring Guru - Behavioral Patterns](https://refactoring.guru/design-patterns/behavioral-patterns)
- [Design Patterns in C#](https://www.dofactory.com/net/design-patterns) 