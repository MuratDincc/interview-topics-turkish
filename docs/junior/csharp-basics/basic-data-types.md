# Temel Veri Tipleri

## Genel Bakış
Bu bölümde, C#'ta kullanılan temel veri tiplerini, bunların özelliklerini ve kullanım alanlarını inceleyeceğiz.

## Mülakat Soruları ve Cevapları

### 1. Value Types ve Reference Types arasındaki temel farklar nelerdir?
**Cevap:**
Value Types:
- Stack'te saklanır
- Değer kopyalama yapar
- Struct, enum ve primitive tipler
- Null olamaz (nullable olmadıkça)
- Daha hızlı erişim

Reference Types:
- Heap'te saklanır
- Referans kopyalama yapar
- Class, interface, delegate, array
- Null olabilir
- Daha yavaş erişim

**Örnek Kod:**
```csharp
// Value Type örneği
int a = 10;
int b = a; // Değer kopyalama
b = 20;    // a değişmez

// Reference Type örneği
class Person { public string Name; }
Person p1 = new Person { Name = "Ahmet" };
Person p2 = p1;      // Referans kopyalama
p2.Name = "Mehmet";  // p1.Name de değişir
```

### 2. Primitive veri tipleri nelerdir ve ne zaman kullanılır?
**Cevap:**
Primitive Tipler:
- **Tam Sayılar:**
  - byte (0-255)
  - short (-32,768 to 32,767)
  - int (-2,147,483,648 to 2,147,483,647)
  - long (çok büyük sayılar)

- **Ondalıklı Sayılar:**
  - float (7 basamak hassasiyet)
  - double (15-16 basamak hassasiyet)
  - decimal (28-29 basamak hassasiyet)

- **Diğer:**
  - char (tek karakter)
  - bool (true/false)

**Örnek Kod:**
```csharp
// Tam sayı örnekleri
byte age = 25;
int population = 85000000;
long distance = 15000000000L;

// Ondalıklı sayı örnekleri
float pi = 3.14f;
double gravity = 9.81;
decimal price = 99.99m;

// Diğer tipler
char grade = 'A';
bool isActive = true;
```

### 3. Nullable tipler nedir ve nasıl kullanılır?
**Cevap:**
Nullable Tipler:
- Value type'lara null değer atama imkanı
- `?` operatörü ile tanımlanır
- `HasValue` ve `Value` özellikleri
- Null-coalescing operatörü (`??`)

**Örnek Kod:**
```csharp
// Nullable tanımlama
int? nullableInt = null;
DateTime? nullableDate = null;

// Null kontrolü
if (nullableInt.HasValue)
{
    int value = nullableInt.Value;
}

// Null-coalescing operatörü
int result = nullableInt ?? 0;

// Null-conditional operatörü
string name = person?.Name ?? "Bilinmiyor";
```

### 4. Tip dönüşümleri (Type Conversion) nasıl yapılır?
**Cevap:**
Tip Dönüşümleri:
- **Implicit Conversion:**
  - Otomatik dönüşüm
  - Veri kaybı yok
  - Küçük tip -> Büyük tip

- **Explicit Conversion:**
  - Manuel dönüşüm
  - Veri kaybı olabilir
  - Cast operatörü kullanımı

- **Convert Sınıfı:**
  - Güvenli dönüşüm
  - Exception handling
  - TryParse metodu

**Örnek Kod:**
```csharp
// Implicit conversion
int i = 10;
long l = i;  // Otomatik dönüşüm

// Explicit conversion
double d = 10.5;
int i = (int)d;  // Manuel dönüşüm

// Convert sınıfı
string str = "123";
int number = Convert.ToInt32(str);

// TryParse
string input = "123";
if (int.TryParse(input, out int result))
{
    // Başarılı dönüşüm
}
```

### 5. Boxing ve Unboxing nedir ve performans etkileri nelerdir?
**Cevap:**
Boxing/Unboxing:
- Boxing: Value type -> Reference type
- Unboxing: Reference type -> Value type
- Performans maliyeti yüksek
- Mümkünse kaçınılmalı

**Örnek Kod:**
```csharp
// Boxing örneği
int i = 123;
object o = i;  // Boxing

// Unboxing örneği
int j = (int)o;  // Unboxing

// Performans etkisi
var stopwatch = new Stopwatch();
stopwatch.Start();

// Boxing/Unboxing içeren kod
for (int k = 0; k < 1000000; k++)
{
    object boxed = k;  // Boxing
    int unboxed = (int)boxed;  // Unboxing
}

stopwatch.Stop();
Console.WriteLine($"Boxing/Unboxing süresi: {stopwatch.ElapsedMilliseconds}ms");
```

## Best Practices
1. **Veri Tipi Seçimi**
   - Uygun boyutta tip seçin
   - Gereksiz büyük tiplerden kaçının
   - Nullable kullanımını sınırlayın

2. **Tip Dönüşümleri**
   - Implicit conversion tercih edin
   - TryParse kullanın
   - Boxing/Unboxing'den kaçının

3. **Performans**
   - Struct kullanımını değerlendirin
   - Referans tiplerini dikkatli kullanın
   - Memory allocation'ı minimize edin

## Kaynaklar
- [C# Veri Tipleri](https://docs.microsoft.com/tr-tr/dotnet/csharp/language-reference/builtin-types/built-in-types)
- [Nullable Value Types](https://docs.microsoft.com/tr-tr/dotnet/csharp/language-reference/builtin-types/nullable-value-types)
- [Type Conversion](https://docs.microsoft.com/tr-tr/dotnet/csharp/programming-guide/types/casting-and-type-conversions) 