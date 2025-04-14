# Veri Tipleri ve Değişkenler

## Genel Bakış
C# programlama dilinde veri tipleri ve değişkenler, uygulamanın temel yapı taşlarıdır. Bu bölümde, C#'taki temel veri tipleri, değişken tanımlama kuralları ve veri tipi dönüşümleri ele alınacaktır.

## Temel Veri Tipleri

### 1. Değer Tipleri (Value Types)

#### Sayısal Tipler
```csharp
// Tam sayı tipleri
byte b = 255;                    // 0 to 255
short s = 32767;                 // -32,768 to 32,767
int i = 2147483647;              // -2,147,483,648 to 2,147,483,647
long l = 9223372036854775807;    // -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807

// Ondalıklı sayı tipleri
float f = 3.14f;                 // 7 basamak hassasiyet
double d = 3.14159265359;        // 15-16 basamak hassasiyet
decimal dec = 3.14159265359m;    // 28-29 basamak hassasiyet
```

#### Diğer Değer Tipleri
```csharp
// Boolean
bool isTrue = true;
bool isFalse = false;

// Karakter
char c = 'A';

// Enum
enum Days { Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, Sunday }
Days today = Days.Monday;
```

### 2. Referans Tipleri (Reference Types)

#### String
```csharp
string name = "John";
string message = "Merhaba Dünya";
string empty = string.Empty;
```

#### Object
```csharp
object obj = new object();
object number = 42;
object text = "Hello";
```

#### Array
```csharp
int[] numbers = new int[5];
string[] names = new string[] { "Ali", "Veli", "Ayşe" };
```

## Değişken Tanımlama

### Değişken Tanımlama Kuralları
- Değişken isimleri harf veya alt çizgi (_) ile başlamalıdır
- Rakam ile başlayamaz
- Özel karakterler kullanılamaz
- C# anahtar kelimeleri kullanılamaz
- Büyük/küçük harf duyarlıdır

### Değişken Tanımlama Örnekleri
```csharp
// Doğru kullanımlar
int age = 25;
string firstName = "Ahmet";
double _price = 99.99;
bool isValid = true;

// Yanlış kullanımlar
// int 1age = 25;        // Rakam ile başlayamaz
// string first-name = "Ahmet";  // Özel karakter kullanılamaz
// bool if = true;       // Anahtar kelime kullanılamaz
```

## Veri Tipi Dönüşümleri

### 1. Implicit Conversion (Örtük Dönüşüm)
```csharp
int i = 10;
long l = i;  // int'ten long'a otomatik dönüşüm

float f = 3.14f;
double d = f;  // float'tan double'a otomatik dönüşüm
```

### 2. Explicit Conversion (Açık Dönüşüm)
```csharp
double d = 3.14;
int i = (int)d;  // double'dan int'e açık dönüşüm

long l = 1000;
int i = Convert.ToInt32(l);  // Convert sınıfı ile dönüşüm
```

### 3. Parse ve TryParse
```csharp
// Parse
string numberStr = "42";
int number = int.Parse(numberStr);

// TryParse
string input = "123";
int result;
if (int.TryParse(input, out result))
{
    Console.WriteLine($"Dönüşüm başarılı: {result}");
}
else
{
    Console.WriteLine("Dönüşüm başarısız");
}
```

## Best Practices

### 1. Değişken İsimlendirme
- Anlamlı isimler kullanın
- camelCase kullanın
- Kısaltmalardan kaçının
- Türkçe karakter kullanmayın

### 2. Veri Tipi Seçimi
- Uygun veri tipini seçin
- Gereksiz büyük tiplerden kaçının
- Hassasiyet gerektiren hesaplamalarda decimal kullanın
- Enum kullanımını tercih edin

### 3. Tip Güvenliği
- var kullanımında dikkatli olun
- Null kontrolü yapın
- TryParse kullanın
- Boxing/Unboxing'den kaçının

## Sık Sorulan Sorular

### 1. var ne zaman kullanılmalıdır?
- LINQ sorgularında
- Anonim tiplerde
- Tip açık olduğunda
- Kod okunabilirliğini artırdığında

### 2. String ve string arasındaki fark nedir?
- string, System.String'in alias'ıdır
- Performans farkı yoktur
- Tercih edilen kullanım string'dir
- İkisi de aynı şeydir

### 3. Nullable tipler nedir?
- Değer tiplerine null değer atanabilmesini sağlar
- ? operatörü ile tanımlanır
- HasValue ve Value özellikleri vardır
- Null kontrolü yapılabilir

## Örnek Kodlar

### 1. Tip Dönüşümü
```csharp
public class TypeConversion
{
    public void ConvertTypes()
    {
        // String to Number
        string strNumber = "123";
        int number = int.Parse(strNumber);
        
        // Number to String
        int value = 42;
        string strValue = value.ToString();
        
        // DateTime
        string dateStr = "2024-01-01";
        DateTime date = DateTime.Parse(dateStr);
    }
}
```

### 2. Nullable Types
```csharp
public class NullableExample
{
    public void HandleNullable()
    {
        int? nullableInt = null;
        if (nullableInt.HasValue)
        {
            int value = nullableInt.Value;
            Console.WriteLine($"Değer: {value}");
        }
        else
        {
            Console.WriteLine("Değer null");
        }
        
        // Null-coalescing operator
        int result = nullableInt ?? 0;
    }
}
```

## Kaynaklar
- [C# Veri Tipleri](https://docs.microsoft.com/tr-tr/dotnet/csharp/language-reference/builtin-types/built-in-types)
- [Değişkenler ve Tipler](https://docs.microsoft.com/tr-tr/dotnet/csharp/language-reference/language-specification/variables)
- [Tip Dönüşümleri](https://docs.microsoft.com/tr-tr/dotnet/csharp/programming-guide/types/casting-and-type-conversions) 