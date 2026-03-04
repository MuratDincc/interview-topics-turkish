# Big O Notation (Algoritma Karmaşıklığı)

## Genel Bakış

Big O Notation, bir algoritmanın performansını veya karmaşıklığını tanımlamak için kullanılan matematiksel bir gösterimdir. Giriş boyutu arttıkça bir algoritmanın çalışma süresinin (zaman karmaşıklığı) veya bellek kullanımının (alan karmaşıklığı) nasıl büyüdüğünü ifade eder.

Big O, algoritmanın **en kötü senaryodaki** davranışını temsil eder ve sabit katsayılar ile alt mertebeden terimler göz ardı edilir. Bu sayede farklı algoritmalar arasında apples-to-apples karşılaştırması yapılabilir.

### Temel Karmaşıklık Sınıfları (Hızlıdan Yavaşa)

| Notation    | İsim               | Örnek                          |
|-------------|--------------------|--------------------------------|
| O(1)        | Sabit              | Dizi elemanına index ile erişim |
| O(log n)    | Logaritmik         | Binary search                  |
| O(n)        | Lineer             | Doğrusal arama                 |
| O(n log n)  | Lineer-logaritmik  | Merge sort, Quick sort (ort.)  |
| O(n²)       | Karesel            | Bubble sort, Selection sort    |
| O(2^n)      | Üstel              | Fibonacci (naive recursive)    |
| O(n!)       | Faktöriyel         | Travelling Salesman (brute)    |

### Zaman Karmaşıklığı vs Alan Karmaşıklığı

- **Zaman Karmaşıklığı (Time Complexity):** Giriş boyutuna bağlı olarak algoritmanın kaç işlem yaptığını ölçer.
- **Alan Karmaşıklığı (Space Complexity):** Algoritmanın çalışması için gereken ekstra bellek miktarını ölçer.

Zaman ve alan karmaşıklığı arasında genellikle bir denge (trade-off) vardır: daha hızlı bir algoritma çoğunlukla daha fazla bellek kullanır.

### Best Case, Average Case, Worst Case

- **Best Case (En İyi Durum):** Algoritmanın en hızlı çalıştığı senaryo. Omega (Ω) notasyonu ile gösterilir.
- **Average Case (Ortalama Durum):** Rastgele giriş için beklenen ortalama performans. Theta (Θ) notasyonu ile gösterilir.
- **Worst Case (En Kötü Durum):** Algoritmanın en yavaş çalıştığı senaryo. Big O (O) notasyonu ile gösterilir.

---

## Mülakat Soruları ve Cevapları

### 1. Soru

**Big O Notation nedir ve neden önemlidir? O(1), O(n) ve O(n²) arasındaki farkı bir C# örneğiyle açıklayın.**

**Cevap:**

Big O Notation, bir algoritmanın ölçeklenebilirliğini ölçmek için kullanılan evrensel bir standarttır. Farklı makine hızlarından bağımsız olarak algoritmaları karşılaştırmamıza olanak tanır. Üç temel karmaşıklık:

- **O(1) - Sabit:** Giriş boyutundan bağımsız olarak her zaman aynı sürede çalışır.
- **O(n) - Lineer:** Giriş boyutuyla doğru orantılı büyür.
- **O(n²) - Karesel:** İç içe döngülerde ortaya çıkar; giriş iki katına çıkarsa çalışma süresi dört katına çıkar.

**Örnek Kod:**

