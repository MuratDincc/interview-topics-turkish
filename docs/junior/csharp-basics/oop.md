# Nesne Yönelimli Programlama (OOP)

## Genel Bakış
Bu bölümde, C#'ta Nesne Yönelimli Programlamanın temel prensiplerini ve uygulamalarını inceleyeceğiz.

## Mülakat Soruları ve Cevapları

### 1. OOP'in temel prensipleri nelerdir?
**Cevap:**
OOP Prensipleri:
- **Kapsülleme (Encapsulation):**
  - Veri gizleme
  - Access modifiers
  - Properties kullanımı

- **Kalıtım (Inheritance):**
  - Base class - Derived class
  - Code reuse
  - Polymorphism temeli

- **Polimorfizm (Polymorphism):**
  - Method overriding
  - Interface implementation
  - Virtual/override

- **Soyutlama (Abstraction):**
  - Abstract classes
  - Interfaces
  - Method signatures

**Örnek Kod:**
```csharp
// Kapsülleme
public class Person
{
    private string _name;
    public string Name
    {
        get { return _name; }
        set { _name = value; }
    }
}

// Kalıtım
public class Employee : Person
{
    public decimal Salary { get; set; }
}

// Polimorfizm
public abstract class Animal
{
    public virtual void MakeSound()
    {
        Console.WriteLine("Ses çıkarıyor");
    }
}

public class Dog : Animal
{
    public override void MakeSound()
    {
        Console.WriteLine("Hav hav!");
    }
}

// Soyutlama
public interface IShape
{
    double CalculateArea();
    double CalculatePerimeter();
}
```

### 2. Interface ve Abstract Class arasındaki farklar nelerdir?
**Cevap:**
Interface vs Abstract Class:
- **Interface:**
  - Sadece method signatures
  - Çoklu kalıtım
  - Implementation zorunlu
  - Varsayılan implementasyon (C# 8+)

- **Abstract Class:**
  - Tam implementasyon içerebilir
  - Tekli kalıtım
  - Abstract methods
  - Constructor olabilir

**Örnek Kod:**
```csharp
// Interface
public interface ILogger
{
    void Log(string message);
    void LogError(string message) => Log($"ERROR: {message}"); // Default implementation
}

// Abstract Class
public abstract class LoggerBase
{
    protected string FormatMessage(string message)
    {
        return $"[{DateTime.Now}] {message}";
    }
    
    public abstract void Log(string message);
}

// Implementation
public class FileLogger : LoggerBase, ILogger
{
    public override void Log(string message)
    {
        File.WriteAllText("log.txt", FormatMessage(message));
    }
}
```

### 3. Constructor ve Destructor nedir?
**Cevap:**
Constructor/Destructor:
- **Constructor:**
  - Nesne oluşturulduğunda çalışır
  - Overloading yapılabilir
  - this/base kullanımı
  - Static constructor

- **Destructor:**
  - Nesne yok edildiğinde çalışır
  - IDisposable pattern
  - Garbage Collection

**Örnek Kod:**
```csharp
public class Person
{
    private string _name;
    private int _age;
    
    // Default constructor
    public Person()
    {
        _name = "Bilinmiyor";
        _age = 0;
    }
    
    // Parameterized constructor
    public Person(string name, int age)
    {
        _name = name;
        _age = age;
    }
    
    // Constructor chaining
    public Person(string name) : this(name, 0)
    {
    }
    
    // Static constructor
    static Person()
    {
        Console.WriteLine("Static constructor çalıştı");
    }
    
    // Destructor
    ~Person()
    {
        Console.WriteLine("Destructor çalıştı");
    }
}
```

### 4. Method Overloading ve Overriding nedir?
**Cevap:**
Method Overloading/Overriding:
- **Overloading:**
  - Aynı isim, farklı parametreler
  - Compile-time polymorphism
  - Return type farklı olabilir

- **Overriding:**
  - Base class methodunu değiştirme
  - Runtime polymorphism
  - virtual/override kullanımı

**Örnek Kod:**
```csharp
public class Calculator
{
    // Method overloading
    public int Add(int a, int b)
    {
        return a + b;
    }
    
    public double Add(double a, double b)
    {
        return a + b;
    }
    
    public int Add(int a, int b, int c)
    {
        return a + b + c;
    }
}

public class ScientificCalculator : Calculator
{
    // Method overriding
    public override int Add(int a, int b)
    {
        Console.WriteLine("Scientific calculator kullanılıyor");
        return base.Add(a, b);
    }
}
```

### 5. SOLID prensipleri nelerdir?
**Cevap:**
SOLID Prensipleri:
- **Single Responsibility:**
  - Tek sorumluluk
  - Sınıf değişimi tek nedene bağlı

- **Open/Closed:**
  - Genişlemeye açık
  - Değişime kapalı

- **Liskov Substitution:**
  - Alt sınıflar üst sınıfın yerine geçebilmeli

- **Interface Segregation:**
  - Küçük, özel interface'ler

- **Dependency Inversion:**
  - Yüksek seviye modüller düşük seviyeye bağımlı olmamalı

**Örnek Kod:**
```csharp
// Single Responsibility
public class User
{
    public string Name { get; set; }
    public string Email { get; set; }
}

public class UserValidator
{
    public bool Validate(User user)
    {
        // Validation logic
    }
}

// Open/Closed
public abstract class Shape
{
    public abstract double CalculateArea();
}

public class Circle : Shape
{
    public double Radius { get; set; }
    
    public override double CalculateArea()
    {
        return Math.PI * Radius * Radius;
    }
}

// Interface Segregation
public interface IPrinter
{
    void Print();
}

public interface IScanner
{
    void Scan();
}

public class MultiFunctionPrinter : IPrinter, IScanner
{
    public void Print() { }
    public void Scan() { }
}
```

## Best Practices
1. **Sınıf Tasarımı**
   - Tek sorumluluk prensibi
   - Yüksek bağlantılılık
   - Düşük bağımlılık

2. **Kalıtım Kullanımı**
   - Composition over inheritance
   - Interface kullanımı
   - Abstract class dikkatli kullanımı

3. **Kod Organizasyonu**
   - Namespace kullanımı
   - Access modifier'lar
   - Property kullanımı

## Kaynaklar
- [C# OOP Concepts](https://docs.microsoft.com/tr-tr/dotnet/csharp/fundamentals/object-oriented/)
- [SOLID Principles](https://docs.microsoft.com/tr-tr/dotnet/architecture/modern-web-apps-azure/architectural-principles)
- [C# Programming Guide](https://docs.microsoft.com/tr-tr/dotnet/csharp/programming-guide/) 