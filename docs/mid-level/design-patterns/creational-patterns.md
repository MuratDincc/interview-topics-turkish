# Creational Patterns (Oluşturucu Kalıplar)

## Genel Bakış
Creational Patterns, nesne oluşturma süreçlerini yöneten ve nesnelerin oluşturulma şeklini kontrol eden tasarım kalıplarıdır. Bu kalıplar, nesne oluşturma mantığını kapsülleyerek, sistemin nesnelerden bağımsız olmasını sağlar.

## Mülakat Soruları ve Cevapları

### 1. Singleton Pattern nedir ve ne zaman kullanılır?
**Cevap:**
Singleton Pattern, bir sınıfın yalnızca bir örneğinin oluşturulmasını ve bu örneğe global bir erişim noktası sağlanmasını garanti eder.

**Örnek Kod:**
```csharp
public class Singleton
{
    private static Singleton _instance;
    private static readonly object _lock = new object();

    private Singleton() { }

    public static Singleton Instance
    {
        get
        {
            if (_instance == null)
            {
                lock (_lock)
                {
                    if (_instance == null)
                    {
                        _instance = new Singleton();
                    }
                }
            }
            return _instance;
        }
    }
}

// Kullanım
var singleton1 = Singleton.Instance;
var singleton2 = Singleton.Instance;
// singleton1 ve singleton2 aynı örneği referans eder
```

### 2. Factory Method Pattern nedir ve ne zaman kullanılır?
**Cevap:**
Factory Method Pattern, bir nesnenin oluşturulması için bir arayüz tanımlar, ancak alt sınıfların hangi sınıfın örneğini oluşturacağına karar vermesine izin verir.

**Örnek Kod:**
```csharp
public abstract class Document
{
    public abstract void CreatePages();
}

public class Resume : Document
{
    public override void CreatePages()
    {
        Pages.Add(new SkillsPage());
        Pages.Add(new EducationPage());
        Pages.Add(new ExperiencePage());
    }
}

public class Report : Document
{
    public override void CreatePages()
    {
        Pages.Add(new IntroductionPage());
        Pages.Add(new ResultsPage());
        Pages.Add(new ConclusionPage());
    }
}

// Kullanım
Document document = new Resume();
document.CreatePages();
```

### 3. Abstract Factory Pattern nedir ve ne zaman kullanılır?
**Cevap:**
Abstract Factory Pattern, ilgili nesnelerin ailelerini oluşturmak için bir arayüz sağlar, somut sınıfları belirtmeden.

**Örnek Kod:**
```csharp
public interface IGUIFactory
{
    IButton CreateButton();
    ICheckbox CreateCheckbox();
}

public class WindowsFactory : IGUIFactory
{
    public IButton CreateButton() => new WindowsButton();
    public ICheckbox CreateCheckbox() => new WindowsCheckbox();
}

public class MacFactory : IGUIFactory
{
    public IButton CreateButton() => new MacButton();
    public ICheckbox CreateCheckbox() => new MacCheckbox();
}

// Kullanım
IGUIFactory factory = new WindowsFactory();
IButton button = factory.CreateButton();
ICheckbox checkbox = factory.CreateCheckbox();
```

### 4. Builder Pattern nedir ve ne zaman kullanılır?
**Cevap:**
Builder Pattern, karmaşık bir nesnenin adım adım oluşturulmasını sağlar. Aynı oluşturma süreci farklı temsiller oluşturabilir.

**Örnek Kod:**
```csharp
public class Pizza
{
    public string Dough { get; set; }
    public string Sauce { get; set; }
    public string Topping { get; set; }
}

public interface IPizzaBuilder
{
    void BuildDough();
    void BuildSauce();
    void BuildTopping();
    Pizza GetPizza();
}

public class HawaiianPizzaBuilder : IPizzaBuilder
{
    private Pizza _pizza = new Pizza();

    public void BuildDough() => _pizza.Dough = "Cross";
    public void BuildSauce() => _pizza.Sauce = "Mild";
    public void BuildTopping() => _pizza.Topping = "Ham+Pinapple";
    public Pizza GetPizza() => _pizza;
}

public class PizzaDirector
{
    private IPizzaBuilder _builder;

    public PizzaDirector(IPizzaBuilder builder)
    {
        _builder = builder;
    }

    public void MakePizza()
    {
        _builder.BuildDough();
        _builder.BuildSauce();
        _builder.BuildTopping();
    }
}

// Kullanım
var builder = new HawaiianPizzaBuilder();
var director = new PizzaDirector(builder);
director.MakePizza();
Pizza pizza = builder.GetPizza();
```

### 5. Prototype Pattern nedir ve ne zaman kullanılır?
**Cevap:**
Prototype Pattern, mevcut nesnelerin kopyalarını oluşturmak için kullanılır. Bu, özellikle nesne oluşturma maliyeti yüksek olduğunda faydalıdır.

**Örnek Kod:**
```csharp
public abstract class Shape : ICloneable
{
    public int X { get; set; }
    public int Y { get; set; }
    public string Color { get; set; }

    public abstract object Clone();
}

public class Circle : Shape
{
    public int Radius { get; set; }

    public override object Clone()
    {
        return new Circle
        {
            X = this.X,
            Y = this.Y,
            Color = this.Color,
            Radius = this.Radius
        };
    }
}

// Kullanım
Circle circle = new Circle
{
    X = 10,
    Y = 20,
    Radius = 15,
    Color = "Red"
};

Circle clonedCircle = (Circle)circle.Clone();
```

## Best Practices
1. **Pattern Seçimi**
   - Singleton: Tek bir örnek gerektiğinde
   - Factory Method: Alt sınıfların nesne oluşturması gerektiğinde
   - Abstract Factory: İlgili nesne aileleri oluşturulduğunda
   - Builder: Karmaşık nesneler adım adım oluşturulduğunda
   - Prototype: Mevcut nesnelerin kopyaları gerektiğinde

2. **Uygulama**
   - Dependency Injection kullanın
   - Interface'leri tercih edin
   - Unit test edilebilirliği göz önünde bulundurun
   - Documentation ekleyin

3. **Bakım**
   - Kod tekrarından kaçının
   - SOLID prensiplerini takip edin
   - Performans etkilerini değerlendirin
   - Team review yapın

## Kaynaklar
- [Microsoft Creational Patterns](https://docs.microsoft.com/tr-tr/dotnet/standard/modern-web-apps-azure-architecture/architectural-principles#creational-patterns)
- [Refactoring Guru - Creational Patterns](https://refactoring.guru/design-patterns/creational-patterns)
- [Design Patterns in C#](https://www.dofactory.com/net/design-patterns) 