```csharp
using System;
using System.Collections.Generic;

public class BigOExamples
{
    // O(1) - Sabit Karmaşıklık
    // Giriş boyutundan bağımsız olarak her zaman tek bir işlem yapar.
    public static int GetFirstElement(int[] array)
    {
        return array[0]; // Her zaman 1 adım, O(1)
    }

    // O(1) - Dictionary erişimi de sabit zamanlıdır
    public static bool ContainsKey(Dictionary<string, int> dict, string key)
    {
        return dict.ContainsKey(key); // Hash tablosu nedeniyle O(1) ortalama
    }

    // O(n) - Lineer Karmaşıklık
    // n eleman için n adım gerekir.
    public static int FindMax(int[] array)
    {
        int max = array[0];
        foreach (int item in array) // n kez döner
        {
            if (item > max)
                max = item;
        }
        return max;
    }

    // O(n²) - Karesel Karmaşıklık
    // İç içe iki döngü: n * n = n² adım
    public static void BubbleSort(int[] array)
    {
        int n = array.Length;
        for (int i = 0; i < n - 1; i++)       // n kez döner
        {
            for (int j = 0; j < n - i - 1; j++) // n kez döner
            {
                if (array[j] > array[j + 1])
                {
                    // Swap
                    (array[j], array[j + 1]) = (array[j + 1], array[j]);
                }
            }
        }
    }

    public static void Demo()
    {
        int[] data = { 64, 34, 25, 12, 22, 11, 90 };

        Console.WriteLine($"İlk eleman: {GetFirstElement(data)}");       // O(1)
        Console.WriteLine($"Maksimum: {FindMax(data)}");                   // O(n)
        BubbleSort(data);                                                  // O(n²)
        Console.WriteLine($"Sıralı: {string.Join(", ", data)}");
    }
}
```

---

### 2. Soru

**O(log n) karmaşıklığı neden bu kadar verimlidir? Binary Search algoritmasını C# ile implement edin.**

**Cevap:**

O(log n), her adımda problem boyutunu yarıya indiren algoritmalarda ortaya çıkar. Bu, n = 1.000.000 eleman için yalnızca yaklaşık 20 adım anlamına gelir (log₂(1.000.000) ≈ 20). Binary Search, sıralı bir dizide çalışır ve her karşılaştırmada arama alanını yarıya böler.

**Örnek Kod:**

```csharp
using System;

public class BinarySearchExample
{
    // O(log n) - İteratif Binary Search
    public static int BinarySearch(int[] sortedArray, int target)
    {
        int left = 0;
        int right = sortedArray.Length - 1;

        while (left <= right)
        {
            int mid = left + (right - left) / 2; // Overflow'u önlemek için

            if (sortedArray[mid] == target)
                return mid;          // Bulundu: O(1) en iyi durum

            if (sortedArray[mid] < target)
                left = mid + 1;      // Sağ yarıya bak
            else
                right = mid - 1;     // Sol yarıya bak
        }

        return -1; // Bulunamadı
    }

    // O(log n) - Recursive Binary Search
    public static int BinarySearchRecursive(int[] sortedArray, int target, int left, int right)
    {
        if (left > right)
            return -1;

        int mid = left + (right - left) / 2;

        if (sortedArray[mid] == target)
            return mid;

        if (sortedArray[mid] < target)
            return BinarySearchRecursive(sortedArray, target, mid + 1, right);
        else
            return BinarySearchRecursive(sortedArray, target, left, mid - 1);
    }

    public static void Demo()
    {
        int[] sorted = { 2, 5, 8, 12, 16, 23, 38, 56, 72, 91 };
        int target = 23;

        int index = BinarySearch(sorted, target);
        Console.WriteLine($"{target} değeri index {index}'te bulundu."); // index 5

        // Karşılaştırma: 10 elemanlı dizide Binary Search en fazla 4 adım atar
        // Linear Search ise en kötü durumda 10 adım atar
    }
}
```

---

### 3. Soru

**.NET koleksiyonlarının (List\<T\>, Dictionary\<T,V\>, HashSet\<T\>, LinkedList\<T\>) zaman karmaşıklıklarını karşılaştırın. Hangi durumda hangisini kullanırsınız?**

**Cevap:**

Her koleksiyonun farklı operasyonlar için farklı karmaşıklıkları vardır. Doğru veri yapısını seçmek performans açısından kritiktir.

| Operasyon       | List\<T\> | Dictionary\<K,V\> | HashSet\<T\> | LinkedList\<T\> | SortedSet\<T\> |
|-----------------|-----------|-------------------|--------------|-----------------|----------------|
| Erişim (index)  | O(1)      | O(1)*             | -            | O(n)            | O(log n)       |
| Arama           | O(n)      | O(1)*             | O(1)*        | O(n)            | O(log n)       |
| Ekleme (sona)   | O(1)*     | O(1)*             | O(1)*        | O(1)            | O(log n)       |
| Ekleme (başa)   | O(n)      | -                 | -            | O(1)            | O(log n)       |
| Silme           | O(n)      | O(1)*             | O(1)*        | O(1)**          | O(log n)       |
| İçerme kontrolü | O(n)      | O(1)*             | O(1)*        | O(n)            | O(log n)       |

