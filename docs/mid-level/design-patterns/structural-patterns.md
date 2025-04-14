# Structural Patterns (Yapısal Kalıplar)

## Genel Bakış
Structural Patterns, nesnelerin ve sınıfların daha büyük yapılar oluşturmak için nasıl bir araya getirileceğini tanımlayan tasarım kalıplarıdır. Bu kalıplar, nesneler arasındaki ilişkileri düzenleyerek, sistemin daha esnek ve verimli olmasını sağlar.

## Mülakat Soruları ve Cevapları

### 1. Adapter Pattern nedir ve ne zaman kullanılır?
**Cevap:**
Adapter Pattern, uyumsuz arayüzlere sahip sınıfların birlikte çalışmasını sağlar. Mevcut bir sınıfın arayüzünü, istemcinin beklediği başka bir arayüze dönüştürür.

**Örnek Kod:**
```csharp
// Mevcut sınıf
public class LegacyPrinter
{
    public void PrintDocument(string document)
    {
        Console.WriteLine($"Legacy Printer: {document}");
    }
}

// Hedef arayüz
public interface IPrinter
{
    void Print(string content);
}

// Adapter
public class PrinterAdapter : IPrinter
{
    private readonly LegacyPrinter _legacyPrinter;

    public PrinterAdapter(LegacyPrinter legacyPrinter)
    {
        _legacyPrinter = legacyPrinter;
    }

    public void Print(string content)
    {
        _legacyPrinter.PrintDocument(content);
    }
}

// Kullanım
var legacyPrinter = new LegacyPrinter();
var adapter = new PrinterAdapter(legacyPrinter);
adapter.Print("Hello World");
```

### 2. Bridge Pattern nedir ve ne zaman kullanılır?
**Cevap:**
Bridge Pattern, bir soyutlamayı onun uygulamasından ayırarak, ikisinin de bağımsız olarak değişebilmesini sağlar.

**Örnek Kod:**
```csharp
// Implementasyon arayüzü
public interface IRenderer
{
    void RenderCircle(float radius);
    void RenderSquare(float side);
}

// Somut implementasyonlar
public class VectorRenderer : IRenderer
{
    public void RenderCircle(float radius)
    {
        Console.WriteLine($"Drawing a circle of radius {radius} using vector graphics");
    }

    public void RenderSquare(float side)
    {
        Console.WriteLine($"Drawing a square of side {side} using vector graphics");
    }
}

public class RasterRenderer : IRenderer
{
    public void RenderCircle(float radius)
    {
        Console.WriteLine($"Drawing a circle of radius {radius} using pixels");
    }

    public void RenderSquare(float side)
    {
        Console.WriteLine($"Drawing a square of side {side} using pixels");
    }
}

// Soyutlama
public abstract class Shape
{
    protected IRenderer renderer;

    protected Shape(IRenderer renderer)
    {
        this.renderer = renderer;
    }

    public abstract void Draw();
}

// Somut şekiller
public class Circle : Shape
{
    private float radius;

    public Circle(IRenderer renderer, float radius) : base(renderer)
    {
        this.radius = radius;
    }

    public override void Draw()
    {
        renderer.RenderCircle(radius);
    }
}

// Kullanım
var vectorRenderer = new VectorRenderer();
var circle = new Circle(vectorRenderer, 5);
circle.Draw();
```

### 3. Composite Pattern nedir ve ne zaman kullanılır?
**Cevap:**
Composite Pattern, nesneleri ağaç yapıları halinde düzenleyerek, tek tek nesneler ve nesne kompozisyonları arasında tutarlı bir şekilde çalışmayı sağlar.

