# Koleksiyonlar

## Genel Bakış
Bu bölümde, C#'ta veri yapılarını yönetmek için kullanılan koleksiyon tiplerini ve LINQ sorgularını inceleyeceğiz.

## Mülakat Soruları ve Cevapları

### 1. Temel koleksiyon tipleri nelerdir ve ne zaman kullanılır?
**Cevap:**
Temel Koleksiyonlar:
- **List<T>:**
  - Dinamik boyutlu dizi
  - Sıralı erişim
  - Add, Remove, IndexOf metodları

- **Dictionary<TKey, TValue>:**
  - Key-Value çiftleri
  - Hızlı arama
  - Unique key'ler

- **HashSet<T>:**
  - Unique elemanlar
  - Hızlı arama
  - Set operasyonları

- **Queue<T>:**
  - FIFO (First In First Out)
  - Enqueue, Dequeue
  - Sıralı işlemler

- **Stack<T>:**
  - LIFO (Last In First Out)
  - Push, Pop
  - Geri alma işlemleri

**Örnek Kod:**
```csharp
// List kullanımı
var numbers = new List<int> { 1, 2, 3 };
numbers.Add(4);
numbers.Remove(2);
int index = numbers.IndexOf(3);

// Dictionary kullanımı
var users = new Dictionary<int, string>();
users.Add(1, "Ahmet");
users.Add(2, "Mehmet");
string name = users[1];

// HashSet kullanımı
var uniqueNumbers = new HashSet<int> { 1, 2, 3 };
uniqueNumbers.Add(2); // Tekrar eklenmez
bool contains = uniqueNumbers.Contains(2);

// Queue kullanımı
var queue = new Queue<string>();
queue.Enqueue("İlk");
queue.Enqueue("İkinci");
string first = queue.Dequeue();

// Stack kullanımı
var stack = new Stack<string>();
stack.Push("Alt");
stack.Push("Üst");
string top = stack.Pop();
```

### 2. LINQ nedir ve nasıl kullanılır?
**Cevap:**
LINQ (Language Integrated Query):
- Veri sorgulama
- Method ve Query syntax
- Deferred execution
- Extension methods

**Örnek Kod:**
```csharp
// Method syntax
var numbers = new List<int> { 1, 2, 3, 4, 5 };
var evenNumbers = numbers.Where(n => n % 2 == 0)
                        .OrderByDescending(n => n)
                        .ToList();

// Query syntax
var query = from n in numbers
            where n % 2 == 0
            orderby n descending
            select n;

// Aggregation
int sum = numbers.Sum();
double avg = numbers.Average();
int max = numbers.Max();

// Grouping
var people = new List<Person>
{
    new Person { Name = "Ahmet", Age = 25 },
    new Person { Name = "Mehmet", Age = 30 }
};

var groups = people.GroupBy(p => p.Age);
```

### 3. IEnumerable ve ICollection arasındaki farklar nelerdir?
**Cevap:**
Interface Farkları:
- **IEnumerable<T>:**
  - Sadece okuma
  - foreach desteği
  - Deferred execution

- **ICollection<T>:**
  - Okuma ve yazma
  - Count özelliği
  - Add, Remove, Clear

**Örnek Kod:**
```csharp
// IEnumerable kullanımı
IEnumerable<int> numbers = new List<int> { 1, 2, 3 };
foreach (var number in numbers)
{
    Console.WriteLine(number);
}

// ICollection kullanımı
ICollection<int> collection = new List<int>();
collection.Add(1);
collection.Remove(1);
int count = collection.Count;
```

### 4. Generic koleksiyonlar ve non-generic koleksiyonlar arasındaki farklar nelerdir?
**Cevap:**
Generic vs Non-Generic:
- **Generic:**
  - Tip güvenliği
  - Boxing/Unboxing yok
  - Performans avantajı
  - Modern kullanım

- **Non-Generic:**
  - Object tipinde
  - Boxing/Unboxing var
  - Eski kodlarda kullanım
  - Tip dönüşümü gerekli

**Örnek Kod:**
```csharp
// Generic koleksiyon
List<int> genericList = new List<int>();
genericList.Add(1);  // Tip güvenli

// Non-generic koleksiyon
ArrayList nonGenericList = new ArrayList();
nonGenericList.Add(1);  // Boxing
nonGenericList.Add("string");  // Tip güvensiz
int number = (int)nonGenericList[0];  // Unboxing
```

### 5. Koleksiyon performans optimizasyonu nasıl yapılır?
**Cevap:**
Performans Optimizasyonu:
- Kapasite belirleme
- Uygun koleksiyon seçimi
- LINQ optimizasyonu
- Memory kullanımı

**Örnek Kod:**
```csharp
// Kapasite belirleme
var list = new List<int>(1000);  // Başlangıç kapasitesi

// Dictionary optimizasyonu
var dict = new Dictionary<string, int>(1000, StringComparer.OrdinalIgnoreCase);

// LINQ optimizasyonu
var numbers = new List<int> { 1, 2, 3, 4, 5 };
// Kötü kullanım
var result = numbers.Where(n => n % 2 == 0)
                   .OrderBy(n => n)
                   .ToList()
                   .FirstOrDefault();

// İyi kullanım
var optimized = numbers.FirstOrDefault(n => n % 2 == 0);
```

## Best Practices
1. **Koleksiyon Seçimi**
   - Uygun veri yapısı seçimi
   - Performans gereksinimleri
   - Memory kullanımı

2. **LINQ Kullanımı**
   - Deferred execution
   - Materialization
   - Query optimizasyonu

3. **Thread Safety**
   - Concurrent koleksiyonlar
   - Lock mekanizmaları
   - Immutable koleksiyonlar

## Kaynaklar
- [C# Collections](https://docs.microsoft.com/tr-tr/dotnet/standard/collections/)
- [LINQ in C#](https://docs.microsoft.com/tr-tr/dotnet/csharp/programming-guide/concepts/linq/)
- [Generic Collections](https://docs.microsoft.com/tr-tr/dotnet/standard/generics/collections) 