*Ortalama durum (amortized veya hash çakışması olmadığında)
**Node referansı bilindiğinde

**Örnek Kod:**

```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;

public class CollectionComplexity
{
    public static void CompareCollections()
    {
        int n = 1_000_000;
        var list = new List<int>();
        var hashSet = new HashSet<int>();
        var dictionary = new Dictionary<int, string>();
        var linkedList = new LinkedList<int>();
        var sortedSet = new SortedSet<int>();

        // Veri doldurma
        for (int i = 0; i < n; i++)
        {
            list.Add(i);
            hashSet.Add(i);
            dictionary[i] = $"value_{i}";
            linkedList.AddLast(i);
            sortedSet.Add(i);
        }

        int searchTarget = 999_999;
        var sw = Stopwatch.StartNew();

        // List<T> - O(n) arama
        sw.Restart();
        bool listResult = list.Contains(searchTarget);
        sw.Stop();
        Console.WriteLine($"List<T> Contains:      {sw.ElapsedMicroseconds} µs (O(n))");

        // HashSet<T> - O(1) ortalama arama
        sw.Restart();
        bool hashSetResult = hashSet.Contains(searchTarget);
        sw.Stop();
        Console.WriteLine($"HashSet<T> Contains:   {sw.ElapsedMicroseconds} µs (O(1) avg)");

        // Dictionary<K,V> - O(1) ortalama arama
        sw.Restart();
        bool dictResult = dictionary.ContainsKey(searchTarget);
        sw.Stop();
        Console.WriteLine($"Dictionary ContainsKey:{sw.ElapsedMicroseconds} µs (O(1) avg)");

        // SortedSet<T> - O(log n) arama
        sw.Restart();
        bool sortedResult = sortedSet.Contains(searchTarget);
        sw.Stop();
        Console.WriteLine($"SortedSet<T> Contains: {sw.ElapsedMicroseconds} µs (O(log n))");
    }

    // Doğru koleksiyonu seçme rehberi
    public static void DataStructureSelectionGuide()
    {
        // Senaryo 1: Sık sık "içeriyor mu?" sorgusu yapılacaksa → HashSet<T>
        var visitedUrls = new HashSet<string>();
        visitedUrls.Add("https://example.com");
        bool alreadyVisited = visitedUrls.Contains("https://example.com"); // O(1)

        // Senaryo 2: Key-value eşleştirmesi gerekiyorsa → Dictionary<K,V>
        var userCache = new Dictionary<int, string>();
        userCache[1] = "Ahmet";
        string userName = userCache[1]; // O(1)

        // Senaryo 3: Sıralı veri ve hızlı arama gerekiyorsa → SortedSet<T>
        var leaderboard = new SortedSet<int> { 100, 250, 75, 300 };
        // Otomatik sıralı: 75, 100, 250, 300

        // Senaryo 4: Başa/sona sık ekleme/silme yapılacaksa → LinkedList<T>
        var recentItems = new LinkedList<string>();
        recentItems.AddFirst("en yeni"); // O(1)
        recentItems.AddLast("en eski");  // O(1)
        recentItems.RemoveFirst();       // O(1)

        // Senaryo 5: Index ile erişim ve sona ekleme gerekiyorsa → List<T>
        var items = new List<string>();
        items.Add("item");    // O(1) amortized
        string first = items[0]; // O(1)
    }
}
```

---

### 4. Soru

**LINQ operasyonlarının zaman karmaşıklıkları nedir? Where, OrderBy, First, Contains gibi metotları karşılaştırın.**

**Cevap:**

LINQ metotları ertelenmiş (deferred) veya anında (immediate) çalışabilir. Karmaşıklıkları, kullandıkları iç algoritmaya bağlıdır.