**Örnek Kod:**
```csharp
public abstract class Component
{
    protected string name;

    public Component(string name)
    {
        this.name = name;
    }

    public abstract void Add(Component component);
    public abstract void Remove(Component component);
    public abstract void Display(int depth);
}

public class Leaf : Component
{
    public Leaf(string name) : base(name) { }

    public override void Add(Component component)
    {
        throw new NotImplementedException();
    }

    public override void Remove(Component component)
    {
        throw new NotImplementedException();
    }

    public override void Display(int depth)
    {
        Console.WriteLine(new string('-', depth) + name);
    }
}

public class Composite : Component
{
    private List<Component> children = new List<Component>();

    public Composite(string name) : base(name) { }

    public override void Add(Component component)
    {
        children.Add(component);
    }

    public override void Remove(Component component)
    {
        children.Remove(component);
    }

    public override void Display(int depth)
    {
        Console.WriteLine(new string('-', depth) + name);
        foreach (var component in children)
        {
            component.Display(depth + 2);
        }
    }
}

// Kullanım
var root = new Composite("root");
root.Add(new Leaf("Leaf A"));
root.Add(new Leaf("Leaf B"));

var comp = new Composite("Composite X");
comp.Add(new Leaf("Leaf XA"));
comp.Add(new Leaf("Leaf XB"));

root.Add(comp);
root.Display(1);
```

### 4. Decorator Pattern nedir ve ne zaman kullanılır?
**Cevap:**
Decorator Pattern, bir nesneye dinamik olarak yeni sorumluluklar eklemek için kullanılır. Alt sınıflama yerine nesneyi sarmalayarak işlevselliği genişletir.

**Örnek Kod:**
```csharp
public abstract class Beverage
{
    public string Description { get; set; }
    public abstract double Cost();
}

public class Espresso : Beverage
{
    public Espresso()
    {
        Description = "Espresso";
    }

    public override double Cost()
    {
        return 1.99;
    }
}

public abstract class CondimentDecorator : Beverage
{
    protected Beverage beverage;
}

public class Mocha : CondimentDecorator
{
    public Mocha(Beverage beverage)
    {
        this.beverage = beverage;
    }

    public override string Description
    {
        get { return beverage.Description + ", Mocha"; }
        set { }
    }

    public override double Cost()
    {
        return 0.20 + beverage.Cost();
    }
}

// Kullanım
Beverage beverage = new Espresso();
beverage = new Mocha(beverage);
Console.WriteLine($"{beverage.Description} ${beverage.Cost()}");
```

### 5. Facade Pattern nedir ve ne zaman kullanılır?
**Cevap:**
Facade Pattern, karmaşık bir alt sistemin kullanımını basitleştirmek için üst düzey bir arayüz sağlar.

**Örnek Kod:**
```csharp
// Alt sistem sınıfları
public class SubsystemA
{
    public void OperationA()
    {
        Console.WriteLine("SubsystemA: OperationA");
    }
}

public class SubsystemB
{
    public void OperationB()
    {
        Console.WriteLine("SubsystemB: OperationB");
    }
}

public class SubsystemC
{
    public void OperationC()
    {
        Console.WriteLine("SubsystemC: OperationC");
    }
}

// Facade
public class Facade
{
    private SubsystemA _subsystemA;
    private SubsystemB _subsystemB;
    private SubsystemC _subsystemC;

    public Facade()
    {
        _subsystemA = new SubsystemA();
        _subsystemB = new SubsystemB();
        _subsystemC = new SubsystemC();
    }

    public void Operation1()
    {
        _subsystemA.OperationA();
        _subsystemB.OperationB();
    }

    public void Operation2()
    {
        _subsystemB.OperationB();
        _subsystemC.OperationC();
    }
}

// Kullanım
var facade = new Facade();
facade.Operation1();
facade.Operation2();
```

## Best Practices
1. **Pattern Seçimi**
   - Adapter: Uyumsuz arayüzleri uyumlu hale getirmek için
   - Bridge: Soyutlama ve implementasyonu ayırmak için
   - Composite: Ağaç yapıları oluşturmak için
   - Decorator: Dinamik olarak sorumluluk eklemek için
   - Facade: Karmaşık alt sistemleri basitleştirmek için

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
- [Microsoft Structural Patterns](https://docs.microsoft.com/tr-tr/dotnet/standard/modern-web-apps-azure-architecture/architectural-principles#structural-patterns)
- [Refactoring Guru - Structural Patterns](https://refactoring.guru/design-patterns/structural-patterns)
- [Design Patterns in C#](https://www.dofactory.com/net/design-patterns) 