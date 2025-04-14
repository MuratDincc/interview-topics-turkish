# Kontrol Yapıları

## Genel Bakış
Bu bölümde, C#'ta program akışını kontrol etmek için kullanılan temel yapıları inceleyeceğiz.

## Mülakat Soruları ve Cevapları

### 1. If-Else yapısı nedir ve nasıl kullanılır?
**Cevap:**
If-Else Yapısı:
- Koşullu program akışı sağlar
- Boolean ifadelerle çalışır
- İç içe kullanılabilir
- Ternary operatör alternatifi

**Örnek Kod:**
```csharp
// Temel if-else
int number = 10;
if (number > 0)
{
    Console.WriteLine("Pozitif");
}
else if (number < 0)
{
    Console.WriteLine("Negatif");
}
else
{
    Console.WriteLine("Sıfır");
}

// Ternary operatör
string result = number > 0 ? "Pozitif" : "Negatif veya Sıfır";

// İç içe if-else
if (number > 0)
{
    if (number % 2 == 0)
    {
        Console.WriteLine("Pozitif çift sayı");
    }
    else
    {
        Console.WriteLine("Pozitif tek sayı");
    }
}
```

### 2. Switch-Case yapısı nedir ve ne zaman kullanılır?
**Cevap:**
Switch-Case:
- Çoklu koşul kontrolü
- Pattern matching desteği
- Case fallthrough özelliği
- Expression-based switch

**Örnek Kod:**
```csharp
// Temel switch-case
int day = 3;
switch (day)
{
    case 1:
        Console.WriteLine("Pazartesi");
        break;
    case 2:
        Console.WriteLine("Salı");
        break;
    case 3:
        Console.WriteLine("Çarşamba");
        break;
    default:
        Console.WriteLine("Geçersiz gün");
        break;
}

// Pattern matching
object obj = "Merhaba";
switch (obj)
{
    case string s:
        Console.WriteLine($"String: {s}");
        break;
    case int i:
        Console.WriteLine($"Integer: {i}");
        break;
    case null:
        Console.WriteLine("Null");
        break;
}

// Expression-based switch
string result = day switch
{
    1 => "Pazartesi",
    2 => "Salı",
    3 => "Çarşamba",
    _ => "Geçersiz gün"
};
```

### 3. Döngüler nelerdir ve nasıl kullanılır?
**Cevap:**
Döngü Türleri:
- **for:** Belirli sayıda tekrar
- **while:** Koşul sağlandıkça
- **do-while:** En az bir kez çalışır
- **foreach:** Koleksiyonlar üzerinde

**Örnek Kod:**
```csharp
// for döngüsü
for (int i = 0; i < 10; i++)
{
    Console.WriteLine(i);
}

// while döngüsü
int j = 0;
while (j < 10)
{
    Console.WriteLine(j);
    j++;
}

// do-while döngüsü
int k = 0;
do
{
    Console.WriteLine(k);
    k++;
} while (k < 10);

// foreach döngüsü
var numbers = new[] { 1, 2, 3, 4, 5 };
foreach (var number in numbers)
{
    Console.WriteLine(number);
}
```

### 4. Break ve Continue ifadeleri ne işe yarar?
**Cevap:**
Break/Continue:
- **Break:**
  - Döngüyü sonlandırır
  - Switch-case'den çıkar
  - İç içe döngülerde etiket kullanımı

- **Continue:**
  - Mevcut iterasyonu atlar
  - Sonraki iterasyona geçer
  - Koşullu atlama sağlar

**Örnek Kod:**
```csharp
// Break kullanımı
for (int i = 0; i < 10; i++)
{
    if (i == 5)
        break;
    Console.WriteLine(i);
}

// Continue kullanımı
for (int i = 0; i < 10; i++)
{
    if (i % 2 == 0)
        continue;
    Console.WriteLine(i);
}

// Etiketli break
outer:
for (int i = 0; i < 3; i++)
{
    for (int j = 0; j < 3; j++)
    {
        if (i == 1 && j == 1)
            break outer;
        Console.WriteLine($"{i}, {j}");
    }
}
```

### 5. Exception handling nasıl yapılır?
**Cevap:**
Exception Handling:
- try-catch-finally blokları
- Özel exception tipleri
- Exception filtreleme
- throw ifadesi

**Örnek Kod:**
```csharp
// Temel try-catch
try
{
    int result = 10 / 0;
}
catch (DivideByZeroException ex)
{
    Console.WriteLine($"Hata: {ex.Message}");
}
catch (Exception ex)
{
    Console.WriteLine($"Genel hata: {ex.Message}");
}
finally
{
    Console.WriteLine("Her durumda çalışır");
}

// Exception filtreleme
try
{
    // Kod
}
catch (Exception ex) when (ex.Message.Contains("özel"))
{
    // Özel hata işleme
}

// Özel exception
public class CustomException : Exception
{
    public CustomException(string message) : base(message)
    {
    }
}

// throw kullanımı
if (condition)
{
    throw new CustomException("Özel hata mesajı");
}
```

## Best Practices
1. **Kod Okunabilirliği**
   - Uygun girintileme
   - Anlamlı koşul ifadeleri
   - Gereksiz iç içe yapılardan kaçınma

2. **Performans**
   - Switch-case kullanımı
   - Döngü optimizasyonu
   - Exception handling maliyeti

3. **Güvenlik**
   - Null kontrolleri
   - Exception handling
   - Input validasyonu

## Kaynaklar
- [C# Control Flow](https://docs.microsoft.com/tr-tr/dotnet/csharp/language-reference/statements/selection-statements)
- [C# Iteration Statements](https://docs.microsoft.com/tr-tr/dotnet/csharp/language-reference/statements/iteration-statements)
- [C# Exception Handling](https://docs.microsoft.com/tr-tr/dotnet/csharp/fundamentals/exceptions/) 