| LINQ Metodu         | Karmaşıklık  | Açıklama                                    |
|---------------------|--------------|---------------------------------------------|
| `Where`             | O(n)         | Her elemanı bir kez tarar                   |
| `Select`            | O(n)         | Her elemanı bir kez dönüştürür              |
| `First/FirstOrDefault` | O(n)     | En kötü durumda tüm koleksiyonu tarar       |
| `OrderBy`           | O(n log n)   | Quicksort/Mergesort tabanlı                 |
| `Contains` (IEnumerable) | O(n)  | Doğrusal arama                              |
| `Contains` (HashSet) | O(1)        | Hash tablosu kullanır                       |
| `GroupBy`           | O(n)         | Hash tablosu ile gruplar                    |
| `Distinct`          | O(n)         | HashSet kullanır                            |
| `Count`             | O(1) veya O(n) | ICollection ise O(1), değilse O(n)        |
| `ToList/ToArray`    | O(n)         | Tüm koleksiyonu materializes eder           |

**Örnek Kod:**

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

public class LinqComplexity
{
    public static void LinqExamples()
    {
        var numbers = Enumerable.Range(1, 1_000_000).ToList();

        // O(n) - Where: tüm koleksiyonu tarar
        var evenNumbers = numbers.Where(n => n % 2 == 0).ToList();

        // O(n log n) - OrderBy: sıralama algoritması kullanır
        var sorted = numbers.OrderBy(n => n).ToList();

        // O(n) - First ile O(1) farkı:
        // İlk eleman için First() O(1) pratik olarak, ama worst case O(n)
        var firstEven = numbers.First(n => n % 2 == 0); // Pratik O(1)

        // UYARI: Bu pattern O(n log n) + O(n) = O(n log n)
        var topFive = numbers
            .OrderByDescending(n => n) // O(n log n)
            .Take(5)                    // O(1)
            .ToList();

        // Daha verimli alternatif - eğer sadece max n değer lazımsa:
        // PriorityQueue veya özel algoritma kullanın
    }

    // LINQ'da performans tuzakları
    public static void LinqPerformancePitfalls()
    {
        var users = new List<User>
        {
            new User { Id = 1, Name = "Ahmet" },
            new User { Id = 2, Name = "Mehmet" },
            // ... binlerce kullanıcı
        };

        // KÖTÜ: O(n) her Contains çağrısı için, toplam O(n²)
        var targetIds = new List<int> { 1, 2, 3, 4, 5 };
        var slowResult = users.Where(u => targetIds.Contains(u.Id)).ToList();
        // targetIds.Contains() her u için O(n) çalışır → O(n * m)

        // İYİ: HashSet kullanarak O(n + m)
        var targetIdSet = new HashSet<int>(targetIds); // O(m)
        var fastResult = users.Where(u => targetIdSet.Contains(u.Id)).ToList(); // O(n)

        // KÖTÜ: Defalarca Count() çağırmak
        var query = users.Where(u => u.Id > 0);
        if (query.Count() > 0) // O(n) - tüm koleksiyonu sayar
        {
            var first = query.First(); // O(n) - tekrar tarar!
        }

        // İYİ: Any() kullanmak - ilk eşleşmede durur
        if (query.Any()) // O(1) en iyi durumda
        {
            var first = query.First();
        }
    }

    private class User { public int Id { get; set; } public string Name { get; set; } }
}
```

---

### 5. Soru

**İç içe döngüler ve recursive fonksiyonların karmaşıklığını nasıl analiz edersiniz? Fibonacci örneği üzerinden açıklayın.**

**Cevap:**

Karmaşıklık analizi için temel kurallar:
- **Sıralı işlemler:** Karmaşıklıkları toplanır, büyük olan alınır → O(n) + O(n²) = O(n²)
- **İç içe döngüler:** Karmaşıklıklar çarpılır → O(n) * O(n) = O(n²)
- **Recursive fonksiyonlar:** Recurrence relation analizi yapılır

Fibonacci örneği, üstel ve lineer karmaşıklık farkını dramatik biçimde gösterir.

**Örnek Kod:**

```csharp
using System;
using System.Collections.Generic;

public class ComplexityAnalysis
{
    // O(2^n) - Naive recursive Fibonacci
    // Her çağrı iki yeni çağrı yaratır: 2^n toplam çağrı
    public static long FibonacciNaive(int n)
    {
        if (n <= 1) return n;
        return FibonacciNaive(n - 1) + FibonacciNaive(n - 2);
        // F(5) için: F(4)+F(3) → F(3)+F(2)+F(2)+F(1) → ...
        // Çakışan alt problemler tekrar hesaplanır!
    }

    // O(n) - Memoization ile Fibonacci (Top-Down Dynamic Programming)
    // Her alt problem yalnızca bir kez hesaplanır
    private static readonly Dictionary<int, long> _memo = new();

    public static long FibonacciMemo(int n)
    {
        if (n <= 1) return n;
        if (_memo.ContainsKey(n)) return _memo[n]; // O(1) cache hit

        _memo[n] = FibonacciMemo(n - 1) + FibonacciMemo(n - 2);
        return _memo[n];
    }

    // O(n) - Iteratif Fibonacci (Bottom-Up DP)
    // Space complexity: O(1) - sadece 2 değişken tutulur
    public static long FibonacciIterative(int n)
    {
        if (n <= 1) return n;
        long prev2 = 0, prev1 = 1;
        for (int i = 2; i <= n; i++)
        {
            long current = prev1 + prev2;
            prev2 = prev1;
            prev1 = current;
        }
        return prev1;
    }

    // İç içe döngü analizi
    public static void NestedLoopExamples()
    {
        int n = 100;

        // O(n²) - İki iç içe n döngü
        for (int i = 0; i < n; i++)         // n kez
            for (int j = 0; j < n; j++)     // n kez
                Console.Write(".");          // Toplam: n²

        // O(n²/2) = O(n²) - Üçgen şekli
        for (int i = 0; i < n; i++)
            for (int j = i; j < n; j++)     // i'den başlar, ama hala O(n²)
                Console.Write(".");

        // O(n³) - Üç iç içe döngü
        for (int i = 0; i < n; i++)         // n kez
            for (int j = 0; j < n; j++)     // n kez
                for (int k = 0; k < n; k++) // n kez
                    Console.Write(".");      // Toplam: n³

        // O(n log n) - Dış döngü lineer, iç döngü logaritmik
        for (int i = 0; i < n; i++)             // n kez
            for (int j = 1; j < n; j = j * 2)  // log n kez (j ikiye katlanır)
                Console.Write(".");              // Toplam: n log n
    }

    public static void PerformanceComparison()
    {
        Console.WriteLine("Fibonacci karşılaştırması:");
        Console.WriteLine($"F(10) Naive:     {FibonacciNaive(10)}");
        Console.WriteLine($"F(10) Memo:      {FibonacciMemo(10)}");
        Console.WriteLine($"F(10) Iterative: {FibonacciIterative(10)}");

        // F(50) için Naive yaklaşık 2^50 = 1 quadrilyon işlem gerektirir!
        // Memo ve Iteratif sadece 50 adımda tamamlar.
        Console.WriteLine($"F(50) Memo:      {FibonacciMemo(50)}");
        Console.WriteLine($"F(50) Iterative: {FibonacciIterative(50)}");
    }
}
```

---

### 6. Soru

**Amortized complexity (itfa edilmiş karmaşıklık) nedir? List\<T\>'nin dinamik boyutlanmasını örnek verin.**

**Cevap:**

Amortized complexity, bir dizi işlemin ortalama maliyetini ifade eder. Bazı operasyonlar nadiren pahalı olsa da, genel ortalama hala düşük kalır. `List<T>` dinamik yeniden boyutlanması buna klasik bir örnektir.

**Örnek Kod:**

```csharp
using System;
using System.Collections.Generic;

public class AmortizedComplexity
{
    // List<T>'nin iç davranışını simüle eden sınıf
    public class SimpleList<T>
    {
        private T[] _array;
        private int _count;
        private int _resizeCount; // Kaç kez yeniden boyutlandırıldı
        private long _totalCopyOperations; // Toplam kopyalama işlemi

        public SimpleList(int initialCapacity = 4)
        {
            _array = new T[initialCapacity];
            _count = 0;
        }

        // Add metodu: Genellikle O(1), bazen O(n) (resize sırasında)
        // Amortized: O(1)
        public void Add(T item)
        {
            if (_count == _array.Length)
            {
                Resize(); // O(n) ama nadiren çağrılır
            }
            _array[_count++] = item; // O(1) - çoğu zaman bu adım
        }

        private void Resize()
        {
            int newCapacity = _array.Length * 2; // Kapasiteyi iki katına çıkar
            T[] newArray = new T[newCapacity];

            // Mevcut elemanları kopyala: O(n)
            for (int i = 0; i < _count; i++)
            {
                newArray[i] = _array[i];
                _totalCopyOperations++;
            }

            _array = newArray;
            _resizeCount++;
            Console.WriteLine($"  Resize! Yeni kapasite: {newCapacity}");
        }

        public void PrintStats(int totalAdded)
        {
            Console.WriteLine($"Toplam eklenen: {totalAdded}");
            Console.WriteLine($"Resize sayısı: {_resizeCount}");
            Console.WriteLine($"Toplam kopya işlemi: {_totalCopyOperations}");
            Console.WriteLine($"Kopya / Ekleme oranı: {(double)_totalCopyOperations / totalAdded:F2}");
            Console.WriteLine($"Amortized maliyet: O(1) (oran 2.0'dan az)");
        }
    }

    public static void Demo()
    {
        Console.WriteLine("=== List<T> Amortized Complexity Demo ===");
        var list = new SimpleList<int>(initialCapacity: 4);

        int n = 32;
        for (int i = 0; i < n; i++)
        {
            list.Add(i);
        }

        list.PrintStats(n);

        /*
        Çıktı analizi (kapasite 4 ile başlayıp, her seferinde 2 katına çıkar):
        - Kapasite 4→8:  4 eleman kopyalanır
        - Kapasite 8→16: 8 eleman kopyalanır
        - Kapasite 16→32: 16 eleman kopyalanır
        Toplam kopya: 4+8+16 = 28 işlem, 32 ekleme için
        Ortalama: 28/32 = 0.875 kopya/ekleme → O(1) amortized

        Matematiksel ispat: n eleman için toplam kopya = n + n/2 + n/4 + ... ≈ 2n = O(n)
        Her ekleme için ortalama maliyet = O(n)/n = O(1)
        */
    }

    // Gerçek List<T> kapasite davranışı
    public static void RealListCapacity()
    {
        var list = new List<int>();
        Console.WriteLine("Eleman Sayısı | Kapasite");

        for (int i = 0; i < 20; i++)
        {
            list.Add(i);
            Console.WriteLine($"{list.Count,13} | {list.Capacity}");
        }

        // List başlangıç kapasitesi bilinen durumda belirtmek daha verimli:
        var optimizedList = new List<int>(capacity: 1000); // Resize olmaz
        for (int i = 0; i < 1000; i++)
            optimizedList.Add(i); // Her zaman O(1), hiç resize yok
    }
}
```

---

### 7. Soru

**Queue\<T\> ve Stack\<T\> koleksiyonlarının karmaşıklıklarını açıklayın. BFS ve DFS algoritmalarında nasıl kullanılırlar?**

**Cevap:**

`Queue<T>` (Kuyruk) FIFO (First In, First Out) yapısıdır ve BFS (Breadth-First Search) için idealdir. `Stack<T>` ise LIFO (Last In, First Out) yapısıdır ve DFS (Depth-First Search) için kullanılır. Her ikisinin temel operasyonları O(1) amortized karmaşıklığa sahiptir.

| Operasyon | Queue\<T\> | Stack\<T\> |
|-----------|------------|------------|
| Enqueue/Push | O(1) amortized | O(1) amortized |
| Dequeue/Pop | O(1) | O(1) |
| Peek | O(1) | O(1) |
| Contains | O(n) | O(n) |

**Örnek Kod:**

```csharp
using System;
using System.Collections.Generic;

public class GraphTraversal
{
    // BFS - Queue kullanır - O(V + E) karmaşıklık
    // V: düğüm sayısı, E: kenar sayısı
    public static List<int> BreadthFirstSearch(
        Dictionary<int, List<int>> graph, int start)
    {
        var visited = new HashSet<int>(); // O(1) lookup için
        var queue = new Queue<int>();     // FIFO yapısı
        var order = new List<int>();

        queue.Enqueue(start); // O(1)
        visited.Add(start);   // O(1)

        while (queue.Count > 0)
        {
            int node = queue.Dequeue(); // O(1)
            order.Add(node);

            foreach (int neighbor in graph[node])
            {
                if (!visited.Contains(neighbor)) // O(1) HashSet lookup
                {
                    visited.Add(neighbor);    // O(1)
                    queue.Enqueue(neighbor);  // O(1)
                }
            }
        }

        return order;
    }

    // DFS - Stack kullanır - O(V + E) karmaşıklık
    public static List<int> DepthFirstSearch(
        Dictionary<int, List<int>> graph, int start)
    {
        var visited = new HashSet<int>();
        var stack = new Stack<int>(); // LIFO yapısı
        var order = new List<int>();

        stack.Push(start); // O(1)

        while (stack.Count > 0)
        {
            int node = stack.Pop(); // O(1)

            if (!visited.Contains(node))
            {
                visited.Add(node);
                order.Add(node);

                foreach (int neighbor in graph[node])
                {
                    if (!visited.Contains(neighbor))
                        stack.Push(neighbor); // O(1)
                }
            }
        }

        return order;
    }

    public static void Demo()
    {
        // Graf: 0-1, 0-2, 1-3, 1-4, 2-5
        var graph = new Dictionary<int, List<int>>
        {
            { 0, new List<int> { 1, 2 } },
            { 1, new List<int> { 0, 3, 4 } },
            { 2, new List<int> { 0, 5 } },
            { 3, new List<int> { 1 } },
            { 4, new List<int> { 1 } },
            { 5, new List<int> { 2 } }
        };

        var bfsOrder = BreadthFirstSearch(graph, 0);
        Console.WriteLine($"BFS: {string.Join(" → ", bfsOrder)}"); // 0 → 1 → 2 → 3 → 4 → 5

        var dfsOrder = DepthFirstSearch(graph, 0);
        Console.WriteLine($"DFS: {string.Join(" → ", dfsOrder)}"); // 0 → 2 → 5 → 1 → 4 → 3
    }
}
```

---

### 8. Soru

**Doğru veri yapısını seçmek uygulama performansını nasıl etkiler? Gerçek dünya senaryosu üzerinden O(n²) yerine O(n) çözüm üretin.**

**Cevap:**

Yanlış veri yapısı seçimi, teoride çalışan ama pratikte ölçeklenemeyen kod üretir. Aşağıdaki örnek, "iki listede ortak elemanları bul" problemini üç farklı karmaşıklıkla gösterir.

**Örnek Kod:**

```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Linq;

public class RealWorldOptimization
{
    // SENARYO: E-ticaret sitesinde kullanıcı sepetindeki ürünlerin
    // indirimli ürünler listesinde olup olmadığını kontrol et.

    // YAKLAŞIM 1: O(n * m) - İki iç içe döngü
    public static List<int> FindDiscountedItemsNaive(
        List<int> cartItems, List<int> discountedItems)
    {
        var result = new List<int>();
        foreach (int cartItem in cartItems)           // n kez
        {
            foreach (int discItem in discountedItems) // m kez
            {
                if (cartItem == discItem)
                {
                    result.Add(cartItem);
                    break;
                }
            }
        }
        return result; // Toplam: O(n * m)
    }

    // YAKLAŞIM 2: O(n + m) - HashSet kullanarak
    public static List<int> FindDiscountedItemsOptimized(
        List<int> cartItems, List<int> discountedItems)
    {
        // İndirimli ürünleri HashSet'e koy: O(m)
        var discountSet = new HashSet<int>(discountedItems);

        // Her sepet ürünü için O(1) lookup
        return cartItems.Where(item => discountSet.Contains(item)).ToList(); // O(n)
        // Toplam: O(n + m)
    }

    // YAKLAŞIM 3: LINQ Intersect - HashSet tabanlı, O(n + m)
    public static List<int> FindDiscountedItemsLinq(
        List<int> cartItems, List<int> discountedItems)
    {
        return cartItems.Intersect(discountedItems).ToList(); // O(n + m)
    }

    // SENARYO 2: Log dosyasından benzersiz IP adreslerini bul
    public static void FindUniqueIPs()
    {
        // Simüle edilmiş log
        var logEntries = new List<string>
        {
            "192.168.1.1 GET /api/users",
            "10.0.0.1 POST /api/orders",
            "192.168.1.1 GET /api/products",
            "10.0.0.2 GET /api/users",
            "10.0.0.1 DELETE /api/cart"
        };

        // KÖTÜ: List.Contains() ile O(n²)
        var uniqueIpsSlowList = new List<string>();
        foreach (var entry in logEntries)
        {
            string ip = entry.Split(' ')[0];
            if (!uniqueIpsSlowList.Contains(ip)) // O(n) her seferinde!
                uniqueIpsSlowList.Add(ip);
        }

        // İYİ: HashSet ile O(n)
        var uniqueIpsFast = new HashSet<string>();
        foreach (var entry in logEntries)
        {
            string ip = entry.Split(' ')[0];
            uniqueIpsFast.Add(ip); // Duplicate'ları otomatik ignore eder, O(1)
        }

        Console.WriteLine($"Benzersiz IP sayısı: {uniqueIpsFast.Count}");
    }

    public static void BenchmarkComparison()
    {
        int n = 10_000;
        int m = 10_000;

        var cartItems = Enumerable.Range(1, n).ToList();
        var discountedItems = Enumerable.Range(5_000, m).ToList();

        var sw = Stopwatch.StartNew();

        sw.Restart();
        var naiveResult = FindDiscountedItemsNaive(cartItems, discountedItems);
        sw.Stop();
        Console.WriteLine($"O(n*m) Naive:     {sw.ElapsedMilliseconds} ms, {naiveResult.Count} eşleşme");

        sw.Restart();
        var optimizedResult = FindDiscountedItemsOptimized(cartItems, discountedItems);
        sw.Stop();
        Console.WriteLine($"O(n+m) HashSet:   {sw.ElapsedMilliseconds} ms, {optimizedResult.Count} eşleşme");

        sw.Restart();
        var linqResult = FindDiscountedItemsLinq(cartItems, discountedItems);
        sw.Stop();
        Console.WriteLine($"O(n+m) Intersect: {sw.ElapsedMilliseconds} ms, {linqResult.Count} eşleşme");

        /*
        Tipik sonuçlar (n=m=10.000):
        O(n*m) Naive:     ~500 ms
        O(n+m) HashSet:   ~2 ms
        O(n+m) Intersect: ~3 ms

        n=m=100.000 için Naive yaklaşık 50.000 ms (50 saniye!) sürer
        HashSet yaklaşımı hala ~20 ms sürer
        */
    }
}
```

---

## Özet: Big O Karmaşıklık Kılavuzu

### Hangi Yapıyı Ne Zaman Seçmeli?

```csharp
// 1. Hızlı arama (O(1)) gerekiyorsa:
var set = new HashSet<T>();        // Benzersiz elemanlar
var dict = new Dictionary<K, V>(); // Key-value çiftleri

// 2. Sıralı veri ve O(log n) arama gerekiyorsa:
var sortedSet = new SortedSet<T>();
var sortedDict = new SortedDictionary<K, V>();
// veya sıralı dizide Array.BinarySearch()

// 3. Sıraya göre işleme (FIFO) gerekiyorsa:
var queue = new Queue<T>(); // BFS, işlem kuyruğu

// 4. Ters sıraya işleme (LIFO) gerekiyorsa:
var stack = new Stack<T>(); // DFS, geri alma işlemleri

// 5. Başa/sona sık ekleme/silme gerekiyorsa:
var linkedList = new LinkedList<T>(); // O(1) başa ekleme

// 6. Index erişimi ve sona ekleme gerekiyorsa:
var list = new List<T>(); // Genel amaçlı, O(1) index erişim
```

### Karmaşıklık Analizi İpuçları

1. **Sabit faktörleri görmezden gel:** O(2n) = O(n), O(500) = O(1)
2. **Alt mertebe terimleri at:** O(n² + n) = O(n²)
3. **İç içe döngülerde çarp:** İki O(n) döngü → O(n²)
4. **Sıralı bloklarda topla, büyüğünü al:** O(n) + O(n²) = O(n²)
5. **Recursive fonksiyonlarda recurrence ilişkisi kur:** T(n) = 2T(n/2) + O(n) → O(n log n)
6. **LINQ'da dikkatli ol:** `OrderBy` O(n log n), `Contains` O(n) (liste üzerinde)
7. **Amortized analizi unut:** `List<T>.Add()` O(1) amortized, ama bazen O(